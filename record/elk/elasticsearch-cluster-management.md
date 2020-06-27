# ElasticSearch 集群管理

![illustrate](https://image.cjyong.com/illustrated-screenshot-hero-apm.png)

本章主要关注ES集群管理: 用户身份校验, SSL加密通信, 集群部署方式, 水平拓展集群, 集群分片设计, 集群容量分配管理, 私有云部署等等.

**笔记主要来自购买的极客时间课程`阮一鸣 - ElasticSearch核心技术和实战`, [原课程链接](https://time.geekbang.org/course/intro/100030501), 主要用于个人温习和查阅, 如有侵权, 请及时邮件联系**

## ES安全管理

### 身份认证和授权

一般在生产环境为了对`ES`集群进行安全的访问, 不推荐直接通过`IP`加端口号进行直接访问, 因为这需要开放`ES`远程访问的端口, 这是非常不安全的. 一般推荐使用`NGINX`反向代理, 而`ES`则设置只允许本地访问(默认设置), 另外加上合适的身份认证.

**Nginx**反向代理配置示例, 假设`es`和`kibana`都部署在本地:

```yml
location ^~/es/ {
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:9200/;
}
location /kibana/ {
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:5601/;
}
```

需要注意的是, 在`kibana`的配置文件中需要添加如下配置:

```yml
server.host: "localhost"
server.basePath: "/kibana"
```

通过代理配置, 我们就可以通过`域名 + /es`访问`ES`集群, 通过`域名+ /kibana`访问`kibana`.

对于用户身份认证, 则可以通过以下三种方式实现:

- nginx 用户认证机制
- 免费`ES`安全插件

  - [Search Guard](https://search-guard.com/)
  - [ReadOnly REST](https://github.com/sscarduzio/elasticsearch-readonlyrest-plugin)

- ES内置的`X-PACK`安全认证机制

  - 内置 Realms (免费): File / Native(用户名密码保存在 Elasticsearch)
  - 外部 Realms (收费): LDAP / Active Directory / PKI / SAML / Kerberos

这里以免费的`X-PACK`进行演示:

- 开启`XPACK`的`security`功能: 在`elasticsearch.yml`配置文件中, 添加: `xpack.security.enabled: true`, 重启`es`服务.
- 初始化内置账号密码: `bin/elasticsearch-setup-passwords interactive`, 初始化各类内置账号.
- 更新`kibana`配置: 在`kibana.yml`配置文件中, 加入账号密码配置:

  - elasticsearch.username: "kibana"
  - elasticsearch.password: "xxx"

- 如有其它系统, 如`logstash`等, 都需要加入内部预设好的账号密码配置.

`ES`内部的安全认证机制为: 定义好各种不同类型的角色类型(`role`), 角色类型具备各种各样的权限. 然后设置用户(`user`)具备哪些角色类型, 从而确定用户的各类权限.

首先可以通过[`API接口创建角色`](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-put-role.html), 使用[`API接口创建用户`](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-put-role.html)

示例:

```java
// 创建角色
POST /_security/role/my_admin_role
{
  "cluster": ["all"],
  "indices": [
    {
      "names": [ "index1", "index2" ],
      "privileges": ["all"],
      "field_security" : { // optional
        "grant" : [ "title", "body" ]
      },
      "query": "{\"match\": {\"title\": \"foo\"}}" // optional
    }
  ],
  "applications": [
    {
      "application": "myapp",
      "privileges": [ "admin", "read" ],
      "resources": [ "*" ]
    }
  ],
  "run_as": [ "other_user" ], // optional
  "metadata" : { // optional
    "version" : 1
  }
}
// 创建用户
POST /_security/user/jacknich
{
  "password" : "j@rV1s",
  "roles" : [ "admin", "other_role1" ],
  "full_name" : "Jack Nicholson",
  "email" : "jacknich@example.com",
  "metadata" : {
    "intelligence" : 7
  }
}
```

然后还可以通过`kibana`可视化界面进行创建和管理, 路径为: `Management -> Security -> Users/Roles`.

设置好用户和角色之后, 我们使用`API`请求或者`登录kibana`时, 就都需要进行用户认证.

### 集群内部间的安全通信

对于集群内部的通信, `ES`支持进行`SSL`加密通信, 可以防止数据抓包, 验证身份, 避免`Impostor Node`.

处理的方式为每一个节点添加证书:

- TLS

  - TLS 协议要求`Trusted Certificate Authority(CA)`签发的`X.509`的证书

- 证书认证的不同级别

  - Certificate – 节点加入需要使用相同 CA 签发的证书
  - Full Verification – 节点加入集群需要相同 CA 签发的证书，还需要验证 Host name 或 IP 地址
  - No Verification – 任何节点都可以加入，开发环境中用于诊断目的

主要步骤:

生成证书

```shell
// 为您的Elasticearch集群创建一个证书颁发机构。例如，使用elasticsearch-certutil ca命令：
bin/elasticsearch-certutil ca

// 为群集中的每个节点生成证书和私钥。例如，使用elasticsearch-certutil cert 命令：
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

// 将证书拷贝到 config/certs目录下
elastic-certificates.p12
```

使用证书启动节点:

```shell
bin/elasticsearch -E node.name=node0 -E cluster.name=geektime -E path.data=node0_data -E http.port=9200 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enabled=true -E xpack.security.transport.ssl.verification_mode=certificate -E xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12
```

如果觉得命令行参数过长, 可以写入到每个节点的配置文件(`elasticsearch.yml`)中:

```yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate

xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
```

### 集群与外部间的安全通信

在外部系统连接`ES`集群时, 同样可以开启`SSL`进行安全通信. 这里以`Kibana`为例:

首先开启`ES`的SSL校验(依照上面的步骤).

生成`kibana`使用的证书:

```shell
// 转换生成kibana使用的证书
openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elastic-ca.pem

// 更新kibana的配置(kibana.yml)
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.certificateAuthorities: [ "/path/to/cert/elastic-ca.pem" ]
elasticsearch.ssl.verificationMode: certificate
```

最后可以开启`kibana`的`HTTPS`访问(如果使用`nginx`代理可以忽略这一步):

```shell
// 生成后解压，包含了instance.crt 和 instance.key
bin/elasticsearch-certutil ca --pem

server.ssl.enabled: true
server.ssl.certificate: config/certs/instance.crt
server.ssl.key: config/certs/instance.key
```

## 水平拓展ES集群

### 常见的集群部署方式

前面了解过节点有不同的节点类型:

- Master eligible
- Data
- Ingest
- Coordinating
- Machine Learning

在开发环境中，一个节点可承担多种角色. 但是在生产环境中, 推荐以下两种做法:

- 根据数据量，写入和查询的吞吐量，选择合适的部署方式
- 建议设置单一角色的节点(dedicated node)

一个节点在默认情况会下同时扮演:master eligible，data node 和 ingest node:

节点类型              | 配置参数        | 默认值
:---------------- | :---------- | :---------------------
master eligible   | node.master | true
data              | node.data   | true
ingest            | node.ingest | ture
coordinating only | 无           | 设置上面三个参数全部为false
machine learning  | node.ml     | true (需要enable x-pack)

如何设置单一角色节点呢:

- `Master节点`: `node.master: true, node.ingest: false, node.data: false`
- `Data节点`: `node.master: false, node.ingest: false, node.data: true`
- `Ingest节点`: `node.master: false, node.ingest: true, node.data: false`
- `Ingest节点`: `node.master: false, node.ingest: false, node.data: false`

单一角色:职责分离的好处

- Dedicated master eligible nodes: 负责集群状态(cluster state)的管理

  - 使用低配置的 CPU，RAM 和磁盘

- Dedicated data nodes: 负责数据存储及处理客户端请求

  - 使用高配置的 CPU, RAM 和磁盘

- Dedicated ingest nodes: 负责数据处理

  - 使用高配置 CPU;中等配置的RAM; 低配置的磁盘

特殊节点配置:

- Dedicate Coordinating Node: 负责接收Client请求, 将请求分发到合适的节点, 最终将结果汇集到一起.

  - 使用中高配置 CUP, 中高配置的 RAM, 低配置磁盘
  - 生产环境中，建议为一些大的集群配置 Coordinating Only Nodes: 扮演 Load Balancers 角色。

    - 降低 Master 和 Data Nodes 的负载
    - 负责搜索结果的 Gather/Reduce
    - 有时候无法预知客户端会发送怎么样的请求: 避免存在大量占用内存的聚合操作，一个深度聚合可能会引发 OOM.

- Dedicate Master Node

  - 从高可用 & 避免脑裂的角度出发

    - 一般在生产环境中配置 3 台
    - 一个集群只有 1 台活跃的主节点
    - 负责分片管理，索引创建，集群管理等操作

  - 如果和数据节点或者 Coordinate 节点混合部署

    - 数据节点相对有比较大的内存占用
    - Coordinate 节点有时候可能会有开销很高的查询，导致 OOM
    - 这些都有可能影响 Master 节点，导致集群的不稳定

**部署方式**:

当磁盘容量无法满足需求时，可以增加数据节点; 磁盘读写压力大时，增加数据节点:

![es-base](https://image.cjyong.com/es-base-deploy.png)

水平扩展:Coordinating Only Node: 当系统中有大量的复杂查询及聚合时候，增加 Coordinating 节点，增加查询的性能.

![es-level-deploy](https://image.cjyong.com/es-level-deploy.png)

读写分离

![es-read-write](https://image.cjyong.com/es-read-write-seperate.png)

在集群中部署 Kibana

![es-kibana](https://image.cjyong.com/es-kibana.png)

异地多活: 集群处在三个数据中心;数据三写;GTM 分发读请求

![multi-place](https://image.cjyong.com/es-multi-place.png)

### Hot & Warm 架构与 Shard Filtering

`Hot & Warm Architecture`:

- 数据通常不会有 Update 操作;适用于 Time based 索引数据(生命周期管理)，同时数据量比较大的场景: 如日志.
- 引入 Warm 节点，低配置大容量的机器存放老数据，以降低部署成本

`两类数据节点, 不同的硬件配置`:

- Hot 节点(通常使用 SSD):索引有不断有新文档写入。通常使用 SSD
- Warm 节点(通常使用 HDD):索引不存在新数据的写入;同时也不存在大量的数据查询

使用 Shard Filtering，步骤分为以下几步

- 标记节点 (Tagging)
- 配置索引到 Hot Node
- 配置旧索引移动到 Warm 节点

**标记节点**:

通过设置`node.attr`的属性设置即可:

```yml
// 标记一个 Hot 节点
bin/elasticsearch -E  path.data=hot_data -E node.attr.my_node_type=hot

// 标记一个 warm 节点
bin/elasticsearch -E path.data=warm_data -E node.attr.my_node_type=warm

// 同样可以在每一个节点的yml文件中进行配置
```

**配置索引到 Hot Node**:

```java
PUT logs-2019-06-27
{
  "settings":{
    "number_of_shards":2,
    "number_of_replicas":0,
    // 这里设置对应attr类型
    "index.routing.allocation.require.my_node_type":"hot"
  }
}
```

**配置旧索引移动到 Warm 节点**:

通过更新旧索引中的`allocation`属性值即可

```java
PUT logs-2019-06-27/_settings
{  
  "index.routing.allocation.require.my_node_type":"warm"
}
```

#### Rack Awareness

`ES`的不同节点可能部署在不同的机架上, 可以通过设置节点的`rack`信息, 将其分布在不同的机架上, 防止单个机架出现问题, 导致数据无法恢复.

**比较节点的机架信息**:

```java
// 标记一个 rack 1
bin/elasticsearch -E node.attr.my_rack_id=rack1

// 标记一个 rack 2
bin/elasticsearch -E node.attr.my_rack_id=rack2

// 同样可以存储在节点的yml文件中
```

**设置集群中的节点信息位置**:

```java
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "my_rack_id",
  }
}
```

**强制使用不同的机架**:

```java
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "my_rack_id",
    "cluster.routing.allocation.awareness.force.my_rack_id.values": "rack1,rack2"
  }
}
```

如果本地只有一个机架的节点(如rack1), 但是索引中设置的副本属性是一(>0), 就会导致无法分配副本分片. 这时候可以通过, 如下接口查询原因:

```java
GET _cluster/allocation/explain?pretty
```

当然除了`Rack Awareness`之外, 还可以设置其他的索引对所分配的节点要求, 即: 节点需要满足索引的要求, 否则无法分配到该节点.

设置                                      | 分配索引到节点，节点的属性规则
:-------------------------------------- | :--------------
Index.routing.allocation.include.{attr} | 至少包含一个值
Index.routing.allocation.exclude.{attr} | 不能包含任何一个值
Index.routing.allocation.require.{attr} | 所有值都需要包含

## 分片的规划和管理

### 分片管理

7.0 开始，新创建一个索引时，默认只有一个主分片:

- 单个分片，查询算分，聚合不准的问题都可以得以避免

单个索引，单个分片时候，集群无法实现水平扩展

- 即使增加新的节点，无法实现水平扩展

如果设置两个主分片: `集群增加一个节点后，Elasticsearch 会自动进行分片的移动，也叫 Shard Rebalancing`.

设计的原则:

当分片数 > 节点数时

- 一旦集群中有新的数据节点加入，分片就可以自动进行分配
- 分片在重新分配时，系统不会有 downtime

多分片的好处:一个索引如果分布在不同的节点，多个节点可以并行执行

- 查询可以并行执行
- 数据写入可以分散到多个机器

如何确定主分片数 ?

从存储的物理角度看

- 日志类应用，单个分片不要大于 50 GB
- 搜索类应用，单个分片不要超过20 GB

为什么要控制分片存储大小?

- 提高 Update 的性能
- Merge 时，减少所需的资源
- 丢失节点后，具备更快的恢复速度 / 便于分片在集群内 Rebalancing

如何确定副本分片数

副本是主分片的拷贝

- 提高系统可用性:相应查询请求，防止数据丢失
- 需要占用和主分片一样的资源

对性能的影响

- 副本会降低数据的索引速度:有几份副本就会有几倍的 CPU 资源消耗在索引上
- 会减缓对主分片的查询压力，但是会消耗同样的内存资源

  - 如果机器资源充分，提高副本数，可以提高整体的查询 QPS

调整分片总数设定，避免分配不均衡: 当一个新的数据节点加入时, 如果其他数据节点基本存储满了, 为了避免`ES`将所有的新索引都放到这个节点上, 导致数据分布不均匀, 热点数据过于集中. 可以通过两个设定来避免这个问题:

```java
// 设定索引在每个节点上最多的分片数
index.routing.allocation.total_shards_per_node: x;
// 设定在集群中, 每个节点最多分配的分片数
cluster.routing.allocation.total_shards_per_node: x;
```

### 集群的容量规划

一个集群总共需要多少个节点? 一个索引需要设置几个分片?

- 规划上需要保持一定的余量，当负载出现波动，节点出现丢失时，还能正常运行

做容量规划时，一些需要考虑的因素:

- 机器的软硬件配置
- 单条文档的尺寸 / 文档的总数据量 / 索引的总数据量(Time base 数据保留的时间)/ 副本分片数
- 文档是如何写入的(Bulk的尺寸)
- 文档的复杂度，文档是如何进行读取的(怎么样的查询和聚合)

评估业务的性能需求

- 数据吞吐及性能需求
- 数据写入的吞吐量，每秒要求写入多少数据?
- 查询的吞吐量?
- 单条查询可接受的最大返回时间?

了解你的数据

- 数据的格式和数据的 Mapping
- 实际的查询和聚合长的是什么样的

常见用例

搜索:固定大小的数据集

- 搜索的数据集增长相对比较缓慢

日志:基于时间序列的数据

- 使用 ES 存放日志与性能指标。数据每天不断写入，增长速度较快
- 结合 Warm Node 做数据的老化处理

硬件配置:

- 选择合理的硬件，数据节点尽可能使用 SSD
- 搜索等性能要求高的场景，建议 SSD

  - 按照 1 :10 的比例配置内存和硬盘

- 日志类和查询并发低的场景，可以考虑使用机械硬盘存储

  - 按照 1:50 的比例配置内存和硬盘

- 单节点数据建议控制在 2 TB 以内，最大不建议超过 5 TB

- JVM 配置机器内存的一半，JVM 内存配置不建议超过 32 G

部署方式

- 按需选择合理的部署方式
- 如果需要考虑可靠性高可用，建议部署 3 台 dedicated 的 Master 节点
- 如果有复杂的查询和聚合，建议设置 Coordinating 节点

**容量规划案例 1: 固定大小的数据集**

- 一些案例:唱片信息库 / 产品信息
- 一些特性:

  - 被搜索的数据集很大，但是增长相对比较慢(不会有大量的写入)。更关心搜索和聚合的读取性能
  - 数据的重要性与时间范围无关。关注的是搜索的相关度

- 估算索引的的数据量，然后确定分片的大小

  - 单个分片的数据不要超过 20 GB
  - 可以通过增加副本分片，提高查询的吞吐量

- 如果业务上有大量的查询是基于一个字段进行 Filter，该字段又是一个数量有限的枚举值

  - 例如订单所在的地区

- 如果在单个索引有大量的数据，可以考虑将索引拆分成多个索引

  - 查询性能可以得到提高
  - 如果要对多个索引进行查询，还是可以在查询中指定多个索引得以实现

- 如果业务上有大量的查询是基于一个字段进行 Filter，该字段数值并不固定

  - 可以启用 Routing 功能，按照 filter 字段的值分布到集群中不同的 shard，降低查询时相关的 shard， 提高 CPU 利用率

**容量规划案例 2: 基于时间序列的数据**:

相关的用案:

- 日志/指标/安全相关的Events
- 舆情分析

一些特性:

- 每条数据都有时间戳;文档基本不会被更新(日志和指标数据)
- 用户更多的会查询近期的数据;对旧的数据查询相对较少
- 对数据的写入性能要求比较高

创建 time-based 索引

- 在索引的名字中增加时间信息
- 按照每天/每周/每月的方式进行划分

带来的好处

- 更加合理的组织索引，例如随着时间推移，便于对索引做的老化处理
- 利用 Hot & Warm Architecture
- 备份和删除以及删除的效率高。( Delete By Query 执行速度慢，底层不也不会立刻释放空间，而 Merge 时又很消 耗资源)

基于时间的索引, 可以使用`ES`内置的`Data Math`进行查询:

如今天为: `2019-08-01T00:00:00`:

- `<logs-{now/d}>`: logs-2019.08.01
- `<logs-{now{YYYY.MM}}>`: logs-2019.08
- `<logs-{now/w}>`: logs-2019.7.29

查询时候需要进行转义:

```java
// POST /<logs-{now/d}/_search
POST /%3Clogs-%7Bnow%2Fd%7D%3E/_search
```

**集群扩容**:

增加 Coordinating / Ingest Node

- 解决 CPU 和 内存开销的问题

增加数据节点

- 解决存储的容量的问题
- 为避免分片分布不均的问题，要提前监控磁盘空间，提前清理数据或增加节点(70%)

### 在私有云上管理 Elasticsearch 的一些方法

推荐的使用方法有:

- `Elastic Stack`: 手动管理, 手工修复和更换节点, 包括数据备份, 版本升级等.
- `ECE: Elastic Cloud Enterprise`: 官方提供收费工具帮助管理多个集群.
- `基于Kubernetes容器化管理`

  - 基于容器技术，使用 Operator 模式进 行编排管理
  - 配置，管理监控多个集群
  - 支持 Hot & Warm
  - 数据快照和恢复

- `基于虚拟机的编排管理方式`

  - Puppet Infrastructure (Puppet / Elasticsearch Puppet Module / Foreman)
  - Workflow based Provision & Management

### 公有云管理 Elasticsearch

推荐使用 `Elastic Cloud`, 支持部署到阿里云和腾讯云.
