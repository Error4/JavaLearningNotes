前面的文章中，我们通过一个入门HelloWorld案例直观感受了下RabbitMQ，创建了一个消息生产者和消息消费者，这种模式可以称之为`简单模式`，除此之外，RabbitMQ还有其他的工作模式，下面我会分别介绍。

# 1.Work Queues工作队列模式

## 1.1 说明

![1589005150325](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589005150325.png)

与简单模式不同，Work Queues模式存在多个消费者，对于人物过多的情况就可以使用这种模式，提高处理速度。

## 1.2 代码示例

生产者，与之前的HelloWorld的示例基本相同，为了演示所用，我们循环创建10条消息，另外，我们将创建Connection的代码提取出来，构建工具类ConnectionUtil

```java
public class ConnectionUtil {

    public static  Connection getConnection() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        // 设置 RabbitMQ 的主机名
        factory.setHost("192.168.124.20");
        // 创建一个连接
        Connection connection = factory.newConnection();
        return  connection;
    }
}
```



```java
private final static String QUEUE_NAME = "work_queue";
public static void main(String[] args) throws IOException, TimeoutException {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();
    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    for (int i = 0; i < 10; i++) {
        String message = "Hello World!"+i;
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
    }
    channel.close();
    connection.close();
}
```

消费者代码不变，并且额外复制一份，有两个消费者。

```java
private final static String QUEUE_NAME = "work_queue";
public static void main(String[] args) throws IOException, TimeoutException {
    Connection connection = ConnectionUtil.getConnection();
    Channel channel = connection.createChannel();
    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
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
```

此时，启动代码可以发现，收到的消息如下，两个消费者是**轮询**去消费消息的，

```
Received Message 'Hello World!1'
Received Message 'Hello World!3'
Received Message 'Hello World!5'
Received Message 'Hello World!7'
Received Message 'Hello World!9'
```

```
Received Message 'Hello World!0'
Received Message 'Hello World!2'
Received Message 'Hello World!4'
Received Message 'Hello World!6'
Received Message 'Hello World!8'
```

# 2.Publish/Subscribe 发布与订阅模式

## 2.1说明

![1589007359831](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589007359831.png)

在前面所说的模式中，我们的案例只包含3个角色：生产者，消费者和消息队列。而在发布订阅模式中，多了一个Exchange，即交换机的角色，如上图的标记X图标。（实际上之前的模式也是存在Exchange的，只不过没有显示声明，绑定到了默认的Exchange）

交换机，一方面接受生产者发送的消息，另一方面，也要处理消息，而这个处理的规则，取决于Exchange的类型。根据分发策略的不同，目前共四种类型：direct、fanout、topic、headers(headers匹配AMQP消息的header而不是路由键(Routing-key)，常用的是前三种，发布订阅模式就可以试做fanout模式。

- **direct**

**消息中的路由键(routing key)如果和Binding中的binding key一致**，交换器就将消息发到对应的队列中。路由键与队列名完全匹配。

![rabitMQ-direct](D:\Program Files\笔记\image\rabitMQ-direct.webp)

- **fanout**

**每个发到fanout类型交换器的消息都会分到所有绑定的队列上去**。fanout交换器不处理该路由键，只是简单的将队列绑定到交换器上，每个发送到交换器的消息都会被转发到与该交换器绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。fanout类型转发消息是最快的。

![rabitMQ-fanout](D:\Program Files\笔记\image\rabitMQ-fanout.webp)

- **topic**

**topic交换器通过模式匹配分配消息的路由键属性**，**将路由键和某个模式进行匹配**，此时队列需要绑定到一个模式上。它将路由键(routing-key)和绑定键(bingding-key)的字符串切分成单词，这些**单词之间用点隔开**。它同样也会识别两个通配符：`#`和`*`。`#`匹配0个或多个单词，`*`匹配不多不少一个单词。

![rabitMQ-topic](D:\Program Files\笔记\image\rabitMQ-topic.webp)

**注意：Exchange只负责转发消息，不具备存储消息的能力，因此如果没有任何队列与Exchange绑定，消息就会丢失。**

## 2.2 代码示例·

生产者

```java
public class Producer {
    private final static String QUEUE_NAME = "fanout_queue";
    private final static String EXCHANGE_NAME = "fanout_exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();;
        // 创建一个通道
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");
        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

控制台如图所示，Exchange创建成功。可以看到，主要是生产者的代码有变化，定义了Exchange并将其与Queue绑定

![1589009363486](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589009363486.png)

消费者

```java
public class Consumer {
    private final static String QUEUE_NAME = "fanout_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
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

# 3.Routing路由模式

## 3.1 说明

在路由模式中，队列Queue与交换机Exchange，不再是任意绑定了，而是需要制定一个routingKey（路由键），消息的发送方向Exchange发送消息时，必须指定routingKey，而Exchange也会根据routingKey判断，只会把消息发给routingKey相匹配的队列。

![1589009981208](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589009981208.png)







## 3.2 代码示例

还是生产者代码变化

```java
public class Producer {
    private final static String QUEUE_NAME = "direct_queue";
    private final static String EXCHANGE_NAME = "direct_exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();;
        // 创建一个通道
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"test");
        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, "test", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

在绑定队列与发送消息时都制定了routingKey为“test”

# 4.Topic通配符模式

## 4.1说明

相比其他模式，Topic可以更灵活的匹配自己想要订阅的信息，可以再制定通配符。其中，`#`匹配0个或多个单词，`*`匹配不多不少一个单词。



![1589010769158](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589010769158.png)



## 4.2 代码示例

```java
public class Producer {
    private final static String QUEUE_NAME = "topic_queue";
    private final static String EXCHANGE_NAME = "topic_exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();;
        // 创建一个通道
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.FANOUT);
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        String routingKey = "item.*";
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,routingKey);
        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, "item.test", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

可以看到，绑定时的routingKey使用了通配符“item.*”，消息发送时routingKey为“item.test”

