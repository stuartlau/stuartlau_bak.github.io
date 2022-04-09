---
layout:     post
title:      "ElasticSearch工程实践中的总结"
subtitle:   "ES in Action"
date:       2020-09-02
author:     S.L
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - ElasticSearch
---
> 本文主要总结在平时工作中使用到的ElasticSearch知识点。


### 问题总结
#### 关于keyword和long的选择问题
- keyword更适合用作term查询和聚合，long则适合用于range查询。并不是说value是数字就一定用long。
比如一些固定的值完全可以用keyword，如ISBN。
- long可能并不支持highlight，如果想高亮，需要用keyword
- 待续。。

#### DisMaxQuery的使用
DisMaxQuery是一个接受多个查询的查询，并返回与任何查询子句匹配的任何文档。
虽然bool查询组合了所有匹配查询的分数，但*dis_max*查询使用单个最佳匹配查询子句的分数。

一个查询，它生成由其子查询生成的文档的并集，并为每个文档评分由任何子查询生成的该文档的最大分数，
以及任何其他匹配子查询的平局增量。

当在具有不同增强因子（boost）的多个字段中搜索单词时，这非常有用（因此不能将字段等效地组合到单个搜索字段中）。
我们希望主要分数是与最高提升相关联的分数，而不是字段分数的总和（如BoolQuery布尔查询所给出的）。
如果查询是“albino elephant”，则这确保匹配一个字段的“albino”和匹配另一个的“elephant”获得比匹配两个字段
的“albino”更高的分数。要获得此结果，请同时使用Boolean Query和DisjunctionMax Query：
对于每个术语，DisjunctionMaxQuery在每个字段中搜索它，而将这些DisjunctionMaxQuery的集合组合成BooleanQuery。　

上面是来自官方的翻译，比较晦涩，这里给一个例子，在单字符串的多段搜索时会非常有用。

举一个例子，来自 [极客ES12单字符串多字段的查询和DisMaXQuery](https://blog.csdn.net/weixin_42142216/article/details/102536700)

![](https://img-blog.csdnimg.cn/20191013193145659.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjE0MjIxNg==,size_16,color_FFFFFF,t_70)
在本例中，文档2出现了 brown fox ，而文档1 只出现了brown，所有理论上文档2的相关度应该更高。然而结果正好相反

```
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.90425634,
    "hits" : [
      {
        "_index" : "blogs",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.90425634,
        "_source" : {
          "title" : "Quick brown rabbits",
          "body" : "Brown rabbits are commonly seen."
        }
      },
      {
        "_index" : "blogs",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.77041256,
        "_source" : {
          "title" : "Keeping pets healthy",
          "body" : "My quick brown fox eats rabbits on a regular basis."
        }
      }
    ]
  }
}
```
这与Bool查询的算分过程有关：
- 1.查询should语句中的两个查询
- 2.加和两个查询的平分
- 3.乘以匹配语句的总数
- 4.除以所有语句的总数

所以，在2个字段都都含有 brown 的文档一比在一个字段中含有 brown fox的文档二的得分高，这显然是不合理的。

DisJunction Max Query查询

上例中，title和body互相竞争，不应该简单分数叠加，而是应该找到单个最佳匹配的字段的评分
![](https://img-blog.csdnimg.cn/2019101319385538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjE0MjIxNg==,size_16,color_FFFFFF,t_70)
这次2的结果就是最高的了。

最佳字段查询调优

![](https://img-blog.csdnimg.cn/20191013194422275.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjE0MjIxNg==,size_16,color_FFFFFF,t_70)

通过tie breaker调优
有时，我们也需要其他字段来影响最终的得分，使用 tie_breaker参数来做到这一点


![](https://img-blog.csdnimg.cn/20191013194434799.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjE0MjIxNg==,size_16,color_FFFFFF,t_70)
文档1和文档2虽然最佳匹配字段都是title且得分相同，但是文档1拥有更高的得分，因为文档1的body字段含有quick而文档2不包含，这个字段上的得分会乘以 
tie_breaker并累计到最终得分上去。



因为DisMaxQuery本身不支持Filter，但是可以用一种组合BoolQuery的方式，
[how-to-use-dismax-query-with-filter](https://stackoverflow.com/questions/43786513/elasticsearch-how-to-use-dismax-query-with-filter)：
```
{
    "query": {
        "bool": {
            "must": {
                "dis_max": {
                    "queries": [{
                         "multi_match": {}
                     },
                     { 
                         "multi_match": {}
                     }]
                }
            }
            "filter": {
                //Term filter
            }           
        }
    }
}
```
#### keyword不支持分词有什么办法吗

使用了keyword之后, 只能用作聚合、排序，不能用来做全文匹配。

为了能够既支持聚合、排序，也支持全文搜索, 那么可以通过使用fields属性来补充支持可以被全文检索。
```
"mappings": {
    "my_type": {
        "properties": {
            "title": {
                "type": "keyword",
                "fields": {
                    "make_title_searchable": {
                        "type": "text"
                    }
                }
            }
        }
    }
}
```


#### nested类型
nested嵌套类型是object中的一个特例，可以让array类型的Object独立索引和查询。 使用Object类型有时会出现问题，比如文档 my_index/my_type/1的结构如下：

```

PUT my_index/my_type/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
```
user字段会被动态添加为Object类型。 
最后会被转换为以下平整的形式：

#### keyword和text
Elasticsearch 5.0.0 版本之后 将string拆分成两个新的类型: text和keyword。

text默认结合standard analyzer(标准解析器)对文本进行分词、倒排索引，默认结合标准分析器进行词命中、词频相关度打分。
。而keyword直接将完整的文本保存到倒排索引中，用于筛选数据(例如: select * from x where status='open')、排序、聚合(统计)。


#### should的特殊地方

返回的文档可能满足should子句的条件。在一个Bool查询中，如果没有must或者filter，有一个或者多个should子句，那么只要满足一个就可以返回。minimum_should_match参数定义了至少满足几个子句。

#### property中fields属性的使用
```
"nickname": {
    "type": "text",
    "fields": {
      "ik": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "ngram": {
        "type": "text",
        "analyzer": "nickname_ngram_analyzer",
        "search_analyzer": "keyword_lowercase_analyzer"
      }
    }
  }
```
这里注意对nickName进行了fields设置，分别设置了名为ik和ngram的两个field，并分别指定了对应的索引和搜索时的分析器：analyzer和search_analyzer属性。
所以可以在索引的时候同时对该字段进行两个fields的独立分词索引，搜索的时候也可以同时对两个field进行查询，只要符合一个就可以，有更好的保证准确性。
再来看一下setting片段：
```
"analysis": {
  "analyzer": {
    "keyword_lowercase_analyzer": {
      "filter": [
        "lowercase"
      ],
      "type": "custom",
      "tokenizer": "keyword"
    },
    "nickname_ngram_analyzer": {
      "filter": [
        "lowercase"
      ],
      "tokenizer": "nickname_ngram_tokenizer"
    },
    "uid_analyzer": {
      "tokenizer": "uid_tokenizer"
    }
  },
  "tokenizer": {
    "uid_tokenizer": {
      "type": "ngram",
      "min_gram": "3",
      "max_gram": "13"
    },
    "nickname_ngram_tokenizer": {
      "type": "ngram",
      "min_gram": "1",
      "max_gram": "13"
    }
  }
}
```


### Reference
- [如何取舍keyword-long](https://stackoverflow.com/questions/47542363/should-i-choose-datatype-of-keyword-or-long-integer-for-document-personid-in-e)
- [keyword中使用fields](https://www.jianshu.com/p/0d13dd7d813a)
- [组合过滤器](https://www.elastic.co/guide/cn/elasticsearch/guide/current/combining-filters.html)
- [DisMaxQuery](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html)
- [Elasticsearch Compound Query 复合查询详解](https://my.oschina.net/UpBoy/blog/700787)

> 本文首次发布于 [S.L's Blog](http://elsef.com), 作者 [@stuartlau](http://github.com/stuartlau) ,
转载请保留原文链接.
