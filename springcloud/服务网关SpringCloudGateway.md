# 1.概述

​		由于1.x版本Zuul 基于 Servlet 2.5（使用 3.x），使用阻塞 API，它不支持任何长连接，如 WebSockets。且2.x版本一直跳票，直到2019 年 5 月，Netflix 才终于开源了支持异步调用模式的 Zuul 2.0 版本。于是，SpringCloud官方自己推出了一个全新项目——SpringCloud Gateway。Spring Cloud Gateway 建立在 Spring Framework 5，Project Reactor 和 Spring Boot 2 之上，使用非阻塞 API，支持 WebSockets，并且由于它与 Spring 紧密集成，所以将会是一个更好的开发体验。

​		其处理流程主要如下图所示：

![](https://s1.ax1x.com/2020/04/30/JHO2Xq.jpg)



​		客户端向 Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。
​		过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑。

# 2.Gate Way实现API 网关

## 2.1 概念补充

​		在Spring Cloud GateWay中，主要有三个核心概念需要我们知晓：

​		**路由（Route）**：路由是网关最基础的部分，路由信息由 ID、目标 URI、一组断言和一组过滤器组成。如果断言路由为真，则说明请求的 URI 和配置匹配。

　　**断言（Predicate）**：Java8 中的断言函数。Spring Cloud Gateway 中的断言函数输入类型是 Spring 5.0 框架中的 ServerWebExchange。Spring Cloud Gateway 中的断言函数允许开发者去定义匹配来自于 Http Request 中的任何信息，比如请求头和参数等。

　　**过滤器（Filter）**：一个标准的 Spring Web Filter。Spring Cloud Gateway 中的 Filter 分为两种类型，分别是 Gateway Filter 和 Global Filter。过滤器将会对请求和响应进行处理。

## 2.2 入门案例

### 添加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### application.yml 配置文件

```yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
#      routes:
#        - id: default_path_to_http  路由 ID，唯一
#          uri: https://windmt.com   目标 URI，路由到微服务的地址
#          predicates:     断言（判断条件）
#            - Path=/**
#          filters:
#            - SetPath=/
server:
  port: 7009
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

配置说明：

- `spring.cloud.gateway.discovery.locator.enabled`：是否与服务注册于发现组件进行结合，通过 serviceId 转发到具体的服务实例。默认为 `false`，设为 `true` 便开启通过服务中心的自动根据 serviceId 创建路由的功能。
- `spring.cloud.gateway.routes` 用于配合具体的路由规则，是一个数组。这里我创建了一个 id 为 `default_path_to_http` 的路由，其中的配置是将未匹配的请求转发到 `https://windmt.com`。实际上开启了服务发现后，如果只使用默认创建的路由规则，这个 routes 不配置也是可以的，所以我就先注释掉了不用它了。
- 网关服务监听 10000 端口
- 指定注册中心的地址，以便使用服务发现功能

可以看到，主要存在两种路由规则，第一种通过设置`spring.cloud.gateway.discovery.locator.enabled`属性，与服务发现组件进行结合，通过 serviceId 转发到具体服务实例。

第二种则通过配置`spring.cloud.gateway.routes`属性，用以设定跟为具体的路由规则，在后面的讲述中会进一步讲解这部分内容。

### 启动类

```java
@SpringBootApplication
public class GateWayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GateWayApplication.class,args);
    }
}
```

启动服务

![](https://s1.ax1x.com/2020/04/30/JbCDAI.md.jpg)

注意：

​		在之前使用Zuul的过程中，请求URL中的服务名称默认是小写的，例如`http://localhost:7008/eureka-provider/product/list`，但在GateWay自动创建的对应路由，默认名称是大写的，`http://localhost:7009/EUREKA-PROVIDER/product/list`，如果要使用小写，需要在配置文件中配置

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          # 是否与服务发现组件进行结合，通过 serviceId 转发到具体服务实例。
          enabled: true                  # 是否开启基于服务发现的路由规则
          lower-case-service-id: true    # 是否将服务名称转小写
```

## 2.3 自定义路由规则

​		在上一小节中，我们配置了GateWay通过服务名称转发，与服务发现组件进行结合，通过 `serviceId` 转发到具体服务实例。默认匹配URL `/微服务名称/**` 路由到具体微服务。还可以利用**Predicate断言**来自定义路由规则。

​		Spring Cloud Gateway 创建 Route 对象时， 使用 RoutePredicateFactory 创建 Predicate 对象，Predicate 对象可以赋值给 Route。

​		路由断言工厂 RoutePredicateFactory 主要包括 Datetime、 请求的远端地址、 路由权重、 请求头、 Host 地址、 请求方法、 请求路径和请求参数等类型的路由断言。

​		下面我会举例说明其中一部分如何使用，注意设置中的uri以个人实际情况为主。以我的程序为例，服务提供者端口7002，存在查询产品列表的接口，`http://localhost:7002/product/list`，消费者端口7003，存在根据订单ID查询信息的接口，`http://localhost:7003/order/1`

### Path

```yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
        - id: product-service           # 路由 ID，唯一
          uri: http://localhost:7002/   # 目标 URI，路由到微服务的地址
          predicates:                   # 断言（判断条件）
            - Path=/product/**          # 匹配对应 URL 的请求，将匹配到的请求追加在目标 URI 之后
        - id: consumer-service           
          uri: http://localhost:7003/   
          predicates:                  
            - Path=/order/**         
```

如上配置，请求`http://localhost:7009/order/1`就会被路由到`http://localhost:7003/order/1`，请求`http://localhost:7009/product/list`就会被路由到`http://localhost:7002/product/list`

### Query

```yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
      - id: product-service           
          uri: http://localhost:7002/  
          predicates:                   
            - Query=token               # 匹配请求参数中包含 token 的请求
abc. 的请求
```

### Method

```yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
      - id: product-service           
          uri: http://localhost:7002/   
          predicates:                   
            - Method=GET                # 匹配任意 GET 请求
```



### Datetime

```yaml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
      - id: product-service           
          uri: http://localhost:7002/   
          predicates:                   
            # 匹配中国上海时间 2020-02-02 20:20:20 之后的请求
            - After=2020-02-02T20:20:20.000+08:00[Asia/Shanghai]
```

更多详情可以参考[官网](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.2.RELEASE/reference/html/#gateway-request-predicates-factories)	

## 2.4 过滤器

​		与Zuul一样，Spring Cloud GateWay也提供了完善的过滤器机制，Spring Cloud Gateway 根据作用范围划分为 `GatewayFilter` 和 `GlobalFilter`，二者区别如下：

- `GatewayFilter`：网关过滤器，需要通过 `spring.cloud.routes.filters` 配置在具体路由下，只作用在当前路由上或通过 `spring.cloud.default-filters` 配置在全局，作用在所有路由上。
- `GlobalFilter`：全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过 `GatewayFilterAdapter` 包装成 `GatewayFilterChain` 可识别的过滤器，它为请求业务以及路由的 URI 转换为真实业务服务请求地址的核心过滤器，不需要配置系统初始化时加载，并作用在每个路由上。

### 网关过滤器GatewayFilter

​		网关过滤器用于拦截并链式处理 Web 请求，可以实现横切与应用无关的需求，比如：安全、访问超时的设置等。以下是Spring Cloud Gateway 包含的网关过滤器工厂

![](https://s1.ax1x.com/2020/04/30/JbnjNF.md.png)

​		以Path路径过滤器为例，简单介绍其使用，详细配置可参考 [官网](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.2.RELEASE/reference/html/#gatewayfilter-factories)

- PrefixPath

  PrefixPath 网关过滤器工厂为匹配的 URI 添加指定前缀。

  ```yaml
  server:
    port: 7009
  spring:
    application:
      name: cloud-gateway
    cloud:
      gateway:
        routes:
          - id: product-service           
            uri: http://localhost:7002/   
            predicates:                  
              - Path=**
             filters:                       # 网关过滤器
              - PrefixPath=/product
  ```

  请求`http://localhost:7009/list`就会被路由到`http://localhost:7002/product/list`

- StripPrefix

  StripPrefix 网关过滤器工厂采用一个参数 StripPrefix，该参数表示在将请求发送到下游之前从请求中剥离的路径个数。

  ```yaml
  server:
    port: 7009
  spring:
    application:
      name: cloud-gateway
    cloud:
      gateway:
        routes:
          - id: product-service           
            uri: http://localhost:7002/   
            predicates:                  
              - Path=**
             ilters:                       # 网关过滤器
              # 将 /api/123/product/list 重写为 /product/list
              - StripPrefix=2
  ```

### 全局过滤器GlobalFilter

​		全局过滤器不需要在配置文件中配置，作用在所有的路由上，最终通过 GatewayFilterAdapter 包装成 GatewayFilterChain 可识别的过滤器，它是请求业务以及路由的 URI 转换为真实业务服务请求地址的核心过滤器，不需要配置系统初始化时加载，并作用在每个路由上。

​		![](https://s1.ax1x.com/2020/04/30/Jbu9j1.png)

### 自定义全局过滤器

​		自定义全局过滤器需要实现以下两个接口 ：`GlobalFilter`，`Ordered`。通过全局过滤器可以实现权限校验，安全性验证等功能。

​		以之前完成的Zuul中的统一鉴权过滤器为例，将其改写

```java
@Component
public class TokenFilter implements GlobalFilter, Ordered {
    /**
     * 过滤器业务逻辑
     *
     * @param exchange
     * @param chain
     * @return
     */
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取请求参数
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        // 业务逻辑处理
        if (null == token) {
            ServerHttpResponse response = exchange.getResponse();
            // 响应类型
            response.getHeaders().add("Content-Type", "application/json; charset=utf-8");
            // 响应状态码，HTTP 401 错误代表用户没有访问权限
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            // 响应内容
            String message = "{\"message\":\"" + HttpStatus.UNAUTHORIZED.getReasonPhrase() + "\"}";
            DataBuffer buffer = response.bufferFactory().wrap(message.getBytes());
            // 请求结束，不在继续向下请求
            return response.writeWith(Mono.just(buffer));
        }
        return chain.filter(exchange);
    }

    /**
     * 过滤器执行顺序，数值越小，优先级越高
     *
     * @return
     */
    @Override
    public int getOrder() {
        return 1;
    }

}
```

















