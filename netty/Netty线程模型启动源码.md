 我们以一个netty服务端的典型代码进行说明：

```java
EventLoopGroup parentGroup = new NioEventLoopGroup(1);
EventLoopGroup childGroup = new NioEventLoopGroup(3);
ServerBootstrap b = new ServerBootstrap(); 
b.group(parentGroup, childGroup)
        .channel(NioServerSocketChannel.class)
.handler(new LoggingHandler(LogLevel.INFO))
        .option(ChannelOption.SO_BACKLOG, 128)
      .attr(AttributeKey.valueOf("ssc.key"),"scc.value")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new DiscardServerHandler());
            }
        }) 
            .childOption(ChannelOption.SO_KEEPALIVE, true); 
        .childAttr(AttributeKey.valueOf("sc.key"),"sc.value")
        .bind(port);
```

其对应的reactor线程模型

![](https://s3.ax1x.com/2020/11/15/DFRh5t.png)

大致包含如下5个步骤：

1. 设置服务端`ServerBootStrap`启动参数
2. 通过`ServerBootStrap`的bind方法启动服务端，bind方法会在`parentGroup`中注册`NioServerScoketChannel`，监听客户端的连接请求
3. Client发起连接CONNECT请求，`parentGroup`中的`NioEventLoop`不断轮循是否有新的客户端请求，如果有，ACCEPT事件触发
4. ACCEPT事件触发后，`parentGroup`中`NioEventLoop`会通过`NioServerSocketChannel`获取到对应的代表客户端的`NioSocketChannel`，并将其注册到`childGroup`中
5. `childGroup`中的`NioEventLoop`不断检测自己管理的`NioSocketChannel`是否有读写事件准备好，如果有的话，调用对应的`ChannelHandler`进行处理

以下是步骤的解析：

## **1.设置服务端ServerBootStrap启动参数**

整体来看，`ServerBootStrap`继承自`AbstractBootstrap`，其代表服务端的启动类，当调用其`bind`方法时，表示启动服务端。在启动之前，我们会调用`group，channel、handler、option、attr、childHandler、childOption、childAttr`等方法来设置一些启动参数。

**group方法**：

group可以认为是设置执行任务的线程池，在Netty中，`EventLoopGroup` 的作用类似于线程池，每个`EventLoopGroup`中包含多个`EventLoop`对象，代表不同的线程。

如上例所示，我们在创建`parentGroup` 、`childGroup` 时，分别传入了构造参数1和3，这对应了上图中红色部分的`parentGroup` 中只有1个`NioEventLoop`，绿色部分的`childGroup` 中有3个`NioEventLoop`。另外如果没有指定参数，或者传入的是0，那么这个`NioEventLoopGroup`包含的`NioEventLoop`个数将会是：cpu核数*2。

之后，我们可以调用`ServerBootStrap` 的`group()`方法来获取parentGroup 的引用，这个方法父类`AbstractBootstrap`继承的。另外可以通过调用`ServerBootStrap`自己定义的`childGroup()`方法来获取`childGroup`的引用。

相关代码如下：

io.netty.bootstrap.ServerBootstrap

```java
public final class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {
..........
private volatile EventLoopGroup childGroup;//ServerBootStrap自己维护childGroup
..........
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);//将parentGroup传递给父类AbstractBootstrap处理
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;//给childGroup赋值
    return this;
}
.......
  public EventLoopGroup childGroup() {//获取childGroup
    return childGroup;
  }
}
```

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
......
private volatile EventLoopGroup group;//这个字段将会被设置为parentGroup
......
public B group(EventLoopGroup group) {
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
    this.group = group;
    return (B) this;
}
......
public final EventLoopGroup group() {//获取parentGroup
    return group;
}
}
```

**channel方法：**

channel方法继承自`AbstractBootstrap`，用于构造通道的工厂类`ChannelFactory`实例，在之后需要创建通道实例，例如`NioServerSocketChannel`的时候，通过调用`ChannelFactory.newChannel()`方法来创建。

```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
........
//这个工厂类最终创建的通道实例，就是channel方法指定的NioServerSocketChannel
private volatile ChannelFactory<? extends C> channelFactory;
..........
public B channel(Class<? extends C> channelClass) {
    ......
    return channelFactory(new BootstrapChannelFactory<C>(channelClass));
}
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    ..........
    this.channelFactory = channelFactory;
    return (B) this;
}
.......
final ChannelFactory<? extends C> channelFactory() {
    return channelFactory;
}
}
```

可以发现，除了以上两种方法外，其他三个方法都是一 一对应的:

- `handler-->childHandler` ：分别用于设置`NioServerSocketChannel`和`NioSocketChannel`的处理器链，也就是当有一个NIO事件的时候，应该按照怎样的步骤进行处理。
- `option-->childOption`：分别用于设置`NioServerSocketChannel`和 `NioSocketChannel`的TCP连接参数，在`ChannelOption`类中可以看到Netty支持的所有TCP连接参数。
- `attr-->childAttr`：用于给`channel`设置一个key/value，之后可以根据key获取

**其中：**

  `handler、option、attr`方法，都是从`AbstractBootstrap`中继承的。这些方法设置的参数，将会被应用到`NioServerSocketChannel`实例上，由于`NioServerSocketChannel`一般只会创建一个，因此这些参数通常只会应用一次。

`childHandler、childOption、childAttr`方法是`ServerBootStrap`自己定义的,这些方法设置的参数，将会被应用到`NioSocketChannel`实例上，由于服务端每次接受到一个客户端连接，就会创建一个`NioSocketChannel`实例，因此每个`NioSocketChannel`实例都会应用这些方法设置的参数。

## 2.**调用ServerBootStrap的bind方法**

**调用bind方法，就相当于启动了服务端。启动的核心逻辑都是在bind方法中。**

bind方法内部，会创建一个`NioServerSocketChannel`实例，并将其在`parentGroup`中进行注册，注意这个过程对用户屏蔽了。

`parentGroup`在接受到注册请求时，会从自己的管理的`NioEventLoop`中，选择一个进行注册。由于我们的案例中，`parentGroup`只有一个`NioEventLoop`，因此只能注册到这个上。

一旦注册完成，我们就可以通过`NioServerSocketChannel`检测有没有新的客户端连接的到来。

如果一步一步追踪`ServerBootStrap.bind`方法的调用链，最终会定位到`ServerBootStrap` 父类`AbstractBootstrap`的`doBind`方法，相关源码如下：

io.netty.bootstrap.AbstractBootstrap#doBind

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();//初始化NioServerSocketChannel，并注册到bossGroup中
    ....//省略
    return promise;
```

`doBind`方法中，最重要的调用的方法是`initAndRegister`，这个方法主要完成3个任务

1、创建`NioServerSocketChannel`实例，这是通过之前创建的`ChannelFactory`实例的`newChannel`方法完成

2、初始化`NioServerSocketChannel`，即将我们前面通过`handler，option，attr`等方法设置的参数应用到`NioServerSocketChannel`上

3、将`NioServerSocketChannel` 注册到`parentGroup`中，`parentGroup`会选择其中一个`NioEventLoop`来运行这个`NioServerSocketChannel`要完成的功能，即监听客户端的连接。

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();//1、创建NioServerSocketChannel实例
        init(channel);//2、初始化NioServerSocketChannel，这是一个抽象方法，ServerBootStrap对此进行了覆盖
    } catch (Throwable t) {
        if (channel != null){
            channel.unsafe().closeForcibly();
            //由于尚未注册频道，因此我们需要强制使用GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
 
    ChannelFuture regFuture = group().register(channel);//3、NioServerSocketChannel注册到parentGroup中
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
```

`ServerBootStrap`实现了`AbstractBootstrap`的抽象方法`init`，对`NioServerSocketChannel`进行初始化。熟悉设计模式的同学会意识到，这是典型的模板设计模式，即父类运行过程中会调用多个方法，子类对特定的方法进行覆写。

在这里，`init`方法主要是为`NioServerSocketChannel`设置运行参数，也就是我们前面通过调用`ServerBootStrap`的`option、attr、handler`等方法指定的参数。

```java
 @Override
    void init(Channel channel) {
        //1、为NioServerSocketChannel设置option方法设置的参数
        setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);
         //2、为NioServerSocketChannel设置attr方法设置的参数
        setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));

        //3、为NioServerSocketChannel设置通过handler方法指定的处理器
        ChannelPipeline p = channel.pipeline();

        //4、为NioSocketChannel设置默认的处理器ServerBootstrapAcceptor，并将相关参数通过构造方法传给ServerBootstrapAcceptor
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions =
                childOptions.entrySet().toArray(newOptionArray(0));
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

特别需要注意的是，在方法的最后，除了我们通过handler方法为`NioServerSocketChannel` 指定的`ChannelHandler`之外，`ServerBootStrap`的`init`方法总是会帮我们在`NioServerSocketChannel` 的处理器链的添加一个默认的处理器`ServerBootstrapAcceptor`。

从`ServerBootstrapAcceptor` 名字上可以看出来，其是客户端连接请求的处理器。当接受到一个客户端请求之后，Netty会将创建一个代表客户端的`NioSocketChannel`对象。而我们通过`ServerBoodStrap`指定的`channelHandler、childOption、childAtrr、childGroup`等参数，也需要设置到`NioSocketChannel`中。但是明显现在，由于只是服务端刚启动，没有接收到任何客户端请求，还没有认为`NioSocketChannel`实例，因此这些参数要保存到`ServerBootstrapAcceptor`中，等到接收到客户端连接的时候，再将这些参数进行设置，我们可以看到这些参数通过构造方法传递给了`ServerBootstrapAcceptor`。

 在初始化完成之后，`ServerBootStrap`通过调用`register`方法将`NioServerSocketChannel`注册到了`parentGroup`中。

```
ChannelFuture regFuture = config().group().register(channel);
```

进一步跟踪，`parentGroup` 的类型是`NioEventLoopGroup`，`NioEventLoopGroup`的register方法继承自`MultithreadEventLoopGroup`。

```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {
    .......
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
    .......
    @Override
    public EventExecutor next() {
        return chooser.next();
    }
    ...
}
```

next方法的返回值，就是`NioEventLoop`，可以看到，真正的注册工作，是`NioEventLoop`完成的。next()方法还提供了通道在`NioEventLoop`中平均分配的机制。

`NioEventLoopGroup`创建的时候，其父类`MultithreadEventExecutorGroup`中会创建一个`EventExecutorChooser`实例，之后通过其来保证通道平均注册到不同的`NioEventLoop`中。

```java
rotected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args){
    ...
    chooser = chooserFactory.newChooser(children);
    ...
    
}
```

`chooserFactory`采用默认实现类`DefaultEventExecutorChooserFactory`

```java
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory{
	 @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        //按照round-robin的方式，来保证平均
        if (isPowerOfTwo(executors.length)) {//如果指定的线程数是2的幂
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }
    
     private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }
    
        private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }

}
```

可以看到，对于线程数是2的幂的情况，用了位运算`&`来进一步优化取模的速度，跟我们熟悉的HasnMap优化方式一致。