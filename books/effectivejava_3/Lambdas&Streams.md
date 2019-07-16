# Lambdas and Streams

In Java8 引入了非常多的特性, 如功能接口, Lambdas, 方法引用等用来创建函数式对象. Streams相关的API则用来提供对数据的链式处理. 本章主要是介绍如何最大化地使用这些特性.

## Item 42: Prefer lambdas to anonymous classes

很久之前就已经存在, 如果一个接口(或者抽象类)只含有一个抽象方法, 通常被称作功能类型, 它的实例也别称为功能对象. 自从JDK1.1发布之后, 功能对象主要的使用方式就是匿名类, 如需要按照字符长度进行比较排序:

```java
Collections.sort(words, new Comparator<String>(){
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
})
```

自从JDK1.8引入Lambdas之后, 功能函数有了更好的表现方式:

```java
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length())
```

这里的使用Lambdas表示之后, 极大的简化了代码的表现形式. 对于多数的情况, Lambdas使用了类型推断都可以很好地进行类型判断, 对于某些无法进行判断的情况则需要我们手动进行标识类型. 关于类型推断较为复杂, 在`JLS`中占据整整一章的内容, 当然并不是需要所有人都要理解: **首先省略所有的类型, 如果编译器没有提示错误, 那就可以正确识别. 如果编译器提示错误, 那就需要手动进行标识.** 其中类型推断很大部分借助了泛型的信息, 因此正如`Item 26 29 30`说的避免使用原始类型, 尽量使用泛型对象和泛型方法. 如前面的排序例子, 如果`word`不是`List<String>`而是`List`, 那么我们的Lambdas表达式就会出错, 需要手动标识类型.

关于Lambdas的用处还有很多, 其中很多都在`java.util.function`中预先定义好了. 如我们可以进一步的缩减上面的比较函数:

```java
Collections.sort(words, comparingInt(String::length));

//More concise
words.sort(comparingInt(String::length));
```

正如前面`Item 34`中的枚举类型`Operation`.

```java
public enum Operation {
    PLUS("+") {public double apply(double x, double y) {return x + y;}},
    MINUS("-") {public double apply(double x, double y) {return x - y;}},
    TIMES("*") {public double apply(double x, double y) {return x * y;}},
    DIVIDE("/") {public double apply(double x, double y) {return x / y;}};

    private final String symbol;

    Operation(String symbol) {this. symbol = symbol;}

    @Override
    public String toString() {return symbol;}

    public abstract double apply(double x, double y);
}
```

这里可以通过Lambdas进行进一步的简化:

```java
public enum Operation {
    PLUS("+", (x, y) -> x + y),
    MINUS("-", (x, y) -> x - y)),
    TIMES("*", (x, y) -> x * y)),
    DIVIDE("/" (x, y) -> x / y));

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol) {this. symbol = symbol;}

    @Override
    public String toString() {return symbol;}

    public double apply(double x, double y) {
        return op.appplyAsDouble(x, y);
    }
}
```

这里使用`DoubleBinaryOperator`接收两个double类型参数,处理之后返回一个double类型的参数. 通过Lambdas可以极大地简化枚举中类型特定的行为.

使用Lambdas需要注意一点, 那就是尽量保证代码的简单性. 因为Lambdas缺乏名称和文档, 为了保证代码的可读性, 尽量让代码通俗易懂, 并且尽量在1-3行之间, 最好不要超过3行, 否则会严重影响代码的可读性. Lambdas是没有办法访问到实例内部的变量的, 这一点需要记住, 如果需要访问内部的实例变量, 那就还是使用枚举类型特定方法来实现. 相比匿名类, Lambdas虽然可以为功能性函数提供简洁的表现方式, 但是也有自己的缺点. 匿名类可以实现抽象类, 或者含有多个抽象方法的接口, 而Lambdas不可以. 另外Lambdas不可以访问内部的实例变量, `this`指向的是外部的包装类, 而匿名函数则不会, 是指向内部实例对象.

总而言之, 对于功能函数, 尽量使用Lambdas表达式来完成, 除非是抽象类或者不是功能函数(多个抽象方法)这时可以使用匿名类. Lambdas为我们开启函数式编程的大门.

## Item 43: Prefer method references to lambdas

Lambdas相对匿名类最大的优势就是简洁, 但是对于Lambdas来说, 方法引用的的优势不仅仅是简洁, 更是提高了代码的可读性. 如我们需要判断map中是否存在一个值, 如果不存在就插入一个1, 存在的话就进行加工处理. 这就是一般的Lambdas的表示方式:

```java
map.merge(key, 1, (count, incr) -> count + incr);
```

这里的`merge`方法是java8在map中引入的默认方法. 第三个参数需要传递一个`BiFunction`来处理对应key的value和传递的默认值. 这里就是简单的相加. 但是这里我们可以使用Integer的Sum函数来处理.

```java
map.merge(key, 1, Integer::sum);
```

这里不仅让代码开起来更加简洁了, 也提高了可读性. 当然方法引用不一定任何时候都会比Lambdas更加简洁和明了. 就比如函数在内部存在定义的时候:

```java
service.execute(GoshThisClassNameIsHumongous::action);

service.execute(() -> action();
```

这时候就需要你自己进行判断了, 如果使用了方法引用并不能减少代码或者提高可读性, 那么还是使用lambdas比较合适. 对于大多数的方法引用一般都是静态方法(static), 但是也存在一些例外, 大多分成四种. 有界的实例函数(`Bound Instance method`), 无界的实例函数(`Unbound Instance method`), 类构造函数和数组的构造函数.

```java
//Static
Integer::parseInt
str -> Integer.parseInt(str);

//Bound
Instant.now::isAfter
Instant then = Instant.now();
t -> then.isAfter();

//Unbound
String::toLowerCase
str -> str.toLowerCase();

//Class Constructor
TreeMap<K,V>::new
() -> new TreeMap<K,V>();

//Array Constructor
int[]::new
len -> new int[len];
```

这几种在Stream的链式处理中非常常见.

总而言之, 相比Lambdas, 方法引用往往可以提供更好的简洁性和代码可读性, 推荐使用方法引用, 除非不满足这两个条件时.

## Item 44: Favor the use of standard functional interfaces

自从Java引入Lambdas之后, 我们设计API时就需要合理的考虑Lambdas的作用了. 如之前的`模板方法模式(Template Method Pattern)`: 通过父类抽取和抽象化公共的属性和行为, 然后由子类通过继承的方式来进行重写来完成特定的功能. 这个模式就变得没有那么大的吸引力了: 因为可以接收函数对象来进行封装处理过程, 设置将函数对象设置为内部的成员变量, 通过构造函数或者静态实例函数进行传递. 而具体的实现过程就可以由调用者来制定. 这里需要考虑的是正确的函数式对象的参数和返回值类型.

这里以`LinkedHashMap`为例, 假设我们使用该对象进行缓存操作, 但是缓存需要限制大小, 不能无限的增长. 这里通过调用一个方法进行判断, 如果超出了限制就需要进行移除最早放入的对象.

```java
protected boolean removeEldestEntry (Map.Entry<k,V> eldest) {
    return size() > 100;
}
```

这里的实现是可以完成任务的, 但是使用Lambdas可以更好. 可以存储一个对应的函数式对象进行处理特定的需求, 这里需要注意一点的是函数式对象是没办法访问到实例的. 而这个方法是实例方法, 通过调用实例的`size()`方法获取数量然后进行处理. 而函数式对象是不可以获取实例的, 因此需要传递实例对象:

```java
//Unnecessary functional interface; use a standard one instead.
@FunctionalInterface
interface EldestEntryRemovalFunction<K,V> {
    boolean remove(Map<K,V) map, Map.Entry<K,V> eldest);
}
```

虽然这个接口是可以正确执行, 但是确是多余的. 因为`java.uti.funtion`包中内置非常多的函数式接口可以很完美的解决这个问题. **如果有标准的函数式接口可以很好的完成需求的话,那优先使用该接口而不是自己单独构建一个接口.** 这样可以减少很多冗余代码, 并且可以增强互通性. 正如上面的这个需求, `Predicate`接口就可以很好的完成需求, 之类使用了两个参数, 这里可以使用变种: `BiPredicate<Map<K,V>, Map.Entry<K,V>>`进行替换.

在`java.util.function`中含有43个接口定义, 这里不需要全部都熟读于心, 只需要知道6类基本的类型及其派生类型的变种方式即可. 6大基本类型:

+ UnaryOperator: 结果和参数都是相同类型的参数, T apply(T t), String::toLowerCase

+ BinaryOperator: 接收两个相同类型参数返回相同类型的对象, T apply(T t1, T t2), BigInteger::add

+ Predicate: 接收参数返回boolean类型结果, boolean test(T t), Collection::isEmpty

+ Function: 接收参数和返回结果类型不一致, R apply(T t), Arrays::asList

+ Supplier: 不接收参数返回对象, T get(), Instant::now

+ Consumer: 不返回对象进行消耗参数, void accept(T t), System.out::println

6大基本类型为3个原始数据类型:int, long, double派生出不同参数的接口函数. 如: `DoubleFunction<R>`: 接收double类型的参数返回对象, `DoublePredicate`: 接收double类型参数返回boolean类型, `LongBinaryOperator`: 接收两个long类型的参数返回long类型的结果.

另外对于`Function`接口还有其它变形, `Function<T,R>`中的定义是参数和返回类型始终不一样, 如果相同可以使用`UnaryOperator`进行替代. 如果参数和返回结果都是原始数据类型的话, 就是用`SrcToResult`命名模式(6种): LongToIntegerFunction, DoubleToIntFunction, IntToLongFunction等等.

第三种情况就是对参数个数的变形, 如接收两个参数的接口: `BiPredicate<T,U>`, `BiFunction<T,U,R>`, `BiConsumer<T,U>`. 其中`BiFunction<T,U,R>`也针对`int`,`long`,`double`返回类型进行了优化派生: `ToIntBiFunction<T,U>`, `ToDoubleBiFunction<T,U>`, `ToLongBiFunction<T,U>`. 其中`BiConsumer<T,U>`对参数中存在`int`,`long`,`double`的接口进行了优化派生: `ObjDoubleConsumer<T>`, `ObjIntConsumer<T>`, `ObjLongConsumer<T>`.

其中有些接口是重复了的, 如`BooleanSupplier`接口接收一个对象返回boolean类型, 但是这个和`Predicate`重复了. 但是是不推荐的, 推荐使用`Predicate`接口函数. 另外使用接口函数式优先使用派生优化的接口, 而不是原生类型, 这样可以减少在装箱之间的消耗. 

这些接口类型可以满足大部分人的需求, 但是如果你发现不满足时: 如需要3个参数的`Predicate`接口, 这时候就需要你自己进行定义了. 当然也有一些特殊情况: 如`Comparator<T>`接口函数, 我们会发现本质和`ToIntBiFunction<T,T>`是一致的. 但是我们还是选择使用`Comparator<T>`, 为什么呢?: 第一, 该接口名称具有很强的文字意义. 第二, 该接口需要满足非常严格的约定. 第三, 已经被广泛使用在底层中了. 如果你发现自定义的接口虽然可以在`java.util.funtion`中有了定义, 但是满足上面的条件, 那么你还是应该单独声明一个接口.

另外在声明函数式接口时, 推荐添加`@FunctionalInterface`注释来表明该接口可以被`Lambdas`使用. 另外在使用函数式对象作为参数的API时, 不要进行重载以接收不同的函数式对象, 这样会造成歧义, 尽量避免这种操作.

总而言之, 使用Lambdas和函数式对象作为输入输出的趋势是不可阻挡的. 如果需要定义函数式接口时, 并且发现可以通过默认的预定义接口完成的话, 那么你需要权衡好自己定义和使用预定义接口之间的优缺点.

## Item 45: Use steams judiciously

`streams`的API在Java8中正式引入来消除连续的冗余操作.`stream`中的元素可以来自任何地方: 集合, 数组, 文件, 伪随机生成数等, 可以是有限的, 也可以是无限的. 其中的对象既可以是对象引用, 也可以是原始数据类型: int, long, double.

`stream`的处理过程叫做`pipeline`, 主要分为两种: 中间处理过程`intermediate operation`, 主要是负责对其中的元素进行转换处理或者过滤操作. 终端处理过程`terminal operation`, 主要对其中的元素进行最终的处理过程, 如存储元素到集合, 返回特定的值, 打印所有的值等等. 其中`stream`是懒加载的: 只要`terminal operation`没有创建之前, 之前的中间处理过程是不会执行的. 所以千万不要忘了添加终端处理过程.

`stream`丰富的API可以为我们处理非常多复杂的运算, 但是这并不是意味着你一定要用`stream`. 合理的使用`stream`可以让我们的代码更加简单和清晰, 如果错误的使用, 则会让我们的代码可读性和维护性变得非常的差.

假设我们有这么一个需求, 有一个文件, 内部存储着非常多的单词, 取出其中的单词按照包含的字母进行分类, 统一放入在一起. 最后如果分类中单词的数量大于一定的要求, 就进行输出. 如`staple`单词分类为`aelpst`, `petals`同样分类为`aelpst`. 接下来进行最初的版本(没有使用stream):

```java
public class Anagrams {
    public static void main(String[] args) {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word), (unused) -> new TreeSet<>()).add(word);
            }
        }

        for(Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(gourp.size() + " : " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

接下来, 我们进行完全的`stream`化处理:

```java
public class Anagrams {
    public static void main(String[] args)  throws IOException {
        Path dictionary = Paths.get(args[0]);
        int miniGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                    groupingBy(word -> word.chars().sorted()
                            .collect(StringBuilder::new, (sb, c) -> sb.append((char) c), StringBuilder::append)
                            .toString()))
                    .values().stream()
                    .filter(group -> group.size() >= miniGroupSize)
                    .map(group -> group.size() + " : " + group)
                    .forEach(System.out::println);
        }
    }
}
```

这时候你会发现这段代码非常难以阅读, 即使你熟悉`stream`的API, 对于那些不熟悉的人来说, 更是难以理解. **过度使用steams会让代码变得非常难以阅读**.

幸运的是还有一个中间版本:

```java
//Testeful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args)  throws IOException {
        Path dictionary = Paths.get(args[0]);
        int miniGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
                    .values().stream()
                    .filter(group -> group.size() >= miniGroupSize)
                    .map(group -> group.size() + " : " + group)
                    .forEach(System.out::println);
        }
    }
    //alphabetize method is the same as in originla version
}
```

这段代码的可读性就非常高了, 就算不是很了解stream的人也可以大概理解这段代码. 有许许多多的代码是你不能确定是否使用`stream`可以带来更好的可读性和简洁性. 如:

```java
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit: Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
}

private static List<Card> newDeck() {
    return Stream.of(Suit.values())
        .flatMap(suit -> Stream.of(Rank.values())
        .map(rank -> new Card(suit, rank)))
        .collect(toList());
}
```

这时候就需要看你的喜好和团队的习惯了, 如果喜欢`stream`的话, 第二个是可以的. 否则就是第一个具有更好的可读性. 当你不知道使用那种合适的话, 最好的方法就是两种都实现, 再去选择更好的一种.

总而言之, 有些任务可以很好的使用streams, 有些则不能, 也有许多任务非常合适使用两者的组合. 这里没有明确的规定一定要使用哪一种, 但是当你不知道该使用那一种时, 你可以一起实现它们, 然后选择最好的一种.

## 46: Prefer side-effect-free functions in streams

如果你是`streams`的新手, 那你需要对此花费一定的时间进行学习. 当你花费了一段时间学习之后, 并且使用之后, 也许你会发现好像没有什么变化. 是的, streams相比于技术, 更多的是一种规范, 一种基于函数式编程的规范, 通过这种规范可以获取良好的代码可读性和表现. 如果最大化`streams`的优点, 那就是最大化函数式编程的本质`函数式编程`: 尽可能使用`pure function`, 即输出的结果只依赖输入, 不依赖任何别的可变的对象. 保证任何时候只要输入相同, 输出也一定相同.

假设需要完成一个需求, 统计一个文章内的单词格数：

```java
//Uses the streams API but not paradigm -- Don't do this
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
   words.forEach(word -> {
       freq.merge(word.toLowerCase(), 1L, Long::sum)
   });
}
```

这段代码虽然可以正确执行, 但却是感觉有股坏味道`bad smell`. 为什么, 因为这里的forEach函数使用了Lambdas进行处理, 但是处理的过程却不是纯函数的, 依赖于外部的freq. 只是一个伪装的streams编程. 那么正确的编程为:

```java
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words
       .collect(groupingBy(String::toLowerCase, counting()));
}
```

这段代码和前面的代码完成一样的事, 但是更加简洁, 可读性也更好. 这里放弃了`forEach`方法, 而是使用了新的`collect`方法. 为什么放弃`forEach`方法, 因为作为一个终端操作, `forEach`对`streams`并不友好, 且不支持并行化. 因此**forEach一般只用来输出数组中的元素**.

这里使用了`collect`方法, 这是一个新的技术, 需要我们传递一个`Collector`对象。 而该对象主要定义在`java.util.stream.Collectors`中. 这里简单的介绍一下, 该类较为复杂, 总共包含39个方法, 其中甚至包含5个参数. 但是该类并没有这么复杂, 其中大部分都可以由基简单的方法进行派生.

首先一类的方法主要的操作是收集`steams`中所有的元素然后将其放入集合中：toList(), toSet(), toCollection(collectionFactory). 就是简单的将元素放入到集合中或者指定类型的集合中. 如：选取频率最频繁的10个单词进行存储.

```java
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

那其余36的方法是什么呢? 主要是封装将stream中的元素转换为map对象： `toMap()`. 要转换为map对象肯定要包含`key`,`value`. 最简单的版本就是`toMap(keyMapper, valueMapper)`, 参数为两个function类型函数, 分别将元素转换为key和value. 但是这个版本有个问题: 如果转换元素的时候, 出现重复的key, 那就会出现问题. 就有了第二个版本, 传递一个merge对象处理碰撞. 第三个版本则是接收第四个参数Supplier, 传递一个指定的Map类型对象实例进行自定义.

```java
//First version
private static final Map<String, Operation> stringToEnum =
    Stream.of(values()).collect(toMap(Object::toString, e -> e));

//Second version
Map<Artist, Album> topHits = albums.collect(
    toMap(Album::artist, a -> a, maxBy(comparing(Album::sales)))
);

//Third version
toMap(keyMapper, valueMapper, (v1, v2) -> v2, TreeMap::new);
```

除了`toMap()`方法, 这里还支持通过`groupingBy`方法进行转换`map`对象：通过传递的分类方法function对streams中的元素进行分类, 相同类型的元素放在集合中. 最简单的版本只要传递一个分类的function：按照分类将不同的元素放到list中. 第二个版本则是需要传递第二个参数Collector对象, 对分类流中的元素再次进行处理: 如第一个版本的就是默认的toList(), 还可以进行count()计数, mapping()进行映射处理等. 第三个版本则是传递一个suppler容器将指定的分类中的对象存储到容器中. 另外还提供了`groupingByConcurrent`方法将元素放入到`ConcurrentHashMap`中.

```java
//First version
words.collect(groupingBy(world -> alphabetize(word)));

//Second version
Map<String, Long> freq = words
    .collect(groupingBy(String::toLowerCase, counting()));

//Third version
Map<City, Set<String>> namesByCity = people.stream()
    .collect(groupingBy(Person::getCity, TreeMap::new, mapping(Person::getLastName, toSet())));
```

上面第二个版本使用了`counting()`函数来进行计数, 此外Collectors还提供了很多别的辅助方法如: `summing, averaging, maxBy, minBy`等都可以使用. 最后需要注意的一个方法是`joining`方法, 来处理字符串类型元素, 将所有元素进行组合, 可以手动传递分隔符和前后缀, 如`[came, saw, conquered]`.

总而言之, 尽量使用纯函数来处理stream中的流对象, 合理地使用Collectors可以带来很好的便利性.

## 47: Prefer Collection to Stream as return type

许多方法返回的对象是一系列的对象. 在Java8之前返回的对象有: 集合(Collection: Set, List, Map), Iterable, 数组. 一般默认的返回是集合. 如果返回的对象流只是单纯地需要遍历, 并不需要支持集合地一些操作, 那么使用Iterable也可以. 如果对性能有严格地要求并且是原始类型数据, 那么返回数组将是最好地选择. Java8之后, 添加了Stream对象, 那相比之前地对象, Stream有什么优缺点吗?

如果一个方法返回的是stream类型对象, 这时候如果你需要对这系列对象进行遍历, 这时候你会发现非常的困惑. 为什么? Stream接口并没有继承Iterable接口. 但是却有着和Iterable接口相同的方法: iterator(). 但是如果想要直接使用进行遍历的话, 却要花费一点功夫.

```java
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    //Do something to ph
}
//Error won't compiler due to limitations on Java'type inference

for (ProcessHandle ph : (Iterable<ProcessHandle>)ProcessHandle.allProcesses()::iterator) {
    //Do something to ph
}
//Will work but too noisy and opaque to use in practice


for (ProcessHandle ph : iterableOf(ProcessHandle.allProcesses())) {
    //Do something to ph
}

public static <E> Iterable<E> iterableOf(Stream<E> stram) {
    return stream::iterator;
}
//Look good
```

这里借助了一个辅助方法, 来将stream转化为Iterable. 如果需要使用stream, 而方法只返回Iterable对象, 怎么办呢? 同样的这里需要进行转换.

```java
//Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

又这里可见, 如果知道代码的使用者要使用那个类型的对象(Stream还是Iterable), 直接进行返回即可. 但是如果不清楚的话, 那么用户就需要自己进行手动转换, 会带来额外的复杂度. 这时候就可以考虑使用Collection, Collection是Iterable的子类, 并且拥有stream()方法, 同时支持遍历和Stream. 因此**Collection是公共方法返回序列对象最好的实现**. 如果这些对象不多, 可以直接新建实例存储返回(TreeSet,HashSet), 但是如果数量特别大, 就不太适合了.  这时候推荐使用自定义的集合类型(通过AbstractList).

如计算一个集合的子集: {A, B, C}的子集为: {},{A},{B},{C},{A, B},{A, C},{B,C},{A,B,C}.对于N个元素的集合, 拥有2^N个子集, 这是指数集合的, 如果将一个不小的集合的所有子集放入内存中, 内存将会爆满. 这时候可以使用自定义的集合类来存储, 使用一个n个bit集合来存储所有的子集, 如果该bit的值来表示该位置的元素是否存在.

```java
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException("Set too big: " + s);
        return new AbstractList<Set<E>>() {
            @Override public int size() {
                return 1 << src.size();
            }
            @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set)o);
            }
            @Override public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            }
        };
    }
}
```

当然这也可以通过Stream实现, 求一个集合的所有子集, 可以从俩个个角度进行求解, 如{A, B, C}的前缀为: {A}, {A, B}, {A, B, C}. 后缀为: {A, B, C}, {B, C}, {C}. 那所有的子集为: 所有后缀的前缀.

```java
//Returns a stream of all the sublists of its input list
public class SubLists {
    public static <E> Stream<List<E>> of(List<E> first) {
        return Stream.concat(Stream.of(Collections.emptyList()), prefixes(list).flatMap(SubLists::suffixes));
    }

    private static <E> Stream<List<E>> prefixes (List<E> list) {
        return IntStream.rangeClosed(1, list.size())
            .mapToObj(end -> list.subList(0, end));
    }

    private static <E> Stream<List<E>> suffixes(List<E> list) {
        return IntStream.range(0, list.size())
            .mapToObj(start -> list.sublist(start, list.size()));
    }
}
```

另外Stream还能直接用来替换for-loop：

```java
for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end < src.size(); end++)
        System.out.println(src.subList(start, end));

public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
        .mapToObj(start ->
            IntStream.rangeClosed(start + 1, list.size())
            .mapToObj(end -> list.subList(start, end)))
        .flatMap(x -> x);
}
```

总而言之, 当你需要返回一系列的元素时, 需要考虑用户可能需要遍历, 也要考虑到Stream的需求. 因此最简单的方法就是直接返回集合类型. 如果元素的数量不是很多, 可以直接使用集合实例(如ArrayList)封装返回. 否则的话, 需要自定义集合进行返回. 如果返回集合不行的话, 那就可以使用Stream或者Iterable进行返回. 在未来的Java版本, 如果Stream实现了Iterable, 那就可以优先考虑返回Stream了.

## Item 48: Use caution when making streams parallel

在所有主流的编程语言中, 对于并发编程, Java一直是走在前面的. 在1996年发布时, 就提供并发支持: synchronization, wait/notify. 在JDK5中引入了`java.util.concurrent`包进行支持. Java7中引入了`fork-join`框架, 高性能的并发框架. Java8中引入了Stream也是支持并发的`parallel()`. 虽然并发编程越来越容易, 但是要写好并发程序却是一点也没有变容易. 这里看下之前的Item 45的代码:

```java
//Steam-based program to generate the first 20 Mersenne primes
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
    .filter(mersenne -> mersene.isProbablePrime(50))
    .limit(20)
    forEach(System.out::println);
}

static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

这段代码可以直接运行, 输出前20个质数. 偶尔, 这时我添加上并行化处理`parallel()`, 进行运算, 这时候会带来任何好的收益吗? 结果并没有, 程序卡住在哪里. 这是为什么? 因为数据的来源依赖`iterate`, 后一个值是依赖前面的值进行获取的, 也就是说每个并行线程中要获取下一个值就必须计算保留前一个值, 就相当于对每一个质数都计算了至少2次. 例外, 这里还限制了输出为20个, 对于多线程就必须控制线程之间的输出. 这里可以明显地知道: **并行化对于数据来源为iterate或者限制数据输出(中间操作包含limit)的stream不太可能带来性能上的优势**.

作为一般的准则, 并行化一般适用于数据来源为ArrayList, HashMap, HashSet, ConcurrentHashMap, 数组和int range, long range. 这些数据来源非常容易切分成不同的小块, 分配给不同的的线程进行分布式执行. 这些切分操作主要事依赖`spliterator`进行完成. 另外一个原因就是这些数据结构拥有良好的引用定位功能, 内部的元素的引用都是在一起的, 减少了定位引用的时间. 其中效果最好就是数组了.

同样的stream的终端操作同样会影响并行化的性能. 如一个数量很大的流, 对于内部的每一个元素, 计算时都需要和之前计算过的元素进行比较, 这就会非常影响性能. 最好的终端操作就是`reuction`, 每次都可以分布式的计算, 并且汇总到一个对象中. 如: `max`, `min`, `count`, `sum`, `anyMatch`, `allMatch`, `noneMatch`等, 就非常适合并行化处理.

并行化是一个非常严格的性能优化, 内部使用`fork-join`, 一个不规范的流操作都可能导致严重的性能损耗. 因此在使用时, 一定要经过严格地测试, 最好是在实际环境中进行测试, 保证性能达到想要的要求.

虽然并行化用起来非常困难, 也不是说一定要严格避免使用它. 只要在合适的环境下使用, 甚至可以给程序带来线性级别的优化, 而你需要的只是添加一个`parallel()`语句.

```java
//Take about 31 seconds in computer pi(10^8)
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}

//Take about 9.2 seconds in the same compute
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .parallel()
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```

当你需要在流中使用随机值时, 可以使用`SplittableRandom`而不是`ThreadLocalRandom`. 前者专门为流设计, 拥有差不多线性的加速. 后者是为单线程设计的, 并行时会因为争抢所有权而极大降低性能.

总而言之, 不要随便并行化处理流, 可能导致严重的性能问题或者更严重的效果. 如果你确定要使用的化, 一定要进行严格的测试, 当代码正确运行并且在真实的环境中可以达到你要的性能, 这时才可以放心的部署到生产环境中.
