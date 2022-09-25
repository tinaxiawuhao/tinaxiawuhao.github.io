---
title: 'gradle-Logging'
date: 2021-07-05 19:34:29
tags: [gradle]
published: true
hideInList: false
feature: /post-images/wYNztHepH.png
isTop: false
---
### Logging

### 日志级别

| 日志级别  | 说明         |
| :-------- | :----------- |
| ERROR     | 错误讯息     |
| QUIET     | 重要信息消息 |
| WARNING   | 警告讯息     |
| LIFECYCLE | 进度信息消息 |
| INFO      | 信息讯息     |
| DEBUG     | 调试信息     |

### 选择日志级别

可以通过命令行选项或者gradle.properties文件配置

| 选项          | 输出日志级别                      |
| :------------ | :-------------------------------- |
| 没有记录选项  | LIFECYCLE及更高                   |
| -q or--quiet  | QUIET及更高                       |
| -w or --warn  | WARNING及更高                     |
| -i or --info  | INFO及更高                        |
| -d or --debug | DEBUG及更高版本（即所有日志消息） |

### Stacktrace命令行选项

**-s or --stacktrace**

打印简洁的堆栈跟踪信息，推荐使用

**-S or --full-stacktrace**

打印完整的堆栈跟踪信息。

### 编写日志

#### 使用stdout编写日志消息

```shell
println 'A message which is logged at QUIET level'
```

#### 编写自己的日志消息

```java
logger.quiet('An info log message which is always logged.')
logger.error('An error log message.')
logger.warn('A warning log message.')
logger.lifecycle('A lifecycle info log message.')
logger.info('An info log message.')
logger.debug('A debug log message.')
logger.trace('A trace log message.') // Gradle never logs TRACE level logs
 
// 用占位符写一条日志消息
logger.info('A {} log message', 'info')
```

logger构建脚本提供了一个属性，该脚本是Logger的实例。该接口扩展了SLF4JLogger接口，并向其中添加了一些Gradle特定的方法

### 使用SLF4J写入日志消息

```java
import org.slf4j.LoggerFactory
 
def slf4jLogger = LoggerFactory.getLogger('some-logger')
slf4jLogger.info('An info log message logged using SLF4j')
```

### 构建生命周期

Gradle的核心是一种基于依赖性的编程语言，这意味着你可以定义任务和任务之间的依赖关系。 Gradle保证这些任务按照其依赖关系的顺序执行，并且每个任务只执行一次。 这些任务形成一个定向无环图。

#### 构建阶段

Gradle构建具有三个不同的阶段。

- 初始化

  Gradle支持单项目和多项目构建。在初始化阶段，Gradle决定要参与构建的项目，并为每个项目创建一个Project实例。

- 配置

  在此阶段，将配置项目对象。执行作为构建一部分的 所有 项目的构建脚本。

- 执行

  Gradle确定要在配置阶段创建和配置的任务子集。子集由传递给gradle命令的任务名称参数和当前目录确定。然后Gradle执行每个选定的任务。

#### 设置文件

默认名称是settings.gradle

项目构建在多项目层次结构的根项目中必须具有一个settings.gradle文件。

对于单项目构建，设置文件是可选的

#### 初始化

查找settings.gradle文件判断是否多项目

没有settings.gradle或settings.gradle没有多项目配置则为单项目

例：将test任务添加到每个具有特定属性集的项目

build.gradle

```java
 allprojects {
    afterEvaluate { project ->
        if (project.hasTests) {
            println "Adding test task to $project"
            project.task('test') {
                doLast {
                    println "Running tests for $project"
                }
            }
        }
    }
}
 
```

输出 gradle -q test

```shell
> gradle -q test
Adding test task to project ':project-a'
Running tests for project ':project-a'
```

### 初始化脚本

初始化脚本与Gradle中的其他脚本相似。但是，这些脚本在构建开始之前运行。初始化脚本不能访问buildSrc项目中的类。

#### 使用初始化脚本

有几种使用初始化脚本的方法：

- 在命令行中指定一个文件。命令行选项是-I或-init-script，后面是脚本的路径。

命令行选项可以出现一次以上，每次都会添加另一个 init 脚本。 如果命令行上指定的文件不存在，编译将失败。

- 在 *USER_HOME* /.gradle/目录中放置一个名为init.gradle（或init.gradle.ktsKotlin）的文件。
- 在 *USER_HOME* /.gradle/init.d/目录中放置一个以.gradle（或.init.gradle.ktsKotlin）结尾的文件。
- 在Gradle发行版的 *GRADLE_HOME* /init.d/目录中放置一个以.gradle（或.init.gradle.ktsKotlin）结尾的文件。这使您可以打包包含一些自定义构建逻辑和插件的自定义Gradle发行版。您可以将其与Gradle Wrapper结合使用，以使自定义逻辑可用于企业中的所有内部版本。

如果发现一个以上的初始化脚本，它们将按照上面指定的顺序依次执行。

示例

build.gradle

```java
repositories {
    mavenCentral()
}
 
task showRepos {
    doLast {
        println "All repos:"
        println repositories.collect { it.name }
    }
}
 
```

init.gradle

```java
allprojects {
    repositories {
        mavenLocal()
    }
}
```

运行任务：

```shell
gradle --init-script init.gradle -q showRepos
```

#### 初始化脚本里面依赖添加依赖

init.gradle

```java
initscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.apache.commons:commons-math:2.0'
    }
}
```

### 多项目

可在settings.gradle文件中设置多个项目关系，如下项目结构

```shell
.
├── app
│   ...
│   └── build.gradle
└── settings.gradle
```

settings.gradle

```java
rootProject.name = 'basic-multiproject' //根项目名
include 'app' //子项目
 
```

子项目间依赖

```java
dependencies {
    implementation(project(":shared"))
}
```