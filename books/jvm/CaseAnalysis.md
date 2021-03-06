# 调优案列分析与实战

## 高性能硬件上的程序部署策略

### 情况概述 1

在一个 15 万 PV(Page View 页面浏览量)/天左右的在线文档类型网站, 最近更换了硬件系统, 新的硬件为 4 个 CPU,16GB 物理内存, 操作系统为 64 位 CentOS5.4, Resin 作为 Web 服务器, 整个服务器暂时没有部署别的应用, 所有的硬件资源都可以提供给这访问量并不算太大的网站使用. 管理员为了尽量利用硬件选用了 64 位的 JDK1.5, 并通过-Xmx 和-Xms 参数将 Java 堆的大小固定为 12GB. 使用一段时间后发现使用效果并不理想, 网站经常不定期的出现长时间失去响应的情况.

### 初步分析 1

监控服务器运行状况后发现网站失去响应是由 GC 停顿导致的, 虚拟机运行在 Server 模式, 默认使用的是吞吐量优先收集器, 回收 12GB 的堆, 一次 Full GC 的停顿时间高达 14 秒, 并且由于程序设计的关系, 访问文档时需要把文档从磁盘提取到内存中, 导致内存中出现很多文档序列化产生的大对象, 这些大对象很多都进入了老年代, 没有在 Minor GC 中清理掉. 在这种情况下即使拥有 12GB 的堆, 内存也很快被消耗殆尽, 由此导致每隔十几分钟出现十几秒的停顿.

### 拓展思考 1

先不考虑程序代码的问题, 程序部署上的主要问题显然是过大的堆内存进行回收时带来的长时间停顿. 硬件升级前使用 32 位系统 1.5GB 的堆, 用户只感觉使用网站比较缓慢, 但不会发生十分明显的卡顿, 因此才考虑升级硬件以提升性能, 如果重新缩小给 Java 堆分配的内存, 那么硬件上的投资就显得很浪费.
在高性能硬件上部署程序, 目前有两种方式:

- 通过 64 位 JDK 来使用大内存
- 使用若干个 32 位虚拟机建立逻辑集群来利用硬件资源

这里, 我们的管理员使用的是第一种方式. 对于用户交互性强, 对停顿时间敏感的系统, 采用第一种方式时, 有一个前提就是: 有把握将应用程序的 Full GC 频率控制的足够的低, 如十几个小时一次乃至一天一次, 这样我们就可以通过深夜定时执行 Full GC 操作甚至重启应用服务来保持内存可用空间维持在一个稳定的水平上.
大多数的网站形式的应用中, 主要对象的生存周期都应该是请求级或者页面级, 会话级和全局级的长生命对象相对较少. 只要代码合理, 应当都能实现在超大堆中正常使用而没有 Full GC, 这样的话, 使用超大内存堆, 网站的响应速度才会比较有保证. 此外使用 64 位 JDK 来管理大内存, 还需要考虑以下问题:

- 内存回收导致的长时间停顿.
- 现阶段, 64 位 JDK 性能测试结果普遍低于 32 位 JDK.
- 需要保证程序足够稳定, 因为这种应用要是产生堆溢出, 几乎无法产生转储快照, 哪怕产生了快照, 也几乎没办法分析.
- 相同程序在 64 位 JDK 消耗的内存一般比 32 位 JDK 大, 由于指针膨胀, 数据类型对齐补白等因素导致的.

现阶段不少人采用第二种方式: 使用若干个 32 位虚拟机建立逻辑集群来利用硬件资源. 具体做法是: 在一台物理集群上启动多个应用服务器进程, 每个服务器进程分配不同的端口, 然后在前端搭建一个负载均衡器, 以反向代理的方式来分配访问请求.
考虑到一台物理机器上建立逻辑集群的目的仅仅是为了尽可能利用硬件资源, 并不需要关心状态保留, 热转移之类的高可用性需求, 也不需要保证每个虚拟机进程有绝对的负载均衡, 因此使用无 Session 复制的亲合式集群是一个相当不错的选择. 我们仅仅需要保证故障集群具备亲合性, 也就是均衡器按照一定的规则算法(一般根据 SessionID 分配)将一个固定用户的请求永远分配到固定的一个集群节点进行处理即可, 这样程序开发阶段也不用为集群环境做什么特别的考虑.
但是集群模式也有自己的缺点:

- 尽量避免节点竞争全局资源, 最典型的就是磁盘竞争, 各个节点如果同时访问某个磁盘文件的话,很容易导致 IO 异常.
- 很难最高效率地利用某些资源池, 如连接池, 一般每个节点都需要建立自己独立的连接池, 这样有可能导致一些节点池满了而另外一些节点仍有空余. 尽管可以使用集中式的 JNDI, 但是也会带来一定的复杂性和额外的性能开销.
- 每个节点仍然受到 32 位内存的限制, 在 32 位 Windows 平台只能使用 2GB 的内存, 考虑到堆以外的内存开销, 一般最大为 1.5GB, 在 Linux 系统中, 一般可以提升到 3GB 接近 4GB,但是仍然受到最高 4GB(2^32)内存的限制.

#### 解决方式 1

建立 5 个 32 位 JDK 的逻辑集群, 每个进程按 2GB 内存计算(堆固定为 1.5GB), 占用了 10GB 内存. 另外建立一个 Apache 服务作为前端均衡代理访问门户. 考虑到用户对响应速度比较关心, 并且文档服务的主要压力集中在磁盘和内存的访问, CPU 资源敏感度较低, 因此采用 CMS 收集器进行垃圾回收.

## 集群间同步导致的内存溢出

### 情况概述 2

一个基于 B/S 的 MIS 系统, 硬件为两台 2 个 CPU,8GB 内存的 HP 小型机, 服务器是 WebLogic 9.2, 每台机器启动了 3 个 WebLogic 实例, 构成一个 6 个节点的亲合式集群. 由于是亲合式集群, 节点之间没有进行 Session 同步, 但是有一些需求要实现部分数据在各个节点间共享. 开始这些数据存放在数据库中, 但是由于读写频繁竞争很激烈, 性能影响较大, 后面使用 JBossCache 构建了一个全局缓存. 全局缓存启用之后, 服务正常使用了一段时间之后, 但最近却不定期的出现多次内存溢出的问题.

### 初步分析 2

内存溢出异常不出现的时候, 服务器内存回收状况一直正常, 每次内存回收后都能恢复到一个稳定的可用空间, 开始怀疑是程序某些不常用的代码路径中存在内存泄漏, 但管理员反映最近程序并未更新或者升级过, 也没有什么特别操作. 只好让服务带着-XX:+HeapDumpOnOutOfMemeoryError 参数允许一段时间. 最近一次溢出之后, 管理员发回了 heapdump 文件, 发现里面存在着大量 org.jgroups.protocols.pbcast.NAKACK 对象.

### 拓展思考 2

JBossCache 是基于自家的 JGroups 进行集群间的数据通信, JGroups 使用协议栈的方式来实现收发数据包的各种所需特性自由组合, 数据包接受和发送时要经过每层协议栈的 up()和 down()方法, 其中的 NAKACK 栈用于保障各个包的有效顺序及重发. 由于消息有传输失败需要重发的可能性, 在确认所有注册在 GMS(Group Member Service)的节点都收到正确的信息前, 发送的消息必须在内存中保留. 而此 MIS 的服务端中有一个负责安全校验的全局 Filter, 每当接收到请求时, 均会更新一次最后操作时间, 且将这个时间同步到所有的节点去, 使得一个用户在一段时间内不能在多台机器上登录, 服务使用过程中, 往往一个页面会产生数次乃至数十次的请求, 因此这个过滤器导致集群各个节点之间网络交互非常频繁. 当网络情况不能满足传输要求时, 重发数据在内存中不断累积, 很快就导致内存溢出.

### 解决方式 2

这一类被集群共享的数据要使用类似 JBossCache 这种集群缓存来同步的话, 可以允许读操作频繁, 因为数据在本地内存有一份副本, 读取动作不会耗费多少资源, 但不应该有过于频繁的写操作, 那样会带来很大的网络同步的开销.

## 堆外内存导致的溢出错误

### 情况概述 3

一个学校的小型项目: 基于 B/S 的电子考试系统, 为了实现客户端能实时地从服务端接收考试数据, 系统使用了逆向的 Ajax 技术,选用了 CometD1.1.1 作为服务端推送框架, 服务器是 Jetty7.1.4, 硬件为一台普通的 PC 机 i5 CPU, 4GB 内存, 运行 32 位 Windows 操作系统. 测试区间发现服务端不定时抛出内存溢出异常, 服务器不一定每次都会出现异常. 但是如果正式考试时崩溃一次, 估计整场考试都会乱套.

### 初步分析 3

网站管理员尝试将堆开到最大 1.6GB, 但是发现开到最大基本没有效果, 好像还更加频繁了. 添加-XX:+HeapDumpOnOutOfMemoryError, 在内存溢出的时候, 居然没有什么文件产生. 无奈只好一直挂着 jstat 并一直紧盯着屏幕, GC 并不频繁, Eden 去,Survivor 区,老年代等都很正常, 但就是不停地抛出内存溢出异常. 最后在内存溢出系统日志中找到异常堆栈:

```java
[org.eclipse.jetty.util.log] handle failed java.lang.OutOfMemoryError: null
   at sun.misc.Unsafe.allocateMemory(Native Method)
    at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:99)
    at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:288)
    at org.eclipse.jetty.io.nio.DirectNIOBuffer.<init>
    ...
```

### 拓展思考 3

看到这里我们基本明白哪里出现问题了: 直接内存(Direct Memeory). 这台服务器使用的 32 位 Windows 平台的限制是 2GB, 其中划了 1.6GB 给 Java 堆, 而 Direct Memory 内存并不算入 1.6GB 的堆之内, 因此他最大也只能在剩余的 0.4GB 空间中分出一部分. 这里的出现内存溢出的情况是: 垃圾收集进行时, 虚拟机虽然会对 Direct Memory 进行回收, 但是 Direct Memory 却不能像新生代, 老年代那样, 发现空间不足了就通知收集器进行垃圾回收, 它只能等待老年代满了后 Full GC, 然后`顺便地`帮它清理掉内存的废弃对象. 不然它只能一直等到抛出内存溢出异常时, 先 catch 掉, 再在 catch 块中里面大喊一声`System.gc()`.要是虚拟机还是不听(比如虚拟机打开了-XX:+DisableExplicitGC), 那就只能眼睁睁地看着堆中还有许多空余内存, 自己不得不抛出内存溢出异常. 这个实例中使用 CometD1.1.1 框架, 正好有大量的 NIO 操作需要使用到 DirectMemeory 内存.
除了 Java 堆和永久代之外, 还有一些区域会占用较多的内存, 所以这里区域的内存总和收到操作系统进程最大内存的限制:

- Direct Memory: 可以通过-XX:MaxDirectMemorySize 调整大小, 内存不足时抛出 OutOfMemoryError 或者 OutOfMemoryError:Direct buffer memory.
- 线程堆栈: 可通过-Xss 调整大小, 内存不足时抛出 StackOverflowError(纵向无法分配, 即无法分配新的栈帧)或者 OutOfMemoryError: unable to create new native thread(横向无法分配, 即无法建立新的线程).
- Socket 缓存区: 每个 Socket 连接都 Receive 和 Send 两个缓存区, 分别占据 37KB 和 25KB 内存, 连接多的话这块内存占用也较为客观. 如果无法分配, 这可能会抛出 IOException: Too many open files 异常.
- JNI 代码: 如果代码使用 JNI 调用本地库, 那本地库使用的内存也不再在堆中.
- 虚拟机和 GC: 虚拟机, GC 的代码执行也需要消耗一定的内存.

## 外部命令导致系统缓慢

### 情况概述 4

一个数字校园应用系统, 运行在一台 4 个 CPU 的 Solaris10 的操作系统上, 中间件为 GlassFish 服务器. 系统在做大并发压力测试的时候, 发现请求响应时间较慢, 通过操作系统的 mpstat 工具发现 CPU 使用率很高, 并且系统占用绝大多数的 CPU 资源的程序不是应用系统本身. 这是一个不正常的现象, 通常情况下用户应用的 CPU 占有率应该占主要地位, 才能说明系统是正常工作的.

### 初步分析 4

通过 Solaris10 的 Dtrace 脚本可以查看当前情况下那些系统调用花费了最多的 CPU 资源, Dtrace 运行后发现最消耗 CPU 资源的居然是`fork`系统调用, 众所周知, `fork`调用时 Linux 用来创建新进程的, 在 Java 虚拟机中, 用户编写的 Java 代码最多只有线程的概念, 不应当有进程的产生.
通过本系统的开发人员, 最终找到了答案: 每个用户请求的处理都需要执行一个外部的 Shell 脚本来获得系统的一些信息. 执行这个 Shell 脚本呢是通过 Runtime.getRuntime().exec()方法来调用的. 这种调用方式可以达到目的, 但是它在虚拟机中是非常消耗资源的操作, 即使外部命令本身很快就可以执行完毕, 频繁调用时创建进程开销也非常可观. Java 虚拟机执行这个命令的过程是: 首先克隆一个和当前虚拟机拥有一样的环境变量的进程, 然后再用这个新的进程去执行这个外部命令, 最后退出这个进程. 如果频繁执行这个操作, 系统的消耗就会很大, 不仅是 CPU, 内存负担也很重.

### 解决方案 4

用户更加建议去掉这个 Shell 脚本, 改为使用 Java 的 API 去获取这些信息, 系统恢复正常.

## 服务器 JVM 进程崩溃

### 情况概述 5

一个基于 B/S 的 MIS 系统, 硬件为 2 台 2 个 CPU,8GB 内存的 HP 系统, 服务器为 WebLogic9.2(第二个实例, 那套系统). 正常运行一段时间后, 最近发现在运行区间频繁出现集群节点的虚拟机进程自动关闭的现象, 留下一个 hs_err_pid###.log 文件后, 进程就消失了, 两台物理机里的每个节点都出现过进程崩溃的现象. 从系统日志中, 可以看出, ,每个节点崩溃前不久, 都出现过大量相同的异常:

```java
java.net.SocketException: Connection reset
at java.net.SocketInputStream.read(SocketInputStream.java:168)
at java.io.BufferedInputStream.fill(BufferedInputStream.java:218)
at java.io.BufferedInputStream.read(BufferedInputStream.java:235)
at org.apache.axis.transport.http.HTTPSender.readHeadersFromSocket(HTTPSender.java:583)
at org.apache.axis.transport.http.HTTPSender.invoke(HTTPSender.java:143)
...
```

### 初步分析 5

这是远端断开连接的异常, 通过系统管理员了解到系统最近与一个 OA 门户做了集成, 在 MIS 系统工作流的代办事项变化时, 需要通过 Web 服务通知 OA 门户系统, 把代办事项的变化同步到 OA 门户之中. 通过 SoapUI 测试了一下同步待办事项的几个 Web 服务, 发现调用居然需要长达 3 分钟才能返回, 而且返回的结果都是连接中断.
由于 MIS 用户众多, 待办事项变化也很快, 为了不被 OA 系统速度所拖慢, 使用了异步的方式调用 Web 服务, 但由于两把服务速度不对等, 时间越长就累积越多 Web 服务没有调用完成, 导致在等待的线程和 Socket 连接越来越多, 最终超过了虚拟机的承受能力后使得虚拟机进程崩溃.

### 解决方法 5

通知 OA 门户方修复无法使用的集成接口, 并将异步调用改为生产者/消费者模式的消息队列实现, 系统恢复正常.

## 不恰当数据结构导致内存占用过大

### 情况概述 6

有一个后台 RPC 服务器, 使用 64 位虚拟机, 内存配置为-Xms4g(初始化内存为 4g) -Xmx8g(最大分配内存 8g) -Xmn1g(年轻代 1g), 使用 ParNew + CMS 的收集器组合. 平时对外服务的 Minor GC 时间约在 30 毫秒以内, 完全可以接受. 但是业务上需要每 10 分钟加载一个约 80MB 的数据文件到内存中进行数据分析, 这些数据会在内存总形成超过 100 万个 HashMap<Long, Long>Entry, 在这段时间里, Minor GC 就会造成超过 500 毫秒的停顿, 对这个停顿时间就接受不了了. 具体情况:

```java
(Heap before GC invocations = 95 (full 4) :
par new generation total 903168K [0x00002aaaae770000, 0x00002aaaebb70000, 0x00002aaaebb70000)
eden space 802816K, 100% used [0x00002aaaae77000, 0x00002aaadf770000, 0x00002aaadf770000)
from space 100352K, 0% used [0x00002aaae5970000, 0x00002aaae59c1910, 0x00002aaaebb70000)
to space 100352K, 0% use [....]
concurrent mark-sweep generation, total 5845540K, used 3898978K[...]
concurrent-mark-sweep perm gen total 65536K, used 40333K[...]
2011-10-28T11:40:45.162+0800:226.504: [GC 226.504: [ParNew: 803142K -> 100352K(903168K), 0.5995670secs] 4702120K->4056332K(6748708K), 0.5997560secs][Times: user=1.46 sys=0.04, real=0.60secs]
Heap after GC invocations = 96 (full 4):
par new generation total 903168K, used 100352K [...]
eden space 802816K, 0% used [...]
from space 100352, 100% used[...]
to space 100352, 0% used[..,]
concurrent mark-sweep generation total 5845540K, used 3955980K[...]
concurrent-mark-sweep perm gen total 65536K, used 4033K[...]
Total time for which application threads were stopped: 0.6070570 seconds.
```

### 初步分析 6

平时的 Minor GC 时间很短, 原因是新生代的对象都是可以清除的, 在 MinorGC 之后, Eden 和 Survivor 基本上就处于完全空闲的状态, 而在分析数据文件的期间, 800MB 的 Eden 空间很快被填满从而引发 GC, 但 MinorGC 之后, 新生代大部分的对象又是存活的. 我们知道 ParNew 收集器采用的是复制算法, 这个算法的基础是建立在对象大多是`朝生夕灭`的特性上, 如果存活数量过多, 就把这些对象复制到 Survivor 并维持这些对象的引用的正确就成为了一个沉重的负担, 因此导致 GC 暂停时间明显变长.

### 解决方式 6

如果不修改程序, 仅仅从 GC 调优的角度去解决这个问题, 可以考虑将 Survivor 空间去掉,(加入参数-XX:SurvivorRatio=65536(Eden 和 Survivor 区的比例), -XX:MaxTenuringThreshold=0(经过 MinorGC 次数,就进入老年区)或者-XX:+AlwaysTenure(等价)). 这个方法治标不治本, 治本的方法还是修改程序, 改变数据的存储方式. 80MB 的数据, 却占据了 800MB 的 Eden 区, 说明数据的储存效率很低:
HashMap<Long,Long>结构中, 只有 key 和 value 中存放的两个长整数是有效数据,(暂时不考虑 TreeNode 的存储情况,浪费更大), 共 16B(2 x 8B). 这两个长整型数据包装成 Long 对象, 分别添加了 8B 的 Mark Word 和 8B 的 Class 指针, 在加 8B 存储数据的 long 值. 两个 Long 对象组成的 Map.Entry 之后, 又添加了 16B 的对象头, 然后一个 8B 的 next 字段和 4B 的 int 型 hash 字段, 为了对齐, 还必须添加 4B 的空白补充, 最后还有 HashMap 中 table[]中存放着这个 Entry 的 8B 的引用. 实际消耗的内存: Long(24B) x 2 + Entry(32B) + Table Ref(8B) = 88B, 空间效率为: 16 / 88 = 18%. 是非常低的.

## 由 Windows 虚拟内存导致的长时间停顿

### 情况概述 7

有一个带心跳检测功能的 GUI 桌面程序, 每 15 秒会发送一次心跳检测信号, 如果对方 30 秒以内都没有信号返回, 那就认为和对方程序连接已经断开. 程序上线之后发现, 心跳检测有误报的情况, 查询日志发现误报的原因是程序会偶尔出现间隔约一分钟左右的完全无日志输出, 处于停顿状态.

### 初步分析 7

经过添加参数-XX:+PringGCApplicationStoppedTime -XX:+PrintGCDateStamps -Xloggc: gclog.log 后, 从 GC 日志文件中发现停顿确实是由 GC 导致, 大部分 GC 时间都控制在 100 毫秒以为, 但偶尔会出现 1 分钟的 GC.
从 GC 日志中找到了长时间停顿的具体日志信息(添加了-XX:+PrintReferenceGC 参数) 找到的日志片段如下所示. 从日志中可以看出, 真正执行 GC 动作的时间并不是很长, 但是准备开始 GC 到真正开始 GC 之间消耗的时间占据绝大部分. 观察到 GUI 程序内存发生变化的一个特点就是, 当它最小化的时候, 资源管理器显示占用内存大幅度减少, 但是虚拟内存并没有变化, 因此怀疑程序在最小化时, 它的工作内存被自动交换到磁盘的页面文件之中, 这样发生 GC 时就有可能因为恢复页面文件操作导致不正常的 GC 停顿.

### 解决方式 7

运行时添加`-Dsun.awt.keepWorkingSetOnMinimize=true`,可以得到解决.
