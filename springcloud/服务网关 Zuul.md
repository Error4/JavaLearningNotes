# 1.什么是服务网关

​	API Gateway（APIGW / API 网关），顾名思义，是出现在系统边界上的一个面向 API 的、串行集中式的强管控服务，这里的边界是企业 IT 系统的边界，可以理解为`企业级应用防火墙`，主要起到`隔离外部访问与内部系统的作用`

# 2.为什么需要 API Gateway

**1、简化客户端调用复杂度**

在微服务架构模式下后端服务的实例数一般是动态的，对于客户端而言很难发现动态改变的服务实例的访问地址信息

**2、数据裁剪以及聚合**

为了优化客户端的使用体验，API Gateway 可以对通用性的响应数据进行裁剪以适应不同客户端的使用需求。

**3、多渠道支持**

可以针对不同的渠道和客户端提供不同的 API Gateway

**4、遗留系统的微服务化改造**

# 3.什么是Zuul

​		Zuul 是从设备和网站到应用程序后端的所有请求的前门。作为边缘服务应用程序，Zuul 旨在实现动态路由，监视，弹性和安全性。Zuul 包含了对请求的**路由**和**过滤**两个最主要的功能。

　　Zuul 是 Netflix 开源的微服务网关，它可以和 Eureka、Ribbon、Hystrix 等组件配合使用。其核心就是一系列过滤器，通过这些过滤器可以完成例如身份认证与安全、审查与监控、动态路由等功能。

# 4.Zuul实现API网关

## POM 配置

在 `pom.xml` 中引入以下依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

## 配置文件

在配置文件 application.yml 中加入服务名、端口号、Eureka 注册中心的地址：

```yaml
spring:
  application:
    name: zuul-server
server:
  port: 7009
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

## 启动类

使用 `@EnableZuulProxy` 注解开启 Zuul 的功能

```java
@SpringBootApplication
@EnableZuulProxy
@EnableEurekaClient
public class ZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

启动服务

![](https://s1.ax1x.com/2020/04/29/J7tiTI.jpg)

​		由于 Spring Cloud Zuul 在整合了 Eureka 之后，具备默认的服务路由功能，即：当我们这里构建的 `api-gateway` 应用启动并注册到 Eureka 之后，服务网关会发现上面我们启动的两个服务 `producer` 和 `consumer`，这时候 Zuul 就会创建两个路由规则。每个路由规则都包含两部分，一部分是外部请求的匹配规则，另一部分是路由的服务 ID。针对当前示例的情况，Zuul 会创建下面的两个路由规则：

- 转发到 `eureka-provider` 服务的请求规则为：`/eureka-provider/**`

- 转发到 `service-consumer` 服务的请求规则为：`/service-consumer/**`

  ​	最后，我们可以通过访问 `7008` 端口的服务网关来验证上述路由的正确性：

- 比如访问：`http://localhost:7008/eureka-provider/product/list`，该请求将最终被路由到服务提供者 `eureka-provider` 的 对应方法上。

​		这样做虽然起到了基本的网关的作用，但同时也暴露了真实的微服务名称，可在yml文件中配置替换或排除，并且还能添加自定义前缀

```yml
zuul:
  routes:
    test-service:  #xxx代表路由id，自定义名称即可
    	serviceId: eureka-provider
    	path: /myProvider/**
  ignored-services: service-consumer  # 服务名称排除，多个服务逗号分隔，'*' 排除所有
  prefix: /test #设置公共的前缀
```

如上，添加routes，表示通过`/myProvider/`这个路径也能访问，并设置`ignored-services`，即`/service-consumer`不能访问，并添加前缀`/test`，即只能够通过`http://localhost:7008/test/myProvider/product/list` 访问

# 5.Zuul 过滤

Zuul 允许开发者在 API 网关上通过定义过滤器来实现对请求的拦截与过滤

## Filter 的生命周期

Filter 的生命周期有 4 个，分别是 “PRE”、“ROUTING”、“POST” 和 “ERROR”，整个生命周期可以用下图来表示

![](https://s1.ax1x.com/2020/04/29/JHp1yj.md.jpg)

Zuul 大部分功能都是通过过滤器来实现的，这些过滤器类型对应于请求的典型生命周期。

- **PRE：**这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。
- **ROUTING：**这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用 Apache HttpClient 或 Netfilx Ribbon 请求微服务。
- **POST：**这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的 HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。
- **ERROR：**在其他阶段发生错误时执行该过滤器。 除了默认的过滤器类型，Zuul 还允许我们创建自定义的过滤器类型。例如，我们可以定制一种 STATIC 类型的过滤器，直接在 Zuul 中生成响应，而不将请求转发到后端的微服务。

## Zuul 中默认实现的 Filter

| 类型  | 顺序 | 过滤器                  | 功能                         |
| :---- | :--- | :---------------------- | :--------------------------- |
| pre   | -3   | ServletDetectionFilter  | 标记处理 Servlet 的类型      |
| pre   | -2   | Servlet30WrapperFilter  | 包装 HttpServletRequest 请求 |
| pre   | -1   | FormBodyWrapperFilter   | 包装请求体                   |
| route | 1    | DebugFilter             | 标记调试标志                 |
| route | 5    | PreDecorationFilter     | 处理请求上下文供后续使用     |
| route | 10   | RibbonRoutingFilter     | serviceId 请求转发           |
| route | 100  | SimpleHostRoutingFilter | url 请求转发                 |
| route | 500  | SendForwardFilter       | forward 请求转发             |
| post  | 0    | SendErrorFilter         | 处理有错误的请求响应         |
| post  | 1000 | SendResponseFilter      | 处理正常的请求响应           |

## 禁用指定的 Filter

可以在 application.yml 中配置需要禁用的 filter，格式为 `zuul.<SimpleClassName>.<filterType>.disable=true`。
比如要禁用 `org.springframework.cloud.netflix.zuul.filters.post.SendResponseFilter` 就设置

```yaml
zuul:
  SendResponseFilter:
    post:
      disable: true
```

## 自定义 Filter

继承 ZuulFilter 抽象类，并添加注解@Component注入到容器当中

```java
public class TokenFilter extends ZuulFilter {

    /**
     * 过滤器的类型，它决定过滤器在请求的哪个生命周期中执行。
     * 这里定义为pre，代表会在请求被路由之前执行。
     * @return
     */
    
    public String filterType() {
        return "pre";
    }

    /**
     * filter执行顺序，通过数字指定。
     * 数字越大，优先级越低。
     *
     * @return
     */
    
    public int filterOrder() {
        return 0;
    }

    /**
     * 判断该过滤器是否需要被执行。这里我们直接返回了true，因此该过滤器对所有请求都会生效。
     * 实际运用中我们可以利用该函数来指定过滤器的有效范围。
     *
     * @return
     */
    
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器的具体逻辑
     *
     * @return
     */
    
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        String token = request.getParameter("token");
        if (token == null || token.isEmpty()) {
            // 请求结束，不在继续向下请求。
            ctx.setSendZuulResponse(false);
            // 响应状态码，HTTP 401 错误代表用户没有访问权限
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("token is empty");
        }
        return null;
    }
}
```

在上面实现的过滤器代码中，我们通过继承 `ZuulFilter` 抽象类并重写了下面的四个方法来实现自定义的过滤器。这四个方法分别定义了：

- `filterType()`：过滤器的类型，它决定过滤器在请求的哪个生命周期中执行。这里定义为 `pre`，代表会在请求被路由之前执行。
- `filterOrder()`：过滤器的执行顺序。当请求在一个阶段中存在多个过滤器时，需要根据该方法返回的值来依次执行。通过数字指定，数字越大，优先级越低。
- `shouldFilter()`：判断该过滤器是否需要被执行。这里我们直接返回了 `true`，因此该过滤器对所有请求都会生效。实际运用中我们可以利用该函数来指定过滤器的有效范围。
- `run()`：过滤器的具体逻辑。这里我们通过 `ctx.setSendZuulResponse(false)` 令 Zuul 过滤该请求，不对其进行路由，然后通过 `ctx.setResponseStatusCode(401)` 设置了其返回的错误码，当然我们也可以进一步优化我们的返回，比如，通过 `ctx.setResponseBody(body)` 对返回 body 内容进行编辑等。

此时，当我们进行访问时，如果URL中没有token参数，就会显示上述设置的返回信息`token is empty`。

相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看