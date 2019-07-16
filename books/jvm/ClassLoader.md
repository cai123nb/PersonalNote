# 虚拟机类加载机制

## 简述

虚拟机把描述类的数据从Class文件加载到内存, 并对数据进行校验, 转换解析和初始化, 最终形成可以被虚拟机直接使用的Java类型, 这就是虚拟机的类加载机制.
类加载的特点: Java的类型的加载, 连接和初始化过程都是在程序运行时完成的, 这种策略虽然会令类加载时稍微增加一些性能开销, 但是会为Java应用程序提供高度的灵活性, Java里天生可以动态拓展的语言特性就是依赖运行期动态加载和动态链接实现的.(如面向接口编程和远程加载二进制流等).

## 类加载的时机

类加载到虚拟机内存到卸载出内存为止, 生命周期包括:
+ 加载(Loading)
+ 验证(Verification)
+ 准备(Preparation)
+ 解析(Resolution)
+ 初始化(Initialization)
+ 使用(Using)
+ 卸载(Unloading)

加载, 验证, 准备, 初始化, 卸载这5个阶段是确定的, 类加载的过程必须按照这个顺序进行开始, 但是解析阶段就不一定了, 它在某些情况下可以在初始化阶段之后再进行, 这是为了支持Java语言运行时绑定.

类在什么时候会进行加载?(这里的加载是指第一阶段), Java虚拟机规范中并没有强制的约束. 但是对于初始化阶段(Initialization), 虚拟机规范严格规定了有且只有五种情况下必须对类进行初始化:
+ 遇到new, getstatic, putstatic或invokestatic这4条字节码指令时, 如果类没有初始化, 则进行初始化操作. 如: new实例化对象, 读取静态字段(被final修饰, 放入常量池中除外), 调用静态方法等.
+ 使用java.lang.reflect包的方法对类进行反射调用时.
+ 初始化一个类的时候, 发现其父类还没有进行初始化, 则需要对父类进行初始化操作.
+ 当虚拟机启动时, 用户需要指定的主类. 如: 包含Main()函数的主类.
+ java.lang.invoke.MethodHandle实例最后的解析结果是REF_getStatic, REF_putStatic, REF_invokeStatic的方法句柄.

## 类加载的过程

### 加载-Loading

虚拟机在这个过程中, 主要完成3件事:
+ 通过一个类的全限定名来获取此类的二进制字节流.
+ 将这个字节流所代表的静态存储结构转换为方法区的运行时的数据结构.
+ 在内存中生成一个代表这个类的Java.lang.Class对象, 作为方法区这个类的各种数据的访问入口.

但是虚拟机并没有限制二进制字节流的来源, 这为我们的提供了一个很好的机会:
+ 从ZIP包中读取, 最终成为日后JAR, EAR, WAR格式的基础.
+ 网络中获取, 如Applet.
+ 运行时计算生成, 如动态代理技术: java.lang.reflect.Proxy中的ProxyGenerator.generateProxyClass为特定接口生成代理类的二进制字节流.
+ 其他文件中生成, 如JSP应用, 生成对应的Class类.
+ 数据库中读取, 如中间件服务器(类似SAP Netweaver).

对于非数组类的加载过程, 是开发人员可控性最强的, 既可以使用系统提供的引导类加载器来完成, 也可以由用户自定义的类加载器来完成, 开发人员可以通过定义自己的类加载器去控制字节流的获取方式.
而对于数组类而言, 情况就有所不同, 数组类本身不通过类加载器创建, 它是由Java虚拟机直接创建的. 但是仍然与类加载器有很密切的关系, 因为数组类的元素类型(Element Type指的是数组去掉所有的维度的类型), 最终是要靠类加载器去创建, 一个数组类(简称C)创建过程遵循以下规则:
+ 如果数组的组件类型(Component Type, 指的是数组去掉一个维度的类型)是引用类型, 那就递归地加载该组件类型, 数组C将在加载该组件类型的类加载器的类名称空间上被标识.
+ 如果数组的组件类型不是引用类型(如int[]), Java虚拟机将会把数组C标记为与引导类加载器关联.
+ 数据类的可见性与它的组件类型可见性一致, 如果组件类型不是引用, 则默认为pulic.

加载完毕之后, 虚拟机外部的二进制字节流就会按照虚拟机所需的格式存储在方法区之中, 方法区的存储格式由虚拟机自行定义, 然后在内存中实例化一个java.lang.Class类的对象(没有明确规定是在Java堆中, 对于HotSpot虚拟机而言, Class对象特殊, 存储在方法区中), 作为程序访问方法区的这些类型数据的外部接口.

### 验证-Verification

验证步骤的目的主要是确保Class文件中的字节流所包含的信息符合当前虚拟机的要求, 不会危害虚拟机自身的安全.
验证过程较为复杂, 大体上分为4个阶段:
+ 文件格式验证: 验证字节流是否符合Class文件格式的规范, 并且可以被当前版本虚拟机处理. 如: 时候以0xCAFEBABE开头, 主次版本号是否可以进行处理, 常量池中是否包含不支持的常量类型等等.
+ 元数据验证: 这个类是否有父类(除了java.lang.Object之外, 都应该有父类), 是否继承了不允许继承的类(如final修饰), 如果不是抽象类是否实现了父类的所要求的方法等等.
+ 字节码验证: 最复杂的一个步骤, 主要目的是通过数据流和控制流分析, 确定程序语义是合法的, 符合逻辑的. 如: 保证任意时刻操作数栈的数据类型与指令代码可以配合工作, 保证跳转指令不会跳转到方法体以外的字节码指令上, 方法体中类型转换是有效的等等.
+ 符号引用验证: 将符号引用转化为直接引用的时候, 这个转化动作将在连接的第三个阶段: 解析阶段中进行. 符号引用验证可以看做是做对类本身以外的信息进行匹配校验: 如符号引用中通过字符串描述的全限定名是否可以找到对应的类, 是否存在符合方法的字段描述符以及简单名称所描述的方法和字段, 访问性等等.

### 准备-Preparation

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段, 这些变量所使用的内存都将在方法区中进行分配. 这里的是指的类变量(static修饰的), 不包括实例变量, 实例变量将会在对象实例化时随着对象一起分配在Java堆中. 一般情况都是赋值为0, 如(int:0, long:0L, short:(short)0, char: '\u000', boolean: false, reference: null, byte: (byte)0, double: 0.0d, float: 0.0f).  但是也有例外:

```java
public static int value = 123; // 这时候, 准备阶段是赋值为0的, 只有在初始化阶段才会赋值为123.
pubLic static final int value = 123; //这种情况就是特殊情况, 123会作为类常量存在类的常量池中, 这时候准备阶段就会根据ConstantValue将value的值赋值为123.
```

### 解析-Resolution

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程.

> + 符号引用(Symbolic References): 符号引用以一组符号来描述所引用的目标, 符号可以是任何形式的字面量, 只要使用时可以无歧义地定位到目标即可. 符号引用与虚拟机实现的内存布局无关, 引用的目标并不一定已经加载到内存中. 各种虚拟机实现的内存布局可以各不相同, 但是它们可以接受的符号引用必须一致.
> + 直接引用(Direct References): 直接引用可以是直接指向目标的指正, 相对偏移量或是一个能间接定位到目标的句柄. 直接引用和虚拟机实现的内存布局相关, 同一个符号引用在不同的虚拟机实例上翻译的直接引用一般不同. 如果有了直接引用, 那引用的目标必定已经在内存中存在了.

虚拟机规范没有规定解析阶段的具体时间, 只是限制在以下命令执行前, 先对它们所使用的符号引用进行解析: **anewarray, checkcast, getfield, getstatic, instanceof, invokedynamic, invokeinterface, invokerspecial, invokerstatic, invokevirtual, ldc, ldc_w, multianewarray, new, putfield, putstatic**. 所以虚拟机需要根据需要来判断是否在类加载时就对常量池中的符号引用进行解析, 还是等到一个符号引用将要被使用前才去解析.
对同一个符号引用进行多次解析请求是很常用的, 除invokerdynamic指令以外, 虚拟机实现可以对第一次解析结果进行缓存(运行时常量池中直接记录直接引用, 把常量标识为已解析状态), 来避免重复解析. 虚拟机需要保证(如果第一次解析成功了, 后续的解析请求也必须成功, 如果第一次失败了, 后续的解析也应该收到同样的异常). 但是对于invokedynamic指令, 则是不一样, 如果某个对象是由invokedynamic指令触发解析, 并不能保证着后续的其他invokerdnamic解析可以得到同样的结果. 因为invokedynamic指令目的本来就是用于动态语言的支持, 它所对应的引用称为`动态调用点限定符(Dynamic Call Site Specifier)`, 即必须等到程序运行到这条指令的时候, 解析动作才能进行. 相对其他可以触发解析的指令而言, 都是`静态`的, 可以在刚刚完成加载阶段, 还没开始执行代码时就可以进行解析.
解析动作主要是针对七类符号引用进行: 类或接口(CONSTANT_Class_info), 字段(CONSTANT_Fieldref_info), 类方法(CONSTANT_Methodref_info), 接口方法(CONSTANT_InterfaceMethodref_info), 方法类型(CONSTANT_MethodType_info), 方法句柄(CONSTANT_MethodHandler_info)和调用点限定符(CONSTANT_InvokeDynamic_info). 这里先介绍前4种引用的解析, 后三种下一节和动态语言调用一起进行讲解.

#### 类或接口的解析

简单的例子:

```java
class D {
	C n = new C();
}
```

解析步骤:
1. 如果C不是数组类型, 虚拟机将会把代表n的全限定传递给D的类加载器去加载类C. 在加载过程中, 由于元数据, 字节码验证的需要, 又可能触发其他相关类的加载动作(如C的父类或者接口等等), 一旦这个加载过程出现异常, 解析过程宣告失败.
2. 如果C是一个数组类型, 并且数组的元素类型也为对象(描述符类似`[Ljava/lang/Integer`), 先按照第一步加载数组元素类型(上面例子为: `java.lang.Integer`), 接着由虚拟机生成一个代表此数组维度和元素的数组对象.
3. 如果上述步骤没问题, 那么C在虚拟机中实际已经成为了一个有效的类或者接口, 但是在完成之前, 还需要进行符号引用验证, 验证是否D对C是否有访问权限.

#### 字段解析

解析一个字段符号引用, 首先解析字段表(Class + NameAndType)中的class_index索引的CONSTANT_Class_info符号引用. 如果解析这个类或者接口引用时出现任何异常都会导致解析过程的失败. 如果解析成功完成, 那将这个字段所属的类或接口用C表示, 虚拟机规范要求按照以下步骤对C进行后续字段的搜索:
+ 如果C本身就包含了简单名称和字段描述符都与目标匹配的字段, 则直接返回这个字段的直接引用, 查找结束.
+ 否则, 如果C中实现了接口, 将会按照继承关系从下向上递归搜索各个接口和它的父接口, 如果接口中包含了简单名称和字段描述符一致的目标, 返回该字段的直接引用, 查找结束.
+ 否则, 如果C不是java.lang.Objct, 将会按照继承关系从下向上递归搜索其父类, 如果父类中包含了简单名称和字段描述符一致的字段, 则返回直接引用, 查找结束.
+ 负责, 查找失败, 抛出java.lang.NoSuchFieldError异常.

查找成功之后, 还会对字段进行权限校验, 如果没有权限就抛出:java.lang.IllegalAccessError. 一般在虚拟机的编译器中会更加严格一点, 如果一个同名字段同时出现在接口和父类, 或者同时在自家或父类的多个接口中出现, 编译器都可能拒绝编译.

#### 类方法解析

类似字段符号解析, 首先解析class_index项中索引的方法所属的类或接口引用, 解析成功, 按照以下步骤进行搜索:
+ 类方法和接口方法的符号引用是分开的, 如果类方法中发现class_index的索引指的是一个接口, 抛出java.lang.IncompatibleClassChangeError异常.
+ 如果通过了第1步, 在类C中查找是否有简单名称和描述符与目标相匹配的方法, 有直接返回这个方法的引用, 查找结束.
+ 否则, 类似字段(父类查找)
+ 否则, 类似字段(接口查找)
+ 否则, 查找失败, 抛出java.lang.NoSunchMethodError.

如果查找成功,对该方法进行权限检验, 如果没有权限就抛出:java.lang.IllegalAccessError.

#### 接口方法解析

类似字段符号解析, 首先解析class_index项中索引的方法所属的类或接口引用, 解析成功, 按照以下步骤进行搜索:
+ 类方法和接口方法的符号引用是分开的, 如果类方法中发现class_index的索引指的是一个类而不是接口, 抛出java.lang.IncompatibleClassChangeError异常.
+ 如果通过了第1步, 在接口C中查找是否有简单名称和描述符与目标相匹配的方法, 有直接返回这个方法的引用, 查找结束.
+ 否则, 类似字段(父接口查找)
+ 否则, 查找失败, 抛出java.lang.NoSunchMethodError.

由于接口都是默认public的, 不用进行权限校验.

### 初始化-Initialization

类初始化阶段是类加载过程的最后一步, 这一步才真正开始执行类中定义的Java程序代码(或者说是字节码). 在准备阶段, 变量以及被赋予了一次初始值, 而在初始化阶段则是按照程序员的需求去初始化类变量和其他资源. 本质上, 初始化阶段就是执行类构造器的<clint>()方法的过程. (这里只针对Java编译的Class文件)
+ <clint>()方法由编译器自动收集类中的所有类变量的赋值动作和静态语句块(static{})中的语句合并产生的, 收集的顺序是按照语句在源文件中出现的顺序决定的, 静态语句块中只能访问到定义在静态语句块之前的变量, 不能访问在其之后定义的变量.(如果出现这种情况, 赋值可以正常编译通过, 使用的时候会提示"非法向前引用").
+ <clint>()方法与类的构造器函数(<init>())不同, 它不需要显示调用父类构造器, 虚拟机会保证子类的<clint>()方法执行之前, 父类的<clint>()已经执行完毕(因此父类定义的静态语句块要优于子类定义的变量赋值操作). 因此虚拟机中第一个被执行的<clint>()方法肯定是java.lang.Object.
+ <clint>()是非必须的, 如果一个类没有静态语句块和类变量的赋值操作, 那么编译器可以不为这个类生成<clint>()方法.
+ 接口中不能使用静态语句块, 但是也有变量的初始化赋值操作, 也会产生<clint>()方法, 但是接口不需要先执行父接口的<clint>()方法, 在接口定义变量使用的时候, 才会进行初始化父接口. 接口的实现类在初始化的时候也一样不会执行接口的<clint>()方法.
+ 虚拟机会保证一个类的<clint>()方法在多线程环境中被正确地加锁, 同步, 如果多个线程同时初始化一个类, 那么只有一个线程执行这个类的<clint>()方法, 其他线程需要阻塞等待, 直到线程执行完毕.(如果<clint>()方法中有耗时很长的操作, 可能造成多个进程的都塞)

## 类加载器

让我们回顾一下加载阶段完成的事:
+ 通过一个类的全限定名来获取此类的二进制字节流.
+ 将这个字节流所代表的静态存储结构转换为方法区的运行时的数据结构.
+ 在内存中生成一个代表这个类的Java.lang.Class对象, 作为方法区这个类的各种数据的访问入口.

虚拟机设计团队把加载的第一个阶段(`通过一个类的全限定名来获取此类的二进制字节流`)放到了Java虚拟机外部去实现, 以便让应用程序自己决定如何获取所需的类. 实现这个动作的代码模板称为`类加载器`.

### 类与类加载器

类加载器虽然只用于实现类的加载动作, 但在java程序中起到的作用, 却远远不限于类加载阶段. 对于任意一个类, 都需要由加载它的类加载器和这个类本身一同确定其在Java虚拟机中的唯一性, 每一个类加载器都拥有独立的类名称空间.(两个类是否`相等`(equals(), isAssignableFrom(), isInstance(), instanceof等), 只有在这两个类是被同一个类加载器加载的前提下才有意义, 否则, 就算来源于同一个Class文件, 同一个虚拟机加载, 类加载器不同, 那么这两个类必定不相等).

```java
public class ClassLoaderTest {
	public static void main(String[] args) throws Exception {
    	ClassLoader myLoader = new ClassLoader() {
        	@Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
            	try {
                	String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                    	return super.laodClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                	throw new ClassNotFoundException(name);
                }
            }
        };
        Object obj = myLoader.loading("org.cjyong.classloading.ClassLoaderTest).newInstance();
        System.out.println(obj.getClass()); // class org.cjyong.classloading.ClassLoadTest
        System.out.println(obj instanceof org.cjyong.classloading.ClassLoadTest); // false (后面的类是由系统的程序类加载器加载的)
    }
}
```

### 双亲委派模型

对于Java虚拟机来说, 只有两个不同的类加载器: 启动类加载器(Bootstrap ClassLoader), 这个加载器使用C++语言来实现(只限于HotSpot, 对于MRP,Maxine等本身就使用Java编写, 自然就是Java实现, 而对于J9和JRockit使用C实现), 是虚拟机的一部分. 另一种就是所有的其他类加载器, 这些类加载器都是由java语言实现, 独立于虚拟机外部, 继承自java.lang.ClassLoader.
对于Java开发人员来说, 系统提供的类加载器可以分为3种:
+ 启动类加载器(Bootstrap ClassLoader): 这个类负责加载${JAVA_HOME}\lib目录中(或者通过-Xbootclasspath指定的)的, 并且是虚拟机可以识别的(按照文件名识别, 如rt.har)类库加载到虚拟机内存中. 无法被Java程序直接引用. 如果用户自定义编写的类加载器, 需要把请求委托给该加载器,  直接使用null代替即可.
+ 拓展类加载器(Extension ClassLoader): 由sun.misc.Launcher$ExtClassLoader实现, 负责加载${JAVA_HOME}\lib\ext目录(或者被java.ext.dirs指定的)中的所有类库.
+ 应用程序类加载器(Application ClassLoader): 这个类加载器由sun.misc.Launcher$AppClassLoader实现, 这是ClassLoader中getSystemClassLoader()方法的返回值, 它负责加载用户类路径上所指定的类库, 开发者可以直接使用这个类加载器. 如果用户没有自定义类加载器, 这就是程序中默认的类加载器.

![show](https://image.cjyong.com/blog/jvm/23.png)

双亲委派模型: 每个类加载器(除了顶层的类加载器)都要有个类加载器(一个父亲, 一个自己(儿子)), 通过组合的形式引入父加载器. 每次来了新的加载请求, 先使用父加载器进行加载, 如果父亲没办法完成这个加载, 就自己进行处理.

使用这种模型有一个很明显的好处就是: 类的加载具有优先级. 如我们java.lang.Object放在rt.jar, 无论在那种环境下, 它都是由Bootstrap ClassLoader进行加载, 在任何环境下都是同一个类. 这保证了Java程序的稳定性, 代码实现:

```java
protected synchronized Class<?> loadClass (String name, boolean resolve) throws ClassNotFoundException {
	Class c = findLoadedClass(name); //检查是否已经加载过了
    if (c == null) {
    	try {
        	if (parent != null) {
            	parent.loadClass(name, false); //父加载器加载
            }
        } else {
        	c = findBootstrapClassOrNull(name); //判断是否为顶层类加载器
        }
        if (c == null) { //父加载器没有成功加载
        	c = findClass(name); //自己进行处理
        }
    }
    if (resolve) {
    	resolveClass(c);
    }
    return c;
}
```

### 破坏双亲委派模型

双亲委派模型不是一个强制的约束模型, 而是Java设计者推荐给开发者的类加载实现方式. 大多数的类加载器都遵守这个约定, 但是也有例外, 出现过3次大规模的破坏情况:
+ 第一次被破坏是在JDK1.2之前, 双亲委派模型是在JDK1.2之后被引入的, 而类加载器和抽象类java.lang.ClassLoader在JDK1.0就已经存在了, 为了向前兼容, JDK1.2之后给java.lang.ClassLoader添加了新的findClass()方法, 在JDK1.2之前, 人们继承java.lang.ClassLoader的唯一目的就是重写loadClass方法, JDK1.2之后, 不提倡用户覆盖改方法, 而是通过实现findClass()方法来保证所写的类加载器符合双亲委派模型.
+ 第二次是由于自身的缺陷导致: 如果基础类要调用用户的代码怎么办? 如JNDI服务(JDK1.3时放入rt.jar), JNDI的目的就是对资源进行集中的管理和查找, 它需要调用独立厂商实现并部署在应用程序的ClassPath下的JNDI接口提供者(SPI, Service Provider Interface)的代码, 但是启动类加载器`不认识`这些代码. 为了解决这个问题, Java设计团队引入了一个不太优雅的设计: 线程上下文类加载器(Thread Context ClassLoader), 这个类加载器可以通过java.lang.Thread类的setContextClassLoaser()方法进行设置, 如果创建线程时还未设置, 它将会从父线程中继承一个, 如果在应用程序的全局范围内都没有进行设置过的话, 那这个类加载器默认就是应用程序类加载器. JNDI就是通过这个线程上下文加载器去加载所需要的SPI代码, 即父类加载器去请求子类加载器去完成类加载动作(违反了双亲委派模型的一般原则), Java中所有涉及SPI加载到动作基本都使用这种方式, 如JNDI, JDBC, JCE, JAXB和JBI等.
+ 双亲委派模型的第三次`破坏`就是由于用户对程序动态性的追求导致的. 如 代码热替换(HotSwap), 模块热部署(Hot Deployment), 就像我们希望应用程序像电脑外设一样, 接上鼠标就可以使用, 鼠标坏了, 换一个重新接上就好了, 而不是重启或者停机. Sun公司提出的JSR-294, JSR-277规范与JCP组织的模块化规范之争中落败给JSR-291(OSGi R4.2), Sun公司不甘失去Java模块化开发的主导权, 独立发展Jigsaw项目, 但是OSGi已经成为了业界`事实上`的Java模块化标准. OSGi实现的模块热部署的关键就是它自定义的类加载机制的实现: 每一个程序模块(OSGi中称为Bundle)都有一个自己的类加载器, 当需要更换一个Bundle时, 就把Bundler连同类加载器一起换掉, 以实现代码的热部署. OSGi环境下, 类加载不在是简单的树状结构, 而是更加复杂的网状结构, 收到类加载请求时, OSGi按照以下顺序进行类搜索:
+ java.*开头的类委派给父类加载器加载.
+ 否则, 将委派列表名单内的类委派给父类加载器加载.
+ 否则, 将Import列表中的类委派给Export这个类的Bundler的类加载器加载.
+ 否则, 查找当前Bundle的ClassPath, 使用自己的类加载器加载.
+ 否则, 查找类是否在自己的Fragment Bundle中, 在, 委派给Fragment Bundle的类加载器加载.
+ 否则, 查找Dynamic Import列表中的Bundle, 委派给对应的Bundler的类加载器加载.
+ 否则, 查找失败.

OSGi中的类加载器不符合传统的双亲委派的类加载器, 为了实现热部署带来了额外的高复杂度. 但是OSGi对类加载器的使用还是很值得学习的, 只有弄懂了OSGi的实现, 就可以算是掌握了类加载器的精髓.

