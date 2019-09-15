# General Programming

本章主要涉及到一些 Java 语言的一些使用技巧, 如本地变量的管理, 权限控制, 库, 数据类型, 反射和本地方法. 最后会讨论一些优化措施和命名规范.

## Item 57: Minimize the scope of local variables

正如`Item15`所说的: `Minimize the accessibility of classes`. 通过最小化变量的使用范围可以提高代码的可读性, 降低潜在的风险. 主要的原因有两点: 一般最小化变量的使用范围时, 都是`使用时才会声明变量`. 这样可以提高代码的可读性, 如果全部变量提前声明放在一起, 那样你可能就需要记住每一个变量到底是干什么的, 非常容易出错. 而这种情况是不会需要额外的记忆工作了. 并且如果在外部声明变量, 等同于破坏了变量的生存周期: 延长或者提前了. 如果别的代码(不可控的)不小心修改了该变量的值, 那结果肯定是灾难的, 并且非常难发现问题所在.

另外`几乎每一个变量声明的时候都应该进行初始化`, 这样可以保证变量的使用范围. 除了一些异常情况: 如对象初始化可能导致异常, 需要进行手动捕获, 这时候需要在`try`语句前提前声明好, 保证对象的可见性. 同时在循环中也是经常出现问题的:

```java
Iterator<Element> i = c.iterator();
while (i.hasNext()) {
    doSomething(i.next());
}
...
Iterator<Element> i2 = c2.iterator();
while (i.hasNext()) {           //Bug here
    doSomething(i2.next());
}


//Preferred idiom
for (Element e : c) {
    ...// Do something with e
}

for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ...//Do something with e and i
}

for (int i = 0, n = expensiveComputation(); i < n; i++) {
    ...//Do something with i
}
```

保证变量使用范围最小化的最后一个技巧就是保证方法的简洁和专注, 每个方法只完成一件事. 如果发现违背了这个要求, 推荐进行拆分.

## Item 58: Prefer for-each loops to tranditional for loops

正如`Item 57: Minimize the scope of local variables`所说的循环, 推荐了好几种写法. 但是这里强调的是, 如果循环中只关注对象集合本身中的每一个元素, 并不需要其它信息的话, 那么使用`for-each`循环可以带来更好的代码可读性, 减少存在 bug 的可能. 因为使用传统的 for 循环会暴露一个 index`i`或者`iterator`, 这时候你可以利用这两个值进行其它元素的操作, 如果你仅仅想操作的是一个元素, 那么这就会破坏了对象的使用范围.

```java
for (Element e : elements) {
    ... //Do something with e
}

for (Suit suit : suits) {
    for (Rank rank : ranks) {
        deck.add(suit, rank);
    }
}
```

但是`for-each`使用是存在限制的, 每次获取的对象只有一个, 你没办法进行前后左右关联或者过滤等操作. 如果你需要进行复杂的操作: `过滤`, `元素转换`, `平行迭代`等复杂操作, 这时候还是使用传统的`for`循环更好. 另外要使用`for-each`循环, 需要实现`Iterable<E>`接口. 如果你设计的类是存储一系列的对象, 那么推荐实现该接口.

总而言之, 如果条件允许的话尽量使用`for-each`循环, 而不是`for`循环.

## Item 59: Know and use the libraries

这边以一个简单的例子来说明, 假设你需要完成一个函数, 函数返回一个在范围内的随机数字. 一个可能的实现是:

```java
static Random rnd = new Random();
static int random(int n) {
    return Math.abs(rnd.nextInt()) % n;
}
```

这个实现看起来是没有问题的, 非常正确. 但是实际上却是相反的. 这个实现有非常多的缺陷: 如果两次访问的时间非常短, 很有可能返回的值是相同的. 并且这个并不是真的随机, 有些数出现的频率会相对高一点. 如运行一百万次, 然后统计小于(n/2)的数量, 你会发现次数并不是预想的 50W, 而是非常接近 666,666. 极端的情况甚至返回超出范围的值: 如果随机产生的数为`Integer.MIN_VALUE`, 那么`Math.abs`返回的也将是`Integer.MIN_VALUE`, 返回的数字也将是负数. 并且这个问题非常难重现.

如果你要自己完善这些问题, 实现一个完美的随机算法, 那你需要学习的知识就非常多了: 伪随机数的生成, 数字原理, 随机算法的底层原理等等. 但是幸运的是, 现在通过`Random.nextInt(int)`, 你就可以完美地实现这些功能. 并且这些算法都是通过广泛验证的, 没有问题缺陷的. 使用第三个库的好处: `你可以借助第三方的专家的智慧为你所用, 并且吸收前人使用的经验`. 在`java7`中, 你不应该在使用`Random`, 而是使用`ThreadLocalRandom`, 后者可以更好地提高效率.

第二个好处就是, 可以极大地节省时间. 如果一个问题对你的项目只是一些小小的影响, 没有必要花费太多的时间在这上面时, 这时候就可以极大地提高效率.

第三个好处就是, 标准库被大量环境使用, 特别是商用环境, 被广泛测试和使用, 这些使用者对这些标准库有着严格的性能要求, 并且也有这个驱动去完善这些库. 一般标准库的性能都是很好的.

第四个好处就是, 标准库被多人使用, 如果有任何问题或者缺陷, 都会被很快提出, 并在后续的版本中进行完善.

最后一个好处就是, 使用标准库, 可以保证你的代码在主流内, 可以提高代码的可读性和重用性.

虽然有这么多好处, 但是程序员往往很少去使用这些库和方法. 这是为什么? 很大一部分的原因是程序员并不知道这些方法和这些库. 每次 Java 版本的更新都会添加许多 API 和类, 作为程序员去学习这些方法都是值得花费时间的. 虽然有时候会觉得这些方法添加的非常多, 没有充足的时间去完成. 也可以优先保证基本包的代码熟悉: `java.lang;java.util,java.io`以及非常有用的多线程包:`java.util.concurrent`. 偶尔这些内置方法并不能满足我们的要求, 这时候我们可以去学习一些别的高质量的第三方包, 如 Google 的`guava`包等.

总而言之, 尽量避免重复造轮子, 如果你需要完成一件事, 并且这件事的功能非常普通和广泛, 往往已经有现成的优秀的实现了. 如果是的, 可以使用它们. 这并不会影响你作为一个程序员的能力, 毕竟时间是有限的, 把有限的时间拿出来做最有价值的事.

## Item 60: Avoid float and double if exact answers are required

`float`和`double`是[浮点类型参数](https://cjyong.com/source/jdk/float.html), 详情可以查看`IEEE754`关于浮点数的定义, 主要用于工程计算和科学领域. 对于十进制的数有时候没办法进行精确的标示, 不适合表示特定的值, 特别是对于金钱的计算. 如:

```java
System.out.println(1.00 - 9 * 0.1);     //0.09999999999999998
System.out.println(1.03 - 0.42);        //0.6100000000000001

//Broken money calculate
public static void main(String[] args) {
    double funds = 1.00;
    int itemBought = 0;
    for (double price = 0.10; funds >= price; price += 0.10) {  //0.1 + 0.2 + 0.3 + 0.4 = 1.0
        funds  -= price;
        itemBought++;
    }
    System.out.println(itemBought + " items bought.");      //3
    System.out.println("Money left over: $" + funds);       //0.39999999999999999
}

//Solution1 but slower
public static void main(String[] args) {
    final BigDecimal TEN_CENTS = new BigDecimal(".10");
    BigDecimal funds = new BigDecimal("1.00");
    int itemBought = 0;
    for (BigDecimal price = TEN_CENTS; founds.compareTo(price) >= 0; price = price.add(TEN_CENTS)) {
        funds = funds.subtract(price);
        itemsBought++;
    }
    System.out.println(itemBought + " items bought.");      //4
    System.out.println("Money left over: $" + funds);       //0.00
}
//Solution2
public static void main(String[] args) {
    int funds = 100;
    int itemBought = 0;
    for (int price = 10; funds >= price; price += 10) {
        funds  -= price;
        itemBought++;
    }
    System.out.println(itemBought + " items bought.");      //4
    System.out.println("Money left over: $" + funds);       //0
}
```

总而言之, 不要使用`float`和`double`来进行精确数值计算. 如果需要进行小数点的记录且不关注消耗, 使用`BigDecimal`进行计算和跟踪. 如果计算不涉及到小数点(或者可以转换为不涉及小数点), 如果计算的数值不超过 9 位数(10 进制), 使用`int`, 如果不超过 18 位数(10 进制), 使用`long`, 否则的话就使用`BigDecimal`.

## Item 61: Prefer primitive types to boxed primitives

在 Java 的数据类型系统中, 存在两大主要类型: 原始数据类型(`int, float, double, byte, long, boolean, char, short`)和对象(如`String, List`). 原始数据类型也有对应的封装类型: `Integer, Float`等. 在 Java 中虽然提供了自动装箱和开箱的功能, 这非常容易让我们忽略两者的差别, 但是这两者还是具有非常大的区别性的.

首先原始数据类型只有值, 而封装类型却有引用等对象信息. 即: 两个原始数据类型只要值一样就一定相同, 而封装类型则不同, 存在内部值相同引用却不同的情况. 原始数据类型的只能是值(类对象中, 如果不赋值的话默认设置为 0), 而封装类型缺存在`null`情况. 另外封装类型在时间和空间上的消耗更大.

```java
private static Comparator<Integer> naturalOrder = (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

这是一个正常的比较器实现, 并且可以在集合中正常使用. 但是这个比较器却存在一个问题`naturalOrder.compare(new Integer(23), new Integer(23))`, 返回的是 1, 而不是 0. 这是为什么呢? Java 在进行`<, >`比较时会自动开箱比较, 首先`i < j`是正常的数值比较. 等到了`==`比较时, java 却不会进行开箱比较了, 而是直接比较引用. 这时候两个对象类型引用肯定不一样, 所以返回 1.`使用==比较两个封装原始数据时, 封装原始数据类型不会进行开箱比较`. 修正的方法:

```java
//Solution1
private static Comparator<Integer> naturalOrder = (i, j) -> (i < j) ? -1 : (i > j ? 1 : 0);

//Solution2
rivate static Comparator<Integer> naturalOrder = (iBoxed, jBoxed) -> {
    int i = iBoxed, j = jBoxed; //Auto-unboxing
    return (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
};
```

第二个例子:

```java
public class Unbelievable {
    static Integer i;

    public static void main(String[] args) {
        if (i == 42) {
            System.out.println("Unbelievable");
        }
    }
}
```

这个程序并不会输出`Unbelievable`, 而是报一个空指针异常. 为什么呢? 因为这里`==`比较时`混合了原始数据类型和封装的数据类型, 会自动开箱比较`. 这里自动替`i`进行开箱, 这时候`i`为`null`, 这时候就报`NOP`.

第三个例子:

```java
public static void main(String[] args) {
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
    System.out.println(sum);
}
```

这个代码虽然可以正确执行, 但是效率是非常的低. 上面三个例子就是忽视了封装数据类型和原始数据类型的区别, 导致的严重的后果. 但是有时候却不得不使用封装类型, 如集合中元素, keys, values. 这时候就只能使用封装对象. 第二种情况就是就是泛型中, 那就必须使用封装数据类型.

总而言之, 如果可以使用封装数据类型和原始数据类型时, 优先使用原始数据类型, 可以带来更好的性能. 另外千万需要注意两者的差别, 自动开箱和装箱虽然简化了代码, 但是没有忽略两者的区别, 特别是`==`比较时.

## Item 62: Avoid strings where other types are more appropriate

`String`对象设计来代表字符串, 并且可以很好的完成这个任务. 但是往往人们会将`String`的任务加重, 给`String`安排一些本不应该属于它的任务.

`Strings are poor substitutes for other value types.` 当我们从网络上, 文件上, 键盘等途径中获取信息时一般都习惯存储到`String`中, 其实这时非常低效的. 后期还需要进行加工处理. 如果接收的数据有问题, 就没办法第一时间知道. 推荐将信息按照本身的特征进行分类处理: 数值类型使用`int,long`进行存储. YesorNo 的问题使用`boolean`类型进行存储.

`Strings are poor substitutes for enum types.` 详情请查看`Item 34`.

`Strings are poor substitutes for aggregate types.` 一般聚合数据类型的数据存储, 如果使用字符串来存储的话, 肯定是需要进行字符串拼接剪裁操作, 这是非常低效的. 更好的方法是声明一个类来存储变量, 私有的静态成员类就非常适合`Item24`.

`Strings are poor substitutes for capabilities.` 字符串在功能性处的实现是非常低效的, 并且不安全的. 如我们需要实现一个`ThreadLocal`类来为每一个线程存储单独的变量.

```java
//Broken - inappropriate use of string as capability
public class ThreadLocal {
    private ThreadLocal() {}

    //Sets the current thread's value
    public static void set(String key, Object value);

    //Returns the current thread's value
    public static Object get(String key);
}
```

这里使用`String`作为`key`来标识内部的变量. 这个就存在一个非常严重的缺陷: 如果两个线程使用同一个`String`来存储变量, 那么变量就会冲突. 更好的使用方式就是使用`Key`:

```java
public class ThreadLocal {
    private ThreadLocal() {}

    public static class Key {
        Key() {}
    }

    //Sets the current thread's value
    public static void set(Key key, Object value);

    //Returns the current thread's value
    public static Object get(Key key);
}

//Typesafe
public class ThreadLocal<T> {
    private ThreadLocal() {}

    public static class Key {
        Key() {}
    }

    //Sets the current thread's value
    public static void set(Key key, T value);

    //Returns the current thread's value
    public static T get(Key key);
}
```

更好的方法就是内部类`Key`来存储和标识变量, 但是这个实现不是类型安全的, 使用的是 Object, 可能存在转换失败的情况. 这里使用泛型可以得到更好的体验. 当然这里只是一个例子, 使用 Java 原生的`java.lang.ThreadLocal`可以带来更好的性能体验.

总而言之, 不要将`String`拿来做太多除了表示`字符串`的功能. 不合理地使用字符串可能带来性能上的低效, 导致更高的风险.

## Item 63: Beware the performance of string concatenation

对字符串进行级联(拼接)操作花费的时间是二次方的层级.

```java
public String statement() {
    String result = "";
    for (int i = 0; i < numItems(); i++) {
        result += lineForItem(i);
    }
    return result;
}

public String statement() {
    StringBuilder b = new StringBuilder(numItems() * LINE_WIDTH);
    for (int i = 0; i < numItems(); i++) {
        b.append(lineForItem(i));
    }
    return b.toString();
}
```

虽然 Java 底层对级联做过优化, 进过测试后面方法的速度是前面的 6 倍左右. `尽量不要使用String来完成级联操作`, `StringBuilder`是一个很好的替代方法.

## Item 64: Refer to objects by their interfaces

当我们声明对象时, 如果存在合适的接口, 优先使用接口进行声明:

```java
//Good
Set<Son> sonSet = new LinkedHashSet<>();

//Bad
LinkedHashSet<Son> sonSet = new LinkedHashSet<>();
```

这样写有什么好处? 可以带来编码的灵活性. 使用接口编程, 后续需要更新代码时, 只需要重新实现一个新版本的实现类来替换原来的声明, 用户并不需要知道这点. 当然这也有一些地方没办法使用接口声明的方式: 如果没有合适使用的接口. 如果类是属于类继承框架, 那是用抽象类是一个很好的替换(如`InputStream`). 如果接口的功能并不能满足我们的要求, 如`Queue`不能满足我们对优先级的要求, 需要使用`PriorityQueue`. 这三种情况, 直接使用具体的类或者抽象类即可.

总而言之, 如果存在合适的接口对象, 使用接口声明是非常合适的. 如果不满足条件, 那就从类继承树里取最小的要求的类声明.

## Item 65: Prefer interfaces to reflection

java 在`java.lang.reflect`中提供了反射相关的支持, 通过反射你可以获取任意类的方法, 构造函数, 域等各类信息. 但是这个便利性是伴随着代价的.

- 你失去了编译时类型检查功能: 通过反射来构建时, 那么在编译时就没办法进行校验, 只有在运行时才知道具体的信息, 并存在随时失败的可能.

- 反射的代码是非常繁琐的容易出错的, 往往我们构造一个函数, 直接调用构造函数即可. 但是通过反射这个方法, 却非常繁琐, 需要读取并调用对应的构造函数.

- 性能差, 通过反射实现的代码往往效率非常差.

真正合理使用反射的地方非常少. 如代码分析工具和依赖注入框架. 有时候想知道自己的代码是否需要反射的支持. 结果往往是不需要的. 而需要的时候非常少: 具体的类型在编译时是不确定的, 但是可以通过父类或者接口进行初始化进行操作. 如一个程序需要`Set<String>`实例, 但是具体的类型需要用户输入之后进行实例化:

```java
public static void main(String[] args) {
    Class<? extends Set<String>> cl = null;
    try {
        cl = (Class<? extends Set<String>>) Class.forName(args[0]); //Unchecked cast!
    } catch(ClassNotFoundException e) {
        fatalError("Class not found.");
    }
    //Get the constructor
    Constructor<? extends Set<String>> cons = null;
    try {
        cons = cl.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
        fatalError("No parameterless constructor");
    }

    //Instantiate the set
    Set<String> s = null;
    try {
        s = cons.newInstance();
    } catch (IllegalAccessException e) {
        fatalError("Constructor not accessible.");
    } catch (InstantiationException e) {
        fatalError("Class not instantiable.");
    } catch (InvocationTargetException e) {
        fatalError("Constructor threw " + e.getCause());
    } catch (ClassCastException e) {
        fatalError("Class doesn't implement set");
    }

    //Exercise the set
    s.addAll(Arrays.asList(args).subList(1, args.length));
    System.out.println(s);
}

private static void fatalError(String msg) {
    System.err.println(msg);
    System.exit(1);
}
```

这个代码只是用于演示, 但是也可以看出反射的强大作用. 但是这个强大作用付出的代价也是非常大的: 本来一个简单的构造函数, 却需要处理六个不同异常, 实现非常繁琐. 并且存在`cast warnning`, 编译时会报警, 因为没有办法保证实例化对象一定是`Set<String>`类型. 另外使用反射在依赖的对象在运行时是不一定存在的时候, 通过反射动态构建是合理的. 这时候就需要对反射获取的对象是否存在进行逻辑处理.

总而言之, 反射是一个强大的工具, 主要用于复杂的系统, 并且使用是含有非常大的代价的. 一般情况下, 是不需要使用的. 如果有这个需求时: 在编译时无法得知具体的类型时, 使用接口和父类来通过反射构建对象来访问内部的元素和方法等.

## Item 66: Use native methods judiciously

Java 提供了`JNI(Java Native Interface)`允许 Java 程序调用`本地方法(Native Method)`, 而本地方法主要使用`C`或者`C++`来完成的. 由于历史遗留问题, 本地方法主要有三大用处: 访问平台特定的工具和资源, 如注册表. 访问本地库的代码. 使用本地库来优化性能.

如果使用本地方法来访问平台特点资源是合理的, 但是这个机会越来越少了. 随着 Java 平台的成熟, 很多以前必须通过本地方法访问的方式都可以通过 Java 本身进行调用了. 但是如果发现没有合适的 Java 库可以访问本地特定资源, 使用本地库来方法也是可以的. 而对于使用本地方法来优化性能的机会基本没有了, 因为随着 Java 平台对代码的优化, 很多时候原生的调用往往比本地库更加快.

使用本地方法也有非常严重的缺点: 本地语言是不安全的. Java 的垃圾收集器没有办法对本地内存进行管理. 如果不合理的使用, 非常容易导致内存溢出等问题, 并且这些问题基本上是不能调试的. 另外本地方法有点像`胶水代码(glue code)`, 非常难去阅读和书写的.

总而言之, 在使用本地库的时候需要好好考虑一波是否真的需要. 如果一定需要使用本地库来访问本地硬件的资源, 那么尽可能少的使用, 并且进行彻底地测试. 因为, 本地方法中一个简单的 bug 很有可能破坏整个程序.

## Item 67: Optimize judiciously

有三句优化的格言:

- **More computing sins are committed in the name of efficiency(without necessarily achieving it) than for any other single reason - including blind stupidity. ----William A. Wulf**

- **We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. ---- Donald E. Knuth**

- **We follow two rules in the matter of optimization: Rule1. Don't do it. Rule2(for experts only). Don't do it yet--that is, not until you have a perfectly clear unoptimized solution. ----M.A.Jackson**

这些格言在 Java 出现 20 年前就出现了, 这些格言都揭示了一个真理: 优化是非常难实现的, 这是非常容易在这个过程做更多的坏处而不是好处. 尽量遵守这个原则: `努力去写好的代码而不是快(自认为性能好)的代码`. 一个好的程序, 如果不够快(性能不够好), 那它本身一定可以运行它自己进行优化. 大多数的程序都是通过`信息隐藏`实现的, 这样你修改某一个模块的时候并不影响其他模块.

这并不意味着你可以忽略性能上的优化, 这是说实现的的问题, 是非常容易修复和改善的. 如果是设计的问题或者底层架构的问题, 那就非常难修改了. 一旦项目实现了, 这些部分基本上是不能进行优化和修改了. **因此避免不良的设计, 可以很大程度保证程序的性能**. 这个设计包括: API 设计, 底层通信的协议, 数据库存储的数据样式等等.

在设计 API 时需要考虑很多问题, 如设置一个公共类型的可变对象可能会导致很多不必要的拷贝, 使用继承而不是组合会限制实现的功能, 使用实现类而不是接口会限制具体的使用类型, 即使你后面构造了一个优化的类型. 如`java.awt.Component`类中的`getSize`方法设计是返回一个`Dimension`实例, 而该实例是可变的. 这就导致在频繁调用地时候, 每次调用都会创建一个新的实例对象, 这是非常浪费的. 即使在`java2`处于性能的考虑, 修改了该方法并发布了新的方法, 但是很多程序员并没有修改之前的调用, 依旧使用旧版的方法. 这也说明一个问题: **这是非常差的主意: 基于性能的考虑去包装一个旧的接口**.

正常的设计流程应该是: 当你小心的设计你的程序, 并撰写一个清晰的, 构造良好的实现. 然后才应该是考虑性能的问题.

另外回到`Jackson`的格言, 这里应该强调一点: `每次尝试优化前后都应该进行精确的测量`. 因为有时候你会很惊讶的发现, 你的优化往往没有取得任何效果, 反而导致更糟糕了. 其中主要的原因就是这是非常难去知道程序是如何使用处理器时间的. 同理, 合适的监控软件可以很好的帮我们观察和统计这些优化过程. 那些工具可以详细地告诉你运行时的信息, 比如该方法调用了几次, 每次花费的时间是多少, 设置一些算法设计问题. 如一个时间复杂度为平方(或者更加糟糕)的函数隐匿在方法内部, 这时候你只需要更换一个实现更加优化的实现就可以达到很好的效果. 代码优化, 就像在麦田里找麦子. 通过这些监控工具, 可以很快的定位热点问题, 先将大问题, 性能损耗大的问题解决, 再去考虑小问题. 这里推荐使用`JMH`.

为什么说 Java 中的性能优化测量是非常重要的? 首先, Java 不像`C`, `C++`是直接编译成本地代码, 让 CPU 运行. 而 Java 更多的是一种中间语言, 是通过 JVM 执行的, 拥有更差的`性能模型`. 也就是说在 Java 代码和 CPU 运行指令之间有一个`gap`, 并不像`C`,`C++`这么直观. 这是非常难去猜测的, 这时候使用测量就非常重要了. 其次, 随着 Java 版本的更新, 硬件的升级, 平台的多样化, Java 虚拟机的差异, 不同的 java 实现在不同的环境中运行的情况都有很大的不同. 如同一个代码在`32位`和`64位`系统中运行情况就非常的不同. 这些硬件, 软件, 平台等等多方面因素都会严重影响 Java 代码的实现, 这时候就需要用测量来保证优化的结果, 因为同一段代码在一个环境中的优化有时在另一个环境中使用时并没有效果.

总而言之, 不要努力去写快的代码, 而是去写好的代码. 好的代码, 性能自然不差. 另外在设计程序时, 优先考虑设计和底层的构造, 这些东西往往很多程度上决定程序的性能. 当项目完成之后, 尽可能地进行精确测量, 如果足够快那就可以了. 如果不行, 那就找到核心问题点, 一个一个去解决. 其中优化的首选目标是算法, 一个坏的算法可以带来巨大的速度差异.

## Item 68: Adhere to generally accepted naming convertions

Java 平台拥有成熟的命名规范, 其中大部分都定义在[Java Language Specification](https://docs.oracle.com/javase/specs/). 其中主要分为两大类: 通用的命名规范和语法命名规范.

通用的命名规范主要是对于包命名, 类命名, 接口命名, 方法命名, 成员变量命名, 类型参数(泛型)命名进行定义. 对于通用的命名规范, 一般不推荐违背, 这基本上是所有程序员的共识. 下面就关于这些方面进行详细的讲解:

对于包的命名一般都是基于分级的思想进行定义. 每个单词都是小写(一般没有数字). 另外如果该代码需要在外部使用的话, 一般前缀以公司的域名, 如`edu.cmu, com.google, org.eff`. 官方的标准库一般使用`java, javax`开头, 这也就不推荐一般的使用者使用`java, javax`作为包的前缀. 同时, 在包命名中, 缩写是可以接收的, 其中缩略词也是可以接收的, 一般单个单词不要超过 8 个字母. 每个包的分级都应该合理, 不推荐给将所有的类都放在同一个包下, 如`java.util, java.util.concurrent, java.util.concurrent.atomic`合理的分级可以起到很好的解释作用.

对于类和接口命名, 这里包括了枚举(特殊的类)和注释(特殊的接口)类型, 一般是由一个或者多个单词组成, 第一个字母大写, 如`List, FutureTask`. 在这里缩写是不推荐的, 除非是特别常用的缩写, 如`max, min`. 除此之外, 一般的单词不推荐进行缩写. 缩略词是可以使用的, 并且这里推荐对缩略词进行驼峰命名(很多人喜欢全部大写), 如`HTTPURL, HttpUrl`.

对于方法和成员变量的命名规则和类,接口的类似, 但是首字母需要小写. 这里需要注意的是一些特殊情况. 如果变量是常量的话, 推荐全部大写并且使用`_`进行单词分割, 如`VALUES, NEGATIVE_INFINITY`. 对于局部变量, 则是没有这么多限制, 这里允许合理的缩写, 如`i, denom, houseNum`. 如果是方法的参数那就另当别论了, 这里需要进行合理的考察和设计. 应为这些参数有可能成为 API 文档中的一部分.

对于类型参数(泛型), 一般有一些通用的约定. 如`T`代表一个任意类型, `E`代表集合中的元素, `K,V`代表 Map 中的 key 和 value, `X`代表异常, `R`代表返回值, 而对于其它任意类型可以使用`T,U,V,T1,T2,T3`.

这里进行简单的小结:

| Identifier Type    | Examples                                         |
| :----------------- | :----------------------------------------------- |
| Package or module  | org.junit.jupiter.api, com.google.common.collect |
| Class or Interface | Stream, FutureTask, LinkedHashMap, HttpClient    |
| Method or Field    | remove, groupingBy, getCrc                       |
| Constant Field     | MIN_VALUE, NEGATIVE_INFINITY                     |
| Local Variable     | i,denom,houseNum                                 |
| Type Parameter     | T,E,K,V,X,R,U,V,T1,T2                            |

语法的命名规则比通用的命名规则更加灵活, 限制也更加少, 另外更多的时候这里没有固定的约定, 往往只是一个建议. 对于普通的可实例化的类, 一般一个简单的单词, 标明用途就可以了, 如`Thread, PriorityQueue, ChessPiece`. 而对于不可实例化的类, 一般使用复数的形式, 如`Collectors, Arrays`. 而对于接口来说往往以`able, ible`结尾: `Iterable, Accessible`. 而对于注释, 则没有这么多的限制, 现在使用的也各式各样: `BindingAnnotation, Inject, ImplementedBy, Singleton`.

 对于方法来说, 一般以动词开头, 标明作用, 如`append, drawImage`. 而对于一些返回`boolean`类型的参数, 一般以`is,has`开头, 如`isDigit, isProbablePrime, isEmpty, hasSiblings`. 另外如果是返回特定的类型的  对象, 直接使用对象名称也是可以的, 如`size, hashCode, getTime`. 最后就是`Java Bean`中的`Getter`和`Setter`方法.

另外对于有特殊需求或者操作的方法, 往往有特殊的命名规则, `toType,toList, toString, asType, asList, asSet, intValue, doubleValue, longValue`. 以及静态方法通用的`from, of, valueOf, instance, getInstance, newInstance, getType, newType`等方法.

对于变量来说, 要求比较少, 一般以特定类型进行命名, 如`height, digit, bodyStyle`等. 其中`boolean`类型的值, 可以使用`is`作为前缀.

总而言之, 合理的使用这些命名规范, 养成习惯可以  带来很好的编码  质量.
