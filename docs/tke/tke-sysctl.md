---
title: 在TKE容器集群中使用sysctl
tags:
  - TKE
  - Kubernetes
categories: TKE
keywords:
  - Kubernetes
  - TKE
description: 在TKE容器集群中使用sysctl
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031845569.jpeg
abbrlink: b81d5734
date: 2021-09-20 14:30:03
updated: 2021-09-20 23:58:58
---

> 在日常使用K8S部署业务时候，需要修改容器部分内核参数来做优化

## 在 Kubernetes 集群中使用 sysctl

### 环境准备

> 节点操作系统镜像名称：tlinux2.4x86_64 
>
> 节点内核版本：4.14+
>
> TKE集群：1.18版本

- 如果镜像是tlinux 宿主机内核跟容器内核是相互的独立的， tlinux 做了特殊的处理 容器运行时 与linux 可以相互独立一部分内核参数，按理来说原生的linux 与docker 是共享内核

- kubelet 配置的目的是为了开启修改容器内核的权限 ，


### 获取 Sysctl 的参数列表

在 Linux系统 中，用户可以通过 sysctl 接口修改内核运行时的参数。在 `/proc/sys/` 虚拟文件系统下存放许多内核参数。这些参数涉及了多个内核子系统，如：

- 内核子系统（通常前缀为: `kernel.`）
- 网络子系统（通常前缀为: `net.`）
- 虚拟内存子系统（通常前缀为: `vm.`）
- MDADM 子系统（通常前缀为: `dev.`）

### 启用非安全的 Sysctl 参数

sysctl 参数分为 *安全* 和 *非安全的*。 *安全* sysctl 参数除了需要设置恰当的命名空间外，在同一 node 上的不同 Pod 之间也必须是 *相互隔离的*。这意味着在 Pod 上设置 *安全* sysctl 参数

- 必须不能影响到节点上的其他 Pod
- 必须不能损害节点的健康
- 必须不允许使用超出 Pod 的资源限制的 CPU 或内存资源。

至今为止，大多数 *有命名空间的* sysctl 参数不一定被认为是 *安全* 的。 以下几种 sysctl 参数是 *安全的*：

- kernel.shm_rmid_forced
- net.ipv4.ip_local_port_range
- net.ipv4.tcp_syncookies
- net.ipv4.ping_group_range （从 Kubernetes 1.18 开始）

```
[root@VM-1-17-tlinux /proc/sys]# sysctl -a | grep -E 'net.ipv4.ip_local_port_range|kernel.shm_rmid_forced|net.ipv4.tcp_syncookies|net.ipv4.pin
g_group_range' 

kernel.shm_rmid_forced = 0
net.ipv4.ip_local_port_range = 32768    60999
net.ipv4.ping_group_range = 1   0
net.ipv4.tcp_syncookies = 1
```

在未来的 Kubernetes 版本中，若 kubelet 支持更好的隔离机制，则上述列表中将会 列出更多 *安全的* sysctl 参数。

所有 *安全的* sysctl 参数都默认启用。

所有 *非安全的* sysctl 参数都默认禁用，且必须由集群管理员在每个节点上手动开启。 那些设置了不安全 sysctl 参数的 Pod 仍会被调度，但无法正常启动。

参考上述警告，集群管理员只有在一些非常特殊的情况下（如：高可用或实时应用调整）， 才可以启用特定的 *非安全的* sysctl 参数。 如需启用 *非安全的* sysctl 参数，需要每个节点上分别设置 kubelet 参数，例如：

```
 --allowed-unsafe-sysctls='kernel.msg*,kernel.shm*,net.*' 
```

### 设置 Pod 的 Sysctl 参数 

目前，在 Linux 内核中，有许多的 sysctl 参数都是 *有命名空间的* 。 这就意味着可以为节点上的每个 Pod 分别去设置它们的 sysctl 参数。 在 Kubernetes 中，只有那些有命名空间的 sysctl 参数可以通过 Pod 的 securityContext 对其进行配置。

以下列出有命名空间的 sysctl 参数，在未来的 Linux 内核版本中，此列表可能会发生变化。

- `kernel.shm*`,
- `kernel.msg*`,
- `kernel.sem`,
- `fs.mqueue.*`,
- `net.*`（内核中可以在容器命名空间里被更改的网络配置项相关参数）。然而也有一些特例 （例如，`net.netfilter.nf_conntrack_max` 和 `net.netfilter.nf_conntrack_expect_max` 可以在容器命名空间里被更改，但它们是非命名空间的）。

没有命名空间的 sysctl 参数称为 *节点级别的* sysctl 参数。 如果需要对其进行设置，则必须在每个节点的操作系统上手动地去配置它们， 或者通过在 DaemonSet 中运行特权模式容器来配置。

可使用 Pod 的 securityContext 来配置有命名空间的 sysctl 参数， securityContext 应用于同一个 Pod 中的所有容器。

此示例中，使用 Pod SecurityContext 来对一个安全的 sysctl 参数 `kernel.shm_rmid_forced` 以及两个非安全的 sysctl 参数 `net.core.somaxconn` 和 `kernel.msgmax` 进行设置。 在 Pod 规约中对 *安全的* 和 *非安全的* sysctl 参数不做区分。

不修改前节点和POD里面参数配置如下：

- CVM节点上的参数配置

```
[root@VM-1-17-tlinux /proc/sys]# sysctl -a | grep -E 'kernel.shm_rmid_forced|net.core.somaxconn|kernel.msgmax'

kernel.msgmax = 65536
kernel.shm_rmid_forced = 0
net.core.somaxconn = 32768
```

- 容器镜像是centos启动POD参数

```
[root@centos-sysctl-7445b79c9f-fchxn /]# sysctl -a | grep -E 'kernel.shm_rmid_forced|net.core.somaxconn|kernel.msgmax'

kernel.msgmax = 65536
kernel.shm_rmid_forced = 0
net.core.somaxconn = 4096
```

#### 使用initContainers容器修改内核参数

> 在不配置kubelet参数allowed-unsafe-sysctls 之前

**1，先修改安全参数 kernel.shm_rmid_forced =1**

![1627548971447](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436514.png)

![image-20220917145559626](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171455727.png)

```
#登陆容器里面查看参数
[root@centos-sysctl-5c6686499c-lhbdv /]# sysctl -a | grep -E 'kernel.shm_rmid_forced'
kernel.shm_rmid_forced = 1
```

![1627549087889](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436630.png)

以上测试表示安全参数是不需要启动kubelet 的allowed-unsafe-sysctls配置就可以修改成功的



**2，同时修改非安全参数net.core.somaxconn=1024， kernel.msgmax=60000**

yaml实例如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: centos-sysctl
    qcloud-app: centos-sysctl
  name: centos-sysctl
  namespace: default
spec:
  selector:
    matchLabels:
      k8s-app: centos-sysctl
      qcloud-app: centos-sysctl
  template:
    metadata:
      labels:
        k8s-app: centos-sysctl
        qcloud-app: centos-sysctl
    spec:
      containers:
      - args:
        - -c
        - sleep 300000
        command:
        - /bin/sh
        image: centos:latest
        imagePullPolicy: IfNotPresent
        name: centos
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 64Mi
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      initContainers:
      - args:
        - -c
        - sysctl  -w  kernel.shm_rmid_forced=1 && sysctl  -w  net.core.somaxconn=1024 && sysctl -w kernel.msgmax=60000
        command:
        - /bin/sh
        image: busybox:latest
        imagePullPolicy: Always
        name: initc
        securityContext:
          privileged: true

```



配置截图如下

![1627549559535](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436668.png)

容器内也是生效的

![1627549596942](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436702.png)

节点上系统参数依据是

![1627549700468](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436729.png)

以上测试表示安全参数 和部分非安全参数也可以直接修改

**3，修改其他内核非安全参数**

示例修改如下参数：

```
net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_syn_retries=1
net.ipv4.tcp_tw_recycle=1
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=1
net.ipv4.tcp_keepalive_time=30
net.ipv4.ip_local_port_range="1024 65000"
```

CVM初始值：

```
[root@VM-1-17-tlinux ~]# sysctl  -a| grep -E 'net.ipv4.tcp_synack_retries|net.ipv4.tcp_syn_retries|net.ipv4.tcp_tw_recycle|net.ipv4.tcp_tw_reuse|net.ipv4.tcp_fin_timeout|net.ipv4.tcp_keepalive_time|net.ipv4.ip_local_port_range'

net.ipv4.ip_local_port_range = 32768    60999
net.ipv4.tcp_fin_timeout = 60
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_tw_reuse = 0
```

![1627549985182](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436768.png)

POD未修改前和操作系统保持默认的

```
[root@centos-sysctl-5c4d8dc5b-rc8gv /]# sysctl  -a| grep -E 'net.ipv4.tcp_synack_retries|net.ipv4.tcp_syn_retries|net.ipv4.tcp_tw_recycle|net.ipv4.tcp_tw_reuse|net.ipv4.tcp_fin_timeout|net.ipv4.tcp_keepalive_time|net.ipv4.ip_local_port_range'
net.ipv4.ip_local_port_range = 32768    60999
net.ipv4.tcp_fin_timeout = 60
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_tw_reuse = 0
```

![1627550034823](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436801.png)



然后使用initContainer去修改这些参数，发现会报错，不让修改，修改失败

**4，kubelet 开启非安全参数  (需要将节点移除重新)**

> *在TKE集群节点默认情况下是没有开启这个参数的，就需要用户提交工单申请开通自定义kubernetes参数，在创建节点时候可以设置，文章结尾会附上设置相关截图

--allowed-unsafe-sysctls=kernel.shm*,kernel.msg*,net.*

![1627550973864](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436828.png)

 

然后再去使用initContainer修改，查看修改成功

```
[root@centos-sysctl-5d6d89c66d-z4mxr /]# sysctl  -a| grep -E 'net.ipv4.tcp_synack_retries|net.ipv4.tcp_syn_retries|net.ipv4.tcp_tw_recycle|net.ipv4.tcp_tw_reuse|net.ipv4.tcp_fin_timeout|net.ipv4.tcp_keepalive_time|net.ipv4.ip_local_port_range'

net.ipv4.ip_local_port_range = 1024     65000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_tw_reuse = 1
```

![1627551531024](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171436812.png)

完整yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: centos-sysctl
    qcloud-app: centos-sysctl
  name: centos-sysctl
  namespace: default
spec:
  replicas: 1
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      k8s-app: centos-sysctl
      qcloud-app: centos-sysctl
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: centos-sysctl
        qcloud-app: centos-sysctl
    spec:
      containers:
      - args:
        - -c
        - sleep 300000
        command:
        - /bin/sh
        image: centos:latest
        imagePullPolicy: IfNotPresent
        name: centos
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 64Mi
        securityContext: {}
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      initContainers:
      - args:
        - -c
        - sysctl -w  net.ipv4.tcp_synack_retries=1 | sysctl -w  net.ipv4.tcp_syn_retries=1 | sysctl -w  net.ipv4.tcp_tw_recycle=1 | sysctl -w  net.ipv4.tcp_tw_reuse=1 | sysctl -w  net.ipv4.tcp_fin_timeout=1 |  sysctl -w  net.ipv4.tcp_keepalive_time=30 | sysctl -w  net.ipv4.ip_local_port_range="1024 65000"
        command:
        - /bin/sh
        image: busybox:latest
        imagePullPolicy: Always
        name: initc
        securityContext:
          privileged: true

```



#### 在TKE里面设置kubelet参数

参考文档：https://cloud.tencent.com/document/product/457/47775

1，需要提交工单申请加白名单（提供APPID，主账号UIN，需要修改的参数名称，组件，集群版本）

2，集群现存节点，需要先封锁节点，驱逐节点POD，然后将节点移出移入（移出移入会重装操作系统，记得做好备份，另外就是移出时候不要勾选销毁按量计费的节点，包年包月的节点不影响），添加已有节点加入时候设置参数，新增节点直接设置

```
allowed-unsafe-sysctls='kernel.msg*,kernel.shm*,net.*'
```

![image-20220917152451453](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171524529.png)



更多参考K8S官方文档：https://kubernetes.io/zh/docs/tasks/administer-cluster/sysctl-cluster/







 
