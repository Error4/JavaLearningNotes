# 1.对象序列化/反序列化

## **1.1 简介**

在实际开发中，为了简化开发，通常请求和响应都使用使用Java对象表示，对使用者屏蔽底层的协议细节。例如一些RPC框架，支持把用户定义的Java对象当做参数去请求服务端，服务端响应也是一个Java对象。

而前面我们讲解的案例都是以字符串作为请求和响应参数，实际上，Netty对于我们的自定义的Java对象作为请求响应参数也是支持的，其默认支持通过以下机制对Java对象进行序列化和反序列化：

- **ObjectEncoder/ObjectDecoder**：使用JDK序列化机制编解码
- **ProtobufEncoder/ ProtobufDecoder：**使用google protocol buffer进行编解码
- **MarshallingEncoder/MarshallingDecoder：**使用JBoss Marshalling进行编解码
- **XmlDecoder：**使用Aalto XML parser进行解码，将xml解析成Aalto XML parser中定义的Java对象，没有提供相应的编码器
- **JsonObjectDecoder：**使用Json格式解码。当检测到匹配数量的"{" 、”}”或”[””]”时，则认为是一个完整的json对象或者json数组。这个解码器只是将包含了一个完整Json格式数据的ByteBuf实例交给之后的ChannelInbounderHandler解析，因此我们需要依赖其他的JSON框架，如Gson、jackson、fastjson等。没有提供相应的编码器。 

除了Netty默认值支持的这些序列化机制，事实上还有很多的其他的序列化框架，如：`hessian`、`Kryo`、`Avro`、`fst`、`msgback`、thrift、`protostuff`等。

 在实际开发中，通常我们没有必要支持上述所有的序列化框架，支持部分即可。主要的选择依据如下表： 

| 选择依据             | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| 效率                 | 即序列化和反序列化的性能。这方面Kryo、Avro、fst、hessian等都不错。 |
| 序列化后的占用字节数 | 对于同一个Java对象，不同的框架序列化后占用的字节数不同。例如JDK序列化体积较大，而Kryo的体积较小。体积过大的话，会增加网络带宽压力。 |
| 是否有可视化需求     | json、xml序列化机制的结果能以文本形式展示；但是其他的框架大多是二进制的，因此可视化。 |
| 开发成本             | 一些序列化框架使用较为复杂，如thrift、protocol buffer；另外则很简单，如JDK序列化、Hessian等 |

从本教程而言，主要是为了介绍如何在Netty中使用这些序列化框架，方式类似，因此不会对每一种都进行介绍。

## **1.2 通信协议格式要求**

另外一点需要注意的是，上面提到的这些序列化框架通常不能单独使用。例如发送方只是将Java对象序列化成二进制字节，对于接收方而言，则无法判断到底哪些字节可以构成一个完整的Java对象，也就无法反序列化。因此我们通常需要结合长度编码，可以使用上一节提到的LengthFieldBasedFrameDecoder/LengthFieldPrepender来协助完成。

最简单的通信协议格式如下： 

```plain
+--------+----------+
| Length |  Content |
+--------+----------+
```

其中：

​    Length：表示Content字段占用的字节数，Length本身占用的字节数我们可以指定为一个固定的值。

​    Content：对象经过序列化后二进制字节内容

​    对于上述协议，通常我们只能选择支持一种序列化框架，如果要支持多个序列化框架，我们可以对通信协议格式稍作改造，增加一个字段来表示使用的序列化框架，如： 

```plain
+--------+-------------+------------+
| Length |  Serializer |   Content  |
+--------+-------------+------------+
```

 其中：Serializer我们可以指定使用1个字节表示，因此可以有256个值可选，我们用不同的值代表不同的框架。在编码时，选择好序列化框架后，进行序列化，并指定Serializer字段的值。在解码时，根据Serializer的值选择对应的框架进行反序列化。



# 2.JDK序列化

## **2.1 ObjectEncoder与ObjectDecoder简介**

​        `ObjectEncoder`/`ObjectDecoder`使用JDK序列化机制编解码，因此我们可以使用Java对象作为请求和响应参数，限制是对象必须实现Serializable。JDK序列化机制的缺点是：序列化的性能以及序列化后对象占用的字节数比较多。优点是：这是对JDK默认支持的机制，不需要引入第三方依赖。

​        如果仅仅是对象序列化，字节通过网络传输后，那么在解码时，无法判断到底多少个字节可以构成一个Java对象。因此需要结合长度编码，也就是添加一个Length字段，表示序列化后的字节占用的字节数。因此ObjectEncoder/ObjectDecoder采用的通信协议如下： 

```plain
+--------+----------+
| Length |  Content |
+--------+----------+
```

   其中：    

- Length：表示Content字段占用的字节数，Length本身占用的字节数我们可以指定为一个固定的值
- Content：对象经过JDK序列化后二进制字节内容     

​       乍一看，这与我们在上一节讲解的`LengthFieldBasedFrameDecoder`/`LengthFieldPrepender`采用的通信协议很类似。事实上ObjectDecoder本身就继承了LengthFieldBasedFrameDecoder。不过ObjectEncoder略有不同，其并没有继承LengthFieldPrepender，而是内部直接添加了Length字段。 

ObjectEncoder源码如下：

```java
@Sharable
public class ObjectEncoder extends MessageToByteEncoder<Serializable> {
    private static final byte[] LENGTH_PLACEHOLDER = new byte[4];
    //当需要编码时，encode方法会被回调 
    //参数msg：就是我们需要序列化的java对象
    //参数out：我们需要将序列化后的二进制字节写到ByteBuf中
    @Override
    protected void encode(ChannelHandlerContext ctx, Serializable msg, ByteBuf out) throws Exception {
        int startIdx = out.writerIndex();
        //ByteBufOutputStream是Netty提供的输出流，数据写入其中之后，可以通过其buffer()方法会的对应的ByteBuf实例
        ByteBufOutputStream bout = new ByteBufOutputStream(out);
        //JDK序列化机制的ObjectOutputStream
        ObjectOutputStream oout = null;
        try {
            //首先占用4个字节，这就是Length字段的字节数，这只是占位符，后面为填充对象序列化后的字节数
            bout.write(LENGTH_PLACEHOLDER);
            //CompactObjectOutputStream是netty提供的类，其实现了JDK的ObjectOutputStream，顾名思义用于压缩
            //同时把bout作为其底层输出流，意味着对象序列化后的字节直接写到了bout中
            oout = new CompactObjectOutputStream(bout);
            //调用writeObject方法，即表示开始序列化
            oout.writeObject(msg);
            oout.flush();
        } finally {
            if (oout != null) {
                oout.close();
            } else {
                bout.close();
            }
        }
        int endIdx = out.writerIndex();
        //序列化完成，设置占位符的值，也就是对象序列化后的字节数量
        out.setInt(startIdx, endIdx - startIdx - 4);
    }
}
```

ObjectDecoder源码如下所示：

```java
//注意ObjectDecoder继承了LengthFieldBasedFrameDecoder
public class ObjectDecoder extends LengthFieldBasedFrameDecoder {
    
    private final ClassResolver classResolver;
    public ObjectDecoder(ClassResolver classResolver) {
        this(1048576, classResolver);
    }
    //参数maxObjectSize：表示可接受的对象反序列化的最大字节数，默认为1048576 bytes，约等于1M
    //参数classResolver：由于需要将二进制字节反序列化为Java对象，需要指定一个ClassResolver来加载这个类的字节码对象
    public ObjectDecoder(int maxObjectSize, ClassResolver classResolver) {
        //调用父类LengthFieldBasedFrameDecoder构造方法，关于这几个参数的作用，参见之前章节的分析
        super(maxObjectSize, 0, 4, 0, 4);
        this.classResolver = classResolver;
    }
    //当需要解码时，decode方法会被回调
    @Override
    protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        //首先调用父类的decode方法进行解码，会解析出包含可解析为java对象的完整二进制字节封装到ByteBuf中，同时Length字段的4个字节会被删除
        ByteBuf frame = (ByteBuf) super.decode(ctx, in);
        if (frame == null) {
            return null;
        }
        //构造JDK ObjectInputStream实例用于解码
        ObjectInputStream ois = new CompactObjectInputStream(new ByteBufInputStream(frame, true), classResolver);
        try {
            //调用readObject方法进行解码，其返回的就是反序列化之后的Java对象
            return ois.readObject();
        } finally {
            ois.close();
        }
    }
}
```

下面我们通过实际案例来演示ObjectEncoder与ObjectDecoder的使用。

## **2.2 使用案例**

首先我们分别编写两个Java对象Request/Response分别表示请求和响应。 

```java
public class Request implements Serializable{
    private String request;
    private Date requestTime;
    //setters getters and toString
}
public class Response implements Serializable{
    private String response;
    private Date responseTime;
    //setters getters and toString
}
```

  接着我们编写Client端代码：

```java
public class JdkSerializerClient {
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
                    ch.pipeline().addLast(new ObjectEncoder());
                    ch.pipeline().addLast(new ObjectDecoder(new ClassResolver() {
                         @Override
                        public Class<?> resolve(String className) throws ClassNotFoundException {
                            return Class.forName(className);
                        }
                    }));
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                        // 在于server建立连接后，即发送请求报文
                         @Override
                        public void channelActive(ChannelHandlerContext ctx) {
                            Request request = new Request();
                            request.setRequest("i am request!");
                            request.setRequestTime(new Date());
                            ctx.writeAndFlush(request);
                        }
                        //接受服务端的响应
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            Response response= (Response) msg;
                            System.out.println("receive response:"+response);
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

 由于client既要发送Java对象Request作为请求，又要接受服务端响应的Response对象，因此在`ChannelPipeline`中，我们同时添加了ObjectDecoder和ObjectEncoder。

另外我们自定义了一个ChannelInboundHandler，在连接建立时，其channelActive方法会被回调，我们在这个方法中构造一个Request对象发送，通过ObjectEncoder进行编码。服务端会返回一个Response对象，此时channelRead方法会被回调，由于已经经过ObjectDecoder解码，因此可以直接转换为Reponse对象，然后打印。

server端： 

```java
public class JdkSerializerServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); 
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new ObjectEncoder());
                            ch.pipeline().addLast(new ObjectDecoder(new ClassResolver() {
                                 @Override
                                public Class<?> resolve(String className) throws ClassNotFoundException {
                                    return Class.forName(className);
                                }
                            }));
                            // 自定义这个ChannelInboundHandler打印拆包后的结果
                            ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                                @Override
                                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                    Request request= (Request) msg;
                                    System.out.println("receive request:"+request);
                                    Response response = new Response();
                                    response.setResponse("response to:"+request.getRequest());
                                    response.setResponseTime(new Date());
                                    ctx.writeAndFlush(response);
                                }
                            });
                        }
                    });
            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(8080).sync(); 
            System.out.println("JdkSerializerServer Started on 8080...");
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

Server端与Client端分析类似，这里不再赘述。

先后启动Server端与Client端，在Server端控制台我们将看到： 

```plain
JdkSerializerServer Started on 8080...
receive request:Request(request=i am request!, requestTime=Wed May 06 21:44:21 GMT+08:00 2020)
```

在Client端控制台我们将看到：

```plain
receive response:Response(response=response to:i am request!, responseTime=Wed May 06 21:44:21 GMT+08:00 2020)
```

到此，我们的案例完成。Request和Response都编码、解码成功了。