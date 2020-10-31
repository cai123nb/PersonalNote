# AQS(AbstractQueuedSynchronizer)

AQS为基于FIFO等待队列的同步器, 为实现`Lock`提供了必要的底层支持. 我们使用的锁实现都是通过AQS来实现的, 如`CountDownLatch`, `ReentrantLock`, `ReentrantReadWriteLock`等.

## 简单的排他锁

通过一个简单的排他锁为例, 深入了解AQS内部的相关知识:

```java
public class Mutex implements Lock {
  private static class Sync extends AbstractQueuedSynchronizer {
    @Override
    protected boolean isHeldExclusively() {
      return getState() == 1;
    }

    @Override
    public boolean tryAcquire(int acquires) {
      if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
      }
      return false;
    }

    @Override
    public boolean tryRelease(int releases) {
      if (getState() == 0) {
        throw new IllegalMonitorStateException();
      }
      setExclusiveOwnerThread(null);
      setState(0);
      return true;
    }

    Condition newCondition() {
      return new ConditionObject();
    }
  }

  private final Sync sync = new Sync();

  @Override
  public void lock() {
    sync.acquire(1);
  }

  @Override
  public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
  }

  @Override
  public boolean tryLock() {
    return sync.tryAcquire(1);
  }

  @Override
  public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(time));
  }

  @Override
  public void unlock() {
    sync.release(1);
  }

  @Override
  public Condition newCondition() {
    return sync.newCondition();
  }
}
```

这里简单实现了一个排他锁(不支持重入), 在内部构建了一个`Sync`对象, 而`Lock`的各项方法其实都是使用`Sync`提供的模板方法而实现的. 可以看出通过`AQS`可以很方便的构造一个`Lock`的实现. 接下来, 我们深入了解一下`AQS`中的底层源码实现, 了解其实现细节.

## AQS整体概况

### AQS之基本构造

AQS内部结构为:

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
...
private transient volatile Node head;
private transient volatile Node tail;
private volatile int state;
...
}
```

可以看出内部的结构非常简单:

- head: 等待队列的头部, 默认为正在执行的节点, 即拥有锁的节点.
- tail: 等待队列的尾部, 每次节点阻塞时默认在尾部添加一个节点.
- state: 需要同步的状态, AQS的控制核心, 通过自定义`State`使用来实现各种各样的自定义的Lock. 自定义Lock的核心.

AQS通过`State`来标明和控制锁的状态, 如上面的`Mutex`排他锁, 就是通过`State`的`0,1`标识当前是否有线程独占锁, 是否允许占据锁. 对于状态的获取和更新, AQS提供以下方法完成:

```java
// 获取状态值
protected final int getState() {
    return state;
}
// 设置状态值
protected final void setState(int newState) {
    state = newState;
}
// CAS设置状态值
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

其次, 在AQS等待队列中, 每个节点对象`Node`结构为:

```java
static final class Node {
    ...
    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;
    Node nextWaiter;
    ...
}
```

可以看出节点对象通过`prev`和`next`构成了一个简单的双指针链表结构, 并携带了额外的信息:

- waitStatus: 当前节点的状态, 取值有:

  - `CANCELLED(1)`: 当前节点已经被取消了, 有可能是超时和中断了, 该类节点不需要进行处理.
  - `SIGNAL(-1)`: 当前节点的后继节点被阻塞了(LockSupport.park), 当前节点运行完(或者取消)之后, 需要去唤醒后继节点.
  - `CONDITION(-2)`: 当前节点在condition的队列中, 如通过`condition`的`await`方法, 类似原生锁的`_WaitSet`.
  - `PROPAGATE(-3)`: 共享锁时需要向后传播, 只在共享锁中用到, 用于解决多线程同步释放锁产生冲突的场景.
  - `0`: 默认值, 其余情况, 非以上情况.
  - 可以看出, 当该值为正值(>0)的时候, 基本没有任何意义, 可以忽略, 只有在0及其以下时才具备具体的含义.

- thread: 当前节点的线程.

- nextWaiter: 存储在`condition`队列中的后继节点.

构成的基本结构如下:

![AQS-1](https://image.cjyong.com/OIP.jpeg)

### AQS方法实现

AQS方法分为两类, 一类是需要外部同步锁自己实现的:

- `isHeldExclusively`: 当前线程是否独占当前的锁对象.
- `tryAcquire(int arg)`: 尝试独占的获取锁,非阻塞(失败会直接返回), 返回boolea值获取结果.
- `tryRelease(int arg)`: 尝试释放独占锁, 返回boolean值的释放结果.
- `tryAcquireShared(int arg)`: 尝试获取共享锁, 返回int值. 约定如下:

  - 负值: 获取共享锁失败.
  - 0: 获取成功, 但是没有剩余的共享锁了, 后序的线程不需要进行唤醒.
  - 正值: 获取成功, 还剩下一些共享锁, 后序线程可以尝试获取(可能满足后序线程的要求).

- `tryReleaseShared(int arg)`: 尝试释放共享锁, 返回boolean值的释放结果.

不同的锁通过实现自定义的以上方法, 以实现不同的功能. 其底层主要是通过操纵`state`来实现不同的功能. 如上面的`Mutex`就通过自定义实现 `tryAcquire` 和 `tryRelease` 方法而实现的独占锁:

- `tryAcquire`: 尝试CAS设置state为1, 然后保存当前的线程.
- `tryRelease`: 因为只有获取了锁才会去释放, 不需要进行同步操作, 这时候只有当前线程在处理, 只需要将状态复原为0即可.
- 排他锁比较简单, 也就没有使用到`arg`参数.

第二类方法, 为内部实现的模板方法, 提供给外部调用:

- `acquire(int arg)`: 排他性获取锁, 不支持中断. 对应`Lock`中的`lock()`方法, 类似原生的`Synchronized`实现.
- `acquireInterruptibly(int arg)`: 排他性获取锁,支持中断. 对应`Lock`中的`lockInterruptibly()`方法. 是对`Synchronize`的拓展.
- `tryAcquireNanos(int arg, long nanosTimeout)`: 尝试一段时间内获取排他性锁, 支持中断. 对应`Lock`中的`tryLock(long time, TimeUnit unit)`方法.
- `acquireShared(int arg)`: 获取共享锁, 不支持中断.
- `acquireSharedInterruptibly(int arg)`: 共享的方式获取锁, 支持中断.
- `tryAcquireSharedNanos(int arg, long nanosTimeout)`: 尝试在一段时间内获取共享锁, 支持中断.
- 注意以上方法都已在内部默认实现, 直接调用即可. 可以看出此处对`Lock`的实现进行了拓展, 支持共享锁的存在, 这也就是信号量`Semaphore`和读写锁`ReentrantReadWriteLock`实现的基础.

这里可以看出`AQS`的整体方法设计, 通过抽象类把大量逻辑封装在内部的模板方法供使用者直接调用. 与此同时暴露出不同的`Hook`方法, 用于定制不同的锁需求. 接下来, 让我们走进内部`AQS`的世界.

## 独占锁解读

### 获取锁-acquire

先从简单的独占锁开始, 获取独占锁的默认方法是`acquire`. 注意该方法不可中断.

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

- 流程1: tryAcquire, 尝试直接获得锁(如果没有竞争,该处获取成功直接返回true结束)
- 流程2: 直接获取锁失败, 即: 存在竞争, 有人已经持有锁或者和其他线程竞争失败了. 这时候做了两件事

  - 流程2.1, addWaiter(Node.EXCLUSIVE): 将当前节点添加到同步队列的尾部.
  - 流程2.2, acquireQueued: 阻塞直到被唤醒(或者中断)获取锁, 返回中断状态.
  - 流程2.3, 获取锁之后, 如果在这段时间内被中断了, 就通过自我打断补上中断标识.
  - 为什么需要补上呢?, 因为我们获取当先线程是否中断状态的时候, 是通过`interrupted()`方法来获取的, 该方法会清空上一次的中断标识. 所以如果这段时间中断了, 会被获取的方法清空标识, 所以这里通过自我打断补上这个标识. 获取线程中断状态, 详情见`parkAndCheckInterrupt`.

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

**流程2.1, 将当前节点(当前线程)添加到同步队列的尾部**

- a. 将当前线程封装成一个排他性节点Node
- b. 尝试直接挂载到尾部(前提是已经初始化完成, 即`tail != null`).

  - b.1 将当前节点的prev指向尾节点.
  - 这一步需要在更新tail之前完成, 保证任何时刻tail的prev都不为null.
  - 如果放到`compareAndSetTail`之后的一起设置`prev`和`next`的话, 存在这种特殊情况, 即刚刚CAS设置完`tail`指针, 这时候线程切换, 没来得及设置尾节点的`prev`和`next`, 从而导致出现一个孤立的`tail`节点. 通过上面的方式, 至少可以保证`tail`的节点的`prev`指针不为null.
  - b.2 CAS设置当前节点为尾节点, 如果不存在冲突即成功, 更新旧尾节点的next指针.

- c. 如果出现了冲突或者未初始化(尾节点为null), 这会调用`enq`方法, 循环调用插入到尾部.

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

**流程2.1.c: 循环插入到尾部**

- enq的逻辑比较简单, for循环插入到队列尾部, 如果没有初始化则先初始化.
- 注意的一点是, 初始化设置的头节点是一个空的头节点, 并不是当前节点.
- 即如果只有一个线程进来是不会进来这里的(tryAcquire直接成功了), 也就不会初始化头节点和尾节点.
- 即如果是第二个线程进来(第一线程已经在使用锁), 这时候才会构造形成如下队列: 空头节点 <--> 当前节点.

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**流程2.2, 阻塞直至获得锁**

- a. 先获取当前节点的prev节点, 这里保证一定可以拿到(因为初始化的存在, 即使是第二个线程进来, 前面也有一个空头节点用以代表当前使用的线程).
- b. 如果前继节点就是头节点, 这时候尝试去获取一下锁.

  - b.1, 获取成功, 即头节点已经用完了锁, 这时候更新并存储头节点即可.
  - 同时清空旧头节点关联的prev和next引用, 使其回收(新头节点的prev在setHead中清空).

- c, 前继节点为非头节点或者获取失败(头节点还在用锁), 这时候做了两件事:

- d, shouldParkAfterFailedAcquire, 检查是否可以阻塞线程, 如果返回false, 即不允许直接进行第二次循环, 再次尝试获取锁, 这里一直自旋到允许阻塞线程.

- e, 如果允许阻塞线程, 调用parkAndCheckInterrupt阻塞线程.

- f, 如果在循环的过程中出现异常, 则会取消获取操作: cancelAcquire

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

**流程2.2.d, 判断当前情况是否可以阻塞线程**

- a. 如果前继节点的等待状态已经为SIGNAL(-1), 直接返回true.这时候前继节点释放锁之后会通知(unPark)后继节点, 这时候直接阻塞当前线程就好了.
- b. 如果前继节点的状态大于0, 说明前继节点已经取消(失效了), 这时候要更新prev节点.更新完前继节点, 返回false, 让当前线程再走一次外面的循环(尝试获取锁).
- c. 如果前继节点未取消, 则更新其状态为SIGNAL, 返回false, 在走一次外面的循环.
- 即可以阻塞的条件为: 前继节点的状态为SIGNAL(这样可以保证后继节点被稳定被唤醒), 出现前节点失效或者状态不为`SIGNAL`的时候都会更新并自旋一次.

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

**流程2.2.e, 调用LockSupport.park阻塞当前线程, 等待被唤醒.**

- 被前继节点唤醒, 即: park函数返回, 返回中断状态.
- 需要注意的是, 这里使用的是`interrupted()`方法, 会清空线程的中断状态.
- 需要注意的是, 当前外部的函数处理中, 中断状态只是记录, 并不会影响其循环获得锁继续执行.所以说 `acquire` 方法是不可中断的.
- 需要注意的是, `park`方法可能被中断返回, 这时候并没有被唤醒. 这时候如果出现中断, 外部函数 `acquire` 方法并不会退出, 只是记录中断状态, 继续循环获取锁. 同样验证了 `acquire` 是不可中断的.

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;
    node.thread = null;
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    Node predNext = pred.next;
    node.waitStatus = Node.CANCELLED;
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
              (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

**流程2.2.f, 取消获取锁**:

- a.如果节点为null, 直接返回.
- b.清空当前节点的线程.
- c. 记录当前节点的前继节点.
- d. 检查前继节点的等待状态, 如果大于0(即被取消了), 这跳过, 继续获取当前节点的前继节点. 保证前继节点一定是有效的.
- f. 获取当前有效前继节点的后继节点.
- g. 更新当前节点的状态为取消(1).
- h. 如果当前节点就是尾节点, 则更新尾节点为当前的前继节点.

  - 如果更新成功, 则设置前继节点的next指针为null(清空前继节点的指针), 处理完毕返回.

- i. 非尾节点或者设置尾节点失败(存在竞争, 又插入了一个新节点成为新的尾节点). 如果前继节点不是头节点, 并且前继节点的线程不为null, 并且前继节点的等待状态为SIGNAL(或者如果前继节点状态不为SIGNAL, 尝试将其设置为SIGNAL), 如果满足这个条件, 则说明前继节点为一个有效节点(不是初始化的空头节点, 且节点的等待状态已经被更新为SIGNAL), 这时候将前继节点的next更新为当前节点的next(跳过当前节点)

- 如果不满足上面条件就说明前继节点为(初始化)空头节点, 当前节点又取消了, 就去唤醒当前节点的后继节点.

- j. 最后设置当前节点的next指针指向自己, 协助回收当前节点(不会包含对其他节点的引用).

### 释放锁-release

常见排他锁的释放调用的模板方法为: `release`.

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

- 流程1: 尝试释放当前锁.

  - 如果释放成功, 即当前的头节点已经使用完了锁. 这时候检查一下是否有后继节点需要唤醒, 一般简单认为存在以下三种情况:

    - 如果从头到尾只有一个线程进来, 那么head是不会进行初始化的, 即为null.
    - 如果从头到尾只有两个线程同时进来并且产生了竞争, 那么第二个线程就会构建一个如下同步队列: `空节点 <--> 竞争节点(第二线程)`, 且空头节点的等待状态为SIGNAL(-1)
    - 如果有多个线程进来并发生了竞争, 则会构建如下队列: `空节点 <--> 竞争节点1(第二线程) <--> 竞争节点2(第三线程)...`

  - 1.1 如果头节点不为`null`且状态小于0, 则认为存在后继节点需要被唤醒, 即为后面两种情况.

  - 调用 `unparkSuccessor` 唤醒后继节点.

- 流程2: 如果释放失败直接返回false.

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

**流程1.1 唤醒后继节点**

- 设置当前节点的状态的等待状态为0, 尝试清空等待标识.
- 获取当前节点的后继节点

  - 如果当前节点为null或者状态大于0(即已经被取消了), 这时候就需要找下一个有效的节点进行唤醒.
  - 注意这里从尾节点向前遍历, 而不是从头节点向后遍历. 因为前面构建队列的时候, 保证了尾节点一定包含向前的指针(prev), 并不能保证一定有指针(next)指向尾节点. 所以从后往前遍历, 可以保证一定找得到.
  - 其次这里没有更新`head`节点, 更新`head`操作是在节点被唤醒之后, 在`acquireQueued`循环中更新的.

- 如果找到了有效的后继节点, 通过`LockSupport.unpark`唤醒即可.

### 可中断获取锁-acquireInterruptibly

可中断锁获取, 则是对应`Lock`中的`lockInterruptibly`, 即在原有的获取锁的基础上, 增加了支持中断的操作: 即线程阻塞在获取锁的过程中, 这时候通过`thread.interrupt()`打断获取的过程, 使其返回.

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

流程过程:

- 流程1: 和普通的获取逻辑相比, 多了一个中断判断, 如果这时候已经被中断了, 直接抛出异常返回即可, 不执行后面的获取锁逻辑了.
- 流程2: 首先尝试获取锁, 如果获取成功, 直接返回.
- 流程3: 如果获取失败, 尝试以支持中断的方式获取锁.

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**支持中断的方式获取锁**:

和普通的获取锁方法 `acquireQueued` 唯一的区别在于, 如果在自旋的过程中发生了中断, 这时候就直接抛出异常结束, 而不是简单的记录异常状态.

### 超时获取锁-tryAcquireNanos

`tryAcquireNanos` 在可中断的基础上, 添加了超时处理, 对应`Lock`中的`tryLock(long, TimeUnit)`.

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

逻辑和普通的中断方法`acquireInterruptibly`基本保持一致, 唯一的区别在于调用了`doAcquireNanos`方法来进行超时获取锁.

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

和中断版的获取锁`doAcquireInterruptibly`的区别在于, 每次自旋的时候需要判断一下截止时间, 如果到了截止时间, 就应该直接返回, 而不是继续等待获取锁.

- 其次需要注意的一点是, 当自旋到可以阻塞线程的时候, 会判断一下剩余时间是否足够大(大于spinForTimeoutThreshold), 如果很小的话, 就继续自旋, 时间足够大才会进行阻塞.

## 共享锁解读

### 获取共享锁-acquireShared

获取共享锁是对`Lock`的拓展, 支持共享的方式获取锁.

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

**获取共享锁**

- 流程1: 尝试直接获取共享锁, 如果返回值大于等于0则认为获取成功, 直接返回.

  - tryAcquireShared 返回约定:

    - 负数: 获取失败
    - 0: 获取成功, 但已经使用完共享锁了.
    - 正数: 获取成功, 并且还有剩余的共享锁, 后面的线程可以尝试再次获取.

- 流程2: 如果直接获取失败, 则会进行阻塞直到获取锁.

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**流程1.阻塞获取共享锁**

- a. 将当前节点共享的节点放入阻塞队列中的尾部.
- b. 如果当前节点的前继节点为头节点, 尝试获取锁.

  - b.1 如果获取锁成功,这时候还需要检查并向后传播.
  - b.2 如果这段时间呗中断了, 这时候还需要中断自己线程.
  - b.3 如果获取锁失败, 则需要自旋设置前继节点状态, 逻辑和之前保持一致.

- c. 如果在循环过程中出现了异常, 则会取消获取锁.

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

**流程b.1: 获取锁成功,检查并向后传播**:

- 首先更新当前节点为头节点.
- 如果满足一下任意条件:

  - `propagate > 0`, 共享锁还剩余锁没有用完, 还有剩余, 需要唤醒后序等待线程尝试获取共享锁.
  - `h == null`: 共享锁已经用完, 但是旧头节点已经为null, 已经丢失了.
  - `h.waitStatus < 0`: 共享锁已经用完, 旧头节点存在, 但是等待状态为负值(SIGNAL,PROPAGATE), 主要是PROPAGATE状态, 这时候需要向后传播,
  - `(h = head) == null`: 重新获取头节点, 即当前节点(前面设置更新了), 如果当前节点为null, 已经丢失了.
  - `h.waitStatus < 0`: 当前节点的状态小于零, 逻辑同上.

- 当不满足上面任何条件的时候需要进行向后传播唤醒等待线程尝试获取锁. 为啥需要额外的节点状态判断呢? 是为了解决多线程同步释放的问题:

  - `propagate == 0`, 即说明获取锁之后, 已经没有剩余的共享锁. 这是前提条件, 也是必要不充分条件. 因为`propagate`只是上一次节点获取锁的时候的返回值(快照), 不能保证在这段时间内没有别的线程又释放了锁, 如果释放了, 那这个值就已经过时了. 为了解决这个问题, `AQS` 引入了 `PROPAGATE` 状态来解决这个问题.

    - 第一种情况, 在`setHead`执行之前, 别的线程又释放了锁, 这时候修改的是旧头锁的等待状态为`PROPAGATE(-3)`. 所以当旧头锁的等待状态为`负数`的时候, 说明设置头之前, 又有线程释放了锁.
    - 第二种情况, 在`setHead`执行完之后, 别的线程又释放了锁, 这时候修改的是刚刚创建的头的等待状态, 所以还需要检查当前头的等待状态, 如果为负数, 则说明设置头之后, 又有别的线程释放了锁.

  - 因为可能出现上面两种情况(即同步释放锁的竞争情况), 所以每次判断不仅仅使用`propagate`, 还需要判断旧头的等待状态和当前头的等待状态. 线程释放锁设置头状态的操作在`doReleaseShared`函数中完成.

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                      !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

**传播唤醒等待的线程**

- 获取头节点.
- 如果头节点不为空且不等于尾节点, 即后面还有后继线程.
- 获取头节点的等待状态

  - 如果头结点的等待状态为`SIGNAL`, 需要唤醒后继节点.
  - 首先CAS更新节点的等待状态为0(正常状态), 如果更新失败, 则认为存在竞争, 继续第一步的循环.
  - 如果更新成功, 唤醒后继节点.
  - 如果头节点状态为0, 则设置为`PROPAGATE`状态, 这样可以保证在多线程竞争的时候. 如果设置失败, 则认为出现竞争, 重复第一步的循环. 这一步在竞争状态下可以设置头状态为 `PROPAGATE`, 通知其他线程存在释放的锁的情况.

- 如果这段时间头节点发生了变化, 继续循环唤醒头节点的后序节点.

### 共享锁的释放-releaseShared

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

释放共享锁的逻辑比较简单, 首先尝试直接释放, 释放成功的话, 就需要唤醒后面等待的线程.

## 总结

`AQS`底层源码有趣而且非常严谨, 推荐阅读`Doug Lea`关于`AQS`的论文, 了解`Doug Lea`大师在设计AQS的一些想法和思路. 通过了解`AQS`的底层设计, 可以帮助我们更好的去了解和使用`java.util.concurrent`包下面各类同步工具. 这是`JAVA`做的非常好的一点, 当别的语言还在自己造轮子的时候, `Java`已经有大师封装好了一系列的同步工具供我们使用. 由于篇幅有限, 有些地方并没有深入解读, 后续有时间会持续更新. 思之,慎之.

## 参考文档

- [Doug Lea的AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [Live In A Dream关于PROPAGATE解读](https://www.cnblogs.com/micrari/p/6937995.html)
- [并发编程网相关文章](http://ifeve.com/introduce-abstractqueuedsynchronizer/)
- [LockSupport.park的JVM实现](https://blog.csdn.net/hengyunabc/article/details/28126139)
