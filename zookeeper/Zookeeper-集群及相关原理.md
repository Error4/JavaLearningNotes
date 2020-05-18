为了保证Zookeeper的高可用，一般在生产中都要将Zookeeper配置为集群使用。下面将会对集群搭建的过程和相关原理做进一步说明。

# 1.集群搭建

对于集群搭建，官网文档说明连接如下：https://zookeeper.apache.org/doc/r3.4.14/zookeeperAdmin.html#sc_zkMulitServerSetup

**说明：在Zookeeper集群中，若超过半数以上服务节点不可用，才会造成整个服务不可用，所以其集群节点数一般都是至少3个节点以上的奇数个**

首先，Zookeeper配置文件默认包含以下内容

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
```

说明如下：

- tickTime =2000：通信心跳数，Zookeeper 服务器与客户端心跳时间，单位毫秒。 Zookeeper使用的基本时间，服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime时间就会发送一个心跳，并且设置最小的session超时时间为两倍心跳时间。
- initLimit =5：LF 初始通信时限 集群中的Follower跟随者服务器与Leader领导者服务器之间初始连接时能容忍的最多心跳数（tickTime的数量），用它来限定集群中的Zookeeper服务器连接到Leader的时限。
- syncLimit =2：LF 同步通信时限 集群中Leader与Follower之间的最大响应时间单位，假如响应超过syncLimit *tickTime，Leader认为Follwer死掉，从服务器列表中删除Follwer。 
- dataDir：数据文件目录+数据持久化路径 主要用于保存 Zookeeper 中的数据。 
- clientPort =2181：客户端连接端口 监听客户端连接的端口。

## 修改配置文件

搭建集群，只需要在配置文件中加入各节点信息即可，格式如下：

```
server.A=B:C:D
```

其中，

- A是一个数字，表示这个是第几号服务器； 集群模式下配置一个文件myid，这个文件在 dataDir 目录下，这个文件里面有一个数据就是 A 的值，Zookeeper 启动时读取此文件，拿到里面的数据与 zoo.cfg 里面的配置信息比较从而判断到底是哪个 server。 
- B 是这个服务器的地址；
- C是这个服务器 Follower 与集群中的 Leader 服务器交换信息的端口；
- D 是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口

比如我们要搭建三个节点的集群，就可以在每个节点的配置文件中加入如下配置

```
server.1=192.168.0.10:2888:3888
server.2=192.168.0.11:2888:3888
server.3=192.168.0.12:2888:3888
```

## 添加myid文件

分别在对应zookeeper目录下创建myid 的文件，在文件中添加server 对应的编号：1,2,3（与上述配置文件中的设置**server.A**对应）。然后分别启动对应不同的实例即可

# 2.相关原理

现在，集群已经搭建好了，那么集群是如何工作的呢？

## 2.1 ZAB协议 

`zab`协议的全称是 ***`Zookeeper Atomic Broadcast`*** (`zookeeper`原子广播)。`zookeeper`是通过`zab`协议来保证最终一致性

基于`zab`协议，`zookeeper`集群中的角色主要有以下三类

- Leader
  一个 Zookeeper 集群同一时间只会有一个实际工作的 Leader，它会发起并维护与各 Follwer 及
  Observer 间的心跳。所有的写操作必须要通过Leader 完成再由 Leader 将写操作广播给其它服务 器。只要有超过半数节点（不包括 observeer 节点）写入成功，该写请求就会被提交（类 2PC 协 议）。
- Follower
  一个Zookeeper 集群可能同时存在多个 Follower，它会响应 Leader 的心跳，Follower 可直接处 理并返回客户端的读请求，同时会将写请求转发给 Leader 处理，并且负责在 Leader 处理写请求 时对请求进行投票
- Observer
  角色与 Follower 类似，但是无投票权。Zookeeper 需保证高可用和强一致性，为了支持更多的客 户端，需要增加更多 Server；Server 增多，投票阶段延迟增大，影响性能；引入 Observer， Observer 不参与投票； Observers 接受客户端的连接，并将写请求转发给 leader 节点； 加入更 多Observer 节点，提高伸缩性，同时不影响吞吐率。

在 ZAB 的 协议的事务编号 Zxid 设计中，Zxid 是一个 64 位的数字，其中低 32 位是一个简单的单调递增的计数器，**针对客户端每 一个事务请求，计数器加 1；而高 32 位则代表 Leader 周期 epoch 的编号，每个当选产生一个新 的 Leader 服务器，就会从这个 Leader 服务器上取出其本地日志中最大事务的ZXID，并从中读取 epoch 值，然后加 1，以此作为新的 epoch，并将低 32 位从 0 开始计数。** 

Zxid（Transaction id）类似于 RDBMS 中的事务 ID，用于标识一次更新操作的 Proposal（提议） ID。为了保证顺序性，该 zkid必须单调递增。

 epoch：可以理解为当前集群所处的年代或者周期，每个 leader 就像皇帝，都有自己的年号，所 以每次改朝换代，leader 变更之后，都会在前一个年代的基础上加 1。这样就算**旧的 leader 崩溃 恢复之后，也没有人听他的了，因为 follower 只听从当前年代的 leader 的命令。**

 Zab协议有两种模式，它们分别是**恢复模式（选主）和广播模式（同步）**。当服务启动或者在领导 者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server 完成了和 leader 的状 态同步以后，恢复模式就结束了。

ZAB协议 可以分为如下4个 阶段 

1. Leader election（选举阶段）：节点在一开始都处于选举阶段，只要有一个节点得到超半数 节点的票数，它就可以当选准 leader。只有到达 广播阶段（broadcast） 准 leader 才会成 为真正的 leader。这一阶段的目的是就是为了选出一个准 leader，然后进入下一个阶段。
2. Discovery（发现阶段）：在这个阶段，followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。这个一阶段的主要目的是发现当前大多数节点接收的最新提议，并且 准 leader 生成新的 epoch，让 followers 接受，更新它们的 acceptedEpoch 。一个 follower 只会连接一个 leader，如果有一个节点 f 认为另一个 follower p 是 leader，f 在尝试连接 p 时会被拒绝，f 被拒绝之后，就会进入重新选举阶段。
3. Synchronization（同步阶段）：同步阶段主要是利用 leader 前一阶段获得的最新提议历史， 同步集群中所有的副本。只有当 大多数节点都同步完成，准 leader 才会成为真正的 leader。 follower 只会接收 zxid 比自己的 lastZxid 大的提议。
4. Broadcast（广播阶段）：到了这个阶段，Zookeeper 集群才能正式对外提供事务服务， 并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。
   ZAB 提交事务并不像 2PC 一样需要全部 follower 都 ACK，只需要得到超过半数的节点的 ACK 就 可以了。

## 2.2 observer角色及其配置

在上文中我们提到Zookeeper集群中，除了常规的主从节点外，还额外引入了observer节点，这里对其做进一步说明。

尽管`ZooKeeper`通过使客户端直接连接到该集合的投票成员而表现良好，但是此体系结构使其很难扩展到大量客户端。问题在于，随着我们添加更多的投票成员，写入性能会下降。这是由于以下事实：写操作需要（通常）集合中至少一半节点的同意，因此，随着添加更多的投票者，投票的成本可能会显着增加。

我们引入了一种称为`Observer`的新型`ZooKeeper`节点，该节点有助于解决此问题并进一步提高`ZooKeeper`的可伸缩性。`Observer`仅能听取投票结果，但没有投票权。除了这种简单的区别之外，`Observer`的功能与`Follower`的功能完全相同：客户端可以连接到`Observer`，并向其发送读写请求。`Observer`像`Follower`一样将这些请求转发给`Leader`，但是他们只是等待听取投票结果。因此，我们可以在不影响投票效果的情况下尽可能增加`Observer`的数量。

`Observer`还有其他优点。因为他们不投票，所以它们不是`ZooKeeper`集群中的关键部分。因此，它们可以在不损害`ZooKeeper`服务可用性的情况下发生故障或与群集断开连接。给用户带来的好处是，观察者可以通过比跟随者更不可靠的网络链接进行连接。

`ovserver`角色**特点**：

1. **不参与集群的`leader`选举**
2. **不参与集群中写数据时的`ack`反馈**

为了使用`observer`角色，在任何想变成`observer`角色的配置文件中加入如下配置：

```shell
peerType=observer
```

并在所有`server`的配置文件中，配置成`observer`模式的`server`的那行配置追加***`:observer`***，例如

```shell
server.1=192.168.133.133:2287:3387  # 注意端口号  
server.2=192.168.133.133:2288:3388
server.3=192.168.133.133:2289:3389:observer
```

## 2.3 选举机制

Zk的选举算法有两种：一种是基于basic paxos实现的，另外一种是基于fast paxos算法实现的

- basic paxos

  每个 sever 首先给自己投票，然后用自己的选票和其他 sever 选票对比，权重大的胜出，使用权 重较大的更新自身选票箱。具体选举过程如下：

1. 每个`server`发出一个投票。由于是初始状态，`server1`和`server2`都会将自己作为`leader`服务器来进行投票，每次投票都会包**含所推举的`myid`和`zxid`，使用(`myid，zxid`)**，此时`server1`的投票为(1，0)，`server2`的投票为(2，0)，然后**各自将这个投票发给集群中的其它机器**

2. 集群中的**每台服务器都接收来自集群中各个服务器的投票**

3. **处理投票**。针对每一个投票，服务器都需要将别人的投票和自己的投票进行pk，规则如下

   - 优先检查`zxid`。`zxid`比较大的服务器优先作为`leader`(**`zxid`较大者保存的数据更多**)

   - 如果`zxid`相同。那么就比较`myid`。`myid`较大的服务器作为`leader`服务器

     **对于`Server1`而言，它的投票是(1，0)**，接收`Server2`的投票为(2，0)，**首先会比较两者的`zxid`**，均为0，**再比较`myid`**，此时`server2`的`myid`最大，于是更新自己的投票为(2，0)，然后重新投票，**对于server2而言，无序更新自己的投票**，只是再次向集群中所有机器发出上一次投票信息即可

4. **统计投票**。每次投票后，服务器都会统计投票信息，判断是否已经有**过半机器接受到相同的投票信息**，对于`server1、server2`而言，都统计出集群中已经有两台机器接受了(2，0)的投票信息，此时便认为已经选举出了`leader`

5. **改变服务器状态**。一旦确定了`leader`,每个服务器就会更新自己的状态，如果是`follower`，那么就变更为`following`，如果是`leader`，就变更为`leading`

- fast paxos

  选举过程中，某Server首先向所有Server提议自己要成为leader，当其它Server收到提议以后，解决epoch和 zxid的冲突，并接受对方的提议，然后向对方发送接受提议完成的消息，重复这个流程，最后一定能选举出Leader。

## 2.4 原理总结

1. Zookeeper 的核心是原子广播，这个机制保证了各个 server 之间的同步。实现这个机制 的协议叫做 Zab协议。Zab 协议有两种模式，它们分别是恢复模式和广播模式。
2. 当服务启动或者在领导者崩溃后，Zab 就进入了恢复模式，当领导者被选举出来，且大多 数 server 的完成了和 leader 的状态同步以后，恢复模式就结束了。
3. 状态同步保证了 leader 和 server 具有相同的系统状态 . 一旦 leader 已经和多数的 follower 进行了状态同步后，他就可以开始广播消息了，即进 入广播状态。这时候当一个 server 加入 zookeeper 服务中，它会在恢复模式下启动，发 现 leader，并和 leader 进行状态同步。待到同步结束，它也参与消息广播。Zookeeper 服务一直维持在 Broadcast 状态，直到 leader 崩溃了或者 leader 失去了大部分的 followers 支持。
4. 广播模式需要保证 proposal 被按顺序处理，因此 zk 采用了递增的事务 id 号(zxid)来保 证。所有的提议(proposal)都在被提出的时候加上了 zxid。
5. 实现中 zxid是一个 64 为的数字，它高 32 位是 epoch 用来标识 leader 关系是否改变， 每次一个 leader 被选出来，它都会有一个新的 epoch。低 32 位是个递增计数。
6. 当 leader 崩溃或者 leader 失去大多数的 follower，这时候 zk 进入恢复模式，恢复模式 需要重新选举出一个新的 leader，让所有的 server 都恢复到一个正确的状态。