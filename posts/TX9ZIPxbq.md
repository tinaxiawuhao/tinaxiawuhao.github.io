---
title: 'k8s-ingress安装使用'
date: 2022-06-03 10:00:45
tags: [k8s]
published: true
hideInList: false
feature: /post-images/TX9ZIPxbq.png
isTop: false
---
## 1.Nginx ingress安装

首先需要安装Nginx Ingress Controller控制器，控制器安装方式包含两种：DaemonSets和Deployments。

- DaemonSets通过hostPort的方式暴露80和443端口，可通过Node的调度由专门的节点实现部署
- Deployments则通过NodePort的方式实现控制器端口的暴露，借助外部负载均衡实现高可用负载均衡

除此之外，还需要部署Namespace，ServiceAccount，RBAC，Secrets，Custom Resource Definitions等资源，如下开始部署。

### 1.1 基础依赖环境准备

**1、github中下载源码包,安装部署文件在kubernetes-ingress/deployments/目录下**

```shell
git clone https://github.com/nginxinc/kubernetes-ingress.git
```

```js
kubernetes-ingress/deployments/
├── common
│   ├── custom-resource-definitions.yaml  自定义资源
│   ├── default-server-secret.yaml        Secrets
│   ├── nginx-config.yaml
│   └── ns-and-sa.yaml                    Namspace+ServiceAccount
├── daemon-set
│   ├── nginx-ingress.yaml                DaemonSets控制器
│   └── nginx-plus-ingress.yaml
├── deployment
│   ├── nginx-ingress.yaml                Deployments控制器
│   └── nginx-plus-ingress.yaml
├── helm-chart                            Helm安装包
│   ├── chart-icon.png
│   ├── Chart.yaml
│   ├── README.md
│   ├── templates
│   │   ├── controller-configmap.yaml
│   │   ├── controller-custom-resources.yaml
│   │   ├── controller-daemonset.yaml
│   │   ├── controller-deployment.yaml
│   │   ├── controller-leader-election-configmap.yaml
│   │   ├── controller-secret.yaml
│   │   ├── controller-serviceaccount.yaml
│   │   ├── controller-service.yaml
│   │   ├── controller-wildcard-secret.yaml
│   │   ├── _helpers.tpl
│   │   ├── NOTES.txt
│   │   └── rbac.yaml
│   ├── values-icp.yaml
│   ├── values-plus.yaml
│   └── values.yaml
├── rbac                                RBAC认证授权
│   └── rbac.yaml
├── README.md
└── service                            Service定义
    ├── loadbalancer-aws-elb.yaml
    ├── loadbalancer.yaml              DaemonSets暴露服务方式
    └── nodeport.yaml                  Deployments暴露服务方式
```

**2、创建Namespace和ServiceAccount **

```yaml
cat common/ns-and-sa.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: nginx-ingress 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress 
  namespace: nginx-ingress
```

```shell
kubectl apply -f common/ns-and-sa.yaml
```

**3、创建Secrets自签名证书**

```
kubectl apply -f common/default-server-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-server-secret
  namespace: nginx-ingress
type: Opaque
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN2akNDQWFZQ0NRREFPRjl0THNhWFhEQU5CZ2txaGtpRzl3MEJBUXNGQURBaE1SOHdIUVlEVlFRRERCWk8KUjBsT1dFbHVaM0psYzNORGIyNTBjbTlzYkdWeU1CNFhEVEU0TURreE1qRTRNRE16TlZvWERUSXpNRGt4TVRFNApNRE16TlZvd0lURWZNQjBHQTFVRUF3d1dUa2RKVGxoSmJtZHlaWE56UTI5dWRISnZiR3hsY2pDQ0FTSXdEUVlKCktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUwvN2hIUEtFWGRMdjNyaUM3QlBrMTNpWkt5eTlyQ08KR2xZUXYyK2EzUDF0azIrS3YwVGF5aGRCbDRrcnNUcTZzZm8vWUk1Y2Vhbkw4WGM3U1pyQkVRYm9EN2REbWs1Qgo4eDZLS2xHWU5IWlg0Rm5UZ0VPaStlM2ptTFFxRlBSY1kzVnNPazFFeUZBL0JnWlJVbkNHZUtGeERSN0tQdGhyCmtqSXVuektURXUyaDU4Tlp0S21ScUJHdDEwcTNRYzhZT3ExM2FnbmovUWRjc0ZYYTJnMjB1K1lYZDdoZ3krZksKWk4vVUkxQUQ0YzZyM1lma1ZWUmVHd1lxQVp1WXN2V0RKbW1GNWRwdEMzN011cDBPRUxVTExSakZJOTZXNXIwSAo1TmdPc25NWFJNV1hYVlpiNWRxT3R0SmRtS3FhZ25TZ1JQQVpQN2MwQjFQU2FqYzZjNGZRVXpNQ0F3RUFBVEFOCkJna3Foa2lHOXcwQkFRc0ZBQU9DQVFFQWpLb2tRdGRPcEsrTzhibWVPc3lySmdJSXJycVFVY2ZOUitjb0hZVUoKdGhrYnhITFMzR3VBTWI5dm15VExPY2xxeC9aYzJPblEwMEJCLzlTb0swcitFZ1U2UlVrRWtWcitTTFA3NTdUWgozZWI4dmdPdEduMS9ienM3bzNBaS9kclkrcUI5Q2k1S3lPc3FHTG1US2xFaUtOYkcyR1ZyTWxjS0ZYQU80YTY3Cklnc1hzYktNbTQwV1U3cG9mcGltU1ZmaXFSdkV5YmN3N0NYODF6cFErUyt1eHRYK2VBZ3V0NHh3VlI5d2IyVXYKelhuZk9HbWhWNThDd1dIQnNKa0kxNXhaa2VUWXdSN0diaEFMSkZUUkk3dkhvQXprTWIzbjAxQjQyWjNrN3RXNQpJUDFmTlpIOFUvOWxiUHNoT21FRFZkdjF5ZytVRVJxbStGSis2R0oxeFJGcGZnPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdi91RWM4b1JkMHUvZXVJTHNFK1RYZUprckxMMnNJNGFWaEMvYjVyYy9XMlRiNHEvClJOcktGMEdYaVN1eE9ycXgrajlnamx4NXFjdnhkenRKbXNFUkJ1Z1B0ME9hVGtIekhvb3FVWmcwZGxmZ1dkT0EKUTZMNTdlT1l0Q29VOUZ4amRXdzZUVVRJVUQ4R0JsRlNjSVo0b1hFTkhzbysyR3VTTWk2Zk1wTVM3YUhudzFtMApxWkdvRWEzWFNyZEJ6eGc2clhkcUNlUDlCMXl3VmRyYURiUzc1aGQzdUdETDU4cGszOVFqVUFQaHpxdmRoK1JWClZGNGJCaW9CbTVpeTlZTW1hWVhsMm0wTGZzeTZuUTRRdFFzdEdNVWozcGJtdlFmazJBNnljeGRFeFpkZFZsdmwKMm82MjBsMllxcHFDZEtCRThCay90elFIVTlKcU56cHpoOUJUTXdJREFRQUJBb0lCQVFDZklHbXowOHhRVmorNwpLZnZJUXQwQ0YzR2MxNld6eDhVNml4MHg4Mm15d1kxUUNlL3BzWE9LZlRxT1h1SENyUlp5TnUvZ2IvUUQ4bUFOCmxOMjRZTWl0TWRJODg5TEZoTkp3QU5OODJDeTczckM5bzVvUDlkazAvYzRIbjAzSkVYNzZ5QjgzQm9rR1FvYksKMjhMNk0rdHUzUmFqNjd6Vmc2d2szaEhrU0pXSzBwV1YrSjdrUkRWYmhDYUZhNk5nMUZNRWxhTlozVDhhUUtyQgpDUDNDeEFTdjYxWTk5TEI4KzNXWVFIK3NYaTVGM01pYVNBZ1BkQUk3WEh1dXFET1lvMU5PL0JoSGt1aVg2QnRtCnorNTZud2pZMy8yUytSRmNBc3JMTnIwMDJZZi9oY0IraVlDNzVWYmcydVd6WTY3TWdOTGQ5VW9RU3BDRkYrVm4KM0cyUnhybnhBb0dCQU40U3M0ZVlPU2huMVpQQjdhTUZsY0k2RHR2S2ErTGZTTXFyY2pOZjJlSEpZNnhubmxKdgpGenpGL2RiVWVTbWxSekR0WkdlcXZXaHFISy9iTjIyeWJhOU1WMDlRQ0JFTk5jNmtWajJTVHpUWkJVbEx4QzYrCk93Z0wyZHhKendWelU0VC84ajdHalRUN05BZVpFS2FvRHFyRG5BYWkyaW5oZU1JVWZHRXFGKzJyQW9HQkFOMVAKK0tZL0lsS3RWRzRKSklQNzBjUis3RmpyeXJpY05iWCtQVzUvOXFHaWxnY2grZ3l4b25BWlBpd2NpeDN3QVpGdwpaZC96ZFB2aTBkWEppc1BSZjRMazg5b2pCUmpiRmRmc2l5UmJYbyt3TFU4NUhRU2NGMnN5aUFPaTVBRHdVU0FkCm45YWFweUNweEFkREtERHdObit3ZFhtaTZ0OHRpSFRkK3RoVDhkaVpBb0dCQUt6Wis1bG9OOTBtYlF4VVh5YUwKMjFSUm9tMGJjcndsTmVCaWNFSmlzaEhYa2xpSVVxZ3hSZklNM2hhUVRUcklKZENFaHFsV01aV0xPb2I2NTNyZgo3aFlMSXM1ZUtka3o0aFRVdnpldm9TMHVXcm9CV2xOVHlGanIrSWhKZnZUc0hpOGdsU3FkbXgySkJhZUFVWUNXCndNdlQ4NmNLclNyNkQrZG8wS05FZzFsL0FvR0FlMkFVdHVFbFNqLzBmRzgrV3hHc1RFV1JqclRNUzRSUjhRWXQKeXdjdFA4aDZxTGxKUTRCWGxQU05rMXZLTmtOUkxIb2pZT2pCQTViYjhibXNVU1BlV09NNENoaFJ4QnlHbmR2eAphYkJDRkFwY0IvbEg4d1R0alVZYlN5T294ZGt5OEp0ek90ajJhS0FiZHd6NlArWDZDODhjZmxYVFo5MWpYL3RMCjF3TmRKS2tDZ1lCbyt0UzB5TzJ2SWFmK2UwSkN5TGhzVDQ5cTN3Zis2QWVqWGx2WDJ1VnRYejN5QTZnbXo5aCsKcDNlK2JMRUxwb3B0WFhNdUFRR0xhUkcrYlNNcjR5dERYbE5ZSndUeThXczNKY3dlSTdqZVp2b0ZpbmNvVlVIMwphdmxoTUVCRGYxSjltSDB5cDBwWUNaS2ROdHNvZEZtQktzVEtQMjJhTmtsVVhCS3gyZzR6cFE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

**4、创建ConfigMap自定义配置文**

```shell
kubectl apply -f common/nginx-config.yaml 
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
```

**5、为主机和主机路由定义自定义资源，支持自定义主机和路由**

```shell
kubectl apply -f common/custom-resource-definitions.yaml
```

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: virtualservers.k8s.nginx.org
spec:
  group: k8s.nginx.org
  versions:
  - name: v1
    served: true
    storage: true
  scope: Namespaced
  names:
    plural: virtualservers
    singular: virtualserver
    kind: VirtualServer
    shortNames:
    - vs
---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: virtualserverroutes.k8s.nginx.org
spec:
  group: k8s.nginx.org
  versions:
  - name: v1
    served: true
    storage: true
  scope: Namespaced
  names:
    plural: virtualserverroutes
    singular: virtualserverroute
    kind: VirtualServerRoute
    shortNames:
    - vsr
```

**6、配置RBAC认证授权，实现ingress控制器访问集群中的其他资源**

```shell
kubectl apply -f rbac/rbac.yaml
```

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-ingress
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
  - update
  - create
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - k8s.nginx.org
  resources:
  - virtualservers
  - virtualserverroutes
  verbs:
  - list
  - watch
  - get
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-ingress
subjects:
- kind: ServiceAccount
  name: nginx-ingress
  namespace: nginx-ingress
roleRef:
  kind: ClusterRole
  name: nginx-ingress
  apiGroup: rbac.authorization.k8s.io
```

### 1.2 部署Ingress控制器

**1、 部署控制器，控制器可以DaemonSets和Deployment的形式部署，如下是DaemonSets的配置文件**

```js
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
     #annotations:
       #prometheus.io/scrape: "true"
       #prometheus.io/port: "9113"
    spec:
      serviceAccountName: nginx-ingress
      containers:
      - image: nginx/nginx-ingress:edge
        imagePullPolicy: Always
        name: nginx-ingress
        ports:
        - name: http
          containerPort: 80
          hostPort: 80            #通过hostPort的方式暴露端口
        - name: https
          containerPort: 443
          hostPort: 443
       #- name: prometheus
         #containerPort: 9113
        securityContext:
          allowPrivilegeEscalation: true
          runAsUser: 101 #nginx
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
          - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
         #- -v=3 # Enables extensive logging. Useful for troubleshooting.
         #- -report-ingress-status
         #- -external-service=nginx-ingress
         #- -enable-leader-election
         #- -enable-prometheus-metrics
```

**Deployments的配置文件**

```js
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  replicas: 1                  #副本的个数
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
     #annotations:
       #prometheus.io/scrape: "true"
       #prometheus.io/port: "9113"
    spec:
      serviceAccountName: nginx-ingress
      containers:
      - image: nginx/nginx-ingress:edge
        imagePullPolicy: Always
        name: nginx-ingress
        ports:                #内部暴露的服务端口，需要通过NodePort的方式暴露给外部
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
       #- name: prometheus
         #containerPort: 9113
        securityContext:
          allowPrivilegeEscalation: true
          runAsUser: 101 #nginx
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
          - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
         #- -v=3 # Enables extensive logging. Useful for troubleshooting.
         #- -report-ingress-status
         #- -external-service=nginx-ingress
         #- -enable-leader-election
         #- -enable-prometheus-metrics
```

**2、我们以DaemonSets的方式部署，DaemonSet部署集群中各个节点都是对等，如果有外部LoadBalancer则通过外部负载均衡路由至Ingress中**

```shell
[root@node-1 deployments]# kubectl apply -f daemon-set/nginx-ingress.yaml 
daemonset.apps/nginx-ingress created
[root@node-1 deployments]# kubectl get daemonsets -n nginx-ingress
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-ingress   3         3         3       3            3           <none>          15s

[root@node-1 ~]# kubectl get pods -n nginx-ingress -o wide 
NAME                  READY   STATUS    RESTARTS   AGE     IP             NODE     NOMINATED NODE   READINESS GATES
nginx-ingress-7mpfc   1/1     Running   0          2m44s   10.244.0.50    node-1   <none>           <none>
nginx-ingress-l2rtj   1/1     Running   0          2m44s   10.244.1.144   node-2   <none>           <none>
nginx-ingress-tgf6r   1/1     Running   0          2m44s   10.244.2.160   node-3   <none>           <none>
```

**3、校验Nginx Ingress安装情况**

此时三个节点均是对等，即访问任意一个节点均能实现相同的效果，统一入口则通过外部负载均衡，如果在云环境下执行`kubectl apply -f service/loadbalancer.yaml`创建外部负载均衡实现入口调度，自建的可以通过lvs或nginx等负载均衡实现接入。

![img](https://ask.qcloudimg.com/draft/4405893/88fb7v82cj.png?imageView2/2/w/1620)

**备注说明**：如果以Deployments的方式部署，则需要执行service/nodeport.yaml创建NodePort类型的Service，实现的效果和DaemonSets类似。

## 2. Ingress资源定义

上面的已安装了一个Nginx Ingress Controller控制器，有了Ingress控制器后，我们就可以定义Ingress资源来实现七层负载转发了，大体上Ingress支持三种使用方式：1. 基于虚拟主机转发，2. 基于虚拟机主机URI转发，3. 支持TLS加密转发。

### 2.1 Ingress定义

**1、环境准备，先创建一个nginx的Deployment应用，包含2个副本**

```shell
[root@node-1 ~]# kubectl run ingress-demo --image=nginx:1.7.9 --port=80 --replicas=2
[root@node-1 ~]# kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
ingress-demo   2/2     2            2           116s
```

**2、以service方式暴露服务端口**

```shell
[root@node-1 ~]# kubectl expose deployment ingress-demo --port=80 --protocol=TCP --target-port=80
service/ingress-demo exposed
[root@node-1 ~]# kubectl get services 
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
ingress-demo   ClusterIP   10.109.33.91   <none>        80/TCP    2m15s
```

**3、上述两个步骤已创建了一个service，如下我们定义一个ingress对象将流量转发至ingress-demo这个service，通过ingress.class指定控制器的类型为nginx**

```yaml
[root@node-1 nginx-ingress]# cat nginx-ingress-demo.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
spec:
  rules:
  - host: www.test.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-demo
          servicePort: 80
```

**4、创建ingress对象**

```shell
[root@node-1 nginx-ingress]# kubectl apply -f nginx-ingress-demo.yaml 
ingress.extensions/nginx-ingress-demo created

查看ingress资源列表
[root@node-1 nginx-ingress]# kubectl get ingresses
NAME                 HOSTS                ADDRESS   PORTS   AGE
nginx-ingress-demo   www.test.cn             80      4m4s
```

**5、查看ingress详情，可以在Rules规则中看到后端Pod的列表，自动发现和关联相关Pod**

```shell
[root@node-1 ~]# kubectl describe ingresses nginx-ingress-demo 
Name:             nginx-ingress-demo
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                Path  Backends
  ----                ----  --------
  www.test.cn  
                      /   ingress-demo:80 (10.244.1.146:80,10.244.2.162:80)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernets.io/ingress.class":"nginx"},"labels":{"ingres-controller":"nginx"},"name":"nginx-ingress-demo","namespace":"default"},"spec":{"rules":[{"host":"www.happylaulab.cn","http":{"paths":[{"backend":{"serviceName":"ingress-demo","servicePort":80},"path":"/"}]}}]}}

  kubernets.io/ingress.class:  nginx
Events:
  Type    Reason          Age   From                      Message
  ----    ------          ----  ----                      -------
  Normal  AddedOrUpdated  9m7s  nginx-ingress-controller  Configuration for default/nginx-ingress-demo was added or updated
  Normal  AddedOrUpdated  9m7s  nginx-ingress-controller  Configuration for default/nginx-ingress-demo was added or updated
  Normal  AddedOrUpdated  9m7s  nginx-ingress-controller  Configuration for default/nginx-ingress-demo was added or updated
```



**6、测试验证**

ingress规则的配置信息已注入到Ingress Controller中，环境中Ingress Controller是以DaemonSets的方式部署在集群中，如果有外部的负载均衡，则将www.test.cn域名的地址解析为负载均衡VIP。由于测试环境没有搭建负载均衡，将hosts解析执行node-1，node-2或者node-3任意一个IP都能实现相同的功能。

![img](https://ask.qcloudimg.com/draft/4405893/47noa95oaa.png?imageView2/2/w/1620)

上述测试解析正常，当然也可以解析为node-1和node-2的IP，如下：

```shell
[root@node-1 ~]# curl -I http://www.happylau.cn --resolve www.happylau.cn:80:10.254.100.101
HTTP/1.1 200 OK
Server: nginx/1.17.6
Date: Tue, 24 Dec 2019 10:32:22 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
ETag: "54999765-264"
Accept-Ranges: bytes

[root@node-1 ~]# curl -I http://www.happylau.cn --resolve www.happylau.cn:80:10.254.100.102
HTTP/1.1 200 OK
Server: nginx/1.17.6
Date: Tue, 24 Dec 2019 10:32:24 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
ETag: "54999765-264"
Accept-Ranges: bytes
```

## 3 Ingress动态配置

上面介绍了ingress资源对象的申明配置，这里我们探究一下Nginx Ingress Controller的实现机制和动态配置更新机制，以方便了解Ingress控制器的工作机制。

**1、 查看Nginx Controller控制器的配置文件，在nginx-ingress pod中存储着ingress的配置文件**

```shell
[root@node-1 ~]# kubectl get pods -n nginx-ingress 
NAME                  READY   STATUS    RESTARTS   AGE
nginx-ingress-7mpfc   1/1     Running   0          6h15m
nginx-ingress-l2rtj   1/1     Running   0          6h15m
nginx-ingress-tgf6r   1/1     Running   0          6h15m

#查看配置文件，每个ingress生成一个配置文件，文件名为：命名空间-ingres名称.conf
[root@node-1 ~]# kubectl exec -it nginx-ingress-7mpfc -n nginx-ingress -- ls -l /etc/nginx/conf.d
total 4
-rw-r--r-- 1 nginx nginx 1005 Dec 24 10:06 default-nginx-ingress-demo.conf

#查看配置文件
[root@node-1 ~]# kubectl exec -it nginx-ingress-7mpfc -n nginx-ingress -- cat /etc/nginx/conf.d/default-nginx-ingress-demo.conf
# configuration for default/nginx-ingress-demo

#upstream的配置，会用least_conn算法，通过service服务发现机制动态识别到后端的Pod
upstream default-nginx-ingress-demo-www.test.cn-ingress-demo-80 {
	zone default-nginx-ingress-demo-www.test.cn-ingress-demo-80 256k;
	random two least_conn;
	server 10.244.1.146:80 max_fails=1 fail_timeout=10s max_conns=0;
	server 10.244.2.162:80 max_fails=1 fail_timeout=10s max_conns=0;
}

server {
	listen 80;
	server_tokens on;
	server_name www.test.cn;
	location / {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		proxy_pass http://default-nginx-ingress-demo-www.test.cn-ingress-demo-80;	#调用upstream实现代理
	}
}
```

通过上述查看配置文件可得知，Nginx Ingress Controller实际是根据ingress规则生成对应的nginx配置文件，以实现代理转发的功能，加入Deployments的副本数变更后nginx的配置文件会发生什么改变呢？

**2、更新控制器的副本数，由2个Pod副本扩容至3个**

```js
[root@node-1 ~]# kubectl scale --replicas=3 deployment ingress-demo 
deployment.extensions/ingress-demo scaled
[root@node-1 ~]# kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
ingress-demo   3/3     3            3           123m
```

**3、再次查看nginx的配置文件，ingress借助于service的服务发现机制，将加入的Pod自动加入到nginx upstream中**

![img](https://ask.qcloudimg.com/draft/4405893/14725qbjdh.png?imageView2/2/w/1620)

**4、查看nginx pod的日志（kubectl logs nginx-ingress-7mpfc -n nginx-ingress），有reload优雅重启的记录，即通过更新配置文件+reload实现配置动态更新。**

![img](https://ask.qcloudimg.com/draft/4405893/lilxwe1u13.png?imageView2/2/w/1620)

通过上述的配置可知，ingress调用kubernetes api去感知kubernetes集群中的变化情况，Pod的增加或减少这些变化，然后动态更新nginx ingress controller的配置文件，并重新载入配置。当集群规模越大时，会频繁涉及到配置文件的变动和重载，因此nginx这方面会存在先天的劣势，专门为微服务负载均衡应运而生，如Traefik，Envoy，Istio，这些负载均衡工具能够提供大规模，频繁动态更新的场景，但性能相比Nginx，HAproxy还存在一定的劣势。

## 4 Ingress路径转发

Ingress支持URI格式的转发方式，同时支持URL重写，如下以两个service为例演示，service-1安装nginx，service-2安装httpd，分别用http://demo.happylau.cn/news和http://demo.happylau.cn/sports转发到两个不同的service

**1、环境准备，创建两个应用并实现service暴露，创建deployments时指定--explose创建service**

```js
[root@node-1 ~]# kubectl run service-1 --image=nginx:1.7.9 --port=80 --replicas=1 --expose=true 
service/service-1 created
deployment.apps/service-1 created

[root@node-1 ~]# kubectl run service-2 --image=httpd --port=80 --replicas=1 --expose=true 
service/service-2 created
deployment.apps/service-2 created

查看deployment状态
[root@node-1 ~]# kubectl get deployments 
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
ingress-demo   4/4     4            4           4h36m
service-1      1/1     1            1           65s
service-2      1/1     1            1           52s

查看service状态，服务已经正常
[root@node-1 ~]# kubectl get services 
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
ingress-demo   ClusterIP   10.109.33.91     <none>        80/TCP    4h36m
kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP   101d
service-1      ClusterIP   10.106.245.71    <none>        80/TCP    68s
service-2      ClusterIP   10.104.204.158   <none>        80/TCP    55s
```

**2、创建ingress对象，通过一个域名将请求转发至后端两个service**

```js
[root@node-1 nginx-ingress]# cat nginx-ingress-uri-demo.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-uri-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: 
    http:
      paths:
      - path: /news
        backend:
          serviceName: service-1 
          servicePort: 80
      - path: /sports
        backend:
          serviceName: service-2
          servicePort: 80
```

**3、创建ingress规则，查看详情**

```js
[root@node-1 nginx-ingress]# kubectl apply -f nginx-ingress-uri-demo.yaml 
ingress.extensions/nginx-ingress-uri-demo created

#查看详情
[root@node-1 nginx-ingress]# kubectl get ingresses.
NAME                     HOSTS              ADDRESS   PORTS   AGE
nginx-ingress-demo       www.happylau.cn              80      4h35m
nginx-ingress-uri-demo   demo.happylau.cn             80      4s
[root@node-1 nginx-ingress]# kubectl describe ingresses nginx-ingress-uri-demo 
Name:             nginx-ingress-uri-demo
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:              #对应的转发url规则
  Host              Path  Backends
  ----              ----  --------
  demo.happylau.cn  
                    /news     service-1:80 (10.244.2.163:80)
                    /sports   service-2:80 (10.244.1.148:80)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernets.io/ingress.class":"nginx","nginx.ingress.kubernetes.io/rewrite-target":"/"},"labels":{"ingres-controller":"nginx"},"name":"nginx-ingress-uri-demo","namespace":"default"},"spec":{"rules":[{"host":"demo.happylau.cn","http":{"paths":[{"backend":{"serviceName":"service-1","servicePort":80},"path":"/news"},{"backend":{"serviceName":"service-2","servicePort":80},"path":"/sports"}]}}]}}

  kubernets.io/ingress.class:                  nginx
  nginx.ingress.kubernetes.io/rewrite-target:  /
Events:
  Type    Reason          Age   From                      Message
  ----    ------          ----  ----                      -------
  Normal  AddedOrUpdated  11s   nginx-ingress-controller  Configuration for default/nginx-ingress-uri-demo was added or updated
  Normal  AddedOrUpdated  11s   nginx-ingress-controller  Configuration for default/nginx-ingress-uri-demo was added or updated
  Normal  AddedOrUpdated  11s   nginx-ingress-controller  Configuration for default/nginx-ingress-uri-demo was added or updated
```

**4、准备测试，站点中创建对应的路径**

```js
[root@node-1 ~]# kubectl exec -it service-1-7b66bf758f-xj9jh /bin/bash
root@service-1-7b66bf758f-xj9jh:/# echo "service-1 website page" >/usr/share/nginx/html/news

[root@node-1 ~]# kubectl exec -it service-2-7c7444684d-w9cv9 /bin/bash
root@service-2-7c7444684d-w9cv9:/usr/local/apache2# echo "service-2 website page" >/usr/local/apache2/htdocs/sports
```

**5、测试验证**

```js
[root@node-1 ~]# curl http://demo.happylau.cn/news --resolve demo.happylau.cn:80:10.254.100.101
service-1 website page
[root@node-1 ~]# curl http://demo.happylau.cn/sports --resolve demo.happylau.cn:80:10.254.100.101
service-2 website page
```

**6、总结**

通过上述的验证测试可以得知，ingress支持URI的路由方式转发，其对应在ingress中的配置文件内容是怎样的呢，我们看下ingress controller生成对应的nginx配置文件内容，实际是通过ingress的location来实现，将不同的localtion转发至不同的upstream以实现service的关联，配置文件如下：

```js
[root@node-1 ~]# kubectl exec -it nginx-ingress-7mpfc -n nginx-ingress /bin/bash
nginx@nginx-ingress-7mpfc:/$ cat /etc/nginx/conf.d/default-nginx-ingress-uri-demo.conf |grep -v "^$"
# configuration for default/nginx-ingress-uri-demo
#定义两个upstream和后端的service关联
upstream default-nginx-ingress-uri-demo-demo.happylau.cn-service-1-80 {
	zone default-nginx-ingress-uri-demo-demo.happylau.cn-service-1-80 256k;
	random two least_conn;
	server 10.244.2.163:80 max_fails=1 fail_timeout=10s max_conns=0;
}

upstream default-nginx-ingress-uri-demo-demo.happylau.cn-service-2-80 {
	zone default-nginx-ingress-uri-demo-demo.happylau.cn-service-2-80 256k;
	random two least_conn;
	server 10.244.1.148:80 max_fails=1 fail_timeout=10s max_conns=0;	
}

server {
	listen 80;
	server_tokens on;
	server_name demo.happylau.cn;
	
  #定义location实现代理，通过proxy_pass和后端的service关联
	location /news {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		proxy_pass http://default-nginx-ingress-uri-demo-demo.happylau.cn-service-1-80;
	}

	location /sports {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		proxy_pass http://default-nginx-ingress-uri-demo-demo.happylau.cn-service-2-80;	
	}	
}
```

## 5 Ingress虚拟主机

ingress支持基于名称的虚拟主机，实现单个IP多个域名转发的需求，通过请求头部携带主机名方式区分开，将上个章节的ingress删除，使用service-1和service-2两个service来做演示。

**1、创建ingress规则，通过主机名实现转发规则**

```js
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-virtualname-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
spec:
  rules:
  - host: news.happylau.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: service-1 
          servicePort: 80
  - host: sports.happylau.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: service-2 
          servicePort: 80
```

**2、生成ingress规则并查看详情，一个ingress对应两个HOSTS**

```js
[root@node-1 nginx-ingress]# kubectl apply -f nginx-ingress-virtualname.yaml 
ingress.extensions/nginx-ingress-virtualname-demo created

#查看列表
[root@node-1 nginx-ingress]# kubectl get ingresses nginx-ingress-virtualname-demo 
NAME                             HOSTS                                 ADDRESS   PORTS   AGE
nginx-ingress-virtualname-demo   news.happylau.cn,sports.happylau.cn             80      12s

#查看详情
[root@node-1 nginx-ingress]# kubectl describe ingresses nginx-ingress-virtualname-demo
Name:             nginx-ingress-virtualname-demo
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                Path  Backends
  ----                ----  --------
  news.happylau.cn    
                      /   service-1:80 (10.244.2.163:80)
  sports.happylau.cn  
                      /   service-2:80 (10.244.1.148:80)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernets.io/ingress.class":"nginx"},"labels":{"ingres-controller":"nginx"},"name":"nginx-ingress-virtualname-demo","namespace":"default"},"spec":{"rules":[{"host":"news.happylau.cn","http":{"paths":[{"backend":{"serviceName":"service-1","servicePort":80},"path":"/"}]}},{"host":"sports.happylau.cn","http":{"paths":[{"backend":{"serviceName":"service-2","servicePort":80},"path":"/"}]}}]}}

  kubernets.io/ingress.class:  nginx
Events:
  Type    Reason          Age   From                      Message
  ----    ------          ----  ----                      -------
  Normal  AddedOrUpdated  28s   nginx-ingress-controller  Configuration for default/nginx-ingress-virtualname-demo was added or updated
  Normal  AddedOrUpdated  28s   nginx-ingress-controller  Configuration for default/nginx-ingress-virtualname-demo was added or updated
  Normal  AddedOrUpdated  28s   nginx-ingress-controller  Configuration for default/nginx-ingress-virtualname-demo was added or updated
```

**3、准备测试数据并测试**

```js
[root@node-1 ~]# kubectl exec -it service-1-7b66bf758f-xj9jh /bin/bash
root@service-1-7b66bf758f-xj9jh:/# echo "news demo" >/usr/share/nginx/html/index.html

[root@node-1 ~]# kubectl exec -it service-2-7c7444684d-w9cv9 /bin/bash  
root@service-2-7c7444684d-w9cv9:/usr/local/apache2# echo "sports demo"  >/usr/local/apache2/htdocs/index.html
```

**测试：**

```js
[root@node-1 ~]# curl http://news.happylau.cn --resolve news.happylau.cn:80:10.254.100.102
news demo
[root@node-1 ~]# curl http://sports.happylau.cn --resolve sports.happylau.cn:80:10.254.100.102
sports demo
```

**4、查看nginx的配置文件内容，通过在server中定义不同的server_name以区分，代理到不同的upstream以实现service的代理。**

```js
# configuration for default/nginx-ingress-virtualname-demo
upstream default-nginx-ingress-virtualname-demo-news.happylau.cn-service-1-80 {
	zone default-nginx-ingress-virtualname-demo-news.happylau.cn-service-1-80 256k;
	random two least_conn;
	server 10.244.2.163:80 max_fails=1 fail_timeout=10s max_conns=0;
}

upstream default-nginx-ingress-virtualname-demo-sports.happylau.cn-service-2-80 {
	zone default-nginx-ingress-virtualname-demo-sports.happylau.cn-service-2-80 256k;
	random two least_conn;
	server 10.244.1.148:80 max_fails=1 fail_timeout=10s max_conns=0;
}

server {
	listen 80;
	server_tokens on;
	server_name news.happylau.cn;
	location / {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		
		proxy_pass http://default-nginx-ingress-virtualname-demo-news.happylau.cn-service-1-80;
	
  }
}
server {
	listen 80;	
	server_tokens on;
	server_name sports.happylau.cn;

	location / {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		
		proxy_pass http://default-nginx-ingress-virtualname-demo-sports.happylau.cn-service-2-80;	

	}	
}
```

## 6 Ingress TLS加密

四层的负载均衡无法支持https请求，当前大部分业务都要求以https方式接入，Ingress能支持https的方式接入，通过Secrets存储证书+私钥，实现https接入，同时还能支持http跳转功能。对于用户的请求流量来说，客户端到ingress controller是https流量，ingress controller到后端service则是http，提高用户访问性能，如下介绍ingress TLS功能实现步骤。

**1、生成自签名证书和私钥**

```js
[root@node-1 ~]# openssl req -x509 -newkey rsa:2048 -nodes -days 365 -keyout tls.key -out tls.crt
Generating a 2048 bit RSA private key
....................................................+++
........................................+++
writing new private key to 'tls.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN        #国家
State or Province Name (full name) []:GD    #省份
Locality Name (eg, city) [Default City]:ShenZhen  #城市
Organization Name (eg, company) [Default Company Ltd]:Tencent    #公司 
Organizational Unit Name (eg, section) []:HappyLau  #组织
Common Name (eg, your name or your server's hostname) []:www.happylau.cn  #域名
Email Address []:573302346@qq.com       #邮箱地址


#tls.crt为证书，tls.key为私钥
[root@node-1 ~]# ls tls.* -l
-rw-r--r-- 1 root root 1428 12月 26 13:21 tls.crt
-rw-r--r-- 1 root root 1708 12月 26 13:21 tls.key
```

**2、配置Secrets，将证书和私钥配置到Secrets中**

```js
[root@node-1 ~]# kubectl create secret tls happylau-sslkey --cert=tls.crt --key=tls.key 
secret/happylau-sslkey created

查看Secrets详情,证书和私要包含在data中，文件名为两个不同的key：tls.crt和tls.key
[root@node-1 ~]# kubectl describe secrets happylau-sslkey 
Name:         happylau-sslkey
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1428 bytes
tls.key:  1708 bytes
```

**3、配置ingress调用Secrets实现SSL证书加密**

```js
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-tls-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
spec:
  tls:
  - hosts:
    - news.happylau.cn
    - sports.happylau.cn
    secretName: happylau-sslkey
  rules:
  - host: news.happylau.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: service-1 
          servicePort: 80
  - host: sports.happylau.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: service-2 
          servicePort: 80
```

**4、创建ingress并查看ingress详情**

```js
[root@node-1 nginx-ingress]# kubectl describe ingresses nginx-ingress-tls-demo 
Name:             nginx-ingress-tls-demo
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<none>)
TLS:
  happylau-sslkey terminates news.happylau.cn,sports.happylau.cn
Rules:
  Host                Path  Backends
  ----                ----  --------
  news.happylau.cn    
                      /   service-1:80 (10.244.2.163:80)
  sports.happylau.cn  
                      /   service-2:80 (10.244.1.148:80)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernets.io/ingress.class":"nginx"},"labels":{"ingres-controller":"nginx"},"name":"nginx-ingress-tls-demo","namespace":"default"},"spec":{"rules":[{"host":"news.happylau.cn","http":{"paths":[{"backend":{"serviceName":"service-1","servicePort":80},"path":"/"}]}},{"host":"sports.happylau.cn","http":{"paths":[{"backend":{"serviceName":"service-2","servicePort":80},"path":"/"}]}}],"tls":[{"hosts":["news.happylau.cn","sports.happylau.cn"],"secretName":"happylau-sslkey"}]}}

  kubernets.io/ingress.class:  nginx
Events:
  Type    Reason          Age   From                      Message
  ----    ------          ----  ----                      -------
  Normal  AddedOrUpdated  22s   nginx-ingress-controller  Configuration for default/nginx-ingress-tls-demo was added or updated
  Normal  AddedOrUpdated  22s   nginx-ingress-controller  Configuration for default/nginx-ingress-tls-demo was added or updated
  Normal  AddedOrUpdated  22s   nginx-ingress-controller  Configuration for default/nginx-ingress-tls-demo was added or updated
```

**5、访问**

 将news.happylau.cn和sports.happylau.cn写入到hosts文件中，并通过[https://news.happylau.cn](https://news.happylau.cn/) 的方式访问，浏览器访问内容提示证书如下，信任证书即可访问到站点内容。

![img](https://ask.qcloudimg.com/draft/4405893/ijw044j6yx.png?imageView2/2/w/1620)

查看证书详情，正是我们制作的自签名证书，生产实际使用时，推荐使用CA机构颁发签名证书。

![img](https://ask.qcloudimg.com/draft/4405893/t3clag54i5.png?imageView2/2/w/1620)

**6、接下来查看一下tls配置https的nginx配置文件内容，可以看到在server块启用了https并配置证书，同时配置了http跳转，因此直接访问http也能够实现自动跳转到https功能。**

```shell
# configuration for default/nginx-ingress-tls-demo
upstream default-nginx-ingress-tls-demo-news.happylau.cn-service-1-80 {
	zone default-nginx-ingress-tls-demo-news.happylau.cn-service-1-80 256k;
	random two least_conn;
	server 10.244.2.163:80 max_fails=1 fail_timeout=10s max_conns=0;	
}

upstream default-nginx-ingress-tls-demo-sports.happylau.cn-service-2-80 {
	zone default-nginx-ingress-tls-demo-sports.happylau.cn-service-2-80 256k;
	random two least_conn;
	server 10.244.1.148:80 max_fails=1 fail_timeout=10s max_conns=0;	
}

server {
	listen 80;

	listen 443 ssl;     #https监听端口，证书和key，实现和Secrets关联
	ssl_certificate /etc/nginx/secrets/default-happylau-sslkey;
	ssl_certificate_key /etc/nginx/secrets/default-happylau-sslkey;

	server_tokens on;
	server_name news.happylau.cn;
	
  #http跳转功能，即访问http会自动跳转至https
	if ($scheme = http) {
		return 301 https://$host:443$request_uri;
	}
	
	location / {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		
		proxy_pass http://default-nginx-ingress-tls-demo-news.happylau.cn-service-1-80;	
  }	
}

server {
	listen 80;
	listen 443 ssl;
	ssl_certificate /etc/nginx/secrets/default-happylau-sslkey;
	ssl_certificate_key /etc/nginx/secrets/default-happylau-sslkey;

	server_tokens on;
	server_name sports.happylau.cn;

	if ($scheme = http) {
		return 301 https://$host:443$request_uri;
	}

	location / {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		proxy_send_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		
		proxy_pass http://default-nginx-ingress-tls-demo-sports.happylau.cn-service-2-80;	
	}	
}
```

## 7. Nginx Ingress高级功能

### 7.1 定制化参数

ingress controller提供了基础反向代理的功能，如果需要定制化nginx的特性或参数，需要通过ConfigMap和Annotations来实现，两者实现的方式有所不同，ConfigMap用于指定整个ingress集群资源的基本参数，修改后会被所有的ingress对象所继承；Annotations则被某个具体的ingress对象所使用，修改只会影响某个具体的ingress资源，冲突时其优先级高于ConfigMap。

#### 7.1.1 ConfigMap自定义参数

安装nginx ingress controller时默认会包含一个空的ConfigMap，可以通过ConfigMap来自定义nginx controller的默认参数，如下以修改一些参数为例：

**1、 定义ConfigMap参数**

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-config
  namespace: nginx-ingress
data:
  proxy-connect-timeout: "10s"
  proxy-read-timeout: "10s"
  proxy-send-timeout: "10"
  client-max-body-size: "3m"
```

**2、 应用配置并查看ConfigMap配置**

```shell
[root@node-1 ~]# kubectl get configmaps -n nginx-ingress nginx-config -o yaml
apiVersion: v1
data:
  client-max-body-size: 3m
  proxy-connect-timeout: 10s
  proxy-read-timeout: 10s
  proxy-send-timeout: 10s
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"client-max-body-size":"3m","proxy-connect-timeout":"10s","proxy-read-timeout":"10s","proxy-send-timeout":"10"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-config","namespace":"nginx-ingress"}}
  creationTimestamp: "2019-12-24T04:39:23Z"
  name: nginx-config
  namespace: nginx-ingress
  resourceVersion: "13845543"
  selfLink: /api/v1/namespaces/nginx-ingress/configmaps/nginx-config
  uid: 9313ae47-a0f0-463e-a25a-1658f1ca0d57
```

**3 、此时，ConfigMap定义的配置参数会被集群中所有的Ingress资源继承（除了annotations定义之外）**

![img](https://ask.qcloudimg.com/draft/4405893/w3dcxxnxfa.png?imageView2/2/w/1620)

有很多参数可以定义，详情配置可参考方文档说明：https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/configmap-and-annotations.md#Summary-of-ConfigMap-and-Annotations

#### 7.1.2 Annotations自定义参数

ConfigMap定义的是全局的配置参数，修改后所有的配置都会受影响，如果想针对某个具体的ingress资源自定义参数，则可以通过Annotations来实现，下面开始以实际的例子演示Annotations的使用。

**1、修改ingress资源，添加annotations的定义,通过nginx.org组修改了一些参数，如proxy-connect-timeout，调度算法为round_robin（默认为least _conn）**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress-demo
  labels:
    ingres-controller: nginx
  annotations:
    kubernets.io/ingress.class: nginx
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-send-timeout: "20s"
    nginx.org/proxy-read-timeout: "20s"
    nginx.org/client-max-body-size: "2m"
    nginx.org/fail-timeout: "5s"
    nginx.org/lb-method: "round_robin" 
spec:
  rules:
  - host: www.happylau.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: ingress-demo
          servicePort: 80
```

**2、 重新应用ingress对象并查看参数配置情况**

![img](https://ask.qcloudimg.com/draft/4405893/b02oqllujm.png?imageView2/2/w/1620)

由上面的演示可得知，Annotations的优先级高于ConfigMapMap，Annotations修改参数只会影响到某一个具体的ingress资源，其定义的方法和ConfigMap相相近似，但又有差别，部分ConfigMap的参数Annotations无法支持，反过来Annotations定义的参数ConfigMap也不一定支持，下图列举一下常规支持参数情况：

![img](https://ask.qcloudimg.com/draft/4405893/9uco9v1rs6.png?imageView2/2/w/1620)通用参数

![img](https://ask.qcloudimg.com/draft/4405893/kvbsl1udm3.png?imageView2/2/w/1620)日志支持

![img](https://ask.qcloudimg.com/draft/4405893/2bexxnq0b4.png?imageView2/2/w/1620)请求头部

![img](https://ask.qcloudimg.com/draft/4405893/hbfmi960e9.png?imageView2/2/w/1620)认证和安全

![img](https://ask.qcloudimg.com/draft/4405893/0h2kio34c5.png?imageView2/2/w/1620)upstream支持

### 7.2 虚拟主机和路由

安装nginx ingress时我们安装了一个customresourcedefinitions自定义资源，其能够提供除了默认ingress功能之外的一些高级特性如

- 虚拟主机VirtualServer
- 虚拟路由VirtualServerRoute
- 健康检查Healthcheck
- 流量切割Split
- 会话保持SessionCookie
- 重定向Redirect

这些功能大部分依赖于Nginx Plus高级版本的支持，社区版本仅支持部分，对于企业级开发而言，丰富更多的功能可以购买企业级Nginx Plus版本。如下以通过VirtualServer和VirtualServerRoute定义upstream配置为例演示功能使用。

**1、定义VirtualServer资源,其配置和ingress资源对象类似，能支持的功能会更丰富一点**

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: cafe
spec:
  host: cafe.example.com
  tls:
    secret: cafe-secret
  upstreams:
  - name: tea
    service: tea-svc
    port: 80
    name: tea
    service: ingress-demo 
    subselector:
    version: canary
    lb-method: round_robin
    fail-timeout: 10s
    max-fails: 1
    max-conns: 32
    keepalive: 32
    connect-timeout: 30s
    read-timeout: 30s
    send-timeout: 30s
    next-upstream: "error timeout non_idempotent"
    next-upstream-timeout: 5s
    next-upstream-tries: 10
    client-max-body-size: 2m
    tls:
      enable: true
  routes:
  - path: /tea
    action:
      pass: tea
```

**2、 应用资源并查看VirtualServer资源列表**

```shell
[root@node-1 ~]# kubectl apply -f vs.yaml 
virtualserver.k8s.nginx.org/cafe unchanged
[root@node-1 ~]# kubectl get virtualserver
NAME                 AGE
cafe                 2m52s
```

**3、检查ingress控制器的配置文件情况,生成的配置和upstream定义一致**

```shell
nginx@nginx-ingress-7mpfc:/etc/nginx/conf.d$ cat vs_default_cafe.conf 
upstream vs_default_cafe_tea {
    zone vs_default_cafe_tea 256k;
    server 10.244.0.51:80 max_fails=1 fail_timeout=10s max_conns=32;
    server 10.244.1.146:80 max_fails=1 fail_timeout=10s max_conns=32;
    server 10.244.1.147:80 max_fails=1 fail_timeout=10s max_conns=32;
    server 10.244.2.162:80 max_fails=1 fail_timeout=10s max_conns=32;
    keepalive 32;
}

server {
    listen 80;
    server_name cafe.example.com;
    listen 443 ssl;
    ssl_certificate /etc/nginx/secrets/default;
    ssl_certificate_key /etc/nginx/secrets/default;
    ssl_ciphers NULL;
    server_tokens "on";

    location /tea {
        proxy_connect_timeout 30s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;
        client_max_body_size 2m;
        proxy_max_temp_file_size 1024m;
        proxy_buffering on;
        proxy_http_version 1.1;
        set $default_connection_header "";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $vs_connection_header;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass https://vs_default_cafe_tea;
        proxy_next_upstream error timeout non_idempotent;
        proxy_next_upstream_timeout 5s;
        proxy_next_upstream_tries 10;   
    }   
}
```