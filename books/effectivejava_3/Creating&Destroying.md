# Creating and Destroying Objects

When and how to create objects, when and how to avoid creating them, how to ensure they are destroyed in a timely manner, and how to manage any cleanup actions that must precede their destruction.

## Item 1: Consider static factory methods instead of constructors.

有时候我们创建一个对象, 并不一定需要调用对象的构造函数, 而是采用静态工厂的方法的形式, 往往可以带来更好的性能和内存优势.

如, 我们新建一个Boolean对象, 我们可以不采用构造函数方法, 而是采用Boolean的valueOf方法:

```java
public static Boolean valueOf(boolean b) {
	return b ? Boolean.TRUE : Boolean.FALSE;
}
```

这样可以避免重复构造Boolean对象, 而采用Boolean内部已经预构造好的对象, 减少了JVM的负担.

采用静态工厂方法有自己的优势, 也有自己的劣势.

优势:

+ 静态工厂方法有自己的方法名. 一个好的方法名不仅利于使用, 还提高了代码的可读性. 在构造函数中, 修改两个参数的顺序就可以产生一个新的构造函数, 但是对于使用者来说不查看对应的API文档就非常容易出错, 导致严重的后果. 但是静态工厂方法则没有这个烦恼, 通过差异性明显的方法名就可以区分.

+ 调用静态方法有时候并不一定需要创建一个新的对象. 很多不变的类往往预实例对象或者缓存一部分对象在构造的时候,  重复使用这些对象可以避免创建重复的对象, 这可以显著提高程序的性能. 如Boolean内部定义的TRUE和FALSE对象, 通过valueOf就可以获取预定义好的对象引用了.

+ 调用静态方法返回的对象不一定是对象本身, 可以是对象的子类. 这在你返回对象的时候, 提供了良好的灵活性: 隐藏内部的实现, 当你需要修改内部实现时, 外部调用却并不需要进行修改, 甚至没有察觉.

+ 调用静态方法可以根据传递的参数进行区分化的对象返回. 更加不同的参数, 可以返回不同的子类型的对象, 来带来更好的定制化的特性. 如EnumSet中的noneOf方法, 根据枚举的数量进行区分, 如果超过了64个对象, 则返回JumboEnumSet对象, 否则返回RegularEnumSet对象.

+ 调用该方法时, 返回的对象这时候并不需要自己实例化. 如我们的Service Provider Framework, 当我们向服务提供商请求某项服务时, 我们并不需要自己去实例化, 我们只需要知道向其申请即可, 不需要知道其是怎么实现的.

劣势:

+ 如果类没有public或protected的构造函数, 该方法就无法子类继承和修改.

+ 很难被用户找到并使用. 常见静态方法名称有: `of`, `from`, `valueOf`, `instance`, `getInstance`, `create`, `newInstance`, `getType(Type是指对象名称, 如getFileStore)`, `newType`, `type`.

当我们面对静态工厂方法和构造函数时, 可以充分考虑两者的优缺点, 大部分的时候静态工厂方法都能提供一个更加灵活的实现方式.

## Item 2: Consider a builder when faced with many constructor parameters.

静态工厂方法和构造函数都不擅长解决参数很多的情况. 当面对参数很多的对象时, 程序员一般会想到3种方法: `伸缩式的构造函数`, `JavaBeans模式`, `Builder模式`.

```java
//Telescoping constructor pattern - does not scale well
public class NutritionFacts {
	private final int servingSize;	// (mL)					required
	private final int servings;		// (per container)		required
	private final int calories;		// (per serving)		optional
	private final int fat;			// (g/serving)			optional
	private final int sodium;		// (mg/serving)			optional
	private final int carbohydrate;	// (g/sering)			optional
	
	public NutritionFacts(int servingSize, int servings) {
		this(servingSize, servings, 0);
	}
	
	public NutritionFacts(int servingSize, int servings, int calories) {
		this(servingSize, servings, calories, 0);
	}
	
	public NutritionFacts(int servingSize, int servings, int calories, int fat) {
		this(servingSize, servings, calories, fat, 0);
	}
	
	public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
		this(servingSize, servings, calories, fat, sodium, 0);
	}
	
	public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
		this.servingSize = servingSize;
		this.servings = servings;
		this.calories = calories;
		this.fat = fat;
		this.sodium = sodium;
		this.carbohydrate = carbohydrate;
	}
}
```

伸缩式的构造函数虽然可以解决部分参数过大的问题, 但是当这个参数变得更大之后, 就变得非常难掌控了, 我们必须小心确认传递的每一个值, 一旦出现细微的错误将导致对象的错误构造. 总而言之, 当参数变得很多的时候, 这时候的代码也变得非常复杂和难以阅读.

```java
//JavaBeans Pattern - allow inconsistency, mandates mutability
public class NutritionFacts {
	private final int servingSize = -1; //Required
	private final int servings = -1; 	//Required
	private final int calories = 0;
	private final int fat = 0;
	private final int sodium = 0;
	private final int carbohydrate = 0;
	
	public NutritionFacts() {}
	
	//Setters
	public void setServingSize(int val) { servingSize = val; }
	public void setServings(int val) { servings = val; }
	public void setCalories(int val) { calories = val; }
	public void setFat(int val) { fat = val; }
	public void setSodium(int val) { sodium = val; }
	public void setCarbohydrate(int val) { carbohydrate = val; }
}
```

当使用JavaBeans模式的时候, 通常这样调用方法:

```java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
...
```

构造的部分会被分成非常多行, 在这过程中对象是无法保证一致性的, 并且没有强制的方法保证一致性, 当错误的调用非一致性的对象时, 可能造成严重的后果(多线程访问). 这同样导致了JavaBean对象无法实现不变性, 需要额外的工作来保证线程安全性.

结合伸缩式的构造函数的安全性和JavaBeans模式的可读性, 这样便形成了一个新的模式: Builder模式. Builder模式不是直接创建对象, 而是通过Builder对象来设置各类属性, 最后通过调用build方法来构建一个对象.

```java
//Builder Pattern
public class NutritionFacts {
	private final int servingSize;
	private final int servings;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;
	
	public static class Builder {
		//Required parameters
		private final int servingSize;
		private final int servings;
		
		//Optional parameters initialized to default valueOf
		private int calories = 0;
		private int fat = 0;
		private int sodium  = 0;
		private int carbohydrate = 0;
		
		public Builder(int servingSize, int servings) {
			this.servingSize = servingSize;
			this.servings = servings;
		}
		
		public Builder calories(int val) {
			calories = val;
			return this;
		}
		public Builder fat(int val) {
			fat = val;
			return this;
		}
		public Builder sodium(int val) {
			sodium = val;
			return this;
		}
		public Builder carbohydrate(int val) {
			carbohydrate = val;
			return this;
		}
		public NutritionFacts build() {
			return new NutritionFacts(this);
		}
	}
	
	private NutritionFacts(Builder builder) {
		servingSize = builder.servingSize;
		servings = builder.servings;
		calories = builder.calories;
		fat = builder.fat;
		sodium = builder.sodium;
		carbohydrate = builder.carbohydrate;
	}
}
```

通过这个模式, NutritionFacts类是不可变的, 通过Builder的setter方法返回新的Builder的特性可以使用`fluent API`进行调用.

```java
NutritionFacts cocaCola = new NutritionFacts.builder(240, 8).calories(100).sodium(35).carbohydrate(27).build();
```

这样的代码是非常容易书写, 更重要的是非常容易阅读. 参数的校验这里并没有实现, 可以在Builder的构造函数里或者单个属性的设置方法内部或者build方法里进行校验都可以快速检查和实现的.

同时Builder模式是支持类继承的, 通过两个平行的类继承, 每个类对应自己的父类.

```java
//Builder pattern for class hierarchies
public abstract class Pizza {
	public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
	final Set<Topping> toppings;
	
	abstract static class Builder<T extends Builder<T>> {
		EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
		public T addTopping(Topping topping) {
			toppings.add(Objects.requireNonNull(topping));
			return self();
		}
		
		abstract Pizza build();
		
		// Subclasses must override this method to return "this"
		protected abstract T self();
	}
	
	Pizza(Builder<?> builder) {
		toppings = builder.toppings.clone();
	}
}
```

这里的Pizza.Builder是使用泛型, 同样添加了一个self方法, 允许方法链调用的时候, 可以很完美的应用于子类, 而不用进行转换.

```java
public class NyPizza extends Pizza {
	public enum Size { SMALL, MEDIUM, LARGE }
	private final Size size;
	
	public static class Builder extends Pizza.Builder<Builder>{
		private final Size size;
		
		public Builder(Size size) {
			this.size = Objects.requireNonNull(size);
		}
		
		@Override
		public MyPizza build() {
			return new NyPizza(this);
		}
		
		@Override
		protected Builder self() {
			return this;
		}
	}
	
	private NyPizza(Builder builder) {
		super(builder);
		size = builder.size;
	}
}

public class Calzone extends Pizza {
	private final boolean sauceInside;
	
	public static class Builder extends Pizza.Builder<Builder> {
		private boolean sauceInside = false; //Default
		
		public Builder sauceInside() {
			sauceInside = true;
			return this;
		}
		
		@Override 
		public Calzone build() {
			return new Calzone(this);
		}
		
		@Override
		protected Builder self() {
			return this;
		}
	}
	
	private Calzone(Builder builder){
		super(builder);
		sauceInside = builder.sauceInside;
	}
}
```

build被声明返回对应的子类, 如NyPizza.Builder返回NyPizza, 而Calzone.Builder返回Calzone. 这种技术, 子类的方法声明返回子类的类型, 被称作协变的返回类型. 而我们构造的方法如下:

```java
NyPizza pizza = new NyPizza.Builder(SMALL).addTopping(SAUSAGE).addTopping(ONION).build();
Calzone calzone = new Calzone.Builder().addTopping(HAM).sauceInside().build();
```

Builder模式也有自己的缺点, 为了创建一个对象, 就必须先创建一个该对象的Buidler对象, 可能在一般情况下没什么, 但是在追求极致性能的情况下, 可能觉得就有些不合适了. 同样Builder模式相对于伸缩式的构造函数来说, 也比较详尽和复杂, 一般需要权衡是否值得这么做. 一般都是超过4个或以上的参数时, 才推荐这么做.

总而言之, 当你设计一个拥有很多成员变量的类时(尤其是有些参数是可选的), Builder模式是一个很好的选择.

## Item 3: Enforce the singleton property with a private constructor or an enum type

单例模式, 一个类只被实例化一次. 单例模式往往代表一个状态不变的对象(如一个函数)或者一个独一无二的系统组件. 同时单例模式是很难测试的, 没办法模拟单例的实现(除非它实现了接口).

一般有两种实现方式, 都是基于私有化构造函数和提供一个静态的 公共的方法来访问实例化对象原理实现.

第一种公开其的静态实例化对象:

```java
//Singleton with public final field
public class Elvis {
	public static final Elvis INSTANCE = new private Elvis(){};
	...
}
```

当Elvis类初始化的时候, 该实例对象也就初始化好了. 通过初始化的时候, 使用public static final来定义实例, 保证该实例只会被初始化一次. 然后私有化构造函数, 保证该类无法继承而被独占.  除了一种情况: 通过反射中, 设置构造函数权限: AccessibleObect.setAccessible方法. 如果你需要防止这种攻击, 可以在构造函数中抛出一个异常来防止被第二次调用创建实例.

第二种方法则是提供一个静态的获取方法来获取实例对象:

```java
// Singleton with static factory
public class Elvis {
	private static final Elvis INSTANCE = new Elvis();
	private Elvis() {};
	
	public static Elvis getInstance() {return INSTANCE; }
	...
}
```

所有通过调用Elvis.getInstance返回同样的实例引用, 同样没有其他Elvis的实例会被创建.

使用第二种方法好处有很多: 可以给你选择, 是否需要实现单例(如果不需要, 你可以修改获取的方法, 而API不用修改), 或者可以用来创建一个通用的单例工厂, 并且使用静态工厂同样可以作为一个Supplier<Elvis>, 如: `Elvis::instance`. 但是如果上面这些特性都不需要的话, 那么第一种还是更好的选择.

但是如果要让一个单例实现序列化, 简单的实现Serializable接口是不够的 (因为你可以反序列化获取一个新的实例), 为了防止这个问题, 可以覆盖反序列化方法, 来返回当前的单例.

```java
//readResolve method to preserve singleton property
private Object readResolve() {
	//Return the one true Elvis and let the garbage collector
	//take care of Elvis impersonator
	return INSTANCE;
}
```

还有一个第三种方法来实现单例模式:

```java
public enum Elvis {
INSTANCE; 

...
}
```

通过枚举可以很简单的实现单例模式, 并且可以防止多实例化的攻击(序列化和反射). 虽然看起来有点奇怪, 但是这的确是最好的实现单例的方法. 有一个缺点就是, 那就是你无法进行继承(这时候可以考虑接口).

## Item 4: Enforce noninstantiability with a private constructor

在程序设计的时候, 我们偶尔想要书写一些类来包含静态的方法和静态属性, 来作为一个工具类被我们调用. 对于这种类, 是不需要实例化的, 但是我们往往不会(或者忘记)给它一个构造函数, 这就可能带来一个问题. JVM会给所有的类默认配置一个public的构造函数(如果你没显式声明的话). 这时候为了防止该工具类被实例化, 我们应该创建一个私有的构造函数来覆盖掉缺省的构造函数, 来让这个类不可实例化.

```java
//Noninstantiable utility class
public class UtilityClass {
	//Suppress default constructor for noninstantiability 
	private UtilityClass() {
		throw new AssertionError();
	}
	... // Remainder omitted
}
```

通过这样的方式, 我们可以保证这个类是不可实例化的. 同样这会带来一些负面的影响: 无法被继承(子类需要调用父类的构造函数). 但是对于工具类来说, 还是可以接受的.

## item 5: Prefer dependency injection to hardwiring resource

许多类都依赖一个或者多个底层的资源(Resources). 如下面的这个例子:

```java
//Inappropriate use of static utility - infelxible & untestable!
public class SpellChecker {
	private static final Lexicon dictionary = ...;
	
	private SpellChecker() {} // Noninstantiable
	
	public static boolean isValid(String word) { ... }
	public static List<String> suggestions(String typo) { ... }
}
```

SpellChecker依赖于Dictionary, 但是一般情况下设计成无实例对象和单例模式都是不恰当的: 当你需要更换依赖的资源时, 或者支持多个依赖的资源时, 这就变得非常难以进行修改和测试了. 静态的工具类和单例都是不恰当的方法来实现一个类, 类需要依赖其他的资源. 有一个简单的模式可以满足这种需求: 在构造函数中传递一份资源, 当你创建实例的时候. 这就是`依赖注入`的一种实例.

```java
public class SpellChecker {
	private final Lexicon dictionary;
	
	public SpellChecker(Lexicon dictionary) {
		this.dictionary = Objects.requireNonNull(dictionary);
	}
	
	public boolean isValid(String word) { ... }
	...
}
```

通过这种模式, 我们可以注入任何资源, 并同时保持了不变性. 虽然依赖注入可以很好的改善灵活性和测试性, 同样会带来一些副作用, 那就是当一个项目很大的时候, 这会使得项目代码非常杂乱. 这可以通过一些依赖注入框架来实现, 如`Dagger`,`Guice`,`Spring`等.

总而言之, 当你的类需要依赖别的资源的时候, 不要使用单例和静态工具类的方法来实现, 也不要直接创建一个资源对象, 而通过依赖注入, 传递一个对象进去, 将会显著改善灵活性, 重用性和测试性.

## Item 6: Avoid creating unnecessary objects.

有时候重用一些不变的对象, 而不是每次创建对象. 如:

```java
String s = new String("bikini");
```

当我们使用的时候, 这个语句每次都创建一个String对象. 如果这个语句出现在一个循环或者一个调用频率很高的方法内部, 就会有成千上万的String对象(不需要的)被创建. 改善的版本也很简单:

```java
String s = "bikini";
```

这个语句就这会创建一个String对象, 对于后面的调用都会调用该对象的引用, 而不是创建一个新的对象.

一般情况下, 我们创建对象的时候, 都应该使用静态工厂方法, 而不是构造函数. 对于某些不变的对象, 内部已经缓存好了很多预构造的对象, 通过构造方法可以避免重复构造这些对象. 如`Boolean.valueOf(String)`.

有一些对象的调用是非常昂贵的, 如果你需要重复使用这些昂贵的对象时, 推荐你把这个对象缓存下来, 避免重复调用. 不幸的是, 很多时候当你创建这些昂贵的对象的时候, 往往是不明显的, 或者容易忽视的.

如我们需要验证一个String是否符合是罗马数字:

```java
//Performance can be greatly improved
static boolean isRomanNumeral(String s) {
	return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
		+ "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

这看上去非常正常, 也是最简单的校验方法, 但是如果在调用次数很多的情况下, 就会带来性能上的负担. 因为方法内部创建了一个Pattern实例来校验这个值, 但是只使用这个Pattern实例一次.

为了改善性能, 我们可以提前缓存好该对象, 然后重复使用这个对象:

```java
//Reusing expensive object for improved performance
public class RomanNumberals {
	private static final Pattern ROMAN = Pattern.compile("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
	
	static boolean isRomanNumberal(String s) {
		return ROMAN.matcher(s).matches();
	}
}
```

另外一个可能创建不必要的对象的途径就是装箱(Autoboxing).

```java
private static long sum() {
	Long sum = 0L;
	for (lont i = 0; i <= Integer.MAX_VALUE; i++) 
		sum += i;
	
	return sum;
}
```

这个方法可以得到正确的计算结果, 但是会变得更慢. 只要我们将sum的定义从Long变成long, 那么程序将减少创建2^31个Long对象. 这是很明显的: 优先考虑原始数据类型而不是装箱类型, 并小心装箱类型的使用.

这不是说一定要避免创建对象, 对于一些小对象的创建和回收的代价是不高的, 有时候为了程序的可读性和简单些, 创建也是值得的. 但是创建一些不必要的对象只会影响性能和样式.

## Item 7: Eliminate obsolete object references

Java和别的语言最大的区别是JVM的内存管理, Java通过内存回收算法动态回收失效内存. 但这种形式也是存在自己的缺陷. 如:

```java
// Can you spot the "memory leak"?
public class Stack {
	private Object[] elements;
	private int size = 0;
	private static final int DEFAULT_INITIAL_CAPACITY = 16;
	
	public Stack() {
		elements = new Object[DEFAULT_INITIAL_CAPACITY];
	}
	
	public void push(Object e) {
		ensureCapacity();
		elements[size++] = e;
	}
	
	public Object pop() {
		if (size == 0)
			throw new EmptyStackException();
		return elements[--size];
	}
	/**
	* Ensure space for at least one more element, roughly
	* doubling the capacity each time the array needs to grow.
	*/
	private void ensureCapacity() {
		if (elements.length == size)
			elements = Arrays.copyOf(elements, 2 * size + 1);
	}
}
```

这个栈实现并没有任何问题. 但是这个程序却存在 `memeroy leak`: 内存泄漏. 当这个栈拓展的很大后收缩(pop), 移除的对象并不会被垃圾收集器收集掉, 因为elements内还保留着这些对象的强引用, 而这些引用的对象是无用的. 要解决这个问题是非常简单的: 对这些引用进行null清空.

```java
public Object pop() {
	if (size == 0)
		throw new EmptyStackException();
	Object result = elements[--size];
	elements[size] = null; // Eliminate obsolete reference  
	return result;
}
```

这不是说程序员需要对每一个不适用的对象的引用都进行null处理. 这是不必要的, 也没有必要成为规范的. 最好的消除引用的方法是让这个变量包含引用的超出scope而失效. 那我们什么时候该使用null清空引用呢? 正如上面这个例子所示的, 程序员知道size后的对象是不可用的, 但是垃圾收集器却不知道, 这就需要程序员和垃圾收集器进行合理的交流和沟通(null清除引用). 总的来说, 一个类应该管理它自己的内存, 程序员应该小心内存泄漏. 一旦一个element被释放, 任何该对象的引用都应该被赋值为null.

另外一个容易导致内存泄漏的就是缓存. 一旦你将一个对象引用放到缓存中去, 这是非常容易忘记的, 很快这个对象就会变得不可用. 那有很多解决方法, 将缓存中的对象放入WeakHashMap中, 如果外部应用的key变得失效之后, 这个对象也会被清除. 需要注意的是, WeakHashMap关注的是key而不是内容. 另一种方法就是定义生存时间, 定期清除. 在更复杂的情况下, 就应该直接使用java.lang.ref.

第三种容易出现内存泄漏的就是listeners和callbacks. 如果你实现的一个API支持客户端注册回调函数(callbacks), 但是没有显示删除, 它们会一直累加. 一种解决方法就是只存储weak references, 如把他们作为keys存储到WeakHashMap中.

内存泄漏往往不会有显式的问题, 但是可能一直存在与系统中. 只能通过代码审查或者调试工具(如`heap profiler`)进行查找. 这都是非常难以发现和找到的, 因此这很有必要提前学习一些预防知识来防止出现.

## Item 8: Avoid finalizers and cleaners

析构函数(Finalizers)是一个不可预测的, 危险的, 一般来说是没有必要的的函数. 使用这种函数会导致不确定的行为, 低的性能, 和不可移植性等问题. 在Java9中, 已经将析构函数标记为的deprecatd(弃用)的, 但是仍然被Java库中使用. Java9中的替代产品为Cleaners, 同样是不可预测的, 缓慢的和通常并不需要的. 析构函数的危险在于, 其的执行是放在后台一个低优先级的线程进行, 无法保证会被及时执行. 如果在执行过程中出现了异常, 并不会抛出异常, 并且队列中的对象将永远不会执行析构函数.

JVM不会对析构函数做出任何保证, 可以被正确运行. 不推荐在析构函数中做任何时间敏感的操作, 或者依靠析构函数来来释放一些共有的资源. 使用析构函数和Cleaner会带来严重的性能代价. 同样Finalize方法可以带来严重的安全问题, 当你为了防止类被实例化, 私有化构造函数, 并在内部抛出异常, 这时候可能会运行finalize()方法, 在其内部, 可以将本对象的实例的引用连接到一个静态变量下, 那这个对象将无法被垃圾收集器所收集. 而这个对象本不应该存在, 就可能导致严重的后果. 为了防止析构函数的攻击, 可以将析构函数声明为final, 并内容为空.

如果你不想要写析构函数或者Cleaner, 但是对象封装了资源(文件,线程等)需要关闭, 你可以实现AutoCloseable接口, 并重写close()方法,进行资源的关闭.

同样析构函数有两种合理的用法. 第一是为了安全的清除, 出于安全的考虑, 为了进行一些数据的安全, 资源的释放, 以免程序员忘记调用, 可以将这部分的关闭放到析构函数里. 但是使用之前, 需要权衡析构函数的不确定性和所付出的代价. 第二是用于回收一些本地方法的对象. 本地方法是不会被JVM所管理的, 它们的关闭可以通过析构函数来完成.

总而言之, 不要使用clearner, finalizer, 除了是出于安全原因考虑或者终结一些本地方法资源. 但是一定要清楚使用这两个方法的不确定性和所带来的性能代价.

## Item 9: Prefer try-with-resource to try-finally

try-with-resource 比 try-finally 更加效率, 可读性.

Java库中包含了很多资源必须调用close进行关闭. 如InputSream, OutputStream等等. 关闭资源却经常被用户所忘记, 这往往会导致严重的后果. 同时为了安全起见, 我们往往会将资源的关闭放到try-finally中去.

```java
//try-finally - No longer the best way to close resources
static String firstLineOfFile(String path) throws IOException {
	BufferedReader br = new BufferedReader(new FileReader(path));
	try {
		return br.readLine();
	} finally {
		br.close();
	}
}
```

但是当我们调用两份资源的时候, 会变成:

```java
//try-finally is ugly when used with more than one resource!
static String copy(String path, String dst) throws IOException {
	InputStream br = new FileInputStream(path);
	try {
		OutputStream out = new FileOutputStream(dst);
		try {
			byte[] buf = new byte[BUFFER_SIZE];
			int n;
			while (( n = in.read(buf)) >= 0)
				out.write(buf, 0, n);
		} finally {
			out.close();
		}
	} finally {
		br.close();
	}
}
```

这是非常丑陋的和不安全的. 在第一个函数中, 如果br.readLine()的时候由于物理的原因出现了异常1, 跳转到finally中执行关闭的时候, 又出现了异常2, 这时候异常2会覆盖掉异常1. 而我们调试的时候, 往往希望获得的是异常1的信息, 而不是异常2的信息.

这些问题最终在Java7中被解决了, Java 7提出了try-with-resource语句, 要使用这个结构, 那么资源就必须实现AutoCloseable接口. 如果你要书写一份资源类的话, 推荐你也实现这个接口, 进行关闭操作.

```java
//Try-with-resources - the the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
	try (BufferedReader br = new BufferedReader(new FileReader(path))) {
			return br.readLine();
	}
}

//Try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
	try (InputStream in = new FileInputStream(src);
		OutputStream out = new FileOutputStream(dst)) {
		byte[] buf = new byte[BUFFER_SIZE];
		int n;
		while ((n = in.read(buf)) >= 0)
			out.write(buf, 0, n);
	}
}
```

代码简单了很多, 也易读了很多. 如果一个异常出现在readLine部分和close部分, 那么前一个将会抑制后一个, 你看到的将是你想要看到的第一个. 同时第二个也没有被删除, 你可以通过getSuppressed方法获取详细的信息.

总而言之, 多用try-with-resource来替换try-finally, 带来更好的代码体验, 异常也将是你需要的.
