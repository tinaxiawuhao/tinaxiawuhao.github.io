---
title: '第三章 Flink 部署'
date: 2021-04-09 15:20:19
tags: [flink]
published: true
hideInList: false
feature: /post-images/aPsJFkgVZ.png
isTop: false
---

## Standalone 模式
### 安装
解压缩 flink-1.10.1-bin-scala_2.12.tgz， 进入 conf 目录中。
1. 修改 flink/conf/flink-conf.yaml 文件：
   ![](https://tinaxiawuhao.github.io/post-images/1617953255377.png)
2. 修改 /conf/slaves 文件：
   ![](https://tinaxiawuhao.github.io/post-images/1617953264929.png)
3. 分发给另外两台机子：
   ![](https://tinaxiawuhao.github.io/post-images/1617953282098.png)
4. 启动
   ![](https://tinaxiawuhao.github.io/post-images/1617953292985.png)
访问 http://localhost:8081 可以对 flink 集群和任务进行监控管理。
![](https://tinaxiawuhao.github.io/post-images/1617953298218.png)

### 提交任务
1.  准备数据文件（ 如果需要）
   ![](https://tinaxiawuhao.github.io/post-images/1617953344217.png)
2. 把含数据文件的文件夹， 分发到 taskmanage 机器中
   ![](https://tinaxiawuhao.github.io/post-images/1617953351280.png)

<p style="text-indent:2em">如 果 从 文 件 中 读 取 数 据 ， 由 于 是 从 本 地 磁 盘 读 取 ， 实 际 任 务 会 被 分 发 到taskmanage 的机器中， 所以要把目标文件分发。</p>

3. 执行程序
```sh
./flink run -c com.atguigu.wc.StreamWordCount –p 2
FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port 7777
```

![](https://tinaxiawuhao.github.io/post-images/1617953534756.png)

4. 到目标文件夹中查看计算结果
注 意 ： 如 果 计 算 结 果 输 出 到 文 件 ， 会 保 存 到 taskmanage 的 机 器 下 ， 不 会 在jobmanage 下。
![](https://tinaxiawuhao.github.io/post-images/1617953549746.png)
1. 在 webui 控制台查看计算过程
   ![](https://tinaxiawuhao.github.io/post-images/1617953465087.png)
## Yarn 模式
<p style="text-indent:2em">以 Yarn 模式部署 Flink 任务时， 要求 Flink 是有 Hadoop 支持的版本， Hadoop环境需要保证版本在 2.2 以上， 并且集群中安装有 HDFS 服务</p>

### Flink on Yarn

<p style="text-indent:2em">Flink 提供了两种在 yarn 上运行的模式， 分别为 Session-Cluster 和 Per-Job-Cluster模式。</p>

1. Session-cluster 模式：
   
   ![](https://tinaxiawuhao.github.io/post-images/1617953616812.png)

<p style="text-indent:2em">Session-Cluster 模式需要先启动集群， 然后再提交作业， 接着会向 yarn 申请一块空间后， 资源永远保持不变。 如果资源满了， 下一个作业就无法提交， 只能等到
yarn 中的其中一个作业执行完成后， 释放了资源， 下个作业才会正常提交。 所有作
业共享 Dispatcher 和 ResourceManager； 共享资源； 适合规模小执行时间短的作业。

<p style="text-indent:2em">在 yarn 中初始化一个 flink 集群， 开辟指定的资源， 以后提交任务都向这里提交。 这个 flink 集群会常驻在 yarn 集群中， 除非手工停止。</p>

1.  Per-Job-Cluster 模式：
   
   ![](https://tinaxiawuhao.github.io/post-images/1617953630802.png)

<p style="text-indent:2em">一个 Job 会对应一个集群， 每提交一个作业会根据自身的情况， 都会单独向 yarn申请资源， 直到作业执行完成， 一个作业的失败与否并不会影响下一个作业的正常
提交和运行。 独享 Dispatcher 和 ResourceManager， 按需接受资源申请； 适合规模大
长时间运行的作业。

<p style="text-indent:2em">每次提交都会创建一个新的 flink 集群， 任务之间互相独立， 互不影响， 方便管理。 任务执行完成之后创建的集群也会消失。</p>

### Session Cluster
1. 启动 hadoop 集群（ 略）
2. 启动 yarn-session
```sh
./yarn-session.sh -n 2 -s 2 -jm 1024 -tm 1024 -nm test -d
```
其中：
>-n(--container)： TaskManager 的数量。
>-s(--slots)： 每个 TaskManager 的 slot 数量， 默认一个 slot 一个 core， 默认每个
>taskmanager 的 slot 的个数为 1， 有时可以多一些 taskmanager， 做冗余。
>-jm： JobManager 的内存（ 单位 MB)。
>-tm： 每个 taskmanager 的内存（ 单位 MB)。
>-nm： yarn 的 appName(现在 yarn 的 ui 上的名字)。
>-d： 后台执行。

![](https://tinaxiawuhao.github.io/post-images/1617953810515.png)

3. 执行任务
```sh
./flink run -c com.atguigu.wc.StreamWordCount
FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port 7777
```
4. 去 yarn 控制台查看任务状态

![](https://tinaxiawuhao.github.io/post-images/1617953816721.png)

5.  取消 yarn-session
```sh
yarn application --kill application_1577588252906_0001
3.2.2 Per Job Cluster
```
6. 启动 hadoop 集群（ 略）
7. 不启动 yarn-session， 直接执行 job
```sh
./flink run –m yarn-cluster -c com.atguigu.wc.StreamWordCount
FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host lcoalhost –port
7777
```
## Kubernetes 部署

<p style="text-indent:2em">容器化部署时目前业界很流行的一项技术， 基于 Docker 镜像运行能够让用户更加 方 便 地 对 应 用 进 行 管 理 和 运 维 。 容 器 管 理 工 具 中 最 为 流 行 的 就 是 Kubernetes（ k8s） ， 而 Flink 也在最近的版本中支持了 k8s 部署模式。</p>

1. 搭建 Kubernetes 集群（ 略）
2. 配置各组件的 yaml 文件
   
<p style="text-indent:2em">在 k8s 上构建 Flink Session Cluster， 需要将 Flink 集群的组件对应的 docker 镜像分别在 k8s 上启动， 包括 JobManager、 TaskManager、 JobManagerService 三个镜像服务。 每个镜像服务都可以从中央镜像仓库中获取。</p>

3. 启动 Flink Session Cluster
   
```sh
// 启动 jobmanager-service 服务
kubectl create -f jobmanager-service.yaml
// 启动 jobmanager-deployment 服务
kubectl create -f jobmanager-deployment.yaml
// 启动 taskmanager-deployment 服务
kubectl create -f taskmanager-deployment.yaml
```

4. 访问 Flink UI 页面
<p style="text-indent:2em">集群启动后， 就可以通过 JobManagerServicers 中配置的 WebUI 端口， 用浏览器输入以下 url 来访问 Flink UI 页面了：</p>

```
http://{JobManagerHost:Port}/api/v1/namespaces/default/services/flink-jobmanager:ui/proxy
```