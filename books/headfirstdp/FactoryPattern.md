# 工厂模式

问题的引入, 每当我们使用new来创建对象的时候, 我们都会实例化一个具体的对象. 但是我们都知道绑定具体的类会导致代码更脆弱, 缺乏弹性. 如:

```java
Duck duck = new MallardDuck();
```

我们声明的时候使用接口, 让代码具有弹性, 我们实现的时候却使用的是具体的类, MallardDuck(). 当我们创建一系列的类的时候我们通常会写出如下代码:

```java
Duck duck;

if (picnic) {
	duck = new MallardDuck();
} else if(hunting) {
	duck = new DecoyDuck();
} else if(inBathTub) {
	duck = new RubberDuck();
}
```

这里我们需要根据运行时的条件进行确定要实例化那种类型的对象. 但是如果我们面对这种代码, 一旦发生了变化或者拓展, 就必须重新修改这部分代码. 通常这样修改代码将会造成部分系统更加难于维护和更新, 并且也更加容易犯错.

针对接口编程, 可以隔离掉以后系统可能发生的一大堆改变. 因为如果代码是针对接口而写, 那么通过多态, 它可以与任何新类实现该接口. 但是, 当代码使用大量的具体类时, 等于是自找麻烦, 因为一旦加入新的具体类, 就必须改变代码. 这就意味着, 你的代码并非是"对修改关闭", 每当你想用新的具体类型来拓展代码, 就必须重新打开它.

假如你是一家Pizza店的老板:

```java
Pizza orderPizza() {
	Pizza pizza = new Pizza();
    
    pizza.prepare();
    pizza.bake();
    pizza.cut();
    pizza.box();
    return pizza;
}
```

但是如果你需要更多类型的pizza:

```java
Pizza orderPizza() {
+	Pizza pizza;
+	if (type.equals("cheese")) {
+		pizza = new CheesePizza();
+	} else if (type.equals("pepperoni")) {
+		pizza = new PepperoniPizza();
+	}

    pizza.prepare();
    pizza.bake();
    pizza.cut();
    pizza.box();
    return pizza;
}
```

如果我们决定要新增Pizza的类型, 就必须修改Pizzad的实例化部分, 添加新的代码实现. 这样每次我们都需要修改这个类里面的代码, 非常不适用于代码复用. 于是我们应该将这部分经常变化的代码封装起来, 作为一个`工厂`每当我们要使用的时候, 我们调用工厂里的创建方法就可以了, 每次修改, 我们只要修改工厂中的创建方法即可.

```java
public class SimplePizzaFactory {
	public Pizza createPizza (String type) {
    	Pizza pizza = null;
        if (type.equals("cheese")) {
            pizza = new CheesePizza();
        } else if (type.equals("pepperoni")) {
            pizza = new PepperoniPizza();
        }
        return pizza;
    }
}

public class PizzaStore {
	SimplePizzaFactory factory;
    
    public PizzaStore(SimplePizzaFactory factory) {
    	this.factory = factory;
    }
    
    public Pizza orderPizza (String type) {
+       Pizza pizza = factory.createPizza(type);
        pizza.prepare();
        pizza.bake();
        pizza.cut();
        pizza.box();
        return pizza;
    }
}
```

这里我们没有将工厂设置成静态的, (静态的工厂方法可以不需要使用创建对象的方法来实例化对象, 但是也没办法通过继承来动态创建工厂).
我们的Pizza店大获成功, 有许多地方想要来加盟Pizza店. 但是不同区域的Pizza店可能需要提供不同的风味的Pizza. 这样我们可以将PizzaStore设置成一个抽象类类, 不同区域的加盟店创建自己的Pizzz工厂, 如NYPizzaFactory, ChicagoPizzaFactory, CaliforniaPizzaFactory来生产属于自己的不同口味的Pizza. 最终和加盟店组合起来来获取不同的口味:

```java
NYPizzaFactory nyFactory = new NYPizzaFactory();
PizzaStore nyStore = new PizzaStore(nyFactory);
nyStore.orderPizza("Cheese");
```

但是你想多一点控制, 如果你的Pizza加盟店的确使用的是你的工厂来生产Pizza, 但是他们在其他部分往往不是按照自己的方法进行生产的: 没有切片, 使用其他厂商的和盒子等等. 我们希望可以建立一个弹性的框架让底下的加盟店进行自己独特的生产方式.

```java
public abstract class PizzaStore {
	public final Pizza orderPizza(String type) {
    	Pizza pizza;
        pizza = createPizza(type);
        
        pizza.prepare();
        pizza.bake();
        pizza.cut();
        pizza.box();
        
        return pizza;
    }
    
    abstract Pizza createPizza(String type); //这里我们将创建Pizza的任务交给子类自己来完成
}
```

这样我们对不同的Pizza店铺, 可以继承PizzaStore, 自定义实现createPiza方法, 这就相当于一个PizzaFactory的功能, 这样我们将PizzaFactory的功能封装在createPizza()中, 由子类进行自我实现. 如:

```java
public class NYPizzaStore extends PizzaStore {
	Pizza createPizza(String item) {
    	if (item.equals("cheese")) {
        	return new NYStyleCheesePizza();
        } else if (item.equals("clam")) {
        	return new NYStyleClamPizza();
        } else return null;
    }
}
```

这里最后我们加上Pizza类的说明

```java
public abstract class Pizza {
	String name;
	String dough;
	String sauce;
	ArrayList<String> toppings = new ArrayList<String>();
 
	void prepare() {
		System.out.println("Prepare " + name);
		System.out.println("Tossing dough...");
		System.out.println("Adding sauce...");
		System.out.println("Adding toppings: ");
		for (String topping : toppings) {
			System.out.println("   " + topping);
		}
	}
  
	void bake() {
		System.out.println("Bake for 25 minutes at 350");
	}
 
	void cut() {
		System.out.println("Cut the pizza into diagonal slices");
	}
  
	void box() {
		System.out.println("Place pizza in official PizzaStore box");
	}
 
	public String getName() {
		return name;
	}

	public String toString() {
		StringBuffer display = new StringBuffer();
		display.append("---- " + name + " ----\n");
		display.append(dough + "\n");
		display.append(sauce + "\n");
		for (String topping : toppings) {
			display.append(topping + "\n");
		}
		return display.toString();
	}
}

//具体子类列子
public class NYStyleClamPizza extends Pizza {

	public NYStyleClamPizza() {
		name = "NY Style Clam Pizza";
		dough = "Thin Crust Dough";
		sauce = "Marinara Sauce";
 
		toppings.add("Grated Reggiano Cheese");
		toppings.add("Fresh Clams from Long Island Sound");
	}
}

//测试例子
public class PizzaTestDrive {

	public static void main(String[] args) {
		PizzaStore nyStore = new NYPizzaStore();
		PizzaStore chicagoStore = new ChicagoPizzaStore();
 
		Pizza pizza = nyStore.orderPizza("cheese");
		System.out.println("Ethan ordered a " + pizza.getName() + "\n");

		pizza = chicagoStore.orderPizza("cheese");
		System.out.println("Joel ordered a " + pizza.getName() + "\n");
	}
}
```

工厂方法的定义:
工厂方法模式定义了一个创建对象的接口, 但是由子类决定要实例化的类是哪一个. 工厂方法让类把实例化推迟到子类.

这里引入一个设计原则: 依赖倒置原则(Dependency Inversion Principle)
**要依赖抽象, 不要依赖具体类.**

> 我们在编写代码的时候, 应该尽量依赖那些抽象类, 而不是具体的实类. 就比如我们前面的PizzaStore, 依赖的也仅仅是Pizza这个抽象类, 而具体的实现抽象类是不会管的.

这里有几个指导方针来帮助我们避免在编程中违反依赖倒置的原则:
+ 变量不可以持有具体类的引用.(如果使用了new, 就会持有具体类的引用, 我们可以考虑改用工厂来避免这样的做法)
+ 不要让类派生自具体类. (如果派生自具体的类, 你就会依赖具体的类, 请派生一个抽象(接口或者抽象类)
+ 不要覆盖基类中已实现的方法. (如果覆盖了基类中已实现的方法, 那么说明你的基类就不是一个真正适合被继承的抽象. 基类中已实现的方法, 应该由所有的子类共享)

再次回到Pizza店, 有些加盟店虽然遵循你的流程, 但是却采用一些低价原料来增加利润. 你必须采取一些手段, 以免长此以往会毁掉Pizza点的品牌. 于是你打算创建自己的原料店, 为每家加盟店提供高质量的原料. 首先我们需要建立一个原料工厂来统一的生产这些原料.

```java
public interface PizzaIngredientFactory {
 
	public Dough createDough();
	public Sauce createSauce();
	public Cheese createCheese();
	public Veggies[] createVeggies();
	public Pepperoni createPepperoni();
	public Clams createClam();
 
}
```

为每一个区域建立属于自己的原料生产工厂, 如:

```java
public class NYPizzaIngredientFactory implements PizzaIngredientFactory {
 
	public Dough createDough() {
		return new ThinCrustDough();
	}
 
	public Sauce createSauce() {
		return new MarinaraSauce();
	}
 
	public Cheese createCheese() {
		return new ReggianoCheese();
	}
 
	public Veggies[] createVeggies() {
		Veggies veggies[] = { new Garlic(), new Onion(), new Mushroom(), new RedPepper() };
		return veggies;
	}
 
	public Pepperoni createPepperoni() {
		return new SlicedPepperoni();
	}

	public Clams createClam() {
		return new FreshClams();
	}
}
```

然后我么重写Pizza类:

```java
public abstract class Pizza {
	String name;

	Dough dough;
	Sauce sauce;
	Veggies veggies[];
	Cheese cheese;
	Pepperoni pepperoni;
	Clams clam;

	abstract void prepare();

	void bake() {
		System.out.println("Bake for 25 minutes at 350");
	}

	void cut() {
		System.out.println("Cutting the pizza into diagonal slices");
	}

	void box() {
		System.out.println("Place pizza in official PizzaStore box");
	}

	void setName(String name) {
		this.name = name;
	}

	String getName() {
		return name;
	}

	public String toString() {
		StringBuffer result = new StringBuffer();
		result.append("---- " + name + " ----\n");
		if (dough != null) {
			result.append(dough);
			result.append("\n");
		}
		if (sauce != null) {
			result.append(sauce);
			result.append("\n");
		}
		if (cheese != null) {
			result.append(cheese);
			result.append("\n");
		}
		if (veggies != null) {
			for (int i = 0; i < veggies.length; i++) {
				result.append(veggies[i]);
				if (i < veggies.length-1) {
					result.append(", ");
				}
			}
			result.append("\n");
		}
		if (clam != null) {
			result.append(clam);
			result.append("\n");
		}
		if (pepperoni != null) {
			result.append(pepperoni);
			result.append("\n");
		}
		return result.toString();
	}
}

//Pizza的具体实现子类
public class CheesePizza extends Pizza {
	PizzaIngredientFactory ingredientFactory;
 
	public CheesePizza(PizzaIngredientFactory ingredientFactory) {
		this.ingredientFactory = ingredientFactory;
	}
 
	void prepare() {
		System.out.println("Preparing " + name);
		dough = ingredientFactory.createDough();
		sauce = ingredientFactory.createSauce();
		cheese = ingredientFactory.createCheese();
	}
}
```

我们的加盟店就需要绑定自己的原料工厂:

```java
public class NYPizzaStore extends PizzaStore {
 
	protected Pizza createPizza(String item) {
		Pizza pizza = null;
		PizzaIngredientFactory ingredientFactory = 
			new NYPizzaIngredientFactory();
 
		if (item.equals("cheese")) {
  
			pizza = new CheesePizza(ingredientFactory);
			pizza.setName("New York Style Cheese Pizza");
  
		} else if (item.equals("veggie")) {
 
			pizza = new VeggiePizza(ingredientFactory);
			pizza.setName("New York Style Veggie Pizza");
 
		} else if (item.equals("clam")) {
 
			pizza = new ClamPizza(ingredientFactory);
			pizza.setName("New York Style Clam Pizza");
 
		} else if (item.equals("pepperoni")) {

			pizza = new PepperoniPizza(ingredientFactory);
			pizza.setName("New York Style Pepperoni Pizza");
 
		} 
		return pizza;
	}
}
```

这里我们采用就是抽象工厂模式:
**提供一个接口, 用于创建相关或依赖对象的家族, 而不需要明确指定具体类**

> 抽象工厂允许客户使用抽象的接口来创建一组相关的产品, 而不需要知道实际的产出的具体产品是什么. 这样一来, 客户就从具体的产品中被解耦.

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

**工厂模式**:

定义了一个创建对象的接口, 但是由子类决定要实例化的类是哪一个. 工厂方法让类把实例化推迟到子类中.

**抽象工厂模式**:

提供一个接口用于创建相关或依赖对象的家族, 而不需要明确指定具体类.

要点:

+ 所有的工厂都是用来封装对象的创建.
+ 简单工厂, 虽然不是真正的设计模式, 但仍不失为一个简单的方法, 可以将客户程序从具体类中进行解耦.
+ 工厂方法使用继承, 把对象的创建委托给子类, 子类实现工厂方法来创建对象.
+ 抽象工厂使用对象组合, 对对象的创建被实现在工厂对接口所暴露出来的方法中.
+ 所有的工厂模式都通过减少应用程序和具体类之间的依赖促进松耦合.
+ 工厂方法允许类将实例化延迟到子类进行
+ 依赖倒置原则, 指导我们避免依赖具体类型而是抽象.
+ 工厂是很有威力的技巧, 帮助我们针对抽象编程, 而不是针对具体编程.

