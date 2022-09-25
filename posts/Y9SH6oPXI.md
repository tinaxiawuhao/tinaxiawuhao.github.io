---
title: 'Spring使用ThreadLocal存储内存泄露风险'
date: 2022-08-23 20:34:58
tags: [spring]
published: true
hideInList: false
feature: /post-images/Y9SH6oPXI.png
isTop: false
---
Spring使用ThreadLocal存储内存泄露风险
Spring 中有时候我们需要存储一些和 Request 相关联的变量，例如用户的登陆有关信息等，它的生命周期和 Request 相同。

一个容易想到的实现办法是使用ThreadLocal
```java
@Slf4j
public class CurrentUserHelper {
    //存储用户信息
    private static final InheritableThreadLocal<SysUser> CURRENT_USER = new InheritableThreadLocal<SysUser>();

    public static void setCurrentUser(SysUser currentUserInfo) {
        CURRENT_USER.set(currentUserInfo);
    }

    public static SysUser getCurrentUser() {
        return (SysUser)CURRENT_USER.get();
    }

    public static void clear() {
        CURRENT_USER.remove();
    }

}
```


使用一个自定义的HandlerInterceptor将有关信息注入进去

```java
@Slf4j
@Component
public class RequestInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws
            Exception {
        try {
            CurrentUserHelper.setCurrentUser(retrieveUserInfo(request));
        } catch (Exception ex) {
            log.warn("读取请求信息失败", ex);
        }
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable
            ModelAndView modelAndView) throws Exception {
        CurrentUserHelper.clear();

}
```

通过这样，我们就可以在 Controller 中直接使用这个 context，很方便的获取到有关用户的信息

```java
@Slf4j
@RestController
class Controller {
  public Result get() {
     SysUser user = CurrentUserHelper.get();
     // ...
  }

}
```

这个方法也是很多博客中使用的。然而这个方法却存在着一个很隐蔽的坑： HandlerInterceptor 的 postHandle 并不总是会调用。

当Controller中出现Exception

```java
@Slf4j
@RestController
class Controller {

  public Result get() {
      SysUser user = CurrentUserHelper.get();
     // ...
     throw new RuntimeException();
  }
}
```

或者在HandlerInterceptor的preHandle中出现Exception

```java
@Slf4j
@Component
public class RequestInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws
            Exception {
        try {
            CurrentUserHelper.setCurrentUser(retrieveUserInfo(request));
        } catch (Exception ex) {
            log.warn("读取请求信息失败", ex);
        }
    
        // ...
        throw new RuntimeException();
        //...
        return true;
    }
}
```

这些情况下， postHandle 并不会调用。这就导致了 ThreadLocal 变量不能被清理。

在平常的 Java 环境中，ThreadLocal 变量随着 Thread 本身的销毁，是可以被销毁掉的。但 Spring 由于采用了线程池的设计，响应请求的线程可能会一直常驻，这就导致了变量一直不能被 GC 回收。更糟糕的是，这个没有被正确回收的变量，由于线程池对线程的复用，可能会串到别的 Request 当中，进而直接导致代码逻辑的错误。

为了解决这个问题，我们可以使用 Spring 自带的 RequestContextHolder ，它背后的原理也是 ThreadLocal，不过它总会被更底层的 Servlet 的 Filter 清理掉，因此不存在泄露的问题。

下面是一个使用RequestContextHolder重写的例子

```java
public class SecurityContextHolder {

    private static final String SECURITY_CONTEXT_ATTRIBUTES = "SECURITY_CONTEXT";
    
    public static void setContext(SysUser context) {
        RequestContextHolder.currentRequestAttributes().setAttribute(
                SECURITY_CONTEXT_ATTRIBUTES,
                context,
                RequestAttributes.SCOPE_REQUEST);
    
    }
    
    public static SecurityContext get() {
        return (SysUser)RequestContextHolder.currentRequestAttributes()
                .getAttribute(SECURITY_CONTEXT_ATTRIBUTES, RequestAttributes.SCOPE_REQUEST);
    }
}
```

项目中实现WebMvcConfigurer接口或者WebMvcConfigurationSupport接口实现跨域资源访问配置时，要重写addInterceptors方法

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        //配置允许跨域访问的路径
        registry.addMapping("/**")
                .allowedOrigins("*")
                // 允许提交请求的方法，*表示全部允许
                .allowedMethods("GET", "POST", "DELETE", "PUT")
                .allowedHeaders("*")
                .exposedHeaders("Access-Control-Allow-Origin")
                //是否允许证书 不再默认开启
                .allowCredentials(true)
                .maxAge(3600);
    }

    @Override
    public void addInterceptors(InterceptorRegistry interceptor) {
        interceptor.addInterceptor(new RequestInterceptor()).addPathPatterns("/**");
    }

}
```

除了使用 RequestContextHolder 还可以使用 Request Scope 的 Bean，或者使用 ThreadLocalTargetSource ，原理上是类似的。

需要时刻注意 ThreadLocal 相当于线程内部的 static 变量，是一个非常容易产生泄露的点，因此使用 ThreadLocal 应该额外小心。

Threadlocal可能会产生内存泄露的问题及原理
刚遇到一个关于threadlocal的内存泄漏问题，刚好总结一下

比较常用的这里先不提，直接提比较重要的部分

为什么会产生内存泄露？

```java
public void set(T value) {

    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

set方法里面，先调用到当前线程thread，每个线程里都会有一个threadlocals成员变量，指向对应的ThreadLocalMap ，然后以new出来的引用作为key，和给定的value一块保存起来。

当外部引用解除以后，对应的ThreadLocal对象由于被内部ThreadLocalMap 引用，不会GC，可能会导致内存泄露。

JVM解决的办法

```java
static class Entry extends WeakReference<ThreadLocal<?>> {

    /** The value associated with this ThreadLocal. */
    Object value;
    
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }  
}
```

继承了一个软引用，在系统进行gc的时候就可以回收

但是回收以后，key变成null，value也无法被访问到，还是可能存在内存泄露。 因此一旦不用了，必须对里面的keyvalue对remove掉，否则就会有内存泄露；而且在threadlocal源码里面，在每次get或者set的时候会清楚里面key为value的记录