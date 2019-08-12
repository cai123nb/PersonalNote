# 状态模式

现在Java烤面包机已经落伍了, 现在人们已经把Java放到糖果机这样的真正的装置中去了. 假设糖果机有四种状态:

1. 有25美分
2. 没有25美分
3. 售出糖果
4. 糖果售罄

4种状态, 开始时糖果机处于第二种状态, 每当人投入25美分, 糖果机就会进入第一种状态, 这时候用户可以选择购买糖果或者退钱. 如果用户选择购买糖果, 就进入第三种状态并且弹出糖果. 弹出之后, 如果糖果还有的话, 就会进入到第二种状态, 如果糖果卖完就会进入第4种状态.

**第一版构思**:
我们可以定义好4中状态:

```java
final static int SOLD_OUT = 0;
final static int NO_QUARTER = 1;
final static int HAS_QUARTER = 2;
final static int SOLD = 3;
```

然后我们定义所有的操作行为:

```java
public void insertQuarter(){}	//放入硬币
public void ejectQuarter() {} //退还硬币
public void turnCrank() {}	//转动臂铠, 确定购买糖果
private void dispense() {} //弹出糖果
public void refill(int numGumBalls) {} //填充糖果
```

最后我们进行编码实现:

```java
public class GumballMachine {
 
	final static int SOLD_OUT = 0;
	final static int NO_QUARTER = 1;
	final static int HAS_QUARTER = 2;
	final static int SOLD = 3;
 
	int state = SOLD_OUT;
	int count = 0;
  
	public GumballMachine(int count) {
		this.count = count;
		if (count > 0) {
			state = NO_QUARTER;
		}
	}
  
	public void insertQuarter() {
		if (state == HAS_QUARTER) {
			System.out.println("You can't insert another quarter");
		} else if (state == NO_QUARTER) {
			state = HAS_QUARTER;
			System.out.println("You inserted a quarter");
		} else if (state == SOLD_OUT) {
			System.out.println("You can't insert a quarter, the machine is sold out");
		} else if (state == SOLD) {
        	System.out.println("Please wait, we're already giving you a gumball");
		}
	}

	public void ejectQuarter() {
		if (state == HAS_QUARTER) {
			System.out.println("Quarter returned");
			state = NO_QUARTER;
		} else if (state == NO_QUARTER) {
			System.out.println("You haven't inserted a quarter");
		} else if (state == SOLD) {
			System.out.println("Sorry, you already turned the crank");
		} else if (state == SOLD_OUT) {
        	System.out.println("You can't eject, you haven't inserted a quarter yet");
		}
	}
 
	public void turnCrank() {
		if (state == SOLD) {
			System.out.println("Turning twice doesn't get you another gumball!");
		} else if (state == NO_QUARTER) {
			System.out.println("You turned but there's no quarter");
		} else if (state == SOLD_OUT) {
			System.out.println("You turned, but there are no gumballs");
		} else if (state == HAS_QUARTER) {
			System.out.println("You turned...");
			state = SOLD;
			dispense();
		}
	}
 
	private void dispense() {
		if (state == SOLD) {
			System.out.println("A gumball comes rolling out the slot");
			count = count - 1;
			if (count == 0) {
				System.out.println("Oops, out of gumballs!");
				state = SOLD_OUT;
			} else {
				state = NO_QUARTER;
			}
		} else if (state == NO_QUARTER) {
			System.out.println("You need to pay first");
		} else if (state == SOLD_OUT) {
			System.out.println("No gumball dispensed");
		} else if (state == HAS_QUARTER) {
			System.out.println("No gumball dispensed");
		}
	}
 
	public void refill(int numGumBalls) {
		this.count = numGumBalls;
		state = NO_QUARTER;
	}

	public String toString() {
		StringBuffer result = new StringBuffer();
		result.append("\nMighty Gumball, Inc.");
		result.append("\nJava-enabled Standing Gumball Model #2004\n");
		result.append("Inventory: " + count + " gumball");
		if (count != 1) {
			result.append("s");
		}
		result.append("\nMachine is ");
		if (state == SOLD_OUT) {
			result.append("sold out");
		} else if (state == NO_QUARTER) {
			result.append("waiting for quarter");
		} else if (state == HAS_QUARTER) {
			result.append("waiting for turn of crank");
		} else if (state == SOLD) {
			result.append("delivering a gumball");
		}
		result.append("\n");
		return result.toString();
	}
}
```

这时候糖果机产生了一个新的变化, 为了刺激消费, 每次购买有10%的几率成为一个赢家, 赢家可以获得2颗糖.
这时候我就必须修改上面所有的代码实现, 判断是否为新的状态. 是时候来一次新的设计了:
1. 首先, 我们定义一个State接口, 在这个接口内, 糖果机的每一个动作都有一个对应的方法.
2. 然后为机器中的每一个状态实现状态类. 这些类将负责在对应的状态下进行机器的行为.
3. 最后我们, 要摆脱条件代码, 取而代之的方式是, 将动作委托到状态类中.

我们先定义个接口:

```java
public interface State {
 
	public void insertQuarter();
	public void ejectQuarter();
	public void turnCrank();
	public void dispense();
	
	public void refill();
}
```

然后我们为不同的状态设计类:

```java
//这里举一个例子
public class HasQuarterState implements State {
	GumballMachine gumballMachine;
 
	public HasQuarterState(GumballMachine gumballMachine) {
		this.gumballMachine = gumballMachine;
	}
  
	public void insertQuarter() {
		System.out.println("You can't insert another quarter");
	}
 
	public void ejectQuarter() {
		System.out.println("Quarter returned");
		gumballMachine.setState(gumballMachine.getNoQuarterState());
	}
 
	public void turnCrank() {
		System.out.println("You turned...");
		gumballMachine.setState(gumballMachine.getSoldState());
	}

    public void dispense() {
        System.out.println("No gumball dispensed");
    }
    
    public void refill() { }
 
	public String toString() {
		return "waiting for turn of crank";
	}
}
public class NoQuarterState implements State {
    GumballMachine gumballMachine;
 
    public NoQuarterState(GumballMachine gumballMachine) {
        this.gumballMachine = gumballMachine;
    }
 
	public void insertQuarter() {
		System.out.println("You inserted a quarter");
		gumballMachine.setState(gumballMachine.getHasQuarterState());
	}
 
	public void ejectQuarter() {
		System.out.println("You haven't inserted a quarter");
	}
 
	public void turnCrank() {
		System.out.println("You turned, but there's no quarter");
	 }
 
	public void dispense() {
		System.out.println("You need to pay first");
	} 
	
	public void refill() { }
 
	public String toString() {
		return "waiting for quarter";
	}
}
```

对了, 我们突然想起来了, 我们还有一个胜利者模式没有实现, 那也很简单, 我只要添加一份新的状态:

```java
public class WinnerState implements State {
    GumballMachine gumballMachine;
 
    public WinnerState(GumballMachine gumballMachine) {
        this.gumballMachine = gumballMachine;
    }
 
	public void insertQuarter() {
		System.out.println("Please wait, we're already giving you a Gumball");
	}
 
	public void ejectQuarter() {
		System.out.println("Please wait, we're already giving you a Gumball");
	}
 
	public void turnCrank() {
		System.out.println("Turning again doesn't get you another gumball!");
	}
 
	public void dispense() {
		gumballMachine.releaseBall();
		if (gumballMachine.getCount() == 0) {
			gumballMachine.setState(gumballMachine.getSoldOutState());
		} else {
			gumballMachine.releaseBall();
			System.out.println("YOU'RE A WINNER! You got two gumballs for your quarter");
			if (gumballMachine.getCount() > 0) {
				gumballMachine.setState(gumballMachine.getNoQuarterState());
			} else {
            	System.out.println("Oops, out of gumballs!");
				gumballMachine.setState(gumballMachine.getSoldOutState());
			}
		}
	}
 
	public void refill() { }
	
	public String toString() {
		return "despensing two gumballs for your quarter, because YOU'RE A WINNER!";
	}
}
//....余下代码类似, 略过
```

取而代之之前的静态状态, 我们使用State进行管理:

```java
public class GumballMachine {
 
	State soldOutState;
	State noQuarterState;
	State hasQuarterState;
	State soldState;
 
	State state;
	int count = 0;
 
	public GumballMachine(int numberGumballs) {
		soldOutState = new SoldOutState(this);
		noQuarterState = new NoQuarterState(this);
		hasQuarterState = new HasQuarterState(this);
		soldState = new SoldState(this);

		this.count = numberGumballs;
 		if (numberGumballs > 0) {
			state = noQuarterState;
		} else {
			state = soldOutState;
		}
	}
 
	public void insertQuarter() {
		state.insertQuarter();
	}
 
	public void ejectQuarter() {
		state.ejectQuarter();
	}
 
	public void turnCrank() {
		state.turnCrank();
		state.dispense();
	}
 
	void releaseBall() {
		System.out.println("A gumball comes rolling out the slot...");
		if (count != 0) {
			count = count - 1;
		}
	}
 
	int getCount() {
		return count;
	}
 
	void refill(int count) {
		this.count += count;
		System.out.println("The gumball machine was just refilled; it's new count is: " + this.count);
		state.refill();
	}

	void setState(State state) {
		this.state = state;
	}
    public State getState() {
        return state;
    }

    public State getSoldOutState() {
        return soldOutState;
    }

    public State getNoQuarterState() {
        return noQuarterState;
    }

    public State getHasQuarterState() {
        return hasQuarterState;
    }

    public State getSoldState() {
        return soldState;
    }
 
	public String toString() {
		StringBuffer result = new StringBuffer();
		result.append("\nMighty Gumball, Inc.");
		result.append("\nJava-enabled Standing Gumball Model #2004");
		result.append("\nInventory: " + count + " gumball");
		if (count != 1) {
			result.append("s");
		}
		result.append("\n");
		result.append("Machine is " + state + "\n");
		return result.toString();
	}
}
```

修改GumballMachine类的实现:

```java
	State soldOutState;
	State noQuarterState;
	State hasQuarterState;
	State soldState;
+	State winnerState;
```

修改HasQuarerState:

```java
+ Random randomWinner = new Random(System.currentTimeMillis());

public void turnCrank() {
    System.out.println("You turned...");
    int winner = randomWinner.nextInt(10);
    if ((winner == 0) && (gumballMachine.getCount() > 1)) {
        gumballMachine.setState(gumballMachine.getWinnerState());
    } else {
        gumballMachine.setState(gumballMachine.getSoldState());
    }
}
```

## 定义

状态模式: 允许对象在内部状态改变时改变它的行为, 对象看起来好像修改了它的类.

封装基于状态的行为, 并将行为委托到当前状态.

区别:

+ 策略: 将可以互换的行为封装起来, 然后使用委托的方法决定使用哪一个方法.
+ 模板方法: 由子类决定如何实现算法中的某些步骤.

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
+ 类应该只有一个改变的理由

状态模式: 允许对象在内部状态改变时改变它的行为, 对象看起来好像修改了它的类.

要点:

+ 状态模式允许一个对象基于内部状态而拥有不同的行为.
+ 和程序状态机(PSM)不同, 状态模式使用类代表状态.
+ Context会将行为委托给当前状态对象.
+ 通过将每个状态封装进一个类, 我们把以后需要做的任何改变局部化.
+ 状态模式和策略模式有相同的类图, 但他们的意图不相同.
+ 策略模式通常会用行为或算法来配置Context类.
+ 状态模式允许Context随着状态的改变而改变行为.
+ 状态转换可以由State类或者Context类控制.
+ 使用状态模式通常会导致设计中类的数目大量增加.
+ 状态类可以别多个Context实例共享.
