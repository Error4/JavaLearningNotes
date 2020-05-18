转载自http://www.tianshouzhi.com/api/tutorials/netty/397

## **1 FixedLengthFrameDecoder简介**

`FixedLengthFrameDecoder`采用的是定长协议：即把固定的长度的字节数当做一个完整的消息

例如，我们规定每3个字节，表示一个有效报文，如果我们分4次总共发送以下9个字节： 

```plain
   +---+----+------+----+
   | A | BC | DEFG | HI |
   +---+----+------+----+
```

那么通过FixedLengthFrameDecoder解码后，实际上只会解析出来3个有效报文

```plain
  +-----+-----+-----+
   | ABC | DEF | GHI |
   +-----+-----+-----+
```

FixedLengthFrameDecodert提供了以下构造方法

```java
public FixedLengthFrameDecoder(int frameLength) {
    if (frameLength <= 0) {
        throw new IllegalArgumentException(
                "frameLength must be a positive integer: " + frameLength);
    }
    this.frameLength = frameLength;
}
```

其中：`frameLength`就是我们指定的长度。

需要注意的是FixedLengthFrameDecoder并没有提供一个对应的编码器，因为接收方只需要根据字节数进行判断即可，发送方无需编码 

## **2 FixedLengthFrameDecoder使用案例**

server端：FixedLengthFrameDecoderServer 

```java
public class FixedLengthFrameDecoderServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); 
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class) 
                    .childHandler(new ChannelInitializer<SocketChannel>() { 
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new FixedLengthFrameDecoder(3));
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
            ChannelFuture f = b.bind(8080).sync(); 
            System.out.println("FixedLengthFrameDecoderServer Started on 8080...");
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

Client端：FixedLengthFrameDecoderClient

```java
public class FixedLengthFrameDecoderClient {
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
                     ByteBuf A = Unpooled.buffer().writeBytes("A".getBytes());
                     ByteBuf BC = Unpooled.buffer().writeBytes("BC".getBytes());
                     ByteBuf DEFG = Unpooled.buffer().writeBytes("DEFG".getBytes());
                     ByteBuf HI = Unpooled.buffer().writeBytes("HI".getBytes());
                     ctx.writeAndFlush(A);
                     ctx.writeAndFlush(BC);
                     ctx.writeAndFlush(DEFG);
                     ctx.writeAndFlush(HI);
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

先后运行server端与client端后，server端控制台输出

```plain
FixedLengthFrameDecoderServer Started on 8080...
Wed May 06 20:58:18 GMT+08:00 2020:ABC
Wed May 06 20:58:18 GMT+08:00 2020:DEF
Wed May 06 20:58:18 GMT+08:00 2020:GHI
```

可以看到FixedLengthFrameDecoder的确将请求的数据，按照每3个字节当做一个完整的请求报文。

通常情况下，很少有client与server交互时，直接使用定长协议，可能会造成浪费。例如你实际要发送的实际只有3个字节，但是定长协议设置的1024，那么可能你就要为这3个字节基础上，在加1021个空格，以便server端可以解析这个请求。在下一节我们将要介绍的LengthFieldBasedFrameDecoder，支持动态指定报文的长度。 