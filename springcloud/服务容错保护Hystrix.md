通过前面的讲述，我们已经基本解决了服务注册与消费的问题，理想情况下，如果服务永远不会发生错误，那自然皆大欢喜，但在复杂的实际环境中，显然是不可能的，更何况在微服务的环境之下，存在数十个甚至数百个服务，服务之间还存在着更复杂的相互调用关系，一个请求需要调用多个服务是非常常见的，比如存在一条A->B->C的服务调用链，由于网络等原因影响，如果B 服务或者 C 服务不能及时响应，A 服务将处于阻塞状态，直到 B 服务 C 服务响应。此时若有大量的请求涌入，容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，造成连锁反应，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的**“雪崩”**效应。

Hystrix正是由Netflix开源的一个延迟和容错库，用于隔离访问远程系统、服务或者第三方库，防止级联失败，从而提升系统的可用性、容错性与局部应用的弹性，是一个实现了超时机制和断路器模式的工具类库。

# 1.使用 Hystrix 预防服务雪崩

## 服务降级（Fallback）

​		对于查询操作，我们可以实现一个 fallback 方法，当请求后端服务出现异常的时候，可以使用 fallback 方法返回的值。fallback 方法的返回值一般是设置的默认值或者来自缓存。

## 资源隔离

​		在 Hystrix 中，主要**通过线程池来实现资源隔离**。通常在使用的时候我们会根据调用的远程服务划分出多个线程池。例如调用产品服务的 Command 放入 A 线程池，调用账户服务的 Command 放入 B 线程池。这样做的主要优点是运行环境被隔离开了。这样就算调用服务的代码存在 bug 或者由于其他原因导致自己所在线程池被耗尽时，不会对系统的其他服务造成影响。

​		Hystrix 中除了使用线程池之外，还可以使用**信号量**来控制单个依赖服务的并发度，信号量的开销要远比线程池的开销小得多，但是它不能设置超时和实现异步访问。

## 断路器模式

![](https://s1.ax1x.com/2020/04/29/JTQ9Rs.jpg)

​		当 Hystrix Command 请求后端服务失败数量超过一定阈值，断路器会切换到开路状态 (Open)。这时所有请求会直接失败而不会发送到后端服务。

​		断路器保持在开路状态一段时间后 (默认 5 秒)，自动切换到半开路状态 (HALF-OPEN)。这时会判断下一次请求的返回情况，如果请求成功，断路器切回闭路状态 (CLOSED)，否则重新切换到开路状态 (OPEN)。

​		**在某一个服务单元发生故障时，通过断路器的故障监控，向调用方返回一个服务预期的，可处理的备选响应，而不是长时间等待或排除无法处理的异常，保证了调用方的线程不会被长时间，不必要的占用**

## 服务熔断

​		服务熔断的作用类似于我们家用的保险丝，当某服务出现不可用或响应超时的情况时，为了防止整个系统出现雪崩，暂时停止对该服务的调用。

## 服务降级

​		服务降级是从整个系统的负荷情况出发和考虑的，对某些负荷会比较高的情况，为了预防某些功能（业务场景）出现负荷过载或者响应慢的情况，在其内部暂时舍弃对一些非核心的接口和数据的请求，而直接返回一个提前准备好的fallback（退路）错误处理信息。这样，虽然提供的是一个有损的服务，但却保证了整个系统的稳定性和可用性。

# 2.Feign中使用Hystrix

​		Feign在整合到Spring Cloud时已经自带了hystrix模块，所以pom.xml中不需要额外引入feign依赖。只需要在配置文件中开启即可。

## 配置文件

​		在原来的 application.yml 配置的基础上修改

```yaml
spring:
  application:
    name: hystrix
server:
  port: 7006
eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:7000/eureka/
feign:
  hystrix:
    enabled: true
```

## 创建回调类

​		注意，必须要实现对应的接口

```java
@Component
public class ProductServiceHystrix implements ProductService {

    @Override
    public Product selectProductById(Integer id) {
        return new Product(id, "不存在", 1, 0D);
    }
}
```

## 添加 fallback 属性

​		在 `@FeignClient` 中添加指定 fallback 类，在服务熔断的时候返回 fallback 类中的内容

```java
@FeignClient(value = "eureka-provider",fallback = ProductServiceHystrix.class)
public interface ProductService {
    /**
     * 根据主键查询商品
     * @param id
     * @return
     */
    @GetMapping("/product/{id}")
    Product selectProductById(@PathVariable("id") Integer id);
}
```

# 3.Hystrix 监控

​		Hystrix 除了可以实现服务容错之外，还提供了近乎实时的监控功能，将服务执行结果和运行指标，请求数量成功数量等等这些状态通过 `Actuator` 进行收集，然后访问 `/actuator/hystrix.stream` 即可看到实时的监控数据。

## 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

## 配置文件

​		开启 `hystrix.stream` 端点。如果希望所有端点暴露，配置为 `'*'`。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

## 启动类

```java
@SpringBootApplication
@EnableHystrix
public class HystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixApplication.class, args);
    }
}
```

​		启动之后，访问`http://localhost:7006/actuator`，可以看到已经开启了 `hystrix.stream` 端点。

![](https://s1.ax1x.com/2020/04/29/JTspYd.jpg)

​		进而通过访问`http://localhost:7006/actuator/hystrix.stream`来监控 Hystrix 的数据。可以看到，这种方式其实不利于观察服务运行状态，Hystrix提供了一套监控中心来进行查看

# 4.Hystrix  Dashboard

## 添加依赖

```xml
<!-- spring boot actuator 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- spring cloud netflix hystrix 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
<!-- spring cloud netflix hystrix dashboard 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

## 配置文件

​		没有变化

## 启动类

​		在 Spring Boot 的启动类上面引入注解 `@EnableHystrixDashboard`，启用 Hystrix Dashboard 功能。

```java
@SpringBootApplication
@EnableHystrix
@EnableHystrixDashboard
public class HystrixApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixApplication.class, args);
    }
}
```

​		启动应用，然后再浏览器中输入`http://localhost:7006/hystrix` 即可访问。监控中心页面如下

![](https://mrhelloworld.com/resources/articles/spring/spring-cloud/hystrix/image-20200205093400619.png)

​		输入能够返回监控数据的URL：`http://localhost:7006/actuator/hystrix.stream`，就能得到如下的监控中心图示。

![](https://s1.ax1x.com/2020/04/29/JTgI2j.png)

# 5.Hystrix 监控数据聚合 Turbine

​		通过 Hystrix Dashboard 我们只能实现对服务当个实例的数据展现，在生产环境我们的服务是肯定需要做高可用的，那么对于多实例的情况，我们就需要将这些度量指标数据进行聚合

### POM 配置

​		在 pom.xml 中添加以下依赖

```xml
 	<dependencies>
        <!-- spring-cloud netflix hystrix 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <!-- spring cloud netflix hystrix dashboard 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
        <!-- spring cloud netflix turbine 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
        </dependency>
    </dependencies>
```

### 启动类

​		在启动类上使用 `@EnableTurbine` 注解开启 Turbine

### 配置文件

​		在 application.yml 加入 Eureka 和 Turbine 的相关配置

```yaml
spring:
  application:
    name: turbine
server:
  port: 7007
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7000/eureka/
turbine:
  app-config: eureka-provider，feign-consumer
  cluster-name-expression: "default"
  combine-host-port: true
```

**参数说明**

- `turbine.app-config` 参数指定了需要收集监控信息的服务名，需要注意，这些被收集的服务都要像上文所述那样添加**Hystrix 监控**才可以；
- `turbine.cluster-name-expression` 参数指定了集群名称为 `default`，当我们服务数量非常多的时候，可以启动多个 Turbine 服务来构建不同的聚合集群，而该参数可以用来区分这些不同的聚合集群，同时该参数值可以在 Hystrix 仪表盘中用来定位不同的聚合集群，只需要在 Hystrix Stream 的 URL 中通过 cluster 参数来指定；
- `turbine.combine-host-port` 参数设置为 `true`，可以让同一主机上的服务通过主机名与端口号的组合来进行区分，默认情况下会以 host 来区分不同的服务，这会使得在本地调试的时候，本机上的不同服务聚合成一个服务来统计。

同理，在监控中心页面输入对应URL即可，以我的示例为例，输入`http://localhost:7007/turbine.stream`

相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看