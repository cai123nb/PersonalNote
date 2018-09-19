# Object源码学习
JDK1.8:

```java
package java.lang;

//位于所有类的继承的顶端,即每个类(包括数组)都有Object作为父类, 
//所有的类都实现了这个类的所有方法. 
public class Object {
	//注册本地方法,保证首先被执行
	private static native void registerNatives();
	static {
		registerNatives();
	}
	
	//返回当前正在运行的类名称(如在类中调用:super.getClass()返回当前运行的类,而不是父类)
	public final native Class<?> getClass();
	
	//返回对象的哈希码,这个方法主要用于支持HashMap, 该方法保证相同的对象肯定会返回一样的
	//值, 对于不同的对象理论上是不会相同的. 可以用于比较对象, 优先比较哈希码,如果哈希码不同的话
	//肯定不同, 就不必进行equal()判断, 否则再进行equal()比较, 可以提升性能.HashMap中就曾用到.
	public native int hashCode();
	
	//判断两个对象是否相等,需要保证自反性(x.equals(x)应该为true), 对称性(x.equals(y)为true, 那么y.equals(x)为true)
	//传递性(如果x.equals(y)为true, y.equals(x)), 一致性(只要两个对象没有修改, 就应该一直返回同一个值).
	//equals方法应该是最权威的辨别方法, 如果两个引用指向同一个对象应该返回true. 另外推荐覆盖hashCode()方法, 保证
	//同一个对象拥有相同的hashCode()方法(有些方法先调用hashCode函数,如果相同再调用equal方法以提高性能), 默认的equal
	//方法直接比较引用地址.
	public boolean equals(Object obj) {
		return (this == obj);
	}
	
	//创建和返回一个对象的拷贝, 拷贝的含义取决于对象具体的类, 一般来说: x.clone() != x, x.clone().getClass() == x.
	//getClass(). 满足: x.clone().equals(x). 当然这些都不是必要的. 按照惯例, 所有的对象都应该调用super.clone()函数(就是
	//重载了该方法(Cloneable)的方法内部都应该调用super.clone())这样可以保证x.clone().getClass() == x.getClass(). 也就是传统意义上的
	//深拷贝, 深拷贝还有一个前提条件就是所有的成员函数都应该实现了Cloneable接口.
	//一般来说clone()返回的对象应该是和原对象是分离的(修改原对象是不会影响到clone()得到的对象), 为了实现这个功能, 在该函数里都应该
	//调用super.clone()函数然后返回. 如果一个类没有实现Cloneable接口, 调用这个方法会抛出一个CloneNotSupportedException.
	//所有的数组都应该实现这个接口, 返回一个数组的克隆. 否则，此方法创建此对象的类的新实例，并使用该对象的相应字段的内容初始化其所有字段，
	//就像通过赋值一样; 这些字段的内容本身不会被克隆。 因此，该方法执行该对象的“浅拷贝”，而不是“深拷贝”操作。
	//Object本身没有实现Cloneable接口, 如果调用Object本身调用该方法,将会在运行时抛出异常. 如果子类不想支持clone()方法, 可以重载该方法
	//在方法中抛出异常即可.
	protected native Object clone() throws CloneNotSupportedException;
	
	//返回对象的描述文字, 描述应该简洁明了, 容易阅读. 推荐所有的子类重写该方法, 对于Object对象来说返回Object的名字和十六进制的哈希码.
	public String toString() {
		return getClass().getName() + "@" + Integer.toHexString(hashCode());
	}
	
	//唤醒一个正在等待该对象监视器(synchronize)的线程, 如果有很多线程在等待, 则随机唤醒一个, 这在设计程序的时候需要小心. 唤醒的线程不会
	//立马进入, 直到当前线程释放该对象的锁. 调用该方法的对象应该是这个监视器的主人,一个线程要变成这个监视器的主人可以通过以下方法获得:
	//同步实例方法(synchronize void methodxx{}),同步块语句(synchronize(this){}),同步静态方法(synchronized static void method(), 这是给类加锁).
	//同一时间只能有一个线程拥有这个监视器的控制权.
	public final native void notify();
	
	//唤醒所有在等待该对象监视器的线程, 每有一个等待的线程就调用一次notify()方法. 唤醒的线程将进行竞争获得这个监视器的控制权. 调用这个方法
	//应该是当前监视器的主人.
	public final native void notifyAll();
	
	//让当前当前线程保持等待状态（休眠状态）直到别的线程调用notify(),notifyAll()方法,或者等待到指定的时间.
	//当前线程必须拥有该对象的监视器.当前执行线程(后面成为T)把自己放入到该对象的等待集中,并释放所有拥有关于
	//该对象的同步锁. 线程T不再被线程调度管控并保持休眠状态, 直到以下情况发生:
	//别的线程(拥有这个对象的监视器)调用这个对象的notify()方法,并随机的选择了这个线程.
	//别的线程调用这个对象的notifyAll()方法.
	//别的线程调用这个线程T的interrupt方法.
	//等待的时间已经过去.
	//当上述情况发生之后, 线程T将自己从对象的等待集中移除, 重新参与线程调度. 并与其他线程开始竞争该对象的
	//控制权. 一旦它获取了对象的控制权, 所有的同步声明和状态都会被恢复到调用wait()方法之前的情况, 这时候
	//从wait()方法进行返回. 也就是说, wait()方法调用前和返回之后, 线程对这个对象的状态应该一样.
	//然而线程有可能醒来而不经历上述4种情况, 虽然实际上概率非常小, 但是还行需要预防. 推荐将wait方法置于一个
	//循环当中,以防这种情况.(synchronized(obj){while(condition not fit){obj.wait(timeout}}...}.
	//如果线程T在wait()的时候, 被别的线程打断(Interrupt), 那么会产生一个InterruptedException, 但是这个异常
	//会立马抛出,而是等待当前线程恢复正常状态的时候(wait()方法调用完毕,重新拥有对象锁)进行抛出(清楚中断表示).
	//注意只有在线程拥有对象锁的时候,才可以调用这个方法,参照notify()方法知道如何获取对象的锁.
	public final native void wait(long timeout) throws InterruptedException;
	
	//类似wait()方法, 但是提供一个更加精准的时间控制(以毫秒为单位, 1000000 * timeout + nanos).
	public final void wait(long timeout, int nanos) throws InterruptedException {
		if (timeout < 0) {
			throw new IllegalArgumentException("timeout value is negative");
		}

		if (nanos < 0 || nanos > 999999) {
			throw new IllegalArgumentException(
								"nanosecond timeout value out of range");
		}

		if (nanos > 0) {
			timeout++;
		}

		wait(timeout);
	}
	
	//类似wait(timeout)方法, 但是不设置等待时长. 参照wait(timeout)方法.
	public final void wait() throws InterruptedException {
		wait(0);
	}
	
	//当垃圾收集器判断当前没有任何到达该对象的路径(可达性分析)时,垃圾收集器会调用该方法. 一个子类重载该方法
	//来释放系统资源或者来执行其他的清除工作.需要注意的是finalize()方法可以执行任何动作, 包括自救(让别的线程重新
	//调用自己). 但是一般的目的就是执行一些清除操作. Object中finalize方法不执行任何特殊操作, 只是单纯的返回.
	//子类可能重载该操作. JVM不保证线程会调用该方法, 但是保证当对线程调用该方法的时候,肯定不拥有任何可见性的同步
	//锁(别的对象). 如果在finalize()方法执行过程中, 出现了异常, 暂停该方法并无视该异常. JVM只会对该方法调用一次,
	//不会调用两次.
	protected void finalize() throws Throwable { };
}
```