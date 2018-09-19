# 早期(编译期)优化
## 概述
编译期的详细定义可能有点复杂: 前端编译期把*.java文件转变为*.class文件的过程; 虚拟机后端运行期编译器(JIT编译器, Just In Time Compiler)把字节码转变为机器码的过程; 还可能是使用静态提前编译器(AOT编译器, Ahead Of Time Compiler)直接把*.java文件编译成本地机器代码的过程.
+ 前端编译器: Sun的javac, Eclipse JDT中的增量式编译器(ECJ).
+ JIT编译器: HotSpot VM的C1, C2编译器.
+ AOT编译器: GNU Compiler for Java(GCJ), Excelsior JET.

这里主要关注的是第一类编译过程. 自从JDK1.3开始,Javac这类编译器对代码的运行效率基本上没有了任何优化措施, 因为虚拟机设计团队把性能的优化集中到了后端的即时编译器中, 这样可以让那些不是由Javac产生的Class文件(如JRuby, Groovy等语言的Class文件)也能享受到编译器优化带来的好处. 但是Javac做了许多针对Java语言编码过程的优化措施来改善程序员的编码风格和提高编码效率, 而不是依赖底层虚拟机的改进来支持.

## Javac编译器
### Javac的源码与调试
虚拟机规范严格定义了Class文件的格式, 但是<<Java虚拟机规范(第2版)>>中, 都是以举例子的形式进行描述, 并没有对如何把Java源码文件转变成Class文件的编译过程进行十分严格的定义, 这导致Class文件编译在某种程度上是与具体JDK实现相关的, 在一些极端情况下, 可能出现一段代码Javac编译器可以编译, 但是ECJ编译就不可以编译的问题. 从Sun Javac的代码来看, 编译的过程大致可以分为3个过程:
+ 解析与填充符号表过程
+ 插入式注解处理器的注解处理过程
+ 分析与字节码生成过程

主要的逻辑代码在com.sun.tools.javac.main.JavaCompiler类中的compile和compile2方法:

```java
/**
     * Main method: compile a list of files, return all compiled classes
     *
     * @param sourceFileObjects file objects to be compiled
     * @param classnames class names to process for annotations
     * @param processors user provided annotation processors to bypass
     * discovery, {@code null} means that no processors were provided
     */
    public void compile(List<JavaFileObject> sourceFileObjects,
                        List<String> classnames,
                        Iterable<? extends Processor> processors)
    {
        if (processors != null && processors.iterator().hasNext())
            explicitAnnotationProcessingRequested = true;
        // as a JavaCompiler can only be used once, throw an exception if
        // it has been used before.
        if (hasBeenUsed)
            throw new AssertionError("attempt to reuse JavaCompiler");
        hasBeenUsed = true;

        // forcibly set the equivalent of -Xlint:-options, so that no further
        // warnings about command line options are generated from this point on
        options.put(XLINT_CUSTOM + "-" + LintCategory.OPTIONS.option, "true");
        options.remove(XLINT_CUSTOM + LintCategory.OPTIONS.option);

        start_msec = now();

        try {
            initProcessAnnotations(processors); //准备过程, 初始化插入式注解处理器

            // These method calls must be chained to avoid memory leaks
            delegateCompiler =
                processAnnotations( //执行注解处理
                    enterTrees(stopIfError(CompileState.PARSE, //输入到符号表         parseFiles(sourceFileObjects))), //词法分析, 语法分析
                    classnames);

            delegateCompiler.compile2(); //分析与字节码生成
            delegateCompiler.close();
            elapsed_msec = delegateCompiler.elapsed_msec;
        } catch (Abort ex) {
            if (devVerbose)
                ex.printStackTrace(System.err);
        } finally {
            if (procEnvImpl != null)
                procEnvImpl.close();
        }
    }
    
/**
     * The phases following annotation processing: attribution,
     * desugar, and finally code generation.
     */
    private void compile2() {
        try {
            switch (compilePolicy) {
            case ATTR_ONLY:
                attribute(todo);
                break;

            case CHECK_ONLY:
                flow(attribute(todo));
                break;

            case SIMPLE:
                generate(desugar(flow(attribute(todo))));
                break;

            case BY_FILE: {
                    Queue<Queue<Env<AttrContext>>> q = todo.groupByFile();
                    while (!q.isEmpty() && !shouldStop(CompileState.ATTR)) {
                        generate(desugar(flow(attribute(q.remove()))));
                    }
                }
                break;

            case BY_TODO:
                while (!todo.isEmpty())
                    generate(desugar(flow(attribute(todo.remove()))));
                    //生成字节码,解语法糖, 数据流分析, 标注
                break;

            default:
                Assert.error("unknown compile policy");
            }
        } catch (Abort ex) {
            if (devVerbose)
                ex.printStackTrace(System.err);
        }

        if (verbose) {
            elapsed_msec = elapsed(start_msec);
            log.printVerbose("total", Long.toString(elapsed_msec));
        }

        reportDeferredDiagnostics();

        if (!log.hasDiagnosticListener()) {
            printCount("error", errorCount());
            printCount("warn", warningCount());
        }
    }
```

### 解析与填充符号表
主要由上图代码中的parseFiles()方法完成, 解析的步骤包括了经典程序编译原理中的词法分析和语法分析两个过程.

```java
            delegateCompiler =
                processAnnotations( //执行注解处理
                    enterTrees(stopIfError(CompileState.PARSE, //输入到符号表         parseFiles(sourceFileObjects))), //词法分析, 语法分析
                    classnames);
```

#### 词法,语法分析
词法分析是将源代码的字符流转变为标记(Token)集合, 单个字符是程序编写过程的最小元素, 而标记则是编译过程的最小元素, 关键字, 变量名, 字面量, 运算符都可以作为标记. 如`int a = b + 2`这句代码包含了6个标记, 分别是int, a, =, b, +, 2, 虽然关键字int由3个字符串构成, 但是它只是一个Token, 不可在拆分. 在Javac源码中, 词法分析中, 词法分析过程由com.sun.tools.javac.parser.Scanner类实现. 

语法分析是将Token序列构造抽象语法树的过程, 抽象语法树(Abstract Syntax Tree, AST)是一种用来描述程序代码语法结构的树形表示方式, 语法树的每一个节点都代表着程序代码中的一个语法结构(Construct)如: 包, 类型, 修饰符, 运算符, 接口, 返回值甚至代码注释. 在Javac源码中, 语法分析过程由com.sun.tools.javac.parser.Parser类实现, 这个阶段产出的抽象语法树由com.sun.tools.javac.tree.JCTree类进行表示. 经过这个步骤之后, 编译器基本不会对源码文件进行操作了, 后续的操作都建立在语法树之上.

#### 填充符号表
完成了语法分析和词法分析之后, 下一步就是填充符号表的过程, 由enterTree()方法所完成. 符号表(Symbol Table)是一组符号地址和符号信息构成的表格, 读者可以把它想象成哈希表中的K-V值对的形式.符号表中所登记的信息在编译的不同阶段都要用到. 在语义分析中, 符号表所登记的内容将用于语义检查(如检查一个名字的使用和原先说明是否一致)和产生中间代码. 在目标代码生成阶段, 当对符号名进行地址分配时, 符号表是地址分配的依据.

在Javac源代码中, 填充符号表的过程由com.sun.tools.javac.comp.Enter类实现, 此过程的出口是一个待处理的列表(To Do List), 包含了每一个编译单元的抽象语法树的顶级节点, 以及package-info.java(如果存在)的顶级节点.

### 注解处理器
在JDK1.5之后, Java语言提供了对注解(Annotation)的支持, 这些注解与普通的Java代码一样, 是在运行期间发挥作用. 在JDK1.6中, 实现了JSR-269规范, 提供了一组插入式注解处理器的标准API, 在编译期对注解进行处理, 我们可以把它看做一组编译器的插件, 在这些插件里面, 可以读取, 修改, 添加抽象语法树中的任意元素.  如果这些插件在处理注解区间对语法树进行了修改, 那么编译器将回到解析及填充符号表的过程重新处理, 知道所有的插入式注解处理器都没有再对语法树进行修改为止, 每次循环称为一个Round. 在Javac源码中, 插入式注解处理器的初始化过程是在initPorcessAnnotations()方法中完成, 而它的执行过程则是在processAnnotations()方法中完成的, 这个方法判断是否还有新的注解处理器需要执行, 如果有的话, 通过com
.sun.tools.javac.processing.JavacProcessingEnvironment类的doProcessing()方法生成一个新的JavaCompiler对象对编译的后续步骤进行处理.

### 语义分析与字节码生成
```java
generate(desugar(flow(attribute(todo.remove()))));
//生成字节码,解语法糖, 数据流分析, 标注
```

语法分析之后, 编译器获得了程序代码的抽象树表示, 语法树可以表示一个结构正确的源代码的抽象, 但是无法保证程序是符合逻辑的. 而语义分析的主要任务是对结构上正确的源程序进行上下文有关性质的审查, 如类型审查.

```java
int a = 1;
boolean b = false;
char c = 2;
//后续可能出现的操作
int d = a + c;
int d = b + c;
char d = a + c;
```
这些都可以构成结构正确的语法树, 但是只有第一种写法在语义上才是正确的, 可以通过编译的, 余下两种是不符合逻辑的.

#### 标注检查
主要是由attribute()方法完成, 检查内容包括变量使用前是否已经被声明, 变量与赋值之间的数据类型是否可以匹配等等. 在标注检查的步骤中, 还有一个很重要的动作就是常量折叠. 如: 我们定义了如下代码

```java
int a = 1 + 2;
```

那么我们在语法树上仍然可以看见字面量"1","2"以及操作符"+", 但是经过常量折叠之后, 他们将会被折叠为字面量"3".
标注检查步骤在Javac源码中实现类是com.sun.tools.javac.comp.Attr类和com.sun.tools.javac.comp.Check类.

#### 数据及控制流分析
数据及控制流分析是对程序上下文逻辑的进一步验证, 它可以检查程序局部变量是否在使用前进行了赋值, 方法的每条路径是否有返回值, 是否所有的受查异常都被正确处理了等问题. 如:

```java
public void foo(final int arg) {	//添加final修饰
	final int var = 0;
    // do something
}

public void foo(int arg) {	//没有添加final
	int var = 0;
    // do somenthing
}
```

这两个foo方法中, 第一个方法的参数和局部变量都使用了final修饰, 第二种没有, 在代码编写的时候程序肯定会受到final修饰符的影响, 不能改变arg和var变量的值. 但是这两段代码编译出来的Class文件是没有区别的, 局部变量与字段是有区别的, 它在常量池中没有CONSTANT_Fieldref_info的符号引用, 自然也没有访问标记(Access_Flags)的信息, 甚至可能连名称都不会保留, 自然在Class文件中不可能知道一个局部变量是不是被声明为final. 因此将局部变量声明为final, 对运行期是没有影响的, 变量的不变性仅仅由编译期间进行保障.

这个步骤由flow()方法完成, 具体操作由com.sun.tools.javac.comp.Flow类来完成.

#### 解语法糖
语法糖(Syntactic Sugar), 也称糖衣语法(英国计算机科学家Peter J.Landin发明)指在计算机语言中添加某种语法, 这种语法对语言的功能并没有影响, 但是方便程序员使用.
Java在现代编程语言之中属于"低糖语言"(相对于C#及许多其他JVM语言来说), 尤其是JDK 1.5之前的版本. java主要的语法糖包括: 泛型, 变长参数, 自动装箱/拆箱等.

在Javac的源码中, 解语法糖的过程由desugar()方法触发, 在com.sun.tools.javac.comp.TransType类和com.sun.tools.javac.comp.Lower类中完成.

#### 字节码生成
字节码生成是Javac编译过程的最后一个阶段, 在Javac源码里面由com.sun.tools.javac.jvm.Gen类来完成. 字节码生成阶段不仅仅是把前面各个步骤生成的信息(语法树, 符号表)转化为字节码写到磁盘中, 编译器还进行了少量的代码添加和转换工作.

如实例构造器<init>()方法和类构造器<clint>()方法就是在这个阶段添加到语法树之中的(默认的构造函数是在填充符号表时完成的, 不是在这个阶段). 这两个构造器的产生过程实际上是一个代码收敛的过程, 编译器会把语句块, 变量初始化, 调用父类的实例构造器等操作收敛到<init>()和<clinit>()方法之中, 并且保证一定是按照顺序执行. 除此之外, 还有其他的一些代码替换工作用于优化程序的实现逻辑, 如把字符串的加操作替换为StringBuffer或StringBuilder的append()操作等等.

完成了对语法树的遍历和调整之后, 就会把填充了所有所需信息的符号表交给com.sun.tools.javac.jvm.ClassWriter类, 由这个类writeClass()方法输出字节码, 生成最终的Class文件, 到此为止, 整个编译过程宣告结束.

## Java语法糖的味道
几乎各种语言或多或少都提供过一些语法糖来方便程序员的代码开发, 这些语法糖虽然不会提供实质性的功能改进, 但是它们或能提高效率, 或能提升语法的严谨性, 或能减少代码出错的机会. 也有人认为语法糖并不一定是有益的, 大量添加和使用"含糖"的语法, 容易让程序员产生依赖, 无法看清程序代码的真实面目. 总之, 使用语法糖可以很大地提升我们的开发的效率, 但我们也应该深入了解这些语法糖背后的真实世界, 才能更好地利用好它们.

### 泛型与类型擦除
泛型是JDK1.5的一项新特效, 本质是参数化类型(Parameterized Type)的应用, 也就是说所操作的数据类型被指定为一个参数. 这个参数可以用在类, 接口和方法的创建之中, 分别称为泛型类, 泛型接口和泛型方法.

在Java语言还处于没有出现泛型的版本, 通过Object(所有类的父类)和类型转换两个特点来实现类型泛化. 如在哈希表的存取中, JDK1.5之前使用HashMap的get()方法, 返回值就是一个Object对象,然后通过类型转换得到特定的类型. 正是由于这个特点, 这就具有了无限的可能性, 就只有程序员和运行期的虚拟机才知道这个Object到底是什么类型对象. 在编译期间, 编译器无法检测这个Object的强制类型转换是否成功, 许多ClassCastException的风险就转嫁给程序运行期之中.

泛型技术在C#和Java之中的使用方式看似相同, 但是实现上却有根本的分歧, C#里面泛型无论在程序源码中, 编译后的IL中(Intermediate Language, 中间语言, 泛型是一个占位符), 或是运行期的CLR中, 都是真实存在的, List<int>和List<String>就是两个不同的类型, 它们在系统运行期生成, 有自己的虚方法和类型数据, 这种实现称为类型膨胀, 基于这种方法实现的泛型称为真实泛型.

Java语言中的泛型则不一样, 它只在程序源码中存在, 在编译后的字节码中已经替换为原来的原生类型(Raw Type, 也称裸类型)了, 并且在相应的地方插入了强制类型转换的代码. 因此对于运行期的Java语言来说, ArrayList<int>和ArrayList<String>就是同一个类, 所以泛型技术其实是Java语言的语法糖, Java语言中的泛型实现方法称为类型擦除, 基于这种方法实现的泛型称为伪泛型.

```java
public static void main(String[] args) {
	Map<String, String> map = new HashMap<String, String>();
    map.put("hello", "你好");
    map.put("hello", "你好");
    System.out.println(map.get("hello"));
    System.out.println(map.get("how are you")):
}
```

这段代码编译成成Class文件, 在通过反编译工具进行反编译后, 将会发现泛型都不见了, 程序又变回了Java泛型出现之前的写法, 泛型类型都变回了原生类型.

```java
public static void main(String[] args) {
	Map map = new HashMap();
        map.put("hello", "你好");
    map.put("hello", "你好");
    System.out.println((String) map.get("hello"));
    System.out.println((String) map.get("how are you")):
}
```

通过擦除法来实现泛型丧失了一些泛型思想应有的优雅. 如:

```java
    public static void method(List<String> list) {
        System.out.println("invoke method(List<String> list)");
    }

    public static void method(List<Integer> list) {
        System.out.println("invoke method(List<String> list)");
    }
```

这段代码时拒绝编译的, 因为参数List<Integer>和参数List<String>编译之后都被擦除了, 变成了一样的原生类型List<E>, 擦除导致这两种方法的特征签名变得一模一样.

由于Java泛型的引入, 各种场景(虚拟机解析, 反射等)下的方法调用都可能对原有的基础产生影响和新的需求, 如在泛型类中如何获取传入的参数化类型等等. 因此, JCP组织对虚拟机规范做出了新的修改, 如引入Signature, LocalVariableTypeTable等新的属性用于解决随着泛型带来的参数类型识别的问题. Signature是其中一个属性, 作用是存储一个方法在字节码层面的特征签名, 这个属性保存的参数类型并不是原生类型, 而是包括了参数化类型的信息. 修改后的虚拟机规范要求所有能识别49.0以上版本的Class文件虚拟机都要能正确识别Signature参数.

从Signature属性的出现我们可以得出结论, 擦除法所谓的擦除, 仅仅是对方法的Code属性中的字节码进行擦除, 实际上元数据还保留了泛型信息, 这也是我们可以通过反射手段取得参数化类型的根本依据.

### 自动装箱, 拆箱与遍历循环
```java
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4);
        int sum = 0;
        for (int i : list) {
            sum += i;
        }
        System.out.println(sum);
    }
```

编译之后:

```java
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(
        	new Integer[] {
            	Integer.valueOf(1), 
                Integer.valueOf(2), 
                Integer.valueOf(3), 
                Integer.valueOf(4) });
        int sum = 0;
        for (Iterator localIterator = list.iterator(); localIterator.hasNext(); ) {
        	int i = ((Integer) localIterator.next()).intValue();
            sum += i;
        }
        System.out.println(sum);
    }
```

这里需要注意一下自动装箱和拆箱的陷阱:

```java
    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        Integer c = 3;
        Integer d = 3;
        Integer e = 321;
        Integer f = 321;
        Long g = 3L;
        System.out.println(c == d); //True, 缓存池中存储-128-127
        System.out.println(e == f); //False, 缓存池中未存储该对象, 新建对象, 地址有差别
        System.out.println(c == (a + b)); //True, (a + b) : new Integer(a.intValue() + b.intValue()); 缓存池存储对象
        System.out.println(c.equals(a + b)); //True, 先判断类型, 在判断值
        System.out.println(g == (a + b)); //True, (a + b)转型为long, 进行判断
        System.out.println(g.equals(a + b)); //False, 首先判断类型, 在判断值
    }
```

### 条件编译
许多程序设计语言都提供了条件编译的途径, 如C, C++中使用预处理器指示符(#ifdef)来完成条件编译. C和C++的预处理器最初的任务是为了解决编译时代码依赖关系. 而在Java语言中并没有使用预处理器, 因为Java语言天然的编译方式(不是一个个进行编译, 而是将所有的编译单元的语法树顶级节点输入到待处理列表后再进行编译, 因此各个文件之间可以提供符号信息)无须使用预处理器.

Java语言当然也可以使用条件编译, 方法就是使用条件为常量的if语句.

```java
public static void main(String[] args) {
	if (true) {
    	System.out.println("1");
    } else {
    	System.out.println("2");
    }
}
```

编译后:

```java
public staitc void main() {
	System.out.println("1");	
}
```

只有使用if语句才可以达到上述效果, 其他常量或条件判断, 则可能出现错误.
```java
while(false) { //Unreachable code
	//do something
}
```

Java语言中的条件编译的实现, 也是Java语言的语法糖, 根据布尔常量的真假, 编译器会将分支中国不成立的代码块消除, 这一工作将在编译器解决语法糖的阶段(sun.tools.javac.comp.Loser类)完成. 由于这种条件编译使用了if语句, 所以必须遵循基本的Java语法, 写在方法内部, 因此它只能实现语句基本块级别的条件编译. 没办法根据条件调整整个Java类的结构.

## 插入式注解处理器
### 实战目标
通过阅读Javac编译器的源码, 我们知道编译器在把Java程序源码编译为字节码的时候, 会对Java程序进行各方面的检查校验, 这些检查主要以程序"写的对不对"为出发点, 虽然有各种WARNING信息, 但总体来说, 很少去校验程序"写的好不好". 有鉴于此, 业界出现了许多针对程序"写的好不好"的辅助校验工具, 如CheckStyle, FindBug, Klocwork等等. 这些校验工具有一些是基于Java的源码进行校验, 还有一些事通过扫描字节码来完成的, 这里我们通过注解处理器的API编写一款拥有自己编码风格的校验工具: NameCheckProcessor.

这里简单以驼峰命名法来完成以下校验: 类(或接口)驼峰写法,是首字母大写. 方法: 驼峰写法, 首字母小写. 字段: 类或实例变量: 符合驼峰写法, 首字母小写; 常量: 要求全部大写字母或下划线, 第一个字母不能是下划线.

### 代码实现
```java
@SupportedAnnotationTypes("*") //这个注解器对所有的注解感兴趣
@SupportedSourceVersion(SourceVersion.RELEASE_8) //只支持JDK1.8版本的Java代码
public class NameCheckProcessor extends AbstractProcessor {

    private NameChecker nameChecker;

    /**
     *
     * 初始化名称检查插件
     *
     * @param processingEnv
     */
    @Override
    public void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        nameChecker = new NameChecker(processingEnv);
    }

    /**
     * 对输入语法树的各个节点进行名称检查
     *
     * @param annotations
     * @param roundEnv
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (!roundEnv.processingOver()) {
            for (Element element : roundEnv.getRootElements()) {
                nameChecker.checkNames(element);
            }
        }
        return false; //表示没有对Round中的代码进行修改, 不用重新生成JavaCompiler
    }

    /**
     * 检查程序名称插件, 如果程序不合规范, 将会输出一个编译器的WARNING信息.
     */
    private static class NameChecker{

        private final Messager messager;

        NameCheckerScanner nameCheckScanner = new NameCheckerScanner();

        NameChecker(ProcessingEnvironment processingEnv) {
            this.messager = processingEnv.getMessager();
        }

        public void checkNames(Element element) {
            nameCheckScanner.scan(element);
        }

        private class NameCheckerScanner extends ElementScanner8<Void, Void> {
            /**
             * 用于检查Java类
             * @param e
             * @param p
             * @return
             */
            @Override
            public Void visitType(TypeElement e, Void p) {
                scan(e.getTypeParameters(), p);
                checkCamelCase(e, true);
                return super.visitType(e, p);
            }

            /**
             * 检查方法命名是否合法
             *
             * @param e
             * @param p
             * @return
             */
            @Override
            public Void visitExecutable(ExecutableElement e, Void p) {
                if (e.getKind() == METHOD) {
                    Name name = e.getSimpleName();
                    if (name.contentEquals(e.getEnclosingElement().getSimpleName())) {
                        messager.printMessage(WARNING,
                                "一个普通方法 "
                                        + name
                                        + "不应该与类名重复, 避免与构造函数产生混淆",
                                e);
                    }
                }
                return super.visitExecutable(e, p);
            }

            @Override
            public Void visitVariable(VariableElement e, Void aVoid) {
                if (e.getKind() == ENUM_CONSTANT || e.getConstantValue() != null || heuristicallyConstant(e)) {
                    checkAllCaps(e);
                } else {
                    checkCamelCase(e, false);
                }
                return super.visitVariable(e, aVoid);
            }

            /**
             * 判断一个变量是否为常量
             * @param e
             * @return
             */
            private boolean heuristicallyConstant(VariableElement e) {
                if (e.getEnclosingElement().getKind() == INTERFACE) {
                    return true;
                } else if (e.getKind() == FIELD
                        && e.getModifiers().containsAll(EnumSet.of(PUBLIC, STATIC, FINAL))) {
                    return true;
                } else {
                    return false;
                }
            }

            /**
             * 检查传入的Element是否为驼峰命名法, 如果不符合, 则输出警告信息
             * @param e
             * @param initialCaps
             */
            private void checkCamelCase(Element e, boolean initialCaps) {
                String name = e.getSimpleName().toString();
                boolean previousUpper = false;
                boolean conventional = true;
                int firstCodePoint = name.codePointAt(0);
                if (Character.isUpperCase(firstCodePoint)) {
                    previousUpper = true;
                    if (!initialCaps) {
                        messager.printMessage(WARNING, "名称" + name +"应当以小写字母开头", e);
                        return;
                    }
                } else if (Character.isLowerCase(firstCodePoint)){
                    if (initialCaps) {
                        messager.printMessage(WARNING, "名称" + name +"应当以大写字母开头", e);
                        return;
                    }
                } else {
                    conventional = false;
                    if (conventional) {
                        int cp = firstCodePoint;
                        for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                            cp = name.codePointAt(i);
                            if (Character.isUpperCase(cp)) {
                                if (previousUpper) {
                                    conventional = false;
                                    break;
                                }
                                previousUpper = true;
                            } else {
                                previousUpper = false;
                            }
                        }
                        if (!conventional) {
                            messager.printMessage(WARNING, "名称" + name +"应当符合驼峰命名法", e);
                        }
                    }
                }
            }

            /**
             * 大写命名检查,要求第一个必须是大写, 其余的可以是下划线或大写字母
             * @param e
             */
            private void checkAllCaps(Element e) {
                String name = e.getSimpleName().toString();
                boolean conventional = true;
                int firstCodePoint = name.codePointAt(0);

                if (!Character.isUpperCase(firstCodePoint)) {
                    conventional = false;
                } else {
                    boolean previousUnderscore = false;
                    int cp = firstCodePoint;
                    for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                        cp = name.codePointAt(i);
                        if (cp == (int) '_') {
                            if (previousUnderscore) {
                                conventional = false;
                                break;
                            }
                            previousUnderscore = true;
                        } else {
                            previousUnderscore = false;
                            if (!Character.isUpperCase(cp) && !Character.isDigit(cp)) {
                                conventional = false;
                                break;
                            }
                        }
                    }
                }
                if (!conventional) {
                    messager.printMessage(WARNING, "名称" + name +"应当全部以大写字母或下划线命名, 并且以字母开头", e);
                }
            }
        }
    }
}
```

### 运行与测试
首先进行编译我们的代码

```java
javac -encoding UTF-8 com/cjyong/jvm/checker/NameCheckProcessor.java
```

我们使用一份很不规范的代码:

```java
public class BADLY_NAMED_CODE {
	enum clors {
		red, blue, green;
	}
	
	static final int _FORTY_TWO = 32;
	
	public static int NOT_A_CONSTANT = _FORTY_TWO;
	
	protected void BADLY_NAMED_CODE() {
		return;
	}
	
	public void NOTCamelCASEmethodNAME() {
		return;
	}
}
```

进行编译:
```java
javac -processor com.cjyong.jvm.checker.NameCheckProcessor BADLY_NAMED_CODE.java
```

结果如下:

![show](https://image.cjyong.com/blog/jvm/26.png)





