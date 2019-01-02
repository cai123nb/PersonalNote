# General Programming

本章主要涉及到一些Java语言的一些使用技巧, 如本地变量的管理, 权限控制, 库, 数据类型, 反射和本地方法. 最后会讨论一些优化措施和命名规范.

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

正如`Item 57: Minimize the scope of local variables`所说的循环, 推荐了好几种写法. 但是这里强调的是, 如果循环中只关注对象集合本身中的每一个元素, 并不需要其它信息的话, 那么使用`for-each`循环可以带来更好的代码可读性, 减少存在bug的可能. 因为使用传统的for循环会暴露一个index`i`或者`iterator`, 这时候你可以利用这两个值进行其它元素的操作, 如果你仅仅想操作的是一个元素, 那么这就会破坏了对象的使用范围.

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

这个实现看起来是没有问题的, 非常正确. 但是实际上却是相反的. 这个实现有非常多的缺陷: 如果两次访问的时间非常短, 很有可能返回的值是相同的. 并且这个并不是真的随机, 有些数出现的频率会相对高一点. 如运行一百万次, 然后统计小于(n/2)的数量, 你会发现次数并不是预想的50W, 而是非常接近666,666. 极端的情况甚至返回超出范围的值: 如果随机产生的数为`Integer.MIN_VALUE`, 那么`Math.abs`返回的也将是`Integer.MIN_VALUE`, 返回的数字也将是负数. 并且这个问题非常难重现.

如果你要自己完善这些问题, 实现一个完美的随机算法, 那你需要学习的知识就非常多了: 伪随机数的生成, 数字原理, 随机算法的底层原理等等. 但是幸运的是, 现在通过`Random.nextInt(int)`, 你就可以完美地实现这些功能. 并且这些算法都是通过广泛验证的, 没有问题缺陷的. 使用第三个库的好处: `你可以借助第三方的专家的智慧为你所用, 并且吸收前人使用的经验`. 在`java7`中, 你不应该在使用`Random`, 而是使用`ThreadLocalRandom`, 后者可以更好地提高效率.

第二个好处就是, 可以极大地节省时间. 如果一个问题对你的项目只是一些小小的影响, 没有必要花费太多的时间在这上面时, 这时候就可以极大地提高效率.

第三个好处就是, 标准库被大量环境使用, 特别是商用环境, 被广泛测试和使用, 这些使用者对这些标准库有着严格的性能要求, 并且也有这个驱动去完善这些库. 一般标准库的性能都是很好的.

第四个好处就是, 标准库被多人使用, 如果有任何问题或者缺陷, 都会被很快提出, 并在后续的版本中进行完善.

最后一个好处就是, 使用标准库, 可以保证你的代码在主流内, 可以提高代码的可读性和重用性.

虽然有这么多好处, 但是程序员往往很少去使用这些库和方法. 这是为什么? 很大一部分的原因是程序员并不知道这些方法和这些库. 每次Java版本的更新都会添加许多API和类, 作为程序员去学习这些方法都是值得花费时间的. 虽然有时候会觉得这些方法添加的非常多, 没有充足的时间去完成. 也可以优先保证基本包的代码熟悉: `java.lang;java.util,java.io`以及非常有用的多线程包:`java.util.concurrent`. 偶尔这些内置方法并不能满足我们的要求, 这时候我们可以去学习一些别的高质量的第三方包, 如Google的`guava`包等.

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

总而言之, 不要使用`float`和`double`来进行精确数值计算. 如果需要进行小数点的记录且不关注消耗, 使用`BigDecimal`进行计算和跟踪. 如果计算不涉及到小数点(或者可以转换为不涉及小数点), 如果计算的数值不超过9位数(10进制), 使用`int`, 如果不超过18位数(10进制), 使用`long`, 否则的话就使用`BigDecimal`.

## Item 61: Prefer primitive types to boxed primitives

在Java的数据类型系统中, 存在两大主要类型: 原始数据类型(`int, float, double, byte, long, boolean, char, short`)和对象(如`String, List`). 原始数据类型也有对应的封装类型: `Integer, Float`等. 在Java中虽然提供了自动装箱和开箱的功能, 这非常容易让我们忽略两者的差别, 但是这两者还是具有非常大的区别性的.

首先原始数据类型只有值, 而封装类型却有引用等对象信息. 即: 两个原始数据类型只要值一样就一定相同, 而封装类型则不同, 存在内部值相同引用却不同的情况. 原始数据类型的只能是值(类对象中, 如果不赋值的话默认设置为0), 而封装类型缺存在`null`情况. 另外封装类型在时间和空间上的消耗更大.

```java
private static Comparator<Integer> naturalOrder = (i, j) -> (i < j) ? -1 : (i == j ? 0 : 1);
```

这是一个正常的比较器实现, 并且可以在集合中正常使用. 但是这个比较器却存在一个问题`naturalOrder.compare(new Integer(23), new Integer(23))`, 返回的是1, 而不是0. 这是为什么呢? Java在进行`<, >`比较时会自动开箱比较, 首先`i < j`是正常的数值比较. 等到了`==`比较时, java却不会进行开箱比较了, 而是直接比较引用. 这时候两个对象类型引用肯定不一样, 所以返回1.`使用==比较两个封装原始数据时, 封装原始数据类型不会进行开箱比较`. 修正的方法:

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