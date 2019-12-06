# Meanngful Names

命名无处不在, 我们命名变量,函数,参数,类和包等等. 我们可以按照一些规则来更好的完成它.

## Use Intention-Revealing Names

名字的选择应该足够表意的, 可以很容易让人明白你的目的. 选择一个好的名称, 虽然耗时却是值得的.

一个表意的好的变量或者名字应该可以回答以下问题: 为什么它存在, 它干了什么事, 它用来干什么. 如果一个变量需要用注释来描述, 那么他就不是一个表意的命名:

```java
int d; // elapsed time in days

// could be better:
int elapsedTimeDays;
int daysSinceCreation;
int fileAgeInDays;
```

接下来我们看下这段代码:

```java
public List<int[]> getThem() {
  List<int[]> list1 = new ArrayList<int[]>();
  for (int[] x : theList)
  if (x[0] == 4)
    list1.add(x);
  return list1;
}
```

这段代码非常简单, 却是非常难以理解的. 为什么难以理解:

- `theList`代表什么.
- 在`theList`中第一个元素代表什么?
- `4`有什么特殊的含义?
- 返回的`List`有什么作用?

单纯从代码中, 我们没办法回答这些问题. 如果我们换成下面的版本:

```java
public List<int[]> getFlaggedCells() {
  List<int[]> flaggedCells = new ArrayList<int[]>();
  for (int[] cell : gameBoard)
    if (cell[STATUS_VALUE] == FLAGGED)
      flaggedCells.add(cell);
  return flaggedCells;
}
```

这里我们可以有一个清晰的概念,关于这些问题. 我们还可以再优化一下:

```java
public List<Cell> getFlaggedCells() {
  List<Cell> flaggedCells = new ArrayList<Cell>();
  for (Cell cell : gameBoard)
    if (cell.isFlagged())
      flaggedCells.add(cell);
  return flaggedCells;
}
```

只是简单的更换名称, 就可以带来阅读上质的飞越, 这就是选择一个好名称的力量.

## Avoid Disinformation

程序员应该避免使用一些容易造成误解的朦胧的名称. 一些小的建议和例子:

- 尽量不要使用固有的名称, 如`hp`,`aix`等.
- 尽量不要使用`accountList`, 除非它本身就是一个`list`. 不然可能引起误解, 这时候`accounts`和`accountsGroup`是一个好的命名.
- 不要使用很小差别的单词, 如: `XYZControllerForEfficientHandlingOfStrings`和`XYZControllerForEfficientStorageOfStrings`.
- 等等

对于这些命名, 只需要简单的重命名即可.

## Make Meaningful Distinctions

当我们需要对变量函数等进行区分时, 我们应该建立一个有效的区分. 类似`a1, a2, a3,....`这些都是不好的实现. 我们不能为了区分而区分. 如:

```java
public static void copyChars(char a1[], char a2[]) {
  for (int i = 0; i < a1.length; i++) {
    a2[i] = a1[i];
  }
}
```

这时候使用`source`和`destination`会比`a1`和`a2`更好.

另外一个常见出现无效区分的是添加一些无用的单词. 如`Product`, `ProductInfo`和`ProductData`中, 谁可以说明这三者的区别? 其中`Info`, `Data`都是无效的信息. 其它常见的无效信息有: `a`, `an`,  `the`等等. 我们应该尽量避免类似的无效信息.

## Use Pronounceable Names

尽量使用可发音的名称, 这样便于理解和使用. 如`genymdhms`(means: generation date, year, month, day, hour, minute and second). 那为什么不使用`generationTimestamp`.

## Use Searchable Names

使用单个单词或者数字作为变量, 会带来一个严重的问题: 它们不容易定位. 如直接使用数字7, 大部分人都不明白它的含义. 使用`MAX_CLASS_PER_STUDENT`来指代7的话, 可以带来良好的阅读体验.

```java
for (int j = 0; j < 34; j++) {
  s += (t[j]*4)/5;
}

// good version
int realDaysPerIdealDay = 4;
const int WORK_DAYS_PER_WEEK = 5;
int sum = 0;
for (int j = 0; j < NUMBER_OF_TASKS; j++) {
  int realTaskDays = taskEstimate[j] * realDaysPerIdealDay;
  int realTaskWeeks = (realdays / WORK_DAYS_PER_WEEK);
  sum += realTaskWeeks;
}
```

## Avoid Encoding

尽量避免使用编码, 这会带来额外的解码阅读工作.

## Hungarian Notation

在很久以前, 我们命名变量的时候必须使用很少位数的单词或者字母. 如早期的`BASIC`语言只允许一个字母和一个单词. 后来`匈牙利表示法`(在变量前添加类型简称)将命名提升了一个层次.

随着语言的增强, 已经不需要通过这种方式来对类型进行校验. 如`Java`就已经是一个强类型语言, 这时候也就不需要使用`HN`和别的一些辅助命名规则了.

## Member Prefixes

现在不需要, 也不推荐使用`m_`作为前缀命名变量了.

## Interfaces and Implementations

有些特殊的命名规则, 如在接口的实现类前面添加`I`, 如`IShapeFactory`实现类`ShapeFactory`. 现在也推荐, 即使非要添加标识符, 使用`ShapeFactoryImp`或者`CShapeFactory`也是一个更好的选择.

## Avoid Mental Mapping

阅读者不应该对变量进行思维的转换. ru在一个简单的循环中, 我们往往会使用`i,j,k`作为循环变量. 如果循环非常简单, 也没有别的别的额外变量影响的话, 使用时无可厚非的. 但是如果情况非常复杂的话, 那么单纯使用单字母就不够充分表意了, 需要进行转换. 这时候就不推荐了.

## Class Names

类名应该不包含动词. 避免无用的信息名称, 如: `Manager`, `Processor`和`Info`.

## Method Names

方法名称应该包含动词, 类似`postPayment`, `deletePage`等. 动词浅醉可以参照`javabean`的标准, 如`get`, `set`和`is`.

推荐对构造函数进行优化成静态的方法, 如:

```java
Complex fulcrumPoint = Complex.FromRealNumber(23.0);

// 而不是
Complex fulcrumPoint = new Complex(23.0);
```

## Don't Be Cute

Say what you mean. Mean what you say.

## Pick One Word per Concept

对于每一个概念, 坚持使用一个名词. 如`fetch, retrieve和get`都是代表类似的含义, 没有必要使用多个.

## Don't Pun

不要用一个单词干两件事.  如我们需要对一个列表进行操作, 往里面添加一个对象.我们将这个方法命名为`add`. 后面我们想要实现一个功能, 往列表中添加一个元素, 然后返回新的列表. 这时候我们可不可以用`add`来命名, 这里不推荐. 这样会对`add`造成歧义, 不如使用`insert`和`append`.

## Use Solution Domain Names'

我们代码的阅读者将会是程序员. 所以使用`computer science`的专业术语, 算法名称, 数字术语等等. 减少直接使用问题领域的名称. 这样会方便理解和阅读.

## Use Problem Domain Names

如果没有程序员相关的名称, 这时候推荐使用问题领域的名称, 这样方便后续的维护.

## Add Meaningful Context

名称单独放在一起是没有特殊的含义, 如果组合起来会带来特殊的含义. 那么推荐把这些属性放在一个上下文中. 如信息`firstName, lastName, street, houseNumber, city, state`组合在一起就变成了有效的`address`信息.

## Don't Add Gratuitous Context

不要随意添加上下文. 如在一个程序(`Gas Station Deluxe`)中, 为每一个类都添加`GSD`前缀是非常多余的.

## Final Words

选择一个好名字最困难的事情是这个名字需要良好的描述能力和共同的文化背景. 这是一个教学问题, 而不是技术, 业务或管理问题. 遵循其中一些规则, 努力去改善它, 大胆地使用重命名. 它不仅会在短期内得到回报, 从长远来看将继续得到回报.
