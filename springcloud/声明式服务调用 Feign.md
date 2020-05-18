通过上一篇文章，我们知道基于Ribbon已经能够实现服务消费，那么Feign又能起到什么作用呢？简而言之，Feign进一步简化了Ribbon 和 RestTemplate，使代码更加简洁

# 1.什么是Feign

​		Feign 是 Spring Cloud Netflix 组件中的一个轻量级 RESTful 的 HTTP 服务客户端，实现了负载均衡和 Rest 调用的开源框架，封装了 Ribbon 和 RestTemplate，实现了 WebService 的面向接口编程，进一步降低了项目的耦合度。

　　Feign 内置了 Ribbon，用来做客户端负载均衡调用服务注册中心的服务。

　　Feign 本身并不支持 Spring MVC 的注解，它有一套自己的注解，为了更方便的使用，Spring Cloud 孵化了 OpenFeign。

　　Feign 支持的注解和用法请参考官方文档：https://github.com/OpenFeign/feign 或 spring.io 官网文档

　　可以说，使用 Feign 实现负载均衡是首选方案。**只需要你创建一个接口，然后在上面添加注解即可。**

​		Feign 是声明式服务调用组件，其核心就是：**像调用本地方法一样调用远程方法，无感知远程 HTTP 请求。**

- 它解决了让开发者调用远程接口就跟调用本地方法一样的体验，开发者完全感知不到这是远程方法，更感知不到这是个 HTTP 请求。无需关注与远程的交互细节，更无需关注分布式环境开发。
- 它像 Dubbo 一样，Consumer 直接调用 Provider 接口方法，而不需要通过常规的 Http Client 构造请求再解析返回数据。

# 2.Feign入门案例

与上文一样，我们同样在父项目下创建Feign相关的工程

引入依赖

```xml
<dependencies>
        <!-- netflix eureka client 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!-- spring cloud openfeign 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
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

配置文件

```yaml
spring:
  application:
    name: feign-consumer
server:
  port: 7005
eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://localhost:7000/eureka/
```

启动类

```java
@SpringBootApplication
@EnableFeignClients
public class FeignConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(FeignConsumerApplication.class, args);
    }
}
```

应用实例

```java
//指明调用的服务名称
@FeignClient("eureka-provider")
public interface ProductService {
    /**
     * 查询商品列表
     * @return
     */
    // 配置需要调用的服务地址及参数
    @GetMapping("/product/list")
    List<Product> selectProductList();
}
```

# 3.Feign负载均衡

​		Feign 封装了 Ribbon 自然也就集成了负载均衡的功能，默认采用轮询策略。修改策略的方式与之前学习 Ribbon 时讲解的配置是一致的，所以这里不再多言。

# 4.Feign请求传参

​		如何基于Feign在请求时传递参数呢？与SpringMVC类似，使用对应的注解即可

## 4.1 GET请求

 使用 `@PathVariable` 注解或 `@RequestParam` 注解接收请求参数。

**服务提供者**

　　ProductService.java

```java
/**
 * 根据主键查询商品
 *
 * @param id
 * @return
 */
Product selectProductById(Integer id);
```

　　ProductServiceImpl.java

```java
/**
 * 根据主键查询商品
 *
 * @param id
 * @return
 */
@Override
public Product selectProductById(Integer id) {
    return new Product(id, "冰箱", 1, 2666D);
}
```

　　ProductController.java

```java
@RestController
@RequestMapping("/product")
public class ProductController {

    @Autowired
    private ProductService productService;

    /**
     * 根据主键查询商品
     *
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    public Product selectProductById(@PathVariable("id") Integer id) {
        return productService.selectProductById(id);
    }

}
```

**服务消费者**

　　Productservice.java

```java
// 声明需要调用的服务
@FeignClient("eureka-provider")
public interface ProductService {

    /**
     * 根据主键查询商品
     *
     * @return
     */
    @GetMapping("/product/{id}")
    Product selectProductById(@PathVariable("id") Integer id);

}
```

　　OrderServiceImpl.java

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private ProductService productService;

    /**
     * 根据主键查询订单
     *
     * @param id
     * @return
     */
    @Override
    public Order selectOrderById(Integer id) {
        return new Order(id, "003", "中国",
                Arrays.asList(productService.selectProductById(5)));
    }

}
```

## 4.2 POST请求

使用 `@RequestBody` 注解接收请求参数。

**服务提供者**

　　PoductService.java

```java
/**
 * 新增商品
 *
 * @param product
 * @return
 */
Map<Object, Object> createProduct(Product product);
```

　　ProductServiceImpl.java

```java
/**
 * 新增商品
 *
 * @param product
 * @return
 */
@Override
public Map<Object, Object> createProduct(Product product) {
    System.out.println(product);
    return new HashMap<Object, Object>() {{
        put("code", 200);
        put("message", "新增成功");
    }};
}
```

　　ProductController.java

```java
@RestController
@RequestMapping("/product")
public class ProductController {

    @Autowired
    private ProductService productService;
    /**
     * 新增商品
     *
     * @param product
     * @return
     */
    @PostMapping("/save")
    public Map<Object, Object> createProduct(@RequestBody Product product) {
        return productService.createProduct(product);
    }

}
```

**服务消费者**

　　ProductService.java

```java
// 声明需要调用的服务
@FeignClient("eureka-provider")
public interface ProductService {
    /**
     * 新增商品
     *
     * @param user
     * @return
     */
    @PostMapping("/product/save")
    Map<Object, Object> createProduct(Product user);

}
```

　　ProductController.java

```java
@RestController
@RequestMapping("/product")
public class ProductController {
    @Autowired
    private ProductService productService;

    /**
     * 新增商品
     *
     * @param product
     * @return
     */
    @PostMapping("/save")
    public Map<Object, Object> createProduct(Product product) {
        return productService.createProduct(product);
    }

}
```

相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看