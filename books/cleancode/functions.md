# CleanCode_Functions

先看一段常见的代码:

```java
// List 3-1
public static String testableHtml(PageData pageData, boolean includeSuiteSetup) throws Exception {
  WikiPage wikiPage = pageData.getWikiPage();
  StringBuffer buffer = new StringBuffer();
  if (pageData.hasAttribute("Test")) {
    if (includeSuiteSetup) {
      WikiPage suiteSetup =
          PageCrawlerImpl.getInheritedPage(SuiteResponder.SUITE_SETUP_NAME, wikiPage);
      if (suiteSetup != null) {
        WikiPagePath pagePath = suiteSetup.getPageCrawler().getFullPath(suiteSetup);
        String pagePathName = PathParser.render(pagePath);
        buffer.append("!include -setup .").append(pagePathName).append("\n");
      }
    }
    WikiPage setup = PageCrawlerImpl.getInheritedPage("SetUp", wikiPage);
    if (setup != null) {
      WikiPagePath setupPath = wikiPage.getPageCrawler().getFullPath(setup);
      String setupPathName = PathParser.render(setupPath);
      buffer.append("!include -setup .").append(setupPathName).append("\n");
    }
  }
  buffer.append(pageData.getContent());
  if (pageData.hasAttribute("Test")) {
    WikiPage teardown = PageCrawlerImpl.getInheritedPage("TearDown", wikiPage);
    if (teardown != null) {
      WikiPagePath tearDownPath = wikiPage.getPageCrawler().getFullPath(teardown);
      String tearDownPathName = PathParser.render(tearDownPath);
      buffer.append("\n").append("!include -teardown .").append(tearDownPathName).append("\n");
    }
    if (includeSuiteSetup) {
      WikiPage suiteTeardown =
          PageCrawlerImpl.getInheritedPage(SuiteResponder.SUITE_TEARDOWN_NAME, wikiPage);
      if (suiteTeardown != null) {
        WikiPagePath pagePath = suiteTeardown.getPageCrawler().getFullPath(suiteTeardown);
        String pagePathName = PathParser.render(pagePath);
        buffer.append("!include -teardown .").append(pagePathName).append("\n");
      }
    }
  }
  pageData.setContent(buffer.toString());
  return pageData.getHtml();
}
```

三分钟之后, 你可以清楚地明白这段代码在干啥吗? 这恐怕很难. 这里包含了太多不同层级的抽象, 各种特殊的属性和判断. 但是如果经过简单的重构:

```java
// List 3-2
public static String renderPageWithSetupsAndTeardowns(PageData pageData, boolean isSuite)
    throws Exception {
  boolean isTestPage = pageData.hasAttribute("Test");
  if (isTestPage) {
    WikiPage testPage = pageData.getWikiPage();
    StringBuffer newPageContent = new StringBuffer();
    includeSetupPages(testPage, newPageContent, isSuite);
    newPageContent.append(pageData.getContent());
    includeTeardownPages(testPage, newPageContent, isSuite);
    pageData.setContent(newPageContent.toString());
  }
  return pageData.getHtml();
}
```

看完这段代码, 虽然对内部实现的细节不太了解. 但是对基本的实现和步骤已经有了一个基本的概念: 插入SetUpPage, 插入内容, 插入TearDownPage. 如果对`JUnit`比较熟悉的话, 应该基本清楚这是一个基于Web的测试框架. 那么如何让我们的函数变得简单明了呢?

## Small

第一条准则就是**小**, 第二条准则就是: **它可以更小.**. 第三条准则就是, **它还可以更小**. 甚至可以比上面示例代码更小:

```java
// List 3-3
public static String renderPageWithSetupsAndTeardowns(PageData pageData, boolean isSuite)
    throws Exception {
  if (isTestPage(pageData))
    includeSetupAndTeardownPages(pageData, isSuite);
  return pageData.getHtml();
}
```

### Blocks and Indenting

这意味着语句中可以包含`if`, `else`, `while`语句, 但是尽量只包含一行, 且这行代码应该是函数调用, 并携带文档般的命名. 这也推荐函数内部不应该包含嵌套结构. 也就是意味着函数内部的缩进不应该超过2个.

## Do One Thing

在`List3-1`的例子中, 函数干了不仅仅一件事: 创建buffer, 获取page信息, 渲染路径等等. 而标准的函数应该只干一件事, 如同`List3-3`. 请记住这句话: **FUNCTIONS SHOULD DO ONE THING. THEY SHOULD DO IT WELL. THEY SHOULD DO IT ONLY.**.

`List3-3`总共有三个步骤:

1. Determining whether the page is a test page.
2. If so, including setups and teardowns.
3. Rendering the page in HTML.

也许你会想这是三件事吧? 为什么这里又算一件事呢? 需要注意的是, 这三步都是在该方法名的同层抽象. 我们可以这样来描述这个方法:

```js
TO RenderPageWithSetupsAndTeardowns, we check to see whether the page is a test page and if so, we include the setups and teardowns. In either case we render the page in HTML.
```

如果一个函数内部只包含相对函数名来说只低一层次的步骤(抽象), 那么我们可以说这个函数只干了一件事. 所以, 写函数其实就是对一个大概念的解构, 对其进行抽象, 然后一层层地进行分解.

这里就可以很清楚的发现`List3-1`包含非常多层次的抽象, 干了不仅一件事. 即使是`List3-2`也是可以优化重构成`List3-3`的. 所以还有一个方法可以判断是否可以判断一个函数干了不仅一件事: 是否可以从函数内部抽取一个新的子方法? 如果是, 那么这个函数就干了不仅一件事.

## One Level of Abstraction per Function

为了保证函数**只做一件事**, 我们需要让函数内部的每一个语句都是同一层次的抽象. 混合不同的抽象层次总是会让人困惑的. 给人一种函数内部所有的细节都是重点的感觉. 更糟糕的是, 根据`破窗理论`, 当函数内部出现了一个不符合抽象层次的细节代码, 会出现越来越多这样的代码.

### Reading Code from Top to Bottom: The Stepdown Rule

我们希望所有的代码阅读起来就跟阅读一段从上到下的叙述一样. 我们希望每一个函数都是跟随着下一个抽象层次的内容, 一层层地细化. 我们称之为`The Step down Rule`.

就是说, 我们希望阅读程序就像阅读一个故事段落, 每个段落都描述当前的层次的抽象, 并且引用了下一个层次的抽象段落. 如:

```js
To include the setups and teardowns, we include setups, then we include the test page con- tent, and then we include the teardowns.

  To include the setups, we include the suite setup if this is a suite, then we include the regular setup.

  To include the suite setup, we search the parent hierarchy for the “SuiteSetUp” page and add an include statement with the path of that page.

  To search the parent. . .
```

事实证明遵从这个规则是非常难的. 但是遵循这个规则来编写代码是非常好的**捷径**. 它可以保证代码写出来向故事段落一样, 并且保证代码简单紧凑且只干一件事.

## Switch Statements

我们不能避免`switch`语句(也包括`if`语句), 但是我们可以让相同`switch`语句只出现一次, 并进行良好的封装, 保证只出现一次.

```java
// List 3-4
public Money calculatePay(Employee e) throws InvalidEmployeeType {
 switch (e.type) {
   case COMMISSIONED:
     return calculateCommissionedPay(e);
   case HOURLY:
     return calculateHourlyPay(e);
   case SALARIED:
     return calculateSalariedPay(e);
   default:
     throw new InvalidEmployeeType(e.type);
 }
}
```

这里存在许多问题:

1. 这是非常大的, 并且每次添加一个新的类型, 都需要修改.
2. 这里干了不仅一件事.
3. 这里违背了单一职责原则(SRP).
4. 违背了开发封闭原则(OCP).

其中更严重的是, 这段代码并不能保证这段逻辑只出现一次. 其它地方也有可能出现这段逻辑, 来进行不同的处理逻辑, 如计算发薪日, 获取发送薪水方式等. 重构的方式也非常简单, 那就是使用`抽象工厂方法`:

```java
public abstract class Employee {
  public abstract boolean isPayday();
  public abstract Money calculatePay();
  public abstract void deliverPay(Money pay);
}

public interface EmployeeFactory {
 public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType;
}


public class EmployeeFactoryImpl implements EmployeeFactory {
 @Override
 public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType {
   switch (r.type) {
     case COMMISSIONED:
       return new CommissionedEmployee(r);
     case HOURLY:
       return new HourlyEmployee(r);
     case SALARIED:
       return new SalariedEmploye(r);
     default:
       throw new InvalidEmployeeType(r.type);
   }
 }
}
```

## Use Descriptive Names

记住`Ward`的名言: `You know you are working on clean code when each routine turns out to be pretty much what you expected.` 取得这场战役一半的胜利来自于为一个简单的函数选择一个尽可能好的名称. 越简单的函数, 越容易选择一个好的名称.

- 不要担心取长的名字. 一个好的长名字比一个神秘的短名称好.
- 不要担心花时间去选一个好名字.
- 选择一致性的名称, 便于理解. 使用相似的名词, 动词和修饰词. 如:`includeSetupAndTeardownPages`, `includeSetupPages`, `includeSuiteSetupPage`, and `includeSetupPage`.

## Function Arguments

对于一个函数来说, 完美的情况应该没有参数, 其次就是一个参数都没有, 再其次就是两个参数. 三个参数要尽可能地避免.超过三个参数地函数需要进行调整重构了, 它们应该避免使用. 为什么尽量避免使用参数?

- 参数相对函数名称来说是一层单独的抽象, 并且让你必须去理解它, 需要花费一定的时间去理解.
- 测试困难, 对于不同参数形成的不同测试是呈现指数爆炸的.
- 将结果携带在参数的情况(传递引用, 修改引用内容)更加糟糕, 我们平常阅读代码时希望, 函数如同一个流, 请求的信息从参数中注入, 结果从返回值出来. 而不是从参数中返回.

### Common Monadic Forms

常见的单参数函数有两种. 一种是提出一个问题, 如`boolean fileExists("MyFile")`. 第二种就是对参数进行额外的操作, 如转换, 处理, 加工, 最后返回. 如`InputStream fileOpen("MyFile")`. 这两种方式都是良好的实践, 需要取一个良好的名称来体现函数的作用.

还有一种比较少见单参数函数: 事件(event). 通过一个交互的函数(事件)来修改系统的状态, 这种情况没有返回值. 如: `void passwordAttemptFailedNtimes(int attempts)`. 在这种情况下, 需要特别小心, 选择一个合适的符合上下文的名称.

这了列举一些错误的单参数函数使用方式, 如`void includeSetupPageInto(StringBuffer pageText)`, 这里应该返回一个结果, 而不是通过参数引用来处理, 即使只是单纯的返回处理后的参数, 也比这种实现要好.

### Flag Arguments

使用`Flag`参数是非常丑陋的, 它强制让函数干不仅一件事. 如`reader(boolean isSuite)`就是一个非常糟糕的实现, 是可以分解为: `readerForSuite`和`readerForSingleTest`.

### Dyadic Functions

一个函数如果包含两个参数比一个参数是更加难以理解. 如`writeField(name)`是非常简单易懂, 而`writeField(output-Stream, name)`确是更加难理解, 即使两个函数的名称都是很好的. 对于两个参数的函数, 我们很容易快速瞄一眼时, 忽略掉其中一个参数, 但是实际上这是不好的, 这非常容易导致bug.

然后有一些情况下, 两个参数的函数确是比较适合的.`Point p = new Point(0, 0)`. 这在上下文和我们的理解中是非常适合的. 这种情况下, 我们的认知中会把`两个参数当做一个有顺序的组合`. 然而在一些别的情况中, 往往就没有这么合适了. 如`assertEquals(expected, actual)`, 估计有很多人都放错过位置.

双参数函数并不是恶魔, 你也肯定会写它. 但是我们应该明白书写双参数会不自觉地携带阅读代价, 我们应该尽可能利用一些机制和技巧优化成单参数函数. 如: 如将方法变成类的一个实例方法, 通过实例来进行调用(`outputStream.writeField(name)`), 或者使用一个类进行封装.

### Triads

三参数函数更是非常难以理解的. 其中原因包括顺序, 语义, 不自觉的忽略等等. 如`assertEquals(message, expected, actual)`, 有多少人曾经默认把`message`作为一个`expected`.因此尽量避免使用三参数函数.

### Argument Objects

当一个函数必须携带两个或者三个以上的参数时, 那么推荐使用对象进行封装. 如:

```java
Circle makeCircle(double x, double y, double radius);

// better version
Circle makeCircle(Point center, double radius);
```

### Argument Lists

当你需要传递一系列的参数时, 考虑使用可变参数:

```java
String.format("%s worked %.2f hours.", name, hours);

// use example
public String format(String format, Object... args)
```

实际上面的参数只有两个. 在使用可变参数时, 也应该尽量不要超过三个.

```java
void monad(Integer... args);
void dyad(String name, Integer... args);
void triad(String name, int count, Integer... args);
```

### Verbs and Keywords

选择一个合适的函数名非常有用, 一个好的函数名应该形成`动词` + `名词`对. 如`write(name)`是非常让人困惑的. 而`writeField(name)`则是可以清楚地告诉你`name`是一个`field`.

合理的使用参数作为关键词, 封装成函数名, 这样可以有效便于我们理解. 如`assertExpectedEqualsActual(expected, actual)`往往比`assertEqual(expected, actual)`更好, 因为前者有效减轻了我们记住参数顺序的压力.

## Have No Side Effects

副作用就是谎言, 当你的函数承诺完成一件事时, 往往它干的比它承诺的还要多, 这就是`副作用`, 这是不受管控的地方, 也是`bug`容易出现的地方.

```java
// List 3-6
public class UserValidator {
  private Cryptographer cryptographer;

  public boolean checkPassword(String userName, String password) {
    User user = UserGateway.findByName(userName);
    if (user != User.NULL) {
      String codedPhrase = user.getPhraseEncodedByPassword();
      String phrase = cryptographer.decrypt(codedPhrase, password);
      if ("Valid Password".equals(phrase)) {
        Session.initialize();
        return true;
      }
    }
    return false;
  }
}
```

这段代码的副作用是什么? `Session.initialize();`. 这个函数承诺校验密码, 却多干了一件事`Session.initialize();`这不是这个函数应该做的. 如果一个光是查看函数名的话, 根本不知道这个函数还干了额外的事, 这个件事就变成了`法外之地`不受监管.

### Output Arguments

参数一般被解释为函数的输入. 但是往往有些参数干了不仅一件事: `appendFooter(s)`. 这里的`s`还干了输出的工作.

```java
public void appendFooter(StringBuffer report)
```

只有通过查看函数的定义, 人们才能明白这个函数的具体实现. 这种事情应该被尽量避免, 如果实在不能避免, 一定要修改状态, 那就修改它自己的状态, 如:

```java
report.appendFooter();
```

## Command Query Separation

函数应该`做事情`或者`回答问题`. 不要同时干. 要不就修改对象内部的一些状态, 要不就获取对象的一些信息. 一些不好的例子, 如:

```java
// if have attribute and set success, return true
// if no attribute exist, return false
public boolean set(String attribute, String value);
```

这个函数就是一个典型的错误, 干了不仅一件事. 使用的时候:`if (set("username", "unclebob"))...`就非常容易造成误解. 这可以非常简单地优化为:

```java
if (attributeExists("username")) {
  setAttribute("username", "unclebob"); ...
}
```

## Prefer Exceptions to Returning Error Codes

使用错误码违背了查询命令分离(`Command Query Separation`), 同时干了两件事. 如`if (deletePage(page) == E_OK)` 这是非常不好的实现. 这不仅会造成理解上的困惑, 还会带来深层次的嵌套:

```java
if (deletePage(page) == E_OK) {
  if (registry.deleteReference(page.name) == E_OK) {
    if (configKeys.deleteKey(page.name.makeKey()) == E_OK) {
      logger.log("page deleted");
    } else {
      logger.log("configKey not deleted");
    }
  } else {
    logger.log("deleteReference from registry failed");
  }
} else {
  logger.log("delete failed");
  return E_ERROR;
}
```

如果使用异常的话, 那将会非常容易实现且容易理解:

```java
try {
  deletePage(page);
  registry.deleteReference(page.name);
  configKeys.deleteKey(page.name.makeKey());
} catch (Exception e) {
  logger.log(e.getMessage());
}
```

### Extract Try/Catch Blocks

`Try/catch`语句如果和业务代码耦合在一起是非常丑陋的, 混合业务逻辑和错误处理逻辑. 所以将内容单独抽取出来是一个良好的选择:

```java
public void delete(Page page) {
  try {
    deletePageAndAllReferences(page);
  } catch (Exception e) {
    logError(e);
  }
}

private void deletePageAndAllReferences(Page page) throws Exception {
  deletePage(page);
  registry.deleteReference(page.name);
  configKeys.deleteKey(page.name.makeKey());
}

private void logError(Exception e) {
  logger.log(e.getMessage());
}
```

这样讲业务逻辑和错误处理单独分开, 非常容易理解.

### Error Handling Is One Thing

函数应该只做一件事, 要不就是逻辑处理, 要不就是错误处理. 就像上面的示例一样.

### The Error.java Dependency Magnet

```java
public enum Error {
  OK,
  INVALID,
  NO_SUCH,
  LOCKED,
  OUT_OF_RESOURCES,
  WAITING_FOR_EVENT;
}
```

使用错误码还有一个显著地问题, 一般错误码是定义成上面这样的枚举. 所有需要进行错误处理的地方都要导入引用. 当一个新的错误码引入时, 就必须对所有引用的地方进行重新编译和部署. 但是如果使用异常的话, 简单的继承自`Exception`类就可以了, 并不需要重新编译和部署.

## Don't Repeat Yourself

避免重复代码同样非常重要. 在`List3-1`中, 我们可以发现有一个算法重复了四次. 这也就意味着, 错误出现的几率翻了四倍, 合理的抽取减少重复代码是非常重要的. 这也是我们程序开发中一直在不断追求的目标.

## Structured Programming

`Edsger Dijkstra`曾经说过每个函数, 函数内的每个块都应该只有一个入口, 一个出口. 这意味着函数中只包含一个`return`语句, 没有`break`, `continue`和`goto`.

但是当一个函数非常的小, 非常的清晰时, 这个准则的作用率就非常的低了. 只有在大型函数中, 这样的准则才能取到良好的作用. 所以尽可能地保持函数的简短, 配合少数的`break`, `return`和`continue`将会更加具备表现力. 另外, `goto`只在大型函数中才有作用, 应该被避免.

## How Do You Write Functions Like This

写代码就像写文章一样, 一开始你有一个想法, 然后你把它记录在纸上, 然后不断地优化它, 直到它可以被人阅读. 初稿可能非常的乱, 非常的杂, 甚至还包含错别字, 但是经过你一遍一遍的打磨, 重构和润色, 最终成为一篇优秀的文章.

当我们写一个函数时, 我们先用一个测试函数保证函数的正常执行, 然后实现初稿. 初稿可能非常的长, 非常的杂, 内部甚至还包含重复的代码. 然后我们对代码进行打磨, 优化直到这个代码变得完美.

## Conclusion

每个系统都是由程序员设计用特定领域语言(`domain-specific`)构建的. 函数是该语言的动词, 而类是名词. 这并不是对可怕的旧观念的回想. 相反, 这是一个更古老的事实. 编程艺术从过去到现在一直是语言设计艺术.

熟练的程序员将系统视为要讲的故事, 而不是要编写的程序. 他们使用自己选择的编程语言的工具来构建更丰富, 更具表现力的语言, 用以讲述这个故事. 该领域特定语言的一部分是功能层次结构, 这些功能描述了该系统中发生的所有操作. 在巧妙的递归操作中, 这些操作被编写为使用他们定义的领域特定语言来讲述自己故事的一小部分.

本章已经很好地介绍了编写函数的机制. 如果您遵循此处的规则, 则您的函数将简短, 命名合理且组织良好. 但是, 永远不要忘记, 您的真正目标是讲述系统的故事, 并且编写的功能需要与清晰, 精确的语言完美地组合在一起, 以帮助您进行讲述.

```java
// Listing 3-7
package fitnesse.html;

import fitnesse.wiki.*;

public class SetupTeardownIncluder {
  private PageData pageData;
  private boolean isSuite;
  private WikiPage testPage;
  private StringBuffer newPageContent;
  private PageCrawler pageCrawler;

  private SetupTeardownIncluder(PageData pageData) {
    this.pageData = pageData;
    testPage = pageData.getWikiPage();
    pageCrawler = testPage.getPageCrawler();
    newPageContent = new StringBuffer();
  }

  public static String render(PageData pageData) throws Exception {
    return render(pageData, false);
  }

  private static String render(PageData pageData, boolean isSuite) throws Exception {
    return new SetupTeardownIncluder(pageData).render(isSuite);
  }

  private String render(boolean isSuite) throws Exception {
    this.isSuite = isSuite;
    if (isTestPage()) {
      includeSetupAndTeardownPages();
    }
    return pageData.getHtml();
  }

  private boolean isTestPage() throws Exception {
    return pageData.hasAttribute("Test");
  }

  private void includeSetupAndTeardownPages() throws Exception {
    includeSetupPages();
    includePageContent();
    includeTeardownPages();
    updatePageContent();
  }

  private void includeSetupPages() throws Exception {
    if (isSuite) {
      includeSuiteSetupPage();
    }
    includeSetupPage();
  }

  private void includeSuiteSetupPage() throws Exception {
    include(SuiteResponder.SUITE_SETUP_NAME, "-setup");
  }

  private void includeSetupPage() throws Exception {
    include("SetUp", "-setup");
  }

  private void includePageContent() throws Exception {
    newPageContent.append(pageData.getContent());
  }

  private void includeTeardownPages() throws Exception {
    includeTeardownPage();
    if (isSuite) {
      includeSuiteTeardownPage();
    }
  }

  private void includeTeardownPage() throws Exception {
    include("TearDown", "-teardown");
  }

  private void includeSuiteTeardownPage() throws Exception {
    include(SuiteResponder.SUITE_TEARDOWN_NAME, "-teardown");
  }

  private void updatePageContent() throws Exception {
    pageData.setContent(newPageContent.toString());
  }

  private void include(String pageName, String arg) throws Exception {
    WikiPage inheritedPage = findInheritedPage(pageName);
    if (inheritedPage != null) {
      String pagePathName = getPathNameForPage(inheritedPage);
      buildIncludeDirective(pagePathName, arg);
    }
  }

  private WikiPage findInheritedPage(String pageName) throws Exception {
    return PageCrawlerImpl.getInheritedPage(pageName, testPage);
  }

  private String getPathNameForPage(WikiPage page) throws Exception {
    WikiPagePath pagePath = pageCrawler.getFullPath(page);
    return PathParser.render(pagePath);
  }

  private void buildIncludeDirective(String pagePathName, String arg) {
    newPageContent
      .append("\n!include ")
      .append(arg)
      .append(" .")
      .append(pagePathName)
      .append("\n");
  }
}
```
