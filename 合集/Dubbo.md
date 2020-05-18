Dubbo

# 1.基础知识

## 1.1RPC

Remote Procedure Call，远程过程调用，一种技术思想。允许程序员调用另一个地址空间的过程或函数。

![RPC原理](D:\Program Files\笔记\image\RPC原理.JPG)

## 1.2 dubbo概念

Apache Dubbo是一款高性能、轻量级的开源Java RPC框架，它提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。

![dubbo结构](D:\Program Files\笔记\image\dubbo结构.JPG)

## 1.3 注册中心zookeeper

初次运行会报错，将conf目录下的zoo_sample.cfg文件复制，修改为zoo.cfg即可。

dataDir=   临时数据存储的目录

clientPort=   端口号

## 1.4 dubbo使用

1. 将服务提供者注册到注册中心

   导入dubbo和zookeeper客户端依赖

   ```xml
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>dubbo</artifactId>
       <version>2.6.5</version>
   </dependency>
   
   <dependency>
       <groupId>org.apache.curator</groupId>
       <artifactId>curator-framework</artifactId>
       <version>4.1.0</version>
   </dependency>
   ```

2. 配置服务提供者

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    
       <!-- 指定当前应用的名字 -->
       <dubbo:application name="hello-world-app"  />
    
       <!-- 指定注册中心地址 -->
       <dubbo:registry address="zookeeper://10.20.153.10:2181" />
    
       <!-- 指定通信规则。用dubbo协议在20880端口暴露服务 -->
       <dubbo:protocol name="dubbo" port="20880" />
    
       <!-- 声明需要暴露的服务接口 ref指向接口的实现，如下面注册的bean-->
       <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService" />
    
       <!-- 和本地bean一样实现服务 -->
       <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl" />
   </beans>
   ```

3. 服务消费者

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans        http://www.springframework.org/schema/beans/spring-beans-4.3.xsd        http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    
       <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->
       <dubbo:application name="consumer-of-helloworld-app"  />
    
       <!-- 使用multicast广播注册中心暴露发现服务地址 -->
       <dubbo:registry address="multicast://224.5.6.7:1234" />
    
       <!-- 生成远程服务代理，可以和本地bean一样使用demoService -->
       <dubbo:reference id="demoService" interface="org.apache.dubbo.demo.DemoService" />
   </beans>
   ```

4. 整合SpringBoot

   ```xml
    <dependency>
           <groupId>org.apache.dubbo</groupId>
           <artifactId>dubbo-spring-boot-starter</artifactId>
           <version>2.7.4.1</version>
       </dependency>
       
       <dependency>
           <groupId>org.apache.dubbo</groupId>
           <artifactId>dubbo</artifactId>
       </dependency>
   ```

首先，我们假设存在一个 Dubbo RPC API ，由服务提供方为服务消费方暴露接口 :

```
public interface DemoService {

    String sayHello(String name);

}
```

实现 Dubbo 服务提供方

1. 实现 `DemoService` 接口

   ```
   @Service(version = "1.0.0")
   public class DefaultDemoService implements DemoService {
   
       /**
        * The default value of ${dubbo.application.name} is ${spring.application.name}
        */
       @Value("${dubbo.application.name}")
       private String serviceName;
   
       public String sayHello(String name) {
           return String.format("[%s] : Hello, %s", serviceName, name);
       }
   }
   ```

2. 编写 Spring Boot 引导程序

   ```
   @EnableAutoConfiguration
   public class DubboProviderDemo {
   
       public static void main(String[] args) {
           SpringApplication.run(DubboProviderDemo.class,args);
       }
   }
   ```

3. 配置 `application.properties` :

   ```
   # Spring boot application
   spring.application.name=dubbo-auto-configuration-provider-demo
   # Base packages to scan Dubbo Component: @org.apache.dubbo.config.annotation.Service
   dubbo.scan.base-packages=org.apache.dubbo.spring.boot.demo.provider.service
   
   # Dubbo Application
   ## The default value of dubbo.application.name is ${spring.application.name}
   ## dubbo.application.name=${spring.application.name}
   
   # Dubbo Protocol
   dubbo.protocol.name=dubbo
   dubbo.protocol.port=12345
   
   ## Dubbo Registry
   dubbo.registry.address=zookeeper://127.0.0.1:2181
   ```

实现 Dubbo 服务消费方

1. 通过 `@Reference` 注入 `DemoService` (要么通过POM坐标引入，要么在消费者本地创建与提供者相同路径的接口)

   ```
   @EnableAutoConfiguration
   public class DubboAutoConfigurationConsumerBootstrap {
   
       private final Logger logger = LoggerFactory.getLogger(getClass());
   
       @Reference(version = "1.0.0", url = "dubbo://127.0.0.1:12345")
       private DemoService demoService;
   
       public static void main(String[] args) {
           SpringApplication.run(DubboAutoConfigurationConsumerBootstrap.class).close();
       }
   
       @Bean
       public ApplicationRunner runner() {
           return args -> {
               logger.info(demoService.sayHello("mercyblitz"));
           };
       }
   }
   ```

2. 配置 `application.yml` :

   ```
   spring:
     application:
       name: dubbo-auto-configure-consumer-sample
   ```

# 2.配置

### 2.1 属性优先级

![dubbo配置优先级](D:\Program Files\笔记\image\dubbo配置优先级.JPG)

### 2.2 覆盖关系

![dubbo配置覆盖关系](D:\Program Files\笔记\image\dubbo配置覆盖关系.JPG)

### 2.3 启动检查

Dubbo 缺省会在启动时检查依赖的服务是否可用，不可用时会抛出异常，阻止 Spring 初始化完成，以便上线时，能及早发现问题，默认 `check="true"`。

关闭某个服务的启动时检查 (没有提供者时报错)：

```xml
<dubbo:reference interface="com.foo.BarService" check="false" />
```

关闭所有服务的启动时检查 (没有提供者时报错)：

```xml
<dubbo:consumer check="false" />
```

关闭注册中心启动时检查 (注册订阅失败时报错)：

```xml
<dubbo:registry check="false" />
```

### 2.4 超时

```xml
<dubbo:reference interface="com.foo.BarService" timeout="2000" />
```

远程服务调用超时时间(毫秒)，默认1000

### 2.5 重试次数

retries 默认为2。远程服务调用重试次数，不包括第一次调用，不需要重试请设为0

```xml
<dubbo:reference interface="com.foo.BarService" retries ="10" />
```

### 2.6 多版本

当一个接口实现，出现不兼容升级时，可以用版本号过渡，版本号不同的服务相互间不引用。

老版本服务提供者配置：

```xml
<dubbo:service interface="com.foo.BarService" version="1.0.0" />
```

新版本服务提供者配置：

```xml
<dubbo:service interface="com.foo.BarService" version="2.0.0" />
```

老版本服务消费者配置：

```xml
<dubbo:reference id="barService" interface="com.foo.BarService" version="1.0.0" />
```

新版本服务消费者配置：

```xml
<dubbo:reference id="barService" interface="com.foo.BarService" version="2.0.0" />
```

### 2.7 本地存根

远程服务后，客户端通常只剩下接口，而实现全在服务器端，但提供方有些时候想在客户端也执行部分逻辑。比如：做 ThreadLocal 缓存，提前验证参数，调用失败后伪造容错数据等等，此时就需要在 API 中带上 Stub。

在 spring 配置文件中按以下方式配置：

```xml
<dubbo:service interface="com.foo.BarService" stub="true" />
```

或

```xml
<dubbo:service interface="com.foo.BarService" stub="com.foo.BarServiceStub" />
```

```java
package com.foo;
public class BarServiceStub implements BarService {
    private final BarService barService;
    
    // 构造函数传入真正的远程代理对象
    public BarServiceStub(BarService barService){
        this.barService = barService;
    }
 
    public String sayHello(String name) {
        // 此代码在客户端执行, 你可以在客户端做ThreadLocal本地缓存，或预先验证参数是否合法，等等
        //在调用服务端接口前执行
        try {
            return barService.sayHello(name);
        } catch (Exception e) {
            // 你可以容错，可以做任何AOP拦截事项
            return "容错数据";
        }
    }
}
```

Stub 必须有可传入 Proxy 的构造函数。

### 2.8 协议

- dubbo

  Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。

  反之，Dubbo 缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

  因为服务的现状大都是服务提供者少，通常只有几台机器，而服务的消费者多，**通过单一连接，保证单一消费者不会压死提供者，长连接，减少连接握手验证等，并使用异步 IO，复用线程池。**

  且也因为Dubbo使用单一长连接，如果每次请求的数据包大小为 500KByte，假设网络为千兆网卡，每条连接最大 7MByte(不同的环境可能不一样，供参考)，单个服务提供者的 TPS(每秒处理事务数)最大为：128MByte / 500KByte = 262。单个消费者调用单个服务提供者的 TPS(每秒处理事务数)最大为：7MByte / 500KByte = 14。如果能接受，可以考虑使用，否则网络将成为瓶颈。

  ![单一场链接](D:\Program Files\笔记\image\单一场链接.png)

  连接一次后，所有的通道都可以在第一次通道的基础上进行通信

- RMI

  RMI 协议采用 JDK 标准的 `java.rmi.*` 实现，采用阻塞式短连接和 JDK 标准序列化方式。

  Java RMI（Java Remote Method Invocation），即Java远程方法调用。是Java编程语言里，一种用于实现远程过程调用的应用程序**编程接口**。

- HTTP

  基于 HTTP 表单的远程调用协议，采用 Spring 的 HttpInvoker 实现 

- Redis

  基于Redis实现，`2.3.0` 以上版本支持

### 2.9 注册中心

- **Multicast 注册中心**

  Multicast 注册中心不需要启动任何中心节点，只要广播地址一样，就可以互相发现。

- **Nacos注册中心**

- **Redis注册中心**

- **Zookeeper注册中心**

# 3.高可用

### 3.1 zookeeper宕机

![dubbo注册中心宕机](D:\Program Files\笔记\image\dubbo注册中心宕机.JPG)

### 3.2 负载均衡

在集群负载均衡时，Dubbo 提供了多种均衡策略，缺省为 `random` 随机调用。

#### Random LoadBalance

- **随机**，按权重设置随机概率。
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

#### RoundRobin LoadBalance

- **轮询**，按公约后的权重设置轮询比率。
- 存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

#### LeastActive LoadBalance

- **最少活跃调用数**，相同活跃数的随机，活跃数指调用前后计数差。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

#### ConsistentHash LoadBalance

- **一致性 Hash**，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 算法参见：http://en.wikipedia.org/wiki/Consistent_hashing
- 缺省只对第一个参数 Hash，如果要修改，请配置 `<dubbo:parameter key="hash.arguments" value="0,1" />`
- 缺省用 160 份虚拟节点，如果要修改，请配置 `<dubbo:parameter key="hash.nodes" value="320" />`

### 3.3  服务降级

可以通过服务降级功能临时屏蔽某个出错的非关键服务，并定义降级后的返回策略。

向注册中心写入动态配置覆盖规则：

```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null"));
```

其中：

- `mock=force:return+null` 屏蔽，表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
- 还可以改为 `mock=fail:return+null` 容错，表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。

### 3.4 服务容错

在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 failover 重试

- Failover Cluster

失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 `retries="2"` 来设置重试次数(不含第一次)。

重试次数配置如下：

```xml
<dubbo:service retries="2" />
```

或

```xml
<dubbo:reference retries="2" />
```

或

```xml
<dubbo:reference>
    <dubbo:method name="findFoo" retries="2" />
</dubbo:reference>
```

- Failfast Cluster

快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。

- Failsafe Cluster

失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

- Failback Cluster

失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

- Forking Cluster

并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 `forks="2"` 来设置最大并行数。

- Broadcast Cluster

广播调用所有提供者，逐个调用，任意一台报错则报错 [[2\]](http://dubbo.apache.org/zh-cn/docs/user/demos/fault-tolerent-strategy.html#fn2)。通常用于通知所有提供者更新缓存或日志等本地资源信息。

# 4.原理

## 4.1 SPI机制

Dubbo 的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。

Dubbo 改进了JDK 标准的 SPI 以下问题：

- JDK 标准的 SPI **会一次性实例化扩展点所有实现**，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
- 增加了**对扩展点 IoC 和 AOP 的支持**，一个扩展点可以直接 setter 注入其它扩展点。

### 约定：

在扩展类的 jar 包内（**放在你自己的 jar 包内，不是 dubbo 本身的 jar 包内**） ，放置扩展点配置文件 `META-INF/dubbo/接口全限定名`，内容为：`配置名=扩展实现类全限定名`，多个实现类用换行符分隔。

### 示例：

以扩展 Dubbo 的协议为例，在协议的实现 jar 包内放置文本文件：`META-INF/dubbo/org.apache.dubbo.rpc.Protocol`，内容为：

```properties
xxx=com.alibaba.xxx.XxxProtocol
```

实现类内容 [[2\]](http://dubbo.apache.org/zh-cn/docs/dev/SPI.html#fn2)：

```java
package com.alibaba.xxx;
 
import org.apache.dubbo.rpc.Protocol;
 
public class XxxProtocol implements Protocol { 
    // ...
}
```

### 源码分析

Dubbo SPI 的相关逻辑被封装在了 `ExtensionLoader` 类中，通过 ExtensionLoader，我们可以加载指定的实现类。Dubbo SPI 所需的配置文件需放置在 META-INF/dubbo 路径下。另外，在测试 Dubbo SPI 时，需要在 对应接口上标注 `@SPI` 注解。下面来演示 Dubbo SPI 的用法：

```java
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = 
            ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

首先通过 ExtensionLoader 的 getExtensionLoader 方法获取一个 ExtensionLoader 实例，然后再通过 ExtensionLoader 的 getExtension 方法获取拓展类对象。这其中，getExtensionLoader 方法用于从缓存中获取与拓展类对应的 ExtensionLoader，若缓存未命中，则创建一个新的实例。

```java
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        // 获取默认的拓展实现类
        return getDefaultExtension();
    }
    // Holder，顾名思义，用于持有目标对象,检查缓存，缓存未命中则创建拓展对象
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 双重检查
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例
                instance = createExtension(name);
                // 设置实例到 holder 中
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

首先检查缓存，缓存未命中则创建拓展对象，其中缓存由ConcurrentHashMap构建

```java
private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
```

创建拓展对象的过程

```java
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

首先需要根据配置文件解析出拓展项名称到拓展类的映射关系表（Map<名称, 拓展类>），之后再根据拓展项名称从映射关系表中取出相应的拓展类即可。相关过程的代码分析如下：

```java
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

loadExtensionClasses 方法总共做了两件事情，**一是对 SPI 注解进行解析，二是调用 loadDirectory 方法加载指定文件夹配置文件。**

## 4.2 RPC原理

![RPC调用过程](D:\Program Files\笔记\image\RPC调用过程.JPG)

## 4.3 服务导出

服务导出的入口方法是 ServiceBean 的 onApplicationEvent。onApplicationEvent 是一个事件响应方法，该方法会在收到 Spring 上下文刷新事件后执行服务导出操作。方法代码如下：

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 是否有延迟导出 && 是否已导出 && 是不是已被取消导出
    if (isDelay() && !isExported() && !isUnexported()) {
        // 导出服务
        export();
    }
}
```

### 前置工作

前置工作主要包含两个部分，分别是配置检查，以及 URL 装配。

Dubbo 使用 **URL 作为配置载体**，所有的拓展点都是通过 URL 获取配置

#### 配置检查

从 export 方法开始进行分析，如下：

```java
public synchronized void export() {
    if (provider != null) {
        // 获取 export 和 delay 配置
        if (export == null) {
            export = provider.getExport();
        }
        if (delay == null) {
            delay = provider.getDelay();
        }
    }
    // 如果 export 为 false，则不导出服务
    if (export != null && !export) {
        return;
    }

    // delay > 0，延时导出服务
    if (delay != null && delay > 0) {
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
        
    // 立即导出服务
    } else {
        doExport();
    }
}
```

如上，实际执行doExport()方法，进行配置检查

#### 组装 URL

URL 组装的过程。

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    // 如果协议名为空，或空串，则将协议名变量设置为 dubbo
    if (name == null || name.length() == 0) {
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
    // 添加 side、版本、时间戳以及进程号等信息到 map 中
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }

    // 通过反射将对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    appendParameters(map, protocolConfig);
    appendParameters(map, this);

    // methods 为 MethodConfig 集合，MethodConfig 中存储了 <dubbo:method> 标签的配置信息
    if (methods != null && !methods.isEmpty()) {
        // 这段代码用于添加 Callback 配置到 map 中，代码太长，待会单独分析
    }

    // 检测 generic 是否为 "true"，并根据检测结果向 map 中添加不同的信息
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }

        // 为接口生成包裹类 Wrapper，Wrapper 中包含了接口的详细信息，比如接口方法名数组，字段信息等
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        // 添加方法名到 map 中，如果包含多个方法名，则用逗号隔开，比如 method = init,destroy
        if (methods.length == 0) {
            logger.warn("NO method found in service interface ...");
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            // 将逗号作为分隔符连接方法名，并将连接后的字符串放入 map 中
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    // 添加 token 到 map 中
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            // 随机生成 token
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    // 判断协议名是否为 injvm
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }

    // 获取上下文路径
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }

    // 获取 host 和 port
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    // 组装 URL
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    
    // 省略无关代码
}
```

上面的代码首先是将一些信息，比如版本、时间戳、方法名以及各种配置对象的字段信息放入到 map 中，map 中的内容将作为 URL 的查询字符串。构建好 map 后，紧接着是获取上下文路径、主机名以及端口号等信息。最后将 map 和主机名等数据传给 URL 构造方法创建 URL 对象。需要注意的是，这里出现的 URL 并非 java.net.URL，而是 com.alibaba.dubbo.common.URL。

# 5.Dubbo2.7升级特性

Dubbo与2019年升级至2.7版本，其中，最重要的有三个更新

- 异步化改造

  ![dubbo远程方法调用方式](D:\Program Files\笔记\image\dubbo远程方法调用方式.png)

  

  oneway 指的是客户端发送消息后，不需要接受响应。对于那些不关心服务端响应的请求，比较适合使用 oneway 通信。

  sync 是最常用的通信方式，也是默认的通信方法。客服端发起req请求到A服务端，然后在设置的超时时间内，一直等待A服务器的响应resp，这个时候，我们说客户端处于阻塞的状态。当A服务器返回resp后，客户端才会继续运行

  future 和 callback 都属于异步调用的范畴，他们的区别是：在接收响应时，future.get() 会导致线程的阻塞;

  callback 通常会设置一个回调线程，当接收到响应时，自动执行，不会对当前线程造成阻塞。

- 三大中心改造

  三大中心指的：注册中心，元数据中心，配置中心。

  在 2.7 之前的版本，Dubbo 只配备了注册中心，主流使用的注册中心为 zookeeper。新增加了元数据中心和配置中心。

  **元数据中心**

  元数据定义为描述数据的数据，在服务治理中，例如服**务接口名，重试次数，版本号等等都可以理解为元数据**

  在 2.7 之前，元数据一股脑丢在了注册中心之中，这造成了一系列的问题：

  **推送量大 -> 存储数据量大 -> 网络传输量大 -> 延迟严重**

  生产者端注册 30+ 参数，有接近一半是不需要作为注册中心进行传递；消费者端注册 25+ 参数，只有个别需要传递给注册中心。有了以上的理论分析，Dubbo 2.7 进行了大刀阔斧的改动，只将真正属于服务治理的数据发布到注册中心之中，大大降低了注册中心的负荷。

  将全量的元数据发布到另外的组件中：元数据中心。元数据中心目前支持 redis（推荐），zookeeper。

  ```xml
  <dubbo:metadata-report address="zookeeper://127.0.0.1:2181"/>
  ```

  **配置中心**

  在 2.7 中，Dubbo 正式支持了配置中心，目前支持的几种注册中心 Zookeeper，Apollo，Nacos（2.7.1-release 支持）。

  在 Dubbo 中，配置中心主要承担了两个作用

  - 外部化配置。启动配置的集中式存储
  - 服务治理。服务治理规则的存储与通知

  ```xml
  <dubbo:config-center address="zookeeper://127.0.0.1:2181"/>
  ```

- 服务治理增强

  在 2.7 中，Dubbo 对其服务治理能力进行了增强，增加了标签路由的能力，并抽象出了应用路由和服务路由的概念。