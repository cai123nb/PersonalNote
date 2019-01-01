# General Programming

本章主要涉及到一些Java语言的一些使用技巧, 如本地变量的管理, 权限控制, 库, 数据类型, 反射和本地方法. 最后会讨论一些优化措施和命名规范.

## Item 57: Minimize the scope of local variables

正如`Item15`所说的: `Minimize the accessibility of classes`. 通过最小化变量的使用范围可以提高代码的可读性, 降低潜在的风险. 主要的原因有两点: 一般最小化变量的使用范围时, 都是`使用时才会声明变量`. 这样可以提高代码的可读性, 如果全部变量提前声明放在一起, 那样你可能就需要记住每一个变量到底是干什么的, 非常容易出错. 而这种情况是不会需要额外的记忆工作了. 并且如果在外部声明变量, 等同于破坏了变量的生存周期: 延长或者提前了. 如果别的代码(不可控的)不小心修改了该变量的值, 那结果肯定是灾难的, 并且非常难发现问题所在.

另外`几乎每一个变量声明的时候都应该进行初始化`, 这样可以保证变量的使用范围. 除了一些异常情况: 如对象初始化可能导致异常, 需要进行手动捕获, 这时候需要在`try`语句前提前声明好, 保证对象的可见性. 同时在循环中也是经常出现问题的:

```java
Iterator<Element> i = c.iterator();
while (i.hasNext()) {
    doSomething(i.next());
}
...
Iterator<Element> i2 = c2.iterator();
while (i.hasNext()) {           //Bug here
    doSomething(i2.next());
}


//Preferred idiom
for (Element e : c) {
    ...// Do something with e
}

for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ...//Do something with e and i
}

for (int i = 0, n = expensiveComputation(); i < n; i++) {
    ...//Do something with i
}
```

保证变量使用范围最小化的最后一个技巧就是保证方法的简洁和专注, 每个方法只完成一件事. 如果发现违背了这个要求, 推荐进行拆分.

## Item 58: Prefer for-each loops to tranditional for loops

正如`Item 57: Minimize the scope of local variables`所说的循环, 推荐了好几种写法. 但是这里强调的是, 如果循环中只关注对象集合本身中的每一个元素, 并不需要其它信息的话, 那么使用`for-each`循环可以带来更好的代码可读性, 减少存在bug的可能. 因为使用传统的for循环会暴露一个index`i`或者`iterator`, 这时候你可以利用这两个值进行其它元素的操作, 如果你仅仅想操作的是一个元素, 那么这就会破坏了对象的使用范围.

```java
for (Element e : elements) {
    ... //Do something with e
}

for (Suit suit : suits) {
    for (Rank rank : ranks) {
        deck.add(suit, rank);
    }
}
```

但是`for-each`使用是存在限制的, 每次获取的对象只有一个, 你没办法进行前后左右关联或者过滤等操作. 如果你需要进行复杂的操作: `过滤`, `元素转换`, `平行迭代`等复杂操作, 这时候还是使用传统的`for`循环更好. 另外要使用`for-each`循环, 需要实现`Iterable<E>`接口. 如果你设计的类是存储一系列的对象, 那么推荐实现该接口.

总而言之, 如果条件允许的话尽量使用`for-each`循环, 而不是`for`循环.

## Item 59: Know and use the libraries