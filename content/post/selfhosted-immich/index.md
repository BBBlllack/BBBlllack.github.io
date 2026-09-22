---
title: 🖼 自建照片库 Immich：从手机相册到私有 AI 图库
summary: 用 Docker 全家桶搭建一套带智能搜索、人脸识别、地理定位的私有照片管理平台，并把它接入家庭网关。
date: 2026-09-16
authors:
  - admin
tags:
  - Immich
  - Docker
  - 照片管理
---

> 把照片交给云厂商，本质是用隐私换便利。这篇记录我用 Immich 把这个交换关系反过来：**便利我全要，隐私我自己留。**

## 一、为什么不用网盘 + 相册 App

传统方案是"网盘存文件 + 手机相册 App 看"。问题是：

- 网盘不懂照片，没有时间线、没有地图、没有"找那只猫"
- 相册 App 不懂备份，换手机要手动导
- 两边的元数据（时间、地点、人物）永远对不齐

Immich 的价值在于**它同时是备份工具和相册应用**，元数据从上传那一刻就统一。

## 二、部署形态

标准 Immich 是四容器结构：

```
immich-server      ← API + Web UI + 任务调度
immich-machine-learning  ← CLIP 模型推理 / 人脸检测
immich-postgres    ← 元数据库
immich-redis       ← 队列与缓存
```

关键设计点：

**1. 数据与容器分离。** 上传目录（`UPLOAD_LOCATION`）和 PostgreSQL 数据目录必须是宿主机绑定挂载，而不是匿名卷 —— 否则容器一重建，照片就没了。

**2. 对外只走 TLS 反代。** Immich 本身监听 2283（HTTP），我用 nginx 在 2284 上做了 HTTPS 反代。手机 App 填的是 `https://<域名>:2284`，不是裸端口。

```nginx
server {
    listen 2284 ssl;
    server_name home.***.cn;

    client_max_body_size 0;   # 相册上传不限体积

    location / {
        proxy_pass http://127.0.0.1:2283;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

`client_max_body_size 0` 是必须的 —— 默认 1MB 会让上传静默失败。`Upgrade`/`Connection` 两个头是为了 WebSocket（移动端上传进度实时回传）。

**3. 高位端口 + SNI。** 在 80/443 受限的网络环境里，HTTPS 挪到 2284 端口仍然能正常握手 —— TLS 不关心端口号，只关心 SNI。

## 三、智能搜索的真实边界

Immich 的智能搜索基于 CLIP 模型（默认 `ViT-B-32__openai`）。这里有一个**必须知道的坑**：

> 默认模型的文本编码器是**纯英文**的。搜 `cat` 能出结果，搜 `猫` 静默返回空。

不是 bug，是模型的词表限制。三条应对路径：

1. **换多语言模型**：需要重新跑一遍索引，代价大且显存要求更高。
2. **用英文检索 + 内部约定**：把常搜的类别固定成英文词（`cat`/`sunset`/`food`/`document`），写进自己的使用习惯。
3. **依赖人脸与地点这类非文本维度** —— 它们不经过文本编码器，中文环境完全可用。

我选了 2 + 3 的组合。人脸识别用的是 InsightFace 的 `buffalo_l` 模型包，识别与聚类都在本地完成，人名只存在你自己的数据库里。

## 四、按人检索的 API

除了 Web UI，Immich 的 REST API 允许按人物 ID 直接检索：

```bash
POST /api/search/metadata
{
  "personIds": ["<person-uuid>"],
  "size": 100
}
```

配合 `/api/people` 列出已聚类的所有人物，就能做出"某个人的全部照片"这种自定义视图 —— 比如生成家庭相册、或者做年度回顾。

## 五、资源占用：别被瞬时峰值骗了

一个真实教训：监控告警说 Immich 峰值吃了 1GB 内存，接近宿主上限。第一反应是"要加内存了"，但拉时间线一看：

```
峰值时间：每天 00:00 前后
持续时间：数分钟
原因：后台扫描任务（缩略图生成 / 人脸检测 / 智能索引）
```

**这是周期性批处理，不是常驻开销。** 稳态内存远低于峰值。结论是：

- 判断是否需要扩容，要看**稳态 + P95**，不是看单点峰值
- 批处理任务可以限流（`IMMICH_...` 并发参数），避开高峰期
- 如果宿主真的很小，把 ML 推理拆到另一台机器上（远程 ML 后端）

## 六、远程 ML 后端：把小主机的活外包出去

当宿主的 CPU 实在扛不住 CLIP 推理时，Immich 支持把机器学习服务拆出去：

```yaml
# immich-server 侧
environment:
  IMMICH_MACHINE_LEARNING_URL: http://<remote-host>:3003
```

远程那边只要跑 `immich-machine-learning` 一个容器，通过 SSH 隧道或内网直连暴露 3003 端口即可。这样**照片库留在本地，算力借用别处** —— 数据不动，计算下沉。

## 七、复盘

| 维度 | 结论 |
|---|---|
| 存储 | 照片目录必须挂宿主机，容器要能随便删 |
| 访问 | 对外只给 TLS 反代，端口用高位 |
| 搜索 | 默认 CLIP 是英文的，中文搜索要么换模型要么改用英文词 |
| 人脸 | 本地模型，人名不上云，中文环境无影响 |
| 容量 | 看稳态不看峰值；峰值多是后台扫描 |
| 扩展 | 算力不够就拆 ML 后端，别动数据 |

一句话：**自建照片库最难的不是装，而是想清楚哪些数据必须留在自己手里。**
