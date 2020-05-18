# 1.Spring Cloud Bus

Spring Cloud Bus 通过轻量消息代理连接各个分布的节点。本质是利用了 MQ 的广播机制在分布式的系统中传播消息

### POM 配置

在 pom.xml 里添加，这 4 个是必须的

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

### 配置文件

application.yml 内容如下

```yaml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/zhaoyibo/spring-cloud-study
          search-paths: config-repo
    bus:
      enabled: true
      trace:
        enabled: true
server:
  port: 12000
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```

### 启动类

加 `@EnableConfigServer` 注解

## 客户端

### POM 配置

在 pom.xml 里添加以下依赖，前 5 个是必须的，最后一个 webflux 你可以用 web 来代替

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

这里容易漏掉 `spring-boot-starter-actuator`，如果缺了这个，当对服务端执行 `/actuator/bus-refresh` 的时候，客户端接收不到信息，Console 里只会显示以下信息

### 配置文件

还是分为两个，分别如下
application.yml

```yaml
spring:
  application:
    name: config-client
  cloud:
    bus:
      trace:
        enabled: true
      enabled: true
server:
  port: 13000
```

bootstrap.yml

```yaml
spring:
  cloud:
    config:
      name: config-client
      profile: dev
      label: master
      discovery:
        enabled: true
        service-id: config-server
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

### Controller

```java
@RefreshScope
public class HelloController {

    @Value("${info.profile:error}")
    private String profile;

    @GetMapping("/info")
    public Mono<String> hello() {
        return Mono.justOrEmpty(profile);
    }

}
```

`@RefreshScope` 必须加，否则客户端会受到服务端的更新消息，但是更新不了，因为不知道更新哪里的。

### 测试

分别启动 eureka、config-server 和两个 config-client

启动后，RabbitMQ 中会自动创建一个 topic 类型的 Exchange 和两个以 `springCloudBus.anonymous.` 开头的匿名 Queue

我们访问 http://localhost:13000/info 和 http://localhost:13001/info 返回内容的都是 `dev`。
将 Git 中的配置信息由 `dev` 改为 `dev bus`，并执行

```
curl -X POST http://localhost:12000/actuator/bus-refresh/
```

再次访问 http://localhost:13000/info 和 http://localhost:13001/info 这时返回内容就是修改之后的 `dev bus` 了，说明成功了。

PS：**不管是对 config-server 还是 config-client 执行 `/actuator/bus-refresh` 都是可以更新配置的。**