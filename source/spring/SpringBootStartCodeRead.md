# Spring Boot 启动源码解读

现在大部分的`Spring`相关的应用都使用`Spring Boot`作为载体, `Spring boot`凭借其少配置, 易拓展, 部署简单等优点成为了大多数`Spring`程序的首选. 这里让我们从源码的角度, 探究`Spring Boot`程序的内部奥秘. 本文主要从`Spring Boot`程序启动代码开始阅读, 为了避免过于文章过于冗长, 源码的阅读深度选择适中, 后续将补充相关深层次源码阅读.

## 起点

一个简单的`Spring Boot`程序, 从一个简单的`main`函数开始:

```java
@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
```

这里可以看出, `Spring Boot`程序的启动, 只是简单的调用了`SpringApplication`类的静态方法: `run`. `SpringApplication`类的注释也恰好解释了这个类的职责:

`SpringApplication`: `Class that can be used to bootstrap and launch a Spring application from a Java main method. By default class will perform the following steps to bootstrap your application:`

- Create an appropriate `ApplicationContext` instance (depending on your classpath).
- Register a `CommandLinePropertySource` to expose command line arguments as Spring properties.
- Refresh the application context, loading all singleton beans.
- Trigger any `CommandLineRunner` beans.

简单来说, 就是`SpringApplication`用于从`mian`方法中启动`Spring Boot`程序, 主要步骤分为四步:

- 创建合适的`ApplicationContext`, 创建的标准取决于`ClassPath`.(后面源码会看到).
- 注册一个`CommandLinePropertySource`用于读取命令行参数, 并转化为`Spring`的属性.
- 刷新`Spring`上下文, 并加载所有的相关单实例Bean.
- 应用启动完之后, 触发所有的`CommandLineRunner`执行一些自定义的操作.

接下来, 让我们一起进入代码的世界:

```java
// actual call
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
  return run(new Class<?>[] { primarySource }, args);
}

// actual impl
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
  return new SpringApplication(primarySources).run(args);
}
```

可以发现启动的过程主要分为两步:

- 构造一个新的`SpringApplication`实例
- 调用实例`run`的方法

## 构造SpringApplication实例

在构造函数中, 主要完成了以下几件事:

- 记录启动类相关的信息.
- 推断`Web应用`类型.
- 读取并存储`ApplicationContextInitializer`实例列表.
- 读取并存储`ApplicationListener`实例列表.

```java
// actual call
// primarySources 为前面的启动类: Application.class
public SpringApplication(Class<?>... primarySources) {
  this(null, primarySources);
}

// actual impl
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  // 记录ResourceLoader, 为null
  this.resourceLoader = resourceLoader;

  Assert.notNull(primarySources, "PrimarySources must not be null");

  // 将传递进来的启动类(Application.class)存储在primarySources中
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

  // 通过ClassPath推断Web应用的类型,(该变量为枚举有: NONE, SERVLET, REACTIVE), 详见下文, 这里返回的为 SERVLET 类型
  this.webApplicationType = WebApplicationType.deduceFromClasspath();

  // 1\. 读取并缓存所有Jar包中META-INF/spring.factories中定义的各种配置类
  // 2\. 读取类型为 ApplicationContextInitializer.class相关的服务类
  // 3\. 将对应的ApplicationContextInitializer实例列表存储在initializers中
  setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

  // 同理读取所有ApplicationListener实例列表存储在listeners中
  setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

  // 存储当前的启动类(main方法入口类)到mainApplicationClass, 详见下文
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 根据ClassPath推断Web应用类型

根据`ClassPath`路径中是否包含对应的依赖, 推断出对应的`Web`应用类型.

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet", "org.springframework.web.context.ConfigurableWebApplicationContext" };
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework." + "web.servlet.DispatcherServlet";
private static final String WEBFLUX_INDICATOR_CLASS = "org." + "springframework.web.reactive.DispatcherHandler";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
  // 优先判断 REACTIVE 类型
  if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
          && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
      return WebApplicationType.REACTIVE;
  }
  // 其次判断 NONE 类型
  for (String className : SERVLET_INDICATOR_CLASSES) {
      if (!ClassUtils.isPresent(className, null)) {
          return WebApplicationType.NONE;
      }
  }

  // 最终缺省为 SERVLET 类型
  return WebApplicationType.SERVLET;
}
```

### 读取spring.factories配置

如 `spring-boot` jar包下的`META-INF/spring.factories`文件:

```java
// # PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader

// # Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

// # Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers
...
```

读取并缓存`META-INF/spring.factories`中定义好的各种配置类:

- 使用`SpringFactoriesLoader`读取并缓存所有`Jar`包中`META-INF/spring.factories`所定义的各种配置类信息.
- 创建对应的实例.
- 根据优先级进行排序

```java
// type传递的是ApplicationContextInitializer类
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
  // 获取类加载器, 这里为 AppClassLoader
  ClassLoader classLoader = getClassLoader();

  // 加载所有的配置类, 从中读取 ApplicationContextInitializer 类型的配置类
  Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));

  // 根据传递进来的的构造参数和类型进行实例化(这里都为空数组)
  List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);

  // 依据实例中实现的 Ordered 注解的大小进行排序
  AnnotationAwareOrderComparator.sort(instances);

  // 返回实例列表
  return instances;
}
```

#### SpringFactoriesLoader加载spring.factories

`SpringFactoriesLoader`读取`spring.factories`:

- 遍历所有`META-INF/spring.factories`, 读取内部定义的配置类信息.
- 将对应的配置类按照类型, 缓存到对应的`Map`中, 并对外提供访问接口.

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
  // 读取要选取的类型,这里为: ApplicationContextInitializer.class
  String factoryClassName = factoryClass.getName();

  // 加载所有的 spring.factories, 读取内部定义的配置类信息
  return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
  // 如果之前已经查找过(已经放入缓存), 则直接返回缓存.
  MultiValueMap<String, String> result = cache.get(classLoader);
  if (result != null) {
    return result;
  }

  // 遍历读取所有的 META-INF/spring.factories 文件并缓存
  try {

    // 加载本地ClassPath下所有的META-INF/spring.factories资源
    Enumeration<URL> urls = (classLoader != null ?
        classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
        ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));

    result = new LinkedMultiValueMap<>();

    // 遍历解析每一个 META-INF/spring.factories 文件
    while (urls.hasMoreElements()) {
      URL url = urls.nextElement();
      UrlResource resource = new UrlResource(url);

      // 将内部文件解析成Properties
      Properties properties = PropertiesLoaderUtils.loadProperties(resource);

      // 遍历解析所有的Properties
      for (Map.Entry<?, ?> entry : properties.entrySet()) {

        // 读取对应的类型
        String factoryClassName = ((String) entry.getKey()).trim();

        // 通过逗号(,)分割, 读取内部的属性, 依次放入到对应的key下面
        for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
          result.add(factoryClassName, factoryName.trim());
        }
      }
    }

    // 将结果放入到对应classLoader的缓存中(注意这里是依据classLoader进行缓存的划分)
    cache.put(classLoader, result);
    return result;
  }
  catch (IOException ex) {
    throw new IllegalArgumentException("Unable to load factories from location [" +
        FACTORIES_RESOURCE_LOCATION + "]", ex);
  }
}
```

#### 创建SpringFactory实例

```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
    ClassLoader classLoader, Object[] args, Set<String> names) {
  List<T> instances = new ArrayList<>(names.size());

  // 根据类名称依次创建对应的实例
  for (String name : names) {
    try {

      // 获取类元数据信息
      Class<?> instanceClass = ClassUtils.forName(name, classLoader);

      // 判断是是否可以正确进行类型转换
      Assert.isAssignable(type, instanceClass);

      // 获取对应参数的构造函数
      Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);

      // 根据对应的构造函数创建实例
      T instance = (T) BeanUtils.instantiateClass(constructor, args);

      // 记录实例
      instances.add(instance);
    }
    catch (Throwable ex) {
      throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
    }
  }
  return instances;
}
```

### 读取入口类信息

获取main方法入口类信息: 这里使用了一个取巧的方法, 生成一个`RuntimeException`异常, 然后读取内部的堆栈的信息, 以查找 `main` 所在的类信息.

```java
// 获取main方法入口类信息
private Class<?> deduceMainApplicationClass() {
  try {
    StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();

    // 通过遍历当前的栈帧信息, 查找main方法的入口类信息
    for (StackTraceElement stackTraceElement : stackTrace) {
      if ("main".equals(stackTraceElement.getMethodName())) {
        return Class.forName(stackTraceElement.getClassName());
      }
    }
  }
  catch (ClassNotFoundException ex) {
    // Swallow and continue
  }
  return null;
}
```

## SpringApplication的run方法

```java
// run函数
public ConfigurableApplicationContext run(String... args) {
  // 启动计时器开始计时
  StopWatch stopWatch = new StopWatch();
  stopWatch.start();


  ConfigurableApplicationContext context = null;
  Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

  // 配置headless属性(设置java.awt.headless属性值, 默认为true), 即默认是缺少显示设备，键盘或鼠标的系统配置.
  configureHeadlessProperty();

  // 初始化SpringApplicationRunListeners, 详情见下文
  SpringApplicationRunListeners listeners = getRunListeners(args);

  // 依次启动所有的listener
  listeners.starting();

  try {
    // 解析命令行参数, 并将其转换为ApplicationArguments, 详情见下文
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

    // 创建和配置environment信息(environment用于配置信息的管理和基础设施的支持如类型校验, 类型转换等) 详情见下文
    ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

    // 检查并配置 spring.beaninfo.ignore 属性, 用于判断是否对 BeanInfo 类型进行扫描, 默认设置为true, 即默认忽略 BeanInfo 类型
    configureIgnoreBeanInfo(environment);

    // 打印 Banner 信息, 支持图片,文本等配置, 可以通过 setBannerMode 关闭输出
    Banner printedBanner = printBanner(environment);

    // 根据Web应用类型动态创建上下文, 这里创建的为: AnnotationConfigServletWebServerApplicationContext
    // 详情见下文
    context = createApplicationContext();

    // 读取并构造在所有spring.factories中提供的定义好的SpringBootExceptionReporter.class 类型配置类
    // 将创建好的上下文传递作为构造参数传递进去
    exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
        new Class[] { ConfigurableApplicationContext.class }, context);

    // 为生成的上下文进行配置, 注入各类Bean, 读取 source中的 BeanDefinition 等, 详见下文
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);

    // 刷新上下文,详情见下文
    refreshContext(context);

    // 上下文刷新之后, 整理和通知操作(现在空实现)
    afterRefresh(context, applicationArguments);

    // 停止计时器
    stopWatch.stop();

    // 打印启动日志
    if (this.logStartupInfo) {
      new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
    }
    // 通知应用启动成功
    listeners.started(context);

    // 排序并调用所有的 ApplicationRunner, CommandLineRunner
    callRunners(context, applicationArguments);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, listeners);
    throw new IllegalStateException(ex);
  }

  try {
    // 通知运行中
    listeners.running(context);
  }
  catch (Throwable ex) {
    handleRunFailure(context, ex, exceptionReporters, null);
    throw new IllegalStateException(ex);
  }
  return context;
}
```

### 初始化SpringApplicationRunListeners

初始化SpringApplicationRunListeners:

- 获取`spring.factories`中定义的所有`SpringApplicationRunListener`实例
- 存储到`SpringApplicationRunListeners`中的`listeners`中
- 返回新构造的`SpringApplicationRunListeners`实例

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
  // 设置构造函数的参数类型为(SpringApplication, String[])
  Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };

  // 读取spring.factories中预定义好的SpringApplicationRunListener的配置类
  // 使用以下配置的构造参数进行实例化
  // 其中参数的类型为(SpringApplication, String[]),
  // 参数: 第一个参数为本身, 第二参数为传进来的args
  return new SpringApplicationRunListeners(logger,
      getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}

// 将log和listeners的实例存储到本地
SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
  this.log = log;
  this.listeners = new ArrayList<>(listeners);
}
```

### 解析命令行参数

对传递的命令行参数进行解析和转换:

- 使用`CommandLineArgs`解析参数, 并存储到本地`source`中.
- 存储命令行参数`args`到本地的`args`中

```java
// 构造的DefaultApplicationArguments返回类型
public DefaultApplicationArguments(String[] args) {
  Assert.notNull(args, "Args must not be null");

  // 解析args, 并转化为 Source 类型
  this.source = new Source(args);

  // 存储传递的 args 到本地
  this.args = args;
}

// 解析的实现类
public CommandLineArgs parse(String... args) {
  // 将解析的结果存储在CommandLineArgs
  CommandLineArgs commandLineArgs = new CommandLineArgs();

  // 遍历每一个命令行参数, 对字符串进行解析: 分为nonOptionArgs和OptionArgs
  for (String arg : args) {
    if (arg.startsWith("--")) {
      String optionText = arg.substring(2, arg.length());
      String optionName;
      String optionValue = null;
      if (optionText.contains("=")) {
        optionName = optionText.substring(0, optionText.indexOf('='));
        optionValue = optionText.substring(optionText.indexOf('=')+1, optionText.length());
      }
      else {
        optionName = optionText;
      }
      if (optionName.isEmpty() || (optionValue != null && optionValue.isEmpty())) {
        throw new IllegalArgumentException("Invalid argument syntax: " + arg);
      }
      commandLineArgs.addOptionArg(optionName, optionValue);
    }
    else {
      commandLineArgs.addNonOptionArg(arg);
    }
  }
  return commandLineArgs;
}
```

### 准备environment

创建并配置`environment`相关的属性:

- 创建对应类型的`environment`.
- 配置`environment`内部的属性信息
- 通知所有的`SpringApplicationRunListeners`: EnvironmentPrepared.
- 绑定 spring.main
- 更新 PropertySources
- 返回配置好的 `environment`.

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
    ApplicationArguments applicationArguments) {
  // Create and configure the environment

  // SERVLET: StandardServletEnvironment, REACTIVE: StandardReactiveWebEnvironment ,default: StandardEnvironment
  // 这里返回的是 StandardServletEnvironment 实例
  ConfigurableEnvironment environment = getOrCreateEnvironment();

  // 配置环境的属性信息, 详情见下文
  configureEnvironment(environment, applicationArguments.getSourceArgs());

  // 将当前environment中所有的PropertySources存储到configurationProperties的中
  ConfigurationPropertySources.attach(environment);

  // 通知所有的SpringApplicationRunListeners: EnvironmentPrepared
  // 默认没有使用线程池, 会依次触发所有listner的
  listeners.environmentPrepared(environment);

  // 将spring.main与当前创建的实例(上面new出来的实例)进行绑定, 细节待补充
  bindToSpringApplication(environment);

  // 将当前的 StandardServletEnvironment 转换成 StandardEnvironment 类型
  // 这里为true, 进行转换, 细节待补充
  if (!this.isCustomEnvironment) {
    environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
        deduceEnvironmentClass());
  }

  // 同上, 更新environment中的configurationProperties中的属性
  ConfigurationPropertySources.attach(environment);

  // 将处理好的环境返回
  return environment;
}
```

#### 配置environment

配置`environment`相关信息:

- 配置`ConversionService`: 用于类型转换.
- 存储命令行参数到 `propertySources`中
- 存储激活的profile, 到 `activeProfiles`中.

```java
// args 参数为传递的命令行参数
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
  // 配置默认的ConversionService: 用于类型转换的服务.
  if (this.addConversionService) {
    ConversionService conversionService = ApplicationConversionService.getSharedInstance();
    environment.setConversionService((ConfigurableConversionService) conversionService);
  }

  // 将命令行参数存储在 environment 的 propertySources 中
  configurePropertySources(environment, args);

  // 配置激活的配置文件, 这里为 dev, 并记录在 environment 的 activeProfiles 中
  configureProfiles(environment, args);
}
```

### 创建应用上下文

这里创建的上下文为: `AnnotationConfigServletWebServerApplicationContext`

```java
// 根据不同的环境创建不同的上下文, 这里可以知道之前推断上下文为: SERVLET
protected ConfigurableApplicationContext createApplicationContext() {
  Class<?> contextClass = this.applicationContextClass;
  if (contextClass == null) {
    try {
      switch (this.webApplicationType) {
      case SERVLET:
        // AnnotationConfigServletWebServerApplicationContext
        contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
        break;
      case REACTIVE:
        // AnnotationConfigReactiveWebServerApplicationContext
        contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
        break;
      default:
        // AnnotationConfigApplicationContext
        contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
      }
    }
    catch (ClassNotFoundException ex) {
      throw new IllegalStateException(
          "Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
          ex);
    }
  }
  // 生成对应类型的实例
  return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

#### 详解 AnnotationConfigServletWebServerApplicationContext

其中 `AnnotationConfigServletWebServerApplicationContext` 依赖关系为:

![AnnotationConfigServletWebServerApplicationContext](https://image.cjyong.com/annotationconfigservletwebserverapplicationcontext.png))

其中类的简单继承关系为: `DefaultResourceLoader` -> `AbstractApplicationContext` -> `GenericApplicationContext` -> `GenericWebApplicationContext` -> `ServletWebServerApplicationContext` -> `AnnotationConfigServletWebServerApplicationContext`. 从上到下依次构造:

**DefaultResourceLoader.java**:

- 简单的存储 classLoader

```java
public DefaultResourceLoader() {
  this.classLoader = ClassUtils.getDefaultClassLoader();
}
```

**AbstractApplicationContext.java**:

- 存储 `ResourcePatternResolver` , 用于解析路径中的资源, 如 `classpath:/context.xml` , 支持 `Ant-style` , 如: `/WEB-INF/*-context.xm`.

```java
public AbstractApplicationContext() {
  this.resourcePatternResolver = getResourcePatternResolver();
}
```

**GenericApplicationContext.java**:

- 创建默认的`BeanFactory`, 这里默认的类型为: `DefaultListableBeanFactory`. 这里不展开详细讲解 `BeanFactory`, 待细节补充.

```java
public GenericApplicationContext() {
  this.beanFactory = new DefaultListableBeanFactory();
}
```

**GenericWebApplicationContext.java**, **ServletWebServerApplicationContext.java**:

- 无操作

```java
public GenericWebApplicationContext() {
  super();
}

public ServletWebServerApplicationContext() {
}
```

**AnnotationConfigServletWebServerApplicationContext.java**:

- 初始化`AnnotatedBeanDefinitionReader`, 向`BeanFactory`中注册各类`BeanProcessor`用于处理 `@Conditional, @Autowired` 等各类注解的实现.
- 初始化`ClassPathBeanDefinitionScanner`, 用于读取 `ClassPath` 下各类注解组件: 如 `@Component`, `@Repository` 等

```java
public AnnotationConfigServletWebServerApplicationContext() {

  // 注册各类 BeanProcessor , 实现各种注解功能: 如 @Conditional, @Autowired, @PostConstruct, @PreDestroy等 详情见下文
  this.reader = new AnnotatedBeanDefinitionReader(this);

  // 用于读取 ClassPath 下各类注解组件: Component, Repository, Service, Controller, ManagedBean等
  // 读取并生成 BeanDefinition 并放到 BeanFactory 中, 注意这里构造的时候, 并没有进行 scan 扫描并生成. 详情见下文
  this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

##### AnnotatedBeanDefinitionReader

- 向`BeanFactory`中注入各类 `BeanFactory` 用于处理各种Bean内注解: `@Conditional, @Autowired, @PostConstruct, @PreDestroy`.

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
  this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
  Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  Assert.notNull(environment, "Environment must not be null");
  this.registry = registry;

  // 生成ConditionEvaluator用于处理@Conditional注解(满足条件才会进行注册)
  this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);

  // 注册Annotation相关的Processors
  AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}

// actual call
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
    BeanDefinitionRegistry registry, @Nullable Object source) {

  // 将registry拓展为GenericApplicationContext(为BeanDefinitionRegistry的实现类)(类型转换)
  DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);

  // 如果转换成功(是DefaultListableBeanFactory的类型实例)
  // 则更新beanFactory中的dependencyComparator和autowireCandidateResolver
  if (beanFactory != null) {
    if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
      beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
    }
    if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
      beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
    }
  }

  // 生成BeanDefinitionHolder列表, 注BeanDefinitionHolder是BeanDefinition的包装类, 内部包含一个
  // beanDefinition, 一个beanName, 一个aliases数组
  Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

  // 检查是否包含ConfigurationClassPostProcessor的BeanDefinition, 没有则新建一个
  // ConfigurationClassPostProcessor实现了@PostConstruct, @PreDestroy的处理, @Resource的注解注入
  // 详细代码阅读暂时省略, 后面待补充. 参考阅读: https://blog.csdn.net/shenchaohao12321/article/details/81235571
  if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // 检查是否包含internalConfigurationAnnotationProcessor的BeanDefinition, 没有则新建一个
  // AutowiredAnnotationBeanPostProcessor实现了@Inject, @Value, @Autowired注解注入
  // 详细代码阅读暂时省略, 后面待补充.
  if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
  // 如果路径中包含javax.annotation.Resource, 即支持JSR250注释
  // CommonAnnotationBeanPostProcessor提供对JSR250注释的@PostConstruct, @PreDestroy,@Resource支持
  // 详细代码阅读暂时省略, 后面待补充.
  if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // 如果路径中包含javax.persistence.EntityManagerFactory, 即支持JPA相关注解
  // 尝试生成PersistenceAnnotationBeanPostProcessor提供相应的支持
  // 详细代码阅读暂时省略, 后面待补充.
  // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
  if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition();
    try {
      def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
          AnnotationConfigUtils.class.getClassLoader()));
    }
    catch (ClassNotFoundException ex) {
      throw new IllegalStateException(
          "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
    }
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
  }

  // 检查是否包含 EventListenerMethodProcessor 的BeanDefinition, 没有则新建一个
  // EventListenerMethodProcessor实现了 @EventListener 的功能
  // 详细代码阅读暂时省略, 后面待补充. 参考阅读: https://blog.csdn.net/baidu_19473529/article/details/97646739
  if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
  }

  // 检查是否包含 DefaultEventListenerFactory 的BeanDefinition, 没有则新建一个
  // DefaultEventListenerFactory 用于支持 @EventListenerFactory注解, 与 @EventListener 相匹配
  // 详细代码阅读暂时省略, 后面待补充.
  if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
    RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
    def.setSource(source);
    beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
  }

  return beanDefs;
}
```

##### ClassPathBeanDefinitionScanner

- 初始化阶段: 记录需要进行注入的类型到`includeFilters`中, 如 `@Component, @Service, @Repository`等.

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
    Environment environment, @Nullable ResourceLoader resourceLoader) {

  Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  this.registry = registry;

  if (useDefaultFilters) {

    // 添加需要处理的注解: @Component, @Service, @Repository, @Controller, @ManagedBean
    registerDefaultFilters();
  }
  setEnvironment(environment);
  setResourceLoader(resourceLoader);
}
```

```java
protected void registerDefaultFilters() {
  // 注意这里只添加了 Component 注解, 因为 Service, Repository等注解中都包含了Component, 所以一个就可以了
  this.includeFilters.add(new AnnotationTypeFilter(Component.class));

  ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();

  try {
    // 尝试添加 ManagedBean 注解
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
    logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
  }
  try {
    // 尝试添加 Named 注解
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
    logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-330 API not available - simply skip.
  }
}
```

### prepareContext

准备上下文, 主要工作为:

- 存储`environment`
- 向`context`中设置相关依赖的服务.
- 注册`Spring boot`相关的`Bean`
- 加载所有的`source`, 并注册成`BeanDefinition`

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
    SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
  // 将 environment 设置到 context, reader, scanner中
  context.setEnvironment(environment);

  // 自定义处理 context: 注册beanNameGenerator, 存储resourceLoader, 设置conversionService, 详见下文
  postProcessApplicationContext(context);

  // 触发所有的 ApplicationContextInitializer 的 initialize 方法, 详见下文
  applyInitializers(context);

  // 通知SpringApplicationRunListeners: Context Prepared
  // 默认没有使用线程池, 会依次触发所有listner的
  listeners.contextPrepared(context);

  // 输出启动日志
  if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
  }

  // Add boot specific singleton beans
  ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();

  // 注册 SpringBoot 相关的Bean

  // 将 ApplicationArguments 作为 Bean 注入进去
  beanFactory.registerSingleton("springApplicationArguments", applicationArguments);

  // 如果 Banner 不为空, 注入 printedBanner 单例
  if (printedBanner != null) {
    beanFactory.registerSingleton("springBootBanner", printedBanner);
  }

  if (beanFactory instanceof DefaultListableBeanFactory) {
    // 设置 BeanFactory 生成 BeanDefinition 时, 是否允许运行时覆盖, 默认为 false.
    ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
  }

  // Load the sources
  Set<Object> sources = getAllSources();
  Assert.notEmpty(sources, "Sources must not be empty");

  // 从 sources 中加载所有的 BeanDefinition, 包括 xml文件, package文件, CharSequence, Class等, 详见下文
  load(context, sources.toArray(new Object[0]));

  // 通知SpringApplicationRunListeners: Context Loaded
  listeners.contextLoaded(context);
}
```

#### postProcessApplicationContext

预处理`Context`, 主要完成三件事:

- 如果`beanNameGenerator`不为空, 则注册对应的单例.
- 如果`resourceLoader`不为空, 则设置到`context`中.
- 如果需要添加转换服务, 则设置默认的`ConversionService`.

```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
  if (this.beanNameGenerator != null) {
    context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
        this.beanNameGenerator);
  }
  if (this.resourceLoader != null) {
    if (context instanceof GenericApplicationContext) {
      ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
    }
    if (context instanceof DefaultResourceLoader) {
      ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
    }
  }
  if (this.addConversionService) {
    context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
  }
}
```

#### applyInitializers

依次触发所有的 ApplicationContextInitializer 的 initialize 方法(注: 在context refresh 之前先执行),当前有十个:

- SharedMetadataReaderFactoryContextInitializer,
- DelegatingApplicationContextInitializer,
- ContextIdApplicationContextInitializer,
- ConditionEvaluationReportLoggingListener
- ConfigurationWarningsApplicationContextInitializer,
- ServerPortInfoApplicationContextInitializer,
- BootstrapApplicationListener$AncestorInitializer
- PropertySourceBootstrapConfiguration
- EnvironmentDecryptApplicationInitializer
- BootstrapApplicationListener

具体职责细节待补充

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
  for (ApplicationContextInitializer initializer : getInitializers()) {
    Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
        ApplicationContextInitializer.class);
    Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
    initializer.initialize(context);
  }
}
```

#### load source

创建 `BeanDefinitionLoader` 用于加载 `souces` 中的`BeanDefinition`, `BeanDefinitionLoader` 内部为不同的 `source` 准备不同的加载工具, 支持 `xml`, `class`, `groovy`等. 这里默认只有一个`Application.class`. 将 `Application` 类注册到`BeanDefinition`中.

```java
protected void load(ApplicationContext context, Object[] sources) {
  if (logger.isDebugEnabled()) {
    logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
  }
  BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
  if (this.beanNameGenerator != null) {
    loader.setBeanNameGenerator(this.beanNameGenerator);
  }
  if (this.resourceLoader != null) {
    loader.setResourceLoader(this.resourceLoader);
  }
  if (this.environment != null) {
    loader.setEnvironment(this.environment);
  }
  loader.load();
}

public int load() {
  int count = 0;
  for (Object source : this.sources) {
    count += load(source);
  }
  return count;
}

private int load(Object source) {
  Assert.notNull(source, "Source must not be null");
  if (source instanceof Class<?>) {
    return load((Class<?>) source);
  }
  if (source instanceof Resource) {
    return load((Resource) source);
  }
  if (source instanceof Package) {
    return load((Package) source);
  }
  if (source instanceof CharSequence) {
    return load((CharSequence) source);
  }
  throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

### refreshContext

刷新上下文, 这里做了非常多的事:

- 设置 Context 相关属性.
- 生成 `messageSource`.
- 生成和启动 `WebServer`等等

```java
// SpringApplication.java
private void refreshContext(ConfigurableApplicationContext context) {

  // 刷新上下文
  refresh(context);
  if (this.registerShutdownHook) {

    // 注册 shutdown hook
    try {
      context.registerShutdownHook();
    }
    catch (AccessControlException ex) {
      // Not allowed in some environments.
    }
  }
}

protected void refresh(ApplicationContext applicationContext) {
  Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
  ((AbstractApplicationContext) applicationContext).refresh();
}

// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    // 设置启动时间, 更新各类标识, 清空applicationListeners等.
    prepareRefresh();

    // 刷新并返回 BeanFactory, 设置beanFactory的SerializationIdID为: application-1
    // Tell the subclass to refresh the internal bean factory.
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    // 向 BeanFactory 中注入一些 BeanProcessor 和 一些环境相关的 Bean. 详见下文
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      // 更新了 Web相关的属性
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      // 调用注册好的 BeanFactoryPostProcessors, 对 BeanFactory 进行自定义的处理
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      // 检测 BeanFactory 中所有的 BeanPostProcessor, 将其添加到 Factory 中的 BeanProcessor
      registerBeanPostProcessors(beanFactory);

      // Initialize message source for this context.
      // 向 BeanFactory 中注册 messageSource Bean, 用户处理国际化资源处理.
      initMessageSource();

      // Initialize event multicaster for this context.
      // 检查并初始化 全局的事件广播器, 详情见下文
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      // 调用 context 中一些特殊 Bean 的操作, 一个hook
      // GenericWebApplicationContext 在初始化 themeSource
      // ServletWebServerApplicationContext 在这里创建了 WebServer 用于启动应用
      // 详见下文
      onRefresh();

      // Check for listener beans and register them.
      // 注册所有相关的 Listener, 包括之前使用的静态监听器, 已经在 BeanFactory 中定义的
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      // 实例化所有的单例
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
            "cancelling refresh attempt: " + ex);
      }

      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      // 清除 Spring 中一些反射的缓存
      resetCommonCaches();
    }
  }
}
```

#### prepareBeanFactory

向`BeanFactory`中注册相关的`BeanProcessor`和相关的`Bean`.

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // Tell the internal bean factory to use the context's class loader etc.
  // 设置classLoader
  beanFactory.setBeanClassLoader(getClassLoader());

  // 设置默认的 Spring EL 的解析器
  beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));

  // 注册PropertyEditorRegistrar, 用于为各类 PropertyEditorRegistry 注入一致的 PropertyEditor
  // PropertyEditor 用于解析转换Bean中各种类型的属性
  beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

  // Configure the bean factory with context callbacks.
  // 注册 ApplicationContextAwareProcessor, 实现功能: 为所有实现了 EnvironmentAware, EmbeddedValueResolverAware,
  // ResourceLoaderAware, ApplicationEventPublisherAware, MessageSourceAware, ApplicationContextAware
  // 接口的Bean, 注入 context.
  beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

  // 在依赖注入的时候, 忽略以下注解, 因为这些成员变量是通过BeanProcessor进行注入,
  // 如EnvironmentAware, 是通过ApplicationContextAwareProcessor注入
  beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
  beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
  beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
  beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
  beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

  // BeanFactory interface not registered as resolvable type in a plain factory.
  // MessageSource registered (and found for autowiring) as a bean.
  // 为当前上下文, 注册一些特定的类型, 如 BeanFactory, ResourceLoader...
  // 现在只支持 ObjectFactory 类型的提前注入
  beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
  beanFactory.registerResolvableDependency(ResourceLoader.class, this);
  beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
  beanFactory.registerResolvableDependency(ApplicationContext.class, this);

  // Register early post-processor for detecting inner beans as ApplicationListeners.
  // 注册 ApplicationListenerDetector 的BeanProcessor, 
  // 用于检测和处理所有自定义的ApplicationListener
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

  // Detect a LoadTimeWeaver and prepare for weaving, if found.
  // 如果 BeanFactory 中包含 loadTimeWeaver 的Bean, 说明需要进行类加载期织入,
  // 如 AspectJ 支持编译期织入、类加载期织入, 而运行期织入则主要是: CGLib和JDK代理
  if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    // 注册LoadTimeWeaverAwareProcessor的BeanProcessor, 如果类实现了 LoadTimeWeaverAware 接口,
    // 则在 Bean中注入 loadTimeWeaver 的引用
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    // 注册临时的 ClassLoader, 主要用于类加载期织入时使用
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }

  // Register default environment beans.
  // 注册环境相关的 Bean: environment, systemProperties, systemEnvironment.
  if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
  }
  if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
  }
}
```

#### postProcessBeanFactory

将`Web`相关的属性进行更新.

```java
// AnnotationConfigServletWebServerApplicationContext.java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // 设置Web应用相关的属性, 见下文
  super.postProcessBeanFactory(beanFactory);

  // 如果设置了 `basePackages` 则进行扫描, 读取 `BeanDefinition`.
  if (this.basePackages != null && this.basePackages.length > 0) {
    this.scanner.scan(this.basePackages);
  }

  // 如果设置了 annotatedClasses, 则注册进去,后续进行处理
  if (!this.annotatedClasses.isEmpty()) {
    this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
  }
}

// ServletWebServerApplicationContext.java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
  // 添加新的BeanPostProcessor: WebApplicationContextServletContextAwareProcessor
// 用于处理 ServletContextAware 接口
  beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));

  // 忽略 ServletContextAware 类型依赖注入
  beanFactory.ignoreDependencyInterface(ServletContextAware.class);

  // 注册Web应用中各类请求的范围, 如: request, session, globalSession, application等
  registerWebApplicationScopes();
}
```

2.1.6.2

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  // 处理所有注册好的 BeanFactoryPostProcessors, 以及做一些特定的处理
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

  // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
  // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
  // 更新 LoadTimeWeaverAwareProcessor, 为什么再做一次, 待确认.
  if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
  }
}
```

#### registerBeanPostProcessors

对 `BeanFactory` 中所有的 `BeanPostProcessor`, 进行定位, 排序, 并将其添加到 `applicationContext`中

```java
public static void registerBeanPostProcessors(
    ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

  String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

  // Register BeanPostProcessorChecker that logs an info message when
  // a bean is created during BeanPostProcessor instantiation, i.e. when
  // a bean is not eligible for getting processed by all BeanPostProcessors.
  int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
  beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

  // Separate between BeanPostProcessors that implement PriorityOrdered,
  // Ordered, and the rest.
  List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
  List<String> orderedPostProcessorNames = new ArrayList<>();
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      priorityOrderedPostProcessors.add(pp);
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
        internalPostProcessors.add(pp);
      }
    }
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // First, register the BeanPostProcessors that implement PriorityOrdered.
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

  // Next, register the BeanPostProcessors that implement Ordered.
  List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
  for (String ppName : orderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  sortPostProcessors(orderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, orderedPostProcessors);

  // Now, register all regular BeanPostProcessors.
  List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
  for (String ppName : nonOrderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    nonOrderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

  // Finally, re-register all internal BeanPostProcessors.
  sortPostProcessors(internalPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, internalPostProcessors);

  // Re-register post-processor for detecting inner beans as ApplicationListeners,
  // moving it to the end of the processor chain (for picking up proxies etc).
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

#### initApplicationEventMulticaster

检查并初始化 全局的事件广播器, 详情见下文

```java
protected void initApplicationEventMulticaster() {
  ConfigurableListableBeanFactory beanFactory = getBeanFactory();
  if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
    this.applicationEventMulticaster =
        beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
    if (logger.isTraceEnabled()) {
      logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
    }
  }
  else {
    this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
    beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
    if (logger.isTraceEnabled()) {
      logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
          "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
    }
  }
}
```

```java
protected void registerListeners() {
  // Register statically specified listeners first.
  for (ApplicationListener<?> listener : getApplicationListeners()) {
    getApplicationEventMulticaster().addApplicationListener(listener);
  }

  // Do not initialize FactoryBeans here: We need to leave all regular beans
  // uninitialized to let post-processors apply to them!
  String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
  for (String listenerBeanName : listenerBeanNames) {
    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
  }

  // Publish early application events now that we finally have a multicaster...
  Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
  this.earlyApplicationEvents = null;
  if (earlyEventsToProcess != null) {
    for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
      getApplicationEventMulticaster().multicastEvent(earlyEvent);
    }
  }
}
```

#### onRefresh

主要完成以下两件事:

- 设置和初始化 `ThemeSource`.
- 生成`WebServer`

```java
// ServletWebServerApplicationContext.java
protected void onRefresh() {
  // 设置和初始化ThemeSource, 见下文
  super.onRefresh();
  try {
    createWebServer();
  }
  catch (Throwable ex) {
    throw new ApplicationContextException("Unable to start web server", ex);
  }
}

private void createWebServer() {
  // webServer 当前为null
  WebServer webServer = this.webServer;
  // servletContext 当前为null
  ServletContext servletContext = getServletContext();
  if (webServer == null && servletContext == null) {
    // 从 BeanFactory 中找到并实例化 ServletWebServerFactory 类型实例
    // 默认如果存在多个BeanDefinition, 则取第一个
    ServletWebServerFactory factory = getWebServerFactory();

    // 调用 factory的 getWebServer, 获取 WebServer
    // 这里使用的是 UndertowServletWebServerFactory, 细节待补充
    this.webServer = factory.getWebServer(getSelfInitializer());
  }
  else if (servletContext != null) {
    try {
      getSelfInitializer().onStartup(servletContext);
    }
    catch (ServletException ex) {
      throw new ApplicationContextException("Cannot initialize servlet context", ex);
    }
  }

  // 更新 Web 应用相关属性
  initPropertySources();
}

// GenericWebApplicationContext
protected void onRefresh() {
  this.themeSource = UiApplicationContextUtils.initThemeSource(this);
}
```

#### finishBeanFactoryInitialization

实例化剩余的所有单例

```java
/**
  * Finish the initialization of this context's bean factory,
  * initializing all remaining singleton beans.
  */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  // Initialize conversion service for this context.
  // 更新 Context 中的 ConversionService
  if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
      beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
    beanFactory.setConversionService(
        beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
  }

  // Register a default embedded value resolver if no bean post-processor
  // (such as a PropertyPlaceholderConfigurer bean) registered any before:
  // at this point, primarily for resolution in annotation attribute values.
  // 添加 ValueResolver, 用于解析 ${} 占位符
  if (!beanFactory.hasEmbeddedValueResolver()) {
    beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
  }

  // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
  String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
  for (String weaverAwareName : weaverAwareNames) {
    // 实例化所有的 LoadTimeWeaverAware 的Bean, 简单的调用 getBean 方法(懒加载)
    getBean(weaverAwareName);
  }

  // Stop using the temporary ClassLoader for type matching.
  // 清空 TempClassLoader
  beanFactory.setTempClassLoader(null);

  // Allow for caching all bean definition metadata, not expecting further changes.
  // 冻结当前的配置, 生成简单 BeanDefinitionName 的快照
  beanFactory.freezeConfiguration();

  // Instantiate all remaining (non-lazy-init) singletons.
  // 实例化剩下的所有的单例: beanDefinitionNames. (这里直接初始化了)
  beanFactory.preInstantiateSingletons();
}
```

#### finishRefresh

```java
// ServletWebServerApplicationContext.java

protected void finishRefresh() {
  // 见下文
  super.finishRefresh();

  // 启动 webServer(前面 refresh的时候生成好了)
  WebServer webServer = startWebServer();

  // 发布 ServletWebServer 启动完成的通知信息
  if (webServer != null) {
    publishEvent(new ServletWebServerInitializedEvent(webServer, this));
  }
}


// AbstractApplicationContext.java
/**
  * Finish the refresh of this context, invoking the LifecycleProcessor's
  * onRefresh() method and publishing the
  * {@link org.springframework.context.event.ContextRefreshedEvent}.
  */
protected void finishRefresh() {
  // Clear context-level resource caches (such as ASM metadata from scanning).
  // 清除上下文的一些缓存
  clearResourceCaches();

  // Initialize lifecycle processor for this context.
  // 给 BeanFactory 注册和初始化 LifecycleProcessor 单例
  initLifecycleProcessor();

  // Propagate refresh to lifecycle processor first.
  // 调用 LifecycleProcessor 通知和触发所有 LifeCycle Bean 中的 start 方法
  getLifecycleProcessor().onRefresh();

  // Publish the final event.
  // 发布上下文刷新完毕通知
  publishEvent(new ContextRefreshedEvent(this));

  // Participate in LiveBeansView MBean, if active.
  LiveBeansView.registerApplicationContext(this);
}
```

## 总结

可以看出 `Spring Boot` 下的启动是一件非常繁琐的事情, 它在`Spring`框架下的基础上, 执行了非常多的自定义操作: 如读取`spring.factories`, 注册各类`BeanProcessor`, 生成和启动`WebServer`等等.

由于篇幅的问题, 很多地方都是浅尝辄止, 没有深入探究. 本文主要是让人了解到`Spring`启动的概况, 后续细节研究都将基于此文. 该系列文章也将一直持续维护, 更新. 如有任何意见或者错误, 敬请邮件联系.
