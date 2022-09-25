---
title: 'Docker安装clickhouse'
date: 2021-11-13 13:45:31
tags: [clickhouse]
published: true
hideInList: false
feature: /post-images/BBgY7nkHa.png
isTop: false
---
ClickHouse 很多大厂都在用，本篇主要使用Docker进行安装

### 安装配置
创建目录并更改权限

```shell
mkdir -p /app/cloud/clickhouse/data
mkdir -p /app/cloud/clickhouse/conf
mkdir -p /app/cloud/clickhouse/log

chmod -R 777 /app/cloud/clickhouse/data
chmod -R 777 /app/cloud/clickhouse/conf
chmod -R 777 /app/cloud/clickhouse/log
```

### 拉取镜像

```shell
docker pull yandex/clickhouse-server:20.3.5.21
docker pull yandex/clickhouse-client:20.3.5.21
```

查看 [https://hub.docker.com/r/yandex/clickhouse-server/dockerfile](https://hub.docker.com/r/yandex/clickhouse-server/dockerfile) 文件，EXPOSE  9000  8123  9009 了三个端口

### 创建临时容器

```shell
docker run --rm -d --name=clickhouse-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9009:9009 -p 9000:9000 yandex/clickhouse-server:20.3.5.21
```

### 复制临时容器内配置文件到宿主机

```shell
docker cp clickhouse-server:/etc/clickhouse-server/config.xml D:/clickhouse/conf/config.xml
docker cp clickhouse-server:/etc/clickhouse-server/users.xml D:/clickhouse/conf/users.xml
```

### 停掉临时容器

```shell
docker stop clickhouse-server
```

### 创建default账号密码

```shell
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

```shell
SEGByR98
211371f5bc54970907173acf6facb35f0acbc17913e1b71b814117667c01d96d
```
会输出明码和SHA256密码  

### 创建root账号密码

```shell
PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

```shell
092j3AnV
35542ded44184b1b4b6cd621e052662578025b58b4187176a3ad2b9548c8356e
```
会输出明码和SHA256密码

修改 D:/clickhouse/conf/users.xml
把default账号设为只读权限，并设置密码yandex-->users-->default-->profile节点设为 `readonly` 注释掉 yandex-->users-->default-->password 节点 新增  yandex-->users-->default-->password_sha256_hex 节点，填入生成的密码

修改default账号

```xml
<?xml version="1.0"?>
<yandex>
    <users>
        <default>
            <password_sha256_hex>211371f5bc54970907173acf6facb35f0acbc17913e1b71b814117667c01d96d</password_sha256_hex>
            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>readonly</profile>
            <quota>default</quota>
        </default>
	</users>
</yandex>
```



新增root账号

```xml
<?xml version="1.0"?>
<yandex>
    <users>
        <root>
                    <password_sha256_hex>35542ded44184b1b4b6cd621e052662578025b58b4187176a3ad2b9548c8356e</password_sha256_hex>
             <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
        </root>
	</users>
</yandex>
```

### 创建容器

```shell
docker run -d --name=clickhouse-server -p 8123:8123 -p 9009:9009 -p 9000:9000 --ulimit nofile=262144:262144 -v D:/clickhouse/data:/var/lib/clickhouse:rw -v D:/clickhouse/conf/config.xml:/etc/clickhouse-server/config.xml -v D:/clickhouse/conf/users.xml:/etc/clickhouse-server/users.xml -v D:/clickhouse/log:/var/log/clickhouse-server:rw yandex/clickhouse-server
```

### 操作

1. docker exec -it docker-clickhouse /bin/bash 进入容器
2. clickhouse-client 进入clickhouse命令行