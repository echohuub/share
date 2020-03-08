上一篇文章讲解并演示了 Deployment 的使用，这篇文章来讲解下 DaemonSet 。

**什么是 DeamonSet 呢？**

DaemonSet 确保全部（或者一些）Node上运行一个 Pod 的副本。当有 Node 加入集群时，也会为它们新增一个 Pod，当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

* 运行集群存储 deamon，例如在每个 Node 上运行  `glusterd`、`ceph`
* 在每个 Node 上运行日志收集 deamon，例如 `fluentd`、`logstash`
* 在每个 Node 上运行监控 daemon，例如 `Prometheus Node Exporter`

下面演示下：

daemonset.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-example
  labels:
    app: daemonset
spec:
  selector:
    matchLabels:
      name: daemonset-example
  template:
    metadata:
      labels:
        name: daemonset-example
    spec:
      containers:
      - name: daemonset-example
        image: heqingbao/k8s_myapp:v1
```

上面清单文件定义了一个 DaemonSet ，下面来创建试试：

```shell
[root@master01 ~]# kubectl create -f daemonset.yaml
daemonset.apps/daemonset-example created

[root@master01 ~]# kubectl get daemonset
NAME                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset-example   2         2         2       2            2           <none>          5m3s

[root@master01 ~]# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
daemonset-example-8hbfl   1/1     Running   0          5m36s   172.16.140.105   node02   <none>           <none>
daemonset-example-x8szq   1/1     Running   0          5m36s   172.16.196.171   node01   <none>           <none>
```

可以看到两个 DaemonSet 分别运行在两个节点上，你可能会问，为什么主节点没有被创建 DaemonSet 呢？不是说所有节点上都会创建吗？

这跟调度机制里面的污点概念有关，Scheduler 在调度 Pod 的时候会有一些策略去决定 Pod 应该运行在哪个节点上，感兴趣的同学可以研究下。

同时你会发现所有的 Pod 都不会在主节点上运行，一样的原理。

如果我把 node01 节点上的 Pod 删掉：

```shell
[root@master01 ~]# kubectl delete pod daemonset-example-x8szq
pod "daemonset-example-x8szq" deleted

[root@master01 ~]# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
daemonset-example-7c75q   1/1     Running   0          16s     172.16.196.172   node01   <none>           <none>
daemonset-example-8hbfl   1/1     Running   0          6m36s   172.16.140.105   node02   <none>           <none>
```

可以看到 node01 上的 Pod 又被创建了，因为它要一直满足副本数为 1 的状态。