# 1.Unix的IO模型

为了提高操作系统的稳定性及可用性，虚拟内存被操作系统划分成两块：内核空间和用户空间。

**内核空间**是操作系统所在区域。内核代码有特别的权力：它能与设备控制器通讯，控制着用户区域进程的运行状态等等。最重要的是，==所有 I/O 都直接或间接通过内核空间==。**Linux**的内核将所有外部设备都看做一个文件来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个`file descriptor`（fd，文件描述符）。

**用户空间**是常规进程所在区域。 JVM 就是常规进程，驻守于用户空间。用户空间是非特权区域：比如，在该区域执行的代码就不能直接访问硬件设备。

## 阻塞 IO 模型

最传统的一种 IO 模型，即在读写数据过程中会发生阻塞现象。当用户线程发出 IO 请求之后，内 核会去查看数据是否就绪，如果没有就绪就会等待数据就绪，而用户线程就会处于阻塞状态，用 户线程交出CPU。当数据就绪之后，内核会将数据拷贝到用户线程，并返回结果给用户线程，用 户线程才解除 block 状态。典型的阻塞 IO 模型的例子为：`data = socket.read()`，如果数据没有就 绪，就会一直阻塞在 read方法。

## 非阻塞 IO 模型

当用户线程发起一个 read操作后，并不需要等待，而是马上就得到了一个结果。如果结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送 read 操作。一旦内核中的数据准备 好了，并且又再次收到了用户线程的请求，那么它马上就将数据拷贝到了用户线程，然后返回。 所以事实上，在非阻塞 IO 模型中，用户线程需要不断地询问内核数据是否就绪，也就说非阻塞 IO 不会交出CPU，而会一直占用CPU。

但是对于非阻塞 IO 就有一个非常严重的问题，在while 循环中需要不断地去询问内核数据是否就 绪，这样会导致CPU占用率非常高

## 多路复用 IO 模型

多路复用 IO 模型是目前使用得比较多的模型。Java NIO实际上就是多路复用 IO。在多路复用 IO 模型中，会有一个线程不断去轮询多个 socket 的状态，只有当 socket 真正有读写事件时，才真 正调用实际的 IO 读写操作。因为在多路复用 IO 模型中，只需要使用一个线程就可以管理多个 socket，系统不需要建立新的进程或者线程，也不必维护这些线程和进程，并且只有在真正有 socket 读写事件进行时，才会使用 IO 资源，所以它大大减少了资源占用。因此，多路复用 IO 比较适合连接数比较多的情况。

其中，IO多路复用的发展大致经历了select、poll、epoll三个阶段。

### select：

函数原型如下：

```c++
int select(int maxfdp1,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout);
```

`fd_set`是表示文件描述符集合的数据结构，`readset`，`writeset`，`exceptset`对应三类描述符集。select机制主要存在问题：

1. 每次调用select，都需要把`fd_set`集合从用户态拷贝到内核态，如果`fd_set`集合很大时，那这个开销也很大
2. 同时每次调用select都需要在内核遍历传递进来的所有`fd_set`，如果`fd_set`集合很大时，那这个开销也很大
3. 为了减少数据拷贝带来的性能损坏，内核对被监控的`fd_set`集合大小做了限制，并且这个是通过宏控制的，大小不可改变(限制为1024)

### poll

函数原型如下：

```c++
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

typedef struct pollfd {
        int fd;                         // 需要被检测或选择的文件描述符
        short events;                   // 对文件描述符fd上感兴趣的事件
        short revents;                  // 文件描述符fd上当前实际发生的事件
} pollfd_t;
```

poll的机制与select类似，但是poll使用链表保存文件描述符，因此没有了监视文件数量的限制，但是其它三个缺点依旧存在。

### epoll

Linux中提供的epoll相关函数如下：

```c++
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

它改进了select与poll的所有缺点，epoll将select与poll分为了三个部分：

1. `epoll_ecreate`创建一个epoll对象
2. `epoll_ctl`向epoll对象中添加socket套接字顺便给内核中断处理程序注册一个callback，高速内核，当文件描述符上有事件到达（或者中断）的时候就调用这个callback
3. 调用`epoll_wait`函数等待事件的就绪

在实现上epoll()的三个核心点是：

1. 使用mmap共享内存，即用户空间和内核空间共享的一块物理地址，这样当内核空间要对文件描述符上的事件进行检查时就不需要来回拷贝数据了
2. 红黑树，用于存储文件描述符，当内核初始化epoll时，会开辟出一块内核高速cache区，这块区域用于存储我们需要监管的所有Socket描述符，由于红黑树的数据结构，对文件描述符增删查效率大为提高
3. rdlist，就绪描述符链表区，这是一个双向链表，epoll_wait()函数返回的也就是这个就绪链表，上面的epoll_ctl说了添加对象的时候会注册一个callback，这个callbakc的作用实际就是将描述符放入rdlist中，所以当一个socket上的数据到达的时候内核就会把网卡上的数据复制到内核，然后把socket描述符插入到就绪链表rdlist中

## 信号驱动 IO 模型

在信号驱动 IO 模型中，当用户线程发起一个 IO 请求操作，会给对应的 socket 注册一个信号函 数，然后用户线程会继续执行，当内核数据就绪时会发送一个信号给用户线程，用户线程接收到 信号之后，便在信号函数中调用 IO 读写操作来进行实际的 IO 请求操作。

## 异步 IO 模型

异步 IO 模型是最理想的 IO 模型，在异步 IO 模型中，当用户线程发起 read 操作之后，立刻就 可以开始去做其它的事。而另一方面，从内核的角度，当它受到一个 asynchronous read 之后， 它会立刻返回，说明 read请求已经成功发起了，因此不会对用户线程产生任何 block。然后，内核会等待数据准备完成，然后将数据拷贝到用户线程，当这一切都完成之后，内核会给用户线程 发送一个信号，告诉它 read 操作完成了。也就说用户线程完全不需要实际的整个 IO 操作是如何 进行的，只需要先发起一个请求，当接收内核返回的成功信号时表示 IO 操作已经完成，可以直接 去使用数据了。

在异步 IO 模型中，IO 操作的两个阶段都不会阻塞用户线程，这两个阶段都是由内核自动完 成，然后发送一个信号告知用户线程操作已完成。

# 2.Java NIO

这里仅对Java NIO的几个核心部分组成进行简短介绍，包括Buffer，Channel，Selector。详情可参考

[Java NIO实现原理之Buffer](https://www.jianshu.com/p/70a0941737b6)

[Java NIO实现原理之Channel](https://www.jianshu.com/p/3c8a65929a36)

[Java NIO实现原理之Selector](https://www.jianshu.com/p/19fa229e5a1a)

## Channel

国内大多翻译成“通道”。Channel 和 IO 中的 Stream(流)是差不多一个 等级的。但也有所区别：

- 流是单向的，通道是双向的，可读可写。
- 流读写是阻塞的，通道可以异步读写。
- 流中的数据可以选择性的先读到缓存中，通道的数据总是要先读到一个缓存中，或从缓存中写入 

NIO中的Channel 的主要实现有：

- FileChannel 
-  DatagramChannel
- SocketChannel 
- ServerSocketChannel

这里看名字就可以猜出个所以然来：分别可以对应文件 IO、UDP和 TCP（Server 和 Client）。

## Buffer

故名思意，缓冲区，实际上是一个容器，是一个连续数组。内部维护几个特殊变量，实现数据的反复利用

- **mark**：初始值为-1，用于备份当前的position;
- **position**：初始值为0，position表示当前可以写入或读取数据的位置，当写入或读取一个数据后，position向前移动到下一个位置；
- **limit**：写模式下，limit表示最多能往Buffer里写多少数据，等于capacity值；读模式下，limit表示最多可以读取多少数据。
- **capacity**：缓存数组大小

![](https://s1.ax1x.com/2020/10/25/BmqhGj.png)

## Selector 

NIO的核心类，Selector 能够检测多个注册的通道上是否有事件发生，如果有事件发生，便获取事件然后针对每个事件进行相应的响应处理。这样一来，只是用一个单线程就可 以管理多个通道，也就是管理多个连接。

为了实现Selector管理多个SocketChannel，必须将具体的SocketChannel对象注册到Selector，并声明需要监听的事件（这样Selector才知道需要记录什么数据），一共有4种事件：

- **connect**：客户端连接服务端事件，对应值为SelectionKey.OP_CONNECT
- **accept**：服务端接收客户端连接事件，对应值为SelectionKey.OP_ACCEPT
- **read**：读事件，对应值为SelectionKey.OP_READ
- **write**：写事件，对应值为SelectionKey.OP_WRITE

这个很好理解，每次请求到达服务器，都是从connect开始，connect成功后，服务端开始准备accept，准备就绪，开始读数据，并处理，最后写回数据返回。

一段典型的服务端的示例代码如下：

```java
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.socket().bind(new InetSocketAddress(port));
Selector selector = Selector.open();
serverChannel.register(selector, SelectionKey.OP_ACCEPT);
while(true){
    int n = selector.select();
    if (n == 0) continue;
    Iterator ite = this.selector.selectedKeys().iterator();
    while(ite.hasNext()){
        SelectionKey key = (SelectionKey)ite.next();
        if (key.isAcceptable()){
            SocketChannel clntChan = ((ServerSocketChannel) key.channel()).accept();
            clntChan.configureBlocking(false);
            //将选择器注册到连接到的客户端信道，
            //并指定该信道key值的属性为OP_READ，
            //同时为该信道指定关联的附件
            clntChan.register(key.selector(), SelectionKey.OP_READ, ByteBuffer.allocate(bufSize));
        }
        if (key.isReadable()){
            handleRead(key);
        }
        if (key.isWritable() && key.isValid()){
            handleWrite(key);
        }
        if (key.isConnectable()){
            System.out.println("isConnectable = true");
        }
      ite.remove();
    }
}
```

需要注意的是，与Selector一起使用时，Channel必须处于非阻塞模式下。这意味着不能将FileChannel与Selector一起使用，因为FileChannel不能切换到非阻塞模式。而套接字通道都可以。

尽管使用 Java NIO可以让我们使用较少的线程处理很多连接，但是**在高负载下可靠和高效地处理和调度I/O操作是一项繁琐而且容易出错的任务**，所以才引出了高性能网络编程专家——**Netty**。