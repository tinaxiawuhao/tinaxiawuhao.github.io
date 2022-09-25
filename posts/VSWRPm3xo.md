---
title: 'Docker '
date: 2022-01-13 10:37:47
tags: [docker]
published: true
hideInList: false
feature: /post-images/VSWRPm3xo.png
isTop: false
---
# Docker  

学习目标：

- 掌握Docker基础知识，能够理解Docker镜像与容器的概念

- 完成Docker安装与启动

- 掌握Docker镜像与容器相关命令

- 掌握Tomcat Nginx 等软件的常用应用的安装

- 掌握docker迁移与备份相关命令

- 能够运用Dockerfile编写创建容器的脚本

- 能够搭建与使用docker私有仓库

  


# 1 Docker简介

## 1.1 什么是虚拟化

​	在计算机中，虚拟化（英语：Virtualization）是一种资源管理技术，是将计算机的各种实体资源，如服务器、网络、内存及存储等，予以抽象、转换后呈现出来，打破实体结构间的不可切割的障碍，使用户可以比原本的组态更好的方式来应用这些资源。这些资源的新虚拟部份是不受现有资源的架设方式，地域或物理组态所限制。一般所指的虚拟化资源包括计算能力和资料存储。

​	在实际的生产环境中，虚拟化技术主要用来解决高性能的物理硬件产能过剩和老的旧的硬件产能过低的重组重用，透明化底层物理硬件，从而最大化的利用物理硬件   对资源充分利用

​	虚拟化技术种类很多，例如：软件虚拟化、硬件虚拟化、内存虚拟化、网络虚拟化(vip)、桌面虚拟化、服务虚拟化、虚拟机等等。

## 1.2 什么是Docker

​	Docker 是一个开源项目，诞生于 2013 年初，最初是 dotCloud 公司内部的一个业余项目。它基于 Google 公司推出的 Go 语言实现。 项目后来加入了 Linux 基金会，遵从了 Apache 2.0 协议，项目代码在 [GitHub](https://github.com/docker/docker) 上进行维护。

​	![](https://tinaxiawuhao.github.io/post-images/1641955175988.png)



​	Docker 自开源后受到广泛的关注和讨论，以至于 dotCloud 公司后来都改名为 Docker Inc。Redhat 已经在其 RHEL6.5 中集中支持 Docker；Google 也在其 PaaS 产品中广泛应用。

​	Docker 项目的目标是实现轻量级的操作系统虚拟化解决方案。 Docker 的基础是 Linux 容器（LXC）等技术。

​	在 LXC 的基础上 Docker 进行了进一步的封装，让用户不需要去关心容器的管理，使得操作更为简便。用户操作 Docker 的容器就像操作一个快速轻量级的虚拟机一样简单。

为什么选择Docker?

（1）上手快。

​	用户只需要几分钟，就可以把自己的程序“Docker化”。Docker依赖于“写时复制”（copy-on-write）模型，使修改应用程序也非常迅速，可以说达到“随心所致，代码即改”的境界。	

         随后，就可以创建容器来运行应用程序了。大多数Docker容器只需要不到1秒中即可启动。由于去除了管理程序的开销，Docker容器拥有很高的性能，同时同一台宿主机中也可以运行更多的容器，使用户尽可能的充分利用系统资源。

（2）职责的逻辑分类

​	使用Docker，开发人员只需要关心容器中运行的应用程序，而运维人员只需要关心如何管理容器。Docker设计的目的就是要加强开发人员写代码的开发环境与应用程序要部署的生产环境一致性。从而降低那种“开发时一切正常，肯定是运维的问题（测试环境都是正常的，上线后出了问题就归结为肯定是运维的问题）”

（3）快速高效的开发生命周期

​	Docker的目标之一就是缩短代码从开发、测试到部署、上线运行的周期，让你的应用程序具备可移植性，易于构建，并易于协作。（通俗一点说，Docker就像一个盒子，里面可以装很多物件，如果需要这些物件的可以直接将该大盒子拿走，而不需要从该盒子中一件件的取。）

（4）鼓励使用面向服务的架构

​	Docker还鼓励面向服务的体系结构和微服务架构。Docker推荐单个容器只运行一个应用程序或进程，这样就形成了一个分布式的应用程序模型，在这种模型下，应用程序或者服务都可以表示为一系列内部互联的容器，从而使分布式部署应用程序，扩展或调试应用程序都变得非常简单，同时也提高了程序的内省性。（当然，可以在一个容器中运行多个应用程序）

## 1.3 容器与虚拟机比较

​	下面的图片比较了 Docker 和传统虚拟化方式的不同之处，可见容器是在操作系统层面上实现虚拟化，直接复用本地主机的操作系统，而传统方式则是在硬件层面实现。

![](https://tinaxiawuhao.github.io/post-images/1641955189369.png)

与传统的虚拟机相比，Docker优势体现为启动速度快、占用体积小。

## 1.4 Docker 组件

### 1.4.1 Docker服务器与客户端

​	Docker是一个客户端-服务器（C/S）架构程序。Docker客户端只需要向Docker服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。Docker提供了一个命令行工具Docker以及一整套RESTful API。你可以在同一台宿主机上运行Docker守护进程和客户端，也可以从本地的Docker客户端连接到运行在另一台宿主机上的远程Docker守护进程。

![](https://tinaxiawuhao.github.io/post-images/1641955202177.png)

![](https://tinaxiawuhao.github.io/post-images/1645597883819.png)

### 1.4.2 Docker镜像与容器

​	镜像是构建Docker的基石。用户基于镜像来运行自己的容器。镜像也是Docker生命周期中的“构建”部分。镜像是基于联合文件系统的一种层式结构，由一系列指令一步一步构建出来。例如：

添加一个文件；

执行一个命令；

打开一个窗口。

也可以将镜像当作容器的“源代码”。镜像体积很小，非常“便携”，易于分享、存储和更新。

​	Docker可以帮助你构建和部署容器，你只需要把自己的应用程序或者服务打包放进容器即可。容器是基于镜像启动起来的，容器中可以运行一个或多个进程。我们可以认为，镜像是Docker生命周期中的构建或者打包阶段，而容器则是启动或者执行阶段。  容器基于镜像启动，一旦容器启动完成后，我们就可以登录到容器中安装自己需要的软件或者服务。

所以Docker容器就是：

​	一个镜像格式；

​	一些列标准操作；

​	一个执行环境。

​	Docker借鉴了标准集装箱的概念。标准集装箱将货物运往世界各地，Docker将这个模型运用到自己的设计中，唯一不同的是：集装箱运输货物，而Docker运输软件。

         和集装箱一样，Docker在执行上述操作时，并不关心容器中到底装了什么，它不管是web服务器，还是数据库，或者是应用程序服务器什么的。所有的容器都按照相同的方式将内容“装载”进去。

         Docker也不关心你要把容器运到何方：我们可以在自己的笔记本中构建容器，上传到Registry，然后下载到一个物理的或者虚拟的服务器来测试，在把容器部署到具体的主机中。像标准集装箱一样，Docker容器方便替换，可以叠加，易于分发，并且尽量通用。

### 1.4.3 Registry（注册中心）

​	Docker用Registry来保存用户构建的镜像。Registry分为公共和私有两种。Docker公司运营公共的Registry叫做Docker Hub。用户可以在Docker Hub注册账号，分享并保存自己的镜像（说明：在Docker Hub下载镜像巨慢，可以自己构建私有的Registry）。

​	https://hub.docker.com/

# 2 Docker安装与启动

## 2.1 安装Docker 

​	Docker官方建议在Ubuntu中安装，因为Docker是基于Ubuntu发布的，而且一般Docker出现的问题Ubuntu是最先更新或者打补丁的。在很多版本的CentOS中是不支持更新最新的一些补丁包的。

​	由于我们学习的环境都使用的是CentOS，因此这里我们将Docker安装到CentOS上。注意：这里建议安装在CentOS7.x以上的版本，在CentOS6.x的版本中，安装前需要安装其他很多的环境而且Docker很多补丁不支持更新。

​	请直接挂载课程配套的Centos7.x镜像	

（1）yum 包更新到最新

```shell
sudo yum update
```

（2）安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

（3）设置yum源为阿里云

```shell
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

（4）安装docker

```shell
sudo yum install docker-ce
```

（5）安装后查看docker版本

```shell
docker -v
```

## 2.2 设置ustc的镜像 

ustc是老牌的linux镜像服务提供者了，还在遥远的ubuntu 5.04版本的时候就在用。ustc的docker镜像加速器速度很快。ustc docker mirror的优势之一就是不需要注册，是真正的公共服务。

[https://lug.ustc.edu.cn/wiki/mirrors/help/docker](https://lug.ustc.edu.cn/wiki/mirrors/help/docker)

编辑该文件：

```shell
vi /etc/docker/daemon.json  
```

在该文件中输入如下内容：

```shell
{
"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

## 2.3 Docker的启动与停止

**systemctl**命令是系统服务管理器指令

启动docker：

```shell
systemctl start docker
```

停止docker：

```shell
systemctl stop docker
```

重启docker：

```shell
systemctl restart docker
```

查看docker状态：

```shell
systemctl status docker
```

开机启动：

```shell
systemctl enable docker
```

查看docker概要信息

```shell
docker info
```

查看docker帮助文档

```shell
docker --help
```

# 3 常用命令

![](https://tinaxiawuhao.github.io/post-images/1645597935806.png)

## 3.1 镜像相关命令

### 3.1.1 查看镜像

```shell
docker images
```

REPOSITORY：镜像名称

TAG：镜像标签

IMAGE ID：镜像ID

CREATED：镜像的创建日期（不是获取该镜像的日期）

SIZE：镜像大小

这些镜像都是存储在Docker宿主机的/var/lib/docker目录下

### 3.1.2 搜索镜像

如果你需要从网络中查找需要的镜像，可以通过以下命令搜索

```shell
docker search 镜像名称
```

NAME：仓库名称

DESCRIPTION：镜像描述

STARS：用户评价，反应一个镜像的受欢迎程度

OFFICIAL：是否官方

AUTOMATED：自动构建，表示该镜像由Docker Hub自动构建流程创建的

### 3.1.3 拉取镜像

拉取镜像就是从中央仓库中下载镜像到本地

```shell
docker pull 镜像名称
```

例如，我要下载centos7镜像

```shell
docker pull centos:7
```

Docker采用联合文件系统，不同镜像的相同文件无需再次下载：
![](https://tinaxiawuhao.github.io/post-images/1645597963050.png)

### 4 删除镜像

按镜像ID删除镜像

```shell
docker rmi 镜像ID
```

删除所有镜像

```shell
docker rmi `docker images -q`
```

## 3.2 容器相关命令

### 3.2.1 查看容器

查看正在运行的容器

```shell
docker ps
```

查看所有容器

```shell
docker ps –a
```

查看最后一次运行的容器

```shell
docker ps –l
```

查看停止的容器

```shell
docker ps -f status=exited
```

### 3.2.2 创建与启动容器

创建容器常用的参数说明：

创建容器命令：docker run

 -i：表示运行容器

 -t：表示容器启动后会进入其命令行。加入这两个参数后，容器创建就能登录进去。即分配一个伪终端。

 --name :为创建的容器命名。

 -v：表示目录映射关系（前者是宿主机目录，后者是映射到宿主机上的目录），可以使用多个－v做多个目录或文件映射。注意：最好做目录映射，在宿主机上做修改，然后共享到容器上。

 -d：在run后面加上-d参数,则会创建一个守护式容器在后台运行（这样创建容器后不会自动登录容器，如果只加-i -t两个参数，创建后就会自动进去容器）。

 -p：表示端口映射，前者是宿主机端口，后者是容器内的映射端口。可以使用多个-p做多个端口映射

（1）交互式方式创建容器

```shell
docker run -it --name=容器名称 镜像名称:标签 /bin/bash
```

这时我们通过ps命令查看，发现可以看到启动的容器，状态为启动状态  

退出当前容器

```shell
exit
```

（2）守护式方式创建容器：

```shell
docker run -di --name=容器名称 镜像名称:标签
```

登录守护式容器方式：

```shell
docker exec -it 容器名称 (或者容器ID)  /bin/bash
```

![](https://tinaxiawuhao.github.io/post-images/1645598588933.jpg)

```bash
Exit                         # 从容器中退回主机 
CTRL+Q+P                     # 容器不停止退出
```

### 3.2.3 停止与启动容器

停止容器：

```shell
docker stop 容器名称（或者容器ID）
```

启动容器：

```shell
docker start 容器名称（或者容器ID）
```

### 3.2.4 文件拷贝

如果我们需要将文件拷贝到容器内可以使用cp命令

```shell
docker cp 需要拷贝的文件或目录 容器名称:容器目录
```

也可以将文件从容器内拷贝出来

```shell
docker cp 容器名称:容器目录 需要拷贝的文件或目录
```

### 3.2.5 目录挂载

我们可以在创建容器的时候，将宿主机的目录与容器内的目录进行映射，这样我们就可以通过修改宿主机某个目录的文件从而去影响容器。
创建容器 添加-v参数 后边为   宿主机目录:容器目录，例如：

```shell
docker run -di -v /usr/local/myhtml:/usr/local/myhtml --name=mycentos3 centos:7
```

如果你共享的是多级的目录，可能会出现权限不足的提示。

这是因为CentOS7中的安全模块selinux把权限禁掉了，我们需要添加参数  --privileged=true  来解决挂载的目录没有权限的问题

#### 匿名挂载

```bash
docker run -d  -v 容器内目录  镜像名/id  # 匿名挂载
```

匿名挂载后，使用**docker volume ls**命令查看所有挂载的卷：

![](https://tinaxiawuhao.github.io/post-images/1645598000069.png)

每一个VOLUME NAME对应一个挂载的卷，由于挂载时未指定主机目录，因此无法直接找到目录。

#### 具名挂载

```bash
docker run -d  -v 卷名：容器内目录  镜像名/id  # 具名挂载
```

![](https://tinaxiawuhao.github.io/post-images/1645598609101.png)
可以发现挂载的卷：volume01，并通过**docker volume inspect 卷名** 命令找到主机内目录：
![](https://tinaxiawuhao.github.io/post-images/1645598023382.png)

所有docker容器内的卷，在未指定主机内目录时，都在：*/var/lib/docker/volumes/卷名/_data* 下，可通过具名挂载可以方便的找到卷，因此广泛使用这种方式进行挂载。

#### 数据卷容器

![](https://tinaxiawuhao.github.io/post-images/1645598045974.png)

```bash
docker run -it --name container02 --volumes from container01 镜像名/id  # 将两个容器进行挂载
```

### 3.2.6 查看容器IP地址

我们可以通过以下命令查看容器运行的各种数据

```shell
docker inspect 容器名称（容器ID） 
```

也可以直接执行下面的命令直接输出IP地址

```shell
docker inspect --format='{{.NetworkSettings.IPAddress}}' 容器名称（容器ID）
```

### 3.2.7 删除容器 

删除指定的容器：

```shell
docker rm 容器名称（容器ID）
```


### 3.2.8 其他命令

```bash
docker start/restart/stop/kill 容器名/id               
docker logs -tf --tail 显示的日志条数 容器名/id  # 查看日志
docker top 容器名/id                 # 查看容器中的进程信息
docker inspect 容器名/id             # 查看镜像的元数据
docker exec -it 容器名/id /bin/bash  # 通常容器以后台方式运行，需要进入其中修改配置：进入容器后开启一个新终端         
docker attach 容器名/id              # 进入容器正在执行的终端
docker cp 容器名/id:容器内路径 主机文件路径       # 从容器内拷贝文件到主机上
```

# 4  Docker镜像详解

![](https://tinaxiawuhao.github.io/post-images/1645598065040.png)

## UnionFS（联合文件系统）

- 联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。
- 特性：一次同时加载多个文件系统，但从外面看起来只能看到一个文件系统。联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

## 镜像加载原理

Docker的镜像实际由一层一层的文件系统组成：

- bootfs（boot file system）主要包含bootloader和kernel。bootloader主要是引导加载kernel，完成后整个内核就都在内存中了。此时内存的使用权已由bootfs转交给内核，系统卸载bootfs。可以被不同的Linux发行版公用。
- rootfs（root file system），包含典型Linux系统中的/dev，/proc，/bin，/etc等标准目录和文件。rootfs就是各种不同操作系统发行版（Ubuntu，Centos等）。因为底层直接用Host的kernel，rootfs只包含最基本的命令，工具和程序就可以了。
- 分层理解
  所有的Docker镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的容器层。
  容器在启动时会在镜像最外层上建立一层可读写的容器层（R/W），而镜像层是只读的（R/O）。
  ![](https://tinaxiawuhao.github.io/post-images/1645598082318.png)

```bash
docker commit -m="描述信息" -a="作者" 容器id 目标镜像名:[tag]  # 编辑容器后提交容器成为一个新镜像
```

# 4 应用部署

## 4.1 MySQL部署

（1）拉取mysql镜像

```shell
docker pull centos/mysql-57-centos7
```

（2）创建容器

```shell
docker run -di --name=tensquare_mysql -p 33306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
```

-p 代表端口映射，格式为  宿主机映射端口:容器运行端口

-e 代表添加环境变量  MYSQL_ROOT_PASSWORD  是root用户的登陆密码

（3）远程登录mysql

连接宿主机的IP  ,指定端口为33306 

 ## 4.2 tomcat部署

（1）拉取镜像

```shell
docker pull tomcat:7-jre7
```

（2）创建容器

创建容器  -p表示地址映射

```shell
docker run -di --name=mytomcat -p 9000:8080 
-v /usr/local/webapps:/usr/local/tomcat/webapps tomcat:7-jre7
```

## 4.3 Nginx部署 

（1）拉取镜像	

```shell
docker pull nginx
```

（2）创建Nginx容器

```shell
docker run -di --name=mynginx -p 80:80 nginx
```

## 4.4 Redis部署

（1）拉取镜像

```shell
docker pull redis
```

（2）创建容器

```shell
docker run -di --name=myredis -p 6379:6379 redis
```

## 4.5 Redis集群部署

```shell
# 创建网卡
docker network create redis --subnet 172.38.0.0/16
# 通过脚本创建六个redis配置
for port in $(seq 1 6);\
do \
mkdir -p /mydata/redis/node-${port}/conf
touch /mydata/redis/node-${port}/conf/redis.conf
cat << EOF >> /mydata/redis/node-${port}/conf/redis.conf
port 6379
bind 0.0.0.0
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.38.0.1${port}
cluster-announce-port 6379
cluster-announce-bus-port 16379
appendonly yes
EOF
done

# 通过脚本运行六个redis
for port in $(seq 1 6);\
docker run -p 637${port}:6379 -p 1667${port}:16379 --name redis-${port} \
-v /mydata/redis/node-${port}/data:/data \
-v /mydata/redis/node-${port}/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 172.38.0.1${port} redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
docker exec -it redis-1 /bin/sh #redis默认没有bash
redis-cli --cluster create 172.38.0.11:6379 172.38.0.12:6379 172.38.0.13:6379 172.38.0.14:6379 172.38.0.15:6379 172.38.0.16:6379  --cluster-replicas 1
```

# 5 迁移与备份

## 5.1 容器保存为镜像

我们可以通过以下命令将容器保存为镜像

```shell
docker commit mynginx mynginx_i
```

## 5.2 镜像备份

我们可以通过以下命令将镜像保存为tar 文件

```shell
docker  save -o mynginx.tar mynginx_i
```

## 5.3 镜像恢复与迁移

首先我们先删除掉mynginx_img镜像  然后执行此命令进行恢复

```shell
docker load -i mynginx.tar
```

-i 输入的文件

执行后再次查看镜像，可以看到镜像已经恢复

# 6 Dockerfile

## 6.1 什么是Dockerfile

Dockerfile是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。

1、对于开发人员：可以为开发团队提供一个完全一致的开发环境； 
2、对于测试人员：可以直接拿开发时所构建的镜像或者通过Dockerfile文件构建一个新的镜像开始工作了； 
3、对于运维人员：在部署时，可以实现应用的无缝移植。

## 6.2 常用命令



| 命令                                 | 作用                                 |
| ---------------------------------- | ---------------------------------- |
| FROM image_name:tag                | 定义了使用哪个基础镜像启动构建流程                  |
| MAINTAINER user_name               | 声明镜像的创建者                           |
| ENV key value                      | 设置环境变量 (可以写多条)                     |
| RUN command                        | 是Dockerfile的核心部分(可以写多条)            |
| ADD source_dir/file dest_dir/file  | 将宿主机的文件复制到容器内，如果是一个压缩文件，将会在复制后自动解压 |
| COPY source_dir/file dest_dir/file | 和ADD相似，但是如果有压缩文件并不能解压              |
| WORKDIR path_dir                   | 设置工作目录                             |

## 6.3 使用脚本创建镜像

步骤：

（1）创建目录

```shell
mkdir –p /usr/local/dockerjdk8
```

（2）下载jdk-8u171-linux-x64.tar.gz并上传到服务器（虚拟机）中的/usr/local/dockerjdk8目录

（3）创建文件Dockerfile  `vi Dockerfile`

```shell
#依赖镜像名称和ID
FROM centos:7
#指定镜像创建者信息
MAINTAINER ITCAST
#切换工作目录
WORKDIR /usr
RUN mkdir  /usr/local/java
#ADD 是相对路径jar,把java添加到容器中
ADD jdk-8u171-linux-x64.tar.gz /usr/local/java/

#配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_171
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH
```

（4）执行命令构建镜像

```shell
docker build -t='jdk1.8' .
```

注意后边的空格和点，不要省略

（5）查看镜像是否建立完成

```shell
docker images
```



# 7 Docker私有仓库

## 7.1 私有仓库搭建与配置

（1）拉取私有仓库镜像（此步省略）

```shell
docker pull registry
```

（2）启动私有仓库容器

```shell
docker run -di --name=registry -p 5000:5000 registry
```

（3）打开浏览器 输入地址http://192.168.184.141:5000/v2/_catalog看到`{"repositories":[]}` 表示私有仓库搭建成功并且内容为空

（4）修改daemon.json

```shell
vi /etc/docker/daemon.json
```

添加以下内容，保存退出。

```json
{"insecure-registries":["192.168.184.141:5000"]} 
```

此步用于让 docker信任私有仓库地址

（5）重启docker 服务

```shell
systemctl restart docker
```

## 7.2 镜像上传至私有仓库

（1）标记此镜像为私有仓库的镜像

```shell
docker tag jdk1.8 192.168.184.141:5000/jdk1.8
```

（2）再次启动私服容器

```shell
docker start registry
```

（3）上传标记的镜像

```shell
docker push 192.168.184.141:5000/jdk1.8
```

# 8 Docker网络

## 8.1 理解Doker0

通过命令**ip addr**查看本地ip地址，我们发现除了本机回环地址和埃里远的内网地址外，还多了一个网卡：Docker0，这是Docker服务启动后自动生成的。
![](https://tinaxiawuhao.github.io/post-images/1645598117584.png)
而如果进入一个正在后台运行的tomcat容器，同样使用**ip addr**命令，发现容器得到了一个新的网络：**12: eth@if13**，ip地址：**172.17.0.2**。这是Docker在容器启动时为其分配的。
![](https://tinaxiawuhao.github.io/post-images/1645598157386.png)
思考一个问题：此时我们的linux主机可以ping通容器内部（**172.17.0.2**）吗？（**注意与容器暴露端口相区分**)
![](https://tinaxiawuhao.github.io/post-images/1645598173437.png)

- linux可以ping通docker容器内部，因为docker0的ip地址为**172.17.0.1**，容器为**172.17.0.2**。
- 原理：我们每启动一个docker容器，docker就会给容器分配一个默认的可用ip，我们只要安装了docker，就会有一个网卡docker0(bridge)。网卡采用桥接模式，并使用veth-pair技术（veth-pair就是一堆虚拟设备接口，成对出现，一段连着协议，一段彼此相连，充当一个桥梁。）。
- 这时我们退出容器，回到主机再次观察主机的ip地址：
  ![](https://tinaxiawuhao.github.io/post-images/1645598197831.png)
  我们惊奇地发现了一个新网络**13: vethda1df4b@if12**，对应容器内网络地址的**12: eth@if13**。
- 容器和容器之间是可以互相ping通的：容器1→Docker0→容器2
  ![](https://tinaxiawuhao.github.io/post-images/1645598226949.png)
  docker中的所有网络接口都是虚拟的 ，转发效率高。删除容器后，对应的网桥也随之删除。

## 8.2 --link
若编写一个微服务并连接数据库，如果数据库ip改变，如何根据容器名而不是ip访问容器？显然，直接使用容器名是无法ping通容器内部的：
![](https://tinaxiawuhao.github.io/post-images/1645598261672.png)
这时我们可以在容器启动命令中加入一个选项：**--link**，使得我们可以根据容器名来访问容器。

```bash
docker run -d -P --link 容器名/id 镜像名/id
```

![](https://tinaxiawuhao.github.io/post-images/1645598278472.png)
然而反向就不可以ping通，这是因为--link的本质是把需要连接的容器名/id写入启动容器的配置文件中，即增加了一个ip和容器名/id的映射：
![](https://tinaxiawuhao.github.io/post-images/1645598294726.png)
目前已经不建议使用这种方式。

## 8.3 自定义网络

我们使用命令：

```bash
docker network ls    # 查看所有的docker网络
```

![](https://tinaxiawuhao.github.io/post-images/1645598309260.png)
docker中的网络模式有：

- bridge：桥接（docker默认）/
- none：不配置网络 /
- host：和宿主机共享网络

**docker run** 命令默认带有一个参数--net bridge，此处的bridge指的就是docker0。如果我们不想使用docker0，那如何创建一个新的网络呢？

```bash
docker  network create --driver 网络模式 --subnet 子网ip --gateway 网关 网络名         
```

![](https://tinaxiawuhao.github.io/post-images/1645598354804.png)
我们不仅在**docker network ls**命令下发现了这个新创建的网络newnet，还可以使用**docker network inspect**命令查看其详细信息，包括了我们创建时定义的子网ip和网关：
![](https://tinaxiawuhao.github.io/post-images/1645598372176.png)
只要两个容器启动时都通过 **--net**，选用了同一个已创建的网络，不同容器间即可通过ip地址或容器名/id连通:
![](https://tinaxiawuhao.github.io/post-images/1645598422921.png)

## 8.4 网络连通

![](https://tinaxiawuhao.github.io/post-images/1645598436013.png)
对于建立在不同网络下(docker0, newnet)的两个容器tomcat01和tomcat02，他们的网段不同，因此是无法彼此ping通容器内部的：
![](https://tinaxiawuhao.github.io/post-images/1645598445887.png)
这时我们需要通过**docker network connect**命令打通容器与网络之间的连接：

```bash
docker network connect 网络名 容器名/id
```

![](https://tinaxiawuhao.github.io/post-images/1645598457014.png)
这个功能类似于将一个容器赋予多个ip地址，同样可以用**docker network inspect**命令查看网络连通后，该网络的变化：
![](https://tinaxiawuhao.github.io/post-images/1645598464633.png)
原本newnet网络中只含有tomcat02，现在增加了tomcat01，因此可以连通。
