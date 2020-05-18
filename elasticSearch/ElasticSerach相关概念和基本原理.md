# 1.背景

## 1.1 定义

​		全文搜索引擎是Elasticsearch目前广泛应用的主流搜索引擎。它的工作原理是计算机索引程序通过扫描文章中的每一个词，对每一个词建立一个索引，指明该词在文章中出现的次数和位置

​		当用户查询时，检索程序就根据事先建立的索引进行查找，并将查找的结果反馈给用户的检索方式。这个过程类似于通过字典中的检索字表查字的过程。

## 1.2 为什么使用

- **数据类型**

  全文索引搜索支持非结构化数据的搜索，可以更好地快速搜索大量存在的任何单词或单词组的非结构化文本。

  例如 Google，百度类的网站搜索，它们都是根据网页中的关键字生成索引，我们在搜索的时候输入关键字，它们会将该关键字即索引匹配到的所有网页返回。还有常见的项目中应用日志的搜索等等。**对于这些非结构化的数据文本，关系型数据库搜索不是能很好的支持。**

- **索引的维护**

  一般传统数据库，全文检索都实现的很鸡肋，因为一般也没人用数据库存文本字段。进行全文检索需要扫描整个表，如果数据量大的话即使对SQL的语法优化，也收效甚微。建立了索引，但是维护起来也很麻烦，对于 insert 和 update 操作都会重新构建索引。

## 1.3 什么时候使用

1. 搜索的数据对象是大量的非结构化的文本数据。
2. 文件记录量达到数十万或数百万个甚至更多。
3. 支持大量基于交互式文本的查询。
4. 需求非常灵活的全文搜索查询。
5. 对高度相关的搜索结果的有特殊需求，但是没有可用的关系数据库可以满足。
6. 对不同记录类型、非文本数据操作或安全事务处理的需求相对较少的情况。

# 2.相关概念和基本原理

## 2.1 相关概念

### Cluster

​		ES天生就是一个分布式的搜索引擎，允许多台服务器协同工作，每台服务器可以运行多个 Elastic 实例。单个 Elastic 实例称为一个节点（node）。一组节点构成一个集群（cluster）。

​		一个集群由一个唯一的名字标识，**节点通过指定集群的名字来加入集群**。可以在config/elasticsearch.yml配置文件中自定义

```java
cluster.name: myes 　
```

### Node

关于Node,有以下一些概念需要注意：

- Coordinating（使协调; 使相配合） Node
  - 处理请求的节点（路由请求到正确的节点，比如创建索引的请求，路由到MASTER节点）
  - 所有节点默认都是Coordinating  Node
- Data Node
  - 保存数据，节点启动后，默认就是Data Node，可以设置node.data:false禁止。由Master Node决定如何把分片分发到数据节点上
  - 通过增加数据节点，可以解决数据水平扩展问题
- Master Node
  - 处理创建，删除索引等请求/决定分片被分配到哪个节点/维护并更新Cluster State
  - 实际使用，为一个集群配置多个Master Node
- Cluster State
  - 集群状态信息，包括所有节点信息，所有的索引和相关的mapping，分片的路由信息
  - 每个节点都会保存Cluster State，但只有Master Node可以修改，并负责同步给其他节点

​		每个节点既可以是**候选Master Node**也可以是**Data Node**，通过在配置文件 `../config/elasticsearch.yml`中设置即可，默认都为 `true`。需要强调，`node.master: true` 仅仅是代表该节点拥有**被选举为Master Node的资格**。

```yaml
node.master: true  //是否候选主节点
node.data: true    //是否数据节点
```

​		**Data Node**负责数据的存储和相关的操作，例如对数据进行增、删、改、查和聚合等操作

​		**候选Master Node**可以被选举为主节点（**Master Node**），集群中只有候选主节点才有选举权和被选举权，其他节点不参与选举的工作。主节点负责创建索引、删除索引、跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点、追踪集群中节点的状态等

### **Document**

​		Elasticsearch是**面向文档(document oriented)**的，这意味着索引或搜索的最小数据单元就是文档。	ELasticsearch使用JSON，作为文档序列化格式

### 索引

​		在Elasticsearch中存储数据的行为就叫做**索引(indexing)**。

​		Elasticsearch集群可以包含多个**索引(indices)**（数据库），每一个索引可以包含多个**类型(types)**（表），每一个类型包含多个**文档(documents)**（行），然后每个文档包含多个**字段(Fields)**（列）。类比传统关系型数据库:

```properties
Relational DB -> Databases -> Tables -> Rows -> Columns
Elasticsearch -> Indices   -> Types  -> Documents -> Fields
```

​		在7.x版本之前，可以用如下方式创建一个索引

```Javascript
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

​		我们看到path:`/megacorp/employee/1`包含三部分信息：

| 名字     | 说明         |
| -------- | ------------ |
| megacorp | 索引名       |
| employee | 类型名       |
| 1        | 这个员工的ID |

​		在7.x版本之后，ES取消了type的定义，默认使用**_doc**，在未来8.0的版本中，type将被彻底删除。

```http
PUT /megacorp/_doc/1
```

### Shard

​		一个索引可以存储超出单个结点硬件限制的大量数据，为了解决这个问题，Elasticsearch提供了将索引划分成多份的能力，这些份就叫做分片(shard)。

​		有两种类型的分片：primary shard和replica shard。

- Primary shard: 每个文档都存储在一个Primary shard。 索引文档时，它首先在Primary shard上编制索引，然后在此分片的所有副本上(replica)编制索引。索引可以包含一个或多个主分片。
- Replica shard: 每个主分片可以具有零个或多个副本。副本的作用一是提高系统的容错性，当某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高es的查询效率，es会自动对搜索请求进行负载均衡。	

​		**在旧版本中，默认Shards为5，replicas为1，在7.0之后，默认Shards为1，replicas为0**。故一般都需要在创建时设置

```json
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, 
            "number_of_replicas" : 2 
        }
    }
}
```

​		注意：**主分片的数量只能在索引创建前指定，并且索引创建后不能更改。**这是由于ES的路由算法所致

```
shard = hash(_routing) % (num_of_primary_shards)
```

​	其中，hash算法确保文档分散均匀，默认路由值_routing是文档ID，其对主分片数量取模，所以主分片数量不能修改，否则可能会找不到相应的shard。

## 2.2 基本原理

### 倒排索引不可变性

​		倒排索引不可变性

- 倒排索引Immutable Design，一旦生成，不可更改
- 优点
  - 无需考虑并发写文件的问题
  - 一旦读入内核，便会留在那里，只有有足够的空间，大部分请求都会直接请求内存
  - 缓存容易生成和维护，数据可以压缩
- 缺点：如果搜索一个新文档，则需要重建整个索引

​		由于Elasticsearch中的文档是不可变的，因此不能被删除或者改动以展示其变更。删除的文档信息保存在".del"文件中。

​		当删除请求发送后，文档并没有真的被删除，而是在.del文件中被标记为删除。该文档依然能匹配查询，但是会在结果中被过滤掉。当段合并时，在.del文件中被标记为删除的文档将不会被写入新段。
　　在新的文档被创建时，Elasticsearch会为该文档指定一个版本号，当执行更新时，旧版本的文档在.del文件中被标记为删除，新版本的文档被索引到一个新段。旧版本的文档依然能匹配查询，但是会在结果中被过滤掉

### Lucene Index

​		ES是基于Lucene开发的，在Lucene 中，**单个倒排索引文件成为Segment，多个Segment汇总在一起，成为Lucene的Index，即ES的Shard。**

​		当有新文档写入时，会生成新的Segment，查询时会查询所有的Segments，并对结果进行汇总。Lucene中有一个文件，用来记录所有Segments信息，叫做提交点（Commit Point）。

​		提交点是一个用来记录所有提交后段信息的文件。一个段一旦拥有了提交点，就说明这个段只有读的权限，失去了写的权限。相反，当段在内存中时，就只有写的权限，而不具备读数据的权限，意味着不能被检索。

### 生命周期

- Refresh

  在ES中，写入新文档时，先把文档写入自己的一个名为Index Buffer的内存空间，在内存和磁盘之间是文件系统缓存（操作系统的内存），当达到默认的时间（1秒钟）或者Index Buffer被占满（默认是JVM 10%）时，将内存中的数据生成到一个新的段上并缓存到文件缓存系统 上，这个过程就是一次Refresh。

  Refresh之后，数据就可以被检索到了。

  简单的说，Index Buffer把数据从Index Buffer写入Segment的过程就叫Refresh。频率默认1秒发生一次，可通过`index.refresh_interval`设置。

  ![](https://s1.ax1x.com/2020/05/02/JxMCxf.th.jpg)

- Transaction Log

  为了保证数据不丢失，在Index文档时，同时写入Transaction Log，Transaction Log默认落盘，每个分片都有一个Transaction Log。

  在ES Refresh时，Index Buffer会被清空，Transaction Log不会清空。

  ![](https://s1.ax1x.com/2020/05/02/JxMXlV.jpg)

  由于Transaction Log的存在，保证持久性，断电恢复后可通过Transaction Log恢复。

- Flush

  该阶段会做以下几件事：

  - 调用Refresh，清空Index Buffer；
  - 调用fsync，将文件系统中缓存中的Segment写入磁盘；
  -  清空Transaction Log；

  默认30分钟调用一次，或者Transaction Log写满时也会自动调用，默认大小512MB。

  ![](https://s1.ax1x.com/2020/05/02/JxQojK.jpg)

- Merge

  由于自动刷新流程每秒会创建一个新的Segment，这样会导致短时间内的Segment数量暴增。而Segment数目太多会带来较大的麻烦。每一个Segment都会消耗文件句柄、内存和cpu运行周期。更重要的是，每个搜索请求都必须轮流检查每个Segment然后合并查询结果，所以Segment越多，搜索也就越慢。所以，需要定期合并，并且**还能真正删除已经删除的文档（即.del文件的内容）**

  默认自动进行，也可通过命令手动执行

  ```
  POST /index/_forcemerge
  ```

### 脑裂现象：

​		如果由于网络或其他原因导致集群中选举出多个Master节点，使得数据更新时出现不一致，这种现象称之为**脑裂**，即集群中不同的节点对于master的选择出现了分歧，出现了多个master竞争。

​		“脑裂”问题可能有以下几个原因造成：

- **网络问题**：集群间的网络延迟导致一些节点访问不到master，认为master挂掉了从而选举出新的master，并对master上的分片和副本标红，分配新的主分片

- **节点负载**：主节点的角色既为master又为data，访问量较大时可能会导致ES停止响应（假死状态）造成大面积延迟，此时其他节点得不到主节点的响应认为主节点挂掉了，会重新选取主节点。

- **内存回收**：主节点的角色既为master又为data，当data节点上的ES进程占用的内存较大，引发JVM的大规模内存回收，造成ES进程失去响应。

为了避免脑裂现象的发生，我们可以从原因着手通过以下几个方面来做出优化措施：

- **适当调大响应时间，减少误判**通过参数 `discovery.zen.ping_timeout`设置节点状态的响应时间，默认为3s，可以适当调大，如果master在该响应时间的范围内没有做出响应应答，判断该节点已经挂掉了。调大参数（如6s，discovery.zen.ping_timeout:6），可适当减少误判。

- **选举触发**我们需要在候选集群中的节点的配置文件中设置参数 `discovery.zen.munimum_master_nodes`的值，这个参数表示在选举主节点时需要参与选举的候选主节点的节点数，默认值是1，官方建议取值 `(master_eligibel_nodes/2)+1`，其中 `master_eligibel_nodes`为候选主节点的个数。这样做既能防止脑裂现象的发生，也能最大限度地提升集群的高可用性，因为只要不少于discovery.zen.munimum_master_nodes个候选节点存活，选举工作就能正常进行。当小于这个值的时候，无法触发选举行为，集群无法使用，不会造成分片混乱的情况。

  **7.0版本之后，移除该参数，交由ES自己选择**

- **角色分离**即是上面我们提到的候选主节点和数据节点进行角色分离，这样可以减轻主节点的负担，防止主节点的假死状态发生，减少对主节点“已死”的误判。

### 选举机制

​		一个集群，支持配置多个Master Eligible节点，可以设置master：false禁止，这些节点在必要时会参与选主流程，成为Master节点。

​		节点互相PING，根据nodeId字典排序，每次选举每个节点都把自己所知道节点排一次序，然后选出第一个（第0位）节点，暂且认为它是master节点。

​		如果对某个节点的投票数达到一定的值（可以成为master节点数n/2+1）并且该节点自己也选举自己，那这个节点就是master。否则重新选举一直到满足上述条件。