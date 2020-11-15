这里，我们首先直接以Netty入门案例入手，先感性认识一下Netty。TimeClient发送“QUERY TIME ORDER”请求，TimeServer接受到这个请求后，返回当前时间。

# Server端

## TimeServer

时间服务器TimeServer在8080端口监听客户端请求，如下 

```java
public class TimeServer {
    private int port=8080;
    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class) // (3)
                    .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new TimeServerHandler());
                        }
                    });
           
            ChannelFuture f = b.bind(port).sync(); // (5)
            System.out.println("TimeServer Started on 8080...");
            f.channel().closeFuture().sync();// (6)
        } finally {
            workerGroup.shutdownGracefully();// (7)
            bossGroup.shutdownGracefully();
        }
    }
    public static void main(String[] args) throws Exception {
        new TimeServer().run();
    }
}
```

对应说明如下：

1. 首先我们创建了两个`EventLoopGroup`实例：bossGroup和workerGroup，可以简单的将bossGroup和workerGroup理解为两个线程池。其中bossGroup用于接受客户端连接，bossGroup在接受到客户端连接之后，将连接交给workerGroup来进行处理，workerGroup进行SocketChannel的网络读写。

2. 接着，我们创建了一个`ServerBootstrap`实例，从名字上就可以看出来这是一个服务端启动辅助启动类，目的是降低服务端的开发复杂度。我们需要给设置一些参数，包括第1步创建的bossGroup和workerGroup。

3. 我们通过channel方法指定了`NioServerSocketChannel`，这是netty中表示服务端的类，用于接受客户端连接，对应于java.nio包中的ServerSocketChannel。

4. 我们通过childHandler方法，设置了一个匿名内部类`ChannelInitializer`实例，用于初始化客户端连接`SocketChannel`实例。在第3步中，我们提到NioServerSocketChannel是用于接受客户端连接，在接收到客户端连接之后，netty会回调ChannelInitializer的initChannel方法需要对这个连接进行一些初始化工作，主要是告诉netty之后如何处理和响应这个客户端的请求。在这里，主要是添加了3个ChannelHandler实例：`LineBasedFrameDecoder`、`StringDecoder`、TimeServerHandler。其中LineBasedFrameDecoder、StringDecoder是netty本身提供的，用于解决TCP粘包、解包的工具类。

   - LineBasedFrameDecoder在解析客户端请求时，遇到字符”\n”或”\r\n”时则认为是一个完整的请求报文，然后将这个请求报文的二进制字节流交给StringDecoder处理。

   - StringDecoder将字节流转换成一个字符串，交给TimeServerHandler来进行处理。
   - TimeServerHandler是我们自己要编写的类，在这个类中，我们要根据用户请求返回当前时间。

5. 在所有的配置都设置好之后，我们调用了ServerBootstrap的bind(port)方法，开启真正的监听在8080端口，接受客户端请求。 随后，调用它的同步阻塞方法sync等待绑定操作完成。完成之后Netty会返回一个ChannelFuture，它的功能类似于JDK的java.util.concurrent.Future，主要用于异步操作的通知回调。

6. 使用f.channel().closeFuture().sync()方法进行阻塞，等待服务端链路关闭之后main函数才退出。

7. 调用NIO线程组的shutdownGracefully进行优雅退出，它会释放跟shutdownGracefully相关联的资源。

## **TimeServerHandler**

TimeServerHandler继承自ChannelHandlerAdapter，它用于对网络事件进行读写操作，用户处理客户端的请求，每当接收到"QUERY TIME ORDER”请求时，就返回当前时间，否则返回"BAD REQUEST”。 通常我们只需要关注channelRead和exceptionCaught方法。

```java
public class TimeServerHandler extends ChannelInboundHandlerAdapter {
   @Override
   public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception { // 1
      String request = (String) msg; //2
      String response = null;
      if ("QUERY TIME ORDER".equals(request)) { // 3
         response = LocalDateTime.now().toString();
      } else {
         response = "BAD REQUEST";
      }
      response +=System.getProperty("line.separator"); // 4
      ByteBuf resp = Unpooled.copiedBuffer(response.getBytes()); // 5
      ctx.writeAndFlush(resp); // 6
   }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close(); //7
    }
}
```

说明：

1. TimeServerHandler继承了`ChannelInboundHandlerAdapter`，并覆盖了channelRead方法，当客户端发送了请求之后，channelRead方法会被回调。参数`ChannelHandlerContext`包含了当前发送请求的客户端的一些上下文信息，msg表示客户端发送的请求信息。
2. 我们直接msg强制转换成了String类型。这是因为我们在前面已经添加过了StringDecoder，其已经将二进制流转换成了一个字符串
3. 构建响应。会判断请求是否合法，如果请求信息是"QUERY TIME ORDER”，则返回当前时间，否则返回"BAD REQUEST”
4. 在响应内容中加上了`System.getProperty("line.separator”)`，也就是所谓的换行符。在linux操作系统中，就是”\n”，在windows操作系统是”\r\n”。加上换行符，主要是因为客户端也要对服务端的响应进行解码，当遇到一个换行符时，就认为是一个完整的响应。
5. 调用了`Unpooled.copiedBuffer`方法创建了一个缓冲区对象`ByteBuf`。在java nio包中，使用ByteBuffer类来表示一个缓冲区对象。在netty中，使用ByteBuf表示一个缓冲区对象。
6. 调用ChannelHandlerContext的writeAndFlush方法，将响应刷新到客户端
7. 当发生异常时，关闭ChannelHandlerContext，释放和ChannelHandlerContext相关联的句柄等资源。

# **Client端代码**

## **TimeClient**

TimeClient负责与服务端的8080端口建立连接 

```java
public class TimeClient {
    public static void main(String[] args) throws Exception {
        String host = "localhost";
        int port = 8080;
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.handler(new ChannelInitializer<SocketChannel>() {// (4)
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                    ch.pipeline().addLast(new StringDecoder());
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });
            ChannelFuture f = b.connect(host, port).sync(); // (5)
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

说明：

1. 首先我们创建了一个`Bootstrap`实例，与ServerBootstrap相对应，这表示一个客户端的启动类
2. 我们调用group方法给Bootstrap实例设置了一个EventLoopGroup实例。前面提到，EventLoopGroup的作用是线程池。前面在创建ServerBootstrap时，设置了一个bossGroup，一个wrokerGroup，这样做主要是为将接受连接和处理连接请求任务划分开，以提升效率。对于客户端而言，则没有这种需求，只需要设置一个EventLoopGroup实例即可。
3. 通过channel方法指定了`NioSocketChannel`，这是netty在nio编程中用于表示客户端的对象实例。
4. 类似server端，在连接创建完成，初始化的时候，我们也给SocketChannel添加了几个处理器类。其中TimeClientHandler是我们自己编写的给服务端发送请求，并接受服务端响应的处理器类。
5. 所有参数设置完成之后，调用Bootstrap的connect(host, port)方法，与服务端建立连接。

## **TimeClientHandler**

TimeClientHandler主要用于给Server端发送"QUERY TIME ORDER”请求，并接受服务端响应。 重点关注三个方法：channelActive、channelRead和exceptionCaught。

```java
public class TimeClientHandler extends ChannelInboundHandlerAdapter {
    private byte[] req=("QUERY TIME ORDER" + System.getProperty("line.separator")).getBytes();
    @Override
    public void channelActive(ChannelHandlerContext ctx) {//1
        ByteBuf message = Unpooled.buffer(req.length);
        message.writeBytes(req);
        ctx.writeAndFlush(message);
    }
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String body = (String) msg;
        System.out.println("Now is:" + body);
    }
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        ctx.close();
    }
}
```

说明：

- TimeClientHandler继承了ChannelInboundHandlerAdapter，并同时覆盖了channelActive、channelRead和exceptionCaught方法。

- 当客户端与服务端连接建立成功后，channelActive方法会被回调，我们在这个方法中给服务端发送"QUERY TIME ORDER”请求。
-  当接受到服务端响应后，channelRead方法会被会回调，我们在这个方法中打印出响应的时间信息。
- 当发生异常时，释放客户端资源。

# 参考文章

http://www.tianshouzhi.com/api/tutorials/netty/390

https://www.cnblogs.com/wade-luffy/p/6165626.html