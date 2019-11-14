# JavaScript 面向对象编程

`ECMA-262`把对象定义为:“无序属性的集合，其属性可以包含基本值、对象或者函数。”严格来讲，这就相当于说对象是一组没有特定顺序的值。对象的每个属性或方法都有一个名字，而每个名字都映射到一个值。可以把`ECMAScript`的对象想象成散列表:无非就是一组名值对，其中值可以是数据或函数。

## 理解对象

创建方式:

```javascript
// create1
var person = new Object();
person.name = 'Nicholas';
person.age = 29;
person.job = 'Software Engineer';
person.sayName = function() {
  alert(this.name);
};

//create2
var person = {
  name: 'Nicholas',
  age: 29,
  job: 'Software Engineer',
  sayName: function() {
    alert(this.name);
  },
};
```

ECMA-262 第 5 版在定义了内部才用的特性(attribute)，用于描述了属性(property)的各种特征。就是我们常称的`属性类型`:

- **Configurable**: 表示能否通过 delete 删除属性从而重新定义属性，能否修改属性的特性，或者能否把属性修改为访问器属性。像前面例子中那样直接在对象上定义的属性，特性默认值为 true。
- **Enumerable**: 表示能否通过 for-in 循环返回属性。像前面例子中那样直接在对象上定义的属性，特性默认值为 true。
- **Writable**: 表示能否修改属性的值。像前面例子中那样直接在对象上定义的属性，特性默认值为 true.
- **Value**: 包含这个属性的数据值。读取属性值的时候，从这个位置读;写入属性值的时候，把新值保存在这个位置。这个特性的默认值为 undefined。
- **Get**: 在读取属性时调用的函数。默认值为 undefined。
- **Set**: 在写入属性时调用的函数。默认值为 undefined。

```javascript
var person = {};

// writable
Object.defineProperty(person, 'name', {
  writable: false, //设置为false之后, 不能写入
  value: 'Nicholas',
});
alert(person.name); //"Nicholas"
person.name = 'Greg';
alert(person.name); //"Nicholas"

// configurable setting
Object.defineProperty(person, 'name', {
  configurable: false, // 设置false之后, 不能删除,不能更改
  value: 'Nicholas',
});
alert(person.name); //"Nicholas"
delete person.name;
alert(person.name); //"Nicholas"

// once set configurable to false, you can't set it to ture
Object.defineProperty(person, 'name', {
  configurable: false,
  value: 'Nicholas',
});
//抛出错误
Object.defineProperty(person, 'name', {
  configurable: true,
  value: 'Nicholas',
});

// get and set
var book = {
  _year: 2004,
  edition: 1,
};
Object.defineProperty(book, 'year', {
  get: function() {
    return this._year;
  },
  set: function(newValue) {
    if (newValue > 2004) {
      this._year = newValue;
      this.edition += newValue - 2004;
    }
  },
});
book.year = 2005;
alert(book.edition); //2

// read configure infos
Object.defineProperties(book, {
  _year: {
    value: 2004,
  },
  edition: {
    value: 1,
  },
  year: {
    get: function() {
      return this._year;
    },
    set: function(newValue) {
      if (newValue > 2004) {
        this._year = newValue;
        this.edition += newValue - 2004;
      }
    },
  },
});
var descriptor = Object.getOwnPropertyDescriptor(book, '_year');
alert(descriptor.value); //2004
alert(descriptor.configurable); //false
alert(typeof descriptor.get); //"undefined"
var descriptor = Object.getOwnPropertyDescriptor(book, 'year');
alert(descriptor.value); //undefined
alert(descriptor.enumerable); //false
alert(typeof descriptor.get); //"function"
```

## 创建对象

虽然 Object 构造函数或对象字面量都可以用来创建单个对象，但这些方式有个明显的缺点:使用同
一个接口创建很多对象，会产生大量的重复代码。

### 工厂模式

根据传递的参数来构建对象,常见的设计模式.

```javascript
function createPerson(name, age, job) {
  var o = new Object();
  o.name = name;
  o.age = age;
  o.job = job;
  o.sayName = function() {
    alert(this.name);
  };
  return o;
}
var person1 = createPerson('Nicholas', 29, 'Software Engineer');
var person2 = createPerson('Greg', 27, 'Doctor');
```

### 构造函数模式

```javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
  this.sayName = function() {
    alert(this.name);
  };
}
var person1 = new Person('Nicholas', 29, 'Software Engineer');
var person2 = new Person('Greg', 27, 'Doctor');
```

这里的构造函数和一般的函数有明显的区别:

- 函数名以大写字母开头
- 必须使用 new 操作符
- 没有显式的创建对象
- 直接将属性和方法赋给了 this 对象
- 没有 return 语句

调用构造函数,一般会经历以下四个步骤:

1. 创建一个新对象;
2. 将构造函数的作用域赋给新对象(因此 this 就指向了这个新对象);
3. 执行构造函数中的代码(为这个新对象添加属性);
4. 返回新对象。

需要注意的是, 构造函数也是函数, 唯一的区别就是通过 new 来调用. 任何函数，只要通过 new 操作符来调用，那它就可以作为构造函数;而 任何函数，如果不通过 new 操作符来调用，那它跟普通函数也不会有什么两样。

```javascript
// 当作构造函数使用
var person = new Person('Nicholas', 29, 'Software Engineer');
person.sayName(); //"Nicholas"
// 作为普通函数调用
Person('Greg', 27, 'Doctor'); // 添加到window
window.sayName(); //"Greg"
// 在另一个对象的作用域中调用
var o = new Object();
Person.call(o, 'Kristen', 25, 'Nurse');
o.sayName(); //"Kristen"
```

构造函数的缺点就是, 没有共享的方法和属性, 任何一个方法或者对象都需要重新创建. 如:

```javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
  this.sayName = new Function('alert(this.name)'); // 与声明函数在逻辑上是等价的
}
```

### 原型模式

我们创建的每个函数都有一个 prototype(原型)属性，这个属性是一个指针，指向一个对象， 而这个对象的用途是包含可以由特定类型的所有实例共享的属性和方法。

```javascript
function Person() {}

Person.prototype.name = 'Nicholas';
Person.prototype.age = 29;
Person.prototype.job = 'Software Engineer';
Person.prototype.sayName = function() {
  alert(this.name);
};

var person1 = new Person();
person1.sayName(); //"Nicholas"
var person2 = new Person();
person2.sayName(); //"Nicholas"
alert(person1.sayName == person2.sayName); //true
```

图例:

![js_protype](https://image.cjyong.com/js_prototype.png)

由于对象属性的搜索方式, 是先从对象本身, 然后再去查找原型链. 我们可以通过在对象本身声明一个和原型链同名的对象来覆盖原型链的值.

```javascript
function Person() {}

Person.prototype.name = "Nicholas";
Person.prototype.age = 29;
Person.prototype.job = "Software Engineer"; 12 Person.prototype.sayName = function(){
    alert(this.name);
};
var person1 = new Person();
var person2 = new Person();
person1.name = "Greg";
alert(person1.name); //"Greg"——来自实例
alert(person2.name); //"Nicholas"——来自原型

delete person1.name;
alert(person1.name); //"Nicholas"——来自原型
```

`hasOwnProperty()`方法可以检测一个属性是否存在于实例中, 还是存在于原型中. 如果对象只存在对象的实例中, 才会返回 `true`. 另外结合`in`函数(通过对象访问给定属性时, 如果存在实例或者原型中, 则返回`true`), 就可以判断是否是在原型中的属性.

```javascript
function Person() {}
Person.prototype.name = 'Nicholas';
Person.prototype.age = 29;
Person.prototype.job = 'Software Engineer';
Person.prototype.sayName = function() {
  alert(this.name);
};
var person1 = new Person();
var person2 = new Person();
alert(person1.hasOwnProperty('name')); //false
alert('name' in person1); //true
person1.name = 'Greg';
alert(person1.name); //"Greg" ——来自实例
alert(person1.hasOwnProperty('name')); //true
alert('name' in person1); //true

function hasPrototypeProperty(object, name) {
  return !object.hasOwnProperty(name) && name in object;
}

var person = new Person();
alert(hasPrototypeProperty(person, 'name')); //true
person.name = 'Greg';
alert(hasPrototypeProperty(person, 'name')); //false
```

简单化声明原型:

```javascript
function Person() {}
Person.prototype = {
  name: 'Nicholas',
  age: 29,
  job: 'Software Engineer',
  sayName: function() {
    alert(this.name);
  },
};

//重设构造函数，只适用于 ECMAScript 5 兼容的浏览器
Object.defineProperty(Person.prototype, 'constructor', {
  enumerable: false,
  value: Person,
});
```

这里通过`defineProperty`来定义`constructor`函数, 不能直接写在字面量上, 那样会直接导致构造函数可枚举(自动化生成的构造函数式不可枚举的).

原型具备动态性, 每次更新原型对象, 都会动态映射到所有的关联的对象中.

```javascript
var friend = new Person();
Person.prototype.sayHi = function() {
  alert('hi');
};
friend.sayHi(); //"hi"
```

尽管可以随时为原型添加属性和方法，并且修改能够立即在所有对象实例中反映出来，但如果是重写整个原型对象，那么情况就不一样了。调用构造函数时会为实例添加一个指向最初原型的`[[Prototype]]`指针，而把原型修改为另外一个对象就等于切断了构造函数与最初原型之间的联系。 请记住:实例中的指针仅指向原型，而不指向构造函数。

![prototype_rewrite](https://image.cjyong.com/js_prototype_rewrite.png)

原型模式存在的问题就在于, 属性共享. 当某一个实例修改了原型中的属性, 那么将会映射到所有的实例中.

```javascript
function Person() {}
Person.prototype = {
  constructor: Person,
  name: 'Nicholas',
  age: 29,
  job: 'Software Engineer',
  friends: ['Shelby', 'Court'],
  sayName: function() {
    alert(this.name);
  },
};
var person1 = new Person();
var person2 = new Person();
person1.friends.push('Van');
alert(person1.friends); //"Shelby,Court,Van"
alert(person2.friends); //"Shelby,Court,Van"
alert(person1.friends === person2.friends); //true
```

### 组合构造函数和原型模式

创建自定义类型的最常见方式，就是组合使用构造函数模式与原型模式。构造函数模式用于定义实例属性，而原型模式用于定义方法和共享的属性。 如前面的例子:

```javascript
function Person(name, age, job){
  this.name = name; 3 this.age = age;
  this.job = job;
  this.friends = ["Shelby", "Court"];
}

Person.prototype = {
    constructor : Person,
    sayName : function(){
        alert(this.name);
    }
}

var person1 = new Person("Nicholas", 29, "Software Engineer");
var person2 = new Person("Greg", 27, "Doctor");

person1.friends.push("Van");
alert(person1.friends);    //"Shelby,Count,Van"
alert(person2.friends);    //"Shelby,Count"
alert(person1.friends === person2.friends); // false
alert(person1.sayName === person2.sayName); // true
```

这是在`ECMAScript`中使用最广泛, 认同度最高的一种创建自定义类型的方法. 也是定义引用类型的默认模式.

### 动态原型模式

其他 OO 语言经验的开发人员在看到独立的构造函数和原型时，很可能会感到非常困惑。动态原型模式正是致力于解决这个问题的一个方案，它把所有信息都封装在了构造函数中，而通过在构造函数中初始化原型(仅在必要的情况下)，又保持了同时使用构造函数和原型的优点。

```javascript
function Person(name, age, job){
  this.name = name; 3 this.age = age;
  this.job = job;
  this.friends = ["Shelby", "Court"];

  // method
  if (typeof this.sayName != "function"){
    Person.prototype.sayName = function(){
        alert(this.name);
    };
  }
}
```

这样就可以将所有的组合操作放到构造函数中进行了, 对外界只暴露了一个构造函数.

### 寄生构造函数模式

在前述的几种模式都不适用的情况下，可以使用寄生(parasitic)构造函数模式。这种模式的基本思想是创建一个函数，该函数的作用仅仅是封装创建对象的代码，然后再返回新创建的对象;但从表面上看，这个函数又很像是典型的构造函数。

假设我们想创建一个具有额外方法的特殊数组。由于不能直接修改 Array 构造函数，因此可以使用这个模式。

```javascript
function SpecialArray() {
  //创建数组
  var values = new Array();
  // 添加值
  values.push.apply(values, arguments);
  //添加方法
  values.toPipedString = function() {
    return this.join('|');
  };
  //返回数组
  return values;
}
var colors = new SpecialArray('red', 'blue', 'green');
alert(colors.toPipedString()); //"red|blue|green"
```

这里通过在构造函数中返回`return`, 重写了调用构造函数时返回的值. 但是这个模式存在一个缺点就是: 返回的对象和构造函数没有任何关系, 就类似于直接在外部创建的字面量. 因此不能使用`instanceof`操作符来确定对象类型. 存在这种情况, 推荐不使用这种模式.

### 稳妥构造函数模式

所谓稳妥对象，指的是没有公共属性，而且其方法也不引用`this`的对象。稳妥对象最适合在一些安全的环境中，或者在防止数据被其他应用程序改动时使用。 如创建一个安全的`Person`对象:

```javascript
function Person(name, age, job) {
  //创建要返回的对象
  var o = new Object();
  //可以在这里定义私有变量和函数
  //添加方法
  o.sayName = function() {
    alert(name);
  };
  //返回对象
  return o;
}
```

这里的`Person`实例, 除了调用`sayName()`方法外, 没有别的方法访问到其他数据成员. 因为这些属性都不是挂载在返回的对象中的.

## 继承

继承是 OO 语言中的一个最为人津津乐道的概念。许多 OO 语言都支持两种继承方式:接口继承和实现继承。 接口继承只继承方法签名，而实现继承则继承实际的方法。如前所述，由于函数没有签名， 在 ECMAScript 中无法实现接口继承。ECMAScript 只支持实现继承，而且其实现继承主要是依靠原型链来实现的。

### 原型链

ECMAScript 中描述了原型链的概念，并将原型链作为实现继承的主要方法。其基本思想是利用原型让一个引用类型继承另一个引用类型的属性和方法: 每个构造函数都有一个原型对象，原型对象都包含一个指向构造函数的指针，而实例都包含一个指向原型对象的内部指针。那么，假如我们让原型对象等于另一个类型的实例，结果会怎么样呢?显然，此时的 原型对象将包含一个指向另一个原型的指针，相应地，另一个原型中也包含着一个指向另一个构造函数 的指针。假如另一个原型又是另一个类型的实例，那么上述关系依然成立，如此层层递进，就构成了实例与原型的链条。这就是所谓原型链的基本概念。

```javascript
function SuperType() {
  this.property = true;
}

SuperType.prototype.getSuperValue = function() {
  return this.property;
};

function SubType() {
  this.subproperty = false;
}

//继承了 SuperType
SubType.prototype = new SuperType();

SubType.prototype.getSubValue = function() {
  return this.subproperty;
};

var instance = new SubType();
alert(instance.getSuperValue()); //true
```

![js_inherit_1](https://image.cjyong.com/js_inherit_1.png)

在通过原型链实现继承的情况下，搜索过程就得以沿着原型链继续向上。就拿上面的例子来说，调用 `instance.getSuperValue()`会经历三个搜索步骤:1)搜索实例; 2)搜索`SubType.prototype`; 3)搜索`SuperType.prototype`. 最后一步才会找到该方法。在找不到属性或方法的情况下，搜索过 程总是要一环一环地前行到原型链末端才会停下来。 当然上面展示的原型链还少了一环, 那就是所有的引用类型默认都继承了`Object`, 而这个继承也是通过原型链实现的.

![js_inherit_1](https://image.cjyong.com/js_inherit_2.png)

`SubType`继承了`SuperType`，而`SuperType`继承了`Object`。当调用`instance.toString()`时，实际上调用的是保存在`Object.prototype`中的那个方法。

怎么确认原型和实例之间的关系: `instanceof`和`isPrototypeOf`方法:

```javascript
alert(instance instanceof Object); // true
alert(instance instanceof SuperType); // true
alert(instance instanceof SubType); // true
alert(Object.prototype.isPrototypeOf(instance)); // true
alert(SuperType.prototype.isPrototypeOf(instance)); // true
alert(SubType.prototype.isPrototypeOf(instance)); // true
```

原型链实现继承, 虽然可以, 但是还存在一些问题: 引用类型的原型. 当我们在一个父类中声明实例属性时, 会想当然的变成原型属性.

```javascript
function SuperType() {
  this.colors = ['red', 'blue', 'green'];
}
//继承了 SuperType
SubType.prototype = new SuperType();
var instance1 = new SubType();
instance1.colors.push('black');
alert(instance1.colors); //"red,blue,green,black"
var instance2 = new SubType();
alert(instance2.colors); //"red,blue,green,black"
```

第二个问题就是在创建子类型的实例时，不能向超类型的构造函数中传递参数。实际上，应该说是没有办法在不影响所有对象实例的情况下，给超类型的构造函数传递参数。 由于上面的问题, 实践中很少会单独使用原型链.

### 借用构造函数

在解决原型中包含引用类型值所带来问题的过程中，开发人员开始使用一种叫做借用构造函数(`constructor stealing`)的技术(有时候也叫做伪造对象或经典继承)。这种技术的基本思想相当简单，即在子类型构造函数的内部调用超类型构造函数。

```javascript
function SuperType() {
  this.colors = ['red', 'blue', 'green'];
}
function SubType() {
  //继承了 SuperType
  SuperType.call(this);
}
var instance1 = new SubType();
instance1.colors.push('black');
alert(instance1.colors); //"red,blue,green,black"
var instance2 = new SubType();
alert(instance2.colors); //"red,blue,green"
```

通过使用`call()`方法(或 `apply()`方法 也可以)，我们实际上是在(未来将要)新创建的 `SubType` 实例的环境下调用了 `SuperType` 构造函数。 这样一来，就会在新 SubType 对象上执行 SuperType()函数中定义的所有对象初始化代码。结果， `SubType` 的每个实例就都会具有自己的 `colors` 属性的副本了. 另外还可以往窃取的构造函数中传递参数.

```javascript
function SuperType(name) {
  this.name = name;
}
function SubType() {
  //继承了 SuperType，同时还传递了参数
  SuperType.call(this, 'Nicholas');
  //实例属性
  this.age = 29;
}
var instance = new SubType();
alert(instance.name); //"Nicholas";
alert(instance.age); //29
```

存在的问题: 只是窃取了父类构造函数内部的属性, 没有办法获取父类原型链中的方法和属性. 因为子类的原型链还是指向自己的原型链, 并没有和父类关联上. 考虑到这些问题, 这种技术也很少单独使用.

### 组合继承

组合继承(combination inheritance)，有时候也叫做伪经典继承，指的是将原型链和借用构造函数的 技术组合到一块，从而发挥二者之长的一种继承模式。其背后的思路是使用原型链实现对原型属性和方 法的继承，而通过借用构造函数来实现对实例属性的继承.

```javascript
function SuperType(name) {
  this.name = name;
  this.colors = ['red', 'blue', 'green'];
}
SuperType.prototype.sayName = function() {
  alert(this.name);
};
function SubType(name, age) {
  //继承属性, 调用父类的构造函数, 继承name和colors
  SuperType.call(this, name);
  this.age = age;
}
//继承方法, 设置父类的实例为子类原型
SubType.prototype = new SuperType();
SubType.prototype.constructor = SubType;
SubType.prototype.sayAge = function() {
  alert(this.age);
};
var instance1 = new SubType('Nicholas', 29);
instance1.colors.push('black');
alert(instance1.colors); //"red,blue,green,black"
instance1.sayName(); //"Nicholas";
instance1.sayAge(); //29

var instance2 = new SubType('Greg', 27);
alert(instance2.colors); //"red,blue,green"
instance2.sayName(); //"Greg";
instance2.sayAge(); //27
```

组合继承避免了原型链和借用构造函数的缺陷，融合了它们的优点，成为 JavaScript 中最常用的继 承模式。而且，instanceof 和 isPrototypeOf()也能够用于识别基于组合继承创建的对象。

### 原型式继承

道格拉斯·克罗克福德在 2006 年写了一篇文章，题为 Prototypal Inheritance in JavaScript (JavaScript 10 中的原型式继承)。在这篇文章中，他介绍了一种实现继承的方法，这种方法并没有使用严格意义上的 构造函数。他的想法是借助原型可以基于已有的对象创建新对象，同时还不必因此创建自定义类型。为 了达到这个目的，他给出了如下函数:

```javascript
function object(o) {
  function F() {}
  F.prototype = o;
  return new F();
}
```

在 object()函数内部，先创建了一个临时性的构造函数，然后将传入的对象作为这个构造函数的 原型，最后返回了这个临时类型的一个新实例。从本质上讲，object()对传入其中的对象执行了一次浅复制.

```javascript
var person = {
  name: 'Nicholas',
  friends: ['Shelby', 'Court', 'Van'],
};
var anotherPerson = object(person);
anotherPerson.name = 'Greg';
anotherPerson.friends.push('Rob');
var yetAnotherPerson = object(person);
yetAnotherPerson.name = 'Linda';
yetAnotherPerson.friends.push('Barbie');
alert(person.friends); //"Shelby,Court,Van,Rob,Barbie"
```

`ECMAScript 5`通过新增`Object.create()`方法规范化了原型式继承。这个方法接收两个参数:一 个用作新对象原型的对象和(可选的)一个为新对象定义额外属性的对象。在传入一个参数的情况下，Object.create()与 object()方法的行为相同。

```javascript
var person = {
  name: 'Nicholas',
  friends: ['Shelby', 'Court', 'Van'],
};
var anotherPerson = Object.create(person);
anotherPerson.name = 'Greg';
anotherPerson.friends.push('Rob');
var yetAnotherPerson = Object.create(person);
yetAnotherPerson.name = 'Linda';
yetAnotherPerson.friends.push('Barbie');
alert(person.friends); //"Shelby,Court,Van,Rob,Barbie"
```

其中第二个参数与 Object.defineProperties()方法的第二个参数格式相同. 可以传递参数描述符, 覆盖原型对象中的值.

```javascript
var anotherPerson = Object.create(person, {
  name: {
    value: 'Greg',
  },
});
```

在没有必要兴师动众地创建构造函数，而只想让一个对象与另一个对象保持类似的情况下，原型式继承是完全可以胜任的。不过别忘了，包含引用类型值的属性始终都会共享相应的值，就像使用原型模式一样。

### 寄生式继承

寄生式(parasitic)继承是与原型式继承紧密相关的一种思路，并且同样也是由克罗克福德推而广 之的。寄生式继承的思路与寄生构造函数和工厂模式类似，即创建一个仅用于封装继承过程的函数，该 函数在内部以某种方式来增强对象，最后再像真地是它做了所有工作一样返回对象。以下代码示范了寄生式继承模式:

```javascript
function createAnother(original) {
  var clone = object(original); //通过调用函数创建一个新对象
  clone.sayHi = function() {
    // 某种方式来增强它
    alert('hi');
  };
  return clone; // 返回这个对象
}
```

### 寄生组合式继承

`组合继承`是`JavaScript`最常用的继承模式;不过，它也有自己的不足。组合继承最大的问题就是无论什么情况下，都会调用两次超类型构造函数:一次是在创建子类型原型的时候，另一次是在子类型构造函数内部。没错，子类型最终会包含超类型对象的全部实例属性，但我们不得不在调用子类型构造函数时重写这些属性。

在第一次调用 SuperType 构造函数时， SubType.prototype 会得到两个属性:name 和 colors;它们都是 SuperType 的实例属性，只不过 现在位于 SubType 的原型中。当调用 SubType 构造函数时，又会调用一次 SuperType 构造函数，这 一次又在新对象上创建了实例属性 name 和 colors。于是，这两个属性就屏蔽了原型中的两个同名属性.

寄生组合式继承，即通过借用构造函数来继承属性，通过原型链的混成形式来继承方法。其基本模式为:

```javascript
function inheritPrototype(subType, superType) {
  // 不调用父类的构造函数, 直接拿父类的原型, 给子类用
  var prototype = object(superType.prototype); // 创建对象
  prototype.constructor = subType; // 增强对象
  subType.prototype = prototype; // 指定对象
}
```

如:

```javascript
function SuperType(name) {
  this.name = name;
  this.colors = ['red', 'blue', 'green'];
}

SuperType.prototype.sayName = function() {
  alert(this.name);
};

function SubType(name, age) {
  // 调用父类的构造函数, 复制父类的name和colors
  SuperType.call(this, name);
  this.age = age;
}
inheritPrototype(SubType, SuperType);

SubType.prototype.sayAge = function() {
  alert(this.age);
};
```

这个例子的高效率体现在它只调用了一次 `SuperType` 构造函数，并且因此避免了在 `SubType. prototype` 上面创建不必要的、多余的属性。与此同时，原型链还能保持不变;因此，还能够正常使用 `instanceof` 和 `isPrototypeOf()`。开发人员普遍认为`寄生组合式继承`是`引用类型`最理想的继承范式。

## 总结

ECMAScript 支持面向对象(OO)编程，但不使用类或者接口。对象可以在代码执行过程中创建和 增强，因此具有动态性而非严格定义的实体。在没有类的情况下，可以采用下列模式创建对象。

- 工厂模式，使用简单的函数创建对象，为对象添加属性和方法，然后返回对象。这个模式后来 被构造函数模式所取代。
- 构造函数模式，可以创建自定义引用类型，可以像创建内置对象实例一样使用 new 操作符。不 过，构造函数模式也有缺点，即它的每个成员都无法得到复用，包括函数。由于函数可以不局 限于任何对象(即与对象具有松散耦合的特点)，因此没有理由不在多个对象间共享函数。
- 原型模式，使用构造函数的 prototype 属性来指定那些应该共享的属性和方法。组合使用构造 函数模式和原型模式时，使用构造函数定义实例属性，而使用原型定义共享的属性和方法。

JavaScript 主要通过`原型链`实现继承。原型链的构建是通过将一个类型的实例赋值给另一个构造函数的原型实现的。这样，子类型就能够访问超类型的所有属性和方法，这一点与基于类的继承很相似。 原型链的问题是对象实例共享所有继承的属性和方法，因此不适宜单独使用。解决这个问题的技术是`借用构造函数`，即在子类型构造函数的内部调用超类型构造函数。这样就可以做到每个实例都具有自己的属性，同时还能保证只使用构造函数模式来定义类型。使用最多的继承模式是`组合继承`，这种模式使用原型链继承共享的属性和方法，而通过借用构造函数继承实例属性。此外，还存在下列可供选择的继承模式。

- 原型式继承，可以在不必预先定义构造函数的情况下实现继承，其本质是执行对给定对象的浅
  复制。而复制得到的副本还可以得到进一步改造。
- 寄生式继承，与原型式继承非常相似，也是基于某个对象或某些信息创建一个对象，然后增强
  对象，最后返回对象。为了解决组合继承模式由于多次调用超类型构造函数而导致的低效率问
  题，可以将这个模式与组合继承一起使用。
- 寄生组合式继承，集寄生式继承和组合继承的优点与一身，是实现基于类型继承的最有效方式。
