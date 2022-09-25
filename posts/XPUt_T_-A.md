---
title: 'Sentinel限流熔断降级——知识点总结'
date: 2022-07-06 20:43:19
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/XPUt_T_-A.png
isTop: false
---
> Sentinei官网地址：https://sentinelguard.io/zh-cn/docs/quick-start.html
>
> Sentinel Github地址：https://github.com/alibaba/Sentinel/wiki
>
> Sentinel生产级使用：https://github.com/alibaba/Sentinel/wiki/在生产环境中使用-Sentinel

### 一 Sentinel是什么,为什么要引入Sentinel  

#### 1 什么是“服务雪崩”？ 

在微服务架构中，存在着大量的服务间调用，有时候，由于某个服务发生了故障，而**导致调用它的服务的资源得不到正常释放**，从而也发生故障，故障不断传播，最终导致整个微服务无法对外提供服务，这就发生了“服务雪崩”！

![](https://tianxiawuhao.github.io/post-images/1658666676257.png)

####  2 “服务雪崩”如何解决？ 

解决“服务雪崩”的核心点有2个：尽量控制流量，不要让服务因为过载还出现故障；当一个服务已经出现故障时，不要让故障在微服务整个调用链路中进行传播！

针对上面两点，“服务雪崩”就有了常见的2大类解决方案：

- 流量控制，避免出现故障：

- - **流量控制**：在微服务调用链路的各个核心资源点，进行流量控制，超出的流量直接不要放行，以对目标资源进行直接保护！

- 出现故障时，防止故障传播：

- - **超时处理**：设置超时时间，超过设定时间就返回报错信息，做对应处理，防止“无休止”等待！
  - **服务隔离**：限定每个业务能使用的线程数，避免耗尽整个服务的所有线程资源，保护其他业务！
  - **熔断降级**：通过断路器统计服务调用的“异常数”或“异常比例”，如果超出设定阈值，则直接熔断，后续一定时间内的请求到达时，直接返回降级处理的结果，而不真正去调用目标服务！

而上面所说的这一系列解决方案，Sentinel都为我们提供了实现，所以我们选择Sentinel！

####  3 阿里的Sentinel和网飞的Hystrix的对比？ 

![](https://tianxiawuhao.github.io/post-images/1658666694073.png)

> 其实很明显，Sentinel比Hystrix强大很多，而且Hystrix已经进入维护期不再更新了，以后的微服务项目中肯定是选择Sentinel！

------

###  二 Sentinel的简单使用和 

首先虽然Sentinel是作为SpringCloud Alibaba的重要组件而出名，但是并不是强依赖于SpringCloud Alibaba！

独立使用Sentinel文档：https://sentinelguard.io/zh-cn/docs/quick-start.html

在spring-cloud-alibaba框架中使用：https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel

####  1 在spring-cloud-alibaba框架中的简单使用Sentinel 

- 引入依赖starter：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

- 启动Sentinel Dashboard控制台：

https://sentinelguard.io/zh-cn/docs/dashboard.html

```shell
# Sentinel的启动非常简单，java -jar sentinel-dashboard.jar 运行即可：
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

- 在我们的项目文件中配置sentinel-dashboard控制台的地址：

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
```

####  2 开启Sentinel对Feign调用的支持 

除了上面的依赖和配置外，我们还需要增加一下配置：

如果我们引入的Feign是通过 spring-cloud-starter-openfeign 引入Feign的，那么 Sentinel starter 中的自动配置类就会生效！

- 此时，我们只需要在 application.yml 配置文件中开启 Sentinel 对 Feign 的支持即可：

```yaml
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```

####  3 Sentinel Dashboard 的配置持久化到 Nacos 配置中心 

https://github.com/alibaba/Sentinel/wiki/在生产环境中使用-Sentinel

Sentinel Dashboard中的所有配置在默认情况下，刷新后就会丢失，那么实际生产中肯定是可允许的，所以我们必须要想办法将Sentinel的配置持久化；

**而正好Nacos就提供了配置的“持久化”与“监听机制”，所以我们就选择通过 Nacos 完成 Sentinel 的配置持久化！（push模式，也是生产中最常用的）**

- 在项目的pom.xml中增加 sentinel-datasource-nacos 的依赖，开启对nacos的配置中心的监听：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

- 在 application.yml 中增加 sentinel 的数据源配置：

```xml
spring:
  application:
    name: orderservice
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080 # sentinel控制台地址
      web-context-unify: false # 关闭context整合
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848 # nacos地址
            dataId: orderservice-flow-rules
            groupId: SENTINEL_GROUP
            rule-type: flow # 还可以是：degrade、authority、param-flow
             
feign:
  sentinel:
    enabled: true # 开启feign对sentinel的支持
```

####  4 Sentinel的三种规则配置管理模式 

- **原始模式**：如果我们简单使用Dashboard，不做任何修改，就是采用的这种模式：

  ​    客户端在启动时，会同时启动一个ServerSocket，默认端口为：8719（如果被占用，尝试3次后+1，继续尝试），告诉Dashboard，当Dashboard中的配置有变化时，通过API接口告诉我们的服务中的Sentinel客户端，Sentinel将这些配置存到内存中，服务重启即丢失，**该模式只能用于简单测试，一定不能用于生产环境！**

  ![](https://tianxiawuhao.github.io/post-images/1658666719119.png)

  可从Env类一路向下研究，就可以知道逻辑：

 ```java
  public class Env {
      public static final Sph sph = new CtSph();
      static {
          // If init fails, the process will exit.
          InitExecutor.doInit();
      }
  }
  ```

- **Pull模式**：Sentinel客户端（我们的应用服务）向远端的配置管理中心主动定期轮询拉取规则，更新到内存缓存，同时写入到本地磁盘文件，这样的话也可以实现持久化，但是数据一致性、实时性不太好，而且大量的轮询对服务性能又有影响！

  ![](https://tianxiawuhao.github.io/post-images/1658666739217.png)

 ```java
  // 需要在客户端注册数据源：
  // —将对应的读数据源注册至对应的 RuleManager，
  // ——将写数据源注册至 transport 的 WritableDataSourceRegistry 中。
  public class FileDataSourceInit implements InitFunc {
   
      @Override
      public void init() throws Exception {
          String flowRulePath = "xxx";
   
          ReadableDataSource<String, List<FlowRule>> ds = new FileRefreshableDataSource<>(
                  flowRulePath, source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {})
          );
          // 将可读数据源注册至 FlowRuleManager.
          FlowRuleManager.register2Property(ds.getProperty());
   
          WritableDataSource<List<FlowRule>> wds = new FileWritableDataSource<>(flowRulePath, this::encodeJson);
          // 将可写数据源注册至 transport 模块的 WritableDataSourceRegistry 中.
          // 这样收到控制台推送的规则时，Sentinel 会先更新到内存，然后将规则写入到文件中.
          WritableDataSourceRegistry.registerFlowDataSource(wds);
      }
   
      private <T> String encodeJson(T t) {
          return JSON.toJSONString(t);
      }
  }
  ```

- **Push模式：生产环境最推荐的一种模式，需要依赖于外部的注册中心，nacos、zookeeper等**

- - 我们在Sentinel Dashboard中修改的规则配置，首先会先持久化到我们的配置中心中（配置中心已经实现了持久化）；

  - sentinel客户端（我们的应用程序）是从配置中心获取数据。

    ![](https://tianxiawuhao.github.io/post-images/1658666808514.png)

- 具体的Push模式（nacos）的实现已经在“第3段”有讲述；

- 但是暂时默认开源的Sentinel Dashboard中并没有直接提供对nacos等注册中心的直接支持，得自己改造，或者直接去Nacos中盲配！

------

###  三 Sentinel Dashboard中得各种规则配置与解读 

首先，当我们第一次打开Sentinel Dashboard时，我们会发现控制台空空如也，这是由于Sentinel使用的是懒加载，所以，我们需要先调用一次服务的接口，然后才可以看到“簇点链路”：

![](https://tianxiawuhao.github.io/post-images/1658666865289.png)

之后我们就可以开始愉快的配置啦！

本章节，不做具体配置和调试，主要是对所有的配置进行解释，加深印象！

####  1 流控规则：见名知意，流量控制 

![](https://tianxiawuhao.github.io/post-images/1658666880890.png)

- **2种阈值类型**：

- - **QPS**：对单位时间内的请求数量进行统计，控制流量；
  - **线程数**：属于服务隔离（线程隔离），Sentinel默认使用的是“信号量隔离”，而Hystrix默认采用的是“线程池隔离”
  - - - 信号量隔离：通过“信号量计数器”实现，开销小，但是隔离性一般；适合“扇出”大的场景，如gateway就是扇出比较大的场景；
      - 线程池隔离：基于线程池实现，额外开销大，但是隔离性更强；适合于“扇出”小的场景；

- **3种流控模式：**

- - **直接**：流量统计对象 和 限流对象 都是当前资源本身；
  - **关联**：流量统计的对象是其它资源，限流的对象却是自己；
  - **链路**：只统计从指定链路访问本资源的请求，触发阈值时，也只对指定的链路进行限流；

> 注意，默认情况下，所有的请求的入口链路都是默认的： sentinel_default_context ，链路模式是不好使用的，所以，如果要使用链路模式，需要关闭“context统一入口配置”
>
> ```yaml
> spring:
>   application:
>     name: orderservice
>   cloud:
>     sentinel:
>       transport:
>         dashboard: localhost:8080 # sentinel控制台地址
>       web-context-unify: false # 关闭context统一入口
> ```

- **3种流控效果**：

- - **快速失败**：达到限流阈值后，直接抛出FlowException异常；
  - **WarmUp预热**：与“快速失败”类似，达到阈值后，也是直接抛出FlowException异常，但是该模式下的阈值是变化的，默认从一个最大值的1/3，逐渐增加到最大值；常用于服务的预热启动阶段，防止服务流量一下子打到最大，导致一些非常态问题发生；
  - **排队等待（流量整形）**：请求到达后，放入一个队列中，按照 1/QPS 的速度进行消费处理，在队列中的请求等待时间，最大不可以超过“设定的超时时间”！

####  2 热点规则（热点限流规则）—— 特殊的流控规则 

![](https://tianxiawuhao.github.io/post-images/1658666900562.png)

由于Sentinel默认知乎将Springmvc的注解如@RequestMapping等注册为资源，所以当我们需要定义其它资源时，需要手动使用@SentinelResource注解去定义：

```java
@SentinelResource("hot")
public Order queryOrderById(Long orderId) {
    // 1.查询订单
    Order order = orderMapper.findById(orderId);
    // 2.用Feign远程调用
    User user = userClient.findById(order.getUserId());
    // 3.封装user到Order
    order.setUser(user);
    // 4.返回
    return order;
}
```

之后，当我们调用过一次后，就可以看到“hot”这个资源了，而后对它进行流控配置：

> 上图的热点流控规则解读为：对于hot这个资源，对于它的第0个请求参数，允许它1秒内相同值的最大请求数为5，同时，包含一个特殊情况，对于value=101，允许的QPS阈值为10。

####  3 降级规则（熔断降级） 

熔断降级通常配置 Feign 服务间调用使用，我们通常会定义两个类 UserClient代理接口 + UserClientFallbackFactory降级工厂类：

UserClient：

```java
@FeignClient(value = "userservice", fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {
 
    @GetMapping("/user/{id}")
    User findById(@PathVariable("id") Long id);
}
```

UserClientFallbackFactory：

```java
@Slf4j
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable throwable) {
        return new UserClient() {
            @Override
            public User findById(Long id) {
                log.info("请求用户数据失败");
                return new User().setId(100L).setUsername("default").setAddress("defaultAddress");
            }
        };
    }
}
```

![](https://tianxiawuhao.github.io/post-images/1658666913571.png)

上图的熔断降级规则解读为：

当ResponseTime时间**超过400ms时为慢调用，统计最近10000ms内的请求，如果请求总数超过10次，且慢调用的比例达到0.6，则触发熔断**OPEN；

熔断时常为5秒；

**5秒后，进入HALF-OPEN状态，放行一次请求做测试**，如果OK，则断路器重新CLOSE闭合，开始正常工作！
![](https://tianxiawuhao.github.io/post-images/1658666926145.png)

**4、授权规则（对请求的来源做控制）**

有时候，我们担心由于服务暴露，导致有些请求会越过Gateway，而直接访问我们的服务，这时候，我们就可以通过Sentinel授权只允许从Gateway过来的请求访问我们的服务。

![](https://tianxiawuhao.github.io/post-images/1658666936655.png)

在gateway中通过filter为经过的请求增加一个header值 origin = gateway：

```yaml
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848 # nacos地址
    gateway:
      routes:
        - id: user-service # 路由标示，必须唯一
          uri: lb://userservice # 路由的目标地址
          predicates: # 路由断言，判断请求是否符合规则
            - Path=/user/** # 路径断言，判断路径是否是以/user开头，如果是则符合
        - id: order-service
          uri: lb://orderservice
          predicates:
            - Path=/order/**
      default-filters:
        - AddRequestHeader=origin,gateway
```

在项目中注入一个RequestOriginParser：

```java
@Component
public class HeaderOriginParser implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        // 1.获取请求头
        String origin = request.getHeader("origin");
        // 2.非空判断
        if (StringUtils.isEmpty(origin)) {
            origin = "blank";
        }
        return origin;
    }
}
```

最后，配置一条授权规则白名单：

![](https://tianxiawuhao.github.io/post-images/1658666945938.png)

> 上图授权规则解读：只允许请求头中，origin = gateway 的请求通过！

------

###  四 补充其他知识点 

####  1 默认情况下，发生限流、授权拦截时，会直接抛出异常到调用方，很不友好，最好要自定义异常返回结果 

需要为 BlockExceptionHandler 写一个实现：

```java
@Component
public class SentinelExceptionHandler implements BlockExceptionHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
        String msg = "未知异常";
        int status = 429;
 
        if (e instanceof FlowException) {
            msg = "请求被限流了";
        } else if (e instanceof ParamFlowException) {
            msg = "请求被热点参数限流";
        } else if (e instanceof DegradeException) {
            msg = "请求被降级了";
        } else if (e instanceof AuthorityException) {
            msg = "没有权限访问";
            status = 401;
        }
 
        response.setContentType("application/json;charset=utf-8");
        response.setStatus(status);
        response.getWriter().println("{\"msg\": " + msg + ", \"status\": " + status + "}");
    }
}
```

####  2 不同的流控规则，使用的技术实现方案不一样 

对于限流的需求，常见的实现方案有多种，滑动时间窗口、令牌桶、漏桶，这三种有各自擅长的业务场景，而Sentinel支持的业务场景很多，在不同的场景下，就选择了不同的限流实现。

- 快速失败、WarmUp预热：滑动时间窗口
- 热点限流：令牌桶
- 排队等待：漏桶

具体的限流实现，请参阅另一篇文章：[Sentinel源码拓展之——限流的各种实现方式](http://tinaxiawuhao.github.io/post/xUda2KT1A/)