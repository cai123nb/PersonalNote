# 函数表达式

在JavaScript中定义函数的方式有两种: 函数声明和函数表达式. **函数声明**的语法是这样的:

```javascript
function functionName(arg0, arg1, arg2) {
  // 函数体
}
```

函数表达式有很多种, 常见的一种就是:

```javascript
var function = function(arg0, arg1, arg2);
```

两者存在一些差别. 其中第一点就是, **函数声明**存在**函数声明提升(function declaration hoisting)**, 意思在执行代码之前会首先读取函数声明. 这就意味着可以使用在前, 声明在后.

```javascript
sayHi();
function sayHi(){
    alert("Hi!");
}
```

而函数表达式则不允许, 函数表达式类似于将一个匿名函数赋值给一个变量, 就必须初始化该变量之后才能使用. 明白两者的区别, 可以看下面这个例子:

```javascript
//不要这样做!
if(condition){
  function sayHi(){
    alert("Hi!");
  }
} else {
  function sayHi(){
    alert("Yo!");
  }
}

//可以这样做 var sayHi;
if(condition){
  sayHi = function(){
    alert("Hi!");
  };
} else {
  sayHi = function(){
    alert("Yo!");
  };
}
```

其中第一个例子的错误在于, 在函数声明提升的过程中时, 该段判断逻辑属于无效逻辑, JavaScript引擎会尝试修正该错误, 但是不同浏览器做法不一致. 使用第二个例子则不会存在这个问题, 另外匿名函数还有很多其他用途, 如:

```javascript
function createComparisonFunction(propertyName) {
  return function(object1, object2){
      var value1 = object1[propertyName];
      var value2 = object2[propertyName];
      if (value1 < value2){
      return -1;
    } else if (value1 > value2){
      return 1;
    } else {
      return 0;
    }
  };
}
```

## 递归

递归函数就是在函数内通过名字调用自身的情况构成, 如下所示:

```javascript
function factorial(num){
  if (num <= 1) {
    return 1;
  } else {
    return num * factorial(num - 1);
  }
}
```

这是一个经典的阶乘函数, 表面上看起来没有问题. 但是下面代码可以让它出错:

```javascript
var anotherFactorial = factorial;
factorial = null; // 这里覆盖了factorial引用
alert(anotherFactorial(4)); //出错!
```

解决方案, 使用`arguments.callee`指针:

```javascript
function factorial(num){
  if (num <= 1){
    return 1;
  } else {
    return num * arguments.callee(num-1);
  }
}
```

因此在写递归函数时, 使用`arguments.callee`总比使用函数名更加保险. 但是在严格模式下, 不可以通过`arguments.callee`来访问函数, 但是可以使用命名函数表达式达成同样的结果:

```javascript
var factorial = (function f(num){
  if (num <= 1) {
    return 1;
  } else {
    return num * f(num - 1);
  }
})
```

以上代码创建了一个名为`f()`的命名函数表达式，然后将它赋值给变量`factorial`。即便把函数 赋值给了另一个变量函数的名字`f`仍然有效，所以递归调用照样能正确完成。这种方式在严格模式和 非严格模式下都行得通.

## 闭包

**闭包**是指有权访问另一个函数作用域中的变量的函数. 创建闭包的常见方式，就是在一个函数内部创建另一个函数. 如:

```javascript
function createComparisonFunction(propertyName) {
  return function(object1, object2){
      var value1 = object1[propertyName];
      var value2 = object2[propertyName];
      if (value1 < value2){
      return -1;
    } else if (value1 > value2){
      return 1;
    } else {
      return 0;
    }
  };
}
```

返回的函数就是内部创建的函数, 但是依然可以访问外部的变量`propertyName`, 即使被返回了. 这是因为内部函数的作用域包含了外部`createComparisonFunction`函数的作用域: 当某个函数被调用时，会创建一个执行环境(`execution context`)及相应的作用域链。 然后，使用 `arguments` 和其他命名参数的值来初始化函数的活动对象(`activation object`)。但在作用域链中，外部函数的活动对象始终处于第二位，外部函数的外部函数的活动对象处于第三位，......直至作为作用域链终点的全局执行环境。

这里看一个简单的例子:

```javascript
function compare(value1, value2){
  if (value1 < value2){
    return -1;
  } else if (value1 > value2){
    return 1;
  } else {
    return 0;
  }
}
var result = compare(5, 10);
```

以上代码先定义了`compare()`函数，然后又在全局作用域中调用了它。当调用`compare()`时，会创建一个包含`arguments`、`value1`和`value2` 的活动对象。全局执行环境的变量对象(包含 `result`和`compare`)在 `compare()`执行环境的作用域链中则处于第二位.

![relation_0](https://image.cjyong.com/js_relation_tree0.png)

后台的每个执行环境都有一个表示变量的对象——变量对象。全局环境的变量对象始终存在，而像 `compare()`函数这样的局部环境的变量对象，则只在函数执行的过程中存在。在创建 `compare()`函数时，会创建一个预先包含全局变量对象的作用域链，这个作用域链被保存在内部的[[Scope]]属性中。 当调用`compare()`函数时，会为函数创建一个执行环境，然后通过复制函数的[[Scope]]属性中的对象构建起执行环境的作用域链。此后，又有一个活动对象(在此作为变量对象使用)被创建并被推入执行环境作用域链的前端。对于这个例子中 compare()函数的执行环境而言，其作用域链中包含两个变 量对象:本地活动对象和全局变量对象。显然，作用域链本质上是一个指向变量对象的指针列表，它只引用但不实际包含变量对象。

无论什么时候在函数中访问一个变量时，就会从作用域链中搜索具有相应名字的变量。一般来讲， 当函数执行完毕后，局部活动对象就会被销毁，内存中仅保存全局作用域(全局执行环境的变量对象)。 但是，闭包的情况又有所不同。

![relation_1](https://image.cjyong.com/js_relation_area1.png)

在另一个函数内部定义的函数会将包含函数(即外部函数)的活动对象添加到它的作用域链中。因此，在 createComparisonFunction()函数内部定义的匿名函数的作用域链中，实际上将会包含外部函数 createComparisonFunction()的活动对象.

在匿名函数从 createComparisonFunction()中被返回后，它的作用域链被初始化为包含 createComparisonFunction()函数的活动对象和全局变量对象。这样，匿名函数就可以访问在 createComparisonFunction()中定义的所有变量。更为重要的是，createComparisonFunction() 函数在执行完毕后，其活动对象也不会被销毁，因为匿名函数的作用域链仍然在引用这个活动对象。换句话说，当 createComparisonFunction()函数返回后，其执行环境的作用域链会被销毁，但它的活动对象仍然会留在内存中;直到匿名函数被销毁后，createComparisonFunction()的活动对象才会被销毁，例如:

```javascript
//创建函数
var compareNames = createComparisonFunction("name");
//调用函数
var result = compareNames({ name: "Nicholas" }, { name: "Greg" });
//解除对匿名函数的引用(以便释放内存)
compareNames = null;
```

由于闭包会携带包含它的函数的作用域，因此会比其他函数占用更多的内存。过度使用闭包可能会导致内存占用过多，我们建议读者只在绝对必要时再考虑使用闭包。虽然像`V8`等优化后的`JavaScript`引擎会尝试回收被闭包占用的内存，但请大家还是要慎重使用闭包。

### 闭包与变量

作用域链的配置机制带来了一个副作用: 闭包只能取得包含函数中任何变量的最后一个值. 闭包所保存的是整个变量对象，而不是某个特殊的变量.

```javascript
function createFunctions(){
  var result = new Array();
  for (var i=0; i < 10; i++){
    result[i] = function(){
      return i;
    };
  }
  return result;
}
```

这个函数会返回一个函数数组。表面上看，似乎每个函数都应该返自己的索引值，即位置`0`的函数返回`0`，位置`1`的函数返回`1`，以此类推。但实际上，每个函数都返回`10`。因为每个函数的作用域链中 都保存着`createFunctions()`函数的活动对象，所以它们引用的都是同一个变量`i`。当`createFunctions()`函数返回后，变量`i`的值是`10`，此时每个函数都引用着保存变量`i`的同一个变量对象，所以在每个函数内部`i`的值都是`10`. 但是我们可以创建另一个匿名函数来强制让闭包符合预期:

```javascript
function createFunctions() {
  var result = new Array();
  for (var i = 0; i < 10; i++) {
    result[i] = function(num){
      return function() {
        return num;
      }
    }(i);
  }
  return result;
}
```

### 关于this对象

在闭包中使用 this 对象也可能会导致一些问题。我们知道，this 对象是在运行时基于函数的执行环境绑定的: 在全局函数中，this 等于 window，而当函数被作为某个对象的方法调用时，this 等于那个对象。不过，匿名函数的执行环境具有全局性，因此其 this 对象通常指向 `window`。但有时候由于编写闭包的方式不同，这一点可能不会那么明显。

```javascript
var name = "The Window";
var object = {
  name : "My Object",
  getNameFunc : function(){
    return function(){
      return this.name;
    };
  }
};
alert(object.getNameFunc()()); //"The Window"(在非严格模式下)
```

解决方案也很简单, 在外部函数的环境中存储`this`变量即可:

```javascript
var name = "The Window";
var object = {
  name : "My Object",
  getNameFunc : function(){
    var that = this;
    return function(){
      return that.name;
    };
  }
};
alert(object.getNameFunc()()); //"My Object"(在非严格模式下)
```

在一些特殊情况, `this`的值可能会被意外地改变.

```javascript
var name = "The Window";
var object = {
  name : "My Object",
  getName: function(){
    return this.name;
  }
};

object.getName(); //"My Object"
(object.getName)(); //"My Object"
(object.getName = object.getName)(); //"The Window"，在非严格模式下
```

第一行代码跟平常一样调用了 object.getName()，返回的是"My Object"，因为 this.name 就是 object.name。第二行代码在调用这个方法前先给它加上了括号。虽然加上括号之后，就好像只 是在引用一个函数，但 this 的值得到了维持，因为 object.getName 和(object.getName)的定义 是相同的。第三行代码先执行了一条赋值语句，然后再调用赋值后的结果。因为这个赋值表达式的值是 函数本身，所以 this 的值不能得到维持，结果就返回了"The Window"。

### 内存泄漏

由于 IE9 之前的版本对 JScript 对象和 COM 对象使用不同的垃圾收集例程. 因此闭包在 IE 的这些版本中会导致一些特殊的问题。

```javascript
 function assignHandler(){
  var element = document.getElementById("someElement");
  element.onclick = function(){
    alert(element.id);
  };
}
```

以上代码创建了一个作为 `element` 元素事件处理程序的闭包，而这个闭包则又创建了一个循环引用。由于匿名函数保存了一个对 `assignHandler()`的活动对象的引用，因此就会导致无法减少 `element` 的引用数。只要匿名函数存在，`element`的引用数至少也是 1，因此它所 占用的内存就永远不会被回收。不过，这个问题可以通过稍微改写一下代码来解决，如下所示:

```javascript
function assignHandler(){
  var element = document.getElementById("someElement");
  var id = element.id;
  element.onclick = function(){
    alert(id);
  };
  element = null;
}
```

在上面的代码中，通过把 `element.id` 的一个副本保存在一个变量中，并且在闭包中引用该变量消除了循环引用。但仅仅做到这一步，还是不能解决内存泄漏的问题。必须要记住:闭包会引用包含函数 的整个活动对象，而其中包含着 `element`。即使闭包不直接引用 `element`，包含函数的活动对象中也仍然会保存一个引用。因此，有必要把 `element` 变量设置为 `null`。这样就能够解除对 `DOM` 对象的引用，顺利地减少其引用数，确保正常回收其占用的内存。

## 模仿块级作用域

JavaScript没有块级作用域, 这就意味着在块语句中定义的变量, 实际上是在包含函数中, 而并非语句中创建的.

```javascript
function outputNumbers(count){
  for (var i=0; i < count; i++){
    alert(i);
  }
  alert(i); //计数
}
```

这个函数中定义了一个 for 循环，而变量 i 的初始值被设置为 0。在 Java、C++等语言中，变量 i 只会在 for 循环的语句块中有定义，循环一旦结束，变量 i 就会被销毁。可是在 JavaScrip 中，变量 i 是定义在 ouputNumbers()的活动对象中的，因此从它有定义开始，就可以在函数内部随处访问它。即 使像下面这样错误地重新声明同一个变量，也不会改变它的值:

```javascript
function outputNumbers(count){
  for (var i=0; i < count; i++){
    alert(i);
  }
  var i; //重新声明变量.
  alert(i); //计数
}
```

JavaScript 从来不会告诉你是否多次声明了同一个变量;遇到这种情况，它只会对后续的声明视而不 见(不过，它会执行后续声明中的变量初始化)。匿名函数可以用来模仿块级作用域并避免这个问题。

```javascript
(function(){
  // 这里是块级作用域
})();

// 类似于
var someFunction = function(){ //这里是块级作用域
};
someFunction();
```

为什么要在函数外面加上`()`, 这是为了将函数声明转换为函数表达式(函数声明后面不可以跟圆括号). 使用方式:

```javascript
function outputNumbers(count){
  (function(){
    for (var i=0; i < count; i++){
      alert(i);
    }
  })();
  alert(i); // 报错
}
```

在这个重写后的 outputNumbers()函数中，我们在 for 循环外部插入了一个私有作用域。在匿名 函数中定义的任何变量，都会在执行结束时被销毁。因此，变量 i 只能在循环中使用，使用后即被销毁。 而在私有作用域中能够访问变量 count，是因为这个匿名函数是一个闭包，它能够访问包含作用域中的所有变量。

这种技术经常在全局作用域中被用在函数外部，从而限制向全局作用域中添加过多的变量和函数。 一般来说，我们都应该尽量少向全局作用域中添加变量和函数。在一个由很多开发人员共同参与的大型 应用程序中，过多的全局变量和函数很容易导致命名冲突。而通过创建私有作用域，每个开发人员既可以使用自己的变量，又不必担心搞乱全局作用域。例如:

```javascript
(function(){
  var now = new Date();
  if (now.getMonth() == 0 && now.getDate() == 1){
      alert("Happy new year!");
  }
})();
```

这种做法可以减少闭包占用的内存问题，因为没有指向匿名函数的引用。只要函数执行完毕，就可以立即销毁其作用域链了。

## 私有变量

严格来讲，JavaScript 中没有私有成员的概念;所有对象属性都是公有的。不过，倒是有一个私有 变量的概念。任何在函数中定义的变量，都可以认为是私有变量，因为不能在函数的外部访问这些变量。

```javascript
function MyObject(){
  //私有变量和私有函数 3
  var privateVariable = 10;
  function privateFunction(){
    return false;
  }

  //特权方法
  this.publicMethod = function (){
    privateVariable++;
    return privateFunction();
  };
}
```

利用私有和特权成员，可以隐藏那些不应该被直接修改的数据，例如:

```javascript
function Person(name){
  this.getName = function(){
    return name;
  };
  this.setName = function (value) {
    name = value;
  };
}
var person = new Person("Nicholas");
alert(person.getName());   //"Nicholas"
person.setName("Greg");
alert(person.getName());   //"Greg"
```

### 静态私有变量

通过在私有作用域中定义私有变量或函数，同样也可以创建特权方法，其基本模式如下所示。

```javascript
(function(){
  //私有变量和私有函数
  var privateVariable = 10;
    function privateFunction(){
      return false;
  }
  //构造函数
  MyObject = function(){ };
  //公有/特权方法
  MyObject.prototype.publicMethod = function(){
    privateVariable++;
    return privateFunction();
  };
})();
```

这个模式创建了一个私有作用域，并在其中封装了一个构造函数及相应的方法。在私有作用域中， 首先定义了私有变量和私有函数，然后又定义了构造函数及其公有方法。公有方法是在原型上定义的， 这一点体现了典型的原型模式。需要注意的是，这个模式在定义构造函数时并没有使用函数声明，而是使用了函数表达式。函数声明只能创建局部函数，但那并不是我们想要的。出于同样的原因，我们也没有在声明 `MyObject` 时使用 `var` 关键字。记住: `初始化未经声明的变量，总是会创建一个全局变量。` 因此，`MyObject` 就成了一个全局变量，能够在私有作用域之外被访问到。但也要知道，在严格模式下 给未经声明的变量赋值会导致错误。

这个模式与在构造函数中定义特权方法的主要区别，就在于私有变量和函数是由实例共享的。由于特权方法是在原型上定义的，因此所有实例都使用同一个函数。而这个特权方法，作为一个闭包，总是保存着对包含作用域的引用。来看一看下面的代码。

```javascript
(function(){
  var name = "";
  Person = function(value){
    name = value;
  };
  Person.prototype.getName = function(){
    return name;
  };
  Person.prototype.setName = function (value){
    name = value;
  };
})();

var person1 = new Person("Nicholas");
alert(person1.getName()); //"Nicholas"
person1.setName("Greg");
 alert(person1.getName()); //"Greg"
var person2 = new Person("Michael");
alert(person1.getName()); //"Michael"
alert(person2.getName()); //"Michael"
```

这个例子中的 Person 构造函数与 getName()和 setName()方法一样，都有权访问私有变量 name。 在这种模式下，变量 name 就变成了一个静态的、由所有实例共享的属性。也就是说，在一个实例上调 用 setName()会影响所有实例。而调用 setName()或新建一个 Person 实例都会赋予 name 属性一个 新值。结果就是所有实例都会返回相同的值。

以这种方式创建静态私有变量会因为使用原型而增进代码复用，但每个实例都没有自己的私有变 量。到底是使用实例变量，还是静态私有变量，最终还是要视你的具体需求而定。

### 模块模式

模块模式(module pattern)则是为单例创建私有变量和特权方法。所谓单例(singleton)，指的就是只有一个实例的对象。 按照惯例，`JavaScript`是以对象字面量的方式来创建单例对象的:

```javascript
// 一般情况
var singleton = {
  name : value,
  method : function () { //这里是方法的代码
  }
};
```

然后我们可以通过`模块模式`来对单例进行增强, 如添加变量和特权方法. 如:

```javascript
var singleton = function(){
  //私有变量和私有函数
  var privateVariable = 10;
  function privateFunction(){
    return false;
  }

  //特权和公有方法和属性
  return {
    publicProperty: true,
      publicMethod : function(){
        privateVariable++;
        return privateFunction();
      }
    };
  }();
```

这个模块模式使用了一个返回对象的匿名函数。在这个匿名函数内部，首先定义了私有变量和函数。 然后，将一个对象字面量作为函数的值返回。返回的对象字面量中只包含可以公开的属性和方法。由于这个对象是在匿名函数内部定义的，因此它的公有方法有权访问私有变量和函数。从本质上来讲，这个对象字面量定义的是单例的公共接口。这种模式在需要对单例进行某些初始化，同时又需要维护其私有 变量时是非常有用的，例如:

```javascript
var application = function(){
  //私有变量和函数
  var components = new Array();
  //初始化
  components.push(new BaseComponent());
  //公共
  return {
    getComponentCount : function(){
      return components.length;
    },
    registerComponent : function(component){
      if (typeof component == "object"){
        components.push(component);
      }
    }
  };
}();
```

### 增强的模块模式

有人进一步改进了模块模式，即在返回对象之前加入对其增强的代码。这种增强的模块模式适合那 些单例必须是某种类型的实例，同时还必须添加某些属性和(或)方法对其加以增强的情况。来看下面的例子。

```javascript
var singleton = function(){
  //私有变量和私有函数
  var privateVariable = 10;
  function privateFunction(){
    return false;
  }

  // 创建对象
  var object = new CustomType();

  // 添加特权/公有属性和方法
  object.publicProperty = true;
  object.publicMethod = function() {
    privateVariable++;
    return privateFunction();
  }

  return object;
  }();
```

如果前面演示模块模式的例子中的 application 对象必须是 BaseComponent 的实例，那么就可 以使用以下代码。

```javascript
var application = function(){
  //私有变量和函数
  var components = new Array();
  //初始化
  components.push(new BaseComponent());
  //创建 application 的一个局部副本
  var app = new BaseComponent();
  //公共接口
  app.getComponentCount = function(){
    return components.length;
  };
  app.registerComponent = function(component){
    if (typeof component == "object"){
      components.push(component);
    }
  }; //返回这个副本
  return app;
}();
```

在这个重写后的应用程序(application)单例中，首先也是像前面例子中一样定义了私有变量。主要的不同之处在于命名变量 app 的创建过程，因为它必须是 BaseComponent 的实例。这个实例实际上是 application 对象的局部变量版。此后，我们又为 app 对象添加了能够访问私有变量的公有方法。 最后一步是返回 app 对象，结果仍然是将它赋值给全局变量 application。

## 小结

在 JavaScript 编程中，函数表达式是一种非常有用的技术。使用函数表达式可以无须对函数命名， 从而实现动态编程。匿名函数，也称为拉姆达函数，是一种使用 JavaScript 函数的强大方式。以下总结了函数表达式的特点。

- 函数表达式不同于函数声明。函数声明要求有名字，但函数表达式不需要。没有名字的函数表达式也叫做匿名函数。
- 在无法确定如何引用函数的情况下，递归函数就会变得比较复杂;
- 递归函数应该始终使用 arguments.callee 来递归地调用自身，不要使用函数名——函数名可能会发生变化。

当在函数内部定义了其他函数时，就创建了闭包。闭包有权访问包含函数内部的所有变量，原理
如下。

- 在后台执行环境中，闭包的作用域链包含着它自己的作用域、包含函数的作用域和全局作用域。
- 通常，函数的作用域及其所有变量都会在函数执行结束后被销毁。
- 但是，当函数返回了一个闭包时，这个函数的作用域将会一直在内存中保存到闭包不存在为止。

使用闭包可以在 JavaScript 中模仿块级作用域(JavaScript 本身没有块级作用域的概念)，要点如下。

- 创建并立即调用一个函数，这样既可以执行其中的代码，又不会在内存中留下对该函数的引用。
- 结果就是函数内部的所有变量都会被立即销毁——除非将某些变量赋值给了包含作用域(即外部作用域)中的变量。

闭包还可以用于在对象中创建私有变量，相关概念和要点如下。

- 即使 JavaScript 中没有正式的私有对象属性的概念，但可以使用闭包来实现公有方法，而通过公有方法可以访问在包含作用域中定义的变量。
- 有权访问私有变量的公有方法叫做特权方法。
- 可以使用构造函数模式、原型模式来实现自定义类型的特权方法，也可以使用模块模式、增强的模块模式来实现单例的特权方法。

JavaScript 中的函数表达式和闭包都是极其有用的特性，利用它们可以实现很多功能。不过，因为创建闭包必须维护额外的作用域，所以过度使用它们可能会占用大量内存.
