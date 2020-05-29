在之前的文章中，我们已经介绍了如何设计良好的表结构，如何建立良好的索引，除了这些，还需要合理的设计查询，只有这三项优化齐头并进，才能得到更优的性能。

# 1.慢查询优化分析

首先，我们要知道，查询性能低下的最基本原因是访问数据太多，可以通过以下两个步骤进行分析：

- 是否向数据库请求了不需要的数据。典型的，比如只需要从表中获取一些字段的值，但却使用了`select *`来进行查询
- MySQL是否在扫描额外的记录。典型的，如果没有索引，在做查询时，MySQL会对数据进行全表扫描。

# 2.重构查询

## 2.1 切分查询

有时候对于一个大查询要“分而治之”，将大查询切分成小查询，每个查询功能完全一样，只完成一小部分，每次只返回一小部分结果。

典型的例子，删除旧的数据。如果用一个语句删除大量数据，则可能需要一次锁住很多数据，占满整个事务日志，消耗较大的系统资源。将一个大的DELETE语句拆分成多个较小的查询可以尽可能小的影响性能。

## 2.2 分解关联查询

​		简单说，就是把对关联的表，每一个表都进行一次单独查询，然后将结果在应用程序中关联。

```sql
 SELECT * FROM tag 
 JOIN tag_post ON tag_post.tag_id = tag.id 
 JOIN post ON tag_post_id = post.id 
 WHERE tag.tag = 'mysql';
```

可分解为以下三部查询:

```sql
  SELECT * FROM tag WHERE tag ='mysql';
  SELECT * FROM tag_post WHERE tag_id  = 1234;
  SELECT * FROM post WHERE post.id in (124,324,553,2324);
```

优点如下：

- 让缓存的效率更高

- 将查询分解后，执行单个查询可以减少锁的竞争
- 在应用层做关联，可以更容易对数据进行拆分，更容易做到高性能和可扩展

# 3.优化特定类型的查询

## 3.1 count()优化

COUNT(常量) 和 COUNT(*) 表示的是直接查询符合条件的数据库表的行数。

而COUNT(列名)表示的是查询符合条件的列的值不为NULL的行数。

另外，MyISAM在统计表的总行数的时候会很快，但是有个大前提，**不能加有任何WHERE条件**。这是因为：MyISAM对于表的行数做了优化，具体做法是有一个变量存储了表的行数。

以MyISAM的这个特性举例，假如有1000条数据，查询ID大于10的数量，可能需要扫描900次，完全可以将条件反转，先查询ID小于10的，再用总数减去这个值就能得到ID大于10的数量。

## 3.2 优化关联查询

- 确保ON或者USING子句上的列有索引
- 确保任何的ORDER BY和GROUP BY的表达式只涉及到一个表的列，这样才可能会用到索引

## 3.3 优化子查询

尽可能使用关联查询代替。执行子查询时，MySQL需要创建临时表，查询完毕后再删除这些临时表，所以，子查询的速度会受到一定的影响，这里多了一个创建和销毁临时表的过程。

 例如，假设我们要将所有没有订单记录的用户取出来，可以用下面这个查询完成： 

```sql
SELECT * FROM customerinfo 
WHERE CustomerID NOT in (SELECT CustomerID FROM salesinfo )   
```

可以优化为如下，尤其是当salesinfo表中对CustomerID建有索引的话，性能将会更好

```sql
SELECT * FROM customerinfo 
LEFT JOIN salesinfoON customerinfo.CustomerID=salesinfo.CustomerID 
WHERE salesinfo.CustomerID IS NULL 
```

## 3.4 优化UNION查询

MySQL总是通过创建并填充临时表的方式来进行UNION查询，通常需要将WHEWE,LIMIT,ORDER BY等子句下推到各个子查询中。

另外，除非确实需要唯一的行，否则一定要是用UNION ALL，如果没有ALL关键字，MySQL会给临时表加上DISTINCT关键字，会导致对临时表做唯一性检查，代价很高。