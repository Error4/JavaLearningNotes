# 一.SpringBoot与缓存

### 1.重要概念与缓存注解

| Cache          | 缓存接口，定义缓存操作，实现有RdisCache,EhCacheCache等       |
| -------------- | ------------------------------------------------------------ |
| CacheManager   | 缓存管理器，管理各种缓存组件                                 |
| @Cacheable     | 主要针对方法配置，能根据方法的请求参数对其结果进行缓存。在**方法运行之前**就会查询缓存 |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 保证方法被调用，又希望结果被缓存，即可以视为更新数据库与缓存。在**方法运行之后**更新缓存 |
| @EnableCaching | 开启基于注解的缓存                                           |
| keyGenerator   | 缓存数据时Key生成策略                                        |
| serialize      | 缓存数据时Value序列化策略                                    |
| @Caching       | @Cacheable,@CacheEvict,@CachePut三合一                       |
| @CacheConfig   | 可定义在类上，抽取缓存的公共配置，比如cacheNames等属性       |

```
public @interface Caching {
	Cacheable[] cacheable() default {};
	CachePut[] put() default {};
	CacheEvict[] evict() default {};
}
```



### 2.使用

```java
//主程序上开启缓存
@SpringBootApplication
@EnableCaching
public class DataApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataApplication.class, args);
    }
}
```

在对应方法上使用注解

@Cacheable

```java
	/**
	CacheManager会管理多个Cache组件，每个组件有一个唯一的名字
	注解@Cacheable主要属性有：
	cacheNames：制定缓存组件的名字
	key：缓存数据使用的key，默认是方法参数
	keyGenerator:制定key的生成器，与key的制定二选一
	condition：符合条件的情况下缓存
	unless：与condition相反，且可以获取到结果(#result)进行缓存 
	e.g: unless="#result>0"
	sync:缓存是否使用异步模式，使用异步则不支持unless
	**/
	@Cacheable
    public int getInt(Integer id){
        return id;
    }
```

@CacheEvict，有两个特殊的属性

```properties
allEntries:是否删除所有缓存
beforeInvocation:是否在方法执行之前清除缓存，默认为false,在方法执行之后执行
```



### 3.工作原理

从自动配置类CacheAutoConfiguration入手

核心：

- 从CacheManager按照名字得到Cache组件，默认由ConcurrentMapCacheManager创建ConcurrentMapCache类型组件
- key使用KeyGenarator生成，默认是SimpleKeyGenarator.
- 利用key去命中缓存信息

### 4.Redis

#### 1.Springboot使用

```java
    //操作k-v都是字符串的
    @Autowired
    StringRedisTemplate stringRedisTemplate;
    //操作k-v都是对象的
    @Autowired
    RedisTemplate redisTemplate;
    @Test
    void testRedis(){
        //一般做一些复杂的计数功能的缓存。
        stringRedisTemplate.opsForValue();
        //使用List的数据结构，可以做简单的消息队列的功能。另外还有一个就是，可以利用lrange命令，做基于redis的分页功能
        stringRedisTemplate.opsForList();
        //全局去重的功能，另外，就是利用交集、并集、差集等操作，可以计算共同喜好，全部的喜好，自己独有的喜好等功能。
        stringRedisTemplate.opsForSet();
        //value存放的是结构化的对象，比较方便的就是操作其中的某个字段
        stringRedisTemplate.opsForHash();
        //多了一个权重参数score,集合中的元素能够按score进行排列。可以做排行榜应用，取TOP N操作。sorted set可以用来做延时任务。最后一个应用就是可以做范围查找。
        stringRedisTemplate.opsForZSet();
    }

```

#### 2.缓存雪崩与缓存穿透

**缓存雪崩**，即缓存同一时间大面积的失效，这个时候又来了一波请求，结果请求都怼到数据库上，从而导致数据库连接异常。

**解决方案：**

在批量往Redis存数据的时候，把每个Key的失效时间都加个随机值就好了，这样可以保证数据不会在同一时间大面积失效

```java
setRedis（Key，value，time + Math.random() * 10000）；
```

**缓存穿透**，即黑客故意去请求缓存中不存在的数据，导致所有的请求都怼到数据库上，从而数据库连接异常。

**解决方案：**

(一)利用互斥锁，缓存失效的时候，先去获得锁，得到锁了，再去请求数据库。没得到锁，则休眠一段时间重试
(二)采用异步更新策略，无论key是否取到值，都直接返回。value值中维护一个缓存失效时间，缓存如果过期，异步起一个线程去读数据库，更新缓存。需要做**缓存预热**(项目启动前，先加载缓存)操作。
(三)提供一个能迅速判断请求是否有效的拦截机制，比如，利用布隆过滤器，内部维护一系列合法有效的key。迅速判断出，请求所携带的Key是否合法有效。如果不合法，则直接返回。

**布隆过滤器**：布隆过滤器的原理是，当一个元素被加入集合时，通过K个散列函数将这个元素映射成一个位数组中的K个点，把它们置为1。检索时，我们只要看看这些点是不是都是1就（大约）知道集合中有没有它了：如果这些点有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能在。

优点是时间和空间上的效率比较高，缺点存在误判，可能要查到的元素并没有在容器中，但是hash之后得到的k个位置上值都是1，删除困难一个放入容器的元素映射到bit数组的k个位置上是1，删除的时候不能简单的直接置为0，可能会影响其他元素的判断。

#### 3.Rdis快速的原因

**Redis**采用的是基于内存的采用的是单进程单线程模型的 KV 数据库，由C语言编写，官方提供的数据是可以达到100000+的**QPS（每秒内查询次数）**。

- **完全基于内存**，绝大部分请求是纯粹的内存操作，非常快速。
- 数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；
- **采用单线程**，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 **CPU**，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；
- **使用多路I/O复用模型**，非阻塞IO；
- 使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，**Redis**直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

#### **4.redis持久化**

有两种方式的。

- RDB：**RDB** 持久化机制，是对 **Redis** 中的数据执行**周期性**的持久化。
- AOF：**AOF** 机制对每条写入命令作为日志，以 **append-only** 的模式写入一个日志文件中，因为这个模式是只追加的方式，所以没有任何磁盘寻址的开销，所以很快，有点像Mysql中的**binlog**。

两种方式都可以把**Redis**内存中的数据持久化到磁盘上，然后再将这些数据备份到别的地方去，**RDB**更适合做**冷备**，**AOF**更适合做**热备**。**两种机制全部开启的时候，Redis在重启的时候会默认使用AOF去重新构建数据**

优缺点：

#### RDB优点：

他会生成多个数据文件，每个数据文件分别都代表了某一时刻**Redis**里面的数据，这种方式，有没有觉得很适合做**冷备**，完整的数据运维设置定时任务，定时同步到远端的服务器，比如阿里的云服务，这样一旦线上挂了，你想恢复多少分钟之前的数据，就去远端拷贝一份之前的数据就好了。

**RDB**对**Redis**的性能影响非常小，是因为在同步数据的时候他只是**fork**了一个子进程去做持久化的，而且他在数据恢复的时候速度比**AOF**来的快。

#### RDB缺点：

**RDB**都是快照文件，都是默认五分钟甚至更久的时间才会生成一次，这意味着你这次同步到下次同步这中间五分钟的数据都很可能全部丢失掉。**AOF**则最多丢一秒的数据，**数据完整性**上高下立判。

还有就是**RDB**在生成数据快照的时候，如果文件很大，客户端可能会暂停几毫秒甚至几秒，你公司在做秒杀的时候他刚好在这个时候**fork**了一个子进程去生成一个大快照，哦豁，出大问题。

#### AOF优点：

上面提到了，**RDB**五分钟一次生成快照，但是**AOF**是一秒一次去通过一个后台的线程`fsync`操作，那最多丢这一秒的数据。

**AOF**在对日志文件进行操作的时候是以`append-only`的方式去写的，他只是追加的方式写数据，自然就少了很多磁盘寻址的开销了，写入性能惊人，文件也不容易破损。

#### AOF缺点：

一样的数据，**AOF**文件比**RDB**还要大。

#### 5.redis过期策略

**Redis**的过期策略，是有**定期删除+惰性删除**两种。

定期好理解，默认100s就随机抽一些设置了过期时间的key，去检查是否过期，过期了就删了。**如果只有定时策略，虽然内存及时释放，但是十分消耗CPU资源**，且由于是随机删除，会导致很多key到时间没有删除。

惰性删除，我不主动删，我懒，我等你来查询了我看看你过期没，过期就删了还不给你返回，没过期该怎么样就怎么样。

都不行最后还有**内存淘汰机制**！在redis.conf中有一行配置

```cml
# maxmemory-policy volatile-lru
```

1）noeviction：当内存不足以容纳新写入数据时，新写入操作会报错。**应该没人用吧。**
2）allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。**推荐使用，目前项目在用这种。**
3）allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key。**应该也没人用吧，你不删最少使用Key,去随机删。**
4）volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key。**这种情况一般是把redis既当缓存，又做持久化存储的时候才用。不推荐**
5）volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key。**依然不推荐**
6）volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除。**不推荐**

### 5.Redis经典问题

**假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如何将它们全部找出来？**

使用**keys**指令可以扫出指定模式的key列表。

**如果这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？**

redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用**scan**指令，**scan**指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。

**使用过Redis做异步队列么，你是怎么用的？**

一般使用list结构作为队列，**rpush**生产消息，**lpop**消费消息。当lpop没有消息的时候，要适当sleep一会再重试。

**如果对方追问可不可以不用sleep呢？**

list还有个指令叫**blpop**，在没有消息的时候，它会阻塞住直到消息到来。

**如果对方接着追问能不能生产一次消费多次呢？**

使用pub/sub主题订阅者模式，可以实现 1:N 的消息队列。

**如果对方继续追问 pub/su b有什么缺点？**

在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如**RocketMQ**等。



# 二.SpringBoot与消息



消息队列作用：**异步处理，应用解耦，流量削峰**，缺点就是**系统可用性降低**，**系统复杂性增加**

## 1.RabbitMQ

### 1.工作模式：

![rebbitMQ](D:\Program Files\笔记\image\rebbitMQ.webp)

**Broker：**标识消息队列服务器实体.

 **Virtual Host：**虚拟主机。标识一批交换机、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器、绑定和权限机制。vhost是AMQP概念的基础，必须在链接时指定，RabbitMQ默认的vhost是 /。

 **Exchange：**交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。

 **Queue：**消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

 **Banding：**绑定，用于消息队列和交换机之间的关联。一个绑定就是基于路由键将交换机和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。

 **Channel：**信道，多路复用连接中的一条独立的双向数据流通道。新到是建立在真实的TCP连接内地虚拟链接，AMQP命令都是通过新到发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说，建立和销毁TCP都是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。

 **Connection：**网络连接，比如一个TCP连接。

 **Publisher：**消息的生产者，也是一个向交换器发布消息的客户端应用程序。

 **Consumer：**消息的消费者，表示一个从一个消息队列中取得消息的客户端应用程序。

 **Message：**消息，消息是不具名的，它是由消息头和消息体组成。消息体是不透明的，而消息头则是由一系列的可选属性组成，这些属性包括routing-key(路由键)、priority(优先级)、delivery-mode(消息可能需要持久性存储[消息的路由模式])等。

### 2.Exchange分发类型

Exchange分发消息时，根据类型的不同分发策略有区别。目前共四种类型：direct、fanout、topic、headers(headers匹配AMQP消息的header而不是路由键(Routing-key)

#### direct

**消息中的路由键(routing key)如果和Binding中的binding key一致**，交换器就将消息发到对应的队列中。路由键与队列名完全匹配。

![rabitMQ-direct](D:\Program Files\笔记\image\rabitMQ-direct.webp)

#### fanout

**每个发到fanout类型交换器的消息都会分到所有绑定的队列上去**。fanout交换器不处理该路由键，只是简单的将队列绑定到交换器上，每个发送到交换器的消息都会被转发到与该交换器绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。fanout类型转发消息是最快的。

![rabitMQ-fanout](D:\Program Files\笔记\image\rabitMQ-fanout.webp)

#### topic

**topic交换器通过模式匹配分配消息的路由键属性**，**将路由键和某个模式进行匹配**，此时队列需要绑定到一个模式上。它将路由键(routing-key)和绑定键(bingding-key)的字符串切分成单词，这些**单词之间用点隔开**。它同样也会识别两个通配符："#"和"*"。#匹配0个或多个单词，*匹配不多不少一个单词。

![rabitMQ-topic](D:\Program Files\笔记\image\rabitMQ-topic.webp)

### 3.如何保证消息不被重复消费？

原因:。正常情况下，消费者在消费消息时候，消费完毕后，会发送一个确认信息给消息队列，消息队列就知道该消息被消费了，就会将该消息从消息队列中删除。但可能因为网络传输等等故障，确认信息没有传送到消息队列，导致消息队列不知道自己已经消费过该消息了，再次将该消息分发给其他的消费者。

解决方案：

​		(1)比如，你拿到这个消息做数据库的insert操作。那就容易了，给这个消息做一个唯一主键，那么就算出现重复消费的情况，就会导致主键冲突，避免数据库出现脏数据。
  (2)再比如，你拿到这个消息做redis的set的操作，那就容易了，不用解决，因为你无论set几次结果都是一样的，set操作本来就算幂等操作。
  (3)如果上面两种情况还不行，上大招。准备一个第三方介质,来做消费记录。以redis为例，给消息分配一个全局id，只要消费过该消息，将<id,message>以K-V形式写入redis。那消费者开始消费前，先去redis中查询有没消费记录即可。

### 4.如何保证消费的可靠性传输?

**(1)生产者丢数据**

RabbitMQ提供transaction和confirm模式来确保生产者不丢消息。

transaction机制就是说，发送消息前，开启事物(channel.txSelect())，然后发送消息，如果发送过程中出现什么异常，事物就会回滚(channel.txRollback())，如果发送成功则提交事物(channel.txCommit())。然而缺点就是吞吐量下降了

一旦channel进入confirm模式，所有在该信道上面发布的消息都将会被指派一个唯一的ID(从1开始)，一旦消息被投递到所有匹配的队列之后，rabbitMQ就会发送一个Ack给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了.如果rabiitMQ没能处理该消息，则会发送一个Nack消息给你，你可以进行重试操作

```java
channel.addConfirmListener(new ConfirmListener() {  
                @Override  
                public void handleNack(long deliveryTag, boolean multiple) throws IOException {  
                    System.out.println("nack: deliveryTag = "+deliveryTag+" multiple: "+multiple);  
                }  
                @Override  
                public void handleAck(long deliveryTag, boolean multiple) throws IOException {  
                    System.out.println("ack: deliveryTag = "+deliveryTag+" multiple: "+multiple);  
                }  
            });  
```

**(2)消息队列丢数据**

处理消息队列丢数据的情况，一般是开启持久化磁盘的配置

1、将queue的持久化标识durable设置为true,则代表是一个持久的队列
2、发送消息的时候将deliveryMode=2

**(3)消费者丢数据**

消费者丢数据一般是因为采用了自动确认消息模式。这种模式下，消费者会自动确认收到信息。这时rahbitMQ会立即将消息删除，这种情况下如果消费者出现异常而没能处理该消息，就会丢失该消息。
至于解决方案，**采用手动确认消息即可**。

### 5.SpringBoot整合

```xml
   <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
```

```properties
spring.rabbitmq.host=192.168.1.10
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

```java
@Autowired
    RabbitTemplate rabbitTemplate;
    @Test
    void testMQ(){
        //需要自定义一个MESSAGE对象
        rabbitTemplate.send(exchange,routeKey,Message);
        //传入要发送的对象，自动序列化发给MQ
        rabbitTemplate.convertAndSend(exchange,routeKey,object);
        //接受消息
        rabbitTemplate.receiveAndConvert(queue);
    }
```

其次，可以自定义其序列化规则，如下就是json格式的序列化

```java
@Configuration
public class MqConfig {
    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }
}
```

利用监听，监听到对应queue收到消息就会执行方法，通过 `@RabbitListener` 注解定义该类对”hello-world” 队列的监听，并用 `@RabbitHandler` 注解来指定对消息的处理方法

```java
//主程序上开启基于注解的RabbitMQ
@SpringBootApplication
@EnableRabbit
public class DataApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataApplication.class, args);
    }
}
```

```java
@RabbitListener(queues="")
public class Tut1Receiver {

    @RabbitHandler
    public void receive(String in) {
        System.out.println(" [x] Received '" + in + "'");
    }

}
```

自定义绑定信息

```java
@Bean
public Binding binding1a(TopicExchange topic, Queue autoDeleteQueue1) {
    return BindingBuilder.bind(autoDeleteQueue1)
        .to(topic)
        .with("*.orange.*");
}
```



利用AmqpAdmin管理组件**

 ```java
 @Autowired
 AmqpAdmin amqpAdmin;

 amqpAdmin.declareExchange(new DirectExchange("wyf"));
 amqpAdmin.declareQueue(new Queue("aa",true));
 amqpAdmin.declareBinding(new Binding("aa",Binding.DestinationType.QUEUE,"wyf","",null));
 ```



# 三.SpringBoot与全文检索

## 1.docker启动ES

```properties
docker run -e ES_JAVA_OPTS="-Xms256m -Xmx256m" -d -p 9200:9200 -p 9300:9300 --name myes b11
79d41a7b4
```

由于ES启动默认2G内存，所以制定内存为256M（-e ES_JAVA_OPTS="-Xms256m -Xmx256m"）。web通信使用9200端口，分布式情况下，ES各节点使用9300端口。

同时，在elasticsearch.yml中，需要如下配置

```yaml
#跨域
http.cors.enabled: true
http.cors.allow-origin: "*"
#新版ES7以上需要
cluster.initial_master_nodes: ["node-1"]
```

## 2.基本概念

### 1.文档

Elasticsearch是**面向文档(document oriented)**的，这意味着它可以存储整个对象或**文档(document)**。ELasticsearch使用**Javascript对象符号(JavaScript Object Notation)**，也就是[**JSON**](http://en.wikipedia.org/wiki/Json)，作为文档序列化格式

### 2.索引

在Elasticsearch中存储数据的行为就叫做**索引(indexing)**。

Elasticsearch集群可以包含多个**索引(indices)**（数据库），每一个索引可以包含多个**类型(types)**（表），每一个类型包含多个**文档(documents)**（行），然后每个文档包含多个**字段(Fields)**（列）。类比传统关系型数据库:

```properties
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

```Javascript
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

我们看到path:`/megacorp/employee/1`包含三部分信息：

| 名字     | 说明         |
| -------- | ------------ |
| megacorp | 索引名       |
| employee | 类型名       |
| 1        | 这个员工的ID |

在7.x版本之后，ES取消了type的定义，使用命令

```http
PUT /megacorp/_doc/1
```



自动创建索引

![es自动创建索引](D:\Program Files\笔记\image\es自动创建索引.JPG)



## 3.检索

简单的检索如下：

```Jacscript
GET /megacorp/employee/1
```

查看多个文档

```
POST /_mget
```

```json
{
    "docs":[
        {
        	"_index":"",
        	"_doc":"",
        	"_id":""
        },{
			"_index":"",
        	"_doc":"",
        	"_id":""
        }
    ]
}
```



我们通过HTTP方法`GET`来检索文档，同样的，我们可以使用`DELETE`方法删除文档，使用`HEAD`方法检查某文档是否存在。如果想更新已存在的文档，我们只需再`PUT`一次。

修改也可利用_update

```
POST /index_name/_update/id
```

消息体

```json
{
    "docs":{
        "age":"xx",
        "name":"xx"
    }
}
```

额外添加一个字段

```
POST /index_name/_update/id
{
	"script":"ctx._source.age=10"
}
```

删除一个字段

```
POST /index_name/_update/id
{
	"script":"ctx._source.remove=(\"age\)"
}
```

更新制定文档的字段

```
POST /index_name/_update/id
{
	"script":{
		"source":"ctx._source.age+=params.age",
		"params":{
			"age"：4
		},
		"upsert":{
			"age":1
		}
	}
}
```

其中，params里面的"age"为自定义名称，upsert则表示如果age存在则修改，不存在则新增

[更多内容可以参考官方文档](https://es.xiaoleilu.com/010_Intro/30_Tutorial_Search.html)

## 4.SpringBoot整合

SpringBoot默认支持两种技术和Elasticsearch进行交互：Spring Data Elasticsearch和Jest。

Jest默认不生效，需要导入io.searchbox.client.JestClient。

Spring Data Elasticsearch：

注册了**client**，属性有clusterNodes和clusterName。

注册了**ElasticsearchTemplate**来操作ES

启用了**ElasticsearchRepository**，可以编写它的子接口来操作ES

- Jest

  配置文件中配置相关属性

  ```
  spring.elasticsearch.jest.uris=http://192.168.205.128:9200
  ```

  ```java
  @Autowired
      private JestClient jestClient;
  
      public String add() throws IOException {
          //user对象的ID添加注解@JestID
          User user = new User(1,"fanqi","123456",1);
          //构建一个索引
          Index index = new Index.Builder(user).index("coreqi").type("user").build();
          //执行
          DocumentResult result =jestClient.execute(index);
          return result.getJsonString();
      }
  
      public String search() throws IOException {
          String searchJson = "{\n" +
                  "    \"query\": {\n" +
                  "        \"match\": {\n" +
                  "            \"UserName\": \"fanqi\"\n" +
                  "        }\n" +
                  "    }\n" +
                  "}";
          //构建一个搜索
          Search search = new Search.Builder(searchJson).addIndex("coreqi").addType("user").build();
          //执行
          SearchResult result = jestClient.execute(search);
          return result.getJsonString();
      }
  ```

- **ElasticsearchRepository使用**

  类似于JPA，编写自定义Repository接口，继承自ElasticsearchRepository：

  ```java
  public interface BookRepository extends ElasticsearchRepository<Book,Integer> {
   
      public List<Book> findByBookNameLike(String bookName);
  }
  
  // 这里注意注解
  @Document(indexName = "elastic",type = "book")
  public class Book（）{
      // 必须指定一个id，
      @Id
      private long id; 
      // 这里配置了分词器，字段类型，可以不配置，默认也可
      @Field(analyzer = "ik_smart", type = FieldType.Text)
      private String bookName;
  }
  
  ```

  测试类

  ```java
  @Autowired
      BookRepository bookRepository;
   
      @Test
      public void testRepository(){
          Book book = new Book();
          book.setAuthor("吴承恩");
          book.setBookName("西游记");
          book.setId(1);
          bookRepository.index(book);
          System.out.println("BookRepository 存入数据成功！");
      }
  
  	@Test
      public void testRepository2(){
          for (Book book : bookRepository.findByBookNameLike("游")) {
              System.out.println("获取的book ： "+book);
          } ;
          Book book = bookRepository.findOne(1);
          System.out.println("根据id查询 ： "+book);
      }
  ```

- **ElasticsearchTemplate使用**

  ```java
  
      @Autowired
      ElasticsearchTemplate elasticsearchTemplate;
   	//测试结果如下
      @Test
      public void testTemplate01(){
          Book book = new Book();
          book.setAuthor("曹雪芹");
          book.setBookName("红楼梦");
          book.setId(2);
          IndexQuery indexQuery = new IndexQueryBuilder().withId(String.valueOf(book.getId())).withObject(book).build();
          elasticsearchTemplate.index(indexQuery);
      }
  //查询数据示例如下：
     @Test
      public void testTemplate02(){
          QueryStringQueryBuilder stringQueryBuilder = new QueryStringQueryBuilder("楼");
          stringQueryBuilder.field("bookName");
          SearchQuery searchQuery = new NativeSearchQueryBuilder().withQuery(stringQueryBuilder).build();
          Page<Book> books = elasticsearchTemplate.queryForPage(searchQuery,Book.class);
          Iterator<Book> iterator = books.iterator();
          while(iterator.hasNext()){
              Book book = iterator.next();
              System.out.println("该次获取的book:"+book);
          }
      }
  ```

  # 四.SpringBoot异步、定时、邮件

## 1.异步

```java
@SpringBootApplication
@EnableAsync
public class DataApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataApplication.class, args);
    }
}
//对应方法添加@Async注解
@Async
 public void test(){}
```

## 2.定时

```java
@SpringBootApplication
@EnableScheduling
public class DataApplication {

    public static void main(String[] args) {
        SpringApplication.run(DataApplication.class, args);
    }
}

@Scheduled
 public void test(){}
```



