前一篇文章讲了 Volume ，除了 Volume 外，Kubernetes 还提供了 Persistent Volume 的方法。Volume 主要是为了存储一些有必要保存或共享的数据，而 Persistent Volume 主要是为了管理集群的存储。

## 一、概念

#### 1.1 PersistentVolume （PV）

PV 也是节点中的资源，类似 Volume 之类的卷插件，但它具有独立于 Pod 的生命周期，当 Pod 被删除，PV 仍然会被保留。

#### 1.2  PersistentVolumeClaim （PVC）

是用户存储的请求，它与 Pod 相似，Pod 消息节点资源，PVC 消耗 PV 资源。Pod 可以请求特定级别的资源（CPU 和内存），声明可以请求特定的大小和访问模式（例如，可以以读/写一次或只读多次模式挂载）

#### 2.3 持久化卷类型

PV 类型以插件形式实现，K8s 目前已经支持了很多种插件类型，比如 AzureFile 、AzureDisk、NFS 等，具体可查阅官方文档。

下面的示例将演示 NFS 方式。

#### 2.4 PV 访问模式

PV 可以以资源提供者支持的任何方式挂载到主机上，既定的模式有：

* ReadWriteOnce —— 该卷可以被单个节点以读/写模式挂载
* ReadOnlyMany —— 该卷可以被多个节点以只读模式挂载
* ReadWriteMany —— 该卷可以被多个节点以读/写模式挂载

#### 2.5 状态

PV 可以处于以下的某种状态：

* Available（可用）—— 一块空闲资源还没有被任何声明绑定
* Bound（已绑定） —— 卷已经被声明绑定
* Release（已释放） —— 声明被删除，但是资源还未被集群重新声明
* Failed（失败） —— 该卷的自动回收失败



## 二、持久化演示说明 - NFS

### 2.1 安装 NFS 服务器

直接在 node02 上安装 NFS 服务器：

```shell
[root@node02 ~]# yum install -y nfs-common nfs-utils rpcbind
[root@node02 ~]# mkdir /nfsdata
[root@node02 ~]# chmod 777 /nfsdata/

[root@node02 ~]# cat > /etc/exports << EOF
/nfsdata *(rw,no_root_squash,no_all_squash,sync)
EOF

# 开启 NFS
[root@node02 ~]# systemctl start rpcbind
[root@node02 ~]# systemctl start nfs
```

然后需要在 K8s 集群中的所有节点为都安装 NFS 的客户端：

```shell
yum install -y nfs-utils rpcbind
```

然后我们随便找个节点测试挂载这个 NFS ，确保 NFS 没有问题。

我这里直接在 master01 节点上验证：

```shell
# 创建本地挂载路径
[root@master01 volume]# mkdir /test

# 显示远程挂载点
[root@master01 volume]# showmount -e 192.168.0.116
Export list for 192.168.0.116:
/nfsdata *

# 挂载到 /test 目录
[root@master01 volume]# mount -t nfs 192.168.0.116:/nfsdata /test

# 验证
[root@master01 volume]# cd /test/
[root@master01 test]# echo hello > test.txt
[root@master01 test]# cat test.txt 
hello
```

到此 NFS 挂载验证成功。

下面将使用 Pod 的方式进行挂载，在操作前先在 master01 上清除挂载：

```shell
[root@master01 /]# umount /test
[root@master01 /]# rm -rf /test/
```

### 2.2 部署 PV

为了演示方便，使用上面创建 NFS 的方式创建三个 NFS 目录，即 `/nfsdata` 、`/nfsdata2` 、`/nfsdata3`

然后创建三个 pv ：

pvs.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfsdata
    server: 192.168.0.116
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv2
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfsdata2
    server: 192.168.0.116
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv3
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: slow
  nfs:
    path: /nfsdata3
    server: 192.168.0.116

```

类型为：PersistentVolume

存储容量为 1GB、5GB、10GB

访问模式：ReadWriteOnce、ReadOnlyMany、ReadWriteMany

回收策略：Retain

类的名称：nfs、slow

NFS 地址是：192.168.0.116 的 `/nfsdata`、 `/nfsdata2`、 `/nfsdata3`

下面来创建这三个 PV ：

```shell
[root@master01 pv]# kubectl create -f pvs.yaml 
persistentvolume/nfspv1 created
persistentvolume/nfspv2 created
persistentvolume/nfspv3 created
[root@master01 pv]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfspv1   1Gi        RWO            Retain           Available           nfs                     9s
nfspv2   5Gi        ROX            Retain           Available           nfs                     9s
nfspv3   10Gi       RWX            Retain           Available           slow                    9s
```

PV 已经创建成功。

这个 PV 其实已经可以挂载到 Pod 去使用，但正常情况下不会这样去用，而是会采用 PVC 的方案去调用。

### 2.3 创建服务并使用 PVC

.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: heqingbao/k8s_myapp:v1
        ports:
        - name: web
          containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 0.5Gi
```

首先创建了一个 Headless Service（`  clusterIP: None`），因为要创建 `StatefulSet` 控制器必须要创建一个无头服务。

然后 StatefulSet 声明了一个挂载点，并被挂载到了容器里面。

这里需要注意卷的声明模板 `volumeClaimTemplates` ：

* 访问模式必须是 ReadWriteOnce
* 类必须是 nfs
* 存储容器至少 0.5GB

创建这个 StatefulSet ：

```shell
[root@master01 pv]# kubectl apply -f statefulset.yaml 
service/nginx unchanged
statefulset.apps/web created

[root@master01 pv]# kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          9s
web-1   0/1     Pending   0          5s
```

可以看到 web-1 这个 Pod 始终 Pending ，我们查看下原因：

```shell
[root@master01 pv]# kubectl describe pod web-1
...
 Warning  FailedScheduling  68s (x3 over 3m39s)  default-scheduler  error while running "VolumeBinding" filter plugin for pod "web-1": pod has unbound immediate PersistentVolumeClaims
```

可以发现是 PV 绑定失败。

我们再看下这个 StatefulSet 需要绑定的 PV 要求：

* 访问模式必须是 ReadWriteOnce
* 类必须是 nfs
* 存储容器至少 0.5GB

而现在 PV 的状态是：

```shell
[root@master01 pv]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS   REASON   AGE
nfspv1   1Gi        RWO            Retain           Bound       default/www-web-0   nfs                     6m37s
nfspv2   5Gi        ROX            Retain           Available                       nfs                     6m37s
nfspv3   10Gi       RWX            Retain           Available                       slow                    6m37s
```

发现能够同时满足上面三个要求的只有一个 PV ，并且可以看到已经被其中一个 Pod 副本绑定了。

那么其它两个 Pod 是无法绑定成功的。

同时我们发现，第三个 Pod 根本就没有创建，还记得之前讲过 StatefulSet 是按序创建的吗？前一个运行成功以后下一个才能够被创建。

为了验证，我们把 `nfspv2` 和 `nfspv3` 改一下，让它满足 StatefulSet 的要求：

```shell
# 先删除 nfspv2 和 nfspv3 
[root@master01 pv]# kubectl delete pv nfspv2
persistentvolume "nfspv2" deleted
[root@master01 pv]# kubectl delete pv nfspv3
persistentvolume "nfspv3" deleted
```

再创建两个新的 PV :

pvs.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv2
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfsdata2
    server: 192.168.0.116
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfspv3
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    path: /nfsdata3
    server: 192.168.0.116
```

这两个 PV 的 `accessModes=ReadWriteOnce` 并且 `storageClassName=nfs`

创建 PV 并验证：

```shell
[root@master01 pv]# kubectl create -f pvs.yaml 
persistentvolume/nfspv2 created
persistentvolume/nfspv3 created

# 验证
[root@master01 pv]# kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
nfspv1   1Gi        RWO            Retain           Bound    default/www-web-0   nfs                     15m
nfspv2   5Gi        RWO            Retain           Bound    default/www-web-1   nfs                     13s
nfspv3   10Gi       RWO            Retain           Bound    default/www-web-2   nfs                     13s

# 验证
[root@master01 pv]# kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          16m
web-1   1/1     Running   0          16m
web-2   1/1     Running   0          2m21s
```

可以看到三个 Pod 都创建成功并且绑定到了 PV 。



下面查看下 `nfspv1` 对应的目录：

```shell
[root@master01 pv]# kubectl describe pv nfspv1
Name:            nfspv1
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    nfs
Status:          Bound
Claim:           default/www-web-0
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:         
Source:
    Type:      NFS (an NFS mount that lasts the lifetime of a pod)
    Server:    192.168.0.116
    Path:      /nfsdata
    ReadOnly:  false
Events:        <none>
```

对应的是 `/nfsdata ` 目录，我们去这个目录下创建个文件试试：

```shell
[root@node02 nfsdata]# echo "hello world" > index.html
```

由上面可知，这个 PV 被绑定到了 `web-0` 这个 Pod ，下面来访问这个 Pod 试试：

```shell
[root@master01 pv]# curl 172.16.140.79
hello world
```

到此 Pod 已经使用了 PV 目录下的文件了。

之前说过 StatefulSet 可以保存状态，下面来删除一个 Pod 验证下：

```shell
[root@master01 pv]# kubectl delete pod web-0
pod "web-0" deleted

[root@master01 pv]# kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
web-0   1/1     Running   0          7s    172.16.140.82    node02   <none>           <none>
web-1   1/1     Running   0          29m   172.16.196.132   node01   <none>           <none>
web-2   1/1     Running   0          15m   172.16.140.81    node02   <none>           <none>

[root@master01 pv]# curl 172.16.140.82
hello world
```

可以看到重新创建了 Pod ，并且访问仍然是成功的。

同时新的 Pod 的 IP 地址已改变，之前说过，StatefulSet 为每个 Pod 副本创建了 DNS 域名，Pod 之间可以使用域名进行通讯，之前演示过，这里就不再演示了。

## 三、关于 StatefulSet

#### 3.1 关于 PV / PVC 的补充说明：

* 匹配 Pod Name （网络标识）的模式为：$(statefulset名称)-$(序号)，如果上面的示例：web-0、web-1、web-2
* StatefulSet 为每个 Pod 副本创建了一个 DNS 域名，这个域名的格式为：$(podname).(headless server name) ，也就意味着服务间是通过 Pod 域名来通讯而非 Pod IP ，因为当 Pod 所在的 Node 发生故障时，Pod 会被飘移到其它  Node 上，Pod IP 会发生变化 ，但是 Pod 域名不会有变化 。
* StatefulSet 使用 Headless 服务来控制 Pod 的域名，这个域名的 FQDN 为 ：$(service.name).$(namespace).svc.cluster.local，其中 "cluster.local" 指的是集群的域名
* 根据 volumeClaimTemplates ，为每个 Pod 创建一个 PVC ，PVC 的命名规则匹配模式：(volumeClaimTemplates.name)-(pod_name)，比如上面的 volumeMounts.name=www， Pod Name=web-[0-2]，因此创建出来的 PVC 是 www-web-0、www-web-1、www-web-2
* 删除 Pod 不会删除其 PVC ，手动删除 PVC 将自动释放 PV

#### 3.2 StatefulSet 的启停顺序：

* 有序部署：部署 StatefulSet 时，如果有多个 Pod 副本，它们会被顺序地创建（从 0 到 N-1 ），并且，在下一个 Pod 运行之前所有之前的 Pod 都必须 是 Running 或者 Ready 状态
* 有序删除：当 Pod 被删除时，它们被终止的顺序是从 N-1 到 0
* 有序扩展：当对 Pod 执行扩展操作时，与部署一样，它前面的 Pod 必须都处于 Running 或 Ready 状态



#### 3.3 StatefulSet 使用场景：

* 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 实现
* 稳定的网络标识符，即 Pod 重新调度后期 PodName 和HostName 不变
* 有序部署，有序扩展，基于 init containers 来实现
* 有序收缩