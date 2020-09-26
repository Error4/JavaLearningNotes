> 本项目来源于B站视频教程[【狂神说Java】ElasticSearch7.6.x最新完整教程通俗易懂](https://www.bilibili.com/video/BV17a4y1x7zq?p=1)
>
> 相关代码已上传至GitHub：https://github.com/Error4/ElasticSearchLearning

# 准备数据

利用`Jsoup`爬取数据， `Jsoup`是一款Java 的HTML解析器，可直接解析某个URL地址、HTML文本内容，可通过DOM，CSS以及类似于jQuery的操作方法来取出和操作数据。详情可参考：[Jsoup中文使用手册](https://www.open-open.com/jsoup/)

以京东搜索的页面为例，检查网页源代码，可以发现，信息设置在如下div中

![](https://s1.ax1x.com/2020/09/26/0ifEtO.md.png)

代码如下：

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class JdContent {
    private String img;
    private String price;
    private String title;
}
```

```java
@Component
public class HtmlParseUtil {
    //京东搜索关键词Java的API
    private final String url = "https://search.jd.com/Search?keyword=";
    public List<JdContent> parseJD(String keyword) throws IOException {
        //解析网页，返回的Document就是JS页面对象
        Document docunment = Jsoup.parse(new URL(url+keyword), 3000);
        //获取需要的标签ID
        Element element = docunment.getElementById("J_goodsList");
        //获取所有的li元素
        Elements elements = element.getElementsByTag("li");
        List<JdContent> list = new ArrayList<JdContent>();
        for (Element el : elements) {
            String img = el.getElementsByTag("img").eq(0).attr("src");
            String price = el.getElementsByClass("p-price").eq(0).text();
            String title = el.getElementsByClass("p-name").eq(0).text();
            list.add(new JdContent(img,price,title));
        }
        return  list;
    }
}
```

但是需要注意，比如受限于网速，图片也有可能会获取不到，为了提高访问速度，对于图片一般使用懒加载，再次观察网页源代码，可以看到`img`标签含有`source-data-lazy-img`属性，可以通过它来访问

![](https://s1.ax1x.com/2020/09/26/0if6CF.png)

```java
String img = el.getElementsByTag("img").eq(0).attr("source-data-lazy-img");
```

# 业务编写

## 插入数据

像已经建立的`jd`索引中插入数据

```java
@Autowired
private RestHighLevelClient restHighLevelClient;
@Autowired
private HtmlParseUtil htmlParseUtil;

public boolean parseContent(String keyword) throws IOException {
        List<JdContent> jdContents = htmlParseUtil.parseJD(keyword);
        BulkRequest bulkRequest = new BulkRequest();
        bulkRequest.timeout("1m");
        for (JdContent jdContent : jdContents) {
            bulkRequest.add(new IndexRequest("jd").source(JSON.toJSONString(jdContent), XContentType.JSON));
        }
        BulkResponse bulkResponse= restHighLevelClient.bulk(bulkRequest, RequestOptions.DEFAULT);
        return !bulkResponse.hasFailures();
    }
```

## 提供查询

```java
public List<Map<String,Object>> search(String keyword, int pageNo, int pageSize) throws IOException {
        if (pageNo<1){
            pageNo = 1;
        }
        //条件搜索
        SearchRequest searchRequest = new SearchRequest("jd");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //分页
        sourceBuilder.from(pageNo);
        sourceBuilder.size(pageSize);
         //匹配数据
        MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("title", keyword);
        sourceBuilder.query(matchQueryBuilder);
        sourceBuilder.timeout(new TimeValue(50, TimeUnit.SECONDS));
        //执行搜索
        searchRequest.source(sourceBuilder);
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        //解析结果
        ArrayList<Map<String, Object>> list = new ArrayList<Map<String, Object>>();
        for (SearchHit hit : searchResponse.getHits().getHits()) {
            list.add(hit.getSourceAsMap());
        }
        return list;
    }
```

## 前端交互

```html
<script th:src="@{/js/axios.min.js}"></script>
<script th:src="@{/js/vue.min.js}"></script>
<script>
    new Vue({
        el:'#app',
        data:{
            keyword:'',//搜索的关键字
            results:[]////搜索的结果
        },
        methods: {
            searchKey() {
                var keyword = this.keyword;
                // console.log(keyword);
                axios.get('search/'+keyword+"/1/10").then(response=>{
                    // console.log(response);
                    this.results = response.data;
                })
            }
        }
    })
</script>
```

其中，app是标签ID，keyword为输入栏绑定的model名称，searchKey触发事件

```html
<div class="page" id="app">
<input v-model="keyword" type="text">
<button type="submit" id="searchbtn" @click.prevent="searchKey">搜索</button>
```

遍历返回值即可

```html
<div class="view grid-nosku" v-for="result in results">
    <div class="product">
        <div class="product-iWrap">
            <!--商品封面-->
            <div class="productImg-wrap">
                <a class="productImg">
                    <img :src="result.img">
                </a>
            </div>
            <!--价格-->
            <p class="productPrice">
                <em><b>¥</b>{{result.price}}</em>
            </p>
            <!--标题-->
            <p class="productTitle">
                <a> {{result.title}} </a>
            </p>
        </div>
    </div>
</div>
```

## 关键字高亮

与普通查询大体逻辑相同，只需要设置自定义的高亮逻辑，并在ES的返回值中用高亮的内容替换原内容即可。

```java
public List<Map<String,Object>> searchHignLight(String keyword, int pageNo, int pageSize) throws IOException {
        if (pageNo<1){
            pageNo = 1;
        }
        //条件搜索
        SearchRequest searchRequest = new SearchRequest("jd");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //分页
        sourceBuilder.from(pageNo);
        sourceBuilder.size(pageSize);
        //匹配数据
        MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("title", keyword);
        sourceBuilder.query(matchQueryBuilder);
        sourceBuilder.timeout(new TimeValue(50, TimeUnit.SECONDS));
        //高亮
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("title");
        highlightBuilder.preTags("<span style='color:red'>");
        highlightBuilder.postTags("</span>");
        //设置关键字高亮一次
        highlightBuilder.requireFieldMatch(false);
        sourceBuilder.highlighter(highlightBuilder);

        //执行搜索
        searchRequest.source(sourceBuilder);
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        //解析结果
        ArrayList<Map<String, Object>> list = new ArrayList<Map<String, Object>>();
        for (SearchHit hit : searchResponse.getHits().getHits()) {
            //解析高亮
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            HighlightField title = highlightFields.get("title");
            //将原来的字段替换为高亮的字段设置
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            if (title!=null){
                Text[] fragments = title.fragments();
                String temValue = "";
                for (Text fragment : fragments) {
                    temValue+=fragment;
                }
                sourceAsMap.put("title",temValue);//替换
            }
            list.add(sourceAsMap);
        }
        return list;
    }
```

前端页面解析返回的HTML即可

```html
<!--标题-->
<p class="productTitle">
    <a v-html="result.title"></a>
</p>
```

