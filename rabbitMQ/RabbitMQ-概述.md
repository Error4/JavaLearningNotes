消息队列（Message Queue），是分布式系统中重要的组件，其通用的使用场景可以简单地描述为：

当不需要立即获得结果，但是并发量又需要进行控制的时候，差不多就是需要使用消息队列的时候。

# 1.消息队列概述

我以一个生活中的例子举例：

小红希望小明多读书，常寻找好书给小明看，之前的方式是这样：小红问小明什么时候有空，把书给小明送去，并亲眼监督小明读完书才走。久而久之，两人都觉得麻烦。

后来的方式改成了：小红对小明说「我放到书架上的书你都要看」，然后小红每次发现不错的书都放到书架上，小明则看到书架上有书就拿下来看。

书架就是一个消息队列，小红是生产者，小明是消费者。

## 1.1 优点

**解耦**：每个成员不必受其他成员影响，可以更独立自主，只通过一个简单的容器来联系。

**异步**：小红只需要将书本放到书架上就可以去干别的事了，并不需要一直等待小明读书完毕。

**广播**：小红只需要劳动一次，就可以让多个小伙伴有书可读，这大大地节省了她的时间，也让新的小伙伴的加入成本很低。

**削峰**：假设小明读书很慢，如果采用小红每给一本书都监督小明读完的方式，小明有压力，小红也不耐烦。反正小红给书的频率也不稳定，如果今明两天连给了五本，之后隔三个月才又给一本，那小明只要在三个月内从书架上陆续取走五本书读完就行了，压力就不那么大了。

此外，为了避免消息队列服务器宕机造成消息丢失，会将成功发送到消息队列的消息存储在消息生产者服务器上，等消息真正被消费者服务器处理后才删除消息。在消息队列服务器宕机后，生产者服务器会选择分布式消息队列服务器集群中的其他服务器发布消息。

## 1.2 缺点

**引入复杂度**：毫无疑问，「书架」这东西是多出来的，需要地方放它，还需要防盗。

**暂时的不一致性**：这中间存在着一段「妈妈认为小明看了某书，而小明其实还没看」的时期，当然，小明最终的阅读状态与妈妈的认知会是一致的，这就是所谓的「最终一致性」。

## 1.3  使用条件

综上，使用消息队列需要满足以下条件：

1. 生产者不需要立刻从消费者得到反馈

2. 允许短暂的不一致性

3. 使用之后的确有效果。即解耦、提速、广播、削峰这些方面的收益，超过放置书架、监控书架这些成本。

# 2.JMS与AMQP

JMS，即Java消息服务(Java Message Service) 是一个Java平台中关于面向消息中间的API，用于在两个应用程序之间或者分布式系统中发送消息，进行异步通信。

**ActiveMQ 就是基于 JMS 规范实现的。**

 AMQP，即Advanced Message Queuing Protocol,是一个提供统一消息服务的应用层标准 **高级消息队列协议**，基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同开发语言等条件的限制。

**RabbitMQ 就是基于 AMQP 协议实现的。**

![](https://s1.ax1x.com/2020/06/22/NY2cxH.png)

# 3.常见消息队列对比

参考来源：https://yq.aliyun.com/articles/679880?utm_content=g_1000031822

| 对比方向 | 概要                                                         |
| :------- | :----------------------------------------------------------- |
| 吞吐量   | 万级的 ActiveMQ 和 RabbitMQ 的吞吐量（ActiveMQ 的性能最差）要比 十万级甚至是百万级的 RocketMQ 和 Kafka 低一个数量级。 |
| 可用性   | 都可以实现高可用。ActiveMQ 和 RabbitMQ 都是基于主从架构实现高可用性。RocketMQ 基于分布式架构。 kafka 也是分布式的，一个数据多个副本，少数机器宕机，不会丢失数据，不会导致不可用 |
| 时效性   | RabbitMQ 基于erlang开发，所以并发能力很强，性能极其好，延时很低，达到微秒级。其他三个都是 ms 级。 |
| 功能支持 | 除了 Kafka，其他三个功能都较为完备。 Kafka 功能较为简单，主要支持简单的MQ功能，在大数据领域的实时计算以及日志采集被大规模使用，是事实上的标准 |
| 消息丢失 | ActiveMQ 和 RabbitMQ 丢失的可能性非常低， RocketMQ 和 Kafka 理论上不会丢失。 |

**总结：**

- ActiveMQ 的社区算是比较成熟，但是较目前来说，ActiveMQ 的性能比较差，而且版本迭代很慢，不推荐使用。
- RabbitMQ 在吞吐量方面虽然稍逊于 Kafka 和 RocketMQ ，但是由于它基于 erlang 开发，所以并发能力很强，性能极其好，延时很低，达到微秒级。但是也因为 RabbitMQ 基于 erlang 开发，所以国内很少有公司有实力做erlang源码级别的研究和定制。如果业务场景对并发量要求不是太高（十万级、百万级），那这四种消息队列中，RabbitMQ 一定是你的首选。如果是大数据领域的实时计算、日志采集等场景，用 Kafka 是业内标准的，绝对没问题，社区活跃度很高，绝对不会黄，何况几乎是全世界这个领域的事实性规范。
- RocketMQ 阿里出品，Java 系开源项目，源代码我们可以直接阅读，然后可以定制自己公司的MQ，并且 RocketMQ 有阿里巴巴的实际业务场景的实战考验。RocketMQ 社区活跃度相对较为一般，不过也还可以，文档相对来说简单一些，然后接口这块不是按照标准 JMS 规范走的有些系统要迁移需要修改大量代码。还有就是阿里出台的技术，你得做好这个技术万一被抛弃，社区黄掉的风险，那如果你们公司有技术实力我觉得用RocketMQ 挺好的
- kafka 的特点其实很明显，就是仅仅提供较少的核心功能，但是提供超高的吞吐量，ms 级的延迟，极高的可用性以及可靠性，而且分布式可以任意扩展。同时 kafka 最好是支撑较少的 topic 数量即可，保证其超高吞吐量。kafka 唯一的一点劣势是有可能消息重复消费，那么对数据准确性会造成极其轻微的影响，在大数据领域中以及日志采集中，这点轻微影响可以忽略这个特性天然适合大数据实时计算以及日志收集。

# 4.RabbitMQ准备

由于RabbitMQ是基于Erlang语言开发的，所以在安装RabbitMQ之前还需要先行安装Erlang。官网地址如下：

Erlang：http://www.erlang.org/downloads

RabbitMQ：https://www.rabbitmq.com/download.html

根据实际需要下载对应的版本即可。以我实验用的window64版本为例，下载安装后，在RabbitMQ安装目录.../sbin\rabbitmq-plugins.bat

```
enable rabbitmq_management
```

开启管理后台。

说明：个人实验用的话，也可以直接用docker,详情可参考:https://www.cnblogs.com/yufeng218/p/9452621.html。

此时，RabbitMQ的安装就算完成了，其中有几个默认值，我们要知晓下：

- 默认的端口号：5672
- 默认的用户和密码是：guest guest
- 管理后台的默认端口号：15672

一个典型的RabbitMQ结构如下图所示：

![rebbitMQ](https://s1.ax1x.com/2020/06/22/NYRj6H.jpg)

**Broker：**标识消息队列服务器实体.

 **Virtual Host：**虚拟主机。标识一批交换机、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器、绑定和权限机制。vhost是AMQP概念的基础，必须在链接时指定，RabbitMQ默认的vhost是 /。

 **Exchange：**交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。

 **Queue：**消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

 **Binding：**绑定，用于消息队列和交换机之间的关联。一个绑定就是基于路由键将交换机和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。

 **Channel：**信道，多路复用连接中的一条独立的双向数据流通道。新到是建立在真实的TCP连接内地虚拟链接，AMQP命令都是通过新到发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说，建立和销毁TCP都是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。

 **Connection：**网络连接，比如一个TCP连接。

 **Publisher：**消息的生产者，也是一个向交换器发布消息的客户端应用程序。

 **Consumer：**消息的消费者，表示一个从一个消息队列中取得消息的客户端应用程序。

 **Message：**消息，消息是不具名的，它是由消息头和消息体组成。消息体是不透明的，而消息头则是由一系列的可选属性组成，这些属性包括routing-key(路由键)、priority(优先级)、delivery-mode(消息可能需要持久性存储[消息的路由模式])等。

# 5.HelloWorld快速入门案例

## 添加依赖

```xml
<dependency>
   <groupId>com.rabbitmq</groupId>
   <artifactId>amqp-client</artifactId>
   <version>5.7.0</version>
</dependency>
```

## 生产者

```java
public class Producer {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException, TimeoutException {
        // 创建连接
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 RabbitMQ 的主机名,默认localhost
        factory.setHost("192.168.124.20");
        //设置虚拟主机名称，默认为/
        factory.setVirtualHost("/");
        // 创建一个连接
        Connection connection = factory.newConnection();
        // 创建一个通道
        Channel channel = connection.createChannel();
        /**
         * 指定一个队列,不存在的话自动创建
         * 参数分别为队列名称，是否持久化，是否独占本次链接，是否在不使用时自动删除，队列其他参数
         */
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        String message = "Hello World!";
         /**
         * 发送消息
         * 参数分别为交换机名称，路由关键字，消息的基本属性，消息体
         */
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

浏览器输入`http://localhost:15672/`连接控制台，可以看到队列已被创建，且有一条信息待消费

​	![](https://s1.ax1x.com/2020/06/22/NYWVXj.md.png)





## 消费者

```java
public class Consumer {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException, TimeoutException {
        // 创建连接
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 RabbitMQ 的主机名
        factory.setHost("localhost");
        // 创建一个连接
        Connection connection = factory.newConnection();
        // 创建一个通道
        Channel channel = connection.createChannel();
        // 指定一个队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 创建队列消费者
        com.rabbitmq.client.Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println("Received Message '" + message + "'");
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

运行消费者，再次观察控制台，可以看到消息已经被消费了。

![](https://s1.ax1x.com/2020/06/22/NYWn7q.md.png)