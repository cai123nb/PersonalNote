# JavaScript 基础

JavaScript 是一个非常简单的语言, 也是一门非常复杂的语言. 简单在于, 使用它是非常简单的, 只需片刻. 复杂在于, 真正掌握它则需要数年时间.

## JavaScript 简介

简要介绍 JavaScript 的历史, 与 ECMAScript 关系和不同版本信息.

### JavaScript 简史

就职与 Netscape 公司的**布兰登-艾奇**开发了一种名为**LiveScript**的脚本语言, 应用在**Netscape Navigator 2**中. 在**Netscape Navigator 2**发布前夕, 为了搭上媒体热炒**Java**的顺风车, 临时将**LiveScript**改名为**JavaScript**.

**JavaScript1.0**获得巨大的成功, **Netscape**随后在**Netscape Navigator 3**中发布了**JavaScript1.1**版本. 与此同时, 微软决定向自家的产品**Internet Explorer**浏览器投入更多资源来和**Navigator**进行竞争, 在**Internet Explorer 3**中加入了名为**JScript**的**JavaScript**的实现. 导致**JavaScript**的实现存在两个不同的版本: **Navigator**中的**JavaScript**和**Internet Explorer**中的**JScript**. 人们对**JavaScript**的规范和标准的需求越来越急切.

1997 年, 以**JavaScript1.1**为蓝本的建议被提交给欧洲计算机制造商协会(ECMA), 该协会指定 39 号技术委员会基于该蓝本建立一份**通用, 跨平台, 供应商中立的脚本语言的语法和语义标准**. 技术委员会经过数月的努力, 完成了**ECMA-262**(脚本语言)标准. 并于第二年, 被**ISO/IEC**采用. 自此以后, 浏览器厂商开始致力于将**ECMAScript**作为各自实现**JavaScript**的基础.

### JavaScript 实现

JavaScript 含义比**ECMA-262**中规定的要多得多, 一个完整 JavaScript 实现应该包含以下三个不同部分:

- 核心(ECMAScript): 对实现**ECMA-262**标准的描述, 包括语法,类型,语句,关键字,保留字,操作符,对象.(我们常说的 Web 浏览器, 只是 EMAScript 实现的**宿主环境之一**, 在宿主环境中不仅提供了基本的 ECMAScript 的实现, 同时会提供该语言的扩展, 以便语言和环境之间交互, 如 DOM. 另外的宿主环境包括: **Node**, **Adobe Flash**)
- 文档对象模型(DOM): 针对 XML 但是经过拓展用于 HTML 的应用程序接口(API), 统一不同浏览器对 HTML 解析和处理规范. 其中 DOM 分成 3 级层次结构:
  - 1 级: 可以映射文档结构成**DOM Tree**, 支持 XML 命名空间.
  - 2 级: 引入了 DOM 视图, DOM 事件, DOM 样式, DOM 遍历和范围四大新模块.
  - 3 级: 引入了加载和保存文档的方法, 新增了验证文档的方法, 对 DOM 核心进行拓展, 开始支持 XML1.0 规范.
- 浏览器对象模型(BOM): 支持可以访问和操作浏览器窗口的浏览器对象模型.

### HTML 中使用 JavaScript

通`<script></script>`元素进行包含处理, 其中可以使用的参数有:

- **src**: 可选, 外部可执行代码的文件路径.
- **async**: 可选, 表示立即下载脚本, 但是不要阻碍其他操作(如下载其他资源或者等待加载其他脚本). 只对外部脚本文件有效(src 引入的). 使用时需要小心, 这个异步是没有顺序保证的.
- **defer**: 可选, 表示脚本可以延迟到文档被完全解析和显示之后再执行, 只对外部脚本有效.

注意如果直接在`<script>`中嵌入代码时, 不要在代码的任何地方出现`<\script>`字符串不然会出错. 如:

```javascript
<script>
  function sayScript(){
    alert("</script>");
  }
</script>
```

以上代码执行时会导致页面报错.

按照惯例, JavaScript 文件带有`.js`拓展名, 但是这个拓展名是可选的, 浏览器不会强制校验 JavaScript 文件的拓展名. 另外`src`中可以引入其他域的 JS 文件, 如: `src="http://www.somewhere.com/afile.js"`. 这个需要小心, 除了这个域是你自己的, 或者这个域值得信任, 一般不推荐这么使用.

一般我们都将`<script>`放在`<body>`的底部, 这样可以防止页面请求脚本, 而导致页面空白形成卡顿.

在 IE5.5 中映入了文档模式的概念, 可以通过使用文档类型来进行切换. 最初的文档模式为: **混杂模式(quirks mode)**, **标准模式(standards mode)**. 前者会让行为与 IE5 相同, 后者则更加接近标准行为. 一般推荐使用标准模式, 这样可以保证不同浏览器的表现行为尽量一致. H5 标准模式开启方式: `<!DOCTYPE html>`.

为了解决部分原始浏览器不支持脚本语言或者脚本语言被禁用的情况, 可以使用`<noscript>`标签进行提示:

```javascript
<noscript>
  <p>本页面需要浏览器支持(启用)JavaScript。
</noscript>
```

## ECMAScript3 基本概念

### 语法

- 区分大小写
- 标识符的第一个字符串必须是字母,下划线或者\$.
- 单行注释: `//`, 多行注释: `/* */`.
- 语句以分号结尾(也可以省略, 但不推荐).
- 关键字: `break, do, instanceof, typeof, case, else, new, var, catch, finally, return, void, continue, for, switch, while, debugger, function, this, with, default, if, throw, delete, in, try, implements, package, public, interface, private, static, let, protected, yield`.

### 数据类型

数据类型: 5 + 1, 5 基本类型: `Undefined, Null, Boolean, Number, String`, 1 复杂类型: `Object`. 通过`typeof`可以获取对象的类型, 返回值如下:

- undefined: 对象未定义.
- boolean: Boolean
- string: 字符串
- number: 数值
- object: 值为对象或者 null.
- function: 函数

5 大基本类型:

- Undefined: 只含有 undefined 一个值, 声明变量未初始化时, 默认的值.
- Null: 只含有特殊值 null, 逻辑上表示为空指针. 使用`typeof`时返回`object`, `undefined`派生自`null`两者相等, 但不全等.
- Boolean: 含有两个字面值: `true, false`. `true`: `非空字符串, 非零数字, 任何对象, n/a`. `false`: `空字符串, 0和NaN, null, undefined`.
- Number: 使用`IEEE754`格式使用 64 位表示整数和浮点数. 整数: 默认 10 进制整数, 0 开头默认八进制(如果内部超过了 7 则认为是十进制整数), 0x 开头默认十六进制. 浮点数计算会存在误差, 请不用测试特定的值(如 0.1 + 0.2 永远不会等于 0.3). 范围为: `Number.MIN_VALUE - Number.MAX_VALUE`. `NaN`表示特殊数值: 本来应该返回的数值操作, 却没有返回数值(通过这种方式, 防止报错). 如`Number('Hello world!')`. 首先, 任何涉及`NaN`的操作都返回`NaN`. 其次, `NaN与任何值都不相等, 包括本身.`
- String: 零或多个 16 位 Unicode 字符组成的字符串序列. 单引号和双引号没有任何区别.

复杂数据类型:

Object: 一组数据和功能的集合. 通过`new + 对象名称`来创建. 基本方法有:

- constructor: 构造函数, 默认新建时调用.
- hasOwnProperty(propertyName): 用于检查当前对象是否包含某项属性(不是在原型中继承的).
- isPrototypeOf(object): 是否是传入对象的原型(在原型链中是不是, 传入对象的爸爸)).
- propertyIsEnumerable(propertyName): 某项属性是否可以枚举(使用 for-in).
- toLocaleString: 返回本地自定义字符串.
- toString: 字符串.
- valueOf: 返回对象的字符串, 数值或布尔值表示.

需要注意的一点是 JS 中的函数, JS 函数对参数没有任何限制, 是使用一个数组进行存储(`arguments`), 可以通过这个数组访问内部所有的数据, 另外函数没有`重载`. 后定义的会覆盖前面的函数.

### 变量和作用域

基本类型(五大基础数据)都是简单的数据段(按值传递), 引用类型(对象)则是多个值构成的对象(按引用传递, 对象本身存储在堆内存中). 可以通过`typeof`来检测对应的数据类型, `instanceof`来校验对应的细化的对象类型.

JS 中的作用域是套娃类型, 主要分为: 全局作用域和函数作用域. 内部作用域可以访问外部作用域对象, 外部作用域无法访问内部作用域. 具体的实现: 全局具备一个全局执行环境(如 window). 每个函数具备自己的执行环境, 每次调用一个函数, 函数的执行环境就会推入一个环境栈中(有点类似 Java 的栈帧), 执行完毕则会退出当前环境, 返回上一层环境.

#### 垃圾回收

JavaScript 内部实现自动内存管理, 截止到 2008 年, 使用的垃圾回收算法为**标记清除算法**: 当一个变量离开执行环境时, 会被标记为待删除变量, 后期进行统一删除. 为了性能问题, 每次不使用变量时, 将变量赋值为 null, 可以清空引用.

## 引用类型

在 JavaScript 中引用类型是一种数据结构, 将数据和功能组织结合在一起, 常被称为**类**. **ECMAScript**提供了很多原生引用类型, 以便开发人员用以实现常见的计算任务.

### Object 类型

Object 是一个基础类型, 其他所有的类型都从 Object 继承基本的行为. 创建方式有两种:

```javascript
// type1
var person = new Object();
person.name = 'Nicholas';
person.age = 29;

//type2
var person = {
  name: 'Nicholas',
  age: 29,
};
```

当我们需要访问内部变量时, 可以通过`.PROPERTY_NAME`或者`[PROPERTY_NAME]`进行访问.

### Array 类型

Array 类型是一组值得有序列表, 同时提供操作和转换这些值得功能.

创建方式:

```javascript
// type1
var colors = new Array(); // empty array

var colors = new Array('red', 'blue', 'green'); // array with contains red, blue, green

var colors = new Array(3); // array with length is 3

//type2
var colors = ['red', 'blue', 'green'];
```

检测方式, `xx instanceof Array`或者`Array.isArray(xx)`, 推荐使用后者, 可以兼容不同的环境.

内置的一些协助方法:

- `toString(),valueOf(), toLocaleString()`: 返回字符串表示的数组, 默认用逗号分隔元素, 后者返回地域性描述.
- `join(x)`: 传递分隔符`x`, 返回用分隔符分隔的字符串表示.
- `push`: 向数组尾部插入元素.
- `pop`: 从数组尾部取出一枚元素.
- `shift`: 从数组头部取出第一项.
- `unshift`: 在数组头部插入元素.
- `reverse`: 将数组内元素进行排序反转.
- `sort`: 对数组内部元素进行排序, 默认最小的放在前面. 可以传递比较方法, 进行排序.
- `concat`: 拷贝当前数组, 在新数组的尾部插入连接新的数组. 是一个函数式方法, 不会影响之前的数组.
- `slice`: 基于当前数组一个或者多个创建新数组, 不会影响原来的数组(切分)).
- `splice`:
  - 删除: 传递两个参数(index 值), 删除 index 值之间的元素对象.
  - 插入: 传递 3 个参数, 分别为: 插入位置, 需要删除的项数, 需要插入的项. 第二个参数设置为 0 即为插入.
  - 替换: 同上, 将第二个参数设置为对应的删除项, 即为替换.
- `indexOf/lastIndexOf`: 从开头向后找/从结尾向前找. 找到返回对应的 index 值,没有找到时, 返回-1.
- `every`: 传递一个判断函数(返回 boolean), 只有所有的元素都满足这个条件时, 返回 true, 否则 false.
- `filter`: 传递判断函数, 将符合条件的元素留下, 其余删除.
- `forEach`: 传递消费函数, 对数组内所有的元素依次传递消费.
- `map`: 传递转换函数, 对每一个元素进行转换, 构成新数组.
- `some`: 传递判断函数, 只要一个元素满足条件直接返回 true, 否则为 false.
- `reduce/reduceRight`: 迭代数组内所有的项, 最终返回一个值. 前者从前往后, 后者为从后往前. 传递一个四参函数: 前一个值, 当前值, 当前索引和数组对象. 返回对象为下一个元素调用函数的前一个值.

### Date 类型

Date 类型提供有关日期和时间的信息, 包括当前日期和时间以及相关的计算功能.

创建方式

```javascript
var now = new Date();
var someDate = new Date(Date.UTC(毫秒数)));
var someDate = new Date(Date.parse(xxxFormat));
// 支持格式有:
// 月/日/年
// 英文月 日,年
// 英文星期几 英文月名 日 年 时:分:秒 时区
// YYYY-MM-DDTHH:mm:ss.sssZ
```

具体方法和参数查阅[相关文档](https://www.w3school.com.cn/jsref/jsref_obj_date.asp).

### RegExp 类型

RegExp 类型是 ECMAScript 支持正则表达式的一个接口, 提供了最基本的和一些高级的正则表达式功能.

创建方式:

`var expression = / pattern / flags ;`

其中`pattern`是任何简单或者复制的正则表达式, `flags`为标识, 支持三种:

- `g`: 表示为全局模式, 并非搜索到第一个就停止.
- `i`: 标识不区分大小写.
- `m`: 标识多行模式, 到达一行文本结尾时, 继续查找下一行.

每个 RegExp 的实例都包含以下属性:

- global: 是否设置 g 标签.
- ignoreCase: 是否设置了 i 标签.
- multiline: 是否设置了 m 标签.
- lastIndex: 开始搜索下一个匹配字符串的位置, 默认从 0 开始.
- source: pattern 的字符串表示, 字面量形式.

实例方法:

- `exec()`: 专门捕获组(用()括起来匹配), 传递需要校验的字符串, 返回一个数组. 数组内第一个元素为符合整体正则表达式的结果, 第二个元素为符合第一个捕获组的结果, 第三个元素为符合第二个元素捕获组的结果, 以此类推. 如`/mom( and dad( and baby)?)?/gi`, 其中整体为一个正则表达式, 结果放在数组的第一个位置, `( and dad( and baby)?)`为第一个捕获组, 存放在数组的第二个位置, `( and baby)`为第二个捕获组, 存放在数组的第三个位置.一般情况简单使用第一个即可.
- `test`: 判断是否匹配正则, 传递需要校验的文本, 返回布尔值.

RegExp 构造函数中存储着一些属性(可以认为是静态的), 是基于最近执行的一次正则操作而变化的.

- input: \$\_(短属性名), 最近一次匹配的字符串.
- lastMatch: \$&, 最近一次的匹配项.
- lastParen: \$+, 最近一次匹配的捕获组.
- leftContext: \$`, input 字符串中 lastMatch 之前的文本.
- multiline: \$\*, 是否是多行模式.
- rightContext: &', input 字符串中 lastMatch 之前的文本.

如:

```javascript
var text = 'this has been a short summer';
var pattern = /(.)hort/g;
if (pattern.test(text)) {
  alert(RegExp.input); // this has been a short summer
  alert(RegExp.leftContext); // this has been a
  alert(RegExp.rightContext); // summer
  alert(RegExp.lastMatch); // short
  alert(RegExp.lastParen); // s
  alert(RegExp.multiline); // false
}
```

### Function 类型

函数为 Function 类型的实例, 函数也是一个对象. 函数名为指向函数对象的指针, 不与函数绑定. 创建方式:

```javascript
function sum(num1, num2) {
  return num1 + num2;
}

var sum = function(num1, num2) {
  return num1 + num2;
};
```

其中最重要的一点就是**没有重载**, 后者会覆盖前者:

```javascript
function addSomeNumber(num) {
  return num + 100;
}
function addSomeNumber(num) {
  return num + 200;
}
var result = addSomeNumber(100); //300
```

第二, 函数声明和函数表达式有区别:

```javascript
// good
alert(sum(10, 10));
function sum(num1, num2) {
  return num1 + num2;
}

// error
alert(sum(10, 10));
var sum = function(num1, num2) {
  return num1 + num2;
};
```

在 JavaScript 执行代码的时候, 解析器会通过一个`function declaration hoisting`的操作, 读取所有的函数声明, 并添加到执行环境中. 所以, 即使函数声明放在后面也是可以正常使用的. 但是如果是函数表达式, 则不会被处理.

JavaScript 中函数即为对象, 也是可以作为参数传递和返回的.

```javascript
function callSomeFunction(someFunction, someArgument) {
  return someFunction(someArgument);
}

function add10(num) {
  return num + 10;
}
var result1 = callSomeFunction(add10, 10);
alert(result1); //20
function getGreeting(name) {
  return 'Hello, ' + name;
}
var result2 = callSomeFunction(getGreeting, 'Nicholas');
alert(result2); //"Hello, Nicholas"
```

在函数内部, 存在两个特殊对象: `arguments`和`this`. 其中`arguments`除了存储参数数组之外, 还存储了一个特殊的属性: `callee`, 指向拥有`arguments`对象的函数. 这个某些情况下非常有用:

```javascript
function factorial(num) {
  if (num <= 1) {
    return 1;
  } else {
    return num * factorial(num - 1);
  }
}
```

这是一个经典的阶乘函数, 但是有一个缺点, 内部的`factorial`和名字紧紧的耦合在一起, 如果外部修改了(覆盖)了该函数名, 那么这个阶乘函数就会出现问题. 这时候就可以通过`callee`来解决这个问题:

```javascript
function factorial(num) {
  if (num <= 1) {
    return 1;
  } else {
    return num * arguments.callee(num - 1);
  }
}
```

另外`callee`中还存储着一个属性为`caller`, 保存当前函数的函数调用. 如果是全局调用那么则返回`null`(注意这里是值往上一层)).

其次, `this`则是指向当前函数的执行环境.

函数的属性和方法, 每个函数实例都包含两个属性: `length`和`prototype`. 其中`length`表示函数希望接收的参数个数. `prototype`则存储所有继承的属性和方法.

另外函数内部单独实现了两个方法: `apply(),call()`, 用于在特定的作用域中调用函数, 实际上相当于设置`this`对象的值.

```javascript
function sum(num1, num2) {
  return num1 + num2;
}
function callSum1(num1, num2) {
  return sum.apply(this, arguments);
}
function callSum2(num1, num2) {
  return sum.apply(this, [num1, num2]);
}
alert(callSum1(10, 10)); //20
alert(callSum2(10, 10)); //20
```

另外一个设置的方法是`bind`方法, 通过传递一个对象给`bind`绑定到方法的`this`属性.

### 基本包装类型

为了便于操作基本类型值, ECMAScript 还提供了 3 个特殊的引用类型: `Boolean, Number和String`. 这些引用类型和其它引用类型相似, 但是也具备了一些特殊的操作. 实际上, 每当读取一个基本类型数据时, 后台就会自动创建一个对应的基本包装类型的对象, 来方便我们调用一些方法来操纵这些数据.

```javascript
// use sample
var s1 = 'some text';
var s2 = s1.substring(2);

// actually
var s1 = 'some text';
var s1Temp = new String('some text');
var s2 = s1Temp.substring(2);
s1Temp = null;

// proof
var s1 = 'some text';
s1.color = 'red'; // here ceate String s1Temp1
alert(s1.color); //undefined, here create String s1temp2, s1Temp1 != s1Temp2, so s1Temp2 has no color
```

平时应该避免显示调用`new String()/Boolean()/Number()`, 这会导致误解, 我们调用`typeof`时会返回`object`. 需要注意的是, 如果是`new +类型`是创建对应的类型对象, 直接调用`类型函数`, 如`var number = Number('23')`, 则是转型函数, 返回对应的原始类型数据.

#### Boolean 类型

Boolean 类型是布尔值对应的引用类型, 可以通过`new Boolean(true)`格式来创建对应的对象, 一般用途较小, 还容易造成误解, 因为对象的实例都是为`true`, 并不会受内部的值影响:

```javascript
var falseObject = new Boolean(false);
var result = falseObject && true;
alert(result); //true
var falseValue = false;
result = falseValue && true;
alert(result); //false
```

#### Number 类型

Number 是与数字值对应的引用类型. 创建方式: `new Number()`. 重写了`valueOf(), toLocaleString()和toString()`方法. 其中`toString()`可以传递对应的进制数, 进行翻译. 内部常用的方法:

```javascript
var num = 10;
alert(num.toString()); //"10"
alert(num.toString(2)); //"1010"
alert(num.toString(8)); //"12"
alert(num.toString(10)); //"10"
alert(num.toString(16)); //"a"

alert(num.toFixed(2)); //"10.00"

var num = 10.005;
alert(num.toFixed(2)); //"10.01"

var num = 10;
alert(num.toExponential(1)); //"1.0e+1"

var num = 99;
alert(num.toPrecision(1)); //"1e+2"
alert(num.toPrecision(2)); //"99"
alert(num.toPrecision(3)); //"99.0"

var numberObject = new Number(10);
var numberValue = 10;
alert(typeof numberObject); //"object"
alert(typeof numberValue); //"number"
alert(numberObject instanceof Number); //true
alert(numberValue instanceof Number); //false
```

#### String 类型

String 类型是字符串对象的包装类型, 通过`new String()`格式来创建. 内部实现的方法和属性:

```javascript
var stringValue = 'hello world';
alert(stringValue.length); //"11"
alert(stringValue.charAt(1)); //"e"
alert(stringValue.charCodeAt(1)); //输出"101"
var stringValue = 'hello ';
var result = stringValue.concat('world'); // 不修改stringValue的值
alert(result); //"hello world"
alert(stringValue); //"hello"

alert(stringValue.slice(3)); //"lo world"
alert(stringValue.substring(3)); //"lo world"
alert(stringValue.substr(3)); //"lo world"
alert(stringValue.slice(3, 7)); //"lo w"
alert(stringValue.substring(3, 7)); //"lo w"
alert(stringValue.substr(3, 7)); //"lo worl"

alert(stringValue.slice(-3)); //"rld"
alert(stringValue.substring(-3)); //"hello world"
alert(stringValue.substr(-3)); //"rld"
alert(stringValue.slice(3, -4)); //"lo w"
alert(stringValue.substring(3, -4)); //"hel"
alert(stringValue.substr(3, -4)); //""(空字符串)

alert(stringValue.indexOf('o')); //4
alert(stringValue.lastIndexOf('o')); //7
alert(stringValue.indexOf('o', 6)); //7
alert(stringValue.lastIndexOf('o', 6)); //4
var stringValue = '   hello world   ';
var trimmedStringValue = stringValue.trim();
alert(stringValue); //"   hello world   "
alert(trimmedStringValue); //"hello world"
var stringValue = 'hello world';
alert(stringValue.toLocaleUpperCase()); //"HELLO WORLD"
alert(stringValue.toUpperCase()); //"HELLO WORLD"
alert(stringValue.toLocaleLowerCase()); //"hello world"
var text = 'cat, bat, sat, fat';
var pattern = /.at/;
//与 pattern.exec(text)相同
var matches = text.match(pattern);
alert(matches.index); //0
alert(matches[0]); //"cat"
alert(pattern.lastIndex); //0

var text = 'cat, bat, sat, fat';
var pos = text.search(/at/);
alert(pos); //1

var text = 'cat, bat, sat, fat';
var result = text.replace('at', 'ond');
alert(result); //"cond, bat, sat, fat"
result = text.replace(/at/g, 'ond');
alert(result); //"cond, bond, sond, fond"

var text = 'cat, bat, sat, fat';
result = text.replace(/(.at)/g, 'word ($1)');
alert(result); //word (cat), word (bat), word (sat), word (fat)
// $&, $', $`与RegExp定义相同
// $n和$nn, 匹配第n(nn)个捕获组的字符串.
alert(String.fromCharCode(104, 101, 108, 108, 111)); //"hello"
```

### 单体内置对象

ECMA-262 定义了`Global和Math`两个单体对象.

#### Global 对象

`Global`定义了很多全局的方法, 如`isNaN, isFinite, parseInt, parseFloat`. 还有一些特定的函数:

```javascript
var uri = 'http://www.wrox.com/illegal value.htm#start';
//"http://www.wrox.com/illegal%20value.htm#start"
alert(encodeURI(uri));
//"http%3A%2F%2Fwww.wrox.com%2Fillegal%20value.htm%23start"
alert(encodeURIComponent(uri));

// 类似ECMAScript解析器, 执行javascript代码
eval("alert('hi')");
// equal
alert('hi');

var msg = 'hello world';
eval('alert(msg)'); //"hello world"

eval("function sayHi() { alert('hi'); }");
sayHi();
```

Global 的对象属性:

| 属性           | 说明             |
| :------------- | :--------------- |
| undefined      | 特殊值 undefined |
| NaN            | 特殊值 NaN       |
| Infinity       | 特殊值 Infinity  |
| Object         | 对应的构造函数   |
| Array          | 对应的构造函数   |
| Function       | 对应的构造函数   |
| Boolean        | 对应的构造函数   |
| String         | 对应的构造函数   |
| Number         | 对应的构造函数   |
| Date           | 对应的构造函数   |
| RegExp         | 对应的构造函数   |
| Error          | 对应的构造函数   |
| EvalError      | 对应的构造函数   |
| RangeError     | 对应的构造函数   |
| ReferenceError | 对应的构造函数   |
| SyntaxError    | 对应的构造函数   |
| TypeError      | 对应的构造函数   |
| URLError       | 对应的构造函数   |

在浏览器中, 全局对象就是`window`对象的一部分来实现的.

```java
var color = "red";
function sayColor(){
    alert(window.color);
}
window.sayColor();  //"red"

// 获取全局对象
var global = function(){
  return this;
}();
```

#### Math 对象

ECMAScript 还为保存数学公式和信息提供了一个公共的位置, `Math`对象. 内部的属性:

| 属性         | 说明                              |
| :----------- | :-------------------------------- |
| Math.E       | 自然对数的底数，即常量 e 的值     |
| Math.LN10    | 10 的自然对数                     |
| Math.LN2     | 2 的自然对数                      |
| Math.LOG2E   | 以 2 为底 e 的对数                |
| Math.LOG10E  | 以 10 为底 e 的对数               |
| Math.PI      | π 的值                            |
| Math.SQRT1_2 | 1/2 的平方根(即 2 的平方根的倒数) |
| Math.SQRT2   | 2 的平方根                        |

一些辅助方法:

```javascript
var max = Math.max(3, 54, 32, 16);
alert(max); //54
var min = Math.min(3, 54, 32, 16);
alert(min); //3
var values = [1, 2, 3, 4, 5, 6, 7, 8];
var max = Math.max.apply(Math, values); // 8

alert(Math.ceil(25.9)); //26
alert(Math.ceil(25.5)); //26
alert(Math.ceil(25.1)); //26
alert(Math.round(25.9)); //26
alert(Math.round(25.5)); //26
alert(Math.round(25.1)); //25
alert(Math.floor(25.9)); //25
alert(Math.floor(25.5)); //25
alert(Math.floor(25.1)); //25

// 1- 10
var num = Math.floor(Math.random() * 10 + 1);
```

其他数学方法:

| 属性                | 说明                    |
| :------------------ | :---------------------- |
| Math.abs(num)       | 返回 num 的绝对值       |
| Math.exp(num)       | 返回 Math.E 的 num 次幂 |
| Math.log(num)       | 返回 num 的自然对数     |
| Math.pow(num,power) | 返回 num 的 power 次幂  |
| Math.sqrt(num)      | 返回 num 的平方根       |
| Math.acos(x)        | 返回 x 的反余弦值       |
| Math.asin(x)        | 返回 x 的反正弦值       |
| Math.atan(x)        | 返回 x 的反正切值       |
| Math.atan2(y,x)     | 返回 y/x 的反正切值     |
| Math.cos(x)         | 返回 x 的余弦值         |
| Math.sin(x)         | 返回 x 的正弦值         |
| Math.tan(x)         | 返回 x 的正切值         |
