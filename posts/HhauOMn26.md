---
title: 'k8s资源操作'
date: 2022-06-04 11:34:10
tags: [k8s]
published: true
hideInList: false
feature: /post-images/HhauOMn26.png
isTop: false
---
### 1.资源类型

| 资源分类     | 类型                     | 具体资源                                                     |
| ------------ | ------------------------ | ------------------------------------------------------------ |
| 名称空间级别 | 工作负载型资源           | Pod(pod:k8s 系统中可以创建和管理的最小单元)                  |
| 名称空间级别 | 工作负载型资源           | ReplicaSet(rs:用来确保容器应用的副本数始终保持在用户定义的副本数) |
| 名称空间级别 | 工作负载型资源           | Deployment(deployment:为 Pod 和 ReplicaSet 提供了一个 声明式定义 (declarative) 方法) |
| 名称空间级别 | 工作负载型资源           | StatefulSet(sts:为了解决有状态服务的问题)                    |
| 名称空间级别 | 工作负载型资源           | DaemonSet(ds:)                                               |
| 名称空间级别 | 工作负载型资源           | Job(jobs:负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod) |
| 名称空间级别 | 工作负载型资源           | CronJob(jobs:管理基于时间的 Job)                             |
| 名称空间级别 | 服务发现及负载均衡型资源 | Service(svc:为一组功能相同的Pod提供一个供外界访问的地址)     |
| 名称空间级别 | 服务发现及负载均衡型资源 | Ingress(ing:官方只能实现四层代理，INGRESS 可以实现七层代理)  |
| 名称空间级别 | 配置与存储型资源         | Volume(存储卷)                                               |
| 名称空间级别 | 配置与存储型资源         | CSI(容器存储接口-- 第三方存储卷)                             |
| 名称空间级别 | 特殊类型的存储卷         | ConfigMap(当配置中心来使用的资源类型)                        |
| 名称空间级别 | 特殊类型的存储卷         | Secret(保存敏感数据)                                         |
| 名称空间级别 | 特殊类型的存储卷         | DownwardApi(把外部环境中的信息输出给容器)                    |
| 集群级资源   | 集群级资源               | Namespace(命名空间)                                          |
| 集群级资源   | 集群级资源               | Node(集群节点)                                               |
| 集群级资源   | 集群级资源               | Role(角色)                                                   |
| 集群级资源   | 集群级资源               | ClusterRole(集群角色)                                        |
| 集群级资源   | 集群级资源               | RoleBinding(角色绑定)                                        |
| 集群级资源   | 集群级资源               | ClusterRoleBinging(集群角色绑定)                             |
| 元数据型资源 | 元数据型资源             | HPA(根据 Pod的 CPU 利用率扩所容)                             |
| 元数据型资源 | 元数据型资源             | PodTemplate(pod资源模板)                                     |
| 元数据型资源 | 元数据型资源             | LimitRange(资源限制)                                         |

### 2.操作指令

|   命令名   | 类型 |          作用          |
| :--------: | :--: | :--------------------: |
|   `get`    |  查  | 列出某个类型的下属资源 |
| `describe` |  查  | 查看某个资源的详细信息 |
|   `logs`   |  查  |  查看某个 pod 的日志   |
|  `create`  |  增  |        新建资源        |
| `explain`  |  查  |  查看某个资源的配置项  |
|  `delete`  |  删  |      删除某个资源      |
|   `edit`   |  改  |  修改某个资源的配置项  |
|  `apply`   |  改  |  应用某个资源的配置项  |

### 3.查看和进入空间
|                            命令名                            |                             作用                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                  kubectl get pod -n名称空间                  |                   查看对应名称空间内的pod                    |
|           kubectl exec -it pod名字 -n名称空间 bash           |                   进入对应名称空间的pod内                    |
|                  kubectl get nodes -o wide                   |            获取节点和服务版本信息，并查看附加信息            |
|                       kubectl get pod                        |              获取pod信息，默认是default名称空间              |
|                   kubectl get pod -o wide                    |      获取pod信息，默认是default名称空间，并查看附加信息      |
|                kubectl get pod -n kube-system                |                    获取指定名称空间的pod                     |
|            kubectl get pod -n kube-system podName            |                 获取指定名称空间中的指定pod                  |
|                      kubectl get pod -A                      |                    获取所有名称空间的pod                     |
|    kubectl get pods -o yaml<br />kubectl get pods -o json    |         查看pod的详细信息，以yaml格式或json格式显示          |
|               kubectl get pod -A --show-labels               |                      查看pod的标签信息                       |
|       kubectl get pod -A --selector=“k8s-app=kube-dns”       |             根据Selector（label query）来查询pod             |
|                   kubectl exec podName env                   |                    查看运行pod的环境变量                     |
| kubectl logs -f --tail 500 -n kube-system kube-apiserver-k8s-master |                      查看指定pod的日志                       |
|                      kubectl get svc -A                      |                查看所有名称空间的service信息                 |
|                kubectl get svc -n kube-system                |                查看指定名称空间的service信息                 |
|                        kubectl get cs                        |                  查看componentstatuses信息                   |
|                      kubectl get cm -A                       |                    查看所有configmaps信息                    |
|                      kubectl get sa -A                       |                 查看所有serviceaccounts信息                  |
|                      kubectl get ds -A                       |                    查看所有daemonsets信息                    |
|                    kubectl get deploy -A                     |                   查看所有deployments信息                    |
|                      kubectl get rs -A                       |                   查看所有replicasets信息                    |
|                      kubectl get sts -A                      |                   查看所有statefulsets信息                   |
|                     kubectl get jobs -A                      |                       查看所有jobs信息                       |
|                      kubectl get ing -A                      |                    查看所有ingresses信息                     |
|                        kubectl get ns                        |                      查看有哪些名称空间                      |
| kubectl describe pod podName<br/>kubectl describe pod -n kube-system kube-apiserver-k8s-master |                      查看pod的描述信息                       |
|        kubectl describe deploy -n kube-system coredns        |            查看指定名称空间中指定deploy的描述信息            |
|             kubectl top node<br/>kubectl top pod             | 查看node或pod的资源使用情况<br/>（需要heapster 或metrics-server支持） |
|    kubectl cluster-info <br /> kubectl cluster-info dump     |                         查看集群信息                         |
|  kubectl -s https://172.16.1.110:6443 get componentstatuses  |          查看各组件信息【172.16.1.110为master机器】          |

### 4.进入pod启动的容器
|                            命令名                            |        作用        |
| :----------------------------------------------------------: | :----------------: |
|          kubectl exec -it podName -n nsName /bin/sh          |      进入容器      |
|         kubectl exec -it podName -n nsName /bin/bash         |      进入容器      |

### 5.添加label值
|                            命令名                            |        作用        |
| :----------------------------------------------------------: | :----------------: |
|          kubectl label nodes k8s-node01 zone=north           | 为指定节点添加标签 |
|             kubectl label nodes k8s-node01 zone              | 为指定节点删除标签 |
|      kubectl label pod podName -n nsName role-name=test      | 为指定pod添加标签  |
| kubectl label pod podName -n nsName role-name=dev --overwrite |  修改lable标签值   |
|        kubectl label pod podName -n nsName role-name         |   删除lable标签    |

### 6.滚动升级
|                            命令名                            |         作用         |
| :----------------------------------------------------------: | :------------------: |
|          kubectl apply -f myapp-deployment-v2.yaml           | 通过配置文件滚动升级 |
| kubectl set image deploy/myapp-deployment myapp=“registry.cn-beijing.aliyuncs.com/google_registry/myapp:v3” |   通过命令滚动升级   |
| kubectl rollout undo deploy/myapp-deployment <br />kubectl rollout undo deploy myapp-deployment | pod回滚到前一个版本  |
| kubectl rollout undo deploy/myapp-deployment --to-revision=2 |  回滚到指定历史版本  |

### 7.动态伸缩

|                         命令名                         |           作用           |
| :----------------------------------------------------: | :----------------------: |
|   kubectl scale deploy myapp-deployment --replicas=5   |         动态伸缩         |
| kubectl scale --replicas=8 -f myapp-deployment-v2.yaml | 动态伸缩（根据yaml文件） |

### 8.操作类命令
|                            命令名                            |                             作用                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                  kubectl create -f xxx.yaml                  |                           创建资源                           |
|                  kubectl apply -f xxx.yaml                   |                           应用资源                           |
|                       kubectl apply -f                       | 应用资源，该目录下的所有 .yaml, .yml, 或 .json 文件都会被使用 |
|                kubectl create namespace test                 |                       创建test名称空间                       |
|       kubectl delete -f xxx.yaml<br/>kubectl delete -f       |                           删除资源                           |
|                  kubectl delete pod podName                  |                        删除指定的pod                         |
|              kubectl delete pod -n test podName              |                  删除指定名称空间的指定pod                   |
| kubectl delete svc svcName<br/>kubectl delete deploy deployName<br/>kubectl delete ns nsName |                         删除其他资源                         |
| kubectl delete pod podName -n nsName --grace-period=0 --force<br/>kubectl delete pod podName -n nsName --grace-period=1<br/>kubectl delete pod podName -n nsName --now |                           强制删除                           |
|                   kubectl edit pod podName                   |                           编辑资源                           |

### 9.状态

|       状态名       |            含义            |
| :----------------: | :------------------------: |
|     `Running`      |           运行中           |
|      `Error`       |     异常，无法提供服务     |
|     `Pending`      |  准备中，暂时无法提供服务  |
|   `Terminaling`    |     结束中，即将被移除     |
|     `Unknown`      | 未知状态，多发生于节点宕机 |
| `PullImageBackOff` |        镜像拉取失败        |

必须存在的属性

| 参数名 | 字段类型 | 说明 |
| ------ | -------- | ---- |
|version	|String	|K8S API版本, 可以使用kubectl api-version命令查询|
|kind	|String	|指的是yaml文件定义的资源类型和角色, 比如Pod|
|metadata	|Object	|元数据对象, 固定值就写metadata|
|metadata.name	|String	|元数据对象的名字, 比如命名Pod的名字|
|metadata.namespace	|String	|元数据对象的命名空间|
|Spec	|Object	|详细定义对象, 固定值就写Spec|
|spec.containers[]	|list	|Spec对象的容器列表定义, 是个列表|
|spec.containers[].name	|String	|容器的名字|
|spec.containers[].image	|String	|使用到的镜像名称|
**主要对象**

```yaml
apiVersion: v1                  #必选，版本号，例如v1，可以用 kubectl api-versions 查询到
kind: Pod                       #必选，指yaml文件定义的k8s 资源类型或角色，比如：Pod
metadata:                       #必选，元数据对象
  name: string                  #必选，元数据对象的名字，自己定义，比如命名Pod的名字
  namespace: string             #必选，元数据对象的名称空间，默认为"default"
  labels:                       #自定义标签
    key1: value1      　        #自定义标签键值对1
    key2: value2      　        #自定义标签键值对2
  annotations:                  #自定义注解
    key1: value1      　        #自定义注解键值对1
    key2: value2      　        #自定义注解键值对2
spec:                           #必选，对象【如pod】的详细定义
  containers:                   #必选，spec对象的容器信息
  - name: string                #必选，容器名称
    image: string               #必选，要用到的镜像名称
    imagePullPolicy: [Always|Never|IfNotPresent]  #获取镜像的策略；(1)Always：意思是每次都尝试重新拉取镜像；(2)Never：表示仅使用本地镜像，即使本地没有镜像也不拉取；(3) IfNotPresent：如果本地有镜像就使用本地镜像，没有就拉取远程镜像。默认：Always
    command: [string]           #指定容器启动命令，由于是数组因此可以指定多个。不指定则使用镜像打包时指定的启动命令。
    args: [string]              #指定容器启动命令参数，由于是数组因此可以指定多个
    workingDir: string          #指定容器的工作目录
    volumeMounts:               #指定容器内部的存储卷配置
    - name: string              #指定可以被容器挂载的存储卷的名称。跟下面volume字段的name值相同表示使用这个存储卷
      mountPath: string         #指定可以被容器挂载的存储卷的路径，应少于512字符
      readOnly: boolean         #设置存储卷路径的读写模式，true或者false，默认为读写模式false
    ports:                      #需要暴露的端口号列表
    - name: string              #端口的名称
      containerPort: int        #容器监听的端口号
      #除非绝对必要，否则不要为 Pod 指定 hostPort。将 Pod 绑定到hostPort时，它会限制 Pod 可以调度的位置数
      #DaemonSet 中的 Pod 可以使用 hostPort，从而可以通过节点 IP 访问到 Pod；因为DaemonSet模式下Pod不会被调度到其他节点。
      #一般情况下 containerPort与hostPort值相同
      hostPort: int     #可以通过宿主机+hostPort的方式访问该Pod。例如：pod在/调度到了k8s-node02【172.16.1.112】，hostPort为8090，那么该Pod可以通过172.16.1.112:8090方式进行访问。
      protocol: string          #端口协议，支持TCP和UDP，默认TCP
    env:                        #容器运行前需设置的环境变量列表
    - name: string              #环境变量名称
      value: string             #环境变量的值
    resources:                  #资源限制和资源请求的设置（设置容器的资源上线）
      limits:                   #容器运行时资源使用的上线
        cpu: string             #CPU限制，单位为core数，允许浮点数，如0.1等价于100m，0.5等价于500m；因此如果小于1那么优先选择如100m的形式，精度为1m。这个数字用作 docker run 命令中的 --cpu-quota 参数。
        memory: string          #内存限制，单位：E,P,T,G,M,K；或者Ei,Pi,Ti,Gi,Mi,Ki；或者字节数。将用于docker run --memory参数
      requests:                 #容器启动和调度时的限制设定
        cpu: string             #CPU请求，容器启动时初始化可用数量，单位为core数，允许浮点数，如0.1等价于100m，0.5等价于500m；因此如果小于1那么优先选择如100m的形式，精度为1m。这个数字用作 docker run 命令中的 --cpu-shares 参数。
        memory: string          #内存请求,容器启动的初始化可用数量。单位：E,P,T,G,M,K；或者Ei,Pi,Ti,Gi,Mi,Ki；或者字节数
    # 参见官网地址：https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
    livenessProbe:      　　    #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器【只需设置其中一种方法即可】
      exec:       　　　　　　  #对Pod内容器健康检查方式设置为exec方式
        command: [string]       #exec方式需要制定的命令或脚本
      httpGet:        　　　　  #对Pod内容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string            #访问 HTTP 服务的路径
        port: number            #访问容器的端口号或者端口名。如果数字必须在 1 ~ 65535 之间。
        host: string            #当没有定义 "host" 时，使用 "PodIP"
        scheme: string          #当没有定义 "scheme" 时，使用 "HTTP"，scheme 只允许 "HTTP" 和 "HTTPS"
        HttpHeaders:            #请求中自定义的 HTTP 头。HTTP 头字段允许重复。
        - name: string
          value: string
      tcpSocket:      　　　　  #对Pod内容器健康检查方式设置为tcpSocket方式
         port: number
      initialDelaySeconds: 5    #容器启动完成后，kubelet在执行第一次探测前应该等待 5 秒。默认是 0 秒，最小值是 0。
      periodSeconds: 60    　　 #指定 kubelet 每隔 60 秒执行一次存活探测。默认是 10 秒。最小值是 1
      timeoutSeconds: 3    　　 #对容器健康检查探测等待响应的超时时间为 3 秒，默认1秒
      successThreshold: 1       #检测到有1次成功则认为服务是`就绪`
      failureThreshold: 5       #检测到有5次失败则认为服务是`未就绪`。默认值是 3，最小值是 1。
    restartPolicy: [Always|Never|OnFailure] #Pod的重启策略，默认Always。Always表示一旦不管以何种方式终止运行，kubelet都将重启；OnFailure表示只有Pod以非0退出码退出才重启；Nerver表示不再重启该Pod
    nodeSelector:           　　#定义Node的label过滤标签，以key：value的格式指定。节点选择，先给主机打标签kubectl label nodes kube-node01 key1=value1 
      key1: value1
    imagePullSecrets:     　　  #Pull镜像时使用的secret名称，以name：secretKeyName格式指定
    - name: string
    hostNetwork: false      　　#是否使用主机网络模式，默认为false。如果设置为true，表示使用宿主机网络，不使用docker网桥
  # volumes 和 containers 是同层级 ******************************
  # 参见官网地址：https://kubernetes.io/zh/docs/concepts/storage/volumes/
  volumes:        　　　　　　#定义了paues容器关联的宿主机或分布式文件系统存储卷列表 （volumes类型有很多种，选其中一种即可）
  - name: string     　　 　　#共享存储卷名称。
    emptyDir: {}      　　    #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。当Pod因为某些原因被从节点上删除时，emptyDir卷中的数据也会永久删除。
    hostPath: string      　　#类型为hostPath的存储卷，表示挂载Pod所在宿主机的文件或目录
      path: string      　　  #在宿主机上文件或目录的路径
      type: [|DirectoryOrCreate|Directory|FileOrCreate|File] #空字符串（默认）用于向后兼容，这意味着在安装 hostPath 卷之前不会执行任何检查。DirectoryOrCreate：如果给定目录不存在则创建，权限设置为 0755，具有与 Kubelet 相同的组和所有权。Directory：给定目录必须存在。FileOrCreate：如果给定文件不存在，则创建空文件，权限设置为 0644，具有与 Kubelet 相同的组和所有权。File：给定文件必须存在。
    secret:       　　　　　  #类型为secret的存储卷，挂载集群预定义的secre对象到容器内部。Secret 是一种包含少量敏感信息例如密码、token 或 key 的对象。放在一个 secret 对象中可以更好地控制它的用途，并降低意外暴露的风险。
      secretName: string      #secret 对象的名字
      items:                  #可选，修改key 的目标路径
      - key: username         #username secret存储在/etc/foo/my-group/my-username 文件中而不是 /etc/foo/username 中。【此时存在spec.containers[].volumeMounts[].mountPath为/etc/foo】
        path: my-group/my-username
    configMap:      　　　　  #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部。ConfigMap 允许您将配置文件与镜像文件分离，以使容器化的应用程序具有可移植性。
      name: string             #提供你想要挂载的 ConfigMap 的名字
```

> kubectl explain pod命令可以查看资源的模板
>
> kubectl explain pod.apiVersion 查看详细字段

