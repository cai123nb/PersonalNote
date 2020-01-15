# CleanCode_Comments

请不要养成经常写注释的习惯, 注释并不能掩盖你代码写的差劲. 记住一个原则: 用代码来解释你自己, 而不是使用注释.

```java
// Check to see if the employee is eligible for full benefits 
if ((employee.flags & HOURLY_FLAG) && (employee.age > 65))
```

或者是下面这段呢:

```java
if (employee.isEligibleForFullBenefits())
```

## Good Comments

有些注释是必须的和有益的. 这里列举一些常用的场景, 但是需要时刻注意, 最好的注释就是找到一种方法不写注释.

### Legal Comments

公司的法律信息, 有些公司强制要求在所有的源代码的文件头添加公司的法律信息. 这是可以接受的, 也是合理的. 与此同时`IDE`也会自动帮我们折叠这部分信息, 影响不大. 如

```java
// Copyright (C) 2003,2004,2005 by Object Mentor, Inc. All rights reserved.
// Released under the terms of the GNU General Public License version 2 or later.
```

但是这样的注释不应该过于细节, 更好的方式是大概描述, 并在参考标准中引用具体的文件进行说明.

### Informative Comments

有时候在评论中提供一些额外的信息帮助人们理解, 如:

```java
// Returns an instance of the Responder being tested.
protected abstract Responder responderInstance();

// format matched kk:mm:ss EEE, MMM dd, yyyy
Pattern timeMatcher = Pattern.compile("\\d*:\\d*:\\d* \\w*, \\w* \\d*, \\d*");
```

在这两个例子中, 通过注释可以提供一些额外的信息, 能够有效地帮助阅读者进行信息转换, 便于理解. 不对还有更好的做法: 在第一个例子中, 使用`responderInstance`会更加合适, 而不是通过注释. 第二个例子可以将代码移动到特定的处理类中也会更加合适.

### Explanation of Intent

有时候注释可以有效的阐释作者这样实现的意图, 避免阅读者困惑. 如下面两个例子, 第一个例子中作者想要把他的类在排序中放在首位.

```java
// example 1
public int compareTo(Object o)
{
  if(o instanceof WikiPagePath) {
    WikiPagePath p = (WikiPagePath) o;
    String compressedName = StringUtil.join(names, ""); String compressedArgumentName = StringUtil.join(p.names, "");
    return compressedName.compareTo(compressedArgumentName);
  }
  return 1; // we are greater because we are the right type.
}

// example 2
public void testConcurrentAddWidgets() throws Exception {
  WidgetBuilder widgetBuilder = new WidgetBuilder(new Class[]{BoldWidget.class});
  String text = "'''bold text'''";
  ParentWidget parent = new BoldWidget(new MockWidgetRoot(), "'''bold text'''");
  AtomicBoolean failFlag = new AtomicBoolean(); failFlag.set(false);
  //This is our best attempt to get a race condition
  //by creating large number of threads.
  for (int i = 0; i < 25000; i++) {
    WidgetBuilderThread widgetBuilderThread =
      new WidgetBuilderThread(widgetBuilder, text, parent, failFlag);
    Thread thread = new Thread(widgetBuilderThread);
    thread.start();
  }
  assertEquals(false, failFlag.get());
}
```

### Clarification

有时候注释可以有效的对一些复杂抽象的概念进行解释, 但是更好的方法是修改该变量或者方法. 但是当你使用的是一个标准库或者其它来源的代码时, 你没有办法修改时, 这时候注释就可以有效的帮助你.

```java
public void testCompareTo() throws Exception {
  WikiPagePath a = PathParser.parse("PageA");
  WikiPagePath ab = PathParser.parse("PageA.PageB");
  WikiPagePath b = PathParser.parse("PageB");
  WikiPagePath aa = PathParser.parse("PageA.PageA");
  WikiPagePath bb = PathParser.parse("PageB.PageB");
  WikiPagePath ba = PathParser.parse("PageB.PageA");

  assertTrue(a.compareTo(a) == 0); // a == a
  assertTrue(a.compareTo(b) != 0); // a != b
  assertTrue(ab.compareTo(ab) == 0); // ab == ab
  assertTrue(a.compareTo(b) == -1);  // a < b
  assertTrue(aa.compareTo(ab) == -1); // aa < ab
  assertTrue(ba.compareTo(bb) == -1); // ba < bb
  assertTrue(b.compareTo(a) == 1); // b > a
  assertTrue(ab.compareTo(aa) == 1);  // ab > aa
  assertTrue(bb.compareTo(ba) == 1); // bb > ba
```

需要注意的是这类注释往往很难一直保证正确, 所以在写这类注释时一定要小心, 看是否可以找到更好的解决方案, 是在不行就需要非常小心.

### Warning of Consequences

有时候备注可以提供一种警示功能, 给代码阅读者, 防止其干出一些明显的错事:

```java
// Don't run unless you
// have some time to kill.
public void _testWithReallyBigFile() {
  writeLinesToFile(10000000);
  response.setBody(testFile);
  response.readyToSend(this);
  String responseString = output.toString(); assertSubString("Content-Length: 1000000000", responseString); assertTrue(bytesSent > 1000000000);
}

public static SimpleDateFormat makeStandardHttpDateFormat() {
  //SimpleDateFormat is not thread safe,
  //so we need to create each instance independently.
  SimpleDateFormat df = new SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss z"); df.setTimeZone(TimeZone.getTimeZone("GMT"));
  return df;
}
```

### TODO Comments

有时候基于某种原因没有完成, 希望别人干什么, 给别人一个提醒等等, 我们都可以建立`TODO`备注:

```java
//TODO-MdM these are not needed
// We expect this to go away when we do the checkout model protected VersionInfo makeVersion() throws Exception
{
  return null;
}
```

但是需要注意的是, 不要堆积过多的`TODO`列表, 这不能成为你写差代码的理由. 及时整理和清理`TODO`列表, 是一件非常重要的事.

### Amplification

注释还可以用于放大某些不合理(看上去)的事情的重要性. 如:

```java
String listItemContent = match.group(3).trim();
// the trim is real important. It removes the starting // spaces that could cause the item to be recognized
// as another list.
new ListItemWidget(this, listItemContent, this.level + 1);
return buildList(text.substring(match.end()));
```

### Javadocs in Public APIs

对于一些公共的API的`Javadoc`是非常重要的, 但是要记得时常维护更新, 防止误导他人.

## Bad Comments

大部分的注释都是坏的注释, 是写糟糕代码的接口, 对错误决策的修正, 基本上都是程序员的自说自话.

### Mumbling

不要随意添加注释, 如果你决定添加注视了, 那就花时间好好写一个.

如下面这段注释:

```java
public void loadProperties() {
  try {
    String propertiesPath = propertiesLocation + "/" + PROPERTIES_FILE;
    FileInputStream propertiesStream = new FileInputStream(propertiesPath); loadedProperties.load(propertiesStream);
  }
  catch(IOException e) {
  // No properties files means all defaults are loaded
  }
}
```

这段注释可能有用, 但是感觉作者非常匆忙, 没有花太多时间写这个注释. 这个注释就会成为阅读者的一个谜团. 这个注释要干什么? 这里面是什么含义: 为什么捕获异常就意味着所有的属性文件都被加载了? 是上面进行了异常处理? 加载是在这个语句之前完成了? 了解的唯一方法就是去阅读代码的上下文, 找到这段代码真正调用的地方. 如果一个注释强制你去查看其他模块的代码, 这个注释本身的解释功能就失败了, 并不值得这个注释.

### Redundant Comments

避免重复注释, 如果代码本身就可以解释自己, 那就不要强行添加注释, 你会发现阅读注释的时间比阅读代码更加耗时:

```java
// Utility method that returns when this.closed is true. Throws an exception // if the timeout is reached.
public synchronized void waitForClose(final long timeoutMillis) throws Exception {
  if(!closed) {
    wait(timeoutMillis); 
    if(!closed)
      throw new Exception("MockResponseSender could not be closed");
  }
}


// example 2

public abstract class ContainerBase implements Container, Lifecycle, Pipeline, MBeanRegistration, Serializable {

/**
* The processor delay for this component. 
*/
protected int backgroundProcessorDelay = -1;

/**
* The lifecycle event support for this component. 
*/
protected LifecycleSupport lifecycle = new LifecycleSupport(this);

/**
* The container event listeners for this Container. 
*/
protected ArrayList listeners = new ArrayList();

/**
* The Loader implementation with which this Container is * associated.
*/
protected Loader loader = null;

/**
* The Logger implementation with which this Container is
* associated.
*/
protected Log logger = null;

/**
* Associated logger name.
*/
protected String logName = null;

...
```

### Misleading Comments

有时候程序员在注释中描述的内容往往不够准确, 就会导致阅读者更大的误导. 还是强调那个原则, 要么不写注释, 要么花时间好好写一个注释, 并且保证其的准确性.

### Mandated Comments

有时候强制每一个函数都必须包含`javadoc`注释, 每个变量也必须包含一个注解. 这是非常愚蠢可笑的. 这种强制的做法只能, 让代码变得散乱, 随意, 更容易误导别人:

```java
/** 
*
* @param title The title of the CD
* @param author The author of the CD
* @param tracks The number of tracks on the CD
* @param durationInMinutes The duration of the CD in minutes */
public void addCD(String title, String author, int tracks, int durationInMinutes) {
  CD cd = new CD();
  cd.title = title;
  cd.author = author;
  cd.tracks = tracks;
  cd.duration = duration;
  cdList.add(cd);
}
```

而上面主要完成的事情是`cdList.add(cd);`, 却因为这些注释别人忽视.

### Journal Comments

```java
* Changes (from 11-Oct-2001)
* --------------------------
* 11-Oct-2001 : Re-organised the class and moved it to new package 
*               com.jrefinery.date (DG); 
* 05-Nov-2001 : Added a getDescription() method, and eliminated NotableDate
*               class (DG);
* 12-Nov-2001 : IBD requires setDescription() method, now that NotableDate
*               class is gone (DG); Changed getPreviousDayOfWeek(),
*               getFollowingDayOfWeek() and getNearestDayOfWeek() to correct
*               bugs (DG);
* 05-Dec-2001 : Fixed bug in SpreadsheetDate class (DG);
```

类似这种日记般的版本信息, 应该由源代码控制系统来完成(如`GIT`), 应该完全删除.

### Noise Comment

避免无用的废话注释.

```java
/**
* Default constructor. 
*/
protected AnnualDateRule() { }

/** The day of the month. */
private int dayOfMonth;

**
* Returns the day of the month. *
* @return the day of the month. */
public int getDayOfMonth() { 
  return dayOfMonth;
}

private void startSending() {
  try {
    doSending(); }
  catch(SocketException e) {
    // normal. someone stopped the request.
  }
  catch(Exception e) {
    try {
      response.add(ErrorResponder.makeExceptionString(e));
      response.closeAll(); }
    catch(Exception e1) {
      //Give me a break!
    }
  }
}


/** The name. */ 
private String name;

/** The version. */ 
private String version;

/** The licenceName. */ 
private String licenceName;

/** The version. */ 
private String info;
```

### Don't Use a Comment When You Can Use a Function or a Variable

能用函数或者变量表达就不要用注释:

```java
// does the module from the global list <mod> depend on the
// subsystem we are part of?
if (smodule.getDependSubsystems().contains(subSysMod.getSubSystem()))

ArrayList moduleDependees = smodule.getDependSubsystems(); 
String ourSubSystem = subSysMod.getSubSystem();
if (moduleDependees.contains(ourSubSystem))
```

### Position Markers

有时候程序员喜欢用一些符号标记特定的位置, 如:

```java
// Actions //////////////////////////////////
```

对于这种情况尽量权衡是否有必要, 如果没有必要就应该及时删除. 如果这类标记栏过多的话, 就会淹没在噪音之中, 失去原来的目的.

### Closing Brace Comments

```java
public class wc {
public static void main(String[] args) {
BufferedReader in = new BufferedReader(new InputStreamReader(System.in)); 
  String line;
  int lineCount = 0;
  int charCount = 0;
  int wordCount = 0; 
  try {
    while ((line = in.readLine()) != null) { lineCount++;
      charCount += line.length();
      String words[] = line.split("\\W"); wordCount += words.length;
    } //while
    System.out.println("wordCount = " + wordCount); System.out.println("lineCount = " + lineCount); System.out.println("charCount = " + charCount);
    } // try
    catch (IOException e) { System.err.println("Error:" + e.getMessage());
    } //catch
} // main
```

更多时候, 我们可以通过编写更小的函数来替代这种无意义的行为.

### Attribution and Bylines

```java
/* Added by Rick */
```

这类信息应该交由源代码控制系统来进行管理(如Git), 而不是通过注释.

### Commented-Out Code

```java
InputStreamResponse response = new InputStreamResponse();
response.setBody(formatter.getResultStream(), formatter.getByteCount());
// InputStream resultsStream = formatter.getResultStream();
// StreamReader reader = new StreamReader(resultsStream);
// response.setContent(reader.read(formatter.getByteCount()));

this.bytePos = writeBytes(pngIdBytes, 0);
//hdrPos = bytePos;
writeHeader(); 
writeResolution(); 
//dataPos = bytePos;
if (writeImageData()) {
  writeEnd();
  this.pngBytes = resizeByteArray(this.pngBytes, this.maxPos);
} else {
  this.pngBytes = null;
}
return this.pngBytes;
```

为什么要保留这些注释的代码? 我们可以轻易的通过源代码控制系统获得历史代码记录, 这些遗留的代码只会造成误解和困惑.

### HTML Comments

注释里的HTML标签是非常令人厌烦, 这类工作应该由工具来完成, 而不是程序员在注释中添加HTML标签, 来为难阅读者.

### Nonlocal Information

注释尽量在原有的地方, 非本地的注释, 容易失去更新和维护, 还带了冗余信息, 给人以误解.

```java
/**
* Port on which fitnesse would run. Defaults to <b>8082</b>. 
*
* @param fitnessePort
*/
public void setFitnessePort(int fitnessePort) {
this.fitnessePort = fitnessePort; 
}
```

### Too Much Information

请不要添加额外的历史话题或者不相干的信息, 这会造成极大地信息污染, 让人忽略真正需要描述的内容.

```java
/*
RFC 2045 - Multipurpose Internet Mail Extensions (MIME)
Part One: Format of Internet Message Bodies
section 6.8\. Base64 Content-Transfer-Encoding
The encoding process represents 24-bit groups of input bits as output strings of 4 encoded characters. Proceeding from left to right, a 24-bit input group is formed by concatenating 3 8-bit input groups. These 24 bits are then treated as 4 concatenated 6-bit groups, each of which is translated into a single digit in the base64 alphabet. When encoding a bit stream via the base64 encoding, the bit stream must be presumed to be ordered with the most-significant-bit first. That is, the first bit in the stream will be the high-order bit in the first 8-bit byte, and the eighth bit will be the low-order bit in the first 8-bit byte, and so on.
*/
```

### Inobious Connection

注释和代码应该紧密相连, 避免一些关联不强的信息, 造成困扰:

```java
*
* start with an array that is big enough to hold all the pixels 
* (plus filter bytes), and an extra 200 bytes for header info 
*/
this.pngBytes = new byte[((this.width + 1) * this.height * 3) + 200];
```

### Functions Headers

一个小函数带上一个好的函数名会比一个函数头注释有用的多.

### Javadocs in Nonpublic Code

对于非公开的API, 请不要使用Javadoc, 这种死板的形式犹如八股文一样.
