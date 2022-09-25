---
title: 'harbor自建镜像仓库'
date: 2022-05-29 21:51:16
tags: [k8s]
published: true
hideInList: false
feature: /post-images/WOO39eqh2.png
isTop: false
---
## centos网络配置

### 1.设置主机名

```shell
[root@localhost ~]# hostnamectl  set-hostname harbor
```

### 2.添加 Host 解析

```shell
[root@harbor ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.40.131 k8s-master
192.168.40.132 k8s-node1
192.168.40.133 k8s-node2
192.168.40.150 hub.test.com
```

**k8s 集群每个节点添加解析（注意：K8s 每个节点，不是 Harbor）**

```shell
[root@k8s-master ~]# echo "192.168.40.150 hub.test.com" >> /etc/hosts
[root@k8s-node1 ~]# echo "192.168.40.150 hub.test.com" >> /etc/hosts
[root@k8s-node2 ~]# echo "192.168.40.150 hub.test.com" >> /etc/hosts
```

### 3.网络环境设置

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33 
```

**内容替换如下：**

```shell
BOOTPROTO=static #静态连接
ONBOOT=yes #网络设备开机启动
IPADDR=192.168.40.150 
NETMASK=255.255.255.0 #子网掩码
GATEWAY=192.168.40.2 #网关
DNS1=114.114.114.114 #DNS解析
```

**网络服务重启**

```shell
service network restart
```

## 安装环境

### 1.安装Docker-CE

```shell
# 卸载旧版本
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine  
# 安装所需的软件包
yum install -y yum-utils device-mapper-persistent-data lvm2    
# 添加docker存储库
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo     
#安装最新版的docker-ce
yum install -y docker-ce docker-ce-cli containerd.io    
#启动docker并设置为开机自启动
systemctl enable --now docker    
```

### 2.配置Docker镜像加速器

```bash
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://hub-mirror.c.163.com/"],
  "insecure-registries": ["https://hub.test.com"]
}
EOF
# 重启
systemctl daemon-reload && systemctl restart docker
```

### 3.安装docker-compose

```shell
#1、安装pip
yum -y install epel-release
yum install python3-pip
pip3 install --upgrade pip
#2、安装docker-compose
pip3 install docker-compose
#3、查看版本
docker-compose version
```

### 4.创建 https 证书

**安装 openssl**

```shell
[root@harbor]# yum install openssl -y
```

**创建证书目录，并赋予权限**

```shell
[root@harbor ~]# mkdir -p /cert/harbor
[root@harbor ~]# chmod -R 777 /cert/harbor
[root@harbor ~]# cd /cert/harbor
```

**创建服务器证书密钥文件 harbor.key**

```shell
[root@harbor harbor]# openssl genrsa -des3 -out harbor.key 2048
```

> 输入密码，确认密码，自己随便定义，但是要记住，后面会用到。

**创建服务器证书的申请文件 harbor.csr**

```shell
[root@harbor harbor]# openssl req -new -key harbor.key -out harbor.csr
```

> 输入密钥文件的密码, 然后一路回车。

**备份一份服务器密钥文件**

```shell
[root@harbor harbor]# cp harbor.key harbor.key.org
```

**去除文件口令**

```shell
[root@harbor harbor]# openssl rsa -in harbor.key.org -out harbor.key
```

> 输入密钥文件的密码

**创建一个自当前日期起为期十年的证书 harbor.crt**

```shell
[root@harbor harbor]# openssl x509 -req -days 3650 -in harbor.csr -signkey harbor.key -out harbor.crt
```

### 5.下载Harbor安装包并解压

> 链接：https://pan.baidu.com/s/1AUEw0Qw3w9lZP-rnAYaWjA 
> 提取码：qutg

```shell
wget https://github.com/goharbor/harbor/releases/download/v2.5.0/harbor-offline-installer-v2.5.0.tgz
tar zxvf harbor-offline-installer-v2.5.0.tgz
```

### 6.配置harbor.cfg和安装Harbor

```bash
vi harbor.yml
```

> 将http端口改成10086，因为默认用的80端口已经被占用，http可以指定任意端口；

![](https://tianxiawuhao.github.io/post-images/1654235318168.png)

接下来运行install.sh安装和启动harbor

```shell
./prepare
./install.sh
```

### 7.测试 Harbor

```shell
[root@k8s-master01 ~]# docker login https://hub.test.com
Username: admin
Password: Harbor12345 # 默认密码，可通过 harbor.yml 配置文件修改
```

**登录时报：Error response from daemon: Get https://hub.test.com/v2/: x509: certificate is not valid for any names, but wanted to match hub.test.com**
解决：修改客户端（即需要登陆harbor的机器）的docker.service 文件

```shell
vi /lib/systemd/system/docker.service
添加 --insecure-registry hub.test.com 
```

![](https://tianxiawuhao.github.io/post-images/1654236299657.png)

重新加载服务配置文件，并且重启docker服务

```shell
systemctl daemon-reload && systemctl restart docker
```

下载镜像推送到 Harbor

```shell
[root@k8s-node01 ~]# docker pull nginx
[root@k8s-node01 ~]# docker tag nginx:latest hub.test.com/library/test:v1
[root@k8s-node01 ~]# docker push hub.test.com/library/test:v1
```

### 8.Windows 访问 Harbor Web界面

**Windows 添加 hosts 解析路径**
```shell
C:\Windows\System32\drivers\etc\hosts
```

**添加信息**
```shell
192.168.40.150 hub.test.com
```
**浏览器访问测试**

> [https://hub.test.com](https://hub.test.com/harbor/projects)

![](https://tianxiawuhao.github.io/post-images/1654235306601.png)

> 用户密码：admin / Harbor12345

