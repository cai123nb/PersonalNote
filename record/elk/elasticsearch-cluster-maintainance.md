# ElasticSearch 生产运维

![illustrate](https://image.cjyong.com/illustrated-screenshot-kibana-7dot8-730x555.png)

本章主要关注ES集群生产环境运维管理: 包括监控集群的健康度, 诊断集群潜在问题, 集群压力测试, 索引的生命周期管理等等.

**笔记主要来自购买的极客时间课程`阮一鸣 - ElasticSearch核心技术和实战`, [原课程链接](https://time.geekbang.org/course/intro/100030501), 主要用于个人温习和查阅, 如有侵权, 请及时邮件联系**

## 生产环境常用配置和上线清单

`Development vs. Production Mode`:

从 ES 5 开始，支持 `Development` 和 `Production` 两种运行模式, 依据的标准为`http.host`和`Transport.bind_host`来进行区分:

`Development`:

```yml
http.port: 9200
http.host: localhost
Transfort.tcp.port: 9300
Transport.bind_host: localhost
```

`Production`:

```yml
http.port: 9200
http.host: 192.168.1.32
Transfort.tcp.port: 9300
Transport.bind_host: 192.168.1.32
```

如果集群是以生产模式启动的话, 就会进行一系列的`Bootstrap Checks`, 必须通过这些检测, 才可以正确启动, 否则启动失败. [Bootstrap Checks列表](https://www.elastic.co/guide/en/elasticsearch/reference/master/bootstrap-checks.html). 具体的设定请查阅, 连接. `check`主要分为两类:

**JVM Checks**:

- Heap size
- Disable swapping
- Not use serial collector
- OnError and OnOOMError q G1GC
- Server JVM

**Linux Checks**:

- Maximum map count
- Maximum size virtual memory
- Maximum number of threads
- File descriptor
- System call filter

具体参照列表链接.

**JVM设定**:

- 从ES6开始，只支持64位的JVM

  - 配置 config / jvm.options

- 避免修改默认配置

  - 将 内存 Xms 和 Xmx 设置成一样，避免 heap resize 时引发停顿
  - Xmx 设置不要超过物理内存的 50%;单个节点上，最大内存建议不要超过 32 G 内存

    - `https://www.elastic.co/blog/a-heap-of-trouble`

- 生产环境，JVM 必须使用 Server 模式

- 关闭 JVM Swapping

**集群的 API 设定**:

按照优先级从小到大排序为(低优先级会被高优先级覆盖):

- Config file settings
- Command-line settings
- Persistent Settings
- Transient Settings

静态配置文件尽量简洁: 按照文档设置所有相关系统参数: `elasticsearch.yml`配置文件中尽量只写必备参数.

动态设定: 其他的设置项可以通过 API 动态进行设定。 动态设定分`transient`和`persistent`两种, 都会覆盖`elasticsearch.yml` 中的设置:

- Transient 在集群重启后会丢失
- Persistent 在集群中重启后不会丢失

示例:

```shell
PUT /_cluster/settings
{
  "persistent" : {
    "indices.recovery.max_bytes_per_sec" : "50mb"
    ...
  },
  "transient" : {
    ...
  },
}
```

**系统设置**:

参照[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/system-config.html)进行正确设置. 主要包括: `Disable Swapping` ， `Increase file descriptor`，`虚拟内存`，`number of thread`等等.

**最佳实践:网络**:

- 单个集群不要跨数据中心进行部署(不要使用 WAN)
- 节点之间的 hops 越少越好
- 如果有多块网卡，最好将 transport 和 http 绑定到不同的网卡，并设置不同的防火墙 Rules
- 按需为 Coordinating Node 或 Ingest Node 配置负载均衡

**最佳实践: 内存设定计算实例**:

- 内存大小要根据 Node 需要存储的数据来进行估算

  - 搜索类的比例建议: 1:16
  - 日志类: 1:48 - 1:96 之间

- 总数据量1T，设置一个副本 = 2T总数据量(假设最大内存31G)

  - 如果搜索类的项目，每个节点 31 *16 = 496 G，加上预留空间。所以每个节点最多 400 G 数据，至少需 要 5 个数据节点
  - 如果是日志类项目，每个节点 31*50 = 1550 GB，2 个数据节点 即可

**最佳实践:存储**:

- 推荐使用 SSD，使用本地存储(Local Disk)。避免使用 SAN NFS / AWS / Azure filesystem
- 可以在本地指定多个 "path.data"，以支持使用多块磁盘
- ES 本身提供了很好的 HA 机制;无需使用 RAID 1/5/10
- 可以在 Warm 节点上使用 Spinning Disk，但是需要关闭 Concurrent Merges

  - Index.merge.scheduler.max_thread_count: 1

- Trim 你的 SSD

  - <https://www.elastic.co/blog/is-your-elasticsearch-trimmed>

**最佳实践:服务器硬件**:

- 建议使用中等配置的机器，不建议使用过于强劲的硬件配置

  - Medium machine over large machine

- 不建议在一台服务器上运行多个节点

**集群设置:Throttles 限流**:

- 为 Relocation 和 Recovery 设置限流，避免过多任务对集群产生性能影响
- Recovery

  - Cluster.routing.allocation.node_concurrent_recoveries: 2

- Relocation

  - Cluster.routing.allocation.cluster_concurrent_rebalance: 2

**集群设置:关闭 Dynamic Indexes**:

- 可以考虑关闭动态索引创建的功能:

```java
PUT /_cluster/settings
{
  "persistent" : {
    "action.auto_create_index" : false
  }
}
```

- 或者通过模版设置白名单

```java
PUT /_cluster/settings
{
  "persistent" : {
    "action.auto_create_index" : "logstash-*, .kibana*"
  }
}
```

**集群安全设定**:

- 为 Elasticsearch 和 Kibana 配置安全功能

  - 打开 Authentication & Authorization
  - 实现索引和和字段级的安全控制

- 节点间通信加密

- Enable HTTPS

- Audit logs

## 集群写性能优化

**提高写入性能的方法**:

- 写性能优化的目标:增大写吞吐量(Events Per Second)，越高越好
- 客户端:多线程，批量写

  - 可以通过性能测试，确定最佳文档数量
  - 多线程: 需要观察是否有 HTTP 429 返回，实现 Retry 以及线程数量的自动调节

- 服务器端:单个性能问题，往往是多个因素造成的。需要先分解问题，在单个节点上进行调整并且结合测试，尽可能压榨硬件资源，以达到最高吞吐量

  - 使用更好的硬件。观察 CPU / IO Block
  - 线程切换 / 堆栈状况

**服务器端优化写入性能的一些手段**:

- 降低IO操作

  - 使用 ES 自动生成的文档 Id / 一些相关的 ES 配置，如 Refresh Interval

- 降低 CPU 和存储开销

  - 减少不必要分词 / 避免不需要的 doc_values /文档的字段尽量保证相同的顺序，可以提高文档的压缩率

- 尽可能做到写入和分片的均衡负载，实现水平扩展

  - Shard Filtering / Write Load Balancer

- 调整 Bulk 线程池和队列

- ES 的默认设置，已经综合考虑了数据可靠性，搜索的实时性质，写入速度，一般不要盲目修改

- 一切优化，都要基于高质量的数据建模

**关闭无关的功能**:

- 只需要聚合不需要搜索， Index 设置成 false
- 不需要算分， Norms 设置成 false
- 不要对字符串使用默认的 dynamic mapping。字段数量过多，会对性能产生比较大的影响
- Index_options 控制在创建倒排索引时，哪些内容会被添加到倒排索引中。优化这些设置，一定程度可以节约CPU
- 关闭 _source，减少 IO 操作;(适合指标型数据)

**针对性能的取舍**:

如果需要追求极致的写入速度，可以牺牲数据可靠性及搜索实时性以换取性能

- 牺牲可靠性: 将副本分片设置为 0，写入完毕再调整回去
- 牺牲搜索实时性: 增加 `Refresh Interval` 的时间
- 牺牲可靠性: 修改 `Translog` 的配置

**数据写入的过程优化**:

**Refresh**:

- 将文档先保存在 Index buffer 中， 以 `refresh_interval` 为间隔时间，定期清空 `buffer`, 生成 `segment`, 借助文件系统缓存的特性, 先将 `segment` 放在文件系统缓存中，并开放查询，以提升搜索的实时性
- 降低 Refresh 的频率

  - 增加 refresh_interval 的数值。默认为 1s ，如果设置成 -1 ，会禁止自动 refresh

    - 避免过于频繁的 refresh，而生成过多的 segment 文件
    - 但是会降低搜索的实时性

  - 增大静态配置参数 indices.memory.index_buffer_size

    - 默认是 10%， 会导致自动触发 refresh

**Translog**:

- Segment 没有写入磁盘，即便发生了当机，重启后数据也能恢复，默认配置是每次请求都会落盘
- 降低写磁盘的频率，但是会降低容灾能力

  - Index.translog.durability:默认是 request，每个请求都落盘。设置成 async，异步写入
  - Index.translog.sync_interval 设置为 60s，每分钟执行一次
  - Index.translog.flush_threshold_size: 默认 512 mb，可以适当调大。 当 translog 超过该值，会触发 flush

- Flush

  - 删除旧的 `translog` 文件
  - 生成 Segment 并写入磁盘 / 更新 commit point 并写入磁盘。 ES 自动完成，可优化点不多

**分片设定优化**:

- 副本在写入时设为 0，完成后再增加
- 合理设置主分片数，确保均匀分配在所有数据节点上

  - Index.routing.allocation.total_share_per_node: 限定每个索引在每个节点上可分配的主分片数
  - 5 个节点的集群。 索引有 5 个主分片，1 个副本，应该如何设置?

    - (5+5)/5=2
    - 生产环境中要适当调大这个数字，避免有节点下线时，分片无法正常迁移

**Bulk，线程池和队列大小优化:**

- 客户端

  - 单个 bulk 请求体的数据量不要太大，官方建议大约5-15mb
  - 写入端的 bulk 请求超时需要足够长，建议60s以上
  - 写入端尽量将数据轮询打到不同节点。

- 服务器端

  - 索引创建属于计算密集型任务，应该使用固定大小的线程池来配置。来不及处理的放入队列，线程数应该配置成CPU核心数 + 1 ，避免过多的上下文切换
  - 队列大小可以适当增加，不要过大，否则占用的内存会成为 GC 的负担

**Demo**:

```java
DELETE myindex
PUT myindex
{
  "settings": {
    "index": {
      // 30s触发一次refresh
      "refresh_interval": "30s",
      "number_of_shards": "2"
    },
    "routing": {
      "allocation": {
        // 设置每个节点最多存储当前索引的三个分片
        "total_shards_per_node": "3"
      }
    },
    "translog": {
      // translog 异步落盘, 30s一次
      "sync_interval": "30s",
      "durability": "async"
    },
    "number_of_replicas": 0
  },
  "mappings": {
    // 关闭自动创建属性
    "dynamic": false,
    "properties": {}
  }
}
```

## 集群读性能优化

**尽量 Denormalize 数据**:

- `Elasticsearch != 关系型数据库`
- 尽可能 `Denormalize` 数据，从而获取最佳的性能

  - 使用 Nested 类型的数据。查询速度会慢几倍
  - 使用 Parent / Child 关系。查询速度会慢几百倍

**数据建模**:

- 尽量将数据先行计算，然后保存到 Elasticsearch 中。尽量避免查询时的 Script 计算
- 尽量使用 Filter Context，利用缓存机制，减少不必要的算分
- 结合 profile，explain API 分析慢查询的问题，持续优化数据模型

  - 严禁使用 * 开头通配符 Terms 查询

**优化分片**:

- 避免 Over Sharing

  - 一个查询需要访问每一个分片，分片过多，会导致不必要的查询开销

- 结合应用场景，控制单个分片的尺寸

  - Search: 20GB
  - Logging:40GB

- Force-merge Read-only 索引

  - 使用基于时间序列的索引，将只读的索引进行 force merge，减少 segment 数量

**读性能优化总结**:

- 影响查询性能的一些因素

  - 数据模型和索引配置是否优化
  - 数据规模是否过大，通过 Filter 减少不必要的数据计算
  - 查询语句是否优化

## 诊断集群的潜在问题

**集群运维所面临的挑战**:

- 用户集群数量多，业务场景差异大
- 使用与配置不当，优化不够 -如何让用户更加高效和正确的使用 ES

  - 如何让用户更全面的了解自己的集群的使用状况

- 发现问题滞后，需要防患于未然

  - 需要"有迹可循"，做到"有则改之，无则加勉"
  - Elastic 有提供 Support Diagnostics Tool - `https://github.com/elastic/support-diagnostics`

**为什么要诊断集群的潜在问题**:

- 防患于未然，避免集群奔溃

  - Master 节点 / 数据节点当机 – 负载过高，导致节点失联
  - 副本丢失，导致数据可靠性受损
  - 集群压力过大，数据写入失败

- 提升集群性能

  - 数据节点负载不均衡(避免单节点瓶颈) / 优化分片，segment
  - 规范操作方式(利用别名 / 避免 Dynamic Mapping 引发过多字段，对索引的合理性进行管控)

推荐参考阿里的`EYOU智能运维工具`, 实现多维度检测，构建自己的诊断工具:

![多维度检测](https://image.cjyong.com/ac7f23beb96043938582ff43b08fbe16.png)

## 解决集群 Yellow 与 Red 的问题

**集群健康度:**

- 分片健康

  - 红:至少有一个主分片没有分配
  - 黄:至少有一个副本没有分配
  - 绿:主副本分片全部正常分配

- 索引健康: 最差的分片的状态

- 集群健康: 最差的索引的状态

**Health**相关API:

API                               | 说明
:-------------------------------- | :---------------------
GET _cluster/health               | 集群的状态(检查节点数量)
GET _cluster/health?level=indices | 所有索引的健康状 态 (查看有问题的 索引)
GET _cluster/health/my_index      | 单个索引的健康状态(查看具体的索引)
GET _cluster/health?level=shards  | 分片级的索引
GET _cluster/allocation/explain   | 返回第一个未分配 Shard 的原因

**分片没有被分配的一些原因**:

- INDEX_CREATE: 创建索引导致。在索引的全部分片分配完成之前，会有短暂的`Red`, 不一定代表有问题
- CLUSTER_RECOVER: 集群重启阶段，会有这个问题
- INDEX_REOPEN: Open 一个之前 Close 的索引
- DANGLING_INDEX_IMPORTED: 一个节点离开集群期间，有索引被删除。 这个节点重新返回时，会导致 `Dangling` 的问题
- [参阅官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/cat-shards.html)

**常见问题与解决方法**:

- 集群变红，需要检查是否有节点离线。如果有，通常通过重启离线的节点可以解决问题
- 由于配置导致的问题，需要修复相关的配置(例如错误的 box_type，错误的副本数)

  - 如果是测试的索引，可以直接删除

- 因为磁盘空间限制，分片规则(Shard Filtering)引发的，需要调整规则或者增加节点

- 对于节点返回集群，导致的 dangling 变红，可直接删除 dangling 索引

### 案例 1

- 症状:集群变红
- 分析:通过 Allocation Explain API 发现 创建索引失败，因为无法找到标记了相应 `box type` 的节点(如: hot/warm)
- 解决:删除索引，集群变绿。重新创建索引，并且指定正确的 `routing box type`，索引创建成功。集群保持绿色状态

### 案例 2

- 症状:集群变黄
- 分析:通过 Allocation Explain API 发现无法在相同的节点上创建副本
- 解决:将索引的副本数设置为0，或者通过增加节点解决

## 集群压力测试

**压力测试的目的**:

- 容量规划 / 性能优化 / 版本间性能比较 / 性能问题诊断
- 确定系统稳定性，考察系统功能极限和隐患

**压力测试的方法与步骤**:

- 测试计划(确定测试场景和测试数据集)
- 脚本开发
- 测试环境搭建(不同的软硬件配置) & 运行测试
- 分析比较结果

**测试目标 & 测试数据**:

- 测试目标

  - 测试集群的读写性能 / 做集群容量规划
  - 对 ES 配置参数进行修改，评估优化效果
  - 修改 Mapping 和 Setting，对数据建模进行优化，并测试评估性能改进 ○ 测试 ES 新版本，结合实际场景和老版本进行比较，评估是否进行升级

- 测试数据

  - 数据量/数据分布

**测试方式**:

ES 本身提供了 REST API，所以，可以通过很多传统的性能测试工具:

- Load Runner (商业软件，支持录制+重放 + DSL )
- JMeter ( Apache 开源 ，Record & Play)
- Gatling (开源，支持写 Scala 代码 + DSL)

专门为 Elasticsearch 设计的工具:

- ES Pref & Elasticsearch-stress-test
- Elastic Rally

**ES Rally 简介**

- Elastic 官方开源，基于 Python 3 的压力测试工具

  - `https://github.com/elastic/rally`
  - 性能测试结果比较: `https://elasticsearch-benchmarks.elastic.co`

- 功能介绍

  - 自动创建，配置，运行测试，并且销毁 ES 集群
  - 支持不同的测试数据的比较，也支持将数据导入 ES 集群，进行二次分析
  - 支持测试时指标数据的搜集，方便对测试结果进行深度的分析

`Rally 的安装以及入门`:

- 安装运行

  - Python3.4+和pip3 /JDK8 /git1.9+
  - 运行 `pip3 install esrally`
  - 运行 `esrally configure`

- 运行

  - 运行 `esrally –distribution-version=7.1.0`
  - 运行 1000 条测试数据: `esrally –distribution-version=7.1.0 --test-mode`

`Rally 基本概念讲解`:

- Tournament – 定义测试目标，由多个 race 组成

  - Esrally list races

- Track – 赛道:测试数据和测试场景与策略

  - <https://github.com/elastic/rally-> tracks
  - esrally list tracks

- Car– 执行测试方案

  - 不同配置的 es 实例

- Award – 测试结果和报告

**实例:比较不同的版本的性能**:

- 测试

  - esrally race --distribution-version=6.0.0 --track=nyc_taxis --challenge=append-no- conflicts --user-tag="version:6.0.0"
  - esrally race --distribution-version=7.1.0 --track=nyc_taxis --challenge=append-no- conflicts --user-tag="version:7.1.0"

- 比较结果

  - esrally list races
  - esrally compare --baseline=[6.0.0 race] --contender=[7.1.0 race]

**实例:比较不同 Mapping 的性能**:

- 测试

  - esrally race --distribution-version=7.1.0 --track=nyc_taxis --challenge=append-no- conflicts --user-tag="enableSource:true" --include-tasks="type:index"
  - 修改: `benchmarks/tracks/default/nyc_taxis/mappings.json`, 修改 `_source.enabled` 为 false
  - esrally race --distribution-version=7.1.0 --track=nyc_taxis --challenge=append-no- conflicts --user-tag="enableSource:false" --include-tasks="type:index

- 比较

  - esrally compare --baseline=[enableAll race] --contender=[disableAll race]

**实例:测试现有集群的性能**:

- esrally race --pipeline=benchmark-only --target-hosts=127.0.0.1:9200 --track=geonames -- challenge=append-no-conflicts

## 段合并优化及注意事项

在 `Lucene` 中，单个倒排索引文件被称为 `Segment`。`Segment`是自包含的，不可变更的。 多个 Segments 汇总在一起，称为 `Lucene` 的 `Index`，其对应的就是 `ES` 中的 `Shard`.

- 当有新文档写入时，并且执行 `Refresh`，就会 会生成一个新 `Segment`。 `Lucene` 中有一个文件，用来记录所有 `Segments` 信息，叫做 `Commit Point`。查询时会同时查询所有`Segments`，并且对结果汇总。
- 删除的文档信息，保存在".del"文件中，查询后会进行过滤。
- `Segment` 会定期 `Merge`,合并成一个，同时删除已删除文档

**Merge 优化**:

- ES 和 Lucene 会自动进行 Merge 操作
- Merge 操作相对比较重，需要优化，降低对系统的影响
- 优化点一: 降低分段产生的数量/频率

  - 可以将 Refresh Interval 调整到分钟级别 / indices.memory.index_buffer_size (默认是 10%)
  - 尽量避免文档的更新操作

- 优化点二: 降低最大分段大小，避免较大的分段继续参与 Merge，节省系统资源。(最终会有多个分段)

  - Index.merge.policy.segments_per_tier，默认为 10， 越小需要越多的合并操作
  - Index.merge.policy.max_merged_segment, 默认 5 GB， 超出此大小以后，就不再参与后续的合并操作

**Force Merge**:

- 当 Index 不再有写入操作的时候，建议对其进行 force merge

  - 提升查询速度 / 减少内存开销

- 最终分成几个 segments 比较合适?

  - 越少越好，最好可以 force merge 成 1 个，但是，Force Merge 会占用大量的网络，IO 和 CPU
  - 如果不能在业务高峰期之前做完，就需要考虑增大最终的分段数

    - Shard 的大小 / Index.merge.policy.max_merged_segment 的大小

## 缓存及使用 Circuit Breaker 限制内存使用

Elasticsearch 的缓存主要分成三大类

- Node Query Cache (Filter Context): 每一个节点有一个 Node Query 缓存

  - 由该节点的所有 Shard 共享，只缓存 Filter Context 相关内容
  - Cache 采用 LRU 算法
  - 静态配置，需要设置在每个 Data Node 上

    - Node Level - indices.queries.cache.size: "10%"
    - Index Level: index.queries.cache.enabled: true

- Shard Query Cache (Cache Query的结果): 缓存每个分片上的查询结果

  - 只会缓存设置了 size=0 的查询对应的结果。不会缓存hits。但是会缓存 Aggregations 和 Suggestions
  - Cache Key: LRU 算法，将整个 JSON 查询串作为 Key，与 JSON 对 象的顺序相关
  - 静态配置: 数据节点中 indices.requests.cache.size: "1%"

- Fielddata Cache

  - 除了 Text 类型，默认都采用 doc_values。节约了内存

    - Aggregation 的 Global ordinals 也保存在 Fielddata cache 中

  - Text 类型的字段需要打开 Fileddata 才能对其进行聚合和排序

    - Text 经过分词，排序和聚合效果不佳，建议不要轻易使用

  - 配置: 可以控制 Indices.fielddata.cache.size, 避免产生 GC (默认无限制)

**缓存失效**:

- Node Query Cache: 保存的是 Segment 级缓存命中的结果。Segment 被合并后，缓存会失效
- Shard Request Cache: 分片 Refresh 时候，Shard Request Cache 会失效。如果 Shard 对应的数据频繁发生变化，该缓存的效率会很差
- Fielddata Cache: Segment 被合并后，会失效

**管理内存的重要性**:

- Elasticsearch 高效运维依赖于内存的合理分配

  - 可用内存一半分配给 JVM，一半留给操作系统，缓存索引文件

- 内存问题，引发的问题

  - 长时间 GC，影响节点，导致集群响应缓慢
  - OOM， 导致丢节点

**诊断内存状况接口列表**:

- `GET _cat/nodes?v`
- `GET _nodes/stats/indices?pretty`
- `GET _cat/nodes?v&h=name,queryCacheMemory,queryCacheEvictions,requestCacheMemory,reques tCacheHitCount,request_cache.miss_count`
- `GET _cat/nodes?h=name,port,segments.memory,segments.index_writer_memory,fielddata.memo ry_size,query_cache.memory_size,request_cache.memory_size&v`

### 内存问题案例1

- Segments 个数过多，导致 full GC
- 现象:集群整体响应缓慢，也没有特别多的数据读写。但是发现节点在持续进行 Full GC
- 分析:查看 Elasticsearch 的内存使用，发现 segments.memory 占用很大空间
- 解决:通过 force merge，把 segments 合并成一个。
- 建议:对于不在写入和更新的索引，可以将其设置成只读。同时，进行 `force merge` 操作。如果问题依然存在，则需要考虑扩容。此外，对索引进行 `force merge`，还可以减少对 `global_ordinals` 数据结构的构建，减少对`fielddata cache` 的开销

### 内存问题案例2

- Field data cache 过大，导致 full GC
- 现象:集群整体响应缓慢，也没有特别多的数据读写。但是发现节点在持续进行 Full GC
- 分析:查看 Elasticsearch 的内存使用，发现 fielddata.memory.size 占用很大空间。同时, 数据不存在写入和更新，也执行过 segments merge。
- 解决:将 indices.fielddata.cache.size 设小，重启节点，堆内存恢复正常
- 建议:Field data cache 的构建比较重，Elasticsearch 不会主动释放，所以这个值应该设置的保守一些。如果业务上确实有所需要，可以通过增加节点，扩容解决

### 内存问题案例3

- 复杂的嵌套聚合，导致集群 full GC
- 现象:节点响应缓慢，持续进行 Full GC
- 分析:导出 Dump 分析。发现内存中有大量 bucket 对象，查看 日志，发现复杂的嵌套聚合
- 解决:优化聚合
- 建议:在大量数据集上进行嵌套聚合查询，需要很大的堆内存来完成。如果业务场景确实需要。则需要增加硬件进行扩展。同时，为了避免这类查询影响整个集群，需要设置 `Circuit Breaker` 和 `search.max_buckets` 的数值

### Circuit Breaker

ES内置多种断路器，避免不合理操作引发的`OOM`，每个断路器可以指定内存使用的限制:

- Parent circuit breaker:设置所有的熔断器可以使用的内存的总量
- Fielddata circuit breaker:加载 fielddata 所需要的内存
- Request circuit breaker:防止每个请求级数据结构超过一定的内存(例如聚合计算的内存) ○ In flight circuit breaker:Request中的断路器
- Accounting request circuit breaker:请求结束后不能释放的对象所占用的内存

**Circuit Breaker 统计信息**:

- GET /_nodes/stats/breaker?

  - Tripped 大于 0， 说明有过熔断
  - Limit size 与 estimated size 约接近，越可能引发熔断

- 千万不要触发了熔断，就盲目调大参数，有可能会导致集群出问题，也不因该盲目调小，需要进行评估

- 建议将集群升级到 7.x，更好的 Circuit Breaker 实现机制

  - 增加了 indices.breaker.total.use_real_memory 配置项，可以更加精准的分析内存状况，避免 OOM

## 监控 Elasticsearch 集群

**Elasticsearch Stats 相关的 API**:

- Node Stats: `_nodes/stats`
- Cluster Stats: `_cluster/stats`
- Index Stats: `index_name/_stats`

**Elasticsearch Task API**:

- 查看 Task 相关的 API

  - Pending Cluster Tasks API: `GET _cluster/pending_tasks`
  - Task Management API : `GET _tasks`(可以用来 Cancel 一个 Task)

- 监控 Thread Pools

  - `GET _nodes/thread_pool`
  - `GET _nodes/stats/thread_pool ○ GET _cat/thread_pool?v`
  - `GET _nodes/hot_threads`

**The Index & Query Slow Log**:

- 支持将分片上， Search 和 Fetch 阶段的慢 查询写入文件
- 支持为 Query 和 Fetch 分别定义阈值
- 索引级的动态设置，可以按需设置，或者通过 Index Template 统一设定
- Slog log 文件通过 log4j2.properties 配置

Demo:

```java
DELETE my_index
//"0" logs all queries
PUT my_index/
{
  "settings": {
    "index.search.slowlog.threshold": {
      "query.warn": "10s",
      "query.info": "3s",
      "query.debug": "2s",
      "query.trace": "0s",
      "fetch.warn": "1s",
      "fetch.info": "600ms",
      "fetch.debug": "400ms",
      "fetch.trace": "0s"
    }
  }
}
```

**如何创建监控 Dashboard**:

- 开发 Elasticsearch plugin，通过读取相关的监控 API，将数据发送到 ES，或者 `TSDB`
- 使用 `Metricbeats` 搜集相关指标
- 使用 `Kibana` 或 `Grafana` 创建 Dashboard
- 可以开发 Elasticsearch Exporter，通过 Prometheus 监控 Elasticsearch 集群

## 一些运维相关的建议

**集群的生命周期管理**:

- 预上线

  - 评估用户的需求及使用场景 / 数据建模 / 容量规划 / 选择合适的部署架构 / 性能测试

- 上线

  - 监控流量 / 定期检查潜在问题 (防患于未然，发现错误的使用方式，及时增加机器)
  - 对索引进行优化(Index Lifecycle Management)，检测是否存在不均衡而导致有部分节点过热 ○ 定期数据备份 / 滚动升级

- 下架前监控流量，实现 Stage Decommission

**部署的建议**:

- 根据实际场景，选择合适的部署方式，选择合理的硬件配置 ○ 搜索类

  - 日志/指标

- 部署要考虑，反亲和性(Anti-Affinity)

  - 尽量将机器分散在不同的机架。例如，3 台 Master 节点必须分散在不同的机架上
  - 善用 Shard Filtering 进行配置

**使用要遵循一定的规范**:

- Mapping

  - 生产环境中索引应考虑禁止 Dynamic Index Mapping，避免过多字段导致 Cluster State 占用过多
  - 禁止索引自动创建的功能，创建时必须提供 Mapping 或通过 Index Template 进行设定

- 设置 Slowlogs，发现一些性能不好，甚至是错误的使用 Pattern

  - 例如:错误的将网址映射成 keyword，然后用通配符查询。应该使用 Text，结合 URL 分词器
  - 严禁一切 "*" 开头的通配符查询

**对重要的数据进行备份**:

- 集群备份
- `https://www.elastic.co/guide/en/elasticsearch/reference/7.1/modules-snapshots.html`

**定期更新到新版本**:

- ES 在新版本中会持续对性能作出优化;提供更多的新功能

  - Circuit breaker 实现的改进

- 修复一些已知的 bug 和安全隐患

- Elasticsearch 的版本格式是: X.Y.Z

- Elasticsearch 可以使用上一个主版本的索引

  - 7.x 可以使用 6.x / 7.x 不支持使用 5.x
  - 5.x 可以使用 2.x(2是直接到了5, 没有3,4)

- Rolling Upgrade v.s Full Cluster Restart

  - Rolling Upgrade: [没有 Downtime](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/rolling-upgrades.html).
  - Full Cluster Restart

    - 集群在更新期间不可用, 升级更快
    - 停止索引数据，同时备份集群
    - Disable Shard Allocation (Persistent)
    - 执行 Synced Flush
    - 关闭并更新所有节点
    - 先运行所有 Master 节点 / 再运行其他节点
    - 等集群变黄后打开 Shard Allocation

**运维常见的命令**:

**移动分片**: 从一个节点移动分片到另外一个节点, 如当一个数据节点上有过多 Hot Shards; 可以通过手动分配分片到特定的节点解决.

```java
POST _cluster/reroute
{
  "commands": [
    {
      "move": {
        "index": "index_name",
        "shard": 0,
        "from_node": "node_name_1",
        "to_node": "node_name_2"
      }
    }
  ]
}
```

**从集群中移除一个节点**: 如当你想移除一个节点，或者对一个机器进行维护。同时你又不希望导致集群的颜色变黄或者变红.

```java
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._ip":"the_IP_of_your_node"
  }
}
```

**控制 Allocation 和 Recovery**: 控制 Allocation 和 Recovery 的速率.

```java
// change the number of moving shards to balance the cluster
PUT /_cluster/settings
{
  "transient": {"cluster.routing.allocation.cluster_concurrent_rebalance":2}
}

// change the number of shards being recovered simultanceously per node
PUT _cluster/settings
{
  "transient": {"cluster.routing.allocation.node_concurrent_recoveries":5}
}

// Change the recovery speed
PUT /_cluster/settings
{
  "transient": {"indices.recovery.max_bytes_per_sec": "80mb"}
}

// Change the number of concurrent streams for a recovery on a single node
PUT _cluster/settings
{
  "transient": {"indices.recovery.concurrent_streams":6}
}
```

**Synced Flush**: 如需要重启一个节点, 通过 synced flush，可以在索引上放置一个 sync ID。这样可以提供这些分片的 Recovery 的时间.

```java
POST _flush/synced
```

**清空节点上的缓存**: 如节点上出现了高内存占用。可以执行清除缓存的操作。这个操作会影响集群的性能， 但是会避免你的集群出现 OOM 的问题.

```java
POST _cache/clear
```

**控制搜索的队列**: 当搜索的响应时间过长，看到有"reject" 指标的增加，都可以适当增加该数值.

```java
PUT _cluster/settings
{
  "transient": {
    "threadpool.search.queue_size":2000
  }
}
```

**设置 Circuit Breaker**: 设置各类 Circuit Breaker。避免 OOM 的发生。

```java
PUT _cluster/settings
{
  "persistent": {
    "indices.breaker.total.limit":"40%"
  }
}
```

## 使用 Shrink 与 Rollover API 管理索引

索引管理 API:

- `Open / Close Index`: 索引关闭后无法进行读写，但是索引数据不会被删除
- `Shrink Index`: 可以将索引的主分片数收缩到较小的值
- `Split Index`: 可以扩大主分片个数
- `Rollover Index`: 类似 Log4J 记录日志的方式，索引尺寸或者时间超过一定值后，创建新的.
- `Rollup Index`: 对数据进行处理后，重新写入，减少数据量

### Open / Close Index API

- 索引关闭后，对集群的相关开销基本降低为0
- 但是无法被读取和搜索
- 当需要的时候，可以重新打开

`Demo`:

```java
// 打开关闭索引
DELETE test
// 查看索引是否存在
HEAD test
// 插入文档
PUT test/_doc/1
{
  "key":"value"
}
// 关闭索引
POST /test/_close
// 索引存在
HEAD test
// 无法查询
POST test/_count
// 打开索引
POST /test/_open
// 可以搜索
POST test/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Shrink API

ES 5.x 后推出的一个新功能，使用场景:

- 索引保存的数据量比较小，需要重新设定主分片数
- 索引从 Hot 移动到 Warm 后，需要降低主分片数

会使用和源索引相同的配置创建一个新的索引，仅仅降低主分片数:

- 源分片数必须是目标分片数的倍数。如果源分片数是素数，目标分片数只能为 1
- 如果文件系统支持硬链接，会将 Segments 硬连接到目标索引，所以性能好

完成后，可以删除源索引. 注意使用时, 存在以下限制:

- 分片必须只读
- 所有的分片必须在同一个节点上
- 集群健康状态为 Green

**Demo**:

```java
// 在一个 hot-warm-cold的集群上进行测试
GET _cat/nodes
GET _cat/nodeattrs

DELETE my_source_index
DELETE my_target_index

// 初始分片数为4
PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0
 }
}

// 插入一条数据
PUT my_source_index/_doc/1
{
  "key":"value"
}

// 查看当前索引的分片
GET _cat/shards/my_source_index

// 压缩到分片数3，提示报错
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 3,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

// 设置分片数为2, 提示没有设置只读
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

// 设置索引只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}

// 报错，提示必须都在一个节点
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

// 删除索引
DELETE my_source_index
// 设置boxtype为 hot, 确保分片都在 hot节点上
PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0,
   "index.routing.allocation.include.box_type":"hot"
 }
}

// 插入数据
PUT my_source_index/_doc/1
{
  "key":"value"
}

// 查看索引在节点上的分布
GET _cat/shards/my_source_index

// 设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}

// 进行shrink
POST my_source_index/_shrink/my_target_index
{
  "settings": {
    "index.number_of_replicas": 0,
    "index.number_of_shards": 2,
    "index.codec": "best_compression"
  },
  "aliases": {
    "my_search_indices": {}
  }
}

// 查看shrink之后的索引信息
GET _cat/shards/my_target_index

// 当前的索引 my_target_index状态为也是只读
PUT my_target_index/_doc/1
{
  "key":"value"
}
```

### Split API

将索引拆分成新的索引, 需要满足一下条件:

- 目标索引必须不存在
- 目标索引的分片数设置必须大于源索引的数量
- 目标索引的分片数必须是源索引分片的倍数
- 磁盘必须有足够的空间

**Demo**:

```java
DELETE my_source_index
DELETE my_target_index

PUT my_source_index
{
 "settings": {
   "number_of_shards": 4,
   "number_of_replicas": 0
 }
}

PUT my_source_index/_doc/1
{
  "key":"value"
}

GET _cat/shards/my_source_index

// 必须是倍数
POST my_source_index/_split/my_target
{
  "settings": {
    "index.number_of_shards": 10
  }
}

// 必须是只读
POST my_source_index/_split/my_target
{
  "settings": {
    "index.number_of_shards": 8
  }
}

// 设置为只读
PUT /my_source_index/_settings
{
  "settings": {
    "index.blocks.write": true
  }
}

POST my_source_index/_split/my_target_index
{
  "settings": {
    "index.number_of_shards": 8,
    "index.number_of_replicas":0
  }
}

GET _cat/shards/my_target_index

// write block
PUT my_target_index/_doc/1
{
  "key":"value"
}
```

### Rollover API

![roll-over](https://image.cjyong.com/es-roll-over.png)

**Rollover API**:

- 当满足一系列的条件，Rollover API 支持将一个 Alias 指向一个新的索引

  - 存活的时间 / 最大文档数 / 最大的文件尺寸

- 应用场景

  - 当一个索引数据量过大

- 一般需要和 Index Lifecycle Management Policies 结合使用

  - 只有调用 `Rollover API` 时，才会去做相应的检测。ES 并不会自动去监控这些索引.

**Demo**:

```java
// 删除nginx相关的索引
DELETE nginx-logs*

// 不设定 is_write_true
// 名字符合命名规范
PUT /nginx-logs-000001
{
  "aliases": {
    "nginx_logs_write": {}
  }
}

// 多次写入文档
POST nginx_logs_write/_doc
{
  "log":"something"
}

// 设置 rollver 条件, 设置最大文档数: 5
POST /nginx_logs_write/_rollover
{
  "conditions": {
    "max_age":   "1d",
    "max_docs":  5,
    "max_size":  "5gb"
  }
}

// 获取当前索引的文档计数, 这里的计数是新索引的数量
// 如果总的数量是6, rollover之后, 新的索引是 1
GET /nginx_logs_write/_count

// 查看 Alias信息
GET /nginx_logs_write

// 删除
DELETE apache-logs*


// 设置 is_write_index之后, 索引会关联所有的索引
PUT apache-logs1
{
  "aliases": {
    "apache_logs": {
      "is_write_index":true
    }
  }
}
POST apache_logs/_count

POST apache_logs/_doc
{
  "key":"value"
}

// 需要指定 target 的名字
POST /apache_logs/_rollover/apache-logs8xxxx
{
  "conditions": {
    "max_age":   "1d",
    "max_docs":  1,
    "max_size":  "5gb"
  }
}


// 查看 Alias信息
GET /apache_logs
```

## 索引全生命周期管理及工具介绍

**时间序列的索引**:

- 特点

  - 索引中的数据随着时间，持续不断增长

- 按照时间序列划分索引的好处 & 挑战

  - 按照时间进行划分索引，会使得管理更加简单。例如，完整删除一个索引，性能比 delete by query 好
  - 如何进行自动化管理，减少人工操作

    - 从 Hot 移动到 Warm
    - 定期关闭或者删除索引

**索引生命周期常见的阶段**:

`Hot` -> `Warm` -> `Cold` -> `Delete`

- Hot: 索引还存在着大量的读写操作
- Warm: 索引不存在写操作，还有被查询的需要
- Cold: 数据不存在写操作，读操作也不多
- Delete: 索引不再需要，可以被安全删除

**Index Lifecycle Management**:

- Elasticsearch 6.6 推出的新功能

  - 基于 X-Pack Basic License，可免费使用

- ILM概念

  - Policy
  - Phase
  - Action

**Demo**:

- 将ILM刷新时间设定为1秒，默认10分钟
- 设置Hot/Warm/Cold和Delete四个阶段

  - 超过 5 个文档以后 rollover
  - 10 秒后进入 warm
  - 15 秒后进入 Cold
  - 20 秒后删除索引

```java
// 设置ILM的刷新时间为1秒
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval":"1s"
  }
}

// 设置 Policy
PUT /_ilm/policy/log_ilm_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_docs": 5
          }
        }
      },
      "warm": {
        "min_age": "10s",
        "actions": {
          "allocate": {
            "include": {
              "box_type": "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "15s",
        "actions": {
          "allocate": {
            "include": {
              "box_type": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "20s",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

// 设置索引模版
PUT /_template/log_ilm_template
{
  "index_patterns" : [
      "ilm_index-*"
    ],
    "settings" : {
      "index" : {
        "lifecycle" : {
          "name" : "log_ilm_policy",
          "rollover_alias" : "ilm_alias"
        },
        "routing" : {
          "allocation" : {
            "include" : {
              "box_type" : "hot"
            }
          }
        },
        "number_of_shards" : "1",
        "number_of_replicas" : "0"
      }
    },
    "mappings" : { },
    "aliases" : { }
}



// 创建索引
PUT ilm_index-000001
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "index.lifecycle.name": "log_ilm_policy",
    "index.lifecycle.rollover_alias": "ilm_alias",
    "index.routing.allocation.include.box_type":"hot"
  },
  "aliases": {
    "ilm_alias": {
      "is_write_index": true
    }
  }
}

// 对 Alias写入文档
POST  ilm_alias/_doc
{
  "dfd":"dfdsf"
}
```

除了通过API创建`ILM`管理之外, 还可以通过`Kibana`进行动态管理, 路径为: `Management -> Elasticsearch -> Index Lifecycle Management`.
