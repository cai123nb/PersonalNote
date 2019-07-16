# 晚期优化

## 概述

在部分商用虚拟机(Sun HotSpot, IBM J9)中, Java程序最初是通过解释器(Interpreter)进行解释执行的, 当虚拟机发现某个方法或代码块的运行特别频繁时, 就会把这些代码认定为`热点代码(Hot Spot Code)`. 为了提高热点代码的执行效率, 在运行时, 虚拟机就会把这些代码编译成与本地平台相关的机器码, 并进行各种层次的优化, 完成这个任务的编译器称为即时编译器(Just In Time Compiler).

即时编译器不是虚拟机必需的部分, Java虚拟机规范中也没有明确规定Java虚拟机必须要有即时编译器的存在, 更没有限定或指导即时编译器该如何实现. 但是即时编译器编译性能的好坏, 代码优化程度高低却是衡量一款商业虚拟机优秀与否的最关键的指标之一, 也是虚拟机最核心且最能体现虚拟机技术水平的部分. 由于没有确定的标准, 这里以HotSPot虚拟机的即时编译的实现进行深入了解.

## HotSpot虚拟机内的即时编译器

学习HotSpot虚拟机内的即时编译器的运作过程, 同时, 解决以下问题:
+ 为何HotSpot虚拟机要使用解释器与编译器并存的架构?
+ 为何HotSpot虚拟机要实现两个不同的即时编译器?
+ 程序何时使用解释器执行? 何时使用编译器执行?
+ 那些程序代码会被编译成本地代码? 如何编译成本地代码?
+ 如何从外部观察即时编译器的编译过程和编译结果?

### 解释器与编译器

尽管并不是所有的Java虚拟机都采用解释器和编译器并存的架构, 但许多商用虚拟机, 如HotSpot, J9(JRockit例外, 没有解释器), 都同时包含解释器和编译器. 解释器和编译器两者各有优势: 当程序需要迅速启动和执行的时候, 解释器可以首先发挥作用, 省去编译的时间, 立即执行. 在程序运行后, 随着时间的推移, 编译器逐渐发挥作用, 把越来越多的代码编译成本地代码之后, 可以获得更高的执行效率. 当程序运行环境资源限制较大(如嵌入式系统中), 可以使用解释器执行, 以解决执行节约内存, 反之可以使用编译执行来提升效率. 同时, 解释器还可以作为编译器激进优化时的一个`逃生门`, 让编译器根据概率选择一些大多数时候都能提升速度的优化手段, 当激进优化的假设不成立, 如加载了新类后类型继承结构发生变化, 出现`罕见陷阱(Uncommon Trap)`时可以通过`逆优化(Deoptimization)`退回到解释器继续执行(部分没有解释器的虚拟机中也会采用不进行激进优化的C1编译器担任`逃生门`的角色), 因此, 整个虚拟机执行架构中, 解释器与编译器经常配合工作.

HotSpot虚拟机中内置了两个即时编译器, 也称为Client Compiler和Server Compiler(简称C1编译器和C2编译器). 目前主流的HotSpot虚拟机中, 默认采用解释器与其中一个编译器直接配合的方式工作, 程序使用哪个编译器, 取决于虚拟机运行的模式, HotSpot虚拟机会根据自身版本与宿主机器的硬件性能自动选择运行模式, 用户可以使用`-client`或`-server`参数去强制指定虚拟机运行在Client模式或Server模式.
无论采用编译器Client Compiler还是Server Compiler, 解释器与编译器搭配使用的方式在虚拟机中称为`混合模式(Mixed Mode)`, 用户可以使用参数`-Xint`强制虚拟机运行于`解释模式(Interpreted Mode)`, 这时编译器完全不介入工作, 全部代码都使用解释方式执行. 另外, 也可以可使用`-Xcomp`强制虚拟机运行于`编译模式(Compiled Mode)`, 这时将优先采用编译方式执行程序, 但是解释器仍然在编译无法进行的情况下介入执行过程. 添加`-version`进行查看详细状态.

由于即时编译器编译本地代码需要占用程序运行时间, 要编译优化程度更高程度的代码, 所花费的时间可能更长. 而且想要编译出优化程度更高的代码, 解释器可能还要替编译器收集性能监控(Profiling)的信息, 这对解释的执行速度也有影响. 为了在程序启动响应速度与运行效率之间达到了最佳平衡, HotSpot虚拟机还会逐渐启用分层编译(Tiered Compilation)的策略, 分层编译的概念在JDK 1.6时期出现, 后台一直处于改进阶段, 最终在JDK1.7 的Server模式虚拟机中作为默认策略被开启. 分层编译根据编译器编译, 优化的规模和耗时, 划分不同的层次:
+ 第0层: 程序解释执行, 解释器不开启性能监控功能, 可触发第一层编译.
+ 第1层: 也称为C1编译, 将字节码编译编译为本地代码, 进行简单, 可靠的优化, 如有必要将要加入性能监控的逻辑.
+ 第2层: 也称为C2编译, 也就是字节码编译为本地代码, 但是会启用一些编译耗时较长的优化, 甚至根据性能监控信息进行一些不可靠的激进优化.

采用分层编译后, Client Compiler和Server Compiler将会同时工作,  许多代码都可能会被多次编译, 用Client Compier获取较高的编译速度, Server Compiler来获得更好的编译质量.

### 编译的对象和触发条件

在运行过程中, 会被即时编译器编译的`热点代码`有两类:
+ 被多次调用的方法
+ 被多次执行的循环体

前者很好理解, 一个方法被调用多了, 方法体内代码执行的次数自然就多, 它成为`热点代码`也是理所当然的. 而后者则是为了解决一个方法只被调用多次, 但是方法内部存在循环次数较多的循环体的问题, 这样循环体的代码也被重复执行多次, 因此这些代码也应该被认为是`热点代码`.

第一种情况, 由于方法触发的编译, 编译器理所当然地会以整个方法作为编译对象, 这种编译也是虚拟机中标准的JIT编译方式. 第二种情况, 经过编译动作是由循环体所触发, 但编译器依然会以整个方法(而不是单独的循环体)作为编译对象, 这种编译方式因为编译发生在方法执行过程中, 因此形象地称为栈上替换(On Stack Replacement, 简称OSR编译, 即方法栈帧还在栈上, 方法就被替换了).

判断一段代码是不是热点代码, 是不是需要触发即时编译, 这样的行为称为热点探测(Hot Spot Detection), 其实进行热点探测并不一定需要知道方法具体被调用了多少次. 目前主流的热点探测判定方式有两种, 分别是:
+ 基于采样的热点探测(Sample Based Hot Spot Detection): 采用这种方法的虚拟机会周期性地检查各个线程的栈项, 如果发现某个(或某些)方法经常出现在栈顶, 那这个方法就是`热点方法`. 基于采样的热点探测的好处就是简单高效, 可以很容易地获取方法的调用关系(展开堆栈即可), 缺点就是很难精确确认一个方法的热度, 容易受到线程堵塞或别的外界因素的影响而扰乱热点探测.
+ 基于计数器的热点探测(Counter Based Hot Spot Detection): 采用这种方法的虚拟机会为每一个方法(甚至是代码块)建立计数器, 统计方法的执行次数, 次数超过一定的阈值, 就标记为`热点方法`. 这种统计方法实现较为复杂, 需要为每一个方法建立和维护一个计数器, 而且不能直接获取到方法的调用关系, 但是统计结果更为精确和严谨.

在HotSpot中采用的是第二种, 基于计数器的热点探测方法, 因此它为每个方法准备了两类计数器: 方法调用计数器(Invocation Counter)和回边计数器(Back Edge Counter).

方法调用计数器, 即统计方法的调用次数, 在Client模式下默认阈值为1500次, 在Server模式下为10000次, 这个阈值可以通过-XX: CompileThreshold来人为设定. 工作模式为: 每当一个方法被调用, 首先检查是否存在被JIT编译的版本, 如果存在, 则优先使用编译后的本地代码进行执行. 如果不存在被编译的版本, 该方法的调用计数器值加1, 然后判断方法的调用计数器与回边计数器的阈值之和, 是否超过了方法调用计数器的阈值, 如果超过了, 提交该方法的编译请求. 默认不会等待编译完成, 而是通过解释器按照解释的方式进行执行, 直到提交的请求被编译器编译完成, 完成之后, 这个方法的调用入口地址将会被系统自动修改为新的调用地址. 下次调用默认使用该编译的版本.

如果不做设置, 方法调用计数器统计的并不是方法被调用的绝对次数, 而是一个相对的执行频率, 即一段时间之内方法的调用次数. 如果在这个时间限度之内, 方法的调用次数达不到阈值, 那么这个方法计数器就会减少一半. 这个过程称为`计数器热度的衰减(Counter Decay)`, 这段时间称为`方法统计的半衰期(Counter Half Life Time)`. 我们可以通过设置-XX: -UseCounterDecay来关闭热度衰减, 让方法计数器统计方法的绝对次数, 这样, 只要系统运行时间足够长, 绝大多数的方法都会被编译成本地代码. 另外, 可以使用-XX: CounterHalfLifeTime参数来设置半衰期的时间, 单位为秒.

回边计数器的阈值, 虽然HotSpot虚拟机也提供了一个类似的参数来进行设置: -XX: BackEdgeThreshold供用户进行设置, 但是当前虚拟机实际上并未使用该参数, 因此我们需要设置另一个参数-XX: OnStackReplacePercentage来设置回边计数器的阈值.
+ 在Client模式下, 回边计数器阈值计算公式为: 方法调用计数器阈值(CompileThreshold) x OSR比例(OnStackReplacePercentage) / 100. 其中OnStackReplacePercentage的默认值为933, 如果都取默认值, 那Client模式下的回边计数器的阈值为13995.
+ 在Server模式下, 计算公式为: 方法调用计数器阈值(CompileThreshold) x (OSR比例(OnStackReplacePercentage) - 解释器监控比率(InterpreterProfilePercentage)) / 100. OnStackReplacePercentage默认值为140, InterpreterProfilePercentage的默认值为33, 都取默认值的话, 在Server模式下的虚拟机回边计数器的阈值为10700.

回边计数器的工作模式为: 当解释器遇到一条回边指令时, 会先查找将要执行的代码片段是否有已经编译好的版本, 有, 则优先执行该版本代码, 否则, 回边计数器值加1, 然后判断方法调用计数器与回边计数器值之和, 是否超过回边计数器的阈值. 超过, 提交一个OSR编译请求, 并且把回边计数器的值降低, 以便继续在解释器中执行循环, 等待编译器输出编译结果. 与方法计数器的差别在于, 回边计数器是没有半衰期的, 该计数器统计的就是绝对的运行次数. 当该计数器溢出的时候, 它还会将方法计数器的值也调整到溢出状态, 这样下次进入该方法时就会执行标准的编译过程.

以上的工作模式都是在CLient模式下进行的流程, 在Server模式下要更加复杂一些. 我们这边查看一下MethodOop.hpp(一个MethodOop.hpp代表一个Java方法), 定义了方法在虚拟机内存中的布局.

```C++
// Note that native_function and signature_handler has to be at fixed offsets (required by the interpreter)
//
// |------------------------------------------------------|
// | header                                               |
// | klass                                                |
// |------------------------------------------------------|
// | constMethodOop                 (oop)                 |
// |------------------------------------------------------|
// | methodData                     (oop)                 |
// | interp_invocation_count                              |
// |------------------------------------------------------|
// | access_flags                                         |
// | vtable_index                                         |
// |------------------------------------------------------|
// | result_index (C++ interpreter only)                  |
// |------------------------------------------------------|
// | method_size             | max_stack                  |
// | max_locals              | size_of_parameters         |
// |------------------------------------------------------|
// |intrinsic_id|   flags    |  throwout_count            |
// |------------------------------------------------------|
// | num_breakpoints         |  (unused)                  |
// |------------------------------------------------------|
// | invocation_counter                                   |
// | backedge_counter                                     |
// |------------------------------------------------------|
// |           prev_time (tiered only, 64 bit wide)       |
// |                                                      |
// |------------------------------------------------------|
// |                  rate (tiered)                       |
// |------------------------------------------------------|
// | code                           (pointer)             |
// | i2i                            (pointer)             |
// | adapter                        (pointer)             |
// | from_compiled_entry            (pointer)             |
// | from_interpreted_entry         (pointer)             |
// |------------------------------------------------------|
// | native_function       (present only if native)       |
// | signature_handler     (present only if native)       |
// |------------------------------------------------------|
```

这里我们可以用清楚的看到: invocation_counter,  backedge_counter, from_compiled_entry, from_interpreted_entry .

### 编译过程

无论是方法调用产生的即时编译请求, 还是OSR编译请求, 虚拟机在代码编译器还未完成之前, 都仍然将按照解释的方式进行, 而编译动作则是在后台的编译线程中进行. 用户可以通过参数: -XX: -BackgroundCompilation来禁止后台编译, 即执行线程向虚拟机提交编译请求后, 将会一直等待, 直到编译完成在开始执行编译器输出的本地代码.

在编译过程中, 编译器做了哪些事呢? Server Compiler和Client Compiler两个编译器的编译过程是不一样的. 对于Client Compiler来说, 它是一个简单快速的三段式编译器, 主要的关注点是在于局部性的优化, 而放弃了许多耗时较长的全局优化手段.

![show](https://image.cjyong.com/blog/jvm/27.png)

第一阶段: 一个平台独立的前端将字节码构造成一个高级中间代码表示(HighLevel Intermediate Respresentation, HIR). HIR采用静态单分配(Static Single Assignment,SSA)的形式来表示代码值, 这样使得可以在HIR构造之中和之后进行某些优化操作: 空值检查消除, 返回检查消除等等. 在此之前需要进行一些基础的优化如: 方法内联, 常量传播等等.

第二阶段: 一个平台无关的后端从HIR中产生低级中间代码表示(Low-Level Intermediate Representation, LIR).

最后一个阶段: 在一个平台无关的后端使用线性扫描算法(Linear Scan Register Allocation)在LIR上分配寄存器, 并在LIR上做窥孔优化(Peephole), 然后产生机器代码.

而在Server Compiler则是专门面对服务端的典型应用, 并为服务端的性能配置特别调整过的编译器, 也是一个充分优化过的高级编译器, 几乎可以达到GNU C++编译器使用-O2参数时的优化强度, 它会执行所有的优化动作: 无用代码消除(Dead Code Elimination), 循环展开(Loop Unrolling), 循环表达式外提(Loop Expression  Hoisting), 消除公共子表达式(Common Subexpression Elimination), 常量传播(Constant Propagation), 基本块重排序(Base Block Reordering)等, 还会实施一些与Java语言特性相关的优化技术, 如范围检查消除(Range Check Elimination), 空值检查消除(Null Check Elimination)等等. 另外, 还可能根据解释器或Client Compiler提供的性能监控信息, 进行一些不稳定的激进优化, 如守护内联(Guarded Inlining), 分支频率预测(Branch Frequency Prediction)等等. Server Compiler的寄存器分配器是一个全局着色分配器, 它可以充分利用某些处理器架构(如RISC)上的大寄存器集合. 以即时编译的角度来看, Server Compiler无疑是缓慢的, 但是它的编译熟读依然远远超过了传统的静态优化编译器, 而且相对于Client Compiler编译输出的代码质量有所提高, 减少了本地代码的执行时间, 从而抵消了额外的编译时间. (很多非服务端的应用选择使用Server模式的虚拟机运行)

### 查看和分析即时编译的结果

```java
public class Test {
    public static final int NUM = 15000;

    public static int doubleValue(int i) {
        for (int j = 0; j < 100000; j++);
        return i * 2;
    }

    public static long calcSum() {
        long sum = 0;
        for (int i = 1; i <= 100; i++) {
            sum += doubleValue(i);
        }
        return sum;
    }

    public static void main(String[] args) {
        for (int i = 0; i < NUM; i++) {
            calcSum();
        }
    }
}
```

我们在虚拟机选项中添加: -XX:+PrintCompilation. 就可以查看触发了即时编译的代码(部分):

```java
180   32       3       java.lang.AbstractStringBuilder::append (29 bytes)
180   30  s    3       java.lang.StringBuffer::append (13 bytes)
183   33 %     3       com.cjyong.jvm.compiler.Test::doubleValue @ 2 (18 bytes)
183   34       3       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)
183   35 %     4       com.cjyong.jvm.compiler.Test::doubleValue @ 2 (18 bytes)
184   33 %     3       com.cjyong.jvm.compiler.Test::doubleValue @ -2 (18 bytes)   made not entrant
184   36       4       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)
185   34       3       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)   made not entrant
185   37       3       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)
186   38 %     4       com.cjyong.jvm.compiler.Test::calcSum @ 4 (26 bytes)
187   39       4       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)
189   37       3       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)   made not entrant
```

我们添加: -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining, 查看内联信息.

```java
226   29  s    3       java.lang.StringBuffer::append (13 bytes)
                          @ 7   java.lang.AbstractStringBuilder::append (29 bytes)
                            @ 7   java.lang.AbstractStringBuilder::ensureCapacityInternal (16 bytes)
                              @ 12   java.lang.AbstractStringBuilder::expandCapacity (50 bytes)   callee is too large
228   32 %     3       com.cjyong.jvm.compiler.Test::doubleValue @ 2 (18 bytes)
228   33       3       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)
228   34 %     4       com.cjyong.jvm.compiler.Test::doubleValue @ 2 (18 bytes)
229   32 %     3       com.cjyong.jvm.compiler.Test::doubleValue @ -2 (18 bytes)   made not entrant
229   35       4       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)
230   33       3       com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)   made not entrant
230   36       3       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)
                          @ 12   com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)   inlining prohibited by policy
231   37 %     4       com.cjyong.jvm.compiler.Test::calcSum @ 4 (26 bytes)
                          @ 12   com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)   inline (hot)
233   38       4       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)
                          @ 12   com.cjyong.jvm.compiler.Test::doubleValue (18 bytes)   inline (hot)
234   36       3       com.cjyong.jvm.compiler.Test::calcSum (26 bytes)   made not entrant
```

我们将hsdis-amd64.dll放入到我们jre/bin/server目录下, 使用+PrintAssembly参数打印方法的汇编代码(部分):

```java
Loaded disassembler from D:\Base\Java\jre\bin\server\hsdis-amd64.dll
Decoding compiled method 0x0000000005551790:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Constants]
  # {method} {0x000000001e090488} '<init>' '()V' in 'java/lang/Object'
  #           [sp+0x60]  (sp of caller)
  0x00000000055518e0: mov    0x8(%rdx),%r10d
  0x00000000055518e4: shl    $0x3,%r10
  0x00000000055518e8: cmp    %rax,%r10
  0x00000000055518eb: jne    0x0000000005495f60  ;   {runtime_call}
  0x00000000055518f1: data16 data16 nopw 0x0(%rax,%rax,1)
  0x00000000055518fc: data16 data16 xchg %ax,%ax
[Verified Entry Point]
...
```

除此之外我们可以添加-XX:PrintIdealGraphFile用于输出本地代码的生成的具体步骤. 但是需要debug版本的VM, 这里暂时没办法实现. 略过.

## 编译优化技术

### 优化技术概览

![show](https://image.cjyong.com/blog/jvm/28.png)

![show](https://image.cjyong.com/blog/jvm/29.png)

这里以一个简单的Java例子来展示优化的过程, 优化前的原始代码:

```java
static class B {
	int value;
    final int get() {
    	return value;
    }
}

public void foo() {
	y = b.get();
    //...do stuff...
    z = b.get();
    sum = y + z;
}
```

首先需要知道的是, 代码优化变换时建立在代码的某种中间表示或机器码之上, 绝不是建立在Java源码之上的. 这里为了展示方便, 使用Java的语法来展示这些优化技术所产生的作用.

第一步进行方法内联(Method Inlining), 方法内联的重要性远远高于其他优化措施: 1, 去除方法的调用成本(如新建栈帧等), 二是为其他优化建立良好的基础, 如内联膨胀之后, 可以在更大范围上采取后续的优化手段, 从而获得更好的优化效果. 因此一般方法内联都是放在优化序列的最靠前的位置. 内联后的代码:

```java
public void foo() {
	y = b.value();
    //... do stuff...
    z = b.value();
    sum = y + z;
}
```

第二步进行冗余访问消除(Redundant Loads Elimination), 如果//... do stuff...中没有对value值进行修改, 那就可以把z = b.value替换为z = y, 这样就可以不同访问b的局部变量. 如果把b.value看做是一个表达式, 那也可以把这项优化看成是公共子表达式消除(Common Subexpression Elimination), 优化后的代码:

```java
public void foo() {
	y = b.value;
    //...do stuff
    z = y;
    sum = y + z;
}
```

第三步进行复写传播(Copy Propagation), 因为在这段程序的逻辑中没有必要使用一个额外的变量z, 因为它和变量y是完全相等的, 因此可以使用y来替代z. 优化后的代码:

```java
public void foo() {
	y = b.value;
    //...do stuff
    y = y;
    sum = y + y;
}
```

第四步我们进行无用代码消除(Dead Code Elimination), 无用代码是永远不会被执行的代码, 也可能是完全没有意义的代码. 比如说上面代码中的y = y, 就是没有意义的. 优化的代码:

```java
public void foo() {
	y = b.value;
    //...do stuff
    sum = y + y;
}
```

这里查看几项具有代表性的优化技术:
+ 语言无关的经典优化技术之一: 公共子表达式消除
+ 语言相关的经典优化技术之一: 数组范围检查消除
+ 最重要的优化技术之一: 方法内联
+ 最前沿的优化技术之一: 逃逸分析

#### 公共子表达式消除

如果一个表达式E已经计算过了, 并且从先前的计算到现在的E中所有变量的值都没有发生变化, 那么E的这次出现就成为了公共子表达式. 对于这种表达式, 没有必要重新计算, 直接使用之前的计算的结果替代即可. 如果这种优化局限于程序内的基本块, 便称为局部公共子表达式消除(Local Common Subexpression Elimination), 如果这种优化的范围涵盖了多个基本块, 那就称为全局公共子表达式消除(Global Common Subexpression Elimination). 这里用一个简单的例子:

```java
int d = (c * b) * 12 + a + (a + b * c);
```

我们使用Javac进行编译, 那么生成的字节码将不会进行任何优化:

```java
//a,b,c分别存放在第2,3,4个局部变量中
iload_3
iload_2	
imul
bipush 12
imul
iload_1
iload_2
iload_3
imul
iadd
iadd
istore 4
```

当这段代码进入了虚拟机的即时编译器之后, 它将进行如下优化: 编译器检测到`c*b`与`b*c`是一样的表达式, 而且在计算期间b与c的值是不变的. 那么, 这条表达式可能会被视为:

```java
int d = E * 12 + a + (a + E);
```

编译器还可能进行另一种优化, 代数化简(Algebraic Simplification):

```java
int d = E * 13 + a * 2;
```

#### 数组边界检查消除

数组边界检查消除(Array Bounds Checking Elimination)是即时编译器中的一项语言相关的经典优化技术. 我们知道Java语言是一门动态安全的语言, 对数组的读写访问不像C/C++那样在本质上是裸指针操作. 如果有一个数组foo[], 在Java语言中访问数组元素foo[i]是系统将自动进行上下界的范围检查, 即:i必须满足i>=0 && i<foo.length这个条件, 否则就抛出运行时异常: java.lang.ArrayIndexOutOfBoundsException.  这对于软件开发者来说是好事, 即使没有专门编写防御代码, 也可以避免大部分的溢出攻击. 对于拥有大量数组访问的程序代码来说, 这无疑是一个性能上的负担.

无论如何, 为了安全, 肯定是要进行检查的. 但是是不是必须要在运行期间一次不漏的进行检查则是可以`商量`的. 如数组的下标为常量foo[3], 编译期根据数据流来确定foo.length的值, 发现下标不会超过该值, 则执行时就无须进行判断. 或者在循环中进行数组访问, 只要编译器通过数据流分析, 判断循环变量的值永远在数组范围之内, 那么也不用进行检查了. 这种思路为: 把数组边界优化尽可能的提到编译期进行完成.

除此之外, 还有一种思路就是: 隐私异常处理. Java中的空指针和算术运算中的除数为零的检查都采用了这种思路. 以Java伪代码的形式表示虚拟机访问foo.value过程如下:

```java
if (foo != null) {
	return foo.value;
} else {
	throw new NullPointException();
}
```

隐式异常优化之后, 虚拟机会把上面的访问过程变为:

```java
try {
	return foo.value;
} catch(segment_fault) {
	uncommon_trap();
}
```

虚拟机将会注册一个Segment Fault信号的异常处理器(uncommon_trap()), 这样foo不为空的时候, 就不用对foo进行额外的一次判空校验. 代价就是, 如果foo为空, 就必须转到异常处理器中恢复并抛出NullPointException异常, 这个过程必须从用户态转到内核态处理, 结束后在回到用户态, 速度远比一次判空慢. 如果foo经常为空, 这反而会让程序变慢, 如果foo极少为空, 这是值得的. 不过这不用我们担心, HotSpot足够`聪明`, 它会根据运行期收集的Profile信息自动选择最优方案.

类似的消除操作还有: 自动装箱消除(Autobox Elimination), 安全点消除(Sofepoint Elimination), 消除反射(Dereflection)等等.

#### 方法内联

方法内联是编译器最重要的优化手段之一, 除了消除方法调用成本之外, 它更重要的意义是为其他优化手段建立良好的基础. 如

```java
public static void foo(Object obj) {
	if (obj != null) {
    	System.out.println("do something");
    }
}

public static void testInline(String[] args) {
	Object obj = null;
    foo(obj);
}
```

如果不做内联, 我们会发现两个方法都是正常的, 即使我们进行无用代码消除, 也无法发现任何`dead code`. 但是如果我们进行内联, 我们会发现testInline方法内, 全部代码都是无用的代码, 可以被消除.

方法内联的优化行为看起来特别简单, 只不过将目标方法的代码`复制`到调用该方法的方法体中, 从而避免方法的调用. 但是实际上Java虚拟机中的内联远远没有这么简单: 大多数的Java方法都没办法进行内联. 从之前的章节我们可以知道, Java方法中只有invokestatic和invokespecial指令调用的静态方法, 私有方法, 实例构造器,父类方法是在编译期进行解析的, 除了这些方法, 其他的Java方法都需要在运行时进行方法接收者的多态选择, 并且都有可能出现多于一个版本的方法接收者. 简而言之, Java语言中默认的实例方法是虚方法, 即没办法在编译期确定执行对象和方法.

为了解决虚方法的内联问题, Java虚拟机团队想了很多方法, 首先是引入了一种名为`类型继承关系分析(Class Hierarchy Analysis, CHA)`的技术: 基于整个应用程序的类型分析技术, 它用于确定在目前已加载的类中, 某个接口是否有多于一种的实现, 某个类是否存在子类, 子类是否是抽象类等信息. 在编译器进行内联的时候, 如果是非虚方法, 直接进行内联即可, 这时候的内联是稳定的. 如果是虚方法, 则会向CHA查询当前程序下是否有多个目标版本可供选择, 如果查询结果只有一个, 那可以进行内联, 不过这种内联属于激进优化, 需要预留一个`逃生门(Guard条件不成立时的Slow Path)`,称为守护内联(Guarded Inlining). 如果程序后续执行过程中, 虚拟机一直没有加载到会令这个方法接收者的继承关系发生变化的类, 那么这个内联优化代码将会一直使用下去. 但是如果加载了导致继承关系发生变化的新类, 那就必需抛弃已经编译的代码, 退回到解释执行或者重新编译.

如果CHA查询结果有多个版本的目标方法可选, 则编译器还将会进行最后一次努力, 使用内联缓存(Inline Cache)来完成方法内联, 建立一个在目标方法正常入口之前的缓存, 工作原理: 在未发生方法调用的时候, 内联缓存为空, 当第一次调用之后, 缓存记录下方法接收者的版本信息, 并且每次调用方法时都比较接收者的版本, 如果以后进来的接收者版本一致, 那这个内联还可以一直用下去. 如果发生了版本不一致的情况, 就说明程序真正使用了虚方法的多态特性, 这时就会取消内联, 查找虚方法表进行方法分派.

所以对于多数的虚拟机来说, 进行内联是一种激进优化, 激进优化的手段在高性能的商用虚拟机中很常见, 除了内联之外, 对于出现概率很小(性能监控信息来确定)的隐式异常, 使用概率很小的分支等都可以被激进优化`移除`, 如果真的出现了小概率事件, 才会从`逃生门`回到解释状态重新执行.

#### 逃逸分析

逃逸分析(Escape Analysis)是目前Java虚拟机中比较前沿的技术, 它与类型关系分析一样, 并不是直接优化代码的手段, 而是为其他优化手段提供依据的分析技术. 逃逸分析的基本行为就是分析对象的动态作用域: 当一个对象在方法中被定义后, 它可能被外部方法引用, 例如作为调用参数传递给其他方法中, 称为方法逃逸. 甚至还可能被外部线程访问到, 如将值赋值给类变量或可以在其他线程中访问的实例变量, 称为线程逃逸.

如果证明了一个对象不会逃逸到方法或线程之外, 也就是别的方法或线程无法通过任何方法访问到这个对象, 就可以对该对象进行一些高效的优化:
+ 栈上分配(Stack Allocation): 在Java虚拟机中, 几乎所有的对象都是在Java堆中进行分配. Java堆中的对象对于各个线程都是共享和可见的, 只要持有这个对象的引用, 就可以访问堆中存储的对象数据. 虚拟机垃圾收集系统也是回收堆中不可使用的对象, 但是无论回收对象还是筛选可回收对象, 还是回收和整理内存都是需要耗费时间的. 如果确定一个对象不会逃逸出方法之外, 那让这个对象在栈上分配内存将会是一个很不错的选择.  对象所占用的内存空间就可以随着栈帧出栈而销毁. 一般应用中, 不会逃逸的局部对象占据的比例很大, 如果使用栈上分配, 那大量的对象就会随着方法的结束而被销毁, 垃圾收集系统的压力也会小很多.
+ 同步消除(Synchronization Elimination): 线程同步本身就是一个耗时的过程, 如果逃逸分析确定一个变量不会逃逸出线程, 无法被其他线程访问, 那这个变量的读写就肯定不会有竞争, 就可以取消掉对该变量的同步措施.
+ 标量替换(Scalar Replacement): 标量(Scalar)是指一个数据没有办法在进行分解成更小的数据表示了, 如Java虚拟机中的原始类型(int, long, reference等)都不能进行分解了, 它们可以称为标量. 相对的, 如果一个数据可以继续分解, 那它就称作聚合量(Aggregate). 在Java中的对象就是典型的聚合量, 如果把一个Java对象拆散, 根据程序的访问情况, 将其使用到的成员变量恢复到原始类型来访问就叫做标量替换. 逃逸分析获知一个对象不会被外部访问, 并且这个对象可以被拆散, 那么程序真正执行的时候, 就可以不创建这个对象, 而是改为直接创建它的若干个被方法使用的成员变量来替换. 将对象拆分后, 除了可以让对象的成员变量在栈上进行分配(栈上分配的数据, 有很大的概率被分配到物理机器的高速寄存器中进行存储), 还可以为后续进一步的优化创造条件.

逃逸分析的论文在1999年就已经发表了, 但直到JDK1.6才实现, 而知道现在这项优化也还没有完全成熟. 不成熟的原因主要是不能保证逃逸分析的性能收益必定大于它的消耗. 如果要完全准确地判断一个对象是否会逃逸, 需要进行数据流敏感的一系列复杂分析, 从而确定程序各个分支执行时对此对象的影响. 这是一个高耗时的过程, 如果分析完发现没有几个不逃逸的对象, 那这些分析所耗用的时间就白白浪费了, 所以目前虚拟机只能选择不那么准确的, 时间压力较小的算法来完成逃逸分析. 另外, 由于基于逃逸分析的一些优化手段, 如`栈上分配`, 由于HotSpot虚拟机的实现方式导致栈上分配实现起来较为复杂, 因此HotSpot还没实现该项优化.

如果有需要, 并且确认对程序有益, 可以使用-XX:+DoEscapeAnalysis手动打开逃逸分析, 开启之后可以通过-XX:+PrintEscapeAnalysis来查看分析结果. 开启逃逸分析之后, 可以通过-XX:+EliminateAllocations来开启标量替换, -XX:+EliminateLocks来开启同步消除,-XX:+PrintEliminateAllocations查看标量替换的情况.

逃逸分析虽然现在还不成熟, 但是未来肯定是编译器优化技术的一个重要发展方向.

## Java与C/C++的编译器对比

Java与C/C++(下面简称C)的编译器对比实际上代表了最经典的即时编译器与静态编译器的对比, 很大程度上也决定了Java和C的性能对比结果, 因为C还是Java代码最终编译之后, 被机器执行的都是本地机器码, 那个语言性能更高, 除了自身API库实现的好坏之外, 其余的比较就变成了一场`拼编译器`和`拼输出代码质量`的游戏.

Java虚拟机的即时编译器与C/C++的静态优化编译器相比, 可能由于以下原因导致输出的本地代码存在一些劣势:
+ 因为即时编译器运行占用的是用户程序的运行时间, 具有很大的时间压力, 它能提供的优化手段也严重受制于编译成本, 如果编译速度不能达到要求, 那用户将在启动程序或程序的某部分觉察到重大延迟, 这点使得即时编译器不敢随便引入大规模的优化技术, 而编译时间成本在静态优化编译器并不是主要的关注点.
+ Java语言是动态的类型安全语言, 这就意味着需要由虚拟机来确保程序不会违反语言语义或访问非结构化的内存. 从实现的角度来看, 虚拟机必须频繁的进行动态检查, 如空指针, 数组上下界, 类型转换等等. 虽然编译器会努力优化, 但是总体上还是要消耗不少运行时间.
+ Java语言虽然没有virtual关键字, 但是使用虚方法的频率要远大于C中,这就意味着运行时对方法接收者进行多态选择的频率要远远大于C语言, 也意味着即时编译器在进行一些优化(如方法内联)时难度要远大于C的静态优化编译器.
+ Java语言是动态拓展语言, 运行时新加载的类可能改变程序类型的继承关系, 这使得很多全局优化很难进行, 因为编译器没办法看见程序的全貌, 很多全局优化的措施都只能以激进优化的方式来完成, 编译器不得不时时刻刻的关注类型变化, 而重新进行一些优化.
+ Java语言中对象的内存分配都是在堆上进行的, 只有方法的局部变量才是在栈上进行分配. 而C的对象, 则是有多种内存分配方式, 即可能在堆上分配, 也可能在栈上分配, 可以减轻内存回收的压力. 另外, C中主要由用户程序代码来回收分配的内存, 这就不存在无用对象的筛选问题了, 因此效率上也比垃圾收集机制高.

上面这些性能上的劣势都是为了换取开发效率的优势而付出的代价, 动态安全, 动态拓展, 垃圾回收等等为Java语言的开发效率住处了很大的贡献. 并且Java的即时编译器可以做到很多C的静态优化编译器不能做或者不好做的事情.如: C中的别名分析的难度要远大于Java.

```java
void foo(ClassA objA, ClassB objB) {
	objA.x = 123;
    objB.y = 456;
    //只要objB.y不是objA.x的别名, 就可以保证输出为123
    print(objA.x);
}
```

确定了objA和objB并非对方的别名之后, 很多优化才能进行(重排序, 变量代换等等). 如上面这个例子, 就不用担心, objA.x和objB.y是同一个内存区域.

另外, Java虚拟机的即时编译还可以根据动态性进行运行时的优化, 而C编译器所有的优化都在编译期完成, 以运行期性能监控为基础的优化措施都无法进行, 如调用频率预测(Call Frequency Prediction), 分支频率预测(Branch Frequency Prediction), 裁剪未被选择的分支(Untaken Branch Pruning)等等, 这都是Java语言的独特性能优势.

