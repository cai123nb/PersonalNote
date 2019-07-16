# Java8 Lamba 学习

## Lambda的基本特点

+ 匿名: 没有明确的名称.
+ 函数: 像函数一样具有参数列表,函数主体,返回类型,还可能有可抛出的异常列表,但不属于某个特定的类.
+ 传递: 可以作为参数传递给方法或者存储在变量.
+ 简介: 可以书写很多模板

## 基本的样式

* (parameters) -> expression
* (parameters) -> { statements; }
  - 注意事项: 当使用第一种样式时如: () -> new Apple(10),编译器会自动检查返回类型,设置返回类型为Apple对象.如果你使用第二种样式时,一定要在每条语句后面添加分号;,并且需要显示的确定返回类型,如: () -> { return "Test";} ,显示的确定返回类型为String.

## 使用Lamba表达式的基本条件

### 函数式接口

+ 自定义一个抽象方法的接口.
- 如:

```java
public interface Runnable{
         void run();
}
public static void process(Runnable r){
         r.run();
}
Runnable r1 = () -> System.out.println("Hello World1");
process(r1);
process(() -> System.out.println("Hello World2"));
```

- 自定义的函数式接口,可以使用@FunctionalInterface接口,效果类型@override起到验证和提醒的作用.

## 常见的函数式接口

### Predicate

```java
@FunctionalInterface
public interface Predicate<T>(){
	boolean test(T t);
}

public static <T> List<T> filter<List<T> list, Predicate<T> p){
	List<T> results = new ArrayList<T>();
    for(T s: list){
    	if(p.test(s)){
        	results.add(s);
        }
    }
    return results;
}

Predicate<String> nonEmptyStringPredicate = (String s) -> !s.isEmpty();
List<String> nonEmpty = filter(listOfStrings,nonEmptyStringPredicate);
```

### Consumer

```java
@FunctionalInterface
public interface Comsumer<T>{
	void accept(T t);
}
public static <T> void forEach(List<T> list, Consumer<T> c){
	for(T i : list){
    	c.accept(i);
    }
}

forEach(
	Arrays.asList(1,2,3,4,5),
    (Integer i) -> System.out.println(i)
);
```

### Function

```java
@FunctionalInterface
public interface Function<T,R>{
	R apply(T t);
}

public static <T,R> List<R> map(List<T> list, Function<T,R> f){
	List<R> result = new ArrayList<>();
    for(T s: list){
    	result.add(f.apply(s));
	}
    return result;
}

List<Integer> ls = map(
	Arrays.asList("test","jake","Ok"),
    (String s) -> s.length()
);
```

#### 原始类型特化

- 以Predicate<T>为例,将泛型接口特化,避免装箱带来的格外开销.如:

```java
public interface IntPredicate{
	boolean test(int t);
}
IntPredicate evenNumbers = (int i) -> i%2 == 0;
evenNumbers.test(1000);	//无装箱
Predicate<Integer> oddNumbers = (Integer i) -> i % 2 == 1;
oddNumbers.test(1000);	//有装箱
```

- Predicate分为DoublePredicate,IntPredicate,LongPredicate

#### 抛出异常

* 直接抛出:

```java
@FunctionalInterface
public interface BufferedReaderProcessor {
	String process(BufferedReader b) throws IOException;
}
```

* 放在try catch模块中

```java
Function<BufferedReader,String> f = (BufferedReader b) -> {
	try {
    	return b.readLine();
    }
    catch(IOException e) {
    	throw new RuntimeException(e);
    }
}
```

### 类型检查

```
List<Apple> heavierThan150g = filter(inventory,(Apple a) -> a.getWeight() > 150);
```

1. 寻找filter方法的声明
2. 将Apple绑定到Predicate<T>中的泛型
3. 寻找Predicate函数式接口内的方法
4. 匹配参数和返回值

### 类型推断

* Java编译器会从上下文中推断出用什么函数式接口来配合Lambda表达式.
* 如:

```java
List<Apple> greenApples = filter(inventory, (Apple a) -> "green".equals(a.getColor()))
List<Apple> greenApples = filter(inventory, a->"green".equals(a.getColor()))
Comparator<Apple> c = (Apple a1,Apple a2) -> a1.getWeight.compareTo(a2.getWeight())
Comparator<Apple> c = (a1,a2) -> a1.getWeight().compareTo(a2.gettWeight())
```

### 使用局部变量

```java
int portNumber = 1337;
Runnable r = () -> System.out.println(portNumber);
```

Good, make no error!

```java
int portNumber = 1337;
Runnable r = () -> System.out.println(portNumber);
portNumber = 13337;
```

Error, local variable must be final or meaning final!

Lambda的闭包是对值闭包,不对变量闭包.原因:
* 局部变量存放在栈中,而Lambda在线程中访问其实是访问局部变量的一个复制,如果局部变量只赋值一次,没有影响.而常量存放在堆中,堆是所有线程共享的,访问并没有影响.

### 方法引用

* 如果一个Lambda代表的是"直接调用这个方法",那最好还是用名称来调用它而不是去描述如何调用它.
* eg:

```java
(Apple a) -> a.getWeight()	  Apple::getWeight()
() -> Thread.currentThread().dumpStack() Thread.currentThread()::dumpStack
(str,i) -> str.subtring(i)  String::substring
(String s) -> System.out.println(s) System.out::println
```

#### 构造函数引用

* 无构造参数

```java
Supplier<Apple> c1 = Apple:new;
Apple a1 = c1.get();
```

* 一参数构造

```java
Function<Integer,Apple> c2 = Apple:new;
Apple a2 = c2.apply(110);
```

* 两参数构造

```java
BiFunction<String,Integer,Apple> c3 = Apple:new;
Apple c3 = c3.apply("green",110);
```

### Lambda复合

#### 比较器复合

* 逆序

```java
inventory.sort(comparing(Apple:getWeight).reversed());
```

* 比较器链

```java
inventory.sort(comparing(Apple:getWeight)
	 .reversed()
         .thenComparing(Apple:getWeight);
```

#### 谓词复合

##### negate(取反),and,or

```java
Predicate<Apple> notRedApple = redApple.negate();
Predicate<Apple> redAndHeavyAppleOrGreen = redApple.and(a -> a.getWeight()>150).or(a -> "green".equals(a.getColor()));
```

#### 函数复合

##### andThen,compose

```java
Function<Integer,Integer> f = x -> x+1;
Function<Integer,Integer> g = x -> x*2;
Function<Integer,Integer> h = f.andThen(g);
int result = h.apply(1);  //4
Function<Integer,Integer> e = f.compose(g);
int result2 = e.compose(g);	//3
```

