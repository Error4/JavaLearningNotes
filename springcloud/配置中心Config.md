# 1.概述

我们知道，微服务将单体应用拆分成一个个的子服务，每个子服务都需要必要的配置信息，随着子服务数量的增加，与之相关的配置文件数量也会随之增加，如果有上百个配置文件，那么管理起来便十分困难。因此，一套集中式的、动态的配置管理中心是必不可少的，Spring Cloud 提供了 ConfigServer 来解决这个问题。

Spring Cloud Config 在微服务分布式系统中，采用 **Server 服务端**和 **Client 客户端**的方式来提供可扩展的配置服务。服务端提供配置文件的存储，以接口的形式将配置文件的内容提供出去；客户端通过接口获取数据、并依据此数据初始化自己的应用。

Spring Cloud Config 默认使用 Git 来存储配置文件（也有其他方式，比如SVN、本地文件，但最推荐的还是 Git），而且使用的是 http/https 访问的形式。

# 2.基本使用

## 2.1 准备配置文件

​		准备一个 Git 仓库，在 Github 上面创建了一个[仓库](https://github.com/Error4/SpringCloudConfig-repo)用来存放配置文件。为了模拟生产环境，我们创建以下三个配置文件：

config-client-dev.yml

```yml
server:
  port: 7778 # 端口

spring:
  application:
    name: config-client # 应用名称

# 自定义配置
name: config-client-dev
```

　　config-client-test.yml

```yml
server:
  port: 7779 # 端口

spring:
  application:
    name: config-client # 应用名称

# 自定义配置
name: config-client-test
```

## 2.2 Server 端

### 添加依赖

​		只需要在 pom.xml 中 加入 spring-cloud-config-server 即可

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

### 配置文件

​		在 application.yml 中添加配置服务的基本信息以及 Git 仓库的相关信息

```yaml
server:
  port: 7010 # 端口
spring:
  application:
    name: config-server # 应用名称
  cloud:
    config:
      server:
        git:
          uri: https://github.com/Error4/SpringCloudConfig-repo # 配置文件所在仓库地址
          #username:             # Github 等产品的登录账号
          #password:             # Github 等产品的登录密码
          #default-label: master # 配置文件分支
          #search-paths:         # 配置文件所在根目录
```

​		Spring Cloud Config 也提供本地存储配置的方式。我们只需要设置属性 `spring.profiles.active=native`，Config Server 会默认从应用的 `src/main/resource` 目录下检索配置文件。也可以通过 `spring.cloud.config.server.native.searchLocations=file:E:/properties/` 属性来指定配置文件的位置。

### 启动类

​		启动类添加 `@EnableConfigServer`，激活对配置中心的支持

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class,args);
    }
}
```

### 测试

仓库中的配置文件会被转换成 Web 接口，访问可以参照以下的规则：

- /{application}/{profile}[/{label}]
- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties

上面的 URL 会映射 `{application}-{profile}.yml` 对应的配置文件，其中 `{label}` 对应 Git 上不同的分支，默认为 master。以 config-client-dev.yml 为例子，它的 application 是 config-client，profile 是 dev。

例如直接访问 `http://localhost:7010/config-client-test.yml`， 返回信息如下，说明配置中心服务端一切正常。

```json
name: config-client-test
server:
  port: 7779
spring:
  application:
    name: config-client
```

## 2.3 Client端

### 添加依赖

```xml
<!-- 项目依赖 -->
    <dependencies>
        <!-- spring cloud starter config 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <!-- spring boot web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- spring boot test 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>
```

### 配置文件

bootstrap.yml

```yaml
spring:
  cloud:
    config:
      uri: http://localhost:7010 # 配置中心的具体地址，即 config-server
      name: config-client # 对应 {application} 部分，需要从git上读取的资源名称，资源名一般为application.profile.yml
      profile: test # 对应 {profile} 部分
      label: master # 对应 {label} 部分，即 Git 的分支。如果配置中心使用的是本地存储，则该参数无用
```

**特别注意**：**上面这些与 Spring Cloud Config 相关的属性必须配置在 bootstrap.yml 中，config 部分内容才能被正确加载。因为 config 的相关配置会先于 application.yml，而 bootstrap.yml 的加载也是先于 application.yml。**

### 启动类

启动类不用修改，在 Controller 中使用 `@Value` 注解来获取 Server 端参数的值

```java
@RestController
public class ConfigController {
    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }
}
```

### 测试

访问 `http://localhost:7779/name`， 返回信息如下，说明client正确加载了对应的config-client-test.yml配置文件。

```json
config-client-test
```

# 3.高可用

## 传统作法

通常在生产环境，Config Server 与服务注册中心一样，我们也需要将其扩展为高可用的集群。在之前实现的 config-server 基础上来实现高可用非常简单，不需要我们为这些服务端做任何额外的配置，只需要遵守一个配置规则：将所有的 Config Server 都指向同一个 Git 仓库，这样所有的配置内容就通过统一的共享文件系统来维护，而客户端在指定 Config Server 位置时，只要配置 Config Server 外的均衡负载

## 注册为服务

虽然通过服务端负载均衡已经能够实现，但是作为架构内的配置管理，本身其实也是可以看作架构中的一个微服务。所以，另外一种方式更为简单的方法就是把 config-server 也注册为服务，这样所有客户端就能以服务的方式进行访问。通过这种方法，只需要启动多个指向同一 Git 仓库位置的 config-server 就能实现高可用了。

### 服务端改造

#### 额外添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 配置文件

在 application.yml 里新增 Eureka 的配置

```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

### 客户端改造

#### 额外添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 配置文件

bootstrap.yml

```yaml
spring:
  cloud:
    config:
      name: config-client
      profile: test
      label: master
      discovery:
        enabled: true
        service-id: config-server
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/

```

主要是去掉了 `spring.cloud.config.uri` 直接指向 Server 端地址的配置，增加了最后的三个配置：

- `spring.cloud.config.discovery.enabled`：开启 Config 服务发现支持
- `spring.cloud.config.discovery.serviceId`：指定 Server 端的 name, 也就是 Server 端 `spring.application.name` 的值
- `eureka.client.service-url.defaultZone`：指向配置中心的地址

访问 `http://localhost:7779/name`， 返回信息如下，说明client正确加载了对应的config-client-test.yml配置文件。

```json
config-client-test
```

# 4.客户端动态刷新

​		当配置中心的配置文件内容发生改动，**服务端可以动态的获取，客户端不能！**

​		因为服务端直接从配置中心获取，而客户端是从上下文环境中获取已加载的属性，配置中心修改后，由于服务没有重启，获取的仍然是之前的属性。

​		所幸，它提供了一个刷新机制，但是需要我们主动触发。那就是 `@RefreshScope` 注解并结合 `Actuator`，注意要引入 `spring-boot-starter-actuator`。

## 添加依赖

​		Config Client 额外添加依赖

```
<!-- spring boot actuator 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## 配置文件

```yaml
spring:
  cloud:
    config:
      name: config-client
      profile: test
      label: master
      discovery:
        enabled: true
        service-id: config-server
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
# 度量指标监控与健康检查
management:
  endpoints:
    web:
      base-path: /actuator    # 访问端点根路径，默认为 /actuator
      exposure:
        include: '*'          # 需要开启的端点，这里主要用到的是 refresh 这个端点
```

## 控制层

​		在需要读取配置的类上增加 `@RefreshScope` 注解。

```java
@RefreshScope
@RestController
public class ConfigController {

    @Value("${name}")
    private String name;

    @GetMapping("/name")
    public String getName() {
        return name;
    }

}
```

以我的程序为例，再修改配置文件后，发送 POST 请求到 `http://localhost:7779/actuator/refresh` 这个接口，访问 `http://localhost:7779/name`，就会看到修改后的name属性。

# 5.在 Github 中配置 Webhook

​		在第四节中，尽管实现了客户端的动态刷新，但是弊端也很明显，总不能每次改了配置后，就重新访问一下 refresh 接口吧？

​		Github 提供了一种 Webhook 的方式，当有代码变更的时候，会调用我们设置的地址，来实现我们想达到的目的。

![](https://s1.ax1x.com/2020/04/30/JqGer4.md.jpg)

填写回调的地址，也就是上面提到的 actuator/refresh 这个地址，但是必须保证**这个地址是可以被 Github 访问的。**

![](https://s1.ax1x.com/2020/04/30/JqGxW6.md.png)

# 6.仍然存在的问题

​		通过Webhook，我们进一步简化了动态刷新的复杂程度，但当存在大量客户端时，仍然显得有一些复杂，有没有更为简便灵活的方式呢？

​		第一种方法，我们可以借助 Spring Cloud Bus 的广播功能，让 Config Client 都订阅配置更新事件，当配置更新时，触发其中一个端的更新事件。

​		参考文章：[SPRING CLOUD 系列之 BUS 消息总线](https://mrhelloworld.cn/articles/spring/spring-cloud/bus/)

​		第二种是我更推荐的，我们可以使用其他更合适的配置中心，比如Consul、Apollo等



​		上文相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看