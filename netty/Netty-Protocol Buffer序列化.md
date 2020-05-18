# 1.Protocol Buffer

## 1.1 .简介

　　Protocol Buffer(简称ProtoBuf)是google的一个语言中立，平台中立，可扩展的对结构化的数据进行序列化的一种机制，和XML类似，但是比XML更小，更快，更简单。你只用一次性的定义你的数据结构，我们可以使用特殊的生成的代码读/写到不同的数据流中或者不同的语言中。这是官网（https://developers.google.com/protocol-buffers/）对Protoco Buffer的描述。详情也可自行查看其他文章对Protocol Buffer的解释，本博文所采用的是Protocol Buffer的最新版本3.11.0。

## 1.2 .Protocol Buffer的使用

### 配置Protocol Buffer的编译器

　　A. 下载地址: https://github.com/protocolbuffers/protobuf/releases/tag/v3.11.0， 如下图所示：

![1588814827844](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588814827844.png)

根据需要下载对应版本即可，本文使用windows64的版本。下载之后解压即可。方便起见，建议将解压后的bin目录加入到环境变量。

### 编写message文件

　　以官网实例demo为例，message文件要以 .proto 结尾，文件内容如下：　

```
syntax = "proto2";

package tutorial;

option java_package = "com.example.tutorial";
option java_outer_classname = "AddressBookProtos";

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

**syntax:** 语法，即编写该文件所采用的是 protocol buffer 的哪个版本的语法，本示例中采用的是 proto2, 也可以写proto3

**package:** 包名，在Java语言中，如果没有指定 java_package，那么在生产Java文件的时候，就以package的值作为Java类的包名；如果指定了java_package，我们依然需要指定package的值，因为在某些非Java语言中是没有包的概念，可以防止命名冲突的问题。

**java_package:** Java的包名，可写可不写，不写的情况下，就以package的值作为生成的Java类的包名。

**java_outer_classname:** 生产的Java类的类名，如果没有指定的情况下，就以文件名按照驼峰式命名法来定义类名，例如文件名叫做 person_test.proto，在没有指定java_outer_classname的情况下，生成的类名为PersonTest。

**message:** 定义消息。消息由一系列的属性组成，每个属性必须有类型，例如int32, int64, bool, float, double, string等。

**enum:** 定义枚举类型。

**“=1”，"=2":** 用来标记每个属性在二进制编码数据中的唯一标签。属性的标签的值介于1-15比标签为15以上的属性在编码后所占的字节数要少，在优化的时候，我们通常将15以上的标签定义给那些可选的元素。而将1-15作为那些必须的或者可重复的元素的标签。

元素必须指定修饰符，如下：

　　**required:** 必须的，必须给元素指定一个值。

　　**optional:** 可选的。

　　**repeated:** 可重复的，你可以将其看作一个数组。

### 生成Java文件

protoc命令的用法如下：

```
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```

$SRC_DIR是.proto文件所在目录，如果都不提供的话，默认就生成在当前目录 ， $DST_DIR是生成的java类存储目录。

最简单的情况，直接输入

```
protoc --java_out=.addressbook.proto
```

生成的文件位于：com/example/tutorial/AddressBookProtos.java

现在我们来看一下protobuf为我们生成的类 AddressBookProtos.java。在这个类中，包含了我们在addressbook.proto 文件中指定的message生成的类。每一个类都有一个自己的 `Builder` ，我们可以用这些builder来创建这些类的实例。

![Image.png](http://static.tianshouzhi.com/ueditor/upload/image/20170205/1486305433619059510.png)

这些根据message生成的类和对应的builder都拥有访问message中每个字段的方法。messages类只拥有getters方法，而对应的builder同时拥有getters和setters方法。

如果你想参考更多的生成Java代码的细节，请参考 [Java generated code reference](https://developers.google.com/protocol-buffers/docs/reference/java-generated).。

### 运行程序

　　先导入 protocol buffer 所需要的jar包（https://github.com/google/protobuf/tree/master/java 中有描述），maven依赖如下：

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.11.0</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java-util</artifactId>
    <version>3.11.0</version>
</dependency>
```

　　Java代码如下：

```java
public class Test {
    public static void main(String[] args) throws InvalidProtocolBufferException {
        AddressBookProtos.AddressBook pb = AddressBookProtos.AddressBook.newBuilder()
                .addPeople(
            AddressBookProtos.Person.newBuilder()
            .setEmail("345@qq.com")
            .setId(34)
            .setName("zhangsn"))
                .addPeople(
            AddressBookProtos.Person.newBuilder()
            .setEmail("123@163.com")
            .setId(12)
            .setName("lisi"))
            .build();
        byte[] bs = pb.toByteArray();
        AddressBookProtos.AddressBook pb1 = AddressBookProtos.AddressBook.parseFrom(bs);
        List<AddressBookProtos.Person> list = pb1.getPeopleList();
        list.forEach(p -> {
            System.out.println(p.getEmail() + ";;" + p.getId() + ";;" + p.getName());
        });
    }
}　　
```

# 2.Netty编码的实现

## 2.1 服务端启动代码

```java
public class PbServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try{
            ServerBootstrap serverBootstrap = new ServerBootstrap();

            serverBootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new PbServerChannelInitilizer());

            ChannelFuture channelFuture = serverBootstrap.bind(8989).sync();
            channelFuture.channel().closeFuture().sync();
        }finally{
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

## 2.2 服务端管道初始化代码

```java
public class PbServerChannelInitilizer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        /**
         * 对采用Base 128 Varints进行编码的数据解码
         */
        pipeline.addLast("protobufVarint32FrameDecoder", new ProtobufVarint32FrameDecoder());
        pipeline.addLast("protobufDecoder", new ProtobufDecoder(AddressBookProtos.AddressBook.getDefaultInstance()));

        /**
         * 采用Base 128 Varints进行编码，在消息头上加上32个整数，来标注数据的长度
         */
        pipeline.addLast("protobufVarint32LengthFieldPrepender", new ProtobufVarint32LengthFieldPrepender());
        pipeline.addLast("protobufEncoder", new ProtobufEncoder());
        pipeline.addLast("serverHandler", new PbServerHandler());
    }
}
```

补充说明：

首先在ChannelPipeline中添加了ProtobufVarint32FrameDecoder，其主要用于半包处理，针对ProtobufVarint32LengthFieldPrepender()所加的长度属性的解码器

随后添加ProtobufDecoder，它的参数类型是com.google.protobuf.MessageLite，实际上就是告诉ProtobufDecoder需要解码的目标类是什么，否则仅仅从字节数组中是无法判断出要解码的目标类型的。

## 2.3 服务端Handler处理

```java
public class PbServerHandler extends SimpleChannelInboundHandler<AddressBookProtos.AddressBook> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, AddressBookProtos.AddressBook msg) throws Exception {
        List<AddressBookProtos.Person> list = msg.getPeopleList();
        list.forEach(p -> System.out.println(p.getName() + ";;" + p.getId() + ";;" + p.getEmail()));
    }
}
```

## 2.4 客户端启动程序

```java
public class PbClient {
    public static void main(String[] args) throws Exception {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        try{
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(eventLoopGroup).channel(NioSocketChannel.class)
                    .handler(new PbClientChannelInitializer());

            ChannelFuture channelFuture = bootstrap.connect("localhost", 8989).sync();
            channelFuture.channel().closeFuture().sync();
        }finally{
            eventLoopGroup.shutdownGracefully();
        }
    }
}
```

## 2.5客户端通道初始化

```java
public class PbClientChannelInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        pipeline.addLast("protobufVarint32FrameDecoder", new ProtobufVarint32FrameDecoder());
        pipeline.addLast("protobufDecoder", new ProtobufDecoder(AddressBookProtos.AddressBook.getDefaultInstance()));
        pipeline.addLast("protobufVarint32LengthFieldPrepender", new ProtobufVarint32LengthFieldPrepender());
        pipeline.addLast("protobufEncoder", new ProtobufEncoder());

        pipeline.addLast("clientHandler", new PbClientHandler());
    }
}
```

## 2.6 客户端Handler处理代码

```java
public class PbClientHandler extends SimpleChannelInboundHandler<AddressBookProtos.AddressBook> {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        AddressBookProtos.AddressBook ab = AddressBookProtos.AddressBook.newBuilder()
                .addPeople(AddressBookProtos.Person.newBuilder().setEmail("123@qq.com").setId(23).setName("zhangsan"))
                .addPeople(AddressBookProtos.Person.newBuilder().setEmail("789@163.com").setId(45).setName("lisi"))
                .build();
        ctx.writeAndFlush(ab);
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, AddressBookProtos.AddressBook msg) throws Exception {

    }
}
```















