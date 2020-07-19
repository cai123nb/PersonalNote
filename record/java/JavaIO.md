# Java IO

![IO](https://image.cjyong.com/IO.png)

Java IO, 在开发中经常用到, 但是却只局限于知道怎么用: 只闻其声, 不见其形. 花费几天的时间, 静下心来好好梳理了一下其内部的结构和使用, 初步了解其形. 在此记录一份, 方便任何查阅和更新.

![Java IO](https://image.cjyong.com/Java%20I%3AO.png)

## Stream

说到IO, 就必须说到流(Stream): 代表任何有能力产出数据的数据源对象或者有能力接受数据的接收端对象. 即在数据源和目的地之间建立一个通道进行数据传输.

### 数据流向划分

衡量标准非常简单, 以应用(或内存)为主, 如果是在内部向外部输出即为输出流, 由外部向内部输入即为输入流. 如对于一个数据库查询流来说: 对于数据库程序来说, 是讲数据从应用内部收集, 然后传输给连接流. 即为输出流. 而对于使用的程序来说, 确是从连接中获取数据, 即为输入流.

- 输出流：从内存输出其他地方（如磁盘，屏幕等）
- 输入流：从其他地方输入到内存。其他地方如（键盘，文件等）

### 传输数据格式划分

如果根据流里面传输的数据格式进行划分, 可以分为两种:

- 字节流（byte，二进制数）: InputStream/OutputStream
- 字符流（char，指代具体含义字符）: Reader/Writer

需要注意的是, 在底层进行传输的时候, 使用的一定是字节流, 即数据一定会被转化为字节进行传输. 而字符根据不同的编码格式可以转化为不同的字节, 两者的数据转换需要单独处理(如转换流).

### 流的功能划分

如果根据流的功能进行划分, 可以简单分为两种:

- 节点流：在不同数据节点中进行数据传输. 如文件流, Socket流, 数组流等等.
- 处理流：对现有的流进行封装和转换等操作. 如:

  - BufferedReader：缓冲流，先将数据读取到一个缓冲数组中，方便后续连续读取.
  - InputStreamReader：将数据从字节流转化为字符流，并进行读取。使用StreamEncoder完成数据编码处理
  - OutputStreamReader: 将数据从字符流转化为字节流，并进行输出。使用StreamDecoder完成数据的解码处理

处理流提供了很多辅助的功能, 如缓冲, 转换(编码和解码等), 可以对底层的节点流进行性能优化或者数据转换等.

## I/O模型

在我们了解IO模型时, 可以先了解一下常见的IO模型分类:

### 同步IO

一个IO模型可以根据同步/异步划分为两种:

- 同步：任务A执行需要依赖任务B的执行，B执行成功之后，A才算完成。一种可靠的任务序列。如电话通知。
- 异步：任务A告诉任务B要干什么，然后任务B去执行，自己继续完成自己的工作直到结束，不可靠的任务序列。如发短信通知。

比如通知别人一件事, 我们可以通过电话直接告诉他人, 这时候, 必须对方接听到电话, 并且听我们讲述完毕才算结束. 这也就是同步模型, 可靠的任务序列. 当然我们也可以通过短信告诉别人, 但是别人是否可以收到(如手机号码是否正确, 手机是否欠费, 是否忽略掉这条短信等)是无法确定的, 是一种不可靠的任务序列, 但是优势在于省时: 发完消息我们就可以干自己的事了.

### 阻塞IO

同样的, IO模型还可以更加阻塞类型划分为以下两种:

- 阻塞：CPU停下来等待一个慢操作执行完成，才可以接下来完成其他事.
- 非阻塞：CPU在这个慢操作的时候，去干别的事，等这个慢操作执行完成再回来继续做慢操作之后的事情（需要切换线程）。

存在疑惑(待完善)!

### Linux的5种IO模型

了解了IO模型基本分类, 我们看看`Linux`操作系统内部支持哪些基本的IO模型. 这里以`recvfrom`函数为例: 从一个套接字上接收信息. 主要的步骤分为两部分:

- 数据从套接字上传输完毕,并存储在内核空间(等待数据).
- 将数据从内核空间拷贝到用户空间.

#### 同步阻塞IO模型

![BIO](https://image.cjyong.com/2017-09-24-23-18-01.png)

默认等待数据返回，然后再将数据拷贝到用户空间，期间线程始终是阻塞的: 第一步和第二步都是阻塞的, 依次完成之后才算完成.

#### 非阻塞IO模型

![NIO](https://image.cjyong.com/2017-09-24-23-19-53.png)

如果当前数据没有准备好，先去干别的事，过一段时间再过来看看数据有没有准备好（定时check），好了就进行数据拷贝. 当数据没有准备好时, 立马返回(第一步不会阻塞). 当数据准备好了, 进行拷贝时, 这一步还是阻塞完成的: 必须等待系统将数据从内核空间拷贝到用户空间完成, 在这段时间线程还是阻塞的(虽然时间相比较短).

#### IO复用模型

![NIO-M](https://image.cjyong.com/2017-09-24-23-21-54.png)

使用select监听一组连接，当其中某一个连接数据准备好了，系统就通知应用拷贝数据到用户空间, 在前面数据准备阶段是非阻塞的, 而后一步拷贝数据依旧是阻塞的. 优化的点在于: 在数据没准备好的时候, 不需要建立线程进行一一对应.

#### 信号驱动IO模型

![NIO-SIGNAL](https://image.cjyong.com/2017-09-24-23-22-38.png)

在系统中注册一个文件准备就绪的信号处理函数，当文件准备就绪时，系统触发处理函数将数据拷贝到用户空间. 注意这里的第二步依旧是阻塞的.

#### 异步IO模型

![AIO](https://image.cjyong.com/2017-09-24-23-23-36.png)

告诉系统我要将文件拷贝到用户空间，然后线程就去干别的事。剩下所有的事交给系统自动完成(包括第一步和第二步). 可以看出这里是真正的异步操作.

## Java IO模型

了解了不同的IO模型, 我们看看`Java`支持哪些IO模型呢. 这里以`Socket`连接为例.

### BIO

BIO(同步阻塞模型)是JDK1.4之前唯一的选择。前面常见的IO操作都是属于这类. 我们看一个常见的`Socket`端示例:

```java
public static void main(String[] args) {
  int port = 4343;
  Thread severThread = new Thread(() -> {
    try {
      ServerSocket serverSocket = new ServerSocket(port);
      while (true) {
        Socket socket = serverSocket.accept();
        // one socket one thread
        Thread handlerThread = new Thread(() -> {
          try {
            // PrintWrite内部使用了 OutputStreamWriter 完成字符向字节流的转换
            PrintWriter printWriter = new PrintWriter(socket.getOutputStream(), true, Charset.forName("UTF-8"));
            printWriter.print("你好 IO");
            // 输出流关闭, 对应的socket也会自动关闭
            printWriter.close();
          } catch (IOException e) {
            e.printStackTrace();
          }
        });
        handlerThread.start();
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  });
  severThread.start();
}
```

客户端连接示例:

```java
public static void main(String[] args) {
  try(Socket socket = new Socket(InetAddress.getLocalHost(), 4343)) {
    // 注意这里使用了 InputStreamReader 进行字节流向字符流的转换
    BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream(), "UTF-8"));
    reader.lines().forEach(str -> System.out.println("来自服务端的消息: " + str));
  } catch (UnknownHostException e) {
    e.printStackTrace();
  } catch (IOException e) {
    e.printStackTrace();
  }
}
```

使用`BIO`非常简单明了, 只需要简单的为每一个请求新建一个线程进行跟踪处理. 比较适合连接数不多的情景.

### NIO

NIO(同步非阻塞模型): 客户端所有的请求都注册到多路复用器中，轮询发现存在数据准备好的请求时，再启动线程进行处理。JDK在1.4引入的基于多路复用的IO模型.

```java
public static void main(String[] args) {
  Thread severThread1 = new Thread(() -> {
    try(Selector selector = Selector.open();
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
      serverSocketChannel.bind(new InetSocketAddress(InetAddress.getLocalHost(), 4343));
      serverSocketChannel.configureBlocking(false);
      serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
      while (true) {
        selector.select();
        Set<SelectionKey> selectionKeys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = selectionKeys.iterator();
        while(iterator.hasNext()) {
          SelectionKey key = iterator.next();
          try (SocketChannel channel = ((ServerSocketChannel) key.channel()).accept()) {
            channel.write(ByteBuffer.wrap("你好 NIO".getBytes("UTF-8")));
          }
          iterator.remove();
        }
      }
    } catch (IOException e) {
      e.printStackTrace();
    }
  });
  severThread1.start();
}
```

客户端不变.

这里可以看出设计的思路非常相近: 将所有的连接交给`Selector`进行管理, 当某一个连接数据准备就绪时, 再进行数据处理.

### AIO

AIO(异步非阻塞):由操作系统完成请求的IO操作之后，通过应用启动线程处理. 于JDK1.7引入, 基于异步IO模型。

```java
public static void main(String[] args) {
  Thread severThread = new Thread(() -> {
    try {
      AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(Executors.newFixedThreadPool(4));
      AsynchronousServerSocketChannel sever = AsynchronousServerSocketChannel.open(group).bind(new InetSocketAddress(InetAddress.getLocalHost(), 4343));
      sever.accept(null, new CompletionHandler<AsynchronousSocketChannel, AsynchronousServerSocketChannel>() {
        @Override
        public void completed(AsynchronousSocketChannel result, AsynchronousServerSocketChannel attachment) {
          sever.accept(null, this);
          try {
            Future<Integer> future = result.write(ByteBuffer.wrap("你好 AIO".getBytes("UTF-8")));
            future.get();
            System.out.println("服务端发送时间：" + LocalDateTime.now());
            result.close();
          } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
          } catch (InterruptedException e) {
            e.printStackTrace();
          } catch (ExecutionException e) {
            e.printStackTrace();
          } catch (IOException e) {
            e.printStackTrace();
          }
        }

        @Override
        public void failed(Throwable exc, AsynchronousServerSocketChannel attachment) {

        }
      });
      group.awaitTermination(20L, TimeUnit.SECONDS);
    } catch (UnknownHostException e) {
      e.printStackTrace();
    } catch (IOException e) {
      e.printStackTrace();
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  });
  severThread.start();
}
```

客户端:

```java
public static void main(String[] args) throws IOException, ExecutionException, InterruptedException {
  AsynchronousSocketChannel client = AsynchronousSocketChannel.open();
  Future<Void> future = client.connect(new InetSocketAddress(InetAddress.getLocalHost(), 4343));
  future.get();
  ByteBuffer byteBuffer = ByteBuffer.allocate(20);
  client.read(byteBuffer, null, new CompletionHandler<Integer, Void>() {
    @Override
    public void completed(Integer result, Void attachment) {
      System.out.println("客户端打印: " + new String(byteBuffer.array()));
    }

    @Override
    public void failed(Throwable exc, Void attachment) {
    }
  });
  Thread.sleep(5_000);
}
```

## 总结

这里只是简单的探讨了一下`JAVA IO`内部的基本原理, 其中还有很多细节需要补充, 如编码问题, 缓存池问题, WebContainer使用等. 后续待更新.

## 参考文档

- [Weixin N.Y IO](https://mp.weixin.qq.com/s/Gr9nhclUPMjrvvcKPRGdLA)
- [Weixin N.Y JAVAIO](https://mp.weixin.qq.com/s/p5qM2UJ1uIWyongfVpRbCg)
- [简述同步IO,异步IO](https://www.cnblogs.com/felixzh/p/10345929.html)
- [IBM-深入分析 Java I/O 的工作机制](https://developer.ibm.com/zh/articles/j-lo-javaio/)
- [IBM-深入分析 Java 中的中文编码问题](https://developer.ibm.com/zh/articles/j-lo-chinesecoding/)
- [IBM-Jetty 的工作原理以及与 Tomcat 的比较](https://developer.ibm.com/zh/articles/j-lo-jetty/)
- [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html)
- [ByteBuffer常用方法详解](https://www.cnblogs.com/JAYIT/p/8384476.html)
