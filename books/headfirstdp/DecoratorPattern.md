# 装饰者模式 Decorator Pattern

装饰者模式: 动态地将责任附加到对象上, 如果想要拓展功能, 装饰者提供了有别于继承的另一种选择.

这里以星巴克咖啡为例:
星巴克是以扩张速度最快而闻名的咖啡连锁店, 原先它们的咖啡设计如下:

| Beverage         | class            |
| :--------------- | ---------------- |
| description      | 描述字段         |
| getDescription() | 获取描述         |
| cost()           | 抽象, 由子类实现 |

子类:

| HouseBlend | class |
| :--------- | ----- |
| cost()     | 实现  |

...

但是新的问题出现了, 如果我们需要在咖啡里面添加各种调料, 如: 豆浆(Soy), 摩卡(Mocha)等等. 这时候星巴克会根据所加入的调料收取不同的费用. 所以订单必须考虑到这些调料的部分. 这是他们第一次尝试:
子类:

| HouseBlendWithSoy | class |
| :---------------- | ----- |
| cost()            | 实现  |

| HouseBlendWithMocha | clas |
| :------------------ | ---- |
| cost()              | 实现 |

| HouseBlendWithSoyAndMocha | class |
| :------------------------ | ----- |
| cost()                    | 实现  |

...

你会发现类爆炸了, Boom! Boom! 而且当你维护的时候, 你会更加爆炸, 如简单修改一个 Soy 的价格, 你会发现全部类都要修改, 如果你添加一个新的调料, 你会发现有很多很多的类需要添加.

**第一种改进方式**: 在父类 Beverage 中添加各种调料的信息, 动态添加控制数量, 这样可以减少类的数量了:

| Beverage         | class    |
| :--------------- | -------- |
| description      | 描述     |
| milk             | 牛奶     |
| soy              | 豆浆     |
| mocha            | 摩卡     |
| getDescription() | 获得描述 |
| cost()           | 价格     |
| hasMilk          | 获取牛奶 |
| setMilk          | 设置牛奶 |
| ...              | ...      |

缺点:

- 调整价钱会改变现有的代码.
- 一旦添加新的调料, 我们必须修改父类中的代码.
- 以后可能出现新的饮料, 某些调料可能不适用, 仍然继承了这些方法.
- 如果顾客需要双倍的摩卡, 就无法实现了.
- 代码非常冗余, 很难修改.

引进新的设计原则:
类应该对拓展开发, 对修改关闭.

> 允许类进行拓展, 在不修改现有代码的时候, 就可搭配新的新的行为.

解决方式: 装饰者模式.

```java
//主体
package com.cjyong.headfirst.decorator.starbucks;

public abstract class Beverage {
    String description =  "Unknow Beverage";

    public String getDescription() {
        return description;
    }

    public abstract double cost();
}
//装饰类, 通过继承实现同一类型
package com.cjyong.headfirst.decorator.starbucks;

public abstract class CondimentDecorator extends Beverage{
    public abstract String getDescription();
}

//具体实现类
package com.cjyong.headfirst.decorator.starbucks;

public class DarkRoast extends Beverage{
    public DarkRoast() {
        description = "DarkRoast";
    }

    public double cost() {
        return 1.53;
    }
}

package com.cjyong.headfirst.decorator.starbucks;

public class Mocha extends CondimentDecorator {
    Beverage beverage;

    public Mocha(Beverage beverage) {
        this.beverage = beverage;
    }

    public String getDescription() {
        return beverage.getDescription() + ", Mocha";
    }

    public double cost() {
        return .20 + beverage.cost();
    }
}

//...
//测试代码
package com.cjyong.headfirst.decorator.starbucks;

public class StarbuzzCoffee {
    public static void main(String[] args) {
        Beverage beverage = new Espresso();
        System.out.println(beverage.getDescription() + " $" + beverage.cost());

        Beverage beverage1 = new DarkRoast();
        beverage1 = new Mocha(beverage1);
        beverage1 = new Mocha(beverage1);
        beverage1 = new Whip(beverage1);
        System.out.println(beverage1.getDescription() + " $" + beverage1.cost());

        Beverage beverage2 = new HouseBlend();
        beverage2 = new Soy(beverage2);
        beverage2 = new Mocha(beverage2);
        beverage2 = new Whip(beverage2);
        System.out.println(beverage2.getDescription() + " $" + beverage2.cost());
    }
}
```

其实在 Java 中, 很多地方也用到了装饰者模式, 如 Java 的 I/O, 我们会发现有许许多多的 Stream 类型的类, 其实本质也就是使用了装饰者模式.如:
FileInputStream <-- BufferedInputSrream <-- LineNumberInputStream.

我们以输入为例. InputStream 就是我们主体, 也是抽象类.
其余的子类, 如 FileInputStream, StringBufferInputStream, ByteArrayInputStream, FilterInputStream 是对其的修饰类.
这里举一个例子:

```java
package com.cjyong.headfirst.decorator.io;

import java.io.FilterInputStream;
import java.io.IOException;
import java.io.InputStream;

public class LowerCaseInputStream extends FilterInputStream {
    public LowerCaseInputStream(InputStream in) {
        super(in);
    }

    public int read() throws IOException {
        int c = super.read();
        return (c == -1 ? c : Character.toLowerCase((char) c));
    }

    public int read(byte[] b, int offset, int len) throws IOException {
        int result = super.read(b, offset, len);
        for (int i = offset; i < offset + result; i++) {
            b[i] = (byte) Character.toLowerCase((char)b[i]);
        }
        return result;
    }
}


//测试
package com.cjyong.headfirst.decorator.io;

import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

public class InputTest {
    public static void main(String[] args) {
        int c;
        try {
            InputStream in = new LowerCaseInputStream(
                    new LowerCaseInputStream(
                            new FileInputStream("test.txt")
                    )
            );
            while ((c = in.read() ) >= 0) {
                System.out.print((char) c);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这里也映射出了一个装饰者模式的缺点: 设计的时候我们必须设计出大量的小雷, 数量实在太多了, 导致我们第一次看到这些 API 的时候就会造成困扰.

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
- 多拓展开放, 对修改关闭

装饰者模式:
动态地将责任附加到对象上, 想要拓展功能, 装饰者提供有别于继承的另一种选择.

要点:

- 继承是属于拓展的形式之一, 但不见得是达到弹性设计的最佳方案.
- 在我们的设计中, 应该允许行为被拓展, 而无需修改现有的代码.
- 组合和委托可用于在运行时动态地添加新的行为.
- 除了继承, 装饰者模式也可以让我们拓展行为.
- 装饰者模式意味着一群装饰者类, 这些类用来包装具体的组件.
- 你可以用无数个装饰者包装成一个组价.(有点像链式写法)
- 装饰者一般对组件的客户透明, 除非客户程序依赖于组件的具体类型.
- 装饰者会导致设计中出现多个小对象, 如果过度使用会让程序变得复杂.
