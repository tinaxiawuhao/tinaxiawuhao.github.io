---
title: 'docker 宿主机定时清除容器的运行日志'
date: 2022-01-17 11:35:08
tags: [docker]
published: true
hideInList: false
feature: /post-images/KW_02xshc.png
isTop: false
---


一般docker容器都是最小化安装，不仅如此系统定时器相关的服务也不存在，自己去安装也很麻烦，故此直接使用宿主机的定时器即可。

#### 一、在容器中编写清除日志脚本

这一部分不论你是把定时器加在宿主机或者是容器都必须要去做的 ；

网上随意一搜就可以看到如下的删除模板：

```shell
find 对应目录 -mtime +天数 -name "文件名" -exec rm -rf {} \;
```

因为本人的日志目录层级比较深 所以改良了如下：

```shell
1. -- /opt/auto-del-log.sh 
2. \#!/bin/sh
3. find /home/schedule_log/ -mtime -5 -type f -iname "*.log" -exec rm -rf {} \;
```

一定记得加可执行权限

```shell
chmod +777 /opt/auto-del-log.sh
```

后面经过验证 其实效果是一样的! 重点就是你要去验证你的脚本有无效！ 你可以这样直接输入验证

```shell
1. find /home/schedule_log/  -type f -iname "*.log"
2. 或者
3. find /home/schedule_log/  -name "*.log"
```

如果能查出你想删除的文件那么后面就可以开始套模板了。

```shell
-mtime：标准语句写法；
+30：查找30天前的文件，这里用数字代表天数；
"*.log"：希望查找的数据类型，"*.jpg"表示查找扩展名为jpg的所有文件，"*"表示查找所有文件，这个可以灵活运用，举一反三；
-exec：固定写法；
rm -rf：强制删除文件，包括目录；
{} \; ：固定写法，一对大括号+空格+\+; 
```

#### 二、宿主机加入定时器

使用docker exec 命令校验之前写的脚本是否有效 如下：

```shell
docker exec -it tomcat8002 /opt/auto-del-log.sh
tomcat8002 ： 容器名称或者ID
/opt/auto-del-log.sh :脚本在容器中的位置
```

如果此命令有效那么就可以编辑定时器了 本人采用的是centos7 具体可以参看网上介绍的挺全的一篇博客如下： [centos7 linux定时任务详解](https://blog.csdn.net/xudailong_blog/article/details/79303785)

```shell
crontab -e
02 4 * * *  docker exec -it tomcat8002 /opt/auto-del-log.sh
```

接下来就OK啦！