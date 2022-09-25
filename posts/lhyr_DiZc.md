---
title: 'gradle项目与任务'
date: 2021-07-03 19:32:31
tags: [gradle]
published: true
hideInList: false
feature: /post-images/lhyr_DiZc.png
isTop: false
---
### 项目与任务

Gradle中的所有内容都位于两个基本概念之上：

- projects ：每个Gradle构建都由一个或多个 projects组成 ，一个projects代表什么取决于您使用Gradle做的事情。例如，一个projects可能代表一个JAR库或一个Web应用程序。
- tasks ：每个projects由一个或多个 tasks组成 。tasks代表构建执行的一些原子工作。这可能是编译某些类，创建JAR，生成Javadoc或将一些存档发布到存储库。

### 项目

表.项目属性

| 名称        | 类型              | 默认值               |
| :---------- | :---------------- | :------------------- |
| project     | Project           | 该Project实例        |
| name        | String            | 项目目录的名称。     |
| path        | String            | 项目的绝对路径。     |
| description | String 项目说明。 |                      |
| projectDir  | File              | 包含构建脚本的目录。 |
| buildDir    | File              | *projectDir* /build  |
| group       | Object            | unspecified          |
| version     | Object            | unspecified          |
| ant         | ant build         | 一个AntBuilder实例   |

### 任务

#### 定义任务

1. 使用字符串作为任务名称定义任务
2. 使用tasks容器定义任务
3. 使用DSL特定语法定义任务

例：
**build.gradle**

```java
// 使用字符串作为任务名称定义任务
task('hello') {
    doLast {
        println "hello"
    }
}
// 使用tasks容器定义任务
tasks.create('hello') {
    doLast {
        println "hello"
    }
}
// 使用DSL特定语法定义任务
task(hello) {
    doLast {
        println "hello"
    }
}
task('copy', type: Copy) {
    from(file('srcDir'))
    into(buildDir)
}
```

#### 定位任务

- 使用DSL特定语法访问任务
- 通过任务集合访问任务
- 通过路径访问
- 按任务类型访问任务

```java
task hello
task copy(type: Copy)
 
 
// 使用DSL特定语法访问任务
println hello.name
println project.hello.name
 
println copy.destinationDir
println project.copy.destinationDir
 
// 通过任务集合访问任务
println tasks.hello.name
println tasks.named('hello').get().name
 
println tasks.copy.destinationDir
println tasks.named('copy').get().destinationDir
 
 
//按任务类型访问任务
tasks.withType(Tar).configureEach {
    enabled = false
}
 
task test {
    dependsOn tasks.withType(Copy)
}
```

通过路径访问

**project-a / build.gradle**

```shell
task hello
```

**build.gradle**

```java
task hello
 
println tasks.getByPath('hello').path
println tasks.getByPath(':hello').path
println tasks.getByPath('project-a:hello').path
println tasks.getByPath(':project-a:hello').path
```

#### 配置任务

##### 使用API配置任务

例：
**build.gradle**

```java
Copy myCopy = tasks.getByName("myCopy")
myCopy.from 'resources'
myCopy.into 'target'
myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')
```

##### 使用DSL特定语法配置任务

例：
**build.gradle**

```java
// Configure task using Groovy dynamic task configuration block
myCopy {
   from 'resources'
   into 'target'
}
myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')
```

##### 用配置块定义一个任务

例：
**build.gradle**

```java
task copy(type: Copy) {
   from 'resources'
   into 'target'
   include('**/*.txt', '**/*.xml', '**/*.properties')
}
```

#### 将参数传递给任务构造函数

与Task在创建后配置可变属性相反，您可以将参数值传递给Task类的构造函数。为了将值传递给Task构造函数，您必须使用@javax.inject.Inject注释相关的构造函数。

首先创建带有@Inject构造函数的任务类

```java
class CustomTask extends DefaultTask {
    final String message
    final int number
 
    @Inject
    CustomTask(String message, int number) {
        this.message = message
        this.number = number
    }
}
```

然后创建一个任务，并在参数列表的末尾传递构造函数参数。

```java
tasks.create('myTask', CustomTask, 'hello', 42)
```

你也可以使用Map创建带有构造函数参数的任务

```java
task myTask(type: CustomTask, constructorArgs: ['hello', 42])
```

#### 向任务添加依赖项

从另一个项目添加对任务的依赖

```java
project('project-a') {
    task taskX {
        dependsOn ':project-b:taskY'
        doLast {
            println 'taskX'
        }
    }
}
 
project('project-b') {
    task taskY {
        doLast {
            println 'taskY'
        }
    }
}
```

#### 使用任务对象添加依赖

```java
task taskX {
    doLast {
        println 'taskX'
    }
}
 
task taskY {
    doLast {
        println 'taskY'
    }
}
 
 
taskX.dependsOn taskY
```

#### 使用惰性块添加依赖项

```java
task taskX {
    doLast {
        println 'taskX'
    }
}
 
// Using a Groovy Closure
taskX.dependsOn {
    tasks.findAll { task -> task.name.startsWith('lib') }
}
 
task lib1 {
    doLast {
        println 'lib1'
    }
}
 
task lib2 {
    doLast {
        println 'lib2'
    }
}
 
task notALib {
    doLast {
        println 'notALib'
    }
}
```

#### 任务排序

控制任务排序的两种方式：

- must run after ：必须在之后运行
- should run after：应该在之后运行

例

```java
task taskX {
    doLast {
        println 'taskX'
    }
}
task taskY {
    doLast {
        println 'taskY'
    }
}
taskY.mustRunAfter taskX
```

should run after被忽略的情况

- 引入排序周期。
- 使用并行执行时，除了 "should run after "任务外，一个任务的所有依赖关系都已被满足，

引入排序周期例子

```java
task taskX {
    doLast {
        println 'taskX'
    }
}
task taskY {
    doLast {
        println 'taskY'
    }
}
task taskZ {
    doLast {
        println 'taskZ'
    }
}
taskX.dependsOn taskY
taskY.dependsOn taskZ
taskZ.shouldRunAfter taskX
```

#### 为任务添加描述

您可以在任务中添加描述。执行gradle tasks时将显示此描述。

build.gradle

```java
task copy(type: Copy) {
   description 'Copies the resource directory to the target directory.'
   from 'resources'
   into 'target'
   include('**/*.txt', '**/*.xml', '**/*.properties')
}
```

#### 跳过任务

onlyIf跳过

```java
hello.onlyIf { !project.hasProperty('skipHello') }
//StopExecutionException跳过
compile.doFirst {
    if (true) { throw new StopExecutionException() }
}
```

禁用任务

```java
task disableMe {
    doLast {
        println 'This should not be printed if the task is disabled.'
    }
}
disableMe.enabled = false
```

任务超时

```java
task hangingTask() {
    doLast {
        Thread.sleep(100000)
    }
    timeout = Duration.ofMillis(500)
}
```

#### 任务规则

有时您想执行一个任务，该任务的行为取决于较大或无限数量的参数值范围。提供此类任务的一种非常好的表达方式是任务规则：

任务规则

```java
tasks.addRule("Pattern: ping<ID>") { String taskName ->
    if (taskName.startsWith("ping")) {
        task(taskName) {
            doLast {
                println "Pinging: " + (taskName - 'ping')
            }
        }
    }
}
 
task groupPing {
    dependsOn pingServer1, pingServer2
}
> gradle -q groupPing
 
Ping：Server1
Ping：Server2
```

#### 终结器任务

计划运行终结任务时，终结任务会自动添加到任务图中。即使完成任务失败，也将执行终结器任务。

```java
task taskX {
    doLast {
        println 'taskX'
    }
}
task taskY {
    doLast {
        println 'taskY'
    }
}
taskX.finalizedBy taskY
> gradle -q taskX
 
TaskX
TaskY
```

#### 动态任务

Groovy或Kotlin的功能可用于定义任务以外的其他功能。例如，您还可以使用它来动态创建任务。

build.gradle

```java
4.times { counter ->
    task "task$counter" {
        doLast {
            println "I'm task number $counter"
        }
    }
}
```

gradle -q task1 输出

```shell
> gradle -q task1
I'm task number 1
```

#### Groovy_DSL快捷方式符号

访问任务有一种方便的表示法。每个任务都可以作为构建脚本的属性来使用：

例.作为构建脚本的属性访问任务

build.gradle

```java
task hello {
    doLast {
        println 'Hello world!'
    }
}
hello.doLast {
    println "Greetings from the $hello.name task."
}
```

输出 gradle -q hello

```shell
> gradle -q hello
Hello world!
Greetings from the hello task.
```

这将启用非常可读的代码，尤其是在使用插件提供的任务（例如compile任务）时。

#### 额外任务属性

您可以将自己的属性添加到任务。要添加名为的属性myProperty，请设置ext.myProperty为初始值。从那时起，可以像预定义的任务属性一样读取和设置属性。

build.gradle

```java
task myTask {
    ext.myProperty = "myValue"
}
 
task printTaskProperties {
    doLast {
        println myTask.myProperty
    }
}
```

输出 gradle -q printTaskProperties

```shell
> gradle -q printTaskProperties
myValue
```

额外的属性不仅限于任务。您可以在Extra属性中阅读有关它们的更多信息。

#### 默认任务

Gradle允许您定义一个或多个默认任务。

build.gradle

```java
defaultTasks 'clean', 'run'
 
task clean {
    doLast {
        println 'Default Cleaning!'
    }
}
 
task run {
    doLast {
        println 'Default Running!'
    }
}
 
task other {
    doLast {
        println "I'm not a default task!"
    }
}
```

输出 gradle -q

```shell
> gradle -q
Default Cleaning!
Default Running!
```

这等效于运行gradle clean run。在多项目构建中，每个子项目可以有其自己的特定默认任务。如果子项目未指定默认任务，则使用父项目的默认任务（如果已定义）。