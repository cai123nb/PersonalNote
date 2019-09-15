# Java Lock 详解

## 处理器原子性实现

首先处理器会自动保证基本内存操作的原子性, 处理器保证从系统内存当中读取或者写入一个字节是原子的, 意思是当一个处理器读取一个字节时, 其他处理器不能访问这个字节的内存地址. 奔腾 6 和最新的处理器能够自动保证`单处理器`对同一个缓存行进行 16/32/64 位操作时原子的, 但是复杂的内存操作处理器不能保证其原子性, 如跨总线宽度, 跨多个缓存行, 跨页表访问. 但是处理器提供总线锁定和缓存锁定来保证复杂内存操作的原子性.

> 缓存行(Cache Line): 缓存的最小操作单位

- 总线锁: 处理器提供一个 LOCK#信号, 当一个处理器在总线上输出此信号时, 其他处理器的请求将被堵塞住(相当于把 CPU 和内存之间的通信锁住了), 那么该处理器可以独占使用共享内存.
- 缓存锁: 使用总线锁的时候, 其他处理器不仅不能操作共享内存的数据, 还不能操作其他内存地址的数据(由于与内存之间的通信被锁住了), 开销非常的大, 缓存锁为了优化总线锁带来的巨大开销. 频繁使用的内存会被缓存到处理器的 L1, L2 和 L3 高速缓存中去, 那么原子操作就可以直接在处理器内部缓存中去, 而不需要在内存中进行总线锁. 在奔腾 6 等最新处理器通过使用`缓存锁定`的方式来实现复杂的原子性. `缓存锁定`: 如果内存中地址被放置到缓存中, 并进行了锁定, 那么别的处理器不能再次缓存该地址, 并且缓存一致性原则将会阻止该缓存被两个以上处理器同时修改.

> 以下两种情况不会使用缓存锁定:
>
> 1. 操作的数据不能缓存在处理器内部或者数据跨多个缓存行.
> 2. 处理器不支持缓存锁定. 对于一些旧版本处理器, 没有支持缓存锁定, 使用的还是总线锁.

以上两个机制我们可以通过 Inter 处理器提供了很多 LOCK 前缀的指令来实现。比如位测试和修改指令 BTS，BTR，BTC，交换指令 XADD，CMPXCHG 和其他一些操作数和逻辑指令，比如 ADD（加），OR（或）等，被这些指令操作的内存区域就会加锁，导致其他处理器不能同时访问它。

## 原子性的实现方法

要保证原子性, 最常见的的方法就是加锁如 Java 中 Synchronied 或 Lock, 还有一种方式就是 CAS.

## CAS

CAS: Compare And Swap/Set
CAS: 有三个操作数, 内存值 V, 旧的预期值 A, 要更新的新值 B. 仅当预期值 A 和内存值 V 相同时, 才允许将 V 值修改为 B 值, 不然什么都不做.

这里以 java.util.concurrent.atomic 包下: AtomicInteger 为例
要实现 i++的原子操作:

```java
//First use volatile modified the value
private volatile int value;

//Second use while loop to make sure it work
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}

//Method track
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
//Method track
private static final Unsafe unsafe = Unsafe.getUnsafe();

//Method track
public final class Unsafe {
...
    private static native void registerNatives();

    private Unsafe() {
    }

    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
...
}
```

我们可以看出最终 Java 通过 CompareAndSet 进行原子操作的保证, 而原子操作的实现通过 JNI 本地方法进行实现的.

但是 CAS 会出现一个问题, 就是常见的 ABA 问题, 假设存在一个原子数据 A, 线程 T1 要将 A 改写成 C, 但是由于这个操作有点慢, 耗时较久, 就在线程 T1 修改的时候, 这时候线程 T2 将 A 替换了成了 B, 然后将 B 替换成了 A, T2 结束了. 这时候 T1 操作完成, 准备写入内存, 进行判断和开始预判值 A 是否相同, 发现没有改变, 于是就写入进去. 但这是不对的, 因为在这段时间 A 的值发生了改变. 解决方法也较为简单, 通过添加版本号进行控制. A1 -> B1 -> A2, 这时候 T1 线程写入的时候就会发现问题. Java 通过 AtomicStampedReference 进行控制.

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABA {
    private static AtomicInteger atomicInt = new AtomicInteger(100);
    private static AtomicStampedReference atomicStampedRef = new AtomicStampedReference(100, 0);

    public static void main(String[] args) throws InterruptedException {
        Thread intT1 = new Thread(new Runnable() {
            @Override
            public void run() {
                atomicInt.compareAndSet(100, 101);
                atomicInt.compareAndSet(101, 100);
            }
        });

        Thread intT2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                }
                boolean c3 = atomicInt.compareAndSet(100, 101);
                System.out.println(c3); // true
            }
        });

        intT1.start();
        intT2.start();
        intT1.join();
        intT2.join();

        Thread refT1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                }
                atomicStampedRef.compareAndSet(100, 101, atomicStampedRef.getStamp(), atomicStampedRef.getStamp() + 1);
                atomicStampedRef.compareAndSet(101, 100, atomicStampedRef.getStamp(), atomicStampedRef.getStamp() + 1);
            }
        });

        Thread refT2 = new Thread(new Runnable() {
            @Override
            public void run() {
                int stamp = atomicStampedRef.getStamp();
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                }
                boolean c3 = atomicStampedRef.compareAndSet(100, 101, stamp, stamp + 1);
                System.out.println(c3); // false
            }
        });

        refT1.start();
        refT2.start();
    }
}
```

## Synchronized

Synchronized 是 Java 中解决并发问题的一种最常用的方法, 也是最简单的一种方法. Synchronized 的作用主要有三个：

1. 确保线程互斥的访问同步代码
2. 保证共享变量的修改能够及时可见
3. 有效解决重排序问题

Synchronized 总共有三种用法：

1. 修饰普通方法
2. 修饰静态方法
3. 修饰代码块

- 没有同步情况

```java
public class SynchronizedTest {
    public void method1(){
        System.out.println("Method 1 start");
        try {
            System.out.println("Method 1 execute");
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public void method2(){
        System.out.println("Method 2 start");
        try {
            System.out.println("Method 2 execute");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
/*
执行结果如下，线程1和线程2同时进入执行状态，线程2执行速度比线程1快，所以线程2先执行完成，这个过程中线程1和线程2是同时执行的.
Method 1 start
Method 1 execute
Method 2 start
Method 2 execute
Method 2 end
Method 1 end
*/
```

- 普通方法同步

```java
public class SynchronizedTest {
    public synchronized void method1(){
        System.out.println("Method 1 start");
        try {
            System.out.println("Method 1 execute");
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public synchronized void method2(){
        System.out.println("Method 2 start");
        try {
            System.out.println("Method 2 execute");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
/*
执行结果如下，跟代码段一比较，可以很明显的看出，线程2需要等待线程1的method1执行完成才能开始执行method2方法。
Method 1 start
Method 1 execute
Method 1 end
Method 2 start
Method 2 execute
Method 2 end
*/
```

- 静态方法同步

```java
public class SynchronizedTest {
     public static synchronized void method1(){
         System.out.println("Method 1 start");
         try {
             System.out.println("Method 1 execute");
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         System.out.println("Method 1 end");
     }

     public static synchronized void method2(){
         System.out.println("Method 2 start");
         try {
             System.out.println("Method 2 execute");
             Thread.sleep(1000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         System.out.println("Method 2 end");
     }

     public static void main(String[] args) {
         final SynchronizedTest test = new SynchronizedTest();
         final SynchronizedTest test2 = new SynchronizedTest();

         new Thread(new Runnable() {
             @Override
             public void run() {
                 test.method1();
             }
         }).start();

         new Thread(new Runnable() {
             @Override
             public void run() {
                 test2.method2();
             }
         }).start();
     }
 }
 /*
 执行结果如下，对静态方法的同步本质上是对类的同步（静态方法本质上是属于类的方法，而不是对象上的方法），所以即使test和test2属于不同的对象，但是它们都属于SynchronizedTest类的实例，所以也只能顺序的执行method1和method2，不能并发执行。
Method 1 start
Method 1 execute
Method 1 end
Method 2 start
Method 2 execute
Method 2 end
 */
```

- 代码块同步

```java
public class SynchronizedTest {
    public void method1(){
        System.out.println("Method 1 start");
        try {
            synchronized (this) {
                System.out.println("Method 1 execute");
                Thread.sleep(3000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public void method2(){
        System.out.println("Method 2 start");
        try {
            synchronized (this) {
                System.out.println("Method 2 execute");
                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
/*
执行结果如下，虽然线程1和线程2都进入了对应的方法开始执行，但是线程2在进入同步块之前，需要等待线程1中同步块执行完成。
Method 1 start
Method 1 execute
Method 2 start
Method 1 end
Method 2 execute
Method 2 end
*/
```

**原理**:

```java
package com.paddx.test.concurrent;

public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("Method 1 start");
        }
    }
}
```

![show](https://image.cjyong.com/blog/r14.jpg)

**monitorenter** ：

Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
• If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
• If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
• If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

这段话的大概意思为：

每个对象有一个监视器锁（monitor）。当 monitor 被占用时就会处于锁定状态，线程执行 monitorenter 指令时尝试获取 monitor 的所有权，过程如下：

1、如果 monitor 的进入数为 0，则该线程进入 monitor，然后将进入数设置为 1，该线程即为 monitor 的所有者。

2、如果线程已经占有该 monitor，只是重新进入，则进入 monitor 的进入数加 1.

3.如果其他线程已经占用了 monitor，则该线程进入阻塞状态，直到 monitor 的进入数为 0，再重新尝试获取 monitor 的所有权。

**monitorexit**：

The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

这段话的大概意思为：

执行 monitorexit 的线程必须是 objectref 所对应的 monitor 的所有者。

指令执行时，monitor 的进入数减 1，如果减 1 后进入数为 0，那线程退出 monitor，不再是这个 monitor 的所有者。其他被这个 monitor 阻塞的线程可以尝试去获取这个 monitor 的所有权。

通过这两段描述，我们应该能很清楚的看出 Synchronized 的实现原理，Synchronized 的语义底层是通过一个 monitor 的对象来完成，其实 wait/notify 等方法也依赖于 monitor 对象，这就是为什么只有在同步的块或者方法中才能调用 wait/notify 等方法，否则会抛出 java.lang.IllegalMonitorStateException 的异常的原因。

```java
package com.paddx.test.concurrent;

public class SynchronizedMethod {
    public synchronized void method() {
        System.out.println("Hello World!");
    }
}
```

![show](https://image.cjyong.com/blog/r15.jpg)

从反编译的结果来看，方法的同步并没有通过指令 monitorenter 和 monitorexit 来完成（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了 ACC_SYNCHRONIZED 标示符。JVM 就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取 monitor，获取成功之后才能执行方法体，方法执行完后再释放 monitor。在方法执行期间，其他任何线程都无法再获得同一个 monitor 对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

**解释**:
有了对 Synchronized 原理的认识，再来看上面的程序就可以迎刃而解了:

1. 代码段 2 结果：
   　　虽然 method1 和 method2 是不同的方法，但是这两个方法都进行了同步，并且是通过同一个对象去调用的，所以调用之前都需要先去竞争同一个对象上的锁（monitor），也就只能互斥的获取到锁，因此，method1 和 method2 只能顺序的执行。
2. 代码段 3 结果：
   　　虽然 test 和 test2 属于不同对象，但是 test 和 test2 属于同一个类的不同实例，由于 method1 和 method2 都属于静态同步方法，所以调用的时候需要获取同一个类上 monitor（每个类只对应一个 class 对象），所以也只能顺序的执行。
3. 代码段 4 结果：
   　　对于代码块的同步实质上需要获取 Synchronized 关键字后面括号中对象的 monitor，由于这段代码中括号的内容都是 this，而 method1 和 method2 又是通过同一的对象去调用的，所以进入同步块之前需要去竞争同一个对象上的锁，因此只能顺序执行同步块。

## 重量级锁

我们应该知道,Synchronized 是通过对象内部的一个叫做监视器锁(monitor)来实现的. 但是监视器锁本质又是依赖于底层的操作系统的 Mutex Lock 来实现的. 而操作系统实现线程之间的切换这就需要从用户态转换到核心态, 这个成本非常高, 状态之间的转换需要相对比较长的时间, 这就是为什么 Synchronized 效率低的原因.因此, 这种依赖于操作系统 Mutex Lock 所实现的锁我们称之为`重量级锁`. JDK 中对 Synchronized 做的种种优化, 其核心都是为了减少这种重量级锁的使用. JDK1.6 以后, 为了减少获得锁和释放锁所带来的性能消耗, 提高性能, 引入了`轻量级锁`和`偏向锁`.

## 轻量级锁

锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁（但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级）。JDK 1.6 中默认是开启偏向锁和轻量级锁的，我们也可以通过-XX:-UseBiasedLocking 来禁用偏向锁。锁的状态保存在对象的头文件中.

![show](https://image.cjyong.com/blog/r5.jpg)

`轻量级`是相对于使用操作系统互斥量来实现的传统锁而言的.但是, 首先需要强调一点的是, 轻量级锁并不是用来代替重量级锁的, 它的本意是在没有多线程竞争的前提下, 减少传统的重量级锁使用产生的性能消耗. 在解释轻量级锁的执行过程之前, 先明白一点, 轻量级锁所适应的场景是**线程交替执行同步块的情况**,如果存在同一时间访问同一锁的情况,就会导致轻量级锁膨胀为重量级锁. 轻量级锁的思想为: 假设大多数情况下线程进入同步块时并没有别的进程进行抢夺.

- 轻量级锁的加锁过程

1. 在代码进入同步块的时候, 如果同步对象锁状态为无锁状态(锁标志位为`01`状态, 是否为偏向锁为`0`), 虚拟机首先将在当前线程的栈帧中建立一个名为锁记录(Lock Record)的空间, 用于存储锁对象目前的 Mark Word 的拷贝, 官方称之为 Displaced Mark Word.这时候线程堆栈与对象头的状态如图所示.
   ![show](https://image.cjyong.com/blog/r11.jpg)

2. 拷贝对象头中的 Mark Word 复制到锁记录中.
3. 拷贝成功后, 虚拟机将使用 CAS 操作尝试将对象的 Mark Word 更新为指向 Lock Record 的指针, 并将 Lock record 里的 owner 指针指向 object mark word. 如果更新成功, 则执行步骤(3), 否则执行步骤(4).
4. 如果这个更新动作成功了, 那么这个线程就拥有了该对象的锁, 并且对象 Mark Word 的锁标志位设置为`00`，即表示此对象处于轻量级锁定状态, 这时候线程堆栈与对象头的状态如图所示.
   ![show](https://image.cjyong.com/blog/r13.jpg)

5. 如果这个更新操作失败了, 虚拟机首先会检查对象的 Mark Word 是否指向当前线程的栈帧, 如果是就说明当前线程已经拥有了这个对象的锁, 那就可以直接进入同步块继续执行. 否则说明多个线程竞争锁, 轻量级锁就要膨胀为重量级锁, 锁标志的状态值变为`10`, Mark Word 中存储的就是指向重量级锁(互斥量)的指针, 后面等待锁的线程也要进入阻塞状态. 而当前线程便尝试使用自旋来获取锁, 自旋就是为了不让线程阻塞, 而采用循环去获取锁的过程.

轻量级锁的解锁过程：

1. 通过 CAS 操作尝试把线程中复制的 Displaced Mark Word 对象替换当前的 Mark Word.
2. 如果替换成功，整个同步过程就完成了.
3. 如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程.

## 偏向锁

引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径, 因为轻量级锁的获取及释放依赖多次 CAS 原子指令, 而偏向锁只需要在置换 ThreadID 的时候依赖一次 CAS 原子指令(由于一旦出现多线程竞争的情况就必须撤销偏向锁, 所以偏向锁的撤销操作的性能损耗必须小于节省下来的 CAS 原子指令的性能消耗). 上面说过, 轻量级锁是为了在线程交替执行同步块时提高性能, 而偏向锁则是在只有一个线程执行同步块时进一步提高性能.

偏向锁获取过程：

1. 访问 Mark Word 中偏向锁的标识是否设置成 1, 锁标志位是否为 01——确认为可偏向状态.
2. 如果为可偏向状态，则测试线程 ID 是否指向当前线程, 如果是, 进入步骤(5), 否则进入步骤(3).
3. 如果线程 ID 并未指向当前线程，则通过 CAS 操作竞争锁. 如果竞争成功, 则将 Mark Word 中线程 ID 设置为当前线程 ID, 然后执行(5); 如果竞争失败, 执行(4).
4. 如果 CAS 获取偏向锁失败, 则表示有竞争. 当到达全局安全点(safepoint)时获得偏向锁的线程被挂起, 偏向锁升级为轻量级锁, 然后被阻塞在安全点的线程继续往下执行同步代码.
5. 执行同步代码.

偏向锁的释放：
偏向锁的撤销在上述第四步骤中有提到. 偏向锁只有遇到其他线程尝试竞争偏向锁时, 持有偏向锁的线程才会释放锁, 线程不会主动去释放偏向锁. 偏向锁的撤销, 需要等待全局安全点(在这个时间点上没有字节码正在执行), 它会首先暂停拥有偏向锁的线程, 判断锁对象是否处于被锁定状态, 撤销偏向锁后恢复到未锁定(标志位为`01`)或轻量级锁(标志位为`00`)的状态.

重量级锁、轻量级锁和偏向锁之间转换

![show](https://image.cjyong.com/blog/r10.jpg)

## 其他优化

1. 适应性自旋(Adaptive Spinning): 从轻量级锁获取的流程中我们知道, 当线程在获取轻量级锁的过程中执行 CAS 操作失败时, 是要通过自旋来获取重量级锁的. 问题在于, 自旋是需要消耗 CPU 的, 如果一直获取不到锁的话, 那该线程就一直处在自旋状态, 白白浪费 CPU 资源. 解决这个问题最简单的办法就是指定自旋的次数, 例如让其循环 10 次, 如果还没获取到锁就进入阻塞状态. 但是 JDK 采用了更聪明的方式——适应性自旋, 简单来说就是线程如果自旋成功了, 则下次自旋的次数会更多, 如果自旋失败了, 则自旋的次数就会减少.

2. 锁粗化(Lock Coarsening)：锁粗化的概念应该比较好理解, 就是将多次连接在一起的加锁、解锁操作合并为一次, 将多个连续的锁扩展成一个范围更大的锁.举个例子：

   ```java
   public class StringBufferTest {
       StringBuffer stringBuffer = new StringBuffer();

       public void append(){
           stringBuffer.append("a");
           stringBuffer.append("b");
           stringBuffer.append("c");
       }
   }
   ```

   - 这里每次调用 stringBuffer.append 方法都需要加锁和解锁, 如果虚拟机检测到有一系列连串的对同一个对象加锁和解锁操作, 就会将其合并成一次范围更大的加锁和解锁操作, 即在第一次 append 方法时进行加锁, 最后一次 append 方法结束后进行解锁.

3. 锁消除(Lock Elimination): 锁消除即删除不必要的加锁操作. 根据代码逃逸技术, 如果判断到一段代码中, 堆上的数据不会逃逸出当前线程. 那么可以认为这段代码是线程安全的, 不必要加锁. 看下面这段程序：

```java
public class SynchronizedTest02 {

    public static void main(String[] args) {
        SynchronizedTest02 test02 = new SynchronizedTest02();
        //启动预热
        for (int i = 0; i < 10000; i++) {
            i++;
        }
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            test02.append("abc", "def");
        }
        System.out.println("Time=" + (System.currentTimeMillis() - start));
    }

    public void append(String str1, String str2) {
        StringBuffer sb = new StringBuffer();
        sb.append(str1).append(str2);
    }
}
```

虽然 StringBuffer 的 append 是一个同步方法, 但是这段程序中的 StringBuffer 属于一个局部变量, 并且不会从该方法中逃逸出去, 所以其实这过程是线程安全的, 可以将锁消除.

## 总结

本文重点介绍了 JDk 中采用轻量级锁和偏向锁等对 Synchronized 的优化, 但是这两种锁也不是完全没缺点的, 比如竞争比较激烈的时候, 不但无法提升效率, 反而会降低效率, 因为多了一个锁升级的过程, 这个时候就需要通过-XX:-UseBiasedLocking 来禁用偏向锁. 下面是这几种锁的对比：

![show](https://image.cjyong.com/blog/r8.jpg)

## 参考文档

[liuxiaopeng 并发博客](http://www.cnblogs.com/paddix/p/5374810.html)
