### 一、主机环境准备

#### 1.1 主机规划

本文的 Kubernetes 集群基于 VirtualBox 创建的三台主机，角色和配置如下

| 主机名   | 操作系统  | CPU  | MEM  | 角色   |
| -------- | --------- | ---- | ---- | ------ |
| master01 | CentOS7.7 | 2    | 2GB  | master |
| worker01 | CentOS7.7 | 2    | 2GB  | worker |
| worker02 | CentOS7.7 | 2    | 2GB  | worker |

其中 CentOS 都是最小化安装，关于怎样虚拟机安装 Linux 这里不展开了，网上教程很多。

三台主机的网络均使用桥接模式，后面会设置静态 IP ：

| 主机名   | IP            |
| -------- | ------------- |
| master01 | 192.168.0.114 |
| worker01 | 192.168.0.115 |
| worker02 | 192.168.0.116 |



>  以下有很多命令需要同时在三台主机执行，为了提高效率，我这里使用 iTerm2 进行多个终端同时输入。进入和退出快捷键：Command + Shift + i 
>
> 如果是在 Windows 上，Xshell 也有类似的功能。



#### 1.2 设置主机名

三台主机安装完成后，首先分别对每台主机设置其主机名：

```shell
# 在每台主机上执行
[root@name ~]# hostnamectl set-hostname master01
```

#### 1.3 修改网络配置

修改`/etc/sysconfig/network-scripts/ifcfg-enp0s3`，配置静态 IP，按 1.1节 的主机 IP 规划进行修改。

三台主机均需要修改

```
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="63e5deae-4f6c-4734-858c-38e2bf601c7f"
DEVICE="enp0s3"
ONBOOT="yes"
IPADDR="192.168.0.114"
PREFIX="24"
GATEWAY="192.168.0.1"
```

修改完成后需要重新启动网卡：

```shell
[root@master01 ~]# systemctl restart network

# 重新启动后查看 IP 地址
[root@master01 ~]# ip a
```

#### 1.4 主机名称解析

配置 IP 和 主机的对应关系，在三台主机的 `/etc/hosts` 添加如下内容：

```
192.168.0.114 master01
192.168.0.115 worker01
192.168.0.116 worker02
```

分别在每个主机 ping 一下其它两个主机名，确保能通：

```
[root@master01 ~]# ping worker01
[root@master01 ~]# ping worker02
```



#### 1.5 主机安全配置

三台主机均需要执行本节操作。

##### 关闭 firewalld

```shell
# 检查是否打开
[root@master01 ~]# systemctl status firewald
# 关闭防火墙
[root@master01 ~]# systemctl stop firewalld
# 禁止开机自启动
[root@master01 ~]# systemctl disable firewalld
# 验证状态
[root@master01 ~]# firewall-cmd --state
not running
```

##### SELINUX 设置

```shell
# 检查是否开启
[root@master01 ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
# 或者另一种方式检查
[root@master01 ~]# getenforce
Enforcing

# 修改配置
[root@master01 ~]# sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 重启
[root@master01 ~]# reboot
```

#### 1.6 主机时间同步

三台主机均需要执行本节操作。

```shell
# 由于是最小安装系统，需要单独安装 ntpdate
[root@master01 ~]# yum install -y ntpdate

# 使用阿里的时钟源，每隔一小时同步一次
[root@master01 ~]# crontab -e
# 将打开编辑，写入 0 */1 * * * ntpdate time1.aliyun.com
# 验证
[root@master01 ~]# crontab -l
0 */1 * * * ntpdate time1.aliyun.com

# 如果是首次，可以主动同步一次
[root@master01 ~]# ntpdate time1.aliyun.com
22 Feb 22:24:18 ntpdate[1659]: adjust time server 203.107.6.88 offset -0.005327 sec

# 验证多台主机时间是否一致
[root@master01 ~]# date
2020年 02月 22日 星期六 22:24:18 CST
```

#### 1.7 关闭 Swap 分区

使用 kubeadm 部署集群必须关闭 Swap分区，三台主机均需要执行本节操作。

```shell
# 注释掉 /etc/fstab 文件的最后一行 swap 配置
[root@master01 ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat Feb 22 20:20:59 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=808954aa-4397-4557-a08c-f8a66be7774f /boot                   xfs     defaults        0 0
# /dev/mapper/centos-swap swap                    swap    defaults        0 0

# 重启前先看下现在的状态（可以看到Swap信息）
[root@master01 ~]# free
              total        used        free      shared  buff/cache   available
Mem:        2047016       97400     1754632        8720      194984     1802148
Swap:       2097148           0     2097148

# 重启
[root@worker01 ~]# reboot

# 查看是否生效（Swap已经为0了）
[root@worker01 ~]# free
              total        used        free      shared  buff/cache   available
Mem:        2047016       95452     1855096        8748       96468     1823464
Swap:             0           0           0

```



#### 1.8 配置主机网桥过滤功能

三台主机均需要执行本节操作。

##### 1.8.1 添加网桥过滤

在kubernetes集群中添加网桥过滤，为了实现内核的过滤。

```shell
# 添加网桥过滤及地址转发
[root@worker01 ~]# cat > /etc/sysctl.d/k8s.conf << EOF
> net.bridge.bridge-nf-call-ip6tables = 1
> net.bridge.bridge-nf-call-iptables = 1
> net.ipv4.ip_forward = 1
> vm.swappiness = 0
> EOF

# 加载 br_netfilter 模块
[root@worker01 ~]# modprobe br_netfilter

# 查看是否加载
[root@worker01 ~]# lsmod | grep br_netfilter
br_netfilter           22256  0
bridge                151336  1 br_netfilter

# 加载网桥过滤配置文件
[root@worker01 ~]# sysctl -p /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
```

##### 1.8.2 开启 IPVS

由于Kubernets在使用Service的过程中需要用到iptables或者是ipvs，ipvs整体上比iptables的转发效率要高，因此这里我们直接部署ipvs就可以了。

**安装 ipset 及 ipvsadm **

```shell
[root@master01 ~]# yum install -y ipset ipvsadm
```

```shell
# 添加需要加载的模块
[root@master01 ~]# cat > /etc/sysconfig/modules/ipvs.modules << EOF
> #!/bin/bash
> modprobe -- ip_vs
> modprobe -- ip_vs_rr
> modprobe -- ip_vs_wrr
> modprobe -- ip_vs_sh
> modprobe -- nf_conntrack_ipv4
> EOF

# 授权、运行、检查是否加载
[root@master01 ~]# chmod +x /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
nf_conntrack_ipv4      15053  0
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  0
ip_vs                 145497  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139224  2 ip_vs,nf_conntrack_ipv4
libcrc32c              12644  3 xfs,ip_vs,nf_conntrack
```

### 二、安装 Docker

Kubernetes 要借助 Docker 这种容器管理工具来完成集群的管理。接下来我们需要在所有主机上安装 Docker。

#### 2.1 安装依赖

```shell
[root@master01 ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 2.2 设置镜像源

由于不可描述原因，国内下载 docker 非常慢，这里使用清华的镜像源：

```shell
# 下载 repo 文件
[root@master01 ~]# wget -O docker-ce.repo https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo

# 把软件仓库地址替换为 TUNA
[root@master01 ~]# sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

# makecache
[root@master01 ~]# ssudo yum makecache fast

# 查看 docker-ce 版本
[root@master01 ~]# yum list docker-ce.x86_64 --showduplicates | sort -r
```

#### 2.3 安装 docker

这里我使用 18.06.3.ce-3.el7 这个版本：

```shell
[root@master01 ~]# yum install -y docker-ce-18.06.3.ce-3.el7
```

然后就是等待安装，由于使用了国内源，安装速度还是很快的。

#### 2.4 设置并验证

```shell
# 开机自启动
[root@master01 ~]# systemctl enable docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
# 启动 docker
[root@master01 ~]# systemctl start docker

#验证
[root@master01 ~]# docker version
Client:
 Version:           18.06.3-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        d7080c1
 Built:             Wed Feb 20 02:26:51 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.3-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       d7080c1
  Built:            Wed Feb 20 02:28:17 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

#### 2.5 修改 docker 配置文件

```shell
# 在 /etc/docker/daemon.json 写入如下内容
[root@master01 ~]# cat > /etc/docker/daemon.json << EOF
> {
>     "exec-opts": ["native.cgroupdriver=systemd"]
> }
> EOF

# 重新启动docker
[root@master01 ~]# systemctl restart docker
```



### 三、集群软件安装及配置

有三个软件需要安装：

* kubeadm: 初始化集群，管理集群等
* kubelet: 用于接收 api-server 指令，对 Pod 生命周期进行管理
* Kubectl: 集群命令管理工具

我这里三个软件的版本均为：1.17.3-0

接下来分别在三台主机上安装这三个软件。

#### 3.1 更改软件源

Kubernetes 官网给的 yum 源是 packages.cloud.google.com ，国内访问不了，此时我们可以使用阿里云的 yum 仓库镜像。

```shell
[root@master01 ~]# cat /etc/yum.repos.d/k8s.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
       
# 验证是否可以找到 kubeadm
[root@master01 ~]# yum list | grep kubeadm
kubeadm.x86_64                              1.17.3-0   
```

#### 3.2 安装 kubeadm kubelet kubectl

```shell
[root@master01 ~]# yum list kubeadm.x86_64 --showduplicates | sort -r
[root@master01 ~]# yum install -y kubeadm-1.17.3-0 kubelet-1.17.3-0 kubectl-1.17.3-0
```

#### 3.3 软件设置

2.5节对 docker 的 cgroup driver 做了修改，这里 kubelet 需要保持一致，需修改如下配置内容：

```shell
[root@master01 ~]# cat /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
```

对 kubelet 设置为开机启动即可（不需要手动启动），集群初始化后会自动启动。

```shell
[root@master01 ~]# systemctl enable kubelet
```



### 四、 Kubernetes 集群容器镜像准备

为避免初始化集群的时候下载镜像太慢，这里提前对需要的镜像进行下载。

#### 4.1 Master 主机镜像

```shell
# 查看集群使用的容器镜像都有哪些
[root@master01 ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.17.3
k8s.gcr.io/kube-controller-manager:v1.17.3
k8s.gcr.io/kube-scheduler:v1.17.3
k8s.gcr.io/kube-proxy:v1.17.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.5

# 可以手动进行下载，类似：
[root@master01 ~]# docker pull k8s.gcr.io/kube-apiserver:v1.17.3
# 或者编写 shell 脚本批量下载
```

注意，由于国内网络环境，无法下载到 k8s.gcr.io 的镜像，我这里使用了阿里云的镜像仓库服务，构建我们自己的 Docker 镜像，具体操作可以参考 [[**如何在国内顺畅下载被墙的 Docker 镜像？**](https://www.yuque.com/heqingbao/cx9ztf/wrordu)]

本文就是基于文章中讲的方式成功搭建 K8s 集群。

我把初始化集群所需要的所有 docker 镜像都打包到百度网盘了，如果不想自己搭建，可以下载下来后 `docker load -i file` 直接导入（可参考 5.2节导入方法）。

网盘链接：链接: https://pan.baidu.com/s/1UZ8P4N_QIJjiePQLTwV55g 提取码: gxah

如果链接失效，请在 `非著名开发者` 公众号后台回复，我看到后将及时更新。 

```shell
# 镜像下载完成
[root@master01 ~]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver            v1.17.3             9f9ca8dae837        8 minutes ago       171MB
k8s.gcr.io/coredns                   1.6.5               0014af41d6b1        10 minutes ago      41.6MB
k8s.gcr.io/pause                     3.1                 98c2afe8d9e1        11 minutes ago      742kB
k8s.gcr.io/kube-proxy                v1.17.3             25618ead4414        12 minutes ago      116MB
k8s.gcr.io/kube-scheduler            v1.17.3             98b01601becf        13 minutes ago      94.4MB
k8s.gcr.io/kube-controller-manager   v1.17.3             5632f27ef964        14 minutes ago      161MB
k8s.gcr.io/etcd                      3.4.3-0             7bbadee3c825        33 minutes ago      288MB
```

#### 4.2 Worker主机镜像

master 主机上已有 docker 镜像了，直接把对应的镜像导出到到 worker 主机即可。

这里只需要 pause 和 kube-proxy 两个镜像：

```shell
# 在 master 上把 docker 镜像导出到文件
[root@master01 ~]# docker save -o kube-proxy_1.17.3.tar k8s.gcr.io/kube-proxy:v1.17.3
[root@master01 ~]# docker save -o pause_3.1.tar k8s.gcr.io/pause:3.1

# 把镜像文件复制到 worker 主机
[root@master01 ~]# scp kube-proxy_1.17.3.tar pause_3.1.tar worker01:/root
```

```shell
# 在 worker 主机上导入 docker 镜像文件
[root@worker01 ~]# docker load -i kube-proxy_1.17.3.tar
[root@worker01 ~]# docker load -i pause_3.1.tar
# 验证
[root@worker01 ~]# docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/pause        3.1                 98c2afe8d9e1        26 minutes ago      742kB
k8s.gcr.io/kube-proxy   v1.17.3             25618ead4414        27 minutes ago      116MB
```

按同样的方式，在 worker02 主机也要导入这两个 docker 镜像。

### 五、Kubernetes 集群初始化

我们将使用 kubeadm 来完成对集群的初始化操作。

#### 5.1 初始化

在 master 节点操作。

```shell
[root@master01 ~]# kubeadm init --kubernetes-version=v1.17.3 --pod-network-cidr=172.16.0.0/16 --apiserver-advertise-address=192.168.0.114

# 会比较慢，稍等一下，不过我们已经提前下载了镜像，所以也不会等太久
# 如果看到如下内容表示执行成功
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.114:6443 --token 2xjokn.mxcdy3sv2teg5qtr \
    --discovery-token-ca-cert-hash sha256:e0289d4f18dd7f1530bad0863492546fd36d6a43617d7e8388b289f18b41a357
```

建议保存上面的输入内容，接下来操作需要用到。

#### 5.2 拷贝配置文件

在 master 节点操作。

```shell
[root@master01 ~]# mkdir -p $HOME/.kube
[root@master01 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# 这里不需要修改属主和属组，我们直接用root用户操作就行了
```

#### 5.3 配置网络

这里我们使用 `Calico`，具体可以参考此文档：https://docs.projectcalico.org/getting-started/kubernetes/quickstart

在 master 节点操作。

```shell
[root@master01 ~]# kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

它会下载几个 docker 镜像，由于网络问题，这里还是通过提前下载方式。（下面的镜像已经包含在 **4.1 节 **给出的网盘里）

##### 5.3.1 下载 calico docker 镜像

```shell
# 下载 calico.yaml
[root@master01 ~]# wget https://docs.projectcalico.org/v3.9/manifests/calico.yaml

# 查看需要哪些 docker 镜像
[root@master01 ~]# cat calico.yaml | grep image
          image: calico/cni:v3.9.5
          image: calico/cni:v3.9.5
          image: calico/pod2daemon-flexvol:v3.9.5
          image: calico/node:v3.9.5
          image: calico/kube-controllers:v3.9.5
          
# 按照第 4.1节 内容，下载 docker 镜像

# 下载完成，检查下
[root@master01 ~]# docker images | grep calico
calico/kube-controllers                                         v3.9.5              8b9346e36939        3 minutes ago       56MB
calico/pod2daemon-flexvol                                       v3.9.5              3f5469b53c71        4 minutes ago       9.78MB
calico/node                                                     v3.9.5              81bcacea423a        5 minutes ago       195MB
registry.cn-hangzhou.aliyuncs.com/heqingbao-docker/calico_cni   latest              c9be39188163        7 minutes ago       167MB
calico/cni                                                      v3.9.5              c9be39188163        7 minutes ago       167MB
```

##### 5.3.2 修改 calico 资源清单文件

```
# 由于 calico 自身网络发现机制有问题，比如集群重启后网络组件会有问题，这里修改下发现机制，添加第 607 和 608 行
604             # Auto-detect the BGP IP address.
605             - name: IP
606               value: "autodetect"
607             - name: IP_AUTODETECTION_METHOD
608               value: "interface=enp0s3.*"
```

注意 interface 后面的网卡名字，我这里是用 VirtualBox 创建的虚拟机，网卡名字叫做 enp0s3 （可以使用 ip a 命令查看）

修改 kubeadm 初始化时指定的 pod-network-cidr 

```shell
621             - name: CALICO_IPV4POOL_CIDR
622               value: "172.16.0.0/16"
```

##### 5.3.3 应用 calico 资源清单文件

```shell
[root@master01 ~]# kubectl apply -f calico.yaml

# 执行完成后 我们来看下集群的节点状态
[root@master01 ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   65m   v1.17.3

# master01节点已经 Ready
```

#### 5.4 把 worker 节点加入集群

还记得 kubeadm 初始化时最后那句 `kubeadm join ` 命令吧？

（那个 token 有效期只有 8 个小时，过期后需要使用 kubeadm 创新一个新的）

我们去 worker01 节点执行下：

```shell
[root@worker01 ~]# kubeadm join 192.168.0.114:6443 --token 2xjokn.mxcdy3sv2teg5qtr \
    --discovery-token-ca-cert-hash sha256:e0289d4f18dd7f1530bad0863492546fd36d6a43617d7e8388b289f18b41a357

# 稍等片刻，看到如下输出表示成功了
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

同样的，去 worker02 节点执行同样的操作。

执行完成后去 master01 查看下集群节点状态：

```shell
[root@master01 ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
master01   Ready    master   76m     v1.17.3
worker01   Ready    <none>   7m55s   v1.17.3
worker02   Ready    <none>   9m10s   v1.17.3
```

OK，三个节点全都有，并且都是 `Ready` 状态，到此三个节点的集群就部署完了。

#### 5.5 验证 K8s 集群可用性方法

```shell
# 查看节点状态
[root@master01 ~]# kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
master01   Ready    master   10h   v1.17.3
worker01   Ready    <none>   8h    v1.17.3
worker02   Ready    <none>   23m   v1.17.3

# 查看集群健康状态
[root@master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}

# 或查看集群信息
[root@master01 ~]# kubectl cluster-info
Kubernetes master is running at https://192.168.0.114:6443
KubeDNS is running at https://192.168.0.114:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

# 或查看所有Pod状态
[root@master01 ~]# kubectl get pod --namespace kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-6b9d4c8765-t2rw8   1/1     Running   1          9h
calico-node-4f5sr                          1/1     Running   1          9h
calico-node-688ct                          1/1     Running   1          26m
calico-node-dhf5t                          1/1     Running   2          8h
coredns-6955765f44-jr5mk                   1/1     Running   1          10h
coredns-6955765f44-tw67g                   1/1     Running   1          10h
etcd-master01                              1/1     Running   1          10h
kube-apiserver-master01                    1/1     Running   1          10h
kube-controller-manager-master01           1/1     Running   1          10h
kube-proxy-9k8tk                           1/1     Running   1          10h
kube-proxy-wkr9x                           1/1     Running   2          8h
kube-proxy-wxp9d                           1/1     Running   1          26m
kube-scheduler-master01                    1/1     Running   1          10h
```

可以发现集群已经处于可用状态了。