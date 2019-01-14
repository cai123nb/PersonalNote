# Exceptions

合理的使用异常可以带来良好的代码可读性, 稳定性和维护性. 如果不合理地使用往往会带来相反的结果, 本章主要介绍如何合理的使用异常.

## Item 69: Use exceptions only for exceptional conditions

某一天当你不幸地路过一串复杂的代码:

```java
//Horrible abuse of exceptions. Don't ever do this
try {
    int i = 0;
    while (true) {
        range[i++].climb();
    }
} catch (ArrayIndexOutOfBoundsException e) {
}
```

咋一看, 有点懵逼. 仔细看看, 你就会发现这个方法就是简单的遍历`range`数组, 这段代码就是等价于:

```java
for (Mountain m : range) {
    m.climb();
}
```

这里为什么出现使用基于异常的循环语句呢? 这里简单地猜测一下, 很有可能是因为用户想对循环进行优化: JVM会在正常地每一次数组循环中进行范围校验, 而通过第一种方式可以省去这个流程进行优化. 但是这显然是过度优化了, 有三个理由可以接受为什么这么做是不对的.

+ 异常只用于异常检测环境, JVM不鼓励对其进行优化.
  
+ 异常地捕获需要放到`try catch`语句中, 而该语句中JVM虚拟机会禁止相关的优化措施.

+ 在标准的循环流程, 大部分的JVM都会优化掉范围检测操作.

事实也可以证明这一点, 经过测试, 第一个版本平均花费的时间约等于第二个版本的两倍(数组为100个).

使用基于异常的控制语句运行是非常慢的, 同时也是非常容易出错的. 如果在中途调用中`climb`, 不小心触发到了`ArrayIndexOutOfBoundsException`, 那么程序就会正常退出, 仿佛一点问题都没有. 但是实际上, 这是一段错误的执行, 很多元素都没有遍历到, 这个问题便被隐藏在内部, 非常难发现.

这个故事的结论非常简单: **不要使用异常来进行流程控制**. 更加广泛的说就是: 不要过度聪明地进行优化, 往往这会为了一点点的性能优化, 而导致代码的可读性和维护性非常差, 甚至会剥夺后续进行优化的可能.

这条准则也同样可以用于API设计: **API的设计不应该强制调用方使用异常来作为正常的控制流程**. 如果一个方法的调用必须基于特定的条件, 那么这时候可以进行条件检测(`State test`). 如`Iterator`的调用`next()`时可以使用`hasNext`进行条件检查:

```java
for (Iterator<Foo> i = collection.iterator(); i.hasNext();) {
    Foo foo = i.next();
    ...
}

//If lack hasNext
try {
    Iterator i = collection.iterator();
    while(true) {
        Foo foo = i.next();
        ...
    }
} catch (NoSuchElementException e) {
}
```

另一个可选的方法是返回`Optional`或者不同的值(如空的时候, 返回`null`其它的时候返回正常值). 这两种方法该怎么选择呢? 一般只有两种情况合适使用后者(不同值或者`Optional`), 一种就是如果方法有可能被多线程调用并且没有`synchronize`语句或者调用返回的结果有可能导致外部状态变化, 这时候就必须使用返回不同的值或者`Optional`. 另外一种情况就是对性能特别严苛的时候, 使用校验的话对性能有所浪费, 这时候也推荐使用这种方法. 而在其它情况下, 就推荐使用状态监测方法, 这种方法可以非常便利的保证方法的正确执行, 并且如果出错的话,也可以很快找到问题.

总而言之, 异常设计的本身就是用作异常环境. 不要使用异常来作为控制流程, API在也不要使用异常作为控制流程.

## Item 70: Use checked exceptions for recoverable conditions and runtime exceptions for programming errors

Java中将抛出的异常分为三种: 可检异常(`checked exceptions`), 运行时异常(`runtime exceptions`)和错误(`errors`).

> 错误: Error及其子类. 运行时异常: RuntimeException及其子类.可检异常: Throwable的子类, 但是不是RuntimeException和Error的子类.

那如何选择使用那种异常呢? 这里有一个简单的准则: **可检异常用于程序出现问题, 但是可以从问题(异常)中恢复的情况**. 通过抛出可检异常, 就强制让调用者进行捕获处理或者继续传播出去. 这是API的一个提示, 说明方法调用出现了问题. 虽然调用者可以捕获然后忽略它(放在catch语句中, 不进行处理), 但是这是不推荐的.

对于不可检异常主要分为两种: 运行时异常和错误. 一般出现这个问题的话, 就说明程序出现了问题, 程序基本上不太可能进行恢复或者后续继续执行的话, 做的事情往往坏处比好处更多.

**运行时异常常被用作程序错误**. 其中一个很大的用途就是来说明前置条件的违背. 当调用一个API方法时, 如果API抛出该异常, 则说明没有遵守API的限制. 如对数组的访问时, 就限定了下标值大于0并且小于数组的长度, 如果违背了就会抛出`ArrayIndexOutOfBoundsException`. 在选择可检异常和运行时异常时, 常常需要进行考量. 考量的标注就是时候可以从程序问题中进行恢复, 如果可以且没有影响的话, 那就使用`可检异常`. 如果不行的话, 那就使用`运行时异常`. 如果不能确定的话, 推荐还是使用`运行时异常`.

对于`Error`的使用, 虽然JLS没有强制的固定, 但是有一个通用的约束: 一般的`Error`常被用作JVM保留使用. 一般程序使用时, 应该尽量避免. 因此如果需要使用不可检异常时, 一般定义为`RuntimeException`的子类即可, 不推荐继承自`Error`.另外可检异常虽然直接继承`Throwable`即可, 但是不推荐(容易造成困扰), 推荐继承`Exception`, 注意不要继承`RuntimeException`.

总而言之, 抛出可检异常用于可恢复的情况, 抛出不可检异常用于不可恢复的情况. 当存在疑惑的时候, 推荐抛出不可检异常. 定义异常时不推荐直接继承自`Throwable`, 要不就是`Exception`, 要不就是`RuntimeException`.

## Item 71: Avoid unnecessary use of checked exceptions

许多程序员不喜欢`可检异常`, 但是如果使用恰当的话是可以提高代码可读性的, 如果使用不恰当的话, 会让API使用非常痛苦: 你必须在每一个抛出可检异常的方法进行`try catch`操作. 这是非常繁琐的, 特别是在Java8之后, 导致在Stream中使用非常不方便.

如果一个异常调用非常难恢复正常, 那么推荐使用不可检异常. 如果这个异常情况可以恢复, 并且你希望用户进行处理的. 一个可选的方法就是返回一个空的值(如Optional), 但是这个方法有一个缺点就是没办法知道原因, 如果抛出异常的话, 可以携带部分特定的信息. 另一种就是使用状态检测函数, 每次调用时, 强制先调用检测函数进行检查看是否可以进行处理.

总而言之, 如果保守地使用可检异常可以增加代码的可读性. 如果过度使用的话就会导致API使用非常通过. 如果该异常非常严重较难恢复, 推荐使用不可检异常. 如果可能进行回复, 那么首要的想法也是使用空值. 只要在空值无法提供足够信息的情况下, 再抛出可检异常.

## Item 72: Favor the use of standard exceptions

在我们使用异常的时候, 尽量重用Java自带的异常类, 这样做有很多好处:

+ 让API更加容易阅读和理解.

+ 重用异常类意味着出现时的内存路径短,易查看.

这里简要的介绍一些常用的异常类. `IllegalArgumentException`: 最常用的异常类之一, 用于传递给方法的参数不合理的时候. 如传递一个负数代表次数. `IllegalStateException`主要用于当方法调用时, 传递的对象或者内部的对象还没有初始完成. 通俗讲, 所有的异常的产生都可以归类为这两种情况: 传递的参数异常和方法调用时状态不对. 但是这两种归纳有些模糊, 后面Java库中定义了一些细分的异常类: `NullPointerException`: 用于说明对象引用为`null`. `IndexOutOfBoundsException`: 传递的索引值超过了数组的范围. `ConcurrentModificationException`: 如果一个单线程的对象或者方法被多线程访问的时候, 但是这个比较难进行判断. `UnsupportedOperationException`用于说明某些操作不被支持, 常用于实现某个接口或者继承某个类, 但是不想实现其中某个方法, 这时候就可以抛出这个异常.

这时候不要直接使用`Exception, RuntimeException, Throwable, Error`. 虽然这些类是可以实例化, 但是尽量不要直接使用. 另外有一些稀有的异常类: `ArithmeticException, NumberFormatException`也是准备了的, 但是用的比较少. 记住所有的异常都实现了序列化(`Throwable implements Serializable`).

总而言之, 尽可能多的复用Java内置的异常.

## Item 73: Throw exceptions appropriate to the abstraction

当我们看到一个异常时, 但是却发现和方法一点关系都没有. 如将文本写入到一个文件中, 却抛出一个`IndexOutOfBoundsException`异常. 这时候是非常困惑的. 这不仅会污染API, 还影响代码的可读性. 这时候一个可选的解决方法就是使用异常解释: 使用高级异常包裹低级异常.

```java
//Exception translation
try {
    ...//Do some low level operation.
} catch (LowerLevelException e) {
    throw new HigherLevelException(...);
}

//Example
public E get(int index) {
    ListIterator<E> i = listIterator(index);
    try {
        return i.next();
    } catch (NoSuchElementException e) {
        throw new IndexOutOfBoundsException("Index: " + index);
    }
}
```

这也就是常称的`异常链`. 这种方式可以很好的帮助开发人员发现问题进行调试. 在推荐实现高级异常的时候, 保存异常的起因, 这样有助于后续的调试.

```java
class HigherLevelException extends Exception {
    HigherLevelException(Throwable cause) {
        super(cause);
    }
}
```

虽然异常链可以起到一个很好的辅助作用. 但是这也是一种性能上的浪费, 要避免过度使用. 最好的做法就是避免它: 尽量保证低级代码的稳定性(尽量不抛出异常), 或者在低级代码的执行前进行校验, 保证一定可以正确执行(状态检查). 如果实在不能避免异常的出现, 最好在高级代码中进行静默处理, 可以通过`log`记录详情, 方便调试.

总而言之, 如果不能保证低级代码不会出现任何异常, 并且该异常不能很好的联系当前方法, 可以使用异常链进行封装.

## Item 74: Document all exception thrown by each method

正如标题所说, 为所有的异常进行文档注释. 这里遵守三个准则:

+ **为每一个可检异常进行单独的文档注释, 详细说明每一个可检异常出现的情况**. 这里特别忌讳直接`throws Exception`或者`throws Throwable`. 不强制要求文档化所有的运行时异常, 但是推荐在文档中进行详细的说明. 这个一般作为方法的前置条件违背时产生的结果. 这一点特别是在接口方法声明时特别重要.

+ **为每一个可检异常进行, `@throws`标识. 但是不必为运行时异常进行标识**. 这可以给每一个方法的调用者明确的信息, 这个方法有可能产生哪些可检异常, 需要单独进行处理. 为什么不推荐为运行时异常添加标识呢? 运行时异常一般不推荐进行捕获处理, 因为默认出现这种异常, 非常难进行恢复. 第二就是, 很难收集齐所有的运行时异常.

+ **如果一个异常被类内所有方法抛出, 可以在类注解中进行声明, 而不是在每一个类方法中进行声明**.

总而言之, 为方法内可能产生的异常都进行文档化注释, 并且为每一个可检异常进行`@throws`标注.

## Item 75: Include failure-capture information in detail messages

当方法出现一个异常时, 系统会自动打印出异常的堆栈信息, 而异常的堆栈信息是调用异常的`toString`方法生成的String信息. 这也是我们可以给外部提供信息的接口, 可以很好的帮助外部尽快定位问题点.

因此尽可能多地在异常内部提供信息, 可以起到很好的辅助作用. 如`IndexOutOfBoundsException`异常可以在内部存储上限, 下限, index值. 这样就可以很好的告诉用户那里出现了问题. 但是这里需要注意一点就是, 尽量不要包含一些私人, 加密信息, 如账号, 密码之类的信息. 另外这里提供的信息, 应该是在用户级别的信息, 尽可能的包含一些用户想知道的, 可以很快理解的信息. 其中一个方法就是自定义构造函数, 强制用户进行信息输入:

```java
public IndexOutOfBoundsException(int lowBound, int upperBound, int index) {
    super(String.format("Lower bound: %d, Upper boud: %d, Index: %d, lowerBound, upperBound, index", lowBound, upperBound, index));

    this.lowBound = lowBound;
    this.upperBound = upperBound;
    this.index = index;
}
```

总而言之, 为了提高异常的可用性, 尽量在异常内部包含足够的信息, 方便用户定位异常问题. 特别是可检异常, 因为这非常方便, 从异常中进行恢复.

## Item 76: Strive for failure atomicity

当一个对象抛出异常时, 一般都需要保证对象在异常前后状态是稳定的, 不变的. 这也就是`异常的原子性`: **一个对象或者方法的状态, 在出现异常之后, 前后的状态应该保持一致**. 也就是一次失败的方法调用(抛出异常), 不能修改对象的状态, 不能让它处在一个中间位置, 而应该是和之前的状态一致.

实现`异常的原子性`有很多方法, 其中一个最简单的方法就是, 使用不变类. 在这种情况下, 这个特性是自动实现的. 因为失败的方法调用只会阻止一个新的对象产生, 并不会对之前的对象状态后任何影响.

对于方法来说, 还有一个通用的解决方法就是, 在方法执行前, 进行状态监测: 如果不能进行执行, 那么就直接抛出异常, 不进行修改对象状态.

```java
public Object pop() {
    if (size == 0) {
        throw new EmptyStackException();
    }
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

这样就可以保证异常的原子性. 这里假设没有进行状态校验, 那么虽然也会抛出异常, 但是size的值却会被修改(变成负数), 改变了对象内部的状态: 调用的前后, 状态变化了. 后面的正常调用就可能受到影响. 还有一个类似的解决方法就是, 在可能发生异常的操作放在前面, 如果前面发生了异常, 对后面就有很大的影响.

第三个解决方案就是进行操作时拷贝一份操作数据, 然后进行处理, 处理成功了之后, 将拷贝的数据替换原来的数据. 如果没有成功, 也就影响了拷贝的数据, 不会影响之前调用时的数据. 如数组排序时, 通常是拷贝一份数组进行排序操作, 如果排序失败了, 也不会影响之前的数组顺序. 如果成功了, 就可以直接进行返回.

当发现需要保证异常的原子性, 但是现实并不能一直保证时. 如两个线程同时访问同一个对象, 修改内部的状态, 这时候就可能导致对象处于中间状态. 这时候如果检测到这种情况`ConcurrentModificationException`就应该作为系统错误进行抛出. 这时候已经不能保证对象的原子性了.

一般来说, 异常的原子性是不难实现的, 如果你发现实现这个的代价特别高, 又不是需要的话, 你可以忽略它.

总而言之, 如果一个方法内部可能抛出异常, 那就需要保证异常的原子性. 如果不能的话, 就需要在注释中说明清楚, 为什么, 调用之后对象会处在一种说明状态下.

## Item 77: Don't ignore exceptions

这看起来是非常明显的, 但是在现实中却又很多人习惯这么做. 

```java
try {
    ...
} catch (SomeException e) {

}
```

这是一种非常不好的习惯. 当一个异常被抛出的时候, 这是API在想你传递一个报警信息, 也是异常设计的初衷. **空的catch语句破坏了异常的设计目的**. 这就好比你在家, 火警报警铃响了, 这时候你直接关闭它, 回去继续睡觉. 这时候没人知道是不是真的着火了, 也不会收到警告. 每次看到空的catch语句时都应该脑子里响起警报.

当然也有一些特殊情况允许这么做, 如关闭`FileInputStream`时, 这时候你可能已经读取完了文件的信息, 不用在意Stream是否关闭失败了. 或者你没有修改任何文件的状态, 也就没有在意了. 这时候没有必要为了这个事让整个程序停下来. 当然最好的方法就是在日志中输出出来, 防止后面的查看. 这些情况都推荐在catch语句中进行注释说明为什么忽略这个异常, 好让代码阅读者明白你的意思.

总而言之, 不要忽略任何异常. 如果真的存在这种情况, 请在注释中进行说明, 并将异常出现的情况记录在日志中.