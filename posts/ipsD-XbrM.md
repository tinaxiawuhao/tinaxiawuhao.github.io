---
title: '网关断言'
date: 2022-07-11 21:21:48
tags: [springCloud]
published: true
hideInList: false
feature: /post-images/ipsD-XbrM.png
isTop: false
---
## 网关断言

断言的英文是Predicate，也可以翻译成谓词。主要的作用就是进行条件判断，可以在网关中实现多种条件判断，只有所有的判断结果都通过时，也就是所有的条件判断都返回true，才会真正的执行路由功能。

### SpringCloud Gateway内置断言

SpringCloud Gateway包括许多内置的断言工厂，所有这些断言都与HTTP请求的不同属性匹配。

#### 基于日期时间类型的断言

基于日期时间类型的断言根据时间做判断，主要有三个：

- AfterRoutePredicateFactory：接收一个日期时间参数，判断当前请求的日期时间是否晚于指定的日期时间。
- BeforeRoutePredicateFactory：接收一个日期时间参数，判断当前请求的日期时间是否早于指定的日期时间。
- BetweenRoutePredicateFactory：接收两个日期时间参数，判断当前请求的日期时间是否在指定的时间时间段内。

#### 使用示例

```java
- After=2022-05-10T23:59:59.256+08:00[Asia/Shanghai]
```

#### 基于远程地址的断言

RemoteAddrRoutePredicateFactory：接收一个IP地址段，判断发出请求的客户端的IP地址是否在指定的IP地址段内。

#### 使用示例

```java
- RemoteAddr=192.168.0.1/24
```

#### 基于Cookie的断言

CookieRoutePredicateFactory：接收两个参数， Cookie的名称和一个正则表达式。判断请求的Cookie是否具有给定名称且值与正则表达式匹配。

#### 使用示例

```java
- Cookie=name, binghe.
```

#### 基于Header的断言

HeaderRoutePredicateFactory：接收两个参数，请求Header的名称和正则表达式。判断请求Header中是否具有给定的名称且值与正则表达式匹配。

#### 使用示例

```java
- Header=X-Request-Id, \d+
```

#### 基于Host的断言

HostRoutePredicateFactory：接收一个参数，这个参数通常是主机名或者域名的模式，例如`**.binghe.com`这种格式。判断发出请求的主机是否满足匹配规则。

#### 使用示例

```java
- Host=**.binghe.com
```

#### 基于Method请求方法的断言

MethodRoutePredicateFactory：接收一个参数，判断请求的类型是否跟指定的类型匹配，通常指的是请求方式。例如，POST、GET、PUT等请求方式。

#### 使用示例

```java
- Method=GET
```

#### 基于Path请求路径的断言

PathRoutePredicateFactory：接收一个参数，判断请求的链接地址是否满足路径规则，通常指的是请求的URI部分。

#### 使用示例

```java
- Path=/binghe/{segment}
```

#### 基于Query请求参数的断言

QueryRoutePredicateFactory ：接收两个参数，请求参数和正则表达式， 判断请求的参数是否具有给定的名称并且参数值是否与正则表达式匹配。

#### 使用示例

```java
- Query=name, binghe.
```

#### 基于路由权重的断言

WeightRoutePredicateFactory：接收一个[组名,权重]格式的数组，然后对于同一个组内的路由按照权重转发。

#### 使用示例

```java
- id: weight1
  uri: http://localhost:8080
  predicates:
    - Path=/api/**
    - Weight=group1,2
  filters:
    - StripPrefix=1
- id: weight2
  uri: http://localhost:8081
  predicates:
    - Path=/api/**
    - Weight=group1,8
  filters:
    - StripPrefix=1
```

### 演示内置断言

在演示的示例中，我们基于Path请求路径的断言判断请求路径是否符合规则，基于远程地址的断言判断请求主机地址是否在地址段中，并且限制请求的方式为GET方式。整个演示的过程以访问用户微服务的接口为例。

（1）由于在开发项目时，所有的服务都是在我本地启动的，首先查看下我本机的IP地址，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658668997017.png)



可以看到，我本机的IP地址为192.168.0.27，属于192.168.0.1/24网段。

（2）在服务网关模块shop-gateway中，将application.yml文件备份成application-sentinel.yml文件，并将application.yml文件中的内容修改成application-simple.yml文件中的内容。接下来，在application.yml文件中的`spring.cloud.gateway.routes`节点下的`- id: user-gateway`下面进行断言配置，配置后的结果如下所示。

```java
spring:
  cloud:
    gateway:
      routes:
        - id: user-gateway
          uri: http://localhost:8060
          order: 1
          predicates:
            - Path=/server-user/**
            - RemoteAddr=192.168.0.1/24
            - Method=GET
          filters:
            - StripPrefix=1
```

**注意：完整的配置参见案例完整源代码。**

（3）配置完成后启动用户微服务和网关服务，通过网关服务访问用户微服务，在浏览器中输入`http://localhost:10001/server-user/user/get/1001`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669013855.png)



可以看到通过`http://localhost:10001/server-user/user/get/1001`链接不能正确访问到用户信息。

接下来，在浏览器中输入`http://192.168.0.27:10001/server-user/user/get/1001`，能够正确获取到用户的信息。

![](https://tianxiawuhao.github.io/post-images/1658669028281.png)



（4）停止网关微服务，将基于远程地址的断言配置成`- RemoteAddr=192.168.1.1/24`，也就是将基于远程地址的断言配置成与我本机IP地址不在同一个网段，这样就能演示请求主机地址不在地址段中的情况，修改后的基于远程地址的断言配置如下所示。

```java
- RemoteAddr=192.168.1.1/24
```

（5）重启网关服务，再次在浏览器中输入`http://localhost:10001/server-user/user/get/1001`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669041992.png)



可以看到通过`http://localhost:10001/server-user/user/get/1001`链接不能正确访问到用户信息。

接下来，在浏览器中输入`http://192.168.0.27:10001/server-user/user/get/1001`，也不能正确获取到用户的信息了。

![](https://tianxiawuhao.github.io/post-images/1658669056498.png)



### 自定义断言

SpringCloud Gateway支持自定义断言功能，我们可以在具体业务中，基于SpringCloud Gateway自定义特定的断言功能。

#### 自定义断言概述

SpringCloud Gateway虽然提供了多种内置的断言功能，但是在某些场景下无法满足业务的需要，此时，我们就可以基于SpringCloud Gateway自定义断言功能，以此来满足我们的业务场景。

#### 实现自定义断言

这里，我们基于SpringCloud Gateway实现断言功能，实现后的效果是在服务网关的application.yml文件中的`spring.cloud.gateway.routes`节点下的`- id: user-gateway`下面进行如下配置。

```java
spring:
  cloud:
    gateway:
      routes:
        - id: user-gateway
          uri: http://localhost:8060
          order: 1
          predicates:
            - Path=/server-user/**
            - Name=binghe
          filters:
            - StripPrefix=1
```

通过服务网关访问用户微服务时，只有在访问的链接后面添加`?name=binghe`参数时才能正确访问用户微服务。

（1）在网关服务shop-gateway中新建`io.binghe.shop.predicate`包，在包下新建NameRoutePredicateConfig类，主要定义一个Spring类型的name成员变量，用来接收配置文件中的参数，源码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 接收配置文件中的参数
 */
@Data
public class NameRoutePredicateConfig implements Serializable {
    private static final long serialVersionUID = -3289515863427972825L;
    private String name;
}
```

（2）实现自定义断言时，需要新建类继承`org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory`类，在`io.binghe.shop.predicate`包下新建NameRoutePredicateFactory类，继承`org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory`类，并覆写相关的方法，源码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 自定义断言功能
 */
@Component
public class NameRoutePredicateFactory extends AbstractRoutePredicateFactory<NameRoutePredicateConfig> {

    public NameRoutePredicateFactory() {
        super(NameRoutePredicateConfig.class);
    }

    @Override
    public Predicate<ServerWebExchange> apply(NameRoutePredicateConfig config) {
        return (serverWebExchange)->{
            String name = serverWebExchange.getRequest().getQueryParams().getFirst("name");
            if (StringUtils.isEmpty(name)){
                name = "";
            }
            return name.equals(config.getName());
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("name");
    }
}
```

（3）在服务网关的application.yml文件中的`spring.cloud.gateway.routes`节点下的`- id: user-gateway`下面进行如下配置。

```java
spring:
  cloud:
    gateway:
      routes:
        - id: user-gateway
          uri: http://localhost:8060
          order: 1
          predicates:
            - Path=/server-user/**
            - Name=binghe
          filters:
            - StripPrefix=1
```

（4）分别启动用户微服务与网关服务，在浏览器中输入`http://localhost:10001/server-user/user/get/1001`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669075580.png)



可以看到，在浏览器中输入`http://localhost:10001/server-user/user/get/1001`，无法获取到用户信息。

（5）在浏览器中输入`http://localhost:10001/server-user/user/get/1001?name=binghe`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669099356.png)



可以看到，在访问链接后添加`?name=binghe`参数后，能够正确获取到用户信息。

至此，我们实现了自定义断言功能。

## 网关过滤器

过滤器可以在请求过程中，修改请求的参数和响应的结果等信息。在生命周期的角度总体上可以分为前置过滤器（Pre）和后置过滤器(Post)。在实现的过滤范围角度可以分为局部过滤器（GatewayFilter）和全局过滤器（GlobalFilter）。局部过滤器作用的范围是某一个路由，全局过滤器作用的范围是全部路由。

- Pre前置过滤器：在请求被网关路由之前调用，可以利用这种过滤器实现认证、鉴权、路由等功能，也可以记录访问时间等信息。
- Post后置过滤器：在请求被网关路由到微服务之后执行。可以利用这种过滤器修改HTTP的响应Header信息，修改返回的结果数据（例如对于一些敏感的数据，可以在此过滤器中统一处理后返回），收集一些统计信息等。
- 局部过滤器（GatewayFilter）：也可以称为网关过滤器，这种过滤器主要是作用于单一路由或者某个路由分组。
- 全局过滤器（GlobalFilter）：这种过滤器主要作用于所有的路由。

### 局部过滤器

局部过滤器又称为网关过滤器，这种过滤器主要是作用于单一路由或者某个路由分组。

#### 局部过滤器概述

在SpringCloud Gateway中内置了很多不同类型的局部过滤器，主要如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669111264.png)



#### 演示内部过滤器

演示内部过滤器时，我们为原始请求添加一个名称为IP的Header，值为localhost，并添加一个名称为name的参数，参数值为binghe。同时修改响应的结果状态，将结果状态修改为1001。

（1）在服务网关的application.yml文件中的`spring.cloud.gateway.routes`节点下的`- id: user-gateway`下面进行如下配置。

```java
spring:
  cloud:
    gateway:
      routes:
        - id: user-gateway
          uri: http://localhost:8060
          order: 1
          predicates:
            - Path=/server-user/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=IP,localhost
            - AddRequestParameter=name,binghe
            - SetStatus=1001
```

（2）在用户微服务的`io.binghe.shop.user.controller.UserController`类中新增apiFilter1()方法，如下所示。

```java
@GetMapping(value = "/api/filter1")
public String apiFilter1(HttpServletRequest request, HttpServletResponse response){
    log.info("访问了apiFilter1接口");
    String ip = request.getHeader("IP");
    String name = request.getParameter("name");
    log.info("ip = " + ip + ", name = " + name);
    return "apiFilter1";
}
```

可以看到，在新增加的apiFilter1()方法中，获取到新增加的Header与参数，并将获取出来的参数与Header打印出来。并且方法返回的是字符串apiFilter1。

（3）分别启动用户微服务与网关服务，在浏览器中输入`http://localhost:10001/server-user/user/api/filter1`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669123448.png)



此时，查看浏览器中的响应状态码，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669132840.png)



可以看到，此时的状态码已经被修改为1001。

接下来，查看下用户微服务的控制台输出的信息，发现在输出的信息中存在如下数据。

```java
访问了apiFilter1接口
ip = localhost, name = binghe
```

说明使用SpringCloud Gateway的内置过滤器成功为原始请求添加了一个名称为IP的Header，值为localhost，并添加了一个名称为name的参数，参数值为binghe。同时修改了响应的结果状态，将结果状态修改为1001，符合预期效果。

#### 自定义局部过滤器

这里，我们基于SpringCloud Gateway自定义局部过滤器实现是否开启灰度发布的功能，整个实现过程如下所示。

（1）在服务网关的application.yml文件中的`spring.cloud.gateway.routes`节点下的`- id: user-gateway`下面进行如下配置。

```java
spring:
  cloud:
    gateway:
      routes:
        - id: user-gateway
          uri: http://localhost:8060
          order: 1
          predicates:
            - Path=/server-user/**
          filters:
            - StripPrefix=1
            - Grayscale=true
```

（2）在网关服务模块shop-gateway中新建`io.binghe.shop.filter`包，在包下新建GrayscaleGatewayFilterConfig类，用于接收配置中的参数，如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 接收配置参数
 */
@Data
public class GrayscaleGatewayFilterConfig implements Serializable {
    private static final long serialVersionUID = 983019309000445082L;
    private boolean grayscale;
}
```

（3）在`io.binghe.shop.filter`包下GrayscaleGatewayFilterFactory类，继承`org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory`类，主要是实现自定义过滤器，模拟实现灰度发布。代码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 自定义过滤器模拟实现灰度发布
 */
@Component
public class GrayscaleGatewayFilterFactory extends AbstractGatewayFilterFactory<GrayscaleGatewayFilterConfig> {

    public GrayscaleGatewayFilterFactory(){
        super(GrayscaleGatewayFilterConfig.class);
    }
    @Override
    public GatewayFilter apply(GrayscaleGatewayFilterConfig config) {
        return (exchange, chain) -> {
            if (config.isGrayscale()){
                System.out.println("开启了灰度发布功能...");
            }else{
                System.out.println("关闭了灰度发布功能...");
            }
            return chain.filter(exchange);
        };
    }

    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("grayscale");
    }
}
```

（4）分别启动用户微服务和服务网关，在浏览器中输入`http://localhost:10001/server-user/user/get/1001`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669147257.png)



可以看到，通过服务网关正确访问到了用户微服务，并正确获取到了用户信息。

接下来，查看下服务网关的终端，发现已经成功输出了如下信息。

```java
开启了灰度发布功能...
```

说明正确实现了自定义的局部过滤器。

### 全局过滤器

全局过滤器是一系列特殊的过滤器，会根据条件应用到所有路由中。

#### 全局过滤器概述

在SpringCloud Gateway中内置了多种不同的全局过滤器，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669168091.png)



#### 演示全局过滤器

（1）在服务网关模块shop-gateway模块下的`io.binghe.shop.config`包下新建GatewayFilterConfig类，并在类中配置几个全局过滤器，如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 网关过滤器配置
 */
@Configuration
@Slf4j
public class GatewayFilterConfig {
    @Bean
    @Order(-1)
    public GlobalFilter globalFilter() {
        return (exchange, chain) -> {
            log.info("执行前置过滤器逻辑");
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.info("执行后置过滤器逻辑");
            }));
        };
    }
}
```

**注意：@Order注解中的数字越小，执行的优先级越高。**

（2）启动用户微服务与服务网关，在浏览器中访问`http://localhost:10001/server-user/user/get/1001`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669177407.png)



在服务网关终端输出如下信息。

```java
执行前置过滤器逻辑
执行后置过滤器逻辑
```

说明我们演示的全局过滤器生效了。

#### 自定义全局过滤器

SpringCloud Gateway内置了很多全局过滤器，一般情况下能够满足实际开发需要，但是对于某些特殊的业务场景，还是需要我们自己实现自定义全局过滤器。

这里，我们就模拟实现一个获取客户端访问信息，并统计访问接口时长的全局过滤器。

（1）在网关服务模块shop-order的`io.binghe.shop.filter`包下，新建GlobalGatewayLogFilter类，实现`org.springframework.cloud.gateway.filter.GlobalFilter`接口和`org.springframework.core.Ordered`接口，代码如下所示。

```java
/**
 * @author binghe
 * @version 1.0.0
 * @description 自定义全局过滤器，模拟实现获取客户端信息并统计接口访问时长
 */
@Slf4j
@Component
public class GlobalGatewayLogFilter implements GlobalFilter, Ordered {
    /**
     * 开始访问时间
     */
    private static final String BEGIN_VISIT_TIME = "begin_visit_time";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        //先记录下访问接口的开始时间
        exchange.getAttributes().put(BEGIN_VISIT_TIME, System.currentTimeMillis());
        return chain.filter(exchange).then(Mono.fromRunnable(()->{
            Long beginVisitTime = exchange.getAttribute(BEGIN_VISIT_TIME);
            if (beginVisitTime != null){
                log.info("访问接口主机: " + exchange.getRequest().getURI().getHost());
                log.info("访问接口端口: " + exchange.getRequest().getURI().getPort());
                log.info("访问接口URL: " + exchange.getRequest().getURI().getPath());
                log.info("访问接口URL参数: " + exchange.getRequest().getURI().getRawQuery());
                log.info("访问接口时长: " + (System.currentTimeMillis() - beginVisitTime) + "ms");
            }
        }));
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

上述代码的实现逻辑还是比较简单的，这里就不再赘述了。

（2）启动用户微服务与网关服务，在浏览器中输入`http://localhost:10001/server-user/user/api/filter1?name=binghe`，如下所示。

![](https://tianxiawuhao.github.io/post-images/1658669189664.png)



接下来，查看服务网关的终端日志，可以发现已经输出了如下信息。

```java
访问接口主机: localhost
访问接口端口: 10001
访问接口URL: /server-user/user/api/filter1
访问接口URL参数: name=binghe
访问接口时长: 126ms
```

说明我们自定义的全局过滤器生效了。