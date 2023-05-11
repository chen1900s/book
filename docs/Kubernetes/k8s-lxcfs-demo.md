---
title: Kubernetes容器资源限制和lxcfs问题
abbrlink: e0d4c727
tags: Kubernetes
categories: Kubernetes
keywords: Kubernetes
description: Kubernetes容器资源限制和lxcfs问题
top_img: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209191613571.jpg
date: 2020-06-01 20:10:42
updated: 2020-06-01 23:58:58
---

## Docker容器资源限制问题

> 以下是基于腾讯云TKE容器服务测试验证

### 背景

Linux利用CGroup实现了对容器资源的限制，但是在容器内部还是默认挂载宿主机 /proc 目录下的资源信息文件，如：meminfo,cpuinfo,stat,uptiem，等。当进入Containers执行free，df，top等命令的时候，这时候默认读取的是 /proc 目录内的资源信息文件内容，而这些资源信息文件使用的是宿主机的，所以我们看到的是宿主机的使用信息。

### 关于LXCFS

LXCFS是一个开源的FUSE（用户态文件系统），用来支持LXC容器，也支持Docker容器，社区中常用此工具来实现容器中的资源可见性。

**LXCFS原理：**

以内存资源为列：通过将宿主机的 /var/lib/lxcfs/meminfo 文件挂载到容器内的/proc/meminfo，然后LXCFS会从容器的CGroup中读取正确的内存限制，然后应用到 /var/lib/lxcfs/meminfo ，这时候容器内部从而就得到了正确的内存信息。

> 说明：/var/lib/lxcfs/meminfo 是服务启动的时候默认指定的目录。 

### 操作步骤

目前腾讯云TKE里面想实现对容器资源的限制，在容器里面执行free，df，top等命令的时候看到容器真正的资源，有两种方案

#### 方案一：**目前 [TencentOS Server](https://github.com/Tencent/TencentOS-kernel/wiki/container-resource-view-isolation ) 特性已支持容器资源展示隔离** 

- 增加主机级开关：内核已实现了类似 LXCFS 特性。用户无需在节点部署 LXCFS 文件系统及修改 POD spec，仅需在节点开启全局开关（`sysctl -w kernel.stats_isolated=1`），`/proc/cpuinfo` 及 `/proc/meminfo` 等文件获取即可按容器隔离 
- 增加容器级开关：针对类似节点监控组件等特殊容器，增加了容器级开关 kernel.container_stats_isolated。在主机级开关开启时，仅需在容器启动脚本中关闭容器级开关（sysctl -w kernel.container_stats_isolated=0），即可在容器中读取 /proc/cpuinfo 及 /proc/meminfo 文件时获取到主机信息。

**1，环境准备**

已创建集群并添加节点，使用操作系统tlinux 2.4，节点规格4C8G

```
[root@VM-0-17-tlinux ~/lxcfs]# uname -r
4.14.105-19-0020.1
[root@VM-0-17-tlinux ~/lxcfs]# sysctl  -a| grep kernel.stats_isolated
kernel.stats_isolated = 0

#默认是0
```

**2，查看当前主机节点上资源情况**

```
[root@VM-0-17-tlinux ~/lxcfs]# cat /proc/meminfo | grep MemTotal
MemTotal:        8035132 kB
[root@VM-0-17-tlinux ~/lxcfs]# cat /proc/cpuinfo  | grep processor | wc -l
4
[root@VM-0-17-tlinux ~/lxcfs]# free -m
              total        used        free      shared  buff/cache   available
Mem:           7846        2394        2571           2        2880        5299
Swap:             0           0           0
```

![1627827269129](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041912729.png)

**3，未修改前登录容器查看资源，看到的是宿主机的使用信息。**

```
[root@VM-0-17-tlinux ~/lxcfs]# kubectl  exec -it centos-5ccb64bbdb-h7nb9  -- /bin/bash
[root@centos-5ccb64bbdb-h7nb9 /]# cat /proc/meminfo | grep MemTotal
MemTotal:        8035132 kB
[root@centos-5ccb64bbdb-h7nb9 /]# cat /proc/cpuinfo  | grep processor | wc -l
4
[root@centos-5ccb64bbdb-h7nb9 /]# free -m
              total        used        free      shared  buff/cache   available
Mem:           7846        2409        2555           2        2881        5285
Swap:             0           0           0
```

**4， 开启全局开关  （`sysctl -w kernel.stats_isolated=1`）** 

```
[root@VM-0-17-tlinux ~]# sysctl -w kernel.stats_isolated=1
kernel.stats_isolated = 1
[root@VM-0-17-tlinux ~]# sysctl  -p
```

**5，进一步查看容器资源情况**

```
[root@centos-5ccb64bbdb-h7nb9 /]# cat /proc/meminfo | grep MemTotal
MemTotal:        1048576 kB
[root@centos-5ccb64bbdb-h7nb9 /]# cat /proc/cpuinfo  | grep processor | wc -l
1
[root@centos-5ccb64bbdb-h7nb9 /]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1024           4        1014           0           4         934
Swap:          1024           0        1024
```

![1627827658184](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041912725.png)

**5，低内核版本不支持该参数**

> 另外一个节点   操作系统是centos 内核版本是 3.10

```
[root@VM-2-46-centos ~]# sysctl -w kernel.stats_isolated=1
sysctl: cannot stat /proc/sys/kernel/stats_isolated: No such file or directory
[root@VM-2-46-centos ~]# uname -a
Linux VM-2-46-centos 3.10.0-1160.11.1.el7.x86_64 #1 SMP Fri Dec 18 16:34:56 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

![1627828366685](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913987.png)

#### 方案二:使用LXCFS

> [lxcfs官方介绍](https://github.com/denverdino/lxcfs-admission-webhook)

在节点上安装LXCFS ， 通过将宿主机的 /var/lib/lxcfs/meminfo 文件挂载到容器内的/proc/meminfo，然后LXCFS会从容器的CGroup中读取正确的内存限制，然后应用到 /var/lib/lxcfs/meminfo ，这时候容器内部从而就得到了正确的内存信息 

**1，环境准备 （ node节点OS：centos 7.6 ）**

```
#安装依赖
yum install -y fuse-libs
git  clone https://github.com/denverdino/lxcfs-admission-webhook.git
cd  lxcfs-admission-webhook
```

**2，部署安装**

```
#部署lxcfs daemonset
kubectl  apply -f deployment/lxcfs-daemonset.yaml
#部署lxcfs admission webhook
#sh  deployment/install.sh  #删除的话使用sh  deployment/uninstall.sh 
#执行kubectl get po ，确认所有pod都处于Running状态
[root@VM-2-46-centos lxcfs-admission-webhook]# kubectl  get pods  | grep lxcfs
lxcfs-4bjxh                                           1/1     Running   0          8m44s
lxcfs-56225                                           1/1     Running   0          8m44s
lxcfs-admission-webhook-deployment-58d6fdcf49-jmxd9   1/1     Running   0          8m3s
lxcfs-f2gt5                                           1/1     Running   0          8m44s
lxcfs-h6smp                                           1/1     Running   0          8m44s
```

**3，验证效果，启动lxcfs**

 对于要使用 lxcfs 的namespace，使用如下命令启用lxcfs admission webhook的自动注入（以lxcf为例）： 

```
 kubectl label namespace lxcfs  lxcfs-admission-webhook=enabled
 kubectl  get ns --show-labels
 kubectl  get pods -n lxcfs
 
 #部署POD到centos 7.6 节点上
```

![1627830668749](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913268.png)

确认内存信息

![1627830753483](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913663.png)

确认CPU信息

> 如果pod设置了cpu limit，看到cpu数量为cpu limit值向上取整

![1627830843444](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913204.png)



**4，卸载清理lxcfs**

 清理 lxcfs-admission-webhook 

```
deployment/uninstall.sh
```

 清理 lxcfs 

```
kubectl delete -f deployment/lxcfs-daemonset.yaml
```

lxcfs 支持容器镜像 Centos系统、Ubuntu系统、Debian系统，但是不支持容器镜像 Alpine系统。因为 Alpine 不是使用 Gnu libc，而是使用 musl libc





#### 附注：1.12 TKE集群版本集群验证

> 在一次处理客户问题时候，客户咨询1.12版本是否支持lxcfs ，所做以下验证，仅作为记录

**1，环境准备**

-  centos7.6.0_x64 

-  集群版本1.12 


```
[root@VM-2-2-centos ~]# cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 
[root@VM-2-2-centos ~]# kubectl  get nodes -o wide
NAME       STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP       OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
10.1.2.2   Ready    <none>   18m   v1.12.4-tke.29   10.1.2.2      114.117.211.163   CentOS Linux 7 (Core)   3.10.0-1160.11.1.el7.x86_64   docker://19.3.9
```

![1628694819462](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913476.png)



**2，查看当前主机资源情况****

![1628694227083](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913463.png)

**3，未安装lxcfs之前部署POD**

```
        resources:
          limits:
            cpu: 800m
            memory: 1Gi
          requests:
            cpu: 200m
            memory: 256Mi
```

![1628694282800](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041913087.png)

![1628694413673](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914606.png)



**4，安装lxcfs组件 并查看是否成功**

![1628694556172](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914350.png)

**45，测试，销毁刚才创建的工作负载**

**5，登录POD里面查看资源**

![1628695298213](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914545.png)

**6 ，CPU 1:2  内存1:1**

![1628695390761](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914300.png)

![1628695575243](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914669.png)

![1628695524786](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914339.png)

![1628695619794](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041914326.png)





参考链接：

- https://github.com/denverdino/lxcfs-admission-webhook
