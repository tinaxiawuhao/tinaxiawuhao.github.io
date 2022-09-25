---
title: 'elasticSearch查询语句'
date: 2021-05-28 14:13:34
tags: [ELK]
published: true
hideInList: false
feature: /post-images/AdeCmc8BU.png
isTop: false
---
<!-- more -->
>原文：https://distributedbytes.timojo.com/2016/07/23-useful-elasticsearch-example-queries.html
作者：Tim Ojo

为了演示不同类型的 ElasticSearch 的查询，我们将使用以下字段搜索书籍文档的集合（ title（标题）, authors（作者）, summary（摘要）, publish_date（发布日期）和 num_reviews（浏览数））。

在这之前，首先我们应该先创建一个新的索引（index），并批量导入一些文档：

创建索引：

```json
PUT /bookdb_index
    { "settings": { "number_of_shards": 1 }} 
```

批量上传文档：

```json
POST /bookdb_index/book/_bulk
    { "index": { "_id": 1 }}
    { "title": "Elasticsearch: The Definitive Guide", "authors": ["clinton gormley", "zachary tong"], "summary" : "A distibuted real-time search and analytics engine", "publish_date" : "2015-02-07", "num_reviews": 20, "publisher": "oreilly" }
    { "index": { "_id": 2 }}
    { "title": "Taming Text: How to Find, Organize, and Manipulate It", "authors": ["grant ingersoll", "thomas morton", "drew farris"], "summary" : "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization", "publish_date" : "2013-01-24", "num_reviews": 12, "publisher": "manning" }
    { "index": { "_id": 3 }}
    { "title": "Elasticsearch in Action", "authors": ["radu gheorge", "matthew lee hinman", "roy russo"], "summary" : "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms", "publish_date" : "2015-12-03", "num_reviews": 18, "publisher": "manning" }
    { "index": { "_id": 4 }}
    { "title": "Solr in Action", "authors": ["trey grainger", "timothy potter"], "summary" : "Comprehensive guide to implementing a scalable search engine using Apache Solr", "publish_date" : "2014-04-05", "num_reviews": 23, "publisher": "manning" }
```

## 例子

### 1.基本比对查询

执行基本的全文本（匹配）查询有两种方式：使用Search Lite API（它希望所有搜索参数都作为URL的一部分传入），或使用完整的JSON请求正文（允许您使用完整的Elasticsearch DSL)。

这是一个基本的匹配查询，它在所有字段中搜索字符串“ guide”

```json
GET /bookdb_index/book/_search?q=guide

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.28168046,
        "_source": {
          "title": "Elasticsearch: The Definitive Guide",
          "authors": [
            "clinton gormley",
            "zachary tong"
          ],
          "summary": "A distibuted real-time search and analytics engine",
          "publish_date": "2015-02-07",
          "num_reviews": 20,
          "publisher": "manning"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.24144039,
        "_source": {
          "title": "Solr in Action",
          "authors": [
            "trey grainger",
            "timothy potter"
          ],
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "publish_date": "2014-04-05",
          "num_reviews": 23,
          "publisher": "manning"
        }
      }
    ]

```


该查询的完整版本如下所示，其产生的结果与上述搜索精简版相同。



```java
{
    "query": {
        "multi_match" : {
            "query" : "guide",
            "fields" : ["_all"]
        }
    }
}
 
```


使用`multi_match`关键字代替关键字`match`是对多个字段运行相同查询的便捷快捷方式。该`fields`属性指定要查询的字段，在这种情况下，我们要查询文档中的所有字段。
这两个API均允许您指定要搜索的字段。例如，要在标题字段中搜索带有“in action”字样的图书，请执行以下操作：



```java
GET /bookdb_index/book/_search?q=title:in action

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.6259885,
        "_source": {
          "title": "Solr in Action",
          "authors": [
            "trey grainger",
            "timothy potter"
          ],
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "publish_date": "2014-04-05",
          "num_reviews": 23,
          "publisher": "manning"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.5975345,
        "_source": {
          "title": "Elasticsearch in Action",
          "authors": [
            "radu gheorge",
            "matthew lee hinman",
            "roy russo"
          ],
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "publish_date": "2015-12-03",
          "num_reviews": 18,
          "publisher": "manning"
        }
      }
    ]
 
```

但是，全身DSL在创建更复杂的查询（如我们将在后面看到）以及指定您希望如何返回结果方面给您更大的灵活性。在下面的示例中，我们指定要返回的结果数，开始的偏移量（用于分页），要返回的文档字段以及术语突出显示。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "match" : {
            "title" : "in action"
        }
    },
    "size": 2,
    "from": 0,
    "_source": [ "title", "summary", "publish_date" ],
    "highlight": {
        "fields" : {
            "title" : {}
        }
    }
}

[Results]
"hits": {
    "total": 2,
    "max_score": 0.9105287,
    "hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.9105287,
        "_source": {
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        },
        "highlight": {
          "title": [
            "Elasticsearch <em>in</em> <em>Action</em>"
          ]
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.9105287,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        },
        "highlight": {
          "title": [
            "Solr <em>in</em> <em>Action</em>"
          ]
        }
      }
    ]
  }
 
```

**注意**：对于多字查询，该`match`查询使您可以指定是否使用`and`运算符而不是默认`or`运算符。您还可以指定`minimum_should_match`选项来调整返回结果的相关性。可以在[这里](https://www.elastic.co/guide/en/elasticsearch/guide/current/match-multi-word.html)找到详细信息。



### 2.多字段搜索

正如我们已经看到的，要在搜索中查询多个文档字段（例如，在标题和摘要中搜索相同的查询字符串），则可以使用该`multi_match`查询。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "multi_match" : {
            "query" : "elasticsearch guide",
            "fields": ["title", "summary"]
        }
    }
}

[Results]
"hits": {
    "total": 3,
    "max_score": 0.9448582,
    "hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.9448582,
        "_source": {
          "title": "Elasticsearch: The Definitive Guide",
          "authors": [
            "clinton gormley",
            "zachary tong"
          ],
          "summary": "A distibuted real-time search and analytics engine",
          "publish_date": "2015-02-07",
          "num_reviews": 20,
          "publisher": "manning"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.17312013,
        "_source": {
          "title": "Elasticsearch in Action",
          "authors": [
            "radu gheorge",
            "matthew lee hinman",
            "roy russo"
          ],
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "publish_date": "2015-12-03",
          "num_reviews": 18,
          "publisher": "manning"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.14965448,
        "_source": {
          "title": "Solr in Action",
          "authors": [
            "trey grainger",
            "timothy potter"
          ],
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "publish_date": "2014-04-05",
          "num_reviews": 23,
          "publisher": "manning"
        }
      }
    ]
  }
 
```

请注意，命中数字3相匹配，因为在摘要中找到了“指南”一词。



### 3.提高字段重要性

由于我们正在多个字段中进行搜索，因此我们可能希望提高特定字段中的得分。在以下人为设计的示例中，我们将摘要字段的得分提高了3倍，以提高摘要字段的重要性，这反过来又会增加文档`_id 4`的相关性。

```java
POST /bookdb_index/book/_search
{
    "query": {
        "multi_match" : {
            "query" : "elasticsearch guide",
            "fields": ["title", "summary^3"]
        }
    },
    "_source": ["title", "summary", "publish_date"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.31495273,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.14965448,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.13094766,
        "_source": {
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      }
    ]
 
```

**注意** ：Boosting不仅仅意味着计算出的分数会乘以Boosting系数。实际应用的升压值经过归一化和一些内部优化。有关增强工作原理的更多信息，请参见 [Elasticsearch指南](https://www.elastic.co/guide/en/elasticsearch/guide/current/query-time-boosting.html)。



### 4.布尔查询

AND / OR / NOT运算符可用于微调我们的搜索查询，以提供更相关或更具体的结果。这是在搜索API中作为`bool`查询实现的。该`bool`查询接受一个`must`参数（等同于AND），一个`must_not`参数（等同于NOT）和一个`should`参数（等同于OR）。例如，如果我要搜索书名中带有“ Elasticsearch”或“ Solr”字样的书，则AND由“克林顿·戈姆利”（clinton gormley）创作，而不由“ radu gheorge”（radu gheorge）创作：



```java
POST /bookdb_index/book/_search
{
    "query": {
        "bool": {
            "must": {
                "bool" : { "should": [
                      { "match": { "title": "Elasticsearch" }},
                      { "match": { "title": "Solr" }} ] }
            },
            "must": { "match": { "authors": "clinton gormely" }},
            "must_not": { "match": {"authors": "radu gheorge" }}
        }
    }
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.3672021,
        "_source": {
          "title": "Elasticsearch: The Definitive Guide",
          "authors": [
            "clinton gormley",
            "zachary tong"
          ],
          "summary": "A distibuted real-time search and analytics engine",
          "publish_date": "2015-02-07",
          "num_reviews": 20,
          "publisher": "oreilly"
        }
      }
    ]
 
```

**注意**：如您所见，布尔查询可以包装任何其他查询类型，包括其他布尔查询，以创建任意复杂或深度嵌套的查询。



### 5.模糊查询

可以在“匹配”和“多匹配”查询中启用模糊匹配，以捕获拼写错误。模糊程度是根据距原始单词的Levenshtein距离指定的。

```java
POST /bookdb_index/book/_search
{
    "query": {
        "multi_match" : {
            "query" : "comprihensiv guide",
            "fields": ["title", "summary"],
            "fuzziness": "AUTO"
        }
    },
    "_source": ["title", "summary", "publish_date"],
    "size": 1
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.5961596,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      }
    ]
 
```

**注意**：模糊度值`"AUTO"`等于指定`2`术语长度大于5时的值。但是，设置80％的人类拼写错误的编辑距离为`1`，并将模糊度设置为`1`可以改善整体搜索性能。有关更多信息，请参见《Elasticsearch最终指南》的“[错别字和拼写错误”](https://www.elastic.co/guide/en/elasticsearch/guide/current/fuzziness.html)一章。



### 6.通配符查询

通配符查询使您可以指定要匹配的模式，而不是整个术语。`?`匹配任何字符并`*`匹配零个或多个字符。例如，要查找所有作者姓名以字母“ t”开头的记录



```java
POST /bookdb_index/book/_search
{
    "query": {
        "wildcard" : {
            "authors" : "t*"
        }
    },
    "_source": ["title", "authors"],
    "highlight": {
        "fields" : {
            "authors" : {}
        }
    }
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 1,
        "_source": {
          "title": "Elasticsearch: The Definitive Guide",
          "authors": [
            "clinton gormley",
            "zachary tong"
          ]
        },
        "highlight": {
          "authors": [
            "zachary <em>tong</em>"
          ]
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 1,
        "_source": {
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "authors": [
            "grant ingersoll",
            "thomas morton",
            "drew farris"
          ]
        },
        "highlight": {
          "authors": [
            "<em>thomas</em> morton"
          ]
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 1,
        "_source": {
          "title": "Solr in Action",
          "authors": [
            "trey grainger",
            "timothy potter"
          ]
        },
        "highlight": {
          "authors": [
            "<em>trey</em> grainger",
            "<em>timothy</em> potter"
          ]
        }
      }
    ] 
 
```

### 7.正则表达式查询

正则表达式查询使您可以指定比通配符查询更复杂的模式。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "regexp" : {
            "authors" : "t[a-z]*y"
        }
    },
    "_source": ["title", "authors"],
    "highlight": {
        "fields" : {
            "authors" : {}
        }
    }
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 1,
        "_source": {
          "title": "Solr in Action",
          "authors": [
            "trey grainger",
            "timothy potter"
          ]
        },
        "highlight": {
          "authors": [
            "<em>trey</em> grainger",
            "<em>timothy</em> potter"
          ]
        }
      }
    ]
 
```

### 8.匹配词组查询

匹配词组查询要求查询字符串中的所有术语都存在于文档中，并按照查询字符串中指定的顺序并且彼此接近。默认情况下，术语必须彼此完全平行，但是您可以指定一个`slop`值，该值指示在仍将文档视为匹配项时允许相隔多远的术语。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "multi_match" : {
            "query": "search engine",
            "fields": ["title", "summary"],
            "type": "phrase",
            "slop": 3
        }
    },
    "_source": [ "title", "summary", "publish_date" ]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.22327082,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.16113183,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      }
    ]
 
```

**注意**：在上面的示例中，对于非短语类型查询，文档的`_id 1`得分通常更高，并且`_id 4`由于其字段长度较短而出现在文档的前面。但是，在进行短语查询时，会考虑到术语的接近度，因此文档`_id 4`得分会更高。



### 9.匹配词组前缀

匹配词组前缀查询可在查询时提供“按需输入”或“穷人”版本的自动完成功能，而无需以任何方式准备数据。像match_phrase查询一样，它接受一个`slop`参数以使单词顺序和相对位置的刚性降低一些。我还接受该`max_expansions`参数来限制匹配项的数量，以降低资源强度。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "match_phrase_prefix" : {
            "summary": {
                "query": "search en",
                "slop": 3,
                "max_expansions": 10
            }
        }
    },
    "_source": [ "title", "summary", "publish_date" ]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.5161346,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.37248808,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      }
    ]
 
```

**注意**：“按类型查询时搜索”会降低性能。更好的解决方案是按类型进行索引时间搜索。请查看[完成提示API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters-completion.html)或使用[Edge-Ngram过滤器](https://www.elastic.co/guide/en/elasticsearch/guide/current/_index_time_search_as_you_type.html)以获取更多信息。



### 10.请求参数

该`query_string`查询提供了一种以简洁的速记语法执行多重匹配查询，布尔查询，增强查询，模糊匹配，通配符，正则表达式和范围查询的方法。在下面的示例中，我们对术语“ saerch算法”执行模糊搜索，其中作者之一是“ grant ingersoll”或“ tom morton”。我们搜索所有字段，但对摘要字段加2。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "query_string" : {
            "query": "(saerch~1 algorithm~1) AND (grant ingersoll)  OR (tom morton)",
            "fields": ["_all", "summary^2"]
        }
    },
    "_source": [ "title", "summary", "authors" ],
    "highlight": {
        "fields" : {
            "summary" : {}
        }
    }
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 0.14558059,
        "_source": {
          "summary": "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization",
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "authors": [
            "grant ingersoll",
            "thomas morton",
            "drew farris"
          ]
        },
        "highlight": {
          "summary": [
            "organize text using approaches such as full-text <em>search</em>, proper name recognition, clustering, tagging, information extraction, and summarization"
          ]
        }
      }
    ]
 
```

### 11.简单查询字符串

该`simple_query_string`查询是该查询的一种版本`query_string`，它更适合在暴露给用户的单个搜索框中使用。它分别用+ / | /-代替了AND / OR / NOT的使用，并且丢弃了查询的无效部分，而不是在用户犯错时抛出异常。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "simple_query_string" : {
            "query": "(saerch~1 algorithm~1) + (grant ingersoll)  | (tom morton)",
            "fields": ["_all", "summary^2"]
        }
    },
    "_source": [ "title", "summary", "authors" ],
    "highlight": {
        "fields" : {
            "summary" : {}
        }
    }
} 
 
```

### 12.术语查询

以上示例是全文搜索的示例。有时，我们对结构化搜索更感兴趣，在结构化搜索中我们希望找到完全匹配并返回结果。在`term`与`terms`查询帮助我们在这里。在下面的示例中，我们正在搜索Manning Publications出版的索引中的所有书籍。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "term" : {
            "publisher": "manning"
        }
    },
    "_source" : ["title","publish_date","publisher"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 1.2231436,
        "_source": {
          "publisher": "manning",
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "publish_date": "2013-01-24"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 1.2231436,
        "_source": {
          "publisher": "manning",
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 1.2231436,
        "_source": {
          "publisher": "manning",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      }
    ]
 
```

可以使用`terms`关键字代替并传递搜索词数组来指定多个词。



```java
{
    "query": {
        "terms" : {
            "publisher": ["oreilly", "packt"]
        }
    }
} 
 
```

### 13.字词查询-排序

术语查询结果（与任何其他查询结果一样）可以轻松地进行排序。也允许多级排序



```java
POST /bookdb_index/book/_search
{
    "query": {
        "term" : {
            "publisher": "manning"
        }
    },
    "_source" : ["title","publish_date","publisher"],
    "sort": [
        { "publish_date": {"order":"desc"}},
        { "title": { "order": "desc" }}
    ]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": null,
        "_source": {
          "publisher": "manning",
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        },
        "sort": [
          1449100800000,
          "in"
        ]
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": null,
        "_source": {
          "publisher": "manning",
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        },
        "sort": [
          1396656000000,
          "solr"
        ]
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": null,
        "_source": {
          "publisher": "manning",
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "publish_date": "2013-01-24"
        },
        "sort": [
          1358985600000,
          "to"
        ]
      }
    ]
 
```

### 14.范围查询

另一个结构化查询示例是范围查询。在此示例中，我们搜索2015年出版的图书。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "range" : {
            "publish_date": {
                "gte": "2015-01-01",
                "lte": "2015-12-31"
            }
        }
    },
    "_source" : ["title","publish_date","publisher"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 1,
        "_source": {
          "publisher": "oreilly",
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 1,
        "_source": {
          "publisher": "manning",
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      }
    ]
 
```

**注意**：范围查询适用于日期，数字和字符串类型字段。



### 15.过滤查询

筛选查询允许您筛选查询结果。对于我们的示例，我们正在查询标题或摘要中带有“ Elasticsearch”一词的图书，但我们希望将搜索结果过滤为仅包含20条或更多评论的图书。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "filtered": {
            "query" : {
                "multi_match": {
                    "query": "elasticsearch",
                    "fields": ["title","summary"]
                }
            },
            "filter": {
                "range" : {
                    "num_reviews": {
                        "gte": 20
                    }
                }
            }
        }
    },
    "_source" : ["title","summary","publisher", "num_reviews"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.5955761,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "publisher": "oreilly",
          "num_reviews": 20,
          "title": "Elasticsearch: The Definitive Guide"
        }
      }
    ]
 
```

**注意**：筛选查询并不要求存在要对其进行筛选的查询。如果未指定`match_all`查询，则运行查询，该查询基本上返回索引中的所有文档，然后对其进行过滤。实际上，首先运行过滤器，以减少需要查询的表面积。此外，过滤器在首次使用后会被缓存，这使其性能非常好。

**更新**：已过滤的查询已从即将推出的Elasticsearch 5.0中删除，以支持bool查询。这是与上面相同的示例，改写为使用bool查询。返回的结果完全相同。

```java
POST /bookdb_index/book/_search
{
    "query": {
        "bool": {
            "must" : {
                "multi_match": {
                    "query": "elasticsearch",
                    "fields": ["title","summary"]
                }
            },
            "filter": {
                "range" : {
                    "num_reviews": {
                        "gte": 20
                    }
                }
            }
        }
    },
    "_source" : ["title","summary","publisher", "num_reviews"]
}
```


在下面的示例中，这也适用于多个过滤器。 

### 16.多个过滤器

可以通过使用过滤器来组合多个过滤`bool`器。在下一个示例中，过滤器确定返回的结果必须至少具有20条评论，不得在2015年之前发布，而应由oreilly发布。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "filtered": {
            "query" : {
                "multi_match": {
                    "query": "elasticsearch",
                    "fields": ["title","summary"]
                }
            },
            "filter": {
                "bool": {
                    "must": {
                        "range" : { "num_reviews": { "gte": 20 } }
                    },
                    "must_not": {
                        "range" : { "publish_date": { "lte": "2014-12-31" } }
                    },
                    "should": {
                        "term": { "publisher": "oreilly" }
                    }
                }
            }
        }
    },
    "_source" : ["title","summary","publisher", "num_reviews", "publish_date"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.5955761,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "publisher": "oreilly",
          "num_reviews": 20,
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      }
    ] 
 
```

### 17.功能评分：字段值因子

在某些情况下，您可能希望将文档中特定字段的值纳入相关性分数的计算中。在您希望根据文档的受欢迎程度提高其相关性的情况下，这是典型的情况。在我们的示例中，我们希望增加受欢迎的书籍（根据评论数判断）。使用`field_value_factor`功能评分可以做到这一点。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "function_score": {
            "query": {
                "multi_match" : {
                    "query" : "search engine",
                    "fields": ["title", "summary"]
                }
            },
            "field_value_factor": {
                "field" : "num_reviews",
                "modifier": "log1p",
                "factor" : 2
            }
        }
    },
    "_source": ["title", "summary", "publish_date", "num_reviews"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.44831306,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "num_reviews": 20,
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.3718407,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "num_reviews": 23,
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.046479136,
        "_source": {
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "num_reviews": 18,
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 0.041432835,
        "_source": {
          "summary": "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization",
          "num_reviews": 12,
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "publish_date": "2013-01-24"
        }
      }
    ]
 
```

**注意1**：我们本来可以运行常规`multi_match`查询并按num_reviews字段进行排序，但是这样就失去了进行相关性评分的好处。
**注意2**：还有许多其他参数可以调整对原始相关性得分的增强效果，例如“修饰符”，“因子”，“ boost_mode”等。这些参数在[Elasticsearch指南](https://www.elastic.co/guide/en/elasticsearch/guide/current/boosting-by-popularity.html)中进行了详细探讨。



### 18.作用分值：衰减功能

假设您不想以某个字段的值递增，而希望拥有理想的目标值，并且希望该增益因子随着距离该值的增加而衰减。这通常在基于纬度/经度，价格或日期等数字字段的提升中很有用。在我们精心设计的示例中，我们正在搜索理想情况下于2014年6月左右出版的“搜索引擎”书籍。



```java
POST /bookdb_index/book/_search
{
    "query": {
        "function_score": {
            "query": {
                "multi_match" : {
                    "query" : "search engine",
                    "fields": ["title", "summary"]
                }
            },
            "functions": [
                {
                    "exp": {
                        "publish_date" : {
                            "origin": "2014-06-15",
                            "offset": "7d",
                            "scale" : "30d"
                        }
                    }
                }
            ],
            "boost_mode" : "replace"
        }
    },
    "_source": ["title", "summary", "publish_date", "num_reviews"]
}

[Results]
"hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.27420625,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "num_reviews": 23,
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.005920768,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "num_reviews": 20,
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 0.000011564,
        "_source": {
          "summary": "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization",
          "num_reviews": 12,
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "publish_date": "2013-01-24"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.0000059171475,
        "_source": {
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "num_reviews": 18,
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      }
    ]
 
```

### 19.功能评分：脚本评分

如果内置的评分功能无法满足您的需求，则可以选择指定Groovy脚本进行评分。在我们的示例中，我们希望指定一个脚本，该脚本在决定要考虑评论数量的因素之前，先考虑publish_date。较新的书可能没有那么多的评论，因此不应因此受到惩罚。
评分脚本如下所示：



```java
publish_date = doc['publish_date'].value
num_reviews = doc['num_reviews'].value

if (publish_date > Date.parse('yyyy-MM-dd', threshold).getTime()) {
  my_score = Math.log(2.5 + num_reviews)
} else {
  my_score = Math.log(1 + num_reviews)
}
return my_score
 
```

要动态使用评分脚本，我们使用`script_score`参数



```java
POST /bookdb_index/book/_search
{
    "query": {
        "function_score": {
            "query": {
                "multi_match" : {
                    "query" : "search engine",
                    "fields": ["title", "summary"]
                }
            },
            "functions": [
                {
                    "script_score": {
                        "params" : {
                            "threshold": "2015-07-30"
                        },
                        "script": "publish_date = doc['publish_date'].value; num_reviews = doc['num_reviews'].value; if (publish_date > Date.parse('yyyy-MM-dd', threshold).getTime()) { return log(2.5 + num_reviews) }; return log(1 + num_reviews);"
                    }
                }
            ]
        }
    },
    "_source": ["title", "summary", "publish_date", "num_reviews"]
}

[Results]
"hits": {
    "total": 4,
    "max_score": 0.8463001,
    "hits": [
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "1",
        "_score": 0.8463001,
        "_source": {
          "summary": "A distibuted real-time search and analytics engine",
          "num_reviews": 20,
          "title": "Elasticsearch: The Definitive Guide",
          "publish_date": "2015-02-07"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "4",
        "_score": 0.7067348,
        "_source": {
          "summary": "Comprehensive guide to implementing a scalable search engine using Apache Solr",
          "num_reviews": 23,
          "title": "Solr in Action",
          "publish_date": "2014-04-05"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "3",
        "_score": 0.08952084,
        "_source": {
          "summary": "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "num_reviews": 18,
          "title": "Elasticsearch in Action",
          "publish_date": "2015-12-03"
        }
      },
      {
        "_index": "bookdb_index",
        "_type": "book",
        "_id": "2",
        "_score": 0.07602123,
        "_source": {
          "summary": "organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization",
          "num_reviews": 12,
          "title": "Taming Text: How to Find, Organize, and Manipulate It",
          "publish_date": "2013-01-24"
        }
      }
    ]
  }
 
```

**注意1**：要使用动态脚本，必须为`config/elasticsearch.yaml`文件中的Elasticsearch实例启用动态脚本。也可以使用存储在Elasticsearch服务器上的脚本。查阅[Elasticsearch参考文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting.html)了解更多信息。

**注意2** ：JSON无法包含嵌入的换行符，因此分号用于分隔语句。