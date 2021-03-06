# SQL 进阶学习之 MYSQL

本章内容主要摘自 _<<高性能 MYSQL(V3)>>_读书笔记, 部分内容来自后序的学习和补充.

## MYSQL 的逻辑架构

![结构](https://image.cjyong.com/mysql-structure.png)

- 客户端: 和 MySQLServer 建立连接, 发送请求信息, 接收响应的结果集, 展示等.
- Server层: MySQL服务中的服务层, 涵盖了MySQL的大部分功能，如存储过程、触发器、视图, BinLog等. 主要的模块有: 连接器、缓存、分析器、优化器、执行器, BinLog等.
- 存储层: 主要负责数据的存储和提取, 支持多个存储引擎实现，例如：InnoDB、MyISAM等. Mysql5.5之后默认引擎为 InnoDB.

### 服务层模块

- 连接器：当客户端登陆MySQL的时候，对身份认证和权限判断。
- 缓存: 执行查询语句的时候，会先查询缓存（MySQL 8.0 版本后移除该模块）。
- 分析器: 假设在没有命中查询缓存的情况下，SQL请求就会来到分析器。分析器负责明确SQL要完成的功能，以及检查SQL的语法是否正确。
- 优化器：为SQL提供优化执行的方案。
- 执行器: 将语句分发到对应的存储引擎执行，并返回数据。

#### 连接器

客户端通过连接器和服务器端建立连接, 连接器模块负责对连接进行身份认证和权限校验. 首先校验连接的账号和密码, 如果校验通过之后会在权限表中查询对应权限信息, 然后将权限赋予给该用户. 到此连接成功建立.

可以通过 `show processlist` 查看所有成功建立的连接及其状态, [官方文档指引](https://dev.mysql.com/doc/refman/5.6/en/show-processlist.html).

#### 缓存

在客户端和服务器端建立连接之后, 就可以执行SQL语句. 在执行SQL语句之前, 会先查询该 `SQL` 语句是否执行过存在缓存中, 如果命中了缓存, 就直接返回缓存的结果集. 如果没有命中, 则进行后续的执行过程, 执行成功之后将结果集, 通过 `key-value` (key: SQL语句, Value: 结果集)的形式存储在缓存中.

注意, Mysql8.0版本之后删除了缓存的使用场景, 即8.0之后, 默认不会查询缓存了. 如果在之前的版本, 不想要查询缓存, 可以设置 `query_cache_type` 为 `DEMAND`, 这样默认的SQL语句的执行就不会查询缓存了.

#### 分析器

当缓存未命中时, 会进入分析器模块. 分析器模块主要执行两部分工作:

- 词法分析(Lexical scanner): 主要负责从SQL 语句中提取关键字，比如：查询的表，字段名，查询条件等等。
- 校验语法规则(Grammar rule module): 主要判断SQL语句是否满足MySQL的语法。

如: `select username from userinfo`, 首先会进行语法分析, 提取关键字:

关键字    | 非关键字     | 关键字  | 非关键字
:----- | :------- | :--- | :-------
select | username | from | userinfo

其次会通过语法规则解析, 校验是否满足MYSQL语法, 生成如下语法树:

![语法树](https://image.cjyong.com/mysql-graph.png)

#### 优化器

当分析器校验完成后, 由优化器对分析器生成的语法树进行优化, 生成最佳的执行方案. 主要的优化过程分为两部分:

- 逻辑变换: 在关系代数基础上进行变换，其目的是为了化简，同时保证SQL变化前后的结果一致. 主要有以下几个方面:

  - 否定消除：针对表达式"和取"或"析取"前面出现"否定"的情况，应将关系条件进行拆分，从而将外层的"NOT"消除。
  - 等值常量传递：利用了等值关系的传递特性，为了能够尽早执行"下推"运算。"下推"的基本策略是，始终将过滤表达式尽可能移至靠近数据源的位置。
  - 常量表达式计算：对于能立刻计算出结果的表达式，直接计算结果，同时将结果与其他条件尽量提前进行化简。

- 代价优化: 用来确定每个表，根据条件是否应用索引，应用哪个索引和确定多表连接的顺序等问题。为了完成代价优化，需要找到一个代价最小的方案。代价消耗主要分为两部分:

  - 服务层代价(主要为CPU代价成本), 5.7版本之后存储于 `mysql.server_cost` 数据表中.

    - row_evaluate_cost (default 0.2) 计算符合条件的行的代价，行数越多，此项代价越大
    - memory_temptable_create_cost (default 2.0) 内存临时表的创建代价
    - memory_temptable_row_cost (default 0.2) 内存临时表的行代价
    - key_compare_cost (default 0.1) 键比较的代价，例如排序
    - disk_temptable_create_cost (default 40.0) 内部myisam或innodb临时表的创建代价
    - disk_temptable_row_cost (default 1.0) 内部myisam或innodb临时表的行代价

  - 引擎层代价(主要为IO代价成本), 5.7版本之后存储于 `mysql.engine_cost` 数据表中.

    - io_block_read_cost (default 1.0) 从磁盘读数据的代价，对innodb来说，表示从磁盘读一个page的代价
    - memory_block_read_cost (default 1.0) 从内存读数据的代价，对innodb来说，表示从buffer pool读一个page的代价

Mysql会生成不同的执行计划, 然后基于上述的代价表计算各自的代价成本, 选择最小的代价的执行计划进行执行.

#### 执行器

当优化器生成执行计划之后, 会到达执行器进行执行. 在执行之前, 首先会校验用户的权限信息, 如果不满足, 则会返回报错信息. 如果满足, 则会调用引擎层接口进行查询.

SQL语句的执行顺序在分析器已经定义好了, 执行时按照以下顺序进行处理:

- from: 定位需要处理的数据表
- on: 对数据库表进行连接和过滤
- join: 数据库表连接操作
- where: 对结果集进行过滤
- group by: 对结果集进行分组
- having+聚合: 对结果进行过滤
- select: 选择特定的字段
- order by: 进行排序操作
- limit: 限制返回的数量

## 事务

MySql支持不同的事务隔离级别: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`. 默认的隔离级别为: `REPEATABLE READ`.

### 基本特性: ACID

事务的基本特性:

- 原子性(Atomicity): 一个事务必须被视为一个不可分割的最小工作单元, 要不全部提交成功, 要不全部失败回滚.
- 一致性(Consistency): 数据库总是从一个一致性的状态转换为另外一个一致性的状态.
- 隔离性(Isolation): 一个事务所做的修改在最终提交之前, 对于其它事务都是不可见的.
- 持久性(Durability): 一旦事务提交, 所做修改都会永久存储到数据库中.

### 隔离级别

隔离级别             | 脏读  | 不可重复读 | 幻读
:--------------- | :-- | :---- | :--
Read uncommitted | 可能  | 可能    | 可能
Read committed   | 不可能 | 可能    | 可能
Repeatable read  | 不可能 | 不可能   | 可能
Serializable     | 不可能 | 不可能   | 不可能

- READ UNCOMMITTED: 未提交可读, 允许脏读，也就是可能读取到其他会话中未提交事务修改的数据.
- READ COMMITTED: 只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别.
- REPEATABLE READ: 可重复读, 在同一个事务内的查询都是事务开始时刻一致的，InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读.
- SERIALIZABLE: 完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞.

异常情况:

- 脏读: 读取到其他会话中未提交事务修改的数据, 即读取的数据是脏数据.
- 不可重复读: 多次读取结果不一致. (如其他事务提交了修改).
- 幻读: 如更新了全部数据然后提交, 这时候其他事务提交了新的数据, 这时候发现有一条数据没有更新成功.

### 锁实现

[InnoDB提供多种锁实现](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html), 包括: 意向锁, 行锁, 间隙锁, 临键锁, 插入意向锁, 自增锁来实现不同的隔离级别.

`InnoDB` 支持两种类型的行锁: 共享锁(S)用于读取(类似于读写锁里面的读锁), 排他锁(X)用于更新(类似于读写锁里面的共享锁). 读读共享, 读写互斥.

- **意向锁**: InnoDB支持多种粒度锁定: 行锁和表锁共存. 行锁存在两种状态: S(共享行锁), X(独占行锁), 表锁也存在两种形态: IS(共享表锁), IX(独占表锁). 其中表锁: IS, IX 称为意向锁. 标明接下来我要在这个表里面获取什么类型的行锁. 一种初步的意向. 当我们获取一个共享的行锁时: `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE LOCK IN SHARE MODE`, 这时候 `InnoDB` 后首先获取这个表的 `IS` 锁(意向锁), 其次再获取这个记录的 `S` 锁(行锁). 即要获取 S 行锁, 必须先获取 IS 意向锁(表锁). 要获取 X 行锁, 必须先获取 IX 意向锁(表锁).

  - 为什么需要意向锁呢? 如果没有意向锁, 给一个表添加独占锁, 需要遍历全表查看是否存在行锁(X,S锁), 才能确认是否可以添加表锁. 而通过意向锁, 只要检查是否兼容性即可. 兼容性:

兼容性 | X        | IX         | S          | IS
:-- | :------- | :--------- | :--------- | :---------
X   | Conflict | Conflict   | Conflict   | Conflict
IX  | Conflict | Compatible | Conflict   | Compatible
S   | Conflict | Conflict   | Compatible | Compatible
IS  | Conflict | Compatible | Compatible | Compatible

- **行锁**: 索引锁, 依赖索引来实现的行锁. 如: `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE`, 锁定 `t` 表里面的 `c1=10` 这条记录(X锁).
- **间隙锁**: 依赖索引实现的间隙锁, 可以锁定某一部分的索引. 如: `SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE`, 锁定 `10 <= c1 <= 20`, 里面的记录, 防止记录插入在里面. 注意间隙锁不是排他锁, 支持共享: 事务A锁定了 10 -> 20 , 还允许其他事务使用间隙锁锁定 10 -> 20的范围, 彼此并不会冲突. 注意在 `RC` 级别并不会激活间隙锁.
- **临键锁**: 行锁 + 间隙锁. 对当前记录行添加行锁, 也会对行内范围添加间隙锁, 这样可以防止其事务在操作的过程中插入新的记录(即幻读的情况).
- **插入意向锁**: 特殊的间隙锁, 只用于插入操作. 目的在于支持并发插入. 即对同一个索引, 如果插入的索引值互不相同, 运行使用插入意向锁同步进行插入.
- **自增锁**: 用于表字段设置了 `AUTO_INCREMENT` 时, 当插入时, 必须等待其他事务插入成功之后, 才可以正常获取自增锁.

### 多版本控制

.MYSQL 中大多数的事务型引擎都不是简单的行级和表锁(悲观锁), 基于性能的考虑一般使用多版本并发控制(`Multiversion Concurrency Control`)(乐观锁), 来减少不必要的加锁操作. 以`InnoDB`为例:

- `SELECT`: 查找符合的条件必须满足两条: 该数据行的事务版本号必须要小于当前事务版本号(保证之前是存在的), 如果存在删除的话, 删除的事务版本号要大于当前的事务版本号(保证事务读取到的行, 当时还没被删除).
- `INSERT`: 插入的数据行保存当前事务版本号作为行版本号.
- `DELETE`: 为删除的数据行保存当前事务版本号作为行删除版本号.
- `UPDATE`: 插入一条新的行记录, 保存当前的事务版本号作为行版本号, 并且作为原来行的行删除版本号.

通过这种多版本控制, 大多数的读操作都不需要加锁, 使的读取更快, 操作简单, 性能更好. 但是需要额外的存储空间, 进行更多的检查操作. 这个只在`REPEATABLE READ`和`READ COMMITTED`两个隔离级别工作.

### 事务日志

通过事务日志可以提高事务的效率, 每次事务操作, 只需要将操作记录在事务日志里(事务日志是追加的, 不用过分的磁盘检索), 然后修改内存快照即可. 而后再将事务日志的操作慢慢刷新到持久磁盘中. 目前大多数的存储引擎都是这般实现的, 常称为`Write-Ahead Logging`, 预写式日志.

## 存储引擎

MYSQL 将每个数据库(对应一个 schema)保存为数据目录下一个子目录, 并为每一个表创建一个`frm`文件保存表的定义. 因为使用目录和文件来保存数据库和表的定义, 所以大小写和平台相关.(Windows 下不敏感,UNIX 敏感).

常见的存储引擎有: `InnoDB`, `MyISAM`, `Archive`, `Blackhole`, `CSV`, `Federated`, `Memory`,`Merge`, `NDB`等. 查看`user`表单的存储引擎: `SHOW TABLE STATUS LIKE 'user' \G`. 这里主要介绍主流的存储引擎: `InnoDB`, `MYISAM`.

### InnoDB 存储引擎

InnoDB 是 MYSQL 默认的事务型存储引擎, 也是最重要, 使用最广泛的存储引擎, 被设计用于处理大量短期(`short-lived`)的事务(耗时短,一般不会回滚). 除了支持事务之外, `InnoDB`的高性能和自动崩溃恢复特性, 也使它在非事务存储的需求中应用非常广泛. 使用 MYSQL 时应该默认优先使用`InnoDB`存储引擎.

InnoDB 将数据存储在表空间中(tableSpace), 由 InnoDB 管理的一个黑盒子, 由一系列文件组成. MySQL4.1 版本之后可以将表数据和索引分别存在不同的文件里面. 使用`MVVC`来支持高并发, 支持四种隔离级别, 默认为`REPEATABLE READ`, 并通过`间隙锁(next-key locking)`来防止幻读的情况. 其中 InnoDB 表是基于聚族索引建立的, 对主键的查询具有很高的效率, 其余索引依赖主键索引, 一般主键索引推荐设置尽可能小. 内部做了很多优化, 如磁盘读写时使用可预测性预读, 在内存中创建自适应哈希索引来加速索引读取速度, 使用插入缓存区来加速插入操作.

### MYISAM 存储引擎

`MYSQL5.1`版本之前的默认存储引擎, 具有大量特性: 全文索引, 压缩, 空间函数(GIS)等, 但是不支持事务和行级锁, 还有就是崩溃之后无法安全恢复.

MYISAM 将表存储到两个文件中: 数据文件(.MYD 结尾)和索引文件(.MYI 结尾). MYISAM 支持特性: 加锁和并发(表锁), 修复(手工和自动检查, 但是存在数据丢失风险), 索引特性(支持全文索引和前 500 字符创建索引等), 延迟更新索引键(DELAY_KEY_WRITE, 每次修改完成之后, 不会立刻将索引文件写入文件, 而是写到内存的键缓冲区中, 清理缓冲区或者关闭表时统一写入, 可以极大提高性能但是存在索引损坏的风险). MYISAM 还支持表压缩(压缩之后的表, 不允许修改)可以极大地提升查询性能.

### 转换表的引擎

- `ALTER TABLE mytable ENGINE = INNODB`: 会加表锁, 新建一个表进行复制操作, 耗时较长, 不推荐在频繁访问表中使用.
- `mysqldump`工具导出数据, 然后修改数据文件中的引擎选项和表名, 导入即可.
- `CREATE TABLE innodb_table LIKE myisam_table; ALTER TABLE innodb_table ENGINE = INNODB; INSERT INTO innodb_table SELECT * FROM myisam_table`: 适用于数据量较小的表. 如果数据量非常大, 可以考虑使用分批次处理: `START TRANSACTION; INSERT INTO innodb_table SELECT * FROM myisam_table WHERE id BETWEEN x AND y; COMMIT;`.

## 表设计优化

良好的逻辑设计和物理设计是高性能的基石。

### 选择合理的数据类型

选择数据类型时,遵循如下原则:

- 更小的通常更好: 一般情况下优先选择可以正确存储数据的最小数据类型.
- 简单就好: 简单数据类型通常需要更少的 CPU 周期. 如整型比字符串操作代价低很多.
- 尽量避免 NULL: 通常情况尽量避免使用 NULL, 推荐指定`NOT NULL`.

整数类型: `TINYINT`, `SMALLINT`, `MEDIUMINT`, `INT`, `BIGINT`: 8,16,24,32,68 位存储空间. 存储范围为(-2^(N - 1))到(2^(N - 1)). 如`TINYINT`为`-128 ~ 127`. 如果设置为`UNSIGNED`属性, 表示为非负值, 可以让上限提高一倍. 如`TINYINT UNSIGNED`范围为`0 ~ 255`. 选择整型时, 尽量选择合适的大小, 避免浪费空间.

实数类型: 带小数的数字. 一般还可以用来存储比`BIGINT`还大的整数. `DECIMAL`: 存储精确的小数位计算. 如`DECIMAL(6,2)`范围为:`-9999.99 ~ 9999.99`. 浮点数: `FLOAT` 4 位字节, `DOUBLE` 8 位字节. `MYSQL`使用`DOUBLE`来进行内部浮点数计算.

字符串类型: `VARCHAR`, 存储可变长字符串. 通过 1 到 2 个字节存储字符串长度.5.0 版本之后会保留末尾空格. `CHAR`定长字符串, 根据字符串长度分配足够的空间. 适合存储定长的短字符串, 如密码的`MD5`值. `BLOB`和`TEXT`: 存储很大的数据而设计的数据类型, 前者采用二进制形式进行存储, 后者采用字符方式存储.

枚举类型: 如`CREATE TABLE enum_test( e ENUM('fish', 'apple', 'dog') NOT NULL);`. 底层进行自动转换, 实际存储的值为数字. 优点: 节约空间, 快速. 缺点: 无法更改枚举. 每次更改必须 `ALTER TABLE`. 除非在`MYSQL5.1`之后, 通过在尾部添加新对象的方式更改, 这样可以避免重建整个表.

日期和时间类型: `DATETIME`和`TIMESTAMP`. `DATETIME`: 保存 1001 年到 9999 年, 精度为秒, 底层封装成`YYYYMMDDHHMMSS`的整数中进行存储, 使用 8 个字节存储. 一般的显示格式为: `2008-01-16 22:37:08`. `TIMESTAMP`存储从 1970 年 1 月 1 日午夜到现在的秒数, 和 UNIX 时间戳定义相同, 使用 4 个字节存储, 范围为`1970 ~ 2038`年, 依赖时区. 如果需要存储比秒更小刻度的时间值, 推荐使用`BIGINT`存储或者使用`MariaDB`替代`MYSQL`.

位数据类型: `BIT`, 存储位数值. 需要注意的`MYSQL`将`BIT`当作字符串类型, 而不是数字类型, 但是在数字上下文时却会转化为数字. 如存储的值为`b'00111001`(二进制值为 57)到`a bit(8)`中, `SELET a, a + 0`时结果为`9, 57`. 前者为`ASCII中57位为9`. 一般情况下谨慎使用`BIT`类型, 尽量避免使用这种类型. 如果需要在一个`BIT`中存储`true/false`值, 可以使用`CHAR(0)`替代, 存储`NULL`或者空字符串. 如果需要存储多个`true/false`值, 可以考虑合并到一个`SET`数据类型中, 如`perms SET('CAN_READ', 'CAN_WRITE', 'CAN_DELETE')`.代价和枚举类型, 修改时非常困难. 或者通过整数的统一封装到对应的 bit 位中.

### 合理的选择范式和反范式

设计关系数据库时，遵从不同的规范要求，设计出合理的关系型数据库，这些不同的规范要求被称为不同的范式，各种范式呈递次规范，越高的范式数据库冗余越小。

- 第一范式: 所谓第一范式（1NF）是指在关系模型中，对于添加的一个规范要求，所有的域都应该是原子性的，即数据库表的每一列都是不可分割的原子数据项，而不能是集合，数组，记录等非原子数据项。
- 第二范式: 在 1NF 的基础上，非码属性必须完全依赖于候选码（在 1NF 基础上消除非主属性对主码的部分函数依赖.
- 第三范式: 在 2NF 基础上，任何非主属性不依赖于其它非主属性（在 2NF 基础上消除传递依赖）
- BCNF: 在 3NF 基础上，任何非主属性不能对主键子集依赖（在 3NF 基础上消除对主码子集的依赖).

范式的好处:

- 更新操作要比反范式快: 数据冗余少, 更新时只需要更新更少的数据.
- 范式会使表更加小.
- 没有多余的数据意味着检索列表数据需要更少的`DISTINCT`和`GROUP BY`语句.

缺点: 查询数据时经常需要关联. 稍微复杂一点的查询语句在符合范式的`SCHEMA`中都可能需要不止一次的关联, 也许更多. 不仅代价昂贵, 还会让一些索引策略无效: 无法将索引建立在不同的表中.

反范式的好处: 通过部分冗余数据, 可以有效的避免关联. 缺点: 对于更新操作时, 往往需要更新更多的数据.

### 缓存表,汇总表和计数表

有时候创建一份完全独立的汇总表或者缓存表(特别是满足检索的需求时), 可以极大地提升性能. 避免对大表进行完整的索引和查找, 而是通过简单的汇总表查询即可. 如一个繁忙的网站需要计算之前 24 小时发送的消息数, 实时维护一个精确的计数器的代价是非常昂贵的. 作为替代方案可以每个小时生成一张汇总表, 这样一个简单的查询就可以获取到. 缺点就是无法达到 100%精确.

假设表的设计为:

```sql
CREATE TABLE msg_per_hr (
    hr DATETIME NOT NULL,
    cnt INT UNSIGNED NOT NULL,
    PRIMARY KEY(hr)
)
```

查询过去 24 小时发送消息的总数:

```sql
SELECT SUM(cnt) FROM msg_per_hr
WHERE hr BETWEEN
    CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 23 HOUR
    AND CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 1 HOUR;
```

如果应用在表中保存计数器, 则在更新计数器时可能遇到并发问题. 计数器表在 Web 应用中非常常见, 可以缓存一个用户的好友数, 文件的下载数等. 简单的一个表存储`cnt`值, 每次点击就更新该值. 问题在于想要更新这一行的事务来说, 这个记录上有一个全局的互斥锁(mutex), 这会使得事务只能串行执行.

要改进这个问题, 可以存储两个值`slot,cnt`. 建立 100 个`slot`, 每次随机更新其中一个`slot`, 最后计数时进行汇总即可. 还有一个需求是每隔一段时间开始一个新的计数器, 如每天一个. 就可以新增一个`day`属性:`(day, slot, cnt, primary key(day, slot))`.

```sql
INSERT INTO daily_hit_counter (day, slot, cnt) VALUES
(CURRENT_DATE, RAND() * 100, 1)
ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```

为了避免生成的行数过多, 可以定期合并所有的结果到 0 号 slot 中:

```sql
UPDATE daily_hit_counter  AS c
    INNER JOIN
    (SELECT day, SUM(cnt) as cnt, MIN(slot) AS mslot FROM daily_hit_counter GROUP BY day)
    AS x USING(day)
    SET c.cnt = IF(c.slot = x.mslot, x.cnt, 0),
        c.slot = IF(c.slot = x.mslot, 0, c.slot);

DELETE FROM daily_hit_counter WHERE slot <> 0 AND cnt = 0;
```

为了更快的读, 有时候我们必须牺牲一点写的代价, 这是需要我们进行权衡的.

### 加快 ALTER TABLE 操作的速度

MYSQL 的 ALTER TABLE 操作对于大表来说是一个非常大的问题. 一般 MYSQL 默认的执行方法是: 使用新的结构创建一个空表, 然后从旧表中查出所有的数据然后插入到新的表中, 最后删除旧表. 这样的操作往往非常耗时, 对于内存不足而表又很大的情况下, 这个操作往往需要花费几个小时甚至数天才能完成. 在这段时间都会导致 MYSQL 服务中断.

对于这种情况, 通用的解决方法有两种: 先在一台不提供服务的机器上执行`ALTER TABLE`操作, 然后和提供服务的主库进行切换. 使用`影子拷贝`, 创建一个和源表无光的新表, 然后通过重命名和删表操作交换两张表(有很多负责工具: `Facebook的online schema change`, `Shlomi Noach的openark toolkit`, `Percona Toolkit`). 对于一些特殊操作: 修改默认值. 可以通过直接修改`frm`文件进行完成, 这个操作比直接修改表示非常的快的:

```sql
ALTER TABLE sakila.film ALTER COLUMN rental_duration SET DEFAULT 5;
```

修改`frm`文件是非常快的, 对于某些列的修改: 移除`AUTO_INCREMENT`, 修改枚举类型数据常量(SET,ENUM)都是可以尝试使用这种方法(注意存在风险, 需要提前备份好数据).

- 创建一个相同结构的表, 并进行相关修改.
- 执行`FLUSH TABLES WITH READ LOCK`, 关闭正在使用的表, 并且禁止任何表打开.
- 交换`.frm`文件.
- 执行`UNLOCK TABLES`进行释放锁.

对于 MYISAM 表来说, 如果需要载入大量的数据, 可以先禁用索引, 载入数据后, 再重新启用索引:

```sql
ALTER TABLE test.load_data DISABLE KEYS;
-- load the data
ALTER TABLE test.load_date ENABLE KEYS;
```

载入数据之后, 可以通过排序来构造索引会快很多, 并且让索引树碎片更少, 更加紧凑. 但是这个对于唯一索引是无效的, 唯一索引需要在载入每一行时进行索引唯一性校验. 有一个取巧的方法就是, 先删除所有非唯一索引, 然后载入数据, 而后进行索引恢复.

## 索引

索引是存储引擎用于快速找到记录的一种数据结构.

### 常见的索引类型

索引是在存储层实现的, 没有统一的标准, 不同的存储引擎中索引的实现方式并不一样, 也不是所有的存储引擎都支持所有类型的索引. 常见的索引有:

#### B-Tree 索引

一般我们常说的索引就是 B 树索引, 大多数的存储引擎也是支持该索引的. 不同的存储引擎使用方式也不太一样, MYISAM 使用前缀压缩技术来压缩索引;INNODB 使用 B-Tree 索引来连接主键索引.

![SimpleB+Tee](https://image.cjyong.com/Bplustree.png)

通过 B+树的特性, 降低了树的高度, 减少在磁盘文件下 IO 的次数. 适合的查询: `全值查询`, `最左前缀查询`,`匹配列前缀`,`匹配范围值`,`精确匹配左前一列,并范围匹配左二列`, `覆盖查询`. 不支持的查询: `不适合非最左列开始查找`, `跳过索引查询`, `最左列范围查询, 会导致右列无法使用索引`.

#### 哈希索引

哈希索引, 实现方法类似 JDK1.7 的`HashMap`, 每次计算哈希值, 然后放到对应的桶里面, 如果存在冲突, 以链表的形式挂在后面. 只有`Memory`存储引擎显式支持该索引(也是默认索引). 优点: 结构紧凑, 查找速度非常快. 限制:

- 只包含哈希值和行指针, 不存储字段值, 无法支持覆盖查询.
- 不是按照索引顺序存储, 无法用于排序.
- 不支持部分匹配查询.
- 只支持等值比较查询. 也不支持范围查询.
- 如果数据冲突大的话, 对性能损耗较大: 查询效率低, 维护代价也很高.

INNODB 存储引擎内部有一个特殊的功能`自适应哈希索引(adaptive hash index)`, 如果 INNODB 发现某些索引值用的非常频繁的话, 会在内存中基于 B-Tree 索引之上, 再建立一个哈希索引, 来提升速度.

如果存储引擎不支持哈希索引, 我们可以`创建自定义的哈希索引`. 这里以一个伪哈希索引为例, 使用 B-Tree 来查找对应的哈希值, 而不是字段值. 这对于字段特别长的查询, 性能提升非常大.

```sql
/* 默认查询,比较字段较长,效率低下  */
SELECT id FROM urls WHERE url = "http://www.mysql.com";

/* 添加一行新的字段url_cc, 存储哈希值, 来作为索引 */
SELECT id FROM urls WHERE url = "http://www.mysql.com" AND url_cc=CRC32("http://www.mysql.com");

/* 创建触发器动态维护url_cc */
DELIMITER //

CREATE TRIGGER pseudohash_crc_ins BEFORE INSERT ON tableName FOR EACH ROW BEGIN SET NEW.url_crc=crc32(NEW.url);


CREATE TRIGGER pseudohash_crc_upd BEFORE UPDATE ON tableName FOR EACH ROW BEGIN SET NEW.url_crc=crc32(NEW.url);
//

DELIMITER ;
```

这里使用 32 位整数作为哈希值, 默认当索引达到 93000 条记录时, 出现冲突的概率为 1%. 如果数据量非常大的话, 可以自定义 64 位的整数哈希索引(如`FNV64()`), 这里提供一个简单的实现:

```sql
SELECT CONV(RIGHT(MD5('http://www.mysql.com'), 16), 16, 10) AS HASH64;
```

#### 空间数据索引(R-Tree)

MYISAM 支持空间索引, 可以用作地理数据存储, 和 B-Tree 索引不同, 无需前缀查询. 空间索引会从所有的维度来索引数据, 可以有效使用任意维度数据来组合查询.

#### 全文索引

全文索引, 特殊的索引, 查找文本中的关键字. 适用于`MATCH AGAINST`操作.

### 索引的优点

- 减少服务器需要扫描的数据量.
- 帮助服务器避免排序和临时表.
- 将随机 IO 转换为顺序 IO.

索引的评价(三星系统): 一星, 将相关记录存放在一起. 二星, 索引中的顺序和查找的顺序一致. 三星, 索引列中包含了所有的全部列.

### 高性能的索引策略

如何正确的创建和使用索引是实现高性能查询的基础. 这里介绍一些构建高性能索引的技巧:

#### 独立的列

使用索引时, 推荐把索引列单独放在一侧. 否则`MYSQL`就不会使用索引:

```sql
/* Bad implementation */
SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5;

/* Good implementation */
SELECT actor_id FROM sakila.actor where actor_id = 4;
```

#### 前缀索引

对于某些特别长的字段, 如大的`varchar`,`blob`,`text`类型的字段, 需要进行索引, 该怎么办. 一种方法就是使用前面的哈希索引. 还有一种方式就是使用前缀索引, 即取字段的一部分进行索引. 但是这也会引来一个新的问题, 取多长的值是合适的呢? 取长的字符串索引固然可以提升前缀索引的选择性(减少冲突性), 但是也会带来一定的性能损耗. 短的字符串虽然快速, 但是会降低前缀选择性.

这里我们以 table `city_demo`为例, 我们需要对表中的`city`建立前缀索引, 但是取多长的值合适呢? 默认的重复数量为 50-70 之间. 这里我们可以进行粗略地统计:

```java
/* 如果取3个字符串, 计算重复率 */
SELECT COUNT(*) AS cnt, LEFT(city, 3) AS pref FROM city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 10;

+------+------+
| cnt  | pref |
+------+------+
| 483  | San  |
| 195  | Cha  |
| 177  | Tan  |
| 167  | Sou  |
| ...  | ...  |
+------+------+
```

我们可以粗略地看出, 取 3 个字符串并不能有效地区分`city`. 我们逐渐测试 4,5,6,7,8,9 等. 最后可以得到取 7 的时候, 就基本上可以有效地区分`city`.

另一个方法就是计算完整列的选择性, 然后使的前缀的选择性接近于完整列的选择性.

```sql
/* 计算完整列的选择性 */
SELECT COUNT(DISTINCT city) / COUNT(*) FROM city_demo;

+---------------------------------+
| COUNT(DISTINCT city) / COUNT(*) |
+---------------------------------+
|                           0.0312|
+---------------------------------+


/* 计算前缀的选择性 */
SELECT COUNT(DISTINCT LEFT(city, 3)) / COUNT(*) AS sel3,
    COUNT(DISTINCT LEFT(city, 4)) / COUNT(*) AS sel4,
    COUNT(DISTINCT LEFT(city, 5)) / COUNT(*) AS sel5,
    COUNT(DISTINCT LEFT(city, 6)) / COUNT(*) AS sel6,
    COUNT(DISTINCT LEFT(city, 7)) / COUNT(*) AS sel7
FROM city_demo

+----------------------------------+
| sel3 | sel4 | sel5 | sel6 | sel7 |
+----------------------------------+
|0.0239|0.0293|0.0305|0.0309|0.0310|
+----------------------------------+
```

我们可以看出, 自从`sel7`之后, 提升就非常小了. 这时候取长度 7 就是合适的长度了.

```sql
ALTER TABLE city_demo ADD KEY (city(7));
```

#### 多列索引

很多人对多列索引的理解都不太够, 行为往往表现为为每一个列单独创建一个索引, 或者按照错误的顺序创建多列索引. 这往往会带来极大地性能损耗.

如如下查询:

```sql
SELECT fild_id, actor_id FROM film_actor WHERE actor_id = 1 OR film_id = 1;
```

对于这个查询, 为每一个列进行单独的索引往往是不合适的. 在以前的旧版本中, 往往会将这种查询转换为 UNION 的方式:

```sql
SELECT fild_id, actor_id FROM film_actor WHERE actor_id = 1
UNION ALL
SELECT fild_id, actor_id FROM film_actor WHERE film_id = 1 AND actor_id <> 1;
```

而在`Mysql5.0`及之后的版本, 往往使用`索引合并`的策略, 合并索引的结果. 这是一种优化措施, 这也说明建立地索引是非常差的.

- 当服务器发现多个索引做相交操作(通常为 AND), 通常意味着需要一个包含所有列的多列索引, 而不是单独的索引.
- 当服务器需要对多个索引做联合操作时, 通常需要花费大量的 CPU 和内存资源在算法的缓存, 排序和合并操作上. 尤其是索引的选择性不高, 数据量特别大的情况下.
- 更重要的是, 优化器并不会将这些操作算入到`查询成本`上, 优化器只关心随机页面的读取. 这会使得查询成本被低估, 导致代价高昂还不如走全表扫描.

#### 选择合适的索引顺序

当我们使用多列索引时, 索引的顺序是非常重要的. 因为在 B-tree 模型中, 索引的顺序是按照最左列进行排序的, 其次是第二列等. 如果选择顺序, 有一个经验法则: 将选择性高的列放在索引的最前列. 但是这并不是适用于任何场景的.

这里以这个查询为例:

```sql
SELECT * FROM payment WHERE staff_id = 2 AND customer_id = 584;
```

对于这个查询, 如果需要构建一个关于`staff_id`和`customer_id`的索引, 那么应该怎么选择顺序? 按照我们通用的经验, 我们计算选择性:

```sql
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity,
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*)
FROM payment;

/*
+-----------------------+------+
|staff_id_selectivity   |0.0001|
|customer_id_selectivity|0.0373|
|COUNT(*)               | 16049|
+-----------------------+------+
*/
```

从这里我们可以看出`customer_id`具有良好的选择性, 应该放在前面:

```java
ALTER TABLE payment ADD KEY(customer_id, staff_id);
```

但是这不能适配任何情况, 在某些场景中某些条件下的数量会非常的大, 这种情况下就需要不同的处理方式. 如对于某些应用, 处理没有登录的用户时, 都是将其用户名记录为`guest`, 在记录用户行为的`session`表中和其他记录用户活动表`guest`就成为了一个特殊的用户 ID. 一旦涉及到这个用户的查询, 就会变得非常不同. 因为数量非常大. 还有就是系统管理员账号, 所有的人都是系统管理员账号的好友, 系统也通过系统管理员账号给所有的用户发送信息. 这个账户的巨大的好友列表很容易导致网站服务器出现性能问题. 所以对于这种情况下, 都需要对特殊用户进行单独的处理, 执行特殊的查询语句.

#### 聚簇索引

聚簇索引并不是一个单纯的索引类型, 而是一种数据存储格式. 我们一般的 B-tree 索引中存储的是索引值和索引对应的数据行地址. 但是聚簇索引不是, 内部存储的不仅是索引值, 还有数据行信息, 即数据行是按照 B-Tree 的形式存储在文件中的, 相邻的行是放在一起的. 这里介绍 INNODB 的存储引擎的实现细节, 也适用于大多数聚簇索引的情况.

INNODB 一般默认使用主键作为聚簇索引. 如果不存在, 就选取一个独一无二的非空索引来作为聚簇索引. 这有时候会带来较大的性能优势, 有时候却会造成严重的性能问题. 优点:

- 可以把相关地址存储在一起, 方便检索相关信息.
- 数据访问更快. 省去了定位数据行和读取数据行的步骤.
- 覆盖索引扫描的查询可以直接使用叶节点中的主键值.

缺点:

- 聚簇索引可以很大程度地提高 I/O 密集型应用的性能, 但是如果数据全部放到内存中, 访问的顺序没这么重要时, 就没有多大的优势.
- 插入速度严重依赖插入顺序.按照主键的顺序插入是加载数据到 INNODB 中最快的方式. 如果不是, 推荐在加载完成之后, 使用`OPTIMIZE TABLE`重新组织一下表.
- 更新聚簇索引的代价非常高, 因为会强制移动行的位置.
- 基于聚簇索引的表在插入新行时, 或者主键被更新导致需要移动行时, 可能出现`页分裂`的问题. 插入到一个已经满了的页中, 有可能导致该页分裂成两页.
- 聚簇索引可能导致全表扫描缓慢, 尤其是行比较稀疏的时候或者页分裂导致数据存储不连续的情况下.
- 二级索引可能比想象的要大,效率较差, 二级索引中的叶子节点包含了引用行的主键值, 且需要二次索引查找.

一般的索引内部存储的是行地址. 而在聚簇索引中, 二级索引存储的是主键值, 不是行地址, 需要在主键索引中进行二次索引查找定位行地址. 这样可以避免表数据更新时更新二次索引. 但是效率会相对差一点. 聚簇索引推荐使用`AUTO_INCREMENT`作为主键的属性, 保证顺序插入.

#### 覆盖索引

覆盖索引, 顾名思义就是, 如果索引的叶节点中已经包含了要查询的数据, 那么就没有必要读取数据行, 直接进行返回就可以了. 覆盖索引可以极大地优化性能, 优点非常显著:

- 索引条目通常远小于数据行的大小, 所以如果只需要读取索引, 那么 MYSQL 就会极大地减少数据访问量.
- 因为索引是按照列值顺序存储的(在单页是这样的), 对于 I/O 密集型的范围查询会比随机从磁盘读取速率会高很多.
- 一些存储引擎如 MyISAM 在内存中只缓存索引, 数据则是依赖操作系统来缓存, 因此访问数据需要系统调用. 这可能导致严重的性能问题.
- 对于 INNODB 的聚簇索引, 覆盖索引是非常有用的. 如果二次索引的叶节点包含了主键值, 如果二级主键可以覆盖查询, 那么就可以避免对主键的二次查询.

不是所有的存储引擎都支持覆盖索引(Memory 不支持), 也不算所有的索引都支持覆盖索引(哈希索引, 空间索引和全文索引不支持). INNODB 一般采用 B-Tree 的形式来支持覆盖索引. 在使用到了覆盖索引的时候, 会在`EXPLAIN的Extra属性中显示Using Index`.

如`inventory`表中有一个多列索引`(store_id, film_id)`, 而 MYSQL 只需要查询这两个属性时:

```sql
EXPLAIN SELECT store_id, film_id FROM inventory;

/*
             id: 1
    select_type: SIMPLE
          table: inventory
           type: index
  possible_keys: NULL
            key: idx_store_id_film_id
        key_len: 3
            ref: NULL
           rows: 4693
          Extra: Using Index
*/
```

这里以一个例子来显示如何使用覆盖索引来优化查询:

```sql
EXPLAIN SELECT * FROM products WHERE actor='SEAN CARREY' AND title like '%APOLLO%';

/*
             id: 1
    select_type: SIMPLE
          table: products
           type: ref
  possible_keys: ACTOR,IX_PROD_ACTOR
            key: ACTOR
        key_len: 52
            ref: const
           rows: 10
          Extra: Using Where
*/
```

对于这个查询是无法使用覆盖查询的.

- 查询中选择了所有的列. 没有任何一个索引可以提供这么多信息.
- MYSQL 不允许在索引中执行`LIKE`操作.

这里使用一个特殊的方式跳过这些限制, 首先拓展索引为(artist, title, prod_id), 然后执行以下查询:

```sql
EXPLAIN SELECT * FROM products
    JOIN (SELECT prod_id FROM products
    WHERE actor='SEAN CARREY' AND title LIKE '%APOLLO%'
    ) AS t1 ON (t1.prod_id=products.prod_id);
```

这里通过临时表的方式, 通过覆盖索引查询到`prod_id`, 然后通过`prod_id`来获取对应的信息. 这也被称作延迟关联(`deferred join`). 注意, 对于 INNODB 的聚簇索引来说, 所有的二级索引默认包含主键值, 这个信息是稳定覆盖的:

```sql
EXPLAIN SELECT actor_id,last_name FROM actor
WHERE last_name = 'HOPPER';

/*
            id: 1
select_type: SIMPLE
        table: actor
        type: ref
possible_keys: idx_actor_last_name
        key: idx_actor_last_name
    key_len: 137
        ref: const
        rows: 2
        Extra: Using Where; Using index
*/
```

#### 使用索引来做排序

MYSQL 有两种方式生成有序结果: 排序操作和按照索引顺序扫描. 如果`EXPLAIN`出来的结果中`type`的值为`index`, 说明使用了索引扫描来做排序. 使用索引扫描排序可以极大地提高效率. 因此, 在设计索引时, 尽可能地同时满足查找任务, 也尽可能地满足排序任务: 索引的顺序和`ORDEY BY`子句的顺序完全一致, 且列的排序方向相同, 如果关联多个表时, 满足第一个表的条件时符合.

假设在表`rental`中存在索引`(rental_date, inventory_id, customer_id)`. 可以使用索引进行排序的查询:

```sql
/* First */
... ORDER BY rental_date, inventory_id, customer_id;

/* Second */
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id, customer_id;

/* Third */
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC;

/* Four */
... WHERE rental_date > '2005-05-25' ORDER BY rental_date inventory_id;
```

不能使用的:

```sql
/* First 排序方向不同*/
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC, customer_id ASC;

/* Second 存在非索引列*/
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id, staff_id;

/* Third 无法组合成最左前缀*/
... WHERE rental_date = '2005-05-25' ORDER BY customer_id;

/* Four 范围查询*/
... WHERE rental_date > '2005-05-25' ORDER BY inventory_id, customer_id;

/* Five 范围查询*/
... WHERE rental_date = '2005-05-25' AND inventory_id IN (1,2) ORDER BY customer_id;
```

#### 压缩(前缀压缩)索引

MYISAM 使用前缀压缩来减少索引大小, 从而将更多的索引放入内存中, 在某些情况下可以带来巨大的性能提升. 默认只压缩字符串, 也可以通过参数设置对整数进行压缩. 一般的压缩方式为: 完整保留第一个值, 后续的值存储差异值. 如第一个索引为`perform`, 第二个索引值为`performance`就存储为`7,ance`.

减少了空间使用, 也带来操作上的复杂性, 每个值都依赖前面的值. 无法二分查找, 只能从头开始扫描. 一般查询效率会变慢, 对于倒序扫描(DESC)效率会更慢.

#### 冗余和重复索引

MYSQL 支持维护重复索引, 这会严重影响性能. 避免使用重复索引. 而对于冗余索引, 则是有些不同, 如创建了(A,B)索引, 那么(A)索引就是冗余索引. 对于 INNODB 中的(A,ID)索引也是冗余索引.对于冗余索引, 基于不同的情况进行考虑. 如(A,B)和(A)来说, 如果 B 是一个非常长的字符串, 那么使用(A,B)时会比(A)要慢上很多, 如果既要保证(A,B)的查询效率, 也要保证(A)的查询效率, 那么保存两者也是可选的. 否则对于一般情况, 都不推荐使用冗余索引.

怎么发现重复索引和冗余索引呢? 推荐使用`Percona Tookit`中的`pt-duplicate-key-checker`来检查重复索引和使用`pt-upgrade`工具来追踪索引变更.

#### 未使用索引

有些索引创建了并未使用, 那就浪费了性能, 建议删除. 打开`userstates`服务器变量, 让服务器运行一段时间, 通过查询`INFOMATION_SCHEMA.INDEX_STATISTICS`就可以查到每个索引的使用频率. 另外还可以使用`Percona Toolkit`中的`pt-index-usage`工具读取日志进行分析.

#### 索引和锁

索引可以让查询锁定更少的行. 如果查询从不访问那些不需要的行, 那么就会锁定更少的行, 这对于性能是非常有好处的.

## 查询优化

前面介绍了如何设计最优的表结果和建立良好的索引。 除了这两者， 还需要合理的设计查询。

### 查询速度慢

一个查询快不快？核心是什么， 就是响应时间。 如果把查询看作一个任务， 那么它由一系列子任务组成， 每个子任务都会消耗一定的时间. 优化查询就是优化其子任务, 要么消除其中一些子任务, 要么减少子任务的执行次数.

推荐设置慢查询日志拦截所有的慢查询情况, 进行分析处理.

- set global slow_query_log = ON
- set global slow_query_time = 3600
- set global log_queries_not_using_indexes = ON

其次, 分析慢查询日志: `explain SLOW_SQL`, 尽可能的减少匹配的表记录(rows).

### 慢查询的基础: 优化数据访问

查询性能低下的基本原因是访问的数据太多, 某些查询可能不可避免地需要筛选大量数据. 大部分性能低下的查询都可以通过减少访问的数据量的方式进行优化:

- 是否检索了大量超过需要的数据.(一般意味着访问到行数过多, 有时候也可能访问了太多列).

  - 查询不需要的记录: 如只需要前 10 行, 却获取所有的数据, 然后取出前 10 行.
  - 多表关联时返回全部列: `SELECT sakila.actor.* FROM sakila.actor...`优于`SELECT * FROM xxx`.
  - 总是取出所有列: `SELECT *`时需要注意是否真的有必要, 因为一旦使用这个, 就意味着没办法使用覆盖索引等优化功能.
  - 重复查询相同的数据: 有时候经常重复查询相同的数据, 优化的方法就是缓存下来.

- 确认是否在分析超过需要的数据行.

  - 响应时间
  - 扫描的行数和返回的行数
  - 扫描的行数和访问类型

### 重构查询方式

#### 一个复杂查询还是多个简单查询

传统的考虑是一个复杂的查询是比多个查询快捷很多的, 认为在网络通信, 查询分析和优化是一件代价非常高的事. 但是对于 MYSQL 来说, 并不一定. MYSQL 在连接, 简单查询的效率都是非常轻量级的. 这时候需要合理的评估测量, 最后选出合适的方式.

#### 切分查询

有时候对于一个大的查询, 我们需要进行`分而治之`, 将大查询切分成小查询. 如删除旧数据, 如果使用一个大的语句一次性完成的话, 可能需要一次锁住很多数据, 沾满整个事务日志和耗尽系统资源等. 这时候进行合理的切分可以尽可能小的影响 MYSQL 的性能.

如 `DELTE FROM messages WHERE created < DATE_SUB(NOW(), INTERVAL 3 MONTH);` 可以切分为如下方法:

```java
int row_affected = 0;
do {
    rows_affected = do_query("DELETE FROM messages WHERE created < DATE_SUB(NOW(), INTERVAL 3 MONTH) LIMIT 10000");
} while (row_affected > 0);
```

甚至可以在每次只需一次删除操作之后, 软件暂停一段时间, 将压力合理地分布到一段合理地时间内.

#### 分解关联查询

很多高性能的应用都会将关联查询进行分解, 即进行单表查询, 然后在程序中进行关联.

如:

```sql
SELECT * FROM tag
    JOIN tag_post USING(tag_id)
    JOIN post USING(post_id)
WHERE tag.tag = 'mysql';
```

可以分解为:

```sql
SELECT * FROM tag WHERE tag='mysql';
SELECT * FROM tag_post WHERE tag_id=1234;
SELECT * FROM post WHERE tag_id in (123, 456, 567, 9098, 8904);
```

优点:

- 缓存效率更高.
- 减少锁的竞争.
- 应用层关联, 容易进行高性能和可拓展.
- 查询效率提升.
- 减少冗余记录的查询.

### 查询执行基础

MYSQL 执行一个查询的详细请求:

![SQL执行流程](https://image.cjyong.com/sqlexpath.png)

- 客户端发送一条查询给服务器.
- 服务器先检查查询缓存, 如果命中缓存, 则立刻返回存储在缓存中的结果. 否则进行下一阶段.
- 服务器进行 SQL 解析,预处理, 再由优化器生成对应的执行计划.
- MYSQL 根据优化器生成的执行计划, 调用存储引擎的 API 来执行查询.
- 将结果返回客户端.

客户端和服务器的通信协议是`半双工`的, 这意味着在任何一个时刻, 要么是由服务器向客户端发送数据, 要么是由客户端向服务器发送数据, 两者不能同时发生. 对于一个 MYSQL 连接, 也就是对应一个线程, 任何时候都有一个状态代表当前正在做什么.

- Sleep: 等待客户端发送请求.
- Query: 线程正在查询或者正在将结果返回给客户端.
- Locked: 正在等待锁, 表锁或者行锁.
- Analyzing and statistics: 收集存储引擎的统计信息, 并生成查询执行计划.
- Copying to tmp table [on disk]: 正在执行查询, 并且将其结果集复制到一个临时表中: GROUP BY 操作, 文件排序操作, UNION 操作.
- Sorting result: 对结果集进行排序.
- Sending Data: 多个状态之间传递数据, 生成结果集, 或者向客户端返回数据.

这个我们可以通过`SHOW FULL PROCESSLIST`命令查看到具体信息.

查询缓存, 缓存是存储在一个哈希表中, 对查询进行哈希计算, 如果命中缓存, 校验用户权限通过之后, 直接进行返回.否则进入下一步阶段.

查询优化处理: 查询缓存后的下一个生命周期就是将一个 SQL 转换成一个执行计划, MYSQL 再依照这个执行计划和存储引擎进行交互. 这里包括多个子阶段: 解析 SQL, 预处理, 优化 SQL 执行计划. 这里挑选几个独立的部分进行介绍:

- 语法解析器和预处理: MYSQL 通过关键字将 SQL 进行解析, 生成一颗对应的`解析树`.解析器使用 MYSQL 语法规则验证和解析查询: 如是否使用错误关键字, 关键字顺序是否正确等. 预处理器则是根据 MYSQL 规则进一步检查解析树是否合理: 如检查数据表和数据列是否存在, 名称是否有歧义.然后就是校验权限.
- 查询优化器: 到这一步`语法树`都是合法的了, 并且由优化器将其转化成执行计划. 一条查询可以有多个执行方式, 最后都返回相同的结果. 优化器的作用就是选择其中最好的执行计划. MYSQL 是通过基于成本的优化器, 它将尝试预测一个查询使用某种执行计划时的成本, 并选择其中成本最小的一个.`SHOW STATUS LIKE 'Last_query_cost'`进行查询上次查询的成本.
- MYSQL 的查询优化器是一个非常复杂的部件, 它使用了很多优化策略来生成最优的执行计划, 优化策略可以简单地分为两种: 静态优化和动态优化. 静态优化直接对解析树进行分析, 完成优化: 如将 Where 条件转化为等价形式等. 是永久性的, 不依赖特定的值, 类似于编译时优化. 动态优化则和上下文有关, 和很多因素有关, 如 WHERE 条件的取值, 索引中的条目数量等等. 需要在每次运行时进行重新评估, 类似于`运行时优化`. 这些策略包括但不限于: 重新定义管理表的顺序, 将外连接转化为内连接, 使用等价变换原则, 优化 COUNT()/MIN()/MAX(), 预估转化为常数表达式, 覆盖索引扫描, 子查询优化, 提前终止查询, 等值传播, 列表 IN()比较等.
- 数据和索引信息等多种统计信息是由存储引擎提供的, 不同的存储引擎可能存储不同的统计信息. 在 MYSQL 中执行关联查询, 不是按照平衡二叉树的方式进行关联, 而是通过单侧(左侧\右侧)深度优先的树进行关联的, 不同的顺序关联效率相差很多, 关联优化器会尝试选择最优的关联方式.
- 排序优化, 如果不能使用索引生成排序结果时, MYSQL 就需要自己进行排序, 如果数据量很小就直接在内存中进行排序, 否则就需要磁盘帮助. MYSQL 将这个过程统一称为`文件排序(filesort)`. 一般分为两种: 取出行指针和需要排序的字段, 排序生成有顺序的行指针, 按照行指针读取有顺序的所有记录. 第二种为读取所有需要排序的数据, 更加排序字段直接进行排序, 适用于单行数据不是特别大的时候. 如果是关联的表且所有排序的字段都在一个表中, 那么先对该表进行排序, 后进行关联. 否则的话就先进行关联, 在进行统一排序.

查询执行引擎: 在解析和优化阶段之后, MYSQL 将生成查询对应的执行计划, MYSQL 的查询执行引擎则根据这个执行计划来完成整个查询. 存储引擎中为每一个表实现了一个`handler 实例`, 执行计划则通过`handler API`接口进行存储访问.

最后将结果返回给客户端. 如果查询可以被缓存, 那么 MYSQL 会将结果放到缓存中.

## 参考文章

- [高性能MySQL(第3版)](https://book.douban.com/subject/23008813)
- [Mysql Api Doc](https://dev.mysql.com/doc/)
- [石杉的架构笔记-崔皓-了解MySQL查询语句执行过程](https://mp.weixin.qq.com/s/jg0Os6ZhiP7KwFd6LLkZ1A)
- [美团技术团队-MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)
