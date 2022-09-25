---
title: 'Gradle基础'
date: 2021-07-02 19:31:04
tags: [gradle]
published: true
hideInList: false
feature: /post-images/Ke4qDxE4l.png
isTop: false
---
# Gradle概述

Gradle是专注于灵活性和性能的开源构建自动化工具，一般使用Groovy或KotlinDSL编写构建脚本。 本文只使用Groovy

## Gradle的特点：

- 高性能

Gradle通过仅运行需要运行的任务来避免不必要的工作。 可以使用构建缓存来重用以前运行的任务输出，甚至可以使用其他计算机（具有共享的构建缓存）重用任务输出。

- JVM基础

Gradle在JVM上运行。熟悉Java的用户来可以在构建逻辑中使用标准Java API，例如自定义任务类型和插件。 这使得Gradle跨平台更加简单。（Gradle不仅限于构建JVM项目，它甚至附带对构建本机项目的支持。）

- 约束

和Maven一样，Gradle通过实现约束使常见类型的项目（例如Java项目）易于构建。 应用适当的插件，您可以轻松地为许多项目使用精简的构建脚本。 但是这些约定并没有限制您：Gradle允许您覆盖它们，添加自己的任务以及对基于约定的构建进行许多其他自定义操作。

- 可扩展性

您可以轻松扩展Gradle以提供您自己的任务类型甚至构建模型。

- IDE支持

支持IDE：Android Studio，IntelliJ IDEA，Eclipse和NetBeans。 Gradle还支持生成将项目加载到Visual Studio所需的解决方案文件。

- 可洞察性

构建扫描提供了有关构建运行的广泛信息，可用于识别构建问题。他们特别擅长帮助您确定构建性能的问题。 您还可以与其他人共享构建扫描，如果您需要咨询以解决构建问题，这将特别有用。

## 您需要了解有关Gradle的五件事

本节在官方文档里面反复提及，具体可见[Gradle文档](https://docs.gradle.org/current/userguide/what_is_gradle.html#five_things)

### 1. Gradle是通用的构建工具

Gradle允许您构建任何软件，因为它不关心你具体的工作。

### 2. 核心模型基于任务

Gradle将其构建模型建模为任务（工作单元）的有向无环图（DAG）。这意味着构建实质上配置了一组任务，并根据它们的依赖关系将它们连接在一起以创建该DAG。创建任务图后，Gradle将确定需要按顺序运行的任务，然后继续执行它们。

任务本身包括以下部分，它们通过依赖链接在一起：

- 动作-做某事的工作，例如复制文件或编译源代码
- 输入-操作使用或对其进行操作的值，文件和目录
- 输出-操作修改或生成的文件和目录

### 3. Gradle有几个固定的构建阶段

重要的是要了解Gradle分三个阶段评估和执行构建脚本：

- 初始化

设置构建环境，并确定哪些项目将参与其中。

- 配置

构造和配置构建的任务图，然后根据用户要运行的任务确定需要运行的任务和运行顺序。

- 执行

运行在配置阶段结束时选择的任务。

这些阶段构成了Gradle的构建生命周期。

### 4. Gradle的扩展方式不止一种

Gradle捆绑的构建逻辑不可能满足所有构建情况，大多数构建都有一些特殊要求，你需要添加自定义构建逻辑。Gradle提供了多种机制来扩展它，例如：

- 自定义任务类型。
- 自定义任务动作。
- 项目和任务的额外属性。
- 自定义约束。
- 自定义module。

### 5. 构建脚本针对API运行

可以将Gradle的构建脚本视为可执行代码，但设计良好的构建脚本描述了构建软件需要哪些步骤，而不关心这些步骤应该如何完成工作。

由于Gradle在JVM上运行，因此构建脚本也可以使用标准Java API。Groovy构建脚本可以另外使用Groovy API，而Kotlin构建脚本可以使用Kotlin。

## 功能的生命周期

功能可以处于以下四种状态之一：

- Internal：内部功能，不提供接口
- Incubating： 孵化功能。在成为公共功能之前会继续更改
- Public：公共功能，可放心使用
- Deprecated：废弃功能，将在未来删除

# Gradle安装

## 安装JDK

安装JDK过程已有太多资料，本文不做详细介绍。可使用命令检测自己电脑是否成功安装 

![](https://tianxiawuhao.github.io/post-images/1658230657737.png)

## 安装Gradle

### 用软件包安装Gradle

SDKMAN

```shell
sdk install gradle
```

Homebrew

```shell
brew install gradle
```

### 手动安装（推荐方式）

#### 下载

[services.gradle.org/distributions](https://services.gradle.org/distributions/) （全部版本目录地址，可以查看最新版本）
[services.gradle.org/distributions/gradle-7.5-all.zip](https://services.gradle.org/distributions/gradle-7.5-all.zip) （截止至2022.07.18最新）

建议下载：[services.gradle.org/distributions/gradle-7.0.2-all.zip](https://services.gradle.org/distributions/gradle-7.0.3-all.zip) (支持jdk8)

#### 文件介绍

```java
gradle-7.0.2-docs.zip  //文档
gradle-7.0.2-src.zip  //源码
gradle-7.0.2-bin.zip  //软件包
gradle-7.0.2-all.zip   //全部文件
   bin ：运行文件
   lib：依赖库
   docs：文档
   src：源文件
   init.d :初始化脚本目录，可自己添加
```

#### 配置环境变量

```shell
export GRADLE_HOME=/Users/temp/gradle-7.0.2
export PATH=$PATH:$GRADLE_HOME/bin
```

#### 运行

输入gradle -v 检测是否配置成功 

![](https://tianxiawuhao.github.io/post-images/1658230676105.png)

### HelloWord

编写一个build.gradle文件，输入以下内容

```java
task hello{
    doLast {
        println 'Hello World'
    }
}
```

命令行输入`gradle -q hello`即可运行

# Gradle Wrapper

## 定义

Gradle Wrapper是一个脚本，可调用Gradle的声明版本，并在必要时预先下载。因此，开发人员可以快速启动并运行Gradle项目，而无需遵循手动安装过程
![](https://tianxiawuhao.github.io/post-images/1658230693887.png)

## 添加wrapper

在build.gradle同级目录下使用命令`gradle wrapper`可以生成gradle wrapper目录

```java
gradle wrapper
```

![](https://tianxiawuhao.github.io/post-images/1658230710999.png)

- gradle-wrapper.jar

  WrapJAR文件，其中包含用于下载Gradle发行版的代码。

- gradle-wrapper.properties

  一个属性文件，负责配置Wrapper运行时行为，例如与该版本兼容的Gradle版本。请注意，更多常规设置（例如，将 Wrap配置为使用代理）需要进入其他文件。

- gradlew， gradlew.bat

  一个shell脚本和一个Windows批处理脚本，用于使用 Wrap程序执行构建。

可以通过命令控制生成选项

```shell
#用于下载和执行 Wrap程序的Gradle版本。
--gradle-version

#Wrap使用的Gradle分布类型。可用的选项是bin和all。默认值为bin。
--distribution-type

#指向Gradle分发ZIP文件的完整URL。使用此选项，--gradle-version并且--distribution-     type过时的网址已经包含此信息。如果您想在公司网络内部托管Gradle发行版，则此选项非常有价值。
--gradle-distribution-url

#SHA256哈希和用于验证下载的Gradle分布。
--gradle-distribution-sha256-sum
```

例：

```shell
gradle wrapper --gradle-version 7.0.2 --distribution-type all
```

## Wrapper属性文件

一般生成Wrapper会得到如下属性文件 gradle-wrapper.properties

```java
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-7.4.1-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

GRADLE_USER_HOME是你的环境变量，如果没配置，则默认是用户目录下的.gradle文件夹

- distributionBase 下载的 Gradle压缩包解压后存储的主目录
- distributionPath 相对于 distributionBase的解压后的 Gradle压缩包的路径
- zipStoreBase 同 distributionBase，只不过是存放 zip压缩包的
- zipStorePath 同 distributionPath，只不过是存放 zip压缩包的
- distributionUrl Gradle发行版压缩包的下载地址

## 使用wrapper构建

在 gradlew目录下执行命令：
windows：

```shell
gradlew.bat build
```

shell：

```shell
./gradlew build
```

## 升级

- 更改gradle-wrapper.properties文件中的distributionUrl属性
- 使用gradlew wrap --gradle-version 命令

```shell
./gradlew wrap --gradle-version 7.4.2
```

## 自定义Gradle_Wrap

可以通过自定义wrapper少去一些重复操作或定制功能，如

build.gradle

```java
tasks.named('wrapper') {
    distributionType = Wrapper.DistributionType.ALL
}
task wrapper(type: Wrapper) {
    gradleVersion = '7.4.2'
}
```

# Gradle 环境

## 环境变量

`GRADLE_OPTS`

指定启动Gradle客户端VM时要使用的JVM参数。客户端VM仅处理命令行输入/输出，因此很少需要更改其VM选项。实际的构建由Gradle守护程序运行，不受此环境变量的影响。

`GRADLE_USER_HOME`

指定Gradle用户的主目录（如果未设置，则默认为$USER_HOME/.gradle）。

`JAVA_HOME`

指定要用于客户端VM的JDK安装目录。除非Gradle属性文件使用org.gradle.java.home指定了另一个虚拟机，否则此虚拟机也用于守护程序。

注意：命令行选项和系统属性优先于环境变量。

## Gradle属性

你可以通过以下方式自己配置你的项目属性，如果存在多个，则从上到下优先读取 ：

- 系统属性，例如在命令行上设置 `-Dgradle.user.home`
- GRADLE_USER_HOME目录中的`gradle.properties`
- 项目根目录中的`gradle.properties`
- Gradle安装目录中的`gradle.properties`

gradle.properties

```shell
# 当设置为true时，Gradle将在可能的情况下重用任何先前构建的任务输出，从而使构建速度更快
org.gradle.caching=true
 
# 设置为true时，单个输入属性哈希值和每个任务的构建缓存键都记录在控制台上。
org.gradle.caching.debug=true
 
# 启用按需孵化配置，Gradle将尝试仅配置必要的项目。
org.gradle.configureondemand=true
 
# 自定义控制台输出的颜色或详细程度。默认值取决于Gradle的调用方式。可选(auto,plain,rich,verbose)
org.gradle.console=auto
 
# 当设置true的Gradle守护进程来运行构建。默认值为true。
org.gradle.daemon=true
 
# 在指定的空闲毫秒数后，Gradle守护程序将自行终止。默认值为10800000（3小时）。
org.gradle.daemon.idletimeout=10800000
 
# 设置true为时，Gradle将在启用远程调试的情况下运行构建，侦听端口5005。
# 请注意，这等同于添加-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005到JVM命令行，并且将挂起虚拟机，直到连接了调试器。
# 默认值为false。
org.gradle.debug=true
 
# 指定用于Gradle构建过程的Java主页。可以将值设置为jdk或jre位置，但是，根据您的构建方式，使用JDK更安全。
# 如果未指定设置，则从您的环境（JAVA_HOME或的路径java）派生合理的默认值。这不会影响用于启动Gradle客户端VM的Java版本（请参阅环境变量）。
org.gradle.java.home=/usr/bin/java
        
# 指定用于Gradle守护程序的JVM参数。该设置对于配置JVM内存设置以提高构建性能特别有用。这不会影响Gradle客户端VM的JVM设置。
org.gradle.jvmargs=-Xmx2048m
 
# 当设置为quiet,warn,lifecycle,info,debug时，Gradle将使用此日志级别。这些值不区分大小写。该lifecycle级别是默认级别。
# 可选(quiet,warn,lifecycle,info,debug)
org.gradle.logging.level=debug
 
# 配置后，Gradle将分叉到org.gradle.workers.maxJVM以并行执行项目
org.gradle.parallel=true
        
# 指定Gradle守护程序及其启动的所有进程的调度优先级。默认值为normal。(low,normal)
org.gradle.priority=normal
        
# 在监视文件系统时配置详细日志记录。 默认为关闭 。
org.gradle.vfs.verbose=true
 
# 切换观看文件系统。允许Gradle在下一个版本中重用有关文件系统的信息。 默认为关闭 。
org.gradle.vfs.watch=true
 
# 当设置为all，summary或者none，Gradle会使用不同的预警类型的显示器。(all,fail,summary,none)
org.gradle.warning.mode=all
 
# 配置后，Gradle将最多使用给定数量的工人。默认值为CPU处理器数。
org.gradle.workers.max=5
```

### 系统属性

```shell
# 指定用户名以使用HTTP基本认证从服务器下载Gradle发行版
systemProp.gradle.wrapperUser = myuser
# 指定使用Gradle Wrapper下载Gradle发行版的密码
systemProp.gradle.wrapperPassword = mypassword
# 指定Gradle用户的主目录
systemProp.gradle.user.home=(path to directory)
```

### 项目属性

```shell
org.gradle.project.foo = bar
```

## 守护程序

Gradle在Java虚拟机（JVM）上运行，并使用一些支持库，这些库需要很短的初始化时间。但有时启动会比较慢。
解决此问题的方法是Gradle Daemon ：这是一个长期存在的后台进程，与以前相比，它可以更快地执行构建。

可通过命令获取运行守护程序状态

![](https://tianxiawuhao.github.io/post-images/1658230731202.png)

IDLE为空闲，BUSY为繁忙，STOPPED则已关闭

守护程序默认打开，可通过以下属性关闭
`.gradle/gradle.properties`

```shell
org.gradle.daemon=false
```

也可用命令gradle --stop手动关闭守护程序

# Gradle命令行

命令行格式

```shell
gradle [taskName ...] [--option-name ...]
```

如果指定了多个任务，则应以空格分隔。

选项和参数之间建议使用`=`来指定。

```shell
--console=plain
```

启用行为的选项具有长形式的选项，并带有由指定的反函数`--no-`。以下是相反的情况。

```shell
--build-cache
--no-build-cache
```

许多命令具有缩写。例如以下命令是等效的：

```shell
--help
-h
```

使用Wrapper时候应该用`./gradlew`或`gradlew.bat`取代`gradle`

```shell
#获取帮助
gradle -?
gradle -h
gradle -help
 
# 显示所选项目的子项目列表，以层次结构显示。
gradle projects
#查看可执行task
gradle task
#查看可执行task帮助
gradle help -task
 
# 在Gradle构建中，通常的`build`任务是指定组装所有输出并运行所有检查。
gradle build
# 执行所有验证任务（包括test和linting）。
gradle check
# 清理项目
gradle clean
#强制刷新依赖
gradle --refresh-dependencies assemble
 
#缩写调用
gradle startCmd
== 
gradle sc
 
 
# 执行任务
gradle myTask
# 执行多个任务
gradle myTask test
# 执行 dist任务但排除test任务
gradle dist --exclude-task test
# 强制执行任务
gradle test --rerun-tasks
 
# 持续构建
# gradle test --continue
 
# 生成扫描会提供完整的可视化报告，说明哪些依赖项存在于哪些配置，可传递依赖项和依赖项版本选择中。
$ gradle myTask --scan
 
# 所选项目的依赖项列表
$ gradle dependencies
```