# 1.线程模型的演变

基本的，我们朴素的考虑一个任务处理流程

![](https://s3.ax1x.com/2020/11/15/DFgIot.png)

一个worker线程来处理用户提交的任务，任务接受和任务处理是在同一个worker线程中进行的，没有进行区分。这样做存在一个很大的问题是，必须要等待某个task处理完成之后，才能接受处理下一个task。而通常情况下，任务的处理过程会比任务的接受流程慢得多，白白浪费了资源。

所以，考虑将任务的接受与处理分为两个线程进行处理，一个只负责接受任务，一个只负责处理任务，也就演化出了第一个模型： 串行工作者模型。

## 1.1 **串行工作者模型**

![](https://s3.ax1x.com/2020/11/15/DFgxwn.png)

在这种情况下，接受任务的线程称之为AcceptThread，其将接受到的任务放到一个任务队列中，因此能立即返回接受下一个任务。而worker线程不断的从这个队列中取出任务进行异步执行。

但同时也存在问题，在于任务处理的太慢，导致队列里积压的任务数量越来愈大，任务不能得到及时的执行。

因此，可以考虑用多个worker thread来处理任务。**这就是串行工作者模型的并发版本-并行工作者模型。**

## **1.2 并行工作者模型**

具体实现上，又可以分为两种模式：

**基于公共任务队列**

![](https://s3.ax1x.com/2020/11/15/DF2ZwR.png)

**每个worker thread维护自己的任务队列**

在第一种方式中，由于多个worker线程同时从一个公共的任务队列中拉取任务进行处理，因此必须要进行加锁，因而影响了效率。因此又有了下面一种设计方式：reactor thread直接将任务转发给各个worker thread，**每个worker thread内部维护一个队列来处理**

![](https://s3.ax1x.com/2020/11/15/DF2Nkt.png)

而**netty的实现，就是为每个worker thread维护了一个队列。**

# 2.Netty线程模型

netty是被设计用于支持`Reactor`线程模型的，而`Reactor`线程模型是一种并发编程模型，更确切的说是一种思想，其具有的是指导意义，详情可参考[深入理解Reactor 网络编程模型](https://zhuanlan.zhihu.com/p/93612337)

根据上文，我们可以看到主要的关注点就是划分任务的接受阶段与任务的处理阶段。也正是因为如此，我们通常将接受任务的线程称之为`Accpet Thread`。而任务的处理过程都是一个线程(`worker thread`)内完成的，在`Reactor`线程模型中，处理任务并且分发的线程，不再称之为`worker thread`，而是`reactor thread`。

## 2.1 Reactor 单线程模型

所有I/O操作都由一个线程完成，即多路复用、事件分发和处理都是在一个`Reactor`线程上完成的。对于高负载、大并发的应用场景不合适。

![](https://s3.ax1x.com/2020/11/15/DF2DXQ.png)

实现方式：

```java
private EventLoopGroup group = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap()
                .group(group);
```

## 2.2 Reactor 多线程模型

`Rector` 多线程模型与单线程模型最大的区别就是有一组 NIO 线程处理 IO 操作。 有专门一个 NIO 线程——`Acceptor` 线程用于监听服务端，接收客户端的 TCP 连接请求； 网络 IO 操作读、写 等由一个 NIO 线程池负责。1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程

![](https://s3.ax1x.com/2020/11/15/DF2gkq.png)

实现方式：

```java
private EventLoopGroup boss = new NioEventLoopGroup(1);
private EventLoopGroup work = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap()
                .group(boss,work);
```

## 2.3 主从 Reactor 多线程模型

服务端用于接收客户端连接的不再是个 1 个单独的 NIO 线程，而是一个独立的NIO线程池。`Acceptor` 接收到客户端 TCP 连接请求处理完成后（可能包含接入认证等），将新创建的 `SocketChannel` 注册到 IO 线程池（sub reactor 线程池）的某个 IO 线程上，由它负责 `SocketChannel` 的读写和编解码工作。

![](https://s3.ax1x.com/2020/11/15/DF2TB9.png)

实现方式：

```java
private EventLoopGroup boss = new NioEventLoopGroup();
private EventLoopGroup work = new NioEventLoopGroup();
ServerBootstrap bootstrap = new ServerBootstrap()
                .group(boss,work);
```

