# 管理良好的集合
有两家餐厅要合并了, 煎饼店专门负责提供早餐, 原先的餐厅要用来提供午餐. 但是两家的菜单使用的不是同一种存储方式进行存储的, 一个使用数组一个使用ArrayList.

```java
//双方都使用同一个菜单项显示
public class MenuItem {
	String name;
	String description;
	boolean vegetarian;
	double price;
 
	public MenuItem(String name, 
	                String description, 
	                boolean vegetarian, 
	                double price) 
	{
		this.name = name;
		this.description = description;
		this.vegetarian = vegetarian;
		this.price = price;
	}
  
	public String getName() {
		return name;
	}
  
	public String getDescription() {
		return description;
	}
  
	public double getPrice() {
		return price;
	}
  
	public boolean isVegetarian() {
		return vegetarian;
	}
	public String toString() {
		return (name + ", $" + price + "\n   " + description);
	}
}

//煎饼店的实现
public class PancakeHouseMenu implements Menu {
	ArrayList<MenuItem> menuItems;
 
	public PancakeHouseMenu() {
		menuItems = new ArrayList<MenuItem>();
    
		addItem("K&B's Pancake Breakfast", 
			"Pancakes with scrambled eggs, and toast", 
			true,
			2.99);
 
		addItem("Regular Pancake Breakfast", 
			"Pancakes with fried eggs, sausage", 
			false,
			2.99);
 
		addItem("Blueberry Pancakes",
			"Pancakes made with fresh blueberries",
			true,
			3.49);
 
		addItem("Waffles",
			"Waffles, with your choice of blueberries or strawberries",
			true,
			3.59);
	}

	public void addItem(String name, String description,
	                    boolean vegetarian, double price)
	{
		MenuItem menuItem = new MenuItem(name, description, vegetarian, price);
		menuItems.add(menuItem);
	}
 
	public ArrayList<MenuItem> getMenuItems() {
		return menuItems;
	}
  
	public Iterator createIterator() {
		return new PancakeHouseMenuIterator(menuItems);
	}
  
	public String toString() {
		return "Objectville Pancake House Menu";
	}

	// other menu methods here
}
//午餐店的实现
public class DinerMenu implements Menu {
	static final int MAX_ITEMS = 6;
	int numberOfItems = 0;
	MenuItem[] menuItems;
  
	public DinerMenu() {
		menuItems = new MenuItem[MAX_ITEMS];
 
		addItem("Vegetarian BLT",
			"(Fakin') Bacon with lettuce & tomato on whole wheat", true, 2.99);
		addItem("BLT",
			"Bacon with lettuce & tomato on whole wheat", false, 2.99);
		addItem("Soup of the day",
			"Soup of the day, with a side of potato salad", false, 3.29);
		addItem("Hotdog",
			"A hot dog, with saurkraut, relish, onions, topped with cheese",
			false, 3.05);
		addItem("Steamed Veggies and Brown Rice",
			"Steamed vegetables over brown rice", true, 3.99);
		addItem("Pasta",
			"Spaghetti with Marinara Sauce, and a slice of sourdough bread",
			true, 3.89);
	}
  
	public void addItem(String name, String description, 
	                     boolean vegetarian, double price) 
	{
		MenuItem menuItem = new MenuItem(name, description, vegetarian, price);
		if (numberOfItems >= MAX_ITEMS) {
			System.err.println("Sorry, menu is full!  Can't add item to menu");
		} else {
			menuItems[numberOfItems] = menuItem;
			numberOfItems = numberOfItems + 1;
		}
	}
 
	public MenuItem[] getMenuItems() {
		return menuItems;
	}
  
	public Iterator createIterator() {
		return new DinerMenuIterator(menuItems);
		// To test Alternating menu items, comment out above line,
		// and uncomment the line below.
		//return new AlternatingDinerMenuIterator(menuItems);
	}
 
	// other menu methods here
}
```

这在平时到没什么问题, 但是当两家店合并的时候就出现问题了. 两家店合并之后, 店铺里的女招待就要提供一份不同的菜单了. 假设女招待的功能如下:
+ printMenu() : 打印所有的菜单
+ printBreakfastMeni() : 打印早餐的菜单
+ printLunchMenu() : 打印午餐的菜单
+ pintVegetarianMenu() : 打印所有的素食菜单
+ isItemVegetarian(name) : 返回该项是否为素食

如果要打印菜单的话:

```java
PancakeHouseMenu pancakeHouseMenu = new PancakeHouseMenu();
DinerMenu dinerMenu = new DinerMenu();

ArrayList<MenuItem> breakfastItems = pancakeHouseMenu.getMenuItems();
MenuItem[] lunchItems = dinerMenu.getMenuItems();

// Hiding implementation
System.out.println("USING FOR EACH");
for (MenuItem menuItem : breakfastItems) {
    System.out.print(menuItem.getName());
    System.out.println("\t\t" + menuItem.getPrice());
    System.out.println("\t" + menuItem.getDescription());
}
for (MenuItem menuItem : lunchItems) {
    System.out.print(menuItem.getName());
    System.out.println("\t\t" + menuItem.getPrice());
    System.out.println("\t" + menuItem.getDescription());
}
```

但是这个实现就存在着问题:
+ 我们的实现是针对PancakeHouseMenu和DinerMenu的具体实现编码, 而不是针对接口.
+ 如果我们添加了新的菜单, 菜单是使用Hashtable进行存放的, 我们就必须需要修改很多代码.
+ 女招待需要知道每个菜单适合存储内部的菜单项的, 这违反了封装.
+ 我们有大量重复的代码.

但是两家的饭店都不想修改自己的实现, 因为那意味着要重写很多代码.  如果我们能够找出一个方法, 让他们的菜单实现一个相同的接口, 那就好了. 而从之前的书中, 我们可以学到要封装变化的部分. 很明显, 这里发生变化的就是遍历过程的区别, 如果我们可以封装起来.

我们可以创建一个对象, 将它称为迭代器(Iterator), 利用它来遍历集合中的每一个对象. 

```java
Iterator Iterator = breakfastMenu.createIterator();

while(iterator.hasNext()) {
	MenuItem menuItem = (MenuItem) iterator.next();
}
```

是的我们可以利用Iterator接口进行内部遍历的实现. 我们首先定义一个接口:

```java
public interface Iterator {
	boolean hasNext();
    Object next();
}

//定义午餐的迭代器
public class DinerMenuIterator implements Iterator {
	MenuItem[] items;
    int position = 0;
    
    public DinerMenuIterator(MenuItem[] items) {
    	this.items = items;
    }
    
    public Object next() {
    	MenuItem menuItem = items[position];
        position = position + 1;
        return menuItem;
    }
    
    public boolean hasNext() {
    	if (position >= items.length || items[position] == null) {
        	return false;
        }
        return true;
    }
}
//午餐类似

//我们就可以这样进行遍历了
public void printMenu() {
    Iterator pancakeIterator = pancakeHouseMenu.createIterator();
    Iterator dinerIterator = dinerMenu.createIterator();

    System.out.println("MENU\n----\nBREAKFAST");
    printMenu(pancakeIterator);
    System.out.println("\nLUNCH");
    printMenu(dinerIterator);
}

private void printMenu(Iterator iterator) {
    while (iterator.hasNext()) {
        MenuItem menuItem = iterator.next();
        System.out.print(menuItem.getName() + ", ");
        System.out.print(menuItem.getPrice() + " -- ");
        System.out.println(menuItem.getDescription());
    }
}
```

其实Java中已经提供了一个接口Iterator, 接口源代码如下:

```java
public interface Iterator<E> {
    /**
     * Returns {@code true} if the iteration has more elements.
     * (In other words, returns {@code true} if {@link #next} would
     * return an element rather than throwing an exception.)
     *
     * @return {@code true} if the iteration has more elements
     */
    boolean hasNext();

    /**
     * Returns the next element in the iteration.
     *
     * @return the next element in the iteration
     * @throws NoSuchElementException if the iteration has no more elements
     */
    E next();

    /**
     * Removes from the underlying collection the last element returned
     * by this iterator (optional operation).  This method can be called
     * only once per call to {@link #next}.  The behavior of an iterator
     * is unspecified if the underlying collection is modified while the
     * iteration is in progress in any way other than by calling this
     * method.
     *
     * @implSpec
     * The default implementation throws an instance of
     * {@link UnsupportedOperationException} and performs no other action.
     *
     * @throws UnsupportedOperationException if the {@code remove}
     *         operation is not supported by this iterator
     *
     * @throws IllegalStateException if the {@code next} method has not
     *         yet been called, or the {@code remove} method has already
     *         been called after the last call to the {@code next}
     *         method
     */
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    /**
     * Performs the given action for each remaining element until all elements
     * have been processed or the action throws an exception.  Actions are
     * performed in the order of iteration, if that order is specified.
     * Exceptions thrown by the action are relayed to the caller.
     *
     * @implSpec
     * <p>The default implementation behaves as if:
     * <pre>{@code
     *     while (hasNext())
     *         action.accept(next());
     * }</pre>
     *
     * @param action The action to be performed for each element
     * @throws NullPointerException if the specified action is null
     * @since 1.8
     */
    default void forEachRemaining(Consumer<<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
	}
}
```

这里唯一的区别是添加了新的remove()方法和JDK1.8支持的default实现的forEachRemaining方法传递一个consumer函数式接口, 对每个成员进行处理. 这里两个方法都添加了default默认实现, 如果实现了的接口的类不想实现, 不实现, 调用接口的默认实现.

如果我们使用默认的接口:

```java
public class DinerMenuIterator implements Iterator<MenuItem> {
	MenuItem[] list;
	int position = 0;
 
	public DinerMenuIterator(MenuItem[] list) {
		this.list = list;
	}
 
	public MenuItem next() {
		MenuItem menuItem = list[position];
		position = position + 1;
		return menuItem;
	}
 
	public boolean hasNext() {
		if (position >= list.length || list[position] == null) {
			return false;
		} else {
			return true;
		}
	}
 
	public void remove() {
		if (position <= 0) {
			throw new IllegalStateException
				("You can't remove an item until you've done at least one next()");
		}
		if (list[position-1] != null) {
			for (int i = position-1; i < (list.length-1); i++) {
				list[i] = list[i+1];
			}
			list[list.length-1] = null;
		}
	}

}
//ArrayList本身实现了, 该接口, 直接放回就可以了.
```

最后一步, 我们为每一个菜单添加一个新的接口Menu

```java
public interface Menu {
	public Iterator createIterator();
}
```

这样我们的女招待就可以针对Menu这个接口进行编程了, 就不需要知道哪些菜单是怎么实现的了.

最终我们的waitress最终实现:

```java
public class Waitress {
    ArrayList<Menu> menus;


    public Waitress(ArrayList<Menu> menus) {
        this.menus = menus;
    }

    public void printMenu() {
        Iterator<?> menuIterator = menus.iterator();
        while(menuIterator.hasNext()) {
            Menu menu = (Menu)menuIterator.next();
            printMenu(menu.createIterator());
        }
    }

    void printMenu(Iterator<?> iterator) {
        while (iterator.hasNext()) {
            MenuItem menuItem = (MenuItem)iterator.next();
            System.out.print(menuItem.getName() + ", ");
            System.out.print(menuItem.getPrice() + " -- ");
            System.out.println(menuItem.getDescription());
        }
    }
} 
```

## 定义
迭代器模式: 提供了一种方法顺序访问一个聚合对象中的各个元素, 而又不用暴露其内部的实现.

这里我们引入一个新的设计原则:
**一个类应该只有一个引起变化的原因**
> 我们知道要避免类内的改变, 因为修改代码很容易造成很多潜在的错误. 如果一个类具有两个改变的原因, 那么这会使得将来该类的变化几率上升, 而当它真的改变时, 我们设计的两个方面都将受到影响. 我们需要将一个责任只指派给一个类.
> 内聚(cohesion): 度量一个类或者模块紧密地达到单一目的或责任. 当一个模块或者一个类被设计成只支持一组相关功能时, 我们称它具有高内聚, 反正, 当被设计成支持一组不相关的功能时, 我们说它具有低内聚.


如果我们打算在午餐之后添加一份餐后甜点的子菜单, 这时候我们会发现, 原先的MenuItem就没办法满足我们的额设计了, 因为MenuItem没有添加子菜单项这份功能.这里我们就必须引入新的设计模式: 组合模式.

## 定义
组合模式: 允许你将对象组合成树形结构来表现"整体/部分"的层次结构. 组合能让客户以一致的方式处理个别对象以及对象组合.
> 以菜单为例, 我们思考: 我们需要创建一个树形结构, 在同一个结构中处理嵌套菜单和菜单项组. 通过将菜单和项放在相同的结构中, 我们创建了一个"整体/部分"层次结构, 即由菜单和菜单项组成的对象树. 但是可以将它视为一个整体, 像是一个丰富的大菜单.

首先我们定义一个通用的抽象类:

```java
public abstract class MenuComponent {
	public void add(MenuComponent menuComponent) {
		throw new UnsupportedOperationException();
	}
	public void remove(MenuComponent menuComponent) {
		throw new UnsupportedOperationException();
	}
	public MenuComponent getChild(int i) {
		throw new UnsupportedOperationException();
	}
  
	public String getName() {
		throw new UnsupportedOperationException();
	}
	public String getDescription() {
		throw new UnsupportedOperationException();
	}
	public double getPrice() {
		throw new UnsupportedOperationException();
	}
	public boolean isVegetarian() {
		throw new UnsupportedOperationException();
	}
  
	public void print() {
		throw new UnsupportedOperationException();
	}
}
```

我们将默认实现都为不支持, 由子类进行选择性的覆盖处理.

```java
//菜单项(相当于叶节点)
public class MenuItem extends MenuComponent {
	String name;
	String description;
	boolean vegetarian;
	double price;
    
	public MenuItem(String name, 
	                String description, 
	                boolean vegetarian, 
	                double price) 
	{ 
		this.name = name;
		this.description = description;
		this.vegetarian = vegetarian;
		this.price = price;
	}
  
	public String getName() {
		return name;
	}
  
	public String getDescription() {
		return description;
	}
  
	public double getPrice() {
		return price;
	}
  
	public boolean isVegetarian() {
		return vegetarian;
	}
  
	public void print() {
		System.out.print("  " + getName());
		if (isVegetarian()) {
			System.out.print("(v)");
		}
		System.out.println(", " + getPrice());
		System.out.println("     -- " + getDescription());
	}
}
//子菜单项
public class Menu extends MenuComponent {
	ArrayList<MenuComponent> menuComponents = new ArrayList<MenuComponent>();
	String name;
	String description;
  
	public Menu(String name, String description) {
		this.name = name;
		this.description = description;
	}
 
	public void add(MenuComponent menuComponent) {
		menuComponents.add(menuComponent);
	}
 
	public void remove(MenuComponent menuComponent) {
		menuComponents.remove(menuComponent);
	}
 
	public MenuComponent getChild(int i) {
		return (MenuComponent)menuComponents.get(i);
	}
 
	public String getName() {
		return name;
	}
 
	public String getDescription() {
		return description;
	}
 
	public void print() {
		System.out.print("\n" + getName());
		System.out.println(", " + getDescription());
		System.out.println("---------------------");
  
		Iterator<MenuComponent> iterator = menuComponents.iterator();
		while (iterator.hasNext()) {
			MenuComponent menuComponent = 
				(MenuComponent)iterator.next();
			menuComponent.print();
		}
	}
}
```

这里添加测试代码:

```java
public class MenuTestDrive {
	public static void main(String args[]) {
		MenuComponent pancakeHouseMenu = 
			new Menu("PANCAKE HOUSE MENU", "Breakfast");
		MenuComponent dinerMenu = 
			new Menu("DINER MENU", "Lunch");
		MenuComponent cafeMenu = 
			new Menu("CAFE MENU", "Dinner");
		MenuComponent dessertMenu = 
			new Menu("DESSERT MENU", "Dessert of course!");
		MenuComponent coffeeMenu = new Menu("COFFEE MENU", "Stuff to go with your afternoon coffee");
  
		MenuComponent allMenus = new Menu("ALL MENUS", "All menus combined");
  
		allMenus.add(pancakeHouseMenu);
		allMenus.add(dinerMenu);
		allMenus.add(cafeMenu);
  
		pancakeHouseMenu.add(new MenuItem(
			"K&B's Pancake Breakfast", 
			"Pancakes with scrambled eggs, and toast", 
			true,
			2.99));
		pancakeHouseMenu.add(new MenuItem(
			"Regular Pancake Breakfast", 
			"Pancakes with fried eggs, sausage", 
			false,
			2.99));
		pancakeHouseMenu.add(new MenuItem(
			"Blueberry Pancakes",
			"Pancakes made with fresh blueberries, and blueberry syrup",
			true,
			3.49));
		pancakeHouseMenu.add(new MenuItem(
			"Waffles",
			"Waffles, with your choice of blueberries or strawberries",
			true,
			3.59));

		dinerMenu.add(new MenuItem(
			"Vegetarian BLT",
			"(Fakin') Bacon with lettuce & tomato on whole wheat", 
			true, 
			2.99));
		dinerMenu.add(new MenuItem(
			"BLT",
			"Bacon with lettuce & tomato on whole wheat", 
			false, 
			2.99));
		dinerMenu.add(new MenuItem(
			"Soup of the day",
			"A bowl of the soup of the day, with a side of potato salad", 
			false, 
			3.29));
		dinerMenu.add(new MenuItem(
			"Hotdog",
			"A hot dog, with saurkraut, relish, onions, topped with cheese",
			false, 
			3.05));
		dinerMenu.add(new MenuItem(
			"Steamed Veggies and Brown Rice",
			"Steamed vegetables over brown rice", 
			true, 
			3.99));
 
		dinerMenu.add(new MenuItem(
			"Pasta",
			"Spaghetti with Marinara Sauce, and a slice of sourdough bread",
			true, 
			3.89));
   
		dinerMenu.add(dessertMenu);
  
		dessertMenu.add(new MenuItem(
			"Apple Pie",
			"Apple pie with a flakey crust, topped with vanilla icecream",
			true,
			1.59));
  
		dessertMenu.add(new MenuItem(
			"Cheesecake",
			"Creamy New York cheesecake, with a chocolate graham crust",
			true,
			1.99));
		dessertMenu.add(new MenuItem(
			"Sorbet",
			"A scoop of raspberry and a scoop of lime",
			true,
			1.89));

		cafeMenu.add(new MenuItem(
			"Veggie Burger and Air Fries",
			"Veggie burger on a whole wheat bun, lettuce, tomato, and fries",
			true, 
			3.99));
		cafeMenu.add(new MenuItem(
			"Soup of the day",
			"A cup of the soup of the day, with a side salad",
			false, 
			3.69));
		cafeMenu.add(new MenuItem(
			"Burrito",
			"A large burrito, with whole pinto beans, salsa, guacamole",
			true, 
			4.29));

		cafeMenu.add(coffeeMenu);

		coffeeMenu.add(new MenuItem(
			"Coffee Cake",
			"Crumbly cake topped with cinnamon and walnuts",
			true,
			1.59));
		coffeeMenu.add(new MenuItem(
			"Bagel",
			"Flavors include sesame, poppyseed, cinnamon raisin, pumpkin",
			false,
			0.69));
		coffeeMenu.add(new MenuItem(
			"Biscotti",
			"Three almond or hazelnut biscotti cookies",
			true,
			0.89));
 
		Waitress waitress = new Waitress(allMenus);
   
		waitress.printMenu();
	}
}
```

但是如果我们的女招待需要选择一些素食菜单, 这就需要遍历整个菜单项进行处理了. 这里我们需要在接口中添加遍历器处理. 我们在MenuComponent抽象类中添加新的抽象方法, 由子类实现.

```java
//其他的代码不变
+ public abstract Iterator<MenuComponent> createIterator();
```

然后我们创建一个组合迭代器用来迭代遍历所有的子菜单项:

```java
public class CompositeIterator implements Iterator<MenuComponent> {
	Stack<Iterator<MenuComponent>> stack = new Stack<Iterator<MenuComponent>>();
   //这里利用一个堆栈进行存储, 每次迭代的位置
   
	public CompositeIterator(Iterator<MenuComponent> iterator) {
		stack.push(iterator);
	}
    //传递进入顶层的菜单的迭代器
   
	public MenuComponent next() {
		if (hasNext()) {	//如果还有下一个菜单项迭代器
			Iterator<MenuComponent> iterator = stack.peek();	//取出迭代器
			MenuComponent component = iterator.next();	//取出迭代器中菜单项
			stack.push(component.createIterator());	//将该迭代器放入堆栈
			return component;	//返回当前的菜单项
		} else {
			return null;
		}
	}
  
	public boolean hasNext() {
		if (stack.empty()) {	//如果堆栈内为空, 返回false
			return false;
		} else {
			Iterator<MenuComponent> iterator = stack.peek();	//取出第一个迭代器
			if (!iterator.hasNext()) {	//如果迭代器没有菜单项
				stack.pop();	//弹出改迭代器
				return hasNext(); //再次调用该函数
			} else {
				return true;	//返回有
			}
		}
	}
}
```

这里是一个外部迭代器, 通过Stack进行存储每次迭代的位置信息.


我们在定义一个空迭代器用来表明是叶节点.

```java
public class NullIterator implements Iterator<MenuComponent> {
   
	public MenuComponent next() {
		return null;
	}
  
	public boolean hasNext() {
		return false;
	}
}
```

最后我们在MenuItem中绑定空迭代器, Menu中绑定组合迭代器

```java
public Iterator<MenuComponent> createIterator() {
    return new NullIterator();
}

public Iterator<MenuComponent> createIterator() {
    if (iterator == null) {
        iterator = new CompositeIterator(menuComponents.iterator());
    }
    return iterator;
}
```

这时候我们就可以利用迭代器进行遍历所有的菜单项了, 注意异常的处理:

```java
public class Waitress {
	MenuComponent allMenus;
 
	public Waitress(MenuComponent allMenus) {
		this.allMenus = allMenus;
	}
 
	public void printMenu() {
		allMenus.print();
	}
  
	public void printVegetarianMenu() {
		Iterator<MenuComponent> iterator = allMenus.createIterator();

		System.out.println("\nVEGETARIAN MENU\n----");
		while (iterator.hasNext()) {
			MenuComponent menuComponent = iterator.next();
			try {
				if (menuComponent.isVegetarian()) {
					menuComponent.print();
				}
			} catch (UnsupportedOperationException e) {}
		}
	}
}
```

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

迭代器模式: 提供了一种方法顺序访问一个聚合对象中的各个元素, 而又不用暴露其内部的实现.

组合模式: 允许你将对象组合成树形结构来表现"整体/部分"的层次结构. 组合能让客户以一致的方式处理个别对象以及对象组合.

要点:
+ 迭代器允许访问聚合的元素, 而不需要暴露其内部结构.
+ 迭代器将遍历聚合的工作封装进一个对象中.
+ 当使用迭代器的时候, 我们依赖聚合提供遍历.
+ 迭代器提供了一个通用的接口, 让我们遍历聚合的项, 当我们编码使用聚合的项时, 就可以使用多态机制
+ 我们应该努力让一个类只分配一个责任.
+ 组合模式提供了一个结构, 可同时包容个别对象和组合对象.
+ 组合模式运行客户端对个别对象已经组合对象一视同仁.
+ 组合结构内的任意对象称为组件, 组件可以是组合或者叶节点.
+ 实现组合模式时, 有许多设计上的妥协, 你需要根据需要平衡透明性和安全性.





