# **1 协议简介**

大多数的协议（私有或者公有），协议头中会携带长度字段，用于标识消息体或者整包消息的长度，例如SMPP、HTTP协议等。由于基于长度解码需求 的通用性，Netty提供了`LengthFieldBasedFrameDecoder`/`LengthFieldPrepender`，自动屏蔽TCP底层的拆包和粘包问题，只需要传入正确的参数，即可轻松解决“读半包“问题。

发送方使用LengthFieldPrepender给实际内容Content进行编码添加报文头**Length字段**，接受方使用LengthFieldBasedFrameDecoder进行解码。协议格式如下所示： 

```plain
+--------+----------+
| Length |  Content |
+--------+----------+
```

**Length字段：**

表示Conent部分的字节数，例如Length值为100，那么意味着Conent部分占用的字节数就是100。

Length字段本身是个整数，也要占用字节，一般会使用固定的字节数表示。例如我们指定使用2个字节(有符号)表示length，那么可以表示的最大值为32767(约等于32K)，也就是说，Content部分占用的字节数，最大不能超过32767。当然，Length字段存储的是Content字段的真实长度。

**Content字段：**

是我们要处理的真实二进制数据。 在发送Content内容之前，首先需要获取其真实长度，添加在内容二进制流之前，然后再发送。Length占用的字节数+Content占用的字节数，就是我们总共要发送的字节。

​     事实上，我们可以把Length部分看做报文头，报文头包含了解析报文体(Content字段)的相关元数据，例如Length报文头表示的元数据就是Content部分占用的字节数。当然，LengthFieldBasedFrameDecoder并没有限制我们只能添加Length报文头，我们可以在Length字段前或后，加上一些其他的报文头，此时协议格式如下所示： 

```plain
  +---------+--------+----------+----------+
  |........ | Length |  ....... |  Content |
  +---------+--------+----------+----------+
```

不过对于LengthFieldBasedFrameDecoder而言，其关心的只是Length字段。因此当我们在构造一个LengthFieldBasedFrameDecoder时，最主要的就是告诉其如何处理Length字段。     

# **2 LengthFieldPrepender参数详解**

LengthFieldPrepender提供了多个构造方法，最终调用的都是：

```java
public LengthFieldPrepender(
            ByteOrder byteOrder, int lengthFieldLength,
            int lengthAdjustment, boolean lengthIncludesLengthFieldLength)
```

其中：

- byteOrder：表示Length字段本身占用的字节数使用的是大端还是小端编码
- lengthFieldLength：表示Length字段本身占用的字节数,只可以指定 1, 2, 3, 4, 或 8
- lengthAdjustment：表示Length字段调整值
- lengthIncludesLengthFieldLength：表示Length字段本身占用的字节数是否包含在Length字段表示的值中。

例如：对于以下包含12个字节的报文 

```plain
   +----------------+
   | "HELLO, WORLD" |
   +----------------+
```

假设我们指定Length字段占用2个字节，lengthIncludesLengthFieldLength指定为false，即不包含本身占用的字节，那么Length字段的值为0x000C(即12)。

```plain
+--------+----------------+
+ 0x000C | "HELLO, WORLD" |
+--------+----------------+
```

如果我们指定lengthIncludesLengthFieldLength指定为true，那么Length字段的值为：0x000E(即14)=Length(2)+Content字段(12)

```plain
+--------+----------------+
+ 0x000E | "HELLO, WORLD" |
+--------+----------------+
```

关于**lengthAdjustment**字段的含义，参见下面的LengthFieldBasedFrameDecoder。

LengthFieldPrepender尤其值得说明的一点是，其提供了实现零拷贝的另一种思路(实际上编码过程，是零拷贝的一个重要应用场景)。

- 在Netty中我们可以使用ByteBufAllocator.directBuffer()创建直接缓冲区实例，从而避免数据从堆内存(用户空间)向直接内存(内核空间)的拷贝，这是系统层面的零拷贝；
- 也可以使用`CompositeByteBuf`把两个ByteBuf合并在一起，例如一个存放报文头，另一个存放报文体。而不是创建一个更大的ByteBuf，把两个小ByteBuf合并在一起，这是应用层面的零拷贝。

​      而LengthFieldPrepender，由于需要在原来的二进制数据之前添加一个Length字段，因此就需要对二者进行合并发送。但是LengthFieldPrepender并没有采用CompositeByteBuf，其编码过程如下：

```java
protected void encode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
        //1 获得Length字段的值：真实数据可读字节数+Length字段调整值
       int length = msg.readableBytes() + lengthAdjustment;
        if (lengthIncludesLengthFieldLength) {
            length += lengthFieldLength;
        }
        ...
        
        //2 根据lengthFieldLength指定的值(1、2、3、4、8)，创建一个ByteBuffer实例，写入length的值，
        //并添加到List类型的out变量中
        switch (lengthFieldLength) {
        case 1:
            if (length >= 256) {
                throw new IllegalArgumentException(
                        "length does not fit into a byte: " + length);
            }
            out.add(ctx.alloc().buffer(1).order(byteOrder).writeByte((byte) length));
            break;
        ...   
        case 8:
            out.add(ctx.alloc().buffer(8).order(byteOrder).writeLong(length));
            break;
        default:
            throw new Error("should not reach here");
        }
        //3 最后，再将msg本身添加到List中(msg.retain是增加一次引用，返回的还是msg本身)
        out.add(msg.retain());
    }
}
```

 可以看到，LengthFieldPrepender实际上是先把Length字段(报文头)添加到List中，再把msg本身(报文头)添加到List中。而在发送数据时，LengthFieldPrepender的父类MessageToMessageEncoder会按照List中的元素下标按照顺序发送，因此相当于间接的把Length字段添加到了msg之前。从而避免了创建一个更大的ByteBuf将Length字段和msg内容合并到一起。作为开发者的我们，在编写编码器的时候，这种一种重要的实现零拷贝的参考思路。

# **3 LengthFieldBasedFrameDecoder参数详解**

​    LengthFieldBasedFrameDecoder提供了多个构造方法，最终调用的都是： 

```java
public LengthFieldBasedFrameDecoder(
        ByteOrder byteOrder, int maxFrameLength, int lengthFieldOffset, int lengthFieldLength,
        int lengthAdjustment, int initialBytesToStrip, boolean failFast)
```

其中：

**byteOrder：**

​    表示协议中Length字段的字节是大端还是小端

**maxFrameLength：** 

​    表示协议中Content字段的最大长度，如果超出，则抛出TooLongFrameException异常。

**lengthFieldOffset：**  

​    表示Length字段的偏移量，即在读取一个二进制流时，跳过指定长度个字节之后的才是Length字段。如果Length字段之前没有其他报文头，指定为0即可。如果Length字段之前还有其他报文头，则需要跳过之前的报文头的字节数。

**lengthFieldLength：** 

​    表示Length字段占用的字节数。指定为多少，需要看实际要求，不同的字节数，限制了Content字段的最大长度。

- ​    如果lengthFieldLength是1个字节，那么限制为128bytes；
- ​    如果lengthFieldLength是2个字节，那么限制为32767(约等于32K)；
- ​    如果lengthFieldLength是3个字节，那么限制为8388608(约等于8M)；
- ​    如果lengthFieldLength是4个字节，那么限制为2147483648(约等于2G)。

​        lengthFieldLength与maxFrameLength并不冲突。例如我们现在希望限制报文Content字段的最大长度为32M。显然，我们看到了上面的四种情况，没有任何一个值，能刚好限制Content字段最大值刚好为32M。那么我们只能指定lengthFieldLength为4个字节，其最大限制2G是大于32M的，因此肯定能支持。但是如果Content字段长度真的是2G，server端接收到这么大的数据，如果都放在内存中，很容易造成内存溢出。

为了避免这种情况，我们就可以指定maxFrameLength字段，来精确的指定Content部分最大字节数，显然，其值应该小于lengthFieldLength指定的字节数最大可以表示的值。

**lengthAdjustment：** 

Length字段补偿值。对于绝大部分协议来说，Length字段的值表示的都是Content字段占用的字节数。但是也有一些协议，Length字段表示的是Length字段本身占用的字节数+Content字段占用的字节数。由于Netty中在解析Length字段的值是，默认是认为其只表示Content字段的长度，因此解析可能会失败，所以要进行补偿。在后面的案例3中进行了演示。

主要用于处理Length字段前后还有其他报文头的情况。具体作用请看后面的案例分析。

**initialBytesToStrip：**

解码后跳过的初始字节数，表示获取完一个完整的数据报文之后，忽略前面指定个数的字节。例如报文头只有Length字段，占用2个字节，在解码后，我们可以指定跳过2个字节。这样封装到ByteBuf中的内容，就只包含Content字段的字节内容不包含Length字段占用的字节。

 **failFast：**

如果为true，则表示读取到Length字段时，如果其值超过maxFrameLength，就立马抛出一个 TooLongFrameException，而为false表示只有当真正读取完长度域的值表示的字节之后，才会抛出 TooLongFrameException，默认情况下设置为true，建议不要修改，否则可能会造成内存溢出。

​     下面通过再几个案例，来说明这些参数是如何控制LengthFieldBasedFrameDecoder解码行为的，首先我们讨论报文只包含Length字段和Content字段的情况；接着讨论报文头除了包含Length字段，还有其他报文头字段的情况。 

## 3.1 报文只包含Length字段和Content字段

​    报文只包含Length字段和Content字段时，协议格式如下：

```plain
+--------+----------+
| Length |  Content |
+--------+----------+
```

假设Length字段占用2个字节，其值为0x000C，意味着Content字段长度为12个字节，假设其内容为”HELLO, WORLD”。下面演示指定不同解析参数时，解码后的效果。

**案例1：**

   lengthFieldOffset   = 0  //因为报文以Length字段开始，不需要跳过任何字节，所以offset为0

   lengthFieldLength   = 2 //因为我们规定Length字段占用字节数为2，所以这个字段值传入的是2

   lengthAdjustment    = 0 //这里Length字段值不需要补偿，因此设置为0

   initialBytesToStrip = 0 //不跳过初始字节，意味着解码后的ByteBuf中，包含Length+Content所有内容 

```plain
 解码前 (14 bytes)                 解码后 (14 bytes)
   +--------+----------------+      +--------+----------------+
   | Length | Actual Content |----->| Length | Actual Content |
   | 0x000C | "HELLO, WORLD" |      | 0x000C | "HELLO, WORLD" |
   +--------+----------------+      +--------+----------------+
```

**案例2：**

   lengthFieldOffset   = 0    //参见案例1

   lengthFieldLength   = 2   //参见案例1 

   lengthAdjustment    = 0  //参见案例1 

   initialBytesToStrip = 2    //这里跳过2个初始字节，也就是Length字段占用的字节数，意味着解码后的ByteBuf中，只包含Content字段

```plain
   BEFORE DECODE (14 bytes)         AFTER DECODE (12 bytes)
   +--------+----------------+      +----------------+
   | Length | Actual Content |----->| Actual Content |
   | 0x000C | "HELLO, WORLD" |      | "HELLO, WORLD" |
   +--------+----------------+      +----------------+
```

**案例3：**

   lengthFieldOffset   =  0      // 参见案例1

   lengthFieldLength   =  2     // 参见案例1

   lengthAdjustment    = -2    // Length字段补偿值指定为-2

   initialBytesToStrip =  0      //  参见案例1 

```plain
  BEFORE DECODE (14 bytes)         AFTER DECODE (14 bytes)
   +--------+----------------+      +--------+----------------+
   | Length | Actual Content |----->| Length | Actual Content |
   | 0x000E | "HELLO, WORLD" |      | 0x000E | "HELLO, WORLD" |
   +--------+----------------+      +--------+----------------+
```

这个案例需要进行一下特殊说明，其Length字段值表示：Length字段本身占用的字节数+Content字节数。所以我们看到解码前，其值为0x000E(14)，而不是0x000C(12)。而真实Content字段内容只有2个字节，因此我们需要用：Length字段值0x000E(14)，减去lengthAdjustment指定的值(-2)，表示的才是Content字段真实长度。

## **3.2 报文头包含Length字段以外的其他字段，同时包含Content字段**

​        通常情况下，一个协议的报文头除了Length字段，还会包含一些其他字段，例如协议的版本号，采用的序列化协议，是否进行了压缩，甚至还会包含一些预留的头字段，以便未来扩展。这些字段可能位于Length之前，也可能位于Length之后，此时的报文协议格式如下所示： 

```plain
 +---------+--------+----------+----------+
  |........ | Length |  ....... |  Content |
  +---------+--------+----------+----------+
```

当然，对于LengthFieldBasedFrameDecoder来说，其只关心Length字段。按照Length字段的值解析出一个完整的报文放入ByteBuf中，也就是说，LengthFieldBasedFrameDecoder只负责粘包、半包的处理，而ByteBuf中的实际内容解析，则交由后续的解码器进行处理。

下面依然通过案例进行说明：

**案例4：**

   这个案例中，在Length字段之前，还包含了一个Header字段，其占用2个字节，Length字段占用3个字节。

   lengthFieldOffset   = 2   // 需要跳过Header字段占用的2个字节，才是Length字段

   lengthFieldLength   = 3  //Length字段占用3个字节

   lengthAdjustment    = 0  //由于Length字段的值为12，表示的是Content字段长度，因此不需要调整

   initialBytesToStrip = 0    //解码后，不裁剪字节 

```plain
   BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
   +----------+----------+----------------+      +----------+----------+----------------+
   | Header   |  Length  | Actual Content |----->| Header   |  Length  | Actual Content |
   |  0xCAFE  | 0x00000C | "HELLO, WORLD" |      |  0xCAFE  | 0x00000C | "HELLO, WORLD" |
   +----------+----------+----------------+      +----------+----------+----------------+
```

**案例5：**

​    在这个案例中，Header字段位于Length字段之后

   lengthFieldOffset   = 0  // 由于一开始就是Length字段，因此不需要跳过

   lengthFieldLength   = 3 // Length字段占用3个字节，其值为0x000C，表示Content字段长度

   lengthAdjustment    = 2 // 由于Length字段之后，还有Header字段，因此需要+2个字节，读取Header+Content的内容

   initialBytesToStrip = 0   //解码后，不裁剪字节 

```plain
  BEFORE DECODE (17 bytes)                      AFTER DECODE (17 bytes)
   +----------+----------+----------------+      +----------+----------+----------------+
   |  Length  | Header   | Actual Content |----->|  Length  | Header   | Actual Content |
   | 0x00000C |  0xCAFE  | "HELLO, WORLD" |      | 0x00000C |  0xCAFE  | "HELLO, WORLD" |
   +----------+----------+----------------+      +----------+----------+----------------+
```

**案例6 :**  

   这个案例中，Length字段前后各有一个报文头字段HDR1、HDR2，各占1个字节

   lengthFieldOffset   = 1   //跳过HDR1占用的1个字节读取Length

   lengthFieldLength   = 2  //Length字段占用2个字段，其值为0x000C(12)，表示Content字段长度

   lengthAdjustment    = 1 //由于Length字段之后，还有HDR2字段，因此需要+1个字节，读取HDR2+Content的内容

   initialBytesToStrip = 3   //解码后，跳过前3个字节 

```plain
   BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
   +------+--------+------+----------------+      +------+----------------+
   | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
   | 0xCA | 0x000C | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
   +------+--------+------+----------------+      +------+----------------+
```

**案例7：**

   //这个案例中，Length字段前后各有一个报文头字段HDR1、HDR2，各占1个字节。Length占用2个字节，表示的是整个报文的总长度。

   lengthFieldOffset   =  1     //跳过HDR1占用的1个字节读取Length

   lengthFieldLength   =  2   //Length字段占用2个字段，其值为0x0010(16)，表示HDR1+Length+HDR2+Content长度

   lengthAdjustment    = -3  //由于Length表示的是整个报文的长度，减去HDR1+Length占用的3个字节后，读取HDR2+Content长度

   initialBytesToStrip =  3    //解码后，跳过前3个字节 

```plain
  BEFORE DECODE (16 bytes)                       AFTER DECODE (13 bytes)
   +------+--------+------+----------------+      +------+----------------+
   | HDR1 | Length | HDR2 | Actual Content |----->| HDR2 | Actual Content |
   | 0xCA | 0x0010 | 0xFE | "HELLO, WORLD" |      | 0xFE | "HELLO, WORLD" |
   +------+--------+------+----------------+      +------+----------------+
```

# **4  代码案例**

了解了LengthFieldPrepender和LengthFieldBasedFrameDecoder的作用后，我们编写一个实际的案例。client发送一个请求，使用LengthFieldPrepender进行编码，server接受请求，使用LengthFieldBasedFrameDecoder进行解码。

我们采用最简单的的通信协议格式，不指定其他报文头： 

```plain
+--------+-----------+
| Length |   Content |
+--------+-----------+
```

Client端代码

```java
public class LengthFieldBasedFrameDecoderClient {
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
                    ch.pipeline().addLast(new LengthFieldPrepender(2,0,false));
               ch.pipeline().addLast(new StringEncoder());
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                  // 在于server建立连接后，即发送请求报文
                  public void channelActive(ChannelHandlerContext ctx) {
                     ctx.writeAndFlush("i am request!");
                     ctx.writeAndFlush("i am a anther request!");
                  }
               });
            }
         });
         // Start the client.
         ChannelFuture f = b.connect("127.0.0.1", 8080).sync(); 
         // Wait until the connection is closed.
         f.channel().closeFuture().sync();
      } finally {
         workerGroup.shutdownGracefully();
      }
   }
}
```

在Client端我们自定义了一个ChannelInboundHandler，在连接被激活时，立即发送两个请求：**"i am request!”、****"i am a anther request!"** 。另外注意，我们是分两次调用ctx.writeAndFlush，每次调用都会导致当前请求数据经过StringEncoder进行编码，得到包含这个请求内容ByteBuf实例，  然后再到LengthFieldPrepender进行编码添加Length字段。

​      因此我们发送的实际上是以下2个报文 

```plain
Length(13)    Content
+--------+-------------------+
+ 0x000D | "i am request!"   |
+--------+-------------------+
Length(23)    Content  
+--------+-----------------------------+
+ 0x0017 | "i am a anther request!"    |
+--------+-----------------------------+
```

Server端代码代码

```java
public class LengthFieldBasedFrameDecoderServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); 
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); 
            b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class) 
                    .childHandler(new ChannelInitializer<SocketChannel>() { 
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(16384, 0, 2, 0, 2));
                            ch.pipeline().addLast(new StringDecoder());
                            ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                    System.out.println("receive req:"+msg);
                                }
                            });
                        }
                    });
            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(8080).sync(); 
            System.out.println("LengthFieldBasedFrameDecoderServer Started on 8080...");
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

在Server端，我们通过LengthFieldBasedFrameDecoder进行解码，并删除Length字段的2个字节，交给之后StringDecoder转换为字符串，最后在我们的自定义的ChannelInboundHandler进行打印。

分别启动server端与client端，在server端，我们将看到输出：

```plain
LengthFieldBasedFrameDecoderServer Started on 8080...
receive req:i am request!
receive req:i am a anther request!
```

 注意这里打印了2次，表示LengthFieldBasedFrameDecoder的确解码成功。

 部分读者可能不信任这个结果，那么可以尝试将Server端的LengthFieldBasedFrameDecoder和Client端的LengthFieldPrepender注释掉，再次运行的话，大概率你在server端控制台将看到： 

```plain
LengthFieldBasedFrameDecoderServer Started on 8080...
receive req:i am request!i am a anther request!
```

也就是说，在没有使用LengthFieldBasedFrameDecoder和LengthFieldPrepender的情况下，发生了粘包，而服务端无法区分。

# **5 总结**

​        LengthFieldBasedFrameDecoder作用实际上只是帮我们处理粘包和半包的问题，其只负责将可以构成一个完整有效的请求报文封装到ByteBuf中，之后还要依靠其他的解码器对报文的内容进行解析，例如上面编写的String将其解析为字符串，只不过在后续的解码器中，不需要处理粘包半包问题了，认为ByteBuf中包含的内容肯定是一个完整的报文即可。

​        对于请求和响应都是字符串的情况下，LengthFieldBasedFrameDecoder/LengthFieldPrepender的威力还没有完全展示出来。我们甚至可以将自定义的POJO类作为请求/响应，在发送数据前对其序列化字节数组，然后通过LengthFieldPrepender为其制定Length；服务端根据Length解析得到二进制字节流，然后反序列化再得到POJO类实例。在下一节我们将会详细的进行介绍。 