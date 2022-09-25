---
title: 'linux安装node'
date: 2022-06-14 12:18:37
tags: [nodejs]
published: true
hideInList: false
feature: /post-images/XMNtsw8ig.png
isTop: false
---
以当前node版本16.13.0为例，新建演示目录/usr/local/node

### 1、下载

```
wget https://nodejs.org/dist/v16.13.0/node-v16.13.0-linux-x64.tar.xz
```

### 2、移动到指定目录

```
mv node-v16.13.0-linux-x64.tar.xz  /usr/local/node
```

### 3、解压

```
xz -d node-v16.13.0-linux-x64.tar.xz
tar -xvf node-v16.13.0-linux-x64.tar
```

测试下：

```
/root/sdemo/node-v16.13.0-linux-x64/bin/node -v
v16.13.0

/root/sdemo/node-v16.13.0-linux-x64/bin/npm -v
8.1.0
```

### 4、设置软连接，相当于windows设置环境变量

```
ln -s /usr/local/node/node-v16.13.0-linux-x64/bin/node   /usr/bin/node
ln -s /usr/local/node/node-v16.13.0-linux-x64/bin/npm   /usr/bin/npm
```

### 5、安装yarn,并设置yarn的软连

```
npm install yarn -g
ln -s /usr/local/node/node-v16.13.0-linux-x64/bin/yarn   /usr/bin/yarn
```

测试下：

```
yarn -v
1.22.17
```

### 6、安装cnpm

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
ln -s /usr/local/node/lib/node-v16.13.0-linux-x64/bin/cnpm  /usr/bin/cnpm
```



