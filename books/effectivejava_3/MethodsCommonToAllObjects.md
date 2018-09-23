# Methods Common to All Objects
虽然Object类是一个具体的类, 但是主要设计出来还是用来进行拓展的. 对于Object类中的一些非不变(nofinal)的方法, 如equals, hashCode, toString, clone和finalize方法, 都是设计来进行覆盖的. 但是在覆盖的同时, 子类也是需要遵循这些方法的约定. 如果 不遵守的话可能导致一些别的类(依赖这些约定的类)使用时出现问题, 如HashMap, HashSet.

本章主要讲解如何正确地覆盖这些方法, 其中finalize方法在`Item8`中讲解了, 不推荐使用. 另外添加了一个新的, 不是Object的方法: Comparable.compareTo方法.

## Item 10: Obey the general contract when override equals
覆盖equals方法看起来非常简单, 但是特别容易出错, 导致的后果也是非常严重的. 最简单的方法就是不要覆盖, 这种情况下下就是单纯的比较实例对象是否相同(比较内存地址), 在以下这些情况中, 是推荐不覆盖的.

+ 每个类的实力对象都是独一无二的. 如每一个Thread对象的实例都是代表一个独一无二的对象, 代表着自己的特性, 因此Object默认的实现正是所需要的.
+ 当一个类没有需要提供`逻辑的等`的需求时. 如java.util.regex.Pattern本来可以覆盖该方法来提供验证两个Pattern对象是否代表同一个表达式, 但是设计者认为用户端是没有这个需求的, 也就不进行覆盖, 使用Object默认的也是可以的.
+ 如果父类已经覆盖了equals方法, 而父类覆盖的行为也适合当前类, 那么也没有必要进行覆盖. 如, Set中equals的实现方法就是继承自父类AbstractSet中的实现, List中的equals方法也是继承自父类AbstractList.
+ 如果一个类是私有的, 你可以百分百保证这个类不会被调用equals方法, 那么也是没有必要覆盖的. 甚至, 如果你为了防止别人碰巧调用了, 可以覆盖equals方法, 在内部抛出一个Error.

什么情况下需要覆盖equals方法呢? 这个类对象需要的更多的是逻辑上的`等操作`, 而不是实例对象的`等操作`(是否引用到同一个对象), 并且父类并没有覆盖equals方法. 这种类一般称为值类型. 值类型一般代表着值对象, 如Integer, String等. 程序员比较两个对象更加倾向于比较内部的值, 而不是外部的引用, 并且希望在Map中, Set中也是按照值比较来区分的话, 覆盖equals方法是一个很好的选择. 但是有一种值类型却不用遵守这个规则, 那就是枚举类型(Enum), 枚举类型通过`控制实例引用`, 保证一个值只有一个引用对象实例来完成这个需求, 对于这种类, Object中引用`等操作`和逻辑上的`等操作`是等价的, 所以也就没有必要覆盖equals方法.

当你覆盖equals方法时, 需要遵守如下约定:

+ Reflexive(自反性): 对于非null的对象引用x, x.equals(x)必须为true.
+ Symmetric(对称性): 对于非null的对象引用x和y, 如果 x.equals(y) 为true, 那么y.equals(x)也必须为true.
+ Transitive(传递性): 对于非null的对象引用x, y和z, 如果x.equals(y)为true, y.equals(z)为true, 那么x.equals(z)也必须为true. 
+ Consistent(一致性): 对于非null的对象引用x和y, 多次调用x.equals(y)必须返回相同的值, 要么全为true, 要么全为false, 即调用equals方法的过程中不能修改x和y内部的信息.
+ null判断: 对于非null的对象引用x, x.equals(null) 必须返回 false.

上面这些要求看起来有些繁琐和吓人, 但是千万不要忽视它们. 一旦违背了其中某条, 你会发现你的程序行为会变得不正确和容易崩溃, 同时这也非常难定位到问题所在. 正如John Donne所说, 没有类是孤岛. 所有的类都会传递给别的类, 而许多类(包括所有的集合类)都依靠传递过来的类需要遵循equals的约定.

虽然这些约定看起来有点吓人, 但是当你真正理解它们的时候, 你会发现遵守它们并不难. 那么什么是`等价关系`呢? 简单来说, 就是将一组元素划分为许多小组, 每个小组内的元素都必须要相等, 这些小组就是等价类对象. 那如何区分呢, 则是需要依靠equals方法. 下面就详细讲解一下equals的内部约定.
 
### Reflexive(自反性)
一个对象必须等于它自己, 很难想象如果你违背了这条准则, 会发生什么后果: 如果你将一个实例放到一个集合中去, 然而集合告诉你集合中没有这个元素.这是非常严重的.

### Symmetric(对称性)
任意两个对象必须满足相等的约定. 考虑如下的覆盖函数:
 
```java
//Broken - violates symmetry
public final class CaseInsensitiveString {
private final String s;

public CaseInsensitiveString(String s) {
	this.s = Objects.requireNonNull(s);
}

//Broken - violates symmetry
pubic boolean equals(Object o) {
	if (o instanceof CaseInsensitiveString)
		return s.equalsIgnoreCase((CaseInsensitiveString) o).s);
	if (o instanceof String)
		return s.equalsIgnoreCase((String) o);
	return false;
	}
}
```

这个方法明显违反了对称性, 如果有以下两个对象:

```java
String str = "abolish";
CaseInsensitiveString cis = new CaseInsensitiveString("Abolish");
```

当我们调用cls.equals(str)时, 肯定是返回true, 但是str.equals(cls)时明显返回false. 当你将cis放入到一个List对象中时, list.contain(str) ?, 谁也不确定. 碰巧在这个JDK中返回的是false, 但是这取决于你在List中的实现(cis.equals(str), 还是相反), 在别的实现中很容易就返回true, 并且出现运行时异常. 一旦你违反了这个等价的约定, 你不会知道别的对象和你的对象比较时, 会发生什么事. 

要解决这个问题也很简单, 简单排除造成困扰的判断:

```java
pubic boolean equals(Object o) {
	return o.instanceof CaseInsensitiveString && ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
 ```
 
### Transitivity(传递性)
如果第一个对象等于第二个对象, 第二个对象等于第三个对象, 那么第一个对象等于第三个对象. 考虑一下如下这种情况:

```java
public class Point {
private final int x;
private final int y;

public Point(int x, int y) {
	this.x = x;
	this.y = y;
}

public boolean equals(Object o) {
	if (!(o instanceof Point))
		return false;
	Point p = (Point) o;
	return p.x == x && p.y == y;
}
}
```

这是一个简单的二维点类, 假设你想拓展它, 添加一个新的属性:

```java
publc class ColorPoint extends Point {
private final Color color;

public ColorPoint(int x, int y, Color color) {
	super(x, y);
	this.color = color;
}
...
}
```
 
刚开始你不打算覆盖equals方法, 但是你很快发现问题, Color属性总是被忽略, 相同x,y不同的Color的ColorPoint总是返回true, 这是不能接受的. 于是你覆盖equals方法.

```java
//Broken - violates symmetry!
public boolean equals(Object o) {
if (!(o instanceof ColorPoint))
	return false;
retrun super.equals(o) && ((ColorPoint) o).color == color;
}
```
 
新的方法满足你的要求, 会比较三个属性. 但是却违背了对称性, Point.equals(ColorPoint)总是为true, 而反过来总是为false. 这个问题也是需要解决, 为了满足对称性, 你重写了equals方法.

```java
//Broken - violates transitivity
public boolean equals(Object o) {
if (!(o instanceof Point))
	return false;
if (!(o instanceof ColorPoint))
	return o.equals(this);
return super.equals(o) && ((ColorPoint) o).color == color;
}
```

这个解决方法虽然解决了对称性: 如果是比较Point, 就单纯的比较x和y, 否则还要比较color. 但是却破坏了传递性.

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```

p1.equals(p2)为true, p2.equals(p3)为true, 但是p1.equals(p3)为false. 很明显这违反了传递性. 并且这还容易导致无限循环. 如果有两个子类`ColorPoint`和`SmellPoint`都是有自己的单独属性, 那么比较的时候就会抛出StackOverflowError(无限循环卡在了第二步判断).

那么有什么解决方法呢? 事实上这是一个原理上的问题: 你没有办法通过继承一个类(可实例化的)来拓展值属性, 却可以遵守equals准则, 除非你放弃继承. 你可能听过通过getClass进行校验是否相同的解决方法:

```java
//Broken - violate Liskov substitution principle
public boolean equals(Object o) {
	if (o == null || o.getClass() != getClass())
		return false;
	Point p = (Point) o;
	return p.x == x && p.y == y;
}
```

这限制了所有的对象只能是同一个具体Class对象, 这同样会带来严重的副作用: 任何Pointer的子类, 都无法进行功能性的比较. 也就没办法用接口编程(父类编程), 它的结果关联了具体的实现类. `Liskov substitution principle`中说一个对象任何重要的属性可以被所有的子类所包含, 因此任何基于这些属性的方法都应该保持一致在所有的子类中. 

实际上还有一个迂回的解决方法, 那就是使用`组合`, 通过使用组合而不是继承可以很好的解决这个问题.

```java
public class Pointer {
	private final Point point;
	private final Color color;
	
	public ColorPoint(int x, int y, Color color) {
		point = new Pointer(x, y);
		this.color = Objects.requireNonNull(color);
	}
	
	public Point asPoint() {
		return point;
	}
	
	public boolean equals(Object o) {
		if (!(o instanceof ColorPoint))
			return false;
		ColorPoint cp = (ColorPoint) o;
		return cp.point.equals(point) && cp.color.equals(color);
	}
	
	...
}
```

在Java类库中有许多类通过继承一个可实例化的类来拓展属性, 如java.sql.Timestamp拓展自java.util.Date, 然后添加了一个新的属性nanoseconds. 其中equals的方法违背了对称性, 可能导致严重的后果, 我们应该在使用时注意这一点(不要放在一起使用). 

同样你可以通过继承一个不可实例化父类(抽象类)来拓展属性, 这是允许的, 且不会违背等价关系的. 因为父类没办法实例化, 也就不会违反对称性.

### Consistent(一致性)
如果两个对象是否相同, 它们必须长期保持一致, 除非中间修改了对象. 这就意味着可变对象可能不一定相同(过程中可以修改), 但是不可变对象就必须保证在任何情况下都要保持等价性(无法修改). 

无论一个类是否是可变的, equals方法都不能依靠不可靠的资源, 否则就很难保证维护这条要求. 如java.net.URL的equals方法依赖比较主机的IP地址, 但是获取IP地址需要网络连接, 不能保证可以一直正确获取. 所以URL的equals方法可能导致严重的后果, 应该避免使用equals方法.

### Non-nullity(非空性)
任何对象都必须不为null, 即所有的对象都要不等于null. 如果我们违反了这个约定, 我们的程序可能随时抛出NullPointerException, 并且很难找到原因. 一般为了保证这个条件都是在equals方法内添加限制:

```java
public boolean equals(Object obj) {
	if (obj == null) {
		return false;
	}
	...
}
```

这是一种显式的判断, 现在也存在一些非显式的判断, 可以更加优化这个功能: 调用instanceof进行判断.

```java
public boolean equals(Object obj) {
	if (obj instanceof XXX) {
		return false;
	}
	...
}
```

在instanceof函数中会自动进行空判断, 如果为空直接返回false, 并且可以对象是否为某个对象的实例.

### 总结
要如何实现合理的equals方法, 这里给出一些建议: 

+ 使用`==`操作来检查传递的对象是否指向同一个对象. 如果是, 返回true. 如果比较的代价较高的话, 可以显著提高性能.

+ 使用`instanceof`来检查传递的对象是否是同一个实例对象. 如果不是, 返回false. 需要注意的是`instanceof`同样支持实现接口. 有些类实现同一个接口, 依旧可以的.

+ 对传递的对象进行转换成正确的类型. 由于上一步的保证, 这一步是肯定可以成功的.

+ 对类对象内所有`重要的`的域, 进行比较. 如果全部都比较通过了, 返回true, 否则返回false. 对于这些域, 如果是int, byte等原始类型(除了flaoat和double)可以直接使用`==`进行值比较. 而对于对象类型, 则递归地调用该对象的equals方法. 对于float和double, 可以使用Float.compare(float, float), Double.compare(double, double)进行比较, 主要是处理flaot和double存在一些特殊值(如Float.NaN, -0.0f等). 这里不推荐使用Float.equals方法和Double的equals方法, 因为这会导致自动装箱. 对于数组类型, 推荐使用Arrays.equals方法, 进行验证. 如果有些对象类型允许空, 可以使用Objects.equals(Object, Object)进行比较. 对于有些类如果比较的代价非常高, 可以存储一个规范的标准域(`CanonicalForm`)(如, 对对象内所有的域取哈希码), 然后比较这个域是否相同, 而不是进行昂贵的比较. 但是这一般用于不变类. 

+ 当你完成equals方法的时候, 问下你自己这个方法是否满足对称性, 是否满足传递性, 是否满足一致性. 有时候一个合理的测试将会起到更好的校验效果.

同时这里有一些equals方法的注意事项: 

+ 覆盖equals方法的时候, 一定要覆盖hashCode方法. 

+ 不要过分强调equals, 对于一些域的判断还是非常容易实现的, 不要添加太多标示为来进行比较.

+ 不要替代equals方法中传递的`Object`参数. 有的人会替代成对应的实际类, 但是这是不推荐的. 首先这不会覆盖默认的equals方法(就算添加了`@Override`, 也会报错), 第二这就强调了比较的对象为具体的实际类, 不推荐这么使用.

有的时候我们可以借助一些工具来帮我们进行覆盖equals方法, 如`Google AutoValue`, `Lombok`等, 这些通过简单地添加一个注释就可以完成任务. 而注释生成的方法往往就是你想要的. 通过IDEs自动生成equals方法是不推荐的, 可读性不是很好, 并且没有办法动态生成(如果你修改了某个一属性, 你就需要重新生成).

总而言之, 不要覆盖`equals`方法, 一般来说默认的实现就是你要的. 如果你要覆盖的话, 一定要比较类中所有的域, 并且比较的时候要满足5大要求.

## Item 11: Always override hashCode when you override equals.
覆盖了`equals`方法但是却不覆盖`hashCode`可能导致严重的后果. 因为这违背了`hashCode`方法的基本准则, 而hashCode的准则为:

+ 保持一致性, 即多次调用hashCode方法, 只要通过equals方法保持为true, 就一定要返回一样的值.

+ 如果两个对象调用equals为true, 那么调用hashCode就一定要返回一样的值.

+ 如果两个对象不相同, 不要求hashCode返回的值一定相同, 但是尽量保持.

而我们覆盖`equals`方法但是却不覆盖`hashCode`的做法, 违背了第二条要求. 如新建两个对象, 赋予一样的成员变量(满足equals方法为true), 但是调用hashCode时是调用Object的hashCode方法(比较引用地址), 肯定是返回false的. 这里用一个例子来说明危害:

```java
Map<PhoneNumber, String> datas = new HashMap<PhoneNumber, String>();
datas.put(new PhoneNumber(172, 168, 32), "Jerry");
String name = datas.get(new PhoneNumber(172, 168, 32)); //null, 并不是 Jerry.
``` 

其中PhoneNumber就是覆盖了equals方法, 没有覆盖hashCode方法. 当我们查询的时候, 传递的是一个不同的hashCode值, 内部查询时从不同的桶进行查询(有可能一样). 

解决方法也很简单, 那就是覆盖一个合适的HashCode方法. 什么样的HashCode方法是合适的呢? 这里有一个简单的HashCode方法:

```java
public int hashCode() {
	return 42;
}
```

这是非常野蛮的, 但是可以使用. 因为这个方法满足了`hashCode`的所有限制. 但是这也可能导致严重的性能后果, 如在HashMap中根据对象的hashCode进行分桶存储, 但是由于hashCode全部相同, 就放到同一个桶里面. 导致HashMap原先的设计特性: 线性查询时间无法实现. 特别是对于包含大量数据的HashMap或者taable就可能导致无法进行查询的后果.

一个好的hashCode函数应该为每一个unequal的对象返回同一个hashCode值. 这是非常完美实现的, 但是可以近乎完美的实现, 这里提供一个实现方法:

+ `int result = first.hashCode()`, 初始化一个int对象result, 取第一个对象的hashCode值.

+ 然后对于剩余的每一个对象(域)取hashCode(c)进行组合.
	+ result = 31 * result + c;
	+ 其中一个域f是原始类型, 通过`Type.hashCode(f)`进行计算, `Type`是对应的封装型. 如: `Integer.hashCode(f)`.
	+ 如果这个域f是一个对象引用, 并且在equals中进行调用比较了, 调用对象的hashCode方法. 如果为null, 就使用0.
	+ 如果域f是一个数组, 并且数组中每个元素都非常重要, 就分离出来(分别调用对应hashCode方法)使用`Arrays.hashCode`进行生成hashCode值, 如果数组中没有成员变量, 使用0来代替.
	
+ 返回result.

## Item 12: Always override toString 
Object提供了原始的toString方法: 对象的类名 + `@` + 十六进制的哈希码(如:PhoneNumber@163b91). 这是非常令人困惑的, 也是很难理解的. `toString`方法本意是让我们返回一个简单而精确的描述字符串, 但这显然没有满足要求. 在Object的toString方法也显示告诉我们去重写这个方法: `It is  recommended that all subclasses override this method.`;

虽然不像`equal`和`hashCode`的限制一样, 但也是非常推荐进行重写该方法的. 不仅可以让类更加容易使用, 在调试的时候也能起到很好的作用. 如`707-867-5309`比`PhoneNumber@163b91`更加直接明了, 让人理解. 并且很多时候, 都会不自觉地调用toString方法: 如传递对象给`print`,`printf`, string的级联操作(如`+`), assert或者debuger中输出. 这些时候都会输出对应toString()的结果, 如果我们重写了该方法, 可以带来很好的帮助.

当重写toString方法的时候, 推荐包含所有对象内重要的信息. 有些特殊情况可以不满足这个: 如果一个对象太大了, 属性过多而无法实现. 或者就是一个对象内部的信息不适合用String来进行描述. 在这些特殊情况下推荐使用一些总结的话语, 如之前的`PhoneNumer`可以返回`Manhattan residential phone directory(1487536 listings)`. 一般这个总结应该是容易让人理解的.

同时在重写toString方法的时候, 你可以在toString的文档中显示表明返回的String的样式. 一旦你规定返回的样式, 那么返回的结果将会是唯一的, 没有异议的, 容易读取的. 同时一旦规定了返回的样式, 那么实现一个静态转换方法或者构造函数来接收这个样式的String进行转换也是非常好的. 就如同大多数原始类型封装类, `BigInteger`, `BidDecimal`等.

但是规定返回的样式也有一个缺点: 如果规定了样式, 那就就需要永久维护它. 如果这个类被广泛的使用, 别的程序员将会频繁使用这个方法, 甚至将String对象写入持久化层(如数据库)中去, 一旦你修改了返回的样式或者不兼容之前的格式, 那别人依赖这个方法的代码就会全部崩溃, 后果非常严重. 如果不规定返回的样式, 你会保留了修改的灵活性(自由添加修改属性).

无论你是否确定规定返回的格式, 都应该显示的在描述中说明返回的样式. 如果你规定了返回的样式, 那么这个需要更加的精确. 如Integer的`toString`实现方法:

```java
/**
 * Returns a string representation of the first argument in the
 * radix specified by the second argument.
 *
 * <p>If the radix is smaller than {@code Character.MIN_RADIX}
 * or larger than {@code Character.MAX_RADIX}, then the radix
 * {@code 10} is used instead.
 *
 * <p>If the first argument is negative, the first element of the
 * result is the ASCII minus character {@code '-'}
 * ({@code '\u005Cu002D'}). If the first argument is not
 * negative, no sign character appears in the result.
 *
 * <p>The remaining characters of the result represent the magnitude
 * of the first argument. If the magnitude is zero, it is
 * represented by a single zero character {@code '0'}
 * ({@code '\u005Cu0030'}); otherwise, the first character of
 * the representation of the magnitude will not be the zero
 * character.  The following ASCII characters are used as digits:
 *
 * <blockquote>
 *   {@code 0123456789abcdefghijklmnopqrstuvwxyz}
 * </blockquote>
 *
 * These are {@code '\u005Cu0030'} through
 * {@code '\u005Cu0039'} and {@code '\u005Cu0061'} through
 * {@code '\u005Cu007A'}. If {@code radix} is
 * <var>N</var>, then the first <var>N</var> of these characters
 * are used as radix-<var>N</var> digits in the order shown. Thus,
 * the digits for hexadecimal (radix 16) are
 * {@code 0123456789abcdef}. If uppercase letters are
 * desired, the {@link java.lang.String#toUpperCase()} method may
 * be called on the result:
 *
 * <blockquote>
 *  {@code Integer.toString(n, 16).toUpperCase()}
 * </blockquote>
 **/
```

如果没有规定特定的返回样式, 那也应该清楚表达你的意思:

```java
/**
 * Returns a brief description of this potion. the exact details of 
 * the representation are unspecified and subject to change,
 * but the following may be regarded as typical:
 *
 * "[Potion #9: type=love, smell=turpentine, look=india ink]"
 **/
```

当使用者看到这些注释的时候, 就知道这个格式可能会改变, 就不会进行格式的持久性的存储或转换.

无论你有没有规定格式, 在toString中返回的信息, 都应该提供一个显示的获取途径(`Accessor`). 不然的话, 程序员需要使用的时候, 需要自己手动去构造String对象, 这是非常容易出错的, 甚至导致系统崩溃. 同时不推荐为静态工具类重写toString方法, 同样也不推荐为`Enum`枚举类型重写该方法(Java库中已经为你准备一个很好的实现), 同时你应该在一个抽象类中去实现这个方法, 那么它的子类就可以进行共享. 如很多集合类的toString实现就是继承自父抽象类的实现.

在Google的`AutoValue`, `lombok`等自动化生成的`toString`方法, 一般不是完美的. 它们更加趋向于告诉别人内部的值. 而不是这个对象代表的含义. 推荐自己进行重写该方法.

总而言之, 为你每一个可实例化的对象重写`toString`方法, 除非父类已经重写了合适的格式. 这会让类更加容易使用和调试. 同时toString方法应该返回一个简洁漂亮的String格式.

