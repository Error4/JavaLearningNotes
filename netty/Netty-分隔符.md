```
本文转载自http://www.tianshouzhi.com/api/tutorials/netty/345
```

# **1 DelimiterBasedFrameDecoder介绍**

​        上一节我们介绍了`LineBasedFrameDecoder`，其以换行符\n或者\r\n作为依据，遇到\n或者\r\n都认为是一条完整的消息。

​        而`DelimiterBasedFrameDecoder`与LineBasedFrameDecoder类似，只不过更加通用，允许我们指定任意特殊字符作为分隔符。我们还可以同时指定多个分隔符，如果在请求中发的确有多个分隔符，将会选择内容最短的一个分隔符作为依据。

例如： 

```plain
   +--------------+
   | ABC\nDEF\r\n |
   +--------------+
```

如果我们指定分隔符为\n，那么将会解码出来2个消息

```plain
   +-----+-----+
   | ABC | DEF |
   +-----+-----+
```

如果我们指定\r\n作为分隔符，那么只会解码出来一条消息

```plain
  +----------+
   | ABC\nDEF |
   +----------+
```

DelimiterBasedFrameDecoder提供了多个构造方法，最终调用的都是以下构造方法：

```java
public DelimiterBasedFrameDecoder(
            int maxFrameLength, boolean stripDelimiter, boolean failFast, ByteBuf... delimiters)
```

其中：

**maxLength：**

​    表示一行最大的长度，如果超过这个长度依然没有检测到\n或者\r\n，将会抛出TooLongFrameException

**failFast：**

​    与maxLength联合使用，表示超过maxLength后，抛出TooLongFrameException的时机。如果为true，则超出maxLength后立即抛出TooLongFrameException，不继续进行解码；如果为false，则等到完整的消息被解码后，再抛出TooLongFrameException异常。

**stripDelimiter：**

​     解码后的消息是否去除分隔符。

**delimiters：**

​     分隔符。我们需要先将分割符，写入到ByteBuf中，然后当做参数传入。

​    需要注意的是，netty并没有提供一个DelimiterBasedFrameDecoder对应的编码器实现(笔者没有找到)，因此在发送端需要自行编码，添加分隔符。 

# **2 Base64编解码**

 对于以特殊字符作为报文分割条件的协议的解码器，如：LineBasedFrameDecoder、DelimiterBasedFrameDecoder。都存在一个典型的问题，如果发送数据当中本身就包含了分隔符，怎么办？如：我们要发送的内容为：

```plain
hello1\nhello2\nhello3\n
```

我们需要把这个内容整体当做一个有效报文来处理，而不是拆分成hello1、hello2、hello3。一些同学可能想到那可以换其他的特殊字符，但是如果内容中又包含你想指定的其他特殊字符怎么办呢？

因此我们通常需要发送的内容进行base64编码，base64中总共只包含了64个字符。 

| **索引** | **对应字符** | **索引** | **对应字符** | **索引** | **对应字符** | **索引** | **对应字符** |
| -------- | ------------ | -------- | ------------ | -------- | ------------ | -------- | ------------ |
| 0        | **A**        | 17       | **R**        | 34       | **i**        | 51       | **z**        |
| 1        | **B**        | 18       | **S**        | 35       | **j**        | 52       | **0**        |
| 2        | **C**        | 19       | **T**        | 36       | **k**        | 53       | **1**        |
| 3        | **D**        | 20       | **U**        | 37       | **l**        | 54       | **2**        |
| 4        | **E**        | 21       | **V**        | 38       | **m**        | 55       | **3**        |
| 5        | **F**        | 22       | **W**        | 39       | **n**        | 56       | **4**        |
| 6        | **G**        | 23       | **X**        | 40       | **o**        | 57       | **5**        |
| 7        | **H**        | 24       | **Y**        | 41       | **p**        | 58       | **6**        |
| 8        | **I**        | 25       | **Z**        | 42       | **q**        | 59       | **7**        |
| 9        | **J**        | 26       | **a**        | 43       | **r**        | 60       | **8**        |
| 10       | **K**        | 27       | **b**        | 44       | **s**        | 61       | **9**        |
| 11       | **L**        | 28       | **c**        | 45       | **t**        | 62       | **+**        |
| 12       | **M**        | 29       | **d**        | 46       | **u**        | 63       | **/**        |
| 13       | **N**        | 30       | **e**        | 47       | **v**        |          |              |
| 14       | **O**        | 31       | **f**        | 48       | **w**        |          |              |
| 15       | **P**        | 32       | **g**        | 49       | **x**        |          |              |
| 16       | **Q**        | 33       | **h**        | 50       | **y**        |          |              |

我们可以指定这64个字符之外的其他字符作为特殊分割字符；而接收端对应的进行base64解码，得到对应的原始的二进制流，然后进行处理。Netty提供了`Base64Encoder`/`Base64Decoder`来帮我们处理这个问题。需要注意的是，只需要对内容进行base64编码，分隔符不需要编码。

# 3 DelimiterBasedFrameDecoder结合Base64编解码案例 

**Server端：**DelimiterBasedFrameDecoderServer

```java
public class DelimiterBasedFrameDecoderServer {
   public static void main(String[] args) throws Exception {
      EventLoopGroup bossGroup = new NioEventLoopGroup(); 
      EventLoopGroup workerGroup = new NioEventLoopGroup();
      try {
         ServerBootstrap b = new ServerBootstrap();
         b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class) 
               .childHandler(new ChannelInitializer<SocketChannel>() { 
                  @Override
                  public void initChannel(SocketChannel ch) throws Exception {
                     ByteBuf delemiter= Unpooled.buffer();
                     delemiter.writeBytes("&".getBytes());
                    //先使用DelimiterBasedFrameDecoder解码，以&作为分割符
                    ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, true, true,delemiter));
                    //之后使用Base64Decoder对数据进行解码，得到报文的原始的二进制流
                    ch.pipeline().addLast(new Base64Decoder());
                    //对请求报文进行处理
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                           if (msg instanceof ByteBuf) {
                              ByteBuf packet = (ByteBuf) msg;
                              System.out.println(
                                    new Date().toLocaleString() + ":" + packet.toString(Charset.defaultCharset()));
                           }
                        }
                     });
                  }
               });
         // Bind and start to accept incoming connections.
         ChannelFuture f = b.bind(8080).sync();
         System.out.println("DelimiterBasedFrameDecoderServer Started on 8080...");
         f.channel().closeFuture().sync();
      } finally {
         workerGroup.shutdownGracefully();
         bossGroup.shutdownGracefully();
      }
   }
}
```

Server接受到的数据，首先我们要去除分隔符，然后才能进行base64解码。因此首选我们添加了DelimiterBasedFrameDecoder根据&处理粘包办包问题，之后使用Base64Decoder进行解码，最后通过一个自定义ChannelInboundHandler打印请求的数据。



**client端**：DelimiterBasedFrameDecoderClient 

```java
public class DelimiterBasedFrameDecoderClient {
   public static void main(String[] args) throws Exception {
      EventLoopGroup workerGroup = new NioEventLoopGroup();
      try {
         Bootstrap b = new Bootstrap(); 
         b.group(workerGroup); 
         b.channel(NioSocketChannel.class); 
         b.option(ChannelOption.SO_KEEPALIVE, true); 
         b.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
               ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                   //在于server建立连接后，即发送请求报文
                  public void channelActive(ChannelHandlerContext ctx) {
                     //先对要发送的原始内容进行base64编码
                     ByteBuf content = Base64.encode(Unpooled.buffer().writeBytes("hello&tianshouzhi&".getBytes
                           ()));
                     //之后添加分隔符
                     ByteBuf req = Unpooled.copiedBuffer(content);
                     req.writeBytes("&".getBytes());
                     ctx.writeAndFlush(req);
                        }
               });
            }
         });
         // Start the client.
         ChannelFuture f = b.connect("127.0.0.1",8080).sync(); 
         // Wait until the connection is closed.
         f.channel().closeFuture().sync();
      } finally {
         workerGroup.shutdownGracefully();
      }
   }
}
```

在编写client端代码时我们先对原始内容进行base64编码，然后添加分割符之后进行输出。

​    需要注意，虽然Netty提供了Base64Encoder进行编码，这里并没有直接使用，如果直接使用Base64Encoder，那么会对我们输出的所有内容进行编码，意味着分隔符也会被编码，这显然不符合我们的预期，所以这里直接使用了Netty提供了Base64工具类来处理。

​    如果一定要使用Base64Encoder，那么代码需要进行相应的修改，自定义的ChannelInboundHandler只输出原始内容，之后通过Base64Encoder进行编码，然后需要额外再定义一个ChannelOutboundHandler添加分隔符，如： 

```java
ch.pipeline().addLast(new ChannelOutboundHandlerAdapter() {
   @Override
   public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
      if(msg instanceof ByteBuf){
         ((ByteBuf) msg).writeBytes("&".getBytes());
      }
   }
});
ch.pipeline().addLast(new Base64Encoder());
ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
    //在于server建立连接后，即发送请求报文
   public void channelActive(ChannelHandlerContext ctx) {
      ByteBuf req = Unpooled.buffer().writeBytes("hello&tianshouzhi&".getBytes());
      ctx.writeAndFlush(req);
                   }
});
```

当我们先后启动server和client后，会看到server端控制台输出： 

```plain
DelimiterBasedFrameDecoderServer Started on 8080...
2018-9-8 13:56:32:hello&tianshouzhi&
```

说明经过base64编码后，我们的请求中可以包含分隔符作为内容。

如果我们将base64编解码相关逻辑去掉，你将会看到的输出是： 

```plain
DelimiterBasedFrameDecoderServer Started on 8080...
2018-9-8 14:13:31:hello
2018-9-8 14:13:31:tianshouzhi
```

也就是说，原始内容也被误分割了，解码失败，读者可以自行验证。

