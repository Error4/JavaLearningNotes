本篇文章主要会讲解 Feign 性能优化的问题，例如Gzip压缩、HTTP连接池、请求超时等。

# 1.Gzip压缩

## 1.1 概述

​		gzip 介绍：gzip 是一种**数据格式**，采用 deflate 算法压缩数据；gzip 是一种流行的文件压缩算法，应用十分广泛，尤其是在 Linux 平台。

　　gzip 能力：当 Gzip 压缩一个**纯文本文件**时，效果是非常明显的，大约可以**减少 70％** 以上的文件大小。

　　gzip 作用：网络数据经过压缩后实际上**降低了网络传输的字节数**，最明显的好处就是可以**加快网页加载的速度**。网页加载速度加快的好处不言而喻，除了节省流量，改善用户的浏览体验外，另一个潜在的好处是 Gzip 与搜索引擎的抓取工具有着更好的关系。

## 1.2 HTTP协议中关于压缩传输的规定

​		客户端向服务器请求中带有：`Accept-Encoding:gzip`，`deflate` 字段，向服务器表示客户端支持的压缩格式（gzip 或者 deflate），如果不发送该消息头，服务端默认是不会压缩的。

​		服务端在收到请求之后，如果发现请求头中含有 `Accept-Encoding` 字段，并且支持该类型压缩，就会对响应报文压缩之后返回给客户端，并且携带 `Content-Encoding:gzip` 消息头，表示响应报文是根据该格式进行压缩的。

​		客户端接收到请求之后，先判断是否有 `Content-Encoding` 消息头，如果有，按该格式解压报文。否则按正常报文处理。

## 1.3 压缩案例

### 局部配置

　　只配置 Consumer 通过 Feign 到 Provider 的请求与相应的 Gzip 压缩。

　　服务消费者 application.yml

```yaml
# Feign gzip 压缩
feign:
  compression:
    request:
      mime-types: text/xml,application/xml,application/json # 配置压缩支持的 MIME TYPE
      min-request-size: 512                                 # 配置压缩数据大小的最小阈值，默认 2048
      enabled: true                                         # 请求是否开启 gzip 压缩
    response:
      enabled: true                                         # 响应是否开启 gzip 压缩
```

### **全局**配置

　　对客户端浏览器的请求以及 Consumer 对 Provider 的请求与响应都实现 Gzip 压缩。

　　服务消费者 application.yml

```yaml
server:
  compression:
    # 是否开启压缩
    enabled: true
    # 配置压缩支持的 MIME TYPE
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
```

# 2.HTTP连接池

​		我们知道，建立 HTTP 连接的过程是很复杂的一个过程，涉及到多个数据包的交换，很耗时间，比如建立TCP连接时，需要的三次握手和四次挥手开销就比较大。

​		**采用 HTTP 连接池**，可以节约大量的 3 次握手 4 次挥手，这样能大大提升吞吐量。

　　Feign 的 HTTP 客户端支持 3 种框架：`HttpURLConnection`、`HttpClient`、`OkHttp`；默认是 `HttpURLConnection`。可以通过查看源码 `org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration.java` 得知。

- 传统的 HttpURLConnection 是 JDK 自带的，并不支持连接池，如果要实现连接池的机制，还需要自己来管理连接对象。

- HttpClient 相比传统 JDK 自带的 HttpURLConnection，它封装了访问 HTTP 的请求头，参数，内容体，响应等等；它不仅使客户端发送 HTTP 请求变得容易，而且也方便了开发人员测试接口（基于 HTTP 协议的），既提高了开发的效率，又提高了代码的健壮性

  那么如何使用呢？

  首先，在对应消费者项目中，添加对应依赖，因为本文中使用的Hoxton.SR1版本已经默认集成了 apache httpclient 依赖，所以只需要添加一个依赖即可

  ```xml
  <!-- 当前版本已经默认集成了 apache httpclient 依赖 -->
  <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
      <version>4.5.11</version>
  </dependency>
  <!-- feign apache httpclient 依赖 -->
  <dependency>
      <groupId>io.github.openfeign</groupId>
      <artifactId>feign-httpclient</artifactId>
      <version>10.7.4</version>
  </dependency>
  ```

  然后，配置文件中开启即可

  ```yaml
  feign:
    httpclient:
      enabled: true # 开启 httpclient
  ```

  **注意：**如果使用 HttpClient 作为 Feign 的客户端工具。那么在定义接口上的注解是需要注意的，如果传递的参数是一个自定义的对象（对象会使用 JSON 格式来专递），需要配置参数类型，例如：`@GetMapping(value = "/single/pojo", consumes = MediaType.APPLICATION_JSON_VALUE)`。**本文中使用的 Spring CLoud 版本，已无需手动配置。**

# 3.请求超时

​		Feign 的负载均衡底层用的就是 Ribbon，所以这里的请求超时配置其实就是配置 Ribbon。

　　在服务压力比较大的情况下，可能处理服务的过程需要花费一定的时间，而**默认请求超时的配置是 1s** 所以我们需要调整该配置延长请求超时时间。

​		在消费者端配置文件添加：

```yaml
ribbon:
  ConnectTimeout: 5000 # 请求连接的超时时间 默认的时间为 1 秒
  ReadTimeout: 5000    # 请求处理的超时时间
```



​		