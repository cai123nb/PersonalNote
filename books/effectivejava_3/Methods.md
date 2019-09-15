# Methods

本章主要讨论方法设计中的一些要素: 如何处理参数和返回值, 如何设计方法的签名, 如何文档化函数. 这些主要是关注于方法的可用性, 健壮性和灵活性.

## Item 49: Check parameters for validity

大多数的方法都对传递的参数有特殊的要求, 如某个整数必须为整数, 对象引用不能为空. 对于这种情况, 一般都习惯性在方法的一开始进行校验, 如果违反  就需要尽早抛出异常, 尽快失败. 为什么要这么做呢?  如果不进行参数校验,  很有可能方法在后面就会出现异常 , 抛出一个困惑的异常, 让真正的问题更加难以定位. 更严重的是, 方法在后续的执行中不正常中断返回, 留下某些中间状态的对象, 从而导致别处出错.

一般的做法就是在方法的开始进行权限校验, 然后在文档中通过`@throws`进行说明. 一般抛出的异常有`IllegalArgumentException`, `IndexOutOfBoundsException`, `NullPointerException`等. 如:

```java
/**
    * Returns a BigInteger whose value is {@code (this mod m}).  This method
    * differs from {@code remainder} in that it always returns a
    * <i>non-negative</i> BigInteger.
    *
    * @param  m the modulus.
    * @return {@code this mod m}
    * @throws ArithmeticException if m is less than or equal to 0.
    */
public BigInteger mod(BigInteger m) {
    if (m.signum <= 0)
        throw new ArithmeticException("BigInteger: modulus not positive");
    .. //Do the computation
}
```

在 Java7 中映入了`Objects.requireNonNull`方法进行空引用判断, 如果需要进行空判断, 也就没有必要手动  获取了. 甚至, 该方法还支持之定义  异常信息.

```java
this.strategy = Objects.requireNonNull(strategy, "strategy");
```

在后面的版本中, Java9 引入了`checkFromIndexSize, checkFromToIndex, checkIndex`等工具方法, 用于检测是否越界判断. 这些方法只能用于数组或者 list, 并且并不可以进行异常信息的定制. 使用的并没有`requireNonNull`这么广. 在一些私有的内部方法中, 为了保证包内的  方法的正确执行, 一般会用到`assertions`作为判断:

```java
//private helper function fro a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    ...// Do the computation
}
```

如果这些判断没有成功就会抛出`AssertionError`, 另外这些  判断需要在命令行中手动开启`-ea(或者: -enableassertions)`.

当然这也有一些例外, 如对于参数验证是不现实的或者代价高昂的, 并且在计算的过程中会进行计算, 不进行检测也是可以的. 如`Collections.sort(arr)`, `arr`中所有的元素都要是`Comparable`的, 但是对内部所有的元素进行检查代价是非常高的, 并且在后续的计算中, 如果对象不是`Comparable`的, 进行比较时自动会抛出异常. 因此这里是不用使用的.

总而言之, 当我们定义一个方法的时候, 应该考虑好参数的限制, 对于这些限制应该在文档中进行说明, 并且在代码强制执行判断.

## Item 50: Make defensive copies when needed

相比别的语言, Java 是属于类型安全的语言, 在没有本地方法调用的情况下, 一般不会出现数组越界, 野生指针等内存异常问题. 但是即使在这种情况下, 如果你不采用一些活动来隔离别的类, 你的类往往还是可能出现问题: 你的代码使用者会尽他们最大的力量来破坏你的代码. 因为使用者除了正常使用的人之外, 精通编程的人往往才是主要的攻击者.

一般来说, 对于不变类, 只要你不提供修改内部的状态的方法, 是不太可能修改内部的状态的. 但是现实往往是相反的:

```java
public final class Period {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        if (start.compareTo(end) > 0) {
            throw new IllegalArgumentException(start + " after " + end);
        }
        this.start = start;
        this.end = end;
    }

    ..//The other getter method is omitted.
}
```

这个类咋一看是非常正常的, 一个标准的不变类. 但是事实上却并不是:

```java
Date start = new Date();
Date end = new Date();
Period period = new Period(start, end);
end.setYear(78); //Modifier status
```

为什么会出现这种问题? 因为 Date 类是可变的, 并不是不变类. 如果是 JDK8 的话, 修复的话可以使用(`LocalDateTime或者ZonedDateTime`)进行替换, 因为这两者是不变的.`Date`是过时且废弃的, 不应该出在现在的代码中. 有些情况下, 内部的成员是可变的, 如在 JDK8 之前的情况下使用`Date`, 那应该怎么办呢? 我们可以进行必要的复制操作:

```java
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (start.compareTo(end) > 0) {
        throw new IllegalArgumentException(start + " after " + end);
    }
}
```

这就可以预防前面的攻击, 注意这里的比较是在后面进行的, 防止多线程时比较后修改状态, 导致赋值的中间状态. 并且这里没有使用`Date.clone`方法进行拷贝, 因为 Date 的`clone`方法有可能返回该对象的子类, 并不一定是`Date`实类, 是不可信的.

```java
//Second attack on internals of a period instance
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end);
p.end().setYear(78);        //Modifies internals of p
```

通过访问函数访问内部的对象, 然后进行修改. 这种修复也非常简单, 就是对所有内部引用暴露时返回一个复制的对象.

```java
public Date start() {
    return new Date(start.getTime());
}

public Date end() {
    return new Date(end.getTime());
}
```

通过对构造函数和 Accessor 函数进行对象拷贝使的`Period`类最终为一个完整的不变类. 这种技术不仅仅适用于不变类, 也适用于可变类. 如果当一个类定义一个方法或者构造函数需要存储外部传递一个对象引用时, 这时候需要考虑一下是不是可以  接受, 传递的对象可能是可变的, 在使用的过程中可能会修改. 如果不能接受的话, 那就可以进行拷贝操作. 如:  将实例引用放入`Set`集合中, 如果后续的实例引用的值修改了, 就有可能导致集合出现冲突, 这时候就可以考虑进行拷贝.

同样的, 当需要给外部暴露内部的对象引用时, 需要  考虑一下, 如果内部引用是可变的, 那么可以接受外部进行修改内部的值吗? 如果不行, 也可以进行拷贝操作.  当然也有一些特殊情况, 如果你相信外部的调用一定不会修改内部的值, 那么你可以不进行拷贝. 如外部的调用只允许在同一个包内, 并且该包是你自己负责撰写. 但是也是需要在文档中声明清楚的.

总而言之, 如果一个不变类需要获取外部的引用或者暴露内部的引用, 必须进行拷贝操作. 如果代价过于高或者可以保证调用的安全性, 那就需要在文档中进行显式的说明.

## Item 51: Design method signatures carefully

本节主要关注于如何设计你的 API, 让它可读性更好并且减少出错的机会.

首先给方法一个良好的名字是非常重要的. 首先, 名字一定要易读且符合包内别的方法一贯的风格, 避免使用特别长的名称.

第二, 不要过分的提供辅助方法. 不要为了便利性, 提供非常多的辅助方法. 特别是对于接口, 那对于接口的实现者将会是灾难. 除非这个 ss 辅助方法经常使用, 否则的话就不要单独列出来.

第三, 避免过长的参数列表, 最好不要超过 4 个. 过长的参数列表不仅会让用户难以理解, 还特别容易出错: 如果不小心修改了顺序并且程序没有报错, 那么结果将不会按照你的意愿执行.

这里有三个方法进行缩短参数列表, 第一个就是通过分解方法, 如果方法的参数过长, 可以将方法分解成不同的子方法, 每个方法接受一部分的参数. 但是这个也有一个缺点就是会导致方法的琐碎. 当然也可以通过方法的正交来减少繁琐的方法, 即有些方法的功能是类似的话, 可以省略掉. 如: `List`接口没有提供获取第一个或者最后一个的方法, 因为这两个可以通过`indexOf`替换掉.第二个方法就是构建辅助类, 如果发现一系列的参数经常出现, 那可以进行封装成一个类. 然后接收的时候只需要接收一个类接口. 第三个方法就是结合前面两者, 通过`item 2`的`Builder`模式来进行参数传递.

第四, 对于参数类型, 偏好接口而不是类. 定义接口而不是类, 可以给使用者更大的灵活性.

第五, 对于两个元素的类型参数, 偏好枚举而不是`boolean`类型. 枚举类型可以带来更好的可读性和拓展性. 如温度的计量方式: `public enum TemperatureScale {FAHRENHEIT, CELSIUS}`, 后续拓展的就非常方便, 并且可读性比较好: `Thermometer.newInstance(TemperatureScale.CELSIUS)` > `hermometer.newInstance(true)`.

## Item 52: Use overloading judiciously

我们先看一下重载的简单使用:

```java
public class CollectionClassifier {
    public static String classify(Set<?> s) {
        return "Set";
    }

    public static String classify(List<?> s) {
        return "List";
    }

    public static String classify(Collection<?> c) {
        return "Unknown Collection";
    }

    public static void main(String[] args) {
        Collection<?> collections = {
            new HashSet<String>(),
            new ArrayList<Integer>(),
            new HashMap<String, String>().values()
        };

        for (Collection<?> c : collections) {
            System.out.println(classify(c));
        }
    }
}
```

我们期待的输出是: `Set, List, Unknown Collection`. 但是结果却是输出`Unknown Collection`三次. 为什么? 因为重载是静态的, 是在编译期确定的. 与之相反的是重写, 重写是动态的, 是在运行期确定的.

```java
class Wine {
    String name() { return "wine"; }
}
class SparklingWine extends Wine {
    @Override
    String name() { return "sparkling wine"; }
}
class Champagne extends Wine {
    @Override
    String name() { return "champagne"; }
}
public class Overriding {
    public static void main(String[] args) {
        List<Wine> wineList = List.of(new Wine(), new SparklingWine(), new Champagne());
        for (Wine wine : wineList) {
            System.out.println(wine.name);
        }
    }
}
```


这个代码是可以正确运行的. 那么修复的方法是什么样的呢:

```java
public static String classify(Collection<?> c) {
    return c instanceof Set ? "Set" : c instanceof List ? : "List" : "Unknown Collection";
}
```

这里可以看出不合理的使用重载非常容易出错,  并且出现问题之后, 程序并不会报错, 只是执行不正确的方法代码, 因此尽量避免使用不合理的重载. 一般来说合理的重载就是要保证参数的数量不同. 这看起来有点严格, 但是, 更多的时候我们仅仅只需要改下方法名, 并不需要使用重载. 如`ObjectOutputStream`中输出函数, 对于不同的对象使用不同的方法名: `writeLong(), writeBoolean, writeInt`等.

对于构造函数来说,  就没有办法取不同的方法名. 但是还是可以使用别的方法的, 如静态构造函数而不是构造函数. 如果非要使用函数的方法时, 即方法名相同时, 那就尽量保证参数  个数不同. 否则的话, 就需要保证参数顺序具有巨大的区别: 保证相邻参数不能合法的进行转换(`cast`).

在 JDK5 之前, 原始类型和对象引用是  非常不同的. 但是对于 JDK5 之后, 随着自动装箱和泛型的加入, 就非常容易困惑相关的操作. 如:`List.remove(int)和List.remove(Object), List.remove(E)和List.remove(int)`等. 并且在 Java8 之后, 引入到了 Lambdas 表达式和方法引用, 更加容易困惑.

```java
for (int i = 0; i < 3; i++) {
    set.remove(i);
    list.remove(Integer(i));
}

new Thread(System.out::println).start();    //compiles
exec.submit(System.out::println);           //Not comiles
```

为什么会出现这个问题呢? submit 方法还重载了一个`Callable<T>`的方法, 这和`Runnable<T>`是类似的, 编译器无法判断是哪个. 因此, 如果一个方法接收的参数**是不同的函数式接口引用, 那么就不要进行重载**.

总而言之, 不要轻易的使用重载, 最好的就是使用不同的方法名. 非要使用的话, 最好保证参数的数量不同. 如果不能满足这个条件, 那么就需要保证相邻参数不会随意地转化: 即任意一组参数不能被传递给多个重载函数. 否则的话, 你就需要进行重构了.

## Item 53: Use varargs judiciously

可变参数是一种语法糖, 用于接收和处理可变的参数类型. 本质上就是初始化一个数组进行存储. 如:

```java
static int sum(int... args) {
    int sum = 0;
    for (int arg : args) {
        sum += arg;
    }
    return sum;
}
```

有时候有些可变函数的方法需要保证至少一个参数:

```java
static int min(int... args) {
    if (args.length == 0) {
        throw new IllegalArgumentException("Too few arguments");
    }
    int min = args[0];
    for (int i = 1; i < args.length; i++) {
        if (args[i] < min) {
            min = args[i];
        }
    }
    return min;
}
```

这种实现是非常丑陋的. 首先这个如果不传递参数时, 错误是发生在运行时, 而不是编译时. 你必须进行显示的错误检测. 这里有一个优化的版本, 就是使用两个参数:

```java
static int min(int first, int... remainingArgs) {
    int min = first;
    for(int arg : remainingArgs) {
        if (min < arg) {
            min = arg;
        }
    }
    return min;
}
```

另外还有一个问题, 因为可变参数使用时也就会意味着数组的创建和初始化. 对于性能非常苛刻的情况下, 这种代价就需要考虑一下了. 当然也有一种变相的解决方法. 如果根据经验来说, 如使用 3 个或者以内的参数的比例高达 90%, 那么就可以对前面 3 个参数进行自定义.

```java
public void foo() {}
public void foo(int al){}
public void foo(int a1, int a2){}
public void foo(int a1, int a2, int a3){}
public void foo(int a1, int a2, int a3, int... rest){}
```

总而言之, 合理地使用可变参数, 如果知道必须的参数数量, 可以预先定义好, 将可变的参数放在后面. 并且小心可变参数带来的性能损耗.

## Item 54: Return empty collections or arrays, not nulls

在我们返回数组或者集合的时候, 如果结果可能为空的话, 不要返回`null`, 而是返回一个空的集合或者数组. 这样可以带来更好的代码体验.

如果为返回`null`的话, 就需要客户端每次调用的时候进行空指针判断, 非常容易出错的. 一旦忘记添加, 就会导致严重的后果. 一般标准的方法是:

```java
//返回集合
public List<Cheese> getCheeses() {
    return new ArrayList<>(cheesesInStock);
}

//Optimizaiton - avoid allocating empty collection
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheesesInStock);
}

public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheeses[0]);
}

//Optimization - avoid allocating empty arrays
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

这里返回的是空的集合, 这里可以进行优化. 就是预先定义好空对象, 集合的话可以使用`Collections.emptyList()`, `Collections.emptySet()`, `Collections.emptyMap()`, 数组的话可以定义好一个空的数组(传递数组来暗示数组类型).

总而言之, 使用空的集合或者数组而不是 null 进行返回, 可以带来更好的代码安全性.

## Item 55: Return optionals judiciously

在 Java8 之前，如果不能确定某一个方法是否会返回一个确定的值，这时候有两个解决方法：如果该值为空的时候, 返回 null, 或者抛出异常. 这两种实现方法都不够完美. 首先抛出异常付出的代价比较高, 需要栈帧中进行手动捕获. 返回 null 虽然没有这个代价, 但是要求后面的客户端使用的时候需要考虑对 null 的处理, 否则一不小心就会出现`NPE`.

在 Java8 之后引入了一个新的工具类`Optional<T>`, 对结果进行封装, 结果可以为空或者不为空. 这提供了另外一种灵活性来解决这种问题. 如`Item30`中的计算最大值:

```java
//Returns maximum value in collection - throws exception if empty
public static <E extends Comprable<E>> E max(Collection<E> c) {
    if (c.isEmpty()) {
        throw new IllegalArgumentException("Empty collection");
    }

    E result = null;
    for (E e : c) {
        if (result == null || e.compareTo(result) > 0) {
            result = Objects.requireNonNull(e);
        }
    }

    return result;
}

//Optional
public staitc <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty()) {
        return Optional.empty();
    }
    E result = null;
    for (E e : c) {
        if (result == null || e.compareTo(result) > 0) {
            result = Objects.requireNonNull(e);
        }
    }
    return Optional.of(result);
}
```

这里对于如果为空的话返回,`Optional.empty()`. 这里可以保证`result`一定不为空, 使用了`Optional.of`进行封装. 在不能确定的情况下, 可以使用`Optional.ofNullable`进行返回, 否则的话内部就会抛出`NPE`. 这里有一个小小的约定: 既然使用了`Optional`, 那么就不要返回`null`了.

`Optional<T>`更像一种设计方式, 提醒使用者, 这个方法可能返回一个空值, 你需要进行检查和处理. 另外`Optional`还提供很多辅助方法方便我们的使用.

```java
//Using optional to provide a chosen default value
String lastWordInLexicon = max(words).orElse("No words...");

//Using optional to throw a chosen exception
Toy myToy = max(toys).orElseThrow(TempertantrumException::new);

//Using optional when you know there's a return value
Element lastNobleGas = max(Elements.NOBLE_GASES).get();

//Using optional when get default value is expensive. (can compute and store first, than return by supplier)
Integer appleCount = max(apples).orElseGet(() -> {return Apple.DEFAULT_APPLE;});

//When you using optional in a stream, you can extract actual object inner.
streamOfOptionals
    .filter(Optional::isPresent)
    .map(Optional::get)
```

虽然使用`Optional`可以作为一些不确定对象的返回值, 但是也有例外. **不使用 Optional 来包含容器类: collections, map, streams, arrays and optionals.** 这时候返回一个空的容器类往往更加恰当. 所以`Optional`的使用情况主要适合返回的值可能不存在的时候, 并且不要用于包装一些容器. 另外需要记住使用`Optional`也是意味着创建一个新的对象, 在一些对性能要求非常严格的时候, 也会需要考量的(通过多次实验). 特别是封装原始数据类型, 相当于封装了两层: 先将原始数据类型封装成对象, 然后封装进`Optional`内部. 为了减少这种损耗`Optional`提供了`OptionInt, OptionalLong, OptionalDouble`三种原始类型数据, 内部存储的对象为原始数据类型. 对于其它原始数据类型: `Boolean`, `Byte`, `Character`, `Short`和`Float`. 就没有这个待遇了, 就尽量避免使用. 另外在 Map 中不适合作为`key`或者`value`, 也尽量不要用作一些静态实例.

总而言之, 当你确定某个方法的返回值可能为空, 并且这非常重要, 需要客户端进行特殊的处理兼容的时候, 可以考虑使用`Optional`. 反之, 需要考虑使用`Optional`带来的性能损耗, 并且避免使用在容器类中.

## Item 56: Write doc comments for all exposed API elements

如果一个 API 是公开使用的, 那么它一定需要进行进行文档注释. 在之前文档注释是通过手工完成的, 根据代码的完成进行同步完善, 这是非常耗时费力的. Java 中引进了`javadoc`工具来帮助我们动态生成 API 文档, 而我们需要做的则是在每一个方法或者对象内添加文档注释.

详细文档可以参照以前的[官方文档的规范](https://www.oracle.com/technetwork/articles/java/index-137868.html), 虽然这篇文章自从 Java4 之后就没有更新了, 但是还是非常详细和值得学习的. Java5 中新增了`@literal,@code`注释, Java8 中新增了`@implSpec`, Java10 中新增了`@index`. 对公开的类和 API 进行文档注释可以极大地提高了 API 的可读性, 这个注释应该包括类, 接口, 构造函数, 方法和域声明. 同时也推荐对非公开的 API 的代码进行合理的文档注释, 因为这可以极大地提高代码的可读性.

一般方法中的注释应该简洁地描述方法的前置条件和后置因果. 注解中应该详细描述或者枚举方法的前置条件, 如方法的调用应该在那种条件下, 该输入那些参数, 对参数的要求, 这个可以通过`@param`进行辅助说明. 其中如果方法会抛出异常的话, 可以使用`@throw`来说明对于每一种前置条件的违背, 都会抛出对应的异常. 后置因果, 可以说明如果正确调用该方法时, 会返回一个怎样的结果, 可以借助`@return`进行说明. 这里需要关注的是方法会做什么, 而不是方法怎么做. 同时如果方法调用会产生任何的副作用, 都应该在文档中叙述清楚.

```java
/**
 * Returns a list iterator over the elements in this list (in proper
 * sequence), starting at the specified position in the list.
 * The specified index indicates the first element that would be
 * returned by an initial call to {@link ListIterator#next next}.
 * An initial call to {@link ListIterator#previous previous} would
 * return the element with the specified index minus one.
 *
 * @param index index of the first element to be returned from the
 *        list iterator (by a call to {@link ListIterator#next next})
 * @return a list iterator over the elements in this list (in proper
 *         sequence), starting at the specified position in the list
 * @throws IndexOutOfBoundsException if the index is out of range
 *         ({@code index < 0 || index > size()})
 */
ListIterator<E> listIterator(int index);
```

一个标准的注释应该包含每一个参数`@param`, 返回值`@return`, 每一个抛出的异常`@throws`. 注意在这个代码中使用了`{@code}`注解, 这个注解的作用就是将内部包含的元素使用代码的格式输出到页面, 这样会比较明显并且可以使用 Html 的一些装饰符号, 如大于号(>)和小于号(<). 如果需要书写多行代码, 可以使用`<pre>{@code xxx}</pre>`进行显示. 这里还用到了`@link`进行跳转, 可以点击跳转查看相关信息. 注意在注释中是可以使用 HTML 一些元素的如`<tt></tt><i></i>`, 都是允许的, 生成时会自动插入到 HTML 文件中.

另外 Java8 中引入了`@implSpec`, 用于标注可以被实现的方法. 但是需要手动开启该功能(java9 中未默认开启), `-tag "implSpec:a:Implementation Requirements:"`.

```java
/**
 *  Returns true if this collection is empty.
 *
 * @implSpec
 * This implementation returns {@code this.size() == 0}.
 *
 * @return true if this collection is empty.
 */
public boolean isEmpty() { ... }
```

注意注释时如果需要使用`> < &`时都需要进行转义, 负责会影响 HTML 的生成, 可以使用`{@literal}`进行包裹, 但是不像`{@code}`那样会用代码的样式进行渲染. 同理写注释的时候尽量保证生成文档的可读性, 也要保证代码的注释的可读性. 一般方法注释的第一句为总结句, 总结句尽量简洁明了, 并且不要相同, 做到没有任何方法或者构造函数的总结句完全一样. 同时需要注意的是生成 HTML 文件时会将单词之间多余的空格会省略, 这时候可以使用`{@literal}`进行包裹. 

Java9 中引入了`{@index}`注释, 和`{@link}`最大的不同就是, 前者主要是引用到外部资源, 只是简单的用一个方框包围. 而后者主要是内部 API 之间的索引跳转.

```java
* This method complies with the {@index IEEE 754} standard.
```

对于一些特殊的类型, 如 map 或者枚举, 推荐为每一个元素进行文档注释. 而对于包级别的信息, 应该存储到`package-info.java`文件中.如果是模块系统的话, 可以将注解存储到`module-info.java`中. 并且无论是类或者方法都应该标注是否是线程安全的(`Thread-safe`).

```java
 * ...
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 *
 * @author  Josh Bloch
 * @see HashMap
 * @see TreeMap
 * @see Hashtable
 * @see SortedMap
 * @see Collection
 * @see Set
 * @since 1.2
 */
public interface Map<K,V> {}

/**
 * A day-of-week, such as 'Tuesday'.
 * ...
 * @implSpec
 * This is an immutable and thread-safe enum.
 *
 * @since 1.8
 */
public enum DayOfWeek implements TemporalAccessor, TemporalAdjuster {

    /**
     * The singleton instance for the day-of-week of Monday.
     * This has the numeric value of {@code 1}.
     */
    MONDAY,
    /**
     * The singleton instance for the day-of-week of Tuesday.
     * This has the numeric value of {@code 2}.
     */
    TUESDAY,
    ...
}
```

总而言之, 检验文档的最好方法就是阅读自己生成的 API 文档, 并持续改善. 合理的使用文档注释可以帮助你生成良好的 API 文档, 提高代码的可读性.
