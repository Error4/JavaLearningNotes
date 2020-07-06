> 本文相关代码已经上传至github,有兴趣的朋友可以自行查看：https://github.com/Error4/RabbitMQDemo

经过前面的讲解，我们已经对RabbitMQ有了初步的了解，做几个demo练习也没有问题，但在实际生产应用中，有一个问题是必须要我们考虑的：**在使用RabbitMQ时，如何保证消息最大程度的不丢失并且被正确消费？**

针对这个问题，RabbitMQ提供了以下三个机制来解决：

1. 生产者确认
2. 持久化
3. 手动Ack

# 1.生产者确认

**要想保证消息不丢失，首先我们得保证生产者能成功的将消息发送到RabbitMQ服务器。**

RabblitMQ针对这个问题，提供了两种解决方案：

1. 通过事务机制实现
2. 通过发送方确认（publisher confirm）机制实现

## 事务机制

RabblitMQ客户端中与事务机制相关的方法有以下3个：

1. channel.txSelect：用于将当前的信道设置成事务模式
2. channel.txCommit：用于提交事务
3. channel.txRollback：用于回滚事务

示例代码如下：

```java
public class TransactionProducer {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.txSelect();
        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        channel.txCommit();
        System.out.println(" [x] Sent '" + message + "'");
        channel.close();
        connection.close();
    }
}
```

然而，虽然事务机制能够解决消息发送方和RabbitMQ之间消息确认的问题，只有消息成功被RabbitMQ接收，事务才能提交成功，否则便可在捕获异常之后进行事务回滚。但是使用事务机制对RabbitMQ的性能较大，不推荐在生产中使用，建议使用下面所讲述的confirm机制。

## confirm机制

confirm机制是指生产者将信道设置成confirm（确认）模式，一旦信道进入confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID（从1开始），一旦消息被投递到RabbitMQ服务器之后，RabbitMQ就会发送一个确认（Basic.Ack）给生产者（包含消息的唯一ID），这就使得生产者知晓消息已经正确到达了目的地了。

如果RabbitMQ因为自身内部错误导致消息丢失，就会发送一条nack（Basic.Nack）命令，生产者应用程序同样可以在回调方法中处理该nack指令。

如果消息和队列是可持久化的，那么确认消息会在消息写入磁盘之后发出。

**事务机制在一条消息发送之后会使发送端阻塞**，以等待RabbitMQ的回应，之后才能继续发送下一条消息。相比之下，**发送方确认机制最大的好处在于它是异步的**，一旦发布一条消息。生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认后，生产者应用程序便可以通过回调方法来处理该确认消息。

**普通Confirm**

```java
public class ConfirmProducer {
    private final static String EXCHANGE_NAME = "confirm-exchange";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        try {
            channel.confirmSelect();
            // 发送消息
            String message = "confirm test";
            channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
            if (channel.waitForConfirms()) {
                System.out.println("send message success");
            } else {
                System.out.println("send message failed");
                // do something else...
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        channel.close();
        connection.close();
    }
}
```

其中，`channel.waitForConfirms()`等待发送消息的确认消息，如果发送成功，则返回ture，如果发送失败，则返回false。

注意：

**事务机制和publisher confirm机制是互斥的，不能共存。**

**事务机制和publisher confirm机制确保的是消息能够正确地发送至RabbitMQ，这里的“发送至RabbitMQ”的含义是指消息被正确地发往至RabbitMQ的交换器，如果此交换器没有匹配的队列，那么消息也会丢失。所以在使用这两种机制的时候要确保所涉及的交换器能够有匹配的队列。**

**异步confirm**

异步confirm模式是在生产者客户端添加ConfirmListener回调接口，重写接口的handAck()和handNack()方法，分别用来处理RabblitMQ回传的Basic.Ack和Basic.Nack。

这两个方法都有两个参数，第1个参数deliveryTag用来标记消息的唯一序列号，第2个参数multiple表示的是是否为多条确认,值为true代表是多个确认，值为false代表是单个确认。

```java
public class AsyncConfirmProducer {
    private final static String EXCHANGE_NAME = "async-confirm-exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        channel.confirmSelect();
        channel.addConfirmListener(new ConfirmListener() {
            @Override
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                System.out.println("Ack,SeqNo：" + deliveryTag + ",multiple：" + multiple);
            }

            @Override
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                System.out.println("Nack,SeqNo：" + deliveryTag + ",multiple：" + multiple);
            }
        });
        channel.basicPublish(EXCHANGE_NAME, "", null, "async confirm test".getBytes());
        channel.close();
        connection.close();
    }
}
```

# 2.持久化

所谓持久化，就是RabbitMQ会将内存中的数据(Exchange 交换器，Queue 队列，Message 消息)固化到磁盘，以防异常情况发生时，数据丢失。

## Exchange持久化

在之前的讲述中，我们都是这样初始化Exchange的

```
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
```

查看源码，会发现它调用的是如下方法

```java
 public Exchange.DeclareOk exchangeDeclare(String exchange, String type,
                                              boolean durable, boolean autoDelete,
                                              Map<String, Object> arguments)
```

其中，第三个参数`boolean durable`就是持久化的配置，可以利用如下重载方法

```
channel.exchangeDeclare(EXCHANGE_NAME, "direct", true);
```

## Queue持久化

与Exchange相同，Queue也有同样的durable参数

```
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                                 Map<String, Object> arguments) throws IOException;
```

设置为true即可。

## Message持久化

经过以上设置，Exchange和Queue都不丢失了，但是存储在Queue里的消息却仍然会丢失，那么如何保证消息不丢失呢？答案是设置消息的投递模式为2，即代表持久化。

```java
String message = "durable exchange test";
AMQP.BasicProperties props = new AMQP.BasicProperties().builder().deliveryMode(2).build();
channel.basicPublish(EXCHANGE_NAME, "", props, message.getBytes());
```

完整代码如下：

```java
public class DurableProducer {
    private final static String EXCHANGE_NAME = "durable-exchange";
    private final static String QUEUE_NAME = "durable-queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection()
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "direct", true);
        channel.queueDeclare(QUEUE_NAME, true, false, false, null);
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");
        // 发送消息
        String message = "durable exchange test";
        AMQP.BasicProperties props = new AMQP.BasicProperties().builder().deliveryMode(2).build();
        channel.basicPublish(EXCHANGE_NAME, "", props, message.getBytes());

        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

# 3.手动Ack

截止目前，我们能够保证消息成功地被生产者发送到RabbitMQ服务器，也能保证RabbitMQ服务器发生异常（重启，宕机等）后消息不会丢失，那么现在消息就一定安全吗？

考虑这么一种情况：**当消费者接收到消息后，还没处理完业务逻辑，消费者挂掉了，那消息是不是也丢失了？**

所以，为了保证消息被消费者成功的消费，RabbitMQ提供了**消息确认机制(message acknowledgement)**，避免因为消费者突然宕机而引起的消息丢失。

在之前的文章中，我们消费者的代码是这样的

```java
channel.basicConsume(QUEUE_NAME, true, consumer);
```

查看源码，可以看到方法如下：

```
   public String basicConsume(String queue, boolean autoAck, Consumer callback)
        throws IOException
    {
        return basicConsume(queue, autoAck, "", callback);
    }
```

注意第二个参数`autoAck`，**指的是是否自动确认，如果设置为ture，RabbitMQ会自动把发送出去的消息置为确认，然后从内存（或者磁盘）中删除，而不管消费者接收到消息是否处理成功；如果设置为false，RabbitMQ会等待消费者显式的回复确认信号后才会从内存（或者磁盘）中删除。**

所以，**建议将autoAck设置为false，这样消费者就有足够的时间处理消息，不用担心处理消息过程中消费者宕机造成消息丢失。**待消费完成后，再手动确认即可

代码示例如下：

```java
public class AckConsumer {
    private final static String QUEUE_NAME = "durable-queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        com.rabbitmq.client.Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                int result = 1 / 0;
                System.out.println("Received Message '" + message + "'");
                long deliveryTag = envelope.getDeliveryTag();
                channel.basicAck(deliveryTag, false);
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```







