# Lambdas and Streams
In Java8 引入了非常多的特性, 如功能接口, Lambdas, 方法引用等用来创建函数式对象. Streams相关的API则用来提供对数据的链式处理. 本章主要是介绍如何最大化地使用这些特性.

## Item 42: Prefer lambdas to anonymous classes.
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
//concise
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

## Item 43: Prefer method references to lambdas.
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

## Item 44: Favor the use of standard functional interfaces.