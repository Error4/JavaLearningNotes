zookeeper的 JavaAPI

POM依赖

```html
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.10</version>
</dependency>
```

`zonde `是 `zookeeper `集合的核心组件，` zookeeper API` 提供了一小组使用 `zookeeper `集群来操作`znode `的所有细节

客户端应该遵循以下步骤，与`zookeeper`服务器进行清晰和干净的交互

- 连接到`zookeeper`服务器。`zookeeper`服务器为客户端分配会话`ID`
- 定期向服务器发送心跳。否则，`zookeeper `服务器将过期会话`ID`，客户端需要重新连接
- 只要会话`Id`处于活动状态，就可以获取/设置`znode`
- 所有任务完成后，断开与`zookeeper`服务器连接，如果客户端长时间不活动，则`zookeeper`服务器将自动断开客户端

# 1.连接到Zookeeper

```java
Zookeeper(String connectionString, int sessionTimeout, watcher watcher)
```

- `connectionString` - `zookeeper `主机地址
- `sessionTimeout `- 会话超时时间
- `watcher` - 实现"监听器" 对象。`zookeeper`集合通过监视器对象返回连接状态

```java
public static void main(String[] args) throws IOException, InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);

        ZooKeeper zookeeper = new ZooKeeper("1192.168.124.20:2181", 5000, (WatchedEvent x) -> {
            if (x.getState() == Watcher.Event.KeeperState.SyncConnected) {
                System.out.println("连接成功");
                countDownLatch.countDown();
            }
        });
        countDownLatch.await();
        System.out.println(zookeeper.getSessionId());
        zookeeper.close();
    }
```

## 新增节点

```java
// 同步
create(String path, byte[] data, List<ACL> acl, CreateMode createMode)
// 异步
create(String path, byte[] data, List<ACL> acl, CreateMode createMode,
      AsynCallback.StringCallback callBack, Object ctx)
```

参数说明如下：

| 参数         | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| `path`       | `znode `路径                                                 |
| `data`       | 数据                                                         |
| `acl`        | 要创建的节点的访问控制列表。`zookeeper API `提供了一个静态接口 `ZooDefs.Ids` 来获取一些基本的`acl`列表。例如，`ZooDefs.Ids.OPEN_ACL_UNSAFE`返回打开`znode`的`acl`列表 |
| `createMode` | 节点的类型，这是一个枚举类型                                 |
| `callBack`   | 异步回调接口                                                 |
| `ctx`        | 传递上下文参数                                               |

示例：

创建一持久化节点，并对所有用户赋予所有权限

```java
    public static void createTest1() throws Exception{
        String str = "node";
        String s = zookeeper.create("/node", str.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        System.out.println(s);
    }
```

自定义权限类型，其中，通过Id类制定授权类型

```java
//world类型    
public static void createTest2() throws Exception{
    ArrayList<ACL> acls = new ArrayList<>();
    Id id = new Id("world","anyone");
    acls.add(new ACL(ZooDefs.Perms.READ,id));
    acls.add(new ACL(ZooDefs.Perms.WRITE,id));
    zookeeper.create("/node","node".getBytes(),acls,CreateMode.PERSISTENT);
}
```

```java
//IP类型    
public static void createTest2() throws Exception{
    ArrayList<ACL> acls = new ArrayList<>();
    Id id = new Id("ip","192.168.0.10");
    acls.add(new ACL(ZooDefs.Perms.READ,id));
    zookeeper.create("/node","node".getBytes(),acls,CreateMode.PERSISTENT);
}
```

```java
// auth类型
    public static void createTest2() throws  Exception{
        //添加授权用户,用户名wyf,密码123456
        zookeeper.addAuthInfo("digest","wyf:12345".getBytes());
        rrayList<ACL> acls = new ArrayList<>();
    	Id id = new Id("auth","wyf");
    	acls.add(new ACL(ZooDefs.Perms.READ,id));
        zookeeper.create("/node","node".getBytes(),
               acls,CreateMode.PERSISTENT);
    }
```

异步方式创建

```java
public static void createTest4() throws  Exception{
        zookeeper.create("/node", "node".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT, new AsyncCallback.StringCallback(){
            /**
             * @param rc 状态，0 则为成功
             * @param path 路径
             * @param ctx 上下文参数
             * @param name 节点名称，通常与path一致
             */
            public void processResult(int rc, String path, Object ctx, String name){
                System.out.println(rc + "——" + path + "——" + name + "——" + ctx);
            }
        }, "I am context");
        TimeUnit.SECONDS.sleep(1);
        System.out.println("结束");
    }
```

## 修改节点

同样也有两种修改方式

```java
// 同步
setData(String path, byte[] data, int version)
// 异步
setData(String path, byte[] data, int version, StatCallback callBack, Object ctx)
```

参数说明

| 参数       | 解释                                                         |
| ---------- | ------------------------------------------------------------ |
| `path`     | 节点路径                                                     |
| `data`     | 数据                                                         |
| `version`  | 数据的版本号， -`1`代表不使用版本号，乐观锁机制              |
| `callBack` | 异步回调 `AsyncCallback.StatCallback`，和之前的回调方法参数不同，这个可以获取节点状态 |
| `ctx`      | 传递上下文参数                                               |

示例说明：

```java
    public static void setData1() throws Exception{
    	// arg1:节点的路径
        // arg2:修改的数据
        // arg3:数据的版本号 -1 代表版本号不参与更新
        Stat stat = zookeeper.setData("/hadoop","hadoop-1".getBytes(),-1);
    }
```



```java
    public static void setData2() throws Exception{
        zookeeper.setData("/hadoop", "hadoop-1".getBytes(), 3 ,new AsyncCallback.StatCallback(){
            @Override
            public void processResult(int rc, String path, Object ctx, Stat stat) {
                System.out.println(rc + " " + path + " " + stat.getVersion() +  " " + ctx);
            }
        }, "I am context");
    }
```

## 删除节点

```java
// 同步
delete(String path, int version)
// 异步
delete(String path, int version, AsyncCallback.VoidCallback callBack, Object ctx)
```

参数说明

| 参数       | 解释                                            |
| ---------- | ----------------------------------------------- |
| `path`     | 节点路径                                        |
| `version`  | 版本                                            |
| `callBack` | 数据的版本号， -`1`代表不使用版本号，乐观锁机制 |
| `ctx`      | 传递上下文参数                                  |

示例：

```java
    public static void deleteData1() throws Exception {
        zookeeper.delete("/hadoop", 1);
    }

    public static void deleteData2() throws Exception {
        zookeeper.delete("/hadoop", 1, new AsyncCallback.VoidCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx) {
                System.out.println(rc + " " + path + " " + ctx);
            }
        }, "I am context");
        TimeUnit.SECONDS.sleep(1);
    }
```

## 查看节点

```java
// 同步
getData(String path, boolean watch, Stat stat)
getData(String path, Watcher watcher, Stat stat)
// 异步
getData(String path, boolean watch, DataCallback callBack, Object ctx)
getData(String path, Watcher watcher, DataCallback callBack, Object ctx)
```

参数说明

| 参数       | 解释                             |
| ---------- | -------------------------------- |
| `path`     | 节点路径                         |
| `boolean`  | 是否使用连接对象中注册的监听器   |
| `stat`     | 元数据                           |
| `callBack` | 异步回调接口，可以获得状态和数据 |
| `ctx`      | 传递上下文参数                   |

示例：

```java
    public static void getData1() throws Exception {
        Stat stat = new Stat();
        byte[] data = zookeeper.getData("/hadoop", false, stat);
        System.out.println(new String(data));
        System.out.println(stat.getCtime());
    }

    public static void getData2() throws Exception {
        zookeeper.getData("/hadoop", false, new AsyncCallback.DataCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx, byte[] bytes, Stat stat) {
                System.out.println(rc + " " + path
                                   + " " + ctx + " " + new String(bytes) + " " + 
                                   stat.getCzxid());
            }
        }, "I am context");
        TimeUnit.SECONDS.sleep(3);
    }
```

## 查看子节点

```java
// 同步
getChildren(String path, boolean watch)
getChildren(String path, Watcher watcher)
getChildren(String path, boolean watch, Stat stat)    
getChildren(String path, Watcher watcher, Stat stat)
// 异步
getChildren(String path, boolean watch, ChildrenCallback callBack, Object ctx)    
getChildren(String path, Watcher watcher, ChildrenCallback callBack, Object ctx)
getChildren(String path, Watcher watcher, Children2Callback callBack, Object ctx)    
getChildren(String path, boolean watch, Children2Callback callBack, Object ctx)
```

参数说明：

| 参数       | 解释                           |
| ---------- | ------------------------------ |
| `path`     | 节点路径                       |
| `boolean`  | 是否使用连接对象中注册的监听器 |
| `callBack` | 异步回调，可以获取节点列表     |
| `ctx`      | 传递上下文参数                 |

示例：

```java
    public static void getChildren_1() throws Exception{
        List<String> hadoop = zookeeper.getChildren("/hadoop", false);
        hadoop.forEach(System.out::println);
    }

    public static void getChildren_2() throws Exception {
        zookeeper.getChildren("/hadoop", false, new AsyncCallback.ChildrenCallback() {
            @Override
            public void processResult(int rc, String path, Object ctx, List<String> list) {
                list.forEach(System.out::println);
                System.out.println(rc + " " + path + " " + ctx);
            }
        }, "I am children");
        TimeUnit.SECONDS.sleep(3);
    }
```

## 检查节点是否存在

```java
// 同步
exists(String path, boolean watch)
exists(String path, Watcher watcher)
// 异步
exists(String path, boolean watch, StatCallback cb, Object ctx)
exists(String path, Watcher watcher, StatCallback cb, Object ctx)
```

参数说明

| 参数       | 解释                           |
| ---------- | ------------------------------ |
| `path`     | 节点路径                       |
| `boolean`  | 是否使用连接对象中注册的监听器 |
| `callBack` | 异步回调，可以获取节点列表     |
| `ctx`      | 传递上下文参数                 |

示例：

```java
public static void exists1() throws Exception{
    Stat exists = zookeeper.exists("/hadoopx", false);
    // 判空
    System.out.println(exists.getVersion() + "成功");
}
public static void exists2() throws Exception{
    zookeeper.exists("/hadoopx", false, new AsyncCallback.StatCallback() {
        @Override
        public void processResult(int rc, String path, Object ctx, Stat stat) {
            // 判空
            System.out.println(rc + " " + path + " " + ctx +" " + stat.getVersion());
        }
    }, "I am children");
    TimeUnit.SECONDS.sleep(1);
}
```

# 2.事件监听机制

## **watcher概念**

- `zookeeper`提供了数据的`发布/订阅`功能，多个订阅者可同时监听某一特定主题对象，当该主题对象的自身状态发生变化时例如节点内容改变、节点下的子节点列表改变等，会实时、主动通知所有订阅者
- `zookeeper`采用了 `Watcher`机制实现数据的发布订阅功能。该机制在被订阅对象发生变化时会异步通知客户端，因此客户端不必在 `Watcher`注册后轮询阻塞，从而减轻了客户端压力
- `watcher`机制事件上与观察者模式类似，也可看作是一种观察者模式在分布式场景下的实现方式

## watcher架构

`watcher`实现由三个部分组成

- `zookeeper`服务端
- `zookeeper`客户端
- 客户端的`ZKWatchManager对象`

客户端**首先将 `Watcher`注册到服务端**，同时将 `Watcher`对象**保存到客户端的`watch`管理器中**。当`Zookeeper`服务端监听的数据状态发生变化时，服务端会**主动通知客户端**，接着客户端的 `Watch`管理器会**触发相关 `Watcher`**来回调相应处理逻辑，从而完成整体的数据 `发布/订阅`流程

![zookeeper-6](assets/zookeeper-6.png)

## watcher特性

| 特性           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| 一次性         | `watcher`是**一次性**的，一旦被触发就会移除，再次使用时需要重新注册 |
| 客户端顺序回调 | `watcher`回调是**顺序串行**执行的，只有回调后客户端才能看到最新的数据状态。一个`watcher`回调逻辑不应该太多，以免影响别的`watcher`执行 |
| 轻量级         | `WatchEvent`是最小的通信单位，结构上只包含**通知状态、事件类型和节点路径**，并不会告诉数据节点变化前后的具体内容 |
| 时效性         | `watcher`只有在当前`session`彻底失效时才会无效，若在`session`有效期内快速重连成功，则`watcher`依然存在，仍可接收到通知； |

## **watcher接口设计**

`Watcher`是一个接口，任何实现了`Watcher`接口的类就算一个新的`Watcher`。`Watcher`内部包含了两个枚举类：`KeeperState`、`EventType`

![zookeeper-7](assets/zookeeper-7.png)

### Watcher通知状态(KeeperState)

`KeeperState`是客户端与服务端**连接状态**发生变化时对应的通知类型。路径为`org.apache.zookeeper.Watcher.EventKeeperState`，是一个枚举类，其枚举属性如下：

| 枚举属性        | 说明                     |
| --------------- | ------------------------ |
| `SyncConnected` | 客户端与服务器正常连接时 |
| `Disconnected`  | 客户端与服务器断开连接时 |
| `Expired`       | 会话`session`失效时      |
| `AuthFailed`    | 身份认证失败时           |

### Watcher事件类型(EventType)

`EventType`是**数据节点`znode`发生变化**时对应的通知类型。**`EventType`变化时`KeeperState`永远处于`SyncConnected`通知状态下**；当`keeperState`发生变化时，`EventType`永远为`None`。其路径为`org.apache.zookeeper.Watcher.Event.EventType`，是一个枚举类，枚举属性如下：

| 枚举属性              | 说明                                                        |
| --------------------- | ----------------------------------------------------------- |
| `None`                | 无                                                          |
| `NodeCreated`         | `Watcher`监听的数据节点被创建时                             |
| `NodeDeleted`         | `Watcher`监听的数据节点被删除时                             |
| `NodeDataChanged`     | `Watcher`监听的数据节点内容发生更改时(无论数据是否真的变化) |
| `NodeChildrenChanged` | `Watcher`监听的数据节点的子节点列表发生变更时               |

注意：客户端接收到的相关事件通知中只包含状态以及类型等信息，不包含节点变化前后的具体内容，变化前的数据需业务自身存储，变化后的数据需要调用`get`等方法重新获取

### 捕获相应的事件

上面讲到`zookeeper`客户端连接的状态和`zookeeper`对`znode`节点监听的事件类型，下面我们来说如何建立`zookeeper`的***`watcher`监听***。在`zookeeper`中采用`zk.getChildren(path,watch)、zk.exists(path,watch)、zk.getData(path,watcher,stat)`这样的方式来为某个`znode`注册监听 。

下表以`node-x`节点为例，说明调用的注册方法和可用监听事件间的关系：

| 注册方式                            | created | childrenChanged | Changed | Deleted |
| ----------------------------------- | ------- | --------------- | ------- | ------- |
| `zk.exists("/node-x",watcher)`      | 可监控  |                 | 可监控  | 可监控  |
| `zk.getData("/node-x",watcher)`     |         |                 | 可监控  | 可监控  |
| `zk.getChildren("/node-x",watcher)` |         | 可监控          |         | 可监控  |

### 示例

##### 监听客户端与服务器端的连接状态

```java
public class ZkConnectionWatcher implements Watcher {
    @Override
    public void process(WatchedEvent watchedEvent) {
        Event.KeeperState state = watchedEvent.getState();
        if(state == Event.KeeperState.SyncConnected){
            // 正常
            System.out.println("正常连接");
        }else if (state == Event.KeeperState.Disconnected){
            // 可以用Windows断开虚拟机网卡的方式模拟
            // 当会话断开会出现，断开连接不代表不能重连，在会话超时时间内重连可以恢复正常
            System.out.println("断开连接");
        }else if (state == Event.KeeperState.Expired){
            // 没有在会话超时时间内重新连接，而是当会话超时被移除的时候重连会走进这里
            System.out.println("连接过期");
        }else if (state == Event.KeeperState.AuthFailed){
            // 在操作的时候权限不够会出现
            System.out.println("授权失败");
        }
        countDownLatch.countDown();
    }
    private static final String IP = "192.168.124.20:2181";
    private static CountDownLatch countDownLatch = new CountDownLatch(1);

    public static void main(String[] args) throws Exception {
        // 5000为会话超时时间
        ZooKeeper zooKeeper = new ZooKeeper(IP, 5000, new ZkConnectionWatcher());
        countDownLatch.await();
        // 模拟授权失败
        zooKeeper.addAuthInfo("digest1","itcast1:123451".getBytes());
        byte[] data = zooKeeper.getData("/hadoop", false, null);
        System.out.println(new String(data));
        TimeUnit.SECONDS.sleep(50);
    }
}
```

##### 监听节点变化

**exists**

```java
//是否使用连接对象的监视器
exists(String path, boolean b)
//使用自定义监视器
exists(String path, Watcher w)
```

可以捕获NodeCreated，NodeDeleted，NodeDataChanged这三种事件类型

```java
    private static final String IP = "192.168.124.20:2181";
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
    private static ZooKeeper zooKeeper;

    // 采用zookeeper连接创建时的监听器
    public static void exists1() throws Exception{
        zooKeeper.exists("/watcher1",true);
    }
    // 自定义监听器
    public static void exists2() throws Exception{
        zooKeeper.exists("/watcher1",(WatchedEvent w) -> {
            System.out.println("自定义" + w.getType());
        });
    }
    // 使用多次的监听器。由于zooKeeper设置的监听只生效一次，如果在接收到事件后还想继续对该节点的数据内容改变进行监听，需要在事件处理逻辑中重新调用exists方法
    public static void exists3() throws Exception{
        zooKeeper.exists("/watcher1", new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                try {
                    System.out.println("自定义的" + watchedEvent.getType());
                } finally {
                    try {
                        zooKeeper.exists("/watcher1",this);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        });
    }
    // 演示一节点注册多个监听器
    public static void exists4() throws Exception{
        zooKeeper.exists("/watcher1",(WatchedEvent w) -> {
            System.out.println("自定义1" + w.getType());
        });
        zooKeeper.exists("/watcher1", new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                try {
                    System.out.println("自定义2" + watchedEvent.getType());
                } finally {
                    try {
                        zooKeeper.exists("/watcher1",this);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        });
    }
    // 测试
    public static void main(String[] args) throws Exception {
        zooKeeper = new ZooKeeper(IP, 5000, new ZKWatcher());
        countDownLatch.await();
        exists1();
        TimeUnit.SECONDS.sleep(50);
    }

    static class ZKWatcher implements Watcher{
        @Override
        public void process(WatchedEvent watchedEvent) {
            if (Event.KeeperState.SyncConnected == watchedEvent.getState()) {
                if (Event.EventType.None == watchedEvent.getType() && null == event.getPath()) {
                    countDownLatch.countDown();
                    System.out.println("Zookeeper session established");
                } else if (Event.EventType.NodeCreated == watchedEvent.getType()) {
                    System.out.println("success create znode");

                } else if (Event.EventType.NodeDataChanged == watchedEvent.getType()) {
                    System.out.println("success change znode: " + watchedEvent.getPath());

                } else if (Event.EventType.NodeDeleted == watchedEvent.getType()) {
                    System.out.println("success delete znode");

                } else if (Event.EventType.NodeChildrenChanged == watchedEvent.getType()) {
                    System.out.println("NodeChildrenChanged");
                }
            }
        }
    }
}
```

另外还有getData与getChildren都能添加监听，方法如下，其用法与exists类似，不在赘述。

**getData**

```java
getData(String path, boolean b, Stat stat)
getData(String path, Watcher w, Stat stat)
```

**getChildren**

```java
getChildren(String path, boolean b)
getChildren(String path, Watcher w)
```

# 3. curator

`curator`是`Netflix`公司开源的一个 `zookeeper`客户端，后捐献给 `apache`,，`curator`框架在`zookeeper`原生`API`接口上进行了包装，解决了很多`zooKeeper`客户端非常底层的细节开发。提供`zooKeeper`各种应用场景(比如:分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等的抽象封装，实现了`Fluent`风格的APl接口，是最好用，最流行的`zookeeper`的客户端。

原生`zookeeperAPI`的不足

- 连接对象异步创建，需要开发人员自行编码等待
- 连接没有自动重连超时机制
- watcher一次注册生效一次
- 不支持递归创建树形节点

`curator`特点

- 解决`session`会话超时重连
- `watcher`反复注册
- 简化开发`api`
- 遵循`Fluent`风格`API`

## POM依赖

Maven依赖(本文使用curator的版本：2.12.0，对应Zookeeper的版本为：3.4.x，**如果跨版本会有兼容性问题，很有可能导致节点操作失败**)：

```xml
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>2.12.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>2.12.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-client</artifactId>
        <version>2.12.0</version>
    </dependency>
```

其中，作用如下：

**curator-framework：**对zookeeper的底层api的一些封装

**curator-client：**提供一些客户端的操作，例如重试策略等

**curator-recipes：**封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式计数器、分布式Barrier等

## 创建会话

- **使用静态工程方法创建客户端**

一个例子如下：

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.newClient(
						connectionInfo,
						5000,
						3000,
						retryPolicy);
```

newClient静态工厂方法包含四个主要参数：

| 参数名              |                           说明                            |
| :------------------ | :-------------------------------------------------------: |
| connectionString    |         服务器列表，格式host1:port1,host2:port2,…         |
| retryPolicy         | 重试策略,内建有四种重试策略,也可以自行实现RetryPolicy接口 |
| sessionTimeoutMs    |            会话超时时间，单位毫秒，默认60000ms            |
| connectionTimeoutMs |          连接创建超时时间，单位毫秒，默认60000ms          |

- **使用Fluent风格的Api创建会话**

核心参数变为流式设置，一个例子如下：

```java
  RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.builder()
		.connectString(connectionInfo)
		.sessionTimeoutMs(5000)
		.connectionTimeoutMs(5000)
		.retryPolicy(retryPolicy)
    	.namespace("base")
		.build();
```

补充说明：

**namespace命名空间**

为了实现不同的Zookeeper业务之间的隔离，需要为每个业务分配一个独立的命名空间（**NameSpace**），即指定一个Zookeeper的根路径（官方术语：***为Zookeeper添加“Chroot”特性***）。例如上文的例子，当客户端指定了独立命名空间为“/base”，那么该客户端对Zookeeper上的数据节点的操作都是基于该目录进行的。通过设置Chroot可以将客户端应用与Zookeeper服务端的一课子树相对应，在多个应用共用一个Zookeeper集群的场景下，这对于实现不同应用之间的相互隔离十分有意义。

**session重连策略**

- `RetryPolicy retry Policy = new RetryOneTime(3000);`

  说明：三秒后重连一次，只重连一次

- `RetryPolicy retryPolicy = new RetryNTimes(3,3000);`

  说明：每三秒重连一次，重连三次

- `RetryPolicy retryPolicy = new RetryUntilElapsed(10000,3000);`

  说明：每三秒重连一次，总等待时间超过`10`秒后停止重连

- `RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000,3)`

  说明：这个策略的重试间隔会越来越长,具体间隔公式如下：

  ```java
  baseSleepTImeMs * Math.max(1,random.nextInt(1 << (retryCount + 1)))
  //baseSleepTimeMs = 1000 例子中的值
  //retryCount = 3 例子中的值
  ```

## 启动客户端

当创建会话成功，得到client的实例然后可以直接调用其start( )方法：

```java
client.start();
```

## 数据节点操作

### 创建数据节点

**Zookeeper的节点创建模式**：

- PERSISTENT：持久化
- PERSISTENT_SEQUENTIAL：持久化并且带序列号
- EPHEMERAL：临时
- EPHEMERAL_SEQUENTIAL：临时并且带序列号

**创建一个节点，初始内容为空**

```java
client.create().forPath("path");
```

注意：如果没有设置节点属性，节点创建模式默认为持久化节点，内容默认为空

**创建一个节点，附带初始化内容**

```java
client.create().forPath("path","init".getBytes());
```

**创建一个节点，指定创建模式（临时节点），内容为空**

```java
client.create().withMode(CreateMode.EPHEMERAL).forPath("path");
```

**创建一个节点，指定创建模式（临时节点），附带初始化内容**

```java
client.create().withMode(CreateMode.EPHEMERAL).forPath("path","init".getBytes());
```

**创建一个节点，指定创建模式（临时节点），附带初始化内容，并且自动递归创建父节点**

```java
client.create()
      .creatingParentContainersIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .forPath("path","init".getBytes());
```

这个creatingParentContainersIfNeeded()接口非常有用，因为一般情况开发人员在创建一个子节点必须判断它的父节点是否存在，如果不存在直接创建会抛出NoNodeException，使用creatingParentContainersIfNeeded()之后Curator能够自动递归创建所有所需的父节点。

### 删除数据节点

**删除一个节点**

```java
client.delete().forPath("path");
```

注意，此方法只能删除**叶子节点**，否则会抛出异常。

**删除一个节点，并且递归删除其所有的子节点**

```java
client.delete().deletingChildrenIfNeeded().forPath("path");
```

**删除一个节点，强制指定版本进行删除**

```java
client.delete().withVersion(10086).forPath("path");
```

**删除一个节点，强制保证删除**

```java
client.delete().guaranteed().forPath("path");
```

guaranteed()接口是一个保障措施，只要客户端会话有效，那么Curator会在后台持续进行删除操作，直到删除节点成功。

**注意：**上面的多个流式接口是可以自由组合的，例如：

```java
client.delete().guaranteed().deletingChildrenIfNeeded().withVersion(10086).forPath("path");
```

### 读取数据节点数据

**读取一个节点的数据内容**

```java
client.getData().forPath("path");
```

注意，此方法返的返回值是byte[ ];

**读取一个节点的数据内容，同时获取到该节点的stat**

```java
Stat stat = new Stat();client.getData().storingStatIn(stat).forPath("path");
```

### 更新数据节点数据

**更新一个节点的数据内容**

```java
client.setData().forPath("path","data".getBytes());
```

注意：该接口会返回一个Stat实例

**更新一个节点的数据内容，强制指定版本进行更新**

```java
client.setData().withVersion(10086).forPath("path","data".getBytes());
```

### 检查节点是否存在

```java
client.checkExists().forPath("path");
```

注意：该方法返回一个Stat实例，用于检查ZNode是否存在的操作. 可以调用额外的方法(监控或者后台处理)并在最后调用`forPath()`指定要操作的ZNode

### 获取某个节点的所有子节点路径

```java
client.getChildren().forPath("path");
```

注意：该方法的返回值为List,获得ZNode的子节点Path列表。 可以调用额外的方法(监控、后台处理或者获取状态watch, background or get stat) 并在最后调用forPath()指定要操作的父ZNode

### 事务

CuratorFramework的实例包含inTransaction( )接口方法，调用此方法开启一个ZooKeeper事务. 可以复合create, setData, check, and/or delete 等操作然后调用commit()作为一个原子操作提交。一个例子如下：

```java
client.inTransaction().check().forPath("path")
      .and()
      .create().withMode(CreateMode.EPHEMERAL).forPath("path","data".getBytes())
      .and()
      .setData().withVersion(10086).forPath("path","data2".getBytes())
      .and()
      .commit();
```

### 异步接口

上面提到的创建、删除、更新、读取等方法都是同步的，Curator提供异步接口，引入了**BackgroundCallback**接口用于处理异步接口调用之后服务端返回的结果信息。**BackgroundCallback**接口中一个重要的回调值为CuratorEvent，里面包含事件类型、响应吗和节点的详细信息。

**CuratorEventType**

| 事件类型 | 对应CuratorFramework实例的方法 |
| :------: | :----------------------------: |
|  CREATE  |           #create()            |
|  DELETE  |           #delete()            |
|  EXISTS  |         #checkExists()         |
| GET_DATA |           #getData()           |
| SET_DATA |           #setData()           |
| CHILDREN |         #getChildren()         |
|   SYNC   |      #sync(String,Object)      |
| GET_ACL  |           #getACL()            |
| SET_ACL  |           #setACL()            |
| WATCHED  |       #Watcher(Watcher)        |
| CLOSING  |            #close()            |

**响应码(#getResultCode())**

| 响应码 |                   意义                   |
| :----: | :--------------------------------------: |
|   0    |              OK，即调用成功              |
|   -4   | ConnectionLoss，即客户端与服务端断开连接 |
|  -110  |        NodeExists，即节点已经存在        |
|  -112  |        SessionExpired，即会话过期        |

一个异步创建节点的例子如下：

```java
Executor executor = Executors.newFixedThreadPool(2);
client.create()
      .creatingParentsIfNeeded()
      .withMode(CreateMode.EPHEMERAL)
      .inBackground((curatorFramework, curatorEvent) -> {      System.out.println(String.format("eventType:%s,resultCode:%s",curatorEvent.getType(),curatorEvent.getResultCode()));
      },executor)
      .forPath("path");
```

注意：如果#inBackground()方法不指定executor，那么会默认使用Curator的EventThread去进行异步处理。