---
title: Kubernetes中pod数据存储
tags:
  - Kubernetes
  - docker
categories: Kubernetes
keywords: Kubernetes
description: Kubernetes中pod数据存储
top_img: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031845569.jpeg
abbrlink: 76f3febd
date: 2020-05-26 20:10:42
updated: 2020-05-26 23:58:58
---

## POD如何使用节点磁盘

K8S中，容器container在运行过程中，会产生一些日志，临时文件，如果没有任何限制的话，会写满POD所在节点磁盘空间，从而会影响对应节点 已经节点上其他POD应用，

 容器的**临时存储，例如 emptyDir**，位于目录/var/lib/kubelet/pods 下 

> 通过如下命令可以查询到集群POD所对应的POD_ID
>
> kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeIP:.status.hostIP,Pod_ID:.metadata.uid

```
[root@VM-249-47-tlinux /var/lib/kubelet/pods]# kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeIP:.status.hostIP,Pod_ID:.metadata.uid
podName                   podIP        podStatus   nodeIP          Pod_ID
centos-74cd685986-rcfqk   10.200.0.5   Running     172.30.249.47   9ad30306-f6cd-49eb-a279-cd58201be8c3

[root@VM-249-47-tlinux /var/lib/kubelet/pods]# 
[root@VM-249-47-tlinux /var/lib/kubelet/pods]# tree  9ad30306-f6cd-49eb-a279-cd58201be8c3
9ad30306-f6cd-49eb-a279-cd58201be8c3   #pod的 uid
├── containers                         # pod 里面的container 容器
│   ├── busybox                        #容器1
│   │   └── 21efdec2
│   └── centos                         #容器2
│       └── 64bbf490
├── etc-hosts                          # 命名空间的Host文件
├── plugins
│   └── kubernetes.io~empty-dir
│       ├── wrapped_cm
│       │   └── ready
│       ├── wrapped_default-token-7llnd
│       │   └── ready
│       └── wrapped_secret
│           └── ready
└── volumes                                           # Pod的卷
    ├── kubernetes.io~configmap                       # ConfigMap类型的卷
    │   └── cm
    │       └── app -> ..data/app
    ├── kubernetes.io~qcloud-cbs                      #CBS类型的数据卷
    │   └── pvc-137607ae-974d-4c25-90dc-efc3c2a6c5a8  #PV名称，对应POD里面挂载点
    │       └── lost+found
    └── kubernetes.io~secret                          #Secret类型的卷
        ├── default-token-7llnd
        │   ├── ca.crt -> ..data/ca.crt
        │   ├── namespace -> ..data/namespace
        │   └── token -> ..data/token
        └── secret
            └── password -> ..data/password

17 directories, 11 files
```

**持久卷的挂载点**也位于/var/lib/kubelet/pods 下，但是**不会导致存储空间的消耗**。

容器的日志，存放在/var/log/pods 目录下

```
[root@VM-249-47-tlinux /var/log/pods]# ls -lrt  | grep centos
drwxr-xr-x 4 root root 4096 Aug  7 18:23 default_centos-74cd685986-rcfqk_9ad30306-f6cd-49eb-a279-cd58201be8c3
```

![1659868302215](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031845790.png)

> 目录命名方式是：命名空间\_POD名称\_POD-UID
>
> 9ad30306-f6cd-49eb-a279-cd58201be8c3 这个指的就是POD的uid

日志是软链接到/var/lib/docker/containers/容器ID/容器ID-json.log

![1659868600279](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031846679.png)

 使用 Docker 时，**容器的 rootfs**位于/var/lib/docker 下，具体位置取决于存储驱动。 

