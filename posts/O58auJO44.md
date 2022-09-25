---
title: 'springBoot概述'
date: 2021-05-31 16:29:40
tags: [springBoot]
published: true
hideInList: false
feature: /post-images/O58auJO44.png
isTop: false
---
## Spring Boot 优点

1. 容易上手，提升开发效率，为 Spring 开发提供一个更快、更广泛的入门体验。
2. 开箱即用，远离繁琐的配置。
3. 提供了一系列大型项目通用的非业务性功能，例如：内嵌服务器、安全管理、运行数据监控、运行状况检查和外部化配置等。
4. 没有代码生成，也不需要XML配置。
5. 避免大量的 Maven 导入和各种版本冲突。

## Spring Boot 的核心注解是哪个？它主要由哪几个注解组成的？

启动类上面的注解是@SpringBootApplication，它也是 Spring Boot 的核心注解，主要组合包含了以下 3 个注解：

`@SpringBootConfiguration`：组合了 @Configuration 注解，实现配置文件的功能。

`@EnableAutoConfiguration`：打开自动配置的功能，也可以关闭某个自动配置的选项，如关闭数据源自动配置功能： `@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })`。

`@ComponentScan`：Spring组件扫描。

所有其它 Spring 组件(如Controller层、Service层、Dao层、定时器等)都必须放在应用  @SpringBootApplication 注解所在类的同包或者其子包下面，因为应用从启动类开始启动，然后会扫描启动类同包及其子包下面的组件，如果放在其它地方则会因为扫描不到而加载不了

## JavaConfig

`Spring JavaConfig` 是 Spring 社区的产品，它提供了配置 Spring IoC 容器的纯Java 方法。因此它有助于避免使用 XML 配置。使用 JavaConfig 的优点在于：

（1）面向对象的配置。由于配置被定义为 JavaConfig 中的类，因此用户可以充分利用 Java 中的面向对象功能。一个配置类可以继承另一个，重写它的@Bean 方法等。

（2）减少或消除 XML 配置。基于依赖注入原则的外化配置的好处已被证明。但是，许多开发人员不希望在 XML 和 Java 之间来回切换。JavaConfig 为开发人员提供了一种纯 Java 方法来配置与 XML 配置概念相似的 Spring 容器。从技术角度来讲，只使用 JavaConfig 配置类来配置容器是可行的，但实际上很多人认为将JavaConfig 与 XML 混合匹配是理想的。

（3）类型安全和重构友好。JavaConfig 提供了一种类型安全的方法来配置 Spring容器。由于 Java 5.0 对泛型的支持，现在可以按类型而不是按名称检索 bean，不需要任何强制转换或基于字符串的查找。

```java
/**
 * @ConfigurationProperties 表示 告诉 SpringBoot 将本类中的所有属性和配置文件中相关的配置进行绑定；
 * prefix = "user" 表示 将配置文件中 key 为 user 的下面所有的属性与本类属性进行一一映射注入值，如果配置文件中
 * 不存在 "user" 的 key，则不会为 POJO 注入值，属性值仍然为默认值
 * @Component 将本来标识为一个 Spring 组件，因为只有是容器中的组件，容器才会为 @ConfigurationProperties 提供此注入功能
 * @PropertySource (value = { " classpath : user.properties " }) 指明加载类路径下的哪个配置文件来注入值
 */
@PropertySource(value = {"classpath:user.properties"})
@Component
@ConfigurationProperties(prefix = "user")
public class User {
    private Integer id;
    private String lastName;
    private Integer age;
    private Date birthday;
    private List<String> colorList;
    private Map<String, String> cityMap;
}
/**
 * 文中的@Configuration 可以替换为@Component运行结果是一样的，但是两者是有不同的，@Configuration会为配置类生成CGLIB代理Class，@Component不会
 */
@PropertySource(value = {"classpath:user.properties"})
@Configuration
public class User {
    @Value(${user.id})
    private Integer id;
     @Value(${user.lastName})
    private String lastName;
     @Value(${user.age})
    private Integer age;
     @Value(${user.birthday})
    private Date birthday;
     @Value(${user.colorList})
    private List<String> colorList;
     @Value(${user.maps})
    private Map<String, String> cityMap;
}
```

**user.properties**

```java
user.id=111
user.lastName=张无忌
user.age=120
user.birthday=2018/07/11
user.colorList=red,yellow,green,blacnk
user.cityMap.mapK1=长沙市
user.cityMap.mapK2=深圳市
user.maps="{mapK1: '长沙市', mapK2: '深圳市'}"
```

## Spring Boot 自动配置

注解 @EnableAutoConfiguration, @Configuration, @ConditionalOnClass 就是自动配置的核心，

@EnableAutoConfiguration 给容器导入META-INF/spring.factories 里定义的自动配置类。

筛选有效的自动配置类。

每一个自动配置类结合对应的 xxxProperties.java 读取配置文件进行自动配置功能

## Spring Boot 配置加载顺序

Spring Boot 支持多种外部配置方式，如下所示，从上往下加载优先级由高到低，内容相同时覆盖，不相同时累加。

![](https://tinaxiawuhao.github.io/post-images/1622190707004.png)

如果在不同的目录中存在多个配置文件，它的读取顺序是：
 1、config/application.properties（项目根目录中config目录下）
 2、config/application.yml
 3、application.properties（项目根目录下）
 4、application.yml
 5、resources/config/application.properties（项目resources目录中config目录下）
 6、resources/config/application.yml
 7、resources/application.properties（项目的resources目录下）
 8、resources/application.yml

## YAML 配置

YAML 现在可以算是非常流行的一种配置文件格式了，无论是前端还是后端，都可以见到 YAML 配置。那么 YAML 配置和传统的 properties 配置相比到底有哪些优势呢？

1. 配置有序，在一些特殊的场景下，配置有序很关键
2. 支持数组，数组中的元素可以是基本数据类型也可以是对象
3. 简洁

相比 properties 配置文件，YAML 还有一个缺点，就是不支持 `@PropertySource` 注解导入自定义的 YAML 配置。

## Spring Boot 是否可以使用 XML 配置 

Spring Boot 推荐使用 Java 配置而非 XML 配置，但是 Spring Boot 中也可以使用 XML 配置，通过 `@ImportResource` 注解可以引入一个 XML 配置。

## spring boot 核心配置文件bootstrap.properties和 application.properties 有何区别

单纯做 Spring Boot 开发，可能不太容易遇到 `bootstrap.properties` 配置文件，但是在结合 Spring Cloud 时，这个配置就会经常遇到了，特别是在需要加载一些远程配置文件的时侯。

spring boot 核心的两个配置文件：

- bootstrap (. yml 或者 . properties)：boostrap 由父 ApplicationContext 加载的，比 applicaton 优先加载，配置在应用程序上下文的引导阶段生效。一般来说我们在 Spring Cloud Config 或者 Nacos 中会用到它。且 boostrap 里面的属性不能被覆盖；
- application (. yml 或者 . properties)： 由ApplicatonContext 加载，用于 spring boot 项目的自动化配置。

## Spring Profiles

![](https://tinaxiawuhao.github.io/post-images/1622190724869.png)

```yaml
spring:
 profiles:
  active: devel #指定激活哪个环境配置，激活后，第一个文档内容失效;不指定时，以第一个文档为准
server:
 port: 8083
--- #"---"用于分隔不同的profiles（）文档块
spring:
 profiles: devel #指定环境标识为"devel",相当于"application-{profile}.properties/yml"中的profile
server:
 port: 8081
---
spring:
 profiles: deploy #指定环境标识为"deploy",相当于"application-{profile}.properties/yml"中的profile
server:
 port: 8082
```

## 比较一下 Spring Security 和 Shiro 各自的优缺点 

由于 Spring Boot 官方提供了大量的非常方便的开箱即用的 Starter ，包括 Spring Security 的 Starter ，使得在 Spring Boot 中使用 Spring Security 变得更加容易，甚至只需要添加一个依赖就可以保护所有的接口，所以，如果是 Spring Boot 项目，一般选择 Spring Security 。当然这只是一个建议的组合，单纯从技术上来说，无论怎么组合，都是没有问题的。Shiro 和 Spring Security 相比，主要有如下一些特点：

1. Spring Security 是一个重量级的安全管理框架；Shiro 则是一个轻量级的安全管理框架
2. Spring Security 概念复杂，配置繁琐；Shiro 概念简单、配置简单
3. Spring Security 功能强大；Shiro 功能简单

## Spring Boot 中如何解决跨域问题 

跨域可以在前端通过 JSONP 来解决，但是 JSONP 只可以发送 GET 请求，无法发送其他类型的请求，在 RESTful 风格的应用中，就显得非常鸡肋，因此我们推荐在后端通过 CORS，(Cross-origin resource sharing） 来解决跨域问题。这种解决方案并非 Spring Boot 特有的，在传统的 SSM 框架中，就可以通过 CORS 来解决跨域问题，只不过之前我们是在 XML 文件中配置 CORS ，现在可以通过实现WebMvcConfigurer接口然后重写addCorsMappings方法解决跨域问题。

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowCredentials(true)
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .maxAge(3600);
    }

}
```

项目中前后端分离部署，所以需要解决跨域的问题。

我们使用cookie存放用户登录的信息，在spring拦截器进行权限控制，当权限不符合时，直接返回给用户固定的json结果。

当用户登录以后，正常使用；当用户退出登录状态时或者token过期时，由于拦截器和跨域的顺序有问题，出现了跨域的现象。

我们知道一个http请求，先走filter，到达servlet后才进行拦截器的处理，如果我们把cors放在filter里，就可以优先于权限拦截器执行。

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        corsConfiguration.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource urlBasedCorsConfigurationSource = new UrlBasedCorsConfigurationSource();
        urlBasedCorsConfigurationSource.registerCorsConfiguration("/**", corsConfiguration);
        return new CorsFilter(urlBasedCorsConfigurationSource);
    }
}
```



## Spring Boot 中的监视器

Spring boot actuator 是 spring 启动框架中的重要功能之一。Spring boot 监视器可帮助您访问生产环境中正在运行的应用程序的当前状态。有几个指标必须在生产环境中进行检查和监控。即使一些外部应用程序可能正在使用这些服务来向相关人员触发警报消息。监视器模块公开了一组可直接作为 HTTP URL 访问的REST 端点来检查状态。

## 注册 Servlet 三大组件 Servlet、Filter、Listener

### 继承，接口实现方式

#### ServletRegistrationBean 注册 Servlet

1、自定义类继承 javax.servlet.http.HttpServlet，然后重写其 doGet 与 doPost 方法，在方法中编写控制代码；

2、第二步将 ServletRegistrationBean 组件添加到 Spring 容器中

```java
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
/**
 * Created by Administrator 
 * 标准的 Servlet 实现 HttpServlet；重写其 doGet 、doPost 方法
 */
public class BookServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doPost(req, resp);
    }
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println(":com.lct.servlet.BookServlet:" + req.getRequestURL());
        /**讲求转发到后台的 user/users 请求去，即会进入*/
        req.getRequestDispatcher("user/users").forward(req, resp);
    }
}
```

3、上面 Serlvet 转发到下面的 UserControllr 控制器中

4、@Configuration 配置类相当于以前的 beans.xml 中的配置，将 ServletRegistrationBean 也添加到 Spring 容器中来

```java
import com.lct.component.MyLocaleResolve;
import com.lct.servlet.BookServlet;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
/**
 * Created by Administrator 
 * 自定义配置类
 */
@Configuration
public class MyMvcConfig {
    /**
     * 注册 Servlet 三大组件 之  Servlet
     * 添加 ServletRegistrationBean ，就相当于以前在 web.xml 中配置的 <servlet></servlet>标签
     */
    @Bean
    public ServletRegistrationBean myServlet() {
        /**第二个参数是个不定参数组，可以配置映射多个请求
         * 相当于以前在 web.xml中配置的 <servlet-mapptin></servlet-mapptin>*/
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(new
                BookServlet(), "/bookServlet");
        return registrationBean;
    }
}
```

5、运行测试：

![](https://tinaxiawuhao.github.io/post-images/1622190757311.gif)

#### FilterRegistrationBean 注册 Filter

1、Filter(过滤器) 是 Servlet 技术中最实用的技术之一

2、Web 开发人员通过 Filter 技术，对 web 服务器管理的所有 web 资源(如动态的 Jsp、 Servlet，以及静态的 image、 html、CSS、JS 文件等) 进行过滤拦截，从而实现一些特殊的功能(如实现 URL 级别的权限访问控制、过滤敏感词汇、压缩响应信息等) 

3、Filter 主要用于对用户请求进行预处理，也可以对 HttpServletResponse 进行后期处理(如编码设置，返回时禁用浏览器缓存等)

4、Filter 使用完整流程：Filter 对用户请求进行预处理，接着将请求交给 Servlet 进行处理并生成响应，最后 Filter 再对服务器响应进行后处理。

5、Servlet 的 Filter 经常会拿来与 Spring MVC 的 Interceptor(拦截器) 做对比

​	1）拦截器是基于 Java 的反射机制的，而过滤器是基于函数回调

​	2）拦截器不依赖与 servle t容器，过滤器依赖与 servlet 容器

​	3）拦截器可以访问 action 上下文、值栈里的对象，而过滤器不能访问

​	4）在 action 的生命周期中，拦截器可以多次被调用，而过滤器只能在容器初始化时被调用一次

​	5）拦截器可以获取 IOC 容器中的各个 bean，而过滤器就不行，这点很重要，在拦截器里注入一个 service，可以调用业务逻辑

​	6）SpringMVC 有自己的拦截器

```java
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
/**
 * Created by Administrator 
 * 标准 Servlet 过滤器，实现 javax.servlet.Filter 接口
 * 并重写它的 三个方法
 */
public class SystemFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("javax.servlet.Filter：：服务器启动....");
    }
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        /**
         * 转为 HttpServletRequest 输出请求路径 容易查看 请求地址
         */
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        System.out.println("javax.servlet.Filter：：过滤器放行前...." + request.getRequestURL());
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("javax.servlet.Filter：：过滤器返回后...." + request.getRequestURL());
    }
    @Override
    public void destroy() {
        System.out.println("javax.servlet.Filter：：服务器关闭....");
    }
}
```

6、使用 FilterRegistrationBean 添加 FIlter ：

```java
import com.lct.component.MyLocaleResolve;
import com.lct.filter.SystemFilter;
import com.lct.servlet.BookServlet;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import javax.servlet.DispatcherType;
import java.util.Arrays;
/**
 * Created by Administrator 
 * 自定义配置类
 */
@Configuration
public class MyMvcConfig {
    /**
     * 注册 Servlet 三大组件 之  Filter (过滤器)
     * 添加 FilterRegistrationBean ，就相当于以前在 web.xml 中配置的 <filter></filter> 标签
     */
    @Bean
    public FilterRegistrationBean myFilter() {
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        /**同样添加自定义的 Filter*/
        registrationBean.setFilter(new SystemFilter());
        /**然后设置过滤的路径，参数是个集合 ,相当于 web.xml中配置的 <filter-mapptin></filter-mapptin>
         * "/*": 表示过滤所有 get 与 post 请求*/
        registrationBean.setUrlPatterns(Arrays.asList("/*"));
        /**
         * setDispatcherTypes 相当于 web.xml 配置中 <filter-mapptin> 下的 <dispatcher> 标签
         * 用于过滤非常规的 get 、post 请求
         * REQUEST：默认方式，写了之后会过滤所有静态资源的请求
         * FORWARD：过滤所有的转发请求，无论是 jsp 中的 <jsp:forward</>、<%@ page errorPage= %>、还是后台的转发
         * INCLUDE：过滤 jsp 中的动态包含<jsp:include 请求
         * ERROR：过滤在 web.xml 配置的全局错误页面
         * 了解即可，实际中也很少这么做
         */
        registrationBean.setDispatcherTypes(DispatcherType.REQUEST);
        return registrationBean;
    }
}
```

#### ServletListenerRegistrationBean 注册 Listener

1、自定义监听器：

```java
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
/**
 * Created by Administrator on 2018/8/11 0011.
 * 标准 Servlet 监听器，实现 javax.servlet.ServletContextListener 接口
 * 然后实现方法
 * ServletContextListener：属于 Servlet 应用启动关闭监听器，监听容器初始化与销毁
 */
public class SystemListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        System.out.println("com.lct.listener.SystemListener::服务器启动.....");
    }
    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        System.out.println("com.lct.listener.SystemListener::服务器关闭.....");
    }
}
```

2、注册 ServletListenerRegistrationBean：

```java
import com.lct.component.MyLocaleResolve;
import com.lct.filter.SystemFilter;
import com.lct.listener.SystemListener;
import com.lct.servlet.BookServlet;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.boot.web.servlet.ServletListenerRegistrationBean;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import java.util.Arrays;
/**
 * Created by Administrator 
 * 自定义配置类
 */
@Configuration
public class MyMvcConfig {
    /**
     * 注册 Servlet 三大组件 之  Listner
     * 添加 ServletListenerRegistrationBean ，就相当于以前在 web.xml 中配置的 <listener></listener>标签
     */
    @Bean
    public ServletListenerRegistrationBean myListener() {
        /**ServletListenerRegistrationBean<T extends EventListener> 属于的是泛型，可以注册常见的任意监听器
         * 将自己的监听器注册进来*/
        ServletListenerRegistrationBean registrationBean = new ServletListenerRegistrationBean(new SystemListener());
        return registrationBean;
    }
}
```

### 注解方式

1、Servlet 三大组件 Servlet、Filter、Listener 在传统项目中需要在 web.xml 中进行相应的配置。Servlet 3.0 开始在 javax.servlet.annotation 包下提供 3 个对应

的 @WebServlet、@WebFilter、@WebListener 注解来简化操作。

2、Spring Boot 应用中这三个注解默认是不被扫描的，需要在项目启动类上添加 @ServletComponentScan 注解, 表示对 Servlet 组件扫描。

#### @WebServlet

```java
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
 
/**
 * 标准的 Servlet ，实现 javax.servlet.http.HttpServlet. 重写其 doGet 、doPost 方法
 * name :表示 servlet 名称，可以不写，默认为空
 * urlPatterns: 表示请求的路径，如 http://ip:port/context-path/userServlet
 */
@WebServlet(name = "UserServlet", urlPatterns = {"/userServlet"})
public class UserServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doPost(req, resp);
    }
 
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        StringBuffer requestURL = req.getRequestURL();
        System.out.println("com.wmx.servlet.UserServlet -- " + requestURL);
        resp.sendRedirect("/index.html");//浏览器重定向到服务器下的 index.html 页面
    }
}
```

#### @WebFilter

```java
import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
 
/**
 * 标准 Servlet 过滤器，实现 javax.servlet.Filter 接口，并重现它的 3 个方法
 * filterName：表示过滤器名称，可以不写
 * value：配置请求过滤的规则，如 "/*" 表示过滤所有请求，包括静态资源，如 "/user/*" 表示 /user 开头的所有请求
 */
@WebFilter(filterName = "SystemFilter", value = {"/*"})
public class SystemFilter implements Filter {
 
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("com.wmx.servlet.SystemFilter -- 系统启动...");
    }
 
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        //转为 HttpServletRequest 输出请求路径
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        System.out.println("com.wmx.servlet.SystemFilter -- 过滤器放行前...." + request.getRequestURL());
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("com.wmx.servlet.SystemFilter -- 过滤器返回后...." + request.getRequestURL());
    }
 
    @Override
    public void destroy() {
        System.out.println("com.wmx.servlet.SystemFilter -- 系统关闭...");
    }
}
```

#### @WebListener

```java
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;
 
/**
 * 标准 Servlet 监听器，实现 javax.servlet.ServletContextListener 接口，并重写方法
 * ServletContextListener 属于 Servlet 应用启动关闭监听器，监听容器初始化与销毁。常用的监听器还有：
 * ServletRequestListener：HttpServletRequest 对象的创建和销毁监听器
 * HttpSessionListener：HttpSession 数据对象创建和销毁监听器
 * HttpSessionAttributeListener 监听HttpSession中属性变化
 * ServletRequestAttributeListener 监听ServletRequest中属性变化
 */
@WebListener
public class SystemListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("com.wmx.servlet.SystemListener -- 服务器启动.");
    }
 
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("com.wmx.servlet.SystemListener -- 服务器关闭.");
    }
}
```

#### @ServletComponentScan

Spring Boot 应用中这三个注解默认是不被扫描的，需要在项目启动类上添加 @ServletComponentScan 注解, 表示对 Servlet 组件扫描。

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletComponentScan;
 
@SpringBootApplication
@ServletComponentScan //对 servlet 注解进行扫描
public class RedisStuWebApplication {
    public static void main(String[] args) {
        SpringApplication.run(RedisStuWebApplication.class, args);
    }
}
```
![](https://tinaxiawuhao.github.io/post-images/1622190795511.gif)
