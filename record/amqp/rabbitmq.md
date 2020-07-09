# RabbitMq入门

![rabbitmq](https://image.cjyong.com/GzNKKPy.png)

`RabbitMq`是现在使用最广泛的开源的消息代理, `RabbitMq`轻巧, 易于部署, 支持多种消息传递协议, 支持分布式部署以满足大规模, 高可用的需求.

## 简要的历史

![rabbitmq-history](https://image.cjyong.com/rabbitmq-history.png)

- 1983年, 孟买的一位工程师提出一个问题: 为什么没有一个软件层面的`总线应用`(总线: 计算机各种功能部件之间传送信息的公共通信干线): 将一些重要的消息从一个程序传递给另一个感兴趣的程序里面. 面对这个问题, 来自`MIT`的`Vivek Ranadivé`设计出一个应用总线: `Teknekron`, 来满足这个消息传递的要求.
- 1985年, 在高盛, `Ranadivé`找到了他的第一位顾客: 金融交易. 在金融交易中, 一个商人需要获取各种各样的消息来完成自己的工作. 使用`Teknekron`实现所有交易消息的通信, 而商人只需要订阅自己感兴趣的消息即可(订阅模型产生了). 这也实现了世界上第一款消息队列: `Teknekron’s The Information Bus`, 简称`TIB`.
- 随着`TIB`的应用的拓展, 人们挖掘出了其各种各样的用途, 尤其是在电信和新闻机构. 于是在`1994年`路透社收购了`Teknekron`.
- 随着`TIB`的蓬勃发展, `IBM`公司发现自家的服务器上运行越来越多的`TIB`程序. 发现商机的他, 于1990年, 开始研发自己的消息队列应用, 并与三年后成功发布`MQSeries`, 并与17后升级为`WebSphere MQ`.
- 与此同时, `TIB`并没有完全消失在路透社, 而是通过重命名成`Rendezvous`和`Teknekron’s re-emergence`于1997年成立独立公司`TIBCO`.
- 同年(1997年), 微软发布自己的消息队列应用: `Microsoft Message Queue (MSMQ)`.
- 由于供应商的封锁和平台的绑定, 这使得消息队列一直都是高成本的产物, 无法被大家广泛使用. 并且由于三家供应商的消息队列实现和API不同, 无法进行很好的交互.
- 为了解决交互的问题, `Java`在2001年提出了`Java Message Service (JMS)`, 提出一套通用的接口服务, 只要每个消息队列实现自己的驱动, 就可以将剩下的交互工作交给`JMS`来完成. 但是这并没有解决接口不一致的情况, 只是提供了一个补丁, 尝试粘合不同的接口实现, 最终现实破灭了.
- 2004年, 摩根大通和`iMatix Corporation`合作开发了`Advanced Message Queuing Protocol (AMQP)`规范: 开发的消息队列的标准, 可以完成绝大多数消息队列的需求. 通过实现该标准, 任何人都可以按照标准与之进行通信.
- 2007年, 基于`AMQP`的开源的`RabbitMQ`发布.

目前`RabbitMq`的最新版本为: 3.8.5\. 基于[AMQP 0-9-1](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf).

## 简单应用: 消息队列

![smaple-one](https://image.cjyong.com/sample-one.webp)

先从一个简单的情景开始: 这里有一个生产者P(向消息队列中发送消息), 后面还有一个消费者C(消费队列里面的消息), 队列里面存储不同的消息.

生产者发送消息:

```java
private final static String QUEUE_NAME = "queue_sample";

// 初始化连接的工厂(工厂可以设置host,username,password等等)
ConnectionFactory factory = new ConnectionFactory();
// 设置host
factory.setHost("localhost");
// 创建一个连接
Connection connection = factory.newConnection();
// 创建一个通信的通道
Channel channel = connection.createChannel();

// 声明注册一个消息队列(幂等的, 即存在就不会重复创建)
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 向消息队列发布信息
channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, "Hello word".getBytes());
```

消费者消费消息:

```java
// 和前面保持一致
private final static String QUEUE_NAME = "queue_sample";
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

// 设置消息的消费回调函数
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
};
// 绑定指定的消息队列
channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
```

## 进阶应用:工作队列

![work queue](https://image.cjyong.com/sample-two.webp)

很多时候, 消费队列面向的场景并不是一对一, 而是多对多的情况. 这里以一个生产者, 两个消费者为例. 实现也非常简单, 只需要开启两个消费者服务即可. `消息队列`默认会按照轮询的方式, 依次将消息发给不同的消费者, 平均来说, 每个消费者会消费相同数量的消息.

消费者实现更新(添加模拟延时处理功能):

```java
// 设置消息消费的回调函数
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");
  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
  }
};

// 绑定消息的消费
boolean autoAck = true;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });

// 模拟消息消费的耗时
private static void doWork(String message) throws InterruptedException {
  for (char ch : message.toCharArray()) {
    if (ch == '.') {
      Thread.sleep(1000);
    }
  }
}
```

### 消费者确认

这里提出一个问题: 如何保证消息被正确的消费掉了呢? 消息是否在发给消费者的中途丢失了? 消息是否在消费者内部正确处理完毕了(如消息处理到一半,服务突然挂了)? 为了处理这些问题, `RabbitMq`提供了消费者消息确认的一种方式. 前面的例子, 都是通过设置`autoAck`为`false`显式关闭了消费者消息确认. 是时候开启消息确认了:

```java
// 设置通道的QoS, 即每次最多处理一条消息
channel.basicQos(1); // accept only one unack-ed message at a time (see below)
// 消息消费的回调函数
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
  String message = new String(delivery.getBody(), "UTF-8");

  System.out.println(" [x] Received '" + message + "'");
  try {
    doWork(message);
  } finally {
    System.out.println(" [x] Done");
    // 通知rabbitmq, 该消息已经被正确消费
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
  }
};
// 开启消费者消息确认
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, deliverCallback, consumerTag -> { });
```

需要注意的是, 当我们开启`消费者消息确认`的时候就需要注意`basicAck`(回复确认)丢失的情况. 这是非常容易发生的. 当发生这种情况的时候, `rabbitmq`会一直保留该条消息, 直到你的客户端关闭之后, 再重新发送该条消息. 如果客户端一直没有关闭, 丢失的情况越来越严重, `rabbitmq`中未确认的消息就会吃掉越来越多的内存. `rabbitmq`提供了一个简单的诊断工具, 查看所有消费但未确认的消息:

```java
rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

### 持久性

前面处理消费者出现问题之后, 如何保证消息不会被丢失: 通过消费者确认. 但是如果`RabbitMq`服务挂掉了, 如何保证消息的不会丢失呢?

`RabbitMq`支持持久化处理, 为队列里面的消息支持持久化, 写入磁盘中.

开启`持久化`处理需要三步:

第一步声明(创建)队列的时候, 设置`durable`为`true`:

```java
boolean durable = true;
channel.queueDeclare("queue_durable", durable, false, false, null);
```

第二步发布消息时, 指定消息支持持久化处理:

```java
channel.basicPublish("", "queue_durable",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

需要注意的是, 这里的持久化并不是强的持久性, 并没有办法保证百分百持久到磁盘: 在`mq`接收到消息到持久化到磁盘之间存在一段时间间隙, 没办法保证消息不会丢失.

### 公平推送

前面讲到, `mq`是按照轮询的方式一次给每一个消费者推送消息, 保证每个消费者都会获取一样数量的消息. 但是往往并不能达到最佳的性能:

- 有些消费者配置比较高, 消息处理比较快, 一直处在饥饿状态.
- 有些消息比较重, 处理比较缓慢, 这时候队列还不断推送消息过来, 导致处理不过来.

对于这种情况, `RabbitMQ`提供了一种思路, 设置每一个每个消费者的最大未确认数(`prefetchCount`)属性, 如果现在某一个消费者的未读消息达到了这个计数, 就不会继续往这个消费者推送消息了. 如设置为1, 如果一个消费者在处理一个很重的消息, 这时候, mq轮完一圈, 准备给他发消息的时候, 发现这个消费者的`未确认消息数`是1, 到达了限制, 就不会继续给这个消费者推送消息了.

## 发布与订阅

![exchange](https://image.cjyong.com/exchanges.png)

在前面的例子都是, 直接将消息传递给消息队列. 但是在`RabbitMq`中, 其实生产者并不会直接将消息传递给`Queue`, 而是通过传递给中间商`Exchange`, 由`Exchange`负责将不同的消息传递给不同的队列. 之前发布的时候, 第一个参数为`Exchange`的名称, 如果传递为空字符串, 则会默认将消息传递给缺省的`Exchange`, 而该`Exchange`会将消息传递给对应的消息队列.

要想查看所有的队列, 可以使用以下命令进行查看:

```shell
rabbitmqctl list_exchanges
```

其中`Exchange`提供了很多默认的类型: `direct`, `topic`, `headers` 和 `fanout`.

### Fanout Exchange

`fanout Exchange`会默认将所有的消息传递给绑定的所有队列. 而我们需要时候却非常简单:

生产者, 声明一个`Exchange`, 并向指定的`Exchange`发送消息(注意生产者没有绑定任何消息队列):

```java
// 声明一个 Exchange, 类型为 fanout
channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

// 向指定的Exchange发送消息
channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
```

消费者, 绑定消息队列, 然后接收消息:

```java
// 声明一个消息队列(防止不存在)
channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
// 生成一个临时的消息队列, 并获取名称
String queueName = channel.queueDeclare().getQueue();
// 将临时消息队列和Exchange进行绑定
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

### Direct Exchange

![direct-exchange](https://image.cjyong.com/direct-exchange.png)

`Direct Exchange`: 更加灵活, 可以更加消息中的`routing key`将消息转发到符合该`key`的消息队列. 而消息队列在和`Direct Exchange`绑定时, 需要指定自己的`binding key`.

生产者, 给`Direct Exchange`发送消息, 然后指定`Routing Key`:

```java
// 发送消息的时候, 指定RoutingKey
channel.basicPublish(EXCHANGE_NAME, RoutingKey, null, message.getBytes());
```

消费者, 需要将队列绑定各自感兴趣的`key`:

```java
// 首先声明(注册)一个direct的EXCHANGE
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
// 其次绑定消息队列时, 设置自己的BindingKey(注, 一个消息队列可以绑定多个KEY)
channel.basicPublish(EXCHANGE_NAME, BindingKey, null, message.getBytes());
```

### Topic Exchange

![topic exchange](https://image.cjyong.com/topic-exchange.png)

`Topic Exchange`比`Direct Exchange`更加自由, 支持匿名和随机的消息订阅. 使用`*`和`#`来代表随机的`binding key`:

- `*`: 代表任意一个随机的子类型. 如: `*.orange`可以匹配: `a.orange, b.orange...`.
- `#`: 则代表零个或者一个的随机子类型. 如: `app.#`可以匹配: `app, app.orange, app.red....`.

使用方式和前面的`direct exchange`完全一致.

## 生产者确认

前面讲到了消费者确认, 可以很大程度上保证消息被消费者正确消费. 但是在生产者的时候, 如何保证消息被正确的放到队列中了呢? 其实那就是生产者确认了. `RabbitMq`提供了三种不同的确认方式:

无论那种确认方式, 都需要显式开启`生产者确认模式`:

```java
Channel channel = connection.createChannel();
channel.confirmSelect();
```

**单个消息确认**: 每次发送消息, 直到消息被确认收到之后才进行下一个信息的发送.

```java
while (thereAreMessagesToPublish()) {
  byte[] body = ...;
  BasicProperties properties = ...;
  channel.basicPublish(exchange, queue, properties, body);
  // 等待5秒消息接收确认, 最长等待5秒
  channel.waitForConfirmsOrDie(5_000);
}
```

**批次消息确认**: 每次发送一批消息之后, 然后等待消息确认返回.

```java
int batchSize = 100;
int outstandingMessageCount = 0;
while (thereAreMessagesToPublish()) {
    byte[] body = ...;
    BasicProperties properties = ...;
    channel.basicPublish(exchange, queue, properties, body);
    outstandingMessageCount++;
    if (outstandingMessageCount == batchSize) {
        ch.waitForConfirmsOrDie(5_000);
        outstandingMessageCount = 0;
    }
}
if (outstandingMessageCount > 0) {
    ch.waitForConfirmsOrDie(5_000);
}
```

**异步消息确认**: 设置消息确认的回调函数, 异步处理确认消息.

```java
Channel channel = connection.createChannel();
channel.confirmSelect();
// 设置异步回调函数
channel.addConfirmListener((sequenceNumber, multiple) -> {
    // code when message is confirmed
}, (sequenceNumber, multiple) -> {
    // code when message is nack-ed
});

// example
ConcurrentNavigableMap<Long, String> outstandingConfirms = new ConcurrentSkipListMap<>();
ConfirmCallback cleanOutstandingConfirms = (sequenceNumber, multiple) -> {
    if (multiple) {
        ConcurrentNavigableMap<Long, String> confirmed = outstandingConfirms.headMap(
          sequenceNumber, true
        );
        confirmed.clear();
    } else {
        outstandingConfirms.remove(sequenceNumber);
    }
};

channel.addConfirmListener(cleanOutstandingConfirms, (sequenceNumber, multiple) -> {
    String body = outstandingConfirms.get(sequenceNumber);
    System.err.format(
      "Message with body %s has been nack-ed. Sequence number: %d, multiple: %b%n",
      body, sequenceNumber, multiple
    );
    cleanOutstandingConfirms.handle(sequenceNumber, multiple);
});
```

总而言之, 强两种方式稳定, 但是影响了吞吐量(发消息的频率), 异步消息处理, 稳定高效, 较为推荐.

## 参考文档

- [RabbitMQ](https://www.rabbitmq.com/)
- [AMQP 0-9-1](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf)
- [RabbitMQ In Action](https://github.com/rabbitinaction/sourcecode)
