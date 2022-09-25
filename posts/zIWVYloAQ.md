---
title: 'Helm简介'
date: 2022-06-08 18:58:52
tags: [k8s]
published: true
hideInList: false
feature: /post-images/zIWVYloAQ.png
isTop: false
---
### 一、简介
Helm是Kubernetes的包管理器。包管理器类似于Ubuntu中使用的apt、Python中的pip一样，能快速查找、下载和安装软件包。
**Helm解决的痛点**

- 在Kubernetes中部署一个可以使用的应用，需要涉及到很多的Kubernetes资源的共同协作。比如安装一个WordPress，用到了一些Kubernetes的一些资源对象，包括Deployment用于部署应用、Service提供服务发现、Secret配置 WordPress的用户名和密码，可能还需要pv和pvc来提供持久化服务。并且WordPress数据是存储在mariadb里面的，所以需要mariadb启动就绪后才能启动 WordPress。这些k8s资源过于分散，不方便进行管理。
- Helm把Kubernetes资源(比如deployments、services或ingress等) 打包到一个chart中，而chart被保存到chart仓库。通过chart仓库可用来存储和分享chart。Helm使发布可配置，支持发布应用配置的版本管理，简化了Kubernetes部署应用的版本控制、打包、发布、删除、更新等操作。

**Helm相关组件及概念**

- **helm**是一个命令行工具，主要用于Kubernetes应用程序Chart的创建、打包、发布以及创建和管理本地和远程的Chart仓库。chart Helm的软件包，采用TAR格式。类似于APT的DEB包或者YUM的RPM包，其包含了一组定义Kubernetes资源相关的 YAML文件。
- **Repoistory** Helm的软件仓库，Repository本质上是一个Web服务器，该服务器保存了一系列的Chart软件包以供用户下载，并且提供了一个该Repository的Chart包的清单文件以供查询。Helm可以同时管理多个不同的Repository。
- **release**使用helm install命令在Kubernetes集群中部署的Chart称为Release。可以理解为Helm使用Chart包部署的一个应用实例。

> 创建release
> helm 客户端从指定的目录或本地tar文件或远程repo仓库解析出chart的结构信息
> helm 客户端根据 chart 和 values 生成一个 release
> helm 将install release请求直接传递给 kube-apiserver
>
> 删除release
> helm 客户端从指定的目录或本地tar文件或远程repo仓库解析出chart的结构信息
> helm 客户端根据 chart 和 values 生成一个 release
> helm 将delete release请求直接传递给 kube-apiserver
>
> 更新release
> helm 客户端从指定的目录或本地tar文件或远程repo仓库解析出chart的结构信息
> helm 将收到的信息生成新的 release，并同时更新这个 release 的 history
> helm 将新的 release 传递给 kube-apiserver 进行更新

- **chart**的基本结构

  ```java
  ## Helm的打包格式叫做chart，所谓chart就是一系列文件, 它描述了一组相关的 k8s 集群资源。
  ## Chart中的文件安装特定的目录结构组织, 最简单的chart 目录如下所示：
  ./
  ├── charts                            #  目录存放依赖的chart
  ├── Chart.yaml                        #  包含Chart的基本信息，包括chart版本，名称等
  ├── templates                         #  目录下存放应用一系列k8s资源的yaml模板
  │   ├── deployment.yaml
  │   ├── _helpers.tpl                  #  此文件中定义一些可重用的模板片断，此文件中的定义在任何资源定义模板中可用
  │   ├── ingress.yaml
  │   ├── NOTES.txt                     #  介绍chart部署后的帮助信息，如何使用chart等
  │   ├── serviceaccount.yaml
  │   ├── service.yaml
  │   └── tests
  │       └── test-connection.yaml
  └── values.yaml                       #  包含了必要的值定义（默认值）, 用于存储templates目录中模板文件中用到变量的值
  ```

### 二、安装Helm
#### 1 安装Helm
```shell
[root@Ansible01 ~]# wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
[root@Ansible01 ~]# tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
[root@Ansible01 ~]# mv linux-amd64/helm /usr/local/bin/helm
[root@Ansible01 ~]# helm version
version.BuildInfo{Version:"v3.8.2", GitCommit:"6e3701edea09e5d55a8ca2aae03a68917630e91b", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

#### 2 添加常用repo
**添加存储库**

```shell
[root@Ansible01 ~]# helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
"aliyun" has been added to your repositories
```

**更新存储库**

```shell
[root@Ansible01 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "aliyun" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
```

**查看存储库**

```shell
[root@Ansible01 ~]# helm repo list
NAME                    URL                                                   
bitnami                 https://charts.bitnami.com/bitnami                    
prometheus-community    https://prometheus-community.github.io/helm-charts    
stable                  http://mirror.azure.cn/kubernetes/charts              
aliyun                  https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```

**删除存储库**

```shell
[root@Ansible01 ~]# helm repo remove  aliyun
"aliyun" has been removed from your repositories
```


### 三、使用Helm
#### 1 使用chart部署一个mysql

```shell
#  查找所有repo下的所有chart
helm search repo

#  查找stable这个repo下的chart mysql
helm search repo stable/mysql
 
#  查看chart信息：
helm show chart stable/mysql

 #  安装包：
helm install db stable/mysql**

# 查看发布状态：
helm status db
```

```shell
[root@Ansible01 ~]# kubectl get pod -n default
NAME                        READY   STATUS    RESTARTS       AGE
db-mysql-599d764c8c-knfqc   0/1     Pending   0              2m10s
```

```shell
[root@Ansible01 ~]# kubectl describe pod db-mysql-599d764c8c-knfqc -n default
......
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  17s (x3 over 2m33s)  default-scheduler  0/4 nodes are available: 4 pod has unbound immediate PersistentVolumeClaims.
```

#### 2 安装前自定义chart配置选项
- 上面部署的mysql并没有成功，这是因为并不是所有的chart都能按照默认配置运行成功，可能会需要一些环境依赖，例如PV。
- 所以需要自定义chart配置选项，安装过程中有两种方法可以传递配置数据：–values（或-f）：指定带有覆盖的YAML文件。这可以多次指定，最右边的文件优先。
- –set：在命令行上指定替代。如果两者都用，–set优先级高

**创建满足的PV**

```shell
[root@Ansible01 mysql]# cat pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-volume
  labels:
    type: local
spec:
  storageClassName: "managed-nfs-storage"
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"

[root@Ansible01 hello-world]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                           STORAGECLASS          REASON   AGE
mysql-pv-volume                            10Gi       RWO            Retain           Available                                                   managed-nfs-storage            4s
```

```shell
[root@Ansible01 hello-world]# helm uninstall db
release "db" uninstalled

[root@Ansible01 hello-world]# cat config.yaml 
persistence:
  enabled: true
  storageClass: "managed-nfs-storage"
  accessMode: ReadWriteOnce
  size: 8Gi
mysqlUser: "k8s"
mysqlPassword: "123456"
mysqlDatabase: "k8s"

[root@Ansible01 hello-world]# helm install db -f config.yaml stable/mysql

[root@Ansible01 hello-world]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                           STORAGECLASS          REASON   AGE
mysql-pv-volume                            10Gi       RWO            Retain           Bound      default/db-mysql                                managed-nfs-storage            17s


[root@Ansible01 hello-world]# kubectl get pod 
NAME                       READY   STATUS    RESTARTS       AGE
db-mysql-f7fbfdd68-tc8cm   1/1     Running   0              11s
```


#### 3 构建一个Helm Chart
- 命令来创建一个名为mychart 的helm chart

```shell
[root@Ansible01 2022-05-21]# helm create mychart
Creating mychart
[root@Ansible01 2022-05-21]# ls
mychart
```

- 创建后会在目录创建一个mychart目录

```shell
[root@Ansible01 2022-05-21]# tree mychart/
mychart/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 10 files
```

- 其中mychart目录下的templates目录中保存有部署的模板文件，values.yaml中定义了部署的变量，Chart.yaml文件包含有version（chart版本）和appVersion（包含应用的版本）。

```shell
[root@Ansible01 2022-05-21]# cat mychart/Chart.yaml  |grep -v "^#" |grep -v ^$
apiVersion: v2
name: mychart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

- 选择镜像及标签和副本数（这里设置1个）

```shell
[root@Ansible01 2022-05-21]# vi mychart/values.yaml
replicaCount: 1

image:
  repository: myapp
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1"
```

- 编辑完成后检查依赖及模板配置是否正确

```shell
[root@Ansible01 2022-05-21]# cd mychart/
[root@Ansible01 mychart]# helm lint .
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- 打包应用，其中0.1.0为在Chart.yaml文件中定义的version（chart版本）信息

```shell
[root@Ansible01 2022-05-21]# helm package mychart/
Successfully packaged chart and saved it to: /root/2022-05-21/mychart-0.1.0.tgz
[root@Ansible01 2022-05-21]# ls
mychart  mychart-0.1.0.tgz
```

- 升级、回滚和删除

```shell
#  发布新版本的chart时，或者当您要更改发布的配置时，可以使用该helm upgrade 命令。
helm upgrade --set imageTag=1.17 web mychart
helm upgrade -f values.yaml web mychart
 
#  如果在发布后没有达到预期的效果，则可以使用helm rollback回滚到之前的版本。
例如将应用回滚到第一个版本：
helm rollback web 2

#  卸载发行版，请使用以下helm uninstall命令：
helm uninstall web

#  查看历史版本配置信息
helm get --revision 1 web
```

- 下载、上传tar包

```shell
[root@Ansible01 2022-05-21]# helm pull stable/traefik
[root@Ansible01 2022-05-21]# ls
traefik-1.87.7.tgz
```

```shell
# 　 一种方式是直接上传charts文件夹，seldon-mab是charts目录，harbor-test-helm是harbor charts repo名称。
helm push seldon-mab harbor-test-helm

#　　另一种是将charts package文件包push
helm push seldon-core-operator-1.5.1.tgz harbor-test-helm
```

- 安装到k8s的其它空间：--namespace=monitoring

```shell
helm install --name prometheus-operator --set rbacEnable=true --namespace=monitoring stable/prometheus-operator
```

### 四、Helm使用minio搭建私有仓库

**minio介绍**

我们一般是从本地的目录结构中的chart去进行部署，如果要集中管理chart,就需要涉及到repository的问题，因为helmrepository都是指到外面的地址，接下来我们可以通过minio建立一个企业私有的存放仓库。

Minio提供对象存储服务。它的应用场景被设定在了非结构化的数据的存储之上了。众所周知，非结构化对象诸如图像/音频/视频/log文件/系统备份/镜像文件…等等保存起来管理总是不那么方便，size变化很大，类型很多，再有云端的结合会使得情况更加复杂，minio就是解决此种场景的一个解决方案。Minio号称其能很好的适应非结构化的数据，支持AWS的S3，非结构化的文件从数KB到5TB都能很好的支持。

Minio的使用比较简单，只有两个文件，服务端minio,客户访问端mc,比较简单。

在项目中，我们可以直接找一台虚拟机作为Minio Server,提供服务，当然minio也支持作为Pod部署。

#### 1 helm3存储库更改

  在Helm 2中，默认情况下包括稳定的图表存储库。在Helm 3中，默认情况下不包含任何存储库。因此需要做的第一件事就是添加一个存储库。官方图表存储库将在有限的时间内继续接收补丁，但是将不再作为默认存储库包含在Helm客户端中。

#### 2 minio介绍

  MinIO 是一个基于Apache License  v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

MinIO是一个非常轻量的服务,可以很简单的和其他应用的结合，类似 NodeJS, Redis 或者 MySQL。

#### 3 安装minio服务端

**1使用容器安装服务端**

```shell
docker pull minio/minio
docker run -p 9000:9000 minio/minio server /data
```

**2使用二进制安装服务端**

```shell
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
mkdir -p /chart
./minio server /chart
```

![](https://tianxiawuhao.github.io/post-images/1657191641142.png)

**访问Browser Access地址：**

![](https://tianxiawuhao.github.io/post-images/1657191649298.png)

**在启动日志中获取access  key和secret  key**

![](https://tianxiawuhao.github.io/post-images/1657191656837.png)

看到这个页面则表示登陆成功

![](https://tianxiawuhao.github.io/post-images/1657191664579.png)

至此服务端部署完成。

#### 4 安装minio客户端

**1.使用容器安装客户端**

```shell
docker pull minio/mc
docker run minio/mc ls play
```

**2.使用二进制安装客户端**

```shell
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
./mc
```

![](https://tianxiawuhao.github.io/post-images/1657191674247.png)

#### 5 连接至服务端

```shell
./mc config host add myminio http://172.17.0.1:9000 XH2LCA4AJIP52RDB4P5M CDDCuoS2FNsdW8S0bodkcs2729N+TH5lFov+rrT3
```

![](https://tianxiawuhao.github.io/post-images/1657191689882.png)

服务端启动时候的access  key和secret  key

#### 6 mc的shell使用别名

```shell
ls=mc ls
cp=mc cp
cat=mc cat
mkdir=mc mb
pipe=mc pipe
find=mc find
```

#### 7 创建bucket

```shell
./mc mb myminio/minio-helm-repo
```

![](https://tianxiawuhao.github.io/post-images/1657191697994.png)

#### 8 设置bucket和objects匿名访问

```shell
./mc policy set download myminio/minio-helm-repo
```

![](https://tianxiawuhao.github.io/post-images/1657191704821.png)

#### 9 helm创建与仓库连接的index.yaml文件

```shell
mkdir /root/helm/repo
helm repo index helm/repo/
```

#### 10 helm与minio仓库进行连接

**1.将index.yaml文件推送到backet中去**

```shell
./mc cp helm/repo/index.yaml myminio/minio-helm-repo
```

![](https://tianxiawuhao.github.io/post-images/1657191712580.png)

**2.helm连接私仓**

```shell
helm repo add fengnan http://192.168.0.119:9000/minio-helm-repo
```

![](https://tianxiawuhao.github.io/post-images/1657191720766.png)

**3.更新repo仓库**

```shell
helm repo update
```

![](https://tianxiawuhao.github.io/post-images/1657191729687.png)

**4.查看repo**

```shell
helm repo list
```

![](https://tianxiawuhao.github.io/post-images/1657191738239.png)

**5.查看repo中的文件**

```shell
./mc ls myminio/minio-helm-repo
```

![](https://tianxiawuhao.github.io/post-images/1657191745072.png)

**6.登录服务端web界面查看**

![](https://tianxiawuhao.github.io/post-images/1657191757447.png)

