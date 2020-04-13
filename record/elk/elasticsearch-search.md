# ElasticSearch 深入搜索

深入了解ElasticSearch搜索相关的内容.

## 基于词项和基于全文搜索

### 基于Term查询

`Term`:是表达语意的最小单位. 搜索和利用统计语言模型进行自然语言处理中都需要处理`Term`. 该查询的特点有:

- `Term`查询有: `Term Query`, `Range Query`, `Exists Query`, `Exists Query`, `Prefix Query`和`WildCast Query`.
- `ES`中对于`Term`查询是不会进行**分词处理**, 而是将输入作为一个整体, 在倒排索引中查找准确的词项, 并且使用相关度算分公式为每一个包含该词项的文档进行**相关度算分**.
- 可以通过设置`Constant Score`将查询转变为一个`Filtering`, `避免算分, 并利用缓存`, 提高性能.

例子:

```json
// 插入三条记录到 products索引下
POST /products/_doc/_bulk
{ "index": {"_id":1}}
{"productID": "FJ3", "desc": "iPhone"}
{ "index": {"_id":2}}
{"productID": "KL5", "desc": "iPad"}
{ "index": {"_id":3}}
{"productID": "PV7", "desc": "MBP"}

// Term搜索, 描述为iphone(可以正确获取)
POST /products/_search
{
  "query": {
    "term": {
      "desc": {
        "value": "iphone"
      }
    }
  }
}

// Term搜索, 描述为iPhone(获取失败,为空)
// 因为插入文档时, ES会对desc做分词处理生成倒排索引, 全部进行小写
POST /products/_search
{
  "query": {
    "term": {
      "desc": {
        "value": "iPhone"
      }
    }
  }
}

// Term搜索, 描述关键词为iPhone
// 通过关键词可以保证完全匹配,不受到分词影响
POST /products/_search
{
  "query": {
    "term": {
      "desc.keyword": {
        "value": "iPhone"
      }
    }
  }
}

// Term搜索, 使用constant_score变成过滤, 避免算分提高速度
POST /products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "productID.keyword": {
            "value": "KL5"
          }
        }
      }
    }
  }
}
```

### 基于全文本查询

- 主要的查询有: `Match Query`, `Match Phrase Query`, `Query String Query`.
- 索引和搜索都会进行分词处理, 查询的字符串会首先进行分词, 生成一个供查询的词项列表.
- 查询的时候, 先会对输入的查询进行分词, 然后各个词项逐个进行底层查询, 最终将结果合并, 并为每一个文档进行算分. 如查询`Matrix reloaded`会查询到所有包含`Matrix`或者`Reload`的结果.

例子:

```json
// Match Query, 电影标题为: Matrix reloaded.(或关系, 存在包含即可)
POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Matrix reloaded"
      }
    }
  }
}


// Match Query, 限制电影标题为: Matrix reloaded.(与关系, 同时包含)
POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Matrix reloaded",
        "operator": "and"
      }
    }
  }
}

// Match Query, 限制电影标题为: Matrix reloaded.
// 使用minimum_should_match, 做到更细致的颗粒度
POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Matrix reloaded",
        "minimum_should_match": 2
      }
    }
  }
}

// Match Pharse查询, 限制电影标题为: Matrix reloaded.
// 使用slop, 限制两个单词之间的距离
POST movies/_search
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "Matrix reloaded",
        "slop": 1
      }
    }
  }
}


// Query String查询, 限制电影标题为: Matrix reloaded或者为Vampire.
POST movies/_search
{
  "query": {
    "query_string" : {
        "query" : "(Matrix reloaded) OR (Vampire)",
        "default_field" : "title"
    }
  }
}
```

## 结构化搜索

结构化搜索即对`结构化数据`的搜索.

结构化数据:

- 日期,布尔类型和数字类型都是结构化的: 这类的数据有精确的格式, 我们可以对这些数据进行逻辑操作, 如比较数字和时间范围, 判定两个值的大小.
- 文本也可以是结构化的: 对于这类的数据可以做到精确匹配或者部分匹配(Term查询和Prefix前缀查询)

  - 如彩色笔的颜色集合: 红, 绿, 蓝.
  - 博得标签.
  - 商品的UPCs等.

- 结构化搜索的结果只有`是`和`否`两个值: 根据具体的场景确定是否需要打分.

例子:

```json
// 清空并插入一些商品数据, 这里包含了很多结构化的数据(如日期和布尔值).
DELETE products

POST /products/_doc/_bulk
{ "index": {"_id":1}}
{"productID": "FJ3", "desc": "iPhone", "price": 10, "avaliable": true, "date": "2018-01-01"}
{ "index": {"_id":2}}
{"productID": "FK4", "desc": "abc", "price": 20, "avaliable": true, "date": "2019-01-01"}
{ "index": {"_id":3}}
{"productID": "PV7", "desc": "MBP", "price": 30, "avaliable": false}

// term 查询过滤布尔值
POST products/_search
{
  "profile": true,
  "explain": true,
  "query": {
    "term": {
      "avaliable": true
    }
  }
}

// term 查询, 使用`constant_score`跳过算分
POST products/_search
{
  "profile": true,
  "explain": true,
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "avaliable": true
        }
      }
    }
  }
}

// 对价格进行范围查询
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "price": {
            "gte": 20,
            "lte": 30
          }
        }
      }
    }
  }
}

// 对日期进行范围查询
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "date": {
            "gte": "now-2y"
          }
        }
      }
    }
  }
}

// 查询是否存在(null判断)
POST products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "exists": {
          "field": "date"
        }
      }
    }
  }
}
```

## 搜索的相关性算分

搜索的相关性算分: 描述一个**文档**和**查询语句**匹配的程度, ES会对每个匹配查询条件的结果进行算分`_score`. 算分的本质就是用于排序, 把最符合用户需求的文档排在前面. 在`ES5`之前默认使用的相关性算法为`TF-IDF`, 现在默认使用`BM 25`.

简单介绍`TF-IDF`相关的概念:

`TF`: 词频, `Term Frequency`, 检索词在一篇文档中出现的频率, 即`词出现的次数`除以`文档的总字数`. 其中一个简单的相关性算法是将所有的分词的`TF`相加, 即为得分. 当然如果分词中包含一些`Stop Word`(如`的`, `是`等)应该去除掉.

`IDF`: 逆文档频率. `DF`: 检索词在所有文档中出现的频率. 如`区块链`在相对较少的文档中出现, 而`应用`在相对较多的文档中出现.`Stop Word`在大量的文档中出现. 而`IDF`就等价于: `log(所有文档数/词出现的文档数)`, 即出现的文档数越少, 权重越大, 也更加重要.

`TF-IDF`实际上就是对所有的分词进行加权求和(以`区块链的应用`为例): `TF(区块链)*IDF(区块链) + TF(的)*IDF(的) + TF(应用)*IDF(应用)`.

`TF-IDF`被公认为信息检索领域的最重要的发明, 被广泛应用于信息检索, 文献分类和其他相关领域.现在搜索引擎基本上都是基于`TF-IDF`算法, 并对其进行了大量细微优化.

在`Lucene`中`TF-IDF`评分公式为:

![TF-IDF](https://image.cjyong.com/TF-IDF.jpg)

`BM25`和`TF-IDF`区别在于, `TF-IDF`在`TF`无限增加时, 得分会不断往上增长, 而`BM25`则会趋近于一个常数.

`BM25`评分公式为:

![TF-IDF](https://image.cjyong.com/BM25.jpg)

可以在设置索引的时候定制化`BM25`:

```json
PUT /my_index
{
  "settings": {
    "similarity": {
      "custom_similarity": {
        "type": "BM25",
        // b默认值为0.75（0-1），0表示禁止Normalization
        "b": 0,
        // k默认值为1.2, 越高饱和度越低
        "k1": 2
      }
    }
  }
}
```

当需要查看查询评分细节时, 只需要打开`explain: true`即可.

例子:

```json
// 插入几条数据
PUT testscore/_bulk
{ "index": { "_id": 1 }}
{ "content":"we use Elasticsearch to power the search" }
{ "index": { "_id": 2 }}
{ "content":"we like elasticsearch" }
{ "index": { "_id": 3 }}
{ "content":"The scoring of documents is caculated by the scoring formula" }
{ "index": { "_id": 4 }}
{ "content":"you know, for search" }

// 查询内容, 打开explain, 查看具体的算分细节
// 这里记录2和记录1,依次返回(记录2得分更高)
POST /testscore/_search
{
  "explain": true,
  "query": {
    "match": {
      "content": "elasticsearch"
    }
  }
}

// 设置boosting, 动态控制评分标准
// 这里通过设置negative boosting降低了like出现之后的得分
// 这样记录1的得分会比记录2得分高(记录1出现在前面)
POST /testscore/_search
{
  "query": {
    "boosting": {
      "positive": {
        "term": {
          "content": "elasticsearch"
        }
      },
      "negative": {
        "term": {
          "content": "like"
        }
      },
      "negative_boost": 0.2
    }
  }
}
```

## 多字段查询

在ES中查询主要分为两种: `Query Context`涉及到相关性算分, `Filter Context`不需要进行相关性算分, 可以利用缓存, 性能更好. 当我们需要进行一些多字段查询时, 就需要组合两种情况来实现复合的查询和更好的性能.

### bool 查询

`bool查询`是一个或者多个查询子语句的组合.支持四种查询子语句:

- `must`: 必须匹配, 贡献算分.
- `should`: 选择性匹配, 贡献算分.
- `must_not`: `Filter Context`, 必须不匹配, 不贡献算分.
- `filter`: `Filter Context`, 必须匹配, 不贡献算分.

例子:

```json
// 清空并插入数据
DELETE products
POST /products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }

POST /products/_search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "price" : "30" }
      },
      "filter": {
        "term" : { "avaliable" : "true" }
      },
      "must_not" : {
        "range" : {
          "price" : { "lte" : 10 }
        }
      },
      "should" : [
        { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
        { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
      ],
      "minimum_should_match" :1
    }
  }
}
```

利用`bool查询`可以解决数组匹配时, 是包含而不是等于的情况(即只要一个数组对象中包含搜索条件, 就一定会被匹配到), 解决方案为, 添加新的计数字段, 然后通过组合条件定位只包含该类型的数组.

```json
// 如这里的genre是类型, 是一个数组类型, 添加一个计数字段.
POST /newmovies/_bulk
{ "index": { "_id": 1 }}
{ "title" : "Father of the Bridge Part II","year":1995, "genre":"Comedy","genre_count":1 }
{ "index": { "_id": 2 }}
{ "title" : "Dave","year":1993,"genre":["Comedy","Romance"],"genre_count":2 }

// 通过must子查询语句, 进行算分排序
POST /newmovies/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"genre.keyword": {"value": "Comedy"}}},
        {"term": {"genre_count": {"value": 1}}}
      ]
    }
  }
}

// 通过filter子查询语句, 强制过滤, 不进行算分
POST /newmovies/_search
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"genre.keyword": {"value": "Comedy"}}},
        {"term": {"genre_count": {"value": 1}}}
        ]
    }
  }
}
```

同时`bool`查询支持嵌套查询:

```json
POST /products/_search
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "price": "30"
        }
      },
      "should": [
        {
          "bool": {
            "must_not": {
              "term": {
                "avaliable": "false"
              }
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

注意嵌套查询会影响打分情况, 同一层级具备相同的权重, 而子一层级的权重是低于父级的权重的. 如果需要单独控制同一层级的各个字段的权重, 那么可以使用`boost`参数:

```json
POST blogs/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {
          "title": {
            "query": "apple,ipad",
            "boost": 1.1
          }
        }},
        {"match": {
          "content": {
            "query": "apple,ipad",
            "boost":2 // 设置成2权重比上一个1.1要高
          }
        }}
      ]
    }
  }
}
```

## 单字符串多字段查询

单字符串多字段查询是非常常见的, 如百度和google搜索的时候, 只需要输入一个字符串, 搜索引擎会默认查找多个字段进行匹配. 当我们查询一个内容, 在多个字段内进行搜索时, 如何合理的进行调优呢?

### Disjunction Max Query

```json
// 插入两条数据
PUT /blogs/_doc/1
{
  "title": "Quick brown rabbits",
  "body": "Brown rabbits are commonly seen."
}

PUT /blogs/_doc/2
{
  "title": "Keeping pets healthy",
  "body": "My quick brown fox eats rabbits on a regular basis."
}

// 在title和body中进行搜索
POST /blogs/_search
{
  "query": {
    "bool": {
      "should": [
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

这时候`ES`默认的处理方式是什么呢?

- 查询 should 语句句中的两个查询
- 加和两个查询的评分
- 乘以匹配语句句的总数
- 除以所有语句句的总数

也就是两者具备同样的权重进行加和处理. 但是默认的处理方式往往不能满足我们的要求, 如上面的例子我们想要搜索一个关于`Brown fox`的内容, 文档2按道理应该更加匹配内容. 但是由于算分差异导致文档1得分更高: 文档2的标题没有包含任何分词, 得分非常低, 相加之和反而不如文档1得分高.

在这种情况下, 不应该简单的进行相加处理. 因为`title`和`body`应该是竞争关系, 不应该简单相加, 而应该找到单个匹配最佳的字段评分. 这时候就可以使用`Disjunction Max Query`:

```json
// 使用dis_max进行查询, 返回匹配程度最高的字段评分
POST /blogs/_search
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

虽然`Disjunction Max Query`可以解决大部分的问题, 在某些情况下还是有些过于极端: 有时候同时匹配多个字段的内容, 相关度还是更高的. 如查询`Quick pets`, 这时候文档1应该具备更好的匹配程度, 结果却先返回文档2\. 这时候可以使用`tie_breaker`参数:

1. 获得最佳匹配语句句的评分 _score 。
2. 将其他匹配语句句的评分与 tie_breaker 相乘
3. 对以上评分求和并规范化

`tie_breaker`介于0-1之间, 0代表使用最佳匹配, 1表示其他字段同等重要.

```json
// 使用Disjunction Max Query不能很好的满足要求
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {
          "match": {
            "title": "Quick pets"
          }
        },
        {
          "match": {
            "body": "Quick pets"
          }
        }
      ]
    }
  }
}

// 添加tie_breaker进行加权处理
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {
          "match": {
            "title": "Quick pets"
          }
        },
        {
          "match": {
            "body": "Quick pets"
          }
        }
      ],
      "tie_breaker": 0.2
    }
  }
}
```

### Multi Match Query

当我们进行单字符串多字段查询时, 主要可以分为以下三种情况:

- `最佳字段(Best Fields)`: 就如前面所说在body和title中查找最优的字段, 和前面的`disjunction max query`类似.
- `多数字段(Most Fields)`: 处理理英⽂文内容时:⼀一种常⻅见的⼿手段是，在主字段( English Analyzer)，抽取词⼲干，加⼊入同义词，以 匹配更更多的⽂文档。相同的⽂文本，加⼊入⼦子字段(Standard Analyzer)，以提供更更加精确的匹配。其他字 段作为匹配⽂文档提⾼高相关度的信号。匹配字段越多则越好.
- `混合字段 (Cross Field)`: 对于某些实体，例例如⼈人名，地址，图书信息。需要在多个字段中确定信息，单个字段只能作为整体 的⼀一部分。希望在任何这些列列出的字段中找到尽可能多的词.

**最佳字段**:

```json
POST /blogs/_search
{
  "query": {
    "multi_match": {
      // 默认为最佳字段
      "type": "best_fields", 
      "query": "Quick pets", 
      "fields": ["title", "body"], 
      "tie_breaker": 0.2,
      "minimum_should_match": "20%"
    }
  }
}

// 等价于
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {
          "match": {
            "title": "Quick pets"
          }
        },
        {
          "match": {
            "body": "Quick pets"
          }
        }
      ],
      "tie_breaker": 0.2
    }
  }
}
```

**多数字段**:

```json
// 插入数据
POST titles/_bulk
{ "index": { "_id": 1 }}
{ "title": "My dog barks" }
{ "index": { "_id": 2 }}
{ "title": "I see a lot of barking dogs on the road " }

// 当我搜索barking dogs时, 分词将barking dogs处理成: bark, dog. 导致第一个文档得分更高.
// 实际上第二个文档更加符合要求
GET titles/_search
{
  "query": {
    "match": {
      "title": "barking dogs"
    }
  }
}

// 处理方案添加stard分词器, 将结果存储在std字段中
DELETE /titles
PUT /titles
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "std": {
            "type": "text",
            "analyzer": "standard"
          }
        }
      }
    }
  }
}

//插入数据
POST titles/_bulk
{ "index": { "_id": 1 }}
{ "title": "My dog barks" }
{ "index": { "_id": 2 }}
{ "title": "I see a lot of barking dogs on the road " }

// 使用多数字段进行匹配.
GET /titles/_search
{
  "query": {
    "multi_match": {
      "query": "barking dogs",
      "type": "most_fields",
      "fields": [
        "title",
        "title.std"
      ]
    }
  }
}

// 设置title权重, 降低其他字段优先级
GET /titles/_search
{
  "query": {
    "multi_match": {
      "query": "barking dogs",
      "type": "most_fields",
      "fields": [
        "title^10",
        "title.std"
      ]
    }
  }
}
```

## SearchTemplate和Index Alias查询

### SearchTemplate

`Search Template`允许你创建一些预定义好的查询模板, 其他人使用的时候, 只需要简单的传递参数就可以了. 这样可以方便的解耦开发人员和使用人员, 使用人员不需要知道底层的实现和优化. 而开发人员可以在他人无感知的情况下对查询进行优化和更新等.

例子:

```json
// 创建一个ID为tmdb的查询模板
POST _scripts/tmdb
{
  "script": {
    // 指定模板语言为: mustache
    "lang": "mustache",
    "source": {
      "_source": [
        "title",
        "overview"
      ],
      // 设置搜索文档数量为20个
      "size": 20,
      // 设置查询语法
      "query": {
        "multi_match": {
          // 设置查询内容为参数q
          "query": "{{q}}",
          "fields": [
            "title",
            "overview"
          ]
        }
      }
    }
  }
}

// 使用模板进行查询, 指定ID和传递参数q即可
POST tmdb/_search/template
{
    "id":"tmdb",
    "params": {
        "q": "basketball with cartoon aliens"
    }
}
```

### Index Alias

`Index Alias`通过给索引一个别名, 可以在用户无感知的情况下对索引进行更新或者替换,实现零停机运维.

例子:

```json
// 创建为movies-2019创建一个别名movies-latest
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-latest"
      }
    }
  ]
}

// 使用别名进行查询
POST movies-latest/_search
{
  "query": {
    "match_all": {}
  }
}

// 切换别名所对应的索引
POST _aliases
{
  "actions": [
    {
      "remove": {
        "index": "movies-2019",
        "alias": "movies-latest"
      }, 
      "add": {
        "index": "movies-2020",
        "alias": "movies-latest"
      }
    }
  ]
}

// 添加别名的时候设置过滤
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-lastest-highrate",
        "filter": {
          "range": {
            "rating": {
              "gte": 4
            }
          }
        }
      }
    }
  ]
}
```

## 综合排序:Function Score Query 优化算分

`ES`默认是以文档的相关度进行评分排序, 可以指定一个或者多个字段进行排序. 但是在某些情境下却不太适合: 一些变量虽然不能影响相关度, 但是也应该作为一个参考变量时, 默认的排序算法就不能满足要求了. 如: 搜索博客内容时, 我们希望在相同相关度的博客中, 点赞数高的博客放在前面. 这时候点赞数就应该作为一个参考变量影响评分.

为了处理这种情况, 我们可以使用`Function Score Query`: 对已经查询好的文档(已经按照相关度排好序), 进行一系列的重新算分, 根据新生成的分数进行排序. `Function Score Query`提供了以下几种默认计算分值的函数:

- **Weight** : 为每⼀一个⽂文档设置⼀一个简单⽽而不不被规范化的权重.
- **Field Value Factor**: 使⽤用该数值来修改 _score，例例如将 "热度"和"点赞数"作为算分的参考因素
- **Random Score**: 为每⼀一个⽤用户使⽤用⼀一个不不同的，随机算分结果.
- **衰减函数**: 以某个字段的值为标准，距离某个值越近，得分越⾼高.
- **Script Score**:⾃自定义脚本完全控制所需逻辑.

例子:

```json
// 插入三篇类似的博客(只有点赞数不同)
DELETE blogs
PUT /blogs/_doc/1
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   0
}

PUT /blogs/_doc/2
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   100
}

PUT /blogs/_doc/3
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   1000000
}

// 使用function_score, 将点赞数作为乘数
// 这里是简单的相乘, 新得分 = 旧得分 * 投票数
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes"
      }
    }
  }
}

// 添加modier,让得分更加平滑
// 新的算分=⽼的算分*log(1+投票数)
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p"
      }
    }
  }
}

// 添加factor, 更加平滑
// 新的算分=⽼的算分*log(1+factor * 投票数)
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p",
        "factor": 0.1
      }
    }
  }
}

// 添加boost_mode, 设置和的最大值不超过3
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p" ,
        "factor": 0.1
      },
      "boost_mode": "sum",
      "max_boost": 3
    }
  }
}

// 设置随机获取文章, 这里的随机的次序取决于seed
// seed不变, 随机的顺序也就不变
// 常用于投放随机广告
POST /blogs/_search
{
  "query": {
    "function_score": {
      "random_score": {
        "seed": 911119
      }
    }
  }
}
```

## Elasticsearch Suggester API

- 现代的搜索引擎，一般都会提供 `Suggest as you type` 的功能
- 帮助⽤用户在输⼊入搜索的过程中，进行自动补全或者纠错。通过协助⽤用户输⼊更加精准的关键 词，提⾼高后续搜索阶段⽂文档匹配的程度.
- 在 `google`上搜索，一开始会⾃自动补全。当输⼊到⼀定⻓长度，如因为单词拼写错误⽆法补全， 就会开始提示相似的词或者句⼦.

`ES`也提供了类似的功能, 通过`Suggester API`实现: 将输入的文本分解为`Token`, 然后在索引字典中查找类似的`Term`进行返回. 根据不同的场景, 主要分为以下四种`Suggesters`:

- Term & Phrase Suggester
- Complete & Context Suggester

### Term Suggester

`ES`会未每一个`Term`进行算分, 通过`Levenshtein Edit Distance`的算法实现的。核⼼心思想就是⼀个词改动多少字符就可以和另外一个词一致.提供了了很多可选参数来控制相似性的模糊程度。例如`max_edits`. 其中提供了几种常见的`Suggestion Mode`:

- `Missing` – 如索引中已经存在，就不提供建议.
- `Popular` – 推荐出现频率更加⾼的词.
- `Always` – ⽆论是否存在，都提供建议.

例子:

```json
// 插入文章数据
DELETE articles
POST articles/_bulk
{ "index" : { } }
{ "title_completion": "lucene is very cool"}
{ "index" : { } }
{ "title_completion": "Elasticsearch builds on top of lucene"}
{ "index" : { } }
{ "title_completion": "Elasticsearch rocks"}
{ "index" : { } }
{ "title_completion": "elastic is the company behind ELK stack"}
{ "index" : { } }
{ "title_completion": "Elk stack rocks"}
{ "index" : {} }

// 搜索lucen rock的推荐词, 其中lucen返回lucene而rock没有
POST /articles/_search
{
  "suggest": {
    "term-suggestion": {
      "text": "lucen rock",
      "term": {
        "suggest_mode": "missing",
        "field": "body"
      }
    }
  }
}

// 使用popular模式, 其中lucen返回lucene而rock返回rocks(出现的频率更高)
POST /articles/_search
{
  "suggest": {
    "term-suggestion": {
      "text": "lucen rock",
      "term": {
        "suggest_mode": "popular",
        "field": "body"
      }
    }
  }
}

// 设置按照频率进行排序, 设置允许前缀不一致. 其中hock返回rock
POST /articles/_search
{
  "suggest": {
    "term-suggestion": {
      "text": "lucen hock",
      "term": {
        "suggest_mode": "always",
        "field": "body",
        "prefix_length":0,
        "sort": "frequency"
      }
    }
  }
}
```

### Pharse Suggester

对语句进行推荐, 常见的参数有:

- **Suggest Mode**: missing, popular, always
- **Max Errors**: 最多可以拼错的 `Terms` 数
- **Confidence**: 限制返回结果数，默认为 1.

例子:

```json
POST /articles/_search
{
  "suggest": {
    "my-suggestion": {
      "text": "lucne and elasticsear rock hello world ",
      "phrase": {
        "field": "body",
        "max_errors":2,
        "confidence":0,
        "direct_generator":[{
          "field":"body",
          "suggest_mode":"always"
        }],
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```

### Complete Suggester

`Complete Suggester`提供了一种自动完成(`Auto Complete`)的功能: 用户每输入一个字符, 就需要即时发送一个请求给`ES`查询匹配项.

- 对性能要求⽐比较苛刻 。`Elasticsearch` 采⽤用了了不同的数据结构，并非通过倒排索引来完成。 ⽽是将 `Analyze`的数据编码成`FST` 和索引⼀起存放。`FST` 会被 `ES` 整个加载进内存， 速度很快.
- `FST` 只能⽤用于前缀查找
- 使用前需要设置索引的`mapping`, 定义自动完成类型.

例子:

```json
// 清空索引
DELETE articles
// 设置title_completion字段提供自动完成功能
PUT articles
{
  "mappings": {
    "properties": {
      "title_completion":{
        "type": "completion"
      }
    }
  }
}
// 插入数据
POST articles/_bulk
{ "index" : { } }
{ "title_completion": "lucene is very cool"}
{ "index" : { } }
{ "title_completion": "Elasticsearch builds on top of lucene"}
{ "index" : { } }
{ "title_completion": "Elasticsearch rocks"}
{ "index" : { } }
{ "title_completion": "elastic is the company behind ELK stack"}
{ "index" : { } }
{ "title_completion": "Elk stack rocks"}
{ "index" : {} }

// 获取elk的自动完成建议
POST articles/_search?pretty
{
  "size": 0,
  "suggest": {
    "article-suggester": {
      "prefix": "elk ",
      "completion": {
        "field": "title_completion"
      }
    }
  }
}
```

### Context Suggester

`Context Suggester`是对`Completion Suggester`的一个拓展, 提供了不同上下文下的自动完成信息. 例如，输⼊入`star`:

- 咖啡相关:建议 "Starbucks"
- 电影相关:"star wars"

例子:

```json
// 清空comments索引
DELETE comments
PUT comments
// 设置category存在不同的上下文
PUT comments/_mapping
{
  "properties": {
    "comment_autocomplete":{
      "type": "completion",
      "contexts":[{
        "type":"category",
        "name":"comment_category"
      }]
    }
  }
}

// 设置该条评论的上下文为movies
POST comments/_doc
{
  "comment":"I love the star war movies",
  "comment_autocomplete":{
    "input":["star wars"],
    "contexts":{
      "comment_category":"movies"
    }
  }
}

// 设置该条评论的上下文为coffee
POST comments/_doc
{
  "comment":"Where can I find a Starbucks",
  "comment_autocomplete":{
    "input":["starbucks"],
    "contexts":{
      "comment_category":"coffee"
    }
  }
}

// 传递上下文信息和前缀搜索
POST comments/_search
{
  "suggest": {
    "MY_SUGGESTION": {
      "prefix": "sta",
      "completion":{
        "field":"comment_autocomplete",
        "contexts":{
          "comment_category":"coffee"
        }
      }
    }
  }
}
```

## 跨集群搜索例子

一个跨集群搜索的例子:

```json
// 启动三个节点, 分别指定节点名,节点类型和端口
bin/elasticsearch -E node.name=cluster0node -E cluster.name=cluster0 -E path.data=cluster0_data -E discovery.type=single-node -E http.port=9200 -E transport.port=9300

bin/elasticsearch -E node.name=cluster1node -E cluster.name=cluster1 -E path.data=cluster1_data -E discovery.type=single-node -E http.port=9201 -E transport.port=9301

bin/elasticsearch -E node.name=cluster2node -E cluster.name=cluster2 -E path.data=cluster2_data -E discovery.type=single-node -E http.port=9202 -E transport.port=9302

//在每个集群上设置动态的设置, 集群的信息
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster0": {
          "seeds": [
            "127.0.0.1:9300"
          ],
          "transport.ping_schedule": "30s"
        },
        "cluster1": {
          "seeds": [
            "127.0.0.1:9301"
          ],
          "transport.compress": true,
          "skip_unavailable": true
        },
        "cluster2": {
          "seeds": [
            "127.0.0.1:9302"
          ]
        }
      }
    }
  }
}

#cURL
curl -XPUT "http://localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'

curl -XPUT "http://localhost:9201/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'

curl -XPUT "http://localhost:9202/_cluster/settings" -H 'Content-Type: application/json' -d'
{"persistent":{"cluster":{"remote":{"cluster0":{"seeds":["127.0.0.1:9300"],"transport.ping_schedule":"30s"},"cluster1":{"seeds":["127.0.0.1:9301"],"transport.compress":true,"skip_unavailable":true},"cluster2":{"seeds":["127.0.0.1:9302"]}}}}}'

#创建测试数据
curl -XPOST "http://localhost:9200/users/_doc" -H 'Content-Type: application/json' -d'
{"name":"user1","age":10}'

curl -XPOST "http://localhost:9201/users/_doc" -H 'Content-Type: application/json' -d'
{"name":"user2","age":20}'

curl -XPOST "http://localhost:9202/users/_doc" -H 'Content-Type: application/json' -d'
{"name":"user3","age":30}'

#查询 正确返回三条数据
GET /users,cluster1:users,cluster2:users/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 20,
        "lte": 40
      }
    }
  }
}
```
