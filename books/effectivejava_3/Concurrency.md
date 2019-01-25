# Concurrency

Effective Java(v3), Chapter 11. 本章主要提出一些建议来帮助你完成正确的,清晰的,具有良好的文档的同步代码(多线程).

## Item 78: Synchronize access to shared mutable data

`synchronize`关键字保证同一时间只能有一个线程执行对应的方法或者方法块. 有很多人认为, `synchronize`主要实现的功能就是`mutual exclusion`(排它锁), 防止一个对象在不连续的情况时(其中一个线程正在访问修改时)被另一个线程访问到. 这种想法是正确的, 但是只是一部分. 因为很多方式都可以实现这一点(保证一个线程修改一个对象状态时不被另一个线程发现, 如拷贝一份进行操作, 操作完成再替换原来的引用). 还有很重要的一个功能就是通过`synchronize`可以保证进入同步块时, 可以保证当前的数据是最新的(先前所有进入的线程进行的数据操作都是映射到数据上的, 即可见性).

JLS(Java Language Specification)中声明了单个变量的读写(除了long和double)都是原子性的: 保证线程对变量的读写是原子性的(不需要添加`synchronize`语句). 但是对可见性却无法保证. **如果多线程需要通过共享变量来进行通信的话, 那么`synchronize`是必不可少的.** 这样可以保证变量修改后的可见性. 如我们设计一个变量来停止线程运行, 代码最初是这样设计的:

```java
//Broken - How long would you expect this program to run
public class StopThread {
    private static boolean stopRequest;
    public void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

我们最初的设想是后台线程运行一秒后, 自动关闭. 但是实际上却是恰恰相反, 后台线程一直在跑. 这是为什么? 虽然`stopRequest`是变量, 读写是原子性的. 但是没办法保证其可见性. JVM虚拟机优化时, 很有可能将上述代码优化如下:

```java
while (!stopRequest) {
    i++;
}

if (!stopRequest) {
    while (true) {
        i++;
    }
}
```

这就是著名的`hoisting`优化. 因为虚拟机默认这是在单线程中运行, 你没有告诉JVM这个变量有可能被多线程访问. 解决这个bug也非常简单, 添加`synchronize`即可:

```java
//Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequest;

    private static synchronized void requestStop() {
        stopRequest = true;
    }
     
    private static synchronized boolean stopRequested() {
        return stopRequested;
    }

    public void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested()) {
                i++;
            }
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```

注意这里的读写方法都进行了加锁操作, 这样可以保证读和写的同步性. 另外一个可选的解决方法就是声明该变量为`volatile`类型:

```java
//Cooperative thread termination with a volatile field
public class StopThread {
    private static volatile boolean stopRequest;

    public void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested) {
                i++;
            }
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
}
```

通过设置`stopRequest`变量为`volatile`类型, 可以保证每当读取`stopRequest`变量时为最新的一次写入值(即保证了可见性). 通过这一点可以很好的实现本次测试代码中的目的. 但是**volatile关键字只能保证变量的可见性, 并不能像`synchronize`一样保证同步**, 使用时需要非常的小心. 如:

```java
//Broken - require synchronization!
private static volatile int nextSerialNumber = 0;

public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```

这里虽然声明了`nextSerialNumber`为`volatile`类型, 但是却是没有办法在多线程时保证一致的, 因为`nextSerialNumber++`不是原子性的. 虽然通过`volatile`保证了读取时读取的是最新的一次写入, 但是`++`操作分为: 读取和写入两部分, 并不是同步的. 当读取时取得了最新的值还没写入, 这时候CPU切换了线程执行时间, 另一个线程读取的依旧是之前的值, 这时候就存在重复写入的情况, 破坏了一致性.

最简单的方法就是添加`synchronize`进行声明, 这时候就可以省略掉`volatile`声明. 还有一种不通过加锁的方式实现的, 那就是通过使用`AtomicLong`进行实现(本质为CAS).

要避免同步导致的各种各样的问题, 最简单的方法就是不要进行同步: 保证每个线程控制和掌握自己的数据, 不进行共享(如ThreadLocal). 如果采用这种措施, 那就应该在文档中进行声明. 这里存在一种特殊情况, 那就是对于一致性没有那么高的要求(同步的结果可以慢一点获取到),  即修改一个数据对象, 然后等一下在更新上去, 只在写入的时候进行同步, 获取时可以不进行加锁. 这种对象常被称为有效的不变量(`effectively immutable`). 线程进行更新也就是叫做`safe publication`. 有很多方法可以实现这种特性: 存储在静态域(类初始化时进行初始化), 声明变量为`volatile`, 声明变量为`final`, 正常的加锁操作, 放入到同步集合中等.

总而言之, 当多线程通过共享变量来进行数据分享时, 对该数据的读写都需要添加`synchronize`. `volatile`是一种可接受的替代方案, 但是使用时需要小心.

## Item 79: Avoid excessive synchronization

`Item78`说明了`synchronize`使用不足的坏处, 这里介绍一下过度使用`synchronize`的危害: 过度使用`synchronize`会导致性能下降, 死锁甚至不确定的行为.

要避免过度使用, 其中一个重要的准则就是**不要在同步块中调用外部方法**. 也就是说尽量保证同步块的精简, 尽量避免调用外部方法, 甚至是需要客户端实现的抽象方法. 这里以观察者模式为例, 实现一个集合实现了观察者模式, 当元素被添加到集合中时, 发送通知给所有的观察者.

```java
//Broken - invokes alien method from synchronized blcok!
public class ObservableSet<E> extends ForwardingSet<E> {
    public ObservableSet(Set<E> set) {
        super(set);
    }

    private final List<SetObserver<E>> observers = new ArrayList<>();

    public void addObserver(SetObserver<E> observer) {
        synchronized(observers) {
            observers.add(observer);
        }
    }

    public boolean removeObserver(SetObserver<E> observer) {
        synchronized(observers) {
            return observers.remove(observer);
        }
    }

    private void notifyElementAdded(E element) {
        synchronized(observers) {
            for (SetObserver<E> observer: observers) {
                observer.added(this, element);
            }
        }
    }

    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        if (added) {
            notifyElementAdded(element);
        }
        return added;
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c) {
            result |= add(element);
        }
        return result;
    }
}

@FunctionalInterface
public interface SetObserver<E> {
    void added(ObservableSet<E> set, E element);
}

//Unit test 1
public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

    set.addObserver((s, e) -> System.out.println(e));

    for (int i = 0; i < 100; i++) {
        set.add(i);
    }
}


//Unit test 2
public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

    set.addObserver(new SetObserver<>(){
        public void added(ObservableSet<Integer> s, Integer e) {
            System.out.println(e);
            if (e == 23) {
                s.removeObserver(this);
            }
        }
    });

    for (int i = 0; i < 100; i++) {
        set.add(i);
    }
}
```

这里创建了两个测试用例, 第一个可以正常执行, 第二个却不可以. 第二个原先的想法是输出0到23的数字, 但是却抛出了`ConcurrentModificationException`. 这是为什么呢? 因为在通知方法时`notifyElementAdded`中锁定了`observers`并进行了遍历, 但是在遍历过程中却想要移除集合中的一个对象. 同步块内防止了任何改变同步对象的操作, 这时候抛出了异常. 虽然同步块可以保证多线程时的同步问题, 但是却不能保证线程本身不会修改`observers`对象. 这里用一个更奇怪的用例来示范死锁:

```java
//Unit test 2
public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());

    set.addObserver(new SetObserver<>(){
        public void added(ObservableSet<Integer> s, Integer e) {
            System.out.println(e);
            if (e == 23) {
                ExecutorService exec = Executors.newSingleThreadExecutor();
                try {
                    exec.submit(() -> s.removeObserver(this)).get();
                } catch (ExecutionException | InterruptedException ex) {
                    throw new AssertionError(ex);
                } finally {
                    exec.shutdown();
                }
            }
        }
    });

    for (int i = 0; i < 100; i++) {
        set.add(i);
    }
}
```

当我们运行这个测试时, 我们并不能捕获任何异常, 因为这是一个死锁. `notifyElementAdded`已经锁定了`observers`对象, `observer`内部又开启了一个线程调用`removeObserver`方法, 又会尝试获取`observers`的锁. 外部在等待内部执行完成, 内部在等待外部释放锁, 这就形成了死锁. 第二个和第三个测试虽然都出现了问题, 但是没有影响`observers`, 还是保持了一致性. 有一种极端情况, 当内部尝试获取外部的锁时, 如果这时候外部的锁暂时失效了(这是有可能发生的, 因为Java内部实现的锁是Reentrant, 可重入锁), 这时候就可以修改对象, 导致不一致. 这种结果是灾难性的.

最简单的解决方法就是将外部调用移出同步块, 更好的方法则是使用`CopyOnWriteArrayList`.

```java
//Solver 1
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    for (SetObserver<E> observer : snapshot) {
        observer.added(this, element);
    }
}

//Solver 2
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers) {
        observer.added(this, element);
    }
}
```

如果你要完成一个可变类, 要实现同步性有两个选择: 省略所有的同步语句, 让客户自己进行同步操作. 或者就是在类内部使用同步, 保证类为线程安全的. 一般都使用第一种方法, 只有当你面临非常大的并发时, 采用第二种才是合理的. 在`java.util`中的集合采用第一种方式, `java.util.concurrent`则是采用第二种方式. 采用第二种方式可以在内部对锁进行优化: 如锁分离, 锁粗化, 消除锁等优化措施. 注意如果访问的对象是静态对象, 如果可能被多线程访问一定需要添加同步语句.

总而言之, 为了避免死锁等异常情况, 尽量保证同步块内部方法尽可能的小.

## Item 80: Prefer executors, tasks and streams to threads

随着`Executor Framework`被添加到`java.util.concurrent`包内, 如今创建多线程的任务变得非常的容易和简单:

```java
//Create one
ExecutorService exec = Executors.newSingleThreadExecutor();

//submit a runnable
exec.execute(runnable);

//terminate the executor
exec.shutdown();
```

通过`Executor Framework`我们可以完成的有更多: 可以等待特定任务的完成(通过get方法), 等待任意数量任务完成(通过invokeAny, invokeAll方法), 可以等待特定地执行任务完成(使用awaitTermination方法), 按照顺序执行任务(ExecutorCompletionService), 可以定时让任务执行(使用ScheduledThreadPoolExecutor)等等. 如果需要零配置创建不同的线程池, 可以直接使用使用`Executors`创建不同的线程池, 如果需要需要自定义线程池的各种特性, 可以直接使用`ThreadPoolExecutor`类进行配置.

在使用`Executors`内置的线程池时需要注意, 如`newFixedThreadPool,newSingleThreadExecutor`需要注意, 这里设置的队列`queue`长度为`Integer.MAX_VALUE`, 如果任务堆积的过多容易导致,内存过大而OOM. 使用`newCachedThreadPool,newScheduledThreadPool`也需要注意, 这里并没有限制子线程的数量, 如果某一时间请求暴增, 非常容易导致线程数量过多, 而OOM(`注, 部分摘自阿里Java开发手册`).

为什么说`Excutor Framework`比`Threads`更好呢? 其中一个很重要的原因就是, `Excutor Framework`将线程和任务进行了抽象化, 直接使用多线程不仅需要管理任务的执行, 还需要管理线程的存活. 而`Excutor Framework`将两者区分开了, 将线程的存活交给`Excutor Framework`进行管理, 而使用者只需要提交需要完成的任务即可. 任务抽象为两个类型: `Runnable`: 运行完直接结束. `Callable`: 运行完可以返回一个值或者抛出异常. 在`Java 7`中`Excutor Framework`引入了对`fork-join`任务的支持, 用户只需要调用`Excutors.newWorkStealingPool`创建一个`ForkJoinPool`, 然后提交`ForkJoinTask`即可. 由于`fork-join`本身`偷取`的特性, 可以最大化的让线程工作量饱和. 但是使用`fork-join`任务是比较复杂的, 幸运的是`Stream`的`Parallel Streams`就是使用了该技术, 简单的并行化就可以享受其带来的极大地效率.

总而言之, 合理的使用`Excutor Framework`可以为我们实现多线程带来很好的便利, 更多的学习可以参照`Java Concurrency In Practice`图书.

## Item 81: Prefer concurrency utilities to wait and notify

之前线程之间的通信和处理主要通过原始的`wait`和`notify`完成, 但是随着`java 5`的到来, 引入了高级的同步工具类, 可以极大地方便我们进行处理. **如果使用wait和notify实现问题会带来很大的复杂度, 可以考虑使用高级的同步工具类进行替换**.

这些高级同步类在`java.util.concurrent`包下面, 主要分为三大类: `Executor Framework`, `Concurrent collections`, `Synchronizers`. 第一个在前面已经介绍了, 这里主要介绍后两者的使用.

`Concurrent collections`主要是提供一些高性能的同步的容器实现类, 如`List, Queue和Map`. 这些集合类在内部高效实现了同步, 不需要在外部再进行同步操作. 正是因为同步集合类是在内部实现的, 如果需要组合一些操作时就没办法保证同步性. 因此集合接口声明和实现了很多`state-dependent modify operations`, 如`putIfAbsent`来尽可能地保证组合操作时的同步性. 这里以模拟实现`String.intern`为例:

```java
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();

//Optional 1
public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}

//Optional 2
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null) {
            result = s;
        }
    }
    return result;
}
```

另外还有很多集合实现了`blocking operations`, 如`BlockingQueue`. 这在生产消费模型中是非常有用的, 可以使用多个线程进行生产或者消费, 任务则可以通过队列进行中转.

第二个对象是`Synchronizers`, 主要用于让线程等待另一些线程完成工作后继续操作, 以此来保证顺序性. 常见的有: `CountDownLatch, Semaphore, CyclicBarrier, Exchanger, Phaser`. 这里以`CountDownLatch`为例: `CountDownLatch`允许先让一个或者多个线程等待另外一些线程(一个或者多个)进行操作. 具体实现: 接收一个整型参数(需要先完成的线程数量)进行计数, 每个需要先完成的线程执行完任务, 调用`countDown()`方法, 计数减少1, 计数为0之前, 阻塞主线程执行(也就是说外面的线程必须要等待内部线程完成之后, 才能继续进行). 

常见的用途有: 系统启动时, 某些服务启动需要依赖一部分服务, 可以使用`CountDownLatch`等待那部分服务启动之后再进行启动. 对多线程进行计时操作, 等所有的多线程都准备好了之后, 发布启动命令, 然后进行计时. 这里以第二个作用为例:

```java
//Simple framework for timing concurrent execution
public static long time(Executor executor, int concurrency, Runnable action) throws InterruptedException {
    CountDownLatch ready = new CountDownLatch(concurrency);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(concurrency);

    for (int i = 0; i < concurrency; i++) {
        executor.execute(() -> {
            ready.countDown();   //Tell timer current thread is ready
            try {
                start.await();  //Current thread wait for start signal.
                action.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown();   //Tell timer current thread are down.
            }
        });
    }

    ready.await();      //Wait for all workers to be ready
    long startNanos = System.nanoTime();
    start.countDown();  //And they're all ready, start all workers.
    done.await();       //Wait for all workers to finish
    return System.nanoTime() - startNanos;
}
```

虽然现在的版本很少使用`wait`和`notify`, 但是如果你需要维护遗留代码(以前的代码使用的是`wait`和`notify`), 这时候你需要对两者有一个基本的认识. `wait`: 在获取对象锁的时候, 如果发现某些条件不符合, 选择释放对象的锁, 并让当前线程堵塞等待唤醒(`notify,notifyAll`). `notify`: 也需要获取对象的锁, 并唤醒在对象中堵塞的线程(随机唤醒一条), 但是这时还不会释放锁, 还会继续执行下去. 因此使用时要保证`wait,notify,notifyAll`是放在同步块内使用. 另外需要确保的是将`wait`置于`while`循环中, 保证唤醒时的情况是正确的情况(防止被错误唤醒), 如果情况不对还会进行进行堵塞.

```java
synchronized(obj) {
    while(<condition does not hold>) {
        obj.wait();
    }
    ...//Perform action appropriate to condition.
}
```

即使如此, 还是很有可能被错误的唤醒, 当循环条件还不成立时:

+ 另外一个线程不小心获取到了该对象的锁, 修改了循坏条件中的变量, 然后调用了`notify`方法.

+ 另外的线程不小心或者故意的调用了`notify`, 这时候循环条件还不成立. 因此一定要保证对象是私有的, 不会轻易被获取到.

+ 当前通知线程过于`慷慨`, 直接调用`notifyAll`, 而这时候循环条件还不成立.

+ 堵塞的线程有一定的非常小的机率自醒(不需要唤醒). 也就是常说的`spurious wakeup`.

另外一个相关的问题就是, 是使用`notify`还是`notifyAll`. 一般保守起见是使用`notifAll`, 这样可以唤醒所有在当前对象堵塞的线程, 一定保证可以唤醒到我们要的那个线程, 别的线程如果被错误的唤醒会判断条件, 然后继续堵塞. 如果有多个的情况下, `notify`并不一定可以保证唤醒指定的线程. 因此这一般是使用`notifyAll`的. 但是也有特殊的情况, 如某些情况只需要唤醒一个线程进行工作即可, 出于性能的考虑, 可以使用`notify`.

总而言之, 当前的Java环境可以使用高级的同步工具类来替换`wait, notify`. 如果需要维护遗留代码时, 保证`wait`方法在`while`循环内, 并且`notifyAll`的优先级高于`notify`.

## Item 82: Document thread safety

当我们需要在多线程的情况下运行时, 线程安全性就非常重要了. 这一点需要我们在文档注释中好好标明. 很多人确定一个方法是不是线程安全的是通过是否含有`synchronize`来确定的, 这是不对的做法, 虽然方法含有`synchronize`一定是线程安全的, 但是这并不是线程安全的唯一依据(有些没有标记的也可以通过别的方式实现线程安全). 为了详细的讲解线程安全, 这里将线程安全划分为5个等级:

+ Immutable: 不变类, 标明这个类是不变类, 调用时不需要担心同步的问题.

+ Unconditionally thread-safe: 绝对线程安全, 虽然这个类是可变的, 但是内部已经通过充足的同步措施进行同步一致性的保证. 如: ConcurrentHashMap和AtomicLong.

+ Conditionally thread-safe: 有条件的线程安全, 类似绝对线程安全, 大部分的方法是线程安全的, 除了个别方法(这些方法调用需要在外部额外进行同步措施). 如Collections.synchronizedwrappers的iterator需要进行额外的同步操作.

+ Not thread-safe: 非线程安全, 这个类是可变的, 如果需要在多线程环境下使用就必须在外部进行同步操作.

+ Thread-hostile: 线程危险的, 这个类不适合用于多线程情况, 即使外部进行了同步操作也不能保证一定线程安全. 这种情况一般都不是故意的, 而是缺少考虑多线程的情况而导致的. 如`Item 78`的`generateSerialNumber`.

通常标明一个类是否是线程安全的是在类注解中声明, 其中条件线程安全应该在类文档注释中说明清楚, 那些操作是不安全的. 如果某个方法有特殊情况也需要进行特殊文档说明, 如条件安全时需要说明如果要实现这个方法的线程安全, 调用时需要采用什么类型的锁操作. 另外如果使用加锁的对象是公开的话, 那也需要小心一点. 虽然可以通过这个对象进行一系列的操作的原子化, 但是这个和很多在内部实现高效同步的集合(如ConcurrentHashMap)不兼容, 也存在DOS(denial-of-service)攻击的可能: 外部调用时获取这个锁不进行释放. 这时候可以使用一个小技巧进行规避:

```java
//Private lock object
private final Object lock = new Object();
public void foo() {
    synchronize(lock) {
        ...
    }
}
```

这样可以保证加锁对象不会被外部接触到. 这种方式特别适合绝对线程安全的实现方式, 如果是有条件的线程安全则是不太适合(这里需要方法指定特定的锁). 这种方式也适合继承, 如果在继承时使用实例来进行加锁的话, 如果对父类的操作都会影响子类, 是非常不好的.

这里有几个需要注意的特别事项, 私有的加锁对象一般设置为`final`, 防止后续修改导致同步失败的问题. 对`map`操作时, 对整个`map`进行加锁而不是`key`或者`value`加锁.

总而言之, 每个类都应该文档注释清楚是否线程安全, 有哪些特殊情况等.

## Item 83: Use lazy initialization judiciously

懒加载`Lazy initialization`是一种延迟初始化的技术. 本质的原理为:`don't do it unless you need to`. 一般的模式为:

```java
//Normal initialization of an instance field
private final FieldType field = computeFieldValue();

//Lazy initialization of an instance field
private FieldType field;

private synchronized FieldType getField() {
    if (field == null) {
        field = computeFieldValue();
    }
    return field;
}
```

懒加载可以用于静态变量引用和实例对象引用. 上面是一个简单的实例引用懒加载例子, 为了避免多线程的干扰, 使用`synchronized`进行标明. 这里介绍一些懒加载的优缺点: 好处就是通过懒加载的模式减轻了类初始化和实例化的时间. 缺点: 损害了变量引用获取的效率(以前访问可以直接调用即可,现在需要通过方法判断获取). 所以当我们使用懒加载的时候需要好好衡量一下是否值得使用懒加载. 如果该变量使用较少并且初始化代价较高, 你们使用懒加载是值得的. 如果变量使用频率较高, 且初始化代价不高, 那么不推荐使用懒加载. 真正唯一的衡量标准是进行测量, 使用前和使用后的效果比较.

如果确定使用懒加载, 那么还需要考虑的一点就是多线程时的情况, 防止被重复初始化. 当前像前面的示例, 加锁是最简单的操作了. 但是这样会对性能有较大的损害, 如果使用静态域对象时, 那么有一个改进方法:

```java
private static clsss FileHolder {
    static final FieldType field = computerFieldValue();
}

private static FieldType getField() {
    return FieldHolder.field;
}
```

这里使用了一个技巧, 就是利用JVM对类加载时(只会进行一次)自动加锁的性质来保证`field`的初始化是唯一的, 而不用每次获取都进行加锁. 如果是对于实例对象呢, 推荐使用`double check`.

```java
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (field == null) {
        synchronized (this) {
            if (field == null) {
                field = result = computeFieldValue();
            }
        }
    }
    return result;
}
```

注意这里对变量添加`volatile`防止指令重排序. 通过这种方法可以避免每次获取时都进行加锁, 而只是在第一次获取时进行加锁获取.

如果对重复初始化不在意的话, 可以对取消中间的`synchronized(this)`, 虽然存在重复初始化的可能, 但是避免了加锁操作.

总而言之, 使用懒加载时合理的评估是否可以值得, 如果确定的话, 合理设计懒加载的模式防止多线程带来的影响和提高性能.

## Item 84: Don't depend on the thread scheduler

当多个线程处于可运行时, 怎么确定是哪个线程先运行和运行多久? 这个是由操作系统中的线程调度器进行确定的. 一个设计良好的线程调度器应该尽量保证公平性, 但是也是不能百分百确定的, 并且不同的平台的底层实现是完全不一样的. **因此依靠线程调度器来保证程序的准确性或者优秀的性能是非常不靠谱的并且不兼容的**.

要解决这种情况的最好方法就是不要让同时运行的线程显著地大于处理器的数量, 尽量减少线程调度器的可选项. 这样即使在不同的平台上也不会差异太大. 要实现这一点可以参照很多有效的原则, 如: **如果线程不干有用的事, 就不要创建该线程**, **线程不应该一直处于busy-wait的状态**.

```java
//Awful CountDownLatch implementation - busy-waits increassantly!
public class SlowCountDownLatch {
    private int count;

    public SlowCountDownLatch(int count) {
        if (count < 0) {
            throw new IllegalArgumentException(count + " < 0");
        }
        this.count = count;
    }

    public void await() {
        while (true) {
            synchronized(this) {
                if (count == 0) {
                    return;
                }
            }
        }
    }

    public synchronized void countDown() {
        if (count != 0) {
            count--;
        }
    }
}
```

这就是一个典型的`busy-wait`的情形. 除此之外很多人会尝试使用`Thread.yield`(让出CPU处理时间, 让CPU重新选择线程)和设置线程的优先级进行调整. 虽然有一定的作用, 但是不能百分百保证有用, 更不能依靠这些操作来保证正确性.

总而言之, 不要依赖线程调度器来保证程序的稳定性或者性能, 这会导致代码不是健壮的和可移植的. 一个良好的解决方法是尽量保证可运行线程的数量适配处理器, 让线程调度器没有太多的可选项. 不要依赖`yield`和设置优先级来保证程序的正确性.