# Serialization

本章主要关注对象序列化相关的知识. 序列化(Serializing): 将对象序列化成字节流(byte streams). 反序列化(Deserializing): 从字节流构造对象. 一旦对象被序列化, 可以从一个 VM 传递给另一个 VM, 存储在本地磁盘, 发送到远程网络等. 本章主要关注序列化的危险性和最小化使用他们.

## Item 85: Prefer alternatives to Java serialization

序列化是在 1997 年加入 Java, 被公认的非常危险的, 其中包含非常多的安全漏洞. 最近一起导致的重大事件的是 2016 年 11 月的旧金山的铁路系统(SFMTA Muni)停工两个小时. 其中最主要的问题是: 反序列化的攻击面太广了, 而难以防备. 反序列化调用`readObject`方法, 而该方法的实现是由任何实现了`Serializable`类自定义实现的, 这是类自定义实现的, 无法保证执行的代码一定是正确且安全的.

引用`Robert Seacord`的话语:

**Java deserialization is clear and present danger as it is widely used both directly by applications and indirectly by Java subsystems such as RMI, JMX and JMS. Desertialization of untrusted streams can result in remote code execution(RCE), denial-of-service(DoS), and a range of other exploits. Applications can be vulnerable to these attacks even if they did nothing wrong.**

很多攻击者和安全专家研究 Java 标准库和常用的第三方库中 Java 序列化方法, 寻找潜在调用时可以执行地一些危险操作, 这些方法就称为`gadgets`. 多个序列化方法会组成`gadget-chain`. 通过`gadget-chain`的组合足够执行致命的操作. 旧金山铁路系统的崩溃就是使用这种方法. 另外还有一个常见的问题就是`deserialization bombs`: 一个对象在序列化的时候需要花费非常长的时间, 可以导致 DoS.

```java
//Deserialization bomb - deserializing this stream takes forever
static byte[] bomb() {
    Set<Object> root = new HashSet<>();
    Set<Object> s1 = root;
    Set<Object> s2 = new HashSet<>();
    for (int i = 0; i < 100; i++) {
        Set<Object> t1 = new HashSet<>();
        Set<Object> t2 = new HashSet<>();
        t1.add("foo");  //Make t1 unequal to t2
        s1.add(t1); s1.add(t2);
        s2.add(t1); s2.add(t2);
        s1 = t1;
        s2 = t2;
    }
    return serialize(root); //Method omitted for brevity
}
```

`root`对象由 201 个`HashSet`组成, 每个实例包含 3 个(或者更少)对象实例引用, 序列化之后为 5744 字节. 但是如果使用序列化的话, 就需要花费非常非常长的时间进行处理. 这个问题是反序列化`HashSet`时需要计算哈希值, 但是 root 内部高度达到 100 层, 递归计算哈希值时需要调用`2^100`.

如何抵抗这些问题呢? 不要反序列任何你不相信来源的`stream`. 最好的方法就是不要反序列化任何对象. **The only winning move is not to play.** 如果是构建新系统的话, 没有任何理由使用序列化相关操作. 如果有将对象转换为字节流的需求的话, 可以使用一些跨平台的通用语言格式, 如`JSON,Protobuf`. 两者的差别是`JSON`是可读的, 基于文本的. 而`Protobuf`是基于二进制流的, 高效率的.

如果不能避免使用序列化, 那就使用`java.io.ObjectInputFilter`, 对序列化的对象进行过滤(Java9 添加). 这里提供了一个类粒度的卡关模式, 在序列化之前进行过滤. 一般提供一个拒绝列表`Black list`和接收列表`White list`. **优先使用白名单而不是黑名单.** 当然还有一个可选项就是: 重构整个系统, 放弃使用 Java 序列化.

总而言之, 序列化操作是非常危险的. 如果设计系统时有这方面的需求, 推荐使用跨平台的数据结构(JSON,Protobuf).不要反序列化任何不安全的数据来源. 如果必须使用序列化, 使用序列化过滤, 但是需要明白这不能完全保证安全性. 避免书写任何序列化的类, 如果一定需要, 一定要小心谨慎.

## Item 86: Implement Serializable

如果想要一个类支持序列化, 简单的声明`implements Serializable`即可. 这看起来是非常简单, 但是实际上的影响确实非常大的, 付出的代价也是很大的.

首先如果声明实现了`序列化`, 会降低类的灵活性. 一旦这个类发布出去, 并被广泛使用的话, 那你就必须一直维护序列化操作. 如果使用默认的序列化格式的话, 那你需要小心, 后续版本迭代中就需要注意属性, 因为一旦新版本中新增和删除某些属性, 会导致序列化失败的问题. 最好的方式就是自定义序列化格式. 也可以使用`serialVersionUID`来保证对象版本的更新.

第二, 序列化增加了潜在的风险和安全问题. 序列化可以看做另类的构造函数, 使用默认的序列化构造机制的话非常容易导致内部信息和变量的泄露.

第三, 实现序列化会增加测试难度. 每次新版本的对象的发布, 你都必须测试以保证新版本对象可以在生产和测试等各种环境汇总可以被正确序列化. 如果这个对象被广泛使用的话, 那么测试需要花费的时间也是非常多的.

综上所诉, 实现序列化并不是一个轻率的决定, 需要你仔细考量其需要付出的代价和收获. 在我们设计类和接口的时候, 尽量不要实现或者继承`Serializable`. 对于内部类的话, 更是如此. g

总而言之, 实现序列化往往是华而不实的, 尽量少地去实现它. 除非这个类只在安全环境中使用, 不会被外部调用, 那么实现的时候也需要仔细和认真.

## Item 87: Consider using a custom serialized form

当你要完成一项任务, 而该任务又非常紧急的时候. 你往往会省略很多内容, 后续的更新中再进行更正. 这些省略的内容往往就有序列化的样式, 一般就直接声明`implements Serializable`. 但是结果往往相反, 在后续的版本更正中往往会变得很难. 因为这时使用的是默认的序列化样式, 而默认的序列化样式往往存在很多问题并且后续更新中不会兼容前面的使用.

因此不要直接使用默认序列化样式, 特别是没有经过考虑的情况. 默认的序列化样式只适合用于对象的物理属性等价于逻辑内容. 一般的使用模型:

```java
//Good candidate for default serialized form
public class Name implements Serializable {
    /**
     * Last name. Must be non-null.
     * @serial
     */
    private final String lastName;
    /**
     * First name. Must be non-null.
     * @serial
     */
    private final String firstName;
    /**
     * Middle name, or null if there is none.
     * @serial
     */
    private final String middleName;

    ...//Remainder omitted
}
```

在这种简单的情况下使用默认的序列化样式即可. 但是即使使用默认的序列化样式, 也推荐重写`writeObject`和`readObject`进行 null 值校验. 对于某些特殊的类, 默认的序列化格式往往不能满足要求:

```java
//Awful candidate for default serialized form
public final class StringList implements Serializable {
    private int size = 0;
    private Entry head = null;

    private static class Entry implements Serializable {
        String data;
        Entry next;
        Entry previous;
    }

    ...//Remainder omitted
}
```

这里使用的默认的序列化样式, 就会完成一致的拷贝所有的`Entry`, 每个`Entry`都包含所有的数据, 和前后的指针. 这就是非常冗余了. 一般来说, 使用默认的序列化样式有如下缺点:

- 限制住了 API 的特性, 后续的维护和开发必须维护当前对象的可实现性.

- 花费更多的空间: 默认的序列化样式不知道类的具体含义, 而是使用递归遍历所有的属性进行存储.

- 可能消耗大量的时间.

- 可能导致堆栈异常(overflow). 如上面的 StringList, 长度在 1000-1800 之间时, 就有可能导致堆栈异常.

合理的`StringList`实现:

```java
public final class StringList implements Serializable {
    private transient int size = 0;
    private transient Entry head = null;

    private static class Entry {
        String data;
        Entry next;
        Entry previous;
    }

    public final void add(String s){ ... }

    /**
     * Serialize this {@code StringList} instance
     *
     * @serialData .....
     */
    private void writeObject(ObjectOutputStream s) throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);
        for (Enetry e = head; e != null; e = e.next) {
            s.writeObject(e.data);
        }
    }

    private void readObject(ObjectInputStream s) throws IOException {
        s.defaultReadObject();
        int numElements = s.readInt();
        for (i = 0; i < numElements; i++) {
            add((String) s.readObject());
        }
    }

    ... //Remainder omitted
}
```

注意这里虽然变量都是`transient`, 但是还是推荐调用`s.defaultWriteObject()`和`s.defaultReadObject()`, 并进行合理的注释. 同时使用默认序列化样式时, 需要注意的是不适用于那些随着运行的时间而变化的对象, 如`HashTable`. 另外为了维护序列化对象的版本迭代, 推荐声明`serialVersionUID`, 这样可以避免在运行时动态产生. 并且保持一致的话, 可以兼容相同的版本(即使修改了内容). 不然地话, 在运行时动态产生`serialVersionUID`, 如果你修改了某一项属性, 就会导致`InvalidClassException`. **尽量不要轻易修改 version UID, 除非你想破坏之前的兼容性**.

总而言之, 不要轻易使用默认的序列化样式, 尽量进行自定义序列化样式和完善`version UID`.

## Item 88: Write readObject methods defensively

当我们需要为不变类进行序列化操作时, 需要非常小心. 因为一不小心就会破坏不变类的不变性. 这里以`Period`类为例:

```java
public final class Period {
    private final Date start;
    private final Date end;

    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (start.compareTo(end) > 0) {
            throw new IllegalArgumentException(start + " after " + end);
        }
    }

    public Date start() {
        return new Date(start.getTime());
    }

    public Date end() {
        return new Date(end.getTime());
    }

    ..//Remainder omitted
}
```

如果需要对`Period`进行序列化, 第一想法肯定是直接添加`implements Serializable`, 毕竟这样最简单. 但是结果却是恰恰相反, 这样会严重破坏`Period`的不变性. 为什么? 因为反序列化相当于一个特殊的构造函数, 通过修改反序列化的内容可以修改内部的访问权限. 这里简单演示一下, 如何攻击`Period`:

```java
public class MutablePeriod {
    //A period instance
    public final Period period;

    //Period's start field, to which we shouldn't have aceess
    public final Date start;

    //Period's end field, to which we shouldn't have access
    public final Date end;

    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream out = new ObjectOutputStream(bos);

            //Serialize a valid Period instance
            out.writeObject(new Period(new Date(), new Date()));

            /**
             * Append rogue "previous object refs" for internal
             * Date fields in Period. For details, see "Java
             * Object Serialization Specification." Section 6.4
             *
             */
            byte[] ref = {0x71, 0, 0x7e, 0, 5};     //Ref #5
            bos.write(ref); // The start field
            ref[4] = 4;     // Ref #4
            bos.write(ref); // The end field

            //Deserialize Period and "stolen" Date references
            ObjectInputStreamin = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
            period = (Period) in.readObject();
            start = (Date) in.readObject();
            end = (Date) in.readObject();
        } catch (IOException | CLassNotFoundException e) {
            throw new AssertionError(e);
        }
    }
}

//Test
public static void main(String[] args) {
    MutablePeriod mp = new MutablePeriod();
    Period p = mp.period;
    Date pEnd = mp.end;

    //Let's turn back the clock
    pEnd.setYear(78);
    System.out.println(p);

    //Bring back the 60s.
    pEnd.setYear(69);
    System.out.println(p);


    //Output
    /**
     * Web Nov 22 00:21:29 PST 2017 - Web Nov 22 00:21:29 PST 1978
     * Web Nov 22 00:21:29 PST 2017 - Web Nov 22 00:21:29 PST 1969
     */
}
```

通过这个例子可以很清楚地看出通过反序列化的操作进而破坏`Period`的不变性. 面对这种问题该怎么做呢? 像构造函数一样, 添加校验和进行复制性拷贝.

```java
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();

    //Defensively copy our mutable components
    start = new Date(start.getTime());
    end = new Date(end.getTime());

    //Check that our invariants are satisfied
    if (start.compareTo(end) < 0) {
        throw new InvalidObjectException(start + " after" + end);
    }
}

/** Test Output
 * Web Nov 22 00:23:41 PST 2017 - Web Nov 22 00:23:41 PST 2017
 * Web Nov 22 00:23:41 PST 2017 - Web Nov 22 00:23:41 PST 2017
 */
```

什么时候需要进行复制性拷贝? 不仅仅是不变类, 只要你不想内部参数被毫无验证地设置, 你都可以进行复制性拷贝和验证. 注意这里有一个默认约定, 那就是不要在`readObject`内调用任何可重写的方法, 防止被异常攻击.

总而言之, 对于`readObject`方法:

- 对于任何实例对象, 都应该保持`private`, 并且在方法中进行检测和复制性拷贝.

- 如果检测出错的需要抛出`InvalidObjectException`.

- 如果整个对象必须序列化之后才能进行校验, 使用`ObjectInputValidation`接口.

- 不要在方法内部调用任何可重写的方法.

## Item 89: For instance control, prefer enum types to readResolve

对某些需要控制实例对象的类, 如单例模式:

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

如果需要实现序列化怎么办? 序列化相当于一个特殊的构造函数, 那怎么在序列化过程中保证实例控制呢? 可以通过`readResolve`方法进行过滤. JVM 在对对象进行序列化后, 如果发现对象实现了`readResolve`方法, 会调用该方法(产生的新对象)替换之前序列化的对象进行返回. 而我们只需要简单地在这个方法中进行过滤:

```java
// readResolve for instance control - you can do better!
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

使用这种方法时, 需要注意一点就是, 必须将内部所有的实例变量声明为`transient`, 防止在序列化的时候被攻击. 攻击的原理为: 修改序列化生成的字节码, 替换实例变量的对象, 在对象的内部实现`readResolve`, 对外部类进行拦截捕获. 简单的例子:

```java
// Broken singleton - has nontransient object reference field!
public class Elvis implements Serializable {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    //该实例域没有声明transient, 存在攻击的可能
    private String[] favoriteSongs ={ "Hound Dog", "Heartbreak Hotel" };
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
    private Object readResolve() {
    return INSTANCE;
    }
}

//用来拦截偷取信息的特殊类
public class ElvisStealer implements Serializable {
    static Elvis impersonator;  //存储拦截的信息为静态, 给外部提供访问接口
    private Elvis payload;      //存储序列化的结果(外部不变类)

    private Object readResolve() {
        // Save a reference to the "unresolved" Elvis instance
        impersonator = payload;
        // Return object of correct type for favoriteSongs field
        return new String[] { "A Fool Such as I" };
    }

    private static final long serialVersionUID =0;
}

public class ElvisImpersonator {
    // Byte stream couldn't have come from a real Elvis instance!
    //修改之后的字节码, 标准序列化有所差异
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x05,
        0x45, 0x6c, 0x76, 0x69, 0x73, (byte)0x84, (byte)0xe6,
        (byte)0x93, 0x33, (byte)0xc3, (byte)0xf4, (byte)0x8b,
        0x32, 0x02, 0x00, 0x01, 0x4c, 0x00, 0x0d, 0x66, 0x61, 0x76,
        0x6f, 0x72, 0x69, 0x74, 0x65, 0x53, 0x6f, 0x6e, 0x67, 0x73,
        0x74, 0x00, 0x12, 0x4c, 0x6a, 0x61, 0x76, 0x61, 0x2f, 0x6c,
        0x61, 0x6e, 0x67, 0x2f, 0x4f, 0x62, 0x6a, 0x65, 0x63, 0x74,
        0x3b, 0x78, 0x70, 0x73, 0x72, 0x00, 0x0c, 0x45, 0x6c, 0x76,
        0x69, 0x73, 0x53, 0x74, 0x65, 0x61, 0x6c, 0x65, 0x72, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 0x01,
        0x4c, 0x00, 0x07, 0x70, 0x61, 0x79, 0x6c, 0x6f, 0x61, 0x64,
        0x74, 0x00, 0x07, 0x4c, 0x45, 0x6c, 0x76, 0x69, 0x73, 0x3b,
        0x78, 0x70, 0x71, 0x00, 0x7e, 0x00, 0x02
    };

    public static void main(String[] args) {
        // Initializes ElvisStealer.impersonator and returns
        // the real Elvis (which is Elvis.INSTANCE)
        Elvis elvis = (Elvis) deserialize(serializedForm);
        Elvis impersonator = ElvisStealer.impersonator;
        elvis.printFavorites();
        impersonator.printFavorites();
        /**
         *  [Hound Dog, Heartbreak Hotel]
         *  [A Fool Such as I]
         *
         */
    }
}
```

这里例子需要理解 Java 序列化字节码信息, 可以[参照 jyt 的博客](https://www.jyt0532.com/2017/10/22/prefer-enum-for-instance-control/)进行深入理解. 当然解决方案也非常简单, 就是声明`favoriteSongs`为`transient`. 除了这种解决方案, 更好的解决方案为枚举类:

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    private String[] favoriteSongs ={ "Hound Dog", "Heartbreak Hotel" };
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
}
```

在某些情况下没办法使用枚举类完成时, 那就还是需要使用`readResolve`进行控制. 除了所有的实例变量都需要声明为`transient`之外, 还需要注意该方法的权限问题. 如果在`final`类上，应该设置为`private`。 如果是非`final`类上，则必须仔细考虑其可访问性。 如果它是私有的，则不适用于任何子类。 如果是`packageprivate`，它将仅适用于同一包中的子类。 如果它是受保护的或公共的，它将适用于所有不覆盖它的子类。 如果`readResolve`方法受保护或公共，并且子类不覆盖它，则反序列化子类实例将生成一个超类实例，这可能会导致`ClassCastException`。

总而言之, 如果需要序列化的对象需要进行实例控制的话, 推荐直接使用枚举类解决方案. 如果不行, 可以考虑使用`readResolve`来进行实例控制, 需要注意的是除了所有的变量都应该设置为`transient`之外, 还需要注意该方法的权限问题.

## Item 90: Consider serialization proxies instead of serialized instances

正如前面说的一个类实现序列化是非常危险的, 非常容易包含一些安全漏洞问题. 当然这里有一个通用的解决方法, 就是用代理序列化类. 简单的说就是将代理通过一个私有的静态内部类来完成, 将序列化的过程交给其完成, 保证与外部的隔离. 具体的例子, 以`Period`为例:

```java
public final class Period implements Serializable {
  private final Date start;
  private final Date end;
  public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0)
      throw new IllegalArgumentException(start + " after " + end);
  }

  private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;

    SerializationProxy(Period p) {
      this.start = p.start;
      this.end = p.end;
    }

  }
}
```

这里首先声明一个私有的静态内部类, 设置属性和外部类的信息一样. 然后我们就需要转交任务, 通过`writeReplace`(该方法会在序列化调用, 其返回的值替换成实际上要进行序列化的对象)进行传递实际上需要被序列化的对象. 这里传递内部的静态类.

```java
private Object writeReplace() {
  return new SerializationProxy(this);
}
```

然后为了防止别人修改序列化后的字节码, 重新序列化外部类, 我们需要在外部类添加过滤:

```java
private void readObject(ObjectInputStream stream)
    throws InvalidObjectException {
  throw new InvalidObjectException("Proxy required");
}
```

最后在`SerializationProxy`中添加`Period`的对象结果:

```java
private Object readResolve() {
  return new Period(start, end); // Uses public constructor
}
```

一个完整的代理类:

```java
public final class Period implements Serializable {
  private final Date start;
  private final Date end;
  public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());
    if (this.start.compareTo(this.end) > 0)
      throw new IllegalArgumentException(start + " after " + end);
  }

  private Object writeReplace() {
    return new SerializationProxy(this);
  }

  private void readObject(ObjectInputStream stream)
      throws InvalidObjectException {
    throw new InvalidObjectException("Proxy required");
  }

  private static class SerializationProxy implements Serializable {
    private final Date start;
    private final Date end;

    SerializationProxy(Period p) {
      this.start = p.start;
      this.end = p.end;
    }

    private Object readResolve() {
      return new Period(start, end); // Uses public constructor
    }
  }
}
```

当然代理类也存在弊端的: 性能降低了(额外引入了一个类). 内部类的`readResolve`方法只能调用构造函数, 不能调用其它实例函数(对象未构建好). 不适用于继承.

总而言之, 如果可以忽视这些弊端的话, 代理序列类可以完成健壮安全的序列化代码.
