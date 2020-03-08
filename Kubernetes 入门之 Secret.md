在上篇文章中介绍了在 K8s 中如果通过 ConfigMap 的方式保存配置文件以及数据，这些数据可以导入到 Pod 内部成为环境变量，或者是文件，也可以实现配置热更新的目的。

但是这些数据都是以明文的方式保存的，如果需要存储一些密钥或者密码等敏感数据，使用 ConfigMap 就不太合适了。

在 K8s 中，提供了一种 Secret 的存储方式，它解决了密码、Token、密钥等敏感数据的存储问题，而不需要把这些第三数据暴露到镜像或者 Pod Spec ，Secret 可以以 Volume 或者环境变量的方式使用。



Secret 有三种类型：

* Service Account：用来访问 Kubernetes API ，由 Kubernetes 自动创建，并且会自动挂载到 Pod 的 `/run/secrets/kubernetes.io/serviceaccount` 目录中
* Opaque：base64 编码格式的 Secret ，用来存储密码、密钥等
* kubernetes.io/dockerconfigjson：用来存储私有 docker registry 的认证信息



## 一、Service Account

上面已经介绍了，这里直接进入 Pod 查看下：

```shell
[root@master01 ~]# kubectl exec kube-proxy-9k8tk -n kube-system ls /run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

* ca.crt：访问 api-server 需要的证书，默认是双向 Https
* namespace：保存当前的名称空间，这里是 kube-system
* token：认证的密钥信息

由这三个东西组成了 SA ，因为有了 SA ，kube-proxy 才能访问 api-service 达成认证的目的。



## 二、Opaque Secret

### 2.1 创建说明

Opaque 类型的数据是一个 map 类型，要求 value 是 base64 编码格式：

```shell
[root@master01 ~]# echo -n "admin" | base64
YWRtaW4=
[root@master01 ~]# echo -n "123456789" | base64
MTIzNDU2Nzg5
```

secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2Nzg5
```

部署：

```shell
[root@master01 ~]# kubectl create -f secret.yaml 
secret/mysecret created

[root@master01 ~]# kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-gpltl   kubernetes.io/service-account-token   3      48s
mysecret              Opaque                                2      4s
```

> 注意，每个名称空间下都有一个 default-token 的 Secret ，K8s 会为每个名称空间都创建一个 SA ，用于 Pod 的挂载。



### 2.2 使用方式

有两种使用方式：

#### 2.2.1 将 Secret 挂载到 Volume 中

secret-test-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: heqingbao/k8s_myapp:v1
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
```

部署它：

```shell
[root@master01 ~]# kubectl create -f secret-test-pod.yaml 
pod/secret-test-pod created
[root@master01 ~]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
secret-test-pod         1/1     Running   0          4s
```

进入 Pod 验证下：

```shell
[root@master01 ~]# kubectl exec -it secret-test-pod sh
/ # ls /etc/secrets/
..2020_03_07_17_12_01.200138406/  password
..data/                           username
/ # cd /etc/secrets/
/etc/secrets # ls
password  username
/etc/secrets # cat username 
admin/etc/secrets # 
/etc/secrets # cat password 
123456789/etc/secrets # 
/etc/secrets # 
```

可以看到 Secret 已经挂载到文件里面了，并且是 Decoded 后的状态。

#### 2.2.2 将 Secret 导出到环境变量中

它的使用方式跟前一篇文章讲的使用 Env 来引入 ConfigMap 类似

secret-env-test.yaml 

```yaml
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
        env:
          - name: TEST_USER
            valueFrom:
              secretKeyRef:
                name: mysecret
                key: username
          - name: TEST_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysecret
                key: password

```

部署它：

```shell
[root@master01 ~]# kubectl apply -f secret-env-test.yaml 
deployment.apps/myapp created

[root@master01 ~]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
myapp-6cb97c76d8-4mrh7   1/1     Running   0          34s
```

验证

```shell
[root@master01 ~]# kubectl exec -it myapp-6cb97c76d8-4mrh7 sh
/ # echo $TEST_USER
admin
/ # echo $TEST_PASSWORD
123456789
```

可以看到 Secret 已经导入到环境变量里面了。



## 三、kubernetes.io/dockerconfigjson

使用 kubectl 创建 docker registry 认证的 secret

```shell
[root@master01 ~]# kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL

secret/myregistrykey created
```

然后在创建 Pod 的时候，通过 imagePullSecrets 来引入刚创建的 `myregistrykey`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: heqingbao/k8s_myapp:v1
  imagePullSecrets:
  - name: myregistrykey
```

这个就不演示了，等后面学了 Harbor 私有仓库的时候再尝试。