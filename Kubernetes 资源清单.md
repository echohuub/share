

在上一篇文章中，我使用 `kubectl run` 和 `kubectl expose` 去创建 Deployment 和暴露服务，这不是常规的使用方案。

一般来说都是使用资源清单的方式，资源清单可以简单理解成一个剧本，里面已经写好每一步应该怎么做，K8s 在接收到这个剧本后会按照既定的步骤去执行。

这篇文章就是讲解 Kubernetes 为我们提供了哪些可以使用的资源。

## 一、Kubernetes 中的资源

K8s 中的资源可以分为三类：

#### 1. 名称空间级别

* **工作负载型资源（ workload）**

Pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、Job、CronJob

* **服务发现及负载均衡型资源（ServiceDiscovery LoadBalance）**

Service、Ingress、...

* **配置与存储型资源**

Volume（存储卷）、CSI（容器存储接口，可以扩展各种各样的第三方存储卷）

* **特殊类型的存储卷**

ConfigMap（当配置中心来使用的资源类型）、Secret（保存敏感数据）、DownwardAPI（所外部环境中的信息输出给容器）

#### 2. 集群级别

Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding

#### 3. 元数据型

HPA、PodTemplate、LimitRange

## 二、资源清单

在 K8s 中，使用 yaml 格式的文件来创建符合我们期望的 Pod ，这样的 yaml 文件称为资源清单。具体的语法就不在这里讲了，后面结合演示的 Case 可以看一下。

## 三、常用字段解释

在定义资源清单的时候有哪些常用的字段

**必填字段：**

| 参数名                  | 字段类型 | 说明                                                         |
| ----------------------- | -------- | :----------------------------------------------------------- |
| version                 | String   | 指 K8s API 的版本，目前基本上是 v1 ，可以用 `kubectl api-versions` 命令查询 |
| kind                    | String   | 指 yaml 文件定义的资源类型和角色，比如：Pod                  |
| metadata                | Object   | 元数据对象                                                   |
| metadata.name           | String   | 元数据对象的名字，比如命名 Pod 的名字                        |
| metadata.namespace      | String   | 元数据对象的命名空间（默认default）                          |
| spec                    | Object   | 详细定义对象                                                 |
| spec.containers[]       | List     | 容器列表的定义                                               |
| spec.containers[].name  | String   | 容器的名字                                                   |
| spec.containers[].image | String   | 容器镜像的名称                                               |

**主要字段：**

| 参数名                                      | 字段类型 | 说明                                                         |
| ------------------------------------------- | -------- | :----------------------------------------------------------- |
| spec.containers[].imagePullPolicy           | String   | 定义镜像的拉取策略，有Always、Never、IfNotPresent三个值可选，（1）Always：意思是每次都尝试重新拉取镜像，（2）Never：表示仅使用本地镜像，（3）IfNotPresent：如果本地有镜像就使用本地镜像，没有就拉取在线镜像。上面三个值都没设置的话，默认是Always。 |
| spec.containers[].command[]                 | List     | 指定容器启动命令，因为是数组可以指定多个，不指定则使用镜像打包时使用的启动命令。 |
| spec.containers[].args[]                    | List     | 批定容器启动命令参数，因为是数组可以指定多个。               |
| spec.containers[].workingDir                | String   | 指定容器的工作目录                                           |
| spec.containers[].volumeMounts[]            | List     | 指定容器内部的存储卷位置                                     |
| spec.containers[].volumeMounts[].name       | String   | 指定可以被容器挂载的存储卷的名称                             |
| spec.containers[].volumeMounts[].mountPath  | String   | 指定可以被挂载的存储卷的路径                                 |
| spec.containers[].volumeMounts[].readOnly   | String   | 设置存储卷路径的读写模式，true或者false，默认为读写模式      |
| spec.containers[].ports[]                   | List     | 指定容器需要用到的端口列表                                   |
| spec.containers[].ports[].name              | String   | 指定端口名称                                                 |
| spec.containers[].ports[].containerPort     | String   | 指定容器需要监听的端口号                                     |
| spec.containers[].ports[].hostPort          | String   | 指定容器所在主机需要监听的端口号，默认跟上面containerPort相同，注意设置了hostPort同一台主机无法启动该容器的相同副本（会端口冲突） |
| spec.containers[].ports[].protocol          | String   | 指定端口协议，支持TCP和UDP，默认为TCP                        |
| spec.containers[].env[]                     | List     | 指定容器运行前需要设置的环境变量列表                         |
| spec.containers[].env[].name                | String   | 指定环境变量名称                                             |
| spec.containers[].env[].value               | String   | 指定环境变量值                                               |
| spec.containers[].resources                 | Object   | 指定资源限制和资源请求的值（这里开始就是设置容器的资源上限） |
| spec.containers[].resources.limits          | Object   | 指定设置容器运行时资源的运行上限                             |
| spec.containers[].resources.limits.cpu      | String   | 指定CPU限制，单位为core数，将用于docker run --cpu-shares参数 |
| spec.containers[].resources.limits.memory   | String   | 指定MEM内存的限制，单位为MIB、GiB                            |
| spec.containers[].resources.requests        | Object   | 指定容器启动和调度时的限制设置                               |
| spec.containers[].resources.requests.cpu    | String   | CPU请求，单位为core数，容器启动时初始化可用数量              |
| spec.containers[].resources.requests.memory | String   | 内存请求，单位为MIB、GiB，容器启动时初始化可用数量           |

**额外字段：**

| 参数名                | 字段类型 | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| spec.restartPolicy    | String   | 定义Pod的重启策略，可选值为Always、OnFailure、默认为Always。<br/> 1. Always：Pod一旦终止运行，则无论容器是如何终止的，kubelet服务都将重启它<br/> 2.OnFailure：只有Pod以非零退出码终止时，kubelet才会重启该容器。如果容器正常结束（退出码为0），则kubelet不会重启它。<br/> 3.Never：Pod终止后，kubelet将退出码报告给master，不会重启该Pod |
| spec.nodeSelector     | Object   | 定义Node的Label过滤标签，以key:value格式指定                 |
| spec.imagePullSecrets | Object   | 定义pull镜像时使用secret名称，以name:secretkey格式指定       |
| spec.hostNetwork      | Boolean  | 定义是否使用主机网络模式，默认值是false，设置true表示使用主机网络，不使用docker网桥，同时设置了true将无法在同一台宿主机上启动第二个副本 |

以上命令也是来自网络整理，大概有个印象即可，需要的时候再来查询。

也可以通过 `kubectl explain [name]` 命令查看每个字段的解释和用法，比如：

```shell
# kubectl explain pod
# kubectl explain pod.spec
# kubectl explain pod.spec.containers
```

下面演示通过一个简单的资源清单文件创建 Pod：

pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: heqingbao/k8s_myapp:v1
```

```shell
[root@master01 ~]# kubectl create -f pod.yaml
pod/myapp-pod created

[root@master01 ~]# kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
myapp-pod   1/1     Running   0          5s
```

如果 Pod 创建失败，可以简单使用以下两个命令查看日志：

* `kubectl describe podname` 可以查看 Pod 的运行状态
* `kubectl logs podname` 查看容器里面的日志信息，如果 Pod 里面有多个容器，后面还需要跟上容器名



## 四、容器生命周期

K8s 管理 Pod ，Pod 里面运行容器，容器里面运行业务进程，比如 Nginx 等。所以理解容器的生命周期很重要。

下面先简单讲述下从 kubectl 提交资源清单到 Pod 创建完成的过程：

1. API Server 在接收到创建 Pod 的请求之后，会根据用户提交的参数值来创建一个运行时的 Pod 对象，并会做一些准备工作（比如验证 namespace 、检查 Pod 名字以及必填字段信息），然后会在 etcd 中持久化这个对象，将异步调用结果封装成 Restful 响应完成结果反馈。此时 Pod 处于 Pending 状态。

2. 接下来该 Scheduler 上场了，它主要是完成 Pod 的调度，决定 Pod 应该运行在集群的哪个节点上。注意，第 1 步 API Server 完成任务之后，将信息写入到了 etcd 中，此时 scheduler 通过 watch 机制监听 etcd 的信息改变然后再进行工作。Scheduler 读取到写入 etcd 中的 Pod 信息，然后基于一系列规则 从集群中挑选一个合适的节点来运行它。比如预选、优选等方案。

   当 Scheduler 通过一系列策略选定 Pod 运行节点之后，会将结果信息提交至 API Server，再由 API Server 更新到 etcd 中，并由 API Server 响应调度结果给 Scheduler。接下来由 Kubelet 在所选定的节点上启动 Pod 。

3. Kubectl 通过 API Server 监听 etcd 目录，同步 Pod 列表，如果发现有新的 Pod 绑定到本节点，则按照 Pod 清单要求创建 Pod ，如果发现是 Pod 更新，则做出相应更改。



读取到 Pod 的信息之后，如果是创建或修改 Pod 的任务，则做如下处理：

1. 为该 Pod 创建一个数据目录
2. 从 API Server 读取该 Pod 清单
3. 为该 Pod 挂载外部卷
4. 下载 Pod 所需的 Secret
5. 检查已经运行在节点中的 Pod ，如果该 Pod 没有容器或者 pause 容器没有启动，则先停止 Pod 里面所有的容器进程。
6. 使用 pause 镜像为每个 Pod 创建一个容器，该容器用于接管 Pod 中所有其他容器的网络。
7. 调用 docker client 下载容器镜像，并启动容器。

在容器的创建过程中，可以为 Pod 对象定义生命周期中的多种行为，比如初始化容器、容器探测以及钩子，下面分别展开说明。

### 4.1 Init 初始容器

初始化容器即 Pod 内主容器启动之前要运行的容器，主要是做一些前置工作，下面直接称为 Init 容器。

Init 容器与普通的容器非常像，除了以下两点：

* Init 容器总是运行到成功完成为止
* 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成

如果 Pod 的 Init 容器启动失败，Kubernetes 会不断地重启该 Pod ，直到 Init 容器成功为止。如果 Pod 对应的 restartPolicy 为 Never ，它失败后不会再重新启动。



因为 Init 容器具有与应用程序容器分离的单独镜像，所以它们的启动相关代码具有如下优势：

* 它们可以包含并运行实用工具。（比如在主容器启动之前，需要有一些文件被创建）
* 应用程序镜像可以分离出创建和部署的角色，而没有必要联合它们构建一个单独的镜像。
* Init 容器使用 Linux Namespace ，所以相对应用程序容器来说具有不同的文件系统视图。因此，它们能够具有访问 Secret 的权限，而应用程序容器则不能。
* 它们必须在应用程序容器启动之前运行完成，而应用程序容器是并行运行的，所以 Init 容器能够提供了一种简单的阻塞或延时应用容器启动的方法，直到满足了一组先决条件。（比如一个 Pod 里面有两个容器，分别运行 mysql 和 apache+php，可以在 apache+php 容器的 Init 检测 mysql 的就绪状态，只有 mysql 就绪了才能创建 apache+php 容器）

**下面演示一下**

init-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 360']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

myapp-container 容器启动后打印一段话，然后休眠 6 分钟，注意声明了两个 Init 初始化容器：

* init-myservice 
* init-mydb

这两个容器都是通过 nslookup dns 查询对应的服务是否存在，如果存在则成功（返回 0 ），不存在则睡眠 2 秒后再重复检查。

```shell
[root@master01 ~]# kubectl apply -f init-pod.yaml
pod/myapp-pod created

[root@master01 ~]# kubectl get pod
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:0/2   0          34s
```

可以看出 myapp-pod 这个 Pod 并没有就绪，总共两个 Init ，一个都没有成功。

```shell
# 查看容器Log，注意这里Pod里面有两个容器，需要指定容器名
[root@master01 ~]# kubectl logs myapp-pod init-myservice
Server:		10.96.0.10
Address:	10.96.0.10:53

** server can't find myservice.default.svc.cluster.local: NXDOMAIN

*** Can't find myservice.svc.cluster.local: No answer
*** Can't find myservice.cluster.local: No answer
*** Can't find myservice.default.svc.cluster.local: No answer
*** Can't find myservice.svc.cluster.local: No answer
*** Can't find myservice.cluster.local: No answer

waiting for myservice
```

可以看出第一个 Init 容器（init-myservice）没有成功。

那下面我们创建一个Service：init-myservice 试试

myservice.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myservice
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

```shell
[root@master01 ~]# kubectl apply -f myservice.yaml
service/myservice created

[root@master01 ~]# kubectl get pod
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:1/2   0          17m
```

可以看到 Init 状态里面成功了 1 个，另一个还没成功，这里我就不演示了，感兴趣的同学自己查看 log 吧。

下面我直接再创建一个 Service：mydb

mydb.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mydb
spec:
  prots:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

```shell
[root@master01 ~]# kubectl apply -f mydb.yaml
service/mydb created

[root@master01 ~]# kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
myapp-pod   1/1     Running   0          21m
```

ok，我们可以看到 Pod 已经处于 Running 状态了，因为两个 Init 容器已经执行成功。

同时我们看下创建的两个 Service

```shell
[root@master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   6d17h
mydb         ClusterIP   10.111.19.63     <none>        80/TCP    8m42s
myservice    ClusterIP   10.105.234.255   <none>        80/TCP    14m
```



**再补充几点说明**

* 在 Pod 启动过程中，Init 容器会按顺序在网络和数据卷初始化（即 pause 容器）之后启动。每个 Init 容器必须在上个容器成功退出后再启动。
* 如果由于运行时或失败退出，将导致容器启动失败，它会根据 Pod 的 restartPolicy 指定的策略进行重试。如果 Pod 的 restartPolicy 设置为 Always，Init 容器失败时会使用 RestartPolicy 策略。
* 在所有的 Init 容器没有成功之前，Pod 将不会变成 Ready 状态。
* 如果 Pod 重启，所有 Init 容器必须重新执行
* 更改 Init 容器的 image 字段，会重启该 Pod
* Init 容器具有应用容器的所有字段，除了 readinessProbe 。因为它一完成后就退出了。
* 在 Pod 中的每个 app 和 Init 容器的名称必须唯一。



### 4.2 容器探测

容器探测分为存活性探测和就绪性探测容器探测是kubelet对容器健康状态进行诊断，容器探测的方式主要以下三种：

* ExecAction：在容器中执行命令，根据返回的状态码判断容器健康状态，返回0即表示成功，否则为失败。
* TCPSocketAction：通过与容器的某TCP端口尝试建立连接进行诊断，端口能打开即为表示成功，否则失败。
* HTTPGetAction：向容器指定 URL 发起 HTTP GET 请求，响应码为2xx或者是3xx为成功，否则失败。

每次探测都将获得以下三种结果之一：

* 成功：容器通过了诊断
* 失败：容器未通过诊断
* 未知：诊断失败，不会采取任何行动

#### 4.2.1 就绪探测（readinessProbe）

指示容器是否准备好服务请求。如果就绪结果探测失败，端口控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 Failure 。如果容器不提供就绪探针，则默认状态为 Success 。

下面演示下 `HTTPGetAction` 方式就绪探测

read.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
spec:
  containers:
  - name: readiness-httpget-container
    image: heqingbao/k8s_myapp:v1
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1
      periodSeconds: 3
```

添加了就绪检测，在容器启动 1 秒后才开始检测，间隔 3 秒重试一下。因为这个容器里根本没有 index1.html ，所以检测会失败。

```shell
[root@master01 ~]# kubectl apply -f read.yaml
pod/readiness-httpget-pod created

[root@master01 ~]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   0/1     Running   0          31s
```

虽然显示的是 Running ，但是没有就绪。可以查看下原因：

```shell
[root@master01 ~]# kubectl describe pod readiness-httpget-pod
...
 Warning  Unhealthy  3m36s (x22 over 4m39s)  kubelet, node02    Readiness probe failed: HTTP probe failed with statuscode: 404
```

可以看到就绪检测失败了，再来看下 Log ：

```shell
[root@master01 ~]# kubectl logs readiness-httpget-pod
2020/02/29 12:40:18 [error] 6#6: *22 open() "/usr/share/nginx/html/index1.html" failed (2: No such file or directory), client: 192.168.0.116, server: localhost, request: "GET /index1.html HTTP/1.1", host: "172.16.140.73:80"
192.168.0.116 - - [29/Feb/2020:12:40:18 +0000] "GET /index1.html HTTP/1.1" 404 169 "-" "kube-probe/1.17" "-"
```

发现 index1.html 资源不存在。

那我们直接进入容器里创建一个 index1.html 试试：

```shell
[root@master01 ~]# kubectl exec readiness-httpget-pod -it -- /bin/sh
/ # echo "hello" >> /usr/share/nginx/html/index1.html
/ # exit

# 再检查Pod状态
[root@master01 ~]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
readiness-httpget-pod   1/1     Running   0          7m31s
```

发现 Pod 已经成功 Ready 了。

#### 4.2.2 存活检测（livenessProbe）

指示容器是否正在运行。如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其重启策略的影响。如果容器不提供存活探针，则默认状态为 Success 。

**演示 `ExecAction` 方式的存活检测**

live-exec.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
  namespace: default
spec:
  containers:
  - name: liveness-exec-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "touch /tmp/live; sleep 60; rm -rf /tmp/live; sleep 3600"]
    livenessProbe:
      exec:
        command: ["test", "-e", "/tmp/live"]
      initialDelaySeconds: 1
      periodSeconds: 3
```

添加了一个存活检测，如果 `/tmp/live` 文件存在则存活，否则不存活，在容器启动 1 秒后开始检测，每间隔 3 秒重复执行。

而主容器在启动的时候会创建 `/tmp/live` 文件，然后睡眠 60 秒后再删除这个文件。可以看出，在容器创建后存活检测应该成功，60 秒后检测应该失败。

```shell
[root@master01 ~]# kubectl get pod -w
NAME                READY   STATUS    RESTARTS   AGE
liveness-exec-pod   1/1     Running   0          8s
liveness-exec-pod   1/1     Running   1          98s
liveness-exec-pod   1/1     Running   2          3m18s
```

可以看出，98 秒后 Pod 开始重启了，因为存活检测失败会发生重启（默认重启策略是 Alwayls ）

**演示 `HTTPGetAction` 方式的存活检测**

live-httpget.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec-pod
  namespace: default
spec:
  containers:
  - name: liveness-httpget-container
    image: heqingbao/k8s_myapp:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    livenessProbe:
      httpGet:
        port: http
        path: /index.html
      initialDelaySeconds: 1
      periodSeconds: 3
      timeoutSeconds: 10
```

跟第一个例子差不多，就不解释了， 这里存活检测应该是成功的。

```shell
[root@master01 ~]# kubectl apply -f live-httpget.yaml
pod/liveness-exec-pod created

[root@master01 ~]# kubectl get pod
NAME                READY   STATUS    RESTARTS   AGE
liveness-exec-pod   1/1     Running   0          3s

[root@master01 ~]# kubectl get pod -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
liveness-exec-pod   1/1     Running   0          14s   172.16.196.138   node01   <none>           <none>

[root@master01 ~]# curl 172.16.196.138/index.html
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>
```

可以发现 Pod 是正常状态的。

然后我们进入容器把 index.html 文件删除，再来看下状态：

```shell
[root@master01 ~]# kubectl exec liveness-exec-pod -it -- /bin/sh
/ # rm /usr/share/nginx/html/index.html
/ # exit

[root@master01 ~]# kubectl get pod -w
NAME                READY   STATUS    RESTARTS   AGE
liveness-exec-pod   1/1     Running   0          57s
liveness-exec-pod   1/1     Running   1          60s
```

可以看到 Pod 又在重启了，因为存活检测失败。不过这里只会重启一次，因为重新创建容器后 index.html 是存在的。

**演示 `TCPSocketAction` 方式的存活检测**

live-tcp.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp-pod
  namespace: default
spec:
  containers:
  - name: liveness-tcp-container
    image: heqingbao/k8s_myapp:v1
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      timeoutSeconds: 1
      periodSeconds: 3
```

这里 8080 端口是不存在的，同样存活检测会失败

```shell
[root@master01 ~]# kubectl apply -f live-tcp.yaml
pod/liveness-tcp-pod created

[root@master01 ~]# kubectl get pod -w
NAME               READY   STATUS    RESTARTS   AGE
liveness-tcp-pod   1/1     Running   1          13s
liveness-tcp-pod   1/1     Running   2          24s
liveness-tcp-pod   1/1     Running   3          36s
liveness-tcp-pod   0/1     CrashLoopBackOff   3          49s
```

可以看到 Liveness 一直会失败，Pod 会不断重启。



### 4.3 生命周期钩子

Kubernetes 为容器提供了两种生命周期钩子：

* postStart：于容器创建完成之后立即运行的钩子程序
* preStop：容器终止之前立即运行的程序，是以同步方式的进行，因此其完成之前会阻塞 删除容器的调用

下面演示下

post.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: heqingbao/k8s_myapp:v1
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handelr > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the preStop heandler > /usr/share/message"]
```

分别在容器启动和退出的时候向 `/usr/share/message` 输出一句话

```shell
[root@master01 ~]# kubectl apply -f post.yaml
pod/lifecycle-demo created

[root@master01 ~]# kubectl get pod
NAME             READY   STATUS    RESTARTS   AGE
lifecycle-demo   1/1     Running   0          26s

# 查看输出的信息
[root@master01 ~]# kubectl exec lifecycle-demo -it -- /bin/sh
/ # cat /usr/share/message
Hello from the postStart handelr
```

由于 Stop 的时候容器也不在了，这里就不演示了。

### 4.4 Pod STATUS 可能存在的值

* Pending

  挂起，Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建，等待时间包括调度 Pod 的时间和通过网络下载镜像的时间。

* Running：

  运行中，该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。

* Successed

  成功，Pod 中的所有容器都被成功终止，并且不会再重启。（比较常显示在 Job 和CronJob 中）

* Failed

  失败，Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。即容器以非 0 状态退出或者被系统终止。

* Unknown

  因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在的主机通信失败。

