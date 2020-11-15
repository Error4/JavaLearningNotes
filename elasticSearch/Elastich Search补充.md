Elastich Search补充

# 1.数据路由

之前我们说过ES的路由算法，shard = hash(routing) % number_of_primary_shards。routing值，默认是_id，也可以手动指定

```json
//先插入数据
PUT /test_index/_doc/11?routing=12
{
  "test_field": "test routing not _id"
}
//获取数据不带routing参数
GET /test_index/_doc/11
{
  "_index" : "test_index",
  "_type" : "_doc",
  "_id" : "11",
  "found" : false
}
//获取数据带routing参数 参数值为自定义的值
GET /test_index/_doc/11?routing=12
  "_index" : "test_index",
  "_type" : "_doc",
  "_id" : "11",
  "_version" : 1,
  "_seq_no" : 9,
  "_primary_term" : 1,
  "_routing" : "12",
  "found" : true,
  "_source" : {
    "test_field" : "test routing not _id"
  }
}
```

手动指定的routing是很有用的，可以保证某一类的document一定被路由到一个shard中去

# 3.doc value

对于ES而言，在建立索引的时候，一方面会建立**倒排索引**，以供**搜索**使用；一方面会建立**正排索引**，也就是doc values,以供**排序，聚合，过滤**等使用。

**doc values是被保存在磁盘上的，此时如果内存足够，OS操作系统会自动将其缓存在内存中**，性能还是会很高的，如果内存不够用，OS操作系统会将其写入磁盘。

# 4.bouncing results 问题

由于搜索请求是在所有有效的分片副本间轮询的，那就有可能发生主分片处理请求时，这两个文档是一种顺序， 而副本分片处理请求时又是另一种顺序。

可以设置 na参数,让同一个用户始终使用同一个分片，这样可以避免这种问题

如下，制定向分片xyzabc123查询

```
GET /test_index/_search?preference=xyzabc123
```

# 6.bool + term

elasticsearch在查询的时候底层都会转换为bool + term的形式

```json
{
    "match": {
        "title": "java elasticsearch"
    }
}
```

使用类似上面的match query进行多值搜索的时候，elasticsearch会在底层自动将这个match query转换为bool的语法

```json
{
    "bool": {
        "should": [
            {
                "term": {
                    "title": "java"
                }
            },
            {
                "term: {
                    "title": "elasticsearch"
                }
            }
        ]
    }
}
```



# 8.相关性打分

Lucene的评分叫做TF/IDF算法，基本意思就是词频算法。

- TF:TF代表分词项在某个点文档中出现的次数(term frequency)
- IDF:IDF代表代表分词项在多少个文档中出现(inverse document frequency)

# 9.多字段搜索

- ## best fields

  假设我们有一个让用户搜索博客文章的网站。其中有两个文档如下：

  ```json
  PUT /test_index/_create/1
  {
      "title": "Quick brown rabbits",
      "body":  "Brown rabbits are commonly seen."
  }
  
  PUT /test_index/_create/2
  {
      "title": "Keeping pets healthy",
      "body":  "My quick brown fox eats rabbits on a regular basis."
  }
  ```

  进行查询：

  ```json
  GET /test_index/_search
  {
      "query": {
          "bool": {
              "should": [
                  { "match": { "title": "Brown fox" }},
                  { "match": { "body":  "Brown fox" }}
              ]
          }
      }
  }
  ```

  我们发现文档1的相关度分数更高，排在了前面。文档1在两个字段中都包含了brown，因此两个match查询都匹配成功并拥有了一个分值。文档2在body字段中包含了brown以及fox，但是在title字段中没有出现任何搜索的单词。因此对body字段查询得到的高分加上对title字段查询得到的零分，然后在乘以匹配的查询子句数量1，最后除以总的查询子句数量2，导致整体分数值比文档1的低。

  相比使用bool查询，我们可以使用dis_max查询（Disjuction Max Query），意思就是返回匹配了任何查询的文档，并且分值是产生了最佳匹配的查询所对应的分值：

  ```json
  GET /test_index/_search
  {
    "query": {
      "dis_max": {
        "queries": [
          {
            "match": {
              "title": "Brown fox"
            }
          },
          {
            "match": {
              "body": "Brown fox"
            }
          }
        ]
      }
    }
  }
  ```

- ## multi_match查询

  multi_match查询提供了一个简便的方法用来对多个字段执行相同的查询。

  ```json
  GET /test_index/_search
  {
    "query": {
      "dis_max": {
        "tie_breaker": 0.3,
        "queries": [
          {
            "match": {
              "title": {
                "query": "Quick brown fox",
                "minimum_should_match": "30%"
              }
            }
          },
          {
            "match": {
              "body": {
                "query": "Quick brown fox",
                "minimum_should_match": "30%"
              }
            }
          }
        ]
      }
    }
  }
  ```

  可以通过multi_match简单地重写如下：

  ```json
  GET /test_index/_search
  {
    "query": {
        "multi_match": {
        "query": "Quick brown fox",
        "type": "best_fields",
        "fields": ["title", "body"],
        "tie_breaker": 0.3,
        "minimum_should_match": "30%"
      }
    }
  }
  ```

- ## most_fields查询

  most_fields是以字段为中心，这就使得它会查询最多匹配的字段。

  有两个文档如下：

  ```json
  PUT /test_index/_create/1
  {
      "street":   "5 Poland Street",
      "city":     "Poland",
      "country":  "United W1V",
      "postcode": "W1V 3DG"
  }
  
  PUT /test_index/_create/2
  {
      "street":   "5 Poland Street W1V",
      "city":     "London",
      "country":  "United Kingdom",
      "postcode": "3DG"
  }
  ```

  使用most_fields进行查询：

  ```json
  GET /test_index/_search
  {
    "query": {
      "multi_match": {
        "query": "Poland Street W1V",
        "type": "most_fields", 
        "fields": ["street", "city", "country", "postcode"]
      }
    }
  }
  ```

  **存在的问题：**

  它被设计用来找到匹配任意单词的多数字段，而不是找到跨越所有字段的最匹配的单词

  **解决方法：**

  **copy_to**参数将多个field组合成一个field

  建立索引

  ```json
  PUT /test_index
  {
    "mappings": {
      "properties": {
        "street": {
          "type": "text",
          "copy_to": "full_address"
        },
        "city": {
          "type": "text",
          "copy_to": "full_address"
        },
        "country": {
          "type": "text",
          "copy_to": "full_address"
        },
        "postcode": {
          "type": "text",
          "copy_to": "full_address"
        },
        "full_address": {
          "type": "text"
        }
      }
    }
  }
  ```

  插入之前的数据

  查询：

  ```json
  GET /test_index/_search
  {
    "query": {
      "match": {
        "full_address": "Poland Street W1V"
      }
    }
  }
  ```

  就会看到跨越所有字段的最匹配的单词的文档1排在前面

# 10 近似搜索

假设有两个句子

```
java is my favourite programming langurage, and I also think spark is a very good big data system.

java spark are very related, because scala is spark's programming langurage and scala is also based on jvm like java. 
```

适用match query 搜索java spark

```
{
    {
        "match": {
            "content": "java spark"
        }
    }
}
```

match query 只能搜索到包含java和spark的document,但是不知道java和spark是不是离得很近。
假设我们想要java和spark离得很近的document优先返回，就要给它一个更高的relevance score,这就涉及到了proximity match近似匹配

使用**match_phrase**

```
GET /test_index/_search
{
  "query": {
    "match_phrase": {
      "content": "java spark"
    }
  }
}
```

**slop**

含义：query string搜索文本中的几个term,要经过几次移动才能与一个document匹配，这个移动的次数就是slop。
实际举一个例子：
对于hello world, java is very good, spark is also very good. 假设我们要用match phrase 匹配到java spark。可以发现直接进行查询会查不到

```json
PUT /test_index/_create/1
{
  "content": "hello world, java is very good, spark is also very good."
}

GET /test_index/_search
{
  "query": {
    "match_phrase": {
      "content": "java spark"
    }
  }
}
```

此时使用

```json
GET /_analyze
{
  "text": ["hello world, java is very good, spark is also very good."],
  "analyzer": "standard"
}
```

结果：

```json
{
  "tokens" : [
    {
      "token" : "hello",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "world",
      "start_offset" : 6,
      "end_offset" : 11,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "java",
      "start_offset" : 13,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "is",
      "start_offset" : 18,
      "end_offset" : 20,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "very",
      "start_offset" : 21,
      "end_offset" : 25,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "good",
      "start_offset" : 26,
      "end_offset" : 30,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "spark",
      "start_offset" : 32,
      "end_offset" : 37,
      "type" : "<ALPHANUM>",
      "position" : 6
    }
```

可以发现java的position是2，spark的position是6，那么我们只需要设置slop大于等于3（也就是移动3词就可以了）就可以搜到了

```json
GET /test_index/_search
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "java spark",
        "slop": 3
      }
    }
  }
}
```

# 11  match和match_phrase实现召回率和精准度的平衡

对于Elasticsearch而言，当使用match查询的时候
**召回率=匹配到的文档数量/所有文档的数量，所以匹配到的文档数量越多，召回率就越高。**
**准确度指的就是匹配到的文档中，我们真正查询想要的文档相关度分数越高，返回结果中排在越前面，准确度就越高。**

如果我们的搜索文本是java spark，那么在返回结果中只要包含java或者是spark的文档就返回，但是如果文档既包含java也包含spark，并且距离非常近，那么这样的文档分数会非常高，会在结果中优先被返回。

**用bool组合match和match_phrase,来实现，must条件中用match,保证尽量匹配更多的结果，should中用match_phrase来提高我们想要的文档的相关度分数，让这些文档优先返回。**

```json
GET /test_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "test_field": "java spark"
          }
        }
      ],
      "should": [
        {
          "match_phrase": {
            "test_field": {
              "query": "java spark",
              "slop": 10
            }
          }
        }
      ]
    }
  }
}
```

# 12 前缀搜索、通配符搜索、正则搜索

### 准备数据：

```json
PUT /test_index/_create/1
{
  "test_field": "C3D0-KD345"
}
PUT /test_index/_create/2
{
  "test_field": "C3K5-DFG65"
}
PUT /test_index/_create/3
{
  "test_field": "C4I8-UI365"
}
```

### 前缀搜索

搜索前缀为C3的文档：

```json
GET /test_index/_search
{
  "query": {
    "match_phrase_prefix": {
      "test_field": "C3"
    }
  }
}
```

### 通配符搜索：

通配符搜索跟前缀搜索类似，比前缀搜索要更加强大。也是需要扫描整个倒排索引，性能也是很差的。
？：表示匹配任意一个字符

- ：表示匹配任意多个字符

示例：通配符搜索条件为*4?的文档

```json
GET /test_index/_search
{
  "query": {
    "wildcard": {
      "test_field": {
        "value": "*4?"
      }
    }
  }
}
```

### 正则搜索：

```json
GET /test_index/_search
{
  "query": {
    "regexp": {
      "test_field": {
        "value": ".*[a-z]{3}[0-9]{2}"
      }
    }
  }
}
```