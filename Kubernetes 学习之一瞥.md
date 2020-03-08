在 [**上一篇文章**](http://mp.weixin.qq.com/s?__biz=MzI1NTU4ODc0OQ==&mid=2247483810&idx=1&sn=b81cb53ac1be974f70ddaed29aed5feb&chksm=ea32e140dd4568560e32ad14c32848378beaf7962b33acd7269b4db0f34c74420d3c8fe32e7a&scene=21#wechat_redirect) 中，带领大家从 0 开始在本地无翻墙部署了一个三节点的 K8s 集群，可能大家还不太明白这玩意儿到底怎么玩，K8s 到底可以干什么，有什么亮点，或者对 K8s 还没什么兴趣。

那看这篇文章的标题就知道了，我们先不用了解 K8s 那些复杂的知识点，先简单在集群上跑点东西试试看，没准看了你就感兴趣了呢？

以下内容基于前一篇文章搭建的 K8s 集群，如果你还没有搭建好，建议先看看 [**从 0 开始部署 Kubernetes 集群**](http://mp.weixin.qq.com/s?__biz=MzI1NTU4ODc0OQ==&mid=2247483810&idx=1&sn=b81cb53ac1be974f70ddaed29aed5feb&chksm=ea32e140dd4568560e32ad14c32848378beaf7962b33acd7269b4db0f34c74420d3c8fe32e7a&scene=21#wechat_redirect)，然后跟着自己输入每个命令，并对比结果，如果有任何不懂，可以公众号后台回复，我看到后会及时回复。

## 一、准备

### 1.1 Docker 镜像准备

以下两种方式任选其中一种即可。

#### 1.1.1 使用现成的镜像

我已经把后面用于演示的镜像推送到了 Docker Hub ，大家可以直接 Pull 下来：

```shell
docker pull heqingbao/k8s_myapp:v1
```

分别有 v1、v2、v3 三个版本。

如果想制作自己的镜像，可以参考下面 1.2 节

#### 1.1.2 制作自己的 Docker 镜像

我这里使用一个自定义 Nginx 镜像，只有一个页面：

index.html

```html
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```

Version 表示版本，后面用来演示升级和回滚；

hostname.html 用来显示当前 Pod 的名字，只是个别名，实际上是执行的容器的 hostname 命令，如下配置文件：

/etc/nginx/conf.d/default.conf

```conf
server {
    ...
    location = /hostname.html {
	        alias /etc/hostname;
    }
    ...
}
```

在 K8s 集群里，Pod 的 hostname 会被自动设置成 Pod 的名字，后面的演示我们也会看到。

稍微说下，自定义镜像有两种方式，一种是自己写 Dockerfile，另一种是直接在 Running 的镜像上修改，然后再 `docker commit` 把修改后的镜像保存下来，类似于保存快照。很明显这里我们使用第二种更方便些。

通过这种方式制作 v1、v2、v3 三个镜像（对应修改 index.html 中的 Version 值），制作完镜像后，通过 `docker push`推送到 Docker Hub，后面我们就用这几个镜像做实验。

### 1.2 集群准备（可选）

在前一篇讲解搭建 K8s 集群的文章中，我把两个 Node 节点的主机名设成了 worker01 和 worker02 ，后面发现不太符合 K8s 的规范，因为我发现 `kubectl get node ` 是用的 Node 命名 ，也参考了其它文章的命名方法，决定把两个主机名称修成 `node01` 和 `node02` 。（这一步可选，改不改都没关系）

修改主要分三步：

1. 把节点从集群删除
2. 在节点修改 hostname ，再重置 kubeadm
3. 把节点再加入集群

代码演示：

```shell
# 先把要改名的节点移出集群
[root@master01 ~]# kubectl delete node worker01

# 再修改节点的 hostname
[root@worker01 ~]# hostnamectl set-hostname node01

# 重置，重要！
[root@worker01 ~]# kubeadm reset

# 再加入集群
[root@worker01 ~]# kubeadm join 192.168.0.114:6443 --token 6wjkat.d7r4els6zc0oo9pl \
    --discovery-token-ca-cert-hash sha256:e0289d4f18dd7f1530bad0863492546fd36d6a43617d7e8388b289f18b41a357
```

如果 token 已过期，可以创建一个新的：

```shell
# 创建新的 token
[root@master01 ~]# kubeadm token create
# 生成 hash
[root@master01 ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

同样的方式对 worker02 节点也做修改。 修改完成后执行 `kubectl get node` 验证下是否修改成功。

## 二、代码演示

### 2.1 快速开始

先从简单的上手吧，用 Deployment 部署个 Pod 试试：

```shell
# 创建一个 Deployment 并设置 Pod 的副本数为 1
[root@master01 ~]# kubectl run nginx-deployment --image=heqingbao/k8s_myapp:v1 --port=80 --replicas=1
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx-deployment created
# 忽略上面的警告，后面会用资源清单的方式创建

# 可以看到已经创建了一个 Deployment 
[root@master01 ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           8s

# 同时 Deployment 是通过 ReplicaSet 管理 Pod 
[root@master01 ~]# kubectl get ReplicaSet
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-747c8b9b58   1         1         1       18s

# 查看一下 Pod ，已经在运行了
[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-747c8b9b58-f9vd9   1/1     Running   0          23s

# 再查看的具体些
[root@master01 ~]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-747c8b9b58-f9vd9   1/1     Running   0          29s   172.16.196.129   node01   <none>           <none>
```

以上，可以看出 Pod 正处于 Running 状态，还可以看到启动的时间，Pod 的 IP 地址，以及位于哪个节点（node01）。

可以 ssh 到 01 节点查看是否真的有运行这个容器：

```shell
[root@node01 ~]# docker ps -a | grep nginx
cf53b198c4a2        bb326977973c           "nginx -g 'daemon of…"   2 minutes ago       Up 2 minutes                                    k8s_nginx-deployment_nginx-deployment-747c8b9b58-f9vd9_default_ccdce165-2059-4be1-9b37-fa40a0876ea7_0
938eea65d289        k8s.gcr.io/pause:3.1   "/pause"                 2 minutes ago       Up 2 minutes                                    k8s_POD_nginx-deployment-747c8b9b58-f9vd9_default_ccdce165-2059-4be1-9b37-fa40a0876ea7_0
```

只要我们有运行一个 Pod ，就会有这个 `pause`，具体后面再讲。

### 2.2 如何访问 Pod

在 K8s 里面，所有容都是扁平化的，我们直接访问下这个 Pod 的 IP ：

```shell
[root@master01 ~]# curl http://172.16.196.129
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
[root@master01 ~]# curl http://172.16.196.129/hostname.html
nginx-deployment-747c8b9b58-f9vd9

[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-747c8b9b58-f9vd9   1/1     Running   0          5m42s
```

可以看出 Pod 是能访问的，并且获取的 hostname 确实是 Pod 的名字。（hostname.html 即是获取容器的 hostname ）

### 2.3 Pod 副本控制

接下来演示下到底  K8s 好在哪里。

比如有一天，我们把这个 Pod 删除了：

```shell
[root@master01 ~]# kubectl delete pod nginx-deployment-747c8b9b58-f9vd9
pod "nginx-deployment-747c8b9b58-f9vd9" deleted
```

然后再查看一下 Pod 状态：

```shell
[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-747c8b9b58-8npjm   0/1     ContainerCreating   0          5s
```

不对呀，删除了怎么还有呢？而且还是个新的。。。

原因就是我们在创建的时候告诉了 K8s 我的副本数为 1 ，当删除后，它发现副本数为 0 了，会自动创建。

又比如某天我们想扩容，扩大到三个副本：

```shell
# 扩容到 3 个副本
[root@master01 ~]# kubectl scale --replicas=3 deployment/nginx-deployment
deployment.apps/nginx-deployment scaled

# 查看 Pod 状态
[root@master01 ~]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-747c8b9b58-7ww5j   1/1     Running   0          10s    172.16.196.130   node01   <none>           <none>
nginx-deployment-747c8b9b58-8npjm   1/1     Running   0          3m6s   172.16.140.65    node02   <none>           <none>
nginx-deployment-747c8b9b58-lf84w   1/1     Running   0          10s    172.16.196.131   node01   <none>           <none>
```

可以看到，创建了三个 Pod ，node01 上两个，node02 上一个。

比如我再删除：

```shell
# 删除一个 Pod
[root@master01 ~]# kubectl delete pod nginx-deployment-747c8b9b58-7ww5j
pod "nginx-deployment-747c8b9b58-7ww5j" deleted

# 查看 Pod
[root@master01 ~]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
nginx-deployment-747c8b9b58-8npjm   1/1     Running   0          4m48s   172.16.140.65    node02   <none>           <none>
nginx-deployment-747c8b9b58-lf84w   1/1     Running   0          112s    172.16.196.131   node01   <none>           <none>
nginx-deployment-747c8b9b58-xmkjf   1/1     Running   0          15s     172.16.140.66    node02   <none>           <none>
```

可以看出还是 3 个 Pod，现在是 node01 上一个，node02 上两个。

### 2.4 多个 Pod 怎么负载

那又有一个问题，现在有三个副本，你想访问哪个副本呢？

在微服务里面，一般使用 Nginx 设置 upstream 去反向代理这三台主机，在 K8s 里面可以使用 Service 去实现。

```shell
# 创建 Service
[root@master01 ~]# kubectl expose deployment nginx-deployment --port=30000 --target-port=80
service/nginx-deployment exposed

# 查看 Service
[root@master01 ~]# kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP     3d22h
nginx-deployment   ClusterIP   10.108.251.69   <none>        30000/TCP   9s

# 不断地访问 Service 的 IP ，可以看到已经在轮循了
[root@master01 ~]# curl http://10.108.251.69:30000/hostname.html
nginx-deployment-747c8b9b58-xmkjf
[root@master01 ~]# curl http://10.108.251.69:30000/hostname.html
nginx-deployment-747c8b9b58-lf84w
[root@master01 ~]# curl http://10.108.251.69:30000/hostname.html
nginx-deployment-747c8b9b58-8npjm
```

当然 `http://10.108.251.69:30000` 这个地址是内部地址，在集群外面是无法访问的。

那如果外部想访问要怎么办呢？

```shell
# 查看 Service Type，发现是 ClusterIP
[root@master01 ~]# kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP     3d22h
nginx-deployment   ClusterIP   10.108.251.69   <none>        30000/TCP   9s

# 把它改成 NodePort ，修改 yaml 文件里面的 Type，改成 NodePort
[root@master01 ~]# kubectl edit svc nginx-deployment
[root@master01 ~]# kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP           3d22h
nginx-deployment   NodePort    10.108.251.69   <none>        30000:30711/TCP   6m5s
```

可以看到改成 NodePort 后，给了一个随机端口：30711

然后我们去集群外访问下：http://192.168.0.114:30711/ ，不出意外这个是可以访问的。并且不但 master01 节点可以访问，其它节点也是可以访问的：

http://192.168.0.115:30711/

http://192.168.0.116:30711/

不信你试试

ok，到此，我猜你已对 K8s 充满兴趣并且迫不及待想要学习了呢？