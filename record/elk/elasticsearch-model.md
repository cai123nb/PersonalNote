# ElasticSearch建模之最佳实践

![elk-model](https://image.cjyong.com/elk-model.png)

本章主要关注`Elasticsearch`中对象的存储, 维护和预处理等基本功能, 以及利用所学的知识点进行合理的建模实践, 以及在实践中需要关注的一些问题.

## 对象及Nested对象

在现实世界中, 很多数据具备很强的关联关系, 如:

- 博客/作者/评论
- 银⾏账户有多次交易记录
- 客户有多个银⾏账户
- 目录⽂件有多个⽂件和⼦目录

在关系数据中处理的方式: 使用各种合理的范式进行设计, 不同的数据存放在不同的表中, 通过`join`的方式进行数据关联.

- 1NF: 所有属性无法再进行分解.(消除非主属性对键的部分函数依赖)
- 2NF: 基于1NF, 所有其它字段都完全依赖主键.(消除⾮主要属性对键的传递函数依赖)
- 3NF: 基于2NF, 消除了非主属性对于主键的传递函数依赖.(消除主属性对键的传递函数依赖)
- BCNF: 基于3NF, 主键之间不存在依赖关系.(主属性不依赖于主属性)

这样的设计存在优点和缺点:

- 优点1: 减少了不必要的更新.
- 优点2: 节省了存储空间.
- 缺点1: 有时候面临"查询缓慢的问题": 范式化程度越高, join表越多.

在面对这些范式化的一些问题时, 出现了很多反范式化设计(Denormalization): 不使用关联关系, 而在文档中保存冗余的数据拷贝:

- 优点1: ⽆需处理 Joins 操作，数据读取性能好
- 缺点1: 不适合频繁更新情况.(一处更新, 多处改动)

在`ES`中优先考虑反范式化, 因为`ES`不擅长处理关联关系, 一般采用以下四种方法处理关联:

- 对象类型
- 嵌套对象(Nested Object)
- ⽗子关联关系(Parent / Child )
- 应⽤端关联

### 对象类型存储

这里以博客内容和评论为例, 将两者存储在一个对象里面:

```java
// 设置tblog的 Mapping
PUT /tblog
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text"
      },
      "time": {
        "type": "date"
      },
      "user": {
        "properties": {
          "city": {
            "type": "text"
          },
          "userid": {
            "type": "long"
          },
          "username": {
            "type": "keyword"
          }
        }
      }
    }
  }
}

PUT blog/_doc/1
{
  "content":"I like Elasticsearch",
  "time":"2019-01-01T00:00:00",
  "user":{
    "userid":1,
    "username":"Jack",
    "city":"Shanghai"
  }
}

// 根据用户名和作者信息, 搜索博客信息
POST blog/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"content": "Elasticsearch"}},
        {"match": {"user.username": "Jack"}}
      ]
    }
  }
}
```

可以看出, 使用对象来存储关联的信息在查询的时候是非常方便的. 但是存在一点不好的就是, 当作者信息发生变更时, 需要更新所有该作者的博客信息. 所以不适合在频繁更新的场景使用.

另外, 对象存储在面对包含对象数组时还存在异常情况:

```java
// 写入一条电影信息
POST my_movies/_doc/1
{
  "title":"Speed",
  "actors":[
    {
      "first_name":"Keanu",
      "last_name":"Reeves"
    },

    {
      "first_name":"Dennis",
      "last_name":"Hopper"
    }

  ]
}

// 这时候根据电影演员进行查询, 指定名称为为 Keanu - Hopper
// 按照常理是不存在这个演员的, 但是结果确返回了两个演员: Keanu - Reeves 和 Dennis - Hopper
POST my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"actors.first_name": "Keanu"}},
        {"match": {"actors.last_name": "Hopper"}}
      ]
    }
  }

}
```

这明显是存在问题的, 那是因为`ES`在处理内部对象时, 内部对象会被处理成扁平式键值对的结构, 如上面的电影, 会被处理成:

```java
"title": "speed",
"actor.first_name": ["Keanu", "Dennis"]
"actor.second_name": ["Reeves", "Hopper"]
```

这会导致在多字段查询时, 往往会出现异常的结果. 这时候可以使用`嵌套对象`来解决这种问题.

### 嵌套对象(Nested Object)存储

- Nested 数据类型:允许对象数组中的对象被独⽴立索引
- 使⽤ nested 和 properties 关键字，将所有 actors 索引到多个分隔的⽂文档
- 在内部， Nested ⽂档会被保存在两个`Lucene`文档中，在查询时做`Join`处理

这里以电影和演员为例, 将演员信息嵌套放在电影对象里面.

```json
// 设置movies的mapping信息
PUT my_movies
{
      "mappings" : {
      "properties" : {
        "actors" : {
          // 这里指定 actors 属性的类型是 nested
          "type": "nested",
          "properties" : {
            "first_name" : {"type" : "keyword"},
            "last_name" : {"type" : "keyword"}
          }},
        "title" : {
          "type" : "text",
          "fields" : {"keyword":{"type":"keyword","ignore_above":256}}
        }
      }
    }
}

// 插入一条数据
POST my_movies/_doc/1
{
  "title":"Speed",
  "actors":[
    {
      "first_name":"Keanu",
      "last_name":"Reeves"
    },

    {
      "first_name":"Dennis",
      "last_name":"Hopper"
    }

  ]
}

// 嵌套查询, 这时候正确返回空数组对象
POST my_movies/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "Speed"
          }
        },
        {
          // 这里指定 nested 类型查询
          "nested": {
            // 指定路径为 actors
            "path": "actors",
            "query": {
              "bool": {
                "must": [
                  {
                    "match": {
                      "actors.first_name": "Keanu"
                    }
                  },
                  {
                    "match": {
                      "actors.last_name": "Hopper"
                    }
                  }
                ]
              }
            }
          }
        }
      ]
    }
  }
}

// 普通的聚合不生效
POST my_movies/_search
{
  "size": 0,
  "aggs": {
    "NAME": {
      "terms": {
        "field": "actors.first_name",
        "size": 10
      }
    }
  }
}

// 需要指定嵌套路径
POST my_movies/_search
{
  "size": 0,
  "aggs": {
    "actors": {
      "nested": {
        "path": "actors"
      },
      "aggs": {
        "actor_name": {
          "terms": {
            "field": "actors.first_name",
            "size": 10
          }
        }
      }
    }
  }
}
```

## ⽂档的⽗子关系

在前面了解到了`对象`和`Nested 对象`的应用场景. 但是两者都存在一个弊端: `每次更新，需要重新索引整个对象(包括根对象和嵌套对象)`. 为了解决这种情况, `ES`也提供了类似关系型数据中`Join`的实现, 通过维护`父子`关系, 将两者进行分离, 两者将完全独立, 互不影响.

### 设置索引Mapping

```java
PUT my_blogs
{
  "mappings": {
    "properties": {
      "blog_comments_relation": {
        // 设置父子关联, 父为 blog, 子为 comment, 两者的关系通过join连接
        "type": "join",
        "relations": {
          "blog": "comment"
        }
      },
      "content": {
        "type": "text"
      },
      "title": {
        "type": "keyword"
      }
    }
  }
}
```

### 索引父文档

```java
PUT my_blogs/_doc/blog1
{
  "title":"Learning Elasticsearch",
  "content":"learning ELK @ geektime",
  // 在关联列表中指定, 当前身份为 父blog
  "blog_comments_relation":{
    "name":"blog"
  }
}
```

### 索引子文档

```java
// 注意这里指定了 routing , 保证其和父文档放在同一个分片中 
PUT my_blogs/_doc/comment1?routing=blog1
{
  "comment":"I am learning ELK",
  "username":"Jack",
  // 在关联列表中指定, 当前身份为 子comment, 管理的父blog的id为 blog1
  "blog_comments_relation":{
    "name":"comment",
    "parent":"blog1"
  }
}
```

### 按需查询

父子文档支持以下几种查询:

- 查询所有⽂文档
- Parent Id 查询
- Has Child 查询
- Has Parent 查询

```java
// 查询所有文档, 包括所有父文档和子文档
POST my_blogs/_search
{}

// 根据父文档ID查看, 返回父文档信息
GET my_blogs/_doc/blog2

// Parent Id 返回对应的所有评论信息
POST my_blogs/_search
{
  "query": {
    "parent_id": {
      "type": "comment",
      "id": "blog2"
    }
  }
}

// Has Child 查询,返回符合条件的父文档列表
POST my_blogs/_search
{
  "query": {
    "has_child": {
      "type": "comment",
      "query" : {
        "match": {
            "username" : "Jack"
        }
      }
    }
  }
}

// Has Parent 查询，返回符合条件的子文档列表
POST my_blogs/_search
{
  "query": {
    "has_parent": {
      "parent_type": "blog",
      "query" : {
        "match": {
            "title" : "Learning Hadoop"
        }
      }
    }
  }
}

// 通过ID ，访问子文档
GET my_blogs/_doc/comment2

// 通过ID和routing ，访问子文档
GET my_blogs/_doc/comment2?routing=blog2

// 更新子文档
PUT my_blogs/_doc/comment2?routing=blog2
{
    "comment": "Hello Hadoop??",
    "blog_comments_relation": {
      "name": "comment",
      "parent": "blog2"
    }
}
```

### 对比

-    | Nested Object      | Parent - Child
:--- | :----------------- | :------------------
优点   | ⽂档存储在⼀起，读取性能⾼      | ⽗子⽂档可以独⽴更新
缺点   | 更新嵌套的⼦文档时，需要更新整个⽂档 | 需要额外的内存维护关系。读取性能相对差
适⽤场景 | ⼦文档偶尔更新，以查询为主      | 子⽂档更新频繁

## Update By Query & Reindex API

一般发生以下几种情况时, 我们需要重建索引:

- 索引的 `Mappings` 发⽣变更: 字段类型更改，分词器及字典更新
- 索引的 `Settings` 发⽣变更: 索引的主分片数发生改变
- 集群内，集群间需要做数据迁移

Elasticsearch 的内置提供了两种API用于重建索引:

- Update By Query: 在现有索引上重建
- Reindex: 在其他索引上重建索引

### Update By Query

**为索引增加子字段**:

```java
// 生成blogs索引
PUT blogs/_doc/1
{
  "content":"Hadoop is cool",
  "keyword":"hadoop"
}

// 给 content 添加额外字段 english, 并指定 english 分词器
PUT blogs/_mapping
{
  "properties": {
    "content": {
      "type": "text",
      "fields": {
        "english": {
          "type": "text",
          "analyzer": "english"
        }
      }
    }
  }
}

// 查询文档之前插入文档, 无法正常获取
POST blogs/_search
{
  "query": {
    "match": {
      "content.english": "Hadoop"
    }
  }
}

// 这时候就需要重建索引
POST blogs/_update_by_query
{}

// 查询文档之前插入文档, 可以正常获取
POST blogs/_search
{
  "query": {
    "match": {
      "content.english": "Hadoop"
    }
  }
}
```

但是有一种情况, 并不能使用 `_update_by_query`, 那就是对原有的`mapping`进行修改. `ES`是不支持这个操作的, 这时候只能使用新建一个索引, 然后使用`reindex`重建索引.

### Reindex

使用方式非常简单, 首先需要设置好新索引的`mapping`信息:

```java
// 新建 bugs_fix 用于接收 blogs 索引
PUT blogs_fix/
{
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "fields": {
          "english": {
            "type": "text",
            "analyzer": "english"
          }
        }
      },
      "keyword": {
        "type": "keyword"
      }
    }
  }
}

// 重建索引
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix"
  }
}
```

使用`reindex`需要注意两点:

- 在源索引中所有文档的`_source`必须开启, 不能关闭.
- `Reindex`并不会拷贝源索引的`mapping`设置, 需要我们给新建索引, 提前设置好.

当`dest`索引中存储了一些额外的数据, 这时候为了防止冲突, 可以在`dest`中设置`op_type: create`. 这样只有在`dest`中不存在的文档, 才会拷贝过来. 如:

```java
POST  _reindex
{
  "source": {
    "index": "blogs"
  },
  "dest": {
    "index": "blogs_fix",
    "op_type": "create"
  }
}
```

`reindex`支持跨集群操作:

- 在配置文件`elasticsearch.yml`中设置白名单: `reindex.remote.whitelist: OTHER_HOST`.
- 在`source`中指定`remote.host`.

```java
POST  _reindex
{
  "source": {
    "remote": {
      "host": "http://otherhost:9200"
    },
    "index": "blogs",
    "size": 100,
    "query": {
      "match": {
        "test": "data"
      }
    }
  },
  "dest": {
    "index": "blogs_fix"
  }
}
```

同理, `Elasticsearch`还支持异步`Reindex`:`POST _reindex?wait_for_completion=false`, 这时候会立马返回一个`TaskId`, 后续可以通过这个`TaskId`查看具体的处理流程. 查看所有`TASK`接口: `GET _tasks?detailed=true&actions=*reindex`.

## Ingest Pipeline 与 Painless Script

### Pipeline

如我们需要对一段输入进行加工:

```java
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}
```

我们需要处理`tags`, 将其存储成数组类型, 便于后面进行聚合分析.

这时候就需要引入一个新的概念: `Ingest Node`.

- Elasticsearch 5.0 后，引⼊的⼀种新的节点类型。 默认配置下，每个节点都是 Ingest Node

  - 具有预处理数据的能力，可拦截 Index 或 Bulk API 的请求
  - 对数据进行转换，并重新返回给 Index 或 Bulk API

- ⽆需`Logstash`，就可以进⾏数据的预处理，例如

  - 为某个字段设置默认值;重命名某个字段的字段名;对字段值进⾏ Split 操作
  - ⽀持设置 Painless 脚本，对数据进⾏更加复杂的加⼯

而`Ingest Node`的处理方式则是通过`pipeline`: 将通过的文档, 依次按顺序进行处理. `Pipeline`内部就是一组`Processor`之和. `ES`内置了[非常多的处理器](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/ingest-processors.html):

- Split Processor (例: 将给定字段值切分成⼀个数组)
- Remove / Rename Processor (例:移除⼀个重命名字段)
- Append (例:为商品增加一个新的标签)
- Convert (例:将商品价格，从字符串转换成 float 类型)
- Date / JSON(例:⽇期格式转换，字符串转 JSON 对象)
- Date Index Name Processor (例:将通过该处理器的⽂档，分配到指定时间格式的索引中)
- Fail Processor (⼀旦出现异常，该 Pipeline 指定的错误信息能返回给⽤户)
- Foreach Process(数组字段，数组的每个元素都会使用到⼀个相同的处理器)
- Grok Processor(⽇志的⽇期格式切割)
- Gsub / Join / Split(字符串替换 / 数组转字符串/ 字符串转数组)
- Lowercase / Upcase(⼤小写转换)
- 当然也可以通过插件实现自己的Processor

以上面的数据为例:

```java
// 使用 _simulate 进行pipeline的模拟处理, 返回处理之后的结果
POST _ingest/pipeline/_simulate
{
  // pipeline 定义
  "pipeline": {
    // 描述
    "description": "to split blog tags",
    // 内部包含的处理器列表(依次处理), 这里只包含一个split处理器
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      }
    ]
  },
  // 要处理的数据
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "title": "Introducing big data......",
        "tags": "hadoop,elasticsearch,spark",
        "content": "You konw, for big data"
      }
    }
  ]
}

// 处理之后:

...
"title" : "Introducing big data......",
"content" : "You konw, for big data",
"tags" : [
  "hadoop",
  "elasticsearch",
  "spark"
]
...
```

当然我们还可以对`pipeline`进行存储维护:

**新建一个pipeline**:

```java
PUT _ingest/pipeline/blog_pipeline
{
  "description": "a blog pipeline",
  "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },

      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
}
```

**查看Pipeline**: `GET _ingest/pipeline/blog_pipeline`.

**测试PipeLine**:

```java
POST _ingest/pipeline/blog_pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "title": "Introducing cloud computering",
        "tags": "openstack,k8s",
        "content": "You konw, for cloud"
      }
    }
  ]
}
```

**使用pipeline更新数据**:

```java
PUT tech_blogs/_doc/2?pipeline=blog_pipeline
{
  "title": "Introducing cloud computering",
  "tags": "openstack,k8s",
  "content": "You konw, for cloud"
}
```

**使用pipeline重建索引**:

```java
// 如果存在处理之后的数据, 则会出现异常
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{}

// 这时候可以添加过滤, 处理之后的数据, 则不进行处理
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{
  "query": {
      "bool": {
          "must_not": {
              "exists": {
                  "field": "views"
              }
          }
      }
  }
}
```

**Pipeline VS Logstash**:

-       | Logstash              | Ingest Node
:------ | :-------------------- | :---------------------------------------
数据输⼊与输出 | 支持从不同的数据源读取，并写入不同的数据源 | 支持从 ES REST API 获取数据, 并且写入 Elasticsearch
数据缓冲    | 实现了简单的数据队列，⽀持重写       | 不⽀持缓冲
数据处理理   | ⽀持⼤量的插件，也支持定制开发       | 内置的插件，可以开发 Plugin 进行扩展(Plugin 更新需要重启)
配置和使⽤   | 增加了一定的架构复杂度           | 无需额外部署

### Painless Script

- 自 Elasticsearch 5.x 后引入，专门为 Elasticsearch 设计，扩展了 Java 的语法。

  - 6.0 开始，ES 只⽀持 Painless。 Groovy， JavaScript 和 Python 都不再⽀持
  - Painless ⽀持所有 Java 的数据类型及 Java API 子集
  - Painless Script 具备以下特性

    - 高性能/安全
    - 支持显示类型或者动态定义类型

Painless 的⽤途

- 可以对⽂档字段进⾏加⼯处理

  - 更新或删除字段，处理数据聚合操作
  - Script Field: 对返回的字段提前进行计算
  - Function Score: 对⽂档的算分进⾏处理

- 在 Ingest Pipeline 中执⾏脚本

- 在 Reindex API，Update By Query 时，对数据进行处理

**通过 Painless 脚本访问字段**:

上下⽂文                 | 语法
:------------------- | :---------------------
Ingestion            | ctx.field_name
Update               | ctx._source.field_name
Search & Aggregation | doc["field_name"]

**Demo**:

```java
// 在 Ingestion (pipeline)中使用Painless Script
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },
      {
        // 判断是否存在 content, 如果存在则将content_length设置为对应的长度或者0
        "script": {
          "source": """
          if(ctx.containsKey("content")){
            ctx.content_length = ctx.content.length();
          }else{
            ctx.content_length=0;
          }
"""
        }
      },
      {
        "set": {
          "field": "views",
          "value": 0
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "index",
      "_id": "id",
      "_source": {
        "title": "Introducing big data......",
        "tags": "hadoop,elasticsearch,spark",
        "content": "You konw, for big data"
      }
    }
  ]
}

// 在Update中使用Script
DELETE tech_blogs
// 插入一条博客数据
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data",
  "views":0
}

GET tech_blogs/_doc/1

// 更新博客数据
POST tech_blogs/_update/1
{
  // 设置博客的 views 为原有的views + 参数new_views
  "script": {
    "source": "ctx._source.views += params.new_views",
    "params": {
      "new_views":100
    }
  }
}

// 保存脚本在 Cluster State, 设置名字为 update_views
POST _scripts/update_views
{
  "script":{
    "lang": "painless",
    "source": "ctx._source.views += params.new_views"
  }
}

// 使用保存的脚本进行更新
POST tech_blogs/_update/1
{
  "script": {
    "id": "update_views",
    "params": {
      "new_views":1000
    }
  }
}

// 在Search中使用脚本, 对结果进行处理
GET tech_blogs/_search
{
  "script_fields": {
    // 生成一个新的字段, 用于存储随机计数值
    "rnd_views": {
      "script": {
        "lang": "painless",
        "source": """
          java.util.Random rnd = new Random();
          doc['views'].value+rnd.nextInt(1000);
        """
      }
    }
  },
  "query": {
    "match_all": {}
  }
}

// 返回结果
...
{
  "_index" : "tech_blogs",
  "_type" : "_doc",
  "_id" : "1",
  "_score" : 1.0,
  "fields" : {
    "rnd_views" : [
      374
    ]
  }
}
...
```

需要注意的一点是,`ES`会对内部的脚本进行缓存(包括内联和存储的), ES会对其进行编译缓存, 最大缓存数为100\. 可以自定义设置其相关的参数:

- script.cache.max_size: 最大缓存数, 默认100.
- script.cache.expire: 缓存超时.
- script.max_compilations_rate: 最大编译次数. 默认75/5m(即5分钟最多75次编译).

## Elasticsearch 数据建模实例

- 数据建模(Data modeling)， 是创建数据模型的过程

  - 数据模型是对真实世界进行抽象描述的一种⼯具和方法，实现对现实世界的映射

    - 博客/作者/⽤户评论

  - 三个过程:概念模型 => 逻辑模型 => 数据模型(第三范式)

    - 数据模型:结合具体的数据库，在满⾜业务读写性能等需求的前提下，确定最终的定义

总而言之: 数据建模 = 功能需求 + 性能需求

- 功能需求(逻辑模型):

  - 实体属性
  - 实体之间的关系
  - 搜索相关的配置

- 物理模型(性能需求)

  - 索引模版

    - 分⽚数量

  - 索引 Mapping

    - 字段配置
    - 关系处理

对字段的建模过程: 字段类型 -> 是否要搜索及分词 -> 是否要聚合及排序 -> 是否要额外的存储

字段类型: `Text v.s Keyword`

- Text: ⽤于全⽂本字段，文本会被 Analyzer 分词, 默认不⽀持聚合分析及排序。需要设置 `fielddata` 为 true.
- Keyword: ⽤于`id`，枚举及不需要分词的⽂本。例如电话号码，email地址，⼿机号码，邮政编码，性别等. 适⽤于 Filter(精确匹配)，Sorting 和 Aggregations.
- 设置多字段类型: 默认会为文本类型设置成`text`，并且设置一个 `keyword` 的子字段. 在处理理⼈类语⾔时，通过增加`英⽂`，`拼音`和`标准`分词器，提⾼搜索结构.

字段类型: 结构化数据.

- 数值类型, 尽量选择贴近的类型。例如可以用 byte，就不要用 long.
- 枚举类型, 设置为 keyword。即便是数字，也应该设置成 keyword，获取更加好的性能.
- 其他: ⽇期/布尔/地理信息

检索:

- 如不需要检索，排序和聚合分析: Enable 设置成 false
- 如不需要检索, Index 设置成 false
- 对需要检索的字段，可以通过如下配置，设定存储粒度: `Index_options / Norms` : 不需要归一化数据时，可以关闭.

聚合和排序:

- 如不需要排序或者聚合分析功能: Doc_values / fielddata 设置成 false.
- 更新频繁，聚合查询频繁的 keyword 类型的字段: 推荐将 eager_global_ordinals 设置为 true

额外的存储:

是否需要专门存储当前字段数据: Store 设置成 true，可以存储该字段的原始内容, ⼀般结合 _source 的 enabled 为 false 时候使⽤.

Disable _source: 节约磁盘;适⽤于指标型数据:

- ⼀般建议先考虑增加压缩⽐.
- ⽆法看到 _source字段，⽆法做 ReIndex，⽆法做 Update
- Kibana 中⽆法做 discovery

### 实例

这里以图书管理为例:

```java
// 插入一条图书记录, 包含以下信息: 标题, 简介, 作者, 发行时间和封面.
PUT books/_doc/1
{
  "title":"Mastering ElasticSearch 5.0",
  "description":"Master the searching, indexing, and aggregation features in ElasticSearch Improve users’ search experience with Elasticsearch’s functionalities and develop your own Elasticsearch plugins",
  "author":"Bharvi Dixit",
  "public_date":"2017",
  "cover_url":"https://images-na.ssl-images-amazon.com/images/I/51OeaMFxcML.jpg"
}
```

优化字段设计:

- 书名:⽀持全文和精确匹配
- 简介:支持全⽂
- 作者: 精确值
- 发⾏日期: ⽇期类型
- 图书封面: 精确值

```java
PUT books
{
  "mappings": {
    "properties": {
      "author": {
        "type": "keyword"
      },
      "cover_url": {
        "type": "keyword",
        "index": false
      },
      "description": {
        "type": "text"
      },
      "public_date": {
        "type": "date"
      },
      "title": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 100
          }
        }
      }
    }
  }
}
```

**需求变更**: 新需求 - 增加图书内容的字段。并要求能被搜索同时⽀持⾼亮显示:

- 新需求会导致 _source 的内容过⼤

  - Source Filtering 只是传输给客户端时进行过滤，Fetch 数据时，ES 节点还是会传输 _source 中的数据.

解决方法:

- 关闭 _source
- 然后将每个字段的 "store" 设置成 true

```java
PUT books
{
  "mappings": {
    "_source": {
      "enabled": false
    },
    "properties": {
      "author": {
        "type": "keyword",
        "store": true
      },
      "cover_url": {
        "type": "keyword",
        "index": false,
        "store": true
      },
      "description": {
        "type": "text",
        "store": true
      },
      "content": {
        "type": "text",
        "store": true
      },
      "public_date": {
        "type": "date",
        "store": true
      },
      "title": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 100
          }
        },
        "store": true
      }
    }
  }
}
```

通过这种处理方案, 只需要在查询中指定`store_fields`即可正确获取结果, 并且支持高亮显示.

```java
// 插入一条记录
PUT books/_doc/1
{
  "title":"Mastering ElasticSearch 5.0",
  "description":"Master the searching, indexing, and aggregation features in ElasticSearch Improve users’ search experience with Elasticsearch’s functionalities and develop your own Elasticsearch plugins",
  "content":"The content of the book......Indexing data, aggregation, searching.    something else. something in the way............",
  "author":"Bharvi Dixit",
  "public_date":"2017",
  "cover_url":"https://images-na.ssl-images-amazon.com/images/I/51OeaMFxcML.jpg"
}

// 查询结果中，Source不包含数据
POST books/_search
{}

// 搜索，通过 store 指定显示字段，同时高亮显示 content的内容
POST books/_search
{
  "stored_fields": ["title","author","public_date"],
  "query": {
    "match": {
      "content": "searching"
    }
  },

  "highlight": {
    "fields": {
      "content":{}
    }
  }
}
```

更多`Mapping`相关属性可以查看[官网](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html):

- Enabled – 设置成 false，仅做存储，不支持搜索和聚合分析 (数据保存在 _source 中)
- Index – 是否倒排索引。设置成 false，⽆法被搜索，但还是⽀持 aggregation，并出现在 _source 中
- Norms – 如果字段⽤来过滤和聚合分析，可以关闭，节约存储
- Doc_values – 是否启用 doc_values，⽤于排序和聚合分析
- Field_data – 如果要对 text 类型启⽤排序和聚合分析， fielddata 需要设置成true
- Store – 默认不存储，数据默认存储在 _source。
- Coerce – 默认开启，是否开启数据类型的⾃动转换(例如，字符串转数字)
- Multifields 多字段特性
- Dynamic – true / false / strict 控制 Mapping 的⾃动更新

## Elasticsearch 数据建模最佳实践

### 如何处理理关联关系

- 优先考虑 Denormalization: 即对象存储方式.
- 当数据包含多数值对象(多个演员)，同时有查询需求: 这时候需要使用到 nested 对象存储.
- 关联⽂档更新⾮常频繁时: 考虑使用 Parent/Child 存储方式.

注意: Kibana ⽬前暂不⽀持 nested 类型和 parent/child 类型 ，在未来有可能会⽀持.

### 避免过多字段

- 一个⽂档中，最好避免⼤量的字段:

  - 过多的字段数不容易维护
  - Mapping 信息保存在 Cluster State 中，数据量过大，对集群性能会有影响 (Cluster State 信息需要和所有的节点同步)
  - 删除或者修改数据需要 reindex

- 默认最⼤字段数是 1000，可以设置 index.mapping.total_fields.limit 限定最大字段数。

- 什么原因会导致⽂档中有成百上千的字段?

  - 主要原因是打开了 Dynamic 属性(生产环境尽量关闭该属性)

    - true - 未知字段会被⾃动加⼊ mapping
    - false - 新字段不会被索引。但是会保存在 _source
    - strict - 新增字段不会被索引，⽂档写⼊失败

如 `Cookie Service 的数据` 例子:

```java
PUT cookie_service/_doc/1
{
 "url":"www.google.com",
 "cookies":{
   "username":"tom",
   "age":32
 }
}

PUT cookie_service/_doc/2
{
 "url":"www.amazon.com",
 "cookies":{
   "login":"2019-01-01",
   "email":"xyz@abc.com"
 }
}
```

来自 Cookie Service 的数据

- Cookie 的键值对很多
- 当 Dynamic 设置为 True
- 同时采⽤扁平化的设计，必然导致字段数量的膨胀

解决⽅案: `Nested Object & Key Value`, 使用`Nested`对象来存储所有的`cookies`信息, 这样只会有一个对象.

```java
PUT cookie_service
{
  "mappings": {
    "properties": {
      "cookies": {
        "type": "nested",
        "properties": {
          "name": {
            "type": "keyword"
          },
          "dateValue": {
            "type": "date"
          },
          "keywordValue": {
            "type": "keyword"
          },
          "IntValue": {
            "type": "integer"
          }
        }
      },
      "url": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      }
    }
  }
}

// 写入和查询
PUT cookie_service/_doc/2
{
 "url":"www.amazon.com",
 "cookies":[
    {
      "name":"login",
      "dateValue":"2019-01-01"
    },
    {
       "name":"email",
      "IntValue":32

    }

   ]
 }


// Nested 查询，通过bool查询进行过滤
POST cookie_service/_search
{
  "query": {
    "nested": {
      "path": "cookies",
      "query": {
        "bool": {
          "filter": [
            {
            "term": {
              "cookies.name": "age"
            }},
            {
              "range":{
                "cookies.intValue":{
                  "gte":30
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

可以减少字段数量，解决`Cluster State` 中保存过多 Meta 信息的问题，但是

- 导致查询语句复杂度增加
- Nested 对象，不利于在 `Kibana` 中实现可视化分析

### 避免正则查询

- 正则，通配符查询，前缀查询属于 Term 查询，但是性能不够好
- 特别是将通配符放在开头，会导致性能的灾难

如: ⽂档中某个字段包含了 Elasticsearch 的版本信息，例如 `version: 7.1.0`. 搜索所有是 `bug fix` 的版本? 每个主要版本号所关联的文档?

解决方案: 将字符串转换为对象.

```java
PUT softwares/
{
  "mappings": {
    "_meta": {
      "software_version_mapping": "1.1"
    },
    "properties": {
      "version": {
        "properties": {
          "display_name": {
            "type": "keyword"
          },
          "hot_fix": {
            "type": "byte"
          },
          "marjor": {
            "type": "byte"
          },
          "minor": {
            "type": "byte"
          }
        }
      }
    }
  }
}

// 写入时指定各类版本信息
PUT softwares/_doc/1
{
  "version":{
  "display_name":"7.1.0",
  "marjor":7,
  "minor":1,
  "hot_fix":0  
  }
}

// 通过bool查询动态过滤版本信息
POST softwares/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "match":{
            "version.marjor":7
          }
        },
        {
          "match":{
            "version.minor":2
          }
        }

      ]
    }
  }
}
```

### 避免空值引起的聚合不准

`null`值在聚合分析的时候是不会进行处理的. 一般推荐设置默认值.

```java
PUT ratings
{
  "mappings": {
      "properties": {
        "rating": {
          "type": "float",
          "null_value": 1.0
        }
      }
    }
}
```

### 为索引的 Mapping 加入 Meta 信息

- Mappings 设置⾮常重要，需要从两个维度进⾏考虑

  - 功能: 搜索，聚合，排序
  - 性能: 存储的开销; 内存的开销; 搜索的性能

- Mappings 设置是⼀个迭代的过程

  - 加入新的字段很容易(必要时需要 update_by_query)
  - 更新删除字段不允许(需要 Reindex 重建数据)
  - 最好能对 Mappings 加⼊ Meta 信息，更好的进⾏版本管理
  - 可以考虑将 Mapping ⽂件上传 git 进行管理
