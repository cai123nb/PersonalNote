# Classes and Interfaces
类和接口是Java编程中的核心, 提供了基础的抽象模块. 本章专注于如何使用类和接口来实现强壮的, 灵活的, 高可用的代码.

## Item 15: Minimize the accessibility of classes and members
程序设计的最基础的准则就是`封装(encapsulation)`, 即封装内部实现, 并对外提供单独的API接口. 这有很多的好处, 其中最重要的当然就是`decoupe the components that comprise the system`, 可以对模块进行解耦. 极大地方便了大型系统的开发, 测试和维护. 当某一个模块出现问题之后, 可以很快地定位到对应API的对应模块进行修复, 而不用影响其它正常的模块. 

Java提供了很多辅助工具来帮助进行信息封装, 如权限控制`accessibility of classes, interface, and members`. 可以通过合理的声明`private`, `default`, `public`, `protected`进行信息的封装.

简单点来说就是让访问权限尽可能的小,  尽可能的都为`private`(除了public的API接口), 如果包内其他类非要访问该变量, 就使用default. 如果这种情况经常发生, 就应该考虑重构代码(从别的角度来解决这个问题). 

另外Java中还有一个规定, 如果子类重写了一个方法, 那么访问限制就必须等于大于父类的访问控制. 这样可以保证声明为父类,实现为子类时的可用性. 如果你不这么做, JVM编译的时候会进行报错. 

在程序中, 实例字段(Instance field)尽可能的减少`public`的使用. 如果是nonfinal的(可变的), 首先是非线程安全的, 任何对象都可以访问都可以修改. 其次丢失了灵活性, 你后面想要进行自定义修改的时候, 往往变得很难. 这个建议往往适用于静态字段(static field), 但是这里有一个例外, 那就是`public static final`的不变静态字段对象, 这个对象往往是原始类型数据或者是指向不变类对象引用.

这里有一种特殊情况需要注意, 那就是非0长度的数组都是可变的, 如果我们将静态不变引用指向一个数组, 那这也是不安全的. 解决方法有两种:

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

对于Java9, 在`module system`添加了额外的两个访问控制: 包访问控制. 模块封装了许多的packages, 正如package封装了很多classes. 在模块声明中(module-info.java)显式声明那些模块可以进行访问(即导出), 那些模块是不可以在外部访问的. 对于外部, 未导出的包是不可见的. 或许现在Java9中的module还没有广泛的使用, 但是到了未来, 这是不可避免的. 从现在开始将代码进行模块化管理, 是非常推荐的.

总而言之, 在Class中尽可能地减少访问权限, 然后预留最少的API接口, 减少流浪类接口或者成员变量. 对于静态不变量保证引用的对象是不可变的.

## Item 16: In public classes, use accessor methods, not public field
有时候我们使用一个类来简单地封装一些属性:

```java
// Degenerate classes like this should not be public
class Pointer {
	public double x;
	public double y;
}
```

这很明显违背了`封装`的特性, 也享受不到`封装`的优点. 这时候就会有封装的版本

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

很显然后者是更好的实现, 为内部实现提供了更大的灵活性. 但是如果一个类只是内部使用, 不承担太大的功能时, 这种实现也没有明显的问题. 前提是不要在外部进行使用或者导出.

在Java包中有很多类都违背了这个要求, 如java.awt.Poiner和java.awt.Dimesion等. 这些都应该需要注意小心使用的. 另外如果一个类中实例对象是不变的, 那这个代价将会大大减小. 因为你不可以重新赋值, 保证了不变性.如:

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

总而言之, 尽量减少对实例对象的公开. 如果使用final进行公开, 可取但是仍然存在疑问. 如果是内部使用的话, 可以考虑简单使用.

## Item 17: Minimize mutability
不变类(immutable class)是简单的类, 类的所有信息和属性都是固定的, 当你创建好的那一刻, 所有的属性和信息都固定了, 不会发生任何改变. Java中提供了很多不变类的实例, 如封装原始类型的对象类, String, BigInteger等等. 不变类有很多好处: 容易实现, 设计和使用. 它们有效减少了代码出错的概率. 如何实现一个不变类, 可以参照下面五条规则:

+ 不要提供修改属性(状态)的方法.
+ 保证类是不可拓展的. 需要防止子类进行继承进而修改状态. 简单的方法就是设置为final, 这是一个可选的解决方法.
+ 对所有的对象和成员声明为final. 这样可以有效防止属性的变换, 特别是线程间共享时.
+ 声明所有成员为private, 这样可以有效防止当成员为一个可变对象的引用时, 别的类通过该引用直接修改引用的对象.
+ 保证对可变对象的引用不会溢出, 即保证可变对象的引用一定要保证在类内部, 不会传递给外部从而被修改.

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

这里是一个简单的复数例子, 包含了实数部分和虚数部分, 每部分都提供了获取的方法, 并且提供了基本的四则运算支持. 需要注意的是这里的四则运算都是返回一个新的Complex对象, 而不是修改原来的实例. 这就是俗称的`函数式编程`, 因为函数并不会修改传递的值和本身, 保证了每次调用该方法都返回同一个对象. 并且从方法名的定义就可以看出, 加法是`plus`, 而不是`add`, 强调方法不会修改原来的对象, 而是返回一个新的结果.

不可变类天生就是线程安全的, 不需要进行任何的同步操作. 不可变类从创建出来状态就保持了一致, 不会进行任何修改. 在多线程访问的情况下, 不会存在任何冲突, 也不会有访问时被修改的风险. 因此不可变类也推荐进行缓存一些经常使用的实例, 避免重复构造对象, 来提高性能. 如Complex中可以添加如下缓存:

```java
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE = new Complex(1, 0);
public static final Complex I = new Complex(0, 1);
```

并且还可以做得更多, 可以使用静态工厂方法来创建对象, 并在静态工厂方法中进行缓存操作, 如果对象已经被缓存了, 就可以返回别缓存的对象引用即可. Java中有很多类都做了类似的操作, 如Integer, Short, Long等等.

另外不变类还有一个好处就是使用时不需要进行深拷贝, 因为不变类本身是不会改变的, 永远等于原先的值, 使用时不用担心修改的情况, 也就不需要进行拷贝. 因此不可变类也不需要实现clone()方法和复制类的方法进行复制操作.

不变类也是存在一个问题, 那就是性能上的缺陷: 每次修改一部分信息(即使只是一小部分信息)都会重新构建一个对象返回. 这可能会带来一些性能上的问题, 特别是当你喜欢使用多个语句来构造一个对象时, 这个问题会显得很明显. 那么有些解决方法呢. 一般有两种, 一种是使用辅助构造对象, 如使用`Builder模式`进行构建对象, 只有在最后一步`build`的时候才会创建一个对象. 另外一种方法就是使用伴生类, 如果你需要对一个不变类进行复杂的操作, 可以使用这个类的伴生类. 如不变类String的伴生类StringBuilder和StringBuffer都是可变的, 如果需要进行复杂操作, 可以使用后两个.

最后来讨论一下设计的问题, 前面说到让一个类变得不可拓展, 最简单的方法就是声明为final, 那么这个类就不会被子类继承. 同样还有一个折中的方法, 那就是将所有的构造函数声明为private, 然后使用一个静态函数进行实例化操作. 这样可以保证该类不会被继承(因为没有提供构造函数给子类). 如Complex类中:

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
	
	... //Remainder unchanged
}
```

这种方法相比final提供了较大的灵活性, 可以添加一些自定义的操作.

有一些细节还需要注意的是, 在BigInteger和BigDecimal类书写的时候, 还不变类这个概念还没普及, 导致这两个类并没有声明为final, 也存在public的构造函数, 这时候存在可能性:一个恶意的子类继承了它, 重写某些方法破坏了不变性. 所以使用的时候, 需要检查一下是否是BigInteger还是它的子类. 如果是子类则需要进行深拷贝.

```java
public static BigInteger safeInstance(BigInteger val) {
	return val.getClass() == BigInteger.class ? val : new BigInteger(val.toByteArray());
}
```

另外如果不变类实现了Serializable接口, 并且存在指向可变对象的引用, 那么就需要显示提供readObject和readResolve方法, 或者使用ObjectOutputStream.writeUnshared和ObjectInputStream.readUnshared方法. 防止别的类利用序列化修改类的信息, 导致可变性(存在风险).

总而言之, 尽量将一个类设计为不必类, 在Java库中有许多类是可变的, 使用时需要注意. 如果非要设计成可变类, 那就尽可能的减少可变性.

## Item 18: Favor composition over inheritance
继承是一个有效的方法来实现代码复用, 但是往往不是最好的方法.因为不合适的使用继承往往会导致脆弱的程序. 继承在什么情况下是安全的呢? 在同一个包内使用继承, 父类和子类都由同一个程序员进行控制. 或者有些类设计来就是用来继承的(如abstract类), 继承这些类是安全的. 注意这里说的继承不涉及到接口的继承.

为什么不合适的继承会导致程序隐患呢? 继承破坏了代码的封装性. 通过继承来实现一个子类, 就需要依赖父类的具体实现. 而父类可能随着版本的更迭进行修改, 这时候子类的实现就可能遭到破坏, 除非你设计之初就考虑到了这个问题. 如父类修改了某个方法, 而子类的实现又依赖这个方法的实现. 那么这个子类将会是脆弱的. 这就违背了安全使用的第一个原则, 你没办法控制父类的行为, 因为不是你写的父类. 另外还有一种情况经常出现, 那就是父类添加了一些新的方法, 新添加的方法可能就会对子类造成安全隐患. 如好几次爆出来的安全隐患就是因为, Java更新了HashTable和Vector, 添加了新的方法而导致的.

有人认为如果单纯的继承一个类,  不去重写父类的方法, 只添加新的方法就不会影响其安全性了吧?不, 这其实还是很危险的. 如果父类后面的版本添加了新的方法, 而幸运你的方法和父类方法拥有相同的签名(Java中的签名,方法名相同,参数相同), 返回不同的对象. 那么你的代码将不会编译. 但是如果你正好返回相同的对象, 那么父类则认为你重写了该方法, 你也无法保证你重写的方法正是父类所需要的. 因为你无法预知父类的书写者的想法.

幸运的是, 有一个很好的解决方法来解决继承所带来的问题, 那就是使用组合: 即把你需要的类当做一个成员对象, 而不是继承它.每次调用的时候, 不直接使用, 而是通过这个成员对象来实现. 这也就是经常上说的装饰模式. 

唯一的缺点就是组合不适用于回调的形式(`callback frameworks`), 因为组合封装了一层, 封装的对象并不能正确的回调. 另外使用继承时需要满足`is-a`关系, 当你要使用B继承A时, 问问你自己B真的是A吗? 如果不是, 那就不应该使用继承, 而应该使用组合. 但是在Java自带的库中有很多都是违背了这个原则的, 如Stack就不是一个Vector, 本来不应该继承Vector的, Property不应该继承HashTable的等, 这些使用组合都是更加合适的.

使用组合时, 你可以封装内部实现, 提供了有限的API接口, 可以给你带来极大的灵活性. 但是如果你继承的话, 就必须考虑父类接口和方法的问题, 必须维护相应的实现. 并且把内部的实现暴露在外面了, 可能带来安全的隐患. 如加上p是`Property`的一个实例, 调用`p.getProperty(key)`和`p.get(key)`明显返回的对象是不同的, 就会造成疑惑. Property的设计是自能保存字符key和value. 但是继承的HashTable却没有这层限制, 如果直接通过p调用父类的方法存储非字符串, 那就会导致Property的接口出错.

总而言之, 继承是一个有用且强大的方法来实现代码复用, 但是容易出错. 避免在不同的包中进行继承, 使用继承的时候, 问问自己是不是真的需要使用, 并且B is really A?, 如果不是推荐使用组合而不是继承. 组合的代码更加强壮且灵活.

## Item 19： Design and document for inheritance or else prohibit it
前面说过不要继承`外部`的类, 尤其是这些类本身不是设计成用来继承的. 那什么类的是设计成继承的呢? 设计成继承的类又应该怎么实现呢?

如果一个类是被设计成用来继承的. 那么就应该为每一个可重写的方法提供详细的描述, 并且在其余`public`或者`protected`方法中, 如果这些方法调用了其它的可重写的方法, 就应该在方法描述中显式说清楚. 简单的说, 就是就是在任何用到可重写方法的地方都进行合理的说明和标注.

如果一个方法调用了可重写方法, 那么这个方法的描述中就应该单独列出一块区域描述内部是如何调用这个可重写方法的, 用`Implementation Requirements`来标示, Javadoc中使用@implSpec注释来说明. 如`java.util.AbstractCollection`中的`public boolean remove(Object o)`方法的描述:

```
This implementation iterates over the collection looking for the
specified element.  If it finds the element, it removes the element
from the collection using the iterator's remove method.

Implementation Requirements: Note that this implementation throws an
UnsupportedOperationException. if the iterator returned by this
collection's iterator method does not implement the remove
method and this collection contains the specified object.
```

在这里的描述中就清楚的说明了内部的`iterator`方法可能影响到该方法: 如果子类重写的`iterator`没有实现`remove`方法就会抛出一个异常.

一个好的API文档不应该描述该API方法做了什么, 而不是该API方法怎么做? 这种特殊的描述是不是违背了这个准则呢? 是的, 但是没办法, 因为继承违背了`封装`特性, 为了让子类可以正确的继承, 只能牺牲一部分, 否则就只能保持`不确定性`.

`@implSpec`在Java8中被加入, Java9中被广泛使用. 但是却不是默认开启的, 如果要在Javadoc中开启注释, 需要添加参数: `-tag "apiNote:a:API Note:"`.

文档描述不仅仅可以用来描述内部的实现, 甚至为了效率可以在可重写方法描述中给出一些合理的建议, 来让程序员选择合适的版本. 如`java.util.AbstractList`中的`protected void removeRange(int fromIndex, int toIndex)`方法.

```java
Removes from this list all of the elements whose index is between
{@code fromIndex}, inclusive, and {@code toIndex}, exclusive.
Shifts any succeeding elements to the left (reduces their index).
This call shortens the list by {@code (toIndex - fromIndex)} elements.
(If {@code toIndex==fromIndex}, this operation has no effect.)

<p>This method is called by the {@code clear} operation on this list
and its subLists.  Overriding this method to take advantage of
the internals of the list implementation can <i>substantially</i>
improve the performance of the {@code clear} operation on this list
and its subLists.

Implementation Requirements: <p>This implementation gets a list iterator 
positioned before {@code fromIndex}, and repeatedly calls {@code ListIterator.next}
followed by {@code ListIterator.remove} until the entire range has
been removed.  <b>Note: if {@code ListIterator.remove} requires linear
time, this implementation requires quadratic time.</b>
```

这个可重写方法提供了一个默认的实现版本, 但是效率不高. 明确表示推荐子类重写该方法来实现一个高效的实现方法.

那在设计可继承类的时候, 该选择那些方法声明为`protected`(即可重写类型). 这没有捷径可以走, 只能尽可能的考虑各种情况下子类的情况. 最好的方法就是自己写子类来测试. 一般的经验表明, 一般3个子类进行测试就足够了, 注意其中一到两个子类最好由其他人(非父类的撰写者)来书写测试. 

还有一些其他的限制需要注意的是, 构造函数中不要直接或者间接调用`可重写方法`. 因为子类的构造默认先调用父类的构造函数, 如果构造函数中调用了可重写方法, 但是这时候子类中重写的方法还没初始化完毕, 就有可能出错. 如:

```java
public class Super {
    //Broken - constructor invokes an overridable method
    public Super() {
        overrideMe();
    }

    public void overrideMe() {

    }

    public static void main(String[] args) {
        Sub sub = new Sub();	//null
        sub.overrideMe();		//2018-10-13T12:11:59.966Z
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

上面可以清楚的知道, 对于final的instant对象输出了两个完全不同的值. 其中一个为null, 如果不是`System.out.println`这类可以接收`null`的方法, 就会抛出空指针异常. 这样的程序是非常脆弱的, 因为无法保证子类的重写方式.

另外如果设计的类用来继承, 那么就尽量不要实现`Cloneable`和`Serializable`接口. 最好的方式是留给子类去决定是否实现, 否则一旦父类实现了, 所有的子类不管有没有这个需求都必须维护对应的实现. 如果非要实现的话, 那就类似构造函数(clone和readResolve,writeResolve都是创建对象)不要在内部实现中直接或者间接调用可重写方法. 理由同上. 

到这里我们知道, 设计一个用来继承的类是需要花费很多功夫的, 存在着非常多的限制. 可以采用一些辅助方法, 如抽象类, `skeletal implementations`等. 如果一个类不是用于继承, 也不希望被继承. 简单的方法就是设置为`final`或者设置所有的构造函数为`private`. 如果需要使用和拓展一个不可继承的类, 推荐使用`组合`的方式进行拓展.

另外如果想要实现一个类可以安全的被继承, 即不让子类重写的方法影响到父类. 技术上也是可以实现的, 将所有可重写的方法实现放到一个新的`private`的方法中, 然后父类所有调用可重写的方法都调用该`private`方法, 进行隔离.

总而言之, 实现一个用于继承的类是非常复杂的. 必须在所有调用可重写方法的地方进行合理的标注, 并且给出可重写方法的详细描述, 并且进行永久维护. 如果这个类没有被继承的需求, 声明为`final`或者声明所有的构造函数为`private`.

## Item 20: Prefer interface to abstract classes 
