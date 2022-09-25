---
title: '使用docker搭建FastDFS文件系统'
date: 2022-01-15 11:20:42
tags: [docker]
published: true
hideInList: false
feature: /post-images/yIe_yl-14.png
isTop: false
---
### 1.首先下载FastDFS文件系统的docker镜像

查询镜像

```shell
[root@localhost /]# docker search fastdfs
```

![](https://tinaxiawuhao.github.io/post-images/1641957924999.png)

安装镜像

```shell
[root@localhost ~]# docker pull season/fastdfs [root@localhost ~]# docker images
```



![](https://tinaxiawuhao.github.io/post-images/1641957936532.png)

### 2.使用docker镜像构建tracker容器（跟踪服务器，起到调度的作用）：

创建tracker容器

```shell
[root@localhost /]# docker run -ti -d --name trakcer -v ~/tracker_data:/fastdfs/tracker/data --net=host season/fastdfs tracker
```



Tracker服务器的端口默认是22122，你可以查看是否启用端口

```shell
[root@localhost /]# netstat -aon | grep 22122
```



### 3.使用docker镜像构建storage容器（存储服务器，提供容量和备份服务）：

```shell
docker run -tid --name storage -v ~/storage_data:/fastdfs/storage/data -v ~/store_path:/fastdfs/store_path --net=host -e TRACKER_SERVER:192.168.115.130:22122 -e GROUP_NAME=group1 season/fastdfs storage
```



### 4.此时两个服务都以启动，进行服务的配置。

进入storage容器，到storage的配置文件中配置http访问的端口，配置文件在**fdfs_conf**目录下的**storage.conf。**

```shell
[root@localhost /]# docker exec -it storage bash root@localhost:/# cd fdfs_conf root@localhost:/fdfs_conf# more storage.conf
```



![](https://tinaxiawuhao.github.io/post-images/1641957952610.png)

往下拉，你会发现storage容器的ip不是你linux的ip，如下：

![](https://tinaxiawuhao.github.io/post-images/1641957960584.png)

接下来，退出storage容器，并将配置文件拷贝一份出来：

```shell
[root@localhost ~]# docker cp storage:/fdfs_conf/storage.conf ~/ [root@localhost ~]# vi ~/storage.conf
```



![](https://tinaxiawuhao.github.io/post-images/1641957970483.png)

将修改后的配置文件拷贝到storagee的配置目录下：

```shell
[root@localhost ~]# docker cp ~/storage.conf storage:/fdfs_conf/
```



重新启动storage容器

```shell
[root@localhost ~]# docker stop storage [root@localhost ~]# docker start storage
```



查看tracker容器和storage容器的关联

```shell
[root@localhost ~]# docker exec -it storage bash root@localhost:/# cd fdfs_conf root@localhost:/fdfs_conf# fdfs_monitor storage.conf
```



![](https://tinaxiawuhao.github.io/post-images/1641957980436.png)

### 5.在docker模拟客户端上传文件到storage容器

开启一个客户端

```shell
[root@localhost 00]# docker run -tid --name fdfs_sh --net=host season/fastdfs sh
```



更改配置文件，因为之前已经改过一次了，所以现在直接拷贝

```shell
[root@localhost 00]# docker cp ~/storage.conf  fdfs_sh:/fdfs_conf/
```



创建一个txt文件

```shell
[root@localhost 00]# docker exec -it fdfs_sh bash root@localhost:/# echo hello>a.txt
```



进入**fdfs_conf**目录，并将文件上传到**storage**容器

```shell
root@localhost:/# cd fdfs_conf root@localhost:/fdfs_conf# fdfs_upload_file storage.conf /a.txt
```



**/a.txt**：指要上传的文件

上传之后，根据返回的路径去找**a.txt**

![](https://tinaxiawuhao.github.io/post-images/1641957990375.png)

退出去查看上传的txt文件

[root@localhost ~]# cd ~/store_path/data/00/00 [root@localhost 00]# ls

![](https://tinaxiawuhao.github.io/post-images/1641958001513.png)

查看是否和输入的值是否相同

```shell
[root@localhost 00]# more wKhzg1wGsieAL-3RAAAABncc3SA337.txt
```

