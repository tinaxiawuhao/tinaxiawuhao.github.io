---
title: 'k8s高可用集群搭建'
date: 2022-06-11 21:48:56
tags: [k8s]
published: true
hideInList: false
feature: /post-images/McgbXRFnx.png
isTop: false
---


## 整体环境

3台master节点，3台[node](https://so.csdn.net/so/search?q=node&spm=1001.2101.3001.7020)节点。采用了Centos 7，有网络，互相可以ping通。

**准备工作查看单集群部署**

### 1.安装keepalived 和 haproxy

#### 1.1安装keepalived和 haproxy

```shell
1yum install keepalived haproxy -y
```

#### 1.2 配置 keepalived

```shell
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 16443) ]]; then exit 0; else exit 1; fi'"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER 
    interface ens33 
    virtual_router_id 51
    priority 250
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        192.168.40.19
    }
    track_script {
        check_haproxy
    }

}

EOF
```

```
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 16443) ]]; then exit 0; else exit 1; fi'"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP 
    interface ens33 
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        192.168.40.19
    }
    track_script {
        check_haproxy
    }

}

EOF
```

```
cat <<EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 16443) ]]; then exit 0; else exit 1; fi'"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP 
    interface ens33 
    virtual_router_id 51
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        192.168.40.19
    }
    track_script {
        check_haproxy
    }

}

EOF
```

> - vrrp_script 用于检测 haproxy 是否正常。如果本机的 haproxy 挂掉，即使 keepalived 劫持vip，也无法将流量负载到 apiserver。
> - 我所查阅的网络教程全部为检测进程, 类似 killall -0 haproxy。这种方式用在主机部署上可以，但容器部署时，在 keepalived 容器中无法知道另一个容器 haproxy 的活跃情况，因此我在此处通过检测端口号来判断 haproxy 的健康状况。
> - weight 可正可负。为正时检测成功 +weight，相当与节点检测失败时本身 priority 不变，但其他检测成功节点 priority 增加。为负时检测失败本身 priority 减少。
> - 另外很多文章中没有强调 nopreempt 参数，意为不可抢占，此时 master 节点失败后，backup 节点也不能接管 vip，因此我将此配置删去

#### 1.3 配置haproxy

```shell
cat <<EOF > /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local0
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      gk8s-master   192.168.40.20:6443 check
    server      gk8s-master2   192.168.40.21:6443 check
    server      gk8s-master3   192.168.40.22:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
    
EOF
```
设置开机启动

```
systemctl enable haproxy && systemctl start haproxy 
systemctl enable keepalived && systemctl start keepalived 
```

### 2.安装kubeadm、kubelet、kubectl

#### 1.配置文件修改

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 2.安装启用

```shell
sudo yum install -y kubelet-1.24.1 kubeadm-1.24.1 kubectl-1.24.1 --disableexcludes=kubernetes 
sudo systemctl enable kubelet && systemctl start kubelet
```

#### 3.修改kubelet的配置文件

先查看配置文件位置

```shell
systemctl status kubelet
```

![](https://tianxiawuhao.github.io/post-images/1653818855680.png)

```shell
vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

并添加以下内容(使用和docker相同的cgroup-driver)。

```shell
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --cgroup-driver=systemd"
```

#### 4.重启kubelet

```shell
systemctl daemon-reload && systemctl restart kubelet
```

### 3.获取K8S镜像（可忽略）

#### 1.获取镜像列表

**使用阿里云镜像仓库下载（国内环境该命令可不执行，下步骤kubeadm init --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers已经默认为国内环境）**

由于官方镜像地址被墙，所以我们需要首先获取所需镜像以及它们的版本。然后从国内镜像站获取。

```bash
kubeadm config images list
```

获取镜像列表后可以通过下面的脚本从阿里云获取：

```shell
vi /usr/local/k8s/k8s-images.sh
```

> 下面的镜像应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本

```bash
images=(  
    kube-apiserver:v1.24.1
    kube-controller-manager:v1.24.1
    kube-scheduler:v1.24.1
    kube-proxy:v1.24.1
    pause:3.7
    etcd:3.5.3-0
    coredns:v1.8.6
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

#### 2.赋权执行

```shell
chmod +x k8s-images.sh && ./k8s-images.sh
```

**以上操作在所有机器执行**

### 4.初始化环境（master操作）

#### 1.安装镜像

**采用模板配置文件加载**

```shell
kubeadm config print init-defaults  > kubeadm-config.yaml
```

```yaml
# 模板随版本更新
[root@master1 ~]# cat kubeadm-config.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.40.131    # 本机IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/cri-docker.sock  # 此处千万不要忘记修改，如果不修改等于没有替换。(此处已经更改完了)
  #criSocket: unix:///run/containerd/containerd.sock      # 此处千万不要忘记修改，如果不修改等于没有替换。(此处已经更改完了)
  name: master1        # 本主机名
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - master.k8s.io
    - 192.168.40.19
    - 192.168.40.20
    - 192.168.40.21
    - 192.168.40.22
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "192.168.40.19:16443"      # 虚拟IP和haproxy端口
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers    # 镜像仓库源要根据自己实际情况修改
kind: ClusterConfiguration
kubernetesVersion: v1.24.1     # k8s版本
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"   #设置网段，和下面网络插件对应
  serviceSubnet: 10.96.0.0/12
scheduler: {}
 
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
```

#### 2.查看kubeadm版本，修改命令参数

```shell
kubeadm version
```

这个就很简单了，只需要简单的一个命令：

```bash
#直接使用已经下载好的镜像
kubeadm init --kubernetes-version=v1.24.1 --control-plane-endpoint "192.168.40.19:16443" --apiserver-advertise-address=192.168.40.20 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap  --cri-socket unix:///var/run/cri-docker.sock | tee kubeadm-init.log
#或者采用aliyuncs镜像下载
kubeadm init --kubernetes-version=v1.24.1 --control-plane-endpoint "192.168.40.19:16443" --apiserver-advertise-address=192.168.40.20 --image-repository  registry.aliyuncs.com/google_containers  --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-docker.sock| tee kubeadm-init.log
#使用上面系统生成配置文件加载
kubeadm init --config kubeadm-config.yaml
```

#### 3.初始化命令说明：

> 虚拟ip节点端口号

```bash
--control-plane-endpoint
```

> 指明用 Master 的哪个 interface 与 Cluster 的其他节点通信。如果 Master 有多个 interface，建议明确指定，如果不指定，kubeadm 会自动选择有默认网关的 interface。
```bash
--apiserver-advertise-address
```
> 指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对 --pod-network-cidr 有自己的要求，这里设置为 10.244.0.0/16 是因为我们将使用 flannel 网络方案，必须设置成这个 CIDR。
```bash
--pod-network-cidr
```
> Kubenetes默认Registries地址是 k8s.gcr.io，在国内并不能访问 gcr.io，在1.19.3版本中我们可以增加–image-repository参数，默认值是 k8s.gcr.io，将其指定为阿里云镜像地址：registry.aliyuncs.com/google_containers。
```bash
--image-repository
```
> 关闭版本探测，因为它的默认值是stable-1，会导致从https://dl.k8s.io/release/stable-1.txt下载最新的版本号，我们可以将其指定为固定版本（最新版：v1.24.1）来跳过网络请求。
```bash
--kubernetes-version=v1.24.1 
```
> 指定启动时使用cri-docker调用docker

```shell
--cri-socket unix:///var/run/cri-docker.sock 
```

#### 4.错误启动重置

```shell
# 重置 如果有需要
kubeadm reset --cri-socket unix:///var/run/cri-docker.sock
```

#### 5.初始化成功后，为顺利使用kubectl，执行以下命令：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 6.添加其他主节点

##### 1.复制密钥及相关文件

```shell
ssh root@192.168.40.21 mkdir -p /etc/kubernetes/pki/etcd
scp /etc/kubernetes/admin.conf root@192.168.40.21:/etc/kubernetes 
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*}root@192.168.40.21:/etc/kubernetes/pki
scp /etc/kubernetes/pki/etcd/ca.* root@192.168.40.21:/etc/kubernetes/pki/etcd
```

```shell
ssh root@192.168.40.22 mkdir -p /etc/kubernetes/pki/etcd
scp /etc/kubernetes/admin.conf root@192.168.40.22:/etc/kubernetes 
scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@192.168.40.22:/etc/kubernetes/pki
scp /etc/kubernetes/pki/etcd/ca.* root@192.168.40.22:/etc/kubernetes/pki/etcd
```

##### 2.添加节点

```shell
kubeadm join 192.168.40.19:16443 --token pqir66.66fy6pexw3kprt2b --discovery-token-ca-cert-hash sha256:cd4c42e956fdc7e0ad48c990484c22cfd43da63cb3f3887bedc481e7f33a0be1 --control-plane --cri-socket unix:///var/run/cri-docker.sock 
```

#### 7.执行kubectl get nodes，查看master节点状态：

```shell
kubectl get node
```

#### 8.通过如下命令查看kubelet状态：

```bash
journalctl -xef -u kubelet -n 20
```

提示未安装cni 网络插件。

### 5.1安装flannel网络插件(CNI)

master执行以下命令安装flannel即可：

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
kube-flannel.yaml文件中的net-conf.json->Network地址默认为命令中–pod-network-cidr=值相同

**输入命令kubectl get pods -n kube-system,等待所有插件为running状态**。

**待所有pod status为Running的时候，再次执行kubectl get nodes：**

```shell
[root@k8s-master ~]# kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   16m   v1.24.1
```

**如上所示，master状态变为，表明Master节点部署成功！**

### 5.2安装calico网络(功能更完善)

#### 1.在master上下载配置calico网络的yaml。

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

#### 2.提前下载所需要的镜像。

```shell
# 查看此文件用哪些镜像：
[root@k8s-master ~]# grep image calico.yaml
image: docker.io/calico/cni:v3.23.1
image: docker.io/calico/node:v3.23.1
image: docker.io/calico/kube-controllers:v3.23.1
```

#### 3.安装calico网络。
 在master上执行如下命令：

```shell
kubectl apply -f calico.yaml
```

#### 5.验证结果。
 再次在master上运行命令 kubectl get nodes查看运行结果：

```shell
[root@k8s-master ~]# kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master01   Ready    control-plane,master   21h   v1.23.4
worker01   Ready    control-plane,master   16h   v1.23.4
worker02   Ready    control-plane,master   16h   v1.23.4
```

### 6.部署k8s-node1、k8s-node2、k8s-node3集群

**1、在k8s-node1、k8s-node2、k8s-node3等三台虚拟机中重复执行上面的步骤，安装好docker、kubelet、kubectl、kubeadm。**

#### 1.node节点加入集群

在上面第初始化master节点成功后，输出了下面的kubeadm join命令：

```shell
kubeadm join 192.168.40.131:6443 --token zj0u08.ge77y7uv76flqgdk --discovery-token-ca-cert-hash sha256:7cd23cec6afb192b2d34c5c719b378082a6315a9d91a22d91b83066c870d4db5 --cri-socket unix:///var/run/cri-docker.sock 
```

该命令就是node加入集群的命令，分别在k8s-node1、k8s-node2上执行该命令加入集群。

如果忘记该命令，可以通过以下命令重新生成：

```shell
kubeadm token create --print-join-command
```

#### 2.在master节点执行下面命令查看集群状态：

```shell
kubectl get nodes
```

```shell
[root@k8s-master ~]# kubectl get node
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   24m     v1.24.1
k8s-master2  Ready    master   24m     v1.24.1
k8s-master3  Ready    master   24m     v1.24.1
k8s-node1    Ready    <none>   5m50s   v1.24.1
k8s-node2    Ready    <none>   5m21s   v1.24.1
k8s-node3    Ready    <none>   5m21s   v1.24.1
```

如上所示，所有节点都为ready，集群搭建成功。

## 卸载集群命令

```shell
#建议所有服务器都执行
#!/bin/bash
kubeadm reset -f
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd
yum -y remove kubeadm* kubectl* kubelet* docker*
reboot
```

