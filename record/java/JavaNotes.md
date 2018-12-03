## JVM内存小解
+ 大多数 JVM 将内存区域划分为 Method Area(Non-Heap)(方法区), Heap(堆), Program Counter Register(程序计数器), VM Stack(虚拟机栈), Native Method Stack(本地方法栈), 其中Method Area 和Heap 是线程共享的, VM Stack, Native Method Stack和Program Counter Register是非线程共享的. 
+ 一个 Java 源程序文件, 会被编译(如javac)为字节码文件(以class 为扩展名, 平台无关字节码文件), 每个java程序都需要运行在自己的JVM上, 然后告知JVM 程序的运行入口, 再被JVM 通过字节码解释器加载(这里暂时不考虑即时编译的问题)运行.
+ 概括地说来, JVM初始运行的时候都会分配好Method Area(方法区)和Heap(堆), 而JVM每遇到一个线程, 就为其分配一个Program Counter Register(程序计数器), VM Stack(虚拟机栈)和Native Method Stack (本地方法栈),  当线程终止时, 三者(虚拟机栈, 本地方法栈和程序计数器)所占用的内存空间也会被释放掉. 这也是内存区域分为线程共享和非线程共享的原因, 非线程共享的那三个区域的生命周期与所属线程相同, 而线程共享的区域与JAVA程序运行的生命周期相同, 所以这也是系统垃圾回收的场所只发生在线程共享的区域(主要是堆)的原因.

![show](https://image.cjyong.com/blog/r1.jpg)

![show](https://image.cjyong.com/blog/r2.jpg)


## Statement/PreparedStatement/CallableStatemetn

+  Statement, PreparedStatement和CallableStatement都是接口(interface).
+  Statement继承自Wrapper, PreparedStatement继承自Statement, CallableStatement继承自PreparedStatement.
+ Statement接口提供了执行语句和获取结果的基本方法; PreparedStatement接口添加了处理IN参数的方法;; CallableStatement接口添加了处理 OUT 参数的方法.
+ Statement: 
	+ 普通的不带参的查询SQL; 支持批量更新,批量删除;
+ PreparedStatement:  
	+ 可变参数的SQL,编译一次,执行多次,效率高;  
	+ 安全性好，有效防止Sql注入等问题;  
	+ 支持批量更新,批量删除;  
+ CallableStatement:  
	+ 继承自PreparedStatement,支持带参数的SQL操作;  
	+ 支持调用存储过程,提供了对输出和输入/输出参数(INOUT)的支持;  

+ Statement每次执行sql语句, 数据库都要执行sql语句的编译, 最好用于仅执行一次查询并返回结果的情形，效率高于PreparedStatement. 

+ PreparedStatement是预编译的, 使用PreparedStatement有几个好处:
	- 在执行可变参数的一条SQL时, PreparedStatement比Statement的效率高, 因为DBMS预编译一条SQL当然会比多次编译一条SQL的效率要高. 
	- 安全性好, 有效防止Sql注入等问题. 
	- 对于多次重复执行的语句, 使用PreparedStament效率会更高一点, 并且在这种情况下也比较适合使用batch.
	- 代码的可读性和可维护性.
	
## @Transactional
+ PROPAGATION_REQUIRED--支持当前事务, 如果当前没有事务, 就新建一个事务.这是最常见的选择.
+ PROPAGATION_SUPPORTS--支持当前事务, 如果当前没有事务, 就以非事务方式执行.
+ PROPAGATION_MANDATORY--支持当前事务, 如果当前没有事务, 就抛出异常. 
+ PROPAGATION_REQUIRES_NEW--新建事务, 如果当前存在事务, 把当前事务挂起.
+ PROPAGATION_NOT_SUPPORTED--以非事务方式执行操作, 如果当前存在事务, 就把当前事务挂起.
+ PROPAGATION_NEVER--以非事务方式执行, 如果当前存在事务, 则抛出异常.

## Sleep/Wait
+ Java中的多线程是一种抢占式的机制, 而不是分时机制(协同式).
+ 共同点:
	- 他们都是在多线程的环境下，都可以在程序的调用处阻塞指定的毫秒数, 并返回.
	-  wait()和sleep()都可以通过interrupt()方法打断线程的暂停状态, 从而使线程立刻抛出InterruptedException. 
	- 如果线程A希望立即结束线程B, 则可以对线程B对应的Thread实例调用interrupt方法. 如果此刻线程B正在wait/sleep/join, 则线程B会立刻抛出InterruptedException, 在catch() {} 中直接return即可安全地结束线程.
	- 需要注意的是, InterruptedException是线程自己从内部抛出的, 并不是interrupt()方法抛出的. 对某一线程调用interrupt()时, 如果该线程正在执行普通的代码, 那么该线程根本就不会抛出InterruptedException. 但是, 一旦该线程进入到 wait()/sleep()/join()后, 就会立刻抛出InterruptedException.
+ 不同点 ：  
	- 每个对象都有一个锁来控制同步访问. Synchronized关键字可以和对象的锁交互, 来实现线程的同步. 
	- sleep方法没有释放锁, 而wait方法释放了锁, 使得其他线程可以使用同步控制块或者方法.
	- wait, notify和notifyAll只能在同步控制方法或者同步控制块里面使用, 而sleep可以在任何地方使用 .
	- sleep必须捕获异常, 而wait, notify和notifyAll不需要捕获异常.
	- sleep是线程类（Thread）的方法, 导致此线程暂停执行指定时间, 给执行机会给其他线程, 但是监控状态依然保持, 到时后会自动恢复. 调用sleep不会释放对象锁.
	- wait是Object类的方法, 对此对象调用wait方法导致本线程放弃对象锁, 进入等待此对象的等待锁定池, 只有针对此对象发出notify方法(或notifyAll)后本线程才进入对象锁定池准备获得对象锁进入运行状态.

## |/||
+ 用法：condition 1 | condition 2. condition 1 || condition 2
+ "|"是按位或: 先判断条件1, 不管条件1是否可以决定结果(这里决定结果为true), 都会执行条件2.
+ "||"是逻辑或: 先判断条件1, 如果条件1可以决定结果(这里决定结果为true), 那么就不会执行条件2, 这就是熟称的短路判断法.

## 原码, 反码, 补码
+ 原码: 如果机器字长为n, 那么一个数的原码就是用一个n位的二进制数, 其中最高位为符号位: 正数为0, 负数为1. 剩下的n-1位表示概数的绝对值. 如在Java中byte, 就是8字节, 最大的话0111 1111: 最大为127. 如果负数用原码进行表示如 -10: 1000 1010. 其中正负0的值为0000 0000.

+ 反码: 如果是正数, 则反码与原码相同. 如果是负数, 在原码的基础上, 保持符号位不变, 其余位数取反即可. 如10的原码为: 0000 1010, 则反码亦为 0000 1010. -10的原码为: 1000 1010, 则反码为: 1111 0101.

+ 补码: 如果是正数, 则补码与原码相同. 在反码的基础上, 对反码进行加一操作. 如10的原码是: 0000 1010, 反码为0000 1010, 则补码也为0000 1010. -10的原码为: 1000 1010, 则反码为: 1111 0101, 则补码为: 1111 0110. Java中使用补码来表示负数, 如byte为8字节, 由于正负0都为0000 0000. 那1000 0000位最小的负数为, 128. 1000 0000 减一得反码: 1111 1111. 取反的原码: 1000 0000(为-128).


## 继承
+ 子类继承父类是继承父类所有的方法和成员变量, 如果父类的方法是私有的, 子类不可以调用.
+ HashMap可以将键和值设为null, HashMap线程不安全, Hashtable线程安全
+ 面向对象的五大基本原则
	- 单一职责原则（SRP）
	- 开放封闭原则（OCP） 
	- 里氏替换原则（LSP） 
	- 依赖倒置原则（DIP） 
	- 接口隔离原则（ISP）

## 简易SSL生成
keytool -genkey -alias cjyong -keyalg RSA -keysize 1024 -keypass aoiwerngasfasd -validity 365 -keystore D:\person\keytools\cjyong.keystore -storepass lasdfqwerfaw  //生成证书

keytool -list  -v -keystore D:\person\keytools\cjyong.keystore -storepass lasdfqwerfaw //查看证书

keytool -export -alias cjyong -keystore D:\person\keytools\cjyong.keystore -file D:\person\keytools\cjyong.crt -storepass lasdfqwerfaw //导出证书

keytool -printcert -file D:\person\keytools\cjyong.crt //查看证书

## Spring Boot在Linux下配置成Service
```c
//笔者的Linux系统为Centos7,系统之间存在差异,请酌情修改
1. 创建自己的service文件

cd /usr/lib/systemd/system/
vim myServe.service
2. 写入配置信息

[Unit]
Description=My personal Service
After=syslog.target

[Service]
ExecStart=/usr/local/java/jdk1.8/bin/java -jar /root/.m2/repository/com/cjyong/cp/personalweb/0.0.1-SNAPSHOT/personalweb-0.0.1-SNAPSHOT.jar
SuccessExitStatus=143
         
[Install] 
WantedBy=multi-user.target 

3. 启动服务

service myServe start

4. 注册开机启动

systemctl enable myservice.service

后台脱离终端启动

nohup /root/.nvm/versions/node/v8.3.0/bin/serve -s /root/build &
setsid /root/.nvm/versions/node/v8.3.0/bin/serve -s /root/build
kill -9 $(lsof -i tcp:5000 -t)
```

### 面向对象编程三大特性(封装、继承、多态)
+ 封装: 封装隐藏了类的内部实现机制, 可以在不影响使用的情况下改变类的内部结构, 同时也保护了数据. 对外界而已它的内部细节是隐藏的, 暴露给外界的只是它的访问方法.
+ 继承: 继承是为了重用父类代码. 两个类若存在IS-A的关系就可以使用继承.
+ 多态: 多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定, 而是在程序运行期间才确定, 即一个引用变量倒底会指向哪个类的实例对象, 该引用变量发出的方法调用到底是哪个类中实现的方法, 必须在由程序运行期间才能决定. 因为在程序运行时才确定具体的类, 这样, 不用修改源程序代码, 就可以让引用变量绑定到各种不同的类实现上, 从而导致该引用调用的具体方法随之改变, 即不修改程序代码就可以改变程序运行时所绑定的具体代码, 让程序可以选择多个运行状态, 这就是多态性. 类似设计模式中的策略模式.
有两种方式: 基于继承实现的和基于接口实现.
	+ 基于继承实现: 通过使用父类引用来控制, 实际的实现由子类实现, 具体的实现过程可由子类重写完成.
	+ 基于接口实现: 通过实现同一个接口, 我们可以针对接口编程, 具体的实现, 我们不需要知道.
简单的例子:

```java
//基于继承实现的多态
abstract class Duck {
    public void quack() {
        System.out.println("Gua Gua Gua");
    }
    public void swim(){
        System.out.println("Swim Swim Swim");
    }
    public abstract void show();
}

class MallardDuck extends Duck {
    public void show() {
        System.out.println("I have a green head.");
    }
}

class PlasticDuck extends Duck {
    public void show() {
        System.out.println("I have a yellow head with plastic body.");
    }
    public void quack() {
        System.out.println("Guata Guata");
    }
}
    public static void main(String[] args) {
        Duck duck1 = new MallardDuck();
        Duck duck2 = new PlasticDuck();
        duck1.show();  // I have a green head.
        duck1.quack(); // Gua Gua Gua
        duck2.show();  //I have a yellow head with plastic body.
        duck2.quack(); // Guata Guata
    }
    
//基于接口实现的多态
interface Quackable {
    void quack();
}

class Dog implements Quackable {
    public void quack() {
        System.out.println("Wang Wang Wang");
    }
}

class Chicken implements Quackable {
    public void quack() {
        System.out.println("Ooo Ooo Ooo");
    }
}

public static void main(String[] args) {
    Quackable [] quackables = new Quackable[2];
    quackables[0]= new Dog();
    quackables[1] = new Chicken();
    for (Quackable q : quackables) {
        q.quack();
    }
    //Wang Wang Wang
	//Ooo Ooo Ooo
}

//这里引用一个基于继承实现的一个经典代码(优先级的问题)
//其实在继承链中对象方法的调用存在一个优先级：this.show(O) > super.show(O) > this.show((super)O) > super.show((super)O)
public class A {
    public String show(D obj) {
        return ("A and D");
    }

    public String show(A obj) {
        return ("A and A");
    } 

}

public class B extends A{
    public String show(B obj){
        return ("B and B");
    }
    
    public String show(A obj){
        return ("B and A");
    } 
}

public class C extends B{

}

public class D extends B{

}

public class Test {
    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new B();
        B b = new B();
        C c = new C();
        D d = new D();
        
        System.out.println("1--" + a1.show(b));
        System.out.println("2--" + a1.show(c));
        System.out.println("3--" + a1.show(d));
        System.out.println("4--" + a2.show(b));
        //以4为例: a2的this指针为A, A中没有show(B b)的方法, A的super为Object也没有, 到第三链
        //super(B) = A和Object, A中存在show(A a), 调用该方法, 但是实际上的实现类为B, 所以由B中重载的B.show(A a)来具体实现, 所以为: B and A. 其余同理
        System.out.println("5--" + a2.show(c));
        System.out.println("6--" + a2.show(d));
        System.out.println("7--" + b.show(b));
        System.out.println("8--" + b.show(c));
        System.out.println("9--" + b.show(d));
        /*
        1--A and A
        2--A and A
        3--A and D
        4--B and A
        5--B and A
        6--A and D
        7--B and B
        8--B and B
        9--A and D
        */
    }
}
```


## 有意思的网址

Android MVP: https://juejin.im/post/5bf787d5e51d450c487d06dd.

程序员小说: https://juejin.im/post/5be15a20f265da614c4c4487.

MySQL: https://time.geekbang.org/column/intro/139?utm_term=zeusR9AYC&utm_source=website&utm_medium=infoq&utm_campaign=139-presell

分布式： https://www.infoq.cn/article/pdCpOo*AcZ0xuI3LP45a
