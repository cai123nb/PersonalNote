### 函数式编程
如果一个方法既不修改它内嵌类的状态, 也不修改其他对象的状态, 使用return返回所有的计算结果. 那么我们称其为纯粹的或者无副作用的.
副作用: 函数的效果已经超出函数本身的范畴. 如: 除了构造器内的初始化操作, 对类内部数据的任何修改, 包括赋值操作(如Stter方法); 抛出异常; 进行输入输出操作, 如向文件写入数据.
> 声明式编程: 我们编程一般使用两种思考方式, 一种专注于如何实现, 如"首先需要做..., 然后需要做..., 最后需要做...". 如我们检索所有的交易记录, 寻找最贵的一条记录:
```java
Transaction mostExpensive = transactions.get(0);
if(mostExpensive == null){
	throw new IllegalArgumentException("Empty list of transactions");
}
for( Transaction t : transactions.subList(1, transactions.size())){
	if(t.getValue() > mostExpensive.getValue()){
    	mostExpensive = t;
    }
}
```
另一种方式更加关注要做什么:
```java
Optional<Transaction> mostExpensive = transactions.stream()
	.max(comparing(Transaction::getValue));
```
我们把要做什么留给函数库进行实现, 我们称这种思想称为内部迭代. 这样我们看起代码来, 就像是看待问题陈述一样. 这种编程风格称为声明式编程. 而函数式编程就是这种思想的实践.

函数式编程是什么: 使用函数进行编程的方式. `函数`接受零个或者多个参数, 生成一个或多个结果, 并且不会产生任何副作用. 就像`黑盒`一样, 一般我们分为两种: 
+ 纯粹的函数式编程: 像数学函数一样, 完全没有副作用, 完全使用函数式操作,
+ 函数式编程: 函数具有一定的副作用, 不过该副作用不会被其他调用者所感知.
其中, 函数式编程需要满足`引用透明性`特性: 函数无论何时/何处调用, 如果使用相同的输入, 总能得到一样的结果.

### 函数式编程实战
给定一个列表List<Value>, 获取他的所有子集. 如给定{1,2}, 返回的子集有: {1}, {2}, {}, {1,2}.
```java
static List<List<Integer>> subsets(List<Integer> list){
	if(list.empty()){
    	List<List<Integer>> ans = new ArrayList<>();
        ans.add(Collections.emptyList());
        return ans;
    }
    
    Integer first = list.get(0);
    List<Integer> rest = list.subList(1, list.size());
    
    List<List<Integer>> subans = subsets(rest);
    List<List<Integer>> subans2 = insertAll(first, subans);
    
    return concat(subans, subans2);
}

    public static List<List<Integer>> insertAll(Integer first, List<List<Integer>> lists) {
        List<List<Integer>> result = new ArrayList<>();
        for (List<Integer> l : lists) {
            List<Integer> copyList = new ArrayList<>(); //注意这里使用拷贝, 防止影响传递的参数
            copyList.add(first);
            copyList.addAll(l);
            result.add(copyList);
        }
        return result;
    }

    static List<List<Integer>> concat(List<List<Integer>> a, List<List<Integer>> b) {
        List<List<Integer>> r = new ArrayList<>(a); //使用新建的对象, 防止污染参数
        r.addAll(b);
        return r;
        //不应该使用 return a.addAll(b);
    }
```

### 递归和迭代
纯粹的函数式编程语言通常不包含while和for这样的迭代器. 因为这种类型的构造器经常隐藏陷阱, 诱导你修改对象.(这一点不太理解或认同) 所以纯粹的函数编程大多使用递归:
如:
```java
//循环
public static int factorialIterative(int n) {
    int r = 1;
    for (int i = 1; i <= n; i++) {
        r*=i;
    }
    return r;
}
//普通的递归
public static long factorialRecursive(long n) {
    return n == 1 ? 1 : n*factorialRecursive(n-1);
}
//使用stream
public static long factorialStreams(long n){
    return LongStream.rangeClosed(1, n)
                     .reduce(1, (long a, long b) -> a * b);
}
//使用`尾递`优化递归, 放置堆栈溢出
public static long factorialTailRecursive(long n) {
    return factorialHelper(1, n);
}

public static long factorialHelper(long acc, long n) {
    return n == 1 ? acc : factorialHelper(acc * n, n-1);
}
```
> Java还没有支持`尾递(tail-call optimization)` 优化, 但是它为编译器优化打开了一扇门.

### 高阶函数
满足任一要求的函数:
+ 接受至少一个函数作为参数
+ 返回的结果是一个函数
如:
```java
Function<Function<Double, Double>  differentiate(Function<Double, Double> func);
```
需要注意的的是传递给高阶函数的函数需要是无副作用的函数.

### 科里化
定义: 将一种具备两个参数(如x,y)的函数f转化为使用一个参数的函数g, 并且这个函数的返回值也是一个函数, 它会作为新函数的一个参数. 前后两者的返回值相同. 如: f(x, y) = (g(x))(y);
一个简单的例子:
```java
public static void main(String[] args) {
    DoubleUnaryOperator convertCtoF = curriedConverter(9.0/5, 32);
    DoubleUnaryOperator convertUSDtoGBP = curriedConverter(0.6, 0);
    DoubleUnaryOperator convertKmtoMi = curriedConverter(0.6214, 0);

    System.out.println(convertCtoF.applyAsDouble(24));
    System.out.println(convertUSDtoGBP.applyAsDouble(100));
    System.out.println(convertKmtoMi.applyAsDouble(20));

    DoubleUnaryOperator convertFtoC = expandedCurriedConverter(-32, 5.0/9, 0);
    System.out.println(convertFtoC.applyAsDouble(98.6));
}

static double converter(double x, double y, double z) {
    return x * y + z;
}

static DoubleUnaryOperator curriedConverter(double y, double z) {
    return (double x) -> x * y + z;
}

static DoubleUnaryOperator expandedCurriedConverter(double w, double y, double z) {
    return (double x) -> (x + w) * y + z;
}
```

### 持久化数据结构
```java
//链表
public static void main(String[] args) {
    TrainJourney tj1 = new TrainJourney(40, new TrainJourney(30, null));
    TrainJourney tj2 = new TrainJourney(20, new TrainJourney(50, null));
    TrainJourney appended = append(tj1, tj2);
    visit(appended, tj -> { System.out.print(tj.price + " - "); });
    System.out.println();
    // A new TrainJourney is created without altering tj1 and tj2.
    TrainJourney appended2 = append(tj1, tj2);
    visit(appended2, tj -> { System.out.print(tj.price + " - "); });
    System.out.println();
    // tj1 is altered but it's still not visible in the results.
    TrainJourney linked = link(tj1, tj2);
    visit(linked, tj -> { System.out.print(tj.price + " - "); });
    System.out.println();
    // ... but here, if this code is uncommented, tj2 will be appended
    // at the end of the already altered tj1. This will cause a
    // StackOverflowError from the endless visit() recursive calls on
    // the tj2 part of the twice altered tj1.
    /*TrainJourney linked2 = link(tj1, tj2);
    visit(linked2, tj -> { System.out.print(tj.price + " - "); });
    System.out.println();*/
}
static class TrainJourney {
    public int price;
    public TrainJourney onward;

    public TrainJourney(int p, TrainJourney t) {
        price = p;
        onward = t;
    }
}
// 破坏性更新, 链接之后a会进行修改, 使得原来所有依赖a的部分出现问题
static TrainJourney link(TrainJourney a, TrainJourney b) {
    if (a == null) {
        return b;
    }
    TrainJourney t = a;
    while (t.onward != null) {
        t = t.onward;
    }
    t.onward = b;
    return a;
}
//构建一个新的对象, 链接a和b, 将结果返回.
static TrainJourney append(TrainJourney a, TrainJourney b) {
    return a == null ? b : new TrainJourney(a.price, append(a.onward, b));
}
static void visit(TrainJourney journey, Consumer<TrainJourney> c) {
    if (journey != null) {
        c.accept(journey);
        visit(journey.onward, c);
    }
}
//Tree
public static void main(String[] args) {
    Tree t = new Tree("Mary", 22,
            new Tree("Emily", 20,
                    new Tree("Alan", 50, null, null),
                    new Tree("Georgie", 23, null, null)
            ),
            new Tree("Tian", 29,
                    new Tree("Raoul", 23, null, null),
                    null
            )
    );
    // found = 23
    System.out.println(lookup("Raoul", -1, t));
    // not found = -1
    System.out.println(lookup("Jeff", -1, t));

    Tree f = fupdate("Jeff", 80, t);
    // found = 80
    System.out.println(lookup("Jeff", -1, f));

    Tree u = update("Jim", 40, t);
    // t was not altered by fupdate, so Jeff is not found = -1
    System.out.println(lookup("Jeff", -1, u));
    // found = 40
    System.out.println(lookup("Jim", -1, u));

    Tree f2 = fupdate("Jeff", 80, t);
    // found = 80
    System.out.println(lookup("Jeff", -1, f2));
    // f2 built from t altered by update() above, so Jim is still present = 40
    System.out.println(lookup("Jim", -1, f2));
}
static class Tree {
    private String key;
    private int val;
    private Tree left, right;

    public Tree(String k, int v, Tree l, Tree r) {
        key = k;
        val = v;
        left = l;
        right = r;
    }
}
public static int lookup(String k, int defaultval, Tree t) {
    if (t == null)
        return defaultval;
    if (k.equals(t.key))
        return t.val;
    return lookup(k, defaultval, k.compareTo(t.key) < 0 ? t.left : t.right);
}
//破坏性的更新, 同上
public static Tree update(String k, int newval, Tree t) {
    if (t == null)
        t = new Tree(k, newval, null, null);
    else if (k.equals(t.key))
        t.val = newval;
    else if (k.compareTo(t.key) < 0)
        t.left = update(k, newval, t.left);
    else
        t.right = update(k, newval, t.right);
    return t;
}
//新建一条从根节点到新节点或者更新节点的路径, 进行返回, 不会损坏现有的结构
public static Tree fupdate(String k, int newval, Tree t) {
    return (t == null) ?
        new Tree(k, newval, null, null) :
         k.equals(t.key) ?
           new Tree(k, newval, t.left, t.right) :
      k.compareTo(t.key) < 0 ?
        new Tree(t.key, t.val, fupdate(k,newval, t.left), t.right) :
        new Tree(t.key, t.val, t.left, fupdate(k,newval, t.right));
}
```

### Stream的延迟计算
Stream: 当你向一个Stream发起一系列的操作请求时, 这些请求会被保存起来, 只有当你向Stream发起一个终端请求时, 才会实际地进行计算. 你可以将一个函数作为值放入到某个数据结构中, 大多数的时候它就静静地待在哪里, 一旦对其进行调用, 它能创建更多的结构.
```java
public class LazyLists {

    public static void main(String[] args) {
        MyList<Integer> l = new MyLinkedList<>(5, new MyLinkedList<>(10,
                new Empty<Integer>()));

        System.out.println(l.head());

        LazyList<Integer> numbers = from(2);
        int two = numbers.head();
        int three = numbers.tail().head();
        int four = numbers.tail().tail().head();
        System.out.println(two + " " + three + " " + four);

        numbers = from(2);
        int prime_two = primes(numbers).head();
        int prime_three = primes(numbers).tail().head();
        int prime_five = primes(numbers).tail().tail().head();
        System.out.println(prime_two + " " + prime_three + " " + prime_five);

        // this will run until a stackoverflow occur because Java does not
        // support tail call elimination
        // printAll(primes(from(2)));
    }

    interface MyList<T> {
        T head();

        MyList<T> tail();

        default boolean isEmpty() {
            return true;
        }

        MyList<T> filter(Predicate<T> p);
    }

    static class MyLinkedList<T> implements MyList<T> {
        final T head;
        final MyList<T> tail;

        public MyLinkedList(T head, MyList<T> tail) {
            this.head = head;
            this.tail = tail;
        }

        public T head() {
            return head;
        }

        public MyList<T> tail() {
            return tail;
        }

        public boolean isEmpty() {
            return false;
        }

        public MyList<T> filter(Predicate<T> p) {
            return isEmpty() ? this : p.test(head()) ? new MyLinkedList<>(
                    head(), tail().filter(p)) : tail().filter(p);
        }
    }

    static class Empty<T> implements MyList<T> {
        public T head() {
            throw new UnsupportedOperationException();
        }

        public MyList<T> tail() {
            throw new UnsupportedOperationException();
        }

        public MyList<T> filter(Predicate<T> p) {
            return this;
        }
    }

    static class LazyList<T> implements MyList<T> {
        final T head;
        final Supplier<MyList<T>> tail;

        public LazyList(T head, Supplier<MyList<T>> tail) {
            this.head = head;
            this.tail = tail;
        }

        public T head() {
            return head;
        }

        public MyList<T> tail() {
            return tail.get();
        }

        public boolean isEmpty() {
            return false;
        }

        public MyList<T> filter(Predicate<T> p) {
            return isEmpty() ? this : p.test(head()) ? new LazyList<>(head(),
                    () -> tail().filter(p)) : tail().filter(p);
        }

    }

    public static LazyList<Integer> from(int n) {
        return new LazyList<Integer>(n, () -> from(n + 1));
    }

    public static MyList<Integer> primes(MyList<Integer> numbers) {
        return new LazyList<>(numbers.head(), () -> primes(numbers.tail()
                .filter(n -> n % numbers.head() != 0)));
    }

    static <T> void printAll(MyList<T> numbers) {
        if (numbers.isEmpty()) {
            return;
        }
        System.out.println(numbers.head());
        printAll(numbers.tail());
    }

}
```

### 缓存
如果你有某一项操作非常耗时, 并且该项操作的对象是固定的, 我们可以为该操作进行缓存:
```java
final Map<Range, Integer> numberOfNodes = new HashMap<>();
Integer computerNumberOfNodesUsingCache(Range range){
	Integer result = numberOfNodes.get(range);
    if(result != null){
    	return result;
    }
    result = computeNumberOfNodes(range);
    numberOfNodes.put(range, result);
    return result;
}
//或者
Integer computerNumberOfNodesUsingCache(Range range) {
	return numberOfNodes.computeIfAbsent(range, this::computeNumberOfNodes);
}
```
但是该代码不是线程安全的, 如果有多个线程并发调用时, 可能出现冲突. 但是如果我们添加锁的话, 可能我们最初的目的提高性能又不能完全得到实现.

### 返回同样的对象
我们在函数式编程中, `返回同样的对象`往往意味着`equal`, 而不去引用的相同.

### 结合器
高阶函数接受两个或者多个函数, 返回一个函数. 常常被称作`接合器`.
```java
public class Combinators {

    public static void main(String[] args) {
        System.out.println(repeat(3, (Integer x) -> 2 * x).apply(10));
    }

    static <A, B, C> Function<A, C> compose(Function<B, C> g, Function<A, B> f) {
        return x -> g.apply(f.apply(x));
    }

    static <A> Function<A, A> repeat(int n, Function<A, A> f) {
        return n == 0 ? x -> x : compose(f, repeat(n - 1, f));
    }
}
```

