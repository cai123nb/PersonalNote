# Methods
本章主要讨论方法设计中的一些要素: 如何处理参数和返回值, 如何设计方法的签名, 如何文档化函数. 这些主要是关注于方法的可用性, 健壮性和灵活性.

## Item 49: Check parameters for validity.
大多数的方法都对传递的参数有特殊的要求, 如某个整数必须为整数, 对象引用不能为空. 对于这种情况, 一般都习惯性在方法的一开始进行校验, 如果违反就需要尽早抛出异常, 尽快失败. 为什么要这么做呢? 如果不进行参数校验, 很有可能方法在后面就会出现异常, 抛出一个困惑的异常, 让真正的问题更加难以定位. 更严重的是, 方法在后续的执行中不正常中断返回, 留下某些中间状态的对象, 从而导致别处出错.

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

在Java7中映入了`Objects.requireNonNull`方法进行空引用判断, 如果需要进行空判断, 也就没有必要手动获取了. 甚至, 该方法还支持之定义异常信息.

```java 
this.strategy = Objects.requireNonNull(strategy, "strategy");
```

在后面的版本中, Java9引入了`checkFromIndexSize, checkFromToIndex, checkIndex`等工具方法, 用于检测是否越界判断. 这些方法只能用于数组或者list, 并且并不可以进行异常信息的定制. 使用的并没有`requireNonNull`这么广. 在一些私有的内部方法中, 为了保证包内的方法的正确执行, 一般会用到`assertions`作为判断:

```java
//private helper function fro a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    ...// Do the computation
}
```

如果这些判断没有成功就会抛出`AssertionError`, 另外这些判断需要在命令行中手动开启`-ea(或者: -enableassertions)`.

当然这也有一些例外, 如对于参数验证是不现实的或者代价高昂的, 并且在计算的过程中会进行计算, 不进行检测也是可以的. 如`Collections.sort(arr)`, `arr`中所有的元素都要是`Comparable`的, 但是对内部所有的元素进行检查代价是非常高的, 并且在后续的计算中, 如果对象不是`Comparable`的, 进行比较时自动会抛出异常. 因此这里是不用使用的.

总而言之, 当我们定义一个方法的时候, 应该考虑好参数的限制, 对于这些限制应该在文档中进行说明, 并且在代码强制执行判断. 

## Item 50: Make defensive copies when needed.
相比别的语言, Java是属于类型安全的语言, 在没有本地方法调用的情况下, 一般不会出现数组越界, 野生指针等内存异常问题. 但是即使在这种情况下, 如果你不采用一些活动来隔离别的类, 你的类往往还是可能出现问题: 你的代码使用者会尽他们最大的力量来破坏你的代码. 因为使用者除了正常使用的人之外, 精通编程的人往往才是主要的攻击者.

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

为什么会出现这种问题? 因为Date类是可变的, 并不是不变类. 如果是JDK8的话, 修复的话可以使用(`LocalDateTime或者ZonedDateTime`)进行替换, 因为这两者是不变的.`Date`是过时且废弃的, 不应该出在现在的代码中. 有些情况下, 内部的成员是可变的, 如在JDK8之前的情况下使用`Date`, 那应该怎么办呢? 我们可以进行必要的复制操作:

```java
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (start.compareTo(end) > 0) {
        throw new IllegalArgumentException(start + " after " + end);
    }
}
```

这就可以预防前面的攻击, 注意这里的比较是在后面进行的, 防止多线程时比较后修改状态, 导致赋值的中间状态. 并且这里没有使用`Date.clone`方法进行拷贝, 因为Date的`clone`方法有可能返回该对象的子类, 并不一定是`Date`实类, 是不可信的.

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

通过对构造函数和Accessor函数进行对象拷贝使的`Period`类最终为一个完整的不变类. 这种技术不仅仅适用于不变类, 也适用于可变类. 如果当一个类定义一个方法或者构造函数需要存储外部传递一个对象引用时, 这时候需要考虑一下是不是可以接受, 传递的对象可能是可变的, 在使用的过程中可能会修改. 如果不能接受的话, 那就可以进行拷贝操作. 如: 将实例引用放入`Set`集合中, 如果后续的实例引用的值修改了, 就有可能导致集合出现冲突, 这时候就可以考虑进行拷贝.

同样的, 当需要给外部暴露内部的对象引用时, 需要考虑一下, 如果内部引用是可变的, 那么可以接受外部进行修改内部的值吗? 如果不行, 也可以进行拷贝操作. 当然也有一些特殊情况, 如果你相信外部的调用一定不会修改内部的值, 那么你可以不进行拷贝. 如外部的调用只允许在同一个包内, 并且该包是你自己负责撰写. 但是也是需要在文档中声明清楚的.

总而言之, 如果一个不变类需要获取外部的引用或者暴露内部的引用, 必须进行拷贝操作. 如果代价过于高或者可以保证调用的安全性, 那就需要在文档中进行显式的说明. 