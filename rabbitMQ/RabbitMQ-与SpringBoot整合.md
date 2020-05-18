在SpringBoot项目中，对RabbitMQ的操作进一步封装，只需引入对应的amqp启动器依赖，就能方便的使用RabbitTemplate发送消息。

# 1.基于Direct模式的使用

## 1.1 准备工作

引入依赖pom.xml

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

配置文件applicaition.yml

```properties
spring:
  rabbitmq:
    host: 192.168.124.20
    port: 5672
    username: wyf
    password: 123456
    #虚拟host 也可以不设置,使用默认即可
    #virtual-host:myHost
```

## 1.2 消息生产者

我们以direct模式为例

```java
@Configuration
public class Producer {
    @Bean
    public Queue TestDirectQueue() {
        return new Queue("TestDirectQueue",true);
    }

    //Direct交换机 起名：TestDirectExchange
    @Bean
    DirectExchange TestDirectExchange() {
        return new DirectExchange("TestDirectExchange",true,false);
    }

    //绑定  将队列和交换机绑定, 并设置路由键：TestDirectRouting
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(TestDirectQueue()).to(TestDirectExchange()).with("TestDirectRouting");
    }
}
```

测试发送消息

```java
@Autowired
RabbitTemplate rabbitTemplate;
@Test
void send() {
    String context = "OneByOneSender";
    System.out.println("DirectProducer发送消息 : " + context);
    rabbitTemplate.convertAndSend("TestDirectExchange", "TestDirectRouting", context);
}
```

## 1.3 消息消费者

利用监听，监听到对应queue收到消息就会执行方法，通过 `@RabbitListener` 注解定义该类对”hello-world” 队列的监听，并用 `@RabbitHandler` 注解来指定对消息的处理方法

```java
@Component
@RabbitListener(queues = "TestDirectQueue")
public class Consumer {
    @RabbitHandler
    public void process(String context) {
        System.out.println("DirectConsumer消费消息  : " + context);
    }
}
```

启动程序，可以看到输出如下

```
DirectProducer发送消息 : OneByOneSender

DirectConsumer消费消息  : OneByOneSender
```

Fanout与Topic类似，只需要创建对应类型的Exchange即可。

```java
@Bean
FanoutExchange fanoutExchange(){
    return new FanoutExchange("fanoutExchange");
}
@Bean
TopicExchange topicExchange(){
    return new TopicExchange("topicExchange");
}
```

# 2.消息回调

消息的回调，其实就是消息确认（生产者推送消息成功，消费者接收消息成功）。

## 2.1 生产者推送消息

首先还是在配置文件中添加消息确认的配置项

```yaml
spring:
  rabbitmq:
    host: 192.168.124.20
    port: 5672
    #确认消息已发送到交换机(Exchange)
    publisher-confirms: true
    #确认消息已发送到队列(Queue)
    publisher-returns: true
```

然后配置相关消息回调函数

```java
@Configuration
public class RabbitConfig {
    @Bean
    public RabbitTemplate createRabbitTemplate(ConnectionFactory connectionFactory){
        RabbitTemplate rabbitTemplate = new RabbitTemplate();
        rabbitTemplate.setConnectionFactory(connectionFactory);
        //设置开启Mandatory,才能触发回调函数,无论消息推送结果怎么样都强制调用回调函数
        rabbitTemplate.setMandatory(true);

        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override  
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                System.out.println("ConfirmCallback:     "+"相关数据："+correlationData);
                System.out.println("ConfirmCallback:     "+"确认情况："+ack);
                System.out.println("ConfirmCallback:     "+"原因："+cause);
            }
        });

        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                System.out.println("ReturnCallback:     "+"消息："+message);
                System.out.println("ReturnCallback:     "+"回应码："+replyCode);
                System.out.println("ReturnCallback:     "+"回应信息："+replyText);
                System.out.println("ReturnCallback:     "+"交换机："+exchange);
                System.out.println("ReturnCallback:     "+"路由键："+routingKey);
            }
        });

        return rabbitTemplate;
    }
}
```

推送消息存在四种情况：

1. 消息推送到server，但是在server里找不到交换机
2. 消息推送到server，找到交换机了，但是没找到队列
3. 消息推送到sever，交换机和队列啥都没找到
4. 消息推送成功

我们可以分别验证下：

- 消息推送到server，但是在server里找不到交换机

  ```java
   @Test
      void send() {
          String context = "OneByOneSender";
          System.out.println("DirectProducer发送消息 : " + context);
          rabbitTemplate.convertAndSend("non-existent-exchange", "TestDirectRouting", context);
      }
  ```

  如代码所示，我们把消息发送到名为`non-existent-exchange`的Exchange上，但实际我们并没有配置

  启动程序，可以看到控制台输出如下：

  ```
  ConfirmCallback:     相关数据：null
  ConfirmCallback:     确认情况：false
  ConfirmCallback:     原因：channel error; protocol method: #method<channel.close>(reply-code=404, reply-text=NOT_FOUND - no exchange 'non-existent-exchange' in vhost '/', class-id=60, method-id=40)
  ```

  所以，**这种情况触发的是 ConfirmCallback 回调函数。**

- 消息推送到server，找到交换机了，但是没找到队列

  我们新配置一个单独的Exchange，但是没有队列与它绑定

  ```java
   @Bean
      DirectExchange lonelyDirectExchange() {
          return new DirectExchange("lonelyExchange");
      }
  ```

  测试发送

  ```java
  ReturnCallback:     消息：(Body:'OneByOneSender' MessageProperties [headers={}, contentType=text/plain, contentEncoding=UTF-8, contentLength=0, receivedDeliveryMode=PERSISTENT, priority=0, deliveryTag=0])
  ReturnCallback:     回应码：312
  ReturnCallback:     回应信息：NO_ROUTE
  ReturnCallback:     交换机：lonelyExchange
  ReturnCallback:     路由键：TestDirectRouting
  
  ConfirmCallback:     相关数据：null
  ConfirmCallback:     确认情况：true
  ConfirmCallback:     原因：null
  
  ```

  可以到看，**ConfirmCallback和RetrunCallback两个回调函数都被调用了**

- 消息推送到sever，交换机和队列啥都没找到

  和第一种情况一致，**触发的是 ConfirmCallback 回调函数。**

- 消息推送成功

  按照之前的程序正常发送即可，可以看到，调用的是**ConfirmCallback 回调函数**

  ```
  ConfirmCallback:     相关数据：null
  ConfirmCallback:     确认情况：true
  ConfirmCallback:     原因：null
  ```

## 2.2 消费者接收到消息的消息确认

在之前的文章中我们提到，为了保证消息消费的可靠性，我们一般都使用消费者手动确认的方式，那么结合SpringBoot如何操作呢？

新建配置类

```java
@Configuration
public class MessageListenerConfig {
    @Autowired
    private CachingConnectionFactory connectionFactory;
    @Autowired
    private MyAckReceiver myAckReceiver;//自定义消息接收处理类

    @Bean
    public SimpleMessageListenerContainer simpleMessageListenerContainer() {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setConcurrentConsumers(1);
        container.setMaxConcurrentConsumers(1);
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL); // RabbitMQ默认是自动确认，这里改为手动确认消息
        //设置一个队列
        container.setQueueNames("TestDirectQueue");
        //如果同时设置多个如下： 前提是队列都是必须已经创建存在的
        //  container.setQueueNames("TestDirectQueue","TestDirectQueue2","TestDirectQueue3");
        //另一种设置队列的方法,如果使用这种情况,那么要设置多个,就使用addQueues
        //container.setQueues(new Queue("TestDirectQueue",true));
        //container.addQueues(new Queue("TestDirectQueue2",true));
        //container.addQueues(new Queue("TestDirectQueue3",true));
        container.setMessageListener(myAckReceiver);
        return container;
    }
}
```

手动确认消息监听类

```java
@Component
public class MyAckReceiver implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            String msg = message.toString();
            System.out.println(msg);//输出message信息
            System.out.println("消息成功消费，消费的主题消息来自："+message.getMessageProperties().getConsumerQueue());
            channel.basicAck(deliveryTag, true);
//			channel.basicReject(deliveryTag, true);//为true会重新放回队列
        } catch (Exception e) {
            channel.basicReject(deliveryTag, false);
            e.printStackTrace();
        }
    }
}
```

启动测试类

```java
    @Test
    void send() {
        String context = "OneByOneSender";
        System.out.println("DirectProducer发送消息 : " + context);
        rabbitTemplate.convertAndSend("TestDirectExchange", "TestDirectRouting", context);
    }
```

可以看到控制台输出如下

```
(Body:'OneByOneSender' MessageProperties [headers={spring_listener_return_correlation=7b8e034a-4389-4ca6-aa54-72dc8d6eb935}, contentType=text/plain, contentEncoding=UTF-8, contentLength=0, receivedDeliveryMode=PERSISTENT, priority=0, redelivered=false, receivedExchange=TestDirectExchange, receivedRoutingKey=TestDirectRouting, deliveryTag=1, consumerTag=amq.ctag-d-2ieMAED2LwH3JBV6VBVw, consumerQueue=TestDirectQueue])

消息成功消费，消费的主题消息来自：TestDirectQueue
```

