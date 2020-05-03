# ElasticSearch 分布式特性

ElasticSearch天生支持分布式集群的架构和部署, 本节深入了解其分布式特性及其实现原理.

## 集群分布式模型及选主与脑裂问题

Elasticsearch 的分布式架构带来的好处

- 存储的水平扩容，⽀支持 `PB` 级数据
- 提⾼系统的可⽤用性，部分节点停⽌服务，整个集群的服务不受影响

Elasticsearch 的分布式架构:

- 不同的集群通过不同的名字来区分，默认名字 `elasticsearch`
- 可以通过配置⽂件修改，或者在命令⾏中 `-E cluster.name=geektime` 进行设定

`节点`:

- 节点是一个 `Elasticsearch` 的实例

  - 其本质上就是⼀一个 `JAVA` 进程
  - 一台机器上可以运行多个 `Elasticsearch` 进程，但是⽣产环境一般建议一台机器上就运行⼀一个 `Elasticsearch` 实例

- 每⼀个节点都有名字，通过配置⽂件配置，或者启动时候设置 `-E node.name=NODE_NAME` 来指定

- 每一个节点在启动之后，会分配⼀一个`UID`, 保存在 `data` 目录下

`Coordinating Node`:

- 处理请求的节点，叫 `Coordinating Node`

- 路由请求到正确的节点，例如创建索引的请求，需要路由到 `Master` 节点

- 所有节点默认都是 `Coordinating Node`

- 通过将该节点的其他类型设置成 `False`，使其成为 `Dedicated Coordinating Node`, 这样保证所有的请求都通过这个节点.

`Data Node`:

- 可以保存数据的节点，叫做 `Data Node`

  - 节点启动后，默认就是数据节点。可以设置 `node.data: false` 禁用数据节点的功能.

- Data Node的职责

  - 保存分片数据。在数据扩展上起到了至关重要的作⽤用(由 `Master Node` 决定如何把分⽚分发到数据节点上)

- 通过增加数据节点

  - 可以解决数据水平扩展和数据单点问题

`Master Node`:

- `Master Node` 的职责

  - 处理理创建，删除索引等请求 /决定分片被分配到哪个节点 / 负责索引的创建与删除
  - 维护并且更新 `Cluster State`

- `Master Node` 的最佳实践

  - `Master` 节点⾮常重要，在部署上需要考虑解决单点的问题
  - 为⼀个集群设置多个 `Master` 节点 / 每个节点只承担 `Master` 的单一⻆色(专用的Master节点, 不承担其他功能)

`Master Eligible Nodes` & 选主流程 :

- 一个集群，⽀持配置多个 `Master Eligible` 节点。这些节点可以在必要时(如 `Master` 节点出现故障，⽹络故障时)参与选主流程，成为 `Master` 节点.

- 每个节点启动后，默认就是一个 `Master eligible` 节点

  - 可以设置 `node.master: false` 禁⽌该特点

- 当集群内第⼀个 `Master eligible` 节点启动时候，它会将⾃己选举成 `Master` 节点

- 其他节点会加入集群，但是不承担 `Master` 节点的⻆色。

- ⼀旦发现被选中的主(Master)节点丢失， 就会选举出新的 `Master` 节点: 互相Ping对方, NodeId低的节点会成为被选举的节点.

`集群状态`:

- 集群状态信息(Cluster State)，维护了了一个集群中，必要的信息

  - 所有的节点信息
  - 所有的索引和其相关的 Mapping 与 Setting 信息
  - 分⽚的路由信息

- 在每个节点上都保存了集群的状态信息

- 但是，只有 Master 节点才能修改集群的状态信息，并负责同步给其他节点

  - 因为，任意节点都能修改信息会导致 `Cluster State` 信息的不一致

### 脑裂问题

`脑裂问题`:

- `Split-Brain`: 分布式系统的经典⽹网络问题，(假设当前情况为三个节点Node1, Node2, Node3, 其中Node1为Master节点) 当出现网络问题，⼀个节点(Node1)和其他节点无法连接时

- Node 2 和 Node 3 会重新选举 Master

- Node 1 ⾃己还是作为 `Master Node`，组成一个集群，同时更新 `Cluster State`

- 导致 2 个 `Master Node`，维护不同的 `cluster state`。当⽹络恢复时，⽆法选择正确恢复.

`如何避免`:

- 限定⼀个选举条件，设置 `quorum(仲裁)`，只有在 `Master eligible` 节点数⼤于 `quorum` 时，才能进⾏选举

  - `Quorum = (master 节点总数 /2)+ 1`
  - 当 3 个 master eligible 时，设置 discovery.zen.minimum_master_nodes 为 2，即可避免脑裂

- 从 7.0 开始，⽆无需这个配置

  - 移除 `minimum_master_nodes` 参数，让Elasticsearch⾃己选择可以形成仲裁的节点。
  - 典型的主节点选举现在只需要很短的时间就可以完成。集群的伸缩变得更安全、更容易，并且可能造成丢失数据的系统配置选项更少了。
  - 节点更清楚地记录它们的状态，有助于诊断为什么它们不能加⼊集群或为什么无法选举出主节点

### 配置节点的类型

一个节点默认情况下是一个 `Master eligible，data and ingest node`:

节点类型              | 配置参数        | 默认值
:---------------- | :---------- | :---------------------
maste eligible    | node.master | true
data              | node.data   | true
ingest            | node.ingest | ture
coordinating only | ⽆           | 设置上⾯三个参数全部为false
machine learning  | node.ml     | true (需要enable x-pack)

## 分片与集群的故障转移

`Primary Shard` - 提升系统存储容量:

- 分⽚是 `Elasticsearch` 分布式存储的基⽯

  - 主分⽚ / 副本分⽚

- 通过主分片，将数据分布在所有节点上

  - `Primary Shard`，可以将⼀份索引的数据，分散在多个 `Data Node` 上，实现存储的⽔平扩展
  - 主分片(Primary Shard)数在索引创建时候指定，后续默认不能修改，如要修改，需重建索引

`Replica Shard` - 提⾼数据可⽤性:

- 数据可⽤性

  - 通过引入副本分片 (Replica Shard) 提⾼数据的可用性。⼀旦主分⽚丢失，副本分片可以 `Promote` 成主分片。副本分⽚数可以动态调整。每个节点上都有完备的数据。如果不设置副本分⽚，⼀旦出现节点硬件故障，就有可能造成数据丢失.

- 提升系统的读取性能

  - 副本分片由主分片(Primary Shard)同步。通过支持增加 Replica 个数，⼀定程度可以提高读取的吞吐量

`如何规划⼀个索引的主分⽚数和副本分⽚数`:

- 主分⽚数过小:例如创建了了 1 个 Primary Shard 的 Index

  - 如果该索引增⻓很快，集群⽆法通过增加节点实现对这个索引的数据扩展

- 主分片数设置过⼤:导致单个 Shard 容量很⼩，引发一个节点上有过多分⽚片，影响性能

- 副本分⽚数设置过多，会降低集群整体的写⼊性能

例子:

```json
PUT tmdb 
{
  "setting": {
    "number_of_shards": 3,
    "number_of_replicas":1
  }
}
```

**单节点集群**:

![single_node](https://image.cjyong.com/sigle_node_cluster.png)

- 副本无法分片，集群状态⻩色

**双节点集群**:

![double_node](https://image.cjyong.com/double_nodes_cluster.png)

- 集群状态转为绿⾊
- 集群具备故障转移能⼒

**三节点集群**:

![three_node](https://image.cjyong.com/three_nodes_cluster.png)

- 集群状态转为绿⾊
- 集群具备故障转移能⼒
- Master 节点会决定分片分配到哪个节点
- 通过增加节点，提⾼集群的计算能⼒

### 故障转移

- 3个节点共同组成。包含了1个索引，索引设置了3个`Primary Shard`和 1 个 `Replica`
- 节点1是 `Master` 节点，节点意外出现故障。集群重新选举 `Master` 节点
- `Node 3` 上的 R0 提升成 P0，集群变⻩
- R0 和 R1 重新分配，集群变绿

![three_node](https://image.cjyong.com/problem_cluster.png)

集群状态信息:

- `GET /_cluster/healty`
- 绿色: 健康, 所有主分片和副本分片都可用.
- 黄色: 亚健康, 所有主分片可用, 副本分片不可用.
- 红色: 不健康, 部分主分片不可用.

## 文档分布式存储

文档存储在分片上:

- ⽂档会存储在具体的某个主分⽚和副本分片上:例如, 文档1会存储在P0和R0分片上.
- ⽂档到分片的映射算法

  - 确保⽂档能均匀分布在所⽤分片上，充分利用硬件资源，避免部分机器空闲，部分机器繁忙
  - 潜在的算法

    - 随机 / Round Robin。当查询⽂档1，分⽚数很多，需要多次查询才可能查到文档1(性能不好)
    - 维护⽂档到分片的映射关系，当⽂档数据量⼤大的时候，维护成本⾼
    - 实时计算，通过文档 1，⾃动算出，需要去那个分⽚上获取⽂档

ES的文档路由算法: `shard = hash(_routing) % number_of_primary_shards`:

- Hash算法保证文档可以均匀的分布到各个分片中
- 默认的_routing值是文档的ID
- 可以自行制定`routing`数值, 如相同国家的商品, 都分配到指定的 shard.(`PUT goods/_doc/100?routing=china`).
- 设置 `Index Settings` 后， 主分片数不能随意修改的根本原因.

**更新文档**:

![update_doc](https://image.cjyong.com/es_update_doc.png)

**删除文档**:

![update_doc](https://image.cjyong.com/es_delete_doc.png)

## 分⽚及其生命周期

什么是`ES`的分⽚: ES 中最⼩的工作单元 / 是一个 `Lucene` 的 `Index`.

**倒排索引不可变性**:

- 倒排索引采用 `Immutable Design`，⼀旦生成，不可更改.
- 不可变性，带来了的好处如下:

  - 无需考虑并发写⽂件的问题，避免了锁机制带来的性能问题
  - 一旦读入内核的⽂件系统缓存，便留在哪里。只要⽂件系统存有⾜够的空间，⼤部分请求就会直接请求内存，不会命中磁盘，提升了了很⼤的性能
  - 缓存容易生成和维护 / 数据可以被压缩

- 不可变更更性，带来了的挑战:如果需要让⼀个新的⽂档可以被搜索，需要重建整个索引.

**Lucene Index**:

- 在 `Lucene` 中，单个倒排索引⽂件被称为`Segment`。`Segment` 是自包含的，不可变更的。 多个 `Segments` 汇总在一起，称为`Lucene`的`Index`，其对应的就是 `ES` 中的 `Shard`.

- 当有新⽂档写⼊时，会⽣成新 `Segment`，查询时会同时查询所有`Segments`，并且对结果汇总。Lucene 中有一个⽂文件，⽤来记录所有 `Segments` 信息，叫做 `Commit Point`.

- 删除的⽂档信息，保存在".del"⽂件中. 用于搜索结果的过滤.

![segment](https://image.cjyong.com/es_segment.png)

**Refresh**:

- 将`Index buffer`写⼊`Segment`的过程叫`Refresh`。 `Refresh` 不执行 `fsync` 操作

- `Refresh`频率: 默认1秒发⽣⼀次，可通过`index.refresh_interval` 配置。`Refresh` 后， 数据就可以被搜索到了。这也是为什么 Elasticsearch 被称为`近实时搜索`.

- 如果系统有大量的数据写入，那就会产⽣很多的`Segment`

- `Index Buffer`被占满时，会触发`Refresh`，默认值是 JVM 的 `10%`.

![segment](https://image.cjyong.com/es_refresh.png)

**Transaction Log**:

- `Segment` 写⼊磁盘的过程相对耗时，借助文件系统缓存，`Refresh`时，先将 `Segment` 写⼊入缓存以开放查询
- 为了保证数据不会丢失。所以在 `Index` 文档时，同时写`Transaction Log`，⾼版本开始，`Transaction Log` 默认落盘。每个分片有一个`Transaction Log`.
- 在 ES `Refresh` 时，`Index Buffer` 被清空， `Transaction log` 不会清空.

![segment](https://image.cjyong.com/es_tractionlog.png)

**Flush**:

`ES Flush` & `Lucene Commit`

- 调用`Refresh，Index Buffer` 清空并且 `Refresh`
- 调用`fsync`，将缓存中的`Segments` 写入磁盘
- 清空(删除)`Transaction Log`
- 默认 30 分钟调⽤用⼀一次
- Transaction Log 满 (默认 512 MB)

![segment](https://image.cjyong.com/es_flush.png)

**Merge**:

- `Segment`很多，需要被定期被合并

  - 减少 `Segments` / 删除已经删除的⽂档

- ES 和 Lucene 会⾃动进行 `Merge` 操作

  - 手动执行: `POST my_index/_forcemerge`

## 剖析分布式查询及相关性算分

- Elasticsearch 的搜索，会分两阶段进行

  - 第⼀阶段 - Query
  - 第⼆阶段 - Fetch

- Query-then-Fetch

![query-then-fetch](https://image.cjyong.com/es_query_and_fetch.png)

**Query阶段**:

- ⽤户发出搜索请求到 ES 节点。节点收到请求后， 会以 `Coordinating` 节点的身份，在6个主副分⽚片中随机选择3个分片，发送查询请求
- 被选中的分⽚执⾏查询，进行排序。然后，每个分⽚都会返回 `From + Size` 个排序后的⽂档Id和排序值给 `Coordinating` 节点

**Fetch阶段**:

- `Coordinating Node` 会将 `Query` 阶段，从每个分⽚获取的排序后的⽂档Id列表，重新进行排序。 选取 `From 到 From + Size` 个⽂文档的Id
- 以 `multi get` 请求的⽅式，到相应的分⽚获取详细的⽂档数据

**Query Then Fetch 潜在的问题**:

- 性能问题

  - 每个分片上需要查的⽂档个数 = from + size
  - 最终协调节点需要处理:number_of_shard * ( from+size )
  - 深度分⻚(from特别大的时候, 需要处理的数据巨增)

- 相关性算分不准

  - 每个分⽚都基于⾃己的分⽚上的数据进⾏相关度计算。这会导致打分偏离的情况，特别是数据量很少时。相关性算分在分片之间是相互独⽴。当文档总数很少的情况下，如果主分片⼤于 1，主分⽚数越多，相关性算分会越不准

**解决算分不准的问题**:

- 数据量不大的时候，可以将主分⽚数设置为 1

  - 当数据量⾜够大时候，只要保证⽂档均匀分散在各个分片上，结果⼀般就不会出现偏差

- 使⽤ `DFS Query Then Fetch`

  - 搜索的URL中指定参数 `_search?search_type=dfs_query_then_fetch`
  - 到每个分片把各分⽚的词频和⽂档频率进⾏搜集，然后完整的进⾏一次相关性算分，耗费更加多的 CPU 和内存，执⾏性能低下，⼀般不建议使⽤

**Example**:

```json
// 新建message索引, 设置分片数为20
DELETE message
PUT message
{
  "settings": {
    "number_of_shards": 20
  }
}

// 获取索引信息
GET message

// 插入三条信息, 设定routing使其分布到不同的分片上
POST message/_doc?routing=1
{
  "content":"good"
}

POST message/_doc?routing=2
{
  "content":"good morning"
}
¡™
POST message/_doc?routing=3
{
  "content":"good morning everyone"
}

// 搜索content中包含good的文档, 发现所有文档的得分一样(打分不准确的情况).
POST message/_search
{
  "explain": true,
  "query": {
    "term": {
      "content": {
        "value": "good"
      }
    }
  }
}

// 使用search_type指定dfs_query_then_fetch, 正确打分
POST message/_search?search_typ=dfs_query_then_fetch
{
  "query": {
    "term": {
      "content": {
        "value": "good"
      }
    }
  }
}
```

## ES之分布式排序

**排序**:

- Elasticsearch 默认采⽤相关性算分对结果进⾏降序排序
- 可以通过设定 `sorting` 参数，⾃行设定排序
- 如果不指定`_score`，算分为 Null
- 组合多个条件
- 优先考虑写在前⾯的排序
- ⽀持对相关性算分进⾏排序

**排序的过程**:

- 排序是针对字段原始内容进行的。 倒排索引⽆法发挥作用
- 需要⽤到正排索引。通过⽂档 Id 和字段快速得到字段原始内容
- Elasticsearch 有两种实现方法

  - Fielddata
  - Doc Values (列式存储，对 Text 类型⽆无效)(对于text字段需要手动开启Fielddata)

属性   | Doc Values       | Field data
:--- | :--------------- | :-------------------------------
何时创建 | 索引时，和倒排索引⼀一起创建   | 搜索时候动态创建
创建位置 | 磁盘⽂文件            | JVM Heap
优点   | 避免⼤大量量内存占⽤用      | 索引速度快，不不占⽤用额外的磁盘空间
缺点   | 降低索引速度，占⽤用额外磁盘空间 | ⽂文档过多时，动态创建开销⼤大，占⽤用 过多 JVM Heap，
缺省值  | ES 2.x 之后        | ES 1.x 及之前

**Fielddata**:

- 对 Text 字段设置 fielddata，⽀持随时修改
- Doc Values 可以在 Mapping 中关闭。但是需要重新索引
- Text 不支持 Doc Values
- 使⽤ docvalue_fields 查看存储的信息

**例子**:

```json
// 指定 order_date进行排序
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}}
  ]
}

// 指定多字段进行排序
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}},
    {"_doc":{"order": "asc"}},
    {"_score":{ "order": "desc"}}
  ]
}

// 对text文本进行排序,会报错
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"customer_full_name": {"order": "desc"}}
  ]
}

// 开启该字段的fielddata
PUT kibana_sample_data_ecommerce/_mapping
{
  "properties": {
    "customer_full_name" : {
          "type" : "text",
          "fielddata": true,
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
  }
}

// 手动关闭doc_values(默认是打开的), 增加索引速度, 减少磁盘空间
PUT test_keyword/_mapping
{
  "properties": {
    "user_name":{
      "type": "keyword",
      "doc_values":false
    }
  }
}

// 新建索引temp_users, 并开启name和desc的fielddata
DELETE temp_users
PUT temp_users
PUT temp_users/_mapping
{
  "properties": {
    "name":{"type": "text","fielddata": true},
    "desc":{"type": "text","fielddata": true}
  }
}
// 插入一条数据
POST temp_users/_doc
{"name":"Jack","desc":"Jack is a good boy!","age":10}

// 打开fielddata 后，查看 docvalue_fields数据
POST  temp_users/_search
{
  "docvalue_fields": [
    "name","desc"
    ]
}

// 查看整型字段的docvalues
POST  temp_users/_search
{
  "docvalue_fields": [
    "age"
    ]
}
```

## 分页与遍历

## 分页

分布式系统中深度分页的问题:

![from_and_size](https://image.cjyong.com/es_from_size.png)

### 使用Search After避免问题

- 避免深度分⻚的性能问题，可以实时获取下⼀⻚文档信息

  - 不⽀持指定页数(From)
  - 只能往下翻

- 第⼀步搜索需要指定 sort，并且保证值是唯一的 (可以通过加⼊ _id 保证唯⼀性)

- 然后使用上一次，最后⼀个⽂档的 sort 值进行查询

原理:

![search_after](https://image.cjyong.com/es_search_after.png)

### Scroll API

- 创建⼀个快照，有新的数据写⼊以后，⽆法被查到
- 每次查询后，输⼊上⼀次的 `Scroll Id`

各类分页API分类:

- Regular

  - 需要实时获取顶部的部分⽂档。例如查询最新的订单

- Scroll

  - 需要全部⽂文档，例如导出全部数据

- Pagination

  - From 和 Size
  - 如果需要深度分页，则选⽤Search After

### Example

```json
// 创建用户索引
DELETE users
POST users/_doc
{"name":"user1","age":10}

POST users/_doc
{"name":"user2","age":11}


POST users/_doc
{"name":"user2","age":12}

// 指定sort类型
POST users/_search
{
    "size": 1,
    "query": {
        "match_all": {}
    },
    "sort": [
        {"age": "desc"} ,
        {"_id": "asc"}    
    ]
}

// 使用search_after, 传递上一次查询中返回的sort值
POST users/_search
{
    "size": 1,
    "query": {
        "match_all": {}
    },
    "search_after":
        [
          10,
          "ZQ0vYGsBrR8X3IP75QqX"],
    "sort": [
        {"age": "desc"} ,
        {"_id": "asc"}    
    ]
}

// 设置scroll为5分钟,生成快照
POST /users/_search?scroll=5m
{
    "size": 1,
    "query": {
        "match_all" : {
        }
    }
}

// 传递上一次返回的scroll_id
POST /_search/scroll
{
    "scroll" : "1m",
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAWAWbWdoQXR2d3ZUd2kzSThwVTh4bVE0QQ=="
}
```

## 处理并发读写

`ES 采⽤的是乐观并发控制`:

- ES 中的⽂档是不可变更的。如果你更新⼀个文档，会将该文档标记为删除，同时增加⼀个全新的文档。同时⽂档的 `version` 字段加 1.(已被废弃)
- 内部版本控制

  - If_seq_no + If_primary_term

- 使⽤用外部版本(使⽤其他数据库作为主要数据存储)

  - version + version_type=external

**例子**:

```json
// 插入一条苹果数据, 库存100
DELETE products
PUT products

PUT products/_doc/1
{
  "title":"iphone",
  "count":100
}

// 获取信息
GET products/_doc/1

// result
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "iphone",
    "count" : 100
  }
}

// 使用_seq_no和_primary_term值进行更新
PUT products/_doc/1?if_seq_no=0&if_primary_term=1
{
  "title":"iphone",
  "count":99
}

// 更新完之后重新获取, _seq_no变为了1
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "iphone",
    "count" : 99
  }
}

// 如果还是用旧的更新命令, 会提示报错

//如果实在外部存储, 进行更新, 需要制定version版本
PUT products/_doc/1?version=30000&version_type=external
{
  "title":"iphone",
  "count":100
}

// 更新完毕之后 _version变成了30000
{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 30000,
  "_seq_no" : 2,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "iphone",
    "count" : 87
  }
}
```
