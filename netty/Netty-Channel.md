通过上一篇文章，已经对Netty的使用有了一个感性的认识，下面我会对Netty中重要的概念加以说明。首先就是Channel。

# 1.概述

基于JDK1.4之前，基于BIO，我们通常使用java.net包中的`ServerSocket`和`Socket`来代表服务端和客户端。

在之后引入NIO编程之后，我们使用java.nio.channels包中的`ServerSocketChannel`和`SocketChannel`来代表服务端与客户端。

在Netty中，对Java中的BIO、NIO编程api都进行了封装，分别：

1. 使用了`OioServerSocketChannel`，`OioSocketChannel`对java.net包中的ServerSocket与Socket进行了封装

2. 使用`NioServerSocketChannel`和`NioSocketChannel`对java.nio.channels包中的ServerSocketChannel和SocketChannel进行了封装。

   具体继承关系如下：

![1588691941553](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588691941553.png)

需要注意，在`OioServerSocketChannel`，`OioSocketChannel`是没有通道Channel的，虽然名字中带有Channel...以`OioSocketChannel`为例，并且也能注意到其已经是过时的类了，推荐还是使用`NioServerSocketChannel`和`NioSocketChannel`

```java
@Deprecated
public class OioSocketChannel extends OioByteStreamChannel implements SocketChannel {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(OioSocketChannel.class);

    private final Socket socket;
    ...
}
```

`NioSocketChannel`和`NioServerSocketChannel`对`AbstractNioChannel`的javaChannel()进行了覆写

```java
@Override
protected ServerSocketChannel javaChannel() {//返回java.nio.channels.ServerSocketChannel
  return (ServerSocketChannel) super.javaChannel();
}

@Override
protected SocketChannel javaChannel() {//返回java.nio.channels.SocketChannel
    return (SocketChannel) super.javaChannel();
}
```

# 2.ChannelConfig

在Netty中，每种Channel都有对应的配置，用`ChannelConfig`来表示，`ChannelConfig`是一个接口，每个特定的Channel实现类都有自己对应的`ChannelConfig`实现类。部分示意如下：

![1588693023086](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588693023086.png)

在Channel接口中定义了一个方法`config()`，用于获取特定通道实现的配置，子类需要实现这个接口。

```java
public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable<Channel>{
    ...
    ChannelConfig config(); 
    ...
}
```

通常Channel实例，在创建的时候，就会创建其对应的`ChannelConfig`实例。例如`NioSocketChannel`在构造方法中创建了其对应的`ChannelConfig`实现。

```java
public class NioSocketChannel extends AbstractNioByteChannel implements io.netty.channel.socket.SocketChannel {
    ...
    private final SocketChannelConfig config;
	...
     public NioSocketChannel(Channel parent, SocketChannel socket) {
        super(parent, socket);
        config = new NioSocketChannelConfig(this, socket.socket());
    }
    ...
    @Override
    public SocketChannelConfig config() {//覆写config方法，返回SocketChannelConfig实例
        return config;
    }
}
```

此外，在`ChannelConfig`中，我们还可以注意到如下方法

```java
 	Map<ChannelOption<?>, Object> getOptions();

    boolean setOptions(Map<ChannelOption<?>, ?> options);

    <T> T getOption(ChannelOption<T> option);

    <T> boolean setOption(ChannelOption<T> option, T value);
```

其中有个`ChannelOption`类，可以认为`ChannelConfig`中用了一个Map来保存参数，Map的key是`ChannelOption`，`ChannelConfig` 定义了相关方法来获取和修改Map中的值。

当我们想修改一个Map中的参数时，例如我们希望NioSocketChannel在工作过程中，使用`PooledByteBufAllocator`来分配内存，则可以使用类似以下方式来设置

```java
Channel ch = ...;
SocketChannelConfig cfg = (SocketChannelConfig) ch.getConfig();
cfg.setOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

或者

```
Channel ch = ...;
SocketChannelConfig cfg = (SocketChannelConfig) ch.getConfig();
cfg.setAllocator(PooledByteBufAllocator.DEFAULT);
```

除此之外，还有更多的自定义配置

```java
ChannelOption.ALLOCATOR
ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK
ChannelOption.WRITE_BUFFER_LOW_WATER_MARK
ChannelOption.MESSAGE_SIZE_ESTIMATOR
ChannelOption.AUTO_CLOSE
.....
```

每一种ChannelOption，除了可以使用setOption方法来进行设置，在ChannelConfig接口中都为其设置了对应的快捷set/get方法。

```java
public interface SocketChannelConfig extends ChannelConfig {
    ...
    @Override
    SocketChannelConfig setAllocator(ByteBufAllocator allocator);

    @Override
    SocketChannelConfig setRecvByteBufAllocator(RecvByteBufAllocator allocator);

    @Override
    SocketChannelConfig setAutoRead(boolean autoRead);

    @Override
    SocketChannelConfig setAutoClose(boolean autoClose);
    ...
}
```

# 3.ChannelHander

在NIO编程中，我们经常需要对channel的输入和输出事件进行处理，Netty抽象出一个`ChannelHandler`概念，专门用于处理此类事件。又因为IO事件分为输入和输出，因此`ChannelHandler`又具体的分为`ChannelInboundHandler`和`ChannelOutboundHandler `，分别用于某个阶段输入输出事件的处理。

类继承如图：

![1588694395086](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588694395086.png)

对于`ChannelHandlerAdapter`、`ChannelInboundHandlerAdapter `、`ChannelOutboundHandlerAdapter`，从名字就可以看出来其作用是适配器。

通常，处理IO事件时，会分成几个阶段。以读取数据为例，通常我们的处理顺序是：

```
处理半包或者粘包问题-->数据的解码(或者说是反序列化)-->数据的业务处理
```

不同的阶段要执行不同的功能，因此通常我们会编写多个`ChannelHandler`，来实现不同的功能。而且多个`ChannelHandler`之间的顺序不能颠倒，例如我们必须先处理粘包解包问题，之后才能进行数据的业务处理。

## **ChannelPipeline**

Netty中通过`ChannelPipeline`来保证`ChannelHandler`之间的处理顺序。每一个Channel对象创建的时候，都会自动创建一个关联的`ChannelPipeline`对象，我们可以通过`io.netty.channel.Channel`对象的`pipeline()`方法获取这个对象实例。

```java
package io.netty.channel;
public abstract class AbstractChannel extends DefaultAttributeMap implements Channel {
....
private final DefaultChannelPipeline pipeline;
....
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();//创建默认的pipeline
}
....
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
....
@Override
    public ChannelPipeline pipeline() {//实现Chnannel定义的pipeline方法，返回pipeline实例
        return pipeline;
    }
}
```

可以看到，`ChannelPipeline`的定义实在`AbstractChannel`的构造方法中，而每个`Channel`只会构建一次，从而确保每个`Channel`实例唯一对应一个`ChannelPipleLine` 实例。

`ChannelPipeline` 除了负责配置handler的顺序，还负责在收到读/写事件之后按照顺序调用这些handler。假设有如下`ChannelPipeline`定义：

```java
ChannelPipeline p = ...;
p.addLast("1", new InboundHandlerA());
p.addLast("2", new InboundHandlerB());
p.addLast("3", new OutboundHandlerA());
p.addLast("4", new OutboundHandlerB());
p.addLast("5", new InboundOutboundHandlerX());
```

其中，`InboundHandlerA`，`InboundHandlerB`实现了`ChannelInboundHandler`接口，`OutboundHandlerA`，`OutboundHandlerB`实现了`ChannelOutboundHandler`接口，`InboundOutboundHandlerX`则同时实现了这两个接口，前面的1、2、3、4、5**并不是handler的编号，而是handler的名字**

当一个输入事件来的时候，输出事件处理器是不会发生作用的；反之亦然。因此：

- 当一个输入事件来了之后，事件处理器的调用顺序为1,2,5

- 当一个输出事件来了之后，事件处理器的处理顺序为5,4,3。(**注意输出事件的处理器发挥作用的顺序与定义的顺序是相反的**)

另外需要说明的是：

- 默认情况下，一个`ChannelPipeline`实例中，同一个类型`ChannelHandler`只能被添加一次，如果需要多次添加，则需要在该`ChannelHandler`实现类上添加`@Sharable`注解。
- 在`ChannelPipeline`中，每一个`ChannelHandler`都是有一个名字的，而且名字必须的是唯一的。如果没有显示的指定名字，则会按照规则起一个默认的名字

## **ChannelHandlerContext**

上文中说到通过`ChannelPipeline`的添加方法，按照顺序添加`ChannelHandler`，并在之后按照顺序进行调用。事实上，每个`ChannelHandler`会被先封装成`ChannelHandlerContext`。之后再封装进ChannelPipeline中。

`ChannelPipeline`的默认实现类是`DefaultChannelPipeline`，以`DefaultChannelPipeline`的`addLast`方法为例，如果查看源码，最终会定位到以下方法：

```java
@Override
public ChannelPipeline addLast(EventExecutorGroup group, final String name, ChannelHandler handler) {
    synchronized (this) {
        checkDuplicateName(name);//check这种类型的handler实例是否允许被添加多次
       //将handler包装成一个DefaultChannelHandlerContext类
        AbstractChannelHandlerContext newCtx = new DefaultChannelHandlerContext(this, group, name, handler);
        addLast0(name, newCtx);//维护AbstractChannelHandlerContext的先后关系
    }
 
    return this;
}
```

`DefaultChannelPipeline`内部是通过一个双向链表记录`ChannelHandler`的先后关系，而双向链表的节点是`AbstractChannelHandlerContext`类。相关源码如下：

```java
public class DefaultChannelPipeline implements ChannelPipeline {
...
private static final String HEAD_NAME = generateName0(HeadContext.class);
private static final String TAIL_NAME = generateName0(TailContext.class);
...
final AbstractChannelHandlerContext head;//双向链表的头元素
final AbstractChannelHandlerContext tail;//双向列表的尾部元素
 
private final Channel channel;
....
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
     ....
    tail = new TailContext(this);//创建双向链表头部元素实例
    head = new HeadContext(this);//创建双向链表的尾部元素实例
    //设置链表关系
    head.next = tail;
    tail.prev = head;
}
....
....
private void addLast0(AbstractChannelHandlerContext newCtx) {
   //设置ChannelHandler的先后顺序关系
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
   }
 }
}
```

## **ChannelHander、ChannelPipeline、ChannelHandlerContext的联合工作过程**

前面提到`DefaultChannelPipeline`是将`ChannelHander`包装成`AbstractChannelHandlerContext`类之后，再添加到链表结构中的，从而实现handler的级联调用。

`ChannelInboundHandler` 接口定义的9个方法：

```java
public interface ChannelInboundHandler extends ChannelHandler {    
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;   
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;    
    void channelActive(ChannelHandlerContext ctx) throws Exception;   
    void channelInactive(ChannelHandlerContext ctx) throws Exception;   
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;   
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;    
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;    	void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;   
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
```

而在`ChannelPipeline`和`ChannelHandlerContext`中，都定义了相同的9个以fire开头的方法，例如在`ChannelPipeline`中：

![1588727700592](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588727700592.png)

可以看到这9个方法与ChannelInboundHandler的方法是一一对应的

**从总体上来说，在调用的时候，是按照如下顺序进行的：**

1、先是`ChannelPipeline`中的`fireXXX`方法被调用

2、`ChannelPipeline`中的`fireXXX`方法接着调用`ChannelPipeline`维护的`ChannelHandlerContext`链表中的第一个节点即`HeadContext` 的`fireXXX`方法

3、`ChannelHandlerContext` 中的`fireXXX`方法调用`ChannelHandler`中对应的`XXX`方法。由于可能存在多个`ChannelHandler`，因此每个`ChannelHandler`的`xxx`方法又要负责调用下一个`ChannelHandlerContext`的`fireXXX`方法，直到整个调用链完成


  