# Synchronized

![header](https://image.cjyong.com/syn-header.png)

Synchronized是Java原生的同步关键字, 通过该关键字保证了在多线程访问时数据的原子性. 这是JVM实现的内置锁, 锁的获取和释放都是由JVM自己进行控制, 因此JVM也对该synchronized做了很多优化.

![Synchronized](https://image.cjyong.com/Synchronized.png)

假设获取锁对象为c, 其对象头的MarkWord为mw, 所属的类对象为C, 类对象中包含一个类似的MarkWord的对象头(prototype_header)为MW.

## 偏向锁补充

获取偏向锁的流程:

1. 判断c是否开启了偏向锁(mw末三位为101则认为开启了), 如果没有开启(不是101), 则走轻量锁的流程. 开启了, 则走下面流程2:
2. 如果mw中线程ID和当前的线程ID一样, 且mw中的epoch和MW的epoch一致, 且mw偏向锁状态和MW的偏向锁状态一致, 同时满足三个条件, 即认为当前线程重复进入, 新建一个LockRecord后, 进入执行同步方法. 如果不满足任一条件, 则走下面流程3:

  1. 判断是否满足三个条件时, 采用的方法是, 将当前线程ID替换MW中的线程ID形成新的MW, 然后和对象里面的mw进行位比较(异或操作), 如果全部相等(即异或结果为0), 即为三种条件都满足.

3. 如果偏向锁状态不一致, 则认为外部类已经禁用了偏向锁的使用, 这时候需要重置所有的偏向锁, 走轻量锁的逻辑.

4. 如果mw中的epoch不一致, 则认为是当前的偏向锁已经吊销, 直接使用当前mw中的线程ID来CAS替换成当前线程ID即可.

5. 如果mw中的epoch一致, 但是线程ID不一致, 则表明当前对象已经偏向另一个锁. 这时候尝试使用CAS用0当做初始值进行尝试, 如果是匿名偏向状态, 则会设置成功, 否则就会走轻量锁的流程.

为什么mw中的epoch和锁状态有可能和MW中的不一致?

每次撤销偏向锁升级成轻量锁的时候都会在C中进行计数, 如果超过了一个20(默认阈值), 则认为这个对象偏向存在问题, 就会进行批量重偏向(bulk rebias)操作. 如果超过了40(默认阈值), 则认为这个对象竞争很激烈不适合偏向锁, 则会进行批量撤销(bulk revoke)操作.

- 批量重偏向操作:

  - 将MW中的epoch值加一, 这样其他c实例, 就算已经偏向了(存在线程ID), 也允许通过CAS重新偏向当前线程, 而不会撤销锁.
  - 将现有的正在加锁的c的mw中的epoch加一, 和MW中的epoch保持一致. 不会影响到还在用偏向锁的对象.
  - 所以发生批量重偏向的时候, 会导致mw和MW中的epoch值不一致, 实质为mw中的epoch已经过期了(吊销了), c还可以再偏向一次.

- 批量撤销操作: 将MW中的锁状态(末三位)设置为001, 即为禁用偏向锁状态.

  - 这时候对象获取c的锁的时候就会直接走轻量锁, 不会走偏向锁了.
  - 对于已有的偏向锁进行撤销, 升级为轻量锁.

为什么对偏向锁对象执行hashCode()函数会让偏向锁升级?

因为偏向锁是存储在mw中, 之前无锁的时候存放的就是hashCode值, 现在偏向锁占了位置, 就没办法存放哈希Code了. 这时候就需要升级为轻量锁或者重量锁, 来存放hashCode.

## 轻量锁

轻量锁是在当前线程的栈中建立一个LockRecord, 里面存储了被替换的mw, 当多次重入的时候, Stack中存储了多个LockRecord, 但是后序LockRecord中的mw为null.

当一个线程释放所有轻量锁的时候, 会把被替换的mwCAS替换到对象头c中.

为什么CAS更新LockRecord时为什么有可能存在冲突?

CAS是构造一个无锁的mw(标志位为01)来作为初始内存值, 进行替换成指向当前LockRecord的指针. 如果没锁就可以正常替换掉, 获得轻量锁.只要是对象有锁, 那么对象头mw就已经被替换掉了, 就会出现问题. 这时候在判断是否是当前线程, 如果是的, 那就将LockRecord中的mw设置为null, 存起来即可. 如果不是, 说明出现了冲突, 需要升级为重量锁.

为什么偏向锁冲突的时候是升级为轻量锁而不是重量锁?

因为偏向锁冲突的时候, 并不知道是否有其他线程正在使用这个偏向锁, 这时候优先升级成轻量锁就可以, 如果没有使用, 升级轻量锁就可以完成了. 如果有人在使用, 那就需要升级为重量锁.

## Synchronized和ReentrantLock的区别

- Synchronized是JVM层次的锁实现，ReentrantLock是JDK层次的锁实现；
- Synchronized的锁状态是无法在代码中直接判断的，但是ReentrantLock可以通过ReentrantLock#isLocked判断；
- Synchronized是非公平锁，ReentrantLock是可以是公平也可以是非公平的；
- Synchronized是不可以被中断的，而ReentrantLock#lockInterruptibly方法是可以被中断的. 在发生异常时Synchronized会自动释放锁（由javac编译时自动实现），而ReentrantLock需要开发者在finally块中显式释放锁；
- ReentrantLock获取锁的形式有多种：如立即返回是否成功的tryLock(),以及等待指定时长的获取，更加灵活；
- Synchronized在特定的情况下对于已经在等待的线程是后来的线程先获得锁（上文有说），而ReentrantLock对于已经在等待的线程一定是先来的线程先获得锁；

## 参考文档

- [偏向锁](https://github.com/farmerjohngit/myblog/issues/13)
- [轻量锁](https://github.com/farmerjohngit/myblog/issues/14)
- [重量锁](https://github.com/farmerjohngit/myblog/issues/15)
