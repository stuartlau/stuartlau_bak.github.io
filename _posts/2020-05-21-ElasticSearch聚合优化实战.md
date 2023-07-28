---
layout:     post
title:      "ElasticSearch聚合实战"
subtitle:   "Aggregation in ES"
date:       2020-05-21
author:     L.S
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Search
    - ElasticSearch
---

> 聚合功能为ES注入了统计分析的血统，使用户在面对大数据提取统计指标时变得游刃有余。同样的工作，你在Hadoop中可能需要写mapreduce或Hive，在mongo中你必须得用大段的mapreduce脚本，而在ES中仅仅调用一个API就能实现了。

### 按时间聚合
从官网找一个[例子](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_returning_empty_buckets.html)，最基本的按照时间进行聚合的：
```
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "month",
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0, 
            "extended_bounds" : { 
                "min" : "2014-01-01",
                "max" : "2014-12-31"
            }
         }
      }
   }
}
```
如根据`sole`字段来进行按日期的聚合，其中每个桶都是按照一个`month`的维度进行，另外我们还对具体的结果进行了格式化以便于阅读。
同时考虑到可能某些月份的结果为空，也需要返回来只做完整的报表，即`min_doc_count : 0`，最关键的，date_histogram （和 histogram 一样）默认只会返回文档数目非零的 buckets。
所以需要使用`extended_bounds`来控制上下界限，防止被默认过滤掉空结果的部分。

### 多层聚合
一个双层聚合的[例子](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_extended_example.html)：
```
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "quarter", 
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0,
            "extended_bounds" : {
                "min" : "2014-01-01",
                "max" : "2014-12-31"
            }
         },
         "aggs": {
            "per_make_sum": {
               "terms": {
                  "field": "make"
               },
               "aggs": {
                  "sum_price": {
                     "sum": { "field": "price" } 
                  }
               }
            },
            "total_sum": {
               "sum": { "field": "price" } 
            }
         }
      }
   }
}
```
最终按照每个季度再按照生产商的维度计算了每个生产商的销售额`per_make_sum`，同时还有一个所有生产商的总销售额`total_sum`，
注意这两个聚合的结果都是在第一个聚合结果下，即每个季度的`sales`下的。
![](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/elas_29in02.png)
如果有了具体的聚合结果，生成报表就方便多了，我们可以对不同的聚合结果用不同的样式展示，如上图的柱状图和折线图，
用类似Kibana和Grafana这种工具很方便生成。

```
{
    "size" : 0,
    "query" : {
        "constant_score": {
            "filter": {
                "range": {
                    "price": {
                        "gte": 10000
                    }
                }
            }
        }
    },
    "aggs" : {
        "single_avg_price": {
            "avg" : { "field" : "price" }
        }
    }
}
```
这里使用了constant_score 查询和 filter 约束：从根本上讲，使用 non-scoring 查询和使用 match 
查询没有任何区别。查询（包括了一个过滤器）返回一组文档的子集，聚合正是操作这些文档。使用 filtering query 会忽略评分，并有可能会缓存结果数据等等。

### Filter Bucket
我们可以指定一个过滤桶，当文档满足过滤桶的条件时，我们将其加入到桶内，[例子](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_filter_bucket.html)：
```
{
   "size" : 0,
   "query":{
      "match": {
         "make": "ford"
      }
   },
   "aggs":{
      "recent_sales": {
         "filter": { 
            "range": {
               "sold": {
                  "from": "now-1M"
               }
            }
         },
         "aggs": {
            "average_price":{
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
```
使用过滤桶在查询范围基础上应用过滤器，avg 度量只会对 ford 和上个月售出的文档计算平均售价。
> 因为 filter 桶和其他桶的操作方式一样，所以可以随意将其他桶和度量嵌入其中。所有嵌套的组件都会 "继承" 这个过滤，这使我们可以按需针对聚合过滤出选择部分。

### 全局桶
```
{
    "size" : 0,
    "query" : {
        "match" : {
            "make" : "ford"
        }
    },
    "aggs" : {
        "single_avg_price": {
            "avg" : { "field" : "price" } 
        },
        "all": {
            "global" : {}, 
            "aggs" : {
                "avg_price": {
                    "avg" : { "field" : "price" } 
                }

            }
        }
    }
}
```
single_avg_price 度量计算是基于查询范围内所有文档，即所有 福特 汽车。avg_price 度量是嵌套在 全局 桶下的，这意味着它完全忽略了范围并对所有文档进行计算。聚合返回的平均值是所有汽车的平均售价。
### 实战
下面的例子来自线上，做了一定的脱敏，这是一个`scoping aggregation`的例子：
```
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "msgText": "hi"
          }
        },
        {
          "bool": {
            "should": [
              {
                "bool": {
                  "filter": [
                    {
                      "term": {
                        "to": "22222222"
                      }
                    },
                    {
                      "range": {
                        "timestamp": {
                          "from": 1589795975534,
                          "to": null,
                          "include_lower": true,
                          "include_upper": true
                        }
                      }
                    }
                  ]
                }
              },
              {
                "bool": {
                  "filter": [
                    {
                      "term": {
                        "to": "11111111"
                      }
                    },
                    {
                      "range": {
                        "timestamp": {
                          "from": 1589771519612,
                          "to": null,
                          "include_lower": true,
                          "include_upper": true
                        }
                      }
                    }
                  ]
                }
              }
            ]
          }
        }
      ]
    }
  },
  "from": 0,
  "aggs": {
    "sessions": {
      "terms": {
        "field": "sessionId",
        "execution_hint": "map",
        "size": 100
      },
      "aggs": {
        "top_ten_hits": {
          "top_hits": {
            "size": 10,
            "sort": {
              "timestamp": {
                "order": "desc"
              }
            }
          }
        }
      }
    }
  }
}
```

结果如下：
```
{
  "took": 4117,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 3659354,
    "max_score": 0.0,
    "hits": [
      
    ]
  },
  "aggregations": {
    "sessions": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "102162",
          "doc_count": 1606188,
          "top_ten_hits": {
            "hits": {
              "total": 1606188,
              "max_score": null,
              "hits": [
                {
                  "_index": "message_v2",
                  "_type": "message",
                  "_id": "id4",
                  "_score": null,
                  "sort": [
                    1577615548758
                  ]
                },
                {
                  "_index": "message_v2",
                  "_type": "message",
                  "_id": "id3",
                  "_score": null,
                  "sort": [
                    1577615548690
                  ]
                }
              ]
            }
          }
        },
        {
          "key": "102273",
          "doc_count": 47396,
          "top_ten_hits": {
            "hits": {
              "total": 47396,
              "max_score": null,
              "hits": [
                {
                  "_index": "message_v2",
                  "_type": "message",
                  "_id": "id2",
                  "_score": null,
                  "sort": [
                    1578917789916
                  ]
                }
              ]
            }
          }
        }
      ]
    }
  }
}

```
由于我们的mapping设置禁止掉了`_source`，所以结果中没有展示`hits`（不是`aggregations`里的）的内容。

再举一个例子，查询全部Nginx的请求URI在时间维度的聚合的个数：
```
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "1589990400000",
              "lte": "1590058860000",
              "format": "epoch_millis"
            }
          }
        },
        {
          "query_string": {
            "analyze_wildcard": true,
            "query": "host: download.xxx.com AND tag: access AND status: >399"
          }
        }
      ]
    }
  },
  "aggs": {
    "3": {
      "terms": {
        "field": "urlpath.keyword",
        "size": 10,
        "order": {
          "_count": "desc"
        },
        "min_doc_count": 100
      },
      "aggs": {
        "2": {
          "date_histogram": {
            "interval": "4h",
            "field": "@timestamp",
            "min_doc_count": 0,
            "extended_bounds": {
              "min": "1589990400000",
              "max": "1590058860000"
            },
            "format": "epoch_millis"
          },
          "aggs": {
            
          }
        }
      }
    }
  }
}
```
结果如下：
```
{
  "took": 46,
  "timed_out": false,
  "_shards": {
    "total": 105,
    "successful": 105,
    "skipped": 95,
    "failed": 0
  },
  "hits": {
    "total": 8669,
    "max_score": 0,
    "hits": [
      
    ]
  },
  "aggregations": {
    "3": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "2": {
            "buckets": [
              {
                "key_as_string": "1589990400000",
                "key": 1589990400000,
                "doc_count": 440
              },
              {
                "key_as_string": "1590004800000",
                "key": 1590004800000,
                "doc_count": 428
              },
              {
                "key_as_string": "1590019200000",
                "key": 1590019200000,
                "doc_count": 243
              },
              {
                "key_as_string": "1590033600000",
                "key": 1590033600000,
                "doc_count": 4238
              },
              {
                "key_as_string": "1590048000000",
                "key": 1590048000000,
                "doc_count": 3121
              }
            ]
          },
          "key": "/rest/v2/app/download",
          "doc_count": 8470
        },
        {
          "2": {
            "buckets": [
              {
                "key_as_string": "1589990400000",
                "key": 1589990400000,
                "doc_count": 15
              },
              {
                "key_as_string": "1590004800000",
                "key": 1590004800000,
                "doc_count": 1
              },
              {
                "key_as_string": "1590019200000",
                "key": 1590019200000,
                "doc_count": 4
              },
              {
                "key_as_string": "1590033600000",
                "key": 1590033600000,
                "doc_count": 67
              },
              {
                "key_as_string": "1590048000000",
                "key": 1590048000000,
                "doc_count": 98
              }
            ]
          },
          "key": "/rest/v2/app/upload",
          "doc_count": 185
        }
      ]
    }
  }
```
### 聚合优化
大多数时候对单个字段的聚合查询还是非常快的， 但是当需要同时聚合多个字段时，就可能会产生大量的分组，最终结果就是占用 es 大量内存，从而导致 OOM 的情况发生。

实践应用发现，以下情况都会比较慢：
1）待聚合文档数比较多（千万、亿、十亿甚至更多）；
2）聚合条件比较复杂（多重条件聚合）；
3）全量聚合（翻页的场景用）

注意到上面的[`execution_hint`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html#search-aggregations-bucket-terms-aggregation-execution-hint)
属性了吗？

```
There are different mechanisms by which terms aggregations can be executed:

by using field values directly in order to aggregate data per-bucket (map)
by using global ordinals of the field and allocating one bucket per global ordinal (global_ordinals)
```
1）global_ordinals是关键字字段（ keyword field ）的默认选项，它使用 全局顺序(global ordinals) 来动态分配存储区，因此内存使用情况与作为聚合作用域一部分的文档值的数量成线性关系。

2）只有极少数文档与查询匹配匹配时才应考虑使用map方式。
默认情况下，只有在脚本上运行聚合时才会使用map，因为它们没有序号( ordinals )。

否则，基于顺序(ordinals) 的执行模式会相对更快。

> 我们的一个场景是按会话的维度聚合前N条符合条件的命中词的消息，所以选择了使用`map`的方式，即直接使用了具体的value内容来进行per-bucket
的聚合，效果比默认方式耗时降低了一个数量级。

当然，最终还是要根据实际场景亲自测试一下，这里有一个[测试数据](https://blog.csdn.net/laoyang360/article/details/79253294)。

### 缓存
ES中经常使用到的聚合结果集可以被缓存起来，以便更快速的系统响应。这些缓存的结果集和你掠过缓存直接查询的结果是一样的。因为，第一次聚合的条件与结果缓存起来后，ES会判断你后续使用的聚合条件，如果聚合条件不变，并且检索的数据块未增更新，ES会自动返回缓存的结果。

注意聚合结果的缓存只针对size=0的请求(参考3.10章节)，还有在聚合请求中使用了动态参数的比如Date Range中的now(参考3.5章节)，ES同样不会缓存结果，因为聚合条件是动态的，即使缓存了结果也没用了。


### 数据的不确定性
使用terms聚合，结果可能带有一定的偏差与错误性。

比如：

我们想要获取name字段中出现频率最高的前5个。

此时，客户端向ES发送聚合请求，主节点接收到请求后，会向每个独立的分片发送该请求。
分片独立的计算自己分片上的前5个name，然后返回。当所有的分片结果都返回后，在主节点进行结果的合并，再求出频率最高的前5个，返回给客户端。

这样就会造成一定的误差，比如最后返回的前5个中，有一个叫A的，有50个文档；B有49。 
但是由于每个分片独立的保存信息，信息的分布也是不确定的。 有可能第一个分片中B的信息有2个，但是没有排到前5，
所以没有在最后合并的结果中出现。 这就导致B的总数少计算了2，本来可能排到第一位，却排到了A的后面。


为了改善上面的问题，就可以使用size和shard_size参数。

- size参数规定了最后返回的term个数(默认是10个)
- shard_size参数规定了每个分片上返回的个数

如果shard_size小于size，那么分片也会按照size指定的个数计算
通过这两个参数，如果我们想要返回前5个，size=5;shard_size可以设置大于5，这样每个分片返回的词条信息就会增多，相应的误差几率也会减小。

### References
- [聚合](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggregations.html)
- [aggs-high-level](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html)
- [elasticsearch 深入 —— Top Hits Aggregation](https://blog.csdn.net/ctwy291314/article/details/82773180)
- [ES之五：ElasticSearch聚合](https://www.cnblogs.com/duanxz/p/6528161.html)


> 本文首次发布于 [LiuShuo's Blog](https://liushuo.me), 
转载请保留原文链接.
