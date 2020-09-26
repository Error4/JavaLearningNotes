# 1.基础的增删改查

ES利用REST接口来实现对数据的增删改查。

## 创建一个索引和文档

创建一个叫做**user**的索引（index），并插入一个文档（document)。在RDMS中，我们通常需要有专用的语句来生产相应的数据库，表格，让后才可以让我们输入相应的记录，但是针对Elasticsearch来说，这个不是必须的。

```
POST user/_doc/1
{
  "name":"A",
  "age":10,
  "city":"Beijing",
  "about":"测试"
}
```

通过如上语句，我们可以自动创建一个index。也可以配置禁用

```
PUT _cluster/settings
{
    "persistent": {
        "action.auto_create_index": "false" 
    }
}
```

ES会默认根据字段的值来配置字段类型，当然也可以在创建索引时自定义数据类型，如下：

```
PUT /user
{
  "mappings":{
    "properties":{
      "name":{
        "type":"text"
      },
      "age":{
        "type":"long"
      },
      "city":{
        "type":"text"
      },
      "about":{
        "type":"text"
      }
    }
  }
}
```

每次修改后版本信息`_version`自动加1

![](https://s1.ax1x.com/2020/09/26/0i2hHf.md.png)

还可以使用`_create`端点接口来实现，

```json
PUT user/_create/1
{
  "name":"A",
  "age":10,
  "city":"Beijing",
  "about":"测试"
}
```

如果文档已经存在的话，我们会收到一个错误的信息：

![](https://s1.ax1x.com/2020/09/26/0iRivR.md.png)

## 查看一个文档

可以使用如下命令来查看

```
GET user/_doc/1
```

![](https://s1.ax1x.com/2020/09/26/0iR1KI.md.png)

如果我们只对source的内容感兴趣的话，我们可以使用：

```
GET user/_doc/1/_source
```

也可以只获取source的部分字段：

```
GET user/_doc/1?_source=name,age
```

如果你想一次请求查找多个文档，我们可以使用_mget接口，例如同时获取id为1及2的文档。

```
GET user/_doc/_mget
{
  "ids": ["1", "2"]
}
```

## 修改一个文档

通常我们使用POST来创建一个新的文档。使用PUT来修改一个文档

```
PUT user/_doc/1
{
  "name":"A",
  "age":10,
  "city":"Beijing",
  "about":"测试修改"
}
```

使用PUT进行修改时，每次修改一个文档时，我们需要把文档的每一项都要写出来。这对于有些情况来说，并不方便，我们可以使用如下的方法来进行修改：

```
POST user/_update/1
{
  "doc": {
     "name":"B"
  }
}
```

以上都是根据文档的ID进行更新的，有些情况下，我们可能不知道ID，需要通过查询的方式来进行查询，让后进行修改，可以用如下的方式，更新`name=A`的数据的`age,about`信息

```
POST user/_update_by_query
{
  "query": {
    "match": {
      "name": "A"
    }
  },
  "script": {
    "source": "ctx._source.age = params.age;ctx._source.about = params.about",
    "lang": "painless",
    "params": {
      "age": 30,
      "about": "测试update_by_query"
    }
  }
}
```

## 检查一个文档是否存在

```
HEAD user/_doc/1
```

这个HEAD接口可以很方便地告诉我们在twitter的索引里是否有一id为1的文档：

![](https://s1.ax1x.com/2020/09/26/0iRwxs.md.png)

## 删除一个文档

删除一个文档的话，我们可以使用如下的命令：

```
DELETE user/_doc/1
```

同样，也可以通过查询的方式来进行查询，让后进行删除。ES也提供了相应的REST接口。

```
POST user/_delete_by_query
{
  "query": {
    "match": {
      "name": "A"
    }
  }
}
```

## 删除一个index

删除一个index是非常直接的。我们可以直接使用如下的命令来进行删除：

```
DELETE user
```

## 批处理命令

上面每一次操作都是一个REST请求，对于大量的数据进行操作的话，这个显得比较慢。ES创建一个批量处理的命令给我们使用。这样我们在一次的REST请求中，我们就可以完成很多的操作

```
POST _bulk
{"index" : { "_index" : "user", "_id": 1} }
{ "name":"A","age":10,"city":"Beijing","about":"测试_bulk"}
{"index" : { "_index" : "user", "_id": 2} }
{ "name":"B","age":20,"city":"Beijing","about":"测试_bulk"}
{"index" : { "_index" : "user", "_id": 3} }
{ "name":"C,"age":30,"city":"Beijing","about":"测试_bulk"}
```

在输入命令时，我们需要特别的注意：千**万不要添加除了换行以外的空格，否则会导致错误**。

查看发现创建成功

![](https://s1.ax1x.com/2020/09/26/0iRfz9.md.png)

删除和修改同理

```
POST _bulk
{ "delete" : { "_index" : "user", "_id": 1 }}
```

```
POST _bulk
{ "update" : { "_index" : "user", "_id": 2 }}
{"doc": { "name": "ABC"}}
```

# 2.中文分词

分词：即把一段中文或者别的划分成一个个的关键字，我们在搜索时候会把自己的信息进行分词，会把 数据库中或者索引库中的数据进行分词，然后进行一个匹配操作，默认的中文分词是将每个字看成一个 词，比如 “我爱中国” 会被分为"我","爱","中","国"，这显然是不符合要求的，所以我们需要安装中文分词器来解决这个问题。
以目前主流的ik分词器为例，IK提供了两个分词算法：**ik_smart 和 ik_max_word。**

ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合，适合 Term Query；

ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”，适合 Phrase 查询。

下载地址：https://github.com/medcl/elasticsearch-analysis-ik ,下载完毕之后，放入到我们的elasticsearch 安装目录下plugins文件夹，重新启动即可。

![](https://s1.ax1x.com/2020/09/26/0iR7dK.md.png)

![](https://s1.ax1x.com/2020/09/26/0iRXzd.md.png)

另外，我们也可以根据自己的需要自定义分词，比如现在有词汇`王大侠`,利用进行分词，会分为`王`和`大侠`，而我们不需要把它分开，可以在配置文件中配置

![](https://s1.ax1x.com/2020/09/26/0iW9df.md.png)

再次测试

![](https://s1.ax1x.com/2020/09/26/0iWKoT.md.png)

# 3.查询

准备数据

```
POST _bulk
{"index" : { "_index" : "user", "_id": 1} }
{ "name":"A","age":10,"city":"Beijing","about":"喜欢足球"}
{"index" : { "_index" : "user", "_id": 2} }
{ "name":"B","age":20,"city":"Chengdu","about":"喜欢篮球"}
{"index" : { "_index" : "user", "_id": 3} }
{ "name":"C","age":30,"city":"Xian","about":"喜欢羽毛球"}
```

## 主体描述

```
GET user/_search
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "A",
          "age" : 10,
          "city" : "Beijing",
          "about" : "喜欢足球"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "name" : "B",
          "age" : 20,
          "city" : "Chengdu",
          "about" : "喜欢篮球"
        }
      },
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "name" : "C",
          "age" : 30,
          "city" : "Xian",
          "about" : "喜欢羽毛球"
        }
      }
    ]
  }
}
```

主要参数意义如下：

- took: 整个搜索请求花费了多少毫秒
- timed_out:表示请求是否超时
- hits.total: value表示返回结果的总数，relation表示关系 例如一般是eq表示相等
- hits.max_score: 表示本次搜索的所有结果中，最大的相关度分数是多少，每一条document对于search的相关度，越相关，_score分数就越大，排位就越靠前
- hits.hits： 表示查询出来document的结果集合
- shards: total表示打到的所有分片，successful表示打到的分片中查询成功的分片,skipped表示打到的分片中跳过的分片,failed表示打到的分片中查询失败的分片

## multi-index搜索模式

所有索引下的所有数据都搜索出来

```http
GET /_search
```

指定一个index,搜索这个索引下的所有数据

```http
GET /test/_search
```

同时搜索两个索引下的数据

```http
GET /test_index,test/_search
```

通过通配符匹配多个索引，查询多个索引下的数据

```http
GET /test*/_search
```

代表所有的index

```http
GET /_all/_search
```

## query string 语法

对于query string只要掌握q=field:search content的语法，以及+和-的含义
+：代表包含这个筛选条件结果
-：代表不包含这个筛选条件的结果

```http
//查询索引user下name字段包含A的文档
GET /user/_search?q=+name:A
//查询索引user下name字段不包含A的文档,并且按age排序
GET /user/_search?q=-name:A&sort=age:desc
```

## query DSL

### Match query

查询所有的用户:

```
GET /user/_search
{
  "query": {
    "match_all": {}
  }
}
```

也可以定制返回字段

```
GET /user/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["name", "age"]
}

```

查询名字带有A的数据，注意match查询的时候,elasticsearch会根据你给定的字段提供合适的分析器，match查询相当于模糊匹配

```
GET /user/_search
{
  "query": {
    "match": {
    	"name":"A"
    }
  }
}
```

### Match_phrase  query

match会对输入进行分词处理后再去查询，部分命中的结果也会按照评分由高到低显示出来。而match_phrase是按短语查询，其要点有三：

1. match_phrase还是分词后去搜的
2. 目标文档需要包含分词后的所有词
3. 目标文档还要保持这些词的相对顺序和文档中的一致

```
GET /user/_search
{
  "query": {
    "match_phrase": {
    	"about":"喜欢足球"
    }
  }
}
```

### Multi_match查询

在上面的搜索之中，我们特别指明一个专有的field来进行搜索，但是在很多的情况下，我们并胡知道是哪一个field含有这个关键词，那么在这种情况下，我们可以使用multi_match来进行搜索：

```json
GET user/_search
{
    "query":{
        "multi_match":{
            "query":"足球",
            "fields":["name","about"]
        }
    }
}
```

### Prefix query

返回在提供的字段中包含特定前缀的文档。

```
GET user/_search
{
  "query": {
    "prefix": {
      "about": {
        "value": "喜"
      }
    }
  }
}
```

查询about字段里以“喜”为开头的所有文档：

### Term query 

Term query会在给定字段中进行精确的字词匹配，它并不知道分词器的存在，这种查询适合keyword、numeric、date等明确值的

```
GET user/_search
{
    "query":{
        "term":{
          "about.keyword":"喜欢足球"
        }
    }
}
```

只返回about字段为喜欢足球的数据

```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.9808291,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.9808291,
        "_source" : {
          "name" : "A",
          "age" : 10,
          "city" : "Beijing",
          "about" : "喜欢足球"
        }
      }
    ]
  }
}
```

也可以对多个terms进行查询

```
GET user/_search
{
    "query":{
        "term":{
          "about.keyword":[
          "喜欢足球",
          "喜欢篮球"
          ]
        }
    }
}
```

### 复合查询

bool查询可以实现组合过滤查询，格式：

{"bool" : {"must":[],"should":[],"must_not":[] } }

must：必须满足的条件 （相当于and）

should：可以满足也可以不满足的条件 （相当于or）

must_not：不需要满足的条件 （相当于not） 

如下，就是查询name必须包含A或者age=10的数据

```
GET  user/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "A"
          }
        }
      ],
      "should": [
        {
          "match": {
            "age": 10
          }
        }
      ]
    }
  }
}
```

另外，符合查询还支持过滤操作

```
GET  user/_search
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "age": {
            "lte": 10
          }
        }
      }
    }
  }
}
```

其中，

- gt 大于 
- gte 大于等于
-  lt 小于 
- lte 小于等于

### 高亮搜索

```
GET user/_search
{
  "query": {
    "match_phrase": {
      "about": "喜欢足球"
    }
  },
  "highlight": {
    "fields": {
      "about":{}
    }
  }
}
```

返回如下：

```
{
  "took" : 53,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.4263072,
    "hits" : [
      {
        "_index" : "user",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.4263072,
        "_source" : {
          "name" : "A",
          "age" : 10,
          "city" : "Beijing",
          "about" : "喜欢足球"
        },
        "highlight" : {
          "about" : [
            "<em>喜</em><em>欢</em><em>足</em><em>球</em>"
          ]
        }
      }
    ]
  }
}
```

## 聚合查询

聚合语法如下：

```
"aggregations" : {  //表明聚合，也可以写缩写aggs
    "<aggregation_name>" : { //聚合名字，自定义
        "<aggregation_type>" : { //聚合类型
            <aggregation_body>  //具体聚合逻辑
        }
    }

}
```

一般可以如下设置：以terms聚合为例

```
{
    "size" : 0,//不需要返回文档,所以直接设置为0.可以提高查询速度
    "aggs" : { //这个是aggregations的缩写,这边用户随意,可以写全称也可以缩写
        "popular_colors" : { //定义一个聚合的名字,与java的方法命名类似,建议用'_'线来分隔单词
            "terms" : { //定义单个桶(集合)的类型为 terms
              "field" : "color"(字段颜色进行分类,类似于sql中的group by color)
            }
        }
    }
}
```

准备数据

```
POST _bulk
{"index":{"_index":"twitter","_id":1}}
{"user":"张三","message":"今儿天气不错啊，出去转转去","uid":2,"age":20,"city":"北京","province":"北京","country":"中国","address":"中国北京市海淀区","location":{"lat":"39.970718","lon":"116.325747"}, "DOB": "1999-04-01"}
{"index":{"_index":"twitter","_id":2}}
{"user":"老刘","message":"出发，下一站云南！","uid":3,"age":22,"city":"北京","province":"北京","country":"中国","address":"中国北京市东城区台基厂三条3号","location":{"lat":"39.904313","lon":"116.412754"}, "DOB": "1997-04-01"}
{"index":{"_index":"twitter","_id":3}}
{"user":"李四","message":"happy birthday!","uid":4,"age":25,"city":"北京","province":"北京","country":"中国","address":"中国北京市东城区","location":{"lat":"39.893801","lon":"116.408986"}, "DOB": "1994-04-01"}
{"index":{"_index":"twitter","_id":4}}
{"user":"老贾","message":"123,gogogo","uid":5,"age":30,"city":"北京","province":"北京","country":"中国","address":"中国北京市朝阳区建国门","location":{"lat":"39.718256","lon":"116.367910"}, "DOB": "1989-04-01"}
{"index":{"_index":"twitter","_id":5}}
{"user":"老王","message":"Happy BirthDay My Friend!","uid":6,"age":26,"city":"北京","province":"北京","country":"中国","address":"中国北京市朝阳区国贸","location":{"lat":"39.918256","lon":"116.467910"}, "DOB": "1993-04-01"}
{"index":{"_index":"twitter","_id":6}}
{"user":"老吴","message":"好友来了都今天我生日，好友来了,什么 birthday happy 就成!","uid":7,"age":28,"city":"上海","province":"上海","country":"中国","address":"中国上海市闵行区","location":{"lat":"31.175927","lon":"121.383328"}, "DOB": "1991-04-01"}
```

### Metrics(指标)聚合

相当于MySQL的聚合函数。

#### max

```json
{
  "size": 0,
  "aggs": {
    "max_id": {
      "max": {
        "field": "id"
      }
    }
  }
}
```

#### min

```json
{
  "size": 0,
  "aggs": {
    "min_id": {
      "min": {
        "field": "id"
      }
    }
  }
}
```

#### avg

```json
{
  "size": 0,
  "aggs": {
    "avg_id": {
      "avg": {
        "field": "id"
      }
    }
  }
}
```

#### sum

```json
{
  "size": 0,
  "aggs": {
    "sum_id": {
      "sum": {
        "field": "id"
      }
    }
  }
}
```

### Bucket（桶）聚合

相当于MySQL的group by操作，注意不要尝试对es中text的字段进行桶聚合，否则会失败

####  range聚合

把用户进行年龄分段，查出来在不同的年龄段的用户：

```
GET twitter/_search
{
  "size": 0,
  "aggs": {
    "age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      }
    }
  }
}
```

#### date_range 聚合

可以使用date_range来统计在某个时间段里的文档数。如下查询出生年月（DOB）从1989-01-01到1990-01-01及从1991-01-01到1992-01-01的文档

```
GET twitter/_search
{
  "size": 0,
  "aggs": {
    "birth_range": {
      "date_range": {
        "field": "",
        "format": "yyyy-MM-dd",
        "ranges": [
          {
            "from": "1989-01-01",
            "to": "1990-01-01"
          },
          {
            "from": "1991-01-01",
            "to": "1992-01-01"
          }
        ]
      }
    }
  }
}
```

#### terms聚合

通过term聚合来查询某一个关键字出现的频率。在如下的term聚合中，我们想寻找在所有的文档出现”Happy birthday”里按照城市进行分类的一个聚合。

```
GET twitter/_search
{
  "query": {
    "match": {
      "message": "happy birthday"
    }
  },
  "size": 0,
  "aggs": {
    "city": {
      "terms": {
        "field": "city",
        "size": 10
      }
    }
  }
}
```

# 参考文章

[Elastic：菜鸟上手指南](https://blog.csdn.net/UbuntuTouch/article/details/102728604)