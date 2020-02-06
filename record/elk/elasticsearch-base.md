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

- `应用程序 -> 写入/更新 -> 数据库 -> ELK -> 查询(应用程序)`
- `beats(采集) -> redis/kafka/rabbitmq(缓冲) -> logstash(收集和预处理) -> elk(处理) -> kibana/grafana(可视化)`

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
PUT users/_create/1
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
// 注意es7.x, 使用: POST users/_create/1, 以下为es6.x使用格式.
GET users/_doc/1/_create

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

Doc Id | Position | Offset
:----- | :------- | :-----
1      | 1        | 10:23
2      | 0        | 0:13
3      | 0        | 0:13

ElasticSearch的JSON文件中的每一个字段, 都有自己的倒排索引. 也可以指定某些字段不做索引.

- 优点: 节省存储空间.
- 缺点: 字段无法被搜索.
