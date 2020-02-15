# ElasticSearch基础

## 诞生

为了解决`Lucene`搜索引擎缺陷而诞生.

[`Lucence`](https://lucene.apache.org/): 基于Java语言开发的开源的搜索引擎库. `Doug Cutting`(Hadoop之父)于1999年创建, 2005年成为Apache顶级开源项目, 具备高性能和易拓展的优点. 但其存在局限性:

- 只能基于Java语言开发
- 类库的接口学习曲线陡峭
- 原生不支持水平拓展

2004年`Shay Banon`基于`Lucene`开发了`Compass`, 于2010年重写, 并取名为`Elasticsearch`.

- 支持分布式, 可水平拓展
- 降低全文检索的学习曲线, 可以被任何编程语言调用

## 主要使用场景

主要的家庭成员:

- logstash/beat: 用于数据抓取和预处理.
- elasticsearch: 数据存储和分析.
- kibana: 数据可视化.
- x-pack: 收费的商业套件, 提供不同的订阅和功能(如机器学习, 图查询等).

常见的集合方式:

- `应用程序 -> 写入/更新 -> 数据库 -> es -> 查询(应用程序)`
- `beats(采集) -> redis/kafka/rabbitmq(缓冲) -> logstash(收集和预处理) -> es(处理) -> kibana/grafana(可视化)`

主要使用场景:

- 网站搜索/垂直搜索/代码搜索
- 日志管理和分析/安全指标监控/应用性能监控/WEB抓取舆情分

## 安装和使用

### elasticsearch

#### 下载elasticsearch

在[官网下载](https://www.elastic.co/)对应的安装包（国内网速感人，建议使用百度网盘）, 解压即可. 解压后文件夹简介:

- bin: 脚本文件, 包括启动elasticsearch,安装插件,运行统计数据等等.
- config: 集群的配置文件, 用户权限的配置, jvm配置等等.
- JDK: Java运行环境.(注意从7.0开始携带JDK环境).
- data: 数据文件.
- lib: Java类库.
- logs: 日志文件.
- modules: 包含所有的ES模块.
- plugins: 包含所有已安装的插件.

其中需要专门配置地方有: `config/jvm.options`, 默认配置堆大小为`1G`, 按照实际机器配置进行调整, 推荐设置:

- 将`Xms`和`Xms`设置成一样
- `Xms`不要超过机器内存的`50%`
- 不要超过30GB, 详情查看`https://www.elastic.co/blog/a-heap-of-trouble`.

#### 启动elasticsearch

启动单节点: `bin/elasticsearch`.

启动多节点:

- `bin/elasticsearch -E node.name=node1 -E cluster.name=clustername -E path.data=node1_data`.
- `bin/elasticsearch -E node.name=node2 -E cluster.name=clustername -E path.data=node2_data`

查看所有节点信息: `http://localhost:9200/_cat/nodes`.

#### 安装插件

- 查看已安装插件: `bin/elasticsearch-plugin list`.
- 安装插件: `bin/elasticsearch-plugin install PLUGIN_NAME`.(国内网速较慢,推荐离线安装)
- 查看插件是否正常启动: `http://localhost:9200/_cat/plugins`.

### kibana

在[官网下载](https://www.elastic.co/)对应的安装包（国内网速感人，建议使用百度网盘）, 解压即可. 解压后文件夹简介:

需要首先启动好`elasticsearch`才可以启用`kibana`. 启动好`elasticsearch`之后:

启动kibana: `bin/kibana`, 访问`localhost:5601`即可.

安装插件:

- 指定插件名称: `bin/kibana-plugin install x-pack`.
- 指定插件URL: `bin/kibana-plugin install https://artifacts.elastic.co/downloads/packs/x-pack/x-pack-6.0.0.zip`.
- 指定插件文件地址: `bin/kibana-plugin install file:///some/local/path/x-pack.zip -d path/to/directory`.

## elasticsearch入门

### 基本概念

#### 文档

- Elasticsearch是面向`文档`的, 文档是所有可搜索数据的最小单位.

  - 日志文件中的日志项
  - 一部电影的具体信息/一张唱片的详细信息
  - MP3播放器里的一首歌/一篇PDF文档中的具体内容(字符串/数值/布尔值/日期/二进制/范围类型)

- 文档会被序列化成`JSON格式`, 保存在`Elasticsearch`中.

  - JSON对象由字段组成
  - 每个字段都有对应的字段类型
  - 字段的类型可以指定或者通过Elasticsearch自动推算
  - 字段支持数组/支持嵌套

- 每个文档都有一个Unique ID

  - 可以自己指定ID
  - 或者通过ElasticSearch自动生成

文档除了基本的字段以外, 还会自动生成并携带元数据. 如:

```json
{
  "_index": "kibana_sample_data_flights",
  "_type": "_doc",
  "_id": "6ccjGHABIO0h4v7QjpZU",
  "_version": 1,
  "_score": 13.89032,
  "_source": {
    "FlightNum": "9VB8Y7X",
    "DestCountry": "US",
    "OriginWeather": "Damaging Wind",
    "OriginCityName": "Genova",
    ...
    "DestRegion": "US-MN",
    "OriginAirportID": "GE01",
    "OriginRegion": "IT-42",
    "DestCityName": "Minneapolis",
    "FlightTimeHour": 7.341976124801466,
    "FlightDelayMin": 0
  }
}
```

这是一份标准的文档, 其中文档基本的字段是存放在`_source`中, 而其它的字段以`_`开头的都是元数据: 用于标注文档的相关信息.

- `_index`: 文档所属的索引名.
- `_type`: 文档所属的类型名.
- `_id`: 文档唯一的ID.
- `_source`: 文档原始Json数据.
- `_all`: 整合所有的字段内容到该字段, 已被废除.
- `_version`: 文档的版本信息.
- `_score`: 相关性打分.

#### 索引

```json
{
  "kibana_sample_data_flights" : {
    "aliases" : { },
    "mappings" : {
      "_doc" : {
        "properties" : {
          "AvgTicketPrice" : {
            "type" : "float"
          },
          ...
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1580952946615",
        "number_of_shards" : "1",
        "number_of_replicas" : "0",
        "uuid" : "tqlAoJKFQAmGhNkcPLi3YA",
        "version" : {
          "created" : "6050099"
        },
        "provided_name" : "kibana_sample_data_flights"
      }
    }
  }
}
```

索引就是文档的容器, 一类文档的集合.

- 索引体现了逻辑空间的概念, 每个索引都有自己的`Mapping`定义, 用于定义包含文档的字段名和字段类型.
- Shard体现了物理空间的概念: 索引的数据分散在Shard上.

索引的`Mapping`和`Settings`:

- `Mapping`: 定义了文档字段的类型.
- `Setting`: 定义了不同的数据分布.

和关系数据库相比, `索引`就类比于`表`, `文档`则类比于`一条记录`, `Maping`则类比于`Schema(定义表字段)`, `DSL`则类比于`SQL`, `Field`则类比于`Column`.

不同之处在于`Elasticsearch`擅长`Schemaless/相关性/高性能全文搜索`的功能, 而关系数据库擅长`事务性/Join`.

索引相关`Rest API`示例, 请参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-indices.html):

```java
// 查看索引的信息
GET /kibana_sample_data_flights

// 查看索引文档的总数
GET kibana_sample_data_flights/_count

// 查看前10条文档，了解文档格式
POST kibana_sample_data_flights/_search


// _cat indices API
// 查看所有indicies(v用于显示列头信息)
GET /_cat/indices?v

// 查看和通配符匹配indicies（index传递给s，用于显示通配符）
GET /_cat/indices/kibana*?v&s=index

// 查看状态为绿的索引
GET /_cat/indices?v&health=green

// 按照文档个数排序(通过docs.count:desc传递给s，指定按照docs.count降序来排列)
GET /_cat/indices?v&s=docs.count:desc

// 指定查看具体的字段(pri参数指定只显示主分片信息，h指定要显示的field)
GET /_cat/indices/kibana*?pri&v&h=health,index,pri,rep,docs,count,mt

// 查看每个索引的所占的内存
GET /_cat/indices/kibana*?v&h=i,tm&s=tm:desc
```

#### 节点

`Elasticsearch`是一个分布式系统, 满足高可用性和拓展性:

- 高可用性

  - 服务可用性: 允许有节点停止服务
  - 数据可用性: 部分节点丢失, 不会丢失数据.

- 可拓展性

  - 请求量的提升/数据量的不断增长(将数据分布在所有的节点上)

`Elasticsearch`支持通过集群进行部署, 不同的集群通过名字进行区分(默认是`elasticsearch`), 可以通过配置文件或者启动时命令行(-E cluster.name=XXX)进行设定, 一个集群支持一个或者多个节点.

- 节点就是一个`Elasticsearch`实例

  - 本质上就是一个`Java进程`
  - 一台机器上可以运行多个`Elasticsearch进程`, 但是在生产上推荐一台机器只允许一个`Elasticsearch实例`.

- 每个节点都有名字, 可以通过配置文件设置或者通过启动时命令行(-E node.name=XXX)进行指定.

- 每一个节点在启动之后, 会分配一个`UID`, 保存在`data`目录下.

`Master-eligible nodes`和`Master node`:

- 每个节点启动后, 默认就是一个`Master-eligible`节点.

  - 可以设置`node.master:false`, 禁止其成为`Master-eligible`节点.

- `Master-eligible`节点可以参与选取`Master node`, 成为`Master node`.

- 当第一个节点启动时, 它会将自己选为`Master node`.

- 每个节点都保存了集群的状态, 但是只有`Master node`才可以修改集群的状态信息.

  - 集群状态(`Cluster State`), 维护了一个集群中必要的信息: 所有的节点信息, 所有的索引及其相关的`Mapping`和`Setting`信息, 分片的路由信息.
  - 任意节点都能修改信息会导致数据的不一致.

`Data node`和`Coordinating Node`:

- 可以保存数据的节点, 叫做`Data node`, 负责保存分片数据, 在数据拓展中起到重要作用.
- `Coordinating Node`负责接收`Client`请求, 将请求分发到合适的节点, 最终将结果汇集到一起. 每个节点都默认是`Coordinating Node`.

其他节点类型:

- `Hot & Warm Node`: `Hot Node`一般是配置较高的节点, 具备较好的磁盘吞吐量和带宽, 用于存储最新的数据. `Warm Node`一般配置较低, 用于存储比较旧的数据. 合理设置两种节点可以有效降低集群部署的成本.
- `Machine Learning Node`: 用于机器学习的节点.

在开发环境中一个节点可以承担多种角色, 但是在生产环境中推荐设置单一的角色节点.

节点类型              | 配置参数        | 默认值
:---------------- | :---------- | :---------------------------------------
master eligible   | node.master | true
data              | node.data   | true
ingest            | node.ingest | true(预处理节点, 每当一个新文档被索引进es时, 进行预处理工作的节点.)
coordinating only | 无           | 所有的节点默认都是该类型, 设置其他类型全部为false
machine learning  | node.ml     | true(需 enable x-pack)

#### 分片

- 主分片: 用于解决数据水平扩展的问题, 通过主分片, 可以将数据分布到集群内所有的节点上.

  - 一个分片是一个运行的`Lucene`实例.
  - 主分片数在创建时指定,后续不允许修改, 除非`Reindex`.

- 副本分片: 用于解决数据高可用的问题. 分片是主分片的拷贝.

  - 副本分片数量可以动态调整.
  - 增加副本数, 可以在一定程度上提高服务的可用性(读取的吞吐).

示例:

```json
PUT /blogs
{
  "setting": {
    "number_of_shards": 3,
    "number_of_replicas":1
  }
}
```

![例子结果](https://image.cjyong.com/es-shard.png)

对于生产环境中分片的设定, 需要提前做好容量规划(后续无法修改):

- 分片数设置过小

  - 导致后续无法增加节点实现水平扩展
  - 单个分片数据量太大, 导致数据重新分配耗时

- 分片数设置过大. 7.0开始默认主分片数量设置成1, 解决`over-sharding`问题.

  - 影响搜索结果的相关性打分, 影响统计结果的准确性.
  - 单个节点上过多的分配, 会导致资源浪费, 同时影响性能.

查询集群健康度: `GET _cluster/health`, 示例:

```json
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 9,
  "active_shards" : 9,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 5,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 64.28571428571429
}
```

其中状态分为`green`,`yellow`和`red`.

- `green`: 主分片与副本分片都正常分配.
- `yellow`: 主分片正常分片, 存在副本分片未能正常分配.
- `red`: 存在主分片未能正常分配.(需要人工介入, 查看原因).

集群API示例:

```java
// 查看集群健康信息
GET _cluster/health

// 查看节点信息
GET _cat/nodes?v

// 查看分片信息
GET _cat/shards
```

### 文档的CRUD

#### 创建文档

```json
// 注意es7.x, 使用: POST users/_create/1, 以下为es6.x使用格式.
PUT users/_doc/1/_create
{
  "firstName": "Jack",
  "lastName": "Johnson",
  "tags": ["guitar", "skateboard"]
}

// 返回结果
{
  "_index" : "users",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

- 注意在`ES6.X`的版本中不支持`_create`, 需要使用`Index 文档`.
- 在上面的请求中, 指定`index`为`users`, 文档ID为`1`.
- 如果调用`POST users/_doc`方式来创建的话, 系统会默认生成一个`ID`.
- 如果`PUT`请求中ID重复, 则会报错.

#### 获取文档

```json
GET users/_doc/1

// 返回结果
{
  "_index" : "users", // 索引名称
  "_type" : "_doc",   // 索引类型(7.0开始固定为_doc)
  "_id" : "1",        // 该条记录ID
  "_version" : 1,     // 该条记录修改的版本
  "found" : true,     // 已经找到
  "_source" : {       // 源数据
    "firstName" : "Jack",
    "lastName" : "Johnson",
    "tags" : [
      "guitar",
      "skateboard"
    ]
  }
}
```

如果找不到文档, 返回`HTTP 404`.

#### 删除文档

```json
DELETE users/_doc/1

// 返回结果
{
  "_index" : "users",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 4,
  "result" : "deleted",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 3,
  "_primary_term" : 1
}
```

#### Index 文档

`Index`和`Create`的区别在于: 如果文档不存在, 就索引新的文档. 如果存在, 会先将当前存在文档删除, 在插入进行索引, 这时候版本号会加一.

```json
PUT users/_doc/1

// 返回结果
{
  "_index" : "users",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "found" : true,
  "_source" : { // 注意这里覆盖了原来的文档
    "tags" : [
      "guitar",
      "skateboard",
      "reading"
    ]
  }
}
```

#### 更新文档

```json
// 注意es7.x, 使用: POST users/_update/1, 以下为es6.x使用格式.
POST users/_doc/1/_update
{
  "doc": {
    "albums": ["Album1", "Album2"]
  }
}

// 返回结果
{
  "_index" : "users",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 4,
  "result" : "noop",
  "_shards" : {
    "total" : 0,
    "successful" : 0,
    "failed" : 0
  }
}
```

- 更新文档不会删除原来的文档， 而是实现真正的数据更新。
- Post方法 / Payload需要包含在`doc`中.

`REST请求汇总`:

```json
DELETE users

// create document. 自动生成_id
POST users/_doc
{
  "name": "Jacika",
  "post_date": "2019-04-15",
  "message": "I trying!"
}

// create document. 指定ID. 如果ID存在则报错。
PUT users/_doc/1?op_type=create
{
  "name": "Jack",
  "post_date": "2019-04-15",
  "message": "I will not try!"
}


// create document. 指定ID. 如果ID存在则报错。es7使用: users/_create/1
PUT users/_doc/1/_create
{
  "name": "Jack",
  "post_date": "2019-04-15",
  "message": "I will not try!"
}

// Get document
GET users/_doc/1

// Delete document
DELETE users/_doc/1

// Index document
PUT users/_doc/1
{
  "name": "Mike"
}

// Update document, es7.x使用 users/_update/1
POST users/_doc/1/_update
{
  "doc": {
    "nick_name": "small mike"
  }
}
```

#### Bulk API

- 支持在一次API调用中, 对不同的索引操作.
- 支持四种操作(`Index`, `Create`, `Update`, `Delete`).
- 可以在URI中指定`Index`, 也可以在请求的`Payload`中进行.
- 操作中单条操作失败, 并不会影响其他操作.
- 返回结果包括了每一条操作执行的结果.

```json
// Bulk 操作
POST _bulk
{ "index" : { "_index" : "test", "_type" : "_doc", "_id" : "1" } } // 指定index
{ "field1" : "value1" } //指定index的field只有一个filed1
{ "delete" : { "_index" : "test", "_type" : "_doc", "_id" : "2" } } // 删除id为2的记录
{ "create" : { "_index" : "test", "_type" : "_doc", "_id" : "3" } } // 创建一个document, 指定id为3
{ "field1" : "value3" } // 内容为{filed1: value3}
{ "update" : {"_id" : "1", "_type" : "_doc", "_index" : "test"} } //更新document
{ "doc" : {"field2" : "value2"} } // 更新的内容为: {field2: value2}

// 返回结果
{
  "took" : 60,
  "errors" : true,
  "items" : [
    {
      "index" : {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "1",
        "_version" : 3,
        "result" : "updated",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 2,
        "_primary_term" : 1,
        "status" : 200
      }
    },
    {
      "delete" : {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "2",
        "_version" : 1,
        "result" : "not_found",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 404
      }
    },
    {
      "create" : {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "3",
        "status" : 409,
        "error" : {
          "type" : "version_conflict_engine_exception",
          "reason" : "[_doc][3]: version conflict, document already exists (current version [1])",
          "index_uuid" : "d6T9QLBLScud6xc4stS-Ag",
          "shard" : "4",
          "index" : "test"
        }
      }
    },
    {
      "update" : {
        "_index" : "test",
        "_type" : "_doc",
        "_id" : "1",
        "_version" : 4,
        "result" : "updated",
        "_shards" : {
          "total" : 2,
          "successful" : 2,
          "failed" : 0
        },
        "_seq_no" : 3,
        "_primary_term" : 1,
        "status" : 200
      }
    }
  ]
}
```

#### 批量读取 - mget

批量操作, 减少网络连接带来的开销.

```json
GET _mget
{
  "docs": [
    {
      "_index": "user",
      "_id": 1
    },
    {
      "_index": "user",
      "_id": 1
    }
  ]
}

// 返回结果

{
  "docs" : [
    {
      "_index" : "user",
      "_type" : null,
      "_id" : "1",
      "error" : {
        "root_cause" : [
          {
            "type" : "index_not_found_exception",
            "reason" : "no such index",
            "resource.type" : "index_expression",
            "resource.id" : "user",
            "index_uuid" : "_na_",
            "index" : "user"
          }
        ],
        "type" : "index_not_found_exception",
        "reason" : "no such index",
        "resource.type" : "index_expression",
        "resource.id" : "user",
        "index_uuid" : "_na_",
        "index" : "user"
      }
    },
    {
      "_index" : "user",
      "_type" : null,
      "_id" : "1",
      "error" : {
        "root_cause" : [
          {
            "type" : "index_not_found_exception",
            "reason" : "no such index",
            "resource.type" : "index_expression",
            "resource.id" : "user",
            "index_uuid" : "_na_",
            "index" : "user"
          }
        ],
        "type" : "index_not_found_exception",
        "reason" : "no such index",
        "resource.type" : "index_expression",
        "resource.id" : "user",
        "index_uuid" : "_na_",
        "index" : "user"
      }
    }
  ]
}
```

#### 批量查询 - msearch

```json
// msearch 操作
POST kibana_sample_data_ecommerce/_msearch
{}  //不设置任何参数
{"query": {"match_all": {}}, "size": 1}
{"index": "kibana_sample_data_flights"} // 设置index为flights
{"query": {"match_all": {}}, "size": 2}
```

### 倒排索引

倒排索引与正排索引的区别:

在图书上:

- 正排索引: 目录页
- 倒排索引: 索引页, 单词到页数.

在搜索引擎中:

- 正排索引: 文档ID到文档内容和单词的关联.
- 倒排索引: 单词到文档ID的关系.

如: 正排索引

文档ID | 文档内容
:--- | :-----------------------
1    | Mastering Elasticsearch
2    | Elasticsearch Server
3    | Elasticsearch Essentials

倒排索引

Term          | Count | DocumentId:Position
:------------ | :---- | :------------------
Elasticsearch | 3     | 1:1, 2:0, 3:0
Mastering     | 1     | 1:0
Server        | 1     | 2:1
Essentials    | 1     | 3:1

#### 核心组成

倒排索引包含两个部分

- 单词词典(Term Dictionary), 记录所有文档的单词, 记录单词到倒排列表的关联关系.

  - 一般词典非常大, 可以通过B+树或哈希拉链法实现, 以满足高性能插入与查询.

- 倒排列表(Posting List): 记录了单词对应的文档集合, 由倒排索引项组成.

  - 倒排索引项(Posting)

    - 文档ID
    - 词频TF: 单词在该文档中出现的次数, 用于相关性评分.
    - 位置(Position): 单词在文档分词的位置, 用于语句搜索.
    - 偏移(Offset): 记录单词开始和结束的位置, 实现高亮显示.

如单词`Elasticsearch`在文档中关联的倒排列表为

Doc Id | 词频TF | Position | Offset
:----- | :--- | :------- | :-----
1      | 1    | 1        | 10:23
2      | 1    | 0        | 0:13
3      | 1    | 0        | 0:13

ElasticSearch的JSON文件中的每一个字段, 都有自己的倒排索引. 也可以指定某些字段不做索引.

- 优点: 节省存储空间.
- 缺点: 字段无法被搜索.

### 分词器简介

- Analysis: 将全文本转换一系列单词(term/token)的过程, 也叫分词.
- Analysis通过Analyzer实现, 可以使用es内置的分词器或者按需定制化分词器.
- 除了在数据写入的时候进行分词处理, 在查询的时候(匹配Query)的时候也分词器进行处理(一般默认是相同的).

#### Analyzer的组成

`Character Filters -> Tokenizer -> Token Filters`:

- `Character Filters`: 针对原始文本进行处理, 如去除`html`标签.
- `Tokenizer`: 按照规则进行切词.
- `Token Filters`: 对切分之后的结果进行进一步加工, 如大写转小写, 删除`stopwards`, 增加同义词等等.

#### es内置的分词器

- `Standard Analyzer`: 默认分词器,按词切分,小写处理.
- `Simple Analyzer`: 按照非字母切分(符号过滤), 小写处理.
- `Stop Analyzer`: 小写处理, 停用词过滤(the, a, is)
- `Whitespace Analyzer`: 按照空格切分, 不转小写.
- `Keyword Analyzer`: 不分词, 直接将输入当输出.
- `Pattern Analyzer`: 正则表达式, 默认\W+(非字符分隔).
- `Language`: 提供了30多种常见语言的分词器.
- `Customer Analyzer`: 自定义分词器.

#### 分词器API的使用

```json
// 指定分词器进行分词
GET /_analyze
{
  "analyzer": "standard",
  "text": "Mastering Elasticsearch, elasticsearch in action"
}

// 指定索引字段所对应的分词器进行分词
POST books/_analyze
{
  "field": "title",
  "text": "Mastering Elasticsearch"
}

// 自定义分词器进行分词
POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase"],
  "text": "Mastering Elasticsearch"
}
```

#### 常见标准分词器介绍

`Standard Analyzer`:

![StandardAnalyzer](https://image.cjyong.com/StandardAnalyzer.png)

```json
// standard
POST /_analyze
{
  "analyzer": "standard",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "2",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<NUM>",
      "position" : 0
    },
    {
      "token" : "running",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "brown",
      "start_offset" : 16,
      "end_offset" : 21,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "foxes",
      "start_offset" : 22,
      "end_offset" : 27,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "lazy",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "dogs",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 8
    },
    {
      "token" : "in",
      "start_offset" : 48,
      "end_offset" : 50,
      "type" : "<ALPHANUM>",
      "position" : 9
    },
    {
      "token" : "the",
      "start_offset" : 51,
      "end_offset" : 54,
      "type" : "<ALPHANUM>",
      "position" : 10
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "<ALPHANUM>",
      "position" : 11
    },
    {
      "token" : "evening",
      "start_offset" : 62,
      "end_offset" : 69,
      "type" : "<ALPHANUM>",
      "position" : 12
    }
  ]
}
```

`Simple Analyzer`:

![SimpleAnalyzer](https://image.cjyong.com/SimpleAnalyzer.png)

```json
// simple
POST /_analyze
{
  "tokenizer": "simple",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "running",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 16,
      "end_offset" : 21,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "foxes",
      "start_offset" : 22,
      "end_offset" : 27,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "lazy",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "dogs",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "in",
      "start_offset" : 48,
      "end_offset" : 50,
      "type" : "word",
      "position" : 8
    },
    {
      "token" : "the",
      "start_offset" : 51,
      "end_offset" : 54,
      "type" : "word",
      "position" : 9
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "word",
      "position" : 10
    },
    {
      "token" : "evening",
      "start_offset" : 62,
      "end_offset" : 69,
      "type" : "word",
      "position" : 11
    }
  ]
}
```

`Whitespace Analyzer`:

![WhitespaceAnalyzer](https://image.cjyong.com/WhitespaceAnalyzer.png)

```json
// simple
POST /_analyze
{
  "tokenizer": "whitespace",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "2",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "running",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "Quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "brown-foxes",
      "start_offset" : 16,
      "end_offset" : 27,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "lazy",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "dogs",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "in",
      "start_offset" : 48,
      "end_offset" : 50,
      "type" : "word",
      "position" : 8
    },
    {
      "token" : "the",
      "start_offset" : 51,
      "end_offset" : 54,
      "type" : "word",
      "position" : 9
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "word",
      "position" : 10
    },
    {
      "token" : "evening.",
      "start_offset" : 62,
      "end_offset" : 70,
      "type" : "word",
      "position" : 11
    }
  ]
}
```

`Stop Analyzer`:

![StopAnalyzer](https://image.cjyong.com/StopAnalyzer.png)

```json
// stop
POST /_analyze
{
  "tokenizer": "stop",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "running",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "brown",
      "start_offset" : 16,
      "end_offset" : 21,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "foxes",
      "start_offset" : 22,
      "end_offset" : 27,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "lazy",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "dogs",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "word",
      "position" : 10
    },
    {
      "token" : "evening",
      "start_offset" : 62,
      "end_offset" : 69,
      "type" : "word",
      "position" : 11
    }
  ]
}
```

`Keyword Analyzer`:

![StopAnalyzer](https://image.cjyong.com/KeywordAnalyzer.png)

```json
// keyword
POST /_analyze
{
  "tokenizer": "keyword",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "2 running Quick brown-foxes leap over lazy dogs in the summer evening.",
      "start_offset" : 0,
      "end_offset" : 70,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

`Pattern Analyzer`:

![PatternAnalyzer](https://image.cjyong.com/PatternAnalyzer.png)

```json
// pattern
POST /_analyze
{
  "tokenizer": "pattern",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "2",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "running",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "word",
      "position" : 1
    },
    {
      "token" : "quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "brown",
      "start_offset" : 16,
      "end_offset" : 21,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "foxes",
      "start_offset" : 22,
      "end_offset" : 27,
      "type" : "word",
      "position" : 4
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "word",
      "position" : 5
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "word",
      "position" : 6
    },
    {
      "token" : "lazy",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "word",
      "position" : 7
    },
    {
      "token" : "dogs",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "word",
      "position" : 8
    },
    {
      "token" : "in",
      "start_offset" : 48,
      "end_offset" : 50,
      "type" : "word",
      "position" : 9
    },
    {
      "token" : "the",
      "start_offset" : 51,
      "end_offset" : 54,
      "type" : "word",
      "position" : 10
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "word",
      "position" : 11
    },
    {
      "token" : "evening",
      "start_offset" : 62,
      "end_offset" : 69,
      "type" : "word",
      "position" : 12
    }
  ]
}
```

`Language Analyzer`:

```json
// language - english
POST /_analyze
{
  "tokenizer": "english",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}

// result
{
  "tokens" : [
    {
      "token" : "2",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<NUM>",
      "position" : 0
    },
    {
      "token" : "run",
      "start_offset" : 2,
      "end_offset" : 9,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "quick",
      "start_offset" : 10,
      "end_offset" : 15,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "brown",
      "start_offset" : 16,
      "end_offset" : 21,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
    {
      "token" : "fox",
      "start_offset" : 22,
      "end_offset" : 27,
      "type" : "<ALPHANUM>",
      "position" : 4
    },
    {
      "token" : "leap",
      "start_offset" : 28,
      "end_offset" : 32,
      "type" : "<ALPHANUM>",
      "position" : 5
    },
    {
      "token" : "over",
      "start_offset" : 33,
      "end_offset" : 37,
      "type" : "<ALPHANUM>",
      "position" : 6
    },
    {
      "token" : "lazi",
      "start_offset" : 38,
      "end_offset" : 42,
      "type" : "<ALPHANUM>",
      "position" : 7
    },
    {
      "token" : "dog",
      "start_offset" : 43,
      "end_offset" : 47,
      "type" : "<ALPHANUM>",
      "position" : 8
    },
    {
      "token" : "summer",
      "start_offset" : 55,
      "end_offset" : 61,
      "type" : "<ALPHANUM>",
      "position" : 11
    },
    {
      "token" : "even",
      "start_offset" : 62,
      "end_offset" : 69,
      "type" : "<ALPHANUM>",
      "position" : 12
    }
  ]
}
```

#### 中文分词的难点

- 中文语句, 切分成一个一个词(而不是一个一个字)
- 英文中, 单词有自然的空格作为间隔
- 一句中文中, 在不同的上下文, 有不同的理解

  - 这个苹果, 不大好吃 / 这个苹果, 不大, 好吃

常见的中文分词器: `ICU Analyzer`, `IK`, `THULAC`.

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "他说的的确在理"
}

// result
{
  "tokens" : [
    {
      "token" : "他",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "说",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "的",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "的",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    },
    {
      "token" : "确",
      "start_offset" : 4,
      "end_offset" : 5,
      "type" : "<IDEOGRAPHIC>",
      "position" : 4
    },
    {
      "token" : "在",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "<IDEOGRAPHIC>",
      "position" : 5
    },
    {
      "token" : "理",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "<IDEOGRAPHIC>",
      "position" : 6
    }
  ]
}


// ik_smart
POST /_analyze
{
  "analyzer": "ik_smart",
  "text": "他说的的确在理"
}
{
  "tokens" : [
    {
      "token" : "他",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "说",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "的",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "CN_CHAR",
      "position" : 2
    },
    {
      "token" : "的确",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "在理",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 4
    }
  ]
}
```

### Search API

- URL Search: 在URL中查询参数
- Request Body Search: 使用es提供的基于JSON格式的更加完备查询语句(DSL).

#### 指定查询的索引

- `/_search`: 集群里所有的索引.
- `/index1/_search`: index1索引.
- `index1,index2/_search`: index1和index2.
- `index*/_search`: index开头的索引.

`URL查询`:

`http://elasticsearch:9200/kibana_sample_data_ecommerce/_search?q=customer_first_name:Eddie`

在`q`后面添加查询参数, KV键值对.

`Request Body`:

```shell
curl -XGET "http://elasticsearch:9200/kibana_sample_data_ecommerce/_search" -H 'Content-Type: application/json' -d '
{
  "query": {
    "match_all": {}
  }
}'
```

#### 搜索的相关性

- 搜索是用户和搜索引擎的对话.
- 用户关心搜索结果的相关性

  - 是否可以找到所有相关的内容.
  - 有多少不相关的内容被返回了
  - 文档的打分是否合理
  - 综合业务需求, 平衡结果排名

衡量标准:

- Precision(查准率): 尽可能返回少的无关文档.
- Recall(查全率): 尽量返回较多的相关文档.
- Ranking: 是否能够按照相关度进行排序.

#### URI Search详解

`GET /movies/_search?q=2012&df=title&sort=year:desc&from=0&size=10&timeout=1s { "profile": true }`

- `q`: 指定查询语句, 使用Query String Syntax.
- `df`: 指定默认字段, 不指定时, 会对所有字段进行查询.
- `sort`: 排序.
- `form/size`: 分页.
- `Profile`: 查看查询是如何执行的, 如果设置为`true`,会把执行的过程展示出来.

##### 指定字段查询

如果没有指定字段则为泛查询, 如:

```json
// 查询所有包含2012字符串的index
GET /movies/_search?q=2012

// 设置profile之后, 执行的过程:
"type" : "DisjunctionMaxQuery",
"description" : "(title.keyword:2012 | id.keyword:2012 | year:[2012 TO 2012] | genre:2012 | @version:2012 | @version.keyword:2012 | id:2012 | genre.keyword:2012 | title:2012)"
```

指定字段查询:

```json
// 方式一: 设置df, 默认字段
GET /movies/_search?q=2012&df=title

// 方式二: 在q后面使用kv结构
GET /movies/_search?q=title:2012

// 执行过程:
"type" : "TermQuery",
"description" : "title:2012"
```

##### Term or Phrase

- Beautiful Mind等效于Beautiful OR Mind.
- "Beautiful Mind"等效于Beautiful AND Mind. Phrase查询, 还要求前后顺序保持一致.

```json
// Phrase search
GET /movies/_search?q=title:"Beautiful Mind"
{
  "profile": "true"
}

// 执行过程
"type" : "PhraseQuery",
"description" : """title:"beautiful mind"""",


// Term search
GET /movies/_search?q=title:(Beautiful Mind)
{
  "profile": "true"
}
// 执行过程
"type" : "BooleanQuery",
"description" : "title:beautiful title:mind",
```

##### 逻辑运算

在分组内(`()`)默认是, `OR`关系, 此外还支持`AND`, `NOT`, `MUST`, `MUST NOT`逻辑操作:

```json
// AND search
GET /movies/_search?q=title:(Beautiful AND Mind)
{
  "profile": "true"
}

// OR search
GET /movies/_search?q=title:(Beautiful OR Mind)
{
  "profile": "true"
}

// NOT search
GET /movies/_search?q=title:(Beautiful NOT Mind)
{
  "profile": "true"
}

// must search, %2B在URL编码中为+
GET /movies/_search?q=title:(Beautiful %2BMind)
{
  "profile": "true"
}

// must not search, %2B在URL编码中为+
GET /movies/_search?q=title:(-Beautiful %2BMind)
{
  "profile": "true"
}
```

##### 范围与数学运算

- 范围查询: []表示闭合区间, {}表示开区间. 如`year:{1989 TO 2018], year:[* TO 2018]`
- 算数符号: `year:>2010, year:(>2010 && <2018), year:(+>2010 +<2018)`

##### 正则查询/通配符查询

- 通配符查询(查询效率低, 占用内存高, 不建议使用)

  - ?:一个字符, _: 0或者多个字符: `title:mi?d/title:be_`

- 正则表达式: `title:[bt]oy`

- 模糊查询: `title:befutifl~1,title:"lord rings"~2`

```json
// 通配符查询，title中包含b开头字母的
GET /movies/_search?q=title:b*
{
  "profile": "true"
}

// 模糊匹配&近似匹配, 预防单词输入错误，如beautifl
GET /movies/_search?q=title:beautifl~1
{
  "profile": "true"
}

// 允许在Lord Rings内部插入两个单词
GET /movies/_search?q=title:"Lord Rings"~2
{
  "profile": "true"
}
```

#### Request Body详解

将查询语句通过`HTTP Request Body`发送给ES. 如下面的例子:

```json
POST kibana_sample_data_ecommerce/_search
{
  "profile": true,  //显示执行过程
  "sort": [{"order_date":"desc"}],  //按照下单日期降序排序
  "_source": ["order_date"],  //只取order_date字段信息,
  "from": 10, //分页
  "size": 5,
  "query": {
    "match_all": {}
  }
}
```

##### 脚本字段

对返回结果中的字段进行处理, 如:

```json
GET kibana_sample_data_ecommerce/_search
{
  "script_fields": {
    "new_field": {  // field的新名称
      "script": {
        "lang": "painless",
        "source": "doc['order_date'].value + 'hello'" // 结果内容
      }
    }
  },
  "from": 0,
  "size": 5,
  "query": {
    "match_all": {}
  }
}
```

##### Term/PharseSearch

```json
// TERM search
POST movies/_search
{
  "query": {
    "match": {
      "title": "Last Christmas"
    }
  }
}

POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "Last Christmas",
        "operator": "and"
      }
    }
  }
}

// Phase search
POST movies/_search
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "one love",
        "slop": 1 // 允许中间插入一个单词
      }
    }
  }
}
```

##### QueryString查询

`Query String`: 类似`URI Query`:

```json
POST /users/_search
{
  "query": {
    "query_string": {
      "default_field": "name",
      "query": "Ruan AND Yiming"
    }
  }
}

POST /users/_search
{
  "query": {
    "query_string": {
      "fields": ["name", "about"],
      "query": "(Ruan AND Yiming) OR (Java AND Elasticsearch)"
    }
  }
}
```

`Simple Query String`:

- 类似`Query String`, 只支持部分语法, 会忽略错误的语法.
- 不支持`AND, OR, NOT`, 会当做字符串进行处理.
- Term之间默认关系为`OR`, 可以指定`Operator`.
- 支持部分逻辑: `+`: AND, `-`: NOT, `|`替代OR.

```json
POST /users/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["name"],
      "query": "Ruan Yiming",
      "default_operator": "AND"
    }
  }
}
```

### Mapping

- Mapping类似数据库中的`schema`的定义:

  - 定义索引中字段名称
  - 定义字段的数据类型, 如字符串, 数字, 布尔值...
  - 字段, 倒排索引相关的配置

- Mapping会把JSON文档隐射成Lucene所需要的扁平格式

- 一个Mapping属于一个索引的Type.

`字段的数据类型`:

- 简单类型: Text/Keyword, Date, Integer/Floating, Boolean, IPv4/IPv6.
- 复杂类型: 对象和嵌套对象.
- 特殊类型: geo_point & geo_shape / percolator

#### Dynamic Mapping

- 写入文档时, 如果索引不存在, 会自动创建一个索引.
- `Dynamic Mapping`使得我们不需要手动定义`Mapping`, 而是通过自动推断得出字段类型(注意这里的推断, 不一定百分百符合预期).

#### 能否更改Mapping的字段

- 新增字段:

  - Dynamic设为true时, 一旦新增字段写入, Mapping也同时被更新.
  - Dynamic设为false时, 一旦新增字段写入, Mapping不会更新, 新增字段的数据无法被索引, 但是信息会出现在_source中.
  - Dynamic设置Strict, 文档写入失败.

- 已有字段, 一旦有数据写入, 就不再支持修改字段定义. 如果希望改变字段类型, 必须Reindex API, 重建索引.

新建时, 设置`dynamic`:

```json
PUT movies
{
  "mappings": {
    "_doc": {
      "dynamic": "false"
    }
  }
}

// 更新dynamic
PUT movies/_mapping
{
  "dynamic": "false"
}
```

#### 自定义Mapping

- 可以参考API手册, 纯手写.
- 为了减少输入的工作量, 减少出错的步骤, 可以参照以下步骤:

  - 创建一个临时的index, 写入一些样本数据
  - 通过访问Mapping API获得临时文件的动态Mapping定义
  - 修改后用, 使用该配置创建你的索引
  - 删除临时索引

在`mapping`中可以设置字段的`index`值, 默认为`true`, 如果设置为`false`, 该字段则无法被搜索. 其中字段的索引(倒排索引)可以设置不同的级别:

- docs: 记录 doc id.(只记录关联文档的ID)
- freqs: 记录 doc id 和 term frequencies.
- positions: 记录 doc id / term frequencies / term position
- offsets: 记录 doc id / term frequencies / term position / character offects

`Text字段`默认的索引级别为`positions`, 其他默认为`docs`. 记录的内容越多, 占用存储空间越大.

对于某些需要进行`NULL`值进行搜索, 可以在`mapping`中设置`null_value`, 记住只有`keyword`类型字段才允许设置该字段.

`copy_to`字段, 则是可以将字段的值拷贝到目标字段, 满足一些特定的需求(类似_all的功能). 但是`copy_to`生成的新字段, 不会出现在`_source`中.

##### 多字段类型

可以在字段定义中, 在某一个字段中添加新字段以满足不同的需求, 也就是多字段类型. 如:

```json
PUT products
{
  "mappings": {
    "properties": {
      "company": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "comment": {
        "type": "text",
        "fields": {
          "english_comment": {
            "type": "text",
            "analyzer": "english",
            "search_analyzer": "english"
          }
        }
      }
    }
  }
}
```

### Index Template

帮助你设定`Mappings`和`Settings`, 并按照一定的规则自动匹配到新创建的索引上.

- Template只在创建的时候, 才会产生作用. 修改模板不会影响已创建的索引.
- 可以设定多个索引模板, 这些设置会被`merge`在一起.
- 可以指定`order`, 控制`merge`的过程(就是覆盖的过程, order低的模板, 会先应用. 即会被后面的覆盖掉).

如:

```json
PUT _template/template_default
{
  "index_patterns": ["*"], //匹配那些索引
  "order": 0,
  "version": 1,
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}
```

#### Dynamic Template

设置在特定的`index`中, 会根据`ES`识别的数据类型, 结合字段名称, 动态设置字段类型. 如:

- is开头的字段设置为boolean.
- long_开头的字段都设置为long类型.

```json
PUT my_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_boolean": {
        "match_mapping_type": "string",
        "match": "is*",
        "mapping": {
          "type": "boolean"
        }
      }
    }
    ]
  }
}
```

### 聚合分析简介

聚合分析分类:

- Bucket Aggregation: 一些列满足特定条件的文档的集合, 类似于SQL中的分组.
- Metric Aggregation: 一些数学运算, 可以对文档字段进行统计分析. 如最大值, 最小值, 平均值等.
- Pipeline Aggregation: 对其他的聚合结果进行二次聚合.
- Matrix Aggregation: 对多个字段操作并提供一个结果矩阵.

`Bucket`例子:

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry"  //根据目的地进行分组
      }
    }
  }
}
```

`Metric`例子:

```json
GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry"
      },
      "aggs": {
      "average_price": {
        "avg": {
          "field": "AvgTicketPrice"
        }
      },
      "max_price": {
        "max": {
          "field": "AvgTicketPrice"
        }
      },
      "min_price": {
        "min": {
          "field": "AvgTicketPrice"
        }
      }
    }
    }
  }
}

GET kibana_sample_data_flights/_search
{
  "size": 0,
  "aggs": {
    "flight_dest": {
      "terms": {
        "field": "DestCountry"
      },
      "aggs": { // 这里进行了嵌套分组, 即: 先按照目标城市分组, 再按照天气分组
        "weather": {
          "terms": {
            "field": "DestWeather"
          }
        },
        "sts_price": {
          "stats": {
            "field": "AvgTicketPrice"
          }
        }
      }
    }
  }
}
```
