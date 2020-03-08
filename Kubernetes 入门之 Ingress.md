上一篇文章[讲了 Service 的相关原理和配置方案](https://www.yuque.com/heqingbao/cx9ztf/qkvxi5)，在讲解 Service 的概念时有提到过默认 Service 仅支持四层代理 ，如果需要七层代理 的功能则没有办法实现。

在 K8s 中，为我们提供了 Ingress 实现七层服务，这篇文章主要讲解 Ingress 的 Nginx 实现方案。

Ingress 官网：https://kubernetes.github.io/ingress-nginx/deploy/

参照文档下面就来快速演示下。

## 一、部署 Ingress

按照文档，应用以下 yaml 文件即可快速启动 Ingress 

```shell
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```

但是在国内由于某种原因里面的镜像无法下载，我们可以先下载这个 mandatory.yaml 文件，拿到具体的镜像名称，再单独通过其它方式去下载镜像。

```shell
[root@master01 ~]# https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml

[root@master01 ~]# cat mandatory.yaml | grep image
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
```

这个镜像就是：`quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0`

这里可以参考我的另一篇文章 [如何在国内顺畅下载被墙的 Docker 镜像？](https://mp.weixin.qq.com/s/kf0SrktAze3bT7LcIveDYw)实际上我刚刚制作这个镜像大概也就花了两分钟，这里不再重复了，建议去看下那篇文章。

如果不想自己制作，也可以使用我制作好的公开镜像：

```shell
[root@node01 ~]# docker pull registry.cn-hangzhou.aliyuncs.com/heqingbao-trans/nginx-ingress-controller:0.30.0
```

然后把 Tag 修改成 mandatory.yaml 里面的名字：

```shell
[root@node01 ~]# docker tag registry.cn-hangzhou.aliyuncs.com/heqingbao-trans/nginx-ingress-controller:0.30.0 quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0

# 验证
[root@node01 ~]# docker images | grep quay.io/kubernetes
quay.io/kubernetes-ingress-controller/nginx-ingress-controller   0.30.0              f351ba0d5604        7 minutes ago       323MB
```

可以看到 node01 机器上已经有这个镜像了，然后在其它 node 节点也准备好这个镜像。

已经有了镜像，下面就来执行这个 yaml ：

```shell
[root@master01 ~]# kubectl apply -f mandatory.yaml 
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
limitrange/ingress-nginx created
```

查看 Pod，注意这个 Pod 运行在` ingress-nginx`名称空间下：

```shell
[root@master01 ~]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-7f74f657bd-mh7xm   1/1     Running   0          42s
```

然后还需要选择服务暴露模式，文档中有介绍针对 AWS、GCE、Azure 等的暴露方式，也有 Bare-metal 的。

因为我们的集群是一个本地裸机结构，这里使用 Bare-metal 模式，使用 NodePort 直接暴露服务：

```shell
[root@master01 ~]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml

[root@master01 ~]# kubectl apply -f service-nodeport.yaml 
service/ingress-nginx created

[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   33s
```

可以看到已经有一个 NodePort 服务暴露了。

节点的（master 或者 node）30682 端口对应容器的 80 端口，节点的 32232 端口对应容器的 443 端口。

下面来演示 Ingress 的常用操作。

## 二、Ingress HTTP 代理访问

目标：

能过域名 www.helloingress.com 访问到 Pod

ingress-demo.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: heqingbao/k8s_myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    name: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: www.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-svc
          servicePort: 80 

```

以上先定义了一个 Deployment ，包含两个 Pod 副本，Pod 开启了 80 端口；

然后定义了一个 Service ，Type 是 ClusterIP ，暴露 80 端口对应 Pod 的 80 端口；

最后定义了一个 Ingress ，访问 `www.helloingress.com` 的根路径将链接到 `nginx-svc`

```shell
[root@master01 ~]# kubectl apply -f ingress-demo.yaml 
deployment.apps/nginx-deploy created
service/nginx-svc created
ingress.extensions/nginx-ingress created

[root@master01 ~]# kubectl get pod 
NAME                           READY   STATUS    RESTARTS   AGE
nginx-deploy-d784f84c4-sdvkd   1/1     Running   0          53s
nginx-deploy-d784f84c4-zjxqx   1/1     Running   0          53s

[root@master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   13d
nginx-svc    ClusterIP   10.100.217.58   <none>        80/TCP    68s

[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   58m
```

然后在 host 文件 `/etc/hosts` 添加：

```
192.168.0.114 www.helloingress.com
```

在浏览器上访问：http://www.helloingress.com:30682/ 看到以下信息表示成功

```
Hello MyApp | Version: v1 | Pod Name
```

然后再访问：http://www.helloingress.com:30682/hostname.html 查看 Pod 地址

```
nginx-deploy-d784f84c4-sdvkd
```

不断刷新，应该可以发现已经实现了负载均衡。

此示例演示了通过部署 Ingress 实现域名的直接访问。



**下面演示下使用 Ingress 创建虚拟主机** 

目标：

通过两个域名（www1.helloingress.com 和 www2.helloingress.com）分别访问不同的内容。

操作前先把上面演示的内容清除掉：

```shell
[root@master01 ~]# kubectl delete -f ingress-demo.yaml
deployment.apps "nginx-deploy" deleted
service "nginx-svc" deleted
ingress.extensions "nginx-ingress" deleted
```

先创建两个 Deployment & Service

ingress-deployment1.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-1
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: heqingbao/k8s_myapp:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: svc-1
spec:
  selector:
    name: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

ingress-deployment2.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-2
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: heqingbao/k8s_myapp:v2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: svc-2
spec:
  selector:
    name: nginx
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

为了后面演示方便，两者做了两点区别：

1. Name 不同
2. Pod 的镜像版本不同，deploy-1 是 v1 版本，deploy-2 是 v2 版本。

下面来部署它们

```shell
[root@master01 ~]# kubectl apply -f ingress-deployment1.yaml 
deployment.apps/deploy-1 created
service/svc-1 created

[root@master01 ~]# kubectl apply -f ingress-deployment2.yaml 
deployment.apps/deploy-2 created
service/svc-2 created

[root@master01 ~]# kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
deploy-1-56969d788-42hfx    1/1     Running   0          10s   172.16.196.188   node01   <none>           <none>
deploy-1-56969d788-hh6xp    1/1     Running   0          10s   172.16.140.124   node02   <none>           <none>
deploy-2-564bcfbcbd-9tqwh   1/1     Running   0          6s    172.16.196.189   node01   <none>           <none>
deploy-2-564bcfbcbd-qhwd2   1/1     Running   0          6s    172.16.140.125   node02   <none>           <none>

[root@master01 ~]# curl 172.16.196.188
Hello MyApp | Version: v1 | <a href="hostname.html">Pod Name</a>

[root@master01 ~]# curl 172.16.196.189
Hello MyApp | Version: v2 | <a href="hostname.html">Pod Name</a>
```

可见访问 deploy-1 的 Pod 输出 v1 版本信息，访问 deploy-2 的 Pod 输出 v2 版本的信息，符合预期。

接下来编写 ingress 规则 ：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-1
spec:
  rules:
  - host: www1.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-1
          servicePort: 80 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-2
spec:
  rules:
  - host: www2.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-2
          servicePort: 80 
```

分别定义了两个域名:

**www1.helloingress.com** 将访问到 **svc-1**

**www2.helloingress.com** 将访问到 **svc-2**

部署它

```shell
[root@master01 ~]# kubectl apply -f ingressrule.yaml 
ingress.extensions/ingress-1 created
ingress.extensions/ingress-2 created

[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   98m
```

然后我们进入 Ingress 容器查看 Nginx 配置信息：

```shell
[root@master01 ~]# kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-7f74f657bd-mh7xm   1/1     Running   0          105m

[root@master01 ~]# kubectl exec -it nginx-ingress-controller-7f74f657bd-mh7xm -n ingress-nginx bash
bash-5.0$ cat nginx.conf
...
        ## start server www1.helloingress.com
        server {
                server_name www1.helloingress.com ;
                ...
                location / {
                        
                        set $namespace      "default";
                        set $ingress_name   "ingress-1";
                        set $service_name   "svc-1";
                        set $service_port   "80";
                        set $location_path  "/";
                        ...
                }
                ...
        }
        ## end server www1.helloingress.com
        
        ## start server www2.helloingress.com
        server {
                server_name www2.helloingress.com ;
                ...
                location / {
                        
                        set $namespace      "default";
                        set $ingress_name   "ingress-2";
                        set $service_name   "svc-2";
                        set $service_port   "80";
                        set $location_path  "/";
                        ...
                }
        }
        ## end server www2.helloingress.com

```

所以我们上面写的那些 Ingress 规则，最终会被转化成 Nginx 的配置文件，以达到访问控制的目的。

同时通过命令也可以看到有两个 Ingress 信息：

```shell
[root@master01 ~]# kubectl get ingress
NAME        HOSTS                   ADDRESS        PORTS   AGE
ingress-1   www1.helloingress.com   10.99.180.14   80      4m11s
ingress-2   www2.helloingress.com   10.99.180.14   80      4m11s
```

然后我们来测试一下

首先需要在 hosts 文件里面添加两个规则 ：

```
192.168.0.114 www1.helloingress.com
192.168.0.114 www2.helloingress.com
```

在浏览器访问：http://www1.helloingress.com:30682/ 将看到 v1 版本信息：

```
Hello MyApp | Version: v1 | Pod Name
```

在浏览器访问：http://www2.helloingress.com:30682/ 将看到 v2 版本信息：

```shell
Hello MyApp | Version: v2 | Pod Name
```

到此，Ingress 虚拟主机功能演示完成。



## 三、Ingress HTTPS 代理访问

目标：

通过创建自定义证书，实现 https://www3.helloingress.com 的访问。

### 3.1 创建证书和私钥

```shell
[root@master01 ~]# openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=ingresssvc/O=ingresssvc"
```

### 3.2 把证书和私钥存储到 Secret

```shell
[root@master01 ~]# kubectl create secret tls tls-secret --key tls.key --cert tls.crt
secret/tls-secret created
```

### 3.3 创建部署 Deployment & Service

创建 Deployment & Service

deployment3.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-3
spec:
  replicas: 2
  selector:
    matchLabels:
      name: nginx-3
  template:
    metadata:
      labels:
        name: nginx-3
    spec:
      containers:
      - name: nginx
        image: heqingbao/k8s_myapp:v3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: svc-3
spec:
  selector:
    name: nginx-3
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

部署它

```shell
[root@master01 ~]# kubectl apply -f ingress-deployment3.yaml 
deployment.apps/deploy-3 created

service/svc-3 created
[root@master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   13d
svc-1        ClusterIP   10.110.61.215   <none>        80/TCP    17m
svc-2        ClusterIP   10.104.68.180   <none>        80/TCP    17m
svc-3        ClusterIP   10.111.150.96   <none>        80/TCP    5s

[root@master01 ~]# curl 10.111.150.96
Hello MyApp | Version: v3 | <a href="hostname.html">Pod Name</a>
```

### 3.4 创建部署 Ingress

创建 Ingress 规则 

ingress-tls.yaml 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
spec:
  tls:
  - hosts:
    - www3.helloingress.com
    secretName: tls-secret
  rules:
  - host: www3.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-3
          servicePort: 80 
```

部署 Ingress ：

```shell
[root@master01 ~]# kubectl apply -f ingress-tls.yaml 
ingress.extensions/ingress-https created

[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   117m
```

可以发现 Https 的端口是 32232，下面来验证下。

在 `/etc/hosts` 添加 `192.168.0.114 www1.helloingress.com`

访问：https://www3.helloingress.com:32232/ 将会看到输出 v3 版本信息

```
Hello MyApp | Version: v3 | Pod Name
```

> 期间会提示连接不是私密连接，因为这个证书是我们自己创建的。

到此，已经完整演示了 Ingress 配置 Https 证书的访问。



## 四、Ingress 实现 BasicAuth

目标：

通过 Ingress 实现 Basic Auth 的认证。

对于 Nginx 的认证方式来说，它采用的是 Apache 的认证模块，所以需要先安装 Apache 模块。

```shell
[root@master01 ~]# yum install -y httpd

# 创建用户名 foo 
[root@master01 ~]# htpasswd -c auth foo
New password: 
Re-type new password: 
Adding password for user foo

# 创建 Secret
[root@master01 ~]# kubectl create secret generic basic-auth --from-file=auth 
secret/basic-auth created
```

创建 Ingress 规则 ：

ingress-basic-auth.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-basic-auth
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo'
spec:
  rules:
  - host: auth.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-1
          servicePort: 80 
```

部署 Ingress ：

```shell
[root@master01 ~]# kubectl apply -f ingress-basic-auth.yaml 
ingress.extensions/ingress-basic-auth created
[root@master01 ~]# kubectl get ingress
NAME                 HOSTS                   ADDRESS        PORTS     AGE
ingress-1            www1.helloingress.com   10.99.180.14   80        33m
ingress-2            www2.helloingress.com   10.99.180.14   80        33m
ingress-basic-auth   auth.helloingress.com                  80        11s
ingress-https        www3.helloingress.com   10.99.180.14   80, 443   14m
[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   132m
```

同样需要在 `/etc/hosts` 里面添加 `192.168.0.114 auth.helloingress.com`

再访问 http://auth.helloingress.com:30682/ 应该会提示输入用户名和密码。

这里输入上面创建的用户名 `foo` 和对应的密码即可访问。



## 五、Ingress 实现 Rewrite

在 Nginx 里面比较常用的还有重写功能。

下面演示当访问 `http://www4.helloingress.com` 的时候重定向到 `https://www3.hellingress.com`

创建 Ingress 规则 ：

ingress-rewrite.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: https://www3.helloingress.com:32232
spec:
  rules:
  - host: www4.helloingress.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc-1
          servicePort: 80
```

部署 Ingress ：

```shell
[root@master01 ~]# kubectl apply -f ingress-rewrite.yaml
ingress.extensions/ingress-rewrite created
[root@master01 ~]# kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.180.14   <none>        80:30682/TCP,443:32232/TCP   143m
```

访问：http://www4.helloingress.com:30682/ 将会跳转到 https://www3.helloingress.com:32232/

并输出 svc-3 的结果：

```
Hello MyApp | Version: v3 | Pod Name
```



除此之外还有一些常用的配置，比如是否强制重定向到 HTTPS、支持路径正则表达式等，具体可以参考 Ingress 官网：https://kubernetes.github.io/ingress-nginx/