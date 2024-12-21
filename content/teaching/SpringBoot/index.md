---
title: SpringBoot笔记文档
summary: 在学习SpringBoot的时候记录的一些组件，依赖的使用方式，坐标等信息。
date: 2024-12-21
type: docs
math: false
tags:
  - Java SpringBoot
  
---

### 1. 配置文件

配置文件的三种格式(加载优先级如下顺序：

- application.properties
- application.yml
- Application.yaml

#### 1.1属性的注入

首先在yml文件中配置属性

```yaml
user:
  name1: lisi
  age: 18
```

之后在对应的类中利用**value**注解进行属性读取

```java
@Controller
@RequestMapping("/dr")
public class DataReadController {

    @Value("${user.name1}")
    private String name1;

    @Value("${user.age}")
    private String age1;

    @GetMapping("/na")
    @ResponseBody
    public String getna(){
        System.out.println("name1 = " + name1);
        System.out.println("age1 = " + age1);
        return name1 + "," + age1;
    }
}
```

#### 1.2配置文件yml中属性的引用

```yaml
baseDir: c:/windows
# 利用${}语法格式进引用
tempDir: ${baseDir}/tmp
```

转义字符也可以生效，但是需要利用引号引起来

#### 1.3一次性读取全部数据

一次性读取配置文件中的全部属性，读取到一个Environment对象中，可以利用getProperty方法进行获取

```java
@Autowired
private Environment env;

@GetMapping("/env")
@ResponseBody
public String getenv(){
  System.out.println(env);
  System.out.println(env.getProperty("user.name1"));
  return "";
}
```

#### 1.4读取属性到对象中

- 定义数据模型封装属性
- 将数据模型交给spring管理，作为一个bean
- 利用configurationProperties注解进行配置

```yaml
datasource:
  driver: com.mysql.jdbc.Driver
  url: jdbc:mysql://localhost/springboot_db
  username: root
  password: 12345678
```

**省略了getter和setter方法和tostring方法**

```java
@Component
@ConfigurationProperties(prefix = "datasource") // 一定要小写！
public class MyDataSource {
    private String driver;
    private String url;
    private String username;
    private String password;
}
```

```java
@Autowired
private MyDataSource myDataSource;

@GetMapping("/db")
@ResponseBody
public String getdv(){
    System.out.println(myDataSource);
    return "";
}
```

### 2. 整合第三方技术

**前言：springboot整合第三方技术都遵循两个步骤：**

1. **导入对应的starter**
2. **根据提供的配置格式，配置非默认的配置项**

#### 2.1整合Junit

无需做很多操作，仅仅需要注意一些事情

- 注入要测试的对象
- 执行对应的方法

**注意：**

- 测试类一定要在引导类所在包及其子包下
- 如果不在可以在测试类注解SpringBootTest(classes = xxx.class)指定启动类

```java
@SpringBootTest(classes = ShjApplication.class)
class ShjApplicationTests {

    @Autowired
    private DataReadController dataReadController;

    @Test
    void contextLoads() {
        dataReadController.getna();
    }

}
```

#### 2.2整合Mybatis

##### 2.2.1整合方法

**需要注意的点有：**

- mybatis版本不要太高，可以用2.2.0的版本
- mysql-connector-j手动添加版本号

pom.xml

```xml
<dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.2.0</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
<!--            手动添加版本号-->
            <version>8.0.33</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter-test</artifactId>
            <version>2.2.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
```

application.yml配置如下

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot
    username: root
    password: 12345678
```

测试案例如下

userDao.java

```java
@Mapper
public interface UserDao {
    @Select("select * from users")
    public User[] getUsers();
}
```

Junit测试方法

```java
@Test
void userDaoTest(){
    System.out.println(Arrays.toString(userDao.getUsers()));
}
```

##### **2.2.2动态sql**

动态sql可以根据一些条件，动态的进行sql拼接

```xml
<if> <!-- 此标签用来判断是否拼接标签内的sql语句，如下当属性atr1不为null时才拼接 --> 
  <if test = "art1 != null">
  	<!-- sql -->
  </if>
  
<where> <!-- 此标签用来判断，当where后无条件时，会自动去除多余的where关键词，且做多条件动态查询时，会自动去除多余的and关键字 -->
  <where>
  	<if test = "art1 != null">
  		<!-- condition sql -->
		</if>
    <if test = "art2 != null">
  	and <!-- condition sql -->
  	</if>
  </where>
  
<foreach> <!-- 此标签用来遍历传递的参数，例如集合 -->
  
<set> <!-- 此标签作用类似于where，set用做更新，同样的可以去除多余的set关键词，且做多字段更新时，会自动去除多余的and关键字 -->
	 <set>
    <if test = "art1 != null">
      <!-- update sql -->
    </if>
    <if test = "art2 != null">
    and <!-- update sql -->
    </if>
  </set>
  
<sql id="id1"> <!-- 此标签用来抽取公共部分的sql，id为此sql的唯一标识 -->
<include refid="id1"> <!-- 此标签用来引入公共部分的sql，refid属性指定引入的sql唯一标识 -->
```

#### 2.3整合MybatisPlus

整合mybatis-plus时，由于spring未收录mp的坐标，故我们需要手动导入，整体步骤如下：

- 创建springboot项目时，勾选mysql驱动
- 在application.yml文件中配置datasource
- 导入mybatis-plus坐标
- dao曾继承一个BaseMapper\<T\>接口，传入要操作的实体类

mybatis-plus 3.4.3版本坐标如下所示：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version>
</dependency>
```

application.yml配置如下：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot
    username: root
    password: 12345678
```

dao层书写如下：

```java
@Mapper
public interface BookDao extends BaseMapper<Book> {
}
```

controller层书写如下：

```java
@Slf4j
@RestController
@RequestMapping("/books")
public class BookController {
    R r = new R();
    @Autowired
    private BookDao bookDao;

    @GetMapping
    private R getAllBooks(){
        List<Book> books = bookDao.selectList(null);
        r.setData(books);
        log.info("mp is running...");
        return r;
    }
}
```

#### 2.4整合Druid

同样的，springboot未收录druid的坐标，需要去 www.nvmrepository.com 找到druid的坐标，整体步骤如下

- 导入druid的坐标
- 在application.yml中配置

druid 1.2.6版本的坐标如下：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.6</version>
</dependency>
```

application.yml配置如下：

```yaml
spring:
  datasource:
    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/springboot
      username: root
      password: 12345678
```



### 3. 异常的统一处理

在springboot中，需要统一在表现层处理异常，可以进行如下操作

- 定义异常处理类，使用注解配置
- 在类中写方法，处理对应的异常

```java
// 异常处理类
@RestControllerAdvice
public class ErrorHandler {
// 下面的注解参数中可以指定为处理的对应的异常
    @ExceptionHandler(Exception.class)
// ex对象表示异常对象
    public Result exceptionHandler(Exception ex){
        ex.printStackTrace();
        return new Result(false, null, "error");
    }
}
```

异常抛出代码

```java
@GetMapping("{id}")
// 当传来的id为0时，会抛出异常
public Result errorTest(@PathVariable Integer id){
    int a = 1 / id;
    return new Result();
}
```

### 4. 运维实用指令

#### 4.1打包指令

- 对springboot工程进行打包

```tex
mvn package 
```

- 利用指令运行项目

```tex
java -jar springboot.jar
```

注意，jar指令的运行一定要依赖于maven中打包插件

```xml
 <plugins>
    <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
```

若无打包插件，运行项目报错如下：

**无主清单属性**

#### 4.2临时属性

例如临时修改端口为指定的，可以采取本策略，利用--进行临时属性配置，各种临时属性之间利用空格分割

```tex
java -jar springboot.jar --server.port = 8080
```

```tex
java -jar springboot.jar --server.port = 8080 --spring.datasource.druid.password = 123
```

#### 4.3配置文件优先级

1. 与jar同级目录下config/application.yml **最高**
2. 与jar同级目录下application.yml
3. 类路径下config/application.yml
4. 类路径下application.yml **最低**

#### 4.4多环境开发yml

我们可以在yml中配置很多种环境，在运行环境中指定我们需要使用什么环境进行运行，环境之间利用 --- 分隔开

```yaml
# 公共配置 运行环境
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot
    username: root
    password: 12345678
  profiles: 
    active: dev # 在此选择要使用的环境

---
# 生产环境
spring:
  config:
    activate:
      on-profile: pro # 环境起名
server:
  port: 8000
---
# 开发环境
spring:
  config:
    activate:
      on-profile: dev # 环境起名
server:
  port: 8001
---
# 测试环境
spring:
  config:
    activate:
      on-profile: test # 环境起名
server:
  port: 8002
```

#### 4.5多环境分组管理

原课程：https://www.bilibili.com/video/BV15b4y1a7yG?p=64&spm_id_from=pageDriver&vd_source=cba09bff3fb893a92d86de5c69b49831

P64

在实际开发中，可能会将不同的配置拆成不同的文件，例如同级目录下有如下四个配置文件：

- application.yml
- application-dev.yml
- application-devDB.yml
- application-devMVC.yml

则需要在application.yml文件中需要做如下配置：

```yaml
spring:
  profiles: 
    active: dev
    group: 
      "dev": devDB,devMVC # 若属性冲突，后加载的生效
      "pro": proDB,proMVC
```

#### 4.6日志相关操作

##### 4.6.1日志级别的开启

控制台默认答应的为info级别的日志，要令控制台答应debug级别的应开启debug级别的日志

在application.yml中进行如下配置

```yaml
# debug: true # 不推荐使用
# 推荐使用如下
logging: 
  level: 
    root: debug # 日志级别，根路径下为debug（所有的包）
```

##### 4.6.2日志记录的使用

步骤如下：

- 新建一个日志记录对象，传进去记录类名的字节码文件
- 利用logger的方法进行日志记录 debug info warn error

代码如下：

```java
@RestController
@RequestMapping("/logs")
public class logController {

    Result result = new Result();

    // 创建日志记录对象，传入参 数为 为当前类服务的日志记录对象
    private static final Logger LOGGER = LoggerFactory.getLogger(logController.class);

    @GetMapping
    public Result getById(){
//        LOGGER.trace("trace..."); // 等级过低，一般不用
        LOGGER.debug("debug...");
        LOGGER.info("info...");
        LOGGER.warn("warn...");
        LOGGER.error("error...");
        return result;
    }
}
```

如果不手动创建日志记录对象，可以采用lombok提供的@slf4j注解进行开发，代码如下：

```java
@Controller
@ResponseBody
@RequestMapping("/user")
@Slf4j // 此注解提供了log对象进行日志的记录
public class UserController {
    @GetMapping("/users")
    public String getUsers(){
        log.warn("slf4j...");
        System.out.println(Arrays.toString(userDao.getUsers()));
        return "";
    }
}
```

lombok坐标如下：

```xml
<dependency>
  	<groupId>org.projectlombok</groupId>
  	<artifactId>lombok</artifactId>
</dependency>
```



##### 4.6.3分组设置日志级别

将不同的包归属为不同的组，对组进行日志级别控制，yml配置如下：

```yaml
logging: 
  group: 
  # 对包进行分组
    group1: com.package.controller1, com.package.controller2, ...
    group2: com.package.controller3, com.package.controller4, ...
  level: 
  # 设置对应分组的日志级别
    group1: warn
    group2: debug
```

##### 4.6.4日志输出格式控制

首先，日志的格式如下：

![image-20230705011907150](/Users/shj/Library/Application Support/typora-user-images/image-20230705011907150.png)

可以设置日志输出格式如下：

```yaml
logging: 
  pattern: 
    console: "%d - %m %n"
# %d 日期 %m 消息 %n 换行 
# %p 级别 %clr(){颜色名} 括号内的属性有颜色 %t 线程名
# %c 类名
```

##### 4.6.5文件记录日志

我们有时需要将日志文件输出到一个文件中，可以在application.yml中做如下配置：

```yaml
logging: 
  file: 
    name: server.log # 此文件在工程同级目录下生成
```

分文件日志记录，当日志过大时，可以分文件记录，当一个日志达到设置的最大值时，系统自动新建文件继续记录日志

```yaml
logging: 
  file: 
    name: server.log # 此文件在工程同级目录下生成
  logback:
    rollingpolicy:
      max-file-size: 4KB # 单个日志文件的最大大小
      file-name-pattern: server.%d.%i.log
```

### 5. 开发常用操作

#### 5.1第三方Bean属性注入

除了自定义的一些对象，有些第三方Bean的属性也需要读取yml文件中的值，例如Druid，我们可以进行如下操作

#### 5.2配置文件计量单位

由于yml中无单位，配置文件时常常需要用到单位，例如时间，由此需要配置单位

- 在配置文件中配置数量
- 在配置类中利用注解配置单位

```yaml
servers:
  ipAddress: 192.168.0.1
  port: 2222
  serverTimeOut: 300
  fileSize: 10
```

下面的示例中，分别利用@DurationUnit注解和@DataSizeUnit注解进行单位配置

```java
@Data
@Component
@ConfigurationProperties(prefix = "servers")
public class ServerConfig {
    private String ipAddress;
    private String port;

    @DurationUnit(ChronoUnit.HOURS)
    private Duration serverTimeOut;

    @DataSizeUnit(DataUnit.MEGABYTES)
    private DataSize fileSize;
}
```

#### 5.3测试环境中启动web环境

##### 5.3.1测试类做成web测试类

```java
// 参数 webEnvironment 的取值为定义的端口
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class WebTest {
    @Test
    void test1(){
        
    }
}
```

##### 5.3.2模拟请求发送

分如下几步：

- 使用注解 @AutoConfigureMockMvc 启动模拟web调用
- 使用@Autowired MockMvc mockMvc 注入调用对象
- 构造请求发送对象
- 发送请求

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
// 启动模拟web请求调用
@AutoConfigureMockMvc
public class WebTest {
    @Test
    // 注入虚拟调用mvc对象
    void test1(@Autowired MockMvc mockMvc) throws Exception {
        // 创建虚拟请求
        // 除了get，还有post，put，delete等
        MockHttpServletRequestBuilder builder = MockMvcRequestBuilders.get("/test/t3");
        mockMvc.perform(builder);
    }
}
```

#### 5.4随机数据的构造

在实际开发中，常常需要构造一些随机输入存入到数据库，可以在yml中做如下配置

```yaml
test:
  book:
    bookName: ${random.uuid} # 随机uuid
    b1: ${random.int} # 随机整数
    b2: ${random.int(10)} # 随机正整数，最大为0
```

javaBean定义如下：

```java
@Data
@Component
@ConfigurationProperties(prefix = "test.book")
public class BookTest {
    private String bookName;
    private Integer b1;
    private Integer b2;
}
```

之后利用 @Autowired 注解自动装配时就有了随机数据

#### 5.5静态资源的映射处理

在springboot开发中，由于默认的mvc会将请求路径当作controller去寻找，会导致有些静态资源例如页面html，图片image，js，css等访问不到，此时需要进行静态资源的处理，需要新建一个类继承WebMvcConfigurationSupport，使用Configuration注解配置，详细代码如下:

```java
@Slf4j
@Configuration
public class WebMvcConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addResourceHandlers(ResourceHandlerRegistry registry) {
        log.info("静态资源映射处理...");
      // 当访问到front目录下所有资源时，去类加载路径下的front寻找
        registry.addResourceHandler("/front/**").addResourceLocations("classpath:/front/");
        registry.addResourceHandler("/templates/**").addResourceLocations("classpath:/templates/");
        registry.addResourceHandler("/static/**").addResourceLocations("classpath:/static/");
    }
}
```

#### 5.6拦截器的使用

在springboot中使用拦截器需要做如下处理，创建类继承WebMvcConfigurer，重写addInterceptors方法，利用registry添加拦截器，具体代码如下:

```java
@Slf4j
@Configuration
@EnableWebMvc //此注解必须要有
public class ResponseConfig implements WebMvcConfigurer {

    @Autowired
    public Inter1 inter1;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        log.info("拦截器配置...");
        registry.addInterceptor(inter1).addPathPatterns("/front/**");

    }
}
```

当然，还需要定义拦截器，方法为，定义一个类实现HandlerInterceptor接口，具体代码如下:

```java
@Slf4j
@Component
public class Inter1 implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("inter1...");
        return true;
    }
}
```

#### 5.7Springboot中过滤器的使用

在springboot中使用filter首先需要在启动类中加上 @ServletComponentScan 注解，这样才能加载servlet的组建，之后需要定义一个类实现servlet中的Filter接口，具体代码如下:

**启动类中**

```java
@SpringBootApplication
@ServletComponentScan //此注解打开了servlet组建，例如filter
public class MPspringBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(MPspringBootApplication.class, args);
    }
}
```

**过滤器的定义**

此时拦截通配符为/*，和拦截器有些不同

```java
@Slf4j
@WebFilter(filterName = "filter1", urlPatterns = {"/front/*"})
public class filter1 implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        log.info("请求的路径:" + httpServletRequest.getRequestURI());
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("filter1...");
    }

    @Override
    public void destroy() {
    }
}
```



### 6. 非关系型数据库

#### 6.1 Redis

​	有如下几步：

- 创建工程时勾选redis坐标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

- 在application.yml文件中配置redis

```yaml
spring:
  redis:
    port: 6379
    host: localhost
#    client-type: jedis # 默认为 lettuce
```

- 测试功能

```java
@SpringBootTest
class NosqlApplicationTests {

    @Autowired // 以对象的形式操作，利用 keys* 在redis-cli中可以查看到
    private RedisTemplate redisTemplate;

    @Autowired // 以字符串的形式操作
    private StringRedisTemplate stringRedisTemplate;
  
    @Test
    void setTest(){
        ValueOperations ops = redisTemplate.opsForValue();
        ops.set("age",18);
        System.out.println(ops.get("age"));
    }

    @Test
    void getTest(){
        ValueOperations ops = redisTemplate.opsForValue();
        System.out.println(ops.get("aa"));
    }

    @Test
    void setTestStr(){
        ValueOperations<String, String> strops = stringRedisTemplate.opsForValue();
        strops.set("name","zhangsan");
        System.out.println(strops.get("name"));
    }
}
```



#### 6.2 MongoDB

mongoDB是一种介于关系型和非关系型数据库之间的数据库，springboot使用它步骤如下：

- 创建工程时导入坐标

```xml
<dependency>
		<groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

- 在application.yml中配置mongoDB

```yaml
spring:
  data:
    mongodb: # 无需配置 host 和 port
      uri: mongodb://localhost/springboot
```

- 测试mongoDB

```java
@SpringBootTest
public class mongoDBTest {

    @Autowired
    private MongoTemplate mongoTemplate;

    @Test
    void saveTest(){
        User user = new User();
        user.setUsername("zhangsan");
        user.setPassword("zhang123");

        mongoTemplate.save(user);
    }

    @Test
    void findTest(){
        List<User> all = mongoTemplate.findAll(User.class);
        System.out.println(all);
    }
}
```



#### 6.3 ES

### 7. Spring中的缓存

spring中内置了缓存技术，使用它有如下步骤：

- 添加缓存坐标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

- 开启缓存功能，利用注释 @EnableCaching 在启动类中添加

```java
@SpringBootApplication
@EnableCaching // 此注释开启了缓存
public class NosqlApplication {

    public static void main(String[] args) {
        SpringApplication.run(NosqlApplication.class, args);
    }

}
```

- 在需要缓存的方法上添加注释

```java
@RestController
@RequestMapping("/users")
public class UserController {
    @GetMapping
  // value 用来指定缓存的位置，key用来指定查询的键（保证唯一性）
    @Cacheable(value = "cacheSpace", key = "#id")
    public String getUserById(Integer id){
        return "";
    }
}
```

### 8. SpringBoot中的定时任务

#### 8.1 Quartz

quartz是springboot支持的一款定时器，使用它有如下几步：

- 导入quartz的坐标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

- 制作定时任务

制作定时任务中，需要创建一个定时任务类，实现 QuartzJobBean , 覆盖executeInternal方法即可

```java
public class TimerTaskApp extends QuartzJobBean {
    @Override // 在此书写要执行的方法
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        System.out.println("quartz run...");
    }
}
```

- 创建配置类

需要单独创建一个配置类，并且使用configuration注解，在此需要定义工作明细对象和触发器，并且绑定对应关系

```java
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail jobDetail(){
        return JobBuilder
                .newJob(TimerTaskApp.class) // 在newjob中需要放入，工作的对象
                .storeDurably()
                .build();
    }
  
    @Bean
    public Trigger trigger(){
        // 定时时间为 每隔三秒执行一次
        CronScheduleBuilder cronSchedule = CronScheduleBuilder.cronSchedule("0/3 * * * * ?");
        return TriggerBuilder
                .newTrigger().
                forJob(jobDetail()) // 在此绑定工作细节对象
                .withSchedule(cronSchedule) // 在此设置定时时间
                .build();
    }
    
}
```

#### 8.2 Task

此功能是为了简化quartz开发，使用它有如下步骤：

- 在引导类中开启定时执行任务的开关 @EnableScheduling

```java
@SpringBootApplication
@EnableCaching // 此注释打开了缓存开关
@EnableScheduling // 此注释打开了定时器开关
public class NosqlApplication {
    public static void main(String[] args) {
        SpringApplication.run(NosqlApplication.class, args);
    }
}
```

- 在想要定时执行的任务中指定定时时间 @Scheduled

```java
@Component // 要定义为Bean
public class TaskApp {
  // 在想要定时执行的方法上添加注释，并且指定cron表达式
    @Scheduled(cron = "0/3 * * * * ?")
    public void task(){
        System.out.println("task...");
    }
}
```

- 在yml中对定时器有如下配置

```yml
spring:
  task:
    scheduling:
      pool:
        # 任务调度线程池的大小
        size: 1
      shutdown:
        # 线程池关闭时等待所有任务执行完毕
        await-termination: false
        # 调度线程关闭前最大等待时间，确保最后一定会关闭
        await-termination-period: 10s
```

### 9. 整合javaMail

#### 9.1发送简单邮件

使用javaMail可以实现使用java发送邮件，使用它有如下几步：

- 导入javaMail坐标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

- 在application.yml中进行配置

```yaml
spring:
  mail:
    host: smtp.qq.com
    username: 1327325289
    password: ktpfhaqbaubdhjgh # 授权码
```

- 使用接口进行发送邮件

```java
@Controller
@Slf4j
@RequestMapping("/mail")
public class MailController {
    @Autowired // springboot 已经提供，直接注入即可
    private JavaMailSender javaMailSender;
    // 发送人 接受人 标题 正文
    @GetMapping("/send")
    @ResponseBody
    public String sendMail(){
        // 构造简单邮件
        SimpleMailMessage message = new SimpleMailMessage();
        String from = "1327325289@qq.com";
        String to = "2976551621@qq.com";
        String subject = "测试标题";
        String context = "测试正文";
        message.setFrom(from + "(传说)"); // from后面加上(content)，发送人会变成content
        message.setTo(to);
        message.setSubject(subject);
        message.setText(context);
        javaMailSender.send(message);
        return "";
    }
}
```



#### 9.2发送多部件邮件 

实际中，有些邮件带有图片等其他信息，此时邮件便不再是简单邮件，需要使用多部件邮件发送，需要做如下设置：

```java
@Controller
@Slf4j
@RequestMapping("/mail")
public class MailController { 
  	@Autowired // springboot 已经提供，直接注入即可
    private JavaMailSender javaMailSender;
		@GetMapping("sendm")
    @ResponseBody
    public String sendMiMeMail(){
        try { // set中会报错，需要异常处理
            MimeMessage message = javaMailSender.createMimeMessage();
            // 第二个参数为true表示允许附件发送
            MimeMessageHelper helper = new MimeMessageHelper(message,true);
            String from = "1327325289@qq.com";
            String to = "2976551621@qq.com";
            String subject = "测试标题";
            String context = "<a href='https://www.baidu.com'>测试正文跳转百度</a>";
            helper.setFrom(from + "(传说)");
            helper.setTo(to);
            // 第二个参数为true表示正文部分支持html语法
            helper.setSubject(subject);
            helper.setText(context,true);
            
            File file = new File("/Users/shj/IdeaProjects/SpringBoot/springboot1/mail/HELP.md");
            helper.addAttachment(file.getName(),file);// 添加附件
            
            javaMailSender.send(message);
        } catch (MessagingException e) {
            e.printStackTrace();
        }
        return "";
    }
}
```

### 10. 消息处理

#### 10.1 ActiveMQ

activeMQ是一款消息队列的实现软件，使用它需要进入安装目录运行 ./activemq start，默认用户名密码都为admin，之后有如下几步：

- 导入activeMQ的坐标

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```

- 在application.yml中对它进行配置

```yaml
spring:
  activemq:
    broker-url: tcp://localhost:61616 # 指定网址和端口
  jms:
    template:
      default-destination: deafultArea # 指定默认对列存储区域，可以不指定
```

- 编写api进行使用（分两种方式，手动使用和监听自动使用）

```java
@Slf4j
@Service
public class MessageServiceActivemqImpl implements MessageService {

    @Autowired
  	// 默认spring注入的对象，直接使用
    private JmsMessagingTemplate messagingTemplate;

    @Override
    public void sendMessage(String id) {
        System.out.println("已纳入处理队列,id:" + id);
      	// 转换和发送，第一个参数为存储对列区域名
        messagingTemplate.convertAndSend("order.queen.id",id);
    }

    @Override
    public String doMessage() {
      	// 接受和转换，第一个参数为从哪个队列中取出，第二个参数为，转换后的对象类型
        String id = messagingTemplate.receiveAndConvert("order.queen.id",String.class);
        log.info("已完成处理,id:" + id);
        return id;
    }
}
```

以上为手动使用

```java
@Slf4j
@Component
public class ActivemqListener {
		// 设置监听器，destination参数为监听的对列名，只要发现对列中有数据自动取出使用
    @JmsListener(destination = "order.queen.id")
  	// 处理完之后发送给哪个队列，会将方法返回值发送给此对列
    @SendTo("order.queen.newid")
    public String receive(String id){
        log.info("已完成处理,id:" + id);
        return "new" + id;
    }
		// 处理上一个方法发来的消息
    @JmsListener(destination = "order.queen.newid")
    public void receiveNewId(String id){
        log.info("已完成处理:" + id);
    }
}
```

#### 10.2 RabbitMQ

### 11. Spring事务管理

#### 11.1 基本概念和操作

spring中有事务管理，所谓事务，类似于操作系统中的原语，必须同时执行成功或者失败，一旦出现异常，必须进行事务的回滚，在spring中，可以通过注释 @Transactional 进行事务管理

- 作用于特定方法上，此方法会被当成事务管理
- 作用于类上，类中所有的方法都会当成事务管理
- 作用于接口上，该接口所有实现类的方法会被当成事务管理

下面列举注释作用于方法上

```java
public class UserService{
  @Transactional // 此时save方法中的语句同时成功或者失败，抛出异常会进行事务会滚。
  public void save(String username, String password){
    // 代码1
    int i = 1/0; // 抛出异常
    // 代码2
  }
}
```

#### 11.2 rollbackFor

默认的 @Transactional 注释，只会在抛出运行时异常，即RunTimeException，才会进行事务的回滚，但Exception异常不会进行事务会滚，此时需要用到 rollbackFor 参数。

```java
public class UserService{
  @Transactional(rollbackFor = Exception.class) // 指定出现Exception异常时，进行事务会滚
  public void save(String username, String password){
    // 代码1
    if (true){
      throw new Exception("error info...");
    }
    // 代码2
  }
}
```

#### 11.3 propagation

此注释用来控制事务的传播行为，它的取值有如下几种：

|      属性值      |                             含义                             |
| :--------------: | :----------------------------------------------------------: |
|   **REQUIRED**   |       **【默认值】**需要事务，有则加入，无则创建新事务       |
| **REQUIRES_NEW** |             需要新事务，无论有无，总是创建新事务             |
|     SUPPORTS     |          支持事务，有则加入，无则在无事务状态中运行          |
|  NOT_SUPPORTED   | 不支持事务，在无事务状态下运行,如果当前存在已有事务,则挂起当前事务 |
|    MANDATORY     |                    必须有事务，否则抛异常                    |
|      NEVER       |                    必须无事务，否则抛异常                    |

例如，现在有一个需求，对于一个删除用户功能，无论操作成功失败与否，都要进行日志记录，此时，如果删除失败抛出异常时，用以前的事务管理，会将日志记录功能一并会滚，无法在日志记录数据表中看到对应的操作，此时需要用到事务传播行为控制。示例代码如下

```java
public class UserService{
  @Transactional
  public void delete(String username){
    // 根据username删除用户
    // 其他代码
    log(username);
  }
  // 此时必须指定 progagation = Propagation.REQUIRES_NEW 在log方法中开启一个新的事务，否则log方法鬼delete事务管控，当delete方法出现异常时，会将log方法一并会滚
  @Transactional(progagation = Propagation.REQUIRES_NEW)
  public void log(String username){
    // 记录操作日志
    newLog(username);
  }
}
```

### 12. MybatisPlus使用

首先根据 2.3 节的内容在springboot下整合mybatisplus，之后就可以正常使用了

#### 12.1 分页的使用

在mybatisplus中，如果要对数据进行分页查询，需要创建一个Ipage分页对象，并为之创建分页拦截器，才可以进行使用，代码如下：

**java代码**

```java
@Test
void MpPageSelect(){
    IPage page = new Page(1,2);
    bookDao.selectPage(page, new Wrapper<Book>() {
        @Override
        public Book getEntity() {
            return null;
        }

        @Override
        public MergeSegments getExpression() {
            return null;
        }

        @Override
        public void clear() {

        }

        @Override
        public String getSqlSegment() {
            return null;
        }
    });
    System.out.println("数据:" + page.getRecords());
    System.out.println("当前页:" + page.getCurrent());
    System.out.println("显示的数据条数:" + page.getSize());
    System.out.println("总页数:" + page.getPages());
    System.out.println("总数据条数:" + page.getTotal());
}
```

**输出结果**

```tex
数据:[Book(bid=1, bookname=math, description=math is too hard), Book(bid=2, bookname=english, description=english is easy)]
当前页:1
显示的数据条数:2
总页数:2
总数据条数:3
```

**拦截器的创建**

```java
@Configuration
public class MpConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
      // 创建大拦截器
        MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();
      // 在大拦截器中添加分页拦截器
        mybatisPlusInterceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return mybatisPlusInterceptor;
    }
}
```

#### 12.2 条件查询

条件查询，mp提供了条件查询类，需要构造，查询方式可以有三种，接下来依次展示：

```java
@Test
void testConditionQuery1(){
    // 方式一: 条件查询
    QueryWrapper wrapper = new QueryWrapper();
    wrapper.lt("price",35);
    List list = bookDao.selectList(wrapper);
    System.out.println(list);
}
```

此方法使用了QueryWrapper对象进行查询，lt为小于条件，可以查阅文档看更多的条件

```java
@Test
void testConditionQuery2(){
    // 方式二: 条件查询 lambda
    QueryWrapper<Book> wrapper = new QueryWrapper();
    wrapper.lambda().lt(Book::getPrice, 35);
    List<Book> books = bookDao.selectList(wrapper);
    System.out.println(books);
}
```

此时查询的时候使用了.lambda()方法，可以在后面使用语法糖，减少出错

```java
@Test
void testConditionQuery3(){
    // 方式三: 条件查询 lambda (接口更方便)
    LambdaQueryWrapper<Book> wrapper = new LambdaQueryWrapper<>();
    wrapper.lt(Book::getPrice, 35);
    List<Book> books = bookDao.selectList(wrapper);
    System.out.println(books);
}
```

此时更换了查询接口使用LambdaQueryWrapper，就可以不使用.lambda()方法

**接下来为多条件查询**

```java
 @Test
// 多条件查询
void testMultConditionsQuery(){
    LambdaQueryWrapper<Book> wrapper = new LambdaQueryWrapper<>();
    // 条件之间为and关系
//        wrapper.lt(Book::getPrice, 35).gt(Book::getPrice, 10);
    // 条件之间为or关系
    wrapper.lt(Book::getPrice, 10).or().gt(Book::getPrice, 30);
    List<Book> books = bookDao.selectList(wrapper);
    System.out.println(books);
}
```

支持链式编程

##### 12.2.1 条件查询空值处理

在查询中，如果条件出现空值，一般会导致查询失败，mp提供了如下方法：

```java
 @Test
void testNullConditionQuery(){
    Book book = new Book();
    book.setPrice(20);
    LambdaQueryWrapper<Book> wrapper = new LambdaQueryWrapper<>();
  // 当book的price字段不为null时，此条件才进行拼接
    wrapper.lt(book.getPrice() != null,Book::getPrice, 10).
            or().gt(Book::getPrice, 30);
    List<Book> books = bookDao.selectList(wrapper);
    System.out.println(books);
}
```

#### 12.3 查询投影

### 13. WebSocket的基本使用

websocket协议实现了服务端和客户端的双向通信, 可以实现在线聊天室，在线客服等需要双向通信的场景, websocket分客户端和服务端, 在此服务端使用springboot集成的websocket, 客户端使用原生js实现服务端和客户端的双向通信

#### 13.1 服务端

首先有一个配置类，采用第三方Bean注入即可

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter(){
        return new ServerEndpointExporter();
    }
}
```

服务端代码框架如下：

```java
package com.file.ws;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.PathVariable;

import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
@ServerEndpoint(value = "/chat/{username}")
public class MyWebSocket {

    // 日志记录对象
    private static final Logger log = LoggerFactory.getLogger(MyWebSocket.class);

    //用来存储每一个session对象
    private static final Map<String, Session> sessionMap = new ConcurrentHashMap<>();

    @OnOpen // 连接建立调用的方法
    public void opOpen(Session session, @PathParam("username") String username){
        sessionMap.put(username, session);
        log.info("{} 加入了聊天室...", username);
    }

    @OnMessage // 收到客户端消息调用的方法 
    public void opMessage(String message, Session session, @PathParam("username") String username){
        log.info("{} 发送了消息 {}", username, message);
    }

    @OnClose // 关闭连接调用的方法
    public void onClose(Session session, @PathParam("username") String username){
        log.info("{} 关闭了链接...", username);
    }

  	// 采用定时任务实现定时向客户端发送消息, 测试使用
    @Scheduled(cron = "0/3 * * * * ?")
    private void sendMsgToAllUser() throws IOException {
        for (String username : sessionMap.keySet()) {
            Session session = sessionMap.get(username);
            session.getBasicRemote().sendText("这是服务端测试消息");
        }
    }
}

```

#### 13.2 客户端

采用了vue + elementUI进行基本的框架搭建

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>聊天室</title>
    <link rel="stylesheet" href="../css/el.css">
</head>
<body>
    <div id="app">
        <div class="title">
            <h2>
                {{ title }}
            </h2>
        </div>
        <div class="chatBody">
            <div style="width: 500px">
                <el-input v-model="smsg"></el-input>
                <el-button @click="send">发送</el-button>
            </div>
        </div>
    </div>
</body>
</html>
<style>
    .title{
        text-align: center;
        color: #3a8ee6;
    }
    .chatBody{
        text-align: center;
    }
</style>
<script src="../js/vue.js"></script>
<script src="../js/el.js"></script>
<script src="../js/axios-0.18.0.js"></script>
<script lang="javascript">
    var socket;
    new Vue({
        el:"#app",
        data(){
            return {
                title: "在线聊天室(基于websocket)",
                username: "zhangsan",
                smsg: "",
                rmsg: "",
            }
        },
        methods: {
            init(){
                if (typeof(WebSocket) === "undefined"){
                    alert("您的浏览器不支持websocket, 功能将无法正常使用!");
                }
                let socketUrl = "ws://localhost/chat/" + this.username;
                console.log("当前用户:",this.username);
                socket = new WebSocket(socketUrl);

                // websocket 创建操作
                socket.onopen = function () {
                    console.log("websocket已经打开!");
                }

                // 收到从服务端发来的消息
                socket.onmessage = function (msg) {
                    let text = "收到消息: " + msg;
                    console.log(text);
                }
            },
            send(){
              // 构造发送的消息 json
                let msg = {
                    text: this.smsg
                };
                socket.send(JSON.stringify(msg));
                console.log("消息已发送...");

            },
        },
        mounted(){

        },
        created(){
            this.init();
        }
    })
</script>
```



### 14. 监控的意义

