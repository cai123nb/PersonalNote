# 模板方法模式

假设我们有一个咖啡和茶,它们地泡制遵循如下规则:
1. 咖啡
+ 把水煮沸
+ 用沸水冲泡咖啡
+ 把咖啡倒入杯子
+ 添加糖和牛奶
2. 茶
+ 把水煮沸
+ 用沸水浸泡茶叶
+ 把茶倒入杯子
+ 加柠檬

这是我们开始的代码:

```java
//咖啡代码
public class Coffee {
	void prepareRecipe() {
    	boilWater();
        brewCoffeeGrinds();
        pourInCup();
        addSugarAndMilk();
    }
    
    public void boilWater() {
    	System.out.println("Boiling water");
    }
    
    public void brewCoffeeGrinds() {
    	System.out.println("Dripping Coffee through filter");
    }
    
    public void pourInCup() {
    	System.out.println("Puring Into cup);
    }
    
    public void addSugarAndMilk() {
    	System.out.println("Adding Sugar and Milk");
    }
}
//茶代码
public class Tea {
	void prepareRecipe() {
    	boilWater();
        steepTeaBag();
        pourInCup();
        addLemon();
    }
    
    public void boilWater() {
    	System.out.println("Boiling water");
    }
    
    public void steepTeaBag() {
    	System.out.println("Steeping the tea");
    }
    
    public void pourInCup() {
    	System.out.println("Puring Into cup);
    }
    
    public void addLemon() {
    	System.out.println("Adding Lemon");
    }
}
```

但是我们觉得两个类有太多重复的代码, 于是我们开始重新设计. 我们发下第一个和第三个方法是相同的, 于是我们可以简化成:

| CaffeineBeverage | abstract class |
|:-----------------|----------------|
| prepareRecipe()  | 抽象的由子类实现       |
| boilWater()      | 固定方法, 由父类实现    |
| pourInCup()      | 固定方法, 由父类实现    |

| Coffee             | class  |
|:-------------------|--------|
| prepateRecipe()    | 重写该方法  |
| brewCoffeeGrinds() | 添加新的方法 |
| addSugarAndMilk()  | 添加新的方法 |

尽管这样, 我们发现我们还是可以进一步简化: 我们可以把第二步泛化成新brew()方法, 第四个步骤泛化成addCondiments()方法. 这样我们的类就变得更加简单了:

```java
public abstract class CaffeineBeverage {
  
	final void prepareRecipe() {	//我们设置为final, 让子类无法重写该方法
		boilWater();
		brew();
		pourInCup();
		addCondiments();
	}
 
	abstract void brew();	//由子类进行实现
  
	abstract void addCondiments();	//由子类进行实现
 
	void boilWater() {
		System.out.println("Boiling water");
	}
  
	void pourInCup() {
		System.out.println("Pouring into cup");
	}
}

public class Coffee extends CaffeineBeverage {
	public void brew() {
		System.out.println("Dripping Coffee through filter");
	}
	public void addCondiments() {
		System.out.println("Adding Sugar and Milk");
	}
}

public class Tea extends CaffeineBeverage {
	public void brew() {
		System.out.println("Steeping the tea");
	}
	public void addCondiments() {
		System.out.println("Adding Lemon");
	}
}
```

这里我们就是采用的模板方法模式

## 定义

模板方法模式: 在一个方法中定义一个算法的骨架, 而将一些步骤延迟到子类中. 模板算法使得子类可以在不改变算法结构的情况下, 重新定义算法中的某些步骤.

一般模板方法中基本框架结构:

```java
abstract class AbstractClass {
	final void templateMethod() {
    	primitiveOperation1();
        primitiveOperation2();
        concreteOperation();
        hook();
    }
    
    abstract void primitiveOperation1(); //由子类进行自定义实现处理
    abstarct void primitiveOperation2(); //同上
    final void concreteOperation() {
    	//实现...
    }
    void hook() {} //钩子函数, 默认什么事都不做, 如果子类需要自定义某些操作, 可以覆盖该方法
}
```

这里还是以咖啡的例子:

```java
public abstract class CaffeineBeverageWithHook {
 
	final void prepareRecipe() {
		boilWater();
		brew();
		pourInCup();
		if (customerWantsCondiments()) {
			addCondiments();
		}
	}
 
	abstract void brew();
 
	abstract void addCondiments();
 
	void boilWater() {
		System.out.println("Boiling water");
	}
 
	void pourInCup() {
		System.out.println("Pouring into cup");
	}
 
	boolean customerWantsCondiments() {	//这就是钩子函数, 子类可以进行覆盖, 决定是否添加调料品
		return true;
	}
}
//使用
public class CoffeeWithHook extends CaffeineBeverageWithHook {
 
	public void brew() {
		System.out.println("Dripping Coffee through filter");
	}
 
	public void addCondiments() {
		System.out.println("Adding Sugar and Milk");
	}
 
	public boolean customerWantsCondiments() {

		String answer = getUserInput();

		if (answer.toLowerCase().startsWith("y")) {
			return true;
		} else {
			return false;
		}
	}
 
	private String getUserInput() {
		String answer = null;

		System.out.print("Would you like milk and sugar with your coffee (y/n)? ");

		BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
		try {
			answer = in.readLine();
		} catch (IOException ioe) {
			System.err.println("IO error trying to read your answer");
		}
		if (answer == null) {
			return "no";
		}
		return answer;
	}
}
```

这里我们引入一个新的原则:
** 好莱坞原则 ** : 别调用(打电话给) 我们, 我们会调用(打电话给) 你.

> 这种原则可以给我们一种防止"依赖腐败"的方法. 但我们高层组件依赖底层组件, 而底层组件又依赖高层组件, 而高层组件又依赖边侧组件, 而边侧组件又依赖底层组件时, 依赖腐败就产生了. 在这种情况下, 没人可以轻易搞清楚系统是如何设计的.
> 而在这种原则下, 我们允许底层组件将自己挂载到系统上, 但是高层组件会决定什么时候好怎样使用这些底层组件. 换句话说, 高层组件对待底层组件的方式是 好莱坞原则: 别来找我, 我会去找你.

## 总结

OO基础:
+ 抽象
+ 封装
+ 多态
+ 继承

OO原则:
+ 封装变化
+ 多用组合, 少用继承
+ 针对接口编程, 不针对实现编程
+ 为交互对象之间的松耦合设计而努力
+ 对拓展开放, 对修改关闭
+ 依赖抽象, 不要依赖具体的类
+ 只和朋友说话
+ 别找我, 我会找你

模板方法模式: 在一个方法中定义一个算法的骨架, 而将一些步骤延迟到子类中. 模板算法使得子类可以在不改变算法结构的情况下, 重新定义算法中的某些步骤.

要点:
+ `模板方法`定义了算法的步骤, 把这些步骤的实现延迟到子类.
+ 模板方法模式为我们提供了一种代码复用的重要技巧.
+ 模板方法的抽象类可以定义具体的方法, 抽象方法和钩子.
+ 抽象方法由子类实现.
+ 钩子是一种方法, 它在抽象类中不做事, 或者只做默认的事情, 子类可以选择要不要去覆盖它.
+ 为了防止子类改变模板方法中的算法, 可以将模板方法声明为final.
+ 好莱坞原则告诉我们, 将决策权放在高层模块中, 以便决定如何以及何时调用底层模块.
+ 在现实世界代码中看到模板方法的很多变体, 不要期待它们都是一眼就可以被你认出的.
+ 策略模式和模板方法模式都封装算法, 一个使用组合一个使用继承.
+ 工厂方法是模板方法中的一个特殊版本.

