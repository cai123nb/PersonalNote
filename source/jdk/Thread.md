# Thread源码学习
JDK1.8:

```java
package java.lang;

public class Thread implements Runnable {
	private static native void registerNatives();
	static {
		registerNatives();
	}
	//对本类中的本地方法进行注册, 放在首行保证首先被执行
	private volatile char  name[];   //对name设置volatile， 保证多线程的可见性和防止指令重排
	private int            priority; //优先级1-10， 默认继承父线程的优先级
	private Thread         threadQ;  //未使用
	private long           eetop;    //未使用
	private boolean		   single_step; //未使用
	private boolean 	   daemon = false; //是否是守护进程， 默认false， 默认继承父线程的优先级， 只有父线程是守护线程才能创造守护子线程
	private boolean 	   stillborn = false; // 虚拟机状态，未使用
	private Runnable       target; //线程如何运行，默认调用Runnable接口的run()方法
	private ThreadGroup	   group; //线程组,存储和管理当前线程所创建的线程
	private ClassLoader    contextClassLoader; //默认类加载器
	private AccessControlContext inheritedAccessControlContext; //权限控制器, 封装了上下文进行系统资源访问权限控制
	private static int threadInitNumber; //线程编号(静态初始化为0), 类静态采用递增的方式进行唯一性管理
	private static synchronized int nextThreadNum() { //添加synchronized进行加锁, 防止出现并发问题
		return threadInitNumber++;
	}
	ThreadLocal.ThreadLocalMap threadLocals = null;  //本线程的ThreadLocals值, 由ThreadLocal变量进行维护.
	ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;  //继承的ThreadLocals值, 存储来自父线程的TL值
	private long stackSize; // 线程申请的栈大小, 默认是0, 取决于虚拟机是否使用这个字段
	private long nativeParkEventPointer; //虚拟机状态值
	private long tid;  //线程ID
	private static long threadSeqNumber; //线程序列号(静态初始化为0), 采用递增的方式进行保持唯一性, 主要用于产生线程ID
	private static synchronized long nextThreadID() { //序列号加锁
		return ++threadSeqNumber;
	}
	private volatile int threadStatus = 0; //线程状态默认为0: 线程未启动. 设置volatile保证多线程的可见性和防止指令重排
	volatile Object parkBlocker;  //用于支持LockSupport对线程进行堵塞(LockSupport.park())和唤醒(LockSupport.unpark())
	@sun.misc.Contended("tlr")
	long threadLocalRandomSeed;
	@sun.misc.Contended("tlr")
	int threadLocalRandomProbe;
	@sun.misc.Contended("tlr")
	int threadLocalRandomSecondarySeed;
	//当线程在可中断I/O过程中调用暂停方法时, 需要设置暂停状态
	private volatile Interruptible blocker; 
	private final Object blockerLock = new Object();
	void blockedOn(Interruptible b) {  
		synchronized (blockerLock) { //通过对blockerLock加锁,设置blocker,防止出现并发问题
			blocker = b;
		}
	}
	public final static int MIN_PRIORITY = 1; //线程最低优先级
	public final static int NORM_PRIORITY = 5; //默认赋予线程的优先级
	public final static int MAX_PRIORITY = 10; //线程的可以赋予的最大优先级
	//返回当前执行线程的句柄
	public static native Thread currentThread();
	//告诉调度器当前的线程愿意让出CPU的处理时间,调度器可以忽视也可以使用这个信号. 这是一种探索式的方式协调线程与CPU的关系. 但是需要进行详细的分析和基准测试以确保达到目标效果. 常用于测试.
	public static native void yield();
	//让当前的正在执行的线程睡眠一段时间(暂时停止操作), 但是并不会失去监控器的所有权(仍然保持monitor)
	public static native void sleep(long millis) throws InterruptedException;
	//初始化线程函数
	private void init(ThreadGroup g, Runnable target, String name,
					  long stackSize, AccessControlContext acc) {
		if (name == null) { //不允许线程名为空
			throw new NullPointerException("name cannot be null");
		}

		this.name = name.toCharArray(); //存储线程名

		Thread parent = currentThread(); //获得当前执行线程的句柄(一般父线程, 即创建该线程的线程)
		SecurityManager security = System.getSecurityManager(); //获取系统安全管理器
		if (g == null) { //如果没有显式传递ThreadGroup
			/* Determine if it's an applet or not */
			
			/* If there is a security manager, ask the security manager
			   what to do. */
			if (security != null) { //向安全管理器获得ThreadGroup,实现是返回当前执行线程的ThreadGroup
				g = security.getThreadGroup();
			}

			/* If the security doesn't have a strong opinion of the matter
			   use the parent thread group. */
			if (g == null) { //如果当前执行线程不存在的话, 返回当前执行线程的threadgroup, 重复
				g = parent.getThreadGroup();
			}
		}

		/* checkAccess regardless of whether or not threadgroup is
		   explicitly passed in. */
		g.checkAccess(); //检查当前执行线程是否有权限管理ThreadGroup(实际上就是自己的, 传递过来的, 除非是显式传递进来的)

		/*
		 * Do we have the required permissions?
		 */
		if (security != null) {
			if (isCCLOverridden(getClass())) {
				security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
			}
		} //检查操作权限

		g.addUnstarted(); //向ThreadGroup中添加个未启动线程计数

		this.group = g; //赋值给group
		this.daemon = parent.isDaemon(); //继承父线程的守护特性
		this.priority = parent.getPriority(); //继承父线程的优先级
		if (security == null || isCCLOverridden(parent.getClass())) //设置类加载器
			this.contextClassLoader = parent.getContextClassLoader();
		else
			this.contextClassLoader = parent.contextClassLoader;
		this.inheritedAccessControlContext =						//设置继承的权限控制器
				acc != null ? acc : AccessController.getContext();
		this.target = target; //设置target
		setPriority(priority); //设置优先级
		if (parent.inheritableThreadLocals != null) //如果父线程的ThreadLocal值不为空的话, 根据其生成自己的ThreadLocals
			this.inheritableThreadLocals =
				ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
		/* Stash the specified stack size in case the VM cares */
		this.stackSize = stackSize; //设置StackSize

		/* Set thread ID */
		tid = nextThreadID(); //设置ID
	}
	//线程不能被克隆
	@Override
	protected Object clone() throws CloneNotSupportedException {
		throw new CloneNotSupportedException();
	}
	//构造函数统一调用init函数
	public Thread() {
		init(null, null, "Thread-" + nextThreadNum(), 0);
	}
	public Thread(Runnable target) {
		init(null, target, "Thread-" + nextThreadNum(), 0);
	}
	Thread(Runnable target, AccessControlContext acc) {
		init(null, target, "Thread-" + nextThreadNum(), 0, acc);
	}
	public Thread(ThreadGroup group, Runnable target) {
		init(group, target, "Thread-" + nextThreadNum(), 0);
	}
	public Thread(String name) {
		init(null, null, name, 0);
	}
	public Thread(ThreadGroup group, String name) {
		init(group, null, name, 0);
	}
	public Thread(Runnable target, String name) {
		init(null, target, name, 0);
	}
	public Thread(ThreadGroup group, Runnable target, String name) {
		init(group, target, name, 0);
	}
	public Thread(ThreadGroup group, Runnable target, String name,
				  long stackSize) {
		init(group, target, name, stackSize);
	}
	//启动当前线程, 虚拟机会调用该线程的run()方法. 调用这个方法之后,就会同时运行着两个线程:一个是启动该线程的父线程(从start()函数返回),一个是该线程正在执行run()函数.
	public synchronized void start() {
		/**
		 * This method is not invoked for the main method thread or "system"
		 * group threads created/set up by the VM. Any new functionality added
		 * to this method in the future may have to also be added to the VM.
		 *
		 * A zero status value corresponds to state "NEW".
		 */
		if (threadStatus != 0) //如果该线程不是新建状态(就绪状态)启动,抛出异常.
			throw new IllegalThreadStateException();

		/* Notify the group that this thread is about to be started
		 * so that it can be added to the group's list of threads
		 * and the group's unstarted count can be decremented. */
		group.add(this); //告诉组该线程启动了,线程组添加该线程并减少未启动的计数

		boolean started = false;
		try {
			start0(); //本地启动线程
			started = true; //启动成功修改状态为true
		} finally {
			try {
				if (!started) { //如果启动失败,告诉线程组
					group.threadStartFailed(this);
				}
			} catch (Throwable ignore) {
				/* do nothing. If start0 threw a Throwable then
				  it will be passed up the call stack */
			}
		}
	}
	//本地方法接口
	private native void start0();
	private native void setPriority0(int newPriority);
	private native void stop0(Object o);
	private native void suspend0();
	private native void resume0();
	private native void interrupt0();
	private native void setNativeName(String name);
	//线程启动时调用方法,子类需要进行覆盖, 新建类需要显示传递Runnable接口对象.
	@Override
	public void run() {
		if (target != null) {
			target.run();
		}
	}
	//在线程彻底退出之前,系统提供的清除函数机会
	private void exit() {
		if (group != null) { //通知线程组移除该线程
			group.threadTerminated(this);
			group = null;
		}
		/* Aggressively null out all reference fields: see bug 4006245 */
		target = null; //积极地使所有的字段失效
		/* Speed the release of some of these resources */
		threadLocals = null; //减速释放一些资源
		inheritableThreadLocals = null;
		inheritedAccessControlContext = null;
		blocker = null;
		uncaughtExceptionHandler = null;
	}
	//强制线程停止执行, 无论是停止自己(自己停止自己)还是停止他人(自己停止别人)都会检查操作权限.这个线程会强制停止工作,无论在做任何事, 并且应用程序不应该显示捕获ThreadDeath对象,如果需要进行额外的清除需要清除完毕之后重新抛出从而让线程最终死亡. 不推荐使用该方法
	//废弃原因: 调用这个方法时会解锁它拥有的所有监视器(这是因为ThreadDeath在栈上传播的结果), 这会导致之前受到这些监视器保护的对象处于一种不一致的状态,可能导致不确定性的行为.
	@Deprecated
	public final void stop() {
		SecurityManager security = System.getSecurityManager();
		if (security != null) {
			checkAccess();
			if (this != Thread.currentThread()) {
				security.checkPermission(SecurityConstants.STOP_THREAD_PERMISSION);
			}
		}
		// A zero status value corresponds to "NEW", it can't change to
		// not-NEW because we hold the lock.
		if (threadStatus != 0) {
			resume(); // Wake up thread if it was suspended; no-op otherwise
		}

		// The VM can handle all thread states
		stop0(new ThreadDeath());
	}
	//废弃,原先是想停止线程并抛出一个自己想要的异常, 这可能导致抛出一个前面并没有进行处理的异常, 充满很大的随机性.
	@Deprecated
	public final synchronized void stop(Throwable obj) {
		throw new UnsupportedOperationException();
	}
	//中断线程,如果线程被堵塞了(wait(),join(),sleep()),会抛出一个异InterruptException, 如果线程堵塞在I/O(NIO)操作中,channel将会关闭, 设置中断标识,并且抛出一个ClosedByInterruptException.
	//如果线程堵塞在java.nio.channels.Selector中, 线程会设置对应的标记,立马返回. 如果上述条件都不满足, 将会设置线程中断标识.
	public void interrupt() {
		if (this != Thread.currentThread())
			checkAccess();

		synchronized (blockerLock) {
			Interruptible b = blocker;
			if (b != null) {
				interrupt0();            // Just to set the interrupt flag
				b.interrupt(this);
				return;
			}
		}
		interrupt0();
	}
	//本地方法,测试线程是否中断.
	private native boolean isInterrupted(boolean ClearInterrupted);
	//测试当前线程是否中断, 线程的暂停状态标识会被清除:如果中断后, 你调用2次, 第二次就会返回false. 第一调用返回true, 说明被中断了. 第二次调用的时候, 已经被第一次清除了标识(变成在运行了), 就会返回false了.
	public static boolean interrupted() {
		return currentThread().isInterrupted(true);
	}
	//测试返回当前线程是否中断, 不会清除暂停状态
	public boolean isInterrupted() {
		return isInterrupted(false);
	}
	//废弃原因: 如果线程持有某个重要资源的锁, 又被强制清除了所有内存, 那份资源将无法进行访问.
	@Deprecated
	public void destroy() {
		throw new NoSuchMethodError();
	}
	//本地方法: 测试接口是否活着: 启动着但是还没死.
	public final native boolean isAlive();
	//挂起线程, 废弃原因: 如果挂起时持有关键资源的锁, 则任何线程都无法访问得到该资源,直到该线程恢复.
	@Deprecated
	public final void suspend() {
		checkAccess();
		suspend0();
	}
	//废弃,原因由上.
	@Deprecated
	public final void resume() {
		checkAccess();
		resume0();
	}
	//判断当前运行线程是否有权限修改该线程
	public final void checkAccess() {
		SecurityManager security = System.getSecurityManager();
		if (security != null) {
			security.checkAccess(this);
		}
	}
	//设置线程的优先级, 不允许超过线程组中最大的优先级.
	public final void setPriority(int newPriority) {
		ThreadGroup g;
		checkAccess();
		if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
			throw new IllegalArgumentException();
		}
		if((g = getThreadGroup()) != null) {
			if (newPriority > g.getMaxPriority()) {
				newPriority = g.getMaxPriority();
			}
			setPriority0(priority = newPriority);
		}
	}
	public final int getPriority() {
		return priority;
	}
	//设置线程名称
	public final synchronized void setName(String name) {
		checkAccess();
		this.name = name.toCharArray();
		if (threadStatus != 0) {
			setNativeName(name);
		}
	}
	public final String getName() {
		return new String(name, true);
	}
	//返回线程组, 如果线程死了返回NULL
	public final ThreadGroup getThreadGroup() {
		return group;
	}
	//返回当前线程组中所有活着的线程数(线程组中如果存在子线程组,也会动态遍历计数)
	public static int activeCount() {
		return currentThread().getThreadGroup().activeCount();
	}
	//拷贝线程组中所有活着的线程到该数组中去, 如果数组的容量过小, 多出的线程将被无视掉. 如果线程非常重要可以使用activeCount进行新建对应大小数组进行储存.推荐用于调试和模拟.
	public static int enumerate(Thread tarray[]) {
		return currentThread().getThreadGroup().enumerate(tarray);
	}
	//返回当前线程的栈帧个数. 线程必须挂起. 
	//废弃原因: 依靠suspend(), 而suspend()方法已经被废弃, 理由同上.
	@Deprecated
	public native int countStackFrames();
	//最多等待millis时长让当前执行线程(调用这个方法的线程)先完成, 通过循环调用该线程的wait()方法保证线程一直等待.
	public final synchronized void join(long millis) throws InterruptedException {
		long base = System.currentTimeMillis();
		long now = 0;

		if (millis < 0) {
			throw new IllegalArgumentException("timeout value is negative");
		}

		if (millis == 0) {
			while (isAlive()) {
				wait(0);
			}
		} else {
			while (isAlive()) {
				long delay = millis - now;
				if (delay <= 0) {
					break;
				}
				wait(delay);
				now = System.currentTimeMillis() - base;
			}
		}
	}
	//同理
	public final synchronized void join(long millis, int nanos) throws InterruptedException {

		if (millis < 0) {
			throw new IllegalArgumentException("timeout value is negative");
		}

		if (nanos < 0 || nanos > 999999) {
			throw new IllegalArgumentException(
								"nanosecond timeout value out of range");
		}

		if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
			millis++;
		}

		join(millis);
	}
	//打印当前线程的堆栈信息到标准的错误Stream流中, 只用于调试
	public static void dumpStack() {
		new Exception("Stack trace").printStackTrace();
	}
	//设置一个线程为守护进程, 只有在线程在启动前调用这个方法. JVM退出之后, 只有守护进程可以运行.
	public final void setDaemon(boolean on) {
		checkAccess();
		if (isAlive()) {
			throw new IllegalThreadStateException();
		}
		daemon = on;
	}
	//返回当前线程是否为守护进程
	public final boolean isDaemon() {
		return daemon;
	}
	//ToString函数
	public String toString() {
		ThreadGroup group = getThreadGroup();
		if (group != null) {
			return "Thread[" + getName() + "," + getPriority() + "," +
						   group.getName() + "]";
		} else {
			return "Thread[" + getName() + "," + getPriority() + "," +
							"" + "]";
		}
	}
	//返回当前线程的上下文类加载器, 用于加载类和资源(最初的类加载器加载整个应用). 如果没有设置, 默认是父线程的.
	@CallerSensitive
	public ClassLoader getContextClassLoader() {
		if (contextClassLoader == null)
			return null; 
		SecurityManager sm = System.getSecurityManager();
		if (sm != null) { //进行权限检查
			ClassLoader.checkClassLoaderPermission(contextClassLoader,
												   Reflection.getCallerClass());
		}
		return contextClassLoader;
	}
	//设置类加载器
	public void setContextClassLoader(ClassLoader cl) {
		SecurityManager sm = System.getSecurityManager();
		if (sm != null) { 
			sm.checkPermission(new RuntimePermission("setContextClassLoader"));
		}
		contextClassLoader = cl;
	}
	//有且只有当前线程持有指定对象(obj)的锁的时候返回true
	public static native boolean holdsLock(Object obj);
	//以数组的形式返回堆栈中所有的元素. 如果线程还没启动, 或者启动了还没被CPU运行, 或者已经结束了就会返回一个空的数组.
	//数组中第一个对象是最近方法(在序列中)调用的栈帧, 最后一个是最久之前调用的. 安全管理器会对权限进行校验. 另外这个需要虚拟机的支持.
	private static final StackTraceElement[] EMPTY_STACK_TRACE = new StackTraceElement[0];
	public StackTraceElement[] getStackTrace() {
		if (this != Thread.currentThread()) {
			// check for getStackTrace permission
			SecurityManager security = System.getSecurityManager();
			if (security != null) {
				security.checkPermission(
					SecurityConstants.GET_STACK_TRACE_PERMISSION);
			}
			// optimization so we do not call into the vm for threads that
			// have not yet started or have terminated
			if (!isAlive()) {
				return EMPTY_STACK_TRACE;
			}
			StackTraceElement[][] stackTraceArray = dumpThreads(new Thread[] {this});
			StackTraceElement[] stackTrace = stackTraceArray[0];
			// a thread that was alive during the previous isAlive call may have
			// since terminated, therefore not having a stacktrace.
			if (stackTrace == null) {
				stackTrace = EMPTY_STACK_TRACE;
			}
			return stackTrace;
		} else {
			// Don't need JVM help for current thread
			return (new Exception()).getStackTrace();
		}
	}
	//本地支持方法
	private native static StackTraceElement[][] dumpThreads(Thread[] threads);
	private native static Thread[] getThreads();
	//返回所有活着线程的堆栈信息, 参照上面的方法
	public static Map<Thread, StackTraceElement[]> getAllStackTraces() {
		// check for getStackTrace permission
		SecurityManager security = System.getSecurityManager();
		if (security != null) {
			security.checkPermission(
				SecurityConstants.GET_STACK_TRACE_PERMISSION);
			security.checkPermission(
				SecurityConstants.MODIFY_THREADGROUP_PERMISSION);
		}

		// Get a snapshot of the list of all threads
		Thread[] threads = getThreads();
		StackTraceElement[][] traces = dumpThreads(threads);
		Map<Thread, StackTraceElement[]> m = new HashMap<>(threads.length);
		for (int i = 0; i < threads.length; i++) {
			StackTraceElement[] stackTrace = traces[i];
			if (stackTrace != null) {
				m.put(threads[i], stackTrace);
			}
			// else terminated so we don't put it in the map
		}
		return m;
	}
	private static final RuntimePermission SUBCLASS_IMPLEMENTATION_PERMISSION =
					new RuntimePermission("enableContextClassLoaderOverride");
	//缓存子类安全的审计结果, 未来版本中以ConcurrentReferenceHashMap进行替换
	private static class Caches {
		/** cache of subclass security audit results */
		static final ConcurrentMap<WeakClassKey,Boolean> subclassAudits =
			new ConcurrentHashMap<>();

		/** queue for WeakReferences to audited subclasses */
		static final ReferenceQueue<Class<?>> subclassAuditsQueue =
			new ReferenceQueue<>();
	}
	//验证该实例是否可以在不违背安全限制下被实例化: 子类不能重载任何安全敏感的非静态方法, 否则就需要"enableContextClassLoaderOverride"权限打开.
	private static boolean isCCLOverridden(Class<?> cl) {
		if (cl == Thread.class)
			return false;
		
		processQueue(Caches.subclassAuditsQueue, Caches.subclassAudits);
		WeakClassKey key = new WeakClassKey(cl, Caches.subclassAuditsQueue);
		Boolean result = Caches.subclassAudits.get(key);
		if (result == null) {
			result = Boolean.valueOf(auditSubclass(cl));
			Caches.subclassAudits.putIfAbsent(key, result);
		}

		return result.booleanValue();
	}
	//从Map中移除在queue中任何键
	static void processQueue(ReferenceQueue<Class<?>> queue,
							 ConcurrentMap<? extends
							 WeakReference<Class<?>>, ?> map)
	{
		Reference<? extends Class<?>> ref;
		while((ref = queue.poll()) != null) {
			map.remove(ref);
		}
	}
	//给指定的子类进行反射检查, 以验证它是否不会覆盖对安全性敏感的非最终方法。 如果子类重写任何方法，则返回true，否则返回false。
	private static boolean auditSubclass(final Class<?> subcl) {
		Boolean result = AccessController.doPrivileged(
			new PrivilegedAction<Boolean>() {
				public Boolean run() {
					for (Class<?> cl = subcl;
						 cl != Thread.class;
						 cl = cl.getSuperclass())
					{
						try {
							cl.getDeclaredMethod("getContextClassLoader", new Class<?>[0]);
							return Boolean.TRUE;
						} catch (NoSuchMethodException ex) {
						}
						try {
							Class<?>[] params = {ClassLoader.class};
							cl.getDeclaredMethod("setContextClassLoader", params);
							return Boolean.TRUE;
						} catch (NoSuchMethodException ex) {
						}
					}
					return Boolean.FALSE;
				}
			}
		);
		return result.booleanValue();
	}
	//返回线程ID
	public long getId() {
		return tid;
	}
	//线程的五种状态
	public enum State {
	NEW,
	//0, 线程新建好,还没启动
	RUNNABLE,
	//1, 线程正在虚拟机中执行, 但是可能在等待资源如处理器.
	BLOCKED,
	//2, 线程被堵塞, 等待监视器锁, 来进入synchronize块或方法 或者在调用Object的wait方法后,重新进入 synchronize 块或方法
	WAITING,
	//3, 无限期地等待别的进程执行特定的操作, 通过调用以下方法进入该状态: Object.wait(),Thread.join(),LockSupport.park(). 需要别的进程进行唤醒, 否则一直等待.
	TIMED_WAITING,
	//4, 等待别的进程执行特定的操作直到一段时间, 可以通过调用以下方法进入: Thread.sleep(long),Object.wait(long),Thread.join(long), LockSupport.parkNanos(long),LockSupport.parkUntil(long).
	TERMINATED;
	//5, 线程退出
	}
	//获取当前的线程状态
	public State getState() {
		// get current thread state
		return sun.misc.VM.toThreadState(threadStatus);
	}
	//处理当线程因为未捕获的异常而终止的情况, 当出现这种情况虚拟机则调用线程中的getUncaughtExceptionHandler的方法获取该接口对象,进行处理(传递线程和异常进行处理).
	//如果当前线程没有UncaughtExceptionHandler, 则从线程的ThreadGroup中寻找, 如果还没有找到, 则调用getDefaultUncaughtExceptionHandler, 用默认的Handler对象进行处理.
	@FunctionalInterface
	public interface UncaughtExceptionHandler {
		/**
		 * Method invoked when the given thread terminates due to the
		 * given uncaught exception.
		 * <p>Any exception thrown by this method will be ignored by the
		 * Java Virtual Machine.
		 * @param t the thread
		 * @param e the exception
		 */
		void uncaughtException(Thread t, Throwable e);
	}
	// null unless explicitly set
	private volatile UncaughtExceptionHandler uncaughtExceptionHandler;
	// null unless explicitly set
	private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
	//设置默认的未捕获异常处理器
	public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
		SecurityManager sm = System.getSecurityManager(); 
		if (sm != null) { //进行权限校验
			sm.checkPermission(
				new RuntimePermission("setDefaultUncaughtExceptionHandler")
					);
		}
		defaultUncaughtExceptionHandler = eh;
	}
	//返回默认的未捕获异常处理器
	public static UncaughtExceptionHandler getDefaultUncaughtExceptionHandler(){
		return defaultUncaughtExceptionHandler;
	}
	public UncaughtExceptionHandler getUncaughtExceptionHandler() {
		return uncaughtExceptionHandler != null ?
			uncaughtExceptionHandler : group;
	}
	public void setUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
		checkAccess();
		uncaughtExceptionHandler = eh;
	}
	//分发未捕获的异常, 只能被JVM调用
	private void dispatchUncaughtException(Throwable e) {
		getUncaughtExceptionHandler().uncaughtException(this, e);
	}
	//Class对象的弱键
	static class WeakClassKey extends WeakReference<Class<?>> {
		/**
		 * saved value of the referent's identity hash code, to maintain
		 * a consistent hash code after the referent has been cleared
		 */
		private final int hash;

		/**
		 * Create a new WeakClassKey to the given object, registered
		 * with a queue.
		 */
		WeakClassKey(Class<?> cl, ReferenceQueue<Class<?>> refQueue) {
			super(cl, refQueue);
			hash = System.identityHashCode(cl);
		}

		/**
		 * Returns the identity hash code of the original referent.
		 */
		@Override
		public int hashCode() {
			return hash;
		}

		/**
		 * Returns true if the given object is this identical
		 * WeakClassKey instance, or, if this object's referent has not
		 * been cleared, if the given object is another WeakClassKey
		 * instance with the identical non-null referent as this one.
		 */
		@Override
		public boolean equals(Object obj) {
			if (obj == this)
				return true;

			if (obj instanceof WeakClassKey) {
				Object referent = get();
				return (referent != null) &&
					   (referent == ((WeakClassKey) obj).get());
			} else {
				return false;
			}
		}
	}
}
```