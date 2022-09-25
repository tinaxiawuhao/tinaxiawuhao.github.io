---
title: 'Groovy基础'
date: 2021-07-04 19:33:24
tags: [gradle]
published: true
hideInList: false
feature: /post-images/4WWjxpN2N.png
isTop: false
---
### Groovy基础

### 基本规则

- 没有分号
- 方法括号可以省略
- 方法可以不写return，返回最后一句代码
- 代码块可以作为参数传递

### 定义

```java
def param = 'hello world'
def param1 = "hello world"
println "${param1} ,li" 
```

${}里面可以放变量，也可以是表达式，只有双引号里面可以使用

### 声明变量

可以在构建脚本中声明两种变量：局部变量和额外属性。

#### 局部变量

局部变量用def关键字声明。它们仅在声明它们的范围内可见。局部变量是基础Groovy语言的功能。

```java
def dest = "dest"
task copy(type: Copy) {
    from "source"
    into dest
}
```

#### ext属性

Gradle的域模型中的所有增强对象都可以容纳额外的用户定义属性。

可以通过拥有对象的ext属性添加，读取和设置其他属性。可以使用一个ext块一次添加多个属性。

```java
plugins {
    id 'java'
}
 
ext {
    springVersion = "3.1.0.RELEASE"
    emailNotification = "build@master.org"
}
 
sourceSets.all { ext.purpose = null }
 
sourceSets {
    main {
        purpose = "production"
    }
    test {
        purpose = "test"
    }
    plugin {
        purpose = "production"
    }
}
 
task printProperties {
    doLast {
        println springVersion
        println emailNotification
        sourceSets.matching { it.purpose == "production" }.each { println it.name }
    }
}
```

输出 gradle -q printProperties

```shell
> gradle -q printProperties
3.1.0.RELEASE
build@master.org
main
plugin
```

#### 变量范围：本地和脚本范围

用类型修饰符声明的变量在闭包中可见，但在方法中不可见。

```java
String localScope1 = 'localScope1'
def localScope2 = 'localScope2'
scriptScope = 'scriptScope'
 
println localScope1
println localScope2
println scriptScope
 
closure = {
    println localScope1
    println localScope2
    println scriptScope
}
 
def method() {
    try {
        localScope1
    } catch (MissingPropertyException e) {
        println 'localScope1NotAvailable'
    }
    try {
        localScope2
    } catch(MissingPropertyException e) {
        println 'localScope2NotAvailable'
    }
    println scriptScope
}
 
closure.call()
method()
```

输出 groovy scope.groovy

```shell
> groovy作用域
localScope1
localScope2
scriptScope
localScope1
localScope2
scriptScope
localScope1NotAvailable
localScope2NotAvailable
scriptScope
```

### 对象

#### 使用对象

您可以按照以下易读的方式配置任意对象。

build.gradle

```java
import java.text.FieldPosition
 
task configure {
    doLast {
        def pos = configure(new FieldPosition(10)) {
            beginIndex = 1
            endIndex = 5
        }
        println pos.beginIndex
        println pos.endIndex
    }
}
> gradle -q configure
1
5
```

#### 使用外部脚本配置任意对象

您也可以使用外部脚本配置任意对象。

build.gradle

```java
task configure {
    doLast {
        def pos = new java.text.FieldPosition(10)
        // Apply the script
        apply from: 'other.gradle', to: pos
        println pos.beginIndex
        println pos.endIndex
    }
}
```

other.gradle

```java
// Set properties.
beginIndex = 1
endIndex = 5
```

输出 gradle -q configure

```shell
> gradle -q configure
1
5
```

### 属性访问器

Groovy自动将属性引用转换为对适当的getter或setter方法的调用。

build.gradle

```java
// Using a getter method
println project.buildDir
println getProject().getBuildDir()
 
// Using a setter method
project.buildDir = 'target'
getProject().setBuildDir('target')
```

### 闭包

闭包（闭合代码块，可以引用传入的变量）

```java
task testClosure {
    doLast {
        func {
            println it
        }
        funa { a, b ->
            println a + b
        }
    }
}
 
def funa(closure) {
    closure(10, 3)
}
 
def func(closure) {
    closure(10)
}
```

#### 闭包委托

每个闭包都有一个`delegate`对象，Groovy使用该对象来查找不是闭包的局部变量或参数的变量和方法引用。

例.闭包委托

```java
class Info {
    int id;
    String code;
 
    def log() {
        println("code:${code};id:${id}")
    }
}
 
def info(Closure<Info> closure) {
    Info p = new Info()
    closure.delegate = p
    // 委托模式优先
    closure.setResolveStrategy(Closure.DELEGATE_FIRST)
    closure(p)
}
 
task configClosure {
    doLast {
        info {
            code = "cix"
            id = 1
            log()
        }
    }
}
```

输出

```shell
> Task :configClosure
code:cix;id:1
 
BUILD SUCCESSFUL in 276ms
```

例：使用必包委托设置依赖

```java
dependencies {
    assert delegate == project.dependencies
    testImplementation('junit:junit:4.13')
    delegate.testImplementation('junit:junit:4.13')
}
```

### 方法

#### 方法调用上的可选括号

括号对于方法调用是可选的。

build.gradle

```java
test.systemProperty 'some.prop', 'value'
test.systemProperty('some.prop', 'value')
```

#### 闭包作为方法中的最后一个参数

当方法的最后一个参数是闭包时，可以将闭包放在方法调用之后：

build.gradle

```java
repositories {
    println "in a closure"
}
repositories() { println "in a closure" }
repositories({ println "in a closure" })
```

### 集合

#### List

```java
def list = [1,2,3,4]
println list[0] // 1
println list[-1] // 4 最后一个
println list[-2] // 3 倒数第二个
println list[0..2] // 第1-3个
 
list.each { //迭代
    println it
}
```

#### Map

```java
def map= ['name':'li', 'age':18]
println map[name] // li
println map.age // 18
 
list.each { //迭代
    println "${it.key}:${it.value}"
}
 
```

### JavaBean

```java
class A{
    private int a; //可通过A().a 获取修改
    public int getB(){//可通过A().b获取，但不能修改
        1
    }
}
```

build.gradle

```java
// List literal
test.includes = ['org/gradle/api/**', 'org/gradle/internal/**']
 
List<String> list = new ArrayList<String>()
list.add('org/gradle/api/**')
list.add('org/gradle/internal/**')
test.includes = list
 
// Map literal.
Map<String, String> map = [key1:'value1', key2: 'value2']
 
// Groovy will coerce named arguments
// into a single map argument
apply plugin: 'java'
```

### 默认导入

为了使构建脚本更简洁，Gradle自动向Gradle脚本添加了一些类。

```java
import org.gradle.*
import org.gradle.api.*
import org.gradle.api.artifacts.*
import org.gradle.api.artifacts.component.*
import org.gradle.api.artifacts.dsl.*
import org.gradle.api.artifacts.ivy.*
import org.gradle.api.artifacts.maven.*
import org.gradle.api.artifacts.query.*
import org.gradle.api.artifacts.repositories.*
import org.gradle.api.artifacts.result.*
import org.gradle.api.artifacts.transform.*
import org.gradle.api.artifacts.type.*
import org.gradle.api.artifacts.verification.*
import org.gradle.api.attributes.*
import org.gradle.api.attributes.java.*
import org.gradle.api.capabilities.*
import org.gradle.api.component.*
import org.gradle.api.credentials.*
import org.gradle.api.distribution.*
import org.gradle.api.distribution.plugins.*
import org.gradle.api.execution.*
import org.gradle.api.file.*
import org.gradle.api.initialization.*
import org.gradle.api.initialization.definition.*
import org.gradle.api.initialization.dsl.*
import org.gradle.api.invocation.*
import org.gradle.api.java.archives.*
import org.gradle.api.jvm.*
import org.gradle.api.logging.*
import org.gradle.api.logging.configuration.*
import org.gradle.api.model.*
import org.gradle.api.plugins.*
import org.gradle.api.plugins.antlr.*
import org.gradle.api.plugins.quality.*
import org.gradle.api.plugins.scala.*
import org.gradle.api.provider.*
import org.gradle.api.publish.*
import org.gradle.api.publish.ivy.*
import org.gradle.api.publish.ivy.plugins.*
import org.gradle.api.publish.ivy.tasks.*
import org.gradle.api.publish.maven.*
import org.gradle.api.publish.maven.plugins.*
import org.gradle.api.publish.maven.tasks.*
import org.gradle.api.publish.plugins.*
import org.gradle.api.publish.tasks.*
import org.gradle.api.reflect.*
import org.gradle.api.reporting.*
import org.gradle.api.reporting.components.*
import org.gradle.api.reporting.dependencies.*
import org.gradle.api.reporting.dependents.*
import org.gradle.api.reporting.model.*
import org.gradle.api.reporting.plugins.*
import org.gradle.api.resources.*
import org.gradle.api.services.*
import org.gradle.api.specs.*
import org.gradle.api.tasks.*
import org.gradle.api.tasks.ant.*
import org.gradle.api.tasks.application.*
import org.gradle.api.tasks.bundling.*
import org.gradle.api.tasks.compile.*
import org.gradle.api.tasks.diagnostics.*
import org.gradle.api.tasks.incremental.*
import org.gradle.api.tasks.javadoc.*
import org.gradle.api.tasks.options.*
import org.gradle.api.tasks.scala.*
import org.gradle.api.tasks.testing.*
import org.gradle.api.tasks.testing.junit.*
import org.gradle.api.tasks.testing.junitplatform.*
import org.gradle.api.tasks.testing.testng.*
import org.gradle.api.tasks.util.*
import org.gradle.api.tasks.wrapper.*
import org.gradle.authentication.*
import org.gradle.authentication.aws.*
import org.gradle.authentication.http.*
import org.gradle.build.event.*
import org.gradle.buildinit.plugins.*
import org.gradle.buildinit.tasks.*
import org.gradle.caching.*
import org.gradle.caching.configuration.*
import org.gradle.caching.http.*
import org.gradle.caching.local.*
import org.gradle.concurrent.*
import org.gradle.external.javadoc.*
import org.gradle.ide.visualstudio.*
import org.gradle.ide.visualstudio.plugins.*
import org.gradle.ide.visualstudio.tasks.*
import org.gradle.ide.xcode.*
import org.gradle.ide.xcode.plugins.*
import org.gradle.ide.xcode.tasks.*
import org.gradle.ivy.*
import org.gradle.jvm.*
import org.gradle.jvm.application.scripts.*
import org.gradle.jvm.application.tasks.*
import org.gradle.jvm.platform.*
import org.gradle.jvm.plugins.*
import org.gradle.jvm.tasks.*
import org.gradle.jvm.tasks.api.*
import org.gradle.jvm.test.*
import org.gradle.jvm.toolchain.*
import org.gradle.language.*
import org.gradle.language.assembler.*
import org.gradle.language.assembler.plugins.*
import org.gradle.language.assembler.tasks.*
import org.gradle.language.base.*
import org.gradle.language.base.artifact.*
import org.gradle.language.base.compile.*
import org.gradle.language.base.plugins.*
import org.gradle.language.base.sources.*
import org.gradle.language.c.*
import org.gradle.language.c.plugins.*
import org.gradle.language.c.tasks.*
import org.gradle.language.coffeescript.*
import org.gradle.language.cpp.*
import org.gradle.language.cpp.plugins.*
import org.gradle.language.cpp.tasks.*
import org.gradle.language.java.*
import org.gradle.language.java.artifact.*
import org.gradle.language.java.plugins.*
import org.gradle.language.java.tasks.*
import org.gradle.language.javascript.*
import org.gradle.language.jvm.*
import org.gradle.language.jvm.plugins.*
import org.gradle.language.jvm.tasks.*
import org.gradle.language.nativeplatform.*
import org.gradle.language.nativeplatform.tasks.*
import org.gradle.language.objectivec.*
import org.gradle.language.objectivec.plugins.*
import org.gradle.language.objectivec.tasks.*
import org.gradle.language.objectivecpp.*
import org.gradle.language.objectivecpp.plugins.*
import org.gradle.language.objectivecpp.tasks.*
import org.gradle.language.plugins.*
import org.gradle.language.rc.*
import org.gradle.language.rc.plugins.*
import org.gradle.language.rc.tasks.*
import org.gradle.language.routes.*
import org.gradle.language.scala.*
import org.gradle.language.scala.plugins.*
import org.gradle.language.scala.tasks.*
import org.gradle.language.scala.toolchain.*
import org.gradle.language.swift.*
import org.gradle.language.swift.plugins.*
import org.gradle.language.swift.tasks.*
import org.gradle.language.twirl.*
import org.gradle.maven.*
import org.gradle.model.*
import org.gradle.nativeplatform.*
import org.gradle.nativeplatform.platform.*
import org.gradle.nativeplatform.plugins.*
import org.gradle.nativeplatform.tasks.*
import org.gradle.nativeplatform.test.*
import org.gradle.nativeplatform.test.cpp.*
import org.gradle.nativeplatform.test.cpp.plugins.*
import org.gradle.nativeplatform.test.cunit.*
import org.gradle.nativeplatform.test.cunit.plugins.*
import org.gradle.nativeplatform.test.cunit.tasks.*
import org.gradle.nativeplatform.test.googletest.*
import org.gradle.nativeplatform.test.googletest.plugins.*
import org.gradle.nativeplatform.test.plugins.*
import org.gradle.nativeplatform.test.tasks.*
import org.gradle.nativeplatform.test.xctest.*
import org.gradle.nativeplatform.test.xctest.plugins.*
import org.gradle.nativeplatform.test.xctest.tasks.*
import org.gradle.nativeplatform.toolchain.*
import org.gradle.nativeplatform.toolchain.plugins.*
import org.gradle.normalization.*
import org.gradle.platform.base.*
import org.gradle.platform.base.binary.*
import org.gradle.platform.base.component.*
import org.gradle.platform.base.plugins.*
import org.gradle.play.*
import org.gradle.play.distribution.*
import org.gradle.play.platform.*
import org.gradle.play.plugins.*
import org.gradle.play.plugins.ide.*
import org.gradle.play.tasks.*
import org.gradle.play.toolchain.*
import org.gradle.plugin.devel.*
import org.gradle.plugin.devel.plugins.*
import org.gradle.plugin.devel.tasks.*
import org.gradle.plugin.management.*
import org.gradle.plugin.use.*
import org.gradle.plugins.ear.*
import org.gradle.plugins.ear.descriptor.*
import org.gradle.plugins.ide.*
import org.gradle.plugins.ide.api.*
import org.gradle.plugins.ide.eclipse.*
import org.gradle.plugins.ide.idea.*
import org.gradle.plugins.javascript.base.*
import org.gradle.plugins.javascript.coffeescript.*
import org.gradle.plugins.javascript.envjs.*
import org.gradle.plugins.javascript.envjs.browser.*
import org.gradle.plugins.javascript.envjs.http.*
import org.gradle.plugins.javascript.envjs.http.simple.*
import org.gradle.plugins.javascript.jshint.*
import org.gradle.plugins.javascript.rhino.*
import org.gradle.plugins.signing.*
import org.gradle.plugins.signing.signatory.*
import org.gradle.plugins.signing.signatory.pgp.*
import org.gradle.plugins.signing.type.*
import org.gradle.plugins.signing.type.pgp.*
import org.gradle.process.*
import org.gradle.swiftpm.*
import org.gradle.swiftpm.plugins.*
import org.gradle.swiftpm.tasks.*
import org.gradle.testing.base.*
import org.gradle.testing.base.plugins.*
import org.gradle.testing.jacoco.plugins.*
import org.gradle.testing.jacoco.tasks.*
import org.gradle.testing.jacoco.tasks.rules.*
import org.gradle.testkit.runner.*
import org.gradle.vcs.*
import org.gradle.vcs.git.*
import org.gradle.work.*
import org.gradle.workers.*
```

### 导入依赖

```java
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath group: 'commons-codec', name: 'commons-codec', version: '1.2'
    }
}
```