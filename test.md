

## 网络通讯方式

Kubernetes 的网络模型假定了所有 Pod 都在一个可以直接连通的扁平的网络空间中，这在 GCE（Google Compute Engine）里面现成的网络模型，Kubernetes 假定这个网络已经存在。

而在私有云里搭建 Kubernetes 集群，就不能假定这个网络已经存在了。我们需要自己实现这个网络假设，将不同节点上的 Docker 容器之间的互相访问先打通，然后运行 Kubernetes 。

### Flannel

Flannel 是 CoreOS 团队针对 Kubernetes 设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的 Docker 容器都具有全集群唯一的虚拟 IP 地址。而且它还能在这些 IP 地址之间建立一个覆盖网络（Overlay Network），通过这个覆盖网络，将数据包原封不动传递到目标容器内。

### Pod 的网络通信

* 同一个 Pod 内部通信

  同一个 Pod 共享同一个网络命名空间，共享同一个 Linux 协议栈

* Pod1 至 Pod2

  **Pod1 与 Pod2 不在同一台主机：**Pod 的地址是与 docker0 在同一个网段的，但 docker0 网段与宿主机网卡是两个完全不同的 IP 网段，并且不同 Node 之间的通信只能通过宿主机的物理网卡进行。将 Pod 的 IP 和所在的 Node 的 IP 关联起来，通过这个关联让 Pod 可以互相访问。

  **Pod1 与 Pod2 在同一台机器：**由 docker0 网桥直接转发请求至 Pod2，不需要经过 Flannel

* Pod 至 Service 的网络

  目前基于性能考虑，全部为 iptables 维护和转发

* Pod 到外网

  Pod 向外网发送请求，查找路由表，转发数据包到宿主机的网卡，宿主网卡完成路由选择后，iptables 执行 Masquerade ，把源 IP 更改为宿主网卡的 IP ，然后向外网服务器发送请求

* 外网访问 Pod

  Service