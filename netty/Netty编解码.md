在上一节中，我们提到了TCP的粘包、拆包问题以及解决思想，在确定了解决思想的情况下，发送方和接收方要完成的工作不同：

**编码：**发送方要将发送的二进制数据转换成协议规定的格式的二进制数据流，称之为编码(encode)，编码功能由编码器(encoder)完成。

**解码：**接收方需要根据协议的格式，对二进制数据进行解析，称之为解码(decode)，解码功能由解码器(decoder)完成。

**编解码：**如果有一种组件，既能编码，又能解码，则称之为编码解码器(codec)。这种组件在发送方和接收方都可以使用。

**因此对于开发人员而言，我们要做的工作主要就是2点：确定协议、编写协议对应的编码/解码器。**

Netty提供了一套完善的编解码框架，不论是公有协议/私有协议，我们都可以在这个框架的基础上，非常容易的实现相应的编码/解码器。输入的数据是在`ChannelInboundHandler`中处理的，数据输出是在`ChannelOutboundHandler`中处理的。因此编码器/解码器实际上是这两个接口的特殊实现类，不过它们的作用仅仅是编码/解码。

# 解码器

对于解码器，Netty中主要提供了抽象基类`ByteToMessageDecoder`和`MessageToMessageDecoder`

![1588746501941](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588746501941.png)



### ByteToMessageDecoder

将接收到的二进制数据(Byte)解码，得到完整的请求报文(Message)。

通常，`ByteToMessageDecoder`解码后内容会得到一个`ByteBuf`实例列表，每个`ByteBuf`实例都包含了一个完整的报文信息。

`ByteToMessageDecoder`提供的一些常见的实现类：

- **`FixedLengthFrameDecoder`：**定长协议解码器，我们可以指定固定的字节数算一个完整的报文
- **`LineBasedFrameDecoder`：**行分隔符解码器，遇到\n或者\r\n，则认为是一个完整的报文
- **`DelimiterBasedFrameDecoder`：**分隔符解码器，与`LineBasedFrameDecoder`类似，只不过分隔符可以自己指定
- **`LengthFieldBasedFrameDecoder`：**长度编码解码器，将报文划分为报文头/报文体，根据报文头中的Length字段确定报文体的长度，因此报文提的长度是可变的
- **`JsonObjectDecoder`：json格式解码器，当检测到匹配数量的"{" 、”}”或”[””]”时，则认为是一个完整的`json`对象或者`json`数组。

由于Netty根本不知道开发者想封装到什么对象中，甚至不知道报文中的具体内容是什么。所以，这些实现类，都只是将接收到的二进制数据，解码成包含完整报文信息的`ByteBuf`实例后，就直接交给了之后的`ChannelInboundHandler`处理

 我们也可以自定义`ByteToMessageDecoder`，此时需要覆盖`ByteToMessageDecoder`的`decode`方法：

```java
protected abstract void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception;
```

其中：

- in：需要解码的二进制数据。
- List<Object> out：解码后的有效报文列表，我们需要将解码后的报文添加到这个List中。之所以使用一个List表示，是因为考虑到粘包问题，因此入参的in中可能包含多个有效报文。当然，也有可能发生了拆包，in中包含的数据还不足以构成一个有效报文，此时不往List中添加元素即可。

需要注意，在解码时，应该首先要判断能否构成一个有效的报文。假如有如下自定义解码类`ToIntegerDecoder`

```java
public class ToIntegerDecoder extends ByteToMessageDecoder {
    @Override
   public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    if (in.readableBytes() >= 4) {
        out.add(in.readInt());
    } }
}
```

只有在可读字节数>=4的情况下，我们才进行解码，即读取一个int，并添加到List中。

在可读字节数小于4的情况下，我们并没有做任何处理。父类`ByteToMessageDecoder`发现这次解码List中的元素没有变化，则会对in中的剩余字节进行缓存，等待后续字节的到来，之后再回到调用`ToIntegerDecoder`的decode方法。

另外，`ByteToMessageDecoder`在每次回调子类的decode方法之后，都会判断==输入的`ByteBuf`中是否还有剩余字节可读====，如果还有，会再次回调子类的decode方法，直到某个回调decode方法List中的元素个数没有变化时才停止，因此此处在实现`ByteToMessageDecoder`时，decode方法每次只解析一个有效报文，没有必要一次全部解析出来。==

### MessageToMessageDecoder

将一个本身就包含完整报文信息的对象转换成另一个Java对象。

Netty提供的`MessageToMessageDecoder`实现类较少，主要是：

**`StringDecoder`：**用于将包含完整的报文信息的`ByteBuf`转换成字符串。我们可以将其与`ByteToMessageDecoder`的一些实现类联合使用，以`LineBasedFrameDecoder`为例，其将二进制数据流按行分割后封装到`ByteBuf`中。我们可以在其之后再添加一个`StringDecoder`，将`ByteBuf`中的数据转换成字符串。

**`Base64Decoder`：**用于`Base64`编码。例如，前面我们提到`LineBasedFrameDecoder`、`DelimiterBasedFrameDecoder`等`ByteToMessageDecoder`实现类，是使用特殊字符作为分隔符作为解码的条件。但是如果报文内容中如果本身就包含了分隔符，那么解码就会出错。此时，对于发送方，可以先使用`Base64Encoder`对报文内容进行`Base64`编码，然后我们选择`Base64`编码包含的64种字符之外的其他特殊字符作为分隔符。在解码时，首先特殊字符进行分割，然后通过`Base64Decoder`解码得到原始的二进制字节流。

同样，我们也可以覆盖`MessageToMessageDecoder`的`decode`来实现自定义逻辑。

现在我们想编写一个`IntegerToStringDecoder`，把前面编写的`ToIntegerDecoder`输出的int参数转换成字符串

```java
public class IntegerToStringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg List<Object> out) throws Exception {
        out.add(String.valueOf(msg));
    }
}
```

其中，

- msg:需要进行解码的参数。

- List<Object> out：将msg经过解析后得到的java对象，添加到放到List<Object> out中

此时我们应该按照如下顺序组织`ChannelPipieline`中`ToIntegerDecoder`和`IntegerToStringDecoder` 的关系：

```java
ChannelPipieline ch=....
ch.addLast(new ToIntegerDecoder());
ch.addLast(new IntegerToStringDecoder());
```

即，前一个`ChannelInboudHandler`输出的参数类型，就是后一个`ChannelInboudHandler`的输入类型。

# 编码器

与`ByteToMessageDecoder`和`MessageToMessageDecoder`相对应，Netty提供了对应的编码器实现`MessageToByteEncoder`和`MessageToMessageEncoder`，二者都实现`ChannelOutboundHandler`接口。

![1588746517652](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588746517652.png)

相对来说，编码器比解码器的实现要更加简单，原因在于解码器除了要按照协议解析数据，还要要处理粘包、拆包问题；而编码器只要将数据转换成协议规定的二进制格式发送即可。

### MessageToByteEncoder

 MessageToByteEncoder也是一个泛型类，泛型参数I表示将需要编码的对象的类型，编码的结果是将信息转换成二进制流放入ByteBuf中。子类通过覆写其抽象方法`encode`，来实现编码，如下所示：

```java
public abstract class MessageToByteEncoder<I> extends ChannelOutboundHandlerAdapter {
	....
     protected abstract void encode(ChannelHandlerContext ctx, I msg, ByteBuf out) throws Exception;
}
```

​     可以看到，`MessageToByteEncoder`的输出对象out是一个`ByteBuf`实例，我们应该将泛型参数msg包含的信息写入到这个out对象中。

​     `MessageToByteEncoder`使用案例：

```java
public class IntegerToByteEncoder extends MessageToByteEncoder<Integer> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
        out.writeInt(msg);//将Integer转成二进制字节流写入ByteBuf中
    }
}
```

### MessageToMessageEncoder

`MessageToMessageEncoder`同样是一个泛型类，泛型参数I表示将需要编码的对象的类型，编码的结果是将信息放到一个List中。子类通过覆写其抽象方法encode，来实现编码，如下所示：

```java
public abstract class MessageToMessageEncoder<I> extends ChannelOutboundHandlerAdapter {
   ...
   protected abstract void encode(ChannelHandlerContext ctx, I msg, List<Object> out) throws Exception;
   ...
}
```

与`MessageToByteEncoder`不同的，`MessageToMessageEncoder`编码后的结果放到的out参数类型是一个List中。例如，你一次发送2个报文，因此msg参数中实际上包含了2个报文，因此应该解码出两个报文对象放到List中。

`MessageToMessageEncoder`提供的常见子类包括：

- **`LineEncoder`：**按行编码，给定一个`CharSequence`(如String)，在其之后添加换行符\n或者\r\n，并封装到`ByteBuf`进行输出，与`LineBasedFrameDecoder`相对应。
- **`Base64Encoder`：**给定一个`ByteBuf`，得到对其包含的二进制数据进行`Base64`编码后的新的`ByteBuf`进行输出，与`Base64Decoder`相对应。
- **`LengthFieldPrepender`：**给定一个`ByteBuf`，为其添加报文头Length字段，得到一个新的`ByteBuf`进行输出。Length字段表示报文长度，与`LengthFieldBasedFrameDecoder`相对应。
- **`StringEncoder`：**给定一个`CharSequence`(如：`StringBuilder`、`StringBuffer`、`String`等)，将其转换成`ByteBuf`进行输出，与`StringDecoder`对应

# 编码解码器

编码解码器同时具有编码与解码功能，特点同时实现了`ChannelInboundHandler`和`ChannelOutboundHandler`接口，因此在数据输入和输出时都能进行处理。Netty提供提供了一个`ChannelDuplexHandler`适配器类，编码解码器的抽象基类 `ByteToMessageCodec` 、`MessageToMessageCodec`都继承与此类

![1588748559037](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588748559037.png)

ByteToMessageCodec内部维护了一个ByteToMessageDecoder和一个MessageToByteEncoder实例，可以认为是二者的功集合，泛型参数I是接受的编码类型：

```java
public abstract class ByteToMessageCodec<I> extends ChannelDuplexHandler {
    private final TypeParameterMatcher outboundMsgMatcher;
    private final MessageToByteEncoder<I> encoder;
    private final ByteToMessageDecoder decoder = new ByteToMessageDecoder(){…}
  
    ...
    protected abstract void encode(ChannelHandlerContext ctx, I msg, ByteBuf out) throws Exception;
    protected abstract void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception;
    ...
}
```

MessageToMessageCodec内部维护了一个MessageToMessageDecoder和一个MessageToMessageEncoder实例，可以认为是二者的功集合，泛型参数`INBOUND_IN`和`OUTBOUND_IN`分别表示需要解码和编码的数据类型。

```java
public abstract class MessageToMessageCodec<INBOUND_IN, OUTBOUND_IN> extends ChannelDuplexHandler {
   private final MessageToMessageEncoder<Object> encoder= ...
   private final MessageToMessageDecoder<Object> decoder =…
   ...
   protected abstract void encode(ChannelHandlerContext ctx, OUTBOUND_IN msg, List<Object> out) throws Exception;
   protected abstract void decode(ChannelHandlerContext ctx, INBOUND_IN msg, List<Object> out) throws Exception;
}
```