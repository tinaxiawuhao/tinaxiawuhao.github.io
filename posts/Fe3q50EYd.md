---
title: 'gradle依赖,插件'
date: 2021-07-06 19:35:25
tags: [gradle]
published: true
hideInList: false
feature: /post-images/Fe3q50EYd.png
isTop: false
---


# 依赖

## configurations

设置configurations 配置依赖信息

build.gradle

```java
configurations {
    // 针对需要组件API的消费者的配置
    exposedApi {
        // canBeResolved 为true 则为可解析配置，为消费者
        canBeResolved = false
        // canBeConsumed 为true 则为消费析配置，为生产者
        canBeConsumed = true
    }
    // 为需要实现该组件的消费者提供的配置。
    exposedRuntime {
        canBeResolved = false
        canBeConsumed = true
    }
}
```

## 依赖方式

### 模块依赖

build.gradle

```java
dependencies {
    runtimeOnly group: 'org.springframework', name: 'spring-core', version: '2.5'
    runtimeOnly 'org.springframework:spring-core:2.5',
            'org.springframework:spring-aop:2.5'
    runtimeOnly(
        [group: 'org.springframework', name: 'spring-core', version: '2.5'],
        [group: 'org.springframework', name: 'spring-aop', version: '2.5']
    )
    runtimeOnly('org.hibernate:hibernate:3.0.5') {
        transitive = true
    }
    runtimeOnly group: 'org.hibernate', name: 'hibernate', version: '3.0.5', transitive: true
    runtimeOnly(group: 'org.hibernate', name: 'hibernate', version: '3.0.5') {
        transitive = true
    }
}
```

### 文件依赖

build.gradle

```java
dependencies {
    runtimeOnly files('libs/a.jar', 'libs/b.jar')
    runtimeOnly fileTree('libs') { include '*.jar' }
}
```

### 项目依赖

build.gradle

```java
dependencies {
    implementation project(':shared')
}
```

## 依赖方式

- compileOnly —用于编译生产代码所必需的依赖关系，但不应该属于运行时类路径的一部分
- implementation（取代compile）-用于编译和运行时
- runtimeOnly（取代runtime）-仅在运行时使用，不用于编译
- testCompileOnly—与compileOnly测试相同
- testImplementation —测试相当于 implementation
- testRuntimeOnly —测试相当于 runtimeOnly

## repositories

流行的公共存储库包括Maven Central， Bintray JCenter和Google Android存储库。
![](https://tianxiawuhao.github.io/post-images/1658230851306.png)

```java
repositories {
    
    mavenCentral() // Maven Central存储库  
    jcenter() // JCenter Maven存储库
    google() // Google Maven存储库
        
    mavenLocal()   // 将本地Maven缓存添加为存储库（不推荐）
    //flat存储库解析器
    flatDir {
        dirs 'lib'
    }
    flatDir {
        dirs 'lib1', 'lib2'
    }
    
    //添加定制的Maven仓库
    maven {
        url "http://repo.mycompany.com/maven2"
        // 为JAR文件添加附加的Maven存储库
        artifactUrls "http://repo.mycompany.com/jars"
    }
    
    //Ivy
     ivy {
        url "http://repo.mycompany.com/repo"
        layout "maven"  // 有效的命名布局值是'gradle'（默认值）'maven'和'ivy'。
    }
}
```

### 声明存储库过滤器

声明存储库内容

build.gradle

```java
repositories {
    maven {
        url "https://repo.mycompany.com/maven2"
        content {
            // this repository *only* contains artifacts with group "my.company"
            includeGroup "my.company"
        }
    }
    jcenter {
        content {
            // this repository contains everything BUT artifacts with group starting with "my.company"
            excludeGroupByRegex "my\\.company.*"
        }
    }
}
```

默认情况下，存储库包含所有内容，不包含任何内容：

- 如果声明include，那么它排除了一切 include 以外的内容。
- 如果声明exclude，则它将包括除exclude之外的所有内容。
- 如果声明include和exclude，则它仅包括显式包括但不排除的内容。

### 分割快照和发行版

build.gradle

```java
repositories {
    maven {
        url "https://repo.mycompany.com/releases"
        mavenContent {
            releasesOnly()
        }
    }
    maven {
        url "https://repo.mycompany.com/snapshots"
        mavenContent {
            snapshotsOnly()
        }
    }
}
```

### 支持的元数据源

受支持的元数据源

| 元数据源         | 描述                  | 排序 | Maven | Ivy/flat dir |
| :--------------- | :-------------------- | :--- | :---- | :----------- |
| gradleMetadata() | 寻找Gradle.module文件 | 1    | 是    | 是           |
| mavenPom()       | 查找Maven.pom文件     | 2    | 是    | 是           |
| ivyDescriptor()  | 查找ivy.xml文件       | 2    | 没有  | 是           |
| artifact()       | 直接寻找artifact      | 3    | 是    | 是           |

从Gradle 5.3开始，解析元数据文件（无论是Ivy还是Maven）时，Gradle将寻找一个标记，指示存在匹配的Gradle Module元数据文件。如果找到它，它将代替Ivy或Maven文件使用。

从Gradle5.6开始，您可以通过添加ignoreGradleMetadataRedirection()到metadataSources声明来禁用此行为。

例.不使用gradle元数据重定向的Maven存储库

build.gradle

```java
repositories {
    maven {
        url "http://repo.mycompany.com/repo"
        metadataSources {
            mavenPom()
            artifact()
            ignoreGradleMetadataRedirection()
        }
    }
}
```

## GAV坐标

GAV坐标一般指group，artifact，version

## 变体

构建变体是针对不同环境的配置，例如android开发中一般有debug和release两种变体
![](https://tianxiawuhao.github.io/post-images/1658230892932.png)

### 声明功能变体

可以通过应用`java`或`java-library`插件来声明功能变体。以下代码说明了如何声明名为`mongodbSupport`的功能：

示例1.声明一个功能变量

```java
Groovy``Kotlin
```

build.gradle

```java
group = 'org.gradle.demo'
version = '1.0'
 
java {
    registerFeature('mongodbSupport') {
        usingSourceSet(sourceSets.main)
    }
}
```

## 元数据

从存储库中提取的每个模块都有与之关联的元数据，例如其组，名称，版本以及它提供的带有工件和依赖项的不同变体

可配置组件元数据规则的示例

build.gradle

```java
class TargetJvmVersionRule implements ComponentMetadataRule {
    final Integer jvmVersion
    @Inject TargetJvmVersionRule(Integer jvmVersion) {
        this.jvmVersion = jvmVersion
    }
 
    @Inject ObjectFactory getObjects() { }
 
    void execute(ComponentMetadataContext context) {
        context.details.withVariant("compile") {
            attributes {
                attribute(TargetJvmVersion.TARGET_JVM_VERSION_ATTRIBUTE, jvmVersion)
                attribute(Usage.USAGE_ATTRIBUTE, objects.named(Usage, Usage.JAVA_API))
            }
        }
    }
}
dependencies {
    components {
        withModule("commons-io:commons-io", TargetJvmVersionRule) {
            params(7)
        }
        withModule("commons-collections:commons-collections", TargetJvmVersionRule) {
            params(8)
        }
    }
    implementation("commons-io:commons-io:2.6")
    implementation("commons-collections:commons-collections:3.2.2")
}
```

可以通过以下方法进行修改变体：

- allVariants：修改组件的所有变体
- withVariant(name)：修改由名称标识的单个变体
- addVariant(name)或addVariant(name, base)：从头开始 或通过 复制 现有变体的详细信息（基础）向组件添加新变体

可以调整每个变体的以下详细信息：

- 标识变体的属性-attributes {}块
- 该变体提供的功能-withCapabilities { }块
- 变体的依赖项，包括丰富的版本-withDependencies {}块
- 变体的依赖关系约束，包括丰富版本-withDependencyConstraints {}块
- 构成变体实际内容的已发布文件的位置-withFiles { }块

## 平台

### 使用平台

获取平台中声明的版本

build.gradle

```java
dependencies {
    // get recommended versions from the platform project
    api platform(project(':platform'))
    // no version required
    api 'commons-httpclient:commons-httpclient'
}
```

platform表示法是一种简写表示法，实际上在后台执行了一些操作：

- 它将org.gradle.category属性设置为platform，这意味着Gradle将选择依赖项的 平台 组件。
- 它默认设置endorseStrictVersions行为， 这意味着如果平台声明了严格的依赖关系，则将强制执行它们。

这意味着默认情况下，对平台的依赖项会触发该平台中定义的所有严格版本的继承， 这对于平台作者确保所有使用者在依赖项的版本方面都遵循自己的决定很有用。 可以通过显式调用doNotEndorseStrictVersions方法来将其关闭。

例.依靠一个BOM导入其依赖约束

build.gradle

```java
dependencies {
    // import a BOM
    implementation platform('org.springframework.boot:spring-boot-dependencies:1.5.8.RELEASE')
 
    // define dependencies without versions
    implementation 'com.google.code.gson:gson'
    implementation 'dom4j:dom4j'
}
```

导入BOM，确保其定义的版本覆盖找到的任何其他版本

build.gradle

```java
dependencies {
    // import a BOM. The versions used in this file will override any other version found in the graph
    implementation enforcedPlatform('org.springframework.boot:spring-boot-dependencies:1.5.8.RELEASE')
 
    // define dependencies without versions
    implementation 'com.google.code.gson:gson'
    implementation 'dom4j:dom4j'
 
    // this version will be overridden by the one found in the BOM
    implementation 'org.codehaus.groovy:groovy:1.8.6'
}
```

## Capability

### 声明组件的capability

build.gradle

```java
configurations {
    apiElements {
        outgoing {
            capability("com.acme:my-library:1.0")
            capability("com.other:module:1.1")
        }
    }
    runtimeElements {
        outgoing {
            capability("com.acme:my-library:1.0")
            capability("com.other:module:1.1")
        }
    }
}
```

### 解决冲突

按Capability（能力）解决冲突，（若存在相同能力的依赖性会失败）

build.gradle

```java
@CompileStatic
class AsmCapability implements ComponentMetadataRule {
    void execute(ComponentMetadataContext context) {
        context.details.with {
            if (id.group == "asm" && id.name == "asm") {
                allVariants {
                    it.withCapabilities {
                        // Declare that ASM provides the org.ow2.asm:asm capability, but with an older version
                        it.addCapability("org.ow2.asm", "asm", id.version)
                    }
                }
            }
        }
    }
}
 
```

一个带有日志框架隐式冲突的构建文件

build.gradle

```java
dependencies {
    // Activate the "LoggingCapability" rule
    components.all(LoggingCapability)
}
 
@CompileStatic
class LoggingCapability implements ComponentMetadataRule {
    final static Set<String> LOGGING_MODULES = ["log4j", "log4j-over-slf4j"] as Set<String>
 
    void execute(ComponentMetadataContext context) {
        context.details.with {
            if (LOGGING_MODULES.contains(id.name)) {
                allVariants {
                    it.withCapabilities {
                        // Declare that both log4j and log4j-over-slf4j provide the same capability
                        it.addCapability("log4j", "log4j", id.version)
                    }
                }
            }
        }
    }
}
```

## 版本

### 版本规则

Gradle支持不同的版本字符串声明方式：

- 一个确切的版本：比如`1.3`，`1.3.0-beta3`，`1.0-20150201.131010-1`

- 一个Maven风格的版本范围：例如

  ```java
  [1.0,)
  ```

  ```java
  [1.1, 2.0)
  ```

  ```java
  (1.2, 1.5]
  ```

  - `[`和`]`的符号表示包含性约束; `(`和`)`表示排他性约束。
  - 当上界或下界缺失时，该范围没有上界或下界。
  - 符号`]`可以被用来代替`(`用于排他性下界，`[`代替`)`用于排他性上界。例如`]1.0, 2.0[`

- 前缀版本范围：例如

  ```java
  1.+
  ```

  ```java
  1.3.+
  ```

  - 仅包含与`+`之前部分完全匹配的版本。
  - `+`本身的范围将包括任何版本。

- 一个latest-status版本：例如latest.integration，latest.release

- Maven的SNAPSHOT版本标识符：例如1.0-SNAPSHOT，1.4.9-beta1-SNAPSHOT

### 版本排序

- 每个版本均分为其组成的“部分”：
  - 字符[. - _ +]用于分隔版本的不同“部分”。
  - 同时包含数字和字母的任何部分都将分为以下各个部分： `1a1 == 1.a.1`
  - 仅比较版本的各个部分。实际的分隔符并不重要：`1.a.1 == 1-a+1 == 1.a-1 == 1a1`
- 使用以下规则比较2个版本的等效部分：
  - 如果两个部分都是数字，则最高数字值 较高 ：`1.1<1.2`
  - 如果一个部分是数值，则认为它 高于 非数字部分：`1.a<1.1`
  - 如果两个部分都不是数字，则按字母顺序比较，区分大小写：`1.A< 1.B< 1.a<1.b`
  - 有额外数字部分的版本被认为比没有数字部分的版本高：`1.1<1.1.0`
  - 带有额外的非数字部分的版本被认为比没有数字部分的版本低：`1.1.a<1.1`
- 某些字符串值出于排序目的具有特殊含义：
  - 字符串dev被认为比任何其他字符串部分低：`1.0-dev< 1.0-alpha< 1.0-rc`。
  - 字符串rc、release和final被认为比任何其他字符串部分都高（按顺序排列：`1.0-zeta< 1.0-rc< 1.0-release< 1.0-final< 1.0`。
  - 字符串SNAPSHOT没有特殊意义，和其他字符串部分一样按字母顺序排序：`1.0-alpha< 1.0-SNAPSHOT< 1.0-zeta< 1.0-rc< 1.0`。
  - 数值快照版本没有特殊意义，和其他数值部分一样进行排序：`1.0< 1.0-20150201.121010-123< 1.1`。

简单来说:数字>final>release>rc>字母>dev

### 声明没有版本的依赖

对于较大的项目，建议的做法是声明没有版本的依赖项， 并将依赖项约束 用于版本声明。 优势在于，依赖关系约束使您可以在一处管理所有依赖关系的版本，包括可传递的依赖关系。

build.gradle

```java
dependencies {
    implementation 'org.springframework:spring-web'
}
 
dependencies {
    constraints {
        implementation 'org.springframework:spring-web:5.0.2.RELEASE'
    }
}
```

### 依赖方式

#### strictly

> 与该版本符号不匹配的任何版本将被排除。这是最强的版本声明。
> 在声明的依赖项上，strictly可以降级版本。
> 在传递依赖项上，如果无法选择此子句可接受的版本，将导致依赖项解析失败。
> 有关详细信息，请参见覆盖依赖项版本。
> 该术语支持动态版本。

定义后，将覆盖先前的require声明并清除之前的 reject。

#### require

> 表示所选版本不能低于require可接受的版本，但可以通过冲突解决方案提高，即使更高版本具有排他性更高的界限。
> 这就是依赖项上的直接版本所转换的内容。该术语支持动态版本。

定义后，将覆盖先前的strictly声明并清除之前的 reject。

#### prefer

> 这是一个非常软的版本声明。仅当对该模块的版本没有更强的非动态观点时，才适用。
> 该术语不支持动态版本。

定义可以补充strictly或require。

在级别层次结构之外还有一个附加术语：

#### reject

> 声明模块不接受特定版本。如果唯一的可选版本也被拒绝，这将导致依赖项解析失败。该术语支持动态版本。

### 动态版本

build.gradle

```java
plugins {
    id 'java-library'
}
 
repositories {
    mavenCentral()
}
 
dependencies {
    implementation 'org.springframework:spring-web:5.+'
}
```

### 版本快照

声明一个版本变化的依赖

build.gradle

```java
 
plugins {
    id 'java-library'
}
 
repositories {
    mavenCentral()
    maven {
        url 'https://repo.spring.io/snapshot/'
    }
}
 
dependencies {
    implementation 'org.springframework:spring-web:5.0.3.BUILD-SNAPSHOT'
}
 
```

### 以编程方式控制依赖项缓存

您可以使用ResolutionStrategy 对配置进行编程来微调缓存的某些方面。 如果您想永久更改设置，则编程方式非常有用。

默认情况下，Gradle将动态版本缓存24小时。 要更改Gradle将解析后的版本缓存为动态版本的时间，请使用：

例.动态版本缓存控制

build.gradle

```java
configurations.all {
    resolutionStrategy.cacheDynamicVersionsFor 10, 'minutes'
}
 
```

默认情况下，Gradle会将更改的模块缓存24小时。 要更改Gradle将为更改的模块缓存元数据和工件的时间，请使用：

例.改变模块缓存控制

build.gradle

```java
configurations.all {
    resolutionStrategy.cacheChangingModulesFor 4, 'hours'
}
```

### 锁定配置

锁定特定配置

build.gradle

```java
configurations {
    compileClasspath {
        resolutionStrategy.activateDependencyLocking()
    }
}
```

锁定所有配置

build.gradle

```java
dependencyLocking {
    lockAllConfigurations()
}
```

解锁特定配置

build.gradle

```java
configurations {
    compileClasspath {
        resolutionStrategy.deactivateDependencyLocking()
    }
}
```

#### 锁定buildscript类路径配置

如果将插件应用于构建，则可能还需要利用依赖锁定。为了锁定用于脚本插件的classpath配置，请执行以下操作：

build.gradle

```java
buildscript {
    configurations.classpath {
        resolutionStrategy.activateDependencyLocking()
    }
}
 
```

#### 使用锁定模式微调依赖项锁定行为

虽然默认锁定模式的行为如上所述，但是还有其他两种模式可用：

- Strict模式 ：在该模式下，除了上述验证外，如果被标记为锁定的配置没有与之相关联的锁定状态，则依赖性锁定将失败。
- Lenient模式：在这种模式下，依存关系锁定仍将固定动态版本，但除此之外，依赖解析的变化不再是错误。

锁定模式可以从dependencyLocking块中进行控制，如下所示：

build.gradle

```java
dependencyLocking {
    lockMode = LockMode.STRICT
}
```

### 版本冲突

用force强制执行一个依赖版本

build.gradle

```java
dependencies {
    implementation 'org.apache.httpcomponents:httpclient:4.5.4'
    implementation('commons-codec:commons-codec:1.9') {
        force = true
    }
}
```

排除特定依赖声明的传递依赖

build.gradle

```java
dependencies {
    implementation('commons-beanutils:commons-beanutils:1.9.4') {
        exclude group: 'commons-collections', module: 'commons-collections'
    }
}
```

版本冲突时失败

build.gradle

```java
configurations.all {
    resolutionStrategy {
        failOnVersionConflict()
    }
}
 
```

使用动态版本时失败

build.gradle

```java
configurations.all {
    resolutionStrategy {
        failOnDynamicVersions()
    }
}
```

改变版本时失败

build.gradle

```java
configurations.all {
    resolutionStrategy {
        failOnChangingVersions()
    }
}
```

解析无法再现时失败

build.gradle

```java
configurations.all {
    resolutionStrategy {
        failOnNonReproducibleResolution()
    }
}
 
```

# 插件

插件作用：将插件应用于项目可以使插件扩展项目的功能。它可以执行以下操作：

- 扩展Gradle模型（例如，添加可以配置的新DSL元素）
- 根据约定配置项目（例如，添加新任务或配置合理的默认值）
- 应用特定的配置（例如，添加组织存储库或强制执行标准）

简单来说，插件可以拓展项目功能，如任务，依赖，拓展属性，约束

## 插件类型

- 二进制插件 ：通过实现插件接口以编程方式编写二进制插件，或使用Gradle的一种DSL语言以声明方式编写二进制插件
- 脚本插件 ：脚本插件是其他构建脚本，可以进一步配置构建，并通常采用声明式方法来操纵构建

插件通常起初是脚本插件（因为它们易于编写），然后，随着代码变得更有价值，它被迁移到可以轻松测试并在多个项目或组织之间共享的二进制插件。

## 应用插件

### 二进制插件

实现了org.gradle.api.Plugin接口

```java
apply plugin: 'com.android.application'
```

#### apply plugin

```java
apply plugin: 'java'  //id
==
apply plugin: org.gradle.api.plugins.JavaPlugin //类型
==
apply plugin: JavaPlugin          //org.gradle.api.plugins默认导入
```

#### plugins DSL

```java
plugins {
    id 'java' //应用核心插件
    id 'com.jfrog.bintray' version '0.4.1' //应用社区插件
    id 'com.example.hello' version '1.0.0' apply false //使用`apply false`语法告诉Gradle不要将插件应用于当前项目
}
```

### 脚本插件

脚本插件会自动解析，可以从本地文件系统或远程位置的脚本中应用。可以将多个脚本插件（任意一种形式）应用于给定目标。

```java
apply from:'version.gradle'
```

apply可传入内容

```java
void apply(Map<String,? options);
void apply(Closure closure);
void apply(Action<? super ObjectConfigurationAction> action);
```

## 定义插件

定义一个带有ID的buildSrc插件
buildSrc / build.gradle

```java
plugins {
    id 'java-gradle-plugin'
}
gradlePlugin {
    plugins {
        myPlugins {
            id = 'my-plugin'
            implementationClass = 'my.MyPlugin'
        }
    }
}
```

## 第三方插件

通过将插件添加到构建脚本classpath中，然后应用该插件，可以将已发布为外部jar文件的二进制插件添加到项目中。可以使用buildscript {}块将外部jar添加到构建脚本classpath中。

```java
buildscript {
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:4.0.1"
    }
}
```

## 插件管理

pluginManagement {}块只能出现在settings.gradle文件中，必须是文件中的第一个块，也可以以settings形式出现在初始化脚本中。

settings.gradle

```java
pluginManagement {
    plugins {
    }
    resolutionStrategy {
    }
    repositories {
    }
}
rootProject.name = 'plugin-management'
```

init.gradle

```java
settingsEvaluated { settings ->
    settings.pluginManagement {
        plugins {
        }
        resolutionStrategy {
        }
        repositories {
        }
    }
}
```

例：通过pluginManagement管理插件版本。

settings.gradle

```java
pluginManagement {
  plugins {
        id 'com.example.hello' version "${helloPluginVersion}"
  }
}
 
```

gradle.properties

```java
helloPluginVersion=1.0.0
```

## 自定义插件存储库

要指定自定义插件存储库，请使用repositories {}块其中的pluginManagement {}：

settings.gradle

```java
pluginManagement {
    repositories {
        maven {
            url '../maven-repo'
        }
        gradlePluginPortal()
        ivy {
            url '../ivy-repo'
        }
    }
}
```

# java库

导入java

```java
apply plugin:'java'
```

自定义路径

```java
sourceSets {
    main {
         java {
            srcDirs = ['src']
         }
    }
 
    test {
        java {
            srcDirs = ['test']
        }
    }
}
sourceSets {
    main {
        java {
            srcDir 'thirdParty/src/main/java'
        }
    }
}
```

导入依赖

```java
 repositories {
        jcenter()
 }
dependencies {
     implementation group:'com.android.support',name:'appcompat-v7',version:'28.0.0'
     implementation 'com.android.support:appcompat-v7:28.0.0'
     implementation protect(':p')
     implementation file('libs/ss.jar','libs/ss2.jar')
         
    implementation fileTree(dir: "libs", include: ["*.jar"])
}
```

多项目 设置 settings.gradle

```java
include ':app'
rootProject.name = "GradleTest"
```

# 安卓实用

## 设置签名

```java
android  {
    signingConfig = {
        release {
            storeFile file("MYKEY.keystore")
            storePassword "storePassword"
            keyAlias "keyAlias"
            keyPassword "keyPassword"
        }
    }
}
```

## 自定义输出apk文件名称

```java
applicationVariants.all { variant ->
       variant.outputs.all { output ->
           def fileName = "自定义名称_${variant.versionName}_release.apk"
           def outFile = output.outputFile
           if (outFile != null && outFile.name.endsWith('.apk')) {
               outputFileName = fileName
           }
       }
   }
```

## 动态AndroidManifest

```java
<meta-data android:name="paramName" android:value="${PARAM_NAME}">
android {
    productFlavors{
        google{
            manifestPlaceholders.put("PARAM_NAME",'google')
        }
    }
}
```

## 多渠道

```java
android {
    productFlavors{
        google{
            
        },
        baidu{
            
        }
    }
    productFlavors.all{ flavor->
        manifestPlaceholders.put("PARAM_NAME",name)
    }
}
```

## adb设置

adb工具

```java
android {
    adbOptions{
        timeOutInMs = 5000 //5s超时
        installOptions '-r','-s' //安装指令
    }
}
```

## dexOptions

dex工具

```java
android {
    dexOptions{
        incremental true //增量
        javaMaxHeapSize '4G'//dx最大队内存
        jumboMode true //强制开启jumbo跳过65535限制
        preDexLibraries true //提高增量构建速度
            threadCount 1 //dx线程数量
    }
}
```

## Ant

例.将嵌套元素传递给Ant任

build.gradle

```java
task zip {
    doLast {
        ant.zip(destfile: 'archive.zip') {
            fileset(dir: 'src') {
                include(name: '**.xml')
                exclude(name: '**.java')
            }
        }
    }
}
```

例.使用Ant类型

build.gradle

```java
task list {
    doLast {
        def path = ant.path {
            fileset(dir: 'libs', includes: '*.jar')
        }
        path.list().each {
            println it
        }
    }
}
```

例.使用定制的Ant任务

build.gradle

```java
task check {
    doLast {
        ant.taskdef(resource: 'checkstyletask.properties') {
            classpath {
                fileset(dir: 'libs', includes: '*.jar')
            }
        }
        ant.checkstyle(config: 'checkstyle.xml') {
            fileset(dir: 'src')
        }
    }
}
```

## Lint

Lint：android tool目录下的工具，一个代码扫描工具，能够帮助我们识别资源、代码结构存在的问题

```java
lintOptions
android {
    lintOptions{
        abortOnError true //发生错误时推出Gradle
        absolutePaths true //配置错误输出是否显示绝对路径
        check 'NewApi','InlinedApi' // 检查lint check的issue id            
        enable 'NewApi','InlinedApi' //启动 lint check的issue id
        disable 'NewApi','InlinedApi' //关闭 lint check的issue id
        checkAllWarnings true //检查所有警告issue
        ignoreWarning true //忽略警告检查，默认false
        checkReleaseBuilds true //检查致命错误，默认true
        explainIssues true //错误报告是否包含解释说明，默认true
        htmlOutput new File("/xx.html") //html报告输出路径
        htmlReport true // 是否生成html报告，默认true
        lintConfig new File("/xx.xml") //lint配置
        noLines true // 输出不带行号 默认true
        quite true // 安静模式
        showAll true //是否显示所有输出，不截断
    }
}
```