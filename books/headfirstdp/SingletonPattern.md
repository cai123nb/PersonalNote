# 独一无二的对象 单件模式

单件模式: 确保一个类只有一个实例, 并提供了一个全局的访问点.

> 我们可以将某个类设计成自己管理的一个单独的实例, 同时也避免其他类再自行产生实例. 想获得单件实例, 通过单件类是唯一的途径.
> 我们提供对这个实例的全局访问点, 当你需要实例时, 向类查询, 它会返回单个实例.

简单的例子:

```java
public class ChocolateBoiler {
	private boolean empty;
	private boolean boiled;
	private static ChocolateBoiler uniqueInstance;
    //这里采用延迟加载, 只有当你需要的时候才进行加载
  
	private ChocolateBoiler() {	//私有化构造函数
		empty = true;
		boiled = false;
	}
  
	public static ChocolateBoiler getInstance() {	//静态获取实例的方式
		if (uniqueInstance == null) {
			System.out.println("Creating unique instance of Chocolate Boiler");
			uniqueInstance = new ChocolateBoiler();
		}
		System.out.println("Returning instance of Chocolate Boiler");
		return uniqueInstance;
	}
	//.... 其它代码省略
}
```

但这时候会出现一个问题, 如果在多线程的情况下, 如果两个线程同时到达getInstance()方法,这时候两者都没有找到uniqueInstance, 这时候就会创建两个不同的实例句柄. 这时候就会出现问题.

## 第一种解决方案

将getInstance()添加同步标志, synchronized.

```java
public static synchronized ChocolateBoiler getInstance() {	//静态获取实例的方式
    if (uniqueInstance == null) {
        System.out.println("Creating unique instance of Chocolate Boiler");
        uniqueInstance = new ChocolateBoiler();
    }
    System.out.println("Returning instance of Chocolate Boiler");
    return uniqueInstance;
}
```

优点: 简单易懂
缺点: 每次调用这个方法都会进行同步处理, 系统消耗较大. 如果该方法的调用频率较高就会导致系统的消耗巨大.

## 第二种解决方法

在类加载的时候, 就直接进行初始化操作.

```java
private static ChocolateBoiler uniqueInstance = new ChocolateBoiler();
public static ChocolateBoiler getInstance() {	//静态获取实例的方式
    return uniqueInstance;
}
```

优点: 简单, 且不需要进行同步操作.
缺点: 如果系统较为庞大, 由于无法延时加载, 就会导致系统启动较慢.

## 第三种解决方法

双重加锁法

```java
public class Singleton {
	private volatile static Singleton uniqueInstance;
 
	private Singleton() {}
 
	public static Singleton getInstance() {
		if (uniqueInstance == null) {
			synchronized (Singleton.class) {
				if (uniqueInstance == null) {
					uniqueInstance = new Singleton();
				}
			}
		}
		return uniqueInstance;
	}
}
```

通过这种方法, 我们只在第一次当uniqueInstance为null的时候进行同步操作, 这时候不要忘记在内部再次进行判断. 如果两个线程同时进入到了同步区, 当第二个线程再次进入的时候,需要进行判断是否已经创建好了. 这时候uniqueInstance需要添加volatile关键字防止多重创建.

## 总结

OO基础:

+ 抽象
+ 封装
+ 多态
+ 继承

OO原则:

+ 封装变化
+ 多用组合, 少用继承
+ 针对接口编程, 不针对实现编程
+ 为交互对象之间的松耦合设计而努力
+ 对拓展开放, 对修改关闭
+ 依赖抽象, 不要依赖具体的类

单件模式: 确保一个类只有一个实例, 并提供了一个全局的访问点.

要点:

+ 单件模式确保一个类只有一个实例, 并提供了一个全局的访问点.
+ 单件模式也提供了访问这个实例的全局点.
+ 在Java中实现单件模式需要私有的构造器, 一个静态方法和一个静态变量.
+ 确定在性能和资源上的限制, 然后小心地选择不同的解决方案来解决多线程的问题.
+ 双重加锁只支持Java5及以上版本.

