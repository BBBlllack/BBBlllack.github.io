---
title: SpringCloud学习笔记
summary: 在学习SpringCloud记录的一些信息, 但是学到后面就懒得记了...实在不好意思。
date: 2024-12-20
type: docs
math: false
tags:
  - Java SpringCloud
  
---

### 1. 版本兼容问题

#### 1.1 cloud和boot

兼容方式如下所示

| Spring Cloud 版本        | Spring Boot 版本范围                                      |
|------------------------|---------------------------------------------------------|
| 2021.0.1-SNAPSHOT      | >=2.6.4-SNAPSHOT and <2.7.0-M1                           |
| 2021.0.0               | >=2.6.1 and <2.6.4-SNAPSHOT                              |
| 2021.0.0-RC1           | >=2.6.0-RC1 and <2.6.1                                    |
| 2021.0.0-M3            | >=2.6.0-M3 and <2.6.0-RC1                                |
| 2021.0.0-M1            | >=2.6.0-M1 and <2.6.0-M3                                 |
| 2020.0.5               | >=2.4.0.M1 and <2.6.0-M1                                 |
| Hoxton.SR12            | >=2.2.0.RELEASE and <2.4.0.M1                            |
| Hoxton.BUILD-SNAPSHOT  | >=2.2.0.BUILD-SNAPSHOT                                   |
| Hoxton.M2              | >=2.2.0.M4 and <=2.2.0.M5                                |
| Greenwich.BUILD-SNAPSHO| >=2.1.9.BUILD-SNAPSHOT and <2.2.0.M4                     |


### 2. Eureka注册中心

#### 2.1 Eureka服务端搭建

eureka的搭建分三步走，如下所示：

- 导入相关依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

- 编写启动类，添加注解 @EnableEurekaServer

```java
package com.shj.eureka;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // 此注解启动了eureka
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }
}
```

- 添加配置文件信息

```yaml
server:
  port: 10086
spring:
  application:
    name: eurekaserver # 应用程序名
eureka:
  client:
    service-url: # eureka 的地址信息
      defaultZone: http://127.0.0.1:10086/eureka
```



#### 2.2 Eureka 客户端注册

eureka的客户端注册分两边进行，如下所示：

- 添加依赖信息

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

- 添加配置文件信息

```yaml
spring:
  application:
    name: userservice # 配置应用程序名称

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10086/eureka # 配置注册到的eureka的服务器地址
```

#### 2.3 服务的拉取

服务拉取是基于服务名称获取服务列表，然后再对服务列表做负载均衡，分如下两步

- 修改访问的url路径，用服务名代替ip和端口

```java
@RestController
@RequestMapping("order")
public class OrderController {
   @Autowired
   private OrderService orderService;
   @Autowired
   private RestTemplate restTemplate;
    @GetMapping("{orderId}")
    public Order queryOrderByUserId(@PathVariable("orderId") Long orderId) {
        Order order = orderService.queryOrderById(orderId);
      	// 在此使用服务名代替ip和端口
        String url = "http://userservice/user/" + order.getUserId();
        // String url = "http://localhost:8081/user/" + order.getUserId();
        User user = restTemplate.getForObject(url, User.class);
        order.setUser(user);
        // 根据id查询订单并返回
        return order;
    }
}
```

- 对RestTemplate使用@LoadBalanced注解做负载均衡

```java
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication // 内含@configuration注解，因此无需单独添加 @configuration
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }

    @Bean
    @LoadBalanced // 此注解完成了负载均衡
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}
```

#### 2.4 负载均衡

### 3. Naocs 注册中心

#### 3.1 配置方式

不同于eureka，nacos是一个单独的客户端，需要进行安装，默认端口为8848，默认访问地址为 http://localhost:8848/nacos/index.html

配置nacos客户端的方法分如下两步：

- 添加nacos客户端依赖

```xml
<!-- nacos客户端依赖包 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

- 在yml文件中添加相应的配置

```yaml
spring:
  cloud:
    nacos: # nacos属于spring的配置
      server-addr: localhost:8848 # 默认地址也是这个
```

#### 3.2 统一配置管理

统一配置管理是在nacos中读取application.yml文件的方式（nacos中配置文件一般不会这样起名），分成如下步骤：

- 导入配置坐标依赖

```xml
<!-- nacos客户端依赖包 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<!-- nacos配置管理依赖包 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

- 引入**bootstrap.yml**文件，在其配置nacos的配置

```yaml
spring:
  application:
    name: userservice # 服务名
  profiles:
    active: dev # 开发环境
  cloud:
    nacos:
      server-addr: localhost:8848 # nacos地址
      config:
        file-extension: yaml # nacos中配置文件扩展名
```

- 在nacos中配置管理新建配置文件，命名要遵守规则 要以name-active.file- extension作为文件名，在其配置需要热更新的配置，而非全部配置

#### 3.3 自动配置刷新

我们希望在nacos中更改了对应模块的配置文件之后，无需重启服务即可感知到文件的变化，此时需要用到自动配置刷新，有两种方法实现。

##### 3.3.1 方法一

在使用@value注解注入属性之后，在对应的类添加注解 @RefreshScope：

```java
@Slf4j
@RestController
@RequestMapping("/user")
@RefreshScope // 此注解即可感知到配置文件的变化
public class UserController {

    @Value("${pattern.dataformat}")
    private String dataformat;

    @GetMapping("/now")
    public String now(){
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern(dataformat));
    }
}
```

##### 3.3.2 方法二

- 使用一个类加载所有配置属性，配合注解@ConfigurationProperties

```java
@Data // lombok
@Component // 注册为bean
@ConfigurationProperties(prefix = "pattern") // 此注解实现了配置的自动刷新
public class PatternProprties {
  // 当prefix和属性名拼接在配置文件中能查到对应的，即可注入
    private String dataformat;
}

```

- 在controller中的使用

```java
@Slf4j
@RestController
@RequestMapping("/user")
@RefreshScope
public class UserController {

    @Autowired
    private UserService userService;

    @Autowired
    private PatternProprties patternProprties;

    @GetMapping("/now")
    public String now(){
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern(patternProprties.getDataformat()));
    }
}
```

#### 3.4 环境共享

配置文件中的重复部分有时写到一个文件中即可，此时会产生环境共享的需求，程序从nacos会读取**服务名.yaml，服务名-环境.yaml**文件，本地还有application.yml文件，此时优先级为 **服务名-环境.yaml > 服务名.yaml > 本地配置**，共享的配置佩在 **服务名.yaml** 文件中即可。

### 4. http客户端Feign

#### **4.1 基本使用**

使用feign时分如下几步：

- 导入feign的依赖

```xml
<!--feign的依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- 在启动类上使用注解 @EnableFeignClients 打开feign

```java
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication
@EnableFeignClients // 此注解打开了feign
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

- 编写feign类，调用方法既是发送http请求

```java
@Component
@FeignClient("userservice") // 服务名
public interface UserClient {
    @GetMapping("/user/{id}") // 请求的路径
    public User findById(@PathVariable("id") Long id);
}

```

- 测试使用

```java
@RestController
@RequestMapping("order")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @Autowired
    private UserClient userClient;

    @GetMapping("/{orderId}")
    public Order queryOrderByUserId(@PathVariable("orderId") Long orderId){
        Order order = orderService.queryOrderById(orderId);
      // 在此调用方法即可发送http请求获取user数据
        User user = userClient.findById(order.getUserId());
        order.setUser(user);
        return order;
    }
}
```

#### 4.2 性能优化

可以通过设置feign的日志级别和底层http的实现方式（默认为jdk自带的，性能相对较差，不支持连接池技术）来优化性能，步骤如下所示：

- 日志级别的设置

```yaml
feign:
  client:
    config:
      default:
        logger-level: full # 全部打印， 一般设为 BASIC比较好
```

- 更改底层实现方式

  - 导入feign-httpclient的依赖

  ```xml
  <!--        httpClient的依赖-->
  <dependency>
      <groupId>io.github.openfeign</groupId>
      <artifactId>feign-httpclient</artifactId>
  </dependency>
  ```

  - 在application.yml文件中进行相关配置

  ```yaml
  feign:
    client:
      config:
        default:
          logger-level: basic
    httpclient:
      max-connections: 200 # 最大连接数
      max-connections-per-route: 50 # 每个路径的最大连接数
      enabled: true # 开启feign对httpclient的支持
  ```

### 5. 统一网关Gateway

#### 5.1 基本使用

gateway作为网关，一般用于全部请求到来之前的拦截，之后可以根据访问路径做权限校验，负载均衡等，基本使用方式如下：

- 添加依赖(注意要添加nacos将gateway注册到nacos)

```xml
<!-- nacos客户端依赖包 --> 
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<!-- gateway依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

- 在application.yml中进行配置

```yaml
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848
    gateway:
      routes:
        - id: user-service # 路由的唯一标识
          uri: lb://userservice # lb代表复杂均衡
          predicates:
            - Path=/user/** # 代表拦截user/下所有请求
        - id: order-service # 路由的唯一标识
          uri: lb://orderservice
          predicates:
            - Path=/order/** # 代表拦截order/下所有请求
```

#### 5.2 全局过滤器

gateway提供的过滤器有时无法满足一些开发中的特殊要求，此时需要用到全局过滤器，实现方式如下：

```java
@Component // 注意要定义为bean
public class Filter1 implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest(); // 获取请求对象，注意和servlet的不一样
        MultiValueMap<String, String> queryParams = request.getQueryParams(); // 参数map
        String username = queryParams.getFirst("username");
        if ("zhangsan".equals(username))
            return chain.filter(exchange); // 放行
        exchange.getResponse().setStatusCode(HttpStatus.METHOD_NOT_ALLOWED);
        return exchange.getResponse().setComplete(); // 拦截
    }
  // 此方法为实现了ordered接口之后要实现的，数字越小代表过滤器优先级越高
    @Override
    public int getOrder() {
        return -1;
    }
}
```

#### 5.3 CORS解决跨域请求

gateway中解决跨域请求的方式是基于cors，只需要在yml文件中添加如下配置即可

```yaml
spring:
    gateway:
      globalcors:
        add-to-simple-url-handler-mapping: true # 允许options请求
        cors-configurations:
          '[/**]':
            allowed-origins: "*" # 允许跨域网址
            allowed-headers: "*" # 允许跨域请求头
            allowed-methods: "*" # 允许跨域方法
```

