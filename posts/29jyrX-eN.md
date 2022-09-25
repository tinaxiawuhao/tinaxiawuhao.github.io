---
title: 'gradle详情'
date: 2021-07-01 19:57:30
tags: [gradle]
published: true
hideInList: false
feature: /post-images/29jyrX-eN.png
isTop: false
---
### 一 依赖管理

implementation:会将指定的依赖添加到编译路径，并且会将该依赖打包到输出，但是这个依赖在编译时不能暴露给其他模块，例如依赖此模块的其他模块。这种方式指定的依赖在编译时只能在当前模块中访问。

api:使用api配置的依赖会将对应的依赖添加到编译路径，并将依赖打包输出，但是这个依赖是可以传递的，比如模块A依赖模块B，B依赖库C，模块B在编译时能够访问到库C，但是与implemetation不同的是，在模块A中库C也是可以访问的。

compileOnly:compileOnly修饰的依赖会添加到编译路径中，但是不会打包，因此只能在编译时访问，且compileOnly修饰的依赖不会传递。

runtimeOnly:这个与compileOnly相反，它修饰的依赖不会添加到编译路径中，但是被打包，运行时使用。没有使用过。

annotationProcessor:用于注解处理器的依赖配置

```java
dependencies {
    // 本地项目
    api project(':com.test.core:core-common')  
    //外部依赖
    implementation 'org.springframework.boot:spring-boot-starter'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### 二 仓库

**远程仓库**
使用Maven中央仓库，maven仓库的URL为：[http://repo1.maven.org/maven2/](https://links.jianshu.com/go?to=http%3A%2F%2Frepo1.maven.org%2Fmaven2%2F)

```java
repositories {
    mavenCentral()
}
```

使用Maven远程仓库

```java
repositories {
    //阿里云远程仓库
    maven { url 'https://maven.aliyun.com/repository/central'}
	maven { url 'https://maven.aliyun.com/repository/public' }
    mavenCentral()
}
```

**本地仓库**
配置Gradle使用maven本地仓库：`CRADLE_USER_HOME：D:.m2\repository`

![img](https://tianxiawuhao.github.io/post-images/1657022747903.png)

修改配置build.gradle：

```java
/**
 * 指定所使用的仓库，mavenCentral()表示使用中央仓库，
 * 此刻项目中所需要的jar包都会默认从中央仓库下载到本地指定目录
 * 配置mavenLocal()表示引入jar包的时候，先从本地仓库中找，没有再去中央仓库下载
 */
repositories {
    mavenLocal()
    mavenCentral()
}
```

**修改或者添加额外的私有仓库地址**
直接修改 settings.gradle 来添加其它仓库：

```java
// settings.gradle
//pluginManagement {}块只能出现在settings.gradle文件中，必须是文件中的第一个块，也可以以settings形式出现在初始化脚本中。
pluginManagement {
    repositories {
		maven { url 'https://maven.aliyun.com/repository/central'}
		maven { url 'https://maven.aliyun.com/repository/public' }
		mavenCentral()
		gradlePluginPortal()
		maven {
			url 'https://repo.spring.io/release'
		}
		if (version.endsWith('-SNAPSHOT')) {
			maven { url "https://repo.spring.io/snapshot" }
		}
	}
}
```

### 三 多项目构建

多项目构成：allProjects = root项目+各子项目
settings文件声明了所需的配置来实例化项目的层次结构。在默认情况下，这个文件被命名为settings.gradle，并且和根项目的build.gradle文件放在一起，该文件在初始化阶段被执行。根项目就像一个容器，子项目会迭代访问它的配置并注入到自己的配置中。

**多项目构建**
多项目构建总是需要指定一个树根，树中的每一个节点代表一个项目，每一个Project对象都指定有一个表示在树中位置的路径；在设置文件中我们还可以使用一套方法来自定义构建项目树。
settings.gradle作用就是用于多项目构建，一般像这样：

```java
//父模块名称
rootProject.name = 'nacos'
//引入子模块
include 'sentinel'
include 'openfeign'
include 'loadbalancer'
include 'gateway'
include 'rocketmq'
include 'sleuth'
include 'seata'
include 'provide'
findProject(':iot-service:user-center')?.name = 'user-center'
```

**buildScript**
buildScript块的repositories主要是为了Gradle脚本自身的执行，获取脚本依赖插件。
buildscript中的声明是gradle脚本自身需要使用的资源。可以声明的资源包括依赖项、第三方插件、maven仓库地址等。
gradle在执行脚本时，会优先执行buildscript代码块中的内容，然后才会执行剩余的build脚本。

```java
buildscript {
    ext {
        //spring-cloud-dependencies 2020.0.0 默认不在加载bootstrap 配置文件，
        //如果项目中要用bootstrap 配置文件 需要手动添加spring-cloud-starter-bootstrap 依赖，不然启动项目会报错的。
        set('springCloudVersion', "2021.0.3")
        //定义一个变量，统一规定springboot的版本
        set('springBootVersion', "2.6.8")
        set('alibabaVersion', "2.2.7.RELEASE")
        set('lombok', "1.18.16")
        set('knife4j', "2.0.9")
        set('hutool', "4.6.3")
        set('rocketmq', "2.2.2")
    }
    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }

    dependencies {
        //用来打包
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}
```

 ext：ext是自定义属性，现在很多人都喜欢把所有关于版本的信息都利用ext放在另一个自己新建的gradle文件中集中管理。

**allprojects**
allprojects块的repositories用于多项目构建，为所有项目提供共同所需依赖包。而子项目可以配置自己的repositories以获取自己独需的依赖包。

> buildscript和allprojects的作用和区别：
> buildscript中的声明是gradle脚本自身需要使用的资源，就是说它自己需要的资源，跟你其它模块其实并没有什么关系。而allprojects声明的却是`所有module`所需要使用的资源，就是说你的每个module都需要用同一个第三库的时候，可以在allprojects里面声明。

```java
allprojects {
    apply plugin: 'java-library'
    apply plugin: 'idea'

    //下面两句必须在所有子项目中添加，否则导致import了看起来没问题，但是编译时找不到其他模块的类。
    group = 'com.example'
    version = '0.0.1-SNAPSHOT'
    // 指定JDK版本
    sourceCompatibility = 1.8
    targetCompatibility = 1.8

    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }

    //指定编码格式
    tasks.withType(JavaCompile) {
        options.encoding = "UTF-8"
    }
}
```

**subprojects子项目通用配置**
subprojects是对所有Child Project的配置:

```java
subprojects {
    apply plugin: 'java-library'
    apply plugin: 'idea'
    apply plugin: 'org.springframework.boot'
    //dependency-management 插件
    apply plugin: 'io.spring.dependency-management'

    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }
    
    dependencies {
        implementation(enforcedPlatform("org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"))
        implementation(enforcedPlatform("com.alibaba.cloud:spring-cloud-alibaba-dependencies:${alibabaVersion}"))
        implementation(enforcedPlatform("org.springframework.boot:spring-boot-dependencies:${springBootVersion}"))
        implementation "com.github.xiaoymin:knife4j-spring-boot-starter:${knife4j}"
        implementation "cn.hutool:hutool-all:${hutool}"
        implementation 'org.springframework.boot:spring-boot-starter'
        implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-config'
        implementation('com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-discovery'){
            exclude group: 'org.springframework.cloud', module: 'spring-cloud-starter-netflix-ribbon'
        }
        implementation 'org.springframework.cloud:spring-cloud-starter-bootstrap'
        compileOnly("org.projectlombok:lombok:${lombok}")
        annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
        annotationProcessor("org.projectlombok:lombok:${lombok}")
        testImplementation("org.springframework.boot:spring-boot-starter-test:${springBootVersion}")
    }

    jar {
        manifest.attributes provider: 'gradle'
    }

}
```

### 四 gradle.properties

gradle中的常用属性可以写在gradle.properties中。
  一般我们都把全局属性都编写在一个工具类中，如果是有环境的切换的话，那么我们还会定义一个标志来进行相应的变换。对于项目而言，有时候需要配置某些敏感信息。比如密码，帐号等。而这些信息需要被很多类共同使用，所以必须有一个全局的配置。当需要把项目push到git上时，我们不希望别人看到我们项目的key，token等。我们可以将这些信息设置在gradle.properties中。
只有在Android中才可使用。

```java
AppKey = 1234567890
```

在build.gradle(module app)中进行变量的重定义，即将配置内容转化成java能够使用的形式:

```java
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            //buildConfigField用于给BuildConfig文件添加一个字段
            buildConfigField("String","KEY","\"${AppKey}\"")
        }
        debug{
            buildConfigField("String","KEY","\"${AppKey}\"")
        }
    }
}
```

### 五 Gradle 插件(Plugins)

Gradle 也可以用下面的方式声明使用的插件：

```java
// plugins DSL
plugins {
    id 'org.springframework.boot' version '2.2.1.RELEASE'
    id 'io.spring.dependency-management' version '1.0.8.RELEASE'
    id 'java'
}

// apply plugin
apply plugin: 'java-library'
apply plugin: 'idea'
apply plugin: 'org.springframework.boot'
//dependency-management 插件
apply plugin: 'io.spring.dependency-management'
```

其实是从 Gradle 官方的插件仓库[https://plugins.gradle.org/m2/](https://links.jianshu.com/go?to=https%3A%2F%2Fplugins.gradle.org%2Fm2%2F)下载的。

### 六 常见的task命令

`build`:当运行gradle build命令时Gradle将会编译和测试你的代码，并且创建一个包含类和资源的JAR文件。
`clean`:当运行gradle clean命令时Gradle将会删除build生成的目录和所有生成的文件。
`assemble`:当运行gradle assemble命令时Gradle将会编译并打包代码，但是并不运行单元测试。
`check`:当运行gradle check命令时Gradle将会编译并测试你的代码，其他的插件会加入更多的检查步骤。

### 七 gradle-wrapper

Wrapper是对Gradle的一层包装，便于在团队开发过程中统一Gradle构建的版本号，这样大家都可以使用统一的Gradle版本进行构建。

![img](https://tianxiawuhao.github.io/post-images/1657022568864.png)

```java
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-7.4.2-bin.zip
#distributionUrl=file:///E:/gradle/gradle-7.4.2-all.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

distributionUrl是要下载的gradle的地址，使用哪个版本的gradle，就在这里修改。

gradle的3种版本：
gradle-xx-all.zip是完整版，包含了各种二进制文件，源代码文件，和离线的文档。
gradle-xx-bin.zip是二进制版，只包含了二进制文件（可执行文件），没有文档和源代码。
gradle-xx-src.zip是源码版，只包含了Gradle源代码，不能用来编译你的工程。

zipStoreBase和zipStorePath组合在一起，是下载的gradle-3.1-bin.zip所存放的位置。
zipStorePath是zipStoreBase指定的目录下的子目录。

distributionBase和distributionPath组合在一起，是解压gradle-5.6.4-bin.zip之后的文件的存放位置。
distributionPath是distributionBase指定的目录下的子目录。

下载位置可以和解压位置不一样。

zipStoreBase和distributionBase有两种取值：GRADLE_USER_HOME和PROJECT。
其中，GRADLE_USER_HOME表示用户目录。
在windows下是%USERPROFILE%/.gradle，例如C:\Users<user_name>.gradle\。
在linux下是$HOME/.gradle，例如~/.gradle。
PROJECT表示工程的当前目录，即gradlew所在的目录。

### 八 依赖分组

Gradle 依赖是分组的 ,分组是在 org.gradle.api.Project 中的 configurations 中配置的 ,如 " implementation " , " compile " 等都是分组 , 这些分组都是在 org.gradle.api.Project#configurations 中进行配置 , 也就是 build.gradle#configurations 中配置 ;

**build.gradle#configurations 自定义依赖分组**

在 build.gradle 中配置 configurations :

```groovy
configurations {
  hello {
  }
}
```

则可以在 dependencies 中使上述在 configurations 配置的依赖分组 hello ,

```java
dependencies {
  hello 'com.android.support:appcompat-v7:28.0.0'
}
```

### 九 依赖版本冲突

Gradle对解决传递依赖提供了两种策略，使用最新版本或者直接导致构建失败。默认的策略是使用最新版本。虽然这样的策略能够解决一些问题，但是还是不够。常见的一种情况是，NoSuchMethond或者ClassNotFound。这时候，你可能需要一些特殊手段，比如排除不想要的传递依赖。
排除传递依赖的方式有两种：

> 1.使用transitive = false排除
>
> 2.在具体的某个dependency中使用exclude排除
>
> 3.使用force强制依赖某个版本

```java
(1) 方案一：针对 A 或 D 配置 transitive。
    这里针对A配置，不解析A模块的传递依赖，因此当前Module的依赖关系树中不再包含 B1 和 C，这里需要手动添加依赖 C
dependencies {
    implementation A {
      transitive = false
    }
    implementation C
    implementation D {
      //transitive = false
    }
}

(2) 方案二：针对 A 或 D 配置 exclude规则，此处针对A配置，依赖关系树中不再包含B1
dependencies {
    implementation A {
      exclude  B1
    }
    implementation D {
      //exclude  B2
    }
}

(3) 方案三：使用force强制依赖某个版本，如强制依赖 B1 或者 B2
        以下是在顶层build.gradle中配置，强制所有module依赖B1
configurations.all {
    resolutionStrategy {
       force B1
       // force B2
    }
}
```

### 十 springcloud-gradle管理

**parent:settings.gradle**

```groovy
rootProject.name = 'nacos'
include 'sentinel'
include 'openfeign'
include 'loadbalancer'
include 'gateway'
include 'rocketmq'
include 'sleuth'
include 'seata'
include 'provide'
```

**parent:build.gradle**

```groovy
buildscript {
    ext {
        //spring-cloud-dependencies 2020.0.0 默认不在加载bootstrap 配置文件，
        //如果项目中要用bootstrap 配置文件 需要手动添加spring-cloud-starter-bootstrap 依赖，不然启动项目会报错的。
        set('springCloudVersion', "2021.0.3")
        //定义一个变量，统一规定springboot的版本
        set('springBootVersion', "2.6.8")
        set('alibabaVersion', "2.2.7.RELEASE")
        set('lombok', "1.18.16")
        set('knife4j', "2.0.9")
        set('hutool', "4.6.3")
        set('rocketmq', "2.2.2")
    }
    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }

    dependencies {
        //用来打包
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

allprojects {
    apply plugin: 'java-library'
    apply plugin: 'idea'

    //下面两句必须在所有子项目中添加，否则导致import了看起来没问题，但是编译时找不到其他模块的类。
    group = 'com.example'
    version = '0.0.1-SNAPSHOT'
    // 指定JDK版本
    sourceCompatibility = 1.8
    targetCompatibility = 1.8

    repositories {
        mavenLocal()
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        mavenCentral()
    }

    //指定编码格式
    tasks.withType(JavaCompile) {
        options.encoding = "UTF-8"
    }
}


subprojects {
    apply plugin: 'java-library'
    apply plugin: 'idea'
    apply plugin: 'org.springframework.boot'
    //dependency-management 插件
    apply plugin: 'io.spring.dependency-management'

    dependencies {
        implementation(enforcedPlatform("org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"))
        implementation(enforcedPlatform("com.alibaba.cloud:spring-cloud-alibaba-dependencies:${alibabaVersion}"))
        implementation(enforcedPlatform("org.springframework.boot:spring-boot-dependencies:${springBootVersion}"))
        implementation "com.github.xiaoymin:knife4j-spring-boot-starter:${knife4j}"
        implementation "cn.hutool:hutool-all:${hutool}"
        implementation 'org.springframework.boot:spring-boot-starter'
        implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-config'
        implementation('com.alibaba.cloud:spring-cloud-starter-alibaba-nacos-discovery'){
            exclude group: 'org.springframework.cloud', module: 'spring-cloud-starter-netflix-ribbon'
        }
        implementation 'org.springframework.cloud:spring-cloud-starter-bootstrap'
        compileOnly("org.projectlombok:lombok:${lombok}")
        annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
        annotationProcessor("org.projectlombok:lombok:${lombok}")
        testImplementation("org.springframework.boot:spring-boot-starter-test:${springBootVersion}")
    }

    jar {
        manifest.attributes provider: 'gradle'
    }

}
```

**gateway:build.gradle**

```groovy
dependencies {
	implementation "org.springframework.cloud:spring-cloud-starter-gateway"
	implementation "org.springframework.cloud:spring-cloud-starter-loadbalancer"
}
```

**openfeign:build.gradle**

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation "org.springframework.cloud:spring-cloud-starter-openfeign"
	implementation "org.springframework.cloud:spring-cloud-starter-loadbalancer"
}

```

**rocketmq:build.gradle**

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation "org.apache.rocketmq:rocketmq-spring-boot-starter:${rocketmq}"
}
```

**sentinel:build.gradle**

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'com.alibaba.cloud:spring-cloud-starter-alibaba-sentinel'
	implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

### 项目构建--Gradle--Docker打包

在`build.gradle`中引入插件。

```java
 id "com.google.cloud.tools.jib" version "3.2.1"
```

![img](https://tianxiawuhao.github.io/post-images/1657022594521.png)

**配置**
配置打包时的基础镜像、容器配置、私服地址等，和Maven 插件中的一样，只是采用闭包的书写方式。

[Jib 官网文档](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#quickstart)

```java
jib {
    // 基础镜像，来自dockerhub,如果是私服，需要加上鉴权信息，和to下的auth节点相同
    // https://hub.docker.com/
    from {
        image = 'xx'
    }
    // 构建后的镜像名称以及私服地址、鉴权信息
    to {
        image = 'xx'
        auth {
            username = '登录账号'
            password = '登录密码'
        }
    }
    // 容器相关设置
    container {
        // 创建时间
        creationTime = new Date()
        // JVM 启动参数
        jvmFlags = ['-Djava.security.egd=file:/dev/./urandom', '-Dspring.profiles.active=prod', '-Dfile.encoding=utf-8', '-Duser.timezone=GMT+08']
        // 启动类
        // mainClass = 'com.xxx.RunApplication'
        // 容器在运行时公开的端口
        ports = ['8080']
        // 放置应用程序内容的容器上的根目录
        appRoot = '/deploy/service'
    }
}
```

#### 使用IdeaDocker插件布署

**Idea安装插件**

![img](https://tianxiawuhao.github.io/post-images/1657022607710.png)

![img](https://tianxiawuhao.github.io/post-images/1657022616011.png)

**安装docker插件**

1. 配置docker：

![img](https://tianxiawuhao.github.io/post-images/1657022629349.png)

![img](https://tianxiawuhao.github.io/post-images/1657022637131.png)

![img](https://tianxiawuhao.github.io/post-images/1657022644128.png)

**配置Dockerfile文件**
1）新建Dockerfile文件会自动弹出。

![img](https://tianxiawuhao.github.io/post-images/1657022651578.png)

在工程根目录下新建Dockerfile文件，内容如下：

```dockerfile
FROM openjdk:8
COPY build/libs/iot-eruake-1.0.0.jar app.jar
RUN bash -c "touch /app.jar"
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

**创建docker镜像**

![img](https://tianxiawuhao.github.io/post-images/1657022660226.png)

![img](https://tianxiawuhao.github.io/post-images/1657022667209.png)

![img](https://tianxiawuhao.github.io/post-images/1657022673583.png)

![img](https://tianxiawuhao.github.io/post-images/1657022680599.png)

**发布完成**

![img](https://tianxiawuhao.github.io/post-images/1657022687210.png)