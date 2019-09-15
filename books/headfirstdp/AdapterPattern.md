# 适配器模式和外观模式

## 适配器模式

在现实生活中, 适配器到处都可以看到, 如电流适配器. 适配器扮演着这样一个角色: 将一个接口转换为另一个接口, 以符合客户的希望.

这里以之前的 Duck 类进行举例子, 如果这就是我们的鸭子接口:

```java
public interface Duck {
   public void quack();
   public void fly();
}

//具体实例之一
public class MallardDuck implements Duck {
   public void quack() {
      System.out.println("Quack");
   }

   public void fly() {
      System.out.println("I'm flying");
   }
}
```

这时候我们这里出现了一个新的类火鸡:

```java
public interface Turkey {
   public void gobble();   //火鸡不会呱呱叫, 而是咕咕叫
   public void fly();   //火鸡虽然也可以飞, 但是飞的很短
}

//具体实例之一
public class WildTurkey implements Turkey {
   public void gobble() {
      System.out.println("Gobble gobble");
   }

   public void fly() {
      System.out.println("I'm flying a short distance");
   }
}
```

现在如果我们缺少鸭子对象, 想用一些火鸡对象来冒充, 显而易见, 因为火鸡接口不同, 所以我们不能公然拿来用, 我们这时候就需要写一个适配器:

```java
public class TurkeyAdapter implements Duck {
   Turkey turkey;

   public TurkeyAdapter(Turkey turkey) {   //我们需要传递一个火鸡对象, 并实现鸭子接口
      this.turkey = turkey;
   }

   public void quack() {
      turkey.gobble();
   }

   public void fly() {
      for(int i=0; i < 5; i++) {
         turkey.fly();
      }
   }
}
```

适配器模式解析:

1. 客户通过目标接口调用适配器的方法对适配器发出请求.
2. 适配器使用被适配器接口吧请求转换成被适配者的一个或者多个调用接口.
3. 客户收到调用结果, 但并未察觉这一切是适配器在起作用.

### 适配器模式定义

适配器模式: 将一个类接口, 转换成客户期望的另一个接口. 适配器让原本接口不兼容的类可以合作无间.

适配器模式主要分为两种, "对象"适配器和"类"适配器. "对象"适配器就类似之前的火鸡适配器一样, 通过实现目标的接口(Duck), 然后利用组合的形式引入 Turkey 对象, 在内部调用 turkey 的方法实现适配的功能.
"类"适配器: 则是通过同时继承目标和被适配对象来获得方法, 而不是通过组合的形式获取和实现接口功能. 但是 Java 中暂不实现多重继承.

让我们回到新的话题, 如果我们想组合一套家庭影院, 打我们想看电影时, 我们需要:

1. 打开爆米花
2. 开始爆米花
3. 将灯光调暗
4. 放下屏幕
5. 打开投影机
6. 将投影机的输入切换到 DVD
7. ...

如果我们要一个一个写这些接口, 就会显的很麻烦:

```java
popper.on();
popper.pop();
lights.dim(10);
screen.down();
projector.on();
projector.wideScreenMode();
amp.on();
amp.setDvd(dvd);
amp.setSurroundSound();
amp.setVolume(5);
dvd.on();
dvd.play(movie);
```

当我们看完电影的时候, 我们还需要将所有的一切关闭掉, 怎么办? 难道要反向地把这一切动作再进行一次.

这里我们可以利用外观模式将一切包裹起来:

```java
public class HomeTheaterFacade {
   Amplifier amp;
   Tuner tuner;
   DvdPlayer dvd;
   CdPlayer cd;
   Projector projector;
   TheaterLights lights;
   Screen screen;
   PopcornPopper popper;

   public HomeTheaterFacade(Amplifier amp,
             Tuner tuner,
             DvdPlayer dvd,
             CdPlayer cd,
             Projector projector,
             Screen screen,
             TheaterLights lights,
             PopcornPopper popper) {

      this.amp = amp;
      this.tuner = tuner;
      this.dvd = dvd;
      this.cd = cd;
      this.projector = projector;
      this.screen = screen;
      this.lights = lights;
      this.popper = popper;
   }

   public void watchMovie(String movie) {
      System.out.println("Get ready to watch a movie...");
      popper.on();
      popper.pop();
      lights.dim(10);
      screen.down();
      projector.on();
      projector.wideScreenMode();
      amp.on();
      amp.setDvd(dvd);
      amp.setSurroundSound();
      amp.setVolume(5);
      dvd.on();
      dvd.play(movie);
   }


   public void endMovie() {
      System.out.println("Shutting movie theater down...");
      popper.off();
      lights.on();
      screen.up();
      projector.off();
      amp.off();
      dvd.stop();
      dvd.eject();
      dvd.off();
   }

   public void listenToCd(String cdTitle) {
      System.out.println("Get ready for an audiopile experence...");
      lights.on();
      amp.on();
      amp.setVolume(5);
      amp.setCd(cd);
      amp.setStereoSound();
      cd.on();
      cd.play(cdTitle);
   }

   public void endCd() {
      System.out.println("Shutting down CD...");
      amp.off();
      amp.setCd(cd);
      cd.eject();
      cd.off();
   }

   public void listenToRadio(double frequency) {
      System.out.println("Tuning in the airwaves...");
      tuner.on();
      tuner.setFrequency(frequency);
      amp.on();
      amp.setVolume(5);
      amp.setTuner(tuner);
   }

   public void endRadio() {
      System.out.println("Shutting down the tuner...");
      tuner.off();
      amp.off();
   }
}
```

这时候我们简单地调用 watchMover(), 即可完成观看电影的所有操作, 而结束观看电影也可以正确的关闭所有操作. 通过外观模式, 我们可以将这些底层操作封装起来, 与我们解耦, 我们只是简单的高层次接口进行操控.

## 外观模式

### 外观模式定义

外观模式: 提供了一个统一的接口用来访问子系统中的一群接口. 外观定义了一个高层接口, 让子系统更容易使用.

这里我们采用了一个新的原则:
**最少知识原则**: 只和你的密友谈话.

> 当你设计一个系统, 不要管任何对象, 你都要注意它所交互的类有哪些, 并注意它和这些类是如何交互的. 希望我们在设计中, 不要让太多的类耦合在一起, 免得修改系统中一部分, 会影响到其他部分. 如果许多类相互依赖, 那么这个系统就会变成一个易碎的系统, 它需要花许多成本维护, 也会因为太复杂而不容易被其他人了解.

**如何不要赢得太多的朋友和影响太多的对象**
就任何对象而言, 在该对象的方法内, 我们只应该调用属于以下范围的方法:

1. 该对象本身的方法
2. 被当做方法的参数而传递进来的对象
3. 此方法所创建或实例化的任何对象
4. 对象的任何组件

如:

```java
public float getTemp() {
   Thermometer thermometer = station.getThermometer();
    return thermometer.getTemperature();
}
//这里就没有采用这个原则, 调用了第三方的对象, 增加了新的依赖.

public float getTemp() {
   return station.getTemperature();
}
//这里我们应该在Station站中添加这个方法, 因为Thermometer本身就是Station中的组件, 我们在Station类中添加这个方法, 并不会添加新的依赖.
```

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

适配器模式: 将一个类接口, 转换成客户期望的另一个接口. 适配器让原本接口不兼容的类可以合作无间.

外观模式: 提供了一个统一的接口用来访问子系统中的一群接口. 外观定义了一个高层接口, 让子系统更容易使用.

要点:

1. 当需要使用一个现有的类而器接口不符合你的需要时, 就使用适配器.
2. 当需要简化并统一一个很大的接口或者一群复杂的接口时, 使用外观.
3. 适配器改变接口以符合客户的期望.
4. 外观将客户从一个复杂的子系统中解耦.
5. 实现一个适配器可能需要一番功夫, 也可能不费功夫, 视接口的大小与复杂度而定.
6. 实现一个外观, 需要将子系统组合进外观, 然后将工作委托给子系统进行执行.
7. 适配器模式有两种形式: 对象和类适配器. 类适配器需要多重继承.
8. 你可以为子系统实现一个以上的外观.
9. 适配器将一个对象包装起来以改变其接口, 装饰者将一个对象包装起来以增加新的行为和能力, 而外观将一群对象包装起来以简化接口.
