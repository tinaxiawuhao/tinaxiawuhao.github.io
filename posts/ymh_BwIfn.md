---
title: 'gitlab提交代码自动触发Jenkins构建操作'
date: 2022-06-16 12:27:16
tags: [jenkins,git]
published: true
hideInList: false
feature: /post-images/ymh_BwIfn.png
isTop: false
---
### Jenkins配置

#### 插件安装
系统管理=====》插件管理=====》可选插件=====》搜索要按照的插件(gitlab hook-plugin和gitlab-plugin)
如果找不到上面两个插件，安装gitlab和gitlab hook即可

![](https://tianxiawuhao.github.io/post-images/1655527394729.gif)

#### 创建测试项目
![](https://tianxiawuhao.github.io/post-images/1655527415958.gif)

#### gitlab配置
打开要关联Jenkins项目的设置选项找到webhooks选项，把Jenkins中的项目触发器url以及Secret token配置到gitlab的webhooks选项中

URL

![](https://tianxiawuhao.github.io/post-images/1655527426929.png)

Secret token

![](https://tianxiawuhao.github.io/post-images/1655527439844.gif)



![](https://tianxiawuhao.github.io/post-images/1655527449909.gif)

#### 验证效果
我们测试下点击刚刚关联的构建动作，Jenkins会不会自动构建
当出现请求状态码200的时候证明我们的关联动作已经执行

![](https://tianxiawuhao.github.io/post-images/1655527459975.png)

下面我们去到Jenkins中看下是否有构建历史

![](https://tianxiawuhao.github.io/post-images/1655527466575.png)

再试下手动push代码到gitlab会不会触发Jenkins的构建任务
手动去到服务器上向远程gitlab分支推送文件
