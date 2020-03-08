前面几篇文章讲解了 Deployment 、DaemonSet 的概念和使用，这篇文章来讲讲 Job 和 CronJob。

## 一、Job

Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

特殊说明：

* spec.template 格式跟 Pod 相同
* RestartPolicy 仅支持 Never 或 OnFailure
* 单个 Pod 时，默认 Pod 成功运行后 Job 即结束
* .spec.completions 标识 Job 结束需要成功运行的 Pod 个数，默认为 1
* .spec.parallelism 标识并运行的 Pod 的个数，默认 1
* .spec.activeDeadlineSeconds 标识失败 Pod 的重试最大时间，超过这个时间不会继续重试

演示一下

job.yaml

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    metadata:
      name: pi
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(100)"]
      restartPolicy: Never
```

创建了一个 Job ，计算 100 位小数点的 PI 值

```shell
[root@master01 ~]# kubectl create -f job.yaml
job.batch/pi created

[root@master01 ~]# kubectl get pod
NAME       READY   STATUS              RESTARTS   AGE
pi-wbq4m   0/1     ContainerCreating   0          5s
```

查看当前的 Job：

```shell
[root@master01 ~]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     0/1           9s         9s
```

这个镜像下载的时间比较长，多等一下

```shell
[root@master01 ~]# kubectl get pod
NAME       READY   STATUS      RESTARTS   AGE
pi-wbq4m   0/1     Completed   0          45s

[root@master01 ~]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           15s        48s
```

可以看到 Job 已经执行完成。

看下结果：

```shell
[root@master01 ~]# kubectl logs pi-wbq4m
3.141592653589793238462643383279502884197169399375105820974944592307816406286208998628034825342117068
```



## 二、CronJob

Cron Job 管理基于时间的 Job（它跟 Deployment 有点像，Deployment 通过创建 RS 去管理 Pod），即：

* 在给定时间点只运行一次
* 周期性地在给定时间点运行

典型的用法示例：

* 在给你写的时间点调度 Job 运行
* 创建周期性运行的 Job，例如：数据库备份、发送邮件

演示一下

cronjob.yaml

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

每分钟输出日期和一句消息。

```shell
[root@master01 ~]# kubectl create -f cronjob.yaml
cronjob.batch/hello created

[root@master01 ~]# kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        <none>          7s
```

查看 Pod 状态：

```shell
[root@master01 ~]# kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
hello-1582990920-bvwfc   0/1     Completed           0          3m20s
hello-1582990980-w4wgr   0/1     Completed           0          2m20s
hello-1582991040-f6bf9   0/1     Completed           0          80s
hello-1582991100-dlddq   0/1     ContainerCreating   0          20s

[root@master01 ~]# kubectl logs hello-1582990920-bvwfc
Sat Feb 29 15:42:24 UTC 2020
Hello from the Kubernetes cluster

[root@master01 ~]# kubectl logs hello-1582990980-w4wgr
Sat Feb 29 15:43:19 UTC 2020
Hello from the Kubernetes cluster
```

可以看到，基本每分钟会创建一个 Pod 去执行 Job 。