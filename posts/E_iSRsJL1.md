---
title: 'spring启动流程'
date: 2022-09-13 19:28:02
tags: [spring]
published: true
hideInList: false
feature: /post-images/E_iSRsJL1.png
isTop: false
---
Spring的启动流程可以归纳为三个步骤：

- 1、初始化Spring容器，注册内置BeanPostProcessor的BeanDefinition到容器中

- 2、将配置类的BeanDefinition注册到容器中

- 3、调用refresh()方法刷新容器

因为是基于 java-config 技术分析源码，所以这里的入口是 AnnotationConfigApplicationContext ，如果是使用 xml 分析，那么入口即为 ClassPathXmlApplicationContext ，它们俩的共同特征便是都继承了 AbstractApplicationContext 类，而大名鼎鼎的 refresh()便是在这个类中定义的。我们接着分析 AnnotationConfigApplicationContext 类，源码如下：

```java
// 初始化容器
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    // 注册 Spring 内置后置处理器的 BeanDefinition 到容器
    this();
    // 注册配置类 BeanDefinition 到容器
    register(annotatedClasses);
    // 加载或者刷新容器中的Bean
    refresh();
}
```

所以整个Spring容器的启动流程可以绘制成如下流程图：

![](https://tianxiawuhao.github.io/post-images/1663068697343.png)

接着我们主要从这三个入口详细分析一下Spring的启动流程：

### 一、初始化流程：
**1、spring容器的初始化时，通过this()调用了无参构造函数，主要做了以下三个事情：**

（1）实例化BeanFactory【DefaultListableBeanFactory】工厂，用于生成Bean对象
（2）实例化BeanDefinitionReader注解配置读取器，用于对特定注解（如@Service、@Repository）的类进行读取转化成  BeanDefinition 对象，（BeanDefinition 是 Spring 中极其重要的一个概念，它存储了 bean 对象的所有特征信息，如是否单例，是否懒加载，factoryBeanName 等）
（3）实例化ClassPathBeanDefinitionScanner路径扫描器，用于对指定的包目录进行扫描查找 bean 对象
**2、核心代码剖析：**

（1）向容器添加内置组件：org.springframework.context.annotation.AnnotationConfigUtils#registerAnnotationConfigProcessors：

根据上图分析，代码运行到这里时候，Spring 容器已经构造完毕，那么就可以为容器添加一些内置组件了，其中最主要的组件便是 ConfigurationClassPostProcessor 和 AutowiredAnnotationBeanPostProcessor ，前者是一个 beanFactory 后置处理器，用来完成 bean 的扫描与注入工作，后者是一个 bean 后置处理器，用来完成 @AutoWired 自动注入。



### 二、注册SpringConfig配置类到容器中：
1、将SpringConfig注册到容器中：org.springframework.context.annotation.AnnotatedBeanDefinitionReader#doRegisterBean：

这个步骤主要是用来解析用户传入的 Spring 配置类，解析成一个 BeanDefinition 然后注册到容器中，主要源码如下：

```java
<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
		@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
	// 解析传入的配置类，实际上这个方法既可以解析配置类，也可以解析 Spring bean 对象
	AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
	// 判断是否需要跳过，判断依据是此类上有没有 @Conditional 注解
	if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
		return;
	}

	abd.setInstanceSupplier(instanceSupplier);
	ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
	abd.setScope(scopeMetadata.getScopeName());
	String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
	// 处理类上的通用注解
	AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
	if (qualifiers != null) {
		for (Class<? extends Annotation> qualifier : qualifiers) {
			if (Primary.class == qualifier) {
				abd.setPrimary(true);
			}
			else if (Lazy.class == qualifier) {
				abd.setLazyInit(true);
			}
			else {
				abd.addQualifier(new AutowireCandidateQualifier(qualifier));
			}
		}
	}
	// 封装成一个 BeanDefinitionHolder
	for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
		customizer.customize(abd);
	}
	BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
	// 处理 scopedProxyMode
	definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
	 
	// 把 BeanDefinitionHolder 注册到 registry
	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);

}
```

### 三、refresh()容器刷新流程：
refresh()主要用于容器的刷新，Spring 中的每一个容器都会调用 refresh() 方法进行刷新，无论是 Spring 的父子容器，还是 Spring Cloud Feign 中的 feign 隔离容器，每一个容器都会调用这个方法完成初始化。refresh()可划分为12个步骤，其中比较重要的步骤下面会有详细说明。

**1、refresh()方法的源码：org.springframework.context.support.AbstractApplicationContext#refresh：**

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// 1. 刷新前的预处理
		prepareRefresh();

		// 2. 获取 beanFactory，即前面创建的【DefaultListableBeanFactory】
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
	 
		// 3. 预处理 beanFactory，向容器中添加一些组件
		prepareBeanFactory(beanFactory);
	 
		try {
			// 4. 子类通过重写这个方法可以在 BeanFactory 创建并与准备完成以后做进一步的设置
			postProcessBeanFactory(beanFactory);
	 
			// 5. 执行 BeanFactoryPostProcessor 方法，beanFactory 后置处理器
			invokeBeanFactoryPostProcessors(beanFactory);
	 
			// 6. 注册 BeanPostProcessors，bean 后置处理器
			registerBeanPostProcessors(beanFactory);
	 
			// 7. 初始化 MessageSource 组件（做国际化功能；消息绑定，消息解析）
			initMessageSource();
	 
			// 8. 初始化事件派发器，在注册监听器时会用到
			initApplicationEventMulticaster();
	 
			// 9. 留给子容器（子类），子类重写这个方法，在容器刷新的时候可以自定义逻辑，web 场景下会使用
			onRefresh();
	 
			// 10. 注册监听器，派发之前步骤产生的一些事件（可能没有）
			registerListeners();
	 
			// 11. 初始化所有的非单实例 bean
			finishBeanFactoryInitialization(beanFactory);
	 
			// 12. 发布容器刷新完成事件
			finishRefresh();
		}
		...
	}

}
```

我们总结一下refresh()方法每一步主要的功能

#### 1,prepareRefresh()刷新前的预处理：

（1）initPropertySources()：初始化一些属性设置，子类自定义个性化的属性设置方法；
（2）getEnvironment().validateRequiredProperties()：检验属性的合法性
（3）earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>()：保存容器中的一些早期的事件；

#### 2,obtainFreshBeanFactory()：获取在容器初始化时创建的BeanFactory：

（1）refreshBeanFactory()：刷新BeanFactory，设置序列化ID；
（2）getBeanFactory()：返回初始化中的GenericApplicationContext创建的BeanFactory对象，即【DefaultListableBeanFactory】类型

#### 3,prepareBeanFactory(beanFactory)：BeanFactory的预处理工作，向容器中添加一些组件：

（1）设置BeanFactory的类加载器、设置表达式解析器等等
（2）添加BeanPostProcessor【ApplicationContextAwareProcessor】
（3）设置忽略自动装配的接口：EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware；
（4）注册可以解析的自动装配类，即可以在任意组件中通过注解自动注入：BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
（5）添加BeanPostProcessor【ApplicationListenerDetector】
（6）添加编译时的AspectJ；
（7）给BeanFactory中注册的3个组件：environment【ConfigurableEnvironment】、systemProperties【Map<String, Object>】、systemEnvironment【Map<String, Object>】
#### 4,postProcessBeanFactory(beanFactory)：子类重写该方法，可以实现在BeanFactory创建并预处理完成以后做进一步的设置

#### 5,invokeBeanFactoryPostProcessors(beanFactory)：在BeanFactory标准初始化之后执行BeanFactoryPostProcessor的方法，即BeanFactory的后置处理器：

（1）先执行BeanDefinitionRegistryPostProcessor： postProcessor.postProcessBeanDefinitionRegistry(registry)

① 获取所有的实现了BeanDefinitionRegistryPostProcessor接口类型的集合
② 先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor
③ 再执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor
④ 最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessors        
（2）再执行BeanFactoryPostProcessor的方法：postProcessor.postProcessBeanFactory(beanFactory)

① 获取所有的实现了BeanFactoryPostProcessor接口类型的集合
② 先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor
③ 再执行实现了Ordered顺序接口的BeanFactoryPostProcessor
④ 最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor
#### 6,registerBeanPostProcessors(beanFactory)：向容器中注册Bean的后置处理器BeanPostProcessor，它的主要作用是干预Spring初始化bean的流程，从而完成代理、自动注入、循环依赖等功能

（1）获取所有实现了BeanPostProcessor接口类型的集合：
（2）先注册实现了PriorityOrdered优先级接口的BeanPostProcessor；
（3）再注册实现了Ordered优先级接口的BeanPostProcessor；
（4）最后注册没有实现任何优先级接口的BeanPostProcessor；
（5）最r终注册MergedBeanDefinitionPostProcessor类型的BeanPostProcessor：beanFactory.addBeanPostProcessor(postProcessor);
（6）给容器注册一个ApplicationListenerDetector：用于在Bean创建完成后检查是否是ApplicationListener，如果是，就把Bean放到容器中保存起来：applicationContext.addApplicationListener((ApplicationListener<?>) bean);
此时容器中默认有6个默认的BeanProcessor(无任何代理模式下)：【ApplicationContextAwareProcessor】、【ConfigurationClassPostProcessorsAwareBeanPostProcessor】、【PostProcessorRegistrationDelegate】、【CommonAnnotationBeanPostProcessor】、【AutowiredAnnotationBeanPostProcessor】、【ApplicationListenerDetector】

#### 7,initMessageSource()：初始化MessageSource组件，主要用于做国际化功能，消息绑定与消息解析：

（1）看BeanFactory容器中是否有id为messageSource 并且类型是MessageSource的组件：如果有，直接赋值给messageSource；如果没有，则创建一个DelegatingMessageSource；
（2）把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；
#### 8,initApplicationEventMulticaster()：初始化事件派发器，在注册监听器时会用到：

（1）看BeanFactory容器中是否存在自定义的ApplicationEventMulticaster：如果有，直接从容器中获取；如果没有，则创建一个SimpleApplicationEventMulticaster
（2）将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件就可以直接自动注入
#### 9,onRefresh()：留给子容器、子类重写这个方法，在容器刷新的时候可以自定义逻辑

#### 10,registerListeners()：注册监听器：将容器中所有的ApplicationListener注册到事件派发器中，并派发之前步骤产生的事件：

 （1）从容器中拿到所有的ApplicationListener
（2）将每个监听器添加到事件派发器中：getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
（3）派发之前步骤产生的事件applicationEvents：getApplicationEventMulticaster().multicastEvent(earlyEvent);
#### 11,finishBeanFactoryInitialization(beanFactory)：初始化所有剩下的非单实例bean，核心方法是preInstantiateSingletons()，会调用getBean()方法创建对象；

（1）获取容器中的所有beanDefinitionName，依次进行初始化和创建对象
（2）获取Bean的定义信息RootBeanDefinition，它表示自己的BeanDefinition和可能存在父类的BeanDefinition合并后的对象
（3）如果Bean满足这三个条件：非抽象的，单实例，非懒加载，则执行单例Bean创建流程：    
（4）所有Bean都利用getBean()创建完成以后，检查所有的Bean是否为SmartInitializingSingleton接口的，如果是；就执行afterSingletonsInstantiated()；

#### 12,finishRefresh()：发布BeanFactory容器刷新完成事件：

（1）initLifecycleProcessor()：初始化和生命周期有关的后置处理器：默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】，如果没有，则创建一个DefaultLifecycleProcessor()加入到容器；
（2）getLifecycleProcessor().onRefresh()：拿到前面定义的生命周期处理器（LifecycleProcessor）回调onRefresh()方法
（3）publishEvent(new ContextRefreshedEvent(this))：发布容器刷新完成事件；
（4）liveBeansView.registerApplicationContext(this);
