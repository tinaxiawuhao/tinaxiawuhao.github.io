---
title: 'Thingsboard源码编译'
date: 2021-12-02 15:49:30
tags: [Thingsboard]
published: true
hideInList: false
feature: /post-images/DseKIIHw3.png
isTop: false
---
## 环境安装

开发环境要求： Jdk 1.11 版本 Postgresql 9 以上 Node.js Npm Maven 3.6 以上 Git 工具 Idea 开发工具 Redis

### JDK

**下载安装**

JDK 官方下载地址： [Java Downloads | Oracle](https://www.oracle.com/java/technologies/downloads/#java11-windows)

JDK 版本选择 JDK11，我本地环境是 Windos10 64 位，所以选择 jdk-11.0.13-windows-x64.exe

![](https://tinaxiawuhao.github.io/post-images/1638431548398.png)
下载好了之后直接默认安装就行 

**免安装版本**

下载jdk11
http://openjdk.java.net/install/index.html

这个页面大部分都是linux系统的；

![](https://tinaxiawuhao.github.io/post-images/1638431591806.png)

然后我们点击jdk.java.net ，接着我们选择下载Java SE 11；

![](https://tinaxiawuhao.github.io/post-images/1638431626019.png)

![](https://tinaxiawuhao.github.io/post-images/1638431650013.png)


下好了后，我们得到这样的文件：openjdk-11+28_windows-x64_bin.zip，解压后得到：jdk-11这样的文件夹；将该文件夹放到你习惯的地方；

**配置环境变量**

**步骤 1：** 在 JAVA_HOME 中增加 JDK 的安装地址：C:\Program Files\Java\jdk1.8.0_221 ![](https://tinaxiawuhao.github.io/post-images/1638431720261.png)

**步骤 2：** 在 CLASSPATH 中增加 JDK 的安装地址中的文件：.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar ![](https://tinaxiawuhao.github.io/post-images/1638431747039.png)

**步骤 3：** 在 Path 中增加 JDK 的地址：%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin; ![](https://tinaxiawuhao.github.io/post-images/1638431804325.png)

**步骤 4** 输入以下命令

```shell
java -version
```

如果能出现以下的提示信息，就算安装成功了![](https://tinaxiawuhao.github.io/post-images/1638431854451.png)

### 安装 IDEA

参考：[IDEA 安装教程](https://www.iotschool.com/topics/72)

### 安装 Maven

步骤 1：下载 maven，进入地址：http://maven.apache.org/download.cgi ![](https://tinaxiawuhao.github.io/post-images/1638431903988.png)

步骤 2：下载到本地 ![](https://tinaxiawuhao.github.io/post-images/1638431943007.png)

步骤 3：配置环境变量 增加 MAVEN_HOME，即 maven 的地址：D:\tb\apache-maven-3.6.1-bin，请注意，如果直接解压，有可能会有两个 apache-maven-3.6.1-bin![](https://tinaxiawuhao.github.io/post-images/1638432216067.png)

![](https://tinaxiawuhao.github.io/post-images/1638432310973.png)

MAVEN_OPTS，参数是 -Xms128m -Xmx1024m![](https://tinaxiawuhao.github.io/post-images/1638432342300.png)

修改 Path，增加 Maven 的地址%MAVEN_HOME%\bin; ![](https://tinaxiawuhao.github.io/post-images/1638432408010.png)

测试 Maven 安装，打开命令行工具。使用命令 mvn -v，如果能出现以下提示，即安装成功 ![](https://tinaxiawuhao.github.io/post-images/1638432460487.png)

### Nodejs 安装

步骤 1：下载 Nodejs 安装包，Nodejs 官网地址：https://nodejs.org/en/download/ ![](https://tinaxiawuhao.github.io/post-images/1638432643509.png)

步骤 2：安装完成后，使用命令查看 Nodejs 是否已经安装完成，能出现以下提示说明已经安装成功 !![](https://tinaxiawuhao.github.io/post-images/1638432779272.png)

### 安装 git

步骤 1：下载 git 安装包，git 官网地址是：https://git-scm.com/download/win![](https://tinaxiawuhao.github.io/post-images/1638433242163.png)

步骤 2：安装完成后，使用命令行测试 git ![](https://tinaxiawuhao.github.io/post-images/1638433294783.png)

### 安装 npm 全局依赖

步骤 1：使用管理员 CMD 命令行，执行下面命令

```shell
#npm 环境读取环境变量包
npm install -g cross-env
#webpack打包工具
npm install -g webpack
```

![](https://tinaxiawuhao.github.io/post-images/1638433336020.png)

### 安装 redis

Redis 安装参考：https://www.iotschool.com/wiki/redis

环境安装到此结束，接下来是通过 Git 拉取代码。

## 克隆 thingsboard 代码

### 确定代码存放位置

在本地创建代码存放位置的文件目录，然后进入当前目录点击鼠标右键，选择 Git Bash Here ![](https://tinaxiawuhao.github.io/post-images/1638433373069.png)

### 输入 git 命令克隆源代码

```shell
git clone https://github.com/thingsboard/thingsboard.git
```

![](https://tinaxiawuhao.github.io/post-images/1638433403642.png)

耐心等待一段时间后，看到以下界面就算下载成功 ![](https://tinaxiawuhao.github.io/post-images/1638433434959.png)

### 切换 git 分支

默认下载的代码是 master 主分支的，我们开发需要切换到最新版本的分支。

查看项目源码的所有分支，下载源码后，需要进入到 thingsboard 文件夹 ![](https://tinaxiawuhao.github.io/post-images/1638433484980.png)

发现最新发布的版本是 2.4，所以我这里选择 2.4，当然你可以根据自己的情况进行分支选择

输入命令以下，即可切换至 2.4 的分支

```shell
git checkout release-2.4
```

看到下图这样，即切换成成功 ![](https://tinaxiawuhao.github.io/post-images/1638433523862.png)
## 准备工作

### 外网连接

因为 TB 在编译过程中需要依赖很多国外的包，那么需要外网才能连接，有连接外网支持，可以到社区求助：https://www.iotschool.com/topics/node8

### 设置 Maven 为淘宝镜像

工程是基于 Maven 管理，直接通过 idea open，之后会自动下载各种依赖包。依赖包的默认存储地址为：C:\Users\用户名.m2\repository，内容如下：

```shell
$tree ~/.m2 -L 2
/home/jay/.m2
└── repository
    ├── antlr
    ├── aopalliance
    ├── asm
    ├── backport-util-concurrent
    ├── ch
    ...
```

一般情况下，使用官方镜像更新依赖包，网速不稳定，可将 Maven 镜像源设置为淘宝的，在 maven 安装包目录下找到 settings.xml 设置

大概位置截图：

![](https://tinaxiawuhao.github.io/post-images/1638433563017.png)

把 settings.xml 里面内容设置成以下：

```xml
<mirrors>
  <mirror>
       <!--This sends everything else to /public -->
       <id>aliyun_nexus</id>
       <mirrorOf>*,!maven_nexus_201</mirrorOf> 
       <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
   </mirror>
</mirrors>
```

不会设置的，可以参考这个文件：https://cdn.iotschool.com/iotschool/settings.xml

thingsboard QQ 群也有这个资源：121202538

### 设置 npm 为淘宝镜像

同上，网速不好 npm 过程中也会下载失败，这是导致很多同学 thingsboard 编译失败的主要原因，所以我们在进行编译之前，也将 npm 替换为淘宝镜像：

```shell
npm install -g mirror-config-china --registry=http://registry.npm.taobao.org        #使用淘宝镜像
npm config get registry                                                             #查询当前镜像
npm config rm registry                                                              #删除自定义镜像，使用官方镜像
npm info express
```

### 设置 IDEA 管理员启动

我本地开发环境编译项目使用 IDEA 工具进行编译，所以需要设置管理员启动，这样才有所有的权限执行编译命令。 步骤 1：点击 IDEA 图标右键，选择属性。![](https://tinaxiawuhao.github.io/post-images/1638433591413.png)

步骤 2：点击兼容性 - 更改所有用户设置 - 以管理员身份运行此程序![](https://tinaxiawuhao.github.io/post-images/1638433662301.png)

## 开始编译

编译项目跟网速有关，最好连接上外网进行编译，一般 5~30 分钟都有可能，超过 30 分钟要检查你的网络。

### 清理项目编译文件

使用 IDEA Maven 工具进行清理 ![](https://tinaxiawuhao.github.io/post-images/1638433705438.png)

### 输入编译命令开始编译

在 IDEA 控制台（左下方）Terminal 输入以下命令进行编译：

```shell
mvn clean install -DskipTests
```

![](https://tinaxiawuhao.github.io/post-images/1638433732682.png)

等一段时间后，看到下面这张图就算编译成功，如果没有编译成功请按照本教程最后的常见问题进行排查，一般都是网络问题。如果还有问题，请到社区[thingsboard 专题](https://www.iotschool.com/topics/node8)中提问。

![](https://tinaxiawuhao.github.io/post-images/1638433762968.png)

## 常见问题

### pom包pkg.name等标签未定义

```xml
    <properties>
        <pkg.name>thingsboard</pkg.name>
        <main.dir>${basedir}</main.dir>
        <pkg.type>java</pkg.type>
        <pkg.mainClass>org.thingsboard.server.ThingsboardServerApplication</pkg.mainClass>
        <pkg.copyInstallScripts>true</pkg.copyInstallScripts>


        <main.dir>${basedir}</main.dir>
        <pkg.disabled>true</pkg.disabled>
        <pkg.process-resources.phase>none</pkg.process-resources.phase>
        <pkg.package.phase>none</pkg.package.phase>
        <pkg.user>thingsboard</pkg.user>
        <pkg.implementationTitle>${project.name}</pkg.implementationTitle>
        <pkg.unixLogFolder>/var/log/${pkg.name}</pkg.unixLogFolder>
        <pkg.installFolder>/usr/share/${pkg.name}</pkg.installFolder>
  </properties>
```





### 缓存导致编译失败

每次编译失败进行二次编译时，要清理缓存，并杀死遗留进程 步骤 1：执行下面命令，杀死遗留进程

```shell
taskkill /f /im java.exe
```

步骤 2：使用 IDEA Maven 工具进行清理 ![](https://tinaxiawuhao.github.io/post-images/1638433793132.png)

**温馨提示：要进行二次编译前，最好重启一次电脑！**

### Server UI 编译失败

```java
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.0:npm (npm install) on project ui: Failed to run task: 'npm install' failed. (error code 1) -> [Help 1]
```

![](https://tinaxiawuhao.github.io/post-images/1638433824944.png)

如果遇到这个问题，可从以下几个原因进行分析：

#### 原因 1：node、npm 版本号问题

本地环境安装的 node、npm 版本号与源码中 pom.xml 文件配置的版本号不一致。

解决方案： 步骤 1：使用 node -v、npm -v 查看安装的 node 和 npm 版本号 ![](https://tinaxiawuhao.github.io/post-images/1638433858043.png)

步骤 2：修改源码中 pom.xml 文件中的版本号

```xml
<configuration>
   <nodeVersion>v12.13.1</nodeVersion>
   <npmVersion>6.12.1</npmVersion>
</configuration>
```

需要修改的文件有三处，位置如下： ![](https://tinaxiawuhao.github.io/post-images/1638433897767.png)

#### 原因 2：node-sass 下载失败

编译 Server UI 时，会下载 node-sass 依赖，如果因为网络原因没有下载成功，也会编译失败。如果你是按照本本教材一步一步来的，应该不会有问题，上面准备工作中，将 npm 镜像源切换为淘宝，那么下载会很快的。

```shell
[INFO] Downloading binary from https://github.com/sass/node-sass/releases/download/v4.12.0/win32-x64-72_binding.node
[ERROR] Cannot download "https://github.com/sass/node-sass/releases/download/v4.12.0/win32-x64-72_binding.node":
[ERROR]
[ERROR] ESOCKETTIMEDOUT
[ERROR]
[ERROR] Hint: If github.com is not accessible in your location
[ERROR]       try setting a proxy via HTTP_PROXY, e.g.
[ERROR]
[ERROR]       export HTTP_PROXY=http://example.com:1234
[ERROR]
[ERROR] or configure npm proxy via
[ERROR]
[ERROR]       npm config set proxy http://example.com:8080
[INFO]
[INFO] > node-sass@4.12.0 postinstall F:\workspace\thingsboard\thingsboard\ui\node_modules\node-sass
[INFO] > node scripts/build.js
[INFO]
```

![](https://tinaxiawuhao.github.io/post-images/1638433935418.png)

解决方案：[切换镜像源为淘宝](https://www.iotschool.com/wiki/tbinstall#设置npm为淘宝镜像)

解决方案：重启电脑，清理缓存

#### 原因 3：Thingsboard 3.0 版本编译遇到的问题

亲测：2.4 版本也可以通过这种方式来解决

```shell
Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.7.5:npm (npm install) on project ui-ngx: Failed to run task: 'npm install' failed. org.apache.commons.exec.ExecuteException: Process exited with an error: -4048 (Exit value: -4048) -> [Help 1]
```

解决方案：https://www.iotschool.com/topics/84

#### 原因 4：二次编译导致残留进程

报错：

```shell
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-clean-plugin:2.5:clean (default-clean) on project ui: Failed to clean project: Failed to delete F:\workspace\thingsboard\thingsboard\ui\target\node\node.exe -> [Help 1]
```

![](https://tinaxiawuhao.github.io/post-images/1638433961674.png)

### Server Tool 编译失败

![](https://tinaxiawuhao.github.io/post-images/1638433985854.png)

```shell
[ERROR] Failed to execute goal on project tools: Could not resolve dependencies for project org.thingsboard:tools:jar:2.4.3: Failed to collect dependencies at org.eclipse.paho:org.eclipse.paho.client.mqttv3:jar:1.1.0: Failed to read artifact descriptor for org.eclipse.paho:org.eclipse.paho.clien
t.mqttv3:jar:1.1.0: Could not transfer artifact org.eclipse.paho:org.eclipse.paho.client.mqttv3:pom:1.1.0 from/to aliyun_nexus (http://maven.aliyun.com/nexus/content/groups/public/): Failed to transfer file http://maven.aliyun.com/nexus/content/groups/public/org/eclipse/paho/org.eclipse.paho.cli
ent.mqttv3/1.1.0/org.eclipse.paho.client.mqttv3-1.1.0.pom with status code 502 -> [Help 1]
```

一般由于网络原因，IoTSchool 小编至少编译了 3 次才成功，每次编译都重启电脑，并清理环境。

解决方案：如果使用的是 mvn clean install -DskipTests 命令进行编译，那么就多尝试几次，每次编译前，要清理环境。

参考：https://github.com/thingsboard/performance-tests/issues/10

### JavaScript Executor 编译失败

JavaScript Executor Microservice 编译失败 ![](https://tinaxiawuhao.github.io/post-images/1638434020950.png)

```shell
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:1.0:npm (npm install) on project js-executor: Failed to run task: 'npm install' failed. (error code 2) -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoFailureException
[ERROR]
[ERROR] After correcting the problems, you can resume the build with the command
[ERROR]   mvn <goals> -rf :js-executor
```

原因：本地缓存缺少 fetched-v10.15.3-linux-x64 和 fetched-v10.15.3-win-x64 这两个文件。

解决方案： 步骤 1：下载这两个文件到本地，下载后记得重命名，下载地址：https://github.com/zeit/pkg-fetch/releases ![](https://tinaxiawuhao.github.io/post-images/1638434130104.png)

步骤 2: 将下载的两个文件放到：放到：C:\Users\你的用户名 \ .pkg-cache\v2.6。并将名字分别修改为：fetched-v10.15.3-linux-x64 和 fetched-v10.15.3-win-x64

参考：https://github.com/thingsboard/thingsboard/issues/2084

### License 检查不通过

```shell
[ERROR] Failed to execute goal com.mycila:license-maven-plugin:3.0:check (default) on project thingsboard: Some files do not have the expected license header -> [Help 1]
```

解决方案：在根目录 pom.xml 中屏蔽 license-maven-plugin

![](https://tinaxiawuhao.github.io/post-images/1638434154303.png)

搜索 license-maven-plugin，将整个 plugin 都注释掉 ![](https://tinaxiawuhao.github.io/post-images/1638434302255.png)

### Web UI 编译失败

Web UI 编译失败请参考[Server UI 编译失败第一个原因](https://www.iotschool.com/wiki/tbinstall#Server Tool编译失败)

### maven:Could not resolve dependencies for project org.thingsboard:application:

错误信息

```shell
[ERROR] Failed to execute goal on project application: Could not resolve dependencies for project org.thingsboard:application:jar:2.4.1: The following artifacts could not be resolved: org.thingsboard.rule-engine:rule-engine-components:jar:2.4.1, org.thingsboard:dao:jar:2.4.1: Could not find artifact org.thingsboard.rule-engine:rule-engine-components:jar:2.4.1 in jenkins (http://repo.jenkins-ci.org/releases) -> [Help 1]
```

解决方案：根目录下去 maven 编译，不要在每个单独编译，否则不能自动解决依赖，如果你已经在子模块进行了编译，请回到根目录先 clean 一下，再重新编译。

### maven:Failed to delete tb-http-transport.rpm

错误信息：

```shell
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-clean-plugin:2.5:clean (default-clean) on project http: Failed to clean project: Failed to delete D:\my_project\thingsboard\transport\http\target\tb-http-transport.rpm -> [Help 1]
```

解决方案：第一次编译失败，再次编译可能会提示该错误，可以手动到报错路径删除，如果提示文件正在使用，需要在任务管理器杀死 java 进程后再手动删除。

### npm:npm:cb() never called!

错误信息：

```shell
npm ERR! cb() never called!
npm ERR! This is an error with npm itself. Please report this error at:
npm ERR!     <https://npm.community>
npm ERR! A complete log of this run can be found in:
npm ERR!     C:\Users\yuren\AppData\Roaming\npm-cache\_logs\2019-11-06T10_55_28_258Z-debug.log
```

解决方案： 尝试 npm cache clean --force 后再次 npm install 无果； 尝试更换淘宝镜像源后再次 npm install 无果； 怀疑有些包下载需要翻墙，全局代理翻墙后问题依然存在； 参考网上关闭所有代理后问题依然存在； 通过 log 日志分析最后一个解包报错的地方，屏蔽需要的 material-design-icons，新 modules rxjs 仍然报错；

```shell
extract material-design-icons@3.0.1 extracted to node_modules\.staging\material-design-icons-61b4d55e (72881ms)
extract rxjs@6.5.2 extracted to node_modules\.staging\rxjs-e901ba4c (24280ms)
```

参考 npm ERR cb() never called 执行

```shell
npm install --no-package-lock
```

之后提示 npm ERR! path git，添加 git 到环境变量后正常。

### npm:npm ERR! path git

错误信息

```shell
npm ERR! path git
npm ERR! code ENOENT
npm ERR! errno ENOENT
npm ERR! syscall spawn git
npm ERR! enoent Error while executing:
npm ERR! enoent undefined ls-remote -h -t git://github.com/fabiobiondi/angular-
```

解决方案：添加 git 到环境变量。

### No compiler is provided in this environment

错误信息：

```shell
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.
1:compile (default-compile) on project netty-mqtt: Compilation failure
[ERROR] No compiler is provided in this environment. Perhaps you are running on
a JRE rather than a JDK?
```

需要在环境变量中设置 java，包含%JAVA_HOME%bin;%JAVA_HOME%lib;



### Failed to execute goal org.thingsboard:gradle-maven-plugin:1.0.10:invoke (default) on project http: org.gradle.tooling.BuildException: Could not execute build using Gradle distribution 'https://services.gradle.org/distributions/gradle-6.3-bin.zip'.



### Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.8.1:compile (default-compile) on project rest-client: Compilation failure

An unknown compilation problem occurred

这个问题主要是jdk版本跟项目不一致导致的，如果项目的版本是大于(不含！)3.2.1，则需要JDK11，反之JDK8

