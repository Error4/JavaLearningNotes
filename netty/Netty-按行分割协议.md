```
本文转载自http://www.tianshouzhi.com/api/tutorials/netty/396
```

`LineBasedFrameDecoder` 和`LineEncoder`采用的通信协议非常简单，即按照行进行分割，遇到一个换行符，则认为是一个完整的报文。在发送方，使用`LineEncoder`为数据添加换行符；在接受方，使用`LineBasedFrameDecoder`对换行符进行解码。

# **1 LineBasedFrameDecoder**

`LineBasedFrameDecoder`采用的通信协议格式非常简单：使用换行符\n或者\r\n作为依据，遇到\n或者\r\n都认为是一条完整的消息。

`LineBasedFrameDecoder`提供了2个构造方法，如下： 

```java
public LineBasedFrameDecoder(final int maxLength) {
    this(maxLength, true, false);
}
public LineBasedFrameDecoder(final int maxLength, final boolean stripDelimiter, final boolean failFast) {
    this.maxLength = maxLength;
    this.failFast = failFast;
    this.stripDelimiter = stripDelimiter;
}
```

其中：

**maxLength：**

​    表示一行最大的长度，如果超过这个长度依然没有检测到\n或者\r\n，将会抛出`TooLongFrameException`

**failFast：**

​    与maxLength联合使用，表示超过maxLength后，抛出TooLongFrameException的时机。如果为true，则超出maxLength后立即抛出TooLongFrameException，不继续进行解码；如果为false，则等到完整的消息被解码后，再抛出TooLongFrameException异常。

**stripDelimiter：**

​    解码后的消息是否去除\n，\r\n分隔符。例如对于以下二进制字节流： 

```plain
   +--------------+
   | ABC\nDEF\r\n |
   +--------------+
```

如果stripDelimiter为true，则解码后的结果为：

```plain
   +-----+-----+
   | ABC | DEF |
   +-----+-----+
```

如果stripDelimiter为false，则解码后的结果为：

```plain
   +-------+---------+
   | ABC\n | DEF\r\n |
   +-------+---------+
```

# **2 LineEncoder**

按行编码，给定一个CharSequence(如String)，在其之后添加换行符\n或者\r\n，并封装到ByteBuf进行输出，与LineBasedFrameDecoder相对应。LineEncoder提供了多个构造方法，最终调用的都是：

```java
public LineEncoder(LineSeparator lineSeparator, //换行符号
               Charset charset) //换行符编码，默认为CharsetUtil.UTF_8
```

Netty提供了`LineSeparator`来指定换行符，其定义了3个常量， 一般使用`DEFAULT`即可。

```java
public final class LineSeparator {
    //读取系统属性line.separator，如果读取不到，默认为\n
    public static final LineSeparator DEFAULT = new LineSeparator(StringUtil.NEWLINE);
    //unix操作系统换行符
    public static final LineSeparator UNIX = new LineSeparator("\n”);
    //windows操作系统换行度
    public static final LineSeparator WINDOWS = new LineSeparator("\r\n”);
    //...
}
```

# **3 LineBasedFrameDecoder / LineEncoder使用案例**

**server端：LineBasedFrameDecoderServer**

```java
public class LineBasedFrameDecoderServer {
   public static void main(String[] args) throws Exception {
      EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
      EventLoopGroup workerGroup = new NioEventLoopGroup();
      try {
         ServerBootstrap b = new ServerBootstrap(); // (2)
         b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class) // (3)
               .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                  @Override
                  public void initChannel(SocketChannel ch) throws Exception {
                     // 使用LineBasedFrameDecoder解决粘包问题，其会根据"\n"或"\r\n"对二进制数据进行拆分，封装到不同的ByteBuf实例中
                     ch.pipeline().addLast(new LineBasedFrameDecoder(1024, true, true));
                     // 自定义这个ChannelInboundHandler打印拆包后的结果
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                           if (msg instanceof ByteBuf) {
                              ByteBuf packet = (ByteBuf) msg;
                              System.out.println(
                                    new Date().toString() + ":" + packet.toString(Charset.defaultCharset()));
                           }
                        }
                     });
                  }
               });
         // Bind and start to accept incoming connections.
         ChannelFuture f = b.bind(8080).sync(); // (7)
         System.out.println("LineBasedFrameDecoderServer Started on 8080...");
         f.channel().closeFuture().sync();
      } finally {
         workerGroup.shutdownGracefully();
         bossGroup.shutdownGracefully();
      }
   }
}
```

LineBasedFrameDecoder要解决粘包问题，根据"\n"或"\r\n"对二进制数据进行解码，可能会解析出多个完整的请求报文，其会将每个有效报文封装在不同的ByteBuf实例中，然后针对每个ByteBuf实例都会调用一次其他的ChannelInboundHandler的channelRead方法。

因此LineBasedFrameDecoder接受到一次数据，其之后的ChannelInboundHandler的channelRead方法可能会被调用多次，且之后的ChannelInboundHandler的channelRead方法接受到的ByteBuf实例参数，包含的都是都是一个完整报文的二进制数据。因此无需再处理粘包问题，只需要将ByteBuf中包含的请求信息解析出来即可，然后进行相应的处理。本例中，我们仅仅是打印。 

**client端：LineBasedFrameDecoderClient**

```java
public class LineBasedFrameDecoderClient {
   public static void main(String[] args) throws Exception {
      EventLoopGroup workerGroup = new NioEventLoopGroup();
      try {
         Bootstrap b = new Bootstrap(); // (1)
         b.group(workerGroup); // (2)
         b.channel(NioSocketChannel.class); // (3)
         b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
         b.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
               //ch.pipeline().addLast(new LineEncoder());自己添加换行符，不使用LineEncoder
               ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                   //在于server建立连接后，即发送请求报文
                  public void channelActive(ChannelHandlerContext ctx) {
                     byte[] req1 = ("hello1" + System.getProperty("line.separator")).getBytes();
                     byte[] req2 = ("hello2" + System.getProperty("line.separator")).getBytes();
                     byte[] req3_1 = ("hello3").getBytes();
                            byte[] req3_2 = (System.getProperty("line.separator")).getBytes();
                     ByteBuf buffer = Unpooled.buffer();
                     buffer.writeBytes(req1);
                     buffer.writeBytes(req2);
                     buffer.writeBytes(req3_1);
                     ctx.writeAndFlush(buffer);
                            try {
                                TimeUnit.SECONDS.sleep(2);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            buffer = Unpooled.buffer();
                            buffer.writeBytes(req3_2);
                            ctx.writeAndFlush(buffer);
                        }
               });
            }
         });
         // Start the client.
         ChannelFuture f = b.connect("127.0.0.1",8080).sync(); // (5)
         // Wait until the connection is closed.
         f.channel().closeFuture().sync();
      } finally {
         workerGroup.shutdownGracefully();
      }
   }
}
```

**需要注意的是，在client端，我们并没有使用LineEncoder进行编码，原因在于我们要模拟`粘包、拆包`。如果使用LineEncoder，那么每次调用ctx.write或者ctx.writeAndflush，LineEncoder都会自动添加换行符，无法模拟拆包问题。**

​     我们通过自定义了一个ChannelInboundHandler，用于在连接建立后，发送3个请求报文req1、req2、req3。其中req1和req2都是一个完整的报文，因为二者都包含一个换行符；req3分两次发送，第一次发送req3_1，第二次发送req3_2。 

首先我们将req1、req2和req3_1一起发送：

```plain
   +------------------------+
   | hello1\nhello2\nhello3 |
   +------------------------+
```

而服务端经过解码后，得到两个完整的请求req1、req2、以及req3的部分数据：

```plain
   +----------+----------+--------
   | hello1\n | hello2\n | hello3 
   +----------+----------+--------
```

由于req1、req2都是一个完整的请求，因此可以直接处理。而req3由于只接收到了一部分(半包)，需要等到2秒后，接收到另一部分才能处理。

​    因此当我们先后启动server端和client之后，在server端的控制台将会有类似以下输出： 

```plain
LineBasedFrameDecoderServer Started on 8080...
Wed May 06 17:13:42 GMT+08:00 2020:hello1
Wed May 06 17:13:42 GMT+08:00 2020:hello2
Wed May 06 17:13:44 GMT+08:00 2020:hello3
```

可以看到hello1和hello2是同一时间打印出来的，而hello3是2秒之后才打印。说明LineBasedFrameDecoder成功帮我们处理了粘包和半包问题。

最后，有几点进行说明：

- 部分同学可能认为调用一个writeAndFlush方法就是发送了一个请求，这是对协议的理解不够深刻。一个完整的请求是由协议规定的，例如我们在这里使用了LineBasedFrameDecoder，潜在的含义就是：一行数据才算一个完整的报文。因此当你调用writeAndFlush方法，如果发送的数据有多个换行符，意味着相当于发送了多次有效请求；而如果发送的数据不包含换行符，意味着你的数据还不足以构成一个有效请求。
- 对于粘包问题，例如是两个有效报文粘在一起，那么服务端解码后，可以立即处理这两个报文。
- 对于拆包问题，例如一个报文是完整的，另一个只是半包，netty会对半包的数据进行缓存，等到可以构成一个完整的有效报文后，才会进行处理。这意味着么netty需要缓存每个client的半包数据，如果很多client都发送半包，缓存的数据就会占用大量内存空间。因此我们在实际开发中，不要像上面案例那样，有意将报文拆开来发送。
- 此外，如果client发送了半包，而剩余部分内容没有发送就关闭了，对于这种情况，netty服务端在销毁连接时，会自动清空之前缓存的数据，不会一直缓存。 