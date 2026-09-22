---
title: 🛡 家庭服务器的安全加固：从端口映射到 CrowdSec 联动封禁
summary: 一个真实演进的加固过程 —— 暴露面收敛、入侵检测联动、限流防风控、以及几个"防护机制反噬自己"的坑。
date: 2026-09-18
authors:
  - admin
tags:
  - 安全
  - CrowdSec
  - nginx
  - 运维
---

> 自托管服务的第一个念头是"怎么把它跑起来"，第二个念头应该是"怎么别让它被人玩坏"。这篇是第二个念头的完整记录。

## 一、先收敛暴露面

在装任何防护软件之前，先做减法。我的检查清单：

```bash
# 哪些端口真的在监听
ss -tlnp

# 容器端口是否误暴露到公网
docker ps --format '{{.Names}}\t{{.Ports}}'
```

三条原则：

1. **容器端口只监听内网**。宁可 `127.0.0.1:2283` 也不要 `0.0.0.0:2283`。
2. **对外只保留必要的入口**。一个 nginx 做统一入口，比十个服务各自开端口好管一百倍。
3. **默认端口不是安全边界**，但也别把管理面板挂在 80 上 —— SSH、面板、数据库都不该出现在公网扫描的常见命中列表里。

## 二、CrowdSec：用社区情报做本地防护

CrowdSec 的架构比 fail2ban 更值得用：

```
日志解析 → 场景判定 → 决策（decision）→ LAPI 存储 → bouncer 执行封禁
```

我用的部署是**两级**的：

- **应用层节点**：跑 LAPI（决策中心）+ 日志解析器，:8080 提供决策 API
- **边缘节点**：跑 bouncer，从远程 LAPI 拉决策，在 nginx 层拒绝请求

这样的好处是**决策与执行解耦**：多个入口（nginx / 防火墙 / 其他主机）可以共享同一份黑名单。

nginx bouncer 的最小配置：

```nginx
# http{} 内
# ...

server {
    listen 8888 ssl;

    location / {
        # bouncer 通过 auth_request 或 lua 拦截
        # 命中黑名单 → 403
    }
}
```

## 三、坑一：限流区定义位置

给防护页面加限流防刷，我第一次这么写：

```nginx
server {
    listen 8888;
    # ❌ 这里不行
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
}
```

nginx 直接起不来。原因：**`limit_req_zone` 是 `http{}` 级别的指令**，必须写在 `http{}` 内、`server{}` 之外。

正确写法：

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;

    server {
        listen 8888;
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_req_status 429;
        }
    }
}
```

`burst` 是允许的突发额度，`nodelay` 表示突发部分立即放行（不排队）。防护类页面上线时这两个参数很关键 —— 小主机上排队比拒绝更危险。

## 四、坑二：防护机制把自己封了

这是最值得记录的一个坑，因为它属于"机制反噬"：

> 我做压测时，防护组件判定**来源 IP 有攻击行为**，于是把发起压测的 IP 封禁了。而这个 IP 恰好是我自己 SSH 用的出口 IP —— 结果表现为 **SSH 假死**，看起来像网络断了，实际是被自己的防护墙拒绝。

排查过程很有代表性：

```bash
# 1. 首先怀疑网络
ping <host>              # 通？
ssh <host>               # 挂？

# 2. 从另一个网络位置尝试
# → 能连上 → 说明服务活着，是"某个来源"被拒

# 3. 直接查决策表
cscli decisions list

# 4. 找到自己的 IP，删除
cscli decisions delete --ip <your-ip>
```

**结论与对策：**

- 压测/扫描**永远从第三方网络位置发起**，不要用管理出口 IP
- 给管理 IP 加**白名单**（trusted IPs），比事后救火靠谱
- 记住 `cscli decisions delete --ip <x>` 这条救命命令
- 反过来这证明防护是**真的在工作** —— 一个从不误封的 IDS 通常也没在工作

## 五、坑三：光猫映射只能增量改

运营商光猫的端口映射接口有个危险特性：**某些操作是整体替换而非单条修改**。我踩过一次映射表被覆盖，服务全断。

现在我的流程固定为：

```
1. 先导出/记录当前完整映射表
2. 只对目标条目做 add-only 操作（新增，绝不批量提交）
3. 立即读回校验：新增条目在？旧条目还在？
4. 任何异常 → 用步骤 1 的记录回滚
```

同样适用于任何"配置整体提交"的管理面板 —— **先备份，再增量，后校验。**

## 六、坑四：CGI 的 `fastcgi_params` 陷阱

我想用 nginx + fcgiwrap 提供一个输出网卡状态的探针。配置写完，所有请求 404：

```nginx
location /eth0 {
    include fastcgi_params;          # ❌ 元凶
    fastcgi_pass unix:/run/fcgiwrap.socket;
}
```

原因：`fastcgi_params` 里有一行

```
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
```

它会**覆盖**你设定的 `SCRIPT_FILENAME`，指向一个不存在的路径，于是 CGI 找不到脚本 → 404。

正确做法是**手工列出需要的参数**，不要 include 这个文件：

```nginx
location /eth0 {
    fastcgi_pass unix:/run/fcgiwrap.socket;
    fastcgi_param SCRIPT_FILENAME /path/to/script.sh;
    fastcgi_param QUERY_STRING    $query_string;
    fastcgi_param REQUEST_METHOD  $request_method;
    fastcgi_param CONTENT_TYPE    $content_type;
    fastcgi_param SERVER_PROTOCOL $server_protocol;
}
```

## 七、坑五：域名到期 = 安全事件

最后这个坑最容易被忽视，但后果最严重：

站点配置里留着一个旧域名的引用。那个域名**我没有续费**。结果：

1. 域名掉落后被**抢注**
2. 抢注者配置了**通配解析**，所有子域指向一个诈骗页面
3. 我的站点 canonical / sitemap / RSS **全部仍然指向旧域名**
4. 搜索引擎和社交分享卡片全部导流到诈骗站

**技术上站点是正常的，但对外身份被劫持了。**

对策清单：

- 定期检查所有在用的域名到期时间（whois / RDAP）
- 域名一旦不打算再用，**立刻从所有配置中清除**，包括 canonical、sitemap、RSS、`server_name`、`/etc/hosts`
- 迁移域名时**检查 baseURL 这类全局变量** —— 只改 DNS 和证书是不够的
- 搜索引擎收录是滞后的，修复后要等重新抓取

## 八、小结

| 层次 | 措施 |
|---|---|
| 暴露面 | 端口收敛 + 容器端口内网化 + 统一入口 |
| 检测 | CrowdSec 日志解析 + 场景判定 |
| 阻断 | bouncer 在边缘执行，决策集中管理 |
| 抗压 | `limit_req` + 热缓存读写分离 |
| 变更安全 | 备份 → 增量 → 校验 → 可回滚 |
| 身份安全 | 域名到期监控 + 全局变量一致性检查 |

**安全加固不是装一个软件，而是让每一次变更都可回滚、每一份对外身份都可追溯。**
