# 代理模式

我们回到糖果机的构建, 这时候 CEO 提出了一个新的要求, 就可以 CEO 希望可以获取一份库存和机器状态的报告. 嗯, 项目组的人认真想了想, 这不是很简单吗? 我们只需要创建一个监听器类进行监听就可以了:

```java
public class GumballMonitor {
   GumballMachine machine;

   public GumballMonitor(GumballMachine machine) {
      this.machine = machine;
   }

   public void report() {
      System.out.println("Gumball Machine: " + machine.getLocation());
      System.out.println("Current inventory: " + machine.getCount() + " gumballs");
      System.out.println("Current state: " + machine.getState());
   }
}
```

然后我们就可以为每一个糖果机创建一个监听器进行监听

```java
public class GumballMachineTestDrive {

   public static void main(String[] args) {
      int count = 0;

        if (args.length < 2) {
            System.out.println("GumballMachine <name> <inventory>");
            System.exit(1);
        }

        try {
           count = Integer.parseInt(args[1]);
      } catch (Exception e) {
         e.printStackTrace();
         System.exit(1);
      }
      GumballMachine gumballMachine = new GumballMachine(args[0], count);

      GumballMonitor monitor = new GumballMonitor(gumballMachine);


      System.out.println(gumballMachine);

      gumballMachine.insertQuarter();
      gumballMachine.turnCrank();
      gumballMachine.insertQuarter();
      gumballMachine.turnCrank();
       }
}
```

就当我们自信的将项目交给 CEO 的时候, CEO 却不买账: 我想的是远程监控这些糖果机, 你可不能叫我离开办公室跑到糖果机旁去检查, 这可不是我想要的.

这时候我该怎么办, 全部重新设计吗? 不不不, 不用的. 我们只需要采用代理模式. 首先我们需要认识远程代理的角色. 远程代理就好比"远程对象的本地代表". 什么叫做`远程对象`? 这是一种对象, 活在不同的 Java 虚拟机(JVM 堆中)(更加一般的说法是, 在不同地址空间运行的远程对象). 何谓`本地代表`, 就是可以由本地方法调用的对象, 其行为会转发到远程对象中. 这就意味着, 我们传递给糖果监视器的对象是本地代理对象, 我们可以通过代理访问其接口, 而代理会将操作通过 Socket 通信的方式发送给远端真正的对象, 而远端对象处理完毕之后, 将结果返回给代理. 最后由代理, 将结果呈现给接口调用.

可能我们觉得自己写这些代理对象会有点复杂, 是的, 但是 Java 已经给我们提供了封装的代理对象. 就在 Java.rmi 包下. Java RMI 提供了客户辅助对象 服务辅助对象, 为客户辅助对象创建和服务对象相同的方法. 使用 RMI 的好处就是, 我们不必写任何网络或 I/O 代码, 客户程序调用远程方法就和运行在自家本地 JVM 上对对象进行正常方法调用一样.

我们详细讲解下如何使用 RMI 的步骤:

1. **制作远程接口**, 远程接口可以让客户远程调用的方法. 客户将用它作为服务的类类型. Stub 和实际的服务都实现此接口.
2. **制作远程实现**, 这就是实际工作的类, 可以为远程接口中定义的远程方法, 提供了真正的实现. 这就是客户真正想要调用方法的地方.
3. **利用 RMIC 产生 Stub 和 Skeleton**, 这是客户和服务的辅助类. 你不需要创建这些类, 甚至不用自己处理, 因为 rmic 会自动创建.
4. **启动 RMI registry(rmiregistry)**, rmireistry 就像是电话簿, 客户可以通过这里查到到大理的位置.
5. **注册远程服务**, 向 rmiregistry 中注册服务, 这样, 客户才可以通过 rmiregistry 找到对应的服务.

> 注意这里有三个常犯的错误:
>
> 1. 启动远程服务时, 未启动 rmiregistry.
> 2. 忘记让变量和返回值成为可序列化的类型.
> 3. 忘记给客户提供 Stub 类.

我们将理论实现到我们的糖果机中:

1. 先创建我们自己的接口

```java
public interface GumballMachineRemote extends Remote {
//记得要继承Remote接口
   public int getCount() throws RemoteException;
   public String getLocation() throws RemoteException;
   public State getState() throws RemoteException;
}
```

这时候我们发现, 我们的 State 类不是可序列的对象, 我们需要进行序列化操作:

```java
public interface State extends Serializable {
   public void insertQuarter();
   public void ejectQuarter();
   public void turnCrank();
   public void dispense();
}
```

然后我们记得我们所有的状态类中, 都保存了一个糖果机的引用, 如:

```java
public class SoldState implements State {
   private static final long serialVersionUID = 2L;
    GumballMachine gumballMachine;
    ...
```

但是这个 gumballMachine 对象, 我们不应该进行序列化传递. 我们需要添加 transient 关键字, 告诉 JVM 不要序列化这个字段.

```java
public class SoldState implements State {
   private static final long serialVersionUID = 2L;
    transient GumballMachine gumballMachine;

    public SoldState(GumballMachine gumballMachine) {
        this.gumballMachine = gumballMachine;
    }
    ...
```

我们需要一个远程实现, 真正的操作是发生在这里, 我们需要修改 GumballMachine 类, 让他可以进行远程通信. 这里也很简单:

```java
public class GumballMachine
      extends UnicastRemoteObject implements GumballMachineRemote {
        //这里只需要继承UnicastRemoteObject和实现前面我们定义的接口就可以了, 让Java RMI为我们做剩下的事
   private static final long serialVersionUID = 2L;
   State soldOutState;
   State noQuarterState;
   State hasQuarterState;
   State soldState;
   State winnerState;

   State state = soldOutState;
   int count = 0;
    String location;

   public GumballMachine(String location, int numberGumballs) throws RemoteException {   //构造器这边记得抛出RemoteException, 因为父类抛出了这个异常
      soldOutState = new SoldOutState(this);
      noQuarterState = new NoQuarterState(this);
      hasQuarterState = new HasQuarterState(this);
      soldState = new SoldState(this);
      winnerState = new WinnerState(this);

      this.count = numberGumballs;
       if (numberGumballs > 0) {
         state = noQuarterState;
      }
      this.location = location;
   }
    ...
```

这里我们将该服务注册到 RMIregistry 中去了, 记得之前要开启 rmiregistry(通过 rmiregistry).

```java
public class GumballMachineTestDrive {

   public static void main(String[] args) {
      GumballMachineRemote gumballMachine = null;
      int count;

      if (args.length < 2) {
         System.out.println("GumballMachine <name> <inventory>");
          System.exit(1);
      }

      try {
         count = Integer.parseInt(args[1]);

         gumballMachine =
            new GumballMachine(args[0], count);
         Naming.rebind("//" + args[0] + "/gumballmachine", gumballMachine);
      } catch (Exception e) {
         e.printStackTrace();
      }
   }
}
```

这时候我们需要修改我们的监视器, 让他调用远程的服务了.

```java
public class GumballMonitor {
   GumballMachineRemote machine;

   public GumballMonitor(GumballMachineRemote machine) {
      this.machine = machine;
   }

   public void report() {
      try {
         System.out.println("Gumball Machine: " + machine.getLocation());
         System.out.println("Current inventory: " + machine.getCount() + " gumballs");
         System.out.println("Current state: " + machine.getState());
      } catch (RemoteException e) {
         e.printStackTrace();
      }
   }
}
```

接下来我们就可以, 测试一下调用远程的服务了:

```java
public class GumballMonitorTestDrive {

   public static void main(String[] args) {
      String[] location = {"rmi://santafe.mightygumball.com/gumballmachine",
                           "rmi://boulder.mightygumball.com/gumballmachine",
                           "rmi://seattle.mightygumball.com/gumballmachine"};

      if (args.length >= 0) {
            location = new String[1];
            location[0] = "rmi://" + args[0] + "/gumballmachine";
        }

      GumballMonitor[] monitor = new GumballMonitor[location.length];

      for (int i=0;i < location.length; i++) {
         try {
                 GumballMachineRemote machine =
                  (GumballMachineRemote) Naming.lookup(location[i]);
                 monitor[i] = new GumballMonitor(machine);
            System.out.println(monitor[i]);
           } catch (Exception e) {
               e.printStackTrace();
           }
      }

      for(int i=0; i < monitor.length; i++) {
         monitor[i].report();
      }
   }
}
```

## 定义

代理模式: 为另一个对象提供替身或占位符以控制对这个对象的访问.

我们上面用的是标准的远程代理, 这里介绍一种新的代理, **虚拟代理**. 虚拟代理作为创建开销大的对象的代表. 虚拟代理经常直到我们真正需要一个对象的时候才创建它. 当这个对象在创建前和创建中时, 由虚拟代理来扮演对象的替身. 对象创建之后, 代理就会将请求直接委托给对象.

这里我们打算建立一个应用程序, 用来展示你最喜欢的 CD 封面, 你可以建立一个 CD 标题菜单, 然后从 Amazon.com 等网站的在线服务中取得 CD 封面的图. 如果你使用 Swing, 可以创建一个 Icon 接口从网络上加载图像. 唯一的问题是,限于连接带宽和网络负载, 下载可能需要一些实践, 所以在等待图像时整个应用程序就被挂起, 一旦图像加载完成, 刚才显示的东西应该消失, 图像就显示出来了.
这里很简单, 我们使用虚拟代理进行实现, 当你还在加载的时候, 显示"CD 正在加载中, 请稍候....", 当你加载完之后, 就显示图片.
这里详述其是怎么工作的:

1. ImageProxy 首先创建一个 ImageIcon, 然后从网络 URL 上加载图像.
2. 在加载的过程中, ImageProxy 显示"CD 封面加载中, 请稍候...".
3. 当图像加载完毕, ImageProxy 把所有的方法调用委托给真正的 ImageIcon, 这些方法包括 paintIcon(), getWidth()和 getHeight().
4. 当用户请求新的图像, 我们就创建新的代理, 重复这样的过程.

```java
class ImageProxy implements Icon {
   volatile ImageIcon imageIcon;
   final URL imageURL;
   Thread retrievalThread;
   boolean retrieving = false;

   public ImageProxy(URL url) { imageURL = url; }

   public int getIconWidth() {
      if (imageIcon != null) {
            return imageIcon.getIconWidth();
        } else {
         return 800;
      }
   }

   public int getIconHeight() {
      if (imageIcon != null) {
            return imageIcon.getIconHeight();
        } else {
         return 600;
      }
   }

   synchronized void setImageIcon(ImageIcon imageIcon) {
      this.imageIcon = imageIcon;
   }

   public void paintIcon(final Component c, Graphics  g, int x,  int y) {
      if (imageIcon != null) {
         imageIcon.paintIcon(c, g, x, y);
      } else {
         g.drawString("Loading CD cover, please wait...", x+300, y+190);
         if (!retrieving) {
            retrieving = true;

            retrievalThread = new Thread(new Runnable() {
               public void run() {
                  try {
                     setImageIcon(new ImageIcon(imageURL, "CD Cover"));
                     c.repaint();
                  } catch (Exception e) {
                     e.printStackTrace();
                  }
               }
            });
            retrievalThread.start();
         }
      }
   }
}
```

我们进行测试:

```java
public class ImageProxyTestDrive {
   ImageComponent imageComponent;
   JFrame frame = new JFrame("CD Cover Viewer");
   JMenuBar menuBar;
   JMenu menu;
   Hashtable<String, String> cds = new Hashtable<String, String>();

   public static void main (String[] args) throws Exception {
      ImageProxyTestDrive testDrive = new ImageProxyTestDrive();
   }

   public ImageProxyTestDrive() throws Exception{
      cds.put("Buddha Bar","http://images.amazon.com/images/P/B00009XBYK.01.LZZZZZZZ.jpg");
      cds.put("Ima","http://images.amazon.com/images/P/B000005IRM.01.LZZZZZZZ.jpg");
      cds.put("Karma","http://images.amazon.com/images/P/B000005DCB.01.LZZZZZZZ.gif");
      cds.put("MCMXC A.D.","http://images.amazon.com/images/P/B000002URV.01.LZZZZZZZ.jpg");
      cds.put("Northern Exposure","http://images.amazon.com/images/P/B000003SFN.01.LZZZZZZZ.jpg");
      cds.put("Selected Ambient Works, Vol. 2","http://images.amazon.com/images/P/B000002MNZ.01.LZZZZZZZ.jpg");

      URL initialURL = new URL((String)cds.get("Selected Ambient Works, Vol. 2"));
      menuBar = new JMenuBar();
      menu = new JMenu("Favorite CDs");
      menuBar.add(menu);
      frame.setJMenuBar(menuBar);

      for (Enumeration<String> e = cds.keys(); e.hasMoreElements();) {
         String name = (String)e.nextElement();
         JMenuItem menuItem = new JMenuItem(name);
         menu.add(menuItem);
         menuItem.addActionListener(event -> {
            imageComponent.setIcon(new ImageProxy(getCDUrl(event.getActionCommand())));
            frame.repaint();
         });
      }

      // set up frame and menus

      Icon icon = new ImageProxy(initialURL);
      imageComponent = new ImageComponent(icon);
      frame.getContentPane().add(imageComponent);
      frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
      frame.setSize(800,600);
      frame.setVisible(true);

   }

   URL getCDUrl(String name) {
      try {
         return new URL((String)cds.get(name));
      } catch (MalformedURLException e) {
         e.printStackTrace();
         return null;
      }
   }
}
```

Java 在 java.lang.reflect 包中有自己的代理支持, 利用这个包我们可以在运行时动态地创建一个代理类, 实现一个或者多个接口, 并将方法的调用转发到你指定的类. 因为实际代理类在运行时创建, 我们称这个 Java 技术为: 动态代理.

每个城镇都需要配对服务, 如果让你负责对象村的约会服务系统.这里是你设计的接口:

```java
public interface PersonBean {

   String getName();
   String getGender();
   String getInterests();
   int getHotOrNotRating();   //这是火热程度, 相当于评价高低1到10

    void setName(String name);
    void setGender(String gender);
    void setInterests(String interests);
    void setHotOrNotRating(int rating);

}
```

然后我们添加一个实现, 这里是我们真正实现的地方:

```java
public class PersonBeanImpl implements PersonBean {
   String name;
   String gender;
   String interests;
   int rating;
   int ratingCount = 0;

   public String getName() {
      return name;
   }

   public String getGender() {
      return gender;
   }

   public String getInterests() {
      return interests;
   }

   public int getHotOrNotRating() {
      if (ratingCount == 0) return 0;
      return (rating/ratingCount);
   }


   public void setName(String name) {
      this.name = name;
   }

   public void setGender(String gender) {
      this.gender = gender;
   }

   public void setInterests(String interests) {
      this.interests = interests;
   }

   public void setHotOrNotRating(int rating) {
      this.rating += rating;
      ratingCount++;
   }
}
```

然而我们发现, 有些人不守规矩, 每次都把自己的评分修改的特别高, 擅自修改他人的信息. 这很不好, 我们需要加入权限校验. 我们定义两个代理, 一个是自己的信息代理, 另一个是对方的信息代理. 两者的权限有差异.

```java
public class OwnerInvocationHandler implements InvocationHandler {
   PersonBean person;

   public OwnerInvocationHandler(PersonBean person) {
      this.person = person;
   }

   public Object invoke(Object proxy, Method method, Object[] args)
         throws IllegalAccessException {

      try {
         if (method.getName().startsWith("get")) {
            return method.invoke(person, args);
            } else if (method.getName().equals("setHotOrNotRating")) {
            throw new IllegalAccessException();
         } else if (method.getName().startsWith("set")) {
            return method.invoke(person, args);
         }
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
      return null;
   }
}

public class NonOwnerInvocationHandler implements InvocationHandler {
   PersonBean person;

   public NonOwnerInvocationHandler(PersonBean person) {
      this.person = person;
   }

   public Object invoke(Object proxy, Method method, Object[] args)
         throws IllegalAccessException {

      try {
         if (method.getName().startsWith("get")) {
            return method.invoke(person, args);
            } else if (method.getName().equals("setHotOrNotRating")) {
            return method.invoke(person, args);
         } else if (method.getName().startsWith("set")) {
            throw new IllegalAccessException();
         }
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
      return null;
   }
}
```

然后我们动态设置:

```java
public class MatchMakingTestDrive {
   HashMap<String, PersonBean> datingDB = new HashMap<String, PersonBean>(); //模拟数据库
   public static void main(String[] args) {
      MatchMakingTestDrive test = new MatchMakingTestDrive();
      test.drive();
   }

   public MatchMakingTestDrive() {
      initializeDatabase();
   }

   public void drive() {
      PersonBean joe = getPersonFromDatabase("Joe Javabean");
      PersonBean ownerProxy = getOwnerProxy(joe);
      System.out.println("Name is " + ownerProxy.getName());
      ownerProxy.setInterests("bowling, Go");
      System.out.println("Interests set from owner proxy");
      try {
         ownerProxy.setHotOrNotRating(10);
      } catch (Exception e) {
         System.out.println("Can't set rating from owner proxy");
      }
      System.out.println("Rating is " + ownerProxy.getHotOrNotRating());

      PersonBean nonOwnerProxy = getNonOwnerProxy(joe);
      System.out.println("Name is " + nonOwnerProxy.getName());
      try {
         nonOwnerProxy.setInterests("bowling, Go");
      } catch (Exception e) {
         System.out.println("Can't set interests from non owner proxy");
      }
      nonOwnerProxy.setHotOrNotRating(3);
      System.out.println("Rating set from non owner proxy");
      System.out.println("Rating is " + nonOwnerProxy.getHotOrNotRating());
   }

   PersonBean getOwnerProxy(PersonBean person) { //这里我们获得个人的信息代理

        return (PersonBean) Proxy.newProxyInstance(
               person.getClass().getClassLoader(),
               person.getClass().getInterfaces(),
                new OwnerInvocationHandler(person));
   }

   PersonBean getNonOwnerProxy(PersonBean person) { //这里我们获得他人的信息代理

        return (PersonBean) Proxy.newProxyInstance(
               person.getClass().getClassLoader(),
               person.getClass().getInterfaces(),
                new NonOwnerInvocationHandler(person));
   }

   PersonBean getPersonFromDatabase(String name) {
      return (PersonBean)datingDB.get(name);
   }

   void initializeDatabase() {
      PersonBean joe = new PersonBeanImpl();
      joe.setName("Joe Javabean");
      joe.setInterests("cars, computers, music");
      joe.setHotOrNotRating(7);
      datingDB.put(joe.getName(), joe);

      PersonBean kelly = new PersonBeanImpl();
      kelly.setName("Kelly Klosure");
      kelly.setInterests("ebay, movies, music");
      kelly.setHotOrNotRating(6);
      datingDB.put(kelly.getName(), kelly);
   }
}
```

其实这里还有很多别的代理实现:
**防火墙代理(Firewall Proxy)**: 控制网络资源的访问, 保护主题免于"坏客户"的破坏.
**智能引用代理(Smart Reference Proxy)**: 当一个主题被引用时, 进行额外的动作, 列如计算一个对象被引用的次数.
**缓存代理(Caching Proxy)**: 为开销大的运算结果提供暂时存储, 它也允许多个客户共享结果, 以减少计算或者网络延迟.
**同步代理(Synchronization Proxy)**: 在多线程的情况下为主题提供安全的访问.
**复杂隐藏代理(Complexity Hiding Proxy)**: 用来隐藏一个类的复杂集合的复杂度, 并进行访问控制. 有时候也称为外观代理. 对接口访问提供限制.
**写入时复制代理(Copy-On-Write Proxy)**: 用来控制对象的复制, 方法是延迟对象的复制, 直到客户真正需要为止.(Java 中 CopyOnWriteArrayList),

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

代理模式: 为另一个对象提供替身或占位符以控制对这个对象的访问.

要点:

- 代理模式为另一个对象提供代表, 以便控制客户对对象的访问, 管理访问的方式有许多种.
- 远程代理管理客户和远程对象之间的交互.
- 虚拟代理控制访问实例化开销大的对象.
- 保护代理基于调用者控制对对象方法的访问.
- 代理模式有许多变种, 列如: 缓存代理, 同步代理, 写入时复制代理等等.
- 代理在结构上类似装饰者, 但是目的不同.
- 装饰者为客户加上新的行为, 而代理是控制访问.
- Java 内置的代理支持, 可以根据需要建立动态代理, 并将所有调用分配到所选的处理器.
- 和其他包装者一样, 代理会增加你设计中类的数量.
