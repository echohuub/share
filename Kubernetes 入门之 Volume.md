前面已经讲解了 ConfigMap 、Secret 用于保存配置或密钥 Key ，至今还没有涉及到 Pod 持久化的相关内容。

容器磁盘上的文件的生命周期是短暂的，当容器崩溃时，kubelet 会重启它，但是容器中的文件将丢失——容器以干净的状态（镜像最初的状态）重新启动。另外在 Pod 中同时运行多个容器时，这些容器之间通常需要共享文件。Kubernetes 中的 `Volume` 就很好的解决了这个问题。

## 一、卷的类型

Kubernetes 支持支持卷的类型有很多，比如 `AzureDisk`、`emptyDir`、`nfs`、`gitRepo`、`hostPath` 等。

下面只简单演示下 `emptyDir` 和 `hostPath` 的用法，其它的我也没试过，以后用到了再说。

## 二、emptyDir

当 Pod 被分配给节点时，首先创建 `emptyDir ` 卷，并且只要该 Pod 在该节点上运行，该卷就会存在。正如卷的名字所述，它最初是空的。Pod 中的容器可以读取和写入 `emptyDir` 卷中的相同文件，尽管该卷可以挂载到每个容器中的相同或不同路径上。当出于任何愿意从节点中删除 Pod 时，`emptyDir` 中的数据将永久删除。

emptyDir 的用法有：

* 暂存空间，例如用于基于磁盘的合并排序
* 用作长时间计算崩溃恢复时的检查点
* Web 服务器容器提供数据时，保存内容管理器容器提供的文件

下面简单演示下

empty.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - name: test-container
    image: heqingbao/k8s_myapp:v1
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir: {}
```

容器挂载了一个 `emptyDir` 类型的 Volume，挂载到了 `/cache` 目录

```shell
[root@master01 volume]# kubectl apply -f empty.yaml 
pod/test-pd created

[root@master01 volume]# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
test-pd   1/1     Running   0          8s

# 验证
[root@master01 volume]# kubectl exec -it test-pd sh
/ # cd /cache/
/cache # ls
```

可以看到已经挂载到容器的 `/cache` 目录下了。

当然 Pod 里面可以有多个容器，下面再演示下：

empty.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir-pod
spec:
  containers:
  - name: myapp1
    image: heqingbao/k8s_myapp:v1
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: myapp2
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "sleep 600"]
    volumeMounts:
    - name: cache-volume
      mountPath: /test
  volumes:
  - name: cache-volume
    emptyDir: {}
```

这个 Pod 有两个容器，都挂载了 `emptyDir` 的 Volume ，只是两个容器挂载的目录有区别。

下面看下效果：

```shell
[root@master01 volume]# kubectl apply -f emptydir.yaml 
pod/test-emptydir-pod created

[root@master01 volume]# kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
test-emptydir-pod   2/2     Running   0          4s

# 进入第一个容器，在挂载目录里创建了一个文件并打印了一句话
[root@master01 volume]# kubectl exec -it test-emptydir-pod -c myapp1 sh
/ # cd /cache/
/cache # echo `date` from myapp1 > time.txt 
/cache # cat time.txt 
Sun Mar 8 07:30:50 UTC 2020 from myapp1

# 进入第二个容器，在挂载目录里已经看到了内容，再加入一句话
[root@master01 volume]# kubectl exec -it test-emptydir-pod -c myapp2 sh
/ # cd /test/
/test # cat time.txt 
Sun Mar 8 07:30:50 UTC 2020 from myapp1
/test #
/test # echo `date` from myapp2 >> time.txt 
/test # cat time.txt 
Sun Mar 8 07:30:50 UTC 2020 from myapp1
Sun Mar 8 07:32:05 UTC 2020 from myapp2

# 再回到第一个容器查看内容
[root@master01 volume]# kubectl exec -it test-emptydir-pod -c myapp1 sh
/ # cat /cache/time.txt 
Sun Mar 8 07:30:50 UTC 2020 from myapp1
Sun Mar 8 07:32:05 UTC 2020 from myapp2
```

可见第一个容器挂载到了 `/cache` 目录，第二个容器挂载到了 `/test` 目录，并且挂载的目录已经共享了。



## 三、hostPath

`hostPath` 卷将主机节点的文件系统中的文件或目录挂载到集群中

`hostPath` 的用途如下：

* 运行需要访问 Docker 内部的容器；使用 `/var/lib/docker` 的 `hostPath`
* 在容器中运行 cAdvisor；使用 `/dev/cgroups` 的 `hostPath`

hostpath.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath-pod
spec:
  containers:
  - name: myapp
    image: heqingbao/k8s_myapp:v1
    volumeMounts:
    - name: hostpath-volume
      mountPath: /hostdata
  volumes:
  - name: hostpath-volume
    hostPath:
      path: /data
      type: Directory
```

上面容器中挂载了 `hostPath` 类型的 Volume ，主机的 `/data` 目录被挂载到了容器的 `/hostdata` 目录。

注意在创建 Pod 之前，需要先确保所有 Node 节点上存在 `/data` 目录，并且目录的权限需要跟运行 Docker 的权限一致。（我这里都是 root 身份运行的，所以不用额外修改） 

```shell
[root@node01 ~]# mkdir /data
[root@node02 ~]# mkdir /data
```

创建 Pod ：

```shell
[root@master01 volume]# kubectl apply -f hostpath.yaml 
pod/test-hostpath-pod created

[root@master01 volume]# kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
test-hostpath-pod   1/1     Running   0          2m57s
```

然后进入 Pod 在挂载目录里面创建文件，再验证是否同步到了主机上 `/data` 目录里。

```shell
# 进入容器挂载目录，创建文件
[root@master01 volume]# kubectl exec -it test-hostpath-pod sh
/ # cd /hostdata/
/hostdata # echo `date` > time.txt

# 去 node02 上查看目录内容（test-hostpath-pod运行在node02节点）
[root@node02 ~]# cd /data/
[root@node02 data]# cat time.txt 
Sun Mar 8 07:53:18 UTC 2020
```

可以看到 Pod 已经成功使用 `hostPath` ，并与主机文件目录实现了共享。