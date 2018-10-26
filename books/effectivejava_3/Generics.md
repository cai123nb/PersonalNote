# Generics
自从Java5开始, Java引入了泛型. 在此之前, 每次从Collection中的读取一个对象都需要进行手动转换(cast), 如果错误的插入一个对象, 就会在运行时出现转换Error. 通过泛型, 告诉编译器该集合支持哪些类型对象, 编译器也会自动帮你转换对象, 如果你不小心插入一个错误的对象时就会在编译时就显示报错. 这样会让代码更加安全和简单. 这些优点的实现是需要付出一些代价的, 本章的关注就是如何最大化这些优点, 最小化产生的代价.

## Item 26: Don't use raw types
一般一个类或者接口在声明时都会使用一个或者多个类型参数. 如List接口中的定义为`List<E>`, 其中`E`就是类型参数, 使用尖括号包起来. `List<E>`就被叫做通用类型类, 可以用它来定义定义一些具体的类, 如: `List<String>`, 这就是一个参数化类型类. 暗示该集合中所有对象类型为String. 而原始类型(raw type)为擦除类型参数之后的对象, 如`List<E>`的原始类型就为`List`. 而原始类型的存在主要是为了兼容之前版本的代码.

```java
//Raw collection type - don't do this
//My stamp collection. Contains only Stamp instances
private final Collection stamps = ...;

//Erroneous insertion of coin into stamp collection
stamps.add(new Coin(...));	//Emits "unchecked call" warning

//Don't get error, until now 
//Raw iterator type - don't do this 
for (Iterator i = stamps.iterator(); i.hasNext();) {
	Stamp stamp = (Stamp) i.next();	//Throws ClassCast Exception 
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
	String s = strings.get(0);	//Has compiler-generated cast 
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

总而言之, 不要直接使用原始类型集合, 那只是用来兼容历史遗留代码使用. Set<Object>用以存储任何对象, Set<?>用于存储不清楚内部存储对象元素时. 推荐使用以替换原始类型. 原始类型只在获取类定义时有些作用.

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
objectArray[0] = "I don't fit in";	//Throws ArrayStoreException

//Won't compiler
List<Object> ol = new ArrayList<Long>();	//Incompatible types
ol.add("I don't fit in");
```

两种方法都不能成功添加, 但是第一种方法是在运行时才报错, 而第二种是在编译时报错. 那当然第二种更好了.

第二, 数组是具体化的(reified), 即数组在运行时可以知道并且强制限制其内部的成员类型. 就如前面往`Long`数组中添加`String`类型对象, 报`ArrayStoreException`异常. 而泛型这是通过擦除实现的, 即内部成员类型的保证是在编译期确定的, 在运行期间, 由于擦除了类型信息, 无法获知和保证成员类型信息.

这两个巨大的区别导致, 泛型和数组往往不能很好的配合. 如创建泛型数组是不合法的, 类似`List<E>[]`, `List<String>[]`等都是不合法的, 在编译期就会报错.


