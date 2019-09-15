# Class

JDK1.8:

```java
package java.lang;

//类(Class)的实例对象代表在Java程序中运行着的类和接口. 枚举也是一种类, 注解是
//一种接口. 每个数组都属于一个类, 反射为一个类对象, 这个类对象被所有具有相同
//维度和元素的数组共享. Java原始数据类型(boolean,byte,char,short,int,long,
//float,double)和关键字void也是一种类对象. Class没有公开的构造函数,而是由JVM
//在加载对象时自动调用defineClass方法为对象创建. 获取一个类的名称:obj.getClass
//().getName(). 同样也可以直接通过类名直接获得: Foo.class.getName().
public final class Class<T> implements java.io.Serializable,
                              GenericDeclaration,
                              Type,
                              AnnotatedElement {
   private static final int ANNOTATION= 0x00002000;
   private static final int ENUM      = 0x00004000;
   private static final int SYNTHETIC = 0x00001000;

   //为本类的本地方法进行注册,放在这里保证首先执行
   private static native void registerNatives();
   static {
         registerNatives();
   }

   //私有的构造函数,防止被创建(这样也就没有了默认构造函数).
   private Class(ClassLoader loader) {
         // Initialize final field for classLoader.  The initialization value of non-null
         // prevents future JIT optimizations from assuming this final field is null.
         classLoader = loader;
   }
   //该字段是在JVM虚拟机中进行初始化, 而不是通过上面私有构造函数初始化,并且该
   //字段无法通过反射(Reflection)获得, 即getDeclaredField将会抛出异常
   private final ClassLoader classLoader;

   //将该对象转换为string类型, 首先给出是否是接口,类还是原始类型,然后加上Java的特定命名规范
   //getName()函数生成的类限定描述符.
   public String toString() {
      return (isInterface() ? "interface " : (isPrimitive() ? "" : "class "))
         + getName();
   }

   //返回实体(类, 接口,数组,原始类型,void)的名字来代表这个对象. 命名具有特定的规范
   //如果是原始类型或者void则返回对应的String名字(如:int.class.getName0 -- int). 如果是
   //对象的一个引用,则返回对象的全称(包名+类名)(如: java.lang.String;cjyong.ABC$DEF,内部
   //类使用$区分,可以叠加). 如果是一个数组的话,先使用[代表维度, 一个[代表一维,依次叠加,
   //右边使用对象的限定名: Z-boolean,B-byte,C-char,L-类,D-double,F-float,I-int,J-long,S-
   //short. 记得类(L)后面需要加上类的全称名字(如new boolean[2][3][4].getClass().getName()-
   //[[[Z, new String[3]-[Ljava.lang.String).
   public String getName() {
      String name = this.name;
      if (name == null)
         this.name = name = getName0();
      return name;
   }

   //缓存类限定描述名,减少调用getName的次数
   private transient String name; //添加transient,不包括在序列化字段中
   private native String getName0();

   //判断是否是8种原始类型(int,long,byte,char,float,double,short,boolean)或者void.
   public native boolean isPrimitive();

   //返回类的修饰符,用数字编码
   //public:0x00000001,private:0x00000002,protected:0x00000004,static:0x00000008,final:
   //0x00000010,abstract:0x00000400,strict:0x00000800. 还有很多别的修饰符,但是用于类的
   //只有这7种.
   //注意在类中,如果一个类是数组, 那么它的public,private和protected状态和数组元素一样,
   //如果一个类似原始类型或者void,那它的public一直为true, protected,private一直为false,
   //如果是一个原始类型的数组,那么它的final修饰符也一直为true,接口修饰符为false. 编码定
   //义位于JVM规范Table4.1
   public native int getModifiers();

   //返回一个TypeVariable对象数组，表示由此GenericDeclaration对象表示的泛型声明声明的
   //类型变量，按声明顺序。 如果基础泛型声明未声明类型变量，则返回长度
   //为0的数组。
   @SuppressWarnings("unchecked")
   public TypeVariable<Class<T>>[] getTypeParameters() {
      ClassRepository info = getGenericInfo();
      if (info != null)
         return (TypeVariable<Class<T>>[])info.getTypeParameters();
      else
         return (TypeVariable<Class<T>>[])new TypeVariable<?>[0];
   }
   //通用信息库的访问入口,通常懒加载
   private ClassRepository getGenericInfo() {
      ClassRepository genericInfo = this.genericInfo;
      if (genericInfo == null) {
         String signature = getGenericSignature0();
         if (signature == null) {
            genericInfo = ClassRepository.NONE;
         } else {
            genericInfo = ClassRepository.make(signature, getFactory());
         }
         this.genericInfo = genericInfo;
      }
      return (genericInfo != ClassRepository.NONE) ? genericInfo : null;
   }
   // Generic signature handling
   private native String getGenericSignature0();
   // Generic info repository; lazily initialized
   private volatile transient ClassRepository genericInfo;

   //Add by 1.8: 返回一个类更加详细的描述包括修饰符信息和类型参数(泛型). 如果是原始类型
   //(int,long,byte,char,float,double,short,boolean)和void就直接调用toString()方法. 否则
   //首先标识出修饰符(private,public,final,protected,abstract,static,strict),如果有多
   //个使用空格间隔开,然后加上类的全称限定(getName()), 最后加上类型参数(泛型中类的类型参数)
   //如: Main类子类: private class Test<T extends Number,U> -- private class Main$Test<T,U>)
   public String toGenericString() {
      if (isPrimitive()) {
         return toString();
      } else {
         StringBuilder sb = new StringBuilder();

         // Class modifiers are a superset of interface modifiers
         int modifiers = getModifiers() & Modifier.classModifiers();
         if (modifiers != 0) {
            sb.append(Modifier.toString(modifiers));
            sb.append(' ');
         }

         if (isAnnotation()) {
            sb.append('@');
         }
         if (isInterface()) { // Note: all annotation types are interfaces
            sb.append("interface");
         } else {
            if (isEnum())
               sb.append("enum");
            else
               sb.append("class");
         }
         sb.append(' ');
         sb.append(getName());

         TypeVariable<?>[] typeparms = getTypeParameters();
         if (typeparms.length > 0) {
            boolean first = true;
            sb.append('<');
            for(TypeVariable<?> typeparm: typeparms) {
               if (!first)
                  sb.append(',');
               sb.append(typeparm.getTypeName());
               first = false;
            }
            sb.append('>');
         }

         return sb.toString();
      }
   }

   //返回指定对象的(className)的Class对象, 调用这个方法类似于: Class.forName(className, true,
   //currentLoader). 其中使用的类加载器是使用当前调用者的类加载器. 如(Class t = Class.
   //forName("java.lang.Thread"). 返回Thread的类描述对象. 调用该方法, 会初始化传递的类对象.
   @CallerSensitive
   public static Class<?> forName(String className)
            throws ClassNotFoundException {
      Class<?> caller = Reflection.getCallerClass();
      return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
   }

   //使用指定的类加载器,返回指定名称对象的Class对象.给定的名称应该是全称, 和类对象的getName()
   //返回的名称一样. 如果传递的类加载器是null, 则使用bootstrap类加载器. 类初始化的情况只有当
   //类没有被初始化且initialize参数为true的情况. 如果传递过来的name是原始类型或者void, 类加载
   //器会尝试在未命名包中查找类名为name的类对象, 所以该方法不能用于原始数据类型或void.
   //如果传递的名称是一个数组对象, 数组内元素类会被加载但是不会被初始化. 如果传递的loader为null
   //并且安全管理器不为空, 调用者类加载器不为空, 然后这个方法会调用安全管理器进行权限校验.

   @CallerSensitive
   public static Class<?> forName(String name, boolean initialize,
                           ClassLoader loader)
      throws ClassNotFoundException
   {
      Class<?> caller = null;
      SecurityManager sm = System.getSecurityManager();
      if (sm != null) {
         // Reflective call to get caller class is only needed if a security manager
         // is present.  Avoid the overhead of making this call otherwise.
         caller = Reflection.getCallerClass();
         if (sun.misc.VM.isSystemDomainLoader(loader)) {
            ClassLoader ccl = ClassLoader.getClassLoader(caller);
            if (!sun.misc.VM.isSystemDomainLoader(ccl)) {
               sm.checkPermission(
                  SecurityConstants.GET_CLASSLOADER_PERMISSION);
            }
         }
      }
      return forName0(name, initialize, loader, caller);
   }


   //经过系统加载安全检测之后方可调用
   private static native Class<?> forName0(String name, boolean initialize,
                                 ClassLoader loader,
                                 Class<?> caller)
      throws ClassNotFoundException;

   //创建一个class对象的实例, 实例化的过程如同调用new的无参数构造函数.如果
   //class对象还没被初始化的话, class对象也会被实例化. 注意这个方法会传播无
   //参数构造函数的异常(包括检查异常), 使用这个方法可以捕获异常进行处理.
   @CallerSensitive
   public T newInstance()
      throws InstantiationException, IllegalAccessException
   {
      if (System.getSecurityManager() != null) {
         checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);
      }

      // NOTE: the following code may not be strictly correct under
      // the current Java memory model.

      // Constructor lookup
      if (cachedConstructor == null) {
         if (this == Class.class) {
            throw new IllegalAccessException(
               "Can not call newInstance() on the Class for java.lang.Class"
            );
         }
         try {
            Class<?>[] empty = {};
            final Constructor<T> c = getConstructor0(empty, Member.DECLARED);
            // Disable accessibility checks on the constructor
            // since we have to do the security check here anyway
            // (the stack depth is wrong for the Constructor's
            // security check to work)
            java.security.AccessController.doPrivileged(
               new java.security.PrivilegedAction<Void>() {
                  public Void run() {
                        c.setAccessible(true);
                        return null;
                     }
                  });
            cachedConstructor = c;
         } catch (NoSuchMethodException e) {
            throw (InstantiationException)
               new InstantiationException(getName()).initCause(e);
         }
      }
      Constructor<T> tmpConstructor = cachedConstructor;
      // Security check (same as in java.lang.reflect.Constructor)
      int modifiers = tmpConstructor.getModifiers();
      if (!Reflection.quickCheckMemberAccess(this, modifiers)) {
         Class<?> caller = Reflection.getCallerClass();
         if (newInstanceCallerCache != caller) {
            Reflection.ensureMemberAccess(caller, this, null, modifiers);
            newInstanceCallerCache = caller;
         }
      }
      // Run constructor
      try {
         return tmpConstructor.newInstance((Object[])null);
      } catch (InvocationTargetException e) {
         Unsafe.getUnsafe().throwException(e.getTargetException());
         // Not reached
         return null;
      }
   }
   //对构造函数和实例进行缓存
   private volatile transient Constructor<T> cachedConstructor;
   private volatile transient Class<?>       newInstanceCallerCache;

   //判断特定的对象(obj)是否是某一个类(Class)的实例. 等价于instanceof操作,
   //当传递的对象可以转换成该类,返回true, 否则返回false. 如果传递的对象是
   //数组对象, 需要保证数组的维度和元素类型相同(维度相同:同纬度即可, 不需要
   //维度内长度相同)返回true.如果是接口的话, 传递的对象实现了接口,返回true.
   //如:String.class.isInstance("ABC"); true
   public native boolean isInstance(Object obj);

   //判断该Class对象是否是cls或者是cls的父类. 如果cls是原始类型,只有在Object
   //时返回true, 否则返回false.
   public native boolean isAssignableFrom(Class<?> cls);

   //返回一个Class对象是否是接口类型
   public native boolean isInterface();

   //返回一个Class对象是否是数组类型
   public native boolean isArray();

   //返回一个Class对象是否是原始类型(8种元素类型和void,注意(Integer(包装类)不是
   //,int才是)
   public native boolean isPrimitive();

   //返回一个Class对象是否是注解类型(所有的注解都是接口, 接口以一定为true)
   public boolean isAnnotation() {
      return (getModifiers() & ANNOTATION) != 0;
   }

   //判断一个Class对象是否是合成类.(JVM虚拟机有时候为了满足某些特殊需求，自动
   //添加一些类作为辅助. 如外部表访问私有内部类的成员变量,Enum的底层实现)
   public boolean isSynthetic() {
      return (getModifiers() & SYNTHETIC) != 0;
   }

   //返回一个Class代表对象的父类. 如果是Object,interface,primitive type, void
   //则会返回null. 如果是数组类型调用则会返回java.lang.Object对象.
   public native Class<? super T> getSuperclass();

   //返回class对象的所有的直接的父类(即父类的父类也会显示,直到Object). 如果父类是
   //含有具体参数化类型(泛型),返回的父类应该包含相应的参数信息.如果是Object,interface,
   //primitive type, void则会返回null. 如果是数组类型调用则会返回java.lang.Object对象.
   public Type getGenericSuperclass() {
      ClassRepository info = getGenericInfo();
      if (info == null) {
         return getSuperclass();
      }

      // Historical irregularity:
      // Generic signature marks interfaces with superclass = Object
      // but this API returns null for interfaces
      if (isInterface()) {
         return null;
      }

      return info.getSuperclass();
   }

   //返回Class对象的包信息, 类加载器用来寻找包信息. 如果是使用bootstrap类加载器进行
   //加载的类, 会从CLASSPATH开始搜寻对应的包信息. 如果创建时没有包信息,返回NULL.
   //包可以包含版本和规范信息, 只有在信息被定义在和类一起的manifests中, 并且类加载器
   //创建包实例的时候使用到了这些信息.
   public Package getPackage() {
      return Package.getPackage(this);
   }

   //返回该类实现的所有接口详情(以数组的形式返回). 数组中的顺序与类实现的接口顺序一样.
   //如果该Class对象是一个接口, 则按照继承链进行获取. 如果没有实现接口,则返回0长度的
   //数组. 如果是数组对象, 则返回Cloneable接口和Serializeable接口.
   public Class<?>[] getInterfaces() {
      ReflectionData<T> rd = reflectionData();
      if (rd == null) {
         // no cloning required
         return getInterfaces0();
      } else {
         Class<?>[] interfaces = rd.interfaces;
         if (interfaces == null) {
            interfaces = getInterfaces0();
            rd.interfaces = interfaces;
         }
         // defensively copy before handing over to user code
         return interfaces.clone();
      }
   }
   private native Class<?>[] getInterfaces0();

   //返回该Class对象实现的所有接口信息, 如果接口包含了具体的参数类型(泛型),需要
   //按照代码中详情进行输出, 详情参照getInterfaces()和getGenericInterfaces().
   public Type[] getGenericInterfaces() {
      ClassRepository info = getGenericInfo();
      return (info == null) ?  getInterfaces() : info.getSuperInterfaces();
   }

   //返回数组内的元素类型, 如果传递的对象不是数组对象则返回null.
   public native Class<?> getComponentType();

   //返回java语言的修饰符, 使用一个int数据进行编码显示.详细定义可参考java.lang.
   //reflect.Modifier类.如果传递的对象是一个数组类,那么它的public,private和
   //protected修饰符等价于数组内的元素类型. 如果对象是原始类型或者void那么它的
   //public始终为true, protected和private始终为false. 其中数组类和原始类型或者void
   //final修饰符始终为true, 接口始终为false.
   public native int getModifiers();

   //返回一个类的签名,如果Class对象是原始类型或者void或者类没有签名则返回null.
   public native Object[] getSigners();

   //设置一个类的签名
   native void setSigners(Object[] signers);

   //用于支持反射的调用
   // 缓存反射的结果
   private static boolean useCaches = true;

   //这些反射数据可能失效,当JVM调用RedefineClasses()之后
   private static class ReflectionData<T> {
      volatile Field[] declaredFields;
      volatile Field[] publicFields;
      volatile Method[] declaredMethods;
      volatile Method[] publicMethods;
      volatile Constructor<T>[] declaredConstructors;
      volatile Constructor<T>[] publicConstructors;
      // Intermediate results for getFields and getMethods
      volatile Field[] declaredPublicFields;
      volatile Method[] declaredPublicMethods;
      volatile Class<?>[] interfaces;

      // Value of classRedefinedCount when we created this ReflectionData instance
      final int redefinedCount;

      ReflectionData(int redefinedCount) {
         this.redefinedCount = redefinedCount;
      }
   }

   //本地缓存的反射数据(添加volatile,防止指令重排和保证线程可见性,添加transient防止序列化)
   private volatile transient SoftReference<ReflectionData<T>> reflectionData;

   //当JVM调用RedefineClasses()函数时,JVM自动增长该值.
   private volatile transient int classRedefinedCount = 0;

   //懒加载和缓存反射数据
   private ReflectionData<T> reflectionData() {
      SoftReference<ReflectionData<T>> reflectionData = this.reflectionData;
      int classRedefinedCount = this.classRedefinedCount;
      ReflectionData<T> rd;
      if (useCaches &&
         reflectionData != null &&
         (rd = reflectionData.get()) != null &&
         rd.redefinedCount == classRedefinedCount) {
         return rd;
      }
      // else no SoftReference or cleared SoftReference or stale ReflectionData
      // -> create and replace new instance
      return newReflectionData(reflectionData, classRedefinedCount);
   }

   //创建新的反射的数据
   private ReflectionData<T> newReflectionData(SoftReference<ReflectionData<T>> oldReflectionData,
                                    int classRedefinedCount) {
      if (!useCaches) return null;

      while (true) {
         //创建新的反射数据
         ReflectionData<T> rd = new ReflectionData<>(classRedefinedCount);
         // try to CAS it...
         if (Atomic.casReflectionData(this, oldReflectionData, new SoftReference<>(rd))) {
            return rd;
         }
         // else retry 清除旧版不相同的数据
         oldReflectionData = this.reflectionData;
         classRedefinedCount = this.classRedefinedCount;
         if (oldReflectionData != null &&
            (rd = oldReflectionData.get()) != null &&
            rd.redefinedCount == classRedefinedCount) {
            return rd;
         }
      }
   }

   //如果一个内部类或者匿名类是在一个方法中定义的, 返回该方法的定义. 否则返回null.
   @CallerSensitive
   public Method getEnclosingMethod() throws SecurityException {
      EnclosingMethodInfo enclosingInfo = getEnclosingMethodInfo();

      if (enclosingInfo == null)
         return null;
      else {
         if (!enclosingInfo.isMethod())
            return null;

         MethodRepository typeInfo = MethodRepository.make(enclosingInfo.getDescriptor(),
                                               getFactory());
         Class<?>   returnType       = toClass(typeInfo.getReturnType());
         Type []    parameterTypes   = typeInfo.getParameterTypes();
         Class<?>[] parameterClasses = new Class<?>[parameterTypes.length];

         // Convert Types to Classes; returned types *should*
         // be class objects since the methodDescriptor's used
         // don't have generics information
         for(int i = 0; i < parameterClasses.length; i++)
            parameterClasses[i] = toClass(parameterTypes[i]);

         // Perform access check
         Class<?> enclosingCandidate = enclosingInfo.getEnclosingClass();
         enclosingCandidate.checkMemberAccess(Member.DECLARED,
                                     Reflection.getCallerClass(), true);
         /*
          * Loop over all declared methods; match method name,
          * number of and type of parameters, *and* return
          * type.  Matching return type is also necessary
          * because of covariant returns, etc.
          */
         for(Method m: enclosingCandidate.getDeclaredMethods()) {
            if (m.getName().equals(enclosingInfo.getName()) ) {
               Class<?>[] candidateParamClasses = m.getParameterTypes();
               if (candidateParamClasses.length == parameterClasses.length) {
                  boolean matches = true;
                  for(int i = 0; i < candidateParamClasses.length; i++) {
                     if (!candidateParamClasses[i].equals(parameterClasses[i])) {
                        matches = false;
                        break;
                     }
                  }

                  if (matches) { // finally, check return type
                     if (m.getReturnType().equals(returnType) )
                        return m;
                  }
               }
            }
         }

         throw new InternalError("Enclosing method not found");
      }
   }
   private native Object[] getEnclosingMethod0();
   private EnclosingMethodInfo getEnclosingMethodInfo() {
      Object[] enclosingInfo = getEnclosingMethod0();
      if (enclosingInfo == null)
         return null;
      else {
         return new EnclosingMethodInfo(enclosingInfo);
      }
   }
   private final static class EnclosingMethodInfo {
      private Class<?> enclosingClass;
      private String name;
      private String descriptor;

      private EnclosingMethodInfo(Object[] enclosingInfo) {
         if (enclosingInfo.length != 3)
            throw new InternalError("Malformed enclosing method information");
         try {
            // The array is expected to have three elements:

            // the immediately enclosing class
            enclosingClass = (Class<?>) enclosingInfo[0];
            assert(enclosingClass != null);

            // the immediately enclosing method or constructor's
            // name (can be null).
            name            = (String)   enclosingInfo[1];

            // the immediately enclosing method or constructor's
            // descriptor (null iff name is).
            descriptor      = (String)   enclosingInfo[2];
            assert((name != null && descriptor != null) || name == descriptor);
         } catch (ClassCastException cce) {
            throw new InternalError("Invalid type in enclosing method information", cce);
         }
      }
            boolean isPartial() {
         return enclosingClass == null || name == null || descriptor == null;
      }

      boolean isConstructor() { return !isPartial() && "<init>".equals(name); }

      boolean isMethod() { return !isPartial() && !isConstructor() && !"<clinit>".equals(name); }

      Class<?> getEnclosingClass() { return enclosingClass; }

      String getName() { return name; }

      String getDescriptor() { return descriptor; }
   }
   private static Class<?> toClass(Type o) {
      if (o instanceof GenericArrayType)
         return Array.newInstance(toClass(((GenericArrayType)o).getGenericComponentType()),
                            0)
            .getClass();
      return (Class<?>)o;
   }

   //如果一个内部类或者匿名类是在一个构造函数中定义的, 返回该构造函数的定义. 否则返回null.
   @CallerSensitive
   public Constructor<?> getEnclosingConstructor() throws SecurityException {
      EnclosingMethodInfo enclosingInfo = getEnclosingMethodInfo();

      if (enclosingInfo == null)
         return null;
      else {
         if (!enclosingInfo.isConstructor())
            return null;

         ConstructorRepository typeInfo = ConstructorRepository.make(enclosingInfo.getDescriptor(),
                                                      getFactory());
         Type []    parameterTypes   = typeInfo.getParameterTypes();
         Class<?>[] parameterClasses = new Class<?>[parameterTypes.length];

         // Convert Types to Classes; returned types *should*
         // be class objects since the methodDescriptor's used
         // don't have generics information
         for(int i = 0; i < parameterClasses.length; i++)
            parameterClasses[i] = toClass(parameterTypes[i]);

         // Perform access check
         Class<?> enclosingCandidate = enclosingInfo.getEnclosingClass();
         enclosingCandidate.checkMemberAccess(Member.DECLARED,
                                     Reflection.getCallerClass(), true);
         /*
          * Loop over all declared constructors; match number
          * of and type of parameters.
          */
         for(Constructor<?> c: enclosingCandidate.getDeclaredConstructors()) {
            Class<?>[] candidateParamClasses = c.getParameterTypes();
            if (candidateParamClasses.length == parameterClasses.length) {
               boolean matches = true;
               for(int i = 0; i < candidateParamClasses.length; i++) {
                  if (!candidateParamClasses[i].equals(parameterClasses[i])) {
                     matches = false;
                     break;
                  }
               }

               if (matches)
                  return c;
            }
         }

         throw new InternalError("Enclosing constructor not found");
      }
   }

   //如果Class对象(类或者接口)是在另一个类的内部定义的(内部类),返回其外部类的名称.如
   //果不是则返回null, 如果该对象是数组,原始类型,void返回null.
   @CallerSensitive
   public Class<?> getDeclaringClass() throws SecurityException {
      final Class<?> candidate = getDeclaringClass0();

      if (candidate != null)
         candidate.checkPackageAccess(
               ClassLoader.getClassLoader(Reflection.getCallerClass()), true);
      return candidate;
   }
   private native Class<?> getDeclaringClass0();

   //等同于上面的方法,差别在于: 在Java中匿名内部类是不属于外部类的成员函数, 上面方法
   //是无法获得其外部类的信息,但是如果使用这个方法是可以的. 常见于方法内部定义的匿名类
   //如:
   /*
   public class Test {
      public static void main(String[] args) {
         new Object() {
            public void test() {
               System.out.println(this.getClass().getDeclaringClass()); //null
               System.out.println(this.getClass().getEnclosingClass()); //not null
            }
         }.test();
      }
   }
   */
   @CallerSensitive
   public Class<?> getEnclosingClass() throws SecurityException {
      // There are five kinds of classes (or interfaces):
      // a) Top level classes
      // b) Nested classes (static member classes)
      // c) Inner classes (non-static member classes)
      // d) Local classes (named classes declared within a method)
      // e) Anonymous classes


      // JVM Spec 4.8.6: A class must have an EnclosingMethod
      // attribute if and only if it is a local class or an
      // anonymous class.
      EnclosingMethodInfo enclosingInfo = getEnclosingMethodInfo();
      Class<?> enclosingCandidate;

      if (enclosingInfo == null) {
         // This is a top level or a nested class or an inner class (a, b, or c)
         enclosingCandidate = getDeclaringClass();
      } else {
         Class<?> enclosingClass = enclosingInfo.getEnclosingClass();
         // This is a local class or an anonymous class (d or e)
         if (enclosingClass == this || enclosingClass == null)
            throw new InternalError("Malformed enclosing method information");
         else
            enclosingCandidate = enclosingClass;
      }

      if (enclosingCandidate != null)
         enclosingCandidate.checkPackageAccess(
               ClassLoader.getClassLoader(Reflection.getCallerClass()), true);
      return enclosingCandidate;
   }
   //返回是否是ASCII码数字
   private static boolean isAsciiDigit(char c) {
      return '0' <= c && c <= '9';
   }

   //返回Class对象的简单名称(不含包名), 如果是匿名类, 则返回空字符串.如果是数组,
   //在后面添加[].
   public String getSimpleName() {
      if (isArray())
         return getComponentType().getSimpleName()+"[]";

      String simpleName = getSimpleBinaryName();
      if (simpleName == null) { // top level class
         simpleName = getName();
         return simpleName.substring(simpleName.lastIndexOf(".")+1); // strip the package name
      }
      // According to JLS3 "Binary Compatibility" (13.1) the binary
      // name of non-package classes (not top level) is the binary
      // name of the immediately enclosing class followed by a '$' followed by:
      // (for nested and inner classes): the simple name.
      // (for local classes): 1 or more digits followed by the simple name.
      // (for anonymous classes): 1 or more digits.

      // Since getSimpleBinaryName() will strip the binary name of
      // the immediatly enclosing class, we are now looking at a
      // string that matches the regular expression "\$[0-9]*"
      // followed by a simple name (considering the simple of an
      // anonymous class to be the empty string).

      // Remove leading "\$[0-9]*" from the name
      int length = simpleName.length();
      if (length < 1 || simpleName.charAt(0) != '$')
         throw new InternalError("Malformed class name");
      int index = 1;
      while (index < length && isAsciiDigit(simpleName.charAt(index)))
         index++;
      // Eventually, this is the empty string iff this is an anonymous class
      return simpleName.substring(index);
   }
   //返回类型的字符描述,数组的根据维度多少在后面添加多少个"[]"
   public String getTypeName() {
      if (isArray()) {
         try {
            Class<?> cl = this;
            int dimensions = 0;
            while (cl.isArray()) {
               dimensions++;
               cl = cl.getComponentType();
            }
            StringBuilder sb = new StringBuilder();
            sb.append(cl.getName());
            for (int i = 0; i < dimensions; i++) {
               sb.append("[]");
            }
            return sb.toString();
         } catch (Throwable e) { /*FALLTHRU*/ }
      }
      return getName();
   }

   //返回类的典范名称(在Java Language Specification有详细定义). 如果是匿名类
   //数组对象返回null.
   public String getCanonicalName() {
      if (isArray()) {
         String canonicalName = getComponentType().getCanonicalName();
         if (canonicalName != null)
            return canonicalName + "[]";
         else
            return null;
      }
      if (isLocalOrAnonymousClass())
         return null;
      Class<?> enclosingClass = getEnclosingClass();
      if (enclosingClass == null) { // top level class
         return getName();
      } else {
         String enclosingName = enclosingClass.getCanonicalName();
         if (enclosingName == null)
            return null;
         return enclosingName + "." + getSimpleName();
      }
   }

   //判断当前Class对象是否是匿名类,如:
   /*
   Runnable x = new Runnable() {
      @Override
      public void run() {
         System.out.println(this.getClass());
      }
   };
   x 就为一个匿名类的实例
   */
   public boolean isAnonymousClass() {
      return "".equals(getSimpleName());
   }

   //判断Class对象是否是一个本地类(有类名,但是定义实在block内,如在方法
   //内部,body内部等)如:
   public boolean isLocalClass() {
      return isLocalOrAnonymousClass() && !isAnonymousClass();
   }
   //返回当前类是否是本地或者匿名类
   private boolean isLocalOrAnonymousClass() {
      // JVM Spec 4.8.6: A class must have an EnclosingMethod
      // attribute if and only if it is a local class or an
      // anonymous class.
      return getEnclosingMethodInfo() != null;
   }

   //判断一个类是否是内部类
   public boolean isMemberClass() {
      return getSimpleBinaryName() != null && !isLocalOrAnonymousClass();
   }

   //返回简单名称的二进制字符,如果该对象是顶级类,则返回null.
   private String getSimpleBinaryName() {
      Class<?> enclosingClass = getEnclosingClass();
      if (enclosingClass == null) // top level class
         return null;
      // Otherwise, strip the enclosing class' name
      try {
         return getName().substring(enclosingClass.getName().length());
      } catch (IndexOutOfBoundsException ex) {
         throw new InternalError("Malformed class name", ex);
      }
   }

   //返回一个数组的Class对象,数组中包含所有在该类中定义的public的内部类和接口
   //包含从父类继承过来的public内部类和接口.如果对象没有public的内部类和接口,
   //或者是原始类型,数组类型,void则返回0长度的数组.
   @CallerSensitive
   public Class<?>[] getClasses() {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), false);

      // Privileged so this implementation can look at DECLARED classes,
      // something the caller might not have privilege to do.  The code here
      // is allowed to look at DECLARED classes because (1) it does not hand
      // out anything other than public members and (2) public member access
      // has already been ok'd by the SecurityManager.

      return java.security.AccessController.doPrivileged(
         new java.security.PrivilegedAction<Class<?>[]>() {
            public Class<?>[] run() {
               List<Class<?>> list = new ArrayList<>();
               Class<?> currentClass = Class.this;
               while (currentClass != null) {
                  Class<?>[] members = currentClass.getDeclaredClasses();
                  for (int i = 0; i < members.length; i++) {
                     if (Modifier.isPublic(members[i].getModifiers())) {
                        list.add(members[i]);
                     }
                  }
                  currentClass = currentClass.getSuperclass();
               }
               return list.toArray(new Class<?>[0]);
            }
         });
   }

   //返回一个数组的Class对象,数组中包含所有在该类中定义的的内部类和接口,包含所有
   //权限的(public,private,protected等).但是不包含从父类继承而来的内部类和接口.
   //如果该Class对象内部没有定义子类或者接口,或者为原始数据类型,数组对象或者void
   //则返回0长度的数组对象.
   @CallerSensitive
   public Class<?>[] getDeclaredClasses() throws SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), false);
      return getDeclaredClasses0();
   }

   //返回一个数组的Field对象,数组中包含当前Class对象中定义的所有public的成员对象
   //(Field), 其中包含继承的父类(父接口)中的对象(Field). 如果Class对象中内部没有
   //成员对象(Field)或者是原始类型,void,数组类型则返回0长度的数组.
   @CallerSensitive
   public Field[] getFields() throws SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      return copyFields(privateGetPublicFields(null));
   }

   //返回当前Class对象内所有的成员对象(Field), 包括所有的权限(public,private,
   //defalt等),但是不包含从父Class对象中继承过来的.其余情况参照上个函数.
   @CallerSensitive
   public Field[] getDeclaredFields() throws SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
      return copyFields(privateGetDeclaredFields(false));
   }

   //返回一个数组的Method对象,数组中存储Class对象中的所有public的方法对象, 包含
   //从父类(接口)继承过来的public方法. 如果有多个方法具有相同的名字和参数(类型和
   //数量都相同),返回值不同, 那么数组中应该为每个方法对应一个Method对象.并且这里
   //不会包含构造函数. 如果这个Class对象是一个数组类型, 那么会返回从Object中继承
   //得来的所有public方法(没有clone()方法). 如果代表的对象是一个接口, 那么不会继
   //承Object所有的方法, 也就是说如果内部不显式定义public方法,将返回0长度的数组.
   //如果是原始类型,void则返回0长度的数组对象. 需要注意的是,数组中的顺序是没有特
   //定的顺序的.
   @CallerSensitive
   public Method[] getMethods() throws SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      return copyMethods(privateGetPublicMethods());
   }

   //返回一个数组的Method对象,数组中存储Class对象中的所有的方法对象, 包含各种权限.
   //不包含从父类(接口)继承过来的方法. 如果有多个方法具有相同的名字和参数(类型和
   //数量都相同),返回值不同, 那么数组中应该为每个方法对应一个Method对象.并且这里
   //不会包含构造函数. 如果方法内部没有定义方法,或者是原始类型,void则返回0长度的
   //数组对象. 需要注意的是,数组中的顺序是没有特定的顺序的.
   @CallerSensitive
   public Method[] getDeclaredMethods() throws SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
      return copyMethods(privateGetDeclaredMethods(false));
   }

   //获得Class对象的所有public的构造方法.如果没有public的构造方法则返回0长度的数组.
   //包含数组类型,原始类型和void.
   @CallerSensitive
   public Constructor<T> getConstructors()
      throws NoSuchMethodException, SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      return getConstructor0(parameterTypes, Member.PUBLIC);
   }

   //获得Class对象的所有的构造方法.如果没有构造方法则返回0长度的数组.包含数组类型,
   //原始类型和void.
   @CallerSensitive
   public Constructor<?>[] getDeclaredConstructors() throws SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
      return copyConstructors(privateGetDeclaredConstructors(false));
   }

   //反射拷贝方法
   private static Method[] copyMethods(Method[] arg) {
      Method[] out = new Method[arg.length];
      ReflectionFactory fact = getReflectionFactory();
      for (int i = 0; i < arg.length; i++) {
         out[i] = fact.copyMethod(arg[i]);
      }
      return out;
   }

   //反射拷贝域
   private static Field[] copyFields(Field[] arg) {
      Field[] out = new Field[arg.length];
      ReflectionFactory fact = getReflectionFactory();
      for (int i = 0; i < arg.length; i++) {
         out[i] = fact.copyField(arg[i]);
      }
      return out;
   }

   //根据成员变量的名称返回成员对象(public),如果当前对象没有,会递归查找父类.
   @CallerSensitive
   public Field getField(String name)
      throws NoSuchFieldException, SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      Field field = getField0(name);
      if (field == null) {
         throw new NoSuchFieldException(name);
      }
      return field;
   }

   //根据名字返回一个域对象
   private Field getField0(String name) throws NoSuchFieldException {
      // Note: the intent is that the search algorithm this routine
      // uses be equivalent to the ordering imposed by
      // privateGetPublicFields(). It fetches only the declared
      // public fields for each class, however, to reduce the number
      // of Field objects which have to be created for the common
      // case where the field being requested is declared in the
      // class which is being queried.
      Field res;
      // Search declared public fields
      if ((res = searchFields(privateGetDeclaredFields(true), name)) != null) {
         return res;
      }
      // Direct superinterfaces, recursively
      Class<?>[] interfaces = getInterfaces();
      for (int i = 0; i < interfaces.length; i++) {
         Class<?> c = interfaces[i];
         if ((res = c.getField0(name)) != null) {
            return res;
         }
      }
      // Direct superclass, recursively
      if (!isInterface()) {
         Class<?> c = getSuperclass();
         if (c != null) {
            if ((res = c.getField0(name)) != null) {
               return res;
            }
         }
      }
      return null;
   }


   //搜索一个域,方法和构造函数
   private static Field searchFields(Field[] fields, String name) {
      String internedName = name.intern();
      for (int i = 0; i < fields.length; i++) {
         if (fields[i].getName() == internedName) {
            return getReflectionFactory().copyField(fields[i]);
         }
      }
      return null;
   }

   //返回Class对象的public方法,其中方法名需要和name完全一样,参数类型和顺序应该完全一样.
   //如果没有参数,传递一个空数组即可.如果传递函数名为构造函数则返回一个NoSuchMethodException
   //异常. 首先搜寻本Class对象中的方法,如果没有,则递归查询父类的方法. 如果查找到了不止一个方
   //法,则优先返回有返回参数的,否则随机挑选一个. 哪里可能有不止一个方法实现, Java语言中不允许
   //定义参数和方法名相同,返回值不同的方法.但是JVM是支持这样的方法签名.静态方法不认定为Class
   //对象的成员类和接口.
   @CallerSensitive
   public Method getMethod(String name, Class<?>... parameterTypes)
      throws NoSuchMethodException, SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      Method method = getMethod0(name, parameterTypes, true);
      if (method == null) {
         throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
      }
      return method;
   }

   //返回Class对象的所有方法声明,不会查找父类对象的方法,参照上面的方法. 其中如果Class对象是一个
   //Class对象是不会获取到clone()方法.
   @CallerSensitive
   public Method getDeclaredMethod(String name, Class<?>... parameterTypes)
      throws NoSuchMethodException, SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
      Method method = searchMethods(privateGetDeclaredMethods(false), name, parameterTypes);
      if (method == null) {
         throw new NoSuchMethodException(getName() + "." + name + argumentTypesToString(parameterTypes));
      }
      return method;
   }

   //根据参数类型和列表返回public的构造函数. 如果是在非静态内容中声明的内部类,那么需要将前面的
   //外部类的实例作为第一个参数.
   @CallerSensitive
   public Constructor<T> getConstructor(Class<?>... parameterTypes)
      throws NoSuchMethodException, SecurityException {
      checkMemberAccess(Member.PUBLIC, Reflection.getCallerClass(), true);
      return getConstructor0(parameterTypes, Member.PUBLIC);
   }

   //参照上面的方法的,获取所有的权限的构造方法.
   @CallerSensitive
   public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)
      throws NoSuchMethodException, SecurityException {
      checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
      return getConstructor0(parameterTypes, Member.DECLARED);
   }

   //根据给定的name返回特定的资源. 搜索资源的方法交给Class对象的类加载器.如果该对象是由Bootsrap
   //类加载器, 这个方法被委托给getSystemResourceAsStream()方法. 在委托之前,资源的名字规则如下:
   //如果名字以"/"开头, 那么就是"/"后面的绝对资源名称.否则就以如下规则: modified_package_name/name
   //其中modified_package_name, 以"/"替换字符串中的".".
   public InputStream getResourceAsStream(String name) {
      name = resolveName(name);
      ClassLoader cl = getClassLoader0();
      if (cl==null) {
         // A system class.
         return ClassLoader.getSystemResourceAsStream(name);
      }
      return cl.getResourceAsStream(name);
   }

   //参照上面的方法
   public java.net.URL getResource(String name) {
      name = resolveName(name);
      ClassLoader cl = getClassLoader0();
      if (cl==null) {
         // A system class.
         return ClassLoader.getSystemResource(name);
      }
      return cl.getResource(name);
   }

   //内部域为空时,返回保护域
   /** protection domain returned when the internal domain is null */
   private static java.security.ProtectionDomain allPermDomain;

   //返回allPermDomain,如果存在SM,进行权限校验
   public java.security.ProtectionDomain getProtectionDomain() {
      SecurityManager sm = System.getSecurityManager();
      if (sm != null) {
         sm.checkPermission(SecurityConstants.GET_PD_PERMISSION);
      }
      java.security.ProtectionDomain pd = getProtectionDomain0();
      if (pd == null) {
         if (allPermDomain == null) {
            java.security.Permissions perms =
               new java.security.Permissions();
            perms.add(SecurityConstants.ALL_PERMISSION);
            allPermDomain =
               new java.security.ProtectionDomain(null, perms);
         }
         pd = allPermDomain;
      }
      return pd;
   }

   /**
    * Returns the ProtectionDomain of this class.
    */
   private native java.security.ProtectionDomain getProtectionDomain0();

   /*
    * Return the Virtual Machine's Class object for the named
    * primitive type.
    */
   static native Class<?> getPrimitiveClass(String name);

   //检查当前客户端是否有权限使用该对象,如果拒绝了,抛出异常. 这个方法同样需要检测包权限.
   //默认的政策: 允许所有的clients有权限处理正常的Java使用权限.
   private void checkMemberAccess(int which, Class<?> caller, boolean checkProxyInterfaces) {
      final SecurityManager s = System.getSecurityManager();
      if (s != null) {
         /* Default policy allows access to all {@link Member#PUBLIC} members,
          * as well as access to classes that have the same class loader as the caller.
          * In all other cases, it requires RuntimePermission("accessDeclaredMembers")
          * permission.
          */
         final ClassLoader ccl = ClassLoader.getClassLoader(caller);
         final ClassLoader cl = getClassLoader0();
         if (which != Member.PUBLIC) {
            if (ccl != cl) {
               s.checkPermission(SecurityConstants.CHECK_MEMBER_ACCESS_PERMISSION);
            }
         }
         this.checkPackageAccess(ccl, checkProxyInterfaces);
      }
   }

   //检查client是否有权限使用包
   private void checkPackageAccess(final ClassLoader ccl, boolean checkProxyInterfaces) {
      final SecurityManager s = System.getSecurityManager();
      if (s != null) {
         final ClassLoader cl = getClassLoader0();

         if (ReflectUtil.needsPackageAccessCheck(ccl, cl)) {
            String name = this.getName();
            int i = name.lastIndexOf('.');
            if (i != -1) {
               // skip the package access check on a proxy class in default proxy package
               String pkg = name.substring(0, i);
               if (!Proxy.isProxyClass(this) || ReflectUtil.isNonPublicProxyClass(this)) {
                  s.checkPackageAccess(pkg);
               }
            }
         }
         // check package access on the proxy interfaces
         if (checkProxyInterfaces && Proxy.isProxyClass(this)) {
            ReflectUtil.checkProxyPackageAccess(ccl, this.getInterfaces());
         }
      }
   }

   //解析名字,详情看函数实现.
   private String resolveName(String name) {
      if (name == null) {
         return name;
      }
      if (!name.startsWith("/")) {
         Class<?> c = this;
         while (c.isArray()) {
            c = c.getComponentType();
         }
         String baseName = c.getName();
         int index = baseName.lastIndexOf('.');
         if (index != -1) {
            name = baseName.substring(0, index).replace('.', '/')
               +"/"+name;
         }
      } else {
         name = name.substring(1);
      }
      return name;
   }

   /**
    * Atomic operations support. 用于支持原子操作的内部类
    */
   private static class Atomic {
      // initialize Unsafe machinery here, since we need to call Class.class instance method
      // and have to avoid calling it in the static initializer of the Class class...
      private static final Unsafe unsafe = Unsafe.getUnsafe();
      // offset of Class.reflectionData instance field
      private static final long reflectionDataOffset;
      // offset of Class.annotationType instance field
      private static final long annotationTypeOffset;
      // offset of Class.annotationData instance field
      private static final long annotationDataOffset;

      static {
         Field[] fields = Class.class.getDeclaredFields0(false); // bypass caches
         reflectionDataOffset = objectFieldOffset(fields, "reflectionData");
         annotationTypeOffset = objectFieldOffset(fields, "annotationType");
         annotationDataOffset = objectFieldOffset(fields, "annotationData");
      }

      private static long objectFieldOffset(Field[] fields, String fieldName) {
         Field field = searchFields(fields, fieldName);
         if (field == null) {
            throw new Error("No " + fieldName + " field found in java.lang.Class");
         }
         return unsafe.objectFieldOffset(field);
      }

      static <T> boolean casReflectionData(Class<?> clazz,
                                  SoftReference<ReflectionData<T>> oldData,
                                  SoftReference<ReflectionData<T>> newData) {
         return unsafe.compareAndSwapObject(clazz, reflectionDataOffset, oldData, newData);
      }

      static <T> boolean casAnnotationType(Class<?> clazz,
                                  AnnotationType oldType,
                                  AnnotationType newType) {
         return unsafe.compareAndSwapObject(clazz, annotationTypeOffset, oldType, newType);
      }

      static <T> boolean casAnnotationData(Class<?> clazz,
                                  AnnotationData oldData,
                                  AnnotationData newData) {
         return unsafe.compareAndSwapObject(clazz, annotationDataOffset, oldData, newData);
      }
   }

   // Annotations handling, 处理注解信息
   native byte[] getRawAnnotations();
   // Since 1.8
   native byte[] getRawTypeAnnotations();
   static byte[] getExecutableTypeAnnotationBytes(Executable ex) {
      return getReflectionFactory().getExecutableTypeAnnotationBytes(ex);
   }
   //获得常量池
   native ConstantPool getConstantPool();

   //返回数组的"root"域,这些域必须不能被传播到外部,必须使用RefectionFactory.copyField
   //进行拷贝.
   private Field[] privateGetDeclaredFields(boolean publicOnly) {
      checkInitted();
      Field[] res;
      ReflectionData<T> rd = reflectionData();
      if (rd != null) {
         res = publicOnly ? rd.declaredPublicFields : rd.declaredFields;
         if (res != null) return res;
      }
      // No cached value available; request value from VM
      res = Reflection.filterFields(this, getDeclaredFields0(publicOnly));
      if (rd != null) {
         if (publicOnly) {
            rd.declaredPublicFields = res;
         } else {
            rd.declaredFields = res;
         }
      }
      return res;
   }

   //同上
   private Field[] privateGetPublicFields(Set<Class<?>> traversedInterfaces) {
      checkInitted();
      Field[] res;
      ReflectionData<T> rd = reflectionData();
      if (rd != null) {
         res = rd.publicFields;
         if (res != null) return res;
      }

      // No cached value available; compute value recursively.
      // Traverse in correct order for getField().
      List<Field> fields = new ArrayList<>();
      if (traversedInterfaces == null) {
         traversedInterfaces = new HashSet<>();
      }

      // Local fields
      Field[] tmp = privateGetDeclaredFields(true);
      addAll(fields, tmp);

      // Direct superinterfaces, recursively
      for (Class<?> c : getInterfaces()) {
         if (!traversedInterfaces.contains(c)) {
            traversedInterfaces.add(c);
            addAll(fields, c.privateGetPublicFields(traversedInterfaces));
         }
      }

      // Direct superclass, recursively
      if (!isInterface()) {
         Class<?> c = getSuperclass();
         if (c != null) {
            addAll(fields, c.privateGetPublicFields(traversedInterfaces));
         }
      }

      res = new Field[fields.size()];
      fields.toArray(res);
      if (rd != null) {
         rd.publicFields = res;
      }
      return res;
   }

   private static void addAll(Collection<Field> c, Field[] o) {
      for (int i = 0; i < o.length; i++) {
         c.add(o[i]);
      }
   }

   //返回数组的"root"的构造函数, 这些构造函数不得传播到外部,必须使用ReflectionFacy.
   //copyConstuctor进行拷贝.
   private Constructor<T>[] privateGetDeclaredConstructors(boolean publicOnly) {
      checkInitted();
      Constructor<T>[] res;
      ReflectionData<T> rd = reflectionData();
      if (rd != null) {
         res = publicOnly ? rd.publicConstructors : rd.declaredConstructors;
         if (res != null) return res;
      }
      // No cached value available; request value from VM
      if (isInterface()) {
         @SuppressWarnings("unchecked")
         Constructor<T>[] temporaryRes = (Constructor<T>[]) new Constructor<?>[0];
         res = temporaryRes;
      } else {
         res = getDeclaredConstructors0(publicOnly);
      }
      if (rd != null) {
         if (publicOnly) {
            rd.publicConstructors = res;
         } else {
            rd.declaredConstructors = res;
         }
      }
      return res;
   }

   //返回数组的"root"的方法, 这些构造函数不得传播到外部,必须使用ReflectionFacy.
   //copyMethod进行拷贝.
   private Method[] privateGetDeclaredMethods(boolean publicOnly) {
      checkInitted();
      Method[] res;
      ReflectionData<T> rd = reflectionData();
      if (rd != null) {
         res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
         if (res != null) return res;
      }
      // No cached value available; request value from VM
      res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
      if (rd != null) {
         if (publicOnly) {
            rd.declaredPublicMethods = res;
         } else {
            rd.declaredMethods = res;
         }
      }
      return res;
   }

   //静态方法数组类
   static class MethodArray {
      // Don't add or remove methods except by add() or remove() calls.
      private Method[] methods;
      private int length;
      private int defaults;

      MethodArray() {
         this(20);
      }

      MethodArray(int initialSize) {
         if (initialSize < 2)
            throw new IllegalArgumentException("Size should be 2 or more");

         methods = new Method[initialSize];
         length = 0;
         defaults = 0;
      }

      boolean hasDefaults() {
         return defaults != 0;
      }

      void add(Method m) {
         if (length == methods.length) {
            methods = Arrays.copyOf(methods, 2 * methods.length);
         }
         methods[length++] = m;

         if (m != null && m.isDefault())
            defaults++;
      }

      void addAll(Method[] ma) {
         for (int i = 0; i < ma.length; i++) {
            add(ma[i]);
         }
      }

      void addAll(MethodArray ma) {
         for (int i = 0; i < ma.length(); i++) {
            add(ma.get(i));
         }
      }

      void addIfNotPresent(Method newMethod) {
         for (int i = 0; i < length; i++) {
            Method m = methods[i];
            if (m == newMethod || (m != null && m.equals(newMethod))) {
               return;
            }
         }
         add(newMethod);
      }

      void addAllIfNotPresent(MethodArray newMethods) {
         for (int i = 0; i < newMethods.length(); i++) {
            Method m = newMethods.get(i);
            if (m != null) {
               addIfNotPresent(m);
            }
         }
      }

      /* Add Methods declared in an interface to this MethodArray.
       * Static methods declared in interfaces are not inherited.
       */
      void addInterfaceMethods(Method[] methods) {
         for (Method candidate : methods) {
            if (!Modifier.isStatic(candidate.getModifiers())) {
               add(candidate);
            }
         }
      }

      int length() {
         return length;
      }

      Method get(int i) {
         return methods[i];
      }

      Method getFirst() {
         for (Method m : methods)
            if (m != null)
               return m;
         return null;
      }

      void removeByNameAndDescriptor(Method toRemove) {
         for (int i = 0; i < length; i++) {
            Method m = methods[i];
            if (m != null && matchesNameAndDescriptor(m, toRemove)) {
               remove(i);
            }
         }
      }

      private void remove(int i) {
         if (methods[i] != null && methods[i].isDefault())
            defaults--;
         methods[i] = null;
      }

      private boolean matchesNameAndDescriptor(Method m1, Method m2) {
         return m1.getReturnType() == m2.getReturnType() &&
               m1.getName() == m2.getName() && // name is guaranteed to be interned
               arrayContentsEq(m1.getParameterTypes(),
                     m2.getParameterTypes());
      }

      void compactAndTrim() {
         int newPos = 0;
         // Get rid of null slots
         for (int pos = 0; pos < length; pos++) {
            Method m = methods[pos];
            if (m != null) {
               if (pos != newPos) {
                  methods[newPos] = m;
               }
               newPos++;
            }
         }
         if (newPos != methods.length) {
            methods = Arrays.copyOf(methods, newPos);
         }
      }

      /* Removes all Methods from this MethodArray that have a more specific
       * default Method in this MethodArray.
       *
       * Users of MethodArray are responsible for pruning Methods that have
       * a more specific <em>concrete</em> Method.
       */
      void removeLessSpecifics() {
         if (!hasDefaults())
            return;

         for (int i = 0; i < length; i++) {
            Method m = get(i);
            if  (m == null || !m.isDefault())
               continue;

            for (int j  = 0; j < length; j++) {
               if (i == j)
                  continue;

               Method candidate = get(j);
               if (candidate == null)
                  continue;

               if (!matchesNameAndDescriptor(m, candidate))
                  continue;

               if (hasMoreSpecificClass(m, candidate))
                  remove(j);
            }
         }
      }

      Method[] getArray() {
         return methods;
      }

      // Returns true if m1 is more specific than m2
      static boolean hasMoreSpecificClass(Method m1, Method m2) {
         Class<?> m1Class = m1.getDeclaringClass();
         Class<?> m2Class = m2.getDeclaringClass();
         return m1Class != m2Class && m2Class.isAssignableFrom(m1Class);
      }
   }

   //类似上面的方法
   // Returns an array of "root" methods. These Method objects must NOT
   // be propagated to the outside world, but must instead be copied
   // via ReflectionFactory.copyMethod.
   private Method[] privateGetPublicMethods() {
      checkInitted();
      Method[] res;
      ReflectionData<T> rd = reflectionData();
      if (rd != null) {
         res = rd.publicMethods;
         if (res != null) return res;
      }

      // No cached value available; compute value recursively.
      // Start by fetching public declared methods
      MethodArray methods = new MethodArray();
      {
         Method[] tmp = privateGetDeclaredMethods(true);
         methods.addAll(tmp);
      }
      // Now recur over superclass and direct superinterfaces.
      // Go over superinterfaces first so we can more easily filter
      // out concrete implementations inherited from superclasses at
      // the end.
      MethodArray inheritedMethods = new MethodArray();
      for (Class<?> i : getInterfaces()) {
         inheritedMethods.addInterfaceMethods(i.privateGetPublicMethods());
      }
      if (!isInterface()) {
         Class<?> c = getSuperclass();
         if (c != null) {
            MethodArray supers = new MethodArray();
            supers.addAll(c.privateGetPublicMethods());
            // Filter out concrete implementations of any
            // interface methods
            for (int i = 0; i < supers.length(); i++) {
               Method m = supers.get(i);
               if (m != null &&
                     !Modifier.isAbstract(m.getModifiers()) &&
                     !m.isDefault()) {
                  inheritedMethods.removeByNameAndDescriptor(m);
               }
            }
            // Insert superclass's inherited methods before
            // superinterfaces' to satisfy getMethod's search
            // order
            supers.addAll(inheritedMethods);
            inheritedMethods = supers;
         }
      }
      // Filter out all local methods from inherited ones
      for (int i = 0; i < methods.length(); i++) {
         Method m = methods.get(i);
         inheritedMethods.removeByNameAndDescriptor(m);
      }
      methods.addAllIfNotPresent(inheritedMethods);
      methods.removeLessSpecifics();
      methods.compactAndTrim();
      res = methods.getArray();
      if (rd != null) {
         rd.publicMethods = res;
      }
      return res;
   }

   private static Method searchMethods(Method[] methods,
                              String name,
                              Class<?>[] parameterTypes)
   {
      Method res = null;
      String internedName = name.intern();
      for (int i = 0; i < methods.length; i++) {
         Method m = methods[i];
         if (m.getName() == internedName
            && arrayContentsEq(parameterTypes, m.getParameterTypes())
            && (res == null
               || res.getReturnType().isAssignableFrom(m.getReturnType())))
            res = m;
      }

      return (res == null ? res : getReflectionFactory().copyMethod(res));
   }

   private Method getMethod0(String name, Class<?>[] parameterTypes, boolean includeStaticMethods) {
      MethodArray interfaceCandidates = new MethodArray(2);
      Method res =  privateGetMethodRecursive(name, parameterTypes, includeStaticMethods, interfaceCandidates);
      if (res != null)
         return res;

      // Not found on class or superclass directly
      interfaceCandidates.removeLessSpecifics();
      return interfaceCandidates.getFirst(); // may be null
   }

   private Constructor<T> getConstructor0(Class<?>[] parameterTypes,
                              int which) throws NoSuchMethodException
   {
      Constructor<T>[] constructors = privateGetDeclaredConstructors((which == Member.PUBLIC));
      for (Constructor<T> constructor : constructors) {
         if (arrayContentsEq(parameterTypes,
                        constructor.getParameterTypes())) {
            return getReflectionFactory().copyConstructor(constructor);
         }
      }
      throw new NoSuchMethodException(getName() + ".<init>" + argumentTypesToString(parameterTypes));
   }

   //一些工具方法
   private static boolean arrayContentsEq(Object[] a1, Object[] a2) {
      if (a1 == null) {
         return a2 == null || a2.length == 0;
      }

      if (a2 == null) {
         return a1.length == 0;
      }

      if (a1.length != a2.length) {
         return false;
      }

      for (int i = 0; i < a1.length; i++) {
         if (a1[i] != a2[i]) {
            return false;
         }
      }

      return true;
   }

   private static <U> Constructor<U>[] copyConstructors(Constructor<U>[] arg) {
      Constructor<U>[] out = arg.clone();
      ReflectionFactory fact = getReflectionFactory();
      for (int i = 0; i < out.length; i++) {
         out[i] = fact.copyConstructor(out[i]);
      }
      return out;
   }

   private native Field[]       getDeclaredFields0(boolean publicOnly);
   private native Method[]      getDeclaredMethods0(boolean publicOnly);
   private native Constructor<T>[] getDeclaredConstructors0(boolean publicOnly);
   private native Class<?>[]   getDeclaredClasses0();

   private static String        argumentTypesToString(Class<?>[] argTypes) {
      StringBuilder buf = new StringBuilder();
      buf.append("(");
      if (argTypes != null) {
         for (int i = 0; i < argTypes.length; i++) {
            if (i > 0) {
               buf.append(", ");
            }
            Class<?> c = argTypes[i];
            buf.append((c == null) ? "null" : c.getName());
         }
      }
      buf.append(")");
      return buf.toString();
   }

   /** use serialVersionUID from JDK 1.1 for interoperability */
   private static final long serialVersionUID = 3206093459760846163L;

   //Class类对象在序列化规则中是特殊的.详情查看java.io.ObjectStreamClass对象.
   private static final ObjectStreamField[] serialPersistentFields =
      new ObjectStreamField[0];

   //如果class对象的断言被初始化,返回断言的状态. 如果没有初始化断言, 返回所在包
   //的默认断言状态, 如果包也没有初始化,返回当前class对象类加载器的默认断言状态,
   //如果是系统类,返回系统默认的断言状态. 很少应用会用到这个方法,主要用于JRE.注意
   //该方法不会保证返回实际的断言状态.
   public boolean desiredAssertionStatus() {
      ClassLoader loader = getClassLoader();
      // If the loader is null this is a system class, so ask the VM
      if (loader == null)
         return desiredAssertionStatus0(this);

      // If the classloader has been initialized with the assertion
      // directives, ask it. Otherwise, ask the VM.
      synchronized(loader.assertionLock) {
         if (loader.classAssertionStatus != null) {
            return loader.desiredAssertionStatus(getName());
         }
      }
      return desiredAssertionStatus0(this);
   }

   // Retrieves the desired assertion status of this class from the VM
   private static native boolean desiredAssertionStatus0(Class<?> clazz);

   //返回是否是枚举
   public boolean isEnum() {
      // An enum must both directly extend java.lang.Enum and have
      // the ENUM bit set; classes for specialized enum constants
      // don't do the former.
      return (this.getModifiers() & ENUM) != 0 &&
      this.getSuperclass() == java.lang.Enum.class;
   }

   // Fetches the factory for reflective objects
   private static ReflectionFactory getReflectionFactory() {
      if (reflectionFactory == null) {
         reflectionFactory =
            java.security.AccessController.doPrivileged
               (new sun.reflect.ReflectionFactory.GetReflectionFactoryAction());
      }
      return reflectionFactory;
   }
   private static ReflectionFactory reflectionFactory;

   // To be able to query system properties as soon as they're available
   private static boolean initted = false;
   private static void checkInitted() {
      if (initted) return;
      AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
               // Tests to ensure the system properties table is fully
               // initialized. This is needed because reflection code is
               // called very early in the initialization process (before
               // command-line arguments have been parsed and therefore
               // these user-settable properties installed.) We assume that
               // if System.out is non-null then the System class has been
               // fully initialized and that the bulk of the startup code
               // has been run.

               if (System.out == null) {
                  // java.lang.System not yet fully initialized
                  return null;
               }

               // Doesn't use Boolean.getBoolean to avoid class init.
               String val =
                  System.getProperty("sun.reflect.noCaches");
               if (val != null && val.equals("true")) {
                  useCaches = false;
               }

               initted = true;
               return null;
            }
         });
   }

   //返回枚举中的对象
   public T[] getEnumConstants() {
      T[] values = getEnumConstantsShared();
      return (values != null) ? values.clone() : null;
   }

   //返回枚举中的对象,如果当前Class对象不是枚举类型返回null,
   T[] getEnumConstantsShared() {
      if (enumConstants == null) {
         if (!isEnum()) return null;
         try {
            final Method values = getMethod("values");
            java.security.AccessController.doPrivileged(
               new java.security.PrivilegedAction<Void>() {
                  public Void run() {
                        values.setAccessible(true);
                        return null;
                     }
                  });
            @SuppressWarnings("unchecked")
            T[] temporaryConstants = (T[])values.invoke(null);
            enumConstants = temporaryConstants;
         }
         // These can happen when users concoct enum-like classes
         // that don't comply with the enum spec.
         catch (InvocationTargetException | NoSuchMethodException |
               IllegalAccessException ex) { return null; }
      }
      return enumConstants;
   }
   private volatile transient T[] enumConstants = null;

   //返回name向枚举类型的映射,主要用于枚举的内部实现.
   Map<String, T> enumConstantDirectory() {
      if (enumConstantDirectory == null) {
         T[] universe = getEnumConstantsShared();
         if (universe == null)
            throw new IllegalArgumentException(
               getName() + " is not an enum type");
         Map<String, T> m = new HashMap<>(2 * universe.length);
         for (T constant : universe)
            m.put(((Enum<?>)constant).name(), constant);
         enumConstantDirectory = m;
      }
      return enumConstantDirectory;
   }
   private volatile transient Map<String, T> enumConstantDirectory = null;

   @SuppressWarnings("unchecked")
   public T cast(Object obj) {
      if (obj != null && !isInstance(obj))
         throw new ClassCastException(cannotCastMsg(obj));
      return (T) obj;
   }

   private String cannotCastMsg(Object obj) {
      return "Cannot cast " + obj.getClass().getName() + " to " + getName();
   }

   //转换Class对象为另一个对象的子类.检查是否可以转换成功.不行,抛出异常. 如果
   //可以返回该对象的应用.
   @SuppressWarnings("unchecked")
   public <U> Class<? extends U> asSubclass(Class<U> clazz) {
      if (clazz.isAssignableFrom(this))
         return (Class<? extends U>) this;
      else
         throw new ClassCastException(this.toString());
   }

   //返回注释
   @SuppressWarnings("unchecked")
   public <A extends Annotation> A getAnnotation(Class<A> annotationClass) {
      Objects.requireNonNull(annotationClass);

      return (A) annotationData().annotations.get(annotationClass);
   }

   @Override
   public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass) {
      return GenericDeclaration.super.isAnnotationPresent(annotationClass);
   }

   @Override
   public <A extends Annotation> A[] getAnnotationsByType(Class<A> annotationClass) {
      Objects.requireNonNull(annotationClass);

      AnnotationData annotationData = annotationData();
      return AnnotationSupport.getAssociatedAnnotations(annotationData.declaredAnnotations,
                                            this,
                                            annotationClass);
   }

   public Annotation[] getAnnotations() {
      return AnnotationParser.toArray(annotationData().annotations);
   }

   @Override
   @SuppressWarnings("unchecked")
   public <A extends Annotation> A getDeclaredAnnotation(Class<A> annotationClass) {
      Objects.requireNonNull(annotationClass);

      return (A) annotationData().declaredAnnotations.get(annotationClass);
   }

   @Override
   public <A extends Annotation> A[] getDeclaredAnnotationsByType(Class<A> annotationClass) {
      Objects.requireNonNull(annotationClass);

      return AnnotationSupport.getDirectlyAndIndirectlyPresent(annotationData().declaredAnnotations,
                                                 annotationClass);
   }

   public Annotation[] getDeclaredAnnotations()  {
      return AnnotationParser.toArray(annotationData().declaredAnnotations);
   }

   // annotation data that might get invalidated when JVM TI RedefineClasses() is called
   private static class AnnotationData {
      final Map<Class<? extends Annotation>, Annotation> annotations;
      final Map<Class<? extends Annotation>, Annotation> declaredAnnotations;

      // Value of classRedefinedCount when we created this AnnotationData instance
      final int redefinedCount;

      AnnotationData(Map<Class<? extends Annotation>, Annotation> annotations,
                  Map<Class<? extends Annotation>, Annotation> declaredAnnotations,
                  int redefinedCount) {
         this.annotations = annotations;
         this.declaredAnnotations = declaredAnnotations;
         this.redefinedCount = redefinedCount;
      }
   }

   // Annotations cache
   @SuppressWarnings("UnusedDeclaration")
   private volatile transient AnnotationData annotationData;

   private AnnotationData annotationData() {
      while (true) { // retry loop
         AnnotationData annotationData = this.annotationData;
         int classRedefinedCount = this.classRedefinedCount;
         if (annotationData != null &&
            annotationData.redefinedCount == classRedefinedCount) {
            return annotationData;
         }
         // null or stale annotationData -> optimistically create new instance
         AnnotationData newAnnotationData = createAnnotationData(classRedefinedCount);
         // try to install it
         if (Atomic.casAnnotationData(this, annotationData, newAnnotationData)) {
            // successfully installed new AnnotationData
            return newAnnotationData;
         }
      }
   }

   private AnnotationData createAnnotationData(int classRedefinedCount) {
      Map<Class<? extends Annotation>, Annotation> declaredAnnotations =
         AnnotationParser.parseAnnotations(getRawAnnotations(), getConstantPool(), this);
      Class<?> superClass = getSuperclass();
      Map<Class<? extends Annotation>, Annotation> annotations = null;
      if (superClass != null) {
         Map<Class<? extends Annotation>, Annotation> superAnnotations =
            superClass.annotationData().annotations;
         for (Map.Entry<Class<? extends Annotation>, Annotation> e : superAnnotations.entrySet()) {
            Class<? extends Annotation> annotationClass = e.getKey();
            if (AnnotationType.getInstance(annotationClass).isInherited()) {
               if (annotations == null) { // lazy construction
                  annotations = new LinkedHashMap<>((Math.max(
                        declaredAnnotations.size(),
                        Math.min(12, declaredAnnotations.size() + superAnnotations.size())
                     ) * 4 + 2) / 3
                  );
               }
               annotations.put(annotationClass, e.getValue());
            }
         }
      }
      if (annotations == null) {
         // no inherited annotations -> share the Map with declaredAnnotations
         annotations = declaredAnnotations;
      } else {
         // at least one inherited annotation -> declared may override inherited
         annotations.putAll(declaredAnnotations);
      }
      return new AnnotationData(annotations, declaredAnnotations, classRedefinedCount);
   }

   @SuppressWarnings("UnusedDeclaration")
   private volatile transient AnnotationType annotationType;

   boolean casAnnotationType(AnnotationType oldType, AnnotationType newType) {
      return Atomic.casAnnotationType(this, oldType, newType);
   }

   AnnotationType getAnnotationType() {
      return annotationType;
   }

   Map<Class<? extends Annotation>, Annotation> getDeclaredAnnotationMap() {
      return annotationData().declaredAnnotations;
   }

   /* Backing store of user-defined values pertaining to this class.
    * Maintained by the ClassValue class.
    */
   transient ClassValue.ClassValueMap classValueMap;

   //返回父类的注解的AnnotatedType,如果是原始类型,数组类型,void,返回值为null.
   public AnnotatedType getAnnotatedSuperclass() {
      if (this == Object.class ||
            isInterface() ||
            isArray() ||
            isPrimitive() ||
            this == Void.TYPE) {
         return null;
      }

      return TypeAnnotationParser.buildAnnotatedSuperclass(getRawTypeAnnotations(), getConstantPool(), this);
   }

   //返回注释的AnnotatedType类型
   public AnnotatedType[] getAnnotatedInterfaces() {
       return TypeAnnotationParser.buildAnnotatedInterfaces(getRawTypeAnnotations(), getConstantPool(), this);
   }
}
```
