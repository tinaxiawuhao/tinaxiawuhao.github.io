---
title: 'Sleuth概述'
date: 2022-07-13 21:28:21
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/ZbHW6UCYN.png
isTop: false
---
## Sleuth概述

Sleuth是SpringCloud中提供的一个分布式链路追踪组件，在设计上大量参考并借用了Google Dapper的设计。

### Span简介

Span在Sleuth中代表一组基本的工作单元，当请求到达各个微服务时，Sleuth会通过一个唯一的标识，也就是SpanId来标记开始通过这个微服务，在当前微服务中执行的具体过程和执行结束。

此时，通过SpanId标记的开始时间戳和结束时间戳，就能够统计到当前Span的调用时间，也就是当前微服务的执行时间。另外，也可以用过Span获取到事件的名称，请求的信息等数据。

**总结：远程调用和Span是一对一的关系，是分布式链路追踪中最基本的工作单元，每次发送一个远程调用服务就会产生一个 Span。Span Id 是一个 64 位的唯一 ID，通过计算 Span 的开始和结束时间，就可以统计每个服务调用所耗费的时间。**

### Trace简介

Trace的粒度比Span的粒度大，Trace主要是由具有一组相同的Trace ID的Span组成的，从请求进入分布式系统入口经过调用各个微服务直到返回的整个过程，都是一个Trace。

也就是说，当请求到达分布式系统的入口时，Sleuth会为请求创建一个唯一标识，这个唯一标识就是Trace Id，不管这个请求在分布式系统中如何流转，也不管这个请求在分布式系统中调用了多少个微服务，这个Trace Id都是不变的，直到整个请求返回。

**总结：一个 Trace 可以对应多个 Span，Trace和Span是一对多的关系。Trace Id是一个64 位的唯一ID。Trace Id可以将进入分布式系统入口经过调用各个微服务，再到返回的整个过程的请求串联起来，内部每通过一次微服务时，都会生成一个新的SpanId。Trace串联了整个请求链路，而Span在请求链路中区分了每个微服务。**

### Annotation简介

Annotation记录了一段时间内的事件，内部使用的重要注解如下所示。

- cs（Client Send）客户端发出请求，标记整个请求的开始时间。
- sr（Server Received）服务端收到请求开始进行处理，通过sr与cs可以计算网络的延迟时间，例如：sr－ cs = 网络延迟（服务调用的时间）。
- ss（Server Send）服务端处理完毕准备将结果返回给客户端， 通过ss与sr可以计算服务器上的请求处理时间，例如：ss - sr = 服务器上的请求处理时间。
- cr（Client Reveived）客户端收到服务端的响应，请求结束。通过cr与cs可以计算请求的总时间，例如：cr - cs = 请求的总时间。

**总结：链路追踪系统内部定义了少量核心注解，用来定义一个请求的开始和结束，通过这些注解，我们可以计算出请求的每个阶段的时间。需要注解的是，这里说的请求，是在系统内部流转的请求，而不是从浏览器、APP、H5、小程序等发出的请求。**

## 项目整合Sleuth

Sleuth提供了分布式链路追踪能力，如果需要使用Sleuth的链路追踪功能，需要在项目中集成Sleuth。

### 最简使用

（1）在每个微服务（用户微服务shop-user、商品微服务shop-product、订单微服务shop-order、网关服务shop-gateway）下的pom.xml文件中添加如下Sleuth的依赖。

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

（2）将项目的application.yml文件备份成application-pre-filter.yml，并将application.yml文件的内容替换为application-sentinel.yml文件的内容，这一步是为了让整个项目集成Sentinel、SpringCloud Gateway和Nacos。application.yml替换后的文件内容如下所示。

```java
server:
  port: 10002
spring:
  application:
    name: server-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    sentinel:
      transport:
        port: 7777
        dashboard: 127.0.0.1:8888
      web-context-unify: false
      eager: true

    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowCredentials: true
            allowedHeaders: "*"
      discovery:
        locator:
          enabled: true
          route-id-prefix: gateway-
```

（3）分别启动Nacos、Sentinel、用户微服务shop-user，商品微服务shop-product，订单微服务shop-order和网关服务shop-gateway，在浏览器中输入链接`localhost:10001/server-order/order/submit_order?userId=1001&productId=1001&count=1`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669367689.png)

（4）分别查看用户微服务shop-user，商品微服务shop-product，订单微服务shop-order和网关服务shop-gateway的控制台输出，每个服务的控制台都输出了如下格式所示的信息。

```java
[微服务名称,TraceID,SpanID,是否将结果输出到第三方平台]
```

具体如下所示。

- 用户微服务shop-user

```java
[server-user,03fef3d312450828,76b298dba54ec579,true]
```

- 商品微服务shop-product

```java
[server-product,03fef3d312450828,41ac8836d2df4eec,true]
[server-product,03fef3d312450828,6b7b3662d63372bf,true]
```

- 订单微服务shop-order

```java
[server-order,03fef3d312450828,cbd935d57cae84f9,true]
```

- 网关服务shop-gateway

```java
[server-gateway,03fef3d312450828,03fef3d312450828,true]
```

可以看到，每个服务都打印出了链路追踪的日志信息，说明引入Sleuth的依赖后，就可以在命令行查看链路追踪情况。

### 抽样采集数据

Sleuth支持抽样采集数据。尤其是在高并发场景下，如果采集所有的数据，那么采集的数据量就太大了，非常耗费系统的性能。通常的做法是可以减少一部分数据量，特别是对于采用Http方式去发送采集数据，能够提升很大的性能。

Sleuth可以采用如下方式配置抽样采集数据。

```java
spring:
  sleuth:
    sampler:
      probability: 1.0
```

### 追踪自定义线程池

Sleuth支持对异步任务的链路追踪，在项目中使用@Async注解开启一个异步任务后，Sleuth会为异步任务重新生成一个Span。但是如果使用了自定义的异步任务线程池，则会导致Sleuth无法新创建一个Span，而是会重新生成Trace和Span。此时，需要使用Sleuth提供的LazyTraceExecutor类来包装下异步任务线程池，才能在异步任务调用链路中重新创建Span。

在服务中开启异步线程池任务，需要使用@EnableAsync。所以，在演示示例前，先在用户微服务shop-user的`io.binghe.shop.UserStarter`启动类上添加@EnableAsync注解，如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 启动用户服的类
 */
@SpringBootApplication
@EnableTransactionManagement(proxyTargetClass = true)
@MapperScan(value = { "io.binghe.shop.user.mapper" })
@EnableDiscoveryClient
@EnableAsync
public class UserStarter {
    public static void main(String[] args){
        SpringApplication.run(UserStarter.class, args);
    }
}
```

#### 演示使用@Async注解开启任务

（1）在用户微服务shop-user的`io.binghe.shop.user.service.UserService`接口中定义一个asyncMethod()方法，如下所示。

```java
void asyncMethod();
```

（2）在用户微服务shop-user的`io.binghe.shop.user.service.impl.UserServiceImpl`类中实现asyncMethod()方法，并在asyncMethod()方法上添加@Async注解，如下所示。

```java
@Async
@Override
public void asyncMethod() {
    log.info("执行了异步任务...");
}
```

（3）在用户微服务shop-user的`io.binghe.shop.user.controller.UserController`类中新增asyncApi()方法，如下所示。

```java
@GetMapping(value = "/async/api")
public String asyncApi() {
    log.info("执行异步任务开始...");
    userService.asyncMethod();
    log.info("异步任务执行结束...");
    return "asyncApi";
}
```

（4）分别启动用户微服务和网关服务，在浏览器中输入链接`http://localhost:10001/server-user/user/async/api`

![](https://tianxiawuhao.github.io/post-images/1658669494937.png)

（5）查看用户微服务与网关服务的控制台日志，分别存在如下日志。

- 用户微服务

```java
[server-user,499d6c7128399ed0,a81bd920de0b07de,true]执行异步任务开始...
[server-user,499d6c7128399ed0,a81bd920de0b07de,true]异步任务执行结束...
[server-user,499d6c7128399ed0,e2f297d512f40bb8,true]执行了异步任务...
```

- 网关服务

```java
[server-gateway,499d6c7128399ed0,499d6c7128399ed0,true]
```

可以看到Sleuth为异步任务重新生成了Span。

#### 演示自定义任务线程池

在演示使用@Async注解开启任务的基础上继续演示自定义任务线程池，验证Sleuth是否为自定义线程池新创建了Span。

（1）在用户微服务shop-user中新建`io.binghe.shop.user.config`包，在包下创建ThreadPoolTaskExecutorConfig类，继承`org.springframework.scheduling.annotation.AsyncConfigurerSupport`类，用来自定义异步任务线程池，代码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description Sleuth异步线程池配置
 */
@Configuration
@EnableAutoConfiguration
public class ThreadPoolTaskExecutorConfig extends AsyncConfigurerSupport {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(10);
        executor.setThreadNamePrefix("trace-thread-");
        executor.initialize();
        return executor;
    }
}
```

（2）以debug的形式启动用户微服务和网关服务，并在`io.binghe.shop.user.config.ThreadPoolTaskExecutorConfig#getAsyncExecutor()`方法中打上断点，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669396054.png)

可以看到，项目启动后并没有进入`io.binghe.shop.user.config.ThreadPoolTaskExecutorConfig#getAsyncExecutor()`方法，说明项目启动时，并不会创建异步任务线程池。

（3）在浏览器中输入链接`http://localhost:10001/server-user/user/async/api`，此时可以看到程序已经执行到`io.binghe.shop.user.config.ThreadPoolTaskExecutorConfig#getAsyncExecutor()`方法的断点位置。

![](https://tianxiawuhao.github.io/post-images/1658669404005.png)

说明异步任务线程池是在调用了异步任务的时候创建的。

接下来，按F8跳过断点继续运行程序，可以看到浏览器上的显示结果如下。

![](https://tianxiawuhao.github.io/post-images/1658669411575.png)

（4）查看用户微服务与网关服务的控制台日志，分别存在如下日志。

- 用户微服务

```java
[server-user,f89f2355ec3f9df1,4d679555674e96a4,true]执行异步任务开始...
[server-user,f89f2355ec3f9df1,4d679555674e96a4,true]异步任务执行结束...
[server-user,0ee48d47e58e2a42,0ee48d47e58e2a42,true]执行了异步任务...
```

- 网关服务

```java
[server-gateway,f89f2355ec3f9df1,f89f2355ec3f9df1,true]
```

可以看到，使用自定义异步任务线程池时，在用户微服务中在执行异步任务时，重新生成了Trace和Span。

**注意对比用户微服务中输出的三条日志信息，最后一条日志信息的TraceID和SpanID与前两条日志都不同。**

#### 演示包装自定义线程池

在自定义任务线程池的基础上继续演示包装自定义线程池，验证Sleuth是否为包装后的自定义线程池新创建了Span。

（1）在用户微服务shop-user的`io.binghe.shop.user.config.ThreadPoolTaskExecutorConfig`类中注入BeanFactory，并在getAsyncExecutor()方法中使用`org.springframework.cloud.sleuth.instrument.async.LazyTraceExecutor()`来包装返回的异步任务线程池，修改后的`io.binghe.shop.user.config.ThreadPoolTaskExecutorConfig`类的代码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description Sleuth异步线程池配置
 */
@Configuration
@EnableAutoConfiguration
public class ThreadPoolTaskExecutorConfig extends AsyncConfigurerSupport {

    @Autowired
    private BeanFactory beanFactory;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(10);
        executor.setThreadNamePrefix("trace-thread-");
        executor.initialize();
        return new LazyTraceExecutor(this.beanFactory, executor);
    }
}
```

（2）分别启动用户微服务和网关服务，在浏览器中输入链接`http://localhost:10001/server-user/user/async/api`

![](https://tianxiawuhao.github.io/post-images/1658669421269.png)
（3）查看用户微服务与网关服务的控制台日志，分别存在如下日志。

- 用户微服务

```java
[server-user,157891cb90fddb65,0a278842776b1f01,true]执行异步任务开始...
[server-user,157891cb90fddb65,0a278842776b1f01,true]异步任务执行结束...
[server-user,157891cb90fddb65,1ba55fd3432b77ae,true]执行了异步任务...
```

- 网关服务

```java
[server-gateway,157891cb90fddb65,157891cb90fddb65,true]
```

可以看到Sleuth为异步任务重新生成了Span。

**综上说明：Sleuth支持对异步任务的链路追踪，在项目中使用@Async注解开启一个异步任务后，Sleuth会为异步任务重新生成一个Span。但是如果使用了自定义的异步任务线程池，则会导致Sleuth无法新创建一个Span，而是会重新生成Trace和Span。此时，需要使用Sleuth提供的LazyTraceExecutor类来包装下异步任务线程池，才能在异步任务调用链路中重新创建Span。**

### 自定义链路过滤器

在Sleuth中存在链路过滤器，并且还支持自定义链路过滤器。

#### 自定义链路过滤器概述

TracingFilter是Sleuth中负责处理请求和响应的组件，可以通过注册自定义的TracingFilter实例来实现一些扩展性的需求。

#### 演示自定义链路过滤器

本案例演示通过过滤器验证只有HTTP或者HTTPS请求才能访问接口，并且在访问的链接不是静态文件时，将traceId放入HttpRequest中在服务端获取，并在响应结果中添加自定义Header，名称为SLEUTH-HEADER，值为traceId。

（1）在用户微服务shop-user中新建`io.binghe.shop.user.filter`包，并创建MyGenericFilter类，继承`org.springframework.web.filter.GenericFilterBean`类，代码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 链路过滤器
 */
@Component
@Order( Ordered.HIGHEST_PRECEDENCE + 6)
public class MyGenericFilter extends GenericFilterBean{

    private Pattern skipPattern = Pattern.compile(SleuthWebProperties.DEFAULT_SKIP_PATTERN);

    private final Tracer tracer;
    public MyGenericFilter(Tracer tracer){
        this.tracer = tracer;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)){
            throw new ServletException("只支持HTTP访问");
        }
        Span currentSpan = this.tracer.currentSpan();
        if (currentSpan == null) {
            chain.doFilter(request, response);
            return;
        }
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = ((HttpServletResponse) response);
        boolean skipFlag = skipPattern.matcher(httpServletRequest.getRequestURI()).matches();
        if (!skipFlag){
            String traceId = currentSpan.context().traceIdString();
            httpServletRequest.setAttribute("traceId", traceId);
            httpServletResponse.addHeader("SLEUTH-HEADER", traceId);
        }
        chain.doFilter(httpServletRequest, httpServletResponse);
    }
}
```

（2）在用户微服务shop-user的`io.binghe.shop.user.controller.UserController`类中新建sleuthFilter()方法，在sleuthFilter()方法中获取并打印traceId，如下所示。

```java
@GetMapping(value = "/sleuth/filter/api")
public String sleuthFilter(HttpServletRequest request) {
    Object traceIdObj = request.getAttribute("traceId");
    String traceId = traceIdObj == null ? "" : traceIdObj.toString();
    log.info("获取到的traceId为: " + traceId);
    return "sleuthFilter";
}
```

（3）分别启动用户微服务和网关服务，在浏览器中输入`http://localhost:10001/server-user/user/sleuth/filter/api`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669436388.png)

查看用户微服务的控制台会输出如下信息。

```java
获取到的traceId为: f63ae7702f6f4bba
```

查看浏览器的控制台，看到在响应的结果信息中新增了一个名称为SLEUTH-HEADER，值为f63ae7702f6f4bba的Header，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669443121.png)

说明使用Sleuth的过滤器可以处理请求和响应信息，并且可以在Sleuth的过滤器中获取到TraceID。