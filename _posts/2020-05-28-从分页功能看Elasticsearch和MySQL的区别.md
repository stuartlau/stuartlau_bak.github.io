---
layout:     post
title:      "从分页功能看Elasticsearch和MySQL的区别"
subtitle:   "Pagination in Search"
date:       2020-05-28
author:     LiuShuo
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Search
    - ElasticSearch
    - MySQL
---


### MySQL分页
熟悉MySQL的limit语法的同学都知道`limit x, y`的含义，即`x`为开始位置，`y`为所需返回的数据条数，这个语法天然适合用于做分页查询。

但是有一个性能问题需要考虑一下，比如10个数据一分页，如果有1000页，那么如果使用`limit 10000, 10`这种方式查询10001页数据的话，
MySQL会先去查到10000条记录，并在后面继续查询10条返回，对于速度来说非常慢，并且浪费了很多内存空间保留临时数据。

可以用`explain`命令查看具体的扫描的行数：
```
mysql> explain SELECT * FROM message ORDER BY id DESC LIMIT 10000, 20
***************** 1. row **************
id: 1
select_type: SIMPLE
table: message
type: index
possible_keys: NULL
key: PRIMARY
key_len: 4
ref: NULL
rows: 10020
Extra:
1 row in set (0.00 sec)
```

所以一般都需要在查询条件上提前做好优化，比如知道1000页的最大的id是`z`，那么直接使用`select * from tb where id > z LIMIT 0, 10`
作为查询下一页的语句即可，这样不但减少了查询的数据条数降低了对内存的占用，还大大降低了查询速度，因为一般都是在索引字段上过滤。

### ES分页
MySQL的特点是查询的表的数据都在一台机器的同一个库的文件夹中，搞不好还在一个文件中（如果表不够大的话），而对于ES来说，由于天生是分布式的，
所以排序需要统筹全局的数据进行排序。

比如，假如你每页是 10 条数据，你现在要查询第 100 页，实际上是会把每个 shard 上存储的前 1000 
条数据都查到一个*协调节点*上（因为无法确认哪个shard上的数据是真正符合本次条件的），
如果你有个 5 个 shard，那么就有 5000 条数据，接着协调节点对这 5000 条数据进行一些合并、处理，再获取到最终第 100 页的 10 条数据。

你翻页的时候，翻的越深，每个 shard 返回的数据就越多，而且协调节点处理的时间越长。同时对内存的使用也非常大，比如同时有1000个并发请求，
每个请求查询5000条记录，每条记录1k大小，整体大小将近5G，如果数据量更大后果是服务器会直接报错。

```
{
  "error" : {
    "root_cause" : [ {
      "type" : "query_phase_execution_exception",
      "reason" : "Result window is too large, from + size must be less than or equal to: [10000] but was [10002]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level parameter."
    } ],
    "type" : "search_phase_execution_exception",
    "reason" : "all shards failed",
    "phase" : "query",
    "grouped" : true,
    "failed_shards" : [ {
      "shard" : 0,
      "index" : "_audit_0102",
      "node" : "f_CQitYESZedx8ZbyZ6bHA",
      "reason" : {
        "type" : "query_phase_execution_exception",
        "reason" : "Result window is too large, from + size must be less than or equal to: [10000] but was [10002]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level parameter."
      }
    } ]
  },
  "status" : 500
}
```

> 虽然ES提供了`max_result_window`参数来调整可以一次性查询的窗口大小，默认是10000，但是这个只能缓解，
不能从根本上解决深度查询的问题。随着页码的增加，系统资源占用成指数级上升，很容易就会出现OOM。

所以，ES应该避免使用深度搜索场景。

#### scroll
scroll查询原理是在第一次查询的时候一次性生成一个快照，根据上一次的查询的id来进行下一次的查询，这个就类似于关系型数据库的游标cursor，
然后每次滑动都是根据产生的游标id进行下一次查询，这种性能比上面说的分页性能要高出很多，基本都是毫秒级的。 

注意：scroll不支持跳页查询。 

使用场景：对实时性要求不高的查询，例如微博或者头条滚动查询。

> 注意：从 scroll 请求返回的结果反映了 search 发生时刻的索引状态，就像一个快照。后续的对文档的改动（索引、更新或者删除）都只会影响后面的搜索请求。

为了使用 scroll，初始搜索请求应该在查询中指定 scroll 参数，这可以告诉 Elasticsearch 需要保持搜索的上下文环境多久，如1m即一分钟：
```
curl -XGET 'localhost:9200/twitter/tweet/_search?scroll=1m' -d '
{
    "query": {
        "match" : {
            "title" : "elasticsearch"
        }
    }
}
'
```
> 如果把查询类型设置成SCAN，那么不能获取结果并且不支持排序，只能获得scrollId，如果使用默认设置或者不设置，那么第一次在获取id的同时也可以获取到查询结果。

使用上面的请求返回的结果中包含一个 scroll_id，这个 ID 可以被传递给 scroll API 来检索下一个批次的结果。

```
curl -XGET  'localhost:9200/_search/scroll'  -d'
{
    "scroll" : "1m", 
    "scroll_id" : "c2Nhbjs2OzM0NDg1ODpzRlBLc0FXNlNyNm5JWUc1" 
}
'
```
- GET 或者 POST 可以使用
- URL不应该包含 index 或者 type 名字——这些都指定在了原始的 search 请求中。
- scroll 参数告诉 Elasticsearch 保持搜索的上下文等待另一个 1m

注意：初始搜索请求和每个后续滚动请求返回一个新的 _scroll_id——只有最近的 _scroll_id 才能被使用。

比如下面的Java代码演示了如何持续的从初始结果中获取所有数据：
```java
SearchResponse response1 = client.prepareSearchScroll(scrollId) // 初始查询返回的scrollId
                                .setScroll(TimeValue.timeValueMinutes(5))
                                .execute()
                                .actionGet();

while (response1.getHits().hits().length>0) {
    for (SearchHit searchHit : response1.getHits().hits()) {
        System.out.println(searchHit.getSource().toString());
    }
    response1 = client.prepareSearchScroll(response1.getScrollId()) // 新的scrollId
                      .setScroll(TimeValue.timeValueMinutes(5))
                      .execute().actionGet();
}
```
#### scroll-scan
scroll API 保持了哪些结果已经返回的记录，所以能更加高效地返回排序的结果。但是，按照默认设定排序结果仍然需要代价。

可以指定使用scan为搜索类型，它适合不关心顺序，只需要扫描数据的场景，需要指定`search_type=scan`：
```
curl -XGET 'localhost:9200/twitter/tweet/_search?scroll=1m&search_type=scan' -d '
{
   "query": {
       "match" : {
           "title" : "elasticsearch"
       }
   }
}
'
```
scroll-scan和scroll的区别：
- scan不算分，关闭排序，性能好，结果会按照在索引中出现的顺序返回。
- 不支持聚合
- 初始 search 请求的响应不会在 hits 数组中包含任何结果。第一批结果就会按照第一个 scroll 请求返回。
- 参数 size 控制了*每个分片*上而*非每个请求*的结果数目，所以 size 为 10 的情况下，如果命中了 5 个分片，那么每个 scroll 请求最多会返回 50 个结果。

#### 保活
一般来说，背后的合并过程通过合并更小的分段创建更大的分段来优化索引，同时会删除更小的分段。
这个过程在滚动时进行，但是一个打开状态的搜索上下文阻止了旧分段在使用的时候不会被删除。
这就是 Elasticsearch 能够不管后续的文档的变化，返回初始搜索请求的结果的原因。

### Reference
- [如何跳过es分页这个坑？](https://my.oschina.net/u/1787735/blog/3024051)
- [search-request-scroll.](https://www.elastic.co/guide/en/elasticsearch/reference/2.0/search-request-scroll.html)

> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
