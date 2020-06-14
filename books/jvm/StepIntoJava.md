# 走近 JAVA

作为一个 Java 程序员, 在编写程序时除了尽情发挥 Java 的各种优势外, 还应该去了解和思考 Java 技术体系中这些技术是如何实现的, 认识这些技术运行的本质, 是自己去思考"程序这样写好不好"的基础和前提. 当我们在使用一种技术时, 不需要依赖书本或者他人就能得到这些问题的答案, 那才算上升到了"不惑"的境界.

## Java 技术体系

广义上说, Clojure, JRuby, Groovy 等运行在 Java 虚拟机上的语言及其相关的程序都属于 Java 技术体系中的一员. 传统意义来讲, Sun 官方所定义的 Java 技术体系包含以下几个组成部分:

- Java 程序语言设计
- 各种硬件平台上的 Java 虚拟机
- Class 文件格式
- Java API 类库
- 来自商业机构和开源社区的第三方 Java 类库 我们可以把 Java 程序设计语言, Java 虚拟机, Java API 类库这三部分统称为 JDK(Java Development Kit), JDK 是支持 Java 程序开发的最小开发环境. 另外, 把 Java API 类库这种的 Java SE 子集和 Java 虚拟机这两部分统称为 JRE(Java Runtime Environment), JRE 是支持 Java 程序运行的标准环境.

以上是根据组成部分功能来划分的, 如果按照技术所服务的领域来划分, 或者说按照技术所关注的重点业务领域来划分, Java 技术体系可以分为 4 个平台:

- Java Card: 支持一些 Java 小程序(Applets)运行在小内存设备(如智能卡)上的平台.
- Java ME(Micro Edition): 支持 Java 程序运行在移动终端(手机, PDA)上的平台, 对 Java API 有所精简, 并加入了移动终端的支持, 以前称为 J2ME.
- Java SE(Standard Edition): 支持面向桌面级应用(如 Windows 下的应用程序)的 Java 平台, 提供了完整的 Java 核心 API, 这个版本以前称为 J2SE.
- Java EE(Enterprise Edition): 支持使用多层架构的企业应用(如 ERP, CRM 应用)的 Java 平台, 除了提供 Java SE API 外, 还对其做了大量的扩充并提供了相关的部署支持, 这个版本以前称为 J2EE.

## Java 发展史

![show](https://image.cjyong.com/blog/jvm/2.jpg)

- 1991 年 4 月, James Gosling 博士领导的绿色计划(Green Project)开始启动, 此计划的目的是开发一种能够在各种消费性电子产品(如机顶盒, 冰箱, 收音机等)上运行的程序架构. 这个计划的产品就是 Java 语言的前身: Oak(橡树). 但在当时并不是很成功.
- 1995 年 5 月 23 号, Oak 改名 Java, 并在 SunWorld 大会上发布 Java1.0 版本. 第一次提出"WriteOnce, Run Anywhere"的口号.
- 1996 年 1 月 23 号, JDK1.0 发布, 提供了一个纯解释执行的 Java 虚拟机实现(Sun Classic VM). JDK1.0 代表技术: Java 虚拟机, Applet, AWT 等.
- 1996 年 4 月, 10 个最主要的操作系统供应商申明将在其产品中嵌入 Java 技术. 5 月底, Sun 公司在美国旧金山举办首届 JavaOne 大会.
- 1997 年 2 月 19 日, Sun 公司发布 JDK 1.1, 提供了 Java 技术的一些最基础的支撑点(如 JDBC 等), JDK1.1 版的技术代表有: JAR 文件格式, JDBC, JavaBean, RMI 等. 语法有了进一步的发展, 如内部类和反射.
- 1998 年 12 月 4 日, JDK 迎来了一个里程碑是的版本 JDK1.2, Sun 在这个版本中将 Java 技术体系拆分为 3 个方向, 分别是面向桌面应用开发的 J2SE(Java 2 Platform, Standard Edition), 面向企业开发的 J2EE(Java 2 Enterprise Edition) 和面向手机等移动终端开发的 J2ME(Java 2 Platform, Micro Edition), 第一次内置了 JIT 编译器.
- 1999 年 4 月 27 日, HotSpot 虚拟机(最初由一家名为"Longview Technologies"小公司开发, 后于 1997 年被 Sun 公司收购)发布. 作为 JDK1.2 的附加程序提供, 成为了 JDK1.3 及以后版本的 Sun JDK 的默认虚拟机.
- 2000 年 5 月 8 日, JDK1.3 发布, 对 JDK1.2 的类库提供了较大的改进(如数学运算, JNDI, JavaSound 等);
- 2002 年 2 月 13 日, JDK1.4 发布, 这是 Java 真正走向成熟的一个版本, 许多大公司(Compaq, Fujitsu, SAS, Symbian, IBM 等) 都参与开发甚至实现自己的独立的 JDK1.4 版本. 这个版本提供了很多技术特性, 如 NIO,正则表达式, 异常链, 日志类, XML 文件解析和 XSLT 转换器.
- 2004 年 9 月 30 日, JDK1.5 发布, JDK 首次在语法易用性上做出了非常大的改进, 如自动装箱, 泛型, 动态注解, 枚举, 可变长参数, 遍历循环(foreach)等, 虚拟机和 API 层面, 这版本改进了 Java 内存模型(JMM), 提供了 Java.util.concurrent 并发包.
- 2006 年 12 月 11 日, JDK1.6 发布, 提供了动态语言支持(通过内置 Mozilla javaScript Rhino 引擎实现), 提供编译 API 和微型 HTTP 服务器 API 等. 对 Java 虚拟机做了大量的改进, 包括锁和同步, 垃圾收集, 类加载等方面的算法.
- 2006 年 11 月 13 日的 JavaOne 大会中, Sun 公司宣布将 Java 开源, 并在随后的一年中, 陆续将 JDK 的各个部分在 GPL v2(GNU General Public License v2)协议下公开了源码.
- 2009 年 4 月 20 日, Oracle 公司宣布以 74 美元的价格收购 Sun 公司, Java 商标正式归 Oracle 所有.

## Java 虚拟机发展史

### Sun Classic / Exact VM

世界上第一款商用 Java 虚拟机, 1996 年 1 月 23 日, JDK1.0 发布的时候 JDK 所携带的虚拟机就是 Classic VM, 当时当前虚拟机只能使用[`纯解释器`](http://www.cnblogs.com/sword03/archive/2010/06/27/1766147.html)的方式执行 Java 代码, 如果想使用 JIT 编译器, 就必须进行外挂. 如果外挂了 JIT 编译器, JIT 编译器就完全接管了虚拟机的执行, 解释器就不在工作了. 如`sunjit`, `SymantecJIT`, `shuJIT`等. 由于二者不能配合工作, 这就意味着编译器就会对每一个方法, 每一行代码进行编译, 无论是否具备编译的价值. 基于程序响应时间的压力, 这些编译器**根本不敢**应用编译耗时稍高的优化技术, 因此这个阶段的虚拟机即使使用了 JIT 编译器输出本地代码, 执行效率也和传统的 C/C++程序有很大的差别, "Java 语言很慢"就是在这个时候开始在用户心中树立起来了. Sun 的虚拟机团队努力去解决 ClassicVM 所面临的各种问题, 提升运行效率, 在 JDK1.2 时, 曾在 Solaris 平台上发不了一款名为 Exact VM 的虚拟机, 它的执行系统已经具备现代高性能虚拟机的雏形: 如两级即时编译器, 编译器与解释器混合工作模式等. Exact VM 使用准确式内存管理(Exact Memory Management)而得名: 虚拟机可以知道内存中的某个位置的数据具体是什么类型, 如内存中有一个 32 位的整数 123456, 它到底是一个 reference 类型指向 123456 内存的地址还是一个 32 位的整数 123456, 虚拟机有能力分辨出来, ExactVM 抛弃了 Classic VM 基于 handle 的对象查找方式. 虽然 ExactVM 相对 ClassicVM 来说先进了很多, 但是只存在了很短的时间便被 HotSpotVM 所取代. 而 ClassicVM 在 JDK1.2 之前都是唯一的虚拟机, 在 JDK1.2 时, 与 ClassicVM 并存, 在 JDK1.3 时作为`备选虚拟机`,在 JDK1.4 中退出了虚拟机舞台.

### Sun HotSPot VM

它是 Sun JDK 和 Open JDK 中所带的虚拟机, 目前世界上使用最广泛的 Java 虚拟机. 最早的时候, 这是由"Longview Technologies"的小公司所设计. 这个虚拟机甚至最初并非是为 Java 语言设计的, 它来源于 Strongtalk VM, 而这款虚拟机中相当多的技术又是来源于一款支持 Self 语言实现"达到 C 语言 50%以上的执行效率"的目标而设计的虚拟机. Sun 公司注意到这款虚拟机在 JIT 编译上有许多优秀的理念和实际效果, 于 1997 年收购了该公司. Exact VM 和 HotSpot 有许多相同的特性, 如 HotSpot 一开始也是使用的准确式内存管理, 同样使用热点代码探测功能: 可以通过执行计数器找出最具有编译价值的代码, 然后通知 JIT 编译器以方法为单位进行编译. 如一个方法被频繁调用, 或方法中有效循环的次数很多, 将会分别触发标准编译和 OSR(栈上替换)的编译动作. 通过编译器和解释器恰当的进行协同工作, 可以在最优化的程序响应时间与最佳的执行性能中取得平衡, 而且无需等待本地代码输出才能执行程序, 即时编译的时间压力也相对减少, 有助于引入更多的代码优化技术, 输出质量更高的本地代码.

### Sun Mobile-Embedded VM / Meta-Circular VM

#### KVM

Kilobyte Virtual Marchine: 强调简单,轻量, 高度可移植, 运行速度较慢. 在 Android, IOS 等智能系统出现之前曾得到非常广泛的应用.

#### CDC/CLDC HotSpot Implementation

Connected (Limited) Device Configuration, 在 JSR-139/JSR-218 规范中被广泛定义, 希望在手机, 电子书, PDA 等设备上简历同意的 Java 编程接口, 而 HI VM 和 CLDC-HI VM 则是它们的一组参考实现.

#### Squawk VM

Sun 公司开发, 运行于 Sun SPOT(Sun Small Programmable Object Technology, 一种手持 WiFI 设备), 也曾经用于 Java Card.

#### JavaInJava

Sun 公司在 1997 年-1998 年间研发的实验室性质的虚拟机, 试图以 Java 语言来实现 Java 语言的运行环境, 即所谓的 Meta-Circular, 必须运行在另外一个宿主虚拟机上, 内部没有 JIT 编译器, 代码只能使用解释模式进行执行.

#### Maxine VM

和 JavaInJava 类似, 几乎全部以 Java 代码实现, 只有用于启动 JVM 的加载器使用 C 语言进行编写. 具有先进的 JIT 编译器和垃圾收集器(没有解释器), 可在宿主模式或独立模式下执行.

### BEA JRockit/IBM J9 VM

BEA 公司在 2002 年从 Appeal Virtual Machines 公司收购的虚拟机, BEA 公司将 JRockit 发展为一款专门为服务器硬件和服务器端应用场景高度优化的虚拟机, 由于专注于服务器端应用, 它不太关注程序的启动速度, 因此 JRockit 内部不包含解析器实现, 全部代码使用即时编译器编译后执行. JRockit 的垃圾收集器和 MissionControl 服务套件等部分, 在众多 Java 虚拟机中处于领先水平. IBM J9 VM 是 IBM 助理发展的 Java 虚拟机, IBM J9 VM 原本是内部开发代号, 正式的名称是"IBM Technology for Java Virtual Machine", 简称 IT4J, 这个名字有些拗口, 不如 J9 普及. J9VM 最初由于 IBM Ottawa 实验室一个名为 SmallTalk 的虚拟机拓展而来, 当时这个虚拟机有一个 Bug 就是 8K 值定义错误引起的, 工程师花了很长的时间发现并解决了这个问题, 而后拓展开来的虚拟机就叫 J9 了. 与 JRockit 专注于服务器端应用不同, IBM J9 市场定位于 HotSPot 比较类似, 是从服务器端到桌面端全面考虑的多用途虚拟机, 主要作为 IBM 公司各类 Java 产品的执行平台, 主要市场是和 IBM 产品(如 IBM WebSphere 等)搭配,以及在 IBM Aix 和 z/OS 这些平台上部署 Java 应用.

### Azul VM / BEA Liquid VM

类似 HotSpot, JRockit, J9 在通用平台上有很好的性能, 然而像 Azul VM 和 BEA Liquid VM 这类特定硬件平台专有的虚拟机才是"高性能"的武器. Azul VM 是 Azul System 公司在 HotSpot 基础上进行大量改进, 运行于 Azul Systems 公司的专有硬件 Vega 系统上的 Java 虚拟机, 每个 Azul VM 实例都可以管理至少数十个 CPU 和数百个 GB 内存的硬件资源, 并提供在巨大内存范围内实现可控的 GC 时间的垃圾收集器, 为专有硬件优化的线程调度等优化特性. 2010 年, Azul System 公司开始从硬件转向软件, 发布自己的 Zing JVM, 在通用 x86 平台提供接近于 Vega 系统的特性. Liquid VM 是现在 JRockit VE(Virtual Edition), 它是 BEA 公司开发的, 可以直接运行在自家 Hypervisor 系统上的 JRockit VM 的虚拟化版本, Liquid VM 不需要操作系统支持, 或者说他自己本身实现了一个专用操作系统的必要功能, 如文件系统, 网络支持等. 由虚拟机越过操作系统直接控制硬件.

### Google Android Dalvik VM

Dalvik VM 是 Android 平台的核心组成部分之一, 它的名字来源于冰岛的一个名为 Dalvik 的小渔村. Dalvik VM 并不是一个 Java 虚拟机, 它没有遵循 Java 虚拟机的规范, 不能直接执行 Java 的 Class 文件, 使用寄存器架构而不是 JVM 中常用的栈架构. 但是他执行的 dex 文件, 可以由 class 文件转化而来, 使用 Java 语法编写应用程序, 可以直接使用大部分的 Java API 等.Android2.2 中, 已经提供即时编译, 在执行性能上有很大的提高.

## 展望 Java 未来

### 模块化

> JDK9, 似乎已经提供了支持.

模块化是解决应用系统与技术平台越来越复杂, 越来越庞大问题的一个重要途径. 用户或者开发人员, 都不希望为了系统中一小块功能而不得不下载,安装,部署及维护庞大的系统. 站在整个软件工业化的高度, 模块化是建立各种功能的标准件的前提.

### 混合语言

### 多核并行

### 64 位虚拟机

几年前主流的 CPU 就开始支持 64 位架构, Java 虚拟机也在很早之前就推出了支持 64 位系统的版本, 但 Java 程序运行在 64 位虚拟机上需要付出较大的额外代价: 首先内存问题, 由于指针膨胀和数据类型对齐补白的原因, 64 位系统上的 Java 应用需要消耗更多的内存(10% - 30%); 其次, 多个机构测试结果显示, 64 位虚拟机在运行速度在各个测试项中全面落后 32 位虚拟机, 约有 15%的性能差距.
