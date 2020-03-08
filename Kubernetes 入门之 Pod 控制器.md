在上一篇文章中，我们一起了解了 Kubernetes 提供的资源，重点讲解了通过 yaml 的方式创建 Pod ，以及 Pod 和容器的生命周期。

这种创建 Pod 的方式我们称为 **自主式创建 Pod **，当 Pod 退出了，此类型的 Pod 不会被创建，如上一节讲的那样，我使用 `kubectl delete pod podname` 删除 Pod 后不会自动创建。

还有一种创建 Pod 的方式我们称为 **控制器管理的 Pod**， 通过字面意思应该能想到，这种 Pod 会被管理起来，始终要维持 Pod 副本的数量，即当 Pod 删除之后会被自动创建。

通过这篇文章你将了解到这些 **控制器** 

## 一、什么是控制器

Kubernetes 中内建了很多 controller（控制器），这些相当于一个状态机，用来控制 Pod 的具体状态和行为。

## 二、控制器类型

* ReplicationController 和 ReplicaSet
* Deployment
* DaemonSet
* StatefulSet
* Job/CronJob
* Horizontal Pod Autoscaling

### 2.1 ReplicationController 和 ReplicaSet

ReplicationController（RC）用来确保容器应用的副本数始终保持在用户定义的副本数，即如果有容器异常退出，会自动创建新的 Pod 来替代；而如果异常多出来的容器也会自动回收。

在新版本的 Kubernetes 中建议使用 ReplicaSet 来取代 ReplicationController，ReplicaSet 跟 ReplicationController 没有本质的不同，只是名字不一样，并且 ReplicaSet 支持集合式的 selector。

### 2.2 Deployment

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义方法，用来替代以前的 ReplicationController 来方便的管理应用。

> 命令式和声明式
>
> 命令式编程：它侧重于如何实现程序，就像我们刚接触编程的时候那样，我们需要把程序的实现过程按照逻辑结果一步步写下来。 
>
> 声明式编程：它侧重于想要什么，然后告诉计算机/引擎，让它帮你去实现。

典型的应用场景包括：

* 定义 Deployment 来创建 Pod 和 ReplicaSet（Deployment 并不是直接管理 Pod，而是通过 RS 去管理 Pod）
* 滚动升级和回滚应用
* 扩容和缩容
* 暂停和继续 Deployment

### 2.3 DeamonSet

DaemonSet 确保全部（或者一些）Node上运行一个 Pod 的副本。当有 Node 加入集群时，也会为它们新增一个 Pod，当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

使用 DaemonSet 的一些典型用法：

* 运行集群存储 deamon，例如在每个 Node 上运行  `glusterd`、`ceph`
* 在每个 Node 上运行日志收集 deamon，例如 `fluentd`、`logstash`
* 在每个 Node 上运行监控 daemon，例如 `Prometheus Node Exporter`

### 2.4 Job

Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

### 2.5 CronJob

Cron Job 管理基于时间的 Job，即：

* 在给定时间点只运行一次
* 周期性地在给定时间点运行

典型的用法示例：

* 在给你写的时间点调度 Job 运行
* 创建周期性运行的 Job，例如：数据库备份、发送邮件

### 2.6 StatefulSet

StatefulSet 作为 Controller 为 Pod 提供唯一的标识，它可以保证部署和 scale 的顺序。

StatefulSet 是为了解决有状态服务的问题（对应 Deployment 和 ReplicaSet 是为无状态服务而设计），其应用场景包括：

* 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现
* 稳定的网络标识，即 Pod 重新调度后其 Pod Name 和 Host Name 不变，基于 Headless Service （即没有 Cluster IP 的 Service）来实现
* 有序部署、有序扩展，即 Pod 是有顺序的，在部署或者扩展的时候要住所定义的顺序依次进行（即从 0 到 N-1，在下一个 Pod 运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态），基于 init containers 来实现
* 有序收缩，有序删除（即从 N-1 到 0 ）

### 2.7 Horizontal Pod Autoscaling

顾名思义，使 Pod 水平自动缩放，提高集群的整体资源利用率。

Horizontal Pod Autoscaling 仅适用于 Deployment 和 ReplicaSet。在 v1 版本中仅支持根据 Pod 的 CPU 利用率扩缩容，在 v1alpha 版本中，支持根据内存和用户自定义的 metric 扩缩容。



下篇文章将对以上几种类型的控制器单独讲解。