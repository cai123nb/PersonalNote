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
- Number: 使用`IEEE754`格式使用64位表示整数和浮点数. 整数: 默认10进制整数, 0开头默认八进制(如果内部超过了7则认为是十进制整数), 0x开头默认十六进制. 浮点数计算会存在误差, 请不用测试特定的值(如0.1 + 0.2永远不会等于0.3). 范围为: `Number.MIN_VALUE - Number.MAX_VALUE`. `NaN`表示特殊数值: 本来应该返回的数值操作, 却没有返回数值(通过这种方式, 防止报错). 如`Number('Hello world!')`. 首先, 任何涉及`NaN`的操作都返回`NaN`. 其次, `NaN与任何值都不相等, 包括本身.`
- String: 零或多个16位Unicode字符组成的字符串序列. 单引号和双引号没有任何区别.

复杂数据类型:

Object: 一组数据和功能的集合. 通过`new + 对象名称`来创建. 基本方法有:

- constructor: 构造函数, 默认新建时调用.
- hasOwnProperty(propertyName): 用于检查当前对象是否包含某项属性(不是在原型中继承的).
- isPrototypeOf(object): 是否是传入对象的原型(在原型链中是不是, 传入对象的爸爸)).
- propertyIsEnumerable(propertyName): 某项属性是否可以枚举(使用for-in).
- toLocaleString: 返回本地自定义字符串.
- toString: 字符串.
- valueOf: 返回对象的字符串, 数值或布尔值表示.

需要注意的一点是JS中的函数, JS函数对参数没有任何限制, 是使用一个数组进行存储(`arguments`), 可以通过这个数组访问内部所有的数据, 另外函数没有`重载`. 后定义的会覆盖前面的函数.
