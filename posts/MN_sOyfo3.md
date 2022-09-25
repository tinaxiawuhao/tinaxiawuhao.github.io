---
title: 'linux安装jenkins'
date: 2022-06-13 12:05:04
tags: [jenkins,linux]
published: true
hideInList: false
feature: /post-images/MN_sOyfo3.png
isTop: false
---
### 1、安装JDK

```shell
yum install -y java
```

### 2、安装jenkins

**添加Jenkins库到yum库，Jenkins将从这里下载安装。**

```shell
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo

rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key

yum install -y jenkins
或
yum install jenkins -y --nogpgcheck
```

### 3、修改jenkins端口
```
[root@tools ~]# rpm -qa | grep jenkins
jenkins-2.340-1.1.noarch
```

```
[root@tools ~]# rpm -ql jenkins-2.340-1.1.noarch
/etc/init.d/jenkins
/etc/logrotate.d/jenkins
/etc/sysconfig/jenkins
/usr/bin/jenkins
/usr/lib/systemd/system/jenkins.service
/usr/sbin/rcjenkins
/usr/share/java/jenkins.war
/usr/share/jenkins
/usr/share/jenkins/migrate
/var/cache/jenkins
/var/lib/jenkins
/var/log/jenkins
```

```
[root@tools ~]# vim /usr/lib/systemd/system/jenkins.service
vi /etc/sysconfig/jenkins

找到修改端口号：
JENKINS_PORT=“9000” #此端口不冲突可以不修改
```

### 4、启动jenkins

```
systemctl start jenkins.service
```

```
[root@tools ~]# netstat -nultp
```

> 安装成功后Jenkins将作为一个守护进程随系统启动
> 系统会创建一个“jenkins”用户来允许这个服务，如果改变服务所有者，同时需要修改/var/log/jenkins, /var/lib/jenkins, 和/var/cache/jenkins的所有者
> 启动的时候将从/etc/sysconfig/jenkins获取配置参数
> 默认情况下，Jenkins运行在8080端口，在浏览器中直接访问该端进行服务配置
> Jenkins的RPM仓库配置被加到/etc/yum.repos.d/jenkins.repo


### 5、打开jenkins
在浏览器中访问 http://192.168.3.200:9000/
首次进入会要求输入初始密码

初始密码在：/var/lib/jenkins/secrets/initialAdminPassword

重置密码：admin/admin