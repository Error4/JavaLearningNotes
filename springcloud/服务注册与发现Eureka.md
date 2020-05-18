　服务注册中心是服务实现服务化管理的核心组件，类似于目录服务的作用，主要用来存储服务信息，譬如提供者 url 串、路由信息等。服务注册中心是微服务架构中最基础的设施之一。

　其实在微服务架构流行之前，注册中心就已经开始出现在分布式架构的系统中。比如Dubbo，它提供了比较完善的服务治理功能，而服务治理的实现主要依靠的就是注册中心，默认使用Zookeeper来作为注册中心。

# 1.常见的注册中心

| 特性            | Eureka      | Nacos                      | Consul            | Zookeeper  |
| :-------------- | :---------- | :------------------------- | :---------------- | :--------- |
| CAP             | AP          | CP + AP                    | CP                | CP         |
| 健康检查        | Client Beat | TCP/HTTP/MYSQL/Client Beat | TCP/HTTP/gRPC/Cmd | Keep Alive |
| 雪崩保护        | 有          | 有                         | 无                | 无         |
| 自动注销实例    | 支持        | 支持                       | 不支持            | 支持       |
| 访问协议        | HTTP        | HTTP/DNS                   | HTTP/DNS          | TCP        |
| 监听支持        | 支持        | 支持                       | 支持              | 支持       |
| 多数据中心      | 支持        | 支持                       | 支持              | 不支持     |
| 跨注册中心同步  | 不支持      | 支持                       | 支持              | 不支持     |
| SpringCloud集成 | 支持        | 支持                       | 支持              | 支持       |

# 2.Eureka注册中心

## 2.1 Eureka注册中心的组成

![](https://s1.ax1x.com/2020/04/28/J51z7D.jpg)

​		如上所示，可以视为包含两个组件，Eureka-Server和Eureka-Client。Server提供服务注册，各节点启动后，会在Server进行注册；Eureka-Client又可以分为Service Provider和Service Consumer，分别进行服务提供和服务消费

## 2.2 入门案例

利用IDEA新建项目，创建一个父工程，其中，在pom文件中添加依赖如下，

```xml
	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.4.RELEASE</version>
    </parent>
    <!--
        集中定义依赖组件版本号，但不引入，
        在子工程中用到声明的依赖时，可以不加依赖的版本号，
        这样可以统一管理工程中用到的依赖版本
     -->
    <properties>
        <!-- Spring Cloud Hoxton.SR1 依赖 -->
        <spring-cloud.version>Hoxton.SR1</spring-cloud.version>
    </properties>

    <!-- 项目依赖管理 父项目只是声明依赖，子项目需要写明需要的依赖(可以省略版本信息) -->
    <dependencyManagement>
        <dependencies>
            <!-- spring cloud 依赖 -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

### 2.2.1 注册中心Eureka-Server

在刚才创建的父工程下新建module，作为Eureka-Server

- 简单注册中心

  **pom.xml中添加依赖**

  ```xml
     <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
      </dependency>
  ```

  **application.yml** 

  在默认设置下，该服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为：

  ```yaml
  spring:
    application:
      name: eureka-server
  server:
    port: 7000
  eureka:
    instance:
      hostname: localhost
    client:
      register-with-eureka: false
      fetch-registry: false
      service-url:
        defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  ```

  - `server.port`：为了与后续要进行注册的服务区分，这里将服务注册中心的端口设置为 7000。
  - `eureka.client.register-with-eureka`：表示是否将自己注册到 Eureka Server，默认为 true。
  - `eureka.client.fetch-registry`：表示是否从 Eureka Server 获取注册信息，默认为 true。
  - `eureka.client.service-url.defaultZone`：设置与 Eureka Server 交互的地址，查询服务和注册服务都需要依赖这个地址。默认是 `http://localhost:7000/eureka` ；多个地址可使用英文逗号（,）分隔。

  **启动类**

  ​		通过 `@EnableEurekaServer` 注解启动一个服务注册中心提供给其他应用进行对话。这一步非常的简单，只需要在一个普通的 Spring Boot 应用中添加这个注解就能开启此功能，比如

  ```java
  @EnableEurekaServer
  @SpringBootApplication
  public class EurekaServerApplication {
  
      public static void main(String[] args) {
          SpringApplication.run(EurekaServerApplication.class, args);
      }
  }
  ```

  访问`localhost:7000`

  ![](https://s1.ax1x.com/2020/04/28/J54ohF.jpg)

- 高可用注册中心

  ​		注册中心这么关键的服务，如果是单点话，遇到故障就是毁灭性的。在一个分布式系统中，服务注册中心是最重要的基础部分，理应随时处于可以提供服务的状态。为了维持其可用性，使用集群是很好的解决方案。Eureka 通过**互相注册**的方式来实现高可用的部署。

  ​		与前文一样，我们可以创建一个新Module，命名为eureka-server02，pom文件的依赖相同，但需要修改他们的配置文件，使注册中心相互注册
  
  eureka-server 的 application.yml
  
  ```yaml
  server:
    port: 7002 # 端口
  
  spring:
    application:
      name: eureka-server # 应用名称(集群下相同)
  
  # 配置 Eureka Server 注册中心
  eureka:
    instance:
      hostname: eureka01            # 主机名，不配置的时候将根据操作系统的主机名来获取
    client:
      # 设置服务注册中心地址，指向另一个注册中心
      service-url:                  # 注册中心对外暴露的注册地址
       register-with-eureka: true
       fetch-registry: true
       defaultZone: http://localhost:7001/eureka/
  ```
  
  eureka-server02 的 application.yml
  
  ```yaml
  server:
    port: 7001 # 端口
  
  spring:
    application:
      name: eureka-server # 应用名称(集群下相同)
  
  # 配置 Eureka Server 注册中心
  eureka:
    instance:
      hostname: eureka02            # 主机名，不配置的时候将根据操作系统的主机名来获取
    client:
      # 设置服务注册中心地址，指向另一个注册中心
      service-url:                  # 注册中心对外暴露的注册地址
       register-with-eureka: true
       fetch-registry: true
       defaultZone: http://localhost:7002/eureka/
  ```

依次启动

访问：`http://localhost:7000`或者 `http://localhost:7001`都出现如下图说明互相注册成功。

![](https://s1.ax1x.com/2020/04/28/J5HQ0A.jpg)

**注意事项**

- `Status` 显示方式为默认值，如果想要清晰可见每个服务的 IP + 端口，需要添加配置

  ```yaml
  eureka:
    instance:
      prefer-ip-address: true       # 是否使用 ip 地址注册
      instance-id: ${spring.cloud.client.ip-address}:${server.port} # ip:port
  ```

- 在搭建 Eureka Server 双节点或集群的时候，要把 `eureka.client.register-with-eureka` 和 `eureka.client.fetch-registry` 均改为 `true`（默认）。否则会出现实例列表为空

- 在注册的时候，配置文件中的 `spring.application.name` 必须一致，否则两个节点会对应不同的application名称，如下所示

  ![](https://s1.ax1x.com/2020/04/28/J5b3DJ.jpg)

### 2.2.2 服务提供者Service Provider

pom

```xml
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

application.yml

```yaml
spring:
  application:
    name: eureka-provider

server:
  port: 7002
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka/ #指明服务注册到该地址
  instance:
    instance-id: eureka-provider #修改euraka默认描述信息
```

主程序

```java
@SpringBootApplication
@EnableEurekaClient
public class EurekaProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class,args);
    }
}
```

启动程序，服务提供者已被注册

![](https://s1.ax1x.com/2020/04/28/JIM3E4.jpg)

​		**补充：**可以注意到，我在`application.yml`中配置了`instance-id`属性，可以修改`euraka`默认描述信息，可以看到途中的Status栏中的名称已经变成了`eureka-provider`

​		另外，Status栏中的数据明显是个链接，但如果此时你点击，只会弹出显示404，这是因为**缺少对应的监控配置。**

`pom`添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

`application.yml`中补充信息，比如下例

```
#info
info:
  app.name: test
  company.name: test
  author.name: wyf
```

此时，启动服务提供者后，点击Status栏下的该链接，就会显示对应的数据，可以做一些描述性的工作

![](https://s1.ax1x.com/2020/04/28/JIlVlq.jpg)

此外，我们也可以在代码中利用`DiscoveryClient`获取服务信息

主程序添加@`EnableDiscoveryClient`

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class EurekaProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class,args);
    }
}
```

方法代码：

```java
@RestController
public class TestController {

    @Autowired
    private DiscoveryClient client;

    @GetMapping("/getService")
    public Object discovery(){
        //获取所有微服务的列表
        List<String> services = client.getServices();
               //根据serviceId（即指定的applicationName）获取对应服务
        List<ServiceInstance> provider = client.getInstances("eureka-provider");
        ServiceInstance serviceInstance = provider.get(0);
        return serviceInstance.getHost() +"-"+
                serviceInstance.getServiceId()+"-"+
                serviceInstance.getPort();
    }
}
```

输入`http://localhost:7002/getService`进行测试

![](https://s1.ax1x.com/2020/04/28/JI3JeK.jpg)

### 2.2.3 服务消费者Service Consumer

​		由于消费者牵扯到Ribbon以及Feign的知识，限于篇幅，这里先不进行说明，下一篇文章再进行展开。

## 2.3 Eureka 自动保护

​		一般情况下，服务在 Eureka 上注册后，会每 30 秒发送心跳包，Eureka 通过心跳来判断服务是否健康，同时会定期删除超过 90 秒没有发送心跳的服务。

但如果在 Eureka Server 的首页看到以下这段提示，则说明 Eureka 已经进入了保护模式。

> EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY’RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.

​		默认配置下，Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些**实例不会过期**，同时提示一个警告。

​		当它收到的心跳数重新恢复到阈值以上时，该 Eureka Server 节点就会自动退出自我保护模式。它的设计哲学前面提到过，那就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。

该模式可以在配置文件中通过 `eureka.server.enable-self-preservation = false` 来禁用



相关代码已提交至[github仓库](https://github.com/Error4/SpringCloudLearning)，有兴趣的朋友可以自行对比查看