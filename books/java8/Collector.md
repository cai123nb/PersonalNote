### 收集器简介
Collect,终端操作,一种归约操作,将流里面的所有元素进行转换操作,累积成一个汇总结果. 通过定义一个Collector接口传递给Collect来实现的.如:

```java
List<Transaction> transactions = transactionStream.collect(Collectors.toList());
//toList源代码:
public static <T> Collector<T, ?, List<T>> toList() {
    return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, 
    							List::add,
                                (left, right) -> { left.addAll(right); 
                                return left; },CH_ID);
}
```

### 预定义的归约和汇总
##### couting

```java
long howManyDishes = menu.stream().collect(Collectors.counting());
// long howManyDishes = menu.stream().count();
```

##### maxBy minBy

```java
Comparator<Dish> dishCaloriesComparator = Comparator.comparingInt(Dish::getCalories);
Optional<Dish> mostCalorieDish = menu.stream()
	.collect(maxBy(dishCaloriesComparator));
```

##### summingInt summingLong .../averagingInt .../summarizingInt ...

```java
int totalCalories = menu.stream().collect(summingInt(Dish::getCalories));
double avgCalories = menu.stream().collect(averagingInt(Dish::getCalories));
IntSummaryStatistics avgCalories =
	menu.stream().collect(summarizingInt(Dish::getCalories));
avgCalories{count=9,sum=4300,min=120,average=477.77,max=800};
```

##### joining

```java
String shortMenu = menu.stream().map(Dish:getName).collect(joining(", "));
```

### 广义的归约汇总
之前讨论的预定义收集器,都是可以使用reducing工厂方法来实现,可以看做是reducing工厂方法定义的归约过程的特殊情况.如:

```java
int totalCalories = menu.stream()
	.collect(reducing(
    0, Dish::getCalories, (i,j) -> i+j
    ));
```

第一个参数: 归约操作的起始值,如果省略,返回的将是Optional<T>类似参数
第二个参数: Function<T,R>转换函数
第三个参数: BinaryOperator<T,T,T>
需要与Stream接口中的reduce方法区分开来
Stream.reduce 提供了三种类型接口

```java
//First
T reduce(T identity, BinaryOperator<T> accumulator);
// int sum = numbers.stream().reduce(0,Integer::sum);

//second
Optional<T> reduce(BinaryOperator<T> accumulator);
// Optional<Integer> min = numbers.steam().reduce(Integer::min);

//third
<U> U reduce(U identity,
             BiFunction<U, ? super T, U> accumulator,
             BinaryOperator<U> combiner);
//使用reduce函数实现,Collector.toList
Stream<Integer> stream = Arrays.asList(1,3,4,6,5).stream();
        List<Integer> numbers = stream.reduce(new ArrayList<Integer>(),
                (List<Integer> l, Integer e) -> {
                l.add(e);
                return l; },
                (List<Integer> l1,List<Integer> l2) -> {
                l1.addAll(l2);
                return l1;});
```

区别:
Sream.reduce: 目标是将两个值结合在一起生成一个新值,是一个不可变的归约过程.在多线程操作过程中,存在着较多弊端.
Collect.reducing: 修改容器来实现累加过程,在多线程过程中更加擅长.

### 分组
使用Collectors.groupingBy工厂方法

```java
public enum CaloricLevel {DIET, NORMAL, FAT};
Map<CaloricLevel,List<Dish>) dishesByCaloricLevel = menu.stream()
	.collect(
    groupingBy(dish -> {
    	if(dish.getCalories() <= 400) 
        	return CaloricLevel.DIET;
        else if(dish.getCalories() <= 700) 
        	return CaloricLevel.NORMAL;
        else 
        	return CaloricLevel.FAT;
    })
    );
```

##### 多级分组

```java
Map<Dish.Type,Map<CaloricLevel,List<Dish>>> dishesByTypeCaloricLevel = 
        menu.stream().collect(
                groupingBy(Dish::getType,
                        groupingBy(dish -> {
                            if(dish.getCalories() <= 400)
                                return CaloricLevel.DIET;
                            else if(dish.getCalories() <= 700)
                                return CaloricLevel.NORMAL;
                            else
                                return CaloricLevel.FAT;
                        }))
        );
```

##### 利用子组进行数据收集
由于groupingBy第二种构造函数的第二个参数允许不同的收集器,我们可以通过传入不同的收集器实现不同的功能,如:

```java
Map<Dish,Long> typesCount = menu.stream()
	.collect(groupingBy(Dish::getType,counting()));
Map<Dish.Type,Optional<Dish>> mostCaloricByType = menu.stream()
	.collect(groupingBy(Dish::getType,maxBy(comparingInt(Dish::getCalories))));
```

+ 将收集器的结果转换为另一种类型,利用Collectors.collectingAndThen工厂方法进行加工处理

```java
Map<Dish.Type,Dish> mostCaloricByType = menu.stream()
        .collect(groupingBy(Dish::getType,collectingAndThen(maxBy(
                Comparator.comparingInt(Dish::getCalories)),Optional::get)));
```

源代码解释: Adapts a {@code Collector} to perform an additional finishing transformation. 

```java
public static<T,A,R,RR> Collector<T,A,RR> collectingAndThen(Collector<T,A,R>   downstream, Function<R,RR> finisher){}
```

接受一个容器,并对该收集器进行Function操作进行装换,最后返回一个收集器. 相当于容器的包装
+ 另一个常用的处理器就是mapping,接受一个函数对流进行转换,然后将转换的结果收集起来.Adapts a {@code Collector} accepting elements of type {@code U} to oneaccepting elements of type {@code T} by applying a mapping function to each input element before accumulation.

```java
public static <T, U, A, R> Collector<T, ?, R> mapping(Function<? super T, ? extends U> mapper,Collector<? super U, A, R> downstream) {}
```

使用示例:

```java
Map<Dish.Type,Set<CaloricLevel>> caloricLevelByType = menu.stream()
        .collect(
                groupingBy(Dish::getType,mapping(dish ->{
                    if(dish.getCalories() <= 400)
                        return CaloricLevel.DIET;
                    else if(dish.getCalories() <= 700)
                        return CaloricLevel.NORMAL;
                    else
                        return CaloricLevel.FAT;
                },toSet() ))
        );
Map<Dish.Type,Set<CaloricLevel>> caloricLevelByType2 = menu.stream()
        .collect(
                groupingBy(Dish::getType,mapping(dish ->{
                    if(dish.getCalories() <= 400)
                        return CaloricLevel.DIET;
                    else if(dish.getCalories() <= 700)
                        return CaloricLevel.NORMAL;
                    else
                        return CaloricLevel.FAT;
                },toCollection(HashSet::new) ))
        );
```

### 分区
+ 分区时分组的特殊情况, 由一个谓词(返回一个布尔值得函数) 作为分类函数. 如:

```java
Map<Boolean, List<Dish>> partitionedMenu = menu.stream()
	.collect(partitioningBy(Dish::isVegetarian));
//返回类型
{
	false = [pork, beef, chinken, prawns, salmon],
    true = [french, rice, season fruit, pizza]
}
```

+ 质数分离实例

```java
public boolean isPrime(int candidate){
	int candidateRoot = (int) Math.sqrt((double) candidate);
    return IntStream.rangeClosed(2, candidateRoot)
    	.noneMatch(i -> candidate % i == 0);
}

public Map<Boolean, List<Integer>> partitionPrimes(int n){
	return IntStream.rangeClosed(2,n).boxed()
    	.partitioningBy(candidate -> isPrime(candidate)));
}
```

### Collectors类静态工厂方法列表:
+ toList,返回List<T>,将流中的项目收集到一个List上

```java
List<Dish> dishes = menuStream.collect(toList());
```

+ toSet,返回Set<T>,将流中项目收集到Set上,删除重复项

```java
List<Dish> dishes = menuStream.collect(toSet());
```

+ toCollection,返回Collections<T>,把流中所有的项目收集到给定的供应源创建的的集合

```java
Collection<Dish> dishes = menuStream.collect(toCollection,ArrayList::new);
```

+ counting,返回Long,计算流中元素的个数

```java
long howMantDishes = menuStream.collect(counting());
```

+ summingInt,返回Long,对流中项目的一个整数属性求和

```java
int totalCalories = menuStream.collect(summingInt(Dish::getCalories));
```

+ averagingInt,返回Double,计算流中项目Integer属性的平均值

```java
double avgCalories = menustram.collect(averagingInt(Dish::getCalories));
```

+ summarizingInt,返回IntSummaryStatistics,收集流中项目Integer属性的统计值,例如最大,最小,总和与平均值.

```java
IntSummaryStatistics menuStatistics = menuStream.collect(summarizingInt(Dish::getCalories));
```

+ joining,返回String,连接对流中每个项目调用toString方法生成的字符串

```java
String shortMenu = menuStream.map(Dish::getName).collect(joining(", "));
```

+ maxBy,返回Optional<T>,选出流中按照比较器选出的最大元素Optional,如果流为空则为Optional.empty().

```java
Optional<Dish> fattest = menuStream.collect(maxBy(comparingInt(Dish::getCalories)));
```

+ minBy,返回Optional<T>,同理
+ reducing,返回归约操作产生的类型,从一个作为累加器的初始值开始,利用BinaryOperator与流中的元素逐个结合,从而将流归约为单个值.

```java
int totalCalories = menuStream.collect(reducing(0,Dish::getCalories,Integer::sum));
```
+ collectingAndThen,转换函数返回的类型,包裹另一个收集器,对其结果应用转换函数

```java
int howManyDishes = menuStream.collect(reducing(0,Dish::getCalories, Integer::sum));
```
+ groupingBy,返回Map<K,List<T>>,根据一个项目的属性额值对流中的项目作分组,并将属性值作为结果Map的键

```java
Map<Dish.Type,List<Dish>> dishesByType = menuStream.collect(groupingBy(Dish::getType));
```
+ partitioningBy, 返回Map<Boolean,List<T>>,根据流中每个项目应用对应谓词的结果来对项目进行分区

```java
Map<Booean,List<Dish>> vegetarianDishes = menuStream.collect(partitioningBy(Dish::isVegetarian));w
```

### 收集器接口
Java中源代码:

```java
public interface Collector<T, A, R> {
    /**
     * A function that creates and returns a new mutable result container.
     *
     * @return a function which returns a new, mutable result container
     */
    Supplier<A> supplier();

    /**
     * A function that folds a value into a mutable result container.
     *
     * @return a function which folds a value into a mutable result container
     */
    BiConsumer<A, T> accumulator();

    /**
     * A function that accepts two partial results and merges them.  The
     * combiner function may fold state from one argument into the other and
     * return that, or may return a new result container.
     *
     * @return a function which combines two partial results into a combined
     * result
     */
    BinaryOperator<A> combiner();

    /**
     * Perform the final transformation from the intermediate accumulation type
     * {@code A} to the final result type {@code R}.
     *
     * <p>If the characteristic {@code IDENTITY_TRANSFORM} is
     * set, this function may be presumed to be an identity transform with an
     * unchecked cast from {@code A} to {@code R}.
     *
     * @return a function which transforms the intermediate result to the final
     * result
     */
    Function<A, R> finisher();

    /**
     * Returns a {@code Set} of {@code Collector.Characteristics} indicating
     * the characteristics of this Collector.  This set should be immutable.
     *
     * @return an immutable set of collector characteristics
     */
    Set<Characteristics> characteristics();
    ...
```

+ 简单来看

```java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    BinaryOperator<A> combiner();
    Function<A, R> finisher();
    Set<Characteristics> characteristics();
}
```

+ T: 我们使用的泛型类型,A累加器,对流中数据进行累加操作,R: 返回数据类型.
+ supplier(): 返回新的结果容器
+ accumulator(): 将元素添加到容器中去,即: A ---> T
+ finisher(): 对结果容器应用最终转换. A --> R
+ combiner(): 合并两个结果容器,定义不同子部分归约所得累加器如何进行合并, T1 + T2 --> T...
+ characteristics(): 返回一个不可变的Characteristics集合,Characteristics定义:

```java
enum Characteristics {
    /**
     * Indicates that this collector is <em>concurrent</em>, meaning that
     * the result container can support the accumulator function being
     * called concurrently with the same result container from multiple
     * threads.
     *
     * <p>If a {@code CONCURRENT} collector is not also {@code UNORDERED},
     * then it should only be evaluated concurrently if applied to an
     * unordered data source.
     */
    CONCURRENT,

    /**
     * Indicates that the collection operation does not commit to preserving
     * the encounter order of input elements.  (This might be true if the
     * result container has no intrinsic order, such as a {@link Set}.)
     */
    UNORDERED,

    /**
     * Indicates that the finisher function is the identity function and
     * can be elided.  If set, it must be the case that an unchecked cast
     * from A to R will succeed.
     */
    IDENTITY_FINISH
}
```

+ 简单的例子:

```java
public class MyCollectors<T> implements Collector<T, List<T>, List<T>> {

    /**
     * 产生一个累加器类型,用于对流中元素进行累加操作
     *
     * @return
     */
    @Override
    public Supplier<List<T>> supplier() {
        return ArrayList::new;
    }

    @Override
    public BiConsumer<List<T>, T> accumulator() {
        return List::add;
    }

    @Override
    public BinaryOperator<List<T>> combiner() {
        return (list1, list2) -> {
            list1.addAll(list2);
            return list1;
        };
    }

    @Override
    public Function<List<T>, List<T>> finisher() {
        return Function.identity();
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of(IDENTITY_FINISH, CONCURRENT));
    }
}
```

### 开发自己的收集器
#### 之前利用分区区分质数获得Map:

```java
public static Map<Boolean, List<Integer>> partitionPrimes(int n) {
        return IntStream.rangeClosed(2, n).boxed()
                .collect(partitioningBy(candidate -> isPrime(candidate)));
    }

public static boolean isPrime(int candidate) {
    return IntStream.rangeClosed(2, candidate-1)
            .limit((long) Math.floor(Math.sqrt((double) candidate)) - 1)
            .noneMatch(i -> candidate % i == 0);
}
```
#### 同样我们可以利用收集器实现我们的想法

```java
package MyCollectors;

import java.util.*;
import java.util.function.*;
import java.util.stream.Collector;
import java.util.stream.Collector.Characteristics.*;

import static java.util.stream.Collector.Characteristics.IDENTITY_FINISH;

/**
 * Created by cjyong on 2017/8/18.
 */
public class PrimeNumbersCollector implements Collector<Integer, Map<Boolean,List<Integer>>,Map<Boolean, List<Integer>>> {
    @Override
    public Supplier<Map<Boolean, List<Integer>>> supplier() {
        return () -> new HashMap<Boolean, List<Integer>>() {{
            put(true, new ArrayList<Integer>());
            put(false, new ArrayList<Integer>());
        }};
    }

    @Override
    public BiConsumer<Map<Boolean, List<Integer>>, Integer> accumulator() {
        return (Map<Boolean, List<Integer>> acc, Integer candidate) -> {
            acc.get( isPrime( acc.get(true),
                    candidate) )
                    .add(candidate);
        };
    }

    @Override
    public BinaryOperator<Map<Boolean, List<Integer>>> combiner() {
        return (Map<Boolean, List<Integer>> map1, Map<Boolean, List<Integer>> map2) -> {
            map1.get(true).addAll(map2.get(true));
            map1.get(false).addAll(map2.get(false));
            return map1;
        };
    }

    @Override
    public Function<Map<Boolean, List<Integer>>, Map<Boolean, List<Integer>>> finisher() {
        return i -> i;
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of(IDENTITY_FINISH));
    }

    public static boolean isPrime(List<Integer> primes, int candidate){
        int candidateRoot = (int) Math.sqrt((double) candidate);
        return takeWhile(primes, i -> i<= candidateRoot)
                .stream()
                .noneMatch(p -> candidate % p == 0);
    }

    public static <T> List<T> takeWhile(List<T> list, Predicate<T> p){
        int i = 0;
        for( T item : list){
            if(!p.test(item)) {
                return list.subList(0,i);
            }
            i++;
        }
        return list;
    }
}

```
#### 性能比较:

```java
long fastest = Long.MAX_VALUE;
for(int i = 0; i< 10; i++){
    long start = System.nanoTime();
    partitionPrimes(1_000_000);
    long duration = (System.nanoTime() - start);
    if(duration < fastest) fastest = duration;
}
System.out.println("Fastest execution done in " + fastest + "msecs");

long fastest2 = Long.MAX_VALUE;
for(int i = 0; i< 10; i++){
    long start = System.nanoTime();
    Stream.iterate(2, a -> a + 1)
            .limit(1_000_000)
            .collect(new PrimeNumbersCollector());
    long duration = (System.nanoTime() - start);
    if(duration < fastest2) fastest2 = duration;
}
System.out.println("Fastest execution done in " + fastest2 + "msecs");
```

![结果比较](https://image.cjyong.com/blog/bj8_1.jpg)

+ 可以看出我们自己的收集器具有更加良好的性能
#### 匿名收集器写法:

```java
public Map<Boolean, List<Integer>> partitionPrimesWithInlineCollector(int n) {
        return Stream.iterate(2, i -> i + 1).limit(n)
                .collect(
                        () -> new HashMap<Boolean, List<Integer>>() {{
                            put(true, new ArrayList<Integer>());
                            put(false, new ArrayList<Integer>());
                        }},
                        (acc, candidate) -> {
                            acc.get( isPrime(acc.get(true), candidate) )
                                    .add(candidate);
                        },
                        (map1, map2) -> {
                            map1.get(true).addAll(map2.get(true));
                            map1.get(false).addAll(map2.get(false));
                        });
    }
```