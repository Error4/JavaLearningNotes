



通用格式为：Statement stmt=con.createStatement(int type，int concurrency)

参数 int type

ResultSet.TYPE_FORWORD_ONLY 结果集的游标只能向下滚动。

ResultSet.TYPE_SCROLL_INSENSITIVE 结果集的游标可以上下移动，当数据库变化时，当前结果集不变。

ResultSet.TYPE_SCROLL_SENSITIVE 返回可滚动的结果集，当数据库变化时，当前结果集同步改变。

参数 int concurrency

ResultSet.CONCUR_READ_ONLY 不能用结果集更新数据库中的表。

ResultSet.CONCUR_UPDATETABLE 能用结果集更新数据库中的表。

```java
PreparedStatement stmt = (PreparedStatement) con.prepareStatement(
sql, ResultSet.TYPE_FORWARD_ONLY,
ResultSet.CONCUR_READ_ONLY);     //设置连接属性statement以TYPE_FORWARD_ONLY打开
stmt.setFetchSize(1000);//设置fetch size参数，表示采用服务器端游标，每次从服务器取fetch_size条数据。
stmt.setFetchDirection(ResultSet.FETCH_REVERSE);
```

statement设置以上属性时，采用的是流数据接收方式，每次只从服务器接收部份数据，直到所有数据处理完毕，不会发生JVM OOM。

我们采用select从数据库查询数据时，数据默认并不是一条一条返回给客户端的，也不是一次全部返回客户端的，而是根据客户端fetch_size参数处理，每次只返回fetch_size条记录，当客户端游标遍历到尾部时再从服务端取数据，直到最后全部传送完成。所以如果我们要从服务端一次取大量数据时，可以加大fetch_size，这样可以减少结果数据传输的交互次数及服务器数据准备时间，提高性能。

当*fetchsize*大于***100***时就基本上没有影响了

oracle分页

```sql
select * from (

         select a.*,rownum rn from

                   (select * from product a where company_id=? order by status) a

         where rownum<=20)

where rn>10;
```

