# 命令模式

我们这里首先先简单地介绍一下餐厅的工作过程:

1. 你, 也就是顾客, 把订单交给女招待
2. 女招待拿到订单, 将其放到订单柜台, 然后喊一声, "订单来了"
3. 快餐厨师根据订单准备餐点

**订单封装了准备餐点的请求.**

订单可以想象成一个用来请求准备餐点的对象, 和一般的对象一样, 订单对象可以被传递. 从女招待传递给订单柜台, 或者从女招待传递给接班的下一个女招待. 订单的接口也很简单, 只包含一个方法, 就是orderUp(). 这个方法, 封装了准备餐点所需要的动作. 订单内有一个到"需要进行准备工作的对象"(也就是厨师)的引用, 这一切都封装起来了, 女招待不需要知道订单上有什么, 也不需要知道是谁来准备餐点.

**女招待的工作就是接受订单, 然后调用订单的orderUp()方法.**

女招待的工作很简单: 接下顾客的订单, 继续帮助下一个顾客, 然后将一定数量的订单放到订单柜台, 并调用orderUp()方法, 让人来准备餐点. 她只需要知道订单有一个orderUp()方法可以调用, 这就够了.

**快餐厨师具备准备餐点的知识.**

快餐厨师是一种对象, 他真正知道如何准备餐点, 一旦女招待调用orderUp()方法, 快餐厨师就接手, 实现创建餐点的所有方法.

通过这种模型, 我们可以将"发出请求的对象"和"接受与执行这些请求的对象"分隔开.

假设我们有个遥控器, 要控制灯的开关应该如何设计呢:

```java
public interface Command { //命令接口
	public void execute();
}

public class LightOnCommand implements Command {
	Light light;
  
	public LightOnCommand(Light light) {
		this.light = light;
	}
 
	public void execute() {
		light.on();
	}
}

public class SimpleRemoteControl {	//简单发送端
	Command slot;
 
	public SimpleRemoteControl() {}
 
	public void setCommand(Command command) {
		slot = command;
	}
 
	public void buttonWasPressed() {
		slot.execute();
	}
}

public class RemoteControlTest {
	public static void main(String[] args) {
		SimpleRemoteControl remote = new SimpleRemoteControl();
		Light light = new Light();
		GarageDoor garageDoor = new GarageDoor();
		LightOnCommand lightOn = new LightOnCommand(light);
		GarageDoorOpenCommand garageOpen = 
		    new GarageDoorOpenCommand(garageDoor);
 
		remote.setCommand(lightOn);
		remote.buttonWasPressed();
		remote.setCommand(garageOpen);
		remote.buttonWasPressed();
    }

}
```

## 定义

命令模式: 将`请求`封装成对象, 以便使用不同的请求, 队列或者日志来参数化其他对象. 命令模式也支持可撤销操作.

让我们回到遥控器, 如果你的遥控器有很多很多个插槽, 不仅要控制灯, 还要控制车库门, 只要是插入插槽里面的卡片, 都能进行控制.
这时候我们需要重新编写远程控制端.

```java
public class RemoteControl {
	Command[] onCommands;
	Command[] offCommands;
 
	public RemoteControl() {	//这里设置7个插槽, 并赋予初始值
		onCommands = new Command[7];
		offCommands = new Command[7];
 
		Command noCommand = new NoCommand();
		for (int i = 0; i < 7; i++) {
			onCommands[i] = noCommand;
			offCommands[i] = noCommand;
		}
	}
  
	public void setCommand(int slot, Command onCommand, Command offCommand) {
    //每次插入即可调用该函数
		onCommands[slot] = onCommand;
		offCommands[slot] = offCommand;
	}
 
	public void onButtonWasPushed(int slot) {
		onCommands[slot].execute();
	}
 
	public void offButtonWasPushed(int slot) {
		offCommands[slot].execute();
	}
  
	public String toString() {
		StringBuffer stringBuff = new StringBuffer();
		stringBuff.append("\n------ Remote Control -------\n");
		for (int i = 0; i < onCommands.length; i++) {
			stringBuff.append("[slot " + i + "] " + onCommands[i].getClass().getName()
				+ "    " + offCommands[i].getClass().getName() + "\n");
		}
		return stringBuff.toString();
	}
}
```

而命令接口不需要改变, 只需要为每个操作创建两个不同的命令接口. 测试代码

```java
public class RemoteLoader {
 
	public static void main(String[] args) {
		RemoteControl remoteControl = new RemoteControl();
 
		Light livingRoomLight = new Light("Living Room");
		Light kitchenLight = new Light("Kitchen");
		CeilingFan ceilingFan= new CeilingFan("Living Room");
		GarageDoor garageDoor = new GarageDoor("");
		Stereo stereo = new Stereo("Living Room");
  
		LightOnCommand livingRoomLightOn = 
				new LightOnCommand(livingRoomLight);
		LightOffCommand livingRoomLightOff = 
				new LightOffCommand(livingRoomLight);
		LightOnCommand kitchenLightOn = 
				new LightOnCommand(kitchenLight);
		LightOffCommand kitchenLightOff = 
				new LightOffCommand(kitchenLight);
  
		CeilingFanOnCommand ceilingFanOn = 
				new CeilingFanOnCommand(ceilingFan);
		CeilingFanOffCommand ceilingFanOff = 
				new CeilingFanOffCommand(ceilingFan);
 
		GarageDoorUpCommand garageDoorUp =
				new GarageDoorUpCommand(garageDoor);
		GarageDoorDownCommand garageDoorDown =
				new GarageDoorDownCommand(garageDoor);
 
		StereoOnWithCDCommand stereoOnWithCD =
				new StereoOnWithCDCommand(stereo);
		StereoOffCommand  stereoOff =
				new StereoOffCommand(stereo);
 
		remoteControl.setCommand(0, livingRoomLightOn, livingRoomLightOff);
		remoteControl.setCommand(1, kitchenLightOn, kitchenLightOff);
		remoteControl.setCommand(2, ceilingFanOn, ceilingFanOff);
		remoteControl.setCommand(3, stereoOnWithCD, stereoOff);
  
		System.out.println(remoteControl);
 
		remoteControl.onButtonWasPushed(0);
		remoteControl.offButtonWasPushed(0);
		remoteControl.onButtonWasPushed(1);
		remoteControl.offButtonWasPushed(1);
		remoteControl.onButtonWasPushed(2);
		remoteControl.offButtonWasPushed(2);
		remoteControl.onButtonWasPushed(3);
		remoteControl.offButtonWasPushed(3);
	}
}
```

但是如果我们需要添加撤销的按钮, 我们该怎么做:
那我们的command接口, 就需要添加撤销的函数, 并由实现类进行实现.

```java
public interface Command {
	public void execute();
	public void undo();
}

//具体例子
public class LightOnCommand implements Command {
	Light light;
	int level;
	public LightOnCommand(Light light) {
		this.light = light;
	}
 
	public void execute() {
        level = light.getLevel();
		light.on();
	}
 
	public void undo() {
		light.dim(level);
	}
}
```

由于我们的卡槽不是单个的, 我们需要记录上次操作是操作的那一个卡槽.

```java
public class RemoteControlWithUndo {
	Command[] onCommands;
	Command[] offCommands;
	Command undoCommand;		//我们将上次操作的记录, 存储下来
 
	public RemoteControlWithUndo() {
		onCommands = new Command[7];
		offCommands = new Command[7];
 
		Command noCommand = new NoCommand();
		for(int i=0;i<7;i++) {
			onCommands[i] = noCommand;
			offCommands[i] = noCommand;
		}
		undoCommand = noCommand;
	}
  
	public void setCommand(int slot, Command onCommand, Command offCommand) {
		onCommands[slot] = onCommand;
		offCommands[slot] = offCommand;
	}
 
	public void onButtonWasPushed(int slot) {
		onCommands[slot].execute();
		undoCommand = onCommands[slot];
	}
 
	public void offButtonWasPushed(int slot) {
		offCommands[slot].execute();
		undoCommand = offCommands[slot];
	}
 
	public void undoButtonWasPushed() {	//当我们点击undo案件的额时候, 进行处理
		undoCommand.undo();
	}
  
	public String toString() {
		StringBuffer stringBuff = new StringBuffer();
		stringBuff.append("\n------ Remote Control -------\n");
		for (int i = 0; i < onCommands.length; i++) {
			stringBuff.append("[slot " + i + "] " + onCommands[i].getClass().getName()
				+ "    " + offCommands[i].getClass().getName() + "\n");
		}
		stringBuff.append("[undo] " + undoCommand.getClass().getName() + "\n");
		return stringBuff.toString();
	}
}
```

如果我们需要这样一个Party按钮, 当我们按下的时候, 灯会关闭, 音乐会打开, 热水器开始加热. 面对这种情况, 我们该怎么办?
很简单, 我们创建一个新的命令:

```java
public class MacroCommand implements Command {
	Command[] commands;
    public MacroCommand(Command[] commands) {
    	this.commands = commands;
    }
    
    public void execute() {
    	for (Command command : commands) {
        	command.execute();
        }
    }
}
```

## 命令模式的其他用途

### 队列请求

命令可以将运算块打包, 一个接受者和一组动作, 然后将其传来传去, 就像一般的对象一样.

想象有一个工作队列, 你在牟一端添加命令, 另一端则是线程. 线程进行下面的动作, 从队列中取出一个命令,调用它的execute()方法, 然后等待这个调用完成, 然后将该命令丢弃, 取出下一个命令...
工作队列和进行进行计算的对象之间完全是解耦的.

### 日志请求

某些应用需要我们将所有的动作都记录在日志中, 并能在系统死机之后, 重新调用这些动作恢复到之前的状态. 通过新增的(store(), load()命令, 命令模式可以支持这一点). 要怎么做呢? 每当我们执行命令的时候, 我们可以将这次记录存储到磁盘中, 一旦系统死机了, 我们就可以将这些命令对象重新加载, 并成批的调用这些对象的execute()方法. 如数据库恢复到某个初始点.

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

命令模式: 将`请求`封装成对象, 以便使用不同的请求, 队列或者日志来参数化其他对象. 命令模式也支持可撤销操作.

要点:

+ 命令模式将发出请求的对象和执行请求的对象解耦.
+ 在被解耦的两者之间是通过命令对象进行沟通的. 命令对象封装了接受者和一个或者一组的动作.
+ 调用者通过调用命令对象的execute()发出请求,这会使得接收者的动作被调用.
+ 调用者可以接受命令当做参数, 甚至在运行时动态地运行.
+ 命令支持撤销, 做法是实现一个undo()函数来回到execute()函数被调用时的状态.
+ 宏命令是命令的一种简单延伸, 允许调用多个命令. 宏命令也支持撤销.
+ 实际操作时, 很常见使用"聪明"命令对象, 也就是直接实现了请求, 而不是将工作委托给接收者.
+ 命令可以用来实现日志和事务系统.
