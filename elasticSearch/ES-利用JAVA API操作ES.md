在[ES的官方文档](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html)中有详细的说明，现摘录如下，推荐使用 Java High Level REST Client进行操作。

# 1.准备工作

## 引入依赖

以最新的版本为例

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.6.2</version>
</dependency>
```

## 初始化

填写自己的ES服务端IP即可，如果是集群，则填写多个。

```java
RestHighLevelClient client = new RestHighLevelClient(
        RestClient.builder(
                new HttpHost("localhost", 9200, "http"),
                new HttpHost("localhost", 9201, "http")));
```

使用完毕后，记得关闭

```
client.close();
```

# 2.利用RestClient进行基本操作

## 准备工作

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

需要注意，当前版本，即2.2.6版的spring-boot-starter-data-elasticsearch，集成的ES客户端版本为6.8.7。

![1588507065101](C:/Users/wyf/AppData/Roaming/Typora/typora-user-images/1588507065101.png)

如果与实际ES版本不符合，需要在pom.xml中手动指定ES版本，以我使用的ES 7.6.1版本为例

```xml
 <properties>
        <elasticsearch.version>7.6.1</elasticsearch.version>
 </properties>
```

## 注入client

```java
@Configuration
public class ESConfig {
     @Bean
    public RestHighLevelClient restHighLevelClient(){
         return new RestHighLevelClient(
                 RestClient.builder(
                         new HttpHost("127.0.0.1", 9200, "http")));
     }
}
```

## 创建Index

```java
@Test
void createIndex() throws IOException {
        // 1、创建索引请求
        CreateIndexRequest request = new CreateIndexRequest("user1");
        // 2、客户端执行请求 IndicesClient,请求后获得响应
        CreateIndexResponse createIndexResponse = client.indices().create(request, RequestOptions.DEFAULT);
        System.out.println(createIndexResponse);
    }
```

## 删除Index

```java
    @Test
    void testDeleteIndex() throws IOException {
        DeleteIndexRequest request = new DeleteIndexRequest("user");
        // 删除
        AcknowledgedResponse delete = client.indices().delete(request, RequestOptions.DEFAULT);
        System.out.println(delete.isAcknowledged());
    }
```

## 判断Index存在

```java
 @Test
    void testExistIndex() throws IOException {
        GetIndexRequest request = new GetIndexRequest("user");
        boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
        System.out.println(exists);
    }
```

## 创建Document

```java
@Test
    void testAddDocument() throws IOException {
        //创建对象
        User user = new User("张三", 3);
        //创建请求
        IndexRequest request = new IndexRequest("user");
        //规则 put /user/_doc/1
        request.id("1");
        //设置超时时间
        request.timeout(TimeValue.timeValueSeconds(1));
        //request.timeout("1s");
        //将我们的数据放入请求 json
        request.source(JSON.toJSONString(user), XContentType.JSON);
        //客户端发送请求 , 获取响应的结果
        IndexResponse indexResponse = client.index(request,
                RequestOptions.DEFAULT);
        System.out.println(indexResponse.toString()); //打印文档的内容
        System.out.println(indexResponse.status()); // 对应我们命令返回的状态 ,第一次创建，返回CREATED
    }
```

## 判断Document存在

```java
    @Test
    void testIsExists() throws IOException {
        GetRequest getRequest = new GetRequest("user", "1");
        boolean exists = client.exists(getRequest, RequestOptions.DEFAULT);
        System.out.println(exists);
    }
```

## 获取Document

```java
 @Test
    void testGetDocument() throws IOException {
        GetRequest getRequest = new GetRequest("user", "1");
        GetResponse getResponse = client.get(getRequest,
                RequestOptions.DEFAULT);
        System.out.println(getResponse.getSourceAsString()); // 打印文档的内容
        System.out.println(getResponse); // 返回的全部内容是和命令式一样的
    }
```

## 更新Document

```java
 @Test
    void testUpdateRequest() throws IOException {
        UpdateRequest updateRequest = new UpdateRequest("user", "1");
        updateRequest.timeout("1s");
        User user = new User("张三", 18);
        updateRequest.doc(JSON.toJSONString(user), XContentType.JSON);
        UpdateResponse updateResponse = client.update(updateRequest,
                RequestOptions.DEFAULT);
        System.out.println(updateResponse.status());
    }

```

## 删除Document

```java
 @Test
    void testDeleteRequest() throws IOException {
        DeleteRequest request = new DeleteRequest("user","1");
        request.timeout("1s");
    DeleteResponse deleteResponse = client.delete(request,
            RequestOptions.DEFAULT);
    System.out.println(deleteResponse.status());
```

## 批量插入Document

```java
 @Test
    void testBulkRequest() throws IOException {
        BulkRequest bulkRequest = new BulkRequest();
        bulkRequest.timeout("10s");
        ArrayList<User> userList = new ArrayList<User>();
        userList.add(new User("张三", 3));
        userList.add(new User("李四", 3));
        userList.add(new User("王五", 3));
        for (int i = 0; i < userList.size(); i++) {
            // 批量更新和批量删除，就在这里修改对应的请求就可以了
            bulkRequest.add(new IndexRequest("user").id("" + (i + 1))
                    .source(JSON.toJSONString(userList.get(i)), XContentType.JSON));
        }
        BulkResponse bulkResponse = client.bulk(bulkRequest,
                RequestOptions.DEFAULT);
        System.out.println(bulkResponse.hasFailures()); // 是否失败，返回fals代表成功
    }
```

## 查询

```java
 	/*
    SearchRequest 搜索请求
    SearchSourceBuilder 条件构造
    HighlightBuilder 构建高亮
    TermQueryBuilder 精确查询
    MatchAllQueryBuilder
    xxxQueryBuilder 与命令行时的查询类型一一对应*/
    @Test
    void testSearch() throws IOException {
        SearchRequest searchRequest = new SearchRequest("user");
        // 构建搜索条件
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        // 查询条件，我们可以使用 QueryBuilders 工具来实现
        // QueryBuilders.termQuery 精确
        // QueryBuilders.matchAllQuery() 匹配所有
        TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("name",
                "张三");
        //MatchAllQueryBuilder matchAllQueryBuilder = QueryBuilders.matchAllQuery();
        sourceBuilder.query(termQueryBuilder);
        sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
        
        searchRequest.source(sourceBuilder);
        SearchResponse searchResponse = client.search(searchRequest,
                RequestOptions.DEFAULT);
        System.out.println(JSON.toJSONString(searchResponse.getHits()));
        System.out.println("=================================");
        for (SearchHit documentFields : searchResponse.getHits().getHits()) {
            System.out.println(documentFields.getSourceAsMap());
        }
    }
```









# 3.与Spring	Boot整合
