---
title: 'springboot启动流程'
date: 2022-08-29 20:37:52
tags: [springBoot]
published: true
hideInList: false
feature: /post-images/Tm9S6oyiS.png
isTop: false
---
SpringBoot 启动流程

![](https://tianxiawuhao.github.io/post-images/1661776742013.png)

### new SpringApplication阶段

一般的springboot的启动类长这个样子。

```java
@SpringBootApplication
public class SampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SampleApplication.class, args);
    }

}
```

可以看到这个main方法里面只有简单的一行code，这一行就是理解springboot启动过程的入口了。点进去看一看它做了什么事情，大致上可以认为分为两个部分，首先是new了一个SpringApplication对象，然后调用了它的run方法。

```java
/**
 * Static helper that can be used to run a {@link SpringApplication} from the
 * specified source using default settings.
 * @param primarySource the primary source to load
 * @param args the application arguments (usually passed from a Java main method)
 * @return the running {@link ApplicationContext}
 */
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[] { primarySource }, args);
}

/**
 * Static helper that can be used to run a {@link SpringApplication} from the
 * specified sources using default settings and user supplied arguments.
 * @param primarySources the primary sources to load
 * @param args the application arguments (usually passed from a Java main method)
 * @return the running {@link ApplicationContext}
 */
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```
new SpringApplication 阶段做了哪些事情
结合code我们看一下在new SpringApplication时，SpringBoot框架做了哪些事情

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //设置primarySources
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //确定application类型
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //读取配置在spring.factories文件中的ApplicationContextInitializer，需要注意一下，在这个方法里面，springboot会开始读取spring.factories里面的配置并且根据ClassLoader的不同，缓存到内存中。这个之后再展开讲。
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //读取配置在spring.factories文件中的ApplicationListener。
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //确定入口类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```
#### 确定应用程序类型(servlet)
首先就是确定一下我们这个项目的类型
判断方法比较简单粗暴，就是看一下你项目中有没有对应类型的标志性的类。举个例子，如果你有WEBFLUX_INDICATOR_CLASS这个类的话，就认为你的项目类型是REACTIVE。

```java
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

private static final String SERVLET_APPLICATION_CONTEXT_CLASS = "org.springframework.web.context.WebApplicationContext";

private static final String REACTIVE_APPLICATION_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.ReactiveWebApplicationContext";

static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
        && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```
#### spi加载spring.factories

然后就是通过SPI机制获取spring.factories的配置
在构造器中，我们可以很明显的看到有两次对于getSpringFactoriesInstances方法的调用，这个方法兜兜转转，最后会调用到SpringFactoriesLoader的loadSpringFactories的方法。在这个方法中，对于相同的classsloader，只会尝试load一次spring.factories文件中的配置。

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        Enumeration<URL> urls = (classLoader != null ?
                                 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```
#### 加载ApplicationContextInitializer(初始化器)和ApplicationListener(监听器)

从load好的spring.factorie中获取配置好的ApplicationContextInitializer和ApplicationListener
这个就没啥好说了。需要注意的是，这些ApplicationListener是在调用run方法之前就已经加载进来的，所以这些ApplicationListener不仅仅能够接收ApplicationStartedEvent以及之后的ApplicationEvent，之前的比如ApplicationStartingEvent、ApplicationEnvironmentPreparedEvent等等事件类型也都可以接收到。

确定主类
这个的逻辑也比较简单，首先获取当前的方法调用栈（这点还是有疑问为啥不通过Thread.currentThread().getStackTrace()这种看起来更加正式的方法来获取stackTrace）。然后从栈顶一个一个找下去，直到找到一个名字为main的方法，这个方法所对应的类就可以认为是主类了。

 ```java
 private Class<?> deduceMainApplicationClass() {
     try {
         StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
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

为了验证我也写个了sample

```java
public class Test {
    public static void main(String[] args) {
        SampleApplication.main(args);
    }
}

@SpringBootApplication
public class SampleApplication {
    public static void main(String[] args) {
    	SpringApplication.run(SampleApplication.class, args);
    }
}
```


发现调用Test.main方法之后，定位出来的mainclass确实是SampleApplication。

### run()方法

run()方法做了哪些事情
还是先看这些稍微加了点注释的源码

```java
public ConfigurableApplicationContext run(String... args) {

    //起了个timer，用于计时，不用管
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    //这个就是配置个系统环境变量 java.awt.headless
    configureHeadlessProperty();
    //获取SpringApplicationRunListeners，这个和之前提到的ApplicationListener是两个不同的接口。两者间的联系在于，
    //在springboot的默认实现中，有一个特殊的类EventPublishingRunListener会继承SpringApplicationRunListeners，
    //他会在springboot的特殊阶段发布对应的ApplicationEvent，并且被ApplicationListener接收。
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //进入starting阶段，发布ApplicationStartingEvent事件。
    listeners.starting();
    try {
        //包装启动参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        //准备environment，确定propertiesSource。发布ApplicationEnvironmentPreparedEvent。
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        //配置环境变量spring.beaninfo.ignore
        configureIgnoreBeanInfo(environment);
        Banner printedBanner = printBanner(environment);
        //创建应用上下文
        context = createApplicationContext();
        //获取SpringBootExceptionReporter
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                         new Class[] { ConfigurableApplicationContext.class }, context);
        //发布ApplicationContextInitializedEvent事件和ApplicationPreparedEvent事件
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        /划重点，进行bean的初始化工作
            refreshContext(context);
        //暂时是个空function。
        afterRefresh(context, applicationArguments);
        //关闭计算器，也就是前面创建的那个timer
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        //发布ApplicationStartedEvent事件
        listeners.started(context);
        //运行ApplicationRunner和CommandLineRunner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {

        //发布ApplicationReadyEvent事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```



可以看到这个方法比较复杂，基本上每一行语句都对应着一个函数。为了更加方便地捋清楚整个flow，我们跟着SpringApplicationRunListener，看一下它的每个方法都在什么时候被trigger，两个方法之间都做了什么事情。

首先看一下SpringApplicationRunListener(s)有哪些方法
SpringApplicationRunListener并没有直接被SpringApplication调用，而是作为一个整体SpringApplicationRunListeners被统一调用，它有starting、environmentPrepared、contextPrepared、contextLoaded、started、running、failed等七个方法。由于这一篇我们只讲SpringBoot的启动流程，因此只讨论前六个方法，看他们的调用时机。

```java
public interface SpringApplicationRunListener {

    /**
     * Called immediately when the run method has first started. Can be used for very
     * early initialization.
     */
    default void starting() {
    }
    
    /**
     * Called once the environment has been prepared, but before the
     * {@link ApplicationContext} has been created.
     * @param environment the environment
     */
    default void environmentPrepared(ConfigurableEnvironment environment) {
    }
    
    /**
     * Called once the {@link ApplicationContext} has been created and prepared, but
     * before sources have been loaded.
     * @param context the application context
     */
    default void contextPrepared(ConfigurableApplicationContext context) {
    }
    
    /**
     * Called once the application context has been loaded but before it has been
     * refreshed.
     * @param context the application context
     */
    default void contextLoaded(ConfigurableApplicationContext context) {
    }
    
    /**
     * The context has been refreshed and the application has started but
     * {@link CommandLineRunner CommandLineRunners} and {@link ApplicationRunner
     * ApplicationRunners} have not been called.
     * @param context the application context.
     * @since 2.0.0
     */
    default void started(ConfigurableApplicationContext context) {
    }
    
    /**
     * Called immediately before the run method finishes, when the application context has
     * been refreshed and all {@link CommandLineRunner CommandLineRunners} and
     * {@link ApplicationRunner ApplicationRunners} have been called.
     * @param context the application context.
     * @since 2.0.0
     */
    default void running(ConfigurableApplicationContext context) {
    }
    
    /**
     * Called when a failure occurs when running the application.
     * @param context the application context or {@code null} if a failure occurred before
     * the context was created
     * @param exception the failure
     * @since 2.0.0
     */
    default void failed(ConfigurableApplicationContext context, Throwable exception) {
    }

}
```

在starting方法之前
springboot在调用run方法后，基本上第一时间就trigger了listener的starting方式，在这之前，基本上没做啥事。

```java
StopWatch stopWatch = new StopWatch();
stopWatch.start();
ConfigurableApplicationContext context = null;
Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
configureHeadlessProperty();
SpringApplicationRunListeners listeners = getRunListeners(args);
```
#### 开启计时器

一个是创建了一个timer（StopWatch）用来之后计算启动耗时，下面这段日志就是它功能的体现。

```java
2021-09-22 16:29:38.241  INFO 8044 --- [           main] com.sample.app.SampleApplication       : Started SampleApplication in 32.023 seconds (JVM running for 121.055)
```

#### headless=true 设置无外接设备启动

一个是调用configureHeadlessProperty方法来设置java.awt.headless的系统环境变量，这个默认值是true。

 ```java
  private boolean headless = true;
 
   private void configureHeadlessProperty() {
       System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
               System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
   }
 ```

#### 获取并启用监听器

再有就是从之前解析好的spring.factories中读取所有的SpringApplicationRunListeners。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger,
            getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```
在starting方法
EventPublishingRunListener发布ApplicationStartingEvent事件

```java
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```
订阅了这个事件的ApplicationListener倒是找到了几个，除了LogbackLoggingSystem进行了一些与log相关的简单操作外，都没做啥事。

```java
LiquibaseServiceLocatorApplicationListener
DelegatingApplicationListener
LogbackLoggingSystem
BackgroundPreinitializer
```

#### 设置应用程序参数

staring之后，environmentPrepared之前
对应的代码在run方法里对应两行，其中第一行是把系统参数包装一下，使得用起来更加方便。第二行则是进行一些环境的准备工作。

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
```
#### 准备环境变量

我们重点看prepareEnvironment这个方法中做了哪些工作。

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```
可以清楚的看到，在调用listeners.environmentPrepared(environment);方法之前，只有三行code，分别做了三件事情。
getOrCreateEnvironment根据应用类型来创建ConfigurableEnvironment

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
    case SERVLET:
        return new StandardServletEnvironment();
    case REACTIVE:
        return new StandardReactiveWebEnvironment();
    default:
        return new StandardEnvironment();
    }
}
```
configureEnvironment 来配置propertiesSource和profiles
这里面其实做了三件事情，除了上面说的propertiesSource和profiles之外还有对conversionService的配置，但是我还不清楚conversionService这个东西到底是干啥用的，所以先空着吧。先解释剩下的两个吧。

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
       if (this.addConversionService) {
           ConversionService conversionService = ApplicationConversionService.getSharedInstance();
           environment.setConversionService((ConfigurableConversionService) conversionService);
       }
       configurePropertySources(environment, args);
       configureProfiles(environment, args);
   }
```

其中，configurePropertySources是配置environment中的propertySources属性。propertySources是用于记录系统的配置参数可以从哪里读取，各个propertySource在其中的先后顺序体现了他们彼此的优先级。默认的propertySources包括systemProperties和systemEnvironment两个。在configurePropertySources步骤中根据代码或者参数的不同，可能会额外加上defaultProperties和commandLineArgs两个propertySource。

```java
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(
                    new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
```
configureProfiles则是试图从现有的propertySources中尝试获取spring.profiles.active参数。需要注意的是，此时的propertySources并不包含与你的配置文件（比如application.properties）相对应的propertySource，所以无论你的配置文件怎么配置，对这个阶段都不造成影响。

```java
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```
ConfigurationPropertySources.attach(environment)将自己置于propertySources的第一个的位置
前面也讲了，各个PropertySource在propertySources中的先后顺序决定了优先级，现在把ConfigurationPropertySources放到第一位，也就意味着它的优先级是最高的。细心的人应该也会注意到ConfigurationPropertySources.attach(environment)这一行前后调用了两次，这个也是为了确保ConfigurationPropertySources优先加载的顺序。

```java
public static void attach(Environment environment) {
    Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
    MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
    PropertySource<?> attached = sources.get(ATTACHED_PROPERTY_SOURCE_NAME);
    if (attached != null && attached.getSource() != sources) {
        sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
        attached = null;
    }
    if (attached == null) {
        sources.addFirst(new ConfigurationPropertySourcesPropertySource(ATTACHED_PROPERTY_SOURCE_NAME,
                new SpringConfigurationPropertySources(sources)));
    }
}
```
在environmentPrepared方法
EventPublishingRunListener发布ApplicationEnvironmentPreparedEvent事件

```java
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```
订阅了这个事件的ApplicationListener也找到了几个，我们先记录下来。

```java
ConfigFileApplicationListener
AnsiOutputApplicationListener
LoggingApplicationListener
ClasspathLoggingApplicationListener
BackgroundPreinitializer
DelegatingApplicationListener
FileEncodingApplicationListener
```

其中比较有意思的是ConfigFileApplicationListener，因此我们展开讲一下，剩下的等有空了再说。
在接收到ApplicationEnvironmentPreparedEvent之后，做了这件事

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {

    // 加载所有EnvironmentPostProcessor并排序
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    postProcessors.add(this);
    AnnotationAwareOrderComparator.sort(postProcessors);
    //依次调用EnvironmentPostProcessor的postProcessEnvironment方法
    for (EnvironmentPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }
}
```

在这块逻辑中，会调用它自身的postProcessEnvironment方法，从而实现配置文件的加载

```java
@Override
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    addPropertySources(environment, application.getResourceLoader());
}

/**
    * Add config file property sources to the specified environment.
    * @param environment the environment to add source to
    * @param resourceLoader the resource loader
    * @see #addPostProcessors(ConfigurableApplicationContext)
    */
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    RandomValuePropertySource.addToEnvironment(environment);
    new Loader(environment, resourceLoader).load();
}
```

其中，loader的load方法我们也记录一下

```java
void load() {
    FilteredPropertySource.apply(this.environment, DEFAULT_PROPERTIES, LOAD_FILTERED_PROPERTY,
                                 (defaultProperties) -> {
                                     this.profiles = new LinkedList<>();
                                     this.processedProfiles = new LinkedList<>();
                                     this.activatedProfiles = false;
                                     this.loaded = new LinkedHashMap<>();
                                     initializeProfiles();
                                     while (!this.profiles.isEmpty()) {
                                         Profile profile = this.profiles.poll();
                                         if (isDefaultProfile(profile)) {
                                             addProfileToEnvironment(profile.getName());
                                         }
                                         load(profile, this::getPositiveProfileFilter,
                                              addToLoaded(MutablePropertySources::addLast, false));
                                         this.processedProfiles.add(profile);
                                     }
                                     load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
                                     addLoadedPropertySources();
                                     applyActiveProfiles(defaultProperties);
                                 });
}
```
#### 忽略bean信息

```java
 private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
		if (System.getProperty(
				CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
			Boolean ignore = environment.getProperty("spring.beaninfo.ignore",
					Boolean.class, Boolean.TRUE);
			System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME,
					ignore.toString());
		}
 }
```

#### 打印banner信息

显而易见，这个流程就是用来打印控制台那个很大的spring的banner的，就是下面这个东东
![](https://tianxiawuhao.github.io/post-images/1661777022899.png)
那他在哪里打印的呢？他在 SpringBootBanner.java 里面打印的，这个类实现了Banner 接口，

而且banner信息是直接在代码里面写死的；
![](https://tianxiawuhao.github.io/post-images/1661777039318.png)
有些公司喜欢自定义banner信息，如果想要改成自己喜欢的图标该怎么办呢，其实很简单,只需要在resources目录下添加一个 banner.txt 的文件即可，一定要添加到resources目录下，别加错了

#### 创建应用程序上下文

environmentPrepared之后，contextPrepared之前
这部分在SpringApplication对应的代码主要是这么三行

```java
//初始化
context = createApplicationContext();
exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                 new Class[] { ConfigurableApplicationContext.class }, context);
prepareContext(context, environment, listeners, applicationArguments, printedBanner);
```

第一行是根据应用类型，创建ApplicationContext的实例

```java
protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
                }
            }
            catch (ClassNotFoundException ex) {
                throw new IllegalStateException(
                        "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
            }
        }
        return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
```

#### 实例化异常报告器

第二行就是获取SpringBootExceptionReporter的实例。

异常报告器是用来捕捉全局异常使用的，当springboot应用程序在发生异常时，异常报告器会将其捕捉并做相应处理，在spring.factories 文件里配置了默认的异常报告器，

![](https://tianxiawuhao.github.io/post-images/1661777103389.png)
需要注意的是，这个异常报告器只会捕获启动过程抛出的异常，如果是在启动完成后，在用户请求时报错，异常报告器不会捕获请求中出现的异常，
![](https://tianxiawuhao.github.io/post-images/1661777116513.png)
了解原理了，接下来我们自己配置一个异常报告器来玩玩；

MyExceptionReporter.java 继承 SpringBootExceptionReporter 接口

```java
package com.spring.application;
 
import org.springframework.boot.SpringBootExceptionReporter;
import org.springframework.context.ConfigurableApplicationContext;
 
public class MyExceptionReporter implements SpringBootExceptionReporter {
 
 
    private ConfigurableApplicationContext context;
    // 必须要有一个有参的构造函数，否则启动会报错
    MyExceptionReporter(ConfigurableApplicationContext context) {
        this.context = context;
    }
 
    @Override
    public boolean reportException(Throwable failure) {
        System.out.println("进入异常报告器");
        failure.printStackTrace();
        // 返回false会打印详细springboot错误信息，返回true则只打印异常信息 
        return false;
    }
}
```

在 spring.factories 文件中注册异常报告器

```bash
# Error Reporters 异常报告器
org.springframework.boot.SpringBootExceptionReporter=\
com.spring.application.MyExceptionReporter
```

![](https://tianxiawuhao.github.io/post-images/1661777135828.png)
接着我们在application.yml 中 把端口号设置为一个很大的值，这样肯定会报错，

```bash
server:
  port: 80828888
```

启动后，控制台打印如下图
![](https://tianxiawuhao.github.io/post-images/1661777149995.png)

#### 准备上下文环境

第三行所对应的方法比较复杂，和我们这个阶段对应的主要是这么几行。

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
            ....
}
```
其中applyInitializers(context)这行就是对我们在new SpringApplication()时找到的ApplicationContextInitializer进行初始化工作。这部分包含两个特殊的ApplicationContextInitializer，他们会增加对应的BeanFactoryPostProcessor。BeanFactoryPostProcessor这个类比较重要，它可以读取并覆盖应用程序上下文中配置的bean属性。

```java
class SharedMetadataReaderFactoryContextInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
...
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.addBeanFactoryPostProcessor(new CachingMetadataReaderFactoryPostProcessor());
    }
...
}

public class ConfigurationWarningsApplicationContextInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {
...
    @Override
    public void initialize(ConfigurableApplicationContext context) {
        context.addBeanFactoryPostProcessor(new ConfigurationWarningsPostProcessor(getChecks()));
    }
...
}
```

在contextPrepared方法
EventPublishingRunListener发布ApplicationContextInitializedEvent事件

```java
public void contextPrepared(ConfigurableApplicationContext context) {
    this.initialMulticaster
            .multicastEvent(new ApplicationContextInitializedEvent(this.application, this.args, context));
}
```
订阅了这个事件的ApplicationListener只找到个两个，我们先记录下来。都没做啥事

```java
BackgroundPreinitializer
DelegatingApplicationListener
```

contextPrepared 之后，contextLoaded之前
这个部分code都被包含在prepareContext这个SpringApplication的私有方法中了。


其中比较重要的load(context, sources.toArray(new Object[0]))这一行。这一行会为之后bean的扫描路径做好准备。

在contextLoaded方法
EventPublishingRunListener将listener注册到创建好的ApplicationContext之中，之后发布ApplicationPreparedEvent事件。

```java
public void contextLoaded(ConfigurableApplicationContext context) {
    for (ApplicationListener<?> listener : this.application.getListeners()) {
        if (listener instanceof ApplicationContextAware) {
            ((ApplicationContextAware) listener).setApplicationContext(context);
        }
        context.addApplicationListener(listener);
    }
    this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
}
```
订阅了这个事件的ApplicationListener，我们记录下来。其中比较特殊的是ConfigFileApplicationListener，它注册了一个PropertySourceOrderingPostProcessor的BeanFactoryPostProcessor实例到context中。

```java
CloudFoundryVcapEnvironmentPostProcessor
ConfigFileApplicationListener
LoggingApplicationListener
BackgroundPreinitializer
DelegatingApplicationListener
```

#### 刷新上下文

contextLoaded之后，started之前
这个在run方法中只对应了一行代码refreshContext(context);。它会间接的对ApplicationContext的refresh方法进行调用。由于我们现在debug的是一个web项目，因此走进去对应的ApplicationContext的实现类是ServletWebServerApplicationContext。它的refresh方法可以简单的视为对父类refresh方法的调用，它的onRefresh方法则在父类(AbstractApplicationContext)的方法之上创建了WebServer。

```java
@Override
public final void refresh() throws BeansException, IllegalStateException {
    try {
        super.refresh();
    }
    catch (RuntimeException ex) {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.stop();
        }
        throw ex;
    }
}

@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start web server", ex);
    }
}
```
我们看一下在父类的refresh方法中做了哪些事情

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        //这个步骤会清除缓存，检查是否缺少环境参数，保存前期加载的ApplicationListener实例(从applicationListeners属性到earlyApplicationListeners)
        // Prepare this context for refreshing.
        prepareRefresh();
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        //这个步骤通过调用ignoreDependencyInterface，registerResolvableDependency，addBeanPostProcessor等方法完成beanFactory的一些配置。其中addBeanPostProcessor方法会被调用至少两次，传参分别为ApplicationContextAwareProcessor和ApplicationListenerDetector
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);
        //这个步骤会清除缓存，检查是否缺少环境参数，保存前期加载的ApplicationListener实例(从applicationListeners属性到earlyApplicationListeners)
        // Prepare this context for refreshing.
        prepareRefresh();
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        //这个步骤通过调用ignoreDependencyInterface，registerResolvableDependency，addBeanPostProcessor等方法完成beanFactory的一些配置。其中addBeanPostProcessor方法会被调用至少两次，传参分别为ApplicationContextAwareProcessor和ApplicationListenerDetector
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);
        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);
            //这个步骤会加载bean的definition
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();
            //Web类型的App将在这个步骤创建WebServer，但是WebServer此时并不会立刻启动
            // Initialize other special beans in specific context subclasses.
            onRefresh();
            //注册监听
            // Check for listener beans and register them.
            registerListeners();
            //这个时候会根据之前获取的的bean的definition，进行bean的初始化
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);
            //Web类型的App将在这个步骤启动之前创建好的WebServer
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
            resetCommonCaches();
        }
    }
}
```
#### 刷新上下文后置处理

afterRefresh 方法是启动后的一些处理，留给用户扩展使用，目前这个方法里面是空的，

```java
/**
	 * Called after the context has been refreshed.
	 * @param context the application context
	 * @param args the application arguments
	 */
	protected void afterRefresh(ConfigurableApplicationContext context,
			ApplicationArguments args) {
	}
```

#### 结束计时器

#### 发布上下文准备就绪事件

started之后，running之前
这个部分就比较简单了，主要是invoke了ApplicationRunner和CommandLineRunner

```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```

这是一个扩展功能，callRunners(context, applicationArguments) 可以在启动完成后执行自定义的run方法；有2中方式可以实现：

1. 实现 ApplicationRunner 接口
2. 实现 CommandLineRunner 接口

接下来我们验证一把，为了方便代码可读性，我把这2种方式都放在同一个类里面

```java
package com.spring.init;
 
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;
 
/**
 * 自定义run方法的2种方式
 */
@Component
public class MyRunner implements ApplicationRunner, CommandLineRunner {
 
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(" 我是自定义的run方法1，实现 ApplicationRunner 接口既可运行"        );
    }
 
    @Override
    public void run(String... args) throws Exception {
        System.out.println(" 我是自定义的run方法2，实现 CommandLineRunner 接口既可运行"        );
    }
}
```

启动springboot后就可以看到控制台打印的信息了
![](https://tianxiawuhao.github.io/post-images/1661777169058.png)
