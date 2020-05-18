Java Web补充

# 1.Web请求过程

## 1.1 DNS域名解析

- 过程
  - 浏览器首先检查缓存中有没有该域名对应的IP，如果有，返回
  - 如果没有，浏览器会查找操作系统缓存中是否存在域名对应的IP（hosts文件中的信息会被操作系统缓存）。
  - 如果没有，则根据网络配置中设置的“DNS”服务器，将域名发送给这里设置的LDNS（Local DNS Server）
  - 如果未命中，就链接到Root Server域名服务器解析，返回给本地域名服务器一个所查询域的主域名服务器（gTLD Server，如.com,.cn）
  - 本地域名服务器在向主域名服务器查询
  - gTLD Server返回此域名对应的Name Server域名服务器的地址，Name Server会返回对应的IP和TTL值
  - 结果返回给用户，并且LDNS缓存该IP与TTL（缓存时间）
  
- 域名结构

  `www.tmall.com`对应的真正的域名为`www.tmall.com.`。末尾的`.`称为根域名，因为每个域名都有根域名，因此我们通常省略。

  根域名的下一级，叫做"顶级域名"（top-level domain，缩写为TLD），比如`.com、.net`；

  再下一级叫做"次级域名"（second-level domain，缩写为SLD），比如`www.tmall.com`里面的`.tmall`，这一级域名是用户可以注册的；

  再下一级是主机名（host），比如`www.tmall.com`里面的`www`，又称为"三级域名"，这是用户在自己的域里面为服务器分配的名称，是用户可以任意分配的。

  那么解析流程就是**分级查询**:

  (1)先在本机的DNS里头查，如果有就直接返回了。

  (2)本机DNS里头发现没有，就去根服务器里查。根服务器发现这个域名是属于`com`域，，因此根域DNS服务器会返回它所管理的`com`域中的DNS 服务器的IP地址，意思是“虽然我不知道你要查的那个域名的地址，但你可以去`com`域问问看”
  (3)本机的DNS接到又会向`com`域的DNS服务器发送查询消息。`com` 域中也没有`www.tmall.com`这个域名的信息，和刚才一样，`com`域服务器会返回它下面的`tmall.com`域的DNS服务器的IP地址。
  以此类推，只要重复前面的步骤，就可以顺藤摸瓜找到目标DNS服务器

## 1.2 CDN工作机制

CDN（Content Delivery Network），内容分布网络，通过在现有的Internet中增加一层新的网络架构，将网络的内容发布到最接近用户的网络边缘，使用户可以就近取得所需的内容。，一般以缓存静态资源为主

## 1.3  HTTPS

由于 HTTP 天生**明文传输**的特性，在 HTTP 的传输过程中，任何人都有可能从中截获、修改或者伪造请求发送，所以可以认为 HTTP 是不安全的；在 HTTP 的传输过程中不会验证通信方的身份，因此 HTTP 信息交换的双方可能会遭到伪装，也就是**没有用户验证**；在 HTTP 的传输过程中，接收方和发送方并**不会验证报文的完整性**

HTTPS则利用传输层安全性(TLS)或安全套接字层(SSL)对通信协议加密，提供了三个关键指标

- **加密(Encryption)**， HTTPS 通过对数据加密来使其免受窃听者对数据的监听，这就意味着当用户在浏览网站时，没有人能够监听他和网站之间的信息交换，或者跟踪用户的活动，访问记录等，从而窃取用户信息。
- **数据一致性(Data integrity)**，数据在传输的过程中不会被窃听者所修改，用户发送的数据会完整的传输到服务端，保证用户发的是什么，服务器接收的就是什么。
- **身份认证(Authentication)**，是指确认对方的真实身份，也就是证明你是你（可以比作人脸识别），它可以防止中间人攻击并建立用户信任。

TLS 在根本上使用`对称加密`和 `非对称加密` 两种形式。

- 对称加密就是指**加密和解密时使用的密钥都是同样的密钥**。只要保证了密钥的安全性，那么整个通信过程也就是具有了机密性，比如AES（Advanced Encryption Standard(高级加密标准)）
- 非对称加密：非对称加密中有两个密钥，一个是公钥，一个是私钥，公钥进行加密，私钥进行解密。公开密钥可供任何人使用，私钥只有你自己能够知道。常见算法有RSA

**RSA 的运算速度非常慢，而 AES 的加密速度比较快，而 TLS 正是使用了这种混合加密方式**

HTTPS 在**内容传输的加密上使用的是对称加密，非对称加密只作用在证书验证阶段。**

### 为什么数据传输是用对称加密？

首先，**非对称加密的加解密效率是非常低的**，而 http 的应用场景中通常端与端之间存在大量的交互，非对称加密的效率是无法接受的；

另外，在 HTTPS 的场景中只有服务端保存了私钥，**一对公私钥只能实现单向的加解密**，所以 HTTPS 中内容传输加密采取的是对称加密，而不是非对称加密。

![HTTPS](D:\Program Files\笔记\image\HTTPS.JPG)

基本过程如下：

1） SSL 客户端通过 TCP 和服务器建立连接之后（443 端口），并且在一般的 tcp 连接协商（握
手）过程中请求证书。即客户端发出一个消息给服务器，这个消息里面包含了自己可实现的算法列表和 其它一些需要的消息，SSL 的服务器端会回应一个数据包，这里面确定了这次通信所需要的算法，然后 服务器向客户端返回证书。（证书里面包含了服务器信息：域名。申请证书的公司，公共秘 钥）。

2）Client 在收到服务器返回的证书后，判断签发这个证书的公共签发机构，并使用这个机构的公 共秘钥确认签名是否有效，客户端还会确保证书中列出的域名就是它正在连接的域名。

3） 如果确认证书有效，那么**客户端生成对称秘钥并使用服务器的公共秘钥进行加密。然后发送给服务 器，服务器使用它的私钥对它进行解密**，这样两台计算机可以开始进行对称加密进行通信。

HTTPS连接建立过程使用非对称加密，后续通信过程使用对称加密，减少耗时所带来的性能损耗。其中，**对称加密加密的是实际的数据，非对称加密加密的是对称加密所需要的客户端的密钥。**

# 2.Spring分析

## 2.1 核心组件

Core，Context，Bean是最核心的三大组件，如果将bean比作舞台的演员，那么Context就是舞台背景，Core就是演出的道具。Context就是要发现每个Bean之间的关系，为他们建立并维护这种关系。Core就是发现和维护这种关系所需要的工具。

- bean

  对于使用者而言，对于bean的定义、创建、解析，只需要关心bean的创建就可以了。

  bean的创建是典型的工厂模式，顶级接口BeanFactoty

  bean的定义由BeanDefinition描述，描述了在配置文件中<bean/>节点所有的信息，包括子节点。

- context

  ApplicationContext是context的顶级父类，同时也继承了BeanFactoty，起到标识应用环境，创建bean对象，保存对象关系，捕获事件的功能

- core

## 2.2 事务失效原因

-  **数据库引擎不支持事务**

- **没有被 Spring 管理** 

- **方法不是 public 的**

- **自身调用问题**

  比如

  ```java
  @Service
  public class OrderServiceImpl implements OrderService {
  
      public void update(Order order) {
          updateOrder(order);
      }
  
      @Transactional
      public void updateOrder(Order order) {
          // update order
      }
  
  }
  ```

- **数据源没有配置事务管理器**

- **异常被吃了**

  ```java
  public class OrderServiceImpl implements OrderService {
  
      @Transactional
      public void updateOrder(Order order) {
          try {
              // update order
          } catch {
  
          }
      }
  }
  ```

  把异常吃了，然后又不抛出来，事务无法回滚！

- **异常类型错误**

  ```java
  public class OrderServiceImpl implements OrderService {
  
      @Transactional
      public void updateOrder(Order order) {
          try {
              // update order
          } catch {
              throw new Exception("更新错误");
          }
      }
  
  }
  ```

  这样事务也是不生效的，因为默认回滚的是：RuntimeException，如果你想触发其他异常的回滚，需要在注解上配置一下，如：

  ```
  @Transactional(rollbackFor = Exception.class)
  ```

  这个配置仅限于 `Throwable` 异常类及其子类。

  **补充知识：7种事务传播机制**

  - REQUIRED

  如果当前方法有事务则加入事务，没有则创建一个事务。

  - NOT_SUPPORTED

  不支持事务，如果当前有事务则挂起事务运行。

  - REQUIREDS_NEW

  新建一个事务并在这个事务中运行，如果当前存在事务就把当前事务挂起。新建的事务的提交与回滚一挂起事务没有联系，不会影响挂起事务的操作。

  - MANDATORY

  强制当前方法使用事务运行，如果当前没有事务则抛出异常。

  - NEVER

  当前方法不能存在事务，即非事务状态运行，如果存在事务则抛出异常。

  - SUPPORTS

  支持当前事务，如果当前没事务也支持非事务状态运行。

  - NESTED

  如果当前存在事务，则在嵌套事务内执行。嵌套事务的提交与回滚与父事务没有任务关系，反之，当父事务提交嵌套事务也一起提交，父事务回滚会也回滚嵌套事务的。如果当前没有事务，则新建一个事务运行，这时候则与PROPAGATION_REQUIRED场景一致。

## 2.3 spring MVC

![1589167702094](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1589167702094.png)

# 3.补充

## 3.1 jsonp原理

原理：动态添加一个<script>标签，使用 script 标签的 src 属性没有跨域的限制的特点实现跨域。首先在客户端注册一个 callback, 然后把 callback 的名字传给服务器。此时，服务器先生成 json 数 据。 然后以 javascript 语法的方式，生成一个 function , function 名字就是传递上来的参数 jsonp。最后将 json 数据直接以入参的方式，放置到 function 中，这样就生成了一段 js 语法的文档，返回给客户端。 客户端浏览器，解析 script 标签，并执行返回的 javascript 文档，此时数据作为参数，传入到了客户端预先定义好的 callback 函数里。

## 3.2 ThreadLocal

#### 内存泄漏

```java
static class ThreadLocalMap {
	static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                //强引用
                value = v;
            }
        }
}
```



当 Thread 一直在运行始终不结束，强引用就不会被回收，存在以下调用链，Thread-->ThreadLocalMap-->Entry(key为null)-->value，因为调用链中的 value 和 Thread 存在强引用，所以 value 无法被回收，就有可能出现 OOM。

**避免内存泄漏，使用完 ThreadLocal 后，要调用 remove() 方法（会扫描到 key 为 null 的 Entry，并且把对应的 value 设置为 null，这样 value 对象就可以被回收。）。**

#### 空指针异常

如果 **get 方法返回值为基本类型，则会报空指针异常，如果是包装类型就不会出错**。这是因为基本类型和包装类型存在装箱和拆箱的关系，造成空指针问题的原因在于使用者。