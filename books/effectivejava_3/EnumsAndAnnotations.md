# Enums And Annotations
Java中支持两种特殊的引用类型: 一种特殊的类, 枚举; 一种特殊的接口, 注释. 本章主要是讲如何高效地使用这两种类型.

## Item 34: Use enums instead of int constants.
枚举类型是一种值, 由许多固定的常量组成. 如一年的四个季节, 太阳系的八大行星等等. 在枚举类型添加到Java之前, Java中有很多方式来代表泛型, 如:

```java
//The int enum pattern - severely deficient!
public static final int APPLE_FUJI          = 0;
public static final int APPLE_PIPPIN        = 1;
public static final int APPLE_GRANNY_SMITH  = 2;

public static final int ORANGE_NAVEL        = 0;
public static final int ORANGE_TEMPLE       = 1;
public static final int ORANGE_BLOOD        = 2;
```

这种技术就是常称的`整形枚举模式(int enum pattern)`, 这种模式是非常简陋的, 拥有非常多的缺点. 如不能保证类型安全, 无法进行文字输出(在编译调试的时候只能显示数字), 无法知道枚举中元素数量, 无法对枚举中的元素进行迭代, 并且只要修改对应的值, 就必须重新编译客户端代码, 否则就会导致不正确的结果. 有时候, 我们甚至还会遇到`字符串枚举模式(String enum pattern)`这种模式用固定字符串来表示, 虽然这个解决了显示的问题, 但是还是存在硬编码的问题, 并且有一定的效率损耗(比较依赖字符串的比较).

在枚举出来之后, 这上面的缺点都被解决了, 还提供了很多别的优点.

```java
public enum Apple {FUJI, PIPPIN, GRANNY_SMITH}
public enum Orange {NAVEL, TEMPLE, BLOOD} 
```

这就是使用枚举的表示方式, 简单明了. 枚举和很多别的编程语言中(C,C++,C#等)的枚举不一样. Java中的枚举是一个完备的类, 而别的语言大多数本质就是int常量. 因此更加成熟, 可用性也大大增强.

Java中的枚举本质非常简单, 就是一类不变类, 然后导出内部的常量, 枚举中的每一个元素就对应着一个静态不变的实例, 并且只有这些实例. 换句话说, 就是枚举类型是`实例可控(instance-controlled)`的类型.

枚举保证了编译时的类型安全, 如声明了方法中的参数为某个泛型, 那么调用该方法时就只能传递该类型的泛型, 任何别的类型(包括别的泛型)都会编译失败. 另外枚举还允许你使用`==`进行枚举的判断, 因为枚举是`实例可控`的.

枚举的元素和名称都是一一对应的, 并且不同的枚举之间的名称即使相同也不会冲突, 因为彼此是分开的(不同的类). 在输出的时候, 简单的调用`toString`方法就可以.

另外枚举作为一个类, 内部封装重写了`Object`所有的方法, 并且实现了`Comparable`和`Serializable`接口, 提供的高质量的内部实现. 另外, 如果你想拓展一个枚举类型, 如向内部添加属性, 方法等, 都是允许的, 并且是非常方便的. 如设计一个`Planet`包含太阳系的八大行星, 每个行星有自己的质量, 半径, 重力等等.

```java 
public enum Planet {
    MERCURY(),
    VENUS(),
    EARTH(),
    MARS(),
    JUPITER(),
    SATURN(),
    URANUS(),
    NEPTUNE();

    private final double mass;  //In kilograms
    private final double radius;    //In meter
    private final double surfaceGravity;    //In m /s^2
    //Universal gravitational constant in m^3 / kg s^2
    private static final double G = 6.67300E-11;

    //Constructor
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }

    public double mass()            {return mass;}
    public double radius()          {return radius;}
    public double surfaceGravity()  {return surfaceGravity;}
    public double surfaceWeight(double mass) {
        return mass * surfaceGravity;   //F = ma
    }
}
```

从这个简单实例, 就可以看出向一个枚举类型添加属性和方法也是非常简单的. 因为枚举本身就是一个类. 这里需要注意的是, 枚举设计的就是不变类, 这里的属性都是默认为final的, 虽然可以声明为public, 但是推荐声明为private, 然后提供public的访问函数.枚举本身实现了静态的`values`方法, 默认返回元素数组, 便于使用和遍历. 另外枚举本身默认实现了`toString`函数, 返回基本的名称, 如果想要自定义, 可以进行重写.

如果我们删除了枚举中的某个元素, 会有什么影响呢? 如太阳系中的九大行星中的冥王星被取消资格了.那么客户端没有指明使用到特定的元素就不会受到影响. 如果使用到了移除对象的引用, 那么会在编译时报错, 这时候编译器会弹出一个详细的错误信息, 告诉客户端该元素已经被移除了. 这比`整形枚举类型`效果好了很多. 另外如果一个枚举类型不是绑定在某个类的, 可以被广泛使用的, 可以声明该枚举类型为`顶级(top-level)类`, 即声明为`public static`, 这样便于代码重用.

一般来说上面的那个`Planet`适用于大部分的枚举使用情景. 但是, 偶尔你想要设计一个方法, 根据枚举元素的区别进行不一样的行为. 比如四则运算:

```java
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
            case PLUS: return x + y;
            case MINUS: return x - y;
            case TIMES: return x * y;
            case DIVIDE: return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```

这里的`apply`函数就是基本的运行函数, 需要根据枚举类型进行区别操作. 这里的实现, 是非常简陋的, 容易出错的. 首先如果不抛出`AssertionError`这里就无法进行编译, 因为编译器默认可能会到达后面这个语句, 但是实际上并不会. 并且如果我们添加了一个新的运算符号, 但是忘记更新了`apply`方法, 那么调用时就会直接出错. 这时候怎么办呢? 这时候可以使用`实例特定方法(Constant-specific)`.

```java
public enum Operation {
    PLUS {public double apply(double x, double y) {return x + y}}, 
    MINUS {public double apply(double x, double y) {return x - y}}, 
    TIMES {public double apply(double x, double y) {return x * y}}, 
    DIVIDE {public double apply(double x, double y) {return x / y}};

    public abstract double apply(double x, double y);
}
```
这种就减少了冗余代码, 并且规定了每个元素分别实现自己的方法. 这样即使我们添加新的代码, 也不太可能忘记(一般都是仿照现有元素添加). 就算忘记了, 编译器也会报错, 因为这是抽象的, 实例必须实现. 完备的`Operation`枚举:

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

//Unit test
public static void main(String[] args) {
    double x = 2.12L;
    double y = 3.23L;
    for (Operation op : Operation.values())
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
    
    // 2.12 + 3.23 = 5.35
    //... 
}
```

注意枚举中内部实现了`valueOf(String)`函数, 来转换String为枚举实例. 如果你重写了`toString`方法, 推荐重写`valueOf(String)`方法, 来保证完整性. 这里提供一个通用的实现方法:

```java 
private static final Map<String, Operation> = stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));

public static Option<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
}
```

注意这里使用了一些`Stream`和`Lambda`表达式, 最后的`fromString`返回的是`Option`类型, 就是存在null值的可能, 但是将这个问题交给客户端进行解决. 注意这里的`Map`对象不允许放在构造函数中进行初始化, 因为在初始化的时候, 枚举中的实例对象还没有进行构造, 会报`NullPointerException`. 也就是说在枚举的构造函数中是不允许修改或者接触枚举中的元素的.

`实例特定方法(Constant-specific)`也有一个明显的缺点, 那就是重用代码的话就会变得非常困难. 但是对于大部分的枚举, 内部的元素并不需要全部重写指定的方法, 很多都可以公用一份代码. 比如一个计算工资的枚举`PayrollDay`, 根据每天的类型来计算工资.

```java 
public enum PayrollDay {
    MONDAY, TUESDAY, WEDENSDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY;

    private static final int MINS_SHIFT = 8 * 60;

    int pay(int minutesWorked, int payRate) {
        int basePay = minutesWorked * payRate;

        int overtimePay;
        switch(this) {
            case SATURDAY:  //Weekend 1.5 
            case SUNDAY:
                overtimePay = basePay / 2;
                break;
            default:    //Weekday
                overtimePay = minutesWorked <= MINS_SHIFT ? 0 : (minutesWorked - MINUS_SHIFT) * payRate / 2;
        }
        return basePay + overtimePay;
    }
}
```

为了重用代码, 这里就回到了`switch`的情况, 这样是非常危险的. 如果添加了一个新的类型`Vocation`, 并且忘了在`switch`中添加方法, 这就会导致系统按照`weekday`的计算公式进行付费. 但是如果改成`实例特定`的模式, 又会产生大量冗余代码(周一到周五都是相同的函数). 这时候可以将`pay`抽象出来, 声明为一种特殊的枚举类型.

```java 
public enum PayrollDay {
    MONDAY, TUESDAY, WEDENSDAY, THURSDAY, FRIDAY, SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);

    private final PayType payType;

    PayrollDay(PayType payType) {this.payType = payType;}
    PayrollDay() {this.payType = PayType.WEEKDAY;}

   int pay(int minutesWorked, int payRate) {
       return payType.pay(minutesWorked, payRate);
   }

    

    private enum PayType {
        WEEKDAY {int overtimePay(int mins, int payRate){
            return mins <= MINS_SHIFT ? 0 : (mins - MINS_SHIFT) * payRate / 2;
        }},
        WEEKEND {int overtimePay(int mins, int payRate){
            return mins * payRate / 2;
        }};

        abstract int overtimePay(int mins, int payRate);
        private static final int MINS_SHIFT = 8 * 60;

        int pay(int minsWorked, int payRate) {
            int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }
}
```

什么时候使用枚举是合适的呢? 当你需要一系列的常量, 并且常量在编译期就可以确定下来, 如一年四个季节, 太阳系的行星等. 这就非常适合使用枚举. 如果常量实例会在运行期保持变化的话, 那还是不要使用枚举.

总而言之, 相比`整形枚举模式`, 枚举根据容易阅读, 安全, 也更加实用. 大部分的情况下, 枚举并不需要自定义属性和方法. 但是枚举是允许的, 你可以只有添加各类属性和方法. 在定义一些实例区分的方法时, 可以考虑使用`实例特定`方法, 而不是`switch`语句. 并且在一些实例共享方法体时, 可以考虑抽离出方法的策略.

## Item 35: Use instance fields instead of ordinals.
每个枚举内部都默认实现了一个实例方法`ordinal`方法, 这个方法返回枚举元素在枚举集合中的次序. 但是依赖这个属性来绑定特定的属性是非常不理智的:

```java
//Abuse of ordinal to derive an associated value
public enum Ensemble {
    SOLO, DUET, TRIO, QUATET, QUINTET, SEXTET, SEPTET, OCTET, NONET, DECTET;

    public int numberofMusicians() {return ordinal() + 1;}
}
```

上面的代码看起来可以很好的工作, 但为什么说这是不理智的呢. 首先, 返回的值是依赖次序的, 而这个次序是随机的(可变的), 只要枚举中的类型变量更换了一个位置, 那么这个方法不会提示任何错误, 只是单纯让程序运行错误. 第二, 这个值是固定的, 缺乏灵活性. 如你想要添加一个新的元素, 元素的值却已经被使用了, 那这个元素就无法添加进去. 如上面这里添加一个`DOUBLE_QUATET`是显然不可以的, 因为`8`已经被`OCTET`使用了. 并且对于一些特殊值, 如`TRIPE_QUATET`那这个值`12`是没有办法实现的.因为没有这么多元素,`ordinal()`没办法返回`11`.

其实解决方法非常简单, 单纯地添加一个属性即可.

```java 
//Abuse of ordinal to derive an associated value
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUATET(4), QUINTET(5), SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUATET(8);

    private final int numberOfMusicians;

    Ensemble(int numberOfMusicians) {
        this.numberOfMusicians = numberOfMusicians;
    }

    public int numberofMusicians() {return numberOfMusicians;}
}
```

这样既保证了灵活性, 也保证了健壮性. 所以尽可能少的依赖`ordianl`函数, 正如该方法所注释的: `Most programmers will have no use for this methods. It is designed for use by general-purpose enum-based data structures such as EnumSet and EnumMap.`;

## Item 36: Use EnumSet instead of bit fields.
如果一个枚举类型中的元素只是单纯的用于`Set`中, 以前一般的使用方法是使用`bit fields`.

```java 
public class Text {
    public static final int STYLE_BOLD	        = 1 << 0;   //1
    public static final int STYLE_ITALIC        = 1 << 1;   //2
    public static final int STYLE_UNDERLINE     = 1 << 2;   //4
    public static final int STYLE_STRIKETHROUGH = 1 << 3;   //8
}

//Use case
text.applyStyles(STYLE_BOLD | STYLE_UNDERLINE);
```

但是这是非常不好的. 这个存在所有`整形枚举模式`的问题. 另外, 这是很难翻译对应的值, 你获得的只是`int`值. 并且很难迭代所有的内部元素. 并且一旦你确定了类型(int, long), 那么枚举内的元素的极限也就确定了: 32和64. 你没有办法进行拓展.

另外这里可以很好的使用`EnumSet`进行替代, 这里提供了所有的枚举的优点, 并且效率来说并不会比`bit field`慢多少, `EnumSet`本质也是通过`bit vector`进行实现的.

```java
public class Text {
    public enum Style {BOLD, ITALIC, UNDERLINE, STRIKETHROUGH};

    //any set could be passed in
    public void applyStyles(Set<Style> styles) {...}
}

//Use case
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

总而言之, 仅仅因为使用元素在`Set`中, 就使用`bit field`是非常不理智的. 可以使用`Enum, EnumSet`进行很好的替换. 这个消除了大部分的问题, 唯一的缺点可能就是`java9`中可能会设置`EnumSet`为可变类, 这时可以使用`Collections.unmodifiableSet`进行替换, 虽然性能可能会受到一定的影响.

## Item 37: Use EnumMap instead of ordinal indexing.
有时候还是可以看到我们使用`ordinal`函数来定位一个元素在数组或者list中的位置. 如

```java 
class Plant {
    enum LifeCycle { ANNUAL, PERENIAL, BIENNIAL }
    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }

    @Override
    public String toString() {
        return name;
    }
}

//Use ordinal() to index into an array
Set<Plant>[] plantsByLifeCycle = (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
for (int i = 0; i < plantsByLifeCycle.length; i++) 
    plantsByLifeCycle[i] = new HashSet<>();

for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);

//Print
for (int i = 0; i < plantsByLifeCycle.length; i++) 
    System.out.printf("%s: %s%n", Plant.LifeCycle.values()[i], plantByLifeCycle[i]);
```

上面这可能就是一个常见的使用`ordinal`来定位数组中的元素. 但是这是非常脆弱的. 首先使用了泛型数组, 编译肯定会抛出警告. 然后数组并不会知道泛型的详细类型, 不能提供类型校验. 如果传递一个错误的对象进去, 很有可能就出错了. 现在存在了更好的解决方法那就是使用`EnumMap`.

```java 
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle = new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values()) 
    plantsByLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
System.out.println(plantsByLifeCycle);
```

看起来简单明了很多, 提供了对泛型的兼容性, 并且性能不一定会比数组的差. 因为`EnumMap`底层也是使用数组实现的. 这里的`EnumMap`需要传递一个`key`的`class`信息来保证运行时的泛型信息.`EnumMap`即结合了数组的速度, 又结合了`Map`的类型安全.这里还有使用`Lambda`表达式的简化版本:

```java 
//Case 1
System.out.println(Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle)));

//Case 2
System.out.println(Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle), () -> new EnumMap<>(LifeCycle.class), toSet()));
```

上面有两个版本, 第一个版本虽然可以使用, 但是使用的是默认的容器(`HashMap`), 并不是`EnumMap`. 第二个版本传递了`EnumMap`参数, 使用的该类型. 但是注意这里有一点区别, 之前的版本是不管对应的`LifeType`有没有元素都会创建一个`HashSet`, 第二版本这并不会, 只有存在对应类型的`Plant`才会进行创建, 也就是说不一定会创建对应的`HashSet`(如果没有对对应对象的话).

另外你可能会简单使用`ordinal`函数两次, 访问内部元素.

```java
public enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;

        //Rows indexed by from-ordinal, cols by to-ordinal
        private static final Transition[][] TRANSITIONS = {
            {null,      MELT,       SUBLIME},
            {FREEZE,    null,       BOIL},
            {DEPOSIT,   CONDENSE,   null}
        };

        //Return the phase transition from one phase to another
        public static Transition from(Phase from, Phase to) {
            return TRANSITIONS[from.ordinal()][to.ordinal()];
        }
    }
}
```

这段代码看起来非常简短, 甚至有些优雅. 但是表象总是骗人的, 编译器是不保证数组的引用和index值之间的关联的. 即, 如果你简单移动了次序, 或者增加修改了元素都会变得特别容易出错. 同样你可以使用`EnumMap`进行修复:

```java 
public enum Phase {
    SOLID, LIQUID, GAS;

    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID), BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID), SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
        
        private final Phase from;
        private final Phase to;

        private static final Map<Phase, Map<Phase, Transition>> m = Stream.of(values())
        .collect(gourpingBy(t -> t.from, () -> new EnumMap<>(Phase.class), 
        toMap(t -> t.to, t -> t, (x, y) -> y, () -> new EnumMap<>(Phase.class))));
    }

    public static Transition from(Phase from, Phase to) {
        return m.get(from).get(to);
    }
}
```

虽然有些复杂, 但是极大的增加了灵活性. 如, 我们这里增加一个新的状态`PLASMA`, 这时候`Transition`也添加对应的状态变化`IONIZE`,`DEIONIZE`. 这时候如果是数组类型的话, 就需要从3 x 3的数组, 变成4 x 4的数组. 但是如果使用第二个版本, 就需要简单添加两行即可:

```java
public enum Phase {
    SOLID, LIQUID, GAS, PLASMA;

        public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID), BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID), SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
        IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
        ...//Remainder unchanged
        }
}
```

总而言之, 尽量少的使用`ordinal`来标记在数组中的顺序, 使用`EnumMap`来替代, 提高了代码的健壮性和稳定性.

## item 38: Emulate extensible enums with interfaces.
枚举类型存在一个缺点就是不支持继承, 但是对于枚举类型的特质来说, 不支持往往是好事. 因为枚举就属于一种类别性质, 如果可以实现继承的话, 内部的元素是属于那种类型呢? 这就会非常困惑. 这会破坏枚举设计本身的抽象和实现.但是对于一类枚举类型(`操作码`), 继承却有着不小的吸引力. 但是本身枚举类型是不支持的, 这时候可以使用实现接口来模拟继承.

```java 
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;

    public double apply(double x, double y) {
        switch (this) {
            case PLUS: return x + y;
            case MINUS: return x - y;
            case TIMES: return x * y;
            case DIVIDE: return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```

这里的`apply`函数就是基本的运行函数, 需要根据枚举类型进行区别操作. 这里的实现, 是非常简陋的, 容易出错的. 首先如果不抛出`AssertionError`这里就无法进行编译, 因为编译器默认可能会到达后面这个语句, 但是实际上并不会. 并且如果我们添加了一个新的运算符号, 但是忘记更新了`apply`方法, 那么调用时就会直接出错. 这时候怎么办呢? 这时候可以使用`实例特定方法(Constant-specific)`.

```java
public interface Operation {
    double apply(double x, double y);
}

public enum BasicOperation implements Operation {
    PLUS("+") {public double apply(double x, double y) {return x + y;}}, 
    MINUS("-") {public double apply(double x, double y) {return x - y;}}, 
    TIMES("*") {public double apply(double x, double y) {return x * y;}}, 
    DIVIDE("/") {public double apply(double x, double y) {return x / y;}};

    private final String symbol;

    BasicOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override
    public String toString() {
        return symbol;
    }
}
```

在这个例子中, 虽然`BasicOperation`不可以继承, 但是`Operation`是可以被实现的. 你可以通过实现该接口, 来拓展新的对象. 如:

```java 
public enum ExtendedOperation implements Operation {
    PLUS("^") {public double apply(double x, double y) {return Math.pow(x,y);}}, 
    MINUS("%") {public double apply(double x, double y) {return x % y;}}, 

    private final String symbol;

    BasicOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override
    public String toString() {
        return symbol;
    }
}
```

注意采用实现接口的方法就不用声明抽象接口了. 这里有两种使用方式:

```java
//Use case 1
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(ExtendedOperation.class, x, y);
}

private static <T extends Enum<T> & Operation> void test(Class<T> opEnumType, double x, double y) {
    for (Operation op : opEnumType.getEnumConstants()) 
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}

//Use case 2
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
}

private static void test(Collection<? extends Operation> opSet, double x, double y) {
    for (Operation op : opSet) 
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}
```

其中第一种使用方式, 通过传递`ExtendedOperation.class`作为有界通配符的密钥, 通过该`Class<T>`来传递泛型信息, 保证运行时的类型安全. 然后通过限制泛型的参数: `T extends Enum<T> & Operation`必须同时为`Enum`和`Operation`, 保证类型类型安全. 第二种则是通过集合来完成.

但是通过实现接口来模拟继承, 有一个小小的缺点, 那就是代码有些冗余. 比如前面的`ExtendedOperation`和`BasicOperation`都含有`Symble`的读取和`toString`, 这就造成了冗余. 如果这部分很少的话, 那没问题, 如果这部分非常多的话. 就需要构建静态帮助类来解决问题了.

总而言之, 当你需要实现枚举继承时, 可以通过实现接口的方式模拟继承.
