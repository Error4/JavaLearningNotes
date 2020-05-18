在上一篇文章中，限于篇幅，并没有对服务消费进行展开讲解，希望通过这篇文章，让大家能对基于Ribbon的服务消费有所认识。废话不都说，我们直接上实例。

# 1.服务消费案例

老规矩，依然在父工程下先创建消费者项目，其中，添加的依赖与配置文件具体如下：

**添加依赖**

pom.xml

```xml
    <!-- 项目依赖 -->
    <dependencies>
        <!-- netflix eureka client 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!-- spring boot web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- lombok 依赖 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
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

**配置文件**

　　application.yml

```yaml
spring:
  application:
    name: service-consumer
server:
  port: 7003
eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

限于篇幅的关系，具体的测试用`service,controll,pojo`类等不再详述，大家可以自行创建或者查看我的github地址，代码都有上传。

对于基于Ribbon的服务消费，主要有两种实现方式：

- LoadBalancerClient：Ribbon 的负载均衡器
- @LoadBalanced：通过注解开启 Ribbon 的负载均衡器

## 1.1 LoadBalancerClient

```java
	@Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancerClient loadBalancerClient; // Ribbon 负载均衡器

    /**
     * 根据主键查询订单
     *
     * @param id
     * @return
     */
    @Override
    public Order selectOrderById(Integer id) {
        return new Order(id, "123456", "中国",
                selectProductList());
    }

    private List<Product> selectProductList() {
        StringBuffer sb = null;

        // 根据服务名称获取服务
        ServiceInstance si = loadBalancerClient.choose("eureka-provider");
        if (null == si) {
            return null;
        }

        sb = new StringBuffer();
        sb.append("http://" + si.getHost() + ":" + si.getPort() + "/product/list");

        // ResponseEntity: 封装了返回数据
        ResponseEntity<List<Product>> response = restTemplate.exchange(
                sb.toString(),
                HttpMethod.GET,
                null,
                new ParameterizedTypeReference<List<Product>>() {});
        return response.getBody();
    }
```

## 1.2 @LoadBalanced

启动类注入 `RestTemplate` 时添加 `@LoadBalanced` 负载均衡注解即可

```java
@SpringBootApplication
@EnableEurekaClient
public class ServiceConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumerApplication.class, args);
    }


    @Bean
    @LoadBalanced // 负载均衡注解
    //Spring Boot 不提供任何自动配置的RestTemplate bean，所以需要在启动类中注入 RestTemplate
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```

具体的业务实现

```java
	@Autowired
    private RestTemplate restTemplate;

    /**
     * 根据主键查询订单
     *
     * @param id
     * @return
     */
    @Override
    public Order selectOrderById(Integer id) {
        return new Order(id, "123456", "中国",
                selectProductListByLoadBalancerAnnotation());
    }
    private List<Product> selectProductListByLoadBalancerAnnotation() {
        // ResponseEntity: 封装了返回数据
        ResponseEntity<List<Product>> response = restTemplate.exchange(
                "http://eureka-provider/product/list",
                HttpMethod.GET,
                null,
                new ParameterizedTypeReference<List<Product>>() {});
        return response.getBody();
    }
```

访问`http://localhost:7003/order/1`

![](https://s1.ax1x.com/2020/04/29/Joltqf.jpg)

# 2.什么是Ribbon

​		相信通过上面的案例，我们已经对Ribbon有了一个感性的认识，它的确起到了服务消费的作用。那么，Ribbon具体是什么呢？

​		Ribbon 是一个基于 HTTP 和 TCP 的 客服端负载均衡工具，它是基于 Netflix Ribbon 实现的。

　　它不像 Spring Cloud 服务注册中心、配置中心、API 网关那样独立部署，但是它几乎存在于每个 Spring Cloud 微服务中。包括 Feign 提供的声明式服务调用也是基于该 Ribbon 实现的。

　　Ribbon 默认提供很多种负载均衡算法，例如轮询、随机等等。甚至包含自定义的负载均衡算法。

# 3 负载均衡策略

​	**Ribbon 提供了一套微服务的负载均衡解决方案。**其具体的负载均衡策略主要有以下一些：

## 3.1 轮询策略（默认）

　　策略对应类名：`RoundRobinRule`

　　实现原理：轮询策略表示每次都顺序取下一个 provider，比如一共有 3 个 provider，第 1 次取第 1 个，第 2 次取第 2 个，以此类推。

## 3.2 权重轮询策略

　　策略对应类名：`WeightedResponseTimeRule`

　　实现原理：

- 根据每个 provider 的响应时间分配一个权重，响应时间越长，权重越小，被选中的可能性越低。
- 原理：一开始为轮询策略，并开启一个计时器，每 30 秒收集一次每个 provider 的平均响应时间，当信息足够时，给每个 provider 附上一个权重，并按权重随机选择 provider，高权越重的 provider 会被高概率选中。

## 3.3 随机策略

　　策略对应类名：`RandomRule`

　　实现原理：从 provider 列表中随机选择一个。

## 3.4 最少并发数策略

　　策略对应类名：`BestAvailableRule`

　　实现原理：选择正在请求中的并发数最小的 provider，除非这个 provider 在熔断中。

## 3.5 重试策略

　　策略对应类名：`RetryRule`

　　实现原理：其实就是轮询策略的增强版，轮询策略服务不可用时不做处理，重试策略服务不可用时会重新尝试集群中的其他节点。

​		此外，如果查看其源码，会发现**这些策略都是实现接口`IRule`的，如果有需要，我们也可以自己实现`IRULE`接口，实现我们自定义的负载均衡策略。**

# 4.负载均衡策略设置

​		为了观察负载均衡的表现，我们将服务提供者复制一份出来，修改端口即可。同时。在消费者调用时，将其选择的服务提供者的地址打印在控制台，方便观察。

![](https://s1.ax1x.com/2020/04/29/JoGCO1.jpg)

​		此时，可以看到已经有了两个服务提供者，端口号分别为7002，7004

​		多次访问`http://localhost:7003/order/1`

![](https://s1.ax1x.com/2020/04/29/JoGh0x.jpg)

​		可以看到，Ribbon默认策略的确是轮询。

​		那么，如何根据自己的需要进行更改呢？也很简单，注入对应的负载均衡策略对象即可，其中，即可对全局进行设置，也可对局部进行设置。

## 4.1 全局设置

​		在启动类或配置类中注入即可，以随机策略为例

```java
@Configuration
public class Config {
    @Bean
    public RandomRule randomRule() {
        return new RandomRule();
    }
}
```

​		此时，多次访问``http://localhost:7003/order/1`，结果如下

![](https://s1.ax1x.com/2020/04/29/JottpD.jpg)

## 4.2 局部设置

​		修改配置文件指定服务的负载均衡策略。格式：`服务应用名.ribbon.NFLoadBalancerRuleClassName`

```yaml
# 负载均衡策略
# service-provider 为调用的服务的名称
service-provider:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```



相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看