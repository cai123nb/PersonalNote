# 类加载及执行子系统的案例与实战

## 概述

在 Class 文件格式与执行引擎这部分中, 用户的程序可以直接影响的内容并不太多, Class 文件以何种格式存储, 类型何时加载, 如何连接等等都是由虚拟机直接控制的行为, 用户程序无法对其进行改变. 可以通过程序进行操作的, 主要是字节码生成与类加载器这两部分的功能.

## 案例分析

### Tomcat: 正统的类加载器架构

主流的 Java Web 服务器, 如 Tomcat, Jetty, WebLogic, WebSphere 等主流的服务器, 都实现了自己定义的类加载器(一般不止一个). 因为一个功能健全的 Web 服务器, 要解决如下问题:

- 部署在同一个服务器上的两个 Web 应用服务器所使用的 Java 类库可以实现相互隔离. 两个不同的应用程序可能会依赖同一个第三方类库的不同版本, 不能要求一个类库在一个服务器上只有一份, 服务器应当保证两个应用程序的类库可以互相独立使用.
- 部署在同一个服务器上的两个 Web 应用所使用的 Java 类库可以互相共享. 如 10 个使用 Spring 组织的应用程序部署在同一台服务器上, 如果把 10 分 Spring 分别存放在各个应用程序的隔离目录上, 就会是很大的资源浪费: 如果类库不能共享, 虚拟机的方法区就会很容易出现过度膨胀的风险.
- 服务器尽可能保证自身安全不受部署的 Web 应用程序的影响. 服务器本身的类库依赖应该与应用程序类库相独立.
- 支持 JSP 应用的 Web 服务器, 大多数都需要支持 HotSwap 功能.

由于存在上述问题, 在部署 Web 应用时, 单独一个 ClassPath 就无法满足需求了, 所以各种服务器都"不约而同"地提供好几个 ClassPath 路径供用户存放第三方类库. 每一个目录都会有一个对应的自定义类加载器去加载放置在里面的 Java 类库.

如 Tomcat 的目录结构中, 有 3 组目录("/common/_", "/server/_"和"/shared/_")可以存放 Java 类库, 另外还可以加上 Web 应用程序自身的目录"/WEB-INF/_", 一共 4 组:

- 放在"/common"目录中: 类库可被 Tomcat 和所有的 Web 应用共同使用.
- 放在"/server"目录中: 类库可被 Tomcat 使用, 对所有的 Web 应用都不可见.
- 放在"/share"目录中: 类库可被所有的 Web 应用可见, 但对 Tomcat 自己不可见.
- 放在"/WebApp/WEB-INF"目录中: 类库只可以对此 Web 应用程序使用.

为了支持这套目录结构, 并对目录结构里面的类库进行加载和隔离, Tomcat 自定义了多个类加载器, 这些类加载器按照经典的双亲委派模型来实现.

![show](https://image.cjyong.com/blog/jvm/7.jpg)

CommonClassLoader, CatalinaClassLoader, SharedClassLoader 和 WebAppClassLoader 是 Tomcat 自定义的类加载器, 它们分别加载/common/_, /server/_, /share/*和/WebApp/WEB-INF/*中的 Java 类库. 其中 WebAppClassLoader 对应每一个 Web 应用程序, 每个引用程序代表具有一个实例. JsperLoader 则是对应每一个 JSP 文件, 每个 JSP 文件具有一个 JsperLoader 的实例(便于实现 HotSwap 功能, 每当 JSP 文件修改了, 替换当前的 JsperLoader 的实例, 并新建一个新的 Jsp 类加载器来实现 HotSwap).

### OSGi: 灵活的类加载器架构

OSGi(Open Service Gateway Initiative)是 OSGI 联盟(OSGi Alliance)制定的一个基于 Java 语言的动态模块化标准规范, 现在已经成为 Java 世界中"事实上"的模块化标准.

OSGi 中每个模块(称为 Bundle)与普通的 Java 类库区别不是很大, 两种一般都以 JAR 格式进行封装, 并且内部存储的是 Java Package 的 Class. 但是一个 Bundle 可以声明它所依赖的 Java Package(通过 Import-Package 描述), 也可以声明它允许导出发布的 Java Package(通过 Export-Package 描述). 这样提供了更加精确的模块划分和可见性控制: 一个模块里只有被 Export 过的 Package 才可能被外界访问, 其他的 Package 和 Class 将会隐藏起来. 另外 OSGi 的程序很可能实现模块级的热拔插功能, 当程序升级更新或调试除错时, 可以只停用, 重新安装然后启用程序中的一部分.

OSGi 实现功能主要依靠它灵活的类加载器架构. OSGi 的 Bundle 类加载器之间只有规则, 没有固定的委派关系. 例如, 某个 Bundle 声明了一个依赖的 Package, 如果其他的 Bundle 发布了这个 Package, 那么所有对这个 Pacakge 的类加载动作都会委派给发布它的 Bundle 类加载器来完成. OSGi 中的类加载中的查找行为和委派行为往往较为复杂, 具体的查找规则为:

- 以 java.\*开头的类, 委派给父类加载器加载.
- 否则, 委派列表名单内的类, 委派给父类加载器加载.
- 否则, Import 列表中的类, 委派给 Export 这个类的 Bundle 的类加载器加载.
- 否则, 查找当前 Bundle 的 Classpath, 使用自己的类加载器加载.
- 否则, 查找是否在自己的 Fragment Bundle 中, 如果是, 委派给 Fragment Bundle 的类加载器加载.
- 否则, 查找 Dynamic Import 列表中的 Bundle, 委派给对应的 Bundle 的类加载器加载.
- 否则, 查找失败.

总体来说, OSGi 描绘了一个很美好的模块化开发的目标, 并且定义了实现这个目标所需要的各种服务, 同时也有很成熟的框架提供支持. 对于单个虚拟机下的应用, 从开发初期就建立在 OSGi 上是一个不错的选择, 这样便于约束依赖. 但并非所有的应用都适合 OSGi 作为基础架构, OSGi 提供强大功能的同时, 也引入了额外的复杂度, 带来了线程死锁和内存泄漏的风险.

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

这里我们通过 Proxy.newProxyInstance 方法获得一个绑定对象的代理类, 然后我们调用代理类执行 sayHello()方法. 实际上代理类执行的方法为先输出"welcome"再调用方法类的方法. 我们这边查看一下 Proxy 的 newProxyInstance 方法.

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

而 Proxy 创建代理类的实质是通过 sun.misc.ProxyGenerator.generateProxyClass()方法来完成生成字节码的动作(主要是根据 Class 文件的格式进行拼装字节码), 这个方法返回一个描述代理类的字节码 byte[]数组. 我们在 main 方法中添加:

```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```

可以保留该代理类的字节码文件, \$Proxy0.class, 反编译后为:

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

代码比较简单, 首先先获取所有需要实现的方法(静态实现), 然后在这个方法的调用时, 调用 super.h.invoke()方法, 这里的 h 就是我们传递进来的 DynamicProxy 类的实例对象. 就是所有的方法都是通过我们设置的 invoke()函数进行执行.

### Retrotranslator: 跨越 JDK 版本

为了解决代码在 JDK 版本之间的跨越, 一种名为`Java逆向移植(Java BackPointing Tools)`的工具应运而生, 而 Retrotranslator 就是这类工具中较为出色的一个.

Retrotranslator 的主要作用就是将 JDK1.5 编译出来的 Class 文件转变为 JDK1.4 或 JDK1.3 上部署的版本, 它可以很好的支持自动装箱, 泛型, 动态注解, 枚举, 变长参数, 遍历循环, 静态导入这些语法特性, 甚至支持 JDK1.5 中新增的集合改进, 并发包以及对泛型, 注解等的反射操作.

每次 JDK 升级, 新增的功能主要分为以下 4 类:

- 在编译器层面做的改进. 如自动装箱拆箱, 实际上就是编译器在程序中使用到了包装对象的地方自动插入了许多 Integer.valueOf()子类的代码等等.
- 对 Java API 的代码增强. 如 JDK1.2 时代引入的 java.util.Collections 等一系列集合类等.
- 需要在字节码中进行支持的改动. 如 JDK1.7 新增的语法特性: 动态语言的支持, 就需要在虚拟机中新增一条 invokedynamic 字节码指令来配合完成相关的工作.
- 虚拟机内部的改进. 如 JDK1.5 中实现的 Java 内存模型, CMS 收集器子类的改动.

对于上述 4 种功能, Retrotranslator 只能模拟前 2 类, 对于后面两类直接在虚拟机内部进行的改进, 是没办法实现的. 对于上面的第 2 类比较简单, 一般引入独立的 jar 包来实现新的功能即可. 但是对于第一类, 则是使用 ASM 框架直接对字节码进行处理.
