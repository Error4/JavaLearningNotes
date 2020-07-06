# 存活时间

存活时间TTL（Time-To-Live）表示可以对消息设置预期的时间，在这个时间内都可以被消费者接受获取，过了这个时间后消息将被自动删除。RabbitMQ可以对消息和队列设置TTL。

通过队列属性设置TTL，队列中所有消息都有相同的存活时间，而对消息则可以进行单独设置。如果同时设置，则消息的存活时间以较小的为准。消息在队列的生存时间一旦超过设置的TTL值，就称为dead message， 消费者将无法再收到该消息。

## 设置队列属性

在queue.declare方法中加入x-message-ttl参数，单位为ms

```java
Map<String, Object>  argss = new HashMap<String, Object>();
argss.put("x-message-ttl",6000);
channel.queueDeclare(queueName, durable, exclusive, autoDelete, argss);
```

## 设置消息属性

针对每条消息设置TTL的方法是在basic.publish方法中加入expiration的属性参数，单位为ms.

```java
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.expiration("6000");
AMQP.BasicProperties  properties = builder.build();        channel.basicPublish(exchangeName,routingKey,mandatory,properties,"ttlTestMessage".getBytes());
```

# 死信队列

DLX, Dead-Letter-Exchange。利用DLX, 当消息在一个队列中变成死信（dead message）之后，它能被重新publish到另一个Exchange，这个Exchange就是DLX。

DLX也是一个正常的Exchange，和一般的Exchange没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性，当这个队列中有死信时，RabbitMQ就会自动的将这个消息重新发布到设置的Exchange上去，进而被路由到另一个队列。

消息变成死信有以下几种情况：

- 消息被拒绝(basic.reject / basic.nack)，并且requeue = false
- 消息TTL过期
- 队列达到最大长度

通过在queueDeclare方法中加入“x-dead-letter-exchange”实现。

```java
channel.exchangeDeclare("some.exchange.name", "direct");
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-dead-letter-exchange", "some.exchange.name");
channel.queueDeclare("myqueue", false, false, false, args);
```

你也可以为这个DLX指定routing key，如果没有特殊指定，则使用原队列的routing key

```java
args.put("x-dead-letter-routing-key", "some-routing-key");
```

在实际生产环境中，我们还需要对死信队列进行一个监听和处理，当然具体的处理逻辑和业务相关。

# 延时队列

简单来说，延时队列就是用来存放需要在指定时间被处理的元素的队列。

利用我们上文提到的TTL+死信队列就能实现延时队列的功能.

生产者生产一条延时消息，根据需要延时时间的不同，利用不同的routingkey将消息路由到不同的延时队列，每个队列都设置了不同的TTL属性，并绑定在同一个死信交换机中，消息过期后，根据routingkey的不同，又会被路由到不同的死信队列中，消费者只需要监听对应的死信队列进行处理即可。

![](https://s1.ax1x.com/2020/07/06/UiqCfH.md.png)

```java
public class TimeoutDLX {
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection connection = ConnectionUtil.getConnection();
        Channel channel = connection.createChannel();
        //创建DLX及死信队列
        channel.exchangeDeclare("dlx.exchange", "direct");
        channel.queueDeclare("dlx.queue", true, false, false, null);
        channel.queueBind("dlx.queue", "dlx.exchange", "dlx.routingKey");
        //创建测试超时的Exchange及Queue
        channel.exchangeDeclare("delay.exchange", "direct");
        Map<String, Object> arguments = new HashMap<>();
        //过期时间10s
        arguments.put("x-message-ttl", 10000);
        //绑定DLX
        arguments.put("x-dead-letter-exchange", "dlx.exchange");
        //绑定发送到DLX的RoutingKey
        arguments.put("x-dead-letter-routing-key", "dlx.routingKey");
        channel.queueDeclare("delay.queue", true, false, false, arguments);
        channel.queueBind("delay.queue", "delay.exchange", "delay.routingKey");
        //发布一条消息
        channel.basicPublish("delay.exchange", "delay.routingKey", null, "该消息将在10s后发送到延迟队列".getBytes());
        channel.close();
        connection.close();
    }
}
```

通过控制台可以看到队列信息变化：

![](https://s1.ax1x.com/2020/07/06/UiqVnP.md.png)

延时10s后：

![](https://s1.ax1x.com/2020/07/06/UiqnAS.md.png)