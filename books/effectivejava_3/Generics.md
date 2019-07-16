# Generics

自从Java5开始, Java引入了泛型. 在此之前, 每次从Collection中的读取一个对象都需要进行手动转换(cast), 如果错误的插入一个对象, 就会在运行时出现转换Error. 通过泛型, 告诉编译器该集合支持哪些类型对象, 编译器也会自动帮你转换对象, 如果你不小心插入一个错误的对象时就会在编译时就显示报错. 这样会让代码更加安全和简单. 这些优点的实现是需要付出一些代价的, 本章的关注就是如何最大化这些优点, 最小化产生的代价.

## Item 26: Don't use raw types

一般一个类或者接口在声明时都会使用一个或者多个类型参数. 如List接口中的定义为`List<E>`, 其中`E`就是类型参数, 使用尖括号包起来. `List<E>`就被叫做通用类型类, 可以用它来定义定义一些具体的类, 如: `List<String>`, 这就是一个参数化类型类. 暗示该集合中所有对象类型为String. 而原始类型(raw type)为擦除类型参数之后的对象, 如`List<E>`的原始类型就为`List`. 而原始类型的存在主要是为了兼容之前版本的代码.

```java
//Raw collection type - don't do this
//My stamp collection. Contains only Stamp instances
private final Collection stamps = ...;

//Erroneous insertion of coin into stamp collection
stamps.add(new Coin(...));    //Emits "unchecked call" warning

//Don't get error, until now
//Raw iterator type - don't do this
for (Iterator i = stamps.iterator(); i.hasNext();) {
    Stamp stamp = (Stamp) i.next();    //Throws ClassCast Exception
        stamp.cancel();
}
```

在泛型引入之前, 使用原始类型是标准使用方式, 然而在引入泛型之后, 原始类型虽然还兼容, 但是远远不是标准了. 原始类型最大的问题便是如上所示. 设计程序的时候, 最希望的就是如果程序出现了问题, 应该尽可能快的反馈, 最好就是在编译期. 在这个例子中, 代码在编译期是不会报错的, 直到这个对象被使用的时候才会出现. 这时候你需要搜寻代码查找问题, 这时候编译器并不会帮你, 因为它不会理解你的注释. 但是如果使用泛型的话, 一切都会变得很简单:

```java
private final Collection<Stamp> stamps = ...; //Not use commet to indicate it.
```

这样编译时, 在我们插入Coin时就会报错, 这样取出对象时, 也不需要显式进行转换(编译器自动进行了转换并保证了成功). 但是如果你使用原始类型就会丢失泛型的所带来的安全性和便利性. 之所以允许原始类型的存在, 就是为了兼容之前的版本, 让其可以无痛迁移到新版本的Java中.

如果想要在集合中插入任意的对象, 那么使用`List<Object>`则是非常合适的. 那么就有人问`List`和`List<Object>`有什么区别呢? `List`则是显式说明需要退出泛型系统, `List<Object>`则是告诉编译器该集合可以存放任何类型的对象. 可以显式传递`List<String>`给`List`, 却不能传递给`List<Object>`. 因为`List<String>`是原始类型`List`的亚型(Subtype), 而不是`List<Object>`的参数化类型. 简单来说, 就是使用原始类型`List`就会放弃了类型安全的检查.

如:

```java
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(32));
    String s = strings.get(0);    //Has compiler-generated cast
}

private static void unsafeAdd(List list, Object o) {
    list.add(o);
}
```

这时候程序可以正常编译, 但是会在`unsafeAdd`中提示一个`unchecked`警告. 最终运行时, 在`String s = strings.get(0)`时出现`ClassCastException`, 因为编译器进行转换的时候, 转换失败了. 这就是典型的使用原始类而导致放弃类型检查出现的问题. 最简单的方法就是在`unsafeAdd`中使用`List<Object>`, 那样在编译时就会在调用`unsafeAdd`时报错, `incompatible types`.

有时候你不关注集合中的对象类型, 使用原始类型来存储任意的类. 如:

```java
static int numElementsInCommon(Set s1, Set s2) {
    int result = 0;
    for (Object o : s1)
        if (s2.contains(o))
            result++;
    return result;
}
```

这个方法可以正确执行, 但是使用了原始类, 这里推荐使用无界通配符`?`. 如`List<E>`的无界通配符的标示为: `List<?>`.

```java
static int numElementsInCommon(Set<?> s1, Set<?> s2) {
    int result = 0;
    for (Object o : s1)
        if (s2.contains(o))
            result++;
    return result;
}
```

那`List`和`List<?>`有什么区别呢? 正如前面的`unsafeAdd`, `List`可以添加任意对象, 而`List<?>`不能, 甚至`List<?>`只能添加null. 记住, 你不能往`Collection<?>`中添加任何对象, 除了`null`.

当然原始类型也有一些地方需要用到, 如类定义的时候. `List.class`是合法的, 而`List<String>.class`是不合法的. 同理可得`instanceof`调用, 传递参数的时候, 如: `if(xxx instanceof Set){}`.

总而言之, 不要直接使用原始类型集合, 那只是用来兼容历史遗留代码使用. `Set<Object>`用以存储任何对象, Set<?>用于存储不清楚内部存储对象元素时. 推荐使用以替换原始类型. 原始类型只在获取类定义时有些作用.

## Item 27: Eliminate unchecked warnings

当使用泛型的时候, 往往在编译的时候会提示各种类型的警告: `unchecked cast warning`, `unchecked method invocation warning`, `unchecked parameterized vararg type warning`和`unchecked conversion warnings`.  当我们对泛型用的越多, 往往警告越少, 但是不要期待一开始写出的代码就完全没有警告, 更多的需要后续的改进和优化.

这些警告往往都是特别容易消除的. 如:

```java
Set<Lark> exaltation = new HashSet();
```

这时候会提示: `warning: [unchecked] unchecked conversion. required: Set<Lark>, found: HashSet.` 这时候只要简单的在`HashSe`后面添加尖括号即可:

```java
Set<Lark> exaltation = new HashSet<>();
```

有些警告是比较难消除的, 需要花费一些大工夫. 但是这是值得的, 尽量消除每一个`unchecked warning`是最终目标. 这样可以保证代码在运行期间肯定不会出现`ClassCastException`, 保证代码的健壮性.

对于一些警告, 你没有办法进行消除, 但是你可以证明弹出警告的代码是安全的, 只有在这种情况下, 你可以通过`@SuppressWarnings("unchecked")`注解压制这个警告. 如果你不能证明该代码是安全的, 那么你只能获得一个虚假的安全感, 在运行时往往还有可能出现`ClassCastException`. 并且相反, 如果你知道该代码(弹出警告)是安全的, 并且你不进行压制. 那么可能后续的代码(不安全的)弹出新警告就会混在其中, 容易让你忽视.

对于`@SuppressWarnings("unchecked")`注解, 可以用在任何地方, 从一个局部变量到整个类都是可以的. 但是请遵循一个原则, 尽可能限制小的使用范围. 通常的方法往往是使用在一些小方法内, 构造函数, 甚至局部变量内. 千万不要将注解使用在一整个类中, 这很有可能隐藏很多致命的警告.

如果将注解放在一个函数外时, 你发现函数很长, 往往不止一行. 这时候推荐你将注解使用在内部变量声明中. 如`ArrayList`中的`toArray`方法.

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    System.arraycopy(elemtns, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

编译的时候, 会在`return (T[]) Arrays.copyOf(elements, size, a.getClass());`这行报`[unchecked] unchecked cast ...`警告. 这时候直接在方法外部声明注释`@SuppressWarnings("unchecked")`是可以的. 但是更好的方式是在内部使用:

```java
public <T> T[] toArray(T[] a) {
    if (a.length < size)  {
        @SuppressWarnings("unchecked") T[] result = (T[]) Arrays.copyOf(elements, size, a.getClass());
        return result;
    }
    System.arraycopy(elemtns, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

另外, 在使用`@SuppressWarnings("unchecked")`时, 添加一行注解, 解释一下为什么这是安全的. 这样便于后人理解代码. 如果你发现则很难写注释, 你可能最后会发现这并没有你想象的安全.

总而言之, 尽量消除每一个`unchecked`警告. 每一个`unchecked`警告说明代码中存在`ClassCastException`的风险. 如果实在无法消除, 且可以保证该代码是安全的, 合理的使用`@SuppressWarnings("unchecked")`进行注解, 但是请保证使用实在最小的范围内, 并为每一个注解添加注释, 解释一下为什么这是安全的.

## Item 28: Prefer lists to arrays

数组和泛型有很大的区别, 主要体现在两个方面. 第一, 数组是协变的(covariant), 简单来说就是: 如果`Sub`是`Super`的子类, 那么`Sub[]`数组也是`Super[]`数组的子类. 而对于泛型来说, 却是不变的(invariant). 对于两个不同的类,`Type1`和`Type2`, `List<Type1>`既不是`List<Type2>`的子类, 也不是父类. 从这里看, 你可能会觉得泛型的功能比不上数组, 但是结果往往相反. 如

```java
Object[] objectArray = new Long[1];
objectArray[0] = "I don't fit in";    //Throws ArrayStoreException

//Won't compiler
List<Object> ol = new ArrayList<Long>();    //Incompatible types
ol.add("I don't fit in");
```

两种方法都不能成功添加, 但是第一种方法是在运行时才报错, 而第二种是在编译时报错. 那当然第二种更好了.

第二, 数组是具体化的(reified), 即数组在运行时可以知道并且强制限制其内部的成员类型. 就如前面往`Long`数组中添加`String`类型对象, 报`ArrayStoreException`异常. 而泛型这是通过擦除实现的, 即内部成员类型的保证是在编译期确定的, 在运行期间, 由于擦除了类型信息, 无法获知和保证成员类型信息.

这两个巨大的区别导致, 泛型和数组往往不能很好的配合. 如创建泛型数组是不合法的, 类似`List<E>[]`, `List<String>[]`等都是不合法的, 在编译期就会报错. 为什么泛型数组是不合法的呢? 因为这不是类型安全的, 如果允许泛型数组就可能在运行时出现转换问题, 导致`ClassCastException`. 如:

```java
//Pretend the line is legal.
List<String> strings = new List<String>[1];
List<Integer> initList = List.of(43);
Object[] objects =  strings;        //Obtain objects from strings.
objects[0] = initList;                //This is legal for array.
String s = strings[0].get(0);        //ClassCastException.
```

从上面可以知道, 非常容易就出现了异常. 其中`objects[0] = initList;`是合法的, 因为在编译器中通过擦除实现的, `List<Integer>`和`List<String>`都是一样的.

像`E, List<E>, List<String>`这种类型都是非具体化的类型. 也就是说这种类型在运行时拥有的信息比编译时要少很多. 因为擦除. 唯一具体化的参数类型是`?, List<?>`, 但是很少使用这种类型来创建数组.

禁止泛型数组的创建, 有时候是非常烦人的. 如你在一个泛型对象中是没有办法返回内部元素的数组对象如`T[]`, 这时候推荐使用`List<E>`, 而不是数组`E[]`.

```java
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray();    //Won't compile
    }

    ...// The other is omitted.
}
```

这时候, 这段代码是不会运行的, 编译出错. 也许你经验丰富, 添加一个转换语句:

```java
choiceArray = (T[]) choices.toArray();
```

这时候代码是不会编译出错的, 但是却会抛出一个警告: unchecked cast. 编译器不能保证在运行时,这行代码可以正确转换, 你必须自己进行证明, 并添加在注释中, 然后压制这个警告`@SuppressWarnings("unchecked")`. 但是更好的方法是使用`List<E>`.

```java
public class Chooser<T> {
    private final List<T> choiceAList;

    public Chooser(Collection<T> choices) {
        choiceAList = new ArrayList<>(choices);    //Won't compile
    }

    ...// The other is omitted.
}
```

虽然这个版本, 可能在性能上会差一点, 但是却带来了程序的可读性和健壮性, 不用担心潜在的转换异常.

总而言之, 数组和泛型有很大的区别. 数组是协变的和具体化的. 泛型是不变的和擦除的. 作为结果, 数组提供运行时的类型安全检查而不是编译时的类型安全检查, 对于泛型却恰恰相反. 所以, 数组和泛型不能很好在一起工作. 当你发现数组和泛型使用时, 存在很大问题时, 第一个可选的方法可以考虑使用`list`替换数组.

## Item 29: Favor generic types

我们经常使用JDK提供的各种泛型, 我们自己写的话, 就有点复杂了, 但是这种努力和时间是值得的. 如:

```java
//Obejct-based collection - a rpime candidate for generics
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEAFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEAFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null;
        return result;
    }

    public boolean isEmpty() {
        reutrn this.size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

这个类一开始设计的时候就应该考虑泛型化, 但是没有. 现在可以对这个类进行泛型化处理, 并且不影响之前的使用. 首先在类的声明中添加一个或者多个参数类型, 这里只需要添加一个参数化类型, 这里声明为`E`. 然后将所有的Object替换为合适的参数化类型.

```java
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEAFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEAFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null;
        return result;
    }

    public boolean isEmpty() {
        reutrn this.size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

编译代码, 对错误的地方进行修复. 首先出现的问题是: `elements = new E[DEAFAULT_INITIAL_CAPACITY];`, 然后程序编译出错. 这是一个最常见的问题, 泛型数组的问题. 这里首先声明为Object, 采用类型转换`(T[])`, 这时候提示警告: Unchecked cast. 编译器无法保证在运行时这段代码可以正确进行转换. 这时候, 我们把理由写上, 并压制警告:

```java
// The elemets array will contain only E instances from push(E).
// This is sufficient to ensure type safety, but the runtime
// type of the array won't be E[]; it will always be Object[]!
@SuppressWarnings("uncheckned")
public Stack() {
    elements = (E) new Object[DEAFAULT_INITIAL_CAPACITY];
}
```

当然你也可以声明elements为`Object[]`, 然后在弹出的时候进行类型转换, 同样进行压制注释等等. 但是前面这种的可读性会更好, 并且减少了`cast`的调用次数.  所以前面这种使用的也更加广泛. 第一种也是存在问题的, 会导致堆污染(Heap pollute, Item 32): 运行时类型和编译时类型不匹配. 如果这个污染严重的话, 可以考虑第二种方法. 这里并不是很严重, 采用第一种.

这时你可能觉得这不是违反了`Item28`说的, 优先使用`list`来替代`array`吗. 是的, 这个是违反了. 那个是优先并不是强制要求. 首先对于list, JDK并没有本地支持, 对于一些泛型类`ArrayList`必须使用数组来说实现, 另外对于一些追求性能的类, 如`HashMap`也是使用数组来实现, 来保证性能.

另外, 这里的参数类型是没有进限制的, 即你可以传递和设置任意的参数, 如: `Stack<Object>, Stack<int[]>, Stack<List<String>>`等等, 注意这里由于JDK本身的限制, 不能直接使用原始类型, 不过可以使用封装类型进行替换. 你这里可以限制传递的参数类型. 如`java.utilconcurrent.DelayQueue`:

```java
class DelayQueue<E extends Delayed> implements BlockingQUeue<E>
```

这里的参数类型E限制为`java.util.concurrent.Dealyed`的子类, 这样在`DelayQueue`中就可以显示使用`Dealyed`的方法, 而不需要进行显示转换. 这就是常称的有界通配符. 注意的是, 虽然说是`E extends Delayed`, 但是传递`Dealyed`也是合理的.

总而言之, 泛型可以更加安全和方便使用, 如果你的类没有泛型化, 那么可以好好考虑发现化.

## Item 30: Favor generic methods

正如类可以被泛型化, 方法同样可以泛型化. 如: `Colletions`内所有的工具方法都是泛型化的. 写泛型方法, 也非常类似泛型类, 比如:

```java
//Use raw types - unaccpetable
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);
    result.addAll(s2);
    return result;
}
```

这个方法编译的时候, 提示两个警告. `Set result = new HashSet(s1);` : `unchecked call HashSet as raw type`. `result.addAll(s2)` : `unchecked call to addAll as raw type Set`. 这暗示调用的过程是使用raw type的. 要消除则警告就要对这个方法使用的三个集合进行泛型化. 添加也简单, 在方法描述符和返回值之间添加泛型参数, 使用`<>`包起来, 修改方法添加对应的泛型参数.

```java
//Generics method
public static <T> Set<T> union(Set<T> s1, Set<T> s2) {
    Set<T> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```

对于简单的方法, 这样做就可以了. 消除了所有的警告, 也方便类调用.

```java
Set<String> guys = Set.of("Amy", "Jake", "Tom");
Set<String> stooges = Set.of("Alika", "Tikala", "Matten");
Set<String> all = union(guys, stooges);
```

这里有一个小小的限制, 那就是三个Set的参数化类型必须完全一样. 可以通过`有界通配符`来更加灵活的完成方法的泛型化.

有时, 你需要创建一个不变的对象, 但是要应用在很多不同类型的对象上. 因为泛型内部实现使用的是擦除, 你可以创建一个通用的全局的对象, 然后通过静态方法为不同的泛型返回不同的泛型的该对象. 这种模式就是常用的`泛型单例工厂模式`. 如`Collections.reverseOrder`.

```java
@SuppressWarnings("unchecked")
public static <T> Comparator<T> reverseOrder() {
    return (Comparator<T>) ReverseComparator.REVERSE_ORDER;
}
```

假设你想要写一个identity方法, 简单的返回自身. 这时候只需要定义一个通用的对象, 然后按照不同的泛型参数类型进行返回.

```java
private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;

public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN;
}
```

注意这里会抛出一个警告`unchecked cast`, 编译器认为`UnaryOperator<Object>`不一定是`UnaryOperator<T>`, 有可能出现转换失败情况. 但是这个函数是特殊的, 这个函数没有修改参数, 只是单纯的返回自身. 可以知道对于任意参数, 这是安全的. 可以压制这个警告, 保证所有的调用都是干净的.

这里有一个特殊的情况, 那就是`递归类型绑定`. 就是类型参数中夹杂着别的类型参数. 最经典的就是`Comparable`接口.

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

这个类型参数T一般是自身, 因为对象比较时一般只允许和自身进行比较. 如`String`就实现`Comparable<String>`, `Integer`就实现`Comparable<Integer>`. 对于实现这个接口的对象, 说明这些对象是有顺序的, 可以进行比较的. 那么放在集合中就可以求最大值, 最小值, 排序等操作. 如:

```java
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);
    return result;
}
```

上面就是一个典型的求最大值的工具方法. 声明中的`<E extends Comparable<E>>`, 就是递归类型绑定. 幸运的是, 这种使用比较少见.

总而言之, 泛型方法类似泛型类, 更加安全和便于使用, 使用时就不需要显式对参数和返回值进行转换. 当你写的方法, 发现经常需要进行转换时, 考虑将它泛型化.

## Item 31: Use bounded wildcards to increase API flexibility

正如前面`Item28`说的, 泛型是不变的和擦除的, `List<String>`即不是`List<Object>`的子类, 也不是其的父类. 虽然这有点奇怪, 但是不难理解: 一个容器装了父亲, 一个容器装了儿子, 我们不能光凭这个就说明前面是后者的父类, 因为这个不仅仅只有父亲, 还有很多别的东西, 如容器, 数量等等. 但是这个特性有时候还是会带来一些不便. 如我们设计的一个简单的Stack类:

```java
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```

这是基本的方法, 后面我们按照需求添加类一个新的接口pushAll(传递一个集合, 将集合中的所有对象添加到原来的对象中):

```java
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```

这个方法成功编译了, 并且运行良好. 但是这还是不够的. 假设我们有一个`Stack<Number>`的数据, 突然产生了一组`List<Integer>`的数据, 想要将这些数据通过这个方法放入内部. 按照`push`方法是可以成功放入的, 因为`Integer`是`Number`的子类, 可以正常放入. 但是调用时, 却爆出了`error: Iterable<Integer> can't be convert to Iterable<Number>`. 很明显这就是因为泛型的不变性导致的.

为了解决这个问题, Java提供了一个很好的工具: 有界通配符. 为了在`pushAll`中兼容E的子类(Number的子类, Integer), 在该方法的声明中使用`<? extends E>`:

```java
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
```

这样就可以成功兼容所有E的子类, `Stack`可以通过该方法, 放入任何包含子类的集合了. 这时候我们想对应该方法, 书写一个`popAll`方法, 传递一个集合, 然后弹出所有该栈中的对象, 放入集合中.

```java
public void popAll(Collection<E> dst) {
    while(!isEmpty())
        dst.add(pop());
}
```

同样可以编译成功. 但是也同样功能不够齐全, 如果我们想将`Stack<Integer>`的数据放入`List<Number>`中, 这个代码同理也会报错. 按照逻辑来说也不应该报错的. 这时候, 可以使用`<? super E>`:

```java
public void popAll(Collection<? super E> dst) {
    while(!isEmpty())
        dst.add(pop());
}
```

这样也就解决了兼容性的问题. 到这里我们可以得出一个结论: 为了最大化灵活性, 为消费者函数和生产者函数使用有界通配符, 是一个很好的选择. 如果函数的输入参数, 既是消费者, 也是生产者, 那么就不要使用通配符, 还是使用确定的参数类型. 这里有一个简单的口诀来记录: `消费者函数使用super, 生产者函数使用extends`. 如这里的`pushAll`是典型的生产者, 生成E实例给Stack, 那就使用`extends`, `popAll`是典型的消费者, 消耗Stack内的对象, 就使用`super`.

带着这样的窍门, 我们重温一下之前的方法和函数. `Item28`中的:

```java
public Chooser(Collection<T> choices);
```

所有的构造函数都是生产者, 这里使用`extends`.

```java
public Chooser(Collection<? extends T> choices);
```

`Item30`中的:

```java
public static <T> Set<T> union(Set<T> s1, Set<T> s2);
```

可以很清楚的知道, 传递过来的两个参数都是生产者, 最后生产一个集合.

```java
public static <T> Set<T> union(Set<? extends T> s1, Set<? extends T> s2);

Set<Integer> integers  = Set.of(1,2,3);
Set<Double> doubles = Set.of(1.2, 3.2, 1.3);
Set<Number> numbers = union(integers, doubles);
```

注意这里不要在返回值里使用通配符`?`, 因为这样会强制用户在使用时添加通配符, 增加了用户的复杂度. 真正良好的通配符应该是无感的, 用户是没有感觉的, 但是却良好地完成工作: 接受该接受的对象, 拒绝该拒绝的对象. 如果你的通配符需要用户在使用时需要顾虑的话, 那你的API可能就不太对.

这里有一点需要注意, 那就是在Java8之前, 编译器的推理功能还没这么强大, 如果需要使用上述的语句时, 需要手动告诉编译器. 不然编译器会不识别.

```java
Set<Number> numbers = Uunion.<Number>union(integers, doubles);
```

`Item30`中的:

```java
public static <E extends Comparable<E>> E max(Collection<E> c);


//revised
public static <E extends Comparable<? super E>> E max(Collection<? extends E> c) {
```

这里的`Collection<E> c`很明显是生产者(原材料), 使用`extends`, 而最终的结果E为消费者, 并且E必须是可比较的, 使用`super`, 这里的含义是调用者(E)可以不实现`Comparable`接口, 实际使用的(?)可以是父类, 父类实现了`Comparable`接口. 注意, 所有的`Comparable<T>`和`Comparator<T>`都是消费者, 推荐优先使用`Comparable<? super T>`和`Comparator<? super T>`.

这里花了很大代价来实现这个功能, 这样做有效果吗? 是的, 是有的.

```java
List<SchduledFuture<?>> schuduledFutures = ...// invoke max
```

这个在之前的版本是不可以的, 但是在修订的版本是可以的. 因为`SchduledFuture`没有实现`Comparable`接口, 所以第一个版本拒绝编译. 而修订版本, 发现该接口的父接口`Delayed`, 实现了`Comparable`接口, 允许调用.

通配符还有一点需要讨论的是. 如果我们需要实现一个静态工具方法, 将一个集合中的两个对象交换位置. 有两种实现方式:

```java
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j);
```

你会选哪种, 那肯定是第二种呀. 第二种简单明了. 你可以传递任何类型的通配符进行匹配. 这里有一个规则, 如果一个泛型参数, 只在方法中出现了一次. 那么推荐使用通配符`?`进行替代. 这里你可能实现的版本为:

```java
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i));
}
```

但是可惜的是, 程序并不能编译通过. `error: incompatible type, object can't be convert to CAP#1`. 为什么会这样呢? 因为`List<?>`为无界通配符, 你不能放入任何对象, 除了null. 因为编译器不能理解?, 会自动去猜对应的值, 然后赋予一个自认为的类型`CAP#1`, 这明显不能匹配任何对象. 怎么解决这个问题呢? 通过辅助方法.

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i));
}
```

这样就可以完美解决了, 通过一个媒介方法, 帮助编译器明白传递的参数类型. 你会发现这个辅助方法和之前的第一个方法, 完全一样.... 虽然用户使用方便了, 但是内部的复杂由后台承受了.

总而言之, 为你的方法添加泛型, 这会使得方法更加灵活. 如果你想让你的API被广泛使用, 合理的使用通配符. 记住PECS原则(Produce extends, Consumer super). 所有的`Comparable`和`Comparator`都是消费者.

## Item 32: Combine generics and varags judiciously

可变参数和泛型都是在Java5加入JDK的. 但是可变参数和泛型却不能很好的配合: 可变的参数的实现, 内部通过编译器传递一个数组来存储这些对象. 而数组是具体化的, 泛型却是相反的, 在运行期是擦除了信息的. 所以当我们声明泛型的可变参数时, 编译器会提示一个警告. 而当我们在方法内部调用该泛型参数时,也会提示警告. 警告类似: `unchecked possible heap pollution ...`. `Heap pollution`就是当参数化类型引用指向的对象不是该类型时, 就会产生. 这个会导致编译器自动产生的`cast`有可能失败, 违背了泛型的原则.

```java
//Mixing generics and varags can violate type safety!
static void dangerous(List<String>... stringLists) {
    List<Integer> intList = List.of(42);
    Object[] objects = stringLists;
    object[0] = intList;                    //Heap pollution
    String s = stringLists[0].get(0);        //ClassCastException
}
```

这么这个例子就可以简单说明泛型的可变参数时不安全的. 其中最后一行代码爆出的异常, 就是由编译器自动生成的`cast`方法调用出错. 这说明了一个很重要的问题: `往泛型参数的可变数组内存储对象(或者修改)是非常不安全的`.

也许你会问, 为什么声明泛型的可变参数合理, 声明泛型数组却不合理呢? 两者本质都是数组呀. 是的, 这是前后矛盾的. 主要是JDK的设计者发现泛型的可变参数在实际使用时, 非常方便, 提供了很好的辅助作用, 也就默认这个的存在. 并且在JDK中, 如`Arrays.asList(T... a), Collections.addAll(Collection<? super T> c, T... elements)`等泛型可变参数都是类型安全的, 并不像前面的这个例子这么危险.

在Java7之前, 人们使用泛型的可变参数时是非常难受的, 因为每次调用这类方法编译器都会抛出警告, 你为了压制这些警告, 只能在调用这类方法的方法外部添加`SuppressWarnings("unchecked")`来压制. 这是非常乏味且影响代码阅读的. 在Java7之后呢, 引入了一个新的注解: `@SafeVarargs`, 含义就是告诉编译器这个泛型的可变参数时类型安全的, 不会出现问题的. 编译器也就不会抛出警告了. 这样其它方法调用的时候, 也就不会得到警告了.

对于`@SafeVarargs`, 这是和编译器的一个约定. 但是这个约定需要你自己来完成: 在完全确定泛型可变参数方法是类型安全之后再添加该注解. 那怎么确保该方法(包含泛型可变参数)时类型安全的呢? 这里有两条准则: `对于泛型的可变参数数组不要进行任何的修改`, `不要让泛型的可变参数数组的引用逃逸出方法外部, 即保证只能在方法内部使用`. 如果保证满足这两个条件的话, 就可以说这个方法是类型安全的. 如这里举一个例子说明:

```java
//UNSAFE - generic parameter array reference escaped out the method
static <T> T[] toArray(T... args) {
    return args;
}
```

这个方法看起来就是一个简单的工具方法, 没什么问题, 但是却是非常危险的, 它会将`Heap pollution`传播到方法调用者. 假设基于这个方法实现一个工具方法:

```java
static <T> T[] pickTwo(T a, T b, T c) {
    switch (ThreadLocalRandom.current().nextInt(3)) {
        case 0: return toArray(a, b);
        case 1: return toArray(b, c);
        case 2: return toArray(a, c);
    }
    throw new AssertionError();    //Can't get here
}

public static void main(String[] args) {
    String[] attributes = pickTwo("Good", "Fast", "Cheap");
}
```

这一切代码都可以正常编译, 没有任何问题. 但是我们运行时, 却会在`pickTwo`调用时爆出`ClassCastException`. 为什么会这样呢? 因为我们运行时调用`pickTwo`时, 编译器并不能理解T是什么, 于是调用`toArray`方法时就创建了`Object[]`数组来进行存储和返回. 而我们实际使用的却是`String[]`, `Object[]`并不是`String[]`的父类或子类, 无法成功进行转换(`cast`), 所以爆出这个异常. 这也就是`toArray`的`Heap pollute`传播到这里导致的.

这个例子说明了第二点: `让别的方法可以访问的泛型可变参数数组是非常危险的`. 这里有两个例外: `除非别的方法是@SafeVarargs类型的方法`, 或者`别的方法是固定参数个数, 并且只是单纯对数组内的元素进行值的计算`. 上面的`pickTwo`方法虽然是固定参数的, 但是却不是对数组进行简单计算, 而是直接传播出去了. 这里举一个简单的安全使用的例子:

```java
@SafeVarargs
static <T> List<T> flatten(List<? extends T>... lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

这里再次重申一下两个准则:

+ ** 对于泛型的可变参数数组不要进行任何的修改 **
+ ** 不然让泛型的可变参数数组被不安全的代码接触 **

当然这里还有一个折中的方法, 正如`Item28`所说的, 使用`List`来替代数组:

```java
static <T> List<T> flatten(List<List<? extends T>> lists) {
    List<T> result = new ArrayList<>();
    for (List<? extends T> list : lists)
        result.addAll(list);
    return result;
}
```

这样就不用担心类型安全的问题了, 唯一的缺点就是相比前面的代码, 性能会差一点.

总而言之, 可变参数和泛型不能很好的搭配使用, 因为可变参数的本质是数组, 而数组和泛型有很多冲突. 但是这依然是合理的, 如果你确定你的泛型可变参数方法是类型安全的, 添加`@SafeVarargs`注释, 来让这个方法更加方便使用.

## Item 33: Consider typesafe heterogeneous containers

在我们使用泛型时, 一般就是通过一些集合, 如`Set<E>`, `Map<K,V>`, 或者一些单元素的容器, 如`ThreadLocal<T>`,`AtomicReference<T>`等等. 一般来说泛型参数的数量都是固定的, 如`Set<E>`只有一个`E`代表`Set`中的元素类型, `Map<K,V>`只有两个`K`,`V`代表Map中的键和值两个对象. 一般正常使用是可以满足的. 但是如果你想获得更大的灵活性: 自定义多少个泛型参数.

在Java中如果想要实现`自定义多少个泛型参数`这种做法是可以实现的. 原理是: 前面的例子都是对容器进行泛型化(如对Set进行泛型化为`Set<E>`), 而我们可以对`key`进行泛型化, `key`就如同`Map`中的key, 用来放置和取出对象. 然后将这个`泛型化的key`放入容器中来进行存储, 最后利用泛型来保证key对应的对象是正确的类型.

如我们设计一个类`Favorites`, 来存储任意类型我们喜欢的实例. 这时候, 我们的`key`就可以设置为`Class`, 因为`Class<T>`是泛型, 如`Integer.class`就为`Class<integer>`, `String.class`就为`Class<String>`. 并且`Class`的文字信息可以在方法中进行传递(无论编译期还是运行期), 这就被称作类型秘钥(`type token`).

类`Favorites`就如同一个简单的Map, 不过其中的`key`是泛型的, 这是API方法:

```java
//Typesafe heterogeneous container pattern - API
public class Favorites {
    public <T> void putFavorite(Class<T> type, T instance);
    public <T> T getFavorite(Class<T> type);
}
```

这里是简单的测试函数:

```java
public static void main(String[] args) {
    Favorites f = new Favorites();
    f.putFavorite(String.class, "Java");
    f.putFavorite(Integer.class, 0xcafebabe);
    f.putFavorite(Class.class, Favorites.class);
    String favoriteString = f.getFavorite(String.class);
    int favoriteInteger = f.getFavorite(Integer.class);
    Class<?> favoriteClass = f.getFavorite(Class.class);
    System.out.printf("%s %s %s%n", favoriteString, favoriteInteger, favoriteClass);
}
```

注意这里使用`%n`来保证平台兼容性的换行. 这里的Favorites类是类型安全的, 当你想要获取什么类型时, 自动获取什么类型. 并且是`异构的`, 不像传统的`Map`, Favorites内的`key`是不同类型的, 泛型化的.

```java
//Typesafe heterogeneous container pattern - implementation
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(Object.requireNonNull(type.class), instance);
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

这里简单说明一些, 首先从代码中可以知道, 所有的对象都是存储在HashMap中. 有人可能会认为`Map`中使用无界通配符, 怎么还可以往里面放入元素(一般来说含通配符的对象只能放入null), 但是这里需要注意的是, 这里的通配符是嵌套的, 并不是指Map是无界的, 而是说内部的key是无界的. 意思就是说, key可以为任何类型的Class. 这就是异构(`heterogeneous`)的来源.

第二个需要说明的是, 这里的`map`存储的是`Class`和`Object`, 也就是说`map`是不会保证`class`和`object`的类型对应的. `map`不会帮你确认存入的`object`是不是就是对应`class`的实例. 但是实际上, 这是可以得到保证的, 只是`Java type system`并没有明说而已, 下面详细说明:

`putFavorite`简单往map中进行放入对象, 使用泛型来保证类型一致. `getFavorite`方法通过`Class`的`cast`进行转换(`cast`中如果不是对应类型的就抛出`ClassCastException`. 这样可以保证, 只要客户端编译通过了(即编译时泛型校验成功了, 可以正确放入), 那么第二个方法取出时就肯定不会出错, 也就是类型安全的(typesafe).

这里有两点是需要注意, 第一, 就是如果客户端使用原始类型`Map`, 不使用`Map<Class<?>, Object>`的话, 很有可能编译通过, 但是运行时出现`ClassCastException`. 这时候为了防止运行时出现这种情况, 可以在`putFavorite`方法中添加校验:

```java
public <T> void putFavorite(Class<T> type, T instance) {
    favorites.put(Object.requireNonNull(type.class), type.cast(instance));
}
```

这种做法在`Collections`中被广泛使用, 如`CheckedList`, `CheckedMap`等, 保证运行时的类型安全.

第二就是, 该类不支持泛型. 因为泛型通过擦除, 所有的泛型类的Class都是一样的.

另外这里使用的是无界通配符, 如果想要添加限制也是可以的, 可以使用有界通配符. 如:

```java
public <T extends Annotation> T getAnnotation(Class<T> annotationType);
```

这里有一个特殊情况, 如果你在一个无界通配符中想要调用一个有界通配符的方法. 就比如上面的`getAnnotation`方法, 直接使用`<? exnteds Annotation>`进行修改也是可以, 但是这个转换是`unchecked`的. 这里推荐使用`Class.asSubclass`方法.

```java
static Annotation getAnnotation(AnnotationElement element, String annotationTypeName) {
    Class<?> annotationType = null;
    try  =
        annotationType = class.forName(annotationTypeName);
    } catch(Exception ex) {
        throw new IllegalArgumentException(ex);
    }
    return element.getAnnotation(annotationType.asSubclass(Annotation.class));
}
```

这样就没有任何警告和错误了.

总而言之, 普通的泛型限制了泛型参数的个数. 可以通过类型安全的异构容器来实现: 不对容器进行泛型化, 而是对于key进行泛型化. 常用的key为Class对象, 当然你可以自定义key对象.
