# Classes and Interfaces

类和接口是 Java 编程中的核心, 提供了基础的抽象模块.本章专注于如何使用类和接口来实现强壮的, 灵活的, 高可用的代码.

## Item 15: Minimize the accessibility of classes and members

程序设计的最基础的准则就是`封装(encapsulation)`, 即封装内部实现, 并对外提供单独的 API 接口.这有很多的好处, 其中最重要的当然就是`decoupe the components that comprise the system`, 可以对模块进行解耦.极大地方便了大型系统的开发, 测试和维护.当某一个模块出现问题之后, 可以很快地定位到对应 API 的对应模块进行修复, 而不用影响其它正常的模块.

Java 提供了很多辅助工具来帮助进行信息封装, 如权限控制`accessibility of classes, interface, and members`.可以通过合理的声明`private`, `default`, `public`, `protected`进行信息的封装.

简单点来说就是让访问权限尽可能的小, 尽可能的都为`private`(除了 public 的 API 接口), 如果包内其他类非要访问该变量, 就使用 default.如果这种情况经常发生, 就应该考虑重构代码(从别的角度来解决这个问题).

另外 Java 中还有一个规定, 如果子类重写了一个方法, 那么访问限制就必须等于大于父类的访问控制.这样可以保证声明为父类,实现为子类时的可用性.如果你不这么做, JVM 编译的时候会进行报错.

在程序中, 实例字段(Instance field)尽可能的减少`public`的使用.如果是 nonfinal 的(可变的), 首先是非线程安全的, 任何对象都可以访问都可以修改.其次丢失了灵活性, 你后面想要进行自定义修改的时候, 往往变得很难.这个建议往往适用于静态字段(static field), 但是这里有一个例外, 那就是`public static final`的不变静态字段对象, 这个对象往往是原始类型数据或者是指向不变类对象引用.

这里有一种特殊情况需要注意, 那就是非 0 长度的数组都是可变的, 如果我们将静态不变引用指向一个数组, 那这也是不安全的.解决方法有两种:

```java
//Potential security hole
public static final Thing[] VALUES = {...};

//Solution1
private static final Thing[] PRIVATE_VALUES = {...};
public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES);

//Solution2
private static final Thing[] PRIVATE_VALUES = {...};
public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
```

对于 Java9, 在`module system`添加了额外的两个访问控制: 包访问控制.模块封装了许多的 packages, 正如 package 封装了很多 classes.在模块声明中(module-info.java)显式声明那些模块可以进行访问(即导出), 那些模块是不可以在外部访问的.对于外部, 未导出的包是不可见的.或许现在 Java9 中的 module 还没有广泛的使用, 但是到了未来, 这是不可避免的.从现在开始将代码进行模块化管理, 是非常推荐的.

总而言之, 在 Class 中尽可能地减少访问权限, 然后预留最少的 API 接口, 减少流浪类接口或者成员变量.对于静态不变量保证引用的对象是不可变的.

## Item 16: In public classes, use accessor methods, not public field

有时候我们使用一个类来简单地封装一些属性:

```java
// Degenerate classes like this should not be public
class Pointer {
    public double x;
    public double y;
}
```

这很明显违背了`封装`的特性, 也享受不到`封装`的优点.这时候就会有封装的版本

```java
//Encapsulation of data by accessor methods and mutators
class Pointer {
    private double x;
    private double y;

    public Pointer(int x, int y) {
        this.x = x;
        this.y = y;
    }

    ...//Getter and Setter is omitted
}
```

很显然后者是更好的实现, 为内部实现提供了更大的灵活性.但是如果一个类只是内部使用, 不承担太大的功能时, 这种实现也没有明显的问题.前提是不要在外部进行使用或者导出.

在 Java 包中有很多类都违背了这个要求, 如 java.awt.Poiner 和 java.awt.Dimesion 等.这些都应该需要注意小心使用的.另外如果一个类中实例对象是不变的, 那这个代价将会大大减小.因为你不可以重新赋值, 保证了不变性.如:

```java
//Less harmful but still questionable
public final class Time {
    private static final int HOUR_PER_DAY = 24;
    private static final int MINITES_PER_HOUR = 60;

    public final int hour;
    public final int minite;

    public Time(int hour, int minute) {
        if (hour < 0 || hour >= HOUR_PER_DAY)
            throw new IllegalArgumentException("Hour": + hour);
        if (minute < 0 || minute >= MINITES_PER_HOUR)
            throw new IllegalArgumentException("Minute": + hour);
        this.hour = hour;
        this.minute = minute;
    }
    ...// Remaider is omitted
}
```

总而言之, 尽量减少对实例对象的公开.如果使用 final 进行公开, 可取但是仍然存在疑问.如果是内部使用的话, 可以考虑简单使用.

## Item 17: Minimize mutability

不变类(immutable class)是简单的类, 类的所有信息和属性都是固定的, 当你创建好的那一刻, 所有的属性和信息都固定了, 不会发生任何改变.Java 中提供了很多不变类的实例, 如封装原始类型的对象类, String, BigInteger 等等.不变类有很多好处: 容易实现, 设计和使用.它们有效减少了代码出错的概率.如何实现一个不变类, 可以参照下面五条规则:

- 不要提供修改属性(状态)的方法.
- 保证类是不可拓展的.需要防止子类进行继承进而修改状态.简单的方法就是设置为 final, 这是一个可选的解决方法.
- 对所有的对象和成员声明为 final.这样可以有效防止属性的变换, 特别是线程间共享时.
- 声明所有成员为 private, 这样可以有效防止当成员为一个可变对象的引用时, 别的类通过该引用直接修改引用的对象.
- 保证对可变对象的引用不会溢出, 即保证可变对象的引用一定要保证在类内部, 不会传递给外部从而被修改.

这里简单的用一个复数的例子来说明:

```java
//Immutable complex number class
pulic final class Complex {
    private final double re;
    private final double im;

    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public double realPart() { return re; }
    public double imaginaryPart() { return im; }

    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }

    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }

    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                            re * c.im + im * c.re);
    }

    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re - im * c.im} / tmp,
                            (re * c.im + im * c.re) / tmp);
    }

    @Override
    public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof Complex)) return false;
        Complex c = (Complex) o;
        return Double.compare(c.re, re) == o && Double.compare(c.im, im);
    }

    @Override
    public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }

    @Override
    public String toString() {
        return "(" + re + "," + im + ")";
    }
}
```

这里是一个简单的复数例子, 包含了实数部分和虚数部分, 每部分都提供了获取的方法, 并且提供了基本的四则运算支持.需要注意的是这里的四则运算都是返回一个新的 Complex 对象, 而不是修改原来的实例.这就是俗称的`函数式编程`, 因为函数并不会修改传递的值和本身, 保证了每次调用该方法都返回同一个对象.并且从方法名的定义就可以看出, 加法是`plus`, 而不是`add`, 强调方法不会修改原来的对象, 而是返回一个新的结果.

不可变类天生就是线程安全的, 不需要进行任何的同步操作.不可变类从创建出来状态就保持了一致, 不会进行任何修改.在多线程访问的情况下, 不会存在任何冲突, 也不会有访问时被修改的风险.因此不可变类也推荐进行缓存一些经常使用的实例, 避免重复构造对象, 来提高性能.如 Complex 中可以添加如下缓存:

```java
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE = new Complex(1, 0);
public static final Complex I = new Complex(0, 1);
```

并且还可以做得更多, 可以使用静态工厂方法来创建对象, 并在静态工厂方法中进行缓存操作, 如果对象已经被缓存了, 就可以返回别缓存的对象引用即可.Java 中有很多类都做了类似的操作, 如 Integer, Short, Long 等等.

另外不变类还有一个好处就是使用时不需要进行深拷贝, 因为不变类本身是不会改变的, 永远等于原先的值, 使用时不用担心修改的情况, 也就不需要进行拷贝.因此不可变类也不需要实现 clone()方法和复制类的方法进行复制操作.

不变类也是存在一个问题, 那就是性能上的缺陷: 每次修改一部分信息(即使只是一小部分信息)都会重新构建一个对象返回.这可能会带来一些性能上的问题, 特别是当你喜欢使用多个语句来构造一个对象时, 这个问题会显得很明显.那么有些解决方法呢.一般有两种, 一种是使用辅助构造对象, 如使用`Builder模式`进行构建对象, 只有在最后一步`build`的时候才会创建一个对象.另外一种方法就是使用伴生类, 如果你需要对一个不变类进行复杂的操作, 可以使用这个类的伴生类.如不变类 String 的伴生类 StringBuilder 和 StringBuffer 都是可变的, 如果需要进行复杂操作, 可以使用后两个.

最后来讨论一下设计的问题, 前面说到让一个类变得不可拓展, 最简单的方法就是声明为 final, 那么这个类就不会被子类继承.同样还有一个折中的方法, 那就是将所有的构造函数声明为 private, 然后使用一个静态函数进行实例化操作.这样可以保证该类不会被继承(因为没有提供构造函数给子类).如 Complex 类中:

```java
//Immutable complex number class with static factories
pulic class Complex {
    private final double re;
    private final double im;

    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }

    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }

    ...//Remainder unchanged
}
```

这种方法相比 final 提供了较大的灵活性, 可以添加一些自定义的操作.

有一些细节还需要注意的是, 在 BigInteger 和 BigDecimal 类书写的时候, 还不变类这个概念还没普及, 导致这两个类并没有声明为 final, 也存在 public 的构造函数, 这时候存在可能性:一个恶意的子类继承了它, 重写某些方法破坏了不变性.所以使用的时候, 需要检查一下是否是 BigInteger 还是它的子类.如果是子类则需要进行深拷贝.

```java
public static BigInteger safeInstance(BigInteger val) {
    return val.getClass() == BigInteger.class ? val : new BigInteger(val.toByteArray());
}
```

另外如果不变类实现了 Serializable 接口, 并且存在指向可变对象的引用, 那么就需要显示提供 readObject 和 readResolve 方法, 或者使用 ObjectOutputStream.writeUnshared 和 ObjectInputStream.readUnshared 方法.防止别的类利用序列化修改类的信息, 导致可变性(存在风险).

总而言之, 尽量将一个类设计为不必类, 在 Java 库中有许多类是可变的, 使用时需要注意.如果非要设计成可变类, 那就尽可能的减少可变性.

## Item 18: Favor composition over inheritance

继承是一个有效的方法来实现代码复用, 但是往往不是最好的方法.因为不合适的使用继承往往会导致脆弱的程序.继承在什么情况下是安全的呢? 在同一个包内使用继承, 父类和子类都由同一个程序员进行控制.或者有些类设计来就是用来继承的(如 abstract 类), 继承这些类是安全的.注意这里说的继承不涉及到接口的继承.

为什么不合适的继承会导致程序隐患呢? 继承破坏了代码的封装性.通过继承来实现一个子类, 就需要依赖父类的具体实现.而父类可能随着版本的更迭进行修改, 这时候子类的实现就可能遭到破坏, 除非你设计之初就考虑到了这个问题.如父类修改了某个方法, 而子类的实现又依赖这个方法的实现.那么这个子类将会是脆弱的.这就违背了安全使用的第一个原则, 你没办法控制父类的行为, 因为不是你写的父类.另外还有一种情况经常出现, 那就是父类添加了一些新的方法, 新添加的方法可能就会对子类造成安全隐患.如好几次爆出来的安全隐患就是因为, Java 更新了 HashTable 和 Vector, 添加了新的方法而导致的.

有人认为如果单纯的继承一个类, 不去重写父类的方法, 只添加新的方法就不会影响其安全性了吧?不, 这其实还是很危险的.如果父类后面的版本添加了新的方法, 而幸运你的方法和父类方法拥有相同的签名(Java 中的签名,方法名相同,参数相同), 返回不同的对象.那么你的代码将不会编译.但是如果你正好返回相同的对象, 那么父类则认为你重写了该方法, 你也无法保证你重写的方法正是父类所需要的.因为你无法预知父类的书写者的想法.

幸运的是, 有一个很好的解决方法来解决继承所带来的问题, 那就是使用组合: 即把你需要的类当做一个成员对象, 而不是继承它.每次调用的时候, 不直接使用, 而是通过这个成员对象来实现.这也就是经常上说的装饰模式.

唯一的缺点就是组合不适用于回调的形式(`callback frameworks`), 因为组合封装了一层, 封装的对象并不能正确的回调.另外使用继承时需要满足`is-a`关系, 当你要使用 B 继承 A 时, 问问你自己 B 真的是 A 吗? 如果不是, 那就不应该使用继承, 而应该使用组合.但是在 Java 自带的库中有很多都是违背了这个原则的, 如 Stack 就不是一个 Vector, 本来不应该继承 Vector 的, Property 不应该继承 HashTable 的等, 这些使用组合都是更加合适的.

使用组合时, 你可以封装内部实现, 提供了有限的 API 接口, 可以给你带来极大的灵活性.但是如果你继承的话, 就必须考虑父类接口和方法的问题, 必须维护相应的实现.并且把内部的实现暴露在外面了, 可能带来安全的隐患.如加上 p 是`Property`的一个实例, 调用`p.getProperty(key)`和`p.get(key)`明显返回的对象是不同的, 就会造成疑惑.Property 的设计是自能保存字符 key 和 value.但是继承的 HashTable 却没有这层限制, 如果直接通过 p 调用父类的方法存储非字符串, 那就会导致 Property 的接口出错.

总而言之, 继承是一个有用且强大的方法来实现代码复用, 但是容易出错.避免在不同的包中进行继承, 使用继承的时候, 问问自己是不是真的需要使用, 并且 B is really A?, 如果不是推荐使用组合而不是继承.组合的代码更加强壮且灵活.

## Item 19： Design and document for inheritance or else prohibit it

前面说过不要继承`外部`的类, 尤其是这些类本身不是设计成用来继承的.那什么类的是设计成继承的呢? 设计成继承的类又应该怎么实现呢?

如果一个类是被设计成用来继承的.那么就应该为每一个可重写的方法提供详细的描述, 并且在其余`public`或者`protected`方法中, 如果这些方法调用了其它的可重写的方法, 就应该在方法描述中显式说清楚.简单的说, 就是就是在任何用到可重写方法的地方都进行合理的说明和标注.

如果一个方法调用了可重写方法, 那么这个方法的描述中就应该单独列出一块区域描述内部是如何调用这个可重写方法的, 用`Implementation Requirements`来标示, Javadoc 中使用@implSpec 注释来说明.如`java.util.AbstractCollection`中的`public boolean remove(Object o)`方法的描述:

```js
This implementation iterates over the collection looking for the
specified element. If it finds the element, it removes the element
from the collection using the iterator's remove method.

Implementation Requirements: Note that this implementation throws an
UnsupportedOperationException.if the iterator returned by this
collection's iterator method does not implement the remove
method and this collection contains the specified object.
```

在这里的描述中就清楚的说明了内部的`iterator`方法可能影响到该方法: 如果子类重写的`iterator`没有实现`remove`方法就会抛出一个异常.

一个好的 API 文档不应该描述该 API 方法做了什么, 而不是该 API 方法怎么做? 这种特殊的描述是不是违背了这个准则呢? 是的, 但是没办法, 因为继承违背了`封装`特性, 为了让子类可以正确的继承, 只能牺牲一部分, 否则就只能保持`不确定性`.

`@implSpec`在 Java8 中被加入, Java9 中被广泛使用.但是却不是默认开启的, 如果要在 Javadoc 中开启注释, 需要添加参数: `-tag "apiNote:a:API Note:"`.

文档描述不仅仅可以用来描述内部的实现, 甚至为了效率可以在可重写方法描述中给出一些合理的建议, 来让程序员选择合适的版本.如`java.util.AbstractList`中的`protected void removeRange(int fromIndex, int toIndex)`方法.

```java
Removes from this list all of the elements whose index is between
{@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
Shifts any succeeding elements to the left (reduces their index).
This call shortens the list by {@code (toIndex - fromIndex)} elements.
(If {@code toIndex==fromIndex}, this operation has no effect.)

<p>This method is called by the {@code clear} operation on this list
and its subLists. Overriding this method to take advantage of
the internals of the list implementation can <i>substantially</i>
improve the performance of the {@code clear} operation on this list
and its subLists.

Implementation Requirements: <p>This implementation gets a list iterator
positioned before {@code fromIndex}, and repeatedly calls {@code ListIterator.next}
followed by {@code ListIterator.remove} until the entire range has
been removed. <b>Note: if {@code ListIterator.remove} requires linear
time, this implementation requires quadratic time.</b>
```

这个可重写方法提供了一个默认的实现版本, 但是效率不高.明确表示推荐子类重写该方法来实现一个高效的实现方法.

那在设计可继承类的时候, 该选择那些方法声明为`protected`(即可重写类型).这没有捷径可以走, 只能尽可能的考虑各种情况下子类的情况.最好的方法就是自己写子类来测试.一般的经验表明, 一般 3 个子类进行测试就足够了, 注意其中一到两个子类最好由其他人(非父类的撰写者)来书写测试.

还有一些其他的限制需要注意的是, 构造函数中不要直接或者间接调用`可重写方法`.因为子类的构造默认先调用父类的构造函数, 如果构造函数中调用了可重写方法, 但是这时候子类中重写的方法还没初始化完毕, 就有可能出错.如:

```java
public class Super {
    //Broken - constructor invokes an overridable method
    public Super() {
        overrideMe();
    }

    public void overrideMe() {

    }

    public static void main(String[] args) {
        Sub sub = new Sub();    //null
        sub.overrideMe();        //2018-10-13T12:11:59.966Z
    }
}

class Sub extends Super {
    private final Instant instant;

    Sub() {
        instant = Instant.now();
    }

    // Overriding method invoked by superclass constructor
    @Override
    public void overrideMe() {
        System.out.println(instant);
    }
}
```

上面可以清楚的知道, 对于 final 的 instant 对象输出了两个完全不同的值.其中一个为 null, 如果不是`System.out.println`这类可以接收`null`的方法, 就会抛出空指针异常.这样的程序是非常脆弱的, 因为无法保证子类的重写方式.

另外如果设计的类用来继承, 那么就尽量不要实现`Cloneable`和`Serializable`接口.最好的方式是留给子类去决定是否实现, 否则一旦父类实现了, 所有的子类不管有没有这个需求都必须维护对应的实现.如果非要实现的话, 那就类似构造函数(clone 和 readResolve,writeResolve 都是创建对象)不要在内部实现中直接或者间接调用可重写方法.理由同上.

到这里我们知道, 设计一个用来继承的类是需要花费很多功夫的, 存在着非常多的限制.可以采用一些辅助方法, 如抽象类, `skeletal implementations`等.如果一个类不是用于继承, 也不希望被继承.简单的方法就是设置为`final`或者设置所有的构造函数为`private`.如果需要使用和拓展一个不可继承的类, 推荐使用`组合`的方式进行拓展.

另外如果想要实现一个类可以安全的被继承, 即不让子类重写的方法影响到父类.技术上也是可以实现的, 将所有可重写的方法实现放到一个新的`private`的方法中, 然后父类所有调用可重写的方法都调用该`private`方法, 进行隔离.

总而言之, 实现一个用于继承的类是非常复杂的.必须在所有调用可重写方法的地方进行合理的标注, 并且给出可重写方法的详细描述, 并且进行永久维护.如果这个类没有被继承的需求, 声明为`final`或者声明所有的构造函数为`private`.

## Item 20: Prefer interface to abstract classes

在 Java 中提供了两种机制来实现类型声明, 允许不同实现: 接口和抽象类.随着`default method`的引入, 两种机制都允许内部定义和实现方法.唯一的区别就是, 抽象类只支持单继承, 如果需要使用抽象类的话, 就只能继承它.而接口没有这么多限制, 只要遵守对应的限制, 实现要求的方法即可声明实现接口.

在新的类中引入一个新的接口是非常容易的, 只要添加对应的方法实现, 遵守对应的限制, 然后在类定义中声明`implements xxx`即可.但是如果想让一个现有的类继承一个新的抽象类的话, 一般来说是非常困难的.大多数的类本身就有父类.如果两个类需要继承同一个类, 只有将该类当做两个类的父类, 让两个类同时继承.但是这样会带来一些副作用: 两个类的子类都会默认继承该类, 无论是否需要.

接口对于`混合类型(mixed)`是完美的实现, 混合类型: 即一个类在实现本身需求时, 另外提供一些可选操作.如`Comparable`接口暗示了这个类的实例可以进行有序的比较.这种添加可选的功能的接口, 一般称作`混合接口`.而对于抽象类则非常困难, 因为抽象类只能支持单继承.

另外接口还可以实现多继承, 这在类中是不可能实现的.如我们定义两个接口: Singer, SongWriter.

```java
public interface Singer {
    AudioClip sing(Song s);
}

public interface SongWriter {
    Song compose(int chartPosition);
}
```

而现实生活中有很多人即是歌手也是写曲人.这时候可以同时实现两个接口, 或者直接使用一个新的接口同时继承两个接口.

```java
public interface SingerSongWriter extends Singer, SongWriter {
    AudioClip strum();
    void actSensitive();
}
```

在这个新的接口中, 还可以添加一些新的方法.这提供了一种极大的灵活性.

接口还提供了安全的, 有用的功能增强方法: 通过 Item 18 介绍的组合方法.而对于抽象类的话, 就只能通过继承来实现.这种方式是脆弱的, 没有想象中的那么强大.

在 Java8 中引入了 default 方法, 给接口带来极大的便利性, 你可以在接口中添加自己想实现的方法来拓展接口的功能, 而那些实现了接口的类无需进行修改.虽然这种方式带来了很大的便利性, 但是 default 方法也有一定的限制.如接口中没办法实现 Objects 的一些默认方法, 如 equals, toString, hashCode 方法.另外接口中所有的对象都是 public static, 不可以存储实例对象(都是静态的).这些要求限制了 default 方法的范围, 只能在有限的区域内添加.

组合接口的优点和抽象类的优点, 也就产生了一种新的方式来实现类定义: 主干抽象类(Skeletal implementation class).即声明一个抽象类来实现对应的接口, 由接口实现主要的功能, 然后在抽象类中实现在接口中不能实现的方法.主干抽象类的命名规范一般为: Abstract + interface name.如在 java 集合框架中的: AbstractCollection, AbstractSet, AbstractList, AbstractMap 等等.合适定义主干抽象类, 可以很快的从这个类构建出自定义的类.

```java
static List<Integer> intArrayAsList(int[] a) {
    Objects.requireNonNull(a);
    return new AbstractList<>(){
        @Override
        public Integer get(int i){
            return a[i];
        }
        @Override
        public Inreger set(int i, Integer val) {
            int oldVal = a[i];
            a[i] = val;
            return oldVal;
        }
        @Override
        public int size() {
            return a.length;
        }
    };
}
```

这里使用简单的匿名类就可以构造出想要的对象, 就是借助了 AbstractList 的便利.主干抽象类提供了完备的支持, 使用起来也非常简单.最直接的方法就是继承它, 如果一个类没办继承它的话, 那直接实现该接口即可.甚至, 实现接口后, 在内部实现一个 private 的内部类继承自主干抽象类, 接口中定义的方法都可以重定向到内部类来实现.这种机制也就是俗称的`模拟多继承(simulated multiple inheritance)`.和组合有点类似, 原理都是相同的.提供了多继承的优点, 规避了其的缺点.

书写一个主干抽象类也是非常容易的.首先了解这个接口, 区分那些方法是主要的, 需要类自己实现的, 将将这些方法声明为 abstract.然后对于可实现的方法(包括 Objects 的一些方法, 如 toString, equals 等), 添加自己的实现.另外这个抽象类不像接口, 你可以添加任何你想要添加的实例, 方法来实现想要的功能.这里以`Map.Entry`接口为例:

```java
interface Entry<K,V> {
    K getKey();

    V getValue();

    V setValue(V value);

    boolean equals(Object o);

    int hashCode();

    //...Other is omitted
}
```

对应的主干抽象类为:

```java
public static class SimpleEntry<K,V>
        implements Entry<K,V>, java.io.Serializable
{
    private static final long serialVersionUID = -8499721149061103585L;

    private final K key;
    private V value;

    /**
     * Creates an entry representing a mapping from the specified
     * key to the specified value.
     *
     * @param key the key represented by this entry
     * @param value the value represented by this entry
     */
    public SimpleEntry(K key, V value) {
        this.key   = key;
        this.value = value;
    }

    /**
     * Creates an entry representing the same mapping as the
     * specified entry.
     *
     * @param entry the entry to copy
     */
    public SimpleEntry(Entry<? extends K, ? extends V> entry) {
        this.key   = entry.getKey();
        this.value = entry.getValue();
    }

    /**
     * Returns the key corresponding to this entry.
     *
     * @return the key corresponding to this entry
     */
    public K getKey() {
        return key;
    }

    /**
     * Returns the value corresponding to this entry.
     *
     * @return the value corresponding to this entry
     */
    public V getValue() {
        return value;
    }

    /**
     * Replaces the value corresponding to this entry with the specified
     * value.
     *
     * @param value new value to be stored in this entry
     * @return the old value corresponding to the entry
     */
    public V setValue(V value) {
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }

    /**
     * Compares the specified object with this entry for equality.
     * Returns {@code true} if the given object is also a map entry and
     * the two entries represent the same mapping. More formally, two
     * entries {@code e1} and {@code e2} represent the same mapping
     * if<pre>
     *   (e1.getKey()==null ?
     *    e2.getKey()==null :
     *    e1.getKey().equals(e2.getKey()))
     *   &amp;&amp;
     *   (e1.getValue()==null ?
     *    e2.getValue()==null :
     *    e1.getValue().equals(e2.getValue()))</pre>
     * This ensures that the {@code equals} method works properly across
     * different implementations of the {@code Map.Entry} interface.
     *
     * @param o object to be compared for equality with this map entry
     * @return {@code true} if the specified object is equal to this map
     *         entry
     * @see    #hashCode
     */
    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;
        return eq(key, e.getKey()) && eq(value, e.getValue());
    }

    /**
     * Returns the hash code value for this map entry. The hash code
     * of a map entry {@code e} is defined to be: <pre>
     *   (e.getKey()==null   ? 0 : e.getKey().hashCode()) ^
     *   (e.getValue()==null ? 0 : e.getValue().hashCode())</pre>
     * This ensures that {@code e1.equals(e2)} implies that
     * {@code e1.hashCode()==e2.hashCode()} for any two Entries
     * {@code e1} and {@code e2}, as required by the general
     * contract of {@link Object#hashCode}.
     *
     * @return the hash code value for this map entry
     * @see    #equals
     */
    public int hashCode() {
        return (key   == null ? 0 :   key.hashCode()) ^
               (value == null ? 0 : value.hashCode());
    }

    /**
     * Returns a String representation of this map entry. This
     * implementation returns the string representation of this
     * entry's key followed by the equals character ("<tt>=</tt>")
     * followed by the string representation of this entry's value.
     *
     * @return a String representation of this map entry
     */
    public String toString() {
        return key + "=" + value;
    }
}
```

这里的抽象主干类添加了 equals, toString 等方法的具体实现, 实现了接口中没办法实现的操作.另外添加了 Serializable 接口的支持.

这里需要注意的是, 抽象主干类是专门用于继承的, 所以需要遵守 Item 19 的要求, 为所有可重写方法提供良好的注释, 并且在所有调用的地方进行标注.这里为了页面简单, 就没有详细说明, 这点需要注意.

总而言之, 接口是实现类型定义很好的选择, 支持多继承等等.当你书写一个接口时, 推荐提供一个对应的主干抽象类来实现对应的功能.

## Item 21: Design interfaces for posterity

在 Java8 之前向接口中添加方法是不可接收的, 其它实现了这个接口的类, 会因为缺少对应的方法而出现编译问题.在 Java8 之后, 这变得不再是问题, 你可以添加 default 方法, 默认提供一个实现方法, 并不会影响现有的类.但是向现有的接口(之前存在的)添加默认方法还是存在一定的风险.

默认方法提供了一个默认的方法实现版本支持, 如果向现有的接口中添加 default 方法是非常危险的, 这是没有保障的.默认方法无法保证在任何条件下都可以在已有的代码中正确执行.并且这个添加是强制的, 没有进过任何使用者的同意.而在 Java8 之前, 默认的约定是接口不添加任何方法实现.

在 Java8 中有大量的默认方法被添加进集合框架中, 其中大部分是用于支持`Lambda`表达式, 其中大部分的代码都是精心设计的, 在一般情况下都可以良好运行.但是这也没办法保证在任何环境中都可以正确运行.如集合中的`removeIf`方法:

```java
//Default method added to the Collection interface in Java8
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean result = false;
    for (Iterator<E> it = iterator(); it.hasNext();) {
        if (filter.test(it.next())) {
            it.remove();
            result = true;
        }
    }
    retirm result;
}
```

`removeIf`方法接收一个 Predicate 类型的参数, 然后递归遍历集合中的所有的对象, 如果满足 predicate 的判断, 就进行移除.这个代码看起来是没有问题的, 在大多数情况下都可以正确运行.但是不幸的是, 在现实生活中还是存在意外: org.apache.commons.collections4.- collection.SynchronizedCollection.对于`SynchronizedCollection`, 除了提供基本的 java.util 中类似的功能, 还支持客户端传递对象进行同步操作.即内部所有的方法都是同步的, 在执行之前, 都需要获取对应对象的锁, 然后委派给内部实现进行操作.就是一个组合类, 或者称装饰类.到现在为止, 这个集合被广泛使用但是还没有重写`removeIf`方法.意味着如果在 Java8 的环境中, 该集合就会默认实现该方法, 而该方法破坏了该集合的承诺: 任何操作都是同步的.如果客户端不小心调用了该方法, 那么整个程序就很可能产生`ConcurrentModificationException`或者一些其他不可预料的行为.

为了防止这类事情的发生, 那这些代码的撰写者就必须手动重写这些默认方法, 这就非常依赖代码的维护人员了, 这是非常不可靠的.你无法确定他们什么时候会进行维护, 以致于现在很多代码还没有实现对应的默认方法.

另外现在的默认方法存在编译成功,但是运行失败的风险.虽然不是很常见, 但是还是存在这个风险的.在 Java8 发布之后, 很多现有的代码都受到了影响.

因此向现有的接口添加默认方法是非常危险的, 应该尽量避免.除非这个需求是非常重要的, 无法避免的.这时候也应该好好考虑下, 添加这个方法, 是否会对现有的实现带来影响.最好的方法还是创建接口之初, 就实现了这些默认方法, 提供默认的实现版本.需要注意的是, 默认方法的本质不是用来移除或修改现有的方法, 不应该打破现有的使用.因此在添加默认方法的时候, 需要非常小心.最好的测试方法还是书写不同的接口实现类来进行测试, 至少保证三个不同的版本的接口实现类, 最大限度的减少风险.虽然可以通过后续的发布进行修补问题, 但是你不能指望着它.

## Item 22: User interface only to define types

接口常常用来定义一种类型或者行为, 经常使用接口来声明一个对象, 具体的实现由具体实现该接口的类进行完成.这就是接口的设计目的.并且我们应该避免设计接口来做其他的事.其中一件经常做的事就是: 常量接口.即在接口中单纯地声明和存储常量, 如果一个类需要使用到这些常量的话, 就通过实现接口的方式可以简单获取到.如:

```java
public interface PhysicalConstants {
    //Avogaro's number (l/mol)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    //Boltzmann constant (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    //Mass of the electron (kg)
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

这是一种非常糟糕的实现, 这不仅会严重的污染了代码, 还容易会让人困惑.实现这个接口会将接口中所有的常量携带, 并且如果以后随着版本更迭, 如果修改了这个类不再需要接口中的参数, 那么为了兼容性还是需要实现该接口.并且如果任何一个非 final 的类实现了该接口, 那么它的所有子类都会默认实现该接口.这就造成了极大的代码污染.在 Java 库中有很多类似的使用, 如`java.io.ObjectStreamConstants`, 这些都是不应该去模仿的.

如果你想要导出静态常量, 这里有一些别的解决方法.如果这些常量是和类或者接口紧密相关的, 那么你应该直接写在类定义中.如`Integer`,`Double`中的`MIN_VALUE`和`MAX_VALUE`.如果这些变量最好以枚举的形式存储, 那么就定义为枚举类型.否则的话, 可以创建一个不可实例化工具类来存储这些常量即可.

```java
public class PhysicalConstants {
    private PhysicalConstants(){};    //Prevent instantiation

    pulic static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    //Boltzmann constant (J/K)
    pulic static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    //Mass of the electron (kg)
    pulic static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

注意这里数字采用了下标符进行书写, 这是在 Java7 后被支持的.推荐在位数较多的时候, 进行这样的书写, 可以极大的方便阅读.一般 3 位一个下标符号.并且如果需要对一个常量频繁调用的话, 可以使用一个方法进行封装, 减少常量名的书写次数.

```java
import xxx.PhysicalConstants;

public class Test {
    double atoms(double mols) {
        return AVOGADROS_NUMBER * mols;
    }

    ...//Other is omitted
}
```

总而言之, 接口应该只用于定义类型, 不应该用于其他用途.

## Item 23: Prefer class hierarchies to tagged classes

偶尔我们会设计一些类用于多个对象, 然后使用 tag 值来进行区分.如:

```java
class Figure {
    enum Shape {RECTANGLE, CIRCLE};

    //Tag field
    final Shape shape;

    //field for rectangle
    double length;
    double width;

    //field for circle
    double radius;

    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area(){
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
        }
    }
}
```

这种类也就是常称的标记类(`Tagged class`).这种类有很多缺点, 如混合了很多重复代码, 如枚举类型, tag field, `switch`语句等.可读性非常差.内部的`field`不能声明为 final, 除非初始化的时候也将不相干的属性初始化.内存的利用率也非常的低: 只用到了一部分的属性.并且修改起来非常容易出错.

幸运的是对于面向对象编程的 Java 来说, 有一个很好的解决方法, 那就是继承.标记类可以看做一种劣质的继承实现.首先需要声明一个抽象类定义抽象方法, 这个方法是依赖 tag 值进行实现的.如果上面 Figure 类只有一个: `area()`.如果还有其他相同的方法或者共有的属性可以放在该抽象父类中.然后用不同的子类存储属于自己的独一无二的属性和方法.

```java
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) { this.radius = radius; }

    @Override
    double area() {
        return Math.PI * (radius * radius);
    }
}

class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }

        @Override
    double area() {
        return length * width;
    }
}
```

这种通过继承的方式消除了标记类所有的缺点.减少了重复的代码, 避免被无关的属性拖累, 提供了极大的灵活性进行修改(通过继承), 所有的域都是 final 的(即不变类)等等.注意这里为了简化篇幅, 没有提供 public 的域获取方法, 而是通过属性直接获取.如果需要使用的时候, 推荐使用 public accessor 方法, 并将属性设为 private.

总而言之, 尽量减少标记类的使用, 如果你尝试去写标记类的时候, 考虑一下使用继承, 往往可以提供了一个更好的实现版本.

## Item 24: Favor static member classes over nonstatic

Java 中经常用到内部类, 而内部类的使用就是用于服务外部类.如果内部类需要在多个外部类中使用, 那推荐将这个类取出来, 声明为外部类使用.内部类主要分为: 静态成员类(`static member class`), 非静态成员类(`nonstatic member class`), 匿名类(`anonymous class`)和局部类(`local class`).

静态成员类是最简单的内部类版本, 作为一个普通的类声明在另一个类内部, 拥有外部类的所有成员的访问权限, 即使是 private.对于外部类来说, 和其它静态成员对象有类似的访问权限.如果声明为 private, 那么就只有外部类可以访问到.常用的用途是作为 public helper class, 为外部类提供服务.如在一个`Calculator`类内部定义一个`Operation`公共静态类, 这样外面的类使用时, 可以使用`Calculator.Operation.PLUS`来访问使用.

非静态成员类和静态成员类非常类似, 语法上唯一的区别就是没有`static`修饰符.但是本质上却非常不同, 一个非静态成员内部类实例肯定是关联一个外部类实例的.即如果创建一个外部类, 就会默认携带一个关联的非静态内部类的.对于每个实例来说是一一对应的.而静态成员内部类是不会关联具体的外部类实例的.非静态成员内部类与外部类之间的关联是在内部类实例化的时候就创建了, 后面是不能修改的.一般是通过外部类实例方法调用内部类的构造函数, 一般是不会通过`enclosingInstance.new MemberClass(args)`来进行构建的.正如你预期的, 这会来一些空间和时间上的负担.

非静态内部类的使用一般用于`Adapter`类型, 允许给父类提供一些额外的功能或者提供额外的辅助类.如:

```java
public class MySet<E> extends AbstractSet<E> {
    ...//other method is omitted

    @Override
    public Iterator<E> iterator() {
        return new MyIterator();
    }

    private class MyIterator implements Iterator<E> {
        ...
    }
}
```

如果你定义的内部类不需要访问外部类的成员对象, 那么就应该定义为静态的.如果定义为了非静态的, 则任何外部类的实例都会默认关联一个内部类的实例, 并且这种类型的内部类很难被垃圾回收, 只要外部类引用存活的话.

`private static member class`私有的成员内部类一般的用途是作为外部类的组件.如, Map 对象中的 Entry 类, map 中使用 entry 来存储每一个键值对, 然后存储 entry 数据.而 Entry 类的`getValue,getKey,setValue`等方法是不需要访问外部类 map 的内部对象的, 所以定义为 static, 而且单个 entry 是没有含义的, 组合成 map 才有作用.所以最终, `private static`就变成了最好的选择.

匿名类是没有名字的, 在声明的时候同时实例化.可以插入在任何块中(如语句,表达式,块代码).匿名类有非常多的限制, 如只能在声明的时候实例化, 不能使用`instanceof`判断, 不能同时实现多个接口或者继承类的同时实现接口, 代码长度不能太长,否则影响可读性等.在`Lambda`表达式出来之前, 匿名类常常用来完成一些简单的处理功能.但是现在`Lambda`表达式往往可以更好的胜任.现在主要用于静态工厂方法, 如前面的 Item20 的`intArrayAsList`.

局部类是四种类型中最少被使用的, 和匿名类一样, 只不过有了名字.可以在任何代码块中插入, 并且可以向普通类一样拥有成员等等.需要注意的是应该尽量保持简洁, 否则影响可读性.

总而言之, 四种不同的内部类各有特性, 适合不同的场所.如果一个内部类需要在外部可见, 或者代码过长, 使用成员内部类.如果成员内部类需要访问外部类的信息, 那就设置为非静态的, 否则就设置为静态的.如果一个类只用于一个方法内部或者静态工厂方法, 那就使用匿名类.否则就声明为局部类.

## Item 25: Limit source files to a single top-level class

Java 编译器是允许在一个.java 文件中创建多个顶级类的(将多个类并列声明在一个 java 文件中).但是不推荐这样做, 这样存在类重复定义的风险.

```java
//File 1 : Main.java
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Deseert.NAME);
    }
}

//File 2 : Utensil.java
//Two classes define in one file.
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}

//File 3 : Deseert.java
//Two classes define in one file.
class Utensil {
    static final String NAME = "pot";
}

class Dessert {
    static final String NAME = "pie";
}
```

当你运行`javac Main.java`或者`javac Main.java Utensil.java`时是打印出来: `pancake`.运行`javac Utensil.java Main.java`, 打印出来`pot pie`.如果运行`javac Main.java Dessert.java`, 这时候就会报重复定义的错误.编译器会在两个文件中都查找到相同的类定义.而出现这个问题是取决于你传递的顺序, 这是不可接受的.

如果要在同一个 java 文件中定义多个类也非常简单, 将一个类设为主类, 其余类设置成静态内部类即可.

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Deseert.NAME);
    }

    private static class Utensil {
        static final String NAME = "pan";
    }

    private static class Dessert {
        static final String NAME = "cake";
    }
}
```

总而言之, 不要在一个文件中定义多个顶级类, 防止出现重复定义.
