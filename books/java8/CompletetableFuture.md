# 背景

这些年有两种趋势不断推动我们反思我们设计软件的方式. 第一种趋势和应用运行的平台向相关, 第二种趋势与应用程序的架构相关. 随着多核处理器的出现, 提升应用程序处理速度最有效的方式是编写可以充分发挥多核能力的软件, 这一点我们可以通过合理地切分大型任务, [并行处理](https://github.com/cai123nb/PersonalNote/blob/master/Java8InAction/Parallel.md)来实现. 第二种趋势反映在公共 API 日益增长的互联网服务应用. 著名的互联网大鳄们纷纷提供了自己的公共 API 服务, 如谷歌的地理信息服务, Facebook 的社交信息服务等. 很少有网站或者网络应用是以完全隔离的方式进行工作的. 但是如果某些网络服务发生响应慢的情况, 你希望依旧为用户提供部分信息. 如听带有问号标记的通用地图, 以文本的方式显示信息, 而不是呆呆地显示一片空白屏幕, 直到服务器返回结果或者超时. 这就需要引入`并发`, 在同一个 CPU 上执行几个松耦合的任务, 充分利用 CPU 的核, 使其足够忙碌.

## 传统的 Future 接口

`Future`接口在 Java5 中被引入, 它使用一种异步计算的方式, 当遇到复杂且耗时的计算时, 通过新建一个 fork 一个线程进行计算, 主体线程依旧进行其他有价值的计算. 如:

```java
ExecutorService executoer = Executors.newCachedThreadPool();
Future<Double> future = executoer.submit(new Callable<Double>() {
    @Override
    public Double call() throws Exception {
        return doSomeLongComutation();
    }
});
doSomethingElse();

try {
    Double result = future.get(1, TimeUnit.SECONDS);
} catch (ExecutionException ee) {
    //计算中抛出异常
} catch (InterruptedException ie) {
    //当前线程在等待过程中抛出异常
} catch (TimeoutException te) {
    //在Future对象完成之前超出已过期
}
```

局限性: Future 接口提供了方法(isDone)来检测异步计算是否已经结束,等待异步操作结束, 获取计算的结果.但是这些特性依然不够你编写简洁的并发代码. 如: 我们很难表述 Future 结果之间的依赖性; 如果我们需要进行两个异步计算, 且第二个异步计算依赖于第一个的结果; 等待 Future 结合中所有的任务都完成; 仅仅等待 Future 结合中最快结束的任务完成等. 因此我们引入了 CompletableFuture.

## CompletableFuture

CompletableFuture 接口实现了 Future 接口, 但是 CompletableFuture 接口比 Future 具有更多的特性, 来帮助我们完成并发的代码实现. 先用一个小小的例子来展示一下 CompletableFuture 的用法:

```java
package main11;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Future;

/**
 * Created by cjyong on 2017/9/1.
 */
public class Shop {
    private String shopName;

    public Shop(String shopName) {
        this.shopName = shopName;
    }

    public double getPrice(String product){
        return calculatePrice(product);
    }

    public Future<Double> getPriceAsync(String product){
        CompletableFuture<Double> futurePrice = new CompletableFuture<>();
      //异步进行计算
        new Thread( () -> {
            double price = calculatePrice(product);
            futurePrice.complete(price);
        }).start();
      //跳过等待, 直接返回
        return futurePrice;
    }


    private double calculatePrice(String product){
        delay(); //模拟延时1秒
        return Math.random() * product.charAt(0) + product.charAt(1);
    }

    public static void delay() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
//调用
Shop shop = new Shop("BestShop");
long start = System.nanoTime();
Future<Double> futurePrice = shop.getPriceAsync("my favorite product");
long invocationTime = ((System.nanoTime() - start) / 1_000_000);
System.out.println("Invocation returened after " + invocationTime + " msecs");

doSomethingElse();

try {
    double price = futurePrice.get();
    System.out.printf("Price is %.2f%n", price);
} catch (Exception e) {
    throw new RuntimeException(e);
}

long retrievalTime = ((System.nanoTime() - start) / 1_000_000);
System.out.println("Price returned after " + retrievalTime + " msecs");
```

![结果展示](https://image.cjyong.com/blog/bj8_4.jpg)

优化 getPriceAsync 方法

```java
public Future<Double> getPriceAsync(String product){
/*CompletableFuture<Double> futurePrice = new CompletableFuture<>();

    new Thread( () -> {
    //添加异常捕获, 放置异常堵塞
        try{
            double price = calculatePrice(product);
            futurePrice.complete(price);
        } catch (Exception ex){
            futurePrice.completeExceptionally(ex);
        }
    }).start();

    return futurePrice;*/
   //使用supplyAsync()方法, 等价于上面代码.
    return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
```

## 并行流操作

如果你面临需要获取很多商店的信息时, 这时候你可能需要并行操作:

```java
static List<Shop> shops = Arrays.asList(new Shop("BestPrice"),
        new Shop("LetsSaveBig"),
        new Shop("MyFavoriteShop"),
        new Shop("BugItAll"),
        new Shop("BuyBuyBuy"));

public static void main(String... args) {
      //普通流操作
       long start = System.nanoTime();
        System.out.println(findPrices0("myPhone27S"));
        long duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Done in " + duration + " mesecs");
      //平行流操作
        start = System.nanoTime();
        System.out.println(findPrices1("myPhone27S"));
        duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Done in " + duration + " mesecs");
        //CompletableFuture操作
        start = System.nanoTime();
        System.out.println(findPrices2("myPhone27S"));
        duration = (System.nanoTime() - start) / 1_000_000;
        System.out.println("Done in " + duration + " mesecs");
}

public static List<String> findPrices0(String product) {
    return shops.stream()
        .map(shop -> String.format("%s price is %.2f", shop.getShopName(), shop.getPrice(product)))
        .collect(toList());
}

public static List<String> findPrices1(String product){
        return shops.parallelStream()
                .map(shop -> String.format("%s price is %.2f", shop.getShopName(), shop.getPrice(product)))
                .collect(toList());
}

public static List<String> findPrices2(String product) {
    List<CompletableFuture<String>> priceFutures = shops
            .stream()
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> String.format("%s price is %.2f", shop.getShopName(), shop.getPrice(product))
            , executor)) //注意这里添加定制的处理器
            .collect(toList());

    return priceFutures.stream()
            .map(CompletableFuture::join) //利用join函数等待所有的异步完成
            .collect(toList());
}
//为CompletableFuture接口定制一个executor, 以便其更好的工作. 平行流使用线程数是默认处理器的核数量`Runtime.getRuntime().availableProcessors()`, 并不能进行定制.
private static final Executor executor = Executors.newFixedThreadPool(Math.min(shops.size(), 100),
        new ThreadFactory() {
            public Thread newThread(Runnable r){
                Thread t = new Thread(r);
                t.setDaemon(true); //以守护进程的方式启动, 便于程序关闭.
                return t;
            }
        });
```

![结果比较](https://image.cjyong.com/blog/bj8_5.jpg)

### 级联多个异步操作

- 相互依赖的异步操作. 假设我们可以获取所有的商品价格, 但是我们还需要添加一个折扣服务, 为不同的价格计算折扣之后的价钱.

```java
//折扣类
package main11;
import static main11.Util.delay;
import static main11.Util.format;
/**
 * Created by cjyong on 2017/9/1.
 */
public class Discount {
    public enum Code {
        NONE(0), SILVER(5), GOLD(10), PLATINUM(15), DIAMOND(20);

        private final int percentage;

        Code(int percentage) {
            this.percentage = percentage;
        }
    }

    public static String applyDiscount(Quota quota) {
        return quota.getShopName() + " price is " + Discount.apply(quota.getPrice(), quota.getDiscountCode());
    }

    private static double apply(double price, Code code){
        delay();
        return format(price * (100 - code.percentage) / 100);
    }
}
//中间DTO
public class Quota {

    private final String shopName;
    private final double price;
    private final Discount.Code discountCode;

    public Quota(String shopName, double price, Discount.Code code) {
        this.shopName = shopName;
        this.price = price;
        this.discountCode =code;
    }

    public static Quota parse(String s){
        String []  split = s.split(":");
        String shopName = split[0];
        double price = Double.parseDouble(split[1]);
        Discount.Code  discountCode = Discount.Code.valueOf(split[2]);
        return new Quota(shopName, price, discountCode);
    }

    public String getShopName() {
        return shopName;
    }

    public double getPrice() {
        return price;
    }

    public Discount.Code getDiscountCode() {
        return discountCode;
    }
}
//修改shop类中获取价格的函数:
public String getPrice(String product){
    double price = calculatePrice(product);
    Discount.Code code = Discount.Code.values()[random.nextInt(Discount.Code.values().length)];
    return String.format("%s:%.2f:%s", shopName, price, code);
}
//普通获取方法
public static List<String> findPrices0(String product) {
    return shops.stream()
            .map(shop -> shop.getPrice(product))
            .map(Quota::parse)
            .map(Discount::applyDiscount)
            .collect(toList());
}//耗时约为10000 msecs.
//组合异步的获取方法
public static List<String> findPrices1(String product) {
    List<CompletableFuture<String>> priceFutures = shops.stream()
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> shop.getPrice(product), executor
            ))  //异步的方式取得商品价格
            .map(future -> future.thenApply(Quota::parse)) //Quote对象存在时, 对返回值进行转换
            .map(future -> future.thenCompose( //异步的方式获取折扣, 这里使用thenCompose方法级联两个异步操作
                    quota -> CompletableFuture.supplyAsync(
                            () -> Discount.applyDiscount(quota), executor
                    )
            ))
            .collect(toList());
    return priceFutures.stream()
            .map(CompletableFuture::join) //等待流中所有的Future执行完毕, 提取返回值
            .collect(toList());
}// 耗时约为2000 msecs
```

我们可以通过 thenApply 方法对对象进行转换, thenCompose 方法对两个异步操作进行级联.

- 不关联的两个异步操作进行关联:

```java
Future<Double> futurePJriceInUSD =
   CompletableFuture.supplyAsync(() -> shop.getPrice(product))
    .thenCombine(
       CompletableFuture.supplyAsync(
           () -> exchangeService.getRate(Money.EUR, Money.USE)),
            (price, rate) -> price * rate);
```

这里, 通过 thenCombine 方法, 两个异步操作会同步进行, 等待最后都获得结果返回计算值, thenCombine 的第二个参数为 BiFunction, 进行合并处理.

### 阶段性获取结果

假设你获取的商品的时间是不确定的, 如果某一个商品的时间过长, 就会导致获取界面, 一直没有信息. 这时候你可以进行分阶段进行处理, 每获得一个信息, 通知用户一条信息, 最后直到所有信息都齐了. 这就需要响应 CompletableFutrede 的 completion 事件.

```java
//修改延时为随机延时
public static void delay() {
    int delay = 500 + random.nextInt(2000);
    try {
        Thread.sleep(delay);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
//修改获取价格的方法
public static Stream<CompletableFuture<String>> findPricesStream(String product){
    return shops.stream()
            .map(shop -> CompletableFuture.supplyAsync(
                    () -> shop.getPrice(product), executor
            ))  //异步的方式取得商品价格
            .map(future -> future.thenApply(Quota::parse)) //Quote对象存在时, 对返回值进行转换
            .map(future -> future.thenCompose( //异步的方式获取折扣, 这里使用thenCompose方法级联两个异步操作
                    quota -> CompletableFuture.supplyAsync(
                            () -> Discount.applyDiscount(quota), executor
                    )
            ));
}
//调用
long start = System.nanoTime();
CompletableFuture[] futures = findPricesStream("myPhone27s")
        .map(f -> f.thenAccept(s -> System.out.println(s + " (done in " +
                ((System.nanoTime() - start) / 1_000_000) + " msecs)")))
        .toArray(size -> new CompletableFuture[size]);
CompletableFuture.allOf(futures).join(); //利用allOf方法, 进行全部完成的判断. 利用anyOf方法, 我们可以终止运行, 只要一个结果完成了.
System.out.println("All shops hava now responded in " +
        ((System.nanoTime() - start) / 1_000_000) + " msecs");
```

结果截图: ![结果展示](https://image.cjyong.com/blog/bj8_6.jpg)

## 总结

1. 执行比较耗时的操作, 尤其是依赖一个或多个远程服务的操作时, 使用异步操作可以改善程序性能, 加快程序的响应时间.
2. 利用 CompletableFuture 类的特性, 我们可以很好的实现这一目标.
3. 如果有多个异步操作, 我们可以合理的级联成一个.
4. 可以利用 thenAccept 注册一个回调函数, 当程序完成时, 我们合理的进行一些处理.
5. 你可以决定什么时候结束程序, 是全部完成还是单个完成.
