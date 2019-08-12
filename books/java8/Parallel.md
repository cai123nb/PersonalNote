# 高效使用并行化

示例代码:

```java
import java.util.function.Function;
import java.util.stream.LongStream;
import java.util.stream.Stream;

/**
 * Created by cjyong on 2017/8/22.
 */
public class Main7 {
    public static void main(String ... args){
        System.out.println("Parallel sum done in:" + measureSumPerf(Main7::parallelSum, 10_000_000) + " msecs");
        System.out.println("Sequential sum done in:" + measureSumPerf(Main7::sequentiveSum, 10_000_000) + " msecs");
        System.out.println("Iterator sum done in:" + measureSumPerf(Main7::iterativeSum, 10_000_000) + " msecs");
        System.out.println("Random sum done in:" + measureSumPerf(Main7::randomSum, 10_000_000) + " msecs");
        System.out.println("Parallel Random sum done in:" + measureSumPerf(Main7::parallelRandomSum, 10_000_000) + " msecs");
    }

    public static long parallelSum(long n){
        return Stream.iterate(1L, i -> i + 1)
                .limit(n)
                .parallel()
                .reduce(0L, Long::sum);
    }

    public static long sequentiveSum(long n){
        return Stream.iterate(1L, i -> i + 1)
                .limit(n)
                .reduce(0L, Long::sum);
    }

    public static long iterativeSum(long n){
        long result = 0;
        for (long i = 1L; i <= n;i++){
            result += i;
        }
        return result;
    }

    public static long randomSum(long n){
        return LongStream.rangeClosed(1,n)
                .reduce(0L, Long::sum);
    }

    public static long parallelRandomSum(long n){
        return LongStream.rangeClosed(1,n)
                .parallel()
                .reduce(0L, Long::sum);
    }

    public static long measureSumPerf(Function<Long, Long> adder, long n){
        long fastest = Long.MAX_VALUE;
        for(int i = 0; i< 10; i++){
            long start = System.nanoTime();
            long sum = adder.apply(n);
            long duration = (System.nanoTime() - start) / 1_000_000;
            if(duration < fastest) fastest = duration;
        }
        return fastest;
    }

}
```

+ 结果示意图:
  ![结果比较](https://image.cjyong.com/blog/bj8_2.jpg)
+ 小结:
  - 并行流不总是比顺序流块, 有时候和你的直觉差别很大,最好的方法就是寻找合适的基准进行校验.
  - 留意装箱带来的额性能损耗, 自动装箱和拆箱会大大降低性能, 尽量使用原始类型流: IntSteam, LongSteam, DoubleStream来避免这些操作.
  - 有些操作本身就不适合并行计算, 如: limit, findFirst.
  - 考虑流操作的流水线计算总成本, 设N是要处理的元素的总数, Q是一个元素通过流水线所需要的大致成本, 则N*Q就是这个对成本的粗略定性估计. Q值较高意味着使用并行流性能好的可能性较大.
  - 对于较小数据量, 不推荐并行流.
  - 考虑流背后的数据结构是否易于分解. 如: ArrayList拆分效率比LinkedList高很多.
  - 流本身的中间操作修改流的方式, 都可能会改变分解过程的性能.
  - 还要考虑终端操作中合并步骤的代价是否大还是小.
  - 并行流背后的使用的基础就是Java7中引入的分支/合并框架.
  - 流的数据源可分解性:
    + ArrayList: 极佳
    + LinkedList: 差
    + IntSteam.range: 极佳
    + Steam.iterate: 差
    + HashSet: 好
    + TreeSet: 好

## 分支合并框架

+ 分支合并框架的目的是以递归的方式将并行的任务拆分成更小的任务, 然后将每个子任务的结果合并生成整体结果.
+ 简单示例:

```java
import java.util.concurrent.RecursiveTask;
/**
 * Created by cjyong on 2017/8/22.
 */
public class ForkJoinSumCalculator extends RecursiveTask<Long>{

    private final long[] numbers;
    private final int start;
    private final int end;
    public static final long THRESHOLD = 10_000;

    public ForkJoinSumCalculator(long[] numbers){
        this(numbers, 0, numbers.length);
    }

    private ForkJoinSumCalculator(long[] numbers, int start, int end){
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if(length <= THRESHOLD){
            return computeSequentially();
        }
        ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers, start, start + length/2);
        leftTask.fork();    //使用另一个ForkJoinPool线程异步执行新创建的子任务
        ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(numbers, start + length/2, end);
        Long rightResult = rightTask.compute();
        Long leftResult = leftTask.compute();

        return rightResult + leftResult;
    }

    private long computeSequentially(){
        long sum = 0;
        for(int i = start; i< end; i++){
            sum += numbers[i];
        }
        return sum;
    }
}
//调用
    public static long forkJoinSum(long n){
        long[] numbers = LongStream.rangeClosed(1, n).toArray();
        ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
        return new ForkJoinPool().invoke(task);
    }
```

+ 注意事项:
  - 对一个任务调用join方法会堵塞调用方, 直到该任务做出结果. 有必要在两个子任务的计算都开始之前再调用它. 否则你得到的版本会比原始的顺序算法更加慢和复杂.
  - 不应该在RecursiveTask内部使用ForkJoinPool的invoke方法, 相反始终调用compute或者fork方法, 只有顺序代码才应该使用invoke来启动并行.
  - 同时对左边, 右边子任务调用fork方法, 看似合理, 但是效率会比只对一个任务调用更加的低, 这样可以为一个子任务重用同一线程.
  - 调试使用分支合并框架的并行计算有点棘手, 无法再stack trace中进行动态调试.
  - 和并行流一样, 我们不能理所当然的任务分支合并一定比顺序执行要快. 有时候合理的分解任务可以带来良好的性能, 此外分支合并框架需要"预热", 执行多遍之后才会被JIT编译器所优化.
+ 分支合并法使用"工作窃取"的方法来保证所有子任务完成时间大体相同: 任务拆分之后放在对应线程的任务队列中, 在队列中这些任务使用双向链式进行连接, 每当一个任务完成, 该线程从队列头取出下一个任务开始执行. 由于某些原因(CPU划分效率, 磁盘读取速度等), 某些线程早早完成了所分配的所有任务, 队列已经清空, 而其他线程还在忙. 这时候, 这个线程随机选取一个别的线程, 从该线程的队列尾部"偷走"一个任务进行执行, 直到所有的子任务完成. 所以我们分解任务时, 不应该只是划分几个大任务, 而是划分成多个小任务, 更加有助于工作线程之间的负载平衡.

## Spliterator

+ Spliterator 是Java8中新添加的接口, 用于并行执行而设计的迭代器.
+ 简单示例:

```java
package MyIterator;
/**
 * Created by cjyong on 2017/8/22.
 */
public class WordCounter {

    private final int counter;
    private final boolean lastSpace;

    public WordCounter(int counter, boolean lastSpace) {
        this.counter = counter;
        this.lastSpace = lastSpace;
    }

    public WordCounter accumulate(Character c) {
        if (Character.isWhitespace(c)) {
            return lastSpace ? this : new WordCounter(counter, true);
        } else {
            return lastSpace ? new WordCounter(counter+1, false) : this;
        }
    }

    public WordCounter combine(WordCounter wordCounter) {
        return new WordCounter(counter + wordCounter.counter, wordCounter.lastSpace);
    }

    public int getCounter() {
        return counter;
    }
}
```

```java
package MyIterator;
import java.util.Spliterator;
import java.util.function.Consumer;
/**
 * Created by cjyong on 2017/8/22.
 */
public class WordCounterSpliterator implements Spliterator<Character> {

    private final String string;
    private int currentChar = 0;

    public WordCounterSpliterator(String string) {
        this.string = string;
    }
	//功能类似传统的Iterator, 遍历所有的元素, 如果还有元素需要遍历返回true, 不然返回false
    @Override
    public boolean tryAdvance(Consumer<? super Character> action) {
        action.accept(string.charAt(currentChar++));
        return currentChar < string.length();
    }
    //将一些元素划分出去给第二个Spliterator, 与其进行并行处理
    @Override
    public Spliterator<Character> trySplit() {
        int currentSize = string.length() - currentChar;
        if(currentSize < 10){ //返回null表示要解析的String已经够小了,可以顺序处理
            return null;
        }
        for(int splitPos = currentSize / 2 + currentChar; splitPos < string.length(); splitPos++){
            if(Character.isWhitespace(string.charAt(splitPos))){
                Spliterator<Character> spliterator = new WordCounterSpliterator(string.substring(currentChar, splitPos));
                currentChar = splitPos;
                return spliterator;
            }
        }
        return null;
    }
    //还需要遍历的元素个数
    @Override
    public long estimateSize() {
        return string.length() - currentChar;
    }
    //设置遍历器的特征, 详情查看源代码
    @Override
    public int characteristics() {
        return ORDERED + SIZED + SUBSIZED + NONNULL + IMMUTABLE;
    }
}
```

```java
import MyIterator.WordCounter;
import MyIterator.WordCounterSpliterator;
import java.util.Spliterator;
import java.util.Spliterators;
import java.util.stream.IntStream;
import java.util.stream.Stream;
import java.util.stream.StreamSupport;
/**
 * Created by cjyong on 2017/8/22.
 */
public class Main7_3 {

    static final String SENTENCE = " Nel   mezzo del cammin di nostra vita "+
            "mi ritrovai in una  selva oscura" +
            " che la dritta via era smarrita ";

    public static void main(String ... args){
        System.out.println("Found " + countWordsIteratively(SENTENCE) + " words");
        Stream<Character> stream = IntStream.rangeClosed(0, SENTENCE.length()-1)
                .mapToObj(SENTENCE::charAt);
        System.out.println("Found " + countWords(stream.parallel()) + " words");
        System.out.println("Found " + countWords(SENTENCE) + " words");
    }

    private static int countWords(Stream<Character> stream){
        WordCounter wordCounter = stream.reduce(new WordCounter(0, true),
                WordCounter::accumulate,
                WordCounter::combine);
        return wordCounter.getCounter();
    }

    private static int countWords(String s){
        Spliterator<Character> spliterator = new WordCounterSpliterator(s);
        Stream<Character> stream = StreamSupport.stream(spliterator, true);

        return countWords(stream);
    }

    public  static int countWordsIteratively(String s){
        int counter = 0;
        boolean lastSpace = true;
        for(char c : s.toCharArray()){
            if(Character.isWhitespace(c)){
                lastSpace = true;
            } else {
                if(lastSpace) counter++;
                lastSpace = false;
            }
        }
        return counter;
    }
}
```

```java
//Spliterator接口源码
package java.util;
import java.util.function.Consumer;
import java.util.function.DoubleConsumer;
import java.util.function.IntConsumer;
import java.util.function.LongConsumer;
/*
 *
 * <p><b>Example.</b> Here is a class (not a very useful one, except
 * for illustration) that maintains an array in which the actual data
 * are held in even locations, and unrelated tag data are held in odd
 * locations. Its Spliterator ignores the tags.
 *
 * <pre> {@code
 * class TaggedArray<T> {
 *   private final Object[] elements; // immutable after construction
 *   TaggedArray(T[] data, Object[] tags) {
 *     int size = data.length;
 *     if (tags.length != size) throw new IllegalArgumentException();
 *     this.elements = new Object[2 * size];
 *     for (int i = 0, j = 0; i < size; ++i) {
 *       elements[j++] = data[i];
 *       elements[j++] = tags[i];
 *     }
 *   }
 *
 *   public Spliterator<T> spliterator() {
 *     return new TaggedArraySpliterator<>(elements, 0, elements.length);
 *   }
 *
 *   static class TaggedArraySpliterator<T> implements Spliterator<T> {
 *     private final Object[] array;
 *     private int origin; // current index, advanced on split or traversal
 *     private final int fence; // one past the greatest index
 *
 *     TaggedArraySpliterator(Object[] array, int origin, int fence) {
 *       this.array = array; this.origin = origin; this.fence = fence;
 *     }
 *
 *     public void forEachRemaining(Consumer<? super T> action) {
 *       for (; origin < fence; origin += 2)
 *         action.accept((T) array[origin]);
 *     }
 *
 *     public boolean tryAdvance(Consumer<? super T> action) {
 *       if (origin < fence) {
 *         action.accept((T) array[origin]);
 *         origin += 2;
 *         return true;
 *       }
 *       else // cannot advance
 *         return false;
 *     }
 *
 *     public Spliterator<T> trySplit() {
 *       int lo = origin; // divide range in half
 *       int mid = ((lo + fence) >>> 1) & ~1; // force midpoint to be even
 *       if (lo < mid) { // split out left half
 *         origin = mid; // reset this Spliterator's origin
 *         return new TaggedArraySpliterator<>(array, lo, mid);
 *       }
 *       else       // too small to split
 *         return null;
 *     }
 *
 *     public long estimateSize() {
 *       return (long)((fence - origin) / 2);
 *     }
 *
 *     public int characteristics() {
 *       return ORDERED | SIZED | IMMUTABLE | SUBSIZED;
 *     }
 *   }
 * }}</pre>
 *
 * <p>As an example how a parallel computation framework, such as the
 * {@code java.util.stream} package, would use Spliterator in a parallel
 * computation, here is one way to implement an associated parallel forEach,
 * that illustrates the primary usage idiom of splitting off subtasks until
 * the estimated amount of work is small enough to perform
 * sequentially. Here we assume that the order of processing across
 * subtasks doesn't matter; different (forked) tasks may further split
 * and process elements concurrently in undetermined order.  This
 * example uses a {@link java.util.concurrent.CountedCompleter};
 * similar usages apply to other parallel task constructions.
 *
 * <pre>{@code
 * static <T> void parEach(TaggedArray<T> a, Consumer<T> action) {
 *   Spliterator<T> s = a.spliterator();
 *   long targetBatchSize = s.estimateSize() / (ForkJoinPool.getCommonPoolParallelism() * 8);
 *   new ParEach(null, s, action, targetBatchSize).invoke();
 * }
 *
 * static class ParEach<T> extends CountedCompleter<Void> {
 *   final Spliterator<T> spliterator;
 *   final Consumer<T> action;
 *   final long targetBatchSize;
 *
 *   ParEach(ParEach<T> parent, Spliterator<T> spliterator,
 *           Consumer<T> action, long targetBatchSize) {
 *     super(parent);
 *     this.spliterator = spliterator; this.action = action;
 *     this.targetBatchSize = targetBatchSize;
 *   }
 *
 *   public void compute() {
 *     Spliterator<T> sub;
 *     while (spliterator.estimateSize() > targetBatchSize &&
 *            (sub = spliterator.trySplit()) != null) {
 *       addToPendingCount(1);
 *       new ParEach<>(this, sub, action, targetBatchSize).fork();
 *     }
 *     spliterator.forEachRemaining(action);
 *     propagateCompletion();
 *   }
 * }}</pre>
 *
 * @implNote
 * If the boolean system property {@code org.openjdk.java.util.stream.tripwire}
 * is set to {@code true} then diagnostic warnings are reported if boxing of
 * primitive values occur when operating on primitive subtype specializations.
 *
 * @param <T> the type of elements returned by this Spliterator
 *
 * @see Collection
 * @since 1.8
 */
public interface Spliterator<T> {
    /**
     * If a remaining element exists, performs the given action on it,
     * returning {@code true}; else returns {@code false}.  If this
     * Spliterator is {@link #ORDERED} the action is performed on the
     * next element in encounter order.  Exceptions thrown by the
     * action are relayed to the caller.
     *
     * @param action The action
     * @return {@code false} if no remaining elements existed
     * upon entry to this method, else {@code true}.
     * @throws NullPointerException if the specified action is null
     */
    boolean tryAdvance(Consumer<? super T> action);

    /**
     * Performs the given action for each remaining element, sequentially in
     * the current thread, until all elements have been processed or the action
     * throws an exception.  If this Spliterator is {@link #ORDERED}, actions
     * are performed in encounter order.  Exceptions thrown by the action
     * are relayed to the caller.
     *
     * @implSpec
     * The default implementation repeatedly invokes {@link #tryAdvance} until
     * it returns {@code false}.  It should be overridden whenever possible.
     *
     * @param action The action
     * @throws NullPointerException if the specified action is null
     */
    default void forEachRemaining(Consumer<? super T> action) {
        do { } while (tryAdvance(action));
    }

    /**
     * If this spliterator can be partitioned, returns a Spliterator
     * covering elements, that will, upon return from this method, not
     * be covered by this Spliterator.
     *
     * <p>If this Spliterator is {@link #ORDERED}, the returned Spliterator
     * must cover a strict prefix of the elements.
     *
     * <p>Unless this Spliterator covers an infinite number of elements,
     * repeated calls to {@code trySplit()} must eventually return {@code null}.
     * Upon non-null return:
     * <ul>
     * <li>the value reported for {@code estimateSize()} before splitting,
     * must, after splitting, be greater than or equal to {@code estimateSize()}
     * for this and the returned Spliterator; and</li>
     * <li>if this Spliterator is {@code SUBSIZED}, then {@code estimateSize()}
     * for this spliterator before splitting must be equal to the sum of
     * {@code estimateSize()} for this and the returned Spliterator after
     * splitting.</li>
     * </ul>
     *
     * <p>This method may return {@code null} for any reason,
     * including emptiness, inability to split after traversal has
     * commenced, data structure constraints, and efficiency
     * considerations.
     *
     * @apiNote
     * An ideal {@code trySplit} method efficiently (without
     * traversal) divides its elements exactly in half, allowing
     * balanced parallel computation.  Many departures from this ideal
     * remain highly effective; for example, only approximately
     * splitting an approximately balanced tree, or for a tree in
     * which leaf nodes may contain either one or two elements,
     * failing to further split these nodes.  However, large
     * deviations in balance and/or overly inefficient {@code
     * trySplit} mechanics typically result in poor parallel
     * performance.
     *
     * @return a {@code Spliterator} covering some portion of the
     * elements, or {@code null} if this spliterator cannot be split
     */
    Spliterator<T> trySplit();

    /**
     * Returns an estimate of the number of elements that would be
     * encountered by a {@link #forEachRemaining} traversal, or returns {@link
     * Long#MAX_VALUE} if infinite, unknown, or too expensive to compute.
     *
     * <p>If this Spliterator is {@link #SIZED} and has not yet been partially
     * traversed or split, or this Spliterator is {@link #SUBSIZED} and has
     * not yet been partially traversed, this estimate must be an accurate
     * count of elements that would be encountered by a complete traversal.
     * Otherwise, this estimate may be arbitrarily inaccurate, but must decrease
     * as specified across invocations of {@link #trySplit}.
     *
     * @apiNote
     * Even an inexact estimate is often useful and inexpensive to compute.
     * For example, a sub-spliterator of an approximately balanced binary tree
     * may return a value that estimates the number of elements to be half of
     * that of its parent; if the root Spliterator does not maintain an
     * accurate count, it could estimate size to be the power of two
     * corresponding to its maximum depth.
     *
     * @return the estimated size, or {@code Long.MAX_VALUE} if infinite,
     *         unknown, or too expensive to compute.
     */
    long estimateSize();

    /**
     * Convenience method that returns {@link #estimateSize()} if this
     * Spliterator is {@link #SIZED}, else {@code -1}.
     * @implSpec
     * The default implementation returns the result of {@code estimateSize()}
     * if the Spliterator reports a characteristic of {@code SIZED}, and
     * {@code -1} otherwise.
     *
     * @return the exact size, if known, else {@code -1}.
     */
    default long getExactSizeIfKnown() {
        return (characteristics() & SIZED) == 0 ? -1L : estimateSize();
    }

    /**
     * Returns a set of characteristics of this Spliterator and its
     * elements. The result is represented as ORed values from {@link
     * #ORDERED}, {@link #DISTINCT}, {@link #SORTED}, {@link #SIZED},
     * {@link #NONNULL}, {@link #IMMUTABLE}, {@link #CONCURRENT},
     * {@link #SUBSIZED}.  Repeated calls to {@code characteristics()} on
     * a given spliterator, prior to or in-between calls to {@code trySplit},
     * should always return the same result.
     *
     * <p>If a Spliterator reports an inconsistent set of
     * characteristics (either those returned from a single invocation
     * or across multiple invocations), no guarantees can be made
     * about any computation using this Spliterator.
     *
     * @apiNote The characteristics of a given spliterator before splitting
     * may differ from the characteristics after splitting.  For specific
     * examples see the characteristic values {@link #SIZED}, {@link #SUBSIZED}
     * and {@link #CONCURRENT}.
     *
     * @return a representation of characteristics
     */
    int characteristics();

    /**
     * Returns {@code true} if this Spliterator's {@link
     * #characteristics} contain all of the given characteristics.
     *
     * @implSpec
     * The default implementation returns true if the corresponding bits
     * of the given characteristics are set.
     *
     * @param characteristics the characteristics to check for
     * @return {@code true} if all the specified characteristics are present,
     * else {@code false}
     */
    default boolean hasCharacteristics(int characteristics) {
        return (characteristics() & characteristics) == characteristics;
    }

    /**
     * If this Spliterator's source is {@link #SORTED} by a {@link Comparator},
     * returns that {@code Comparator}. If the source is {@code SORTED} in
     * {@linkplain Comparable natural order}, returns {@code null}.  Otherwise,
     * if the source is not {@code SORTED}, throws {@link IllegalStateException}.
     *
     * @implSpec
     * The default implementation always throws {@link IllegalStateException}.
     *
     * @return a Comparator, or {@code null} if the elements are sorted in the
     * natural order.
     * @throws IllegalStateException if the spliterator does not report
     *         a characteristic of {@code SORTED}.
     */
    default Comparator<? super T> getComparator() {
        throw new IllegalStateException();
    }

    /**
     * Characteristic value signifying that an encounter order is defined for
     * elements. If so, this Spliterator guarantees that method
     * {@link #trySplit} splits a strict prefix of elements, that method
     * {@link #tryAdvance} steps by one element in prefix order, and that
     * {@link #forEachRemaining} performs actions in encounter order.
     *
     * <p>A {@link Collection} has an encounter order if the corresponding
     * {@link Collection#iterator} documents an order. If so, the encounter
     * order is the same as the documented order. Otherwise, a collection does
     * not have an encounter order.
     *
     * @apiNote Encounter order is guaranteed to be ascending index order for
     * any {@link List}. But no order is guaranteed for hash-based collections
     * such as {@link HashSet}. Clients of a Spliterator that reports
     * {@code ORDERED} are expected to preserve ordering constraints in
     * non-commutative parallel computations.
     */
    public static final int ORDERED    = 0x00000010;

    /**
     * Characteristic value signifying that, for each pair of
     * encountered elements {@code x, y}, {@code !x.equals(y)}. This
     * applies for example, to a Spliterator based on a {@link Set}.
     */
    public static final int DISTINCT   = 0x00000001;

    /**
     * Characteristic value signifying that encounter order follows a defined
     * sort order. If so, method {@link #getComparator()} returns the associated
     * Comparator, or {@code null} if all elements are {@link Comparable} and
     * are sorted by their natural ordering.
     *
     * <p>A Spliterator that reports {@code SORTED} must also report
     * {@code ORDERED}.
     *
     * @apiNote The spliterators for {@code Collection} classes in the JDK that
     * implement {@link NavigableSet} or {@link SortedSet} report {@code SORTED}.
     */
    public static final int SORTED     = 0x00000004;

    /**
     * Characteristic value signifying that the value returned from
     * {@code estimateSize()} prior to traversal or splitting represents a
     * finite size that, in the absence of structural source modification,
     * represents an exact count of the number of elements that would be
     * encountered by a complete traversal.
     *
     * @apiNote Most Spliterators for Collections, that cover all elements of a
     * {@code Collection} report this characteristic. Sub-spliterators, such as
     * those for {@link HashSet}, that cover a sub-set of elements and
     * approximate their reported size do not.
     */
    public static final int SIZED      = 0x00000040;

    /**
     * Characteristic value signifying that the source guarantees that
     * encountered elements will not be {@code null}. (This applies,
     * for example, to most concurrent collections, queues, and maps.)
     */
    public static final int NONNULL    = 0x00000100;

    /**
     * Characteristic value signifying that the element source cannot be
     * structurally modified; that is, elements cannot be added, replaced, or
     * removed, so such changes cannot occur during traversal. A Spliterator
     * that does not report {@code IMMUTABLE} or {@code CONCURRENT} is expected
     * to have a documented policy (for example throwing
     * {@link ConcurrentModificationException}) concerning structural
     * interference detected during traversal.
     */
    public static final int IMMUTABLE  = 0x00000400;

    /**
     * Characteristic value signifying that the element source may be safely
     * concurrently modified (allowing additions, replacements, and/or removals)
     * by multiple threads without external synchronization. If so, the
     * Spliterator is expected to have a documented policy concerning the impact
     * of modifications during traversal.
     *
     * <p>A top-level Spliterator should not report both {@code CONCURRENT} and
     * {@code SIZED}, since the finite size, if known, may change if the source
     * is concurrently modified during traversal. Such a Spliterator is
     * inconsistent and no guarantees can be made about any computation using
     * that Spliterator. Sub-spliterators may report {@code SIZED} if the
     * sub-split size is known and additions or removals to the source are not
     * reflected when traversing.
     *
     * @apiNote Most concurrent collections maintain a consistency policy
     * guaranteeing accuracy with respect to elements present at the point of
     * Spliterator construction, but possibly not reflecting subsequent
     * additions or removals.
     */
    public static final int CONCURRENT = 0x00001000;

    /**
     * Characteristic value signifying that all Spliterators resulting from
     * {@code trySplit()} will be both {@link #SIZED} and {@link #SUBSIZED}.
     * (This means that all child Spliterators, whether direct or indirect, will
     * be {@code SIZED}.)
     *
     * <p>A Spliterator that does not report {@code SIZED} as required by
     * {@code SUBSIZED} is inconsistent and no guarantees can be made about any
     * computation using that Spliterator.
     *
     * @apiNote Some spliterators, such as the top-level spliterator for an
     * approximately balanced binary tree, will report {@code SIZED} but not
     * {@code SUBSIZED}, since it is common to know the size of the entire tree
     * but not the exact sizes of subtrees.
     */
    public static final int SUBSIZED = 0x00004000;

    /**
     * A Spliterator specialized for primitive values.
     *
     * @param <T> the type of elements returned by this Spliterator.  The
     * type must be a wrapper type for a primitive type, such as {@code Integer}
     * for the primitive {@code int} type.
     * @param <T_CONS> the type of primitive consumer.  The type must be a
     * primitive specialization of {@link java.util.function.Consumer} for
     * {@code T}, such as {@link java.util.function.IntConsumer} for
     * {@code Integer}.
     * @param <T_SPLITR> the type of primitive Spliterator.  The type must be
     * a primitive specialization of Spliterator for {@code T}, such as
     * {@link Spliterator.OfInt} for {@code Integer}.
     *
     * @see Spliterator.OfInt
     * @see Spliterator.OfLong
     * @see Spliterator.OfDouble
     * @since 1.8
     */
    public interface OfPrimitive<T, T_CONS, T_SPLITR extends Spliterator.OfPrimitive<T, T_CONS, T_SPLITR>>
            extends Spliterator<T> {
        @Override
        T_SPLITR trySplit();

        /**
         * If a remaining element exists, performs the given action on it,
         * returning {@code true}; else returns {@code false}.  If this
         * Spliterator is {@link #ORDERED} the action is performed on the
         * next element in encounter order.  Exceptions thrown by the
         * action are relayed to the caller.
         *
         * @param action The action
         * @return {@code false} if no remaining elements existed
         * upon entry to this method, else {@code true}.
         * @throws NullPointerException if the specified action is null
         */
        @SuppressWarnings("overloads")
        boolean tryAdvance(T_CONS action);

        /**
         * Performs the given action for each remaining element, sequentially in
         * the current thread, until all elements have been processed or the
         * action throws an exception.  If this Spliterator is {@link #ORDERED},
         * actions are performed in encounter order.  Exceptions thrown by the
         * action are relayed to the caller.
         *
         * @implSpec
         * The default implementation repeatedly invokes {@link #tryAdvance}
         * until it returns {@code false}.  It should be overridden whenever
         * possible.
         *
         * @param action The action
         * @throws NullPointerException if the specified action is null
         */
        @SuppressWarnings("overloads")
        default void forEachRemaining(T_CONS action) {
            do { } while (tryAdvance(action));
        }
    }

    /**
     * A Spliterator specialized for {@code int} values.
     * @since 1.8
     */
    public interface OfInt extends OfPrimitive<Integer, IntConsumer, OfInt> {

        @Override
        OfInt trySplit();

        @Override
        boolean tryAdvance(IntConsumer action);

        @Override
        default void forEachRemaining(IntConsumer action) {
            do { } while (tryAdvance(action));
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code IntConsumer} then it is cast
         * to {@code IntConsumer} and passed to
         * {@link #tryAdvance(java.util.function.IntConsumer)}; otherwise
         * the action is adapted to an instance of {@code IntConsumer}, by
         * boxing the argument of {@code IntConsumer}, and then passed to
         * {@link #tryAdvance(java.util.function.IntConsumer)}.
         */
        @Override
        default boolean tryAdvance(Consumer<? super Integer> action) {
            if (action instanceof IntConsumer) {
                return tryAdvance((IntConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfInt.tryAdvance((IntConsumer) action::accept)");
                return tryAdvance((IntConsumer) action::accept);
            }
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code IntConsumer} then it is cast
         * to {@code IntConsumer} and passed to
         * {@link #forEachRemaining(java.util.function.IntConsumer)}; otherwise
         * the action is adapted to an instance of {@code IntConsumer}, by
         * boxing the argument of {@code IntConsumer}, and then passed to
         * {@link #forEachRemaining(java.util.function.IntConsumer)}.
         */
        @Override
        default void forEachRemaining(Consumer<? super Integer> action) {
            if (action instanceof IntConsumer) {
                forEachRemaining((IntConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfInt.forEachRemaining((IntConsumer) action::accept)");
                forEachRemaining((IntConsumer) action::accept);
            }
        }
    }

    /**
     * A Spliterator specialized for {@code long} values.
     * @since 1.8
     */
    public interface OfLong extends OfPrimitive<Long, LongConsumer, OfLong> {

        @Override
        OfLong trySplit();

        @Override
        boolean tryAdvance(LongConsumer action);

        @Override
        default void forEachRemaining(LongConsumer action) {
            do { } while (tryAdvance(action));
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code LongConsumer} then it is cast
         * to {@code LongConsumer} and passed to
         * {@link #tryAdvance(java.util.function.LongConsumer)}; otherwise
         * the action is adapted to an instance of {@code LongConsumer}, by
         * boxing the argument of {@code LongConsumer}, and then passed to
         * {@link #tryAdvance(java.util.function.LongConsumer)}.
         */
        @Override
        default boolean tryAdvance(Consumer<? super Long> action) {
            if (action instanceof LongConsumer) {
                return tryAdvance((LongConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfLong.tryAdvance((LongConsumer) action::accept)");
                return tryAdvance((LongConsumer) action::accept);
            }
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code LongConsumer} then it is cast
         * to {@code LongConsumer} and passed to
         * {@link #forEachRemaining(java.util.function.LongConsumer)}; otherwise
         * the action is adapted to an instance of {@code LongConsumer}, by
         * boxing the argument of {@code LongConsumer}, and then passed to
         * {@link #forEachRemaining(java.util.function.LongConsumer)}.
         */
        @Override
        default void forEachRemaining(Consumer<? super Long> action) {
            if (action instanceof LongConsumer) {
                forEachRemaining((LongConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfLong.forEachRemaining((LongConsumer) action::accept)");
                forEachRemaining((LongConsumer) action::accept);
            }
        }
    }

    /**
     * A Spliterator specialized for {@code double} values.
     * @since 1.8
     */
    public interface OfDouble extends OfPrimitive<Double, DoubleConsumer, OfDouble> {

        @Override
        OfDouble trySplit();

        @Override
        boolean tryAdvance(DoubleConsumer action);

        @Override
        default void forEachRemaining(DoubleConsumer action) {
            do { } while (tryAdvance(action));
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code DoubleConsumer} then it is
         * cast to {@code DoubleConsumer} and passed to
         * {@link #tryAdvance(java.util.function.DoubleConsumer)}; otherwise
         * the action is adapted to an instance of {@code DoubleConsumer}, by
         * boxing the argument of {@code DoubleConsumer}, and then passed to
         * {@link #tryAdvance(java.util.function.DoubleConsumer)}.
         */
        @Override
        default boolean tryAdvance(Consumer<? super Double> action) {
            if (action instanceof DoubleConsumer) {
                return tryAdvance((DoubleConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfDouble.tryAdvance((DoubleConsumer) action::accept)");
                return tryAdvance((DoubleConsumer) action::accept);
            }
        }

        /**
         * {@inheritDoc}
         * @implSpec
         * If the action is an instance of {@code DoubleConsumer} then it is
         * cast to {@code DoubleConsumer} and passed to
         * {@link #forEachRemaining(java.util.function.DoubleConsumer)};
         * otherwise the action is adapted to an instance of
         * {@code DoubleConsumer}, by boxing the argument of
         * {@code DoubleConsumer}, and then passed to
         * {@link #forEachRemaining(java.util.function.DoubleConsumer)}.
         */
        @Override
        default void forEachRemaining(Consumer<? super Double> action) {
            if (action instanceof DoubleConsumer) {
                forEachRemaining((DoubleConsumer) action);
            }
            else {
                if (Tripwire.ENABLED)
                    Tripwire.trip(getClass(),
                                  "{0} calling Spliterator.OfDouble.forEachRemaining((DoubleConsumer) action::accept)");
                forEachRemaining((DoubleConsumer) action::accept);
            }
        }
    }
}
```

+ 结果比较:

![结果比较](https://image.cjyong.com/blog/bj8_3.jpg)
