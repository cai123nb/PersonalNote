# Methods Common to All Objects

虽然 Object 类是一个具体的类, 但是主要设计出来还是用来进行拓展的.对于 Object 类中的一些非不变(nofinal)的方法, 如 equals, hashCode, toString, clone 和 finalize 方法, 都是设计来进行覆盖的.但是在覆盖的同时, 子类也是需要遵循这些方法的约定.如果 不遵守的话可能导致一些别的类(依赖这些约定的类)使用时出现问题, 如 HashMap, HashSet.

本章主要讲解如何正确地覆盖这些方法, 其中 finalize 方法在`Item8`中讲解了, 不推荐使用.另外添加了一个新的, 不是 Object 的方法: Comparable.compareTo 方法.

## Item 10: Obey the general contract when override equals

覆盖 equals 方法看起来非常简单, 但是特别容易出错, 导致的后果也是非常严重的.最简单的方法就是不要覆盖, 这种情况下下就是单纯的比较实例对象是否相同(比较内存地址), 在以下这些情况中, 是推荐不覆盖的.

- 每个类的实力对象都是独一无二的.如每一个 Thread 对象的实例都是代表一个独一无二的对象, 代表着自己的特性, 因此 Object 默认的实现正是所需要的.
- 当一个类没有需要提供`逻辑的等`的需求时.如 java.util.regex.Pattern 本来可以覆盖该方法来提供验证两个 Pattern 对象是否代表同一个表达式, 但是设计者认为用户端是没有这个需求的, 也就不进行覆盖, 使用 Object 默认的也是可以的.
- 如果父类已经覆盖了 equals 方法, 而父类覆盖的行为也适合当前类, 那么也没有必要进行覆盖.如, Set 中 equals 的实现方法就是继承自父类 AbstractSet 中的实现, List 中的 equals 方法也是继承自父类 AbstractList.
- 如果一个类是私有的, 你可以百分百保证这个类不会被调用 equals 方法, 那么也是没有必要覆盖的.甚至, 如果你为了防止别人碰巧调用了, 可以覆盖 equals 方法, 在内部抛出一个 Error.

什么情况下需要覆盖 equals 方法呢? 这个类对象需要的更多的是逻辑上的`等操作`, 而不是实例对象的`等操作`(是否引用到同一个对象), 并且父类并没有覆盖 equals 方法.这种类一般称为值类型.值类型一般代表着值对象, 如 Integer, String 等.程序员比较两个对象更加倾向于比较内部的值, 而不是外部的引用, 并且希望在 Map 中, Set 中也是按照值比较来区分的话, 覆盖 equals 方法是一个很好的选择.但是有一种值类型却不用遵守这个规则, 那就是枚举类型(Enum), 枚举类型通过`控制实例引用`, 保证一个值只有一个引用对象实例来完成这个需求, 对于这种类, Object 中引用`等操作`和逻辑上的`等操作`是等价的, 所以也就没有必要覆盖 equals 方法.

当你覆盖 equals 方法时, 需要遵守如下约定:

- Reflexive(自反性): 对于非 null 的对象引用 x, x.equals(x)必须为 true.
- Symmetric(对称性): 对于非 null 的对象引用 x 和 y, 如果 x.equals(y) 为 true, 那么 y.equals(x)也必须为 true.
- Transitive(传递性): 对于非 null 的对象引用 x, y 和 z, 如果 x.equals(y)为 true, y.equals(z)为 true, 那么 x.equals(z)也必须为 true.
- Consistent(一致性): 对于非 null 的对象引用 x 和 y, 多次调用 x.equals(y)必须返回相同的值, 要么全为 true, 要么全为 false, 即调用 equals 方法的过程中不能修改 x 和 y 内部的信息.
- null 判断: 对于非 null 的对象引用 x, x.equals(null) 必须返回 false.

上面这些要求看起来有些繁琐和吓人, 但是千万不要忽视它们.一旦违背了其中某条, 你会发现你的程序行为会变得不正确和容易崩溃, 同时这也非常难定位到问题所在.正如 John Donne 所说, 没有类是孤岛.所有的类都会传递给别的类, 而许多类(包括所有的集合类)都依靠传递过来的类需要遵循 equals 的约定.

虽然这些约定看起来有点吓人, 但是当你真正理解它们的时候, 你会发现遵守它们并不难.那么什么是`等价关系`呢? 简单来说, 就是将一组元素划分为许多小组, 每个小组内的元素都必须要相等, 这些小组就是等价类对象.那如何区分呢, 则是需要依靠 equals 方法.下面就详细讲解一下 equals 的内部约定.

### Reflexive(自反性)

一个对象必须等于它自己, 很难想象如果你违背了这条准则, 会发生什么后果: 如果你将一个实例放到一个集合中去, 然而集合告诉你集合中没有这个元素.这是非常严重的.

### Symmetric(对称性)

任意两个对象必须满足相等的约定.考虑如下的覆盖函数:

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

当我们调用 cls.equals(str)时, 肯定是返回 true, 但是 str.equals(cls)时明显返回 false.当你将 cis 放入到一个 List 对象中时, list.contain(str) ?, 谁也不确定.碰巧在这个 JDK 中返回的是 false, 但是这取决于你在 List 中的实现(cis.equals(str), 还是相反), 在别的实现中很容易就返回 true, 并且出现运行时异常.一旦你违反了这个等价的约定, 你不会知道别的对象和你的对象比较时, 会发生什么事.

要解决这个问题也很简单, 简单排除造成困扰的判断:

```java
pubic boolean equals(Object o) {
    return o.instanceof CaseInsensitiveString && ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

### Transitivity(传递性)

如果第一个对象等于第二个对象, 第二个对象等于第三个对象, 那么第一个对象等于第三个对象.考虑一下如下这种情况:

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

刚开始你不打算覆盖 equals 方法, 但是你很快发现问题, Color 属性总是被忽略, 相同 x,y 不同的 Color 的 ColorPoint 总是返回 true, 这是不能接受的.于是你覆盖 equals 方法.

```java
//Broken - violates symmetry!
public boolean equals(Object o) {
if (!(o instanceof ColorPoint))
    return false;
retrun super.equals(o) && ((ColorPoint) o).color == color;
}
```

新的方法满足你的要求, 会比较三个属性.但是却违背了对称性, Point.equals(ColorPoint)总是为 true, 而反过来总是为 false.这个问题也是需要解决, 为了满足对称性, 你重写了 equals 方法.

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

这个解决方法虽然解决了对称性: 如果是比较 Point, 就单纯的比较 x 和 y, 否则还要比较 color.但是却破坏了传递性.

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```

p1.equals(p2)为 true, p2.equals(p3)为 true, 但是 p1.equals(p3)为 false.很明显这违反了传递性.并且这还容易导致无限循环.如果有两个子类`ColorPoint`和`SmellPoint`都是有自己的单独属性, 那么比较的时候就会抛出 StackOverflowError(无限循环卡在了第二步判断).

那么有什么解决方法呢? 事实上这是一个原理上的问题: 你没有办法通过继承一个类(可实例化的)来拓展值属性, 却可以遵守 equals 准则, 除非你放弃继承.你可能听过通过 getClass 进行校验是否相同的解决方法:

```java
//Broken - violate Liskov substitution principle
public boolean equals(Object o) {
    if (o == null || o.getClass() != getClass())
        return false;
    Point p = (Point) o;
    return p.x == x && p.y == y;
}
```

这限制了所有的对象只能是同一个具体 Class 对象, 这同样会带来严重的副作用: 任何 Pointer 的子类, 都无法进行功能性的比较.也就没办法用接口编程(父类编程), 它的结果关联了具体的实现类.`Liskov substitution principle`中说一个对象任何重要的属性可以被所有的子类所包含, 因此任何基于这些属性的方法都应该保持一致在所有的子类中.

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

在 Java 类库中有许多类通过继承一个可实例化的类来拓展属性, 如 java.sql.Timestamp 拓展自 java.util.Date, 然后添加了一个新的属性 nanoseconds.其中 equals 的方法违背了对称性, 可能导致严重的后果, 我们应该在使用时注意这一点(不要放在一起使用).

同样你可以通过继承一个不可实例化父类(抽象类)来拓展属性, 这是允许的, 且不会违背等价关系的.因为父类没办法实例化, 也就不会违反对称性.

### Consistent(一致性)

如果两个对象是否相同, 它们必须长期保持一致, 除非中间修改了对象.这就意味着可变对象可能不一定相同(过程中可以修改), 但是不可变对象就必须保证在任何情况下都要保持等价性(无法修改).

无论一个类是否是可变的, equals 方法都不能依靠不可靠的资源, 否则就很难保证维护这条要求.如 java.net.URL 的 equals 方法依赖比较主机的 IP 地址, 但是获取 IP 地址需要网络连接, 不能保证可以一直正确获取.所以 URL 的 equals 方法可能导致严重的后果, 应该避免使用 equals 方法.

### Non-nullity(非空性)

任何对象都必须不为 null, 即所有的对象都要不等于 null.如果我们违反了这个约定, 我们的程序可能随时抛出 NullPointerException, 并且很难找到原因.一般为了保证这个条件都是在 equals 方法内添加限制:

```java
public boolean equals(Object obj) {
    if (obj == null) {
        return false;
    }
    ...
}
```

这是一种显式的判断, 现在也存在一些非显式的判断, 可以更加优化这个功能: 调用 instanceof 进行判断.

```java
public boolean equals(Object obj) {
    if (obj instanceof XXX) {
        return false;
    }
    ...
}
```

在 instanceof 函数中会自动进行空判断, 如果为空直接返回 false, 并且可以对象是否为某个对象的实例.

### 总结

要如何实现合理的 equals 方法, 这里给出一些建议:

- 使用`==`操作来检查传递的对象是否指向同一个对象.如果是, 返回 true.如果比较的代价较高的话, 可以显著提高性能.

- 使用`instanceof`来检查传递的对象是否是同一个实例对象.如果不是, 返回 false.需要注意的是`instanceof`同样支持实现接口.有些类实现同一个接口, 依旧可以的.

- 对传递的对象进行转换成正确的类型.由于上一步的保证, 这一步是肯定可以成功的.

- 对类对象内所有`重要的`的域, 进行比较.如果全部都比较通过了, 返回 true, 否则返回 false.对于这些域, 如果是 int, byte 等原始类型(除了 flaoat 和 double)可以直接使用`==`进行值比较.而对于对象类型, 则递归地调用该对象的 equals 方法.对于 float 和 double, 可以使用 Float.compare(float, float), Double.compare(double, double)进行比较, 主要是处理 flaot 和 double 存在一些特殊值(如 Float.NaN, -0.0f 等).这里不推荐使用 Float.equals 方法和 Double 的 equals 方法, 因为这会导致自动装箱.对于数组类型, 推荐使用 Arrays.equals 方法, 进行验证.如果有些对象类型允许空, 可以使用 Objects.equals(Object, Object)进行比较.对于有些类如果比较的代价非常高, 可以存储一个规范的标准域(`CanonicalForm`)(如, 对对象内所有的域取哈希码), 然后比较这个域是否相同, 而不是进行昂贵的比较.但是这一般用于不变类.

- 当你完成 equals 方法的时候, 问下你自己这个方法是否满足对称性, 是否满足传递性, 是否满足一致性.有时候一个合理的测试将会起到更好的校验效果.

同时这里有一些 equals 方法的注意事项:

- 覆盖 equals 方法的时候, 一定要覆盖 hashCode 方法.

- 不要过分强调 equals, 对于一些域的判断还是非常容易实现的, 不要添加太多标示为来进行比较.

- 不要替代 equals 方法中传递的`Object`参数.有的人会替代成对应的实际类, 但是这是不推荐的.首先这不会覆盖默认的 equals 方法(就算添加了`@Override`, 也会报错), 第二这就强调了比较的对象为具体的实际类, 不推荐这么使用.

有的时候我们可以借助一些工具来帮我们进行覆盖 equals 方法, 如`Google AutoValue`, `Lombok`等, 这些通过简单地添加一个注释就可以完成任务.而注释生成的方法往往就是你想要的.通过 IDEs 自动生成 equals 方法是不推荐的, 可读性不是很好, 并且没有办法动态生成(如果你修改了某个一属性, 你就需要重新生成).

总而言之, 不要覆盖`equals`方法, 一般来说默认的实现就是你要的.如果你要覆盖的话, 一定要比较类中所有的域, 并且比较的时候要满足 5 大要求.

## Item 11: Always override hashCode when you override equals

覆盖了`equals`方法但是却不覆盖`hashCode`可能导致严重的后果.因为这违背了`hashCode`方法的基本准则, 而 hashCode 的准则为:

- 保持一致性, 即多次调用 hashCode 方法, 只要通过 equals 方法保持为 true, 就一定要返回一样的值.

- 如果两个对象调用 equals 为 true, 那么调用 hashCode 就一定要返回一样的值.

- 如果两个对象不相同, 不要求 hashCode 返回的值一定相同, 但是尽量保持.

而我们覆盖`equals`方法但是却不覆盖`hashCode`的做法, 违背了第二条要求.如新建两个对象, 赋予一样的成员变量(满足 equals 方法为 true), 但是调用 hashCode 时是调用 Object 的 hashCode 方法(比较引用地址), 肯定是返回 false 的.这里用一个例子来说明危害:

```java
Map<PhoneNumber, String> datas = new HashMap<PhoneNumber, String>();
datas.put(new PhoneNumber(172, 168, 32), "Jerry");
String name = datas.get(new PhoneNumber(172, 168, 32)); //null, 并不是 Jerry.
```

其中 PhoneNumber 就是覆盖了 equals 方法, 没有覆盖 hashCode 方法.当我们查询的时候, 传递的是一个不同的 hashCode 值, 内部查询时从不同的桶进行查询(有可能一样).

解决方法也很简单, 那就是覆盖一个合适的 HashCode 方法.什么样的 HashCode 方法是合适的呢? 这里有一个简单的 HashCode 方法:

```java
public int hashCode() {
    return 42;
}
```

这是非常野蛮的, 但是可以使用.因为这个方法满足了`hashCode`的所有限制.但是这也可能导致严重的性能后果, 如在 HashMap 中根据对象的 hashCode 进行分桶存储, 但是由于 hashCode 全部相同, 就放到同一个桶里面.导致 HashMap 原先的设计特性: 线性查询时间无法实现.特别是对于包含大量数据的 HashMap 或者 taable 就可能导致无法进行查询的后果.

一个好的 hashCode 函数应该为每一个 unequal 的对象返回同一个 hashCode 值.这是非常完美实现的, 但是可以近乎完美的实现, 这里提供一个实现方法:

- `int result = first.hashCode()`, 初始化一个 int 对象 result, 取第一个对象的 hashCode 值.
- 然后对于剩余的每一个对象(域)取 hashCode(c)进行组合.
- result = 31 \* result + c;
- 其中一个域 f 是原始类型, 通过`Type.hashCode(f)`进行计算, `Type`是对应的封装型.如: `Integer.hashCode(f)`.
- 如果这个域 f 是一个对象引用, 并且在 equals 中进行调用比较了, 调用对象的 hashCode 方法.如果为 null, 就使用 0.
- 如果域 f 是一个数组, 并且数组中每个元素都非常重要, 就分离出来(分别调用对应 hashCode 方法)使用`Arrays.hashCode`进行生成 hashCode 值, 如果数组中没有成员变量, 使用 0 来代替.
- 返回 result.

## Item 12: Always override toString

Object 提供了原始的 toString 方法: 对象的类名 + `@` + 十六进制的哈希码(如:PhoneNumber@163b91).这是非常令人困惑的, 也是很难理解的.`toString`方法本意是让我们返回一个简单而精确的描述字符串, 但这显然没有满足要求.在 Object 的 toString 方法也显示告诉我们去重写这个方法: `It is recommended that all subclasses override this method.`;

虽然不像`equal`和`hashCode`的限制一样, 但也是非常推荐进行重写该方法的.不仅可以让类更加容易使用, 在调试的时候也能起到很好的作用.如`707-867-5309`比`PhoneNumber@163b91`更加直接明了, 让人理解.并且很多时候, 都会不自觉地调用 toString 方法: 如传递对象给`print`,`printf`, string 的级联操作(如`+`), assert 或者 debuger 中输出.这些时候都会输出对应 toString()的结果, 如果我们重写了该方法, 可以带来很好的帮助.

当重写 toString 方法的时候, 推荐包含所有对象内重要的信息.有些特殊情况可以不满足这个: 如果一个对象太大了, 属性过多而无法实现.或者就是一个对象内部的信息不适合用 String 来进行描述.在这些特殊情况下推荐使用一些总结的话语, 如之前的`PhoneNumer`可以返回`Manhattan residential phone directory(1487536 listings)`.一般这个总结应该是容易让人理解的.

同时在重写 toString 方法的时候, 你可以在 toString 的文档中显示表明返回的 String 的样式.一旦你规定返回的样式, 那么返回的结果将会是唯一的, 没有异议的, 容易读取的.同时一旦规定了返回的样式, 那么实现一个静态转换方法或者构造函数来接收这个样式的 String 进行转换也是非常好的.就如同大多数原始类型封装类, `BigInteger`, `BidDecimal`等.

但是规定返回的样式也有一个缺点: 如果规定了样式, 那就就需要永久维护它.如果这个类被广泛的使用, 别的程序员将会频繁使用这个方法, 甚至将 String 对象写入持久化层(如数据库)中去, 一旦你修改了返回的样式或者不兼容之前的格式, 那别人依赖这个方法的代码就会全部崩溃, 后果非常严重.如果不规定返回的样式, 你会保留了修改的灵活性(自由添加修改属性).

无论你是否确定规定返回的格式, 都应该显示的在描述中说明返回的样式.如果你规定了返回的样式, 那么这个需要更加的精确.如 Integer 的`toString`实现方法:

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
 * ({@code '\u005Cu002D'}).If the first argument is not
 * negative, no sign character appears in the result.
 *
 * <p>The remaining characters of the result represent the magnitude
 * of the first argument.If the magnitude is zero, it is
 * represented by a single zero character {@code '0'}
 * ({@code '\u005Cu0030'}); otherwise, the first character of
 * the representation of the magnitude will not be the zero
 * character. The following ASCII characters are used as digits:
 *
 * <blockquote>
 *   {@code 0123456789abcdefghijklmnopqrstuvwxyz}
 * </blockquote>
 *
 * These are {@code '\u005Cu0030'} through
 * {@code '\u005Cu0039'} and {@code '\u005Cu0061'} through
 * {@code '\u005Cu007A'}.If {@code radix} is
 * <var>N</var>, then the first <var>N</var> of these characters
 * are used as radix-<var>N</var> digits in the order shown.Thus,
 * the digits for hexadecimal (radix 16) are
 * {@code 0123456789abcdef}.If uppercase letters are
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
 * Returns a brief description of this potion.the exact details of
 * the representation are unspecified and subject to change,
 * but the following may be regarded as typical:
 *
 * "[Potion #9: type=love, smell=turpentine, look=india ink]"
 **/
```

当使用者看到这些注释的时候, 就知道这个格式可能会改变, 就不会进行格式的持久性的存储或转换.

无论你有没有规定格式, 在 toString 中返回的信息, 都应该提供一个显示的获取途径(`Accessor`).不然的话, 程序员需要使用的时候, 需要自己手动去构造 String 对象, 这是非常容易出错的, 甚至导致系统崩溃.同时不推荐为静态工具类重写 toString 方法, 同样也不推荐为`Enum`枚举类型重写该方法(Java 库中已经为你准备一个很好的实现), 同时你应该在一个抽象类中去实现这个方法, 那么它的子类就可以进行共享.如很多集合类的 toString 实现就是继承自父抽象类的实现.

在 Google 的`AutoValue`, `lombok`等自动化生成的`toString`方法, 一般不是完美的.它们更加趋向于告诉别人内部的值.而不是这个对象代表的含义.推荐自己进行重写该方法.

总而言之, 为你每一个可实例化的对象重写`toString`方法, 除非父类已经重写了合适的格式.这会让类更加容易使用和调试.同时 toString 方法应该返回一个简洁漂亮的 String 格式.

## Item 13: Override clone judiciously

`Cloneable`接口被设计成一个简单的声明式接口, 没有任何方法.只是用来决定 Object 中的`clone()`方法的可用性.如果一个对象实现了`Cloneable`接口, 那就说明这个对象支持克隆功能, Object's 的`clone()`方法应该返回一个域拷贝的新对象, 否则的话调用该方法则会抛出一个`CloneNotSupportedException`异常.

理论上来说实际上只要一个对象实现了`Cloneable`接口, 那么就应该重写`clone()`方法来提供一个合适 public 的`clone()`方法.这个机制是非常脆弱的, 危险的, 超语言的(创建一个对象却不调用对象的构造函数).

`clone()`方法的一般约定为:

```java
x.clone() != x;                            //Must be true, clone object is not the same
x.clone().getClass() == x.getClass();    //Must be true, class is the same
x.clone().equals(x);                    //Not absolute require.
```

一般约定调用 clone()方法时, 首先调用 super.clone()进行 clone(有点类似构造函数).通过这个约定, 可以保证上述的第二个约定一定可以完成.当然你可以直接调用构造函数进行直接创建对象, 这样的话可能存在问题: 如果子类调用`super.clone()`方法, 返回的 class 和当前的克隆对象的 class 就不相同.子类就不能满足第二条约定, 除非将`clone()`方法声明为`final`, 这样就不用担心(但是子类就无法重写了).不推荐使用构造函数, 而是推荐使用`super.clone()`.另外不可变的类不应该重写这个方法(防止无用拷贝).这是一个标准的`clone()`函数.

```java
//Clone method for class with no references to mutable state
@Override
public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();    //Can't happen
    }
}
```

为了让`clone()`函数正确执行, `PhoneNumber`必须实现`Cloneable`接口.来保证方法调用不会抛出异常.这里的返回的是 PhoneNumber 利用了 Java 的`covariant return types`, 返回的是 Object 的子类, 是允许的.并且放在 try 语句中, 保证如果类没有实现接口的话, 报出异常.这适用于类里面所有的变量都是原始数据类型或不变的(即 final 的).

对于那些存在可变变量的对象, 直接调用`super.clone()`将会导致严重的错误:

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null;    //Eliminate obsolete reference
        return result;
    }

    //Ensure space for at least one more element
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

如果在 Stack 类的 clone 方法中直接返回`super.clone()`方法, 那么返回的对象拥有正确的 size 值, 但是在`elements`上却是指向同一个数组, 并没有进行拷贝.修改原数组中对象时, 克隆的数组中也进行了变化.这破坏了克隆的不变性.

最简单解决方法就是在`clone()`实现构造函数的功能, 返回一个全新的对象(为可变对象进行克隆).因为你必须保证克隆出来的对象不会影响原来的对象.

```java
//Clone method for class with references to mutable state
@Override
public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

注意这里的`result.elements = elements.clone();`, 中数组没有进行类型转换, 因为数组的 clone 函数会根据实际情况进行返回, 不需要进行转换(数组的拷贝特别适合克隆).注意这里也可能存在一个问题, 那就是如果将 elements 设置为 final 的, 那这个解决方法就不能生效了(因为你无法重新赋值 elements). 并且这是设计的 Bug(一直存在的): Cloneable 与 final 引用(指向可变对象)不兼容.除非这个可变对象可以安全的分享给所有克隆对象.

并且有时候单纯对可变成员对象进行 clone 也不能很好的解决问题.如当你克隆`HashTable`时, `HashTable`使用`Entry[] buckets`来存储对象, 而`Entry`为一个单链表形式.

```java
public class HashTable  implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
    ...
}
```

当你简单的使用数组的拷贝:

```java
//Broken clone method - result in shared mutable state!
@Override
public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

看起来很完美, buckets 变成一个新的 buckets 存储新的对象列表.但是内部存在一个问题, 虽然 buckets 中对象是新的对象.但是对象内的链表却是指向原来的(即只修改了链表头的对象, 剩余部分并没有修改).这样就破坏了 clone 的不变性, 使用的时候可能造成不确定行为.

为了解决这个问题, 那就必须处理内部的所有的链表对象.其中一个解决方法如下:

```java
public class HashTable  implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }

        Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
        }
    }
    @Override
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = buckets.clone();
            for (Entry entry: result.buckets)
                if (entry != null)
                    entry = entry.deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    ...
}
```

这是一种解决方法, 并且运行时也可以达到我们的要求.但是这里隐藏一个问题, 那就是 deepCopy()方法使用了递归进行完成, 如果链表足够长的话, 就很有可能导致 Stack over flow 的问题.因此改进的方法为修改递归为循环.

```java
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

虽然这解决了我们的问题, 但是这个 clone 方法没有我们预想的跑的那么快, 也破坏了 clone 方法的简单和优雅的特性.

类似构造函数, clone 方法内部不能调用任何可重写的方法.一旦你这么做了, 如果子类重写了这些方法, 会给 clone 函数带来不可预知的风险.另外, 在 Object 的 clone 方法中抛出了 CloneNotSupportedException, 但是如果你实现 Cloneable 接口, 并不需要抛出这个异常(因为并不会出现), 可以进行省略来简化使用.如果要设计一个类用来继承, 那么推荐不实现 Cloneable 接口, 模仿 Object 的方法进行抛出异常.由子类自行决定是否实现 clone 方法.因为一旦父类实现了该接口, 那么子类也必须进行维护, 以保证兼容性.甚至有些限制方法, 禁止子类实现该接口:

```java
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

另外当你写一个线程安全的类时, 记住让 clone 方法进行同步, 就像别的方法一样.

总而言之, 所有实现了 Cloneable 接口的类都应该重写 clone 方法(以 public 的形式), 返回的类型为自己本身.首先调用`super.clone()`, 然后修复需要修复的成员变量(指向可变类型的变量): 对于指向任何可变对象的引用, 对该可变变量进行深拷贝, 然后将引用指向新的拷贝.一般的做法就是对其可变变量进行克隆, 虽然这不是最好的解决方法.对于不可变的变量和原始类型数据, 则不需要进行`修复`, 但是也有一些例外, 如 serial number 或其他 unique id, 这些虽然是不变, 仍然需要进行修复.

换句话说, 付出这么多努力来维护 clone 方法是必须的吗? 答案是否定的, 但是如果父类实现了 Cloneable 接口, 那当然没有别的选择, 自能进行维护.否则的话, 还有一些更好的方法来实现`对象拷贝`.那就是: `copy constructor`和`copy factory`.传递一个对象, 然后拷贝发返回一个新的对象.

```java
//Copy constructor
public Yum(Yum yum) { ...};

//Copy factory
public static Yum newInstance(Yum yum) { ...};
```

相比 clone 方法, 这种方式有很多好处:

- 不依赖特殊的, 充满风险的创建方式(clone 不调用构造函数).
- 不和 final 域使用冲突.
- 不会抛出异常
- 不需要显式转换.
- 可以更加参数进行自定义返回对象.正如`conversion constructor`和`conversion factories`.

使用 Cloneable 接口时, 需要经常想到这个方法的带来的负面影响.用于继承的类不推荐实现该接口, final 类也不推荐实现该接口.并且作为对象的拷贝功能, 构造函数和静态工厂类往往更加合适.最好使用该方法对象, 那一定是数组.

## Item 14: Consider implementing Comparable

不像别的方法都是定义在 Obect 对象内, ‘public int compareTo(T o);‘, compareTo 方法是定义在 Comparable 接口中的一个单独的方法.这个方法有点类似`equals`方法, 但是作用要更大一点, 提供次序的比较.一般来说一个对象实现了`Comparable`接口, 意味着这个对象的实例默认拥有次序.如对于这类对象的数组 a, 如果需要进行排序:

```java
Arrays.sort(a);
```

如果需要对一个 String 数组进行去重和排序:

```java
Set<String> set = new TreeSet<>();
Collections.addAll(s, args);
System.out.println(s);
```

上面这些数组排序和集合排序等操作都是依赖于对象中`compareTO`方法.而`compareTo`方法的定义如下:

Compares this object with the specified object for order. Returns a negative integer, zero, or a positive integer as this object is less than, equal to, or greater than the specified object.

在`compareTo`方法中, 使用 sgn 来处理返回值, 负数返回-1, 正数 1, 相等 0.方法的约定为:

- 对于所有的 x 和 y, sign(x.compareTo(y)) == -sign(y.compareTo(x)).
- 比较需要有传递性: if (x.compareTo(y) > 0 && y.compareTo(z) > 0) then x.compareTo(z) must >0.
- if (x.compareTo(y) == 0), then ( sign(x.compareTo(z)) == sign(y.compareTo(z)));
- 强烈要求 if (x.compareTo(y) == 0) then x.equals(y) == true.如果不满足这个条件, 应该在注释中显示说明这一点.如`Note: This class has a natural ordering that is inconsistent with equals.`;

`compareTo`方法的限制没有`equals`方法那么复杂, 因为`equals`方法面对的是所有的对象, 而`compareTo`一般用于相同对象之间的比较(即类相同), 一般用于内部比较.当出现不同的类型进行比较的时候, 往往会抛出异常.

有点类似`hashCode`函数, 如果不遵守`hashCode`的约定, 就会让很多依赖`hashCode`的方法或者对象就会出错。 如果不遵守`compareTo`方法的约定, 那么很多依赖`compareTo`的方法和对象就会出错.如排序的集合: TreeMap, TreeSet, 集合工具类 Clollections 数组工具类 Arrays 中的排序和搜索功能.

前三个规定有点类似`equals`中的限制: 对称性, 传递性, 自反性.因此这里也存在同样的限制: 如果想通过继承一个对象来添加新的属性, 而这个对象实现了 Comparable 接口, 那也会破坏这三条特性(详细查看 Item10), 推荐使用组合的形式完成.

最后一条限制, 强烈推荐兼容 equals 方法, 如果不兼容 equals 方法, 在一些集合类中容易出现问题.因为默认的等价判断应该是使用 equals 方法, 但是有些集合类中使用的是 compareTo 进行替换, 如果 compareTo 不兼容 equals 方法的话, 会导致严重的后果.如 BigDecimal 类中 compareTo 方法就不兼容 equals 方法, 如果往一个 HashSet 中添加 new BigDecimal("1.0")和 new BigDecimal("1.00"), 那么可以成功添加两个不同的对象.但是如果你使用 TreeSet 的话, 就会只添加一个对象.因为二者等价关系的判断是不一样的.并且这种问题是很难发现的.

在 compareTo 方法中, 按照顺序比较对象内所有的成员(即递归地调用 compareTo 方法), 如果有一个成员对象没有实现 Comparable 接口或者你需要自定义排序的规则, 可以使用 Comparator, 自己进行构造一个特殊的比较器进行比较.

```java
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis);
    }
    ...//Remainder omitted
}
```

在 compareTo 方法中比较原始类型数据时, 推荐使用对应装箱类中的工具方法, 如 Integer.compare, Float.compare 等.而不是显式的使用`<>`, 可以很好的提高代码的阅读性, 减少犯错机会.

如果一个对象有多个成员变量, 那么比较时候的排序就非常重要了.一般推荐先从最重要的成员进行比较, 轮流进行比较.如:

```java
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0)
            result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}
```

在 Java8 中, Comparator 接口被广泛使用.通过 Comparator 接口可以很快的构建一个良好的比较器, 虽然会带来一些性能上的损失.在使用比较器的时候, 推荐预先构建好静态的对象(static), 并让命名简单明了.

```java
private static final Comparator<PhoneNumber> COMPARATOR = comparingInt((PhoneNumber pn) -> pn.areaCode)
    .thenComparingInt(pn -> pn.prefix)
    .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

注意这里使用了 Lambda 表达式, 并且后续的传递对象并没有进行类型转换(PhoneNumber), 因为 JVM 足够聪明可以识别.需要注意的是, 有些 Comparator 使用 hashCode 进行比较:

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() -  o2.hashCode();
    }
}
```

这是非常危险的, 因为可能存在 Integer 的溢出或者浮点数(浮点数存储方式的不同).正确的方法应该使用 Integer.compare 方法.

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
}
//Simply
static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
```

总而言之, 当你实现一个值类型, 并且有敏感的次序的时候.推荐实现 Compreable 接口.这样在数组或者集合中时可以很容易被排序或者查找.另外不要显式使用`<>`, 而是使用原始类型封装类的 compare 方法进行比较或者使用 Comparator 进行比较.
