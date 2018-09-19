# 虚拟机字节码执行引擎

## 概述
执行引擎是Java虚拟机最核心的组成部分之一. `虚拟机`是相对于`物理机`而言的, 两者都具备代码执行能力. 区别在于, 物理机的执行引擎是直接建立在处理器, 硬件, 指令集和操作系统的层面上的, 而虚拟机的执行引擎是由自己实现的, 可以自定义指定操作指令集与执行引擎的结构体系, 并且可以执行那些不被硬件直接支持的指令集格式.

Java虚拟机规范中制定了虚拟机字节码的执行引擎的概念模型, 这个概念模型成为各个虚拟机执行引擎的统一外观(Facade): 输入是字节码文件, 处理过程是字节码解析的等效过程, 输出的是执行结果. 但是内部实现在不同的虚拟机中是不一样的, 在不同的虚拟机的实现中, 执行引擎在执行java代码时候可能会有解释执行(通过解释器进行执行)和编译执行(通过即时编译器产生本地代码选择)或者两者兼备, 甚至还可能包含不同级别的编译器执行引擎.

## 运行时栈帧结构
栈帧(Stack Frame)是用于支持虚拟机进行方法调用和方法执行的数据结构, 它是虚拟机运行时数据区的虚拟机栈(Virtual Machine Stack)的栈元素. 栈帧存储了方法的局部变量表, 操作数栈, 动态链接和方法的放回地址等信息. 每一个方法从调用开始到执行完成的过程, 都对应着一个栈帧在虚拟机里面从入栈到出栈的过程.

每一个栈帧都包含局部变量表, 操作数栈, 动态链接, 方法的放回地址和一些额外的附加信息. 在编译程序代码的时候, 栈帧中需要多大的局部变量表, 多深的操作数栈都已经完全确定了, 并且写入到方法的Code属性之中, 因此每个栈帧需要分配多少内存, 不会受到运行时变量数据的影响, 而仅仅是取决于具体虚拟机的实现.

一个线程中的方法调用链可能很长, 很多方法都同时处于执行状态. 对于执行引擎来说, 在活动线程中, 只有位于栈顶的栈帧才是有效的, 称为当前栈帧(Current Stack Frame), 与这个栈帧相关联的方法称为当前方法(Current Method). 执行引擎运行的所有字节码都只针对当前栈帧进行操作.

### 局部变量表
局部变量表(Local Variable Table)是一组变量值存储空间, 用于存放方法参数和方法内部定义的局部变量. 在Java程序编译为Class文件时, 就在方法的Code属性中max_locals数据项确定了该方法所需要分配的局部变量表的最大容量.

局部变量表的容量以变量槽(Variable Slot)为最小单位, 虚拟机中没有明确表明一个Slot应占用的大小(可以更加不同的虚拟机实现, 来进行定制如32位虚拟机占据32位, 64位虚拟机占据64位来实现一个Slot, 但是要通过对齐和补白的方式填充空白), 而是很导向地说每个Slot都应该可以存放一个boolean, byte, char, short, int, float, reference或returnAddress. 这八种数据都可以通过32位或更小的物理内存进行存放.

对于1个Slot进行存放的数据类型: boolean, byte, char, short, int, float, reference和returnAddress. 前6中可以在Java中的方式进行理解, 而reference表示对一个对象实例的引用, 虚拟机规范没有说明它的长度, 也没有明确指出该引用该有什么结构. 但一般来说, reference至少可以完成以下两种功能:
+ 可以通过该引用直接或间接地查找到对象在Java堆中的数据存放的起始地址索引.
+ 此引用中直接或间接地查找到对象所属的数据类型在方法区中的存储的类型信息, 否则无法进行语法约束判断.

而returnAddress则是用的比较少, 为jsr, jsr_w和ret服务的, 指向一条字节码指令的地址, 很老之前用于异常处理, 现在已经被异常表所替代.
对于64位的数据类型, 虚拟机会以高位对齐的方式为其分配2个连续的Slot空间. Java中明确的数据类型只有Long和Double(reference不能确定).  由于局部变量表建立在线程的堆栈上, 是线程的私有数据, 无论读写两个连续的Slot是否为原子操作, 都不会引起数据安全的问题.

虚拟机通过索引定位的方式使用局部变量表, 索引值从0开始到局部变量表最大的Slot数量. 如果访问的是32位数据类型变量, 索引n就代表了使用第n个Slot. 如果是64位数据类型, 则会说明会同时使用n和n+1两个Slot, 不允许以任何方式单独访问其中的某一个, 如果发生该种情况, 虚拟机应该在类加载的校验阶段抛出异常.

方法执行时, 虚拟机通过局部变量表完成参数值到参数列表的传递过程. 如果执行的是非实例方法(非static), 局部变量表中的第0位索引的Slot默认是对象实例的引用, 可以通过`this`访问到该隐藏变量. 其余参数按照顺序进行排列, 占用从1开始的局部变量Slot, 参数分配完毕后, 在根据方法体内部定义的变量顺序和作用域分配其余的Slot.

为了节省栈帧空间, 局部变量表中的Slot是可以重用的, 方法体中定义的变量其作用域不一定会覆盖整个方法体, 如果当前字节码PC计数器的值已经超过了某个变量的作用域, 那么这个变量的Slot就可以给你其他变量进行使用. 但是这也会带来额外的副作用, 影响系统的垃圾收集操作:

```java
    public static void main(String[] args) {
        byte[] placeholer = new byte[64 * 1024 * 1024];
        System.gc();
    }
```

在虚拟机运行参数中添加: -verbose:gc, 输出结果:

```java
[GC (System.gc())  70779K->66256K(251392K), 0.0021884 secs]
[Full GC (System.gc())  66256K->66170K(251392K), 0.0075567 secs]
```

我们可以发现并没有回收64M的内存, 这应该是可以理解的, 因为在gc的时候, placeholder还在方法区的作用域内. 我们修改一下palceholder的作用域:

```java
    public static void main(String[] args) {
        {
            byte[] placeholer = new byte[64 * 1024 * 1024];
        }
        System.gc();
    }
```

这时候, 我们在gc的时候, palceholder应该已经超出了作用域, 没有人可以访问到palceholder, 按道理应该是要被回收的. 结果:

```java
[GC (System.gc())  70779K->66256K(251392K), 0.0019584 secs]
[Full GC (System.gc())  66256K->66170K(251392K), 0.0074940 secs]
```

我们发现并没有进行回收, 这就很神奇了, 我们添加一个新的代码:

```java
    public static void main(String[] args) {
        {
            byte[] placeholer = new byte[64 * 1024 * 1024];
        }
+       int a = 0;
        System.gc();
    }
```

我们再次运行, 发现:

```java
[GC (System.gc())  70779K->66288K(251392K), 0.0010001 secs]
[Full GC (System.gc())  66288K->634K(251392K), 0.0066372 secs]
```

placeholder的空间居然被回收了. 我们可以知道, placeholder是否可以被回收的关键是: 局部变量表中是否还存有palceholder对象的引用. 第一次修改, 虽然代码已经离开了placeholder的作用域, 但是在此之后, 没有任何的局部变量表的读写操作, placeholder原本的Slot空间还没被复用掉, 所以GC Roots一部分的局部变量仍然保持关联. 在大部分情况下, 这都是影响很小的. 但是如果你前面定义了占用大量内存, 而又不进行使用的变量, 我们可以进行设置为null(类似`int a = 0`), 进行局部变量表的复用, 清除关联.

另外还需要注意的是, 局部变量表是不会进行初始化赋值的, 如果使用为初始化的值, 编译器会进行报错. 前面所说的两次赋值: 准备阶段系统赋初值, 初始化阶段自定义赋值,是对于类变量而言的.

### 操作数栈
操作数栈(Operand Stack), 常称为操作栈, 一个后入先出(Last In First Out, LIFO)栈. 同局部变量表一样, 操作数栈的最大深度也在编译的时候写入到Code属性的max_stacks数据项中. 操作数栈的每一个元素可以是任意的Java数据类型. 32位数据类型占栈容量1, 64位数据类型占栈容量2. 方法执行过程中, 栈的深度都不会超过max_stacks数据项设定的值.

一个方法执行的时候, 操作栈是空的, 随着方法的进行, 不断的入栈和出栈, 如iadd指令就是取栈顶的两个元素出栈, 相加, 将计算结果入栈. 操作数栈中的元素的数据类型必须和字节码指令的序列严格匹配, 如iadd, 那么栈顶的两个元素就必须是int型, 不能是long或者float. 这一点在编译程序的时候需要进行校验, 类校验阶段的数据流分析中还要再次验证这一点.

在概念模型中, 两个栈帧作为虚拟机栈的元素是完全独立的, 但是在虚拟机的实现里都会进行一些优化处理, 让两个栈帧出现一部分重叠.让下面栈帧的部分操作数栈与上面的栈帧的部分局部变量重叠在一起, 实现参数的复制传递效果.

### 动态链接
每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用, 持有这个引用是为了支持方法调用过程中的动态链接(Dynamic Linking). 在字节码中的方法包含了指向常量池中的方法符号引用, 这部分的符号引用有一部分会在类加载阶段或者第一次使用时候就转化为直接引用, 这种转化成为`静态解析`. 另一部分将在每一次运行区间转化为直接引用, 这部分称为`动态链接`.

### 方法返回地址
当一个方法开始执行后, 有两种方式可以退出这个方法:
+ 正常完成出口(Normal Method Invocation Completion): 执行引擎遇到任意一个方法返回字节码指令, 根据字节码指令决定是否有返回值和返回值类型, 进行退出.
+ 异常完成出口(Abrupt Method Invocation Completion): 方法执行中出现异常, 而异常没有在方法体内得到处理, 导致方法的退出.

方法退出时, 需要返回到方法被调用的位置, 程序才能继续进行, 方法返回时需要在栈帧中保存一些信息, 用于恢复上层方法的执行状态, 如PC计数器的值(异常退出时, 可以用来确认异常出现的位置). 实际过程就等同于把当前栈帧进行出栈, 具体的操作: 恢复上层方法的局部变量表和操作数栈, 把返回值(如果有的话)压入调用者栈顶的操作数栈, 跳转PC计数器的值使其指向方法调用指令的后面一条指令.

### 附加信息
虚拟机规范中允许具体的虚拟机实现增加一些规范里没有的描述信息到栈帧中, 如调试相关的信息. 在实际开发中, 动态链接, 方法返回地址与其他附加的信息归为一类, 称为栈帧信息.

## 方法的调用
方法的调用不是方法的执行, 而是单纯地确定调用方法的版本. CLass文件的编译过程中不包含传统编译中的连接步骤, 一起方法调用在Class文件里面存储的都只是符号引用, 而不是方法在实际运行时的内存入口(直接引用), 这个特性给Java带来了动态拓展能力, 也使得Java的方法调用过程变得相对复杂起来, 需要在类的加载区间, 甚至在运行区间才能确定目标方法的直接引用.

### 解析
所有方法在调用中的目标方法在Class文件里面都是一个指向常量池的符号引用, 在类加载的解析阶段, 会将其中一部分符号引用直接转换为直接引用. 但是这有一个前提: 这些方法在运行期间是不可变的, 即调用目标在程序代码写好, 编译器进行编译时就必须确定下来, 不会进行修改. 这类方法的调用称为解析(Resolution).

满足`编译期可知, 运行期不变`这个要求的方法, 在Java中有, 静态方法和私有方法. 前者直接与类型关联, 后者在外部不可访问, 保证了二者不会被继承等其他方式进行重写. 与之对应的Java虚拟机提供了5条方法的指令:
1. invokestatic: 调用静态方法
2. invokespecial: 调用实例的构造器方法(<init>), 私有方法和父类方法.
3. invokevirtual: 调用所有的虚方法.
4. invokeinterface:  调用接口方法, 会在运行时在确定一个实现此接口的对象.
5. invokedynamic: 先在运行时动态解析出调用点限定符所引用的方法, 然后在执行该方法, 之前4条指令, 分派逻辑都是固化在Java虚拟机内部, 而invokedynamic是由用户所设定方法来决定的, 即把方法的分派权给了用户.

所以只要是使用invokestatic或者invokespecial指令调用的方法(静态, 构造器函数, 私有, 父类方法), 都可以在解析阶段来确定唯一的调用版本. 在类加载的时候就会把符号引用解析为该方法的直接引用, 这些方法称为`非虚方法`, 与之相反, 其他方法称为`虚方法`. 这里有一个例外, 就是使用final修饰的方法, 虽然也是使用invokevirtual进行调用的, 但是也是唯一确定的, 不可以被修改也是`非虚方法`.

### 分派
解析调用是一个静态的过程, 在编译期就完全确定, 在类加载的解析阶段就会把涉及的符号引用全部转变为可确定的直接引用, 不会延迟到运行期在完成. 而分派(Dispa)调用可能是静态的, 也可能是动态的, 根据分派依据的`宗量数`可分为单分派和多分派, 两两组合为: 静态单分派, 静态多分派, 动态单分派, 动态多分派.

### 静态分派

```java
public class StaticDispatch {
	static abstract class Human {
    
    }
    
    static class Man extends Human {
    
    }
    
    static class Woman extends Human {
    
    }
    
    public void sayHello(Human guy) {
    	System.out.println("hello, guy!");
    }
    
    public void sayHello(Man guy) {
    	System.out.println("hello, gentleman!");
    }
    
    public void sayHello(Woman guy) {
    	System.out.println("hello, lady!");
    }
    
    public static void main(String[] args) {
    	Human man = new Man();
      	Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man); //hello, guy
        sr.sayHello(woman); //hello, guy
    }
}
```

这里我们可以清楚的看到区别, `Human man = new Man()`中`Human`为变量的静态类型, 或者叫做外观类型`Apparent Type`, 后面的`Man`被称为变量的实际类型`Actual Type`. 静态类型和实际类型都可以发生变化, 区别在于静态类型的最终类型是可以在编译期可知的, 而实际类型变化结果是在运行期才可知的.如:

```java
Human man = new Man(); //实际类型的变化
sr.sayHello((Man) man); //静态类型的变化
```

在main函数里面的两次调用sayHello()函数, 在方法的接收者已经确定为sr的前提下, 使用哪个重载版本是由参数的数量和数据类型来进行确定的. 代码中使用两个静态类型相同而实际类型不同的变量, 但是虚拟机(准确的来说是编译器)是通过参数的静态类型而不是实际类型作为判断依据的, 并且参数的静态类型在编译期是可知的.

所有依赖静态类型来定位方法执行版本的分派动作称为静态分派. 静态分派最典型的引用就是方法重载, 静态分派发生在编译阶段, 因此确定静态分派动作实际上不是由虚拟机来进行确定的, 而是由编译器进行确定的, 并且有时候编译器并不能`完全确定`是哪个重载版本, 而是通过优先级来确定`更加适合`的版本.

```java
package com.cjyong.jvm.garbage;

/**
 * Created with IntelliJ IDEA By
 * User: cjyong
 * Date: 2018/1/7
 * Time: 19:48
 * Description:
 */
public class Overload {

    public static void sayHello(Object arg) {
        System.out.println("hello Object");
    }

    public static void sayHello(int arg) {
        System.out.println("hello int");
    }

    public static void sayHello(long arg) {
        System.out.println("hello long");
    }

    public static void sayHello(Character arg) {
        System.out.println("hello Character");
    }

    public static void sayHello(char arg) {
        System.out.println("hello char");
    }

    public static void sayHello(char... arg) {
        System.out.println("hello char ...");
    }
    
    public static void sayHello(Serializable arg) {
        System.out.println("hello Serializable");
    }

    public static void main(String... args) {
        sayHello('a');
    }

}
```

直接运行, 结果为: `hello char`. 这很好理解, 'a'为cha类型的数据, 自然寻找参数类型为char的重载方法. 如果我们注释掉sayHell(char arg), 输出为: `hello int`, 这里发生了一次类型转换, 'a'除了可以代表字符串, 还可以代表数字97, 因此参数类型为int的重载也是合适的, 我们继续注释掉该函数, 这时候的输出变成了`hello long`, `a`转型为97之后, 进一步转型为97L, 匹配了参数为long的重载, 这里还会继续转型下去(char -> int -> long -> float -> double), 但是不会匹配到byte和short, 因为char到byte和short的转换是不安全的. 继续注释掉该函数, 结果变成: `hello Character`, 这里进行了装箱操作, 'a'被包装成了java.lang.Character, 所以匹配到了Character的重载, 我们继续注释, 结果变成了: `hello Serializable`, Serializable是Character实现的一个接口, 如果自动装箱之后还没找到, 但是找到了装箱类实现的接口类型, 所以又发生了一次转型. 查看Character的源码, 其还实现了Comprable<Character>接口, 如果同时出现Serializatble和Comparable<Character>的重载方法, 那么它们的优先级是一样的, 编译器无法判断就会拒绝编译. 用户必须显示指定转换的的静态类型如`sayHello((Comparble<Character>'a'))才能通过编译. 我们再次注释掉该方法, 结果变为: `hello Object`, char装箱之后转型为了父类, 如果有多个父类, 那将在继承关系中从下向上开始搜索. 最后我们将该方法注释, 结果变成: `hello char ...`, 可见变长参数的重载优先级是最低的, 这时候`a`被当做一个数组元素.

静态方法会在类加载期就进行解析, 而静态方法显然也是可以拥有重载版本的, 选择重载版本的过程也是通过静态分派实现的.

### 动态分派
动态分派和重写(Override)有着很密切的联系.

```java
public class DynamicDispatch {
    static abstract class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello(); //man say hello
        woman.sayHello(); //woman say hello
        man = new Woman();
        man.sayHello(); //woman say hello
    }
}
```
这很正常, 符合我们对Java的认识, 但很显然这里没有使用静态分派. 那么虚拟机又是如何确定该调用谁的sayHello()函数?

我们查看一下, 编译之后的字节码文件:

![show](https://image.cjyong.com/blog/jvm/24.png)

+ 0: 新建一个Man对象, 进行默认的初始化, 并将该对象的引用置入栈顶.
+ 3: dup, 拷贝栈顶对象, 现在栈顶有两个Man对象引用.
+ 4: 取出栈顶对象, 作为构造函数的, this参数传入, 进行构造.
+ 7: 将栈顶对象Man对象的引用存储到第二个本地变量. 这时候栈清空了.
+ 8,11,12,15类似略过.
+ 16: 加载本地第二个本地变量到栈顶, 即Man对象的引用.
+ 17: 调用栈顶对象的sayHello()方法,
+ 余下类似略过.

这里主要讲解下invokevirtual运行时解析过程:
1. 找到操作数栈顶的第一个元素所指向的对象的实际类型(注意不是静态类型), 记作C.
2. 如果在类型C中找到与常量中的描述符合简单名称都相符的方法, 进行权限校验, 如果通过则返回这个方法的直接引用, 查找结束. 不然抛出IllegalAccessError异常.
3. 否则, 按照继承关系从下向上依次对C的父类进行第2步的查找过程.
4. 如果始终没找到合适的方法, 抛出AbstractMethodError异常.

invokervirtual第一步就是在运行期确定接收者的实际类型, 所以invokervirtual指令把常量池中的类方法符号引用解析到了不同的直接引用上了. 这就是重写的本质. 把这种在运行期根据实际类型确定方法的执行版本的分派过程称为`动态分派`.

### 单分派与多分派
根据分派基于多少种总量, 可以将分派分为单分派和多分派两种. 单分派是根据一个宗量对目标方法进行选择, 多分派则是根据多于一个宗量对目标方法进行选择.

```java
public class Dispatch {

    static class QQ{}

    static class _360{}

    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("father choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360()); //father choose 360
        son.hardChoice(new QQ()); //son choose qq
    }
}
```

静态分派阶段, 即编译器的选择过程, 这时候依据有两点: 静态类型是Father还是Son, 而是方法参数是QQ还是360. 导致产生两个invokervirtual指令, 2条指令的参数分别为Father.hardChose(360)及Father.hardChose(QQ).因为是根据两个宗量进行选择, 所以Java的静态分派是多分派类型.

运行时虚拟机的选择阶段, 动态分派的过程. 在执行`son.hardChose(new QQ))`的时候, 即在执行invoervirtual指令时, 由于编译期已经决定了方法的签名必须为hardChoice(QQ), 虚拟机不关心传递过来的参数`QQ`是`QQ`还是它的子类, 应为这时参数的静态类型和实际类型都不会对方法的选择产生影响, 唯一影响虚拟机的选择的是接受者的实际类型, 是Father还是Son. 因为只有一个总量作为选择的依据, 所以Java语言动态分派属于单分派类型.

### 虚拟机动态分派的实现

通过前面的介绍, 我们已经明白了虚拟中在动态分派中该`做什么`. 但是`怎么做`在不同的虚拟机中可能存在着实现的差异. 由于动态分派是非常频繁的动作, 并且动态分派的方法版本选择过程需要运行时在类的方法元数据中搜索合适的目标和方法, 因此虚拟机一般都不会进行如此频繁的搜索, 而是通过一个为类在方法区建立一个虚方法表(Virtual Method Table, 简称vtable, invokerinterface也会存在一个Inteface Method Table, 简称itable), 表里面存在着对应的方法的直接引用地址. 如果子类重写了父类的某一个方法, 就将地址修改为直接的实现地址即可.

![show](https://image.cjyong.com/blog/jvm/25.png)

为了程序实现上的方便, 具有相同签名的方法, 在父类, 子类的虚方法表中都应当具有一样的索引号, 这样当类型变换时, 只需要变更查找的方法表, 就可以从不同的虚方法中按索引转换出所需的入口地址.

方法表一般在类加载的连接阶段进行初始化, 准备了类的变量初始化之后, 虚拟机会把该类的方法表也初始化完毕. 方法表只是分派调用的`稳定优化`的手段, 除此在外, 在条件允许的情况下, 虚拟机还会使用`内联缓存(Inline Cache)`和基于`类型继承关系分析(Class Hierarchy Analysis, CHA)技术的守护内联(Guarded Inlining)`两种非稳定的`激进优化`, 来获得更高的性能.

### 动态语言的支持
随着JDK1.7的到来, 字节码指令集终于迎来了第一位新成员: invokedynamic指令, 这条指令是JDK 7实现`动态类型语言(Dynamically Typed Language)` 支持而进行的改进之一, 也是为JDK 8可以顺利实现Lambda表达式的技术准备.

#### 动态类型语言
什么是动态类型语言? 动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期, 满足这个特征的语言有很多, 如: APL, Clojure, Erlang, Groovy, JavaScript, Jyphon, Lisp, Lua, PHP等. 相对的, 在编译期就进行类型检查过程的语言(如C++和Java等)就是最常用的静态类型语言.

首先我们先看一下, 什么是`在编译期/运行期进行` ?

```java
public static void main(String[] args) {
	int[][][] array = new int[1][0][-1];
}
```

这个代码可以正常编译, 但是在运行时会报NegativeArraySizeException异常. 这是一个运行时异常, 通俗点来说就是, 只要代码不运行到这一行就不会有问题. 与运行时异常相对应的是连接时异常, 如NoClassDefFoundError便是属于连接时异常, 类加载时(Java的连接过程不在编译阶段, 而在类加载阶段) 也照样会抛出异常.

但是在C语言中, 含义相同的代码会在编译期报错:

```c
int main(void) {
	int i[1][0][-1]; //GCC拒绝编译, 报"size of array is negative".
    return 0;
}
```

因此我们可以看出, 一门语言的哪一种检查行为要在运行期进行, 哪一种是要在编译期进行并没有必然的因果逻辑关系, 关键是语言规范中的认为规定.

这里在来看一个`类型检查`的例子:

```java
obj.println("hello world");
```

假设这是在Java语言中, 并且obj的静态类型为java.io.PrintStream, 那么变量obj的实际类型就必须是PrintStream的子类(实现了PrintStream接口的类)才是合法的. 否则, 哪怕obj属于一个确定有用print(String)方法, 但是和PrintStream接口没有继承关系, 代码仍然不可能运行, 因为类型检查不合法.

但是如果这是在ECMAScript(JavaSrcipt)中情况则不一样, 无论obj的具体类型是何种类型, 只要这种类型的定义确实包含println(String)方法, 那方法调用便可以成功. 

这种差别产生的原因是Java语言在编译期间已将println(String)方法的完整的符号引用(CONSTANT_IntefaceMethodref_info常量)生成出来, 作为方法调用指令的参数,存储到Class文件中, 如:
```java
invokevirtual #4 //Method java/io/PrintStream.println: (Ljaga/lang/String;)V
```

该符号引用已经包含了此方法定义在那个具体的类型中, 方法的名称以及参数的顺序, 参数的类型和方法的返回值等信息. 通过这个符号引用, 虚拟机可以翻译出这个方法的直接引用. 而在ECMAScript等动态语言中, 变量obj本身是没有类型的, 变量obj的值才具有类型, 编译时最多只能确定方法名称, 参数和返回值这些信息, 而不会去确定方法所在的具体类型(即方法的接受者不固定). 即`变量无类型而变量值才有类型`.

静态类型语言和动态类型语言, 各有优缺点. 静态类型语言: 在编译期确定类型, 最显著的好处是编译器可以提供严谨的类型检查, 这样与类型相关的问题就可以在编码的时候及时发现, 利于稳定性和代码规模的扩大. 动态类型语言提供了更大的灵活性, 实现某些功能显得更加清晰和简洁.

#### JDK1.7与动态类型
Java虚拟机不仅仅是Java语言的运行平台, 为了支持更多的动态类型语言可以运行在Java虚拟机上, 如Clojure,  Groovy, Jython和JRuby等, 能够在一个虚拟机上可以达到静态类型语言的严谨性与动态类型语言的灵活性, 这是非常美妙的. 

Java虚拟机层面上对动态类型语言的支持一直是有所欠缺的, 主要表现在方法的调用方面. 在JDK1.7之前, 只有4条方法的调用指令(invokervirtual, invokespecial, invokerstatic, invoerinterface),这四条调用指令的第一个参数都是调用方法的符号引用(CONSTANT_Methodref_info), 就是在编译时产生定死了. 为了支持动态类型语言只有在运行期才能确定接收者类型, 在Java虚拟机上实现的动态语言不得不采用别的方法(如编译时留个占位符, 运行时动态生成字节码实现具体类型到占位符类型的适配)来实现, 这样势必让动态类型语言实现复杂增加, 也带来了额外的性能和内存开销. 因此在Java虚拟机层面上提供对动态类型语言的支持就成了Java虚拟机平台发展的趋势之一. 这也是invokedynamic指令以及java.lang.invoke包出现的技术背景.

#### java.lang.invoke包
JDK1.7新加入的java.lang.invoke包就是在这个技术背景下出现的, 主要目的就是在之前单纯依靠符号引用来确定调用目标方法这种方式之外, 提供一种新的动态确定目标的机制, 称为MethodHandle. 

```java
public class MethodHandleTest {
    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable{
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();

        getPrintlnMH(obj).invokeExact("cjyong");
    }

    private static MethodHandle getPrintlnMH(Object receiver) throws Throwable {
        //确定方法的类型, 第一个参数为返回值, 后续参数为方法的参数
        MethodType mt =  MethodType.methodType(void.class, String.class);
        //调用MethodHandles的lookup()方法, 查找对应实例方法的实现,
        //第一参数为要寻找的类, 第二个参数为方法的简易名称, 第三个参数为方法类型
        //按照虚拟机的实现, 实例方法的第一个参数为隐式的自己本身(虚拟机中通过参数列表进行传递), 
        // 这里通过bindTo方法进行传递.
        return MethodHandles.lookup().findVirtual(receiver.getClass(),
                "println",
                mt).bindTo(receiver);
    }
}
```

仔细看这个方法, 或许有的人会觉得反射不是已经可以完成这个功能了吗? 是的, MethodHandle和Reflection有很多相同的地方, 但是它们之间还是有很大的区别的:
+ 本质上来讲, Reflection和MethodHandle机制都是在模拟方法调用, 但Reflection是在模拟Java代码层次的方法调用(只能为Java语言服务), 而MethodHandle是在模拟字节码层次的方法调用(可以为任何运行在Java虚拟机上的语言服务). 在MethodHandles.lookup中的三个方法: findStatic(), findVirtual, findSpecial正好对应invokestatic, invokevirtual&invokerinterface, invokerspecial这几条字节码指令的执行权限行为, 而这些底层细节Reflection API时是不需要关心的.
+ Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息多. 前者是方法在Java一端的全面映像, 包含了方法的签名, 描述符以及方法属性表中各种属性的Java端的表示方式, 还包含了执行权限等运行期信息. 而后者仅仅是包含于执行该方法相关的信息. 通俗的来讲, Reflection是重量级的, MethodHandle是轻量级的.
+ MethodHandle是对字节码指令调用的模拟, 所以理论上虚拟机在这方面做的各种优化(如方法内联), 在MethodHandle上也应当可以采用类似的思路去支持, 而反射则不行.

#### invokedynamic指令
在某种程度上, invokedynamic指令和MethodHandle机制的作用是一样的, 都是为了解决原先的4条`invoke*`指令方法分派规则固化在虚拟机之中的问题, 把如何查找目标的方法的决定权从虚拟机转嫁给具体的用户代码, 让用户有更高的自由度. 并且两者都是为了实现同一个目标而采用不同的方式, 一个使用的是Java代码和API来实现, 一个是用字节码和Class中的其他属性, 常量来完成.

每一处含有invokedynamic指令的位置被称作`动态调用点(Dynamic Call Site)`, 这条指令的第一个参数不再是CONSTANT_Method_info常量, 而是JDK1.7新加入的CONSTANT_InvokeDynamic_info常量, 从这个新常量可以得到3项信息: 引导方法(Boostrap Method, 此方法存放在新增的BootstrapMethods属性中), 方法类型(MethodType)和名称. 引导方法是有固定的参数, 并且返回值是java.lang.invoke.CallSite对象, 这个代表真正要执行的目标方法的调用. 根据CONSTANT_InvokeDynamic_info常量中提供的信息, 虚拟机可以找到并且执行引导方法, 从而获得一个CallSite对象, 最终调用要执行的目标方法.

```java
public class InvokeDynamicTest {
    public static void main(String[] args) throws Throwable{
        INDY_BootstrapMethod().invokeExact("cjyong");
    }

    public static void testMethod(String s) {
        System.out.println("Hello String: " + s);
    }

    private static MethodHandle INDY_BootstrapMethod() throws Throwable {
        CallSite cs = (CallSite) MH_BootstrapMethod().invokeWithArguments(lookup(),
                "testMethod",
                MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
        return cs.dynamicInvoker();
    }

    private static MethodHandle MH_BootstrapMethod() throws Throwable {
        return lookup().findStatic(InvokeDynamicTest.class,
                "BootstrapMethod",
                MT_BootstrapMethod());
    }
    private static MethodType MT_BootstrapMethod() {
        return MethodType.fromMethodDescriptorString(
                "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;" +
                        "Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",
                null);
    }
34
    public static CallSite BootstrapMethod(MethodHandles.Lookup lookup, String name, MethodType mt) throws Throwable {
        return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
    }
}
```

由于invokedynamic指令所面对的对象并非Java语言, 而是Java虚拟机上的动态语言, 简单依靠javac没办法生成带有invokedynamic指令的字节码. 这时候我们需要使用INDY攻击进行完成, 由于使用INDY工具读取, 某些方法不能进行修改.

## 基于栈的字节码接受执行引擎
### 解释执行
大部分的程序代码到物理机的目标代码或虚拟机可以执行的指令集之前, 都需要经历以下步骤:
+ 程序源码 -> 语法分析 -> 单词流 -> 语法分析 -> 抽象语法树
+ 如果是解释执行的接下来为: 指令流(可选) -> 解释器 -> 解释执行
+ 如果为编译执行的接下来为: 优化器(可选) -> 中间代码(可选) -> 生成器 -> 目标代码

对于一门具体语言的实现来说, 语法分析, 以致后来的优化器和目标代码生成器都可以选择独立于执行引擎, 形成一个完整意义的编译器去实现, 这类代表是C/C++语言. 也可以选择其中的一部分步骤(如生成抽象语法树之前的步骤) 实现为一个半独立的编译器, 这类代表为Java语言. 又或者把这些步骤和执行引擎全部集中封装在一个封闭的黑盒子之中, 如大多数的JavaScript解释器.

在Java语言中, Javac编译器完成了程序代码经过语法分析, 到抽象语法树, 再到遍历语法树生成线性的字节码指令流的过程. 这部分是在Java虚拟机之外进行的, 而解释器在虚拟机内部, 所以java程序的编译是半独立的实现.

### 基于栈的指令集与基于寄存器的指令集
Java编译器输出的指令流, 基本上是一种基于栈的指令集架构(Instruction Set Architecture, ISA), 指令流中的指令大部分都是零地址指令, 它们依赖操作数栈进行工作. 与之对应的是, 基于寄存器的指令集, 最典型的就是x86的二地址指令集.

这里用一个简单的例子进行阐述, 比如我们计算1 + 1, 基于栈的指令集还是:
```
iconst_1	//将常量1压入栈中
iconst_1	//将常量1压入栈中
iadd		//将栈顶2个元素相加, 压入栈中
istore_0 	//将栈顶元素存储到局部变量表中的第一个位置
```

基于寄存器的指令集:
```
mov eax, 1 	//将eax寄存器的值设为1
add eax, 1	//将eax寄存器的值加1, 结果保存在eax寄存器中
```

两者各有优缺点, 基于栈的指令集主要优点是可移植. 而基于寄存器的指令集, 就会受到硬件的约束, 如,32位的80x86体系的处理器提供了8个32位的寄存器, 而ARM体系中CPU提供了16个32位的通用寄存器. 如果使用栈的架构的指令集, 用户程序不会直接使用这些寄存器, 就可以由虚拟机来自行决定把一些访问最频繁的数据放到寄存器中以获取尽量好的性能, 这样实现起来也更加简单, 代码也会更加紧凑, 编译器实现较为简单.

但是基于栈的指令集主要的缺点是执行速度较慢: 虽然栈架构指令集代码紧凑, 但是完成相同功能所需的指令数量一般会比寄存器架构多, 因为出入栈操作本身就产生了相当多的指令数量. 并且, 栈实现在内存中, 频繁的栈访问意味着频繁的内存访问, 对于处理器来说, 内存始终是执行速度的瓶颈. 尽管虚拟机可以采用栈顶缓存技术(把常用的操作映射到寄存器中, 避免直接内存访问), 但是这只是优化, 没有解决本质问题.

### 基于栈的解释器执行过程

```java
public int calc() {
	int a = 100;
    int b = 200;
    int c = 300;
    return (a + b) * c;
}
```

编译后的字节码:
```java
public int calc();
	Code:
    	Stack=2, Locals=4,	Args_size=1 //操作数栈深度最大为2, 局部变量最多为4个, 参数只有1个(this, 实例本身, 隐式参数)
        0:	bipush 100	//将整数常量100推入操作数栈顶
        2: istore_1		//将100存入局部变量表的第二个位置(第一个位置存储的为this)
        3: sipush  200  //将整数200推入操作数栈顶(200不是常量, 常量为-128-127), 
        6: istore_2		//将200存入局部变量表中的第三个位置
        7: sipush 300 //类似
        10: istore_3  //类似
        11: ilaod_1   //将局部变量表中第二个位置存储的对象(为int类型)取出, 放到操作数栈顶
        12: iload_2	 //同理
        13:iadd		//将操作数栈顶的两个对象取出(为int类型)进行相加, 将结果压入栈顶
        14:iload_3	//同理
        15: imul	//将操作数栈顶的两个对象取出(为int类型)进行相乘, 将结果压入栈顶
        16: ireturn	//将栈顶元素(为int), 返回
```




