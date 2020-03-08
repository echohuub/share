在上一篇文章中，我罗列了所有的 Pod 控制器，但未做进一步讲解。

这篇文章主要讲解 Deployment 控制器。由于 Deployment 本质上是通过 ReplicaSet 管理的 Pod ，所以这篇文章也会包含 ReplicaSet 的内容。

## 一、ReplicaSet

ReplicaSet 简称 RS ，它的主要作用就是用来确保 Pod 的副本数始终保持一致。即如果有 Pod 异常退出，会自动创建新的 Pod 来替代，而如果异常多出来的容器也会自动回收。

> 有些文章可能会提到 ReplicationController ，即 RC ，它跟 RS 没有本质不同，Kubernetes 官方建议使用 RS 代替 RC 部署容器，RS 支持集合式的 selector 

下面演示一个创建 RS 的模板信息：

rs.yaml

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: myapp
        image: heqingbao/k8s_myapp:v1
        ports:
        - containerPort: 80
```

先看一下清单文件的结构，可以理解为 Pod 模板外面套了一层 RS ，前一篇文章讲的 Pod 那那些探针等内容依然适用。

RS 还有很多复杂的标签，可以通过 `kubectl explain rs` 查看完整的模板信息。

此 RS 定义 Pod 的副本数为 3 个，下面来创建试试：

```shell
[root@master01 ~]# kubectl apply -f rs.yaml
replicaset.apps/frontend created

[root@master01 ~]# kubectl get pod
NAME             READY   STATUS    RESTARTS   AGE
frontend-cmnlh   1/1     Running   0          3s
frontend-dfzs5   1/1     Running   0          3s
frontend-vk2lt   1/1     Running   0          3s
```

可见，已经创建了 3 个 Pod 。

细心的你应该注意到了 RS 有个标签叫 `matchLabels`，而里面的 Pod 也刚好配置了 labels ，什么意思呢？

下面我们来修改下 label 试试：

```shell
[root@master01 ~]# kubectl label pod frontend-4gd7j tier=frontend1 --overwrite=True
pod/frontend-4gd7j labeled

[root@master01 ~]# kubectl get pod --show-labels
NAME             READY   STATUS    RESTARTS   AGE     LABELS
frontend-4gd7j   1/1     Running   0          2m31s   tier=frontend1
frontend-4w9kq   1/1     Running   0          2m31s   tier=frontend
frontend-9chbs   1/1     Running   0          7s      tier=frontend
frontend-x9vwz   1/1     Running   0          2m31s   tier=frontend
```

为什么会出现 4 个 Pod 呢？

因为 RS 会始终保持 match labels 的条件限制。即这里的 RS 会始终保持 Label= `tier=frontend` 的 Pod 的数量为 3 个。

对于 Kubernetes 来说， 它的很多副本数目监控都是以为 labels 为基础的。

## 二、Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义方法，用来替代以前的 ReplicationController 来方便的管理应用。

典型的应用场景包括：

* 定义 Deployment 来创建 Pod 和 ReplicaSet（Deployment 并不是直接管理 Pod，而是通过 RS 去管理 Pod）
* 滚动升级和回滚应用
* 扩容和缩容
* 暂停和继续 Deployment

下面来演示下

### 2.1 部署一个简单的应用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
     labels:
       app: myapp
    spec:
      containers:
      - name: myapp
        image: heqingbao/k8s_myapp:v1
        ports:
        - containerPort: 80
```

通过 Deployment 部署一个 3 个副本的 Pod 。

```shell
[root@master01 ~]# kubectl apply -f deployment.yaml
deployment.apps/myapp-deployment created

[root@master01 ~]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           10s

# Deployment 会创建对应的 RS
[root@master01 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-669d4c567d   3         3         3       15s

[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
myapp-deployment-669d4c567d-84lq8   1/1     Running   0          18s
myapp-deployment-669d4c567d-bmrkf   1/1     Running   0          18s
myapp-deployment-669d4c567d-d7d4h   1/1     Running   0          18s
```

从上面可以看出，Deployment 是通过 RS 间接管理 Pod 的。

> 注意 Pod 的名字结构：
>
> Deployment：myapp-deployment
>
> ReplicaSet：myapp-deployment-669d4c567d
>
> Pod：myapp-deployment-669d4c567d-84lq8
>
> 都是在上一层名称上加了个 hash



### 2.2 扩容

扩大到 10 个副本数

```shell
[root@master01 ~]# kubectl scale deployment myapp-deployment --replicas=10
deployment.apps/myapp-deployment scaled

[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
myapp-deployment-669d4c567d-84lq8   1/1     Running   0          40s
myapp-deployment-669d4c567d-9gprm   1/1     Running   0          5s
myapp-deployment-669d4c567d-bb2pb   1/1     Running   0          5s
myapp-deployment-669d4c567d-bmrkf   1/1     Running   0          40s
myapp-deployment-669d4c567d-cqt79   1/1     Running   0          5s
myapp-deployment-669d4c567d-d7d4h   1/1     Running   0          40s
myapp-deployment-669d4c567d-jbtnw   1/1     Running   0          5s
myapp-deployment-669d4c567d-kftpx   1/1     Running   0          5s
myapp-deployment-669d4c567d-tp4lh   1/1     Running   0          5s
myapp-deployment-669d4c567d-w8b8g   1/1     Running   0          5s
```

你会发现，一旦部署进 k8s 以后，应用的扩缩容（尤其是无状态应用的扩缩容）是非常简单的。

同时我们再查看 RS：

```shell
[root@master01 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-669d4c567d   10        10        10      48s
```

可以看到，RS 的名字并没有变，也即还是扩容前的 RS ，也就是说 RS 的模板没有被改变。所以这种 scale 的方式无法做到回滚（或回退）。

如果集群支持 HPA（Horizontal Pod Autoscaling）的话，还可以为 Deployment 设置自动扩展：

```shell
kubectl autoscale deployment myapp-deployment --min=10 --max=15 --cpu-percent=80
```



### 2.3 更新镜像

更新镜像也比较简单

```shell
# 把镜像升级到 v2 版本
[root@master01 ~]# kubectl set image deployment/myapp-deployment myapp=heqingbao/k8s_myapp:v2
deployment.apps/myapp-deployment image updated

# 发现创建了一个新的 RS ，并且所有的 Pod 都更新到了 v2 版本
[root@master01 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-669d4c567d   0         0         0       4m19s
myapp-deployment-7597b547d7   10        10        10      8s

[root@master01 ~]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE    IP               NODE     NOMINATED NODE   READINESS GATES
myapp-deployment-7597b547d7-2mld5   1/1     Running   0          118s   172.16.140.91    node02   <none>           <none>
myapp-deployment-7597b547d7-2x87k   1/1     Running   0          118s   172.16.196.158   node01   <none>           <none>
myapp-deployment-7597b547d7-7zn2g   1/1     Running   0          118s   172.16.196.157   node01   <none>           <none>
myapp-deployment-7597b547d7-d4gdj   1/1     Running   0          115s   172.16.140.93    node02   <none>           <none>
myapp-deployment-7597b547d7-ddmb8   1/1     Running   0          118s   172.16.196.156   node01   <none>           <none>
myapp-deployment-7597b547d7-g7q7n   1/1     Running   0          115s   172.16.196.159   node01   <none>           <none>
myapp-deployment-7597b547d7-jk827   1/1     Running   0          114s   172.16.140.94    node02   <none>           <none>
myapp-deployment-7597b547d7-nxq2h   1/1     Running   0          115s   172.16.196.160   node01   <none>           <none>
myapp-deployment-7597b547d7-wlv5s   1/1     Running   0          118s   172.16.140.90    node02   <none>           <none>
myapp-deployment-7597b547d7-xm7n6   1/1     Running   0          116s   172.16.140.92    node02   <none>           <none>

# 验证
[root@master01 ~]# curl 172.16.140.91/index.html
Hello MyApp | Version: v2 | <a href="hostname.html">Pod Name</a>
```

可以看到所有的 Pod 都已经全部更新到 v2 版本了。

这就是我们在 Deployment 里面更新镜像的方式，如果提交了新的代码，编译了新的镜像，，我们只需要这句简单的命令就可以把应用程序更新到最新版本，非常简单。



### 2.4 回滚

如果新的镜像有问题，还可以回滚：

```shell
[root@master01 ~]# kubectl rollout undo deployment/myapp-deployment
deployment.apps/myapp-deployment rolled back

[root@master01 ~]# kubectl get pod
NAME                                READY   STATUS        RESTARTS   AGE
myapp-deployment-669d4c567d-5h68s   1/1     Running       0          10s
myapp-deployment-669d4c567d-8h5m5   1/1     Running       0          6s
myapp-deployment-669d4c567d-d57bm   1/1     Running       0          10s
myapp-deployment-669d4c567d-d8bc9   1/1     Running       0          7s
myapp-deployment-669d4c567d-fkvtk   1/1     Running       0          6s
myapp-deployment-669d4c567d-ggq9x   1/1     Running       0          10s
myapp-deployment-669d4c567d-hsrts   1/1     Running       0          10s
myapp-deployment-669d4c567d-rjcvm   1/1     Running       0          6s
myapp-deployment-669d4c567d-tf94t   1/1     Running       0          8s
myapp-deployment-669d4c567d-tkj55   1/1     Running       0          10s
myapp-deployment-7597b547d7-2mld5   0/1     Terminating   0          5m18s
myapp-deployment-7597b547d7-7zn2g   0/1     Terminating   0          5m18s
myapp-deployment-7597b547d7-ddmb8   0/1     Terminating   0          5m18s
myapp-deployment-7597b547d7-g7q7n   0/1     Terminating   0          5m15s
myapp-deployment-7597b547d7-wlv5s   0/1     Terminating   0          5m18s

# 所有Pod已回滚到之前的RS管理了
[root@master01 ~]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
myapp-deployment-669d4c567d   10        10        10      9m41s
myapp-deployment-7597b547d7   0         0         0       5m30s

[root@master01 ~]# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
myapp-deployment-669d4c567d-5h68s   1/1     Running   0          56s   172.16.140.95    node02   <none>           <none>
myapp-deployment-669d4c567d-8h5m5   1/1     Running   0          52s   172.16.196.165   node01   <none>           <none>
myapp-deployment-669d4c567d-d57bm   1/1     Running   0          56s   172.16.196.162   node01   <none>           <none>
myapp-deployment-669d4c567d-d8bc9   1/1     Running   0          53s   172.16.196.164   node01   <none>           <none>
myapp-deployment-669d4c567d-fkvtk   1/1     Running   0          52s   172.16.140.99    node02   <none>           <none>
myapp-deployment-669d4c567d-ggq9x   1/1     Running   0          56s   172.16.196.161   node01   <none>           <none>
myapp-deployment-669d4c567d-hsrts   1/1     Running   0          56s   172.16.140.96    node02   <none>           <none>
myapp-deployment-669d4c567d-rjcvm   1/1     Running   0          52s   172.16.140.98    node02   <none>           <none>
myapp-deployment-669d4c567d-tf94t   1/1     Running   0          54s   172.16.196.163   node01   <none>           <none>
myapp-deployment-669d4c567d-tkj55   1/1     Running   0          56s   172.16.140.97    node02   <none>           <none>

# 验证
[root@master01 ~]# curl 172.16.140.95/index.html
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```

可见所有 Pod 都已经回退到 v1 版本了。

还可以回滚到指定版本：

```shell
# 查看历史版本
[root@master01 ~]# kubectl rollout history deployment/myapp-deployment
deployment.apps/myapp-deployment
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

# 回滚到指定版本
[root@master01 ~]# kubectl rollout undo deployment/myapp-deployment --to-revision=2
deployment.apps/myapp-deployment rolled back
```



可以使用 `kubectl rollout status ` 查看 Deployment 是否完成，如果 rollout 成功完成，`kubectl rollout status` 将返回一个 0 值的 Exit Code

```shell
[root@master01 ~]# kubectl rollout status deployment/myapp-deployment
deployment "myapp-deployment" successfully rolled out

[root@master01 ~]# echo $?
[root@master01 ~]# 0


```

### 2.5 清理策略

可以通过设置 `.spec.revisonHistoryLimit`项来指定 deployment 最多保留多少 revision 历史记录，默认会保留所有，如果设置为 0 ，Deployment 就不允许回退了。