---
title: '代理模式'
date: 2021-09-13 11:18:02
tags: [java,spring]
published: true
hideInList: false
feature: /post-images/CO5c8R39E.png
isTop: false
---
静态代理、JDK动态代理以及CGLIB动态代理
代理模式是java中最常用的设计模式之一，尤其是在spring框架中广泛应用。对于java的代理模式，一般可分为：静态代理、动态代理、以及CGLIB实现动态代理。
对于上述三种代理模式，分别进行说明。
## 静态代理
静态代理其实就是在程序运行之前，提前写好被代理方法的代理类，编译后运行。在程序运行之前，class已经存在。
下面我们实现一个静态代理demo:

![](https://tinaxiawuhao.github.io/post-images/1634527030816.png)

定义一个接口Target

```java
package com.test.proxy;

public interface Target {

    public String execute();

}
```

TargetImpl 实现接口Target

```java
package com.test.proxy;

public class TargetImpl implements Target {

    @Override
    public String execute() {
        System.out.println("TargetImpl execute！");
        return "execute";
    }

}
```



代理类

```java
package com.test.proxy;

public class Proxy implements Target{

    private Target target;
    
    public Proxy(Target target) {
        this.target = target;
    }
    
    @Override
    public String execute() {
        System.out.println("perProcess");
        String result = this.target.execute();
        System.out.println("postProcess");
        return result;
    }

}
```



测试类:

```java
package com.test.proxy;

public class ProxyTest {

    public static void main(String[] args) {
    
        Target target = new TargetImpl();
        Proxy p = new Proxy(target);
        String result =  p.execute();
        System.out.println(result);
    }

}
```



运行结果:

> perProcess
> TargetImpl execute！
> postProcess
> execute



静态代理需要针对被代理的方法提前写好代理类，如果被代理的方法非常多则需要编写很多代码，因此，对于上述缺点，通过动态代理的方式进行了弥补。

## 动态代理

### jdk代理

动态代理主要是通过反射机制，在运行时动态生成所需代理的class.

![](https://tinaxiawuhao.github.io/post-images/1634527039701.png)

接口

```java
package com.test.dynamic;

public interface Target {

    public String execute();

}
```



实现类

```java
package com.test.dynamic;

public class TargetImpl implements Target {

    @Override
    public String execute() {
        System.out.println("TargetImpl execute！");
        return "execute";
    }

}
```



代理类

```java
package com.test.dynamic;


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class DynamicProxyHandler implements InvocationHandler{

    private Target target;
    
    public DynamicProxyHandler(Target target) {
        this.target = target;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("========before==========");
        Object result = method.invoke(target,args);
        System.out.println("========after===========");
        return result;
    }

}
```



测试类

```java
package com.test.dynamic;

import java.lang.reflect.Proxy;

public class DynamicProxyTest {

    public static void main(String[] args) {
        Target target = new TargetImpl();
        DynamicProxyHandler handler = new DynamicProxyHandler(target);
        Target proxySubject = (Target) Proxy.newProxyInstance(TargetImpl.class.getClassLoader(),TargetImpl.class.getInterfaces(),handler);
        String result = proxySubject.execute();
        System.out.println(result);
    }

}
```



运行结果：

> ========before==========
> TargetImpl execute！
> ========after===========
> execute



无论是动态代理还是静态带领，都需要定义接口，然后才能实现代理功能。这同样存在局限性，因此，为了解决这个问题，出现了第三种代理方式：cglib代理。
### cglib代理
CGLib采用了非常底层的字节码技术，其原理是通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。JDK动态代理与CGLib动态代理均是实现Spring AOP的基础。

![](https://tinaxiawuhao.github.io/post-images/1634527048720.png)

目标类

```java
package com.test.cglib;

public class Target {

    public String execute() {
        String message = "-----------test------------";
        System.out.println(message);
        return message;
    }

}
```



通用代理类

```java
package com.test.cglib;

import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class MyMethodInterceptor implements MethodInterceptor{

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println(">>>>MethodInterceptor start...");
        Object result = proxy.invokeSuper(obj,args);
        System.out.println(">>>>MethodInterceptor ending...");
        return "result";
    }

}
```



测试类

```java
package com.test.cglib;

import net.sf.cglib.proxy.Enhancer;

public class CglibTest {

    public static void main(String[] args) {
        System.out.println("***************");
        Target target = new Target();
        CglibTest test = new CglibTest();
        Target proxyTarget = (Target) test.createProxy(Target.class);
        String res = proxyTarget.execute();
        System.out.println(res);
    }
    
    public Object createProxy(Class targetClass) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetClass);
        enhancer.setCallback(new MyMethodInterceptor());
        return enhancer.create();
    }

}
```




执行结果:
***************
>MethodInterceptor start...
>-----------test------------
>MethodInterceptor ending...
>result

代理对象的生成过程由Enhancer类实现，大概步骤如下：

1. 生成代理类Class的二进制字节码；
2. 通过Class.forName加载二进制字节码，生成Class对象；
3. 通过反射机制获取实例构造，并初始化代理类对象。