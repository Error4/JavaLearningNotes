## 1.分布式查询与相关性算分

- 分布式搜索的运行机制

  - Query

    用户发出搜索请求到ES节点，节点收到请求后，会以Coordinating Node的身份，在所有主副分片中，随机选择（比如有6个，就会选择3个），发送查询请求

    选中的分片进行查询，**排序**，然后，每个分片都会返回**From+Size**个排序后的文档ID和排序值给Coordinating Node

  - Fetch

    Coordinating Node会将从其他分片获取的值，重新排序，选取From到From+Size个文档的ID

    以multi get请求的方式，到相应分片获取详细数据

  - 潜在的问题

    性能问题，比如每个分片都需要查询From+Size个文档，最终的Coordinating Node还需要处理number_of_shard*（From+Size）

    ![](https://s1.ax1x.com/2020/05/03/JzpERA.jpg)

- 相关性算分问题

  ​		每个分片都是基于自己的分片进行相关度计算，会导致分数偏离的情况，尤其是数据量很少时，主片分数越多，相关性算分越不准，解决方案如下：

  - 在数据量不大时，可以将主分片数设为1
  - 数据量大时，尽量保证数据均匀分布
  - 在URL中制定参数“_search?search_type=dfs_query_then_fetch”，会把每个分片的词频和文档频率搜集后，完整的进行一次算分。但会耗费更多的CPU和内存，一般不建议使用

## 2.排序

​		ES默认使用相关性算分对结果进行**降序排序**

​		可以设定sorting参数，自行设定排序，如果不指定_score,算分为null

![ES-sort](D:\Program Files\笔记\image\ES-sort.JPG)

​		排序是针对文档原始内容排序，倒排索引无法使用。使用正排索引，通过文档ID和字段快速得到文档原始内容

ES中有两种实现方法：

- Fielddata
- Doc Values(列式存储，对Text类型无效)

![](https://s1.ax1x.com/2020/05/03/JzpeMt.jpg)

​		明确不需要做排序和聚合分析时，可通过mapping设置关闭Doc Values，增加索引速度，减少磁盘空间，但重新打开就需要重建索引

```
PUT xxx/_mapping
{
	"properties"：{
		“xxx”：{
			“doc_values”：false
		}
	}
}
```

## 3. 分页与遍历

​	默认情况下，按照相关性算分排序后，返回前10条

- 深度分页问题

  由于ES的分布式，当有一个查询From=990,size=10时

  - 会在每个分片上都获取前1000个文档，然后通过Coordinating Node聚合所有结果，最后在通过排序取前1000个文档
  - 页数越深，就会占用越多的内存，**故ES规定，默认限定到10000个文档**

- Search  After 避免深度分页问题

  可以实时获取下一页文档信息，**但不支持指定页数（From），只能往下翻**

  第一步搜索需要制定sort,并且需要保证唯一(可以通过加入_id保证)

  然后使用上一次，最后一个文档的sort值查询

  **就是通过唯一的这个排序值，将每次在分片上处理的文档数控制在制定的size个**

  举例：文档内容如下

  ```json
  POST user/_doc
  {
    "name":"user1",
    "age":10
  }
  POST user/_doc
  {
    "name":"user2",
    "age":20
  }
  POST user/_doc
  {
    "name":"user3",
    "age":30
  }
  POST user/_doc
  {
    "name":"user4",
    "age":40
  }
  ```

  执行搜索，按年龄排序，并加入_id保证唯一

  ```json
  POST users/_search
  {
      "size": 10,
      "query": {
          "match" : {
              "title" : "elasticsearch"
          }
      },
      "sort": [
          {"age": "asc"},
          {"_id": "desc"}
      ]
  }
  ```

  ​		上面的请求会为每一个文档返回一个包含sort排序值的数组。这些sort排序值可以被用于 search_after 参数里以便抓取下一页的数据。

  ```json
      "hits" : [
        {
          "_index" : "user",
          "_type" : "_doc",
          "_id" : "sYpD2HEBzZv4JvprWBrO",
          "_score" : null,
          "_source" : {
            "name" : "user1",
            "age" : 10
          },
          "sort" : [
            10,
            "sYpD2HEBzZv4JvprWBrO"
          ]
        }
      ]
  ```

  然后，我们可以使用最后的一个文档的sort排序值，将它传递给 search_after 参数,进行下一步查询

  ```json
  POST user/_search
  {
      "size": 1,
      "query": {
        "match_all": {}
      },
      "search_after":[
            10,
            "sYpD2HEBzZv4JvprWBrO"
          ],
      "sort": [
          {"age": "asc"},
          {"_id": "desc"}
      ]
  }
  ```

- 遍历Scroll

  指定scroll存活的时间，创建一个快照，但有新的数据写入以后，无法被查到。如下，创建一存活时间为5分钟的快照

  ```json
  POST user/_search?scroll=5m
  {
      "size": 1,
      "query": {
        "match_all": {}
      }
  }
  ```

  查询后，返回的结果会包含_scroll_id

  ```
  "_scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAB7YWSHlDLW1XeFVTMXVhT1N1TXVYM3J6QQ==",
  ```

  每次查询后，输入上一次的_scroll_id，`"scroll":"1m"`表示开启scroll1分钟

  ```
  POST _search/scroll
  {
    "scroll":"1m",
    "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAACgEWSHlDLW1XeFVTMXVhT1N1TXVYM3J6QQ=="
  }
  ```

## 4. 并发读写

​		ES使用乐观并发控制，如果更新一个文档，会将旧文档标记为删除，同时增加一个全新的文档，并将version+1

​		可以通过添加额外的属性来控制：`if_seq_no+if_peimary_term`

同样，创建一个文档

```json
POST user/_doc/1
{
  "name":"user1",
  "age":10
}
```

结果显示：

```json
{
  "_index" : "user",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

```

更新数据

```json
PUT user/_doc/1?if_seq_no=0&&if_primary_term=1
{
   "name":"user1",
  "age":20
}
```

更新之后，if_seq_no改变，其他线程如果任然按照之前的if_seq_no更新，调用失败

```json
{
  "error": {
    "root_cause": [
      {
        "type": "version_conflict_engine_exception",
        "reason": "[1]: version conflict, required seqNo [0], primary term [1]. current document has seqNo [1] and primary term [1]",
        "index_uuid": "n7G3bZsOSkmbF8yo-hKA2w",
        "shard": "0",
        "index": "user"
      }
    ],
    "type": "version_conflict_engine_exception",
    "reason": "[1]: version conflict, required seqNo [0], primary term [1]. current document has seqNo [1] and primary term [1]",
    "index_uuid": "n7G3bZsOSkmbF8yo-hKA2w",
    "shard": "0",
    "index": "user"
  },
  "status": 409
}
```

## 5. 性能优化

- 存储设备

  Elasticsearch 重度使用磁盘，你的磁盘能处理的吞吐量越大，你的节点就越稳定，比如使用SSD

- 调整配置参数

  - 给每个文档指定有序的具有压缩良好的序列模式ID，避免随机的UUID-4 这样的 ID，这样的ID压缩比很低，会明显拖慢 Lucene。
  - 对于那些**不需要聚合和排序的索引字段禁用Doc values**。Doc Values是有序的基于document => field value的映射列表；
  - **不需要做模糊检索的字段使用 keyword类型代替 text 类型**，这样可以避免在建立索引前对这些文本进行分词。
  - 如果你的搜索结果**不需要近实时的准确度，考虑把每个索引的 index.refreshinterval 改到 30s** 。如果你是在做大批量导入，导入期间你可以通过设置这个值为 -1 关掉刷新，还可以通过设置 index.numberof_replicas: 0关闭副本。别忘记在完工的时候重新开启它。
  - **避免深度分页查询建议使用Scroll进行分页查询**。普通分页查询时，会创建一个from + size的空优先队列，每个分片会返回from + size 条数据，默认只包含文档id和得分score给协调节点，如果有n个分片，则协调节点再对（from + size）× n 条数据进行二次排序，然后选择需要被取回的文档。当from很大时，排序过程会变得很沉重占用CPU资源严重。
  - **减少映射字段，只提供需要检索，聚合或排序的字段。其他字段可存在其他存储设备上**，例如Hbase，在ES中得到结果后再去Hbase查询这些字段。
  - **创建索引和查询时指定路由routing值，这样可以精确到具体的分片查询，提升查询效率**。路由的选择需要注意数据的分布均衡。

- JVM调优

  - **确保堆内存最小值（ Xms ）与最大值（ Xmx ）的大小是相同的，防止程序在运行时改变堆内存大小**。Elasticsearch 默认安装后设置的堆内存是 1 GB。可通过 `../config/jvm.option`文件进行配置，但是最好不要超过物理内存的50%和超过32GB。
  - **GC 默认采用CMS**的方式，并发但是有STW的问题，可以考虑使用G1收集器。
  - ES非常依赖文件系统缓存（Filesystem Cache），快速搜索。一般来说，应该至少确保**物理上有一半的可用内存分配到文件系统缓存。**

## 6. 集群搭建

在同一个子网内，只需要在每个节点上设置相同的集群名,elasticsearch就会自动的把这些集群名相同的节点组成一个集群。

PS：注意清除ES目录下的data数据文件

更改每个节点的配置文件elasticsearch.yml，主要属性如下：

```yml
###节点集群名称,保证三台服务器相同
cluster.name: myes 　
#### 实际服务器ip地址,即本机地址
network.host: 192.168.212.180 　　
#### 每个节点名称不一样 其他两台为node-1 ,node-2
node.name: node-1 　　
## 服务端口号，同一服务器下必须不一致
http.port：9200
## 集群通信端口号，同一服务器下必须不一致
transport.tcp.port:9300
##多个服务集群ip
discovery.zen.ping.unicast.hosts: ["192.168.212.184", "192.168.212.185","192.168.212.186"]　　
```

逐个启动各个节点，集群自动搭建