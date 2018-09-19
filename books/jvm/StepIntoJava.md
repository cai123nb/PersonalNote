# 走近JAVA
作为一个Java程序员, 在编写程序时除了尽情发挥Java的各种优势外, 还应该去了解和思考Java技术体系中这些技术是如何实现的, 认识这些技术运行的本质, 是自己去思考"程序这样写好不好"的基础和前提. 当我们在使用一种技术时, 不需要依赖书本或者他人就能得到这些问题的答案, 那才算上升到了"不惑"的境界.
## Java技术体系
广义上说, Clojure, JRuby, Groovy等运行在Java虚拟机上的语言及其相关的程序都属于Java技术体系中的一员. 传统意义来讲, Sun官方所定义的Java技术体系包含以下几个组成部分:
+ Java程序语言设计
+ 各种硬件平台上的Java虚拟机
+ Class文件格式
+ Java API类库
+ 来自商业机构和开源社区的第三方Java类库
我们可以把Java程序设计语言, Java虚拟机, Java API类库这三部分统称为JDK(Java Development Kit), JDK是支持Java程序开发的最小开发环境. 另外, 把Java API类库这种的Java SE子集和Java虚拟机这两部分统称为JRE(Java Runtime Environment), JRE是支持Java程序运行的标准环境.

![show](https://raw.githubusercontent.com/cai123nb/PersonalNote/master/JVMLearning/img/1.jpg)

以上是根据组成部分功能来划分的, 如果按照技术所服务的领域来划分, 或者说按照技术所关注的重点业务领域来划分, Java技术体系可以分为4个平台:
+ Java Card: 支持一些Java小程序(Applets)运行在小内存设备(如智能卡)上的平台.
+ Java ME(Micro Edition): 支持Java程序运行在移动终端(手机, PDA)上的平台, 对Java API有所精简, 并加入了移动终端的支持, 以前称为J2ME.
+ Java SE(Standard Edition): 支持面向桌面级应用(如Windows下的应用程序)的Java平台, 提供了完整的Java核心API, 这个版本以前称为J2SE.
+ Java EE(Enterprise Edition): 支持使用多层架构的企业应用(如ERP, CRM应用)的Java平台, 除了提供Java SE API外, 还对其做了大量的扩充并提供了相关的部署支持, 这个版本以前称为J2EE.

## Java发展史

![show](https://image.cjyong.com/blog/jvm/2.jpg)

+ 1991年4月, James Gosling博士领导的绿色计划(Green Project)开始启动, 此计划的目的是开发一种能够在各种消费性电子产品(如机顶盒, 冰箱, 收音机等)上运行的程序架构. 这个计划的产品就是Java语言的前身: Oak(橡树). 但在当时并不是很成功.
+ 1995年5月23号, Oak改名Java, 并在SunWorld大会上发布Java1.0版本. 第一次提出"WriteOnce, Run Anywhere"的口号.
+ 1996年1月23号, JDK1.0发布, 提供了一个纯解释执行的Java虚拟机实现(Sun Classic VM). JDK1.0代表技术: Java虚拟机, Applet, AWT等.
+ 1996年4月, 10个最主要的操作系统供应商申明将在其产品中嵌入Java技术. 5月底, Sun公司在美国旧金山举办首届JavaOne大会.
+ 1997年2月19日, Sun公司发布JDK 1.1, 提供了Java技术的一些最基础的支撑点(如JDBC等), JDK1.1版的技术代表有: JAR文件格式, JDBC, JavaBean, RMI等. 语法有了进一步的发展, 如内部类和反射.
+ 1998年12月4日, JDK迎来了一个里程碑是的版本JDK1.2, Sun在这个版本中将Java技术体系拆分为3个方向, 分别是面向桌面应用开发的J2SE(Java 2 Platform, Standard Edition), 面向企业开发的J2EE(Java 2 Enterprise Edition) 和面向手机等移动终端开发的J2ME(Java 2 Platform, Micro Edition), 第一次内置了JIT编译器.
+ 1999年4月27日, HotSpot虚拟机(最初由一家名为"Longview Technologies"小公司开发, 后于1997年被Sun公司收购)发布. 作为JDK1.2的附加程序提供, 成为了JDK1.3及以后版本的Sun JDK的默认虚拟机.
+ 2000年5月8日, JDK1.3发布, 对JDK1.2的类库提供了较大的改进(如数学运算, JNDI, JavaSound等);
+ 2002年2月13日, JDK1.4发布, 这是Java真正走向成熟的一个版本, 许多大公司(Compaq, Fujitsu, SAS, Symbian, IBM等) 都参与开发甚至实现自己的独立的JDK1.4版本. 这个版本提供了很多技术特性, 如NIO,正则表达式, 异常链, 日志类, XML文件解析和XSLT转换器.
+ 2004年9月30日, JDK1.5发布, JDK首次在语法易用性上做出了非常大的改进, 如自动装箱, 泛型, 动态注解, 枚举, 可变长参数, 遍历循环(foreach)等, 虚拟机和API层面, 这版本改进了Java内存模型(JMM), 提供了Java.util.concurrent并发包.
+ 2006年12月11日, JDK1.6发布, 提供了动态语言支持(通过内置Mozilla javaScript Rhino引擎实现), 提供编译API和微型HTTP服务器API等. 对Java虚拟机做了大量的改进, 包括锁和同步, 垃圾收集, 类加载等方面的算法. 
+ 2006年11月13日的JavaOne大会中, Sun公司宣布将Java开源, 并在随后的一年中, 陆续将JDK的各个部分在GPL v2(GNU General Public License v2)协议下公开了源码.
+ 2009年4月20日, Oracle公司宣布以74美元的价格收购Sun公司, Java商标正式归Oracle所有.

##Java虚拟机发展史
### Sun Classic / Exact VM
世界上第一款商用Java虚拟机, 1996年1月23日, JDK1.0发布的时候JDK所携带的虚拟机就是Classic VM, 当时当前虚拟机只能使用[`纯解释器`](http://www.cnblogs.com/sword03/archive/2010/06/27/1766147.html)的方式执行Java代码, 如果想使用JIT编译器, 就必须进行外挂. 如果外挂了JIT编译器, JIT编译器就完全接管了虚拟机的执行, 解释器就不在工作了. 如`sunjit`, `SymantecJIT`, `shuJIT`等. 由于二者不能配合工作, 这就意味着编译器就会对每一个方法, 每一行代码进行编译, 无论是否具备编译的价值. 基于程序响应时间的压力, 这些编译器**根本不敢**应用编译耗时稍高的优化技术, 因此这个阶段的虚拟机即使使用了JIT编译器输出本地代码, 执行效率也和传统的C/C++程序有很大的差别, "Java语言很慢"就是在这个时候开始在用户心中树立起来了.
Sun的虚拟机团队努力去解决ClassicVM所面临的各种问题, 提升运行效率, 在JDK1.2时, 曾在Solaris平台上发不了一款名为Exact VM的虚拟机, 它的执行系统已经具备现代高性能虚拟机的雏形: 如两级即时编译器, 编译器与解释器混合工作模式等. Exact VM使用准确式内存管理(Exact Memory Management)而得名: 虚拟机可以知道内存中的某个位置的数据具体是什么类型, 如内存中有一个32位的整数123456, 它到底是一个reference类型指向123456内存的地址还是一个32位的整数123456, 虚拟机有能力分辨出来, ExactVM抛弃了Classic VM基于handle的对象查找方式. 虽然ExactVM相对ClassicVM来说先进了很多, 但是只存在了很短的时间便被HotSpotVM所取代. 而ClassicVM在JDK1.2之前都是唯一的虚拟机, 在JDK1.2时, 与ClassicVM并存, 在JDK1.3时作为`备选虚拟机`,在JDK1.4中退出了虚拟机舞台.
### Sun HotSPot VM
它是Sun JDK和Open JDK中所带的虚拟机, 目前世界上使用最广泛的Java虚拟机. 最早的时候, 这是由"Longview Technologies"的小公司所设计. 这个虚拟机甚至最初并非是为Java语言设计的, 它来源于Strongtalk VM, 而这款虚拟机中相当多的技术又是来源于一款支持Self语言实现"达到C语言50%以上的执行效率"的目标而设计的虚拟机. Sun公司注意到这款虚拟机在JIT编译上有许多优秀的理念和实际效果, 于1997年收购了该公司.
Exact VM和HotSpot有许多相同的特性, 如HotSpot一开始也是使用的准确式内存管理, 同样使用热点代码探测功能: 可以通过执行计数器找出最具有编译价值的代码, 然后通知JIT编译器以方法为单位进行编译. 如一个方法被频繁调用, 或方法中有效循环的次数很多, 将会分别触发标准编译和OSR(栈上替换)的编译动作. 通过编译器和解释器恰当的进行协同工作, 可以在最优化的程序响应时间与最佳的执行性能中取得平衡, 而且无需等待本地代码输出才能执行程序, 即时编译的时间压力也相对减少, 有助于引入更多的代码优化技术, 输出质量更高的本地代码.
### Sun Mobile-Embedded VM / Meta-Circular VM
#### KVM
Kilobyte Virtual Marchine: 强调简单,轻量, 高度可移植, 运行速度较慢. 在Android, IOS等智能系统出现之前曾得到非常广泛的应用.
#### CDC/CLDC HotSpot Implementation
Connected (Limited) Device Configuration, 在JSR-139/JSR-218规范中被广泛定义, 希望在手机, 电子书, PDA等设备上简历同意的Java编程接口, 而HI VM和CLDC-HI VM则是它们的一组参考实现.
#### Squawk VM
Sun公司开发, 运行于Sun SPOT(Sun Small Programmable Object Technology, 一种手持WiFI设备), 也曾经用于Java Card.
#### JavaInJava
Sun公司在1997年-1998年间研发的实验室性质的虚拟机, 试图以Java语言来实现Java语言的运行环境, 即所谓的Meta-Circular, 必须运行在另外一个宿主虚拟机上, 内部没有JIT编译器, 代码只能使用解释模式进行执行.
#### Maxine VM
和JavaInJava类似, 几乎全部以Java代码实现, 只有用于启动JVM的加载器使用C语言进行编写. 具有先进的JIT编译器和垃圾收集器(没有解释器), 可在宿主模式或独立模式下执行.

### BEA JRockit/IBM J9 VM
BEA公司在2002年从Appeal Virtual Machines公司收购的虚拟机, BEA公司将JRockit发展为一款专门为服务器硬件和服务器端应用场景高度优化的虚拟机, 由于专注于服务器端应用, 它不太关注程序的启动速度, 因此JRockit内部不包含解析器实现, 全部代码使用即时编译器编译后执行. JRockit的垃圾收集器和MissionControl服务套件等部分, 在众多Java虚拟机中处于领先水平.
IBM J9 VM 是IBM助理发展的Java虚拟机, IBM J9 VM 原本是内部开发代号, 正式的名称是"IBM Technology for Java Virtual Machine", 简称IT4J, 这个名字有些拗口, 不如J9普及. J9VM最初由于IBM Ottawa实验室一个名为SmallTalk的虚拟机拓展而来, 当时这个虚拟机有一个Bug就是8K值定义错误引起的, 工程师花了很长的时间发现并解决了这个问题, 而后拓展开来的虚拟机就叫J9了. 与JRockit专注于服务器端应用不同, IBM J9市场定位于HotSPot比较类似, 是从服务器端到桌面端全面考虑的多用途虚拟机, 主要作为IBM公司各类Java产品的执行平台, 主要市场是和IBM产品(如IBM WebSphere等)搭配,以及在IBM Aix和z/OS这些平台上部署Java应用.

### Azul VM / BEA Liquid VM
类似HotSpot, JRockit, J9在通用平台上有很好的性能, 然而像Azul VM和BEA Liquid VM这类特定硬件平台专有的虚拟机才是"高性能"的武器.
Azul VM是Azul System公司在HotSpot基础上进行大量改进, 运行于Azul Systems公司的专有硬件Vega系统上的Java虚拟机, 每个Azul VM实例都可以管理至少数十个CPU和数百个GB内存的硬件资源, 并提供在巨大内存范围内实现可控的GC时间的垃圾收集器, 为专有硬件优化的线程调度等优化特性. 2010年, Azul System公司开始从硬件转向软件, 发布自己的Zing JVM, 在通用x86平台提供接近于Vega系统的特性.
Liquid VM是现在JRockit VE(Virtual Edition), 它是BEA公司开发的, 可以直接运行在自家Hypervisor系统上的JRockit VM的虚拟化版本, Liquid VM不需要操作系统支持, 或者说他自己本身实现了一个专用操作系统的必要功能, 如文件系统, 网络支持等. 由虚拟机越过操作系统直接控制硬件.

### Google Android Dalvik VM
Dalvik VM是Android平台的核心组成部分之一, 它的名字来源于冰岛的一个名为Dalvik的小渔村. Dalvik VM并不是一个Java虚拟机, 它没有遵循Java虚拟机的规范, 不能直接执行Java的Class文件, 使用寄存器架构而不是JVM中常用的栈架构. 但是他执行的dex文件, 可以由class文件转化而来, 使用Java语法编写应用程序, 可以直接使用大部分的Java API等.Android2.2中, 已经提供即时编译, 在执行性能上有很大的提高.

## 展望Java未来
### 模块化
> JDK9, 似乎已经提供了支持.

模块化是解决应用系统与技术平台越来越复杂, 越来越庞大问题的一个重要途径. 用户或者开发人员, 都不希望为了系统中一小块功能而不得不下载,安装,部署及维护庞大的系统. 站在整个软件工业化的高度, 模块化是建立各种功能的标准件的前提.
### 混合语言

### 多核并行

### 64位虚拟机
几年前主流的CPU就开始支持64位架构, Java虚拟机也在很早之前就推出了支持64位系统的版本, 但Java程序运行在64位虚拟机上需要付出较大的额外代价: 首先内存问题, 由于指针膨胀和数据类型对齐补白的原因, 64位系统上的Java应用需要消耗更多的内存(10% - 30%); 其次, 多个机构测试结果显示, 64位虚拟机在运行速度在各个测试项中全面落后32位虚拟机, 约有15%的性能差距.
