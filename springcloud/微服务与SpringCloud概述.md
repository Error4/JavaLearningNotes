# 1.什么是微服务

​		基本认为，微服务（Microservice）这个概念是2012年出现的，作为加快Web和移动应用程序开发进程的一种方法，2014年开始受到各方的关注，而微服务的流行，Martin Fowler功不可没。

​		Martin Fowler 是这样解释微服务概念的：

> 　　In short, the microservice architectural style is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API. These services are built around business capabilities and independently deployable by fully automated deployment machinery. There is a bare minimum of centralized management of these services, which may be written in different programming languages and use different data storage technologies.

　　翻译如下：

> 　　简而言之，微服务架构风格是一种将单个应用程序作为一套小型服务开发的方法，每种应用程序都在自己的进程中运行，并与轻量级机制（通常是HTTP资源API）进行通信。 这些服务是围绕业务功能构建的，可以通过全自动部署机制独立部署。 这些服务的集中管理最少，可以用不同的编程语言编写，并使用不同的数据存储技术。

​		微服务就是将一个单体架构的应用按业务划分为一个个的独立运行的程序即服务，它们之间通过 HTTP 协议进行通信（也可以采用消息队列来通信，如 RabbitMQ，Kafaka 等），可以采用不同的编程语言，使用不同的存储技术，自动化部署（如 Jenkins）减少人为控制，降低出错概率。服务数量越多，管理起来越复杂，因此采用集中化管理。例如 Eureka，Zookeeper 等都是比较常见的服务集中化管理框架。

　　微服务是一种架构风格。一个大型的复杂软件应用，由一个或多个微服务组成。系统中的各个微服务可被独立部署，各个微服务之间是松耦合的。每个微服务仅关注于完成一件任务并很好的完成该任务。

## 1.1 优点

- 测试容易
- 可伸缩性强
- 可靠性强
- 跨语言
- 协同开发
- 方便系统迭代

## 1.2 缺点

- 运维成本高，部署项目多
- 接口兼容版本问题
- 分布式系统的复杂性
- 分布式事务问题

## 1.3 Dubbo vs Spring Cloud

SpringCloud与Dubbo就像是品牌机与组装机的区别，SpringCloud覆盖更广更强大，但如果想更改，就要对其基础有足够的了解，Dubbo则与此相反。

借用一张以前的表格说明下，大家能体会到意思即可。

| 对比项       | Spring Cloud                 | Dubbo     |
| :----------- | :--------------------------- | :-------- |
| 服务注册中心 | Spring Cloud Netflix Eureka  | ZooKeeper |
| 服务调用方式 | REST API                     | RPC       |
| 服务网关     | Spring Cloud Netflix Zuul    | 无        |
| 断路由       | Spring Cloud Netflix Hystrix | 集群容错  |
| 分布式配置   | Spring Cloud Config          | 无        |
| 服务跟踪     | Spring Cloud Sleuth          | 无        |
| 消息总线     | Spring Cloud Bus             | 无        |
| 数据流       | Spring Cloud Stream          | 无        |
| 批量任务     | Spring Cloud Task            | 无        |

# 2.微服务设计原则

![](https://s1.ax1x.com/2020/04/28/J5mVgO.jpg)

### 2.1 AFK拆分原则

​		一般的，对于可扩展的系统架构设计，朴素的想法就是通过加机器可以解决容量和可用性问题，一台不行就两台甚至多台。

​		然后随着时间的推移，系统规模的增长，除了面对性能与容量的问题外，还需要面对功能与模块数量上增长带来的系统复杂性问题，以及业务变化带来的提供差异化服务问题。而许多系统在架构设计时并未充分考虑到这些问题，导致系统的重构成为常态。从而影响业务交付能力，还浪费人力财力。对此《The Art of Scalability》一书提出了一个更加系统的可扩展模型——AKF 可扩展立方。这个立方体中沿着三个坐标轴设置分别为 X，Y，Z。

![](https://s1.ax1x.com/2020/04/28/J5n2Sf.jpg)

- X 轴：指的是水平复制，很好理解，就是讲单体系统多运行几个实例，成为集群加负载均衡的模式。
- Z 轴：是基于类似的数据分区，比如一个互联网打车应用突然火了，用户量激增，集群模式撑不住了，那就按照用户请求的地区进行数据分区，北京、上海、四川等多建几个集群。
- Y 轴：就是我们所说的微服务的拆分模式，就是基于不同的业务拆分。

### 2.2 前后端分离

​		前后端分离原则，简单的将就是前端和后端的代码分离，最好采用物理分离的方式部署，进一步更彻底的分离。

​		这种前后端分离有几个好处：

- 前后端技术分离，可以由各自的专家来对各自的领域进行优化，这样前端的用户体验会更好。
- 分离模式下，前后端交互界面更清晰，就剩下接口模型，后端接口简介明了，更易于维护。
- 前端多渠道继承场景更容易实现，后端服务无需变更，采用统一的数据和模型，可以支持多个前端，例如：微信h5前端、PC前端、安卓前端、IOS前端。

### 2.3无状态服务

![](https://s1.ax1x.com/2020/04/28/J5u3jS.jpg)

​		首先，如果一个数据需要被多个服务共享，才能完成一笔交易，那么这个数据被称为状态。进而依赖这个状态的服务被称为有状态的服务，反之成为无状态服务。

​		这个无状态服务原则并不是说在微服务架构里不允许存在状态，表达的真实意思就是要把有状态的业务服务改变为无状态的计算类服务，那么状态数据也就相应的迁移到对应的“有状态数据服务”中。

​		比如如上图所示，从前在本地内存中建立的数据缓存、Session缓存，到现在微服务架构中就应该把数据迁移到分布式缓存中存储，让业务服务变成一个无状态的计算节点。

### 2.4 Rest通讯风格

​		基于**无状态通信原则**，在这里我们直接推荐一个实践优选的 Restful 通信风格 ，优点如下：

- 无状态协议 HTTP，具备先天优势，扩展能力很强。例如需要安全加密时，有现成的成熟方案 HTTPS 可用。
- JSON 报文序列化，轻量简单，人与机器均可读，学习成本低，搜索引擎友好。
- 语言无关，各大热门语言都提供成熟的 Restful API 框架，相对其他的一些 RPC 框架生态更完善。

# 3.Spring Cloud是什么

## 3.1 概念定义

​		Spring Cloud 是一个服务治理平台，提供了一些服务框架。包含了：服务注册与发现、配置中心、消息中心 、负载均衡、数据监控等等。

　　Spring Cloud 是一个微服务框架，相比 Dubbo 等 RPC 框架，Spring Cloud 提供了全套的分布式系统解决方案。

　　Spring Cloud 对微服务基础框架 Netflix 的多个开源组件进行了封装，同时又实现了和云端平台以及 Spring Boot 框架的集成。

　　Spring Cloud 是一个基于 Spring Boot 实现的云应用开发工具，它为开发中的配置管理、服务发现、断路器、智能路由、微代理、控制总线、全局锁、决策竞选、分布式会话和集群状态管理等操作提供了一种简单的开发方式。

## 3.2 Spring Cloud的发展

### 3.2.1 Spring Cloud Netflix

针对多种 Netflix 组件提供的开发工具包，其中包括 Eureka、Hystrix、Ribbon、Zuul、Archaius 等。

- `Netflix Eureka`：一个基于 Rest 服务的服务治理组件，包括服务注册中心、服务注册与服务发现机制的实现，实现了云端负载均衡和中间层服务器的故障转移。
- `Netflix Hystrix`：容错管理工具，实现断路器模式，通过控制服务的节点，从而对延迟和故障提供更强大的容错能力。
- `Netflix Ribbon`：客户端负载均衡的服务调用组件。
- `Netflix Feign`：基于 Ribbon 和 Hystrix 的声明式服务调用组件。
- `Netflix Zuul`：微服务网关，提供动态路由，访问过滤等服务。

### 3.2.2 Spring Cloud Alibaba

​		由于Netflix宣布对旗下多个组件不再维护，基于阿里的中间件构建的Spring Cloud Alibaba解决方案也逐渐流行起来。部分组件如下：

- `Nacos`：阿里巴巴开源产品，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。
- `Sentinel`：面向分布式服务架构的轻量级流量控制产品，把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
- `RocketMQ`：一款开源的分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠的消息发布与订阅服务。
- `Dubbo`：Apache Dubbo™ 是一款高性能 Java RPC 框架。
- `Seata`：阿里巴巴开源产品，一个易于使用的高性能微服务分布式事务解决方案。
- `Alibaba Cloud ACM`：一款在分布式架构环境中对应用配置进行集中管理和推送的应用配置中心产品。

### 3.3.3 对比

如果我们将基于Netflix的称为Spring Cloud一代，将后续改进的方案称为Spring Cloud二代，各组件变化对比如下：

|                | Spring Cloud 第一代       | Spring Cloud 第二代                               |
| :------------- | :------------------------ | :------------------------------------------------ |
| 网关           | Spring Cloud Zuul         | Spring Cloud Gateway                              |
| 注册中心       | Eureka，Consul，ZooKeeper | 阿里 Nacos，拍拍贷 Radar 等可选                   |
| 配置中心       | Spring Cloud Config       | 阿里 Nacos，携程 Apollo，随行付 Config Keeper     |
| 客户端负载均衡 | Ribbon                    | spring-cloud-commons 的 Spring Cloud LoadBalancer |
| 熔断器         | Hystrix                   | spring-cloud-r4j(Resilience4J)，阿里 Sentinel     |

## 3.3 常用组件

尽管 Netflix的Eureka，Hystrix 等不再继续开发或维护，但就目前来说，对大部分使用该套解决方案的公司不影响使用。

- `Spring Cloud Netflix Eureka`：服务注册中心。
- `Spring Cloud Netflix Ribbon`：客户端负载均衡。
- `Spring Cloud Netflix Hystrix`：服务容错保护。
- `Spring Cloud Netflix Feign`：声明式服务调用。
- `Spring Cloud OpenFeign(可替代 Feign)`：OpenFeign 是 Spring Cloud 在 Feign 的基础上支持了 Spring MVC 的注解，如 @RequesMapping等等。OpenFeign 的 @FeignClient 可以解析 SpringMVC 的 @RequestMapping 注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。
- `Spring Cloud Netflix Zuul`：API 网关服务，过滤、安全、监控、限流、路由。
- `Spring Cloud Gateway(可替代 Zuul)`：Spring Cloud Gateway 是 Spring 官方基于 Spring 5.0，Spring Boot 2.0 和 Project Reactor 等技术开发的网关，Spring Cloud Gateway 旨在为微服务架构提供一种简单而有效的统一的 API 路由管理方式。Spring Cloud Gateway 作为 Spring Cloud 生态系中的网关，目标是替代 Netflix Zuul，其不仅提供统一的路由方式，并且基于 Filter 链的方式提供了网关基本的功能，例如：安全，监控/埋点，和限流等。
- `Spring Cloud Config`：分布式配置中心。配置管理工具，支持使用 Git 存储配置内容，支持应用配置的外部化存储，支持客户端配置信息刷新、加解密配置内容等。
- `Spring Cloud Bus`：事件、消息总线，用于在集群（例如，配置变化事件）中传播状态变化，可与 Spring Cloud Config 联合实现热部署。
- `Spring Cloud Stream`：消息驱动微服务。
- `Spring Cloud Sleuth`：分布式服务跟踪。