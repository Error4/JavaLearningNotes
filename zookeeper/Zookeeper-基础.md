

# 1.简介

在[官网](https://zookeeper.apache.org/)上是这样对Zookeeper进行说明的：

```
ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them, which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.
```

概括而言，ZooKeeper是一个分布式服务框架，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。

# 2.典型应用场景说明

## 维护配置信息

`java`编程经常会遇到配置项，比如数据库的`url`、 `schema`、`user`和 `password`等。通常这些配置项我们会放置在配置文件中，再将配置文件放置在服务器上当需要更改配置项时，需要去服务器上修改对应的配置文件。

但是随着分布式系统的兴起,由于许多服务都需要使用到该配置文件,因此有**必须保证该配置服务的高可用性**(`highavailability`)和各台服务器上配置数据的一致性。

通常会将配置文件部署在一个集群上，然而一个**集群动辄上千台**服务器，此时如果再一台台服务器逐个修改配置文件那将是非常繁琐且危险的的操作，因此就**需要一种服务**，**能够高效快速且可靠地完成配置项的更改等操作**，并能够保证各配置项在每台服务器上的数据一致性。

**`zookeeper`就可以提供这样一种服务**，其使用`Zab`这种一致性协议来保证一致性。现在有很多开源项目使用`zookeeper`来维护配置，如在 `hbase`中，客户端就是连接一个 `zookeeper`，获得必要的 `hbase`集群的配置信息，然后才可以进一步操作。还有在开源的消息队列 `kafka`中，也便用`zookeeper`来维护 `brokers`的信息。在 `alibaba`开源的`soa`框架`dubbo`中也广泛的使用`zookeeper`管理一些配置来实现服务治理。

![](https://s1.ax1x.com/2020/06/22/NY4Vje.png)

## 分布式锁服务

一个集群是一个分布式系统，由多台服务器组成。为了提高并发度和可靠性，多台服务器上运行着同一种服务。当多个服务在运行时就需要协调各服务的进度，有时候需要保证当某个服务在进行某个操作时，其他的服务都不能进行该操作，即对该操作进行加锁，如果当前机器挂掉后，释放锁并 `fail over`到其他的机器继续执行该服务

## 集群管理

一个集群有时会因为各种软硬件故障或者网络故障，出现棊些服务器挂掉而被移除集群，而某些服务器加入到集群中的情况，`zookeeper`会将这些服务器加入/移出的情况通知给集群中的其他正常工作的服务器，以及时调整存储和计算等任务的分配和执行等。此外`zookeeper`还会对故障的服务器做出诊断并尝试修复。

![](https://s1.ax1x.com/2020/06/22/NY4NHs.md.png)

## 生产分布式唯一ID

在过去的单库单表型系统中，通常可以使用数据库字段自带的`auto_ increment`属性来自动为每条记录生成一个唯一的`ID`。但是分库分表后，就无法在依靠数据库的`auto_ Increment`属性来唯一标识一条记录了。此时我们就可以用`zookeeper`在分布式环境下生成全局唯一`ID`。

# 3.Zookeeper的设计目标

`zooKeeper`致力于为分布式应用提供一个高性能、高可用，且具有严格顺序访问控制能力的分布式协调服务

1. 高性能
   
   `zooKeeper`将全量数据存储在**内存**中，并直接服务于客户端的所有非事务请求，尤其用于以读为主的应用场景
2. 高可用
   
   `zooKeeper`一般以集群的方式对外提供服务，一般`3~5`台机器就可以组成一个可用的 `Zookeeper`集群了，每台机器都会在内存中维护当前的服务器状态，井且每台机器之间都相互保持着通信。只要集群中超过一半的机器都能够正常工作，那么整个集群就能够正常对外服务
3. 严格顺序访问
   
   对于来自客户端的每个更新请求，`zooKeeper`都会分配一个全局唯一的递增编号，这个编号反应了所有事务操作的先后顺序

# 4.数据模型

`zookeeper`的数据结点可以视为树状结构(或目录)，树中的各个结点被称为`znode `(即`zookeeper node`)，一个`znode`可以有多个子结点。

使用路径`path`来定位某个`znode`，比如`/ns-1/itcast/mysqml/schemal1/table1`，此处`ns-1，itcast、mysql、schemal1、table1`分别是`根结点、2级结点、3级结点以及4级结点`；其中`ns-1`是`itcast`的父结点，`itcast`是`ns-1`的子结点，`itcast`是`mysql`的父结点....以此类推

`znode`，兼具文件和目录两种特点，即像文件一样维护着数据、元信息、ACL、时间戳等数据结构，又像目录一样可以作为路径标识的一部分

![](https://s1.ax1x.com/2020/06/22/NY4cDJ.png)

一个`znode`大体上分为`3`个部分：

- 结点的数据：即`znode data `(结点`path`，结点`data`)的关系就像是`Java map `中的 `key value `关系
- 结点的子结点`children`
- 结点的状态`stat`：用来描述当前结点的创建、修改记录，包括`cZxid`、`ctime`等

其中，zooKeeper可以使用 `stat `命令查看指定路径结点的`stat`信息

```shell
[zk: localhost:2181(CONNECTED) 2] stat /test

cZxid = 0x2
ctime = Thu May 07 03:45:27 UTC 2020
mZxid = 0x2
mtime = Thu May 07 03:45:27 UTC 2020
pZxid = 0x2
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
```

属性说明：

- `cZxid`数据结点创建时的事务ID——针对于`zookeeper`数据结点的管理：我们对结点数据的一些写操作都会导致`zookeeper`自动地为我们去开启一个事务，并且自动地去为每一个事务维护一个事务`ID`
- `ctime`数据结点创建时的时间
- `mZxid`数据结点最后一次更新时的事务ID
- `mtime`数据结点最后一次更新时的时间
- `pZxid`数据节点最后一次修改此`znode`子节点更改的`zxid`
- `cversion`子结点的更改次数
- `dataVersion`结点数据的更改次数
- `aclVersion`结点的ACL更改次数——类似`linux`的权限列表，维护的是当前结点的权限列表被修改的次数
- `ephemeralOwner`如果结点是临时结点，则表示创建该结点的会话的`SessionID`；如果是持久结点，该属性值为0
- `dataLength`数据内容的长度
- `numChildren`数据结点当前的子结点个数

# 5.基本命令

## 操作结点

需要说明，在`zooKeeper`中，可以将节点分为**临时结点**和**永久结点**，再细分，还能分为有序节点和无序节点。结点的类型在创建时被确定，并且不能改变。

- 临时节点：

  该节点的生命周期依赖于创建它们的会话。一旦会话( `Session`）结束，临时节点将被自动删除，当然可以也可以手动删除。虽然每个临时的 `Znode`都会绑定到一个客户端会话，但他们对所有的客户端还是可见的。另外，`Zookeeper`的临时节点不允许拥有子节点

- 持久化结点：

  该结点的生命周期不依赖于会话，并且只有在客户端显示执行删除操作的时候，它们才能被删除

- 有序节点

  节点创建时，Zookeeper会对该节点名称进行**顺序编号**

- 无序节点

  节点创建时，Zookeeper不会对该节点名称进行顺序编号

### **创建**

创建结点并写入数据：

`create [-s] [-e] path data` # 其中 -s 为有序结点，-e 临时结点（默认是持久结点）

```shell
create /test "123456"  # 此时，如果退出后重新登入， 输入 get /test 获取，结点依然存在
				       
create -s /a "a"         # 创建一个持久化有序结点，创建的时候可以观察到返回的数据带上了一个id       

create -s /b "b"         # 继续创建持久化有序结点，会发现返回的值，id递增了

create -s -e /aa "aa"    # 依然还会返回自增的id，quit后再进来，继续创建，id依然是往后推的
```

### **查询**

`get /test`  查看结点的数据     `stat /test` 查看结点的属性，上文已经提到过。也可以直接使用

`get -s /test`，同时获取数据和属性

### 更新

更新结点的命令是`set`，可以直接进行修改，如下：

`set path [-v version]`

在旧版本中，也可以直接使用`set path [version]`

```shell
set /test "345"        # 修改结点值

set /test "345" -v 1 # 也可以基于版本号进行更改，类似于乐观锁，当传入版本号(dataVersion)和当前结点的数据版本号不一致时，zookeeper会拒绝本次修改
```

### **删除**

删除结点的语法如下：

`delete path [-v version]` 和 `set` 方法相似，也可以传入版本号

```shell
delete /test           # 删除结点
delete /test -v 1         # 乐观锁机制，与set 方法一致
```

要想删除某个结点及其所有后代结点，可以使用递归删除，命令为 `rmr path`

```shell
create /test/node1 "node1"
rmr /test
```

### **查看结点列表**

```shell
ls /test               # 可以查看其子节点的列表
ls /                   # 查看根节点下的列表
```

### 监听器

使用`get -w path` 注册的监听器能够在结点**内容发生改变**的时候，向客户端发出通知。需要注意的是`zookeeper`的触发器是一次性的(`One-time trigger`)，即触发一次后就会立即失效

```shell
get -w /test    # get 的时候添加监听器，当值改变的时候，监听器返回消息
set /test 45678        # 测试

#返回提示如下：
WATCHER::
WatchedEvent state:SyncConnected type:NodeDataChanged path:/test
```

使用 `ls -w path` 注册的监听器能够监听该结点下**所有子节点**的**增加**和**删除**操作

```shell
ls -w  /test       # 添加监听器
set /test/node  "node"
```

在旧版本中，可以使用`get path watch`和`ls path watch`达到同样的效果

更多详情可参考[官网文档](https://zookeeper.apache.org/doc/)

## 权限控制

### Zookeeper权限分类

Zookeeper ACL（Access Control List）一共分为5种权限：

| 权限   | ACL简写 | 描述               |
| ------ | ------- | ------------------ |
| ADMIN  | a       | 设置节点ACL        |
| DELETE | d       | 注意，是删除子节点 |
| CREATE | c       | 创建节点           |
| WRITE  | w       | 写节点数据         |
| READ   | r       | 读节点数据或子节点 |

与Linux权限类似，5个权限可以使用int型数字perms的5个二进制位表示，顺序为adcwr，如perms=5对应权限--cwr，perms=31对应权限adcwr。
 		一般在命令行设置权限时会使用ACL简写，在API调用时会使用perms数字表示权限。

### Zookeeper权限验证

Zookeeper权限验证方式（scheme）如下：

| 方式   | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| world  | 所有人都有权限，默认用户anyone，即没有权限控制               |
| auth   | 验证过的用户，不再验证                                       |
| ip     | 根据ip验证                                                   |
| digest | 使用用户名、密码进行验证，digest密码生成方式：明文密码进行Sha1哈希后再进行Base64编码 |

Zookeeper的默认权限为`world: anyone :adcwr`，即任何人拥有所有权限，如果将Zookeeper服务暴露给外网，不进行权限修改非常危险的。

### Zookeeper权限设置说明

- 请注意，Zookeeper节点权限是不继承的，也就是说需要对每个新创建的节点进行权限设置，否则会使用默认权限，可能会出现父节点无权限，但子节点有权限的情况。

命令行设置权限的格式：`setAcl {path} {scheme}:{id}:{auth},{scheme}:{id}:{auth}...`可以用逗号分隔

1.  `path`：节点路径
2.  `scheme`：权限验证方式
3.  `id`：使用auth验证时，形式为“用户名:密码”，如`tomcat:apache`；使用ip验证时，形式为“ip/mask”如`172.2.0.0/24`，务必同时给`127.0.0.1`加一个权限，否则本机都无法操作了
4.  `auth`：权限，对应ACL简写，如rwd等

### 命令行设置权限过程

```shell
# 登录cli
bin/zkCli.sh -server 127.0.0.1:2181

# 如果使用auth验证，必须先给当前连接加上权限，否则后续操作会因为没权限而失败！
# 格式为addauth digest {username}:{password}
addauth digest tomcat:apache

# 给节点设置权限，注意auth验证时需要使用上边digest验证的用户名:密码明文
# 格式为setAcl {path} {scheme}:{id}:{auth}
setAcl /test auth:tomcat:apache:adcwr
# 验证一下是否设置成功
getAcl /test
```

好了，现在每个节点只有拥有对应权限的连接能够进行对应操作了。

### 忘记密码或出现权限设置错误的情况

可以修改配置文件，跳过ACL检查，把权限设置正确后再打开ACL检查

```shell
# 打开配置文件
vim conf/zoo.cfg
# 写入跳过ACL
skipACL=yes
# 重启Zookeeper服务
bin/zkServer.sh restart
```




