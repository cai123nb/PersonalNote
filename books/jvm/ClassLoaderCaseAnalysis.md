# 类加载及执行子系统的案例与实战

## 概述
在Class文件格式与执行引擎这部分中, 用户的程序可以直接影响的内容并不太多,  Class文件以何种格式存储, 类型何时加载,  如何连接等等都是由虚拟机直接控制的行为, 用户程序无法对其进行改变. 可以通过程序进行操作的, 主要是字节码生成与类加载器这两部分的功能.

## 案例分析
### Tomcat: 正统的类加载器架构
主流的Java Web服务器, 如Tomcat, Jetty, WebLogic, WebSphere等主流的服务器, 都实现了自己定义的类加载器(一般不止一个). 因为一个功能健全的Web服务器, 要解决如下问题:
+ 部署在同一个服务器上的两个Web应用服务器所使用的Java类库可以实现相互隔离. 两个不同的应用程序可能会依赖同一个第三方类库的不同版本, 不能要求一个类库在一个服务器上只有一份, 服务器应当保证两个应用程序的类库可以互相独立使用.
+ 部署在同一个服务器上的两个Web应用所使用的Java类库可以互相共享. 如10个使用Spring组织的应用程序部署在同一台服务器上, 如果把10分Spring分别存放在各个应用程序的隔离目录上, 就会是很大的资源浪费: 如果类库不能共享, 虚拟机的方法区就会很容易出现过度膨胀的风险.
+ 服务器尽可能保证自身安全不受部署的Web应用程序的影响. 服务器本身的类库依赖应该与应用程序类库相独立.
+ 支持JSP应用的Web服务器, 大多数都需要支持HotSwap功能.

由于存在上述问题, 在部署Web应用时, 单独一个ClassPath就无法满足需求了, 所以各种服务器都"不约而同"地提供好几个ClassPath路径供用户存放第三方类库. 每一个目录都会有一个对应的自定义类加载器去加载放置在里面的Java类库.

如Tomcat的目录结构中, 有3组目录("/common/*", "/server/*"和"/shared/*")可以存放Java类库, 另外还可以加上Web应用程序自身的目录"/WEB-INF/*", 一共4组:
+ 放在"/common"目录中: 类库可被Tomcat和所有的Web应用共同使用.
+ 放在"/server"目录中: 类库可被Tomcat使用, 对所有的Web应用都不可见.
+ 放在"/share"目录中: 类库可被所有的Web应用可见, 但对Tomcat自己不可见.
+ 放在"/WebApp/WEB-INF"目录中: 类库只可以对此Web应用程序使用.

为了支持这套目录结构, 并对目录结构里面的类库进行加载和隔离, Tomcat自定义了多个类加载器, 这些类加载器按照经典的双亲委派模型来实现.

![show](https://image.cjyong.com/blog/jvm/7.jpg)

CommonClassLoader, CatalinaClassLoader, SharedClassLoader和WebAppClassLoader是Tomcat自定义的类加载器, 它们分别加载/common/*, /server/*, /share/*和/WebApp/WEB-INF/*中的Java类库. 其中WebAppClassLoader对应每一个Web应用程序, 每个引用程序代表具有一个实例. JsperLoader则是对应每一个JSP文件, 每个JSP文件具有一个JsperLoader的实例(便于实现HotSwap功能, 每当JSP文件修改了, 替换当前的JsperLoader的实例, 并新建一个新的Jsp类加载器来实现HotSwap).

### OSGi: 灵活的类加载器架构
OSGi(Open Service Gateway Initiative)是OSGI联盟(OSGi Alliance)制定的一个基于Java语言的动态模块化标准规范, 现在已经成为Java世界中"事实上"的模块化标准.

OSGi中每个模块(称为Bundle)与普通的Java类库区别不是很大, 两种一般都以JAR格式进行封装, 并且内部存储的是Java Package的Class. 但是一个Bundle可以声明它所依赖的Java Package(通过Import-Package描述), 也可以声明它允许导出发布的Java Package(通过Export-Package描述). 这样提供了更加精确的模块划分和可见性控制: 一个模块里只有被Export过的Package才可能被外界访问, 其他的Package和Class将会隐藏起来. 另外OSGi的程序很可能实现模块级的热拔插功能, 当程序升级更新或调试除错时, 可以只停用, 重新安装然后启用程序中的一部分.

OSGi实现功能主要依靠它灵活的类加载器架构. OSGi的Bundle类加载器之间只有规则, 没有固定的委派关系. 例如, 某个Bundle声明了一个依赖的Package, 如果其他的Bundle发布了这个Package, 那么所有对这个Pacakge的类加载动作都会委派给发布它的Bundle类加载器来完成. OSGi中的类加载中的查找行为和委派行为往往较为复杂, 具体的查找规则为:
+ 以java.*开头的类, 委派给父类加载器加载.
+ 否则, 委派列表名单内的类, 委派给父类加载器加载.
+ 否则, Import列表中的类, 委派给Export这个类的Bundle的类加载器加载.
+ 否则, 查找当前Bundle的Classpath, 使用自己的类加载器加载.
+ 否则, 查找是否在自己的Fragment Bundle中, 如果是, 委派给Fragment Bundle的类加载器加载.
+ 否则, 查找Dynamic Import列表中的Bundle, 委派给对应的Bundle的类加载器加载.
+ 否则, 查找失败.

总体来说, OSGi描绘了一个很美好的模块化开发的目标, 并且定义了实现这个目标所需要的各种服务, 同时也有很成熟的框架提供支持. 对于单个虚拟机下的应用, 从开发初期就建立在OSGi上是一个不错的选择, 这样便于约束依赖. 但并非所有的应用都适合OSGi作为基础架构, OSGi提供强大功能的同时, 也引入了额外的复杂度, 带来了线程死锁和内存泄漏的风险.

### 字节码生成技术与动态代理的实现

```java
public class DynamicProxyTest {
    interface IHello {
        void sayHello();
    }

    static class Hello implements IHello {
        @Override
        public void sayHello() {
            System.out.println("hello world");
        }
    }

    static class DynamicProxy implements InvocationHandler {

        Object originalObj;

        Object bind(Object originalObj) {
            this.originalObj = originalObj;
            return Proxy.newProxyInstance(originalObj.getClass().getClassLoader(),
                    originalObj.getClass().getInterfaces(),
                    this);
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("welcome");
            return method.invoke(originalObj, args);
        }
    }

    public static void main(String[] args) {
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        IHello hello = (IHello) new DynamicProxy().bind(new Hello());
        hello.sayHello();
        //welcome
        //hello world
    }
}
```

这里我们通过Proxy.newProxyInstance方法获得一个绑定对象的代理类, 然后我们调用代理类执行sayHello()方法. 实际上代理类执行的方法为先输出"welcome"再调用方法类的方法. 我们这边查看一下Proxy的newProxyInstance方法.

```java
/**
     * Returns an instance of a proxy class for the specified interfaces
     * that dispatches method invocations to the specified invocation
     * handler.
     *
     * <p>{@code Proxy.newProxyInstance} throws
     * {@code IllegalArgumentException} for the same reasons that
     * {@code Proxy.getProxyClass} does.
     *
     * @param   loader the class loader to define the proxy class
     * @param   interfaces the list of interfaces for the proxy class
     *          to implement
     * @param   h the invocation handler to dispatch method invocations to
     * @return  a proxy instance with the specified invocation handler of a
     *          proxy class that is defined by the specified class loader
     *          and that implements the specified interfaces
     * @throws  IllegalArgumentException if any of the restrictions on the
     *          parameters that may be passed to {@code getProxyClass}
     *          are violated
     * @throws  SecurityException if a security manager, <em>s</em>, is present
     *          and any of the following conditions is met:
     *          <ul>
     *          <li> the given {@code loader} is {@code null} and
     *               the caller's class loader is not {@code null} and the
     *               invocation of {@link SecurityManager#checkPermission
     *               s.checkPermission} with
     *               {@code RuntimePermission("getClassLoader")} permission
     *               denies access;</li>
     *          <li> for each proxy interface, {@code intf},
     *               the caller's class loader is not the same as or an
     *               ancestor of the class loader for {@code intf} and
     *               invocation of {@link SecurityManager#checkPackageAccess
     *               s.checkPackageAccess()} denies access to {@code intf};</li>
     *          <li> any of the given proxy interfaces is non-public and the
     *               caller class is not in the same {@linkplain Package runtime package}
     *               as the non-public interface and the invocation of
     *               {@link SecurityManager#checkPermission s.checkPermission} with
     *               {@code ReflectPermission("newProxyInPackage.{package name}")}
     *               permission denies access.</li>
     *          </ul>
     * @throws  NullPointerException if the {@code interfaces} array
     *          argument or any of its elements are {@code null}, or
     *          if the invocation handler, {@code h}, is
     *          {@code null}
     */
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs); //创建代理类

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            //通过调用代理类的构造方法, 传递this指针(即DynamicProxy类的实例)
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

而Proxy创建代理类的实质是通过sun.misc.ProxyGenerator.generateProxyClass()方法来完成生成字节码的动作(主要是根据Class文件的格式进行拼装字节码), 这个方法返回一个描述代理类的字节码byte[]数组. 我们在main方法中添加:

```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

可以保留该代理类的字节码文件, $Proxy0.class, 反编译后为:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.cjyong.jvm.classloader;

import com.cjyong.jvm.classloader.DynamicProxyTest.IHello;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

final class $Proxy0 extends Proxy implements IHello {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.cjyong.jvm.classloader.DynamicProxyTest$IHello").getMethod("sayHello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

代码比较简单, 首先先获取所有需要实现的方法(静态实现), 然后在这个方法的调用时, 调用super.h.invoke()方法, 这里的h就是我们传递进来的DynamicProxy类的实例对象. 就是所有的方法都是通过我们设置的invoke()函数进行执行.

### Retrotranslator: 跨越JDK版本

为了解决代码在JDK版本之间的跨越, 一种名为`Java逆向移植(Java BackPointing Tools)`的工具应运而生, 而Retrotranslator就是这类工具中较为出色的一个.

Retrotranslator的主要作用就是将JDK1.5编译出来的Class文件转变为JDK1.4或JDK1.3上部署的版本, 它可以很好的支持自动装箱, 泛型, 动态注解, 枚举, 变长参数, 遍历循环, 静态导入这些语法特性, 甚至支持JDK1.5中新增的集合改进, 并发包以及对泛型, 注解等的反射操作. 

每次JDK升级, 新增的功能主要分为以下4类:
+ 在编译器层面做的改进. 如自动装箱拆箱, 实际上就是编译器在程序中使用到了包装对象的地方自动插入了许多Integer.valueOf()子类的代码等等.
+ 对Java API的代码增强. 如JDK1.2时代引入的java.util.Collections等一系列集合类等.
+ 需要在字节码中进行支持的改动. 如JDK1.7新增的语法特性: 动态语言的支持, 就需要在虚拟机中新增一条invokedynamic字节码指令来配合完成相关的工作.
+ 虚拟机内部的改进. 如JDK1.5中实现的Java内存模型, CMS收集器子类的改动.

对于上述4种功能, Retrotranslator只能模拟前2类, 对于后面两类直接在虚拟机内部进行的改进, 是没办法实现的. 对于上面的第2类比较简单, 一般引入独立的jar包来实现新的功能即可. 但是对于第一类, 则是使用ASM框架直接对字节码进行处理.





