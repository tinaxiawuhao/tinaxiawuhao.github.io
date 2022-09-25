---
title: 'k8s1.24.1安装（基于Centos 7）'
date: 2022-05-28 17:26:47
tags: [k8s]
published: true
hideInList: false
feature: /post-images/oqAZb-4wU.png
isTop: false
---



**注意：除了Master节点初始化及Node节点添加分别在Master节点和Node节点执行外，其余所有命令均在所有节点执行**

## 整体环境

一台master节点，2台[node](https://so.csdn.net/so/search?q=node&spm=1001.2101.3001.7020)节点。采用了Centos 7，有网络，互相可以ping通。

### 1.内核升级（可忽略）

为避免出现不可预知的问题，提升centos 7内核到最新版本

#### 联网升级内核
##### 1. 查看内核版本

```shell
uname -r
```

##### 2. 导入ELRepo软件仓库的公共秘钥

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```
##### 3. 安装ELRepo软件仓库的yum源

```shell
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```
##### 4. 启用 elrepo 软件源并下载安装最新稳定版内核

```shell
yum --enablerepo=elrepo-kernel install kernel-ml -y
```

##### 5. 查看系统可用内核，并设置内核启动顺序
```shell
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```
##### 6. 生成 grub 配置文件
机器上存在多个内核，我们要使用最新版本，可以通过 grub2-set-default 0 命令生成 grub 配置文件
```shell
grub2-set-default 0 　　#初始化页面的第一个内核将作为默认内核
grub2-mkconfig -o /boot/grub2/grub.cfg　　#重新创建内核配置
```
##### 7. 重启系统并验证
```shell
yum update
reboot
uname -r
```

##### 8. 删除旧内核
```shell
yum -y remove kernel kernel-tools
```

### 2.centos网络配置文件

网络配置文件名可能会有不同，在输入到ifcfg时，可以连续按两下tab键，获取提示，比如我的机器 为 ifcfg-ens33

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33 
```

#### 1.内容替换如下：

```shell
BOOTPROTO=static #静态连接
ONBOOT=yes #网络设备开机启动
IPADDR=192.168.40.131 #192.168.40.132,192.168.40.133.
NETMASK=255.255.255.0 #子网掩码
GATEWAY=192.168.40.2 #网关
DNS1=114.114.114.114 #DNS解析
```

#### 2.网络服务重启

```shell
service network restart
```

#### 3.查看IP地址

![](https://tianxiawuhao.github.io/post-images/1653818770568.png)

### 3.安装依赖包

```shell
yum install -y  wget 
```

### 4.修改yum源（视网络情况操作）

#### 1.备份

```shell
cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
#### 2.下载


```shell
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

或者使用清华大学站

```shell
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos|g' -i.bak /etc/yum.repos.d/CentOS-Base.repo
```

#### 3.清除生成缓存

```shell
yum clean all     # 清除系统所有的yum缓存
yum makecache     # 生成yum缓存
```

### 5.修改主机名

```shell
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
```

#### 查看主机名称

![](https://tianxiawuhao.github.io/post-images/1653818810010.png)

### 6.添加hosts解析

```shell
echo -e "192.168.40.131 k8s-master\n192.168.40.132 k8s-node1\n192.168.40.133 k8s-node2" >> /etc/hosts
```


### 7.关闭防火墙firewalld

```shell
systemctl stop firewalld && systemctl disable firewalld
```

### 8.关闭selinux

```shell
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```


### 9.关闭swap分区交换

```shell
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 10.配置内核参数

**将桥接的IPv4流量传递倒iptables的链**

#### 1.设置内核参数

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用swap空间，只有当系统00M时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启oom
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52786963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
```

#### 2.加载内核模块

```shell
modprobe br_netfilter && echo "modprobe br_netfilter" >> /etc/rc.local
```

#### 3.使内核参数生效

```shell
sysctl -p /etc/sysctl.d/k8s.conf
```

### 11.时间同步

```shell
timedatectl set-timezone Asia/Shanghai && timedatectl set-local-rtc 0
#重启依赖于系统时间的服务 
systemctl restart rsyslog && systemctl restart crond
```

### 12.安装iptables，设置空表

```shell
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save
```

检查服务的的规则：`iptables -L  -n`

### 13.开启IPVS

**由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块**

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#！/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
```

#### 授权启动

```shell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod 
```
**接下来还需要确保各个节点上已经安装了ipset软件包。 为了便于查看ipvs的代理规则，最好安装一下管理工具ipvsadm。**

```shell
yum install ipset ipvsadm -y
```

> service底层实现主要由两个网络模式组成：iptables与IPVS。他们都是有kube-proxy维护
>
> **Iptables VS IPVS**
>
> **Iptables：**
>
> **• 灵活，功能强大**
>
> **• 规则遍历匹配和更新，呈线性时延**
>
> **IPVS：**
>
> **• 工作在内核态，有更好的性能**
>
> **• 调度算法丰富：rr，wrr，lc，wlc，ip hash**

等集群部署成功，mode由空值修改成ipvs模式

```
kubectl edit configmap kube-proxy -n kube-system configmap/kube-proxy edited
```

![](https://tianxiawuhao.github.io/post-images/1654316987021.png)



**删除所有kube-proxy的pod,等待重启**

```
kubectl delete pod/kube-proxy-84p9n -n kube-system
# 查看ipvs相关规则
ipvsadm
```

### 14.安装docker软件

>
> 自1.20版本被弃用之后，dockershim组件终于在1.24的kubelet中被删除。从1.24开始，大家需要使用其他受到支持的运行时选项（例如containerd或CRI-O）；如果您选择Docker Engine作为运行时，则需要使用cri-dockerd。

#### 1.删除自带的docker

```shell
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```

#### 2.安装依赖包

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 3.安装yum源

```shell
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### 4.安装docker-ce

```shell
yum -y install docker-ce
```

#### 5.设置docker

```shell
cat > /etc/docker/daemon.conf <<EOF
{
  "registry-mirrors": ["https://registry.docker-cn.com","https://heusyzko.mirror.aliyuncs.com"],
  "insecure-registries": ["https://hub.test.com"],
  "exec-opts":["native.cgroupdriver=systemd"],
  "log-driver":"json-file",
  "log-opts":{
    "max-size": "100m"
    },
    "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d 
```

#### 6.重启docker服务

```shell
systemctl daemon-reload && systemctl start docker && systemctl enable docker
systemctl daemon-reload && systemctl restart docker 
```

以下的NO_PROXY表示对这些网段的服务器不使用代理，如果不需要用到代理服务器，以下的配置可以不写，注意，以下的代理是不通的。不建议使用代理，因为国内有资源可以访问到gcr.io需要的镜像，下文会介绍

```shell
#放在Type=notify下面
vi /usr/lib/systemd/system/docker.service
Environment="HTTPS_PROXY=http://www.ik8s.io:10080"
Environment="HTTP_PROXY=http://www.ik8s.io:10080"
Environment="NO_PROXY=127.0.0.0/8,172.20.0.0/16"
#保存退出后，执行
systemctl  daemon-reload
#确保如下两个参数值为1，默认为1。
 cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
 cat /proc/sys/net/bridge/bridge-nf-call-iptables
#启动docker-ce
systemctl restart docker
#设置开机启动
systemctl enable docker.service
```

```shell
#想要删除容器，则要先停止所有容器（当然，也可以加-f强制删除，但是不推荐）：
docker stop $(docker ps -a -q)
#删除所有容器
docker  rm $(docker ps -a -q)
#.删除所有镜像（慎重）
docker rmi $(docker images -q)
```

### 14.2.cri-dockerd安装

**CRI-Dockerd 其实就是从被移除的 Docker Shim 中，独立出来的一个项目，用于解决历史遗留的节点升级 Kubernetes 的问题。**

> kubelet并没有直接和dockerd交互，而是通过了一个dockershim的组件间接操作dockerd。dockershim提供了一个标准的接口，让kubelet能够专注于容器调度逻辑本身，而不用去适配dockerd的接口变动。而其他实现了相同标准接口的容器技术也可以被kubelet集成使用，这个接口称作CRI。dockershim和CRI的出现也是容器生态系统演化的历史产物。在k8s最早期的版本中是不存在dockershim的，kubelet直接和dockerd交互。但为了支持更多不同的容器技术（避免完全被docker控制容器技术市场），kubelet在之后的版本开始支持另一种容器技术rkt。这给kubelet的维护工作造成了巨大的挑战，因为两种容器技术没有统一的接口和使用逻辑，kubelet同时支持两种技术的使用还要保证一致的容器功能表现，对代码逻辑和功能可靠性都有很大的影响。为了解决这个问题，k8s提出了一个统一接口CRI，kubelet统一通过这个接口来调用容器功能。但是dockerd并不支持CRI，k8s就自己实现了配套的dockershim将CRI接口调用转换成dockerd接口调用来支持CRI。因此，dockershim并不是docker技术的一部分，而是k8s系统的一部分

**使用 CRI-Dockerd 项目**

```html
项目地址：https://github.com/Mirantis/cri-dockerd
```

#### 1.下载cri-dockerd二进制包或者源码自己编译

```shell
# 下载文件
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.0/cri-dockerd-v0.2.0-linux-amd64.tar.gz
# 解压文件
tar -xvf cri-dockerd-v0.2.0-linux-amd64.tar.gz
# 复制二进制文件到指定目录
cp cri-dockerd /usr/bin/
```


#### 2.配置启动文件
```shell
# 配置启动文件
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.service
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify

ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint=unix:///var/run/cri-docker.sock --network-plugin=cni --cni-bin-dir=/opt/cni/bin \
          --cni-conf-dir=/etc/cni/net.d --image-pull-progress-deadline=30s --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.7 \
          --docker-endpoint=unix:///var/run/docker.sock --cri-dockerd-root-directory=/var/lib/docker
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target

EOF
```
```shell
# 生成socket 文件
cat <<"EOF" > /usr/lib/systemd/system/cri-docker.socket
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target

EOF
```

```shell
# 启动 cri-dockerd
systemctl daemon-reload
systemctl start cri-docker
#设置开机启动
systemctl enable cri-docker
# 查看启动状态
systemctl status cri-docker
```

#### 3.下载cri-tools验证cri-docker 是否正常

```shell
# 下载二进制文件
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.0/crictl-v1.24.0-linux-amd64.tar.gz
# 解压
tar -xvf crictl-v1.24.0-linux-amd64.tar.gz
# 复制二进制文件到指定目录
cp crictl /usr/bin/
# 创建配置文件
vim /etc/crictl.yaml
runtime-endpoint: "unix:///var/run/cri-docker.sock"
image-endpoint: "unix:///var/run/cri-docker.sock"
timeout: 10
debug: false
pull-image-on-create: true
disable-pull-on-run: false
# 测试能否访问docker
# 查看运行的容器
crictl ps
# 查看拉取的镜像
 crictl images
 # 拉取镜像
 crictl pull busybox
 [root@k8s-node-4 ~]# crictl pull busybox
Image is up to date for busybox@sha256:5ecba83a746c7608ed511dc1533b87c737a0b0fb730301639a0179f9344b13448
返回一切正常cri-dockerd接入docker完整
```

### 15.部署 containerd(k8s-1.24版本以上)
**服务版本**

| 服务名称   | 版本号                     |
| ---------- | -------------------------- |
| 内核       | 5.14.3-1.el7.elrepo.x86_64 |
| containerd | v1.6.4（加入）             |
| ctr        | v1.6.4                     |
| k8s        | 1.24                       |

#### 1.安装containerd

**创建配置文件**

```shell
mkdir /etc/modules-load.d/containerd.conf 
```

**创建完配置文件执行以下命令**

```shell
modprobe overlay && modprobe br_netfilter
```

**立即生效**

```shell
sysctl --system
```

**下载 docker-ce 源**

```shell
wget http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
或者
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

**安装 containerd 服务并加入开机启动**

```shell
yum install -y containerd.io
systemctl enable containerd && systemctl start containerd
```

#### 2.配置 containerd

**创建路径**

```shell
mkdir -p /etc/containerd
```

**获取默认配置文件**

```shell
containerd config default | sudo tee /etc/containerd/config.toml
```

**修改配置文件，新增 "SystemdCgroup = true"，使用 systemd 作为 cgroup 驱动程序**

```shell
[root@master1 ~]# vi /etc/containerd/config.toml 
#修改以下内容
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
     SystemdCgroup = true                               ## 修改为true
```

**替换默认pause镜像地址**

默认情况下k8s.gcr.io无法访问，所以使用阿里云镜像仓库地址即可

```shell
# 所有节点更换默认镜像地址
sed -i 's/k8s.gcr.io/registry.cn-beijing.aliyuncs.com\/abcdocker/' /etc/containerd/config.toml 
```

**重启 containerd**

```shell
systemctl restart containerd
```

**查看 containerd 运行状态(以下状态视为正常)**

```shell
[root@master1 ~]# systemctl status containerd
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-03-06 08:09:00 CST; 1h 43min ago
     Docs: https://containerd.io
  Process: 931 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
 Main PID: 941 (containerd)
    Tasks: 11
   Memory: 61.4M
   CGroup: /system.slice/containerd.service
           └─941 /usr/bin/containerd
```

### 16.安装kubeadm、kubelet、kubectl
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

### 17.获取K8S镜像（可忽略）

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

### 18.初始化环境（master操作）

#### 1.安装镜像

**采用模板配置文件加载**

```shell
kubeadm config print init-defaults  > kubeadm-config.yaml
```

```yaml

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
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "192.168.40.151:16443"      # 虚拟IP和haproxy端口
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
kubeadm init --kubernetes-version=v1.24.1 --apiserver-advertise-address=192.168.40.131 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap  --cri-socket unix:///var/run/cri-docker.sock | tee kubeadm-init.log
#或者采用aliyuncs镜像下载
kubeadm init --kubernetes-version=v1.24.1 --apiserver-advertise-address=192.168.40.131 --image-repository  registry.aliyuncs.com/google_containers  --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/cri-docker.sock| tee kubeadm-init.log
#使用上面系统生成配置文件加载
kubeadm init --config kubeadm-config.yaml
```

#### 3.初始化命令说明：

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

#### 6.添加节点

```shell
kubeadm join 192.168.40.131:6443 --token eyr8v6.j84sxak8aptse8j9 --discovery-token-ca-cert-hash sha256:c082f3c546bdbac02d0d0a3b696de4004b0d449e37838fa38d4752b39682676b --cri-socket unix:///var/run/cri-docker.sock 
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

### 19.1安装flannel网络插件(CNI)

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
k8s-master   Ready    master   16m   v1.19.3
```

**如上所示，master状态变为，表明Master节点部署成功！**

### 19.2安装calico网络(功能更完善)

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

```csharp
kubectl apply -f calico.yaml
```

#### 5.验证结果。
 再次在master上运行命令 kubectl get nodes查看运行结果：

```css
[root@k8s-master ~]# kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master01   Ready    control-plane,master   21h   v1.23.4
worker01   Ready    <none>                 16h   v1.23.4
worker02   Ready    <none>                 16h   v1.23.4
```

### 20.部署k8s-node1、k8s-node2集群

**1、在k8s-node1、k8s-node2等两台虚拟机中重复执行上面的步骤，安装好docker、kubelet、kubectl、kubeadm。**

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
k8s-master   Ready    master   24m     v1.19.3
k8s-node1    Ready    <none>   5m50s   v1.19.3
k8s-node2    Ready    <none>   5m21s   v1.19.3

```

如上所示，所有节点都为ready，集群搭建成功。

### 21.安装ingress-nginx

```yaml
vi ingress-nginx-deploy.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  name: ingress-nginx
---
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resourceNames:
  - ingress-controller-leader
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
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
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
rules:
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - validatingwebhookconfigurations
  verbs:
  - get
  - update
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx-admission
subjects:
- kind: ServiceAccount
  name: ingress-nginx-admission
  namespace: ingress-nginx
---
apiVersion: v1
data:
  allow-snippet-annotations: "true"
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  ports:
  - appProtocol: http
    name: http
    port: 80
    protocol: TCP
    targetPort: http
  - appProtocol: https
    name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: NodePort
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller-admission
  namespace: ingress-nginx
spec:
  ports:
  - appProtocol: https
    name: https-webhook
    port: 443
    targetPort: webhook
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        image: registry.aliyuncs.com/google_containers/nginx-ingress-controller:v1.2.0
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      dnsPolicy: ClusterFirst
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: ingress-nginx
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission-create
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.2.0
      name: ingress-nginx-admission-create
    spec:
      containers:
      - args:
        - create
        - --host=ingress-nginx-controller-admission,ingress-nginx-controller-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=ingress-nginx-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: create
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission-patch
  namespace: ingress-nginx
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/component: admission-webhook
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
        app.kubernetes.io/version: 1.2.0
      name: ingress-nginx-admission-patch
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=ingress-nginx-admission
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=ingress-nginx-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: registry.aliyuncs.com/google_containers/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: patch
        securityContext:
          allowPrivilegeEscalation: false
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: ingress-nginx-admission
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    app.kubernetes.io/component: admission-webhook
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.2.0
  name: ingress-nginx-admission
webhooks:
- admissionReviewVersions:
  - v1
  clientConfig:
    service:
      name: ingress-nginx-controller-admission
      namespace: ingress-nginx
      path: /networking/v1/ingresses
  failurePolicy: Fail
  matchPolicy: Equivalent
  name: validate.nginx.ingress.kubernetes.io
  rules:
  - apiGroups:
    - networking.k8s.io
    apiVersions:
    - v1
    operations:
    - CREATE
    - UPDATE
    resources:
    - ingresses
  sideEffects: None
  
kubectl create -f ingress-nginx-deploy.yaml
```

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

