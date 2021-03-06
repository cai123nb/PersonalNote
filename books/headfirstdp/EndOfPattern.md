# 与设计模式相处

## 定义设计模式

**模式: 模式是在某种情境下, 针对某问题的某种解决方案.**

- 情境: 就是应用某个模式的情况. 这应该是会不断出现的情况.
- 问题: 就是你想在某情境下达到的目标, 但也可以是某情境下的约束.
- 解决方案: 就是你所追求的, 一个通用的设计, 用来解决约束, 达到目标.

> 如果你发现自己处于某个情境下, 面对着所欲达到的目标被一群约束影响着的问题, 然而你能够应用某个设计, 客服这些约束并达到这个目标, 将你引向某个解决方案.

如:

- 问题: 我要如何准时上班
- 情境: 我将钥匙锁在车上了.
- 解决方案: 打破窗户, 进入车内, 启动引擎, 然后开车上班.

这里小结一下之前学习的各种模式:

- 装饰者: 包装一个对象, 以提供新的行为.
- 状态: 封装了基于状态的行为, 并使用委托在行为之前切换.
- 迭代器: 在对象集合之中游走, 而不暴露集合的实现.
- 外观: 简化一群类的接口.
- 策略: 封装可以互换的行为, 并使用委托来决定要使用哪一个.
- 代理: 包装对象, 以控制对此对象的访问.
- 工厂方法: 由子类决定要创建的具体类是哪一个.
- 适配器: 封装对象, 并提供不同的接口.
- 观察者: 让对象能够在状态改变时被通知.
- 模板方法: 由子类决定如何实现一个算法中的步骤.
- 组合: 客户使用一致的方式来处理对象集合和单的对象.
- 单件: 确保有且只有一个对象被创建.
- 抽象工厂: 允许客户创建对象家族, 而无需指定他们的具体类.
- 命令: 封装请求成为对象.

模式可以主要分为 3 中类型: 创建型, 行为型和结构型.

- 创建型: 涉及到将对象实例化, 这类模式都体用一个方法, 将客户从所需要实例化对象中解耦.代表模式有: 工厂, 抽象工厂, 单例.
- 行为型: 只要是行为型模式, 都涉及到类和对象如何交互及分配职责.代表模式有: 模板方法, 迭代器, 命令, 观察者, 状态, 策略.
- 结构型: 可以让你把类型或对象组合到更大的结构中.代表模式有: 代理, 装饰, 组合, 外观, 适配器.

还可以这样分: 类模式和对象模式.

- 类模式: 描述类之间的关系如何通过继承定义. 类模式之间的关系是在编译时建立的.常见的模式有: 模板方法, 适配器, 工厂方法等.
- 对象模式: 描述对象之间的关系, 而且主要是利用组合定义的. 对象模式之间的关系通常在运行时建立, 而且更加动态, 更有弹性.常见的模式有: 组合, 装饰, 命令, 迭代器, 状态, 抽象工厂, 策略, 代理, 外观, 观察者等等.

## 以反模式歼灭恶势力

反模式: 告诉你如何采用一个不好的解决方案解决一个问题.

> 反模式看起来总像一个号的解决方案, 但是当它真正被采用后, 就会带来麻烦.
> 通过将反模式归档. 我们能够帮助其他人在实现它们之前, 分辨出不好的解决方案.
> 像模式一样, 有许多类型的反模式, 包括开发反模式, OO 反模式, 组织反模式和领域特定反模式.

## 总结

OO 基础:

- 抽象
- 封装
- 多态
- 继承

OO 原则:

- 封装变化
- 多用组合, 少用继承
- 针对接口编程, 不针对实现编程
- 为交互对象之间的松耦合设计而努力
- 对拓展开放, 对修改关闭
- 依赖抽象, 不要依赖具体的类
- 只和朋友说话
- 别找我, 我会找你
- 类应该只有一个改变的理由

要点:

- 让设计模式自然而然地出现在你的设计中, 而不是为了使用而使用.
- 设计模式并非僵化的教条, 你可以依据自己的需要采用或者调整.
- 总是满足的最简单解决方案, 不管它用不用模式.
- 学习设计模式的类目, 可以帮你熟悉这些模式以及他们之间的关系.
- 模式的分类是将模式分成不同的族群, 如果这么做对你有帮助, 那就采用吧.
- 你必须相当专注才能够成为一个模式的专家, 这需要时间也需要耐心, 同时还必须乐意做大量的精化工作.
- 请牢记: 你所遇到的大多数的模式都是现有模式的变体, 而非新模式.
- 模式能够为你带来最大好处之一就是: 让你的团队拥有共享词汇.

## 剩下的模式

### 桥接模式(Bridge Pattern)

不只是改变你的实现, 也改变你的抽象.

就拿我们之前的遥控器的例子来说, 我们第一版使用的设计就是:

| RemoteControl | interface |
| :------------ | --------- |
| on()          | 打开      |
| close()       | 关闭      |
| setChannel()  | 选台      |

然后让不同的遥控器去实现该接口, 添加不同的类如 RCAControl, SonyControl 等等.

但是这样就会出现一个问题, 随着时间的不断推移, 我们要实现的功能越来越多, 我们的遥控器的型号也越来越多, 要控制的电视屏幕的种类也不同. 我们这些类不仅要控制 TV 界面信息, 还要添加新的功能. 因此我们可以通过桥接的方式让抽象和实现进行解耦:

| RemoteControl | interface                                |
| :------------ | ---------------------------------------- |
| implementor   | 实现者, 这里通过组合的形式进行结合       |
| on()          | 打开, 基本方法 通过调用 implementor.on() |
| close()       | 关闭, 基本方法                           |
| setChannel()  | 设置频道, 基本方法                       |

然后我们创建一个是一个实现者接口:

| TV           | interface              |
| :----------- | ---------------------- |
| on()         | 打开, 实现基本方法     |
| close()      | 关闭, 实现基本方法     |
| setChannel() | 设置频道, 实现基本方法 |

我们可以通过实现这个基本接口, 来控制不同的外观样式, 如 RCA, Sony 等.

而我们的功能设计则是通过实现 RemoteControl 来实现进行分别实现. 这样我们就有了两个结构层次, 一个是遥控器, 一个是平台特定的电视实现. 通过桥接, 我们就可以独立的改变两个层次.

**优点**:

- 将实现予以解耦, 让它和界面之间不再永久绑定.
- 抽象和实现可以独立拓展, 不会影响到对方.
- 对于 `具体的抽象类`所做的改变, 不会影响到客户.

**用途和缺点**:

- 适合需要跨越多个平台的图形和窗口系统上.
- 当需要不同的方式改变接口和实现时, 你会发现桥接模式很好用.
- 桥接模式的缺点就是增加了复杂度.

### 生成器

**使用生成器模式(Build Pattern)封装一个产品的构造过程, 并允许按照步骤构造.**

假设你有一份假期安排的工具, 该工具可以按照你的假期长短, 花费, 特殊条件等等. 帮你构建良好的度假计划.

| Client  | Class        |
| :------ | ------------ |
| builder | 构建的工具类 |

我们定义一个接口用来处理对应的请求:

| AbstractBuilder      | interface    |
| :------------------- | ------------ |
| buildDay()           | 添加日期     |
| addHotel()           | 添加旅馆     |
| addSpecialEvenet()   | 添加特殊需求 |
| ...                  | ...          |
| getVacationPlanner() | 获得度假计划 |

具体实现由类来实现该接口进行实现. 而我们要使用的话就可以这样:

```java
builder.buildDay(2).addHotel("xxx").getVacationPlanner();
```

**优点**:

- 将一个复杂对象的创建过程封装起来.
- 允许对象通过多个步骤来创建, 并且可以改变过程.
- 向客户隐藏内部的表现.
- 产品的实现可以被替换, 因为客户只看到一个抽象的接口.

**用途和缺点**:

- 经常用来创建组合接口.
- 与工厂模式相比, 采用生成器模式创建对象额客户, 需要具备更多的领域知识.

### 责任链

**当你想让一个以上的对象有机会处理某个请求的时候, 可以使用责任链模式(Chain Of Responsibility Pattern)**.

假如你是公司的 CEO, 你收到了很多邮件, 但是有很多是垃圾邮件, 也有很多事正常的邮件, 如家人寄过来的, 店家寄过来的, 粉丝寄过来的等等. 你需要进行处理. 我们可以这样进行如理: 我们可以利用一系列的处理器进行处理, 如果该处理器没办法处理, 就让下一个处理器进行处理, 一直以此类推.

**优点**:

- 将请求的发送者和接收者解耦.
- 可以简化你的对象, 因为它不需要知道链的结构.
- 通过改变链内成员或调动它们的次序, 允许你动态地新增或者删除责任.

**用途和缺点**:

- 经常用在窗口系统中, 处理鼠标和键盘之类的事件.
- 并不保证请求一定会被执行.如果没有任何对象处理它的话, 可能会落到链尾端之外.
- 可能不容易观察运行时的特征, 不容易进行调试.

### 蝇量

**如果想让某个类的一个实例能用来提供许多虚拟实例, 就可以使用蝇量模式(Flyweight Pattern)**.

比如你在设计一处风景, 你可以想要添加一些树来进行装饰. 问题是: 用户可能要在他们的家庭景观中添加非常多非常多的树, 但是这样程序就开始变得卡顿了. 这时候, 你可以统一使用一个类进行树的管理, 这个类里面通过一个二维数组进行存储所有的树对象. 这样每棵树都不需要进行实例化了.

**优点**:

- 减少运行时对象实例的个数, 节省内存.
- 将许多"虚拟"对象的状态集中管理.

**用途和缺点**:

- 当一个类有很多实例, 而这些实例能被同一个方法控制时, 我们就可以使用蝇量模式.
- 一旦你实现了它, 就无法实现单个的逻辑, 没办法拥有独立的行为.

### 解释器

**使用解释器模式(Interpreter Pattern)为语言创建解释器**.
**优点**:

- 将每一个语法规则表示成一个类, 方便实现语言.
- 因为语法由许多类表示, 你可以轻易地拓展和改变此语言.
- 通过在类结构中加入新的方法, 可以在解释的同时增加新的行为. 如打印格式的美化或者进行复杂的程序校验.

**用途和缺点**:

- 当你实现一个简单的语言时, 使用解释器.
- 当你有个简单的语法, 而且简单比效率更重要时, 使用解释器.
- 可以处理脚本语言和编程语言.
- 当语法规则数目变大时, 这个模式可能变得非常繁杂, 使用解释器/编译器的产生器可能更加合适.

### 中介者

\*\*使用中介者模式(Mediator Pattern)来集中相关对象之间复杂的沟通和控制方式.

假设你有一个 Java 版本的自动屋, 每当你点击了打盹按钮, 他的闹钟就会告诉咖啡壶开始煮咖啡, 每当要丢垃圾的日子到来了, 就会告诉让闹钟提前 10 分钟等等.如果你将关系写死在这些具体的类中, 这会变得非常复杂和繁琐. 这时候你需要一个中介者, 由它统一地处理你的行为, 将对象之间彻底解耦. 如果你点击打盹按钮, 中介者会告诉闹钟要什么时候响, 咖啡机开始工作等等.

**优点**:

- 通过将对象彼此解耦, 可以增加对象的复用性.
- 可以通过控制逻辑集中, 简化系统维护.
- 可以让对象之见所传递的消息变得简单而且大幅减少.

**用途和缺点**:

- 常常用来协调相关的 GUI 组件.
- 如果设计不当, 中介者会变得非常复杂.

### 备忘录

**当你需要让对象返回之前的某个状态(如撤销), 就使用备忘录模式(MementoPattern)**
例如游戏存档, 恢复等.
**优点**:

- 将被存储的状态放在外面, 不要和关键对象混在一起, 这可以帮助维护内聚.
- 保持关键对象的数据封装.
- 提供了容易实现的恢复能力.

**用途和缺点**:

- 用户存储状态.
- 存储和恢复状态的过程可能相当耗时.
- 可以考虑使用序列化操作.

### 原型

**当创建给定类的过程很昂贵或很复杂时, 使用原型模式(Prototype Pattern).**

原型模式允许你通过复制现有的实例对象来创建新的实例(如 clone(), 反序列化). 当客户代码不知道要序列化何种特定类的情况下, 可以制造出新的实例.

**优点**:

- 向客户隐藏制造新实例的复杂性.
- 提供让客户能够产生未知类型对象的选项.
- 在某些环境下, 复制对象比创建对象更有效.

**用途和缺点**:

- 复杂类层次中, 当系统必须从其中许多类型创建新的对象时, 可以考虑原型.
- 缺点: 对象的复制有时相当复杂.

### 访问者

**当你想要成为一个对象的组合增加新的能力, 且封装并不重要时, 就是用访问者模式(Visitor Pattern)**.

煎饼屋有些常客, 最近他们非常重视养生, 每次订餐之前, 都会询问营养信息, 甚至连原材料都不放过. 如果这时候, 我们在给一个类中写死这些方法, 就会变得非常复杂. 你不知道用户需要哪些类的信息, 类的数量非常多的话, 代码量非常复杂. 这时候你需要创建一个访问者类, 访问者可以访问所有对象的信息, 你可以给访问者添加新的接口, 以满足不同用户的信息要求.

**优点**:

- 允许你对组合结构加入新的操作, 而无需改变结构本身.
- 想要添加新的操作, 相对容易.
- 访问者所进行操作, 其代码时集中在一起的.

**用途和缺点**:

- 打破组合类的封装.
- 因为游走功能牵涉其中, 组合结构的改变相对困难.
