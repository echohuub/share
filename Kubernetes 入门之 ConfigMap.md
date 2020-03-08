通过前面的文章已经学习了很多 K8s 组件了，包括资源控制器，当时有提到一个控制器叫 StatefulSet ，它就是专门用来处理有状态服务的。

既然它是为了有状态服务而建立，那就意味着 K8s 集群有一定的存储能力，那到底有哪些存储机制呢？

在 K8s 中存储类型分为四种：

* ConfigMap - 存储配置信息
* Secret - 存储加密信息（比如密钥、用户名密码）
* Volume - 为 Pod 提供共享存储卷（比如本地磁盘、NFS）
* PersistentVolume（PV） - 通过其它服务构建持久卷



这篇文章主要讲 ConfigMap

## 一、介绍

许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。ConfigMap API 给我们提供了向容器中注入配置信息的机制，ConfigMap 可以保存单个属性，也可以用来保存整个配置文件或者 JSON 二进制大对象。



## 二、ConfigMap 创建

ConfigMap 的创建方式有多种，这里分别简单演示：

### 2.1 使用目录创建

```shell
[root@master01 configmap]# ll
-rw-r--r-- 1 root root 16 3月   7 23:31 logic.properties
-rw-r--r-- 1 root root 48 3月   7 23:31 ui.properties

[root@master01 configmap]# cat logic.properties 
x=y
hello=world

[root@master01 configmap]# cat ui.properties 
topbar=yes
banner.textcolor=red
banner.bg=green

[root@master01 configmap]# kubectl create configmap app-config --from-file=/root/configmap
configmap/app-config created
```

创建完 ConfigMap 后，查看一下：

```shell
[root@master01 configmap]# kubectl get configmap
NAME         DATA   AGE
app-config   2      31s

[root@master01 configmap]# kubectl get configmap app-config -o yaml
apiVersion: v1
data:
  logic.properties: |
    x=y
    hello=world
  ui.properties: |
    topbar=yes
    banner.textcolor=red
    banner.bg=green
kind: ConfigMap
metadata:
  creationTimestamp: "2020-03-07T15:33:53Z"
  name: app-config
  namespace: default
  resourceVersion: "319387"
  selfLink: /api/v1/namespaces/default/configmaps/app-config
  uid: 36dfb432-b530-4495-a054-86fd48ea5810

# 使用 describe 也可以查看详细信息
[root@master01 configmap]# kubectl describe configmap app-config
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
logic.properties:
----
x=y
hello=world

ui.properties:
----
topbar=yes
banner.textcolor=red
banner.bg=green

Events:  <none>
```



### 2.2 使用文件创建

跟上面指定目录差不多，区别就在指定单个文件

```shell
[root@master01 configmap]# kubectl create configmap app-config-2 --from-file=/root/configmap/ui.properties
configmap/app-config-2 created

[root@master01 configmap]# kubectl get configmap
NAME           DATA   AGE
app-config     2      4m52s
app-config-2   1      9s

# 查看 ConfigMap
[root@master01 configmap]# kubectl get configmap app-config-2 -o yaml
apiVersion: v1
data:
  ui.properties: |
    topbar=yes
    banner.textcolor=red
    banner.bg=green
kind: ConfigMap
metadata:
  creationTimestamp: "2020-03-07T15:38:36Z"
  name: app-config-2
  namespace: default
  resourceVersion: "320099"
  selfLink: /api/v1/namespaces/default/configmaps/app-config-2
  uid: bc7123af-d728-4c0c-a9a6-11bfc15632ab

# 查看 ConfigMap
[root@master01 configmap]# kubectl describe configmap app-config-2
Name:         app-config-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
ui.properties:
----
topbar=yes
banner.textcolor=red
banner.bg=green

Events:  <none>
```



### 2.3 使用字面值创建

利用 `--from-literal` 参数传递配置信息

```shell
[root@master01 configmap]# kubectl create configmap app-config-3 --from-literal=hello=world --from-literal=bottom=false
configmap/app-config-3 created

[root@master01 configmap]# kubectl get configmap
NAME           DATA   AGE
app-config     2      8m34s
app-config-2   1      3m51s
app-config-3   2      15s

# 查看 ConfigMap
[root@master01 configmap]# kubectl get configmap app-config-3 -o yaml
apiVersion: v1
data:
  bottom: "false"
  hello: world
kind: ConfigMap
metadata:
  creationTimestamp: "2020-03-07T15:42:12Z"
  name: app-config-3
  namespace: default
  resourceVersion: "320651"
  selfLink: /api/v1/namespaces/default/configmaps/app-config-3
  uid: f81ff8a4-da2f-4d74-b80d-90021d0d480f

# 查看 ConfigMap
[root@master01 configmap]# kubectl describe configmap app-config-3
Name:         app-config-3
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
bottom:
----
false
hello:
----
world
Events:  <none>
```

## 三、在 Pod 中使用 ConfigMap

### 3.1 使用 ConfigMap 来替代环境变量

special.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
```

env.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

创建它

```shell
[root@master01 configmap]# kubectl apply -f special.yaml 
configmap/special-config created

[root@master01 configmap]# kubectl apply -f env.yaml 
configmap/env-config created

[root@master01 configmap]# kubectl get cm
NAME             DATA   AGE
env-config       1      2s
special-config   1      49s
```

然后创建 Pod ：

pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: test-container
    image: heqingbao/k8s_myapp:v1
    command: ["/bin/sh", "-c", "env"]
    env:
      - name: SPECIAL_HOW_KEY
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.how
    envFrom:
      - configMapRef:
          name: env-config
  restartPolicy: Never
```

两种注入方式：

* 直接 env : 可以单独注入 ConfigMap 里面的单个 Key
* envFrom 方式：注入 ConfigMap 中的所有 Key

创建后查看下环境变量：

```shell
[root@master01 configmap]# kubectl get pod
NAME            READY   STATUS      RESTARTS   AGE
configmap-pod   0/1     Completed   0          3m16s

[root@master01 configmap]# kubectl logs configmap-pod
MYAPP_SVC_PORT_80_TCP_ADDR=10.98.57.156
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
MYAPP_SVC_PORT_80_TCP_PORT=80
HOSTNAME=configmap-pod
SHLVL=1
MYAPP_SVC_PORT_80_TCP_PROTO=tcp
HOME=/root
SPECIAL_HOW_KEY=very
MYAPP_SVC_PORT_80_TCP=tcp://10.98.57.156:80
NGINX_VERSION=1.12.2
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
MYAPP_SVC_SERVICE_HOST=10.98.57.156
log_level=INFO
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
PWD=/
KUBERNETES_SERVICE_HOST=10.96.0.1
MYAPP_SVC_SERVICE_PORT=80
MYAPP_SVC_PORT=tcp://10.98.57.156:80
```

可以看出`SPECIAL_HOW_KEY=very` 和 `log_level=INFO` 都已经注入到 Pod 的环境变量里面了。

### 3.2 用 ConfigMap 设置命令行参数

special.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.when: now
```

command-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: test-container
    image: heqingbao/k8s_myapp:v1
    command: ["/bin/sh", "-c", "echo $(SPECIAL_HOW_KEY) $(SPECIAL_WHEN_KEY)"]
    env:
      - name: SPECIAL_HOW_KEY
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.how
      - name: SPECIAL_WHEN_KEY
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.when
  restartPolicy: Never
```

部署它们：

```shell
[root@master01 configmap]# kubectl apply -f special.yaml 
configmap/special-config created

[root@master01 configmap]# kubectl create -f command-pod.yaml 
pod/configmap-pod created

[root@master01 configmap]# kubectl get pod
NAME            READY   STATUS      RESTARTS   AGE
configmap-pod   0/1     Completed   0          13s

# 查看环境变量
[root@master01 configmap]# kubectl logs configmap-pod
very now
```

可以看出，注入的两环境变量 Key 也成功了。

### 2.4 通过数据卷插件使用 ConfigMap

special.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
  special.when: now
```

volumn-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
  - name: test-container
    image: heqingbao/k8s_myapp:v1
    command: ["/bin/sh", "-c", "sleep 600"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: special-config
  restartPolicy: Never
```

部署它们：

```shell
[root@master01 configmap]# kubectl create -f volume-pod.yaml 
pod/configmap-pod created

[root@master01 configmap]# kubectl get pod
NAME            READY   STATUS    RESTARTS   AGE
configmap-pod   1/1     Running   0          4s

# 验证
[root@master01 configmap]# kubectl exec -it configmap-pod sh
/ # cat /etc/config/special.
special.how   special.when
/ # cat /etc/config/special.when
now/ # 
/ # cat /etc/config/special.how
very/ # 
/ # 
```

可以看了，ConfigMap 已经挂载到文件系统了。Key 作为文件名，Value 作为文件内容。



## 四、ConfigMap 热更新

先清理环境，删除所有的 Pod 和 ConfigMap ：

```shell
[root@master01 configmap]# kubectl delete pod --all
[root@master01 configmap]# kubectl delete cm --all
```

test.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-config
  namespace: default
data:
  log_level: INFO
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
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
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: log-config
```

上面的内容跟 **2.4 讲的数据卷挂载 ConfigMap** 类似，就不再描述了。

下面来看一下效果：

```shell
[root@master01 configmap]# kubectl apply -f test.yaml 
configmap/log-config created
deployment.apps/myapp created

[root@master01 configmap]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
myapp-f46458fdc-68pkh   1/1     Running   0          8s

[root@master01 configmap]# kubectl exec -it myapp-f46458fdc-68pkh sh
/ # ls /etc/config
log_level
/ # cat /etc/config/log_level 
INFO/ #
```

发现一切正常，log_level 被正确挂载到了 Pod 里面。

下面来试下热更新，我们直接修改 ConfigMap ：

```shell
# 把 log_level 值改成 DEBUG
[root@master01 configmap]# kubectl edit configmap log-config
configmap/log-config edited

```

把 `log_level: INFO` 改成 `log_level: DEBUG` ，再进入 Pod 验证下；

```shell
[root@master01 configmap]# kubectl exec -it myapp-f46458fdc-68pkh sh
/ # cat /etc/config/log_level 
DEBUG/ # 
```

可见 Pod 里的文件已经发生了变化，热更新成功。

> 注意：更新ConfigMap 并不触发相关 Pod 的滚动更新