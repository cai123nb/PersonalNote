# Elasticsearch之聚合分析

本章主要讲解 `Elasticsearch` 聚合相关知识点, 以实例为主, 细化每一个聚合知识点.

## Bucket & Metric 聚合分析及嵌套聚合

聚合分析是属于`Search`的一部分, 一般建议将`size`设置为0\. 如:

```json
// 调用 _search API
POST employees/_search
{
  // 将返回的size设置为0, 不需要查看返回的值
  "size": 0,
  // 配置聚合的信息
  "aggs": {
    // 该聚合名称为 min_salary (自由命名, 建议规范化)
    "min_salary": {
      // 使用 min 聚合求取最小值
      "min": {
        // 对应的字段为 salary
        "field":"salary"
      }
    },
    // 该聚合名称为 avg_salary (自由命名, 建议规范化)
    "avg_salary": {
      // 使用 avg 聚合求取平均值
      "avg": {
        // 对应的字段为 salary
        "field": "salary"
      }
    }
  }
}
```

返回的结果分析:

```json
{
  // 捕获21条记录
  "took" : 21,
  "timed_out" : false,
  // 分片信息
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  // 命中数量为0
  "hits" : {
    "total" : {
      "value" : 20,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  // 聚合信息
  "aggregations" : {
    // 平均工资为 24700
    "avg_salary" : {
      "value" : 24700.0
    },
    // 最小工资为 9000
    "min_salary" : {
      "value" : 9000.0
    }
  }
}
```

### Metric 聚合

主要那个对数据进行统计分析, 按照返回结果分为两类:

- 单值分析: 只输出⼀一个分析结果

  - min, max, avg, sum
  - Cardinality (类似SQL中: distinct Count)

- 多值分析: 输出多个分析结果

  - stats, extended stats
  - percentile, percentile rank
  - top hits (排在前⾯面的示例例)

**Demo**:

```json
// Metric 聚合，找到最高的工资
POST employees/_search
{
  "size": 0,
  "aggs": {
    "max_salary": {
      "max": {
        "field":"salary"
      }
    }
  }
}

// 多个 Metric 聚合，找到最低,最高和平均工资
POST employees/_search
{
  "size": 0,
  "aggs": {
    "max_salary": {
      "max": {
        "field": "salary"
      }
    },
    "min_salary": {
      "min": {
        "field": "salary"
      }
    },
    "avg_salary": {
      "avg": {
        "field": "salary"
      }
    }
  }
}

// Metric聚合，输出多值(最大,最小,平均,数量,总和)
POST employees/_search
{
  "size": 0,
  "aggs": {
    "stats_salary": {
      "stats": {
        "field":"salary"
      }
    }
  }
}
```

### Bucket 聚合

按照一定的规则，将文档分配到不同的桶中，从⽽达到分类的目的(类似于SQL group by)。ES 提供的一些常⻅的`Bucket Aggregation`:

- Terms
- 数字类型

  - Range / Data Range
  - Histogram / Date Histogram

Bucket聚合同样支持嵌套, 可以在桶里面再进行分组. 在使用`Term`聚合的时候需要注意一点就是, 对于`text`类型的字段, 需要打开 `fielddata` 才可以进行聚合. 另外在对`text`字段进行聚合的时候会先进行分词, 进行分组. 如果想要完全按照字段进行分组, 推荐使用`keyword`.

**Demo**:

```json
// 对 job.keword 进行 Bucket 聚合
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"
      }
    }
  }
}

// 对 Text 字段进行 terms 聚合查询，返回失败
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job"
      }
    }
  }
}

// 对 Text 字段打开 fielddata，支持terms aggregation
// 再次对 job 进行 bucket聚合, 成功
PUT employees/_mapping
{
  "properties" : {
    "job":{
       "type":     "text",
       "fielddata": true
    }
  }
}

// 指定size，不同工种中，年纪最大的3个员工的具体信息
POST employees/_search
{
  "size": 0,
  "aggs": {
    // 首先对 job.keyword 进行分桶
    "jobs": {
      "terms": {
        "field":"job.keyword"
      },
      "aggs":{
        // 其次使用 top_hits 读取前三位员工的具体信息
        "old_employee":{
          "top_hits":{
            "size":3,
            "sort":[
              {
                "age":{
                  "order":"desc"
                }
              }
            ]
          }
        }
      }
    }
  }
}

// 对 salary 进行 range 聚合, 分为 >1000, 1000-20000, <2000三类
POST employees/_search
{
  "size": 0,
  "aggs": {
    "salary_range": {
      "range": {
        "field":"salary",
        "ranges":[
          {
            "to":10000
          },
          {
            "from":10000,
            "to":20000
          },
          {
            "key":">20000",
            "from":20000
          }
        ]
      }
    }
  }
}

// Salary Histogram,工资0到10万，以 5000一个区间进行分桶
POST employees/_search
{
  "size": 0,
  "aggs": {
    "salary_histrogram": {
      "histogram": {
        "field":"salary",
        "interval":5000,
        "extended_bounds":{
          "min":0,
          "max":100000

        }
      }
    }
  }
}

// 嵌套聚合1，按照工作类型分桶，并统计工资信息
POST employees/_search
{
  "size": 0,
  "aggs": {
    "Job_salary_stats": {
      "terms": {
        "field": "job.keyword"
      },
      "aggs": {
        "salary": {
          "stats": {
            "field": "salary"
          }
        }
      }
    }
  }
}

// 多次嵌套。根据工作类型分桶，然后按照性别分桶，计算工资的统计信息
POST employees/_search
{
  "size": 0,
  "aggs": {
    "Job_gender_stats": {
      "terms": {
        "field": "job.keyword"
      },
      "aggs": {
        "gender_stats": {
          "terms": {
            "field": "gender"
          },
          "aggs": {
            "salary_stats": {
              "stats": {
                "field": "salary"
              }
            }
          }
        }
      }
    }
  }
}
```

对于`Term`查询, 为了提升`Term`查询的效率, 可以[设置`eager_global_ordinals`参数](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/tune-for-search-speed.html#_warm_up_global_ordinals):

Global ordinals are a data-structure that is used in order to run terms aggregations on keyword fields. They are loaded lazily in memory because Elasticsearch does not know which fields will be used in terms aggregations and which fields won't. You can tell Elasticsearch to load global ordinals eagerly at refresh-time by configuring mappings as described below:

```json
PUT index
{
  "mappings": {
    "properties": {
      "foo": {
        "type": "keyword",
        "eager_global_ordinals": true
      }
    }
  }
}
```

## Pipeline聚合分析

- 管道的概念: ⽀持对聚合分析的结果，再次进行聚合分析

- Pipeline 的分析结果会输出到原结果中，根据位置的不同，分为两类

  - Sibling - 结果和现有分析结果同级

    - Max，min，Avg & Sum Bucket
    - Stats，Extended Status Bucket
    - Percentiles Bucket

  - Parent - 结果内嵌到现有的聚合分析结果之中

    - Derivative (求导)
    - Cumultive Sum (累计求和)
    - Moving Function (滑动窗)

**Demo**:

```json
// Sibling Pipe: 平均工资最低的工作类型
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      // 在 jobs 中首先按照 job.keyword 进行分桶(最大选取10个桶)
      // 按照 doc_count 降序排序
      "terms": {
        "field": "job.keyword",
        "size": 10
      },
      // 在每个桶中进行 avg 聚合求取平均工资
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        }
      }
    },
    // sibling pipeline, 不在 jobs 内部, 而是同级
    // 使用 min_bucket 求取最小值桶, 路径为 jobs>avg_salary
    "min_salary_by_job":{
      "min_bucket": {
        "buckets_path": "jobs>avg_salary"
      }
    }
  }
}


// 平均工资最高的工作类型
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field": "job.keyword",
        "size": 10
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        }
      }
    },
    "max_salary_by_job":{
      "max_bucket": {
        "buckets_path": "jobs>avg_salary"
      }
    }
  }
}

// 平均工资的百分位数
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field": "job.keyword",
        "size": 10
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        }
      }
    },
    "percentiles_salary_by_job":{
      "percentiles_bucket": {
        "buckets_path": "jobs>avg_salary"
      }
    }
  }
}

// Parent pipeline 按照年龄对平均工资求导
POST employees/_search
{
  "size": 0,
  "aggs": {
    // 首先按照 age 进行直方图处理, 间隔为1
    "age": {
      "histogram": {
        "field": "age",
        "min_doc_count": 1,
        "interval": 1
      },
      // 在年龄桶里面, 对 salary 进行 avg 聚合, 计算平均工资
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        },
        // 和 avg_salary 同级, 在 age 的下一级
        // 计算 平均工资的求导
        "derivative_avg_salary":{
          "derivative": {
            "buckets_path": "avg_salary"
          }
        }
      }
    }
  }
}

// Cumulative_sum
POST employees/_search
{
  "size": 0,
  "aggs": {
    "age": {
      "histogram": {
        "field": "age",
        "min_doc_count": 1,
        "interval": 1
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        },
        "cumulative_salary":{
          "cumulative_sum": {
            "buckets_path": "avg_salary"
          }
        }
      }
    }
  }
}

// Moving Function
POST employees/_search
{
  "size": 0,
  "aggs": {
    "age": {
      "histogram": {
        "field": "age",
        "min_doc_count": 1,
        "interval": 1
      },
      "aggs": {
        "avg_salary": {
          "avg": {
            "field": "salary"
          }
        },
        "moving_avg_salary":{
          "moving_fn": {
            "buckets_path": "avg_salary",
            "window":10,
            "script": "MovingFunctions.min(values)"
          }
        }
      }
    }
  }
}
```

## 聚合的作⽤范围及排序

### 聚合的作用范围

ES 聚合分析的默认作用范围是 query 的查询结果集, 所以可以在查询的结果集中进行限制. 另外 ES 还支持其他方式改变聚合的作用范围:

- filter: 在当前聚合内对结果集进行过滤(注意范围只限于当前聚合类中)
- post_filter: 是对聚合分析后的⽂档进⾏再次过滤.
- global: 无视 query, 直接对全局文档进行统计.

**Demo**:

```json
// Query, 使用 query 进行范围限定: 年龄大于20
POST employees/_search
{
  "size": 0,
  "query": {
    "range": {
      "age": {
        "gte": 20
      }
    }
  },
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"
      }
    }
  }
}


// Filter, 使用 filter 对结果集进行过滤
POST employees/_search
{
  "size": 0,
  "aggs": {
    "older_person": {
      // 在 older_person 聚合处理中限制年龄大于35
      "filter":{
        "range":{
          "age":{
            "from":35
          }
        }
      },
      "aggs":{
         "jobs":{
           "terms": {
        "field":"job.keyword"
      }
      }
    }},
    "all_jobs": {
      // all_jobs 中上面过滤条件无效
      "terms": {
        "field":"job.keyword"

      }
    }
  }
}

// Post field. 一条语句，找出所有的job类型。还能找到聚合后符合条件的结果
// 首先按照 job.keyword 进行分组, 然后使用 post_filter 筛选所有工作为 Dev Manager 的开发者
POST employees/_search
{
  "aggs": {
    "jobs": {
      "terms": {
        "field": "job.keyword"
      }
    }
  },
  "post_filter": {
    "match": {
      "job.keyword": "Dev Manager"
    }
  }
}


// 设置 global ,无视query限定
POST employees/_search
{
  "size": 0,
  "query": {
    "range": {
      "age": {
        "gte": 40
      }
    }
  },
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword"

      }
    },

    "all":{
      // 这里设置 global, 无视查询限定条件
      "global":{},
      "aggs":{
        "salary_avg":{
          "avg":{
            "field":"salary"
          }
        }
      }
    }
  }
}
```

### 排序

指定 order， 按照 count 和 key 进⾏行行排序

- 默认情况，按照 count 降序排序
- 指定 size，就能返回相应数量的桶

**Demo**:

```json
// 排序 order
// count and key
POST employees/_search
{
  "size": 0,
  // 首先 query 限制条件为 年龄大于20岁
  "query": {
    "range": {
      "age": {
        "gte": 20
      }
    }
  },
  "aggs": {
    // 设置按照 job.keyword 进行分桶
    // 设置 order 值, 按照 count 升序, key 降序
    "jobs": {
      "terms": {
        "field":"job.keyword",
        "order":[
          {"_count":"asc"},
          {"_key":"desc"}
          ]

      }
    }
  }
}

// 设置按照 avg_salary 子聚合进行排序
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword",
        "order":[  {
            "avg_salary":"desc"
          }]
      },
    "aggs": {
      "avg_salary": {
        "avg": {
          "field":"salary"
        }
      }
    }
    }
  }
}

// 设置按照 stats_salary 子聚合中的最小值进行排序
POST employees/_search
{
  "size": 0,
  "aggs": {
    "jobs": {
      "terms": {
        "field":"job.keyword",
        "order":[  {
            "stats_salary.min":"desc"
          }]
      },
    "aggs": {
      "stats_salary": {
        "stats": {
          "field":"salary"
        }
      }
    }
    }
  }
}
```

## 聚合分析原理和精准度问题

**分布式系统的近似统计算法**:

![distrute-computing](https://image.cjyong.com/distributed-%20computing.png)

在数量较小的时候, ES 可以实现`精准度`和`实时性`的功能. 当数据量增加的时候, `ES`会采用一些措施来保证`实时性`的近似计算.

探究 `Term` 聚合的处理方式, 这里以 `min` 聚合为例:

![min-term-computing](https://image.cjyong.com/min-term-example.png)

首先`Min`聚合会在每一个节点中分别进行计算, 最后在`coordinating node`中进行汇总.

在这里的`min`聚合是可以保证数据的准确性的. 但是在其他聚合运算中, 却是无法保证数据的准确性, 这里以`top 3`来演示一个具体的例子:

![term-wrong-example](https://image.cjyong.com/term-wrong-example.png)

这里准确的数据返回应该是: `A(12), B(6), D(6)`, 结果却返回了: `A(12), B(6), C(3)`. 这里存在数据丢失导致的不准确的问题. `ES` 使用以下两个参数来衡量这种情况:

- `doc_count_error_upper_bound` : 被遗漏的 term 分桶中包含的⽂档有可能的最⼤值
- `sum_other_doc_count`: 除了返回结果 bucket 的 terms 以外，其他 terms 的⽂档总数(总数 - 返回的总数)

要显示这两个参数, 只需要在聚合查询中添加`show_term_doc_count_error`参数即可, 如:

```json
GET my_flights/_search
{
  "size": 0,
  "aggs": {
    "weather": {
      "terms": {
        "field":"OriginWeather",
        "size":1,
        "shard_size":3,
        // 这里打开显示准确度参数
        "show_term_doc_count_error":true
      }
    }
  }
}
```

为了解决准确的问题, 一般存在两种解决方案:

- 数据量不大的时候, 通过设置 `primary shard` 为1, 可以实现完全的准确.
- 数据量较大的时候, 在聚合分析查询中传递`shard_size`参数, 来提高准确度(每次需要额外从Shard中获取数据,来提高准确度). 如上面的查询设置为`1`. 增加计算量, 提高了响应时间, 从而提高准确度. 默认值为((size * 1.5 + 10))
