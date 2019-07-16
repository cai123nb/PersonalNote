# 类文件结构

## 概述

"我们知道计算机只认识0和1, 所以我们的程序需要经编译器翻译成0和1构成的二进制格式才能由计算机执行", 随着10多年时间的过去, 今天的计算机仍然只能识别0和1, 但是我们编写的程序编译成二进制本地代码(Native Code)不再是唯一的选择, 越来越多的程序选择编译成与平台无关的中立格式作为程序编译后的存储格式.

## 无关性的基础

实现语言无关性的基础仍然是虚拟机和字节码存储格式. Java虚拟机不和Java在内的任何语言关联, 它只和`Class文件`(一种特定的二进制文件格式)关联. Class文件包含了Java虚拟机指令集和符号表以及若干其他辅助信息, 基于安全考虑, JAVA虚拟机规范要求在Class文件中使用许多强制性的语法和结构化约束, 但是任何一门功能性的语言都可以表示为一个能被Java虚拟机所接受的有效的Class文件. 作为一个通用的, 机器无关的执行平台, 其他任何语言的实现者都可以将Java虚拟机作为语言的产品交付媒介. 如Java编译器可以把Java代码编译成存储字节码的Class文件, 而JRuby, Groovy等语言的编译器同样可以把程序编译成Class文件, 虚拟机不关心Class文件的来源.

## Class类文件的结构

具体资料请参考周志明老师的著作`<<Java虚拟机规范(Java SE 7)>>`, Class文件是一组以8位字节为基础单位的二进制流, 每个数据项目严格按照顺序紧凑地排列在Class文件之中, 中间没有添加任何分隔符. 当遇到需要占用8位字节以上空间的数据项时, 就会按照高位在前(Big-Endian)的方式分割成若干个8位字节进行存储.
根据Java虚拟机规范的规定, Class文件格式采用一种类似C语言结构体的伪结构来存储数据, 这种伪结构只有两种数据类型: 无符号数和表.
无符号数: 基本数据类型, 这里用u1, u2, u4, u8分别代表1个字节,2字节, 4个字节和8个字节的无符号数, 用于描述数字, 索引引用, 数量或者UTF-8编码的字符串值.
表: 多个符号是或者其他表作为数据项构成的复合数据类型, 所有的表习惯性以"_info"结尾. 用来描述具有层次结构的数据. 如整个Class文件的本质:

|       类型       |         名称          | 数量                      |
|----------------|:-------------------:|:------------------------|
| u4             |        magic        | 1                       |
| u2             |    minor_version    | 1                       |
| u2             |    major_version    | 1                       |
| u2             | constant_pool_count | 1                       |
| cp_info        |    constant_pool    | constant_pool_count - 1 |
| u2             |    access_flags     | 1                       |
| u2             |     this_class      | 1                       |
| u2             |     super_class     | 1                       |
| u2             |  interfaces_count   | 1                       |
| u2             |     interfaces      | interfaces_count        |
| u2             |    fields_count     | 1                       |
| field_info     |       fields        | fields_count            |
| u2             |    methods_count    | 1                       |
| method_info    |       methods       | methods_count           |
| u2             |  attributes_count   | 1                       |
| attribute_info |     attributes      | attributes_count        |

接下来我们就根据这个表从头到尾解析一遍Class文件. 这里我们引入我们的代码:

```java
package cjyong.classz;
public class TestClass {
	
	private int m;
	
	public int inc() {
		return m + 1;
	}
	
	synchronized public int setM(int m) {
		try {
			this.m = 12 / m;
			return this.m;
		} catch (Exception e) {
			this.m = 1;
			return this.m;
		} finally {
			this.m = 2;
			return this.m;
		}
	}
}
```

这里使用javac TestClass.java进行编译(读者使用的是JDK1.8), 生成TestClass.class文件, 利用WinHex打开TestClass.class文件:

![show](https://image.cjyong.com/blog/jvm/4.png)

### 魔数和版本

这里解析第一行的:
+ **CA FE BA BE 00 00 00 34**
+ 由Class文件表可知, 前4个字节(u4)是为mageic number: 0 x CAFEBABE, 对于Java的魔数为CAFEBABE, 一个充满了浪漫气息的魔数.
+ 接下来的是u2(两个字节的)minor version: 小版本号为00 00
+ 然后是u2的major version: 00 34 换算成十进制为 52.

> Java的版本号从45开始, 从JDK1.1开始的每个JDK大版本向上加一(如JDK1.0 - 1.1使用: 45.0 - 45.3), 高版本的JDK可以向下兼容以前版本的Class文件, 但不能运行以后版本的Class文件, 即使文件格式没有丝毫的改变, 虚拟机也必须拒绝运行. 52 - 45 = 7, 正好代表是JDK1.8

### 常量池

![show](https://image.cjyong.com/blog/jvm/8.png)

根据Class文件表, 我们知道第四个对象为u2的常量池的对象数量, 我们依次查找过去为:
+ **00 1B **
+ 换算成十进制为16 + 11 = 27, 我们可以知道常量有26(27 - 1)个常量, 这里预留了一个0空位, 预留的0空位常量主要满足后面某些指向常量池索引值的数据在特定情况下需要表达 " 不引用任何一个常量池项目"的含义, 这种情况下就可以把索引值置为0来表示.

常量池里面的常量是有不同的类型常量的, 每个类型的常量具体存储的内容也是不一样的, 这里用一个表详细介绍:

![show](https://image.cjyong.com/blog/jvm/5.png)

详细内容表:

![show](https://image.cjyong.com/blog/jvm/6.png)
![show](https://image.cjyong.com/blog/jvm/7.png)

最后后3个常量类型是在JDK1.7之后, 为了更好地支持动态语言的调用, 额外添加的三种类型. 依据这个表我们可以把我们的27个常量进行解析出来:
1. **0A 00 05 00 15**
+ 0A: 表示为CONSTANT_Methodref_info类型的常量, 参照详细列表, 读取内部信息.
+ 00 05 : 换算成十进制: 5, 指向第5个常量, 该常量为一个CONSTANT_Class_info类型.
+ 00 15 : 换算十进制: 21, 指向第21个常量, 该常量是一个CONSTANT_NameAndType类型.
2. **09 00 04 00 16**
+ 09: 表示为CONSTANT_Fieldref_info类型常量
+ 00 04 : 换算十进制: 4, 指向第4个常量, 该常量为CONSTANT_Class_info类型.
+ 00 16 : 换算十进制: 22, 指向第22个常量, 该常量是CONSTANT_NameAndType类型.
3. **07 00 17**
+ 07: 表示为CONSTANT_Class_info类型常量
+ 00 17: 换算十进制: 23, 指向第23个常量, 该常量为全限定名常量项索引类型.
4. **07 00 18**
+ 07: 表示为CONSTANT_Class_info类型常量
+ 00 18: 换算十进制: 24, 指向第24个常量, 该常量为全限定名常量项索引类型.
5. **07 00 19**
+ 07: 表示为CONSTANT_Class_info类型常量
+ 00 19: 换算十进制: 25, 指向第25个常量, 该常量为全限定名常量项索引类型.
6. **01 00 01 6D**
+ 01: 表示为CONSTANT_Utf8_info类型常量
+ 00 01: 换算十进制: 1, UTF-8字符串占用字节数1字节.
+ 6D: UTF-8编码字符串: m.
7. **01 00 01 49**
+ 01: 表示为CONSTANT_Utf8_info类型常量
+ 00 01: 换算十进制: 1, UTF-8字符串占用字节数1字节.
+ 49: UTF-8编码字符串: I.
8. **01 00 06 3C 69 6E 69 74 3E**
+ 01: 表示为CONSTANT_Utf8_info类型常量
+ 00 06: 换算十进制: 6, UTF-8字符串占用字节数6字节.
+ 3C 69 6E 69 74 3E: UTF-8编码字符串: <init>.
9. **01 00 03 28 29 56**
+ 01: 表示为CONSTANT_Utf8_info类型常量
+ 00 03: 换算十进制: 3, UTF-8字符串占用字节数3字节.
+ 3C 69 6E 69 74 3E: UTF-8编码字符串: ()V.
10. **01 00 04 43 6F 64 65**: Code.
11. **01 00 0F 4C 69 6E 65 4E 75 6D 62 65 72 54 61 62 6C 65**: LineNumberTable.
12. **01 00 03 69 6E 63**: inc.
13. **01 00 03 28 29 49**:()I.
14. **01 00 04 73 65 74 4D**: setM.
15. **01 00 04 28 49 29 49**: (I)I.
16. **01 00 0D 53 74 61 63 6B 4D 61 70 54 61 62 6C 65**: StackMapTable.
17. **07 00 17**
+ 07: 表示为CONSTANT_Class_info类型常量.
+ 00 17:  换算十进制: 23, 执行第23个常量, 该常量为全限定名常量项.
18. **07 00 1A**
+ 同理指向26.
19. **01 00 0A 53 6F 75 72 63 65 46 69 6C 65**: SourceFile.
20. **01 00 0E 54 65 73 74 43 6C 61 73 73 2E 6A 61 76 61**: TestClass.java.
21. **0C 00 08 00 09**
+ 0C: 表示为CONSTANT_Name-AndType_info类型常量.
+ 00 08: 换算十进制: 8, 字段或方法名称常量, 指向第8个常量.
+ 00 09: 换算十进制: 9, 字段或者方法描述常量, 指向第9个常量.
22. **0C 00 06 00 07**:同理, 分别指向第6和第7个常量.
23. **01 00 13 6A 61 76 61 2F 6C 61 6E 67 2F 45 78 63 65 70 74 69 6F 6E**: java/lang/Exception.
24. **01 00 17 63 6A 79 6F 6E 67 2F 63 6C 61 73 73 7A 2F 54 65 73 74 43 6C 61 73 73**: cjyong/classz/TestClass.
25. **01 00 10 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A 65 63 74**: java/lang/object.
26. **01 00 13 6A 61 76 61 2F 6C 61 6E 67 2F 54 68 72 6F 77 61 62 6C 65**: java/lang/Throwable.

### 访问标记

常量池结束之后, 紧接着的u2两个字节代表访问标记(access_flags), 这个标记用来识别这个类或者接口的访问信息: 这是接口还是类, 是否为public, 是否定义为abstract, 是否是final等.

|      标记名称      |  标记值   | 含义                                         |
|----------------|:------:|:-------------------------------------------|
| ACC_PUBLIC     | 0x0001 | 是否public类型                                 |
| ACC_FINAL      | 0x0010 | 是否为final, 只有类可以设置                          |
| ACC_SUPER      | 0x0020 | 是否允许使用invokespecial字节码的新语意, JDK1.0.2之后必须为真 |
| ACC_INTERFAC   | 0x0200 | 标识接口                                       |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract, 对于抽象类和接口为true, 其余为false       |
| ACC_SYNTHETIC  | 0x1000 | 标识这个类不是由用户产生                               |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                   |
| ACC_ENUM       | 0x4000 | 标记这是一个枚举                                   |

查看我们的访问标记为: **00 21**, 为public, q且符合ACC_SUPER.

### 类索引, 父类索引与接口索引集合

接下来的是2个u2字节: 类索引, 父类索引. Clas文件由这两项数据来确定这个类的继承关系. 接下来是一组u2的接口索引, 因为Java可以实现多实现接口, 所以可能有很多组接口实现, 第一个u2数据为interface_count, 表明接口的数量, 然后按照接口的从左向右的次序依次存储. 这里存储的都是指向常量池中的索引.
我们本地的代码为: **00 04 00 05 00 00**
+ 00 04: 类索引, 指向第四项常量常量, 查询之前的常量表为: cjyong/classz/TestClass.
+ 00 05: 父类索引, 指向第5项常量, 查询常量表为: java/lang/object.(注: 只有Object没有父类索引)
+ 00 00: 接口数量, 为0个, 后面没有对应的索引.

### 字段表集合

字段表(field_info)用于描述接口或者类中声明的变量. 字段(field)包括类级变量以及实例变量, 但不考虑方法内部的变量和父类的变量.
首先有一个u2的fields_count, 表明字段表中成员的数量.每个字段表主要的结构有:
u2的access_flag + u2的name_index + u2的descriptor_index + u2的attributes_count + attributes_count个attribute_info.
其中access_flag标准为:

|     标记名称      |  标记值   | 含义           |
|---------------|:------:|:-------------|
| ACC_PUBLIC    | 0x0001 | 是否public类型   |
| ACC_PRIVATE   | 0x0002 | 是否为private   |
| ACC_PROTECTED | 0x0004 | 是否为protected |
| ACC_STATIC    | 0x0008 | 是否为static    |
| ACC_FINAL     | 0x0010 | 是否为final     |
| ACC_VOLATITLE | 0x0040 | 是否为volatile  |
| ACC_TRANSIENT | 0x0080 | 是否为transient |
| ACC_SYNTHETIC | 0x1000 | 是否由编译器自动产生   |
| ACC_ENUM      | 0x4000 | 是否为enum      |

而name_index和descriptor_index为指向常量池中的引用, 分别代表着字段的简单名称以及字段和方法的描述符.

> + 全限定名: 详细具体的名称, 如cjyong/classz/TestClas,只不过将.替换成了/, 如果有多个的话, 在最后一个添加;表示结束.
> + 简单名称: 没有类型和参数休息的方法或者字段名称. 如, inc()何m的简单名称为: inc和m.

方法的描述符则复制一点, 描述符用来描述字段的数据类型, 方法的参数列表(数量, 类型顺序)和返回值.所有的数据类型都用一个大写的字符来表示:
+ 基本类型 B :byte;C:char;D:double;F:float;I:int;J:long;S:short;Z:boolean;
+ 特殊类型 V:void;L:对象类型, 如Ljava/lang.Object.

对于数组类型, 每一个维度添加一个前置`[`来进行描述, 如java.lang.String[][]的描述为`[[Ljava/lang/String;`, int[][][]的描述为`[[[I`.

如果用来描述方法的话, 按照先参数,后返回值的顺序进行描述. 参数按照顺序放入`()`之中. 如方法java.lang.String toString()的描述符为`()Ljava/lang/String;`, 方法int indexOf(char[]source, int sourceOffset, int sourceCount, char[] target, int targetOffset, int tartgetCount, int fromIndex)的描述符为: `([CII[CIII)I`.

这里查看我们的代码为:
** 00 01 00 02 00 06 00 07 00 00**
+ 00 01: field_count, 表示只有一个变量.
+ 00 02: access_tag, 表示为private.
+ 00 06: name_index, 指向第6个常量, 查询常量池为: m. 表明变量为m.
+ 00 07: descriptor_index, 指向第7个常量, 查询得: I. 表示为int类型.
+ 00 00: attribute_count, 为0, 表示没有额外的属性. 注意: 如果我们定义为final static int m = 123. 这里会存储一项ConstValue的值指向常量123.

> 字段表不会列出超类或者父接口中继承得来的字段, 但是在内部类中为了保持对外部类的访问性, 会自动添加外部类的实例字段(即可能出现非类中字段). Java中不能使用相同的名称(无论数据类型或者修饰符是否相同), 但是对于字节码来说, 只要描述符不同, 重名就是合法的.

### 方法表集合

接下来就是方法表的内容了, 首先有一个u2的method_count表明方法表中的方法数量. 每个方法表的成员基本和成员对象描述一模一样:
u2的access_flag + u2的name_index + u2的descriptor_index + u2的attributes_count + attributes_count个attribute_info.

但是access_flag略有不同, 没有ACC_VOLATILE和ACC_TRANSIENT标志,但是多了ACC_SYNCHRONIZED, ACC_NATIVE, ACC_STRICTER, ACC_ABSTRACT.

|       标记名称       |  标记值   | 含义              |
|------------------|:------:|:----------------|
| ACC_PUBLIC       | 0x0001 | 是否public类型      |
| ACC_PRIVATE      | 0x0002 | 是否为private      |
| ACC_PROTECTED    | 0x0004 | 是否为protected    |
| ACC_STATIC       | 0x0008 | 是否为static       |
| ACC_FINAL        | 0x0010 | 是否为final        |
| ACC_SYNCHRONIZED | 0x0020 | 是否为synchronized |
| ACC_BRIDGE       | 0x0040 | 是否是由编译器产生的桥接方法  |
| ACC_VARARGS      | 0x0080 | 方法是否接受不定参数      |
| ACC_NATIVE       | 0x0100 | 方法是否为native方法   |
| ACC_ABSTRACT     | 0x0400 | 是否为abstract     |
| ACC_STRICTFP     | 0x0800 | 是否为strictfp     |
| ACC_SYNTHETIC    | 0x1000 | 是否由编译器自动产生      |

其他的对象和成员的描述符一样, 但是方法的代码是存放在属性集合中的Code属性中.属性集合(attribute_info)下一节详细讲解.查看我们的代码:
** 00 03 00 01 00 08 00 09 00 01 **
+ 00 03: method_count, 3, 表明有3个方法.
+ 以下是第一个方法的信息:
+ 00 01: access_flag, 表明第一个方法是public.
+ 00 08: name_index, 指向第8个常量, <init>
+ 00 09: descriptor_index, 指向第9个常量, ()V, 可知是零参数, 返回值为void.
+ 00 01: attribute_num, 表明有一个额外的属性.(属性下节详讲, 这里先不显示了)

### 属性表集合

属性表(attribute_info)被Class文件, 字段表, 方法表之中进行携带, 用于描述某些场景专用的信息. 与Class文件中其他数据项目要求的严格顺序, 长度和内容不同, 属性表集合的限制稍微宽松一点, 不在要求严格的顺序, 并且不与现在的属性名重复, 任何人都可以写入自己定义的属性信息, java虚拟机运行时会忽略掉他不认识的属性.为了可以正确解析Class文件, Java虚拟机规范中预定义了9项虚拟机应该识别的属性, Java SE7中已经增加到了21项:

|                属性名称                |    使用位置     | 含义                                                                                  |
|------------------------------------|:-----------:|:------------------------------------------------------------------------------------|
| Code                               |     方法表     | Java代码编译成的字节码指令                                                                     |
| ConstantValue                      |     字段表     | final关键字定义的常量值                                                                      |
| Deprecated                         | 类, 方法表, 字段表 | 被声明为deprecated                                                                      |
| Exceptions                         |     方法表     | 方法抛出的异常                                                                             |
| EnclosingMethod                    |     类文件     | 局部类或者匿名类才可以拥有这个属性, 用于标记该类外围方法                                                       |
| InnerClasses                       |     类文件     | 内部类列表                                                                               |
| LineNumberTable                    |   Code属性    | Java源码的行号与字节码指令对应关系                                                                 |
| LocalVariableTable                 |   Code属性    | 方法的局部变量描述                                                                           |
| StackMapTable                      |   Code属性    | JDK1.6中新增的属性, 供新的类型检测器(Type Checker)检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配                   |
| Signature                          | 类, 方法表, 字段表 | JDK1.5中新增的属性, 这个属性用于支持泛型情况下的方法签名, 由于Java的泛型采用擦除法施行, 在为了避免类型信息被擦除之后导致签名混乱, 用这个属性进行记录 |
| SourceFile                         |     类文件     | 记录源文件的名称                                                                            |
| SourceDebugExtension               |     类文件     | JDK1.6中新增属性, 用于存储额外的调试信息.                                                           |
| Synthetic                          | 类, 方法表, 字段表 | 标识方法或字段为编译器自动生成的                                                                    |
| LocalVariableTypeTable             |      类      | JDK1.5中新增的属性, 它使用特征签名代替描述符, 是为了引入泛型语法之后能描述泛型参数化类型而添加的                               |
| RuntimeVisibleAnnotations          | 类, 方法表, 字段表 | JDK1.5中新增的属性, 为动态注解提供支持. 实际上运行时进行反射调用                                               |
| RuntimeInvisibleAnnotations        | 类, 方法表, 字段表 | JDK1.5中新增的属性, 与上一个属性正好相反, 标明那些注解运行时不可见                                              |
| RuntimeVisibleParameterAnnotations |     方法表     | JDK1.5中新增, 作用与RuntimeVisibleAnnotation类似, 只不过作用对象为方法参数                              |
| RuntimeInvisibleParamterAnnotation |     方法表     | JDK1.5新增, 类似                                                                        |
| AnnotationDefault                  |     方法表     | JDK1.5中新增属性, 用于记录注解类元素的默认值                                                          |
| BootstrapMethods                   |     类文件     | JDK1.7新增, 用于保存invokedynamic指令引用的引导方法限定符                                             |

并且对于每一个属性, 基本结构为:
u2的attribute_name_index(指向常量池中CONSTANT_Utf8_info对象) + u4的attribute_length + attribute_length个u1的info.

#### Code属性

这里我们重新回到原来的代码中, 我们代码中的第一个方法, 存在一个格外属性:

![show](https://image.cjyong.com/blog/jvm/9.png)

而Code属性的结构又是怎么样的呢:
u2的attribute_name_index(指向常量池中CONSTANT_Utf8_info对象) + u4的attribute_length(前面一样, 符合属性表的基本结构, 区别在于info内容的差别) + u2的max_stack + u2的max_local + u4的code_length + code_length个u1的code + u2的exception_table_length + exception_table_length个exceptiontable(在exception_info + u2的attributes_count + attributes_count个attributes(在attribute_info中).

详细解析:
+ attribute_name_index: 指向CONSTANT_Utf8_info类型的常量的索引, 这里应该固定为"Code".
+ attribute_length: 为整个属性表的长度 - 6(attribute_name_index占据2, attribute_length占据4).
+ max_stack: 操作数的栈(Operand Stacks)深度的最大值. 在这个方法执行中任意时刻, 操作数栈都不能超过这个深度, 虚拟机运行时需要根据这个值进行分配栈帧中操作数栈的深度.
+ max_locals: 局部变量所需的存储空间. 这里的单位为Slot(虚拟机分配内存所使用的最小单位), 对于byte, char, float, int, short, boolean和returnAddress等长度不超过32位数据统一占用一个Slot, 而像double, long这两种64位的数据类型则需要2个Slot进行存放. 方法参数(包括实例方法中的隐藏参数`this`),显式异常处理的参数(ExceptionHandler Parameter, catch中定义的参数), 方法体中定义的局部变量都需要局部变量表来存放.并不是所有的局部变量所占Slot之和作为max_locals, Slot可以重用, 当某个局部变量的超出了其作用范围, 该Slot会被回收, 最终由javac智能计算所有的空间占比作为max_locals的值.
+  code_length + code: 存储Java源程序编译后生成的字节码指令. code_length:代表字节码的长度, code是用于存储字节码指令的一系列字流. 既然叫字节码, 每个指令就是一个u1类型的单字节, 当虚拟机读取到这个字节码时, 就可以对应找出这个字节码代表的是什么指令, 并且可以知道这条指令后面是否需要跟随参数, 以及参数该怎么处理. 一个u1数据类型取值范围为: 0x00 - 0xFF, 对应0 - 255. 最大只有256条指令. 目前Java虚拟机已经定义了约200条指令:

> code_length虽然为u4类型的长度, 但是实际上只允许使用u2的长度(即2^32 -1, 65535)条指令码, 如果超过了这个限制, javac会拒绝编译.

![show](https://image.cjyong.com/blog/jvm/10.png)

![show](https://image.cjyong.com/blog/jvm/11.png)

![show](https://image.cjyong.com/blog/jvm/12.png)

![show](https://image.cjyong.com/blog/jvm/13.png)

![show](https://image.cjyong.com/blog/jvm/14.png)

![show](https://image.cjyong.com/blog/jvm/15.png)

这里我们查看我们的代码

![show](https://image.cjyong.com/blog/jvm/9.png)

+ **00 01**: atrribute_count, 1, 该方法有一个额外的属性, 以下为属性的详细内容.
+ **00 0A**: attribute_name_index, 指向第10个常量, 查询得: Code. 说明, 里面存储的是编译的二进制代码.
+ **00 00 00 1D**: attribute_length, 属性长度, 换算十进制: 29个字节.
+ **00 01 **: max_stack, 操作数栈深最大为1
+ **00 01**: max_locals, 局部变量, 占据最大内存为1slot.
+ **00 00 00 05**: code_length, 代码长度5字节.
+ **2A B7 00 01 B1**: code, 编译生成的二进制代码
  + **2A**, 查表得: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B7**, 查表得: invokespecial, 调用超类的构造方法, 实例初始化方法. 该方法需要传递一个u2类型的参数说明调用的对象.
  + *00 01**, 上一个方法的参数, 指向第1个常量, 为: java/lang/Object."<init>":()V
  + **B1**, 查表得: 从当前方法返回void.
+ **00 00**: exception_table_length, 长度为0, 说明没有异常处理表的存在.
+ **00 01**: attributes_count, 为1, 说明还有一项额外的信息.
+ **00 0B**: attribute_name_index, 指向第11个常量, 查询得: LineNumberTable, 说明里面存储的是LineNumberTable类型的额外信息.

> LineNumberTable属性用于描述Java源码行号与字节码行号之间的对应关系(一般是指偏移量). 非必须, 可以使用-g:none或者-g:lines来取消生成这项信息. 取消之后, 抛出异常时将不显示行号和无法根据源码行来设置断点. 组成: u2的attribute_name_index + u4的attribute_length + u2的line_number_table_length + length个line_number_info(u2的start_pc和line_number).
> + **00 00 00 06**: attribute_length, 6个字节.
> + **00 01**: line_number_table_length, 1个line_number_info.
> + **00 00**: line_number_info.start_pc, 字节码行号为0.
> + **00 02**: line_number_info.line_number, 源码行号为2.

我们可以清晰地看出这是一个构造函数. 接下来我们再来解析第二个函数:

```java
public int inc() {
    return m + 1;
}
```

![show](https://image.cjyong.com/blog/jvm/16.png)

+ **00 01**: access_flas, 查询的为, public类型的方法.
+ **00 0C**: name_index, 方法名的简单描述索引, 指向第12个常量, 查询得: inc.
+ **00 0D**: descriptor_index, 函数的描述索引, 指向第13个常量, 查询得: ()I, 可知参数为0个, 返回值为int.
+ **00 01**: attributes_count, 1, 存在一个额外的属性.
+ **00 0A**: attribute_name_index, 指向第10个常量, 查询得: Code. 说明, 里面存储的是编译的二进制代码.
+ **00 00 00 1F**: attribute_length, 属性长度, 换算十进制: 31个字节.
+ **00 02 **: max_stack, 操作数栈深最大为2
+ **00 01**: max_locals, 局部变量, 占据最大内存为1slot.
+ **00 00 00 07**: code_length, 代码长度7字节.
+ **2A B4 00 02 04 60 AC**: Code, 二进制字节代码
  + **2A**: 查表得,aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: 查表得, getfield, 获取指定类的实例域, 并将其值压入栈顶, 00 02: 指向一个FiledRef, 指向第二个常量, 查询的为:cjyong/classz/TestClass.m:I. 说明该变量为m, 类型为int.
  + **04**: 查表得, iconst_1, 将int型1推送到栈顶.
  + **60**: iadd, 将栈顶两个int型数值相加并将结果压入栈顶.
  + **AC**: ireturn, 从当前方法, 返回int.
+ 接下来的额LineNumberTable省略.

我们再来解析我们的第三个函数

```java
synchronized public int setM(int m) {
    try {
        this.m = 12 / m;
        return this.m;
    } catch (Exception e) {
        this.m = 1;
        return this.m;
    } finally {
        this.m = 2;
        return this.m;
    }
}
```

![show](https://image.cjyong.com/blog/jvm/17.png)

+ **00 21**: access_flas, 查询的为, public和synchronized类型的方法.
+ **00 0E**: name_index, 方法名的简单描述索引, 指向第14个常量, 查询得: setM.
+ **00 0F**: descriptor_index, 函数的描述索引, 指向第15个常量, 查询得: (I)I, 可知参数为1个int类型, 返回值为int.
+ **00 01**: attributes_count, 1, 存在一个额外的属性.
+ **00 0A**: attribute_name_index, 指向第10个常量, 查询得: Code. 说明, 里面存储的是编译的二进制代码.
+ **00 00 00 A8**: attribute_length, 属性长度, 换算十进制: 168个字节.
+ **00 03 **: max_stack, 操作数栈深最大为3
+ **00 05**: max_locals, 局部变量, 占据最大内存为5slot.
+ **00 00 00 38**: code_length, 代码长度56字节.
+ **2A 10 0C 1B 6C B5 00 02 2A B4 00 02 3D 2A 05 B5 00 02 2A B4 00 02 AC 4D 2A 04 B5 00 02 2A B4 00 02 3E 2A 05 B5 00 02 2A B4 00 02 AC 3A 04 2A 05 B5 00 02 2A B4 00 02 AC**: Code
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **10 0C**: bipush, 将单字节的常量值推送到栈顶, 0C代表12, 常量值. 即将12推送到栈顶.
  + **1B**: iload_1, 将第二个int型本地变量推送到栈顶.
  + **6C**: idiv, 将栈顶两个数相除并将结果压入栈顶.
  + **B5 00 02**: putfield,为指定类的实例域赋值(默认栈顶的值12/m), 00 02指向第二个常量为: cjyong/classz/TestClass.m:I. 说明该变量为m, 类型为int.
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: getfield, 获取指定类的实例域, 并将其值压入栈顶,即获取m的值并压入栈顶.
  + **3D**: istore_2, 将栈顶int型数值存入第三个本地变量.
  + **2A**: aload_0, 将第一个引用变量推送到栈顶
  + **05**: iconst_2, 将int型2推送至栈顶.
  + **B5 00 02**: putfield, 为指定类的实例域赋值(默认栈顶的值2), 00 02指向第二个常量为: cjyong/classz/TestClass.m:I. 说明该变量为m, 类型为int.
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: getfield, 获取指定类的实例域, 并将其值压入栈顶,即获取m的值并压入栈顶.
  + **AC**: ireturn, 返回当前栈顶的int值.
  + **4D**: astore_2, 将栈顶引用的数字类型存入第3个本地变量, 即: Exception e
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.(0位对象本身指针, 预留的参数)
  + **04**: iconst_1, 将int型1压入栈顶.
  + **B5 00 02**: putfield, 为指定类的实例域赋值(m=1).
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: getfield, 获取指定类的实例域, 并将其值压入栈顶,即获取m的值并压入栈顶.
  + **3E**: istore_3, 将栈顶int型数据存入第4个本地变量.
  + **2A**: aload_0, 将第一个引用变量推送到栈顶
  + **05**: iconst_2, 将int型2推送至栈顶.
  + **B5 00 02**: putfield, 为指定类的实例域赋值(默认栈顶的值2), 00 02指向第二个常量为: cjyong/classz/TestClass.m:I. 说明该变量为m, 类型为int.
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: getfield, 获取指定类的实例域, 并将其值压入栈顶,即获取m的值并压入栈顶.
  + **AC**: ireturn, 返回当前栈顶的int值2.
  + **3A 04**: astore 4, 将栈顶引用型数据存入指定的而本地变量.
  + **2A**: aload_0, 将第一个引用变量推送到栈顶
  + **05**: iconst_2, 将int型2推送至栈顶.
  + **B5 00 02**: putfield, 为指定类的实例域赋值(默认栈顶的值2), 00 02指向第二个常量为: cjyong/classz/TestClass.m:I. 说明该变量为m, 类型为int.
  + **2A**: aload_0, 将第一个引用变量(因为标记时a, a是引用对象)推送到栈顶.
  + **B4 00 02**: getfield, 获取指定类的实例域, 并将其值压入栈顶,即获取m的值并压入栈顶.
  + **AC**: ireturn, 返回当前栈顶的int值2.
+ **00 04**: exception_table_length, 长度为4, 说明有4个异常处理表的存在.

> u2的attribute_name_index + u4的attribute_length + u2的number_of_exceptions + nums个u2的exception_index_table.
> + **00 00 00 0D 00 17 00 03**: 第一个异常表, 从0字节(00 00)到13字节(00 0D)出现Exception类型异常(00 03所指向异常)就到23字节进行处理.(00 17)
> + **00 00 00 0D 00 2C 00 00**: 第二个异常表, 从0字节到13字节, 出现除了上一个异常以外的任意异常(00 00), 到44字节进行处理.
> + 接下两个省略. 00 17 00 22 00 2C 00 00/00 2C 00 2E 00 2C 00 00
> + **00 02**: attributes_count, 为2, 说明还有两项额外的信息.
> + 省略LineNumberTable的额外信息
> + **00 10**: 指向第16项常量, 为StackMapTable, 说明为StackMapTable类型额外信息.
> + **00 00 00 0A **: 长度为10字节.
> + **00 02**: number_of_entries, 2个
> + **57 07 00 11 **: stack_map_frame1
> + **54 07 00 12**: stack_map_frame2

终于到了最后一行:
** 00 01 00 13 00 00 00 02 00 14**
+ **00 01**: 说明class文件由一个额外的信息.
+ **00 13**: 指向第19项常量, SourceFile, 说明这里存储着的是文件信息.
+ **00 00 00 02**: 长度为2字节.
+ **00 14**: 指向第20个常量, TestClass.java.

到此所有的Class字节码解析完毕, 其实, 我们可以使用Javap -verbose xxx, 进行查看. Javap是JDK自带的用来解析Class文件的工具.

![show](https://image.cjyong.com/blog/jvm/18.png)

![show](https://image.cjyong.com/blog/jvm/19.png)

![show](https://image.cjyong.com/blog/jvm/20.png)

![show](https://image.cjyong.com/blog/jvm/21.png)

![show](https://image.cjyong.com/blog/jvm/22.png)

> 注意, 这里可能有人发现, 函数的参数args_size是多一个的, 那是如果这时实例方法的话, 为了保证对内部类的访问,这里把对象的实例引用默认传递过去了, 我们调用的aload_0, 其实加载的就是对象的默认引用对象. 当然如果是静态的方法, 就不会传递了.

