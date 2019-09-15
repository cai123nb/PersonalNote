# 复合模式

模式通常被一起使用, 并被组合在同一个设计解决方案中.复合模式在一个解决方案中结合两个或者多个模式, 以解决一般或重复发生的问题.

让我们重新回到鸭子的设计中, 这一次我们从头开始. 我们首先设计鸭子的接口, 假设鸭子都实现了 Quackable 接口, 不同种类的鸭子有不同的实现方式, 有的呱呱叫, 有的咕咕叫.

```java
public interface Quackable {
   public void quack();
}
```

然后我们设计一些鸭子实例类来实现这些接口:

```java
public class MallardDuck implements Quackable {
   public void quack() {
       System.out.println("Quack");
    }
}

public class RubberDuck implements Quackable {
   public void quack() {
       System.out.println("Squeak");
    }
}
```

这里我们设计一个模拟器, 让我们的鸭子模拟叫:

```java
public class DuckSimulator {
   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      simulator.simulate();
   }

   void simulate() {
      Quackable mallardDuck = new MallardDuck();
      Quackable redheadDuck = new RedheadDuck();
      Quackable duckCall = new DuckCall();
      Quackable rubberDuck = new RubberDuck();

      System.out.println("\nDuck Simulator");

      simulate(mallardDuck);
      simulate(redheadDuck);
      simulate(duckCall);
      simulate(rubberDuck);
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

好, 到现在为止, 一切看起来都显得很正常没有问题. 这时候, 我么发现有一只鹅跑过来了, 这时候该怎么办呢?

```java
public class Goose {
   public void honk() {
       System.out.println("Honk");
    }
}
```

我们发现鹅并不没有实现 Quack 接口, 而是使用自己独特的叫喊方式 honk(), 我们想让模拟器可以使用, 该怎么办呢? 这时候我们可以使用适配者模式.

```java
public class GooseAdapter implements Quackable {
   Goose goose;

   public GooseAdapter(Goose goose) {
      this.goose = goose;
   }

   public void quack() {
      goose.honk();
   }

   public String toString() {
      return "Goose pretending to be a Duck";
   }
}
```

我们给鹅类添加一个适配者类, 用这个适配者来实现 Quack 接口, 这样我们就可以将其放入我们的的模拟器中使用了:

```java
public class DuckSimulator {
   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      simulator.simulate();
   }

   void simulate() {
      Quackable mallardDuck = new MallardDuck();
      Quackable redheadDuck = new RedheadDuck();
      Quackable duckCall = new DuckCall();
      Quackable rubberDuck = new RubberDuck();
      Quackable gooseDuck = new GooseAdapter(new Goose());

      System.out.println("\nDuck Simulator: With Goose Adapter");

      simulate(mallardDuck);
      simulate(redheadDuck);
      simulate(duckCall);
      simulate(rubberDuck);
      simulate(gooseDuck);
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

突然现在又出现了一个新的要求, 有一个位呱呱叫学家希望可以统计叫声的次数. 这可怎么办, 我们又不希望修改原来鸭子的代码. 是的这时候我们就需要使用装饰者模式:

```java
public class QuackCounter implements Quackable {
   Quackable duck;
   static int numberOfQuacks;

   public QuackCounter (Quackable duck) {
      this.duck = duck;
   }

   public void quack() {
      duck.quack();
      numberOfQuacks++;
   }

   public static int getQuacks() {
      return numberOfQuacks;
   }
   public String toString() {
      return duck.toString();
   }
}
```

我们在装饰者类中添加新的静态成员变量, 每当我们使用鸭子对象的时候, 我们只要将鸭子对象用该类进行包裹, 就可以了. 现在看看我们的模拟器代码:

```java
public class DuckSimulator {
   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      simulator.simulate();
   }

   void simulate() {
      Quackable mallardDuck = new QuackCounter(new MallardDuck());
      Quackable redheadDuck = new QuackCounter(new RedheadDuck());
      Quackable duckCall = new QuackCounter(new DuckCall());
      Quackable rubberDuck = new QuackCounter(new RubberDuck());
      Quackable gooseDuck = new GooseAdapter(new Goose());

      System.out.println("\nDuck Simulator: With Decorator");

      simulate(mallardDuck);
      simulate(redheadDuck);
      simulate(duckCall);
      simulate(rubberDuck);
      simulate(gooseDuck);

      System.out.println("The ducks quacked " +
                         QuackCounter.getQuacks() + " times");
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

这里我们又发现了很多新的问题, 这个装饰者类是很好用, 我们可以利用它进行叫声统计, 但是有时候也会出现问题. 就是鸭子类和数量都很多, 很多人都忘记使用它进行包装, 导致叫声统计不准确, 很多叫声都没有记录上去.

那我们为啥不将创建鸭子的部分统一起来, 将创建和装饰的部分封装起来呢. 是的我们可以利用工厂模式进行鸭子的产生.
我们首先定义一下鸭子的工厂接口:

```java
public abstract class AbstractDuckFactory {

   public abstract Quackable createMallardDuck();
   public abstract Quackable createRedheadDuck();
   public abstract Quackable createDuckCall();
   public abstract Quackable createRubberDuck();
}
```

我们然后定义一个普通的鸭子工厂类:

```java
public class DuckFactory extends AbstractDuckFactory {

   public Quackable createMallardDuck() {
      return new MallardDuck();
   }

   public Quackable createRedheadDuck() {
      return new RedheadDuck();
   }

   public Quackable createDuckCall() {
      return new DuckCall();
   }

   public Quackable createRubberDuck() {
      return new RubberDuck();
   }
}
```

其次我们定义可以用来统计数量的鸭子工厂类:

```java
public class CountingDuckFactory extends AbstractDuckFactory {

   public Quackable createMallardDuck() {
      return new QuackCounter(new MallardDuck());
   }

   public Quackable createRedheadDuck() {
      return new QuackCounter(new RedheadDuck());
   }

   public Quackable createDuckCall() {
      return new QuackCounter(new DuckCall());
   }

   public Quackable createRubberDuck() {
      return new QuackCounter(new RubberDuck());
   }
}
```

现在看看我们的鸭子模拟器的实现:

```java
public class DuckSimulator {
   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      AbstractDuckFactory duckFactory = new CountingDuckFactory();

      simulator.simulate(duckFactory);
   }

   void simulate(AbstractDuckFactory duckFactory) {
      Quackable mallardDuck = duckFactory.createMallardDuck();
      Quackable redheadDuck = duckFactory.createRedheadDuck();
      Quackable duckCall = duckFactory.createDuckCall();
      Quackable rubberDuck = duckFactory.createRubberDuck();
      Quackable gooseDuck = new GooseAdapter(new Goose());

      System.out.println("\nDuck Simulator: With Abstract Factory");

      simulate(mallardDuck);
      simulate(redheadDuck);
      simulate(duckCall);
      simulate(rubberDuck);
      simulate(gooseDuck);

      System.out.println("The ducks quacked " +
                         QuackCounter.getQuacks() +
                         " times");
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

好的问题完美解决了, 这时候管理员又有了新的需求, 他希望可以按照家族来管理不同的鸭子, 而不是单个单个的创建和管理鸭子. 这时候我们就需要使用组合模式进行管理一群 Quackable.

```java
public class Flock implements Quackable {
   ArrayList quackers = new ArrayList();

    public void add(Quackable quacker) {
       quackers.add(quacker);
    }

    public void quack() {
       Iterator iterator = quackers.iterator();
        while (iterator.hasNext()) {
           Quackable quacker = (Quackable) iterator.next();
            quacker.quack();
        }
    }
}
```

使用组合之后我们模拟器的代码:

```java
public class DuckSimulator {

   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      AbstractDuckFactory duckFactory = new CountingDuckFactory();

      simulator.simulate(duckFactory);
   }

   void simulate(AbstractDuckFactory duckFactory) {
      Quackable redheadDuck = duckFactory.createRedheadDuck();
      Quackable duckCall = duckFactory.createDuckCall();
      Quackable rubberDuck = duckFactory.createRubberDuck();
      Quackable gooseDuck = new GooseAdapter(new Goose());

      System.out.println("\nDuck Simulator: With Composite - Flocks");

      Flock flockOfDucks = new Flock();

      flockOfDucks.add(redheadDuck);
      flockOfDucks.add(duckCall);
      flockOfDucks.add(rubberDuck);
      flockOfDucks.add(gooseDuck);

      Flock flockOfMallards = new Flock();

      Quackable mallardOne = duckFactory.createMallardDuck();
      Quackable mallardTwo = duckFactory.createMallardDuck();
      Quackable mallardThree = duckFactory.createMallardDuck();
      Quackable mallardFour = duckFactory.createMallardDuck();

      flockOfMallards.add(mallardOne);
      flockOfMallards.add(mallardTwo);
      flockOfMallards.add(mallardThree);
      flockOfMallards.add(mallardFour);

      flockOfDucks.add(flockOfMallards);

      System.out.println("\nDuck Simulator: Whole Flock Simulation");
      simulate(flockOfDucks);

      System.out.println("\nDuck Simulator: Mallard Flock Simulation");
      simulate(flockOfMallards);

      System.out.println("\nThe ducks quacked " +
                         QuackCounter.getQuacks() +
                         " times");
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

组合模式工作的非常顺利, 但是有个别的鸭子专家希望可以实时的观察某一个鸭子的鸣叫情况. 这时候该怎么办呢? 是的, 是时候使用观察者模式了.
我们首先定义一下观察接口:

```java
public interface QuackObserveable {
   public void registerObserver(Observer obsever);
    public void notifyObservers();
}
```

然后我们让所有的 Quackable 接口都实现该接口:

```java
public interface Quackable extends QuackObservable {
   public void quack();
}
```

这里我们使用一个类专门来负责观察者的功能, 就不整合到鸭子的实现中去了:

```java
public class Observable implements QuackObservable {
   ArrayList<Observer> observers = new ArrayList<Observer>();
   QuackObservable duck;

   public Observable(QuackObservable duck) {
      this.duck = duck;
   }

   public void registerObserver(Observer observer) {
      observers.add(observer);
   }

   public void notifyObservers() {
      Iterator<Observer> iterator = observers.iterator();
      while (iterator.hasNext()) {
         Observer observer = iterator.next();
         observer.update(duck);
      }
   }

   public Iterator<Observer> getObservers() {
      return observers.iterator();
   }
}
```

我们修改每个鸭子的代码:

```java
public class MallardDuck implements Quackable {
   Observable observable;

   public MallardDuck() {
      observable = new Observable(this);
   }

   public void quack() {
      System.out.println("Quack");
      notifyObservers();
   }

   public void registerObserver(Observer observer) {
      observable.registerObserver(observer);
   }

   public void notifyObservers() {
      observable.notifyObservers();
   }

   public String toString() {
      return "Mallard Duck";
   }
}
```

最后我们将 Observer 接口进行实现:

```java
public interface Observer {
   public void update(QuackObservable duck);
}
```

这里实现一个简单的观察者类:

```java
public  class Quackologist implements Observer {
   public void update(QuackObservable duck) {
       System.out.println("Quackologist: " + duck + " just quacked.");
    }
}
```

这时候我们在看看我们的额模拟器:

```java
public class DuckSimulator {
   public static void main(String[] args) {
      DuckSimulator simulator = new DuckSimulator();
      AbstractDuckFactory duckFactory = new CountingDuckFactory();

      simulator.simulate(duckFactory);
   }

   void simulate(AbstractDuckFactory duckFactory) {

      Quackable redheadDuck = duckFactory.createRedheadDuck();
      Quackable duckCall = duckFactory.createDuckCall();
      Quackable rubberDuck = duckFactory.createRubberDuck();
      Quackable gooseDuck = new GooseAdapter(new Goose());

      Flock flockOfDucks = new Flock();

      flockOfDucks.add(redheadDuck);
      flockOfDucks.add(duckCall);
      flockOfDucks.add(rubberDuck);
      flockOfDucks.add(gooseDuck);

      Flock flockOfMallards = new Flock();

      Quackable mallardOne = duckFactory.createMallardDuck();
      Quackable mallardTwo = duckFactory.createMallardDuck();
      Quackable mallardThree = duckFactory.createMallardDuck();
      Quackable mallardFour = duckFactory.createMallardDuck();

      flockOfMallards.add(mallardOne);
      flockOfMallards.add(mallardTwo);
      flockOfMallards.add(mallardThree);
      flockOfMallards.add(mallardFour);

      flockOfDucks.add(flockOfMallards);

      System.out.println("\nDuck Simulator: With Observer");

      Quackologist quackologist = new Quackologist();
      flockOfDucks.registerObserver(quackologist);

      simulate(flockOfDucks);

      System.out.println("\nThe ducks quacked " +
                         QuackCounter.getQuacks() +
                         " times");
   }

   void simulate(Quackable duck) {
      duck.quack();
   }
}
```

MVC 模式介绍在此略过.

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

复合模式: 复合模式结合两个或以上的模式, 组成一个解决方案, 解决一再发生的一般性问题.

要点:

- MVC 是复合模式, 结合了观察者模式, 策略模式和组合模式.
- 模型使用观察者模式, 以便观察者更新, 同时保持了两者之间的解耦.
- 控制器是视图的策略, 视图可以使用不同的控制器实现, 得到不同的行为.
- 视图使用组合模式实现用户界面, 用户界面通常组合了嵌套的组件, 像面板, 框架和按钮.
- 这些模式携手合作, 把 MVC 模型的三层解耦, 这样可以保持设计的干净和弹性.
- 适配器模式用来将新的模型适配成已有的视图控制器.
- Model2 是 MVC 在 Web 上的应用.
- 在 Model 2 中, 控制器实现成 Servlet, 而 JSP/HTML 实现视图.
