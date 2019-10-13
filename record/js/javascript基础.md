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
