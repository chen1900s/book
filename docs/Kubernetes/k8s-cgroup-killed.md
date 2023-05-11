---
title: Kubernetes中的cgroup Killed问题
tags: Kubernetes
categories: Kubernetes
keywords: Kubernetes
description: Kubernetes中的cgroup Killed问题
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210231937256.jpeg
abbrlink: dcec15df
date: 2022-12-11 17:07:38
updated: 2022-12-11 17:07:38
---

> 环境：
>
> 1，腾讯云TKE集群
>
> 2，CVM节点 内核：tlinux 4.14.105-19-0024
>
> 3，容器目录和kubelet目录默认设置

**Cgroup OOM原因**

一般是由于容器的内存实际使用量超过了容器内存限制值limit而导致的事件。比如容器的内存限制值配置了1Gi，而容器的内存随着容器内进程内存使用量的增加超过了1Gi，就会导致容器被操作系统Cgroup Kill。发生容器被Kill之后，容器已经被停止，所以后续会出现应用实例被重启的情况

**解决方案**

检查容器内进程是否有内存泄漏问题，同时适当调整容器内存的限制值limit大小。可以结合应用监控来看变化趋势。需要注意的是，容器内存限制值大小不应该过大，否则可能导致极端资源争抢情况下，容器被迫驱逐的问题。 

**oom score**

在遇到较高内存使用压力时，Linux 内核会杀掉一些不太重要的进程，腾出空间保障系统正常运行。它会给每个进程（`/proc/$pid/oom_score`）分配一个得分（`oom_score`），分数越高，被 OOM 的概率就越大。 **这个参数本身只反映该进程的可用资源在系统中所占的百分比，并没有“该进程有多重要”的概念** 



在kubernetes中各个POD之间资源是通过cgroup进行资源隔离，查看POD cgroup相关信息，登录POD所在节点，执行

```bash
cd  /sys/fs/cgroup/memory/kubepods/burstable/pod[pod的uuid]    #uuid可以通过下面命令查询

示例：
cd /sys/fs/cgroup/memory/kubepods/burstable/pod7b5d76c8-a37b-4f1c-8db9-383017063244
```

查下集群POD的uuid（根据具体情况可以根据命名空间查询）

```bash
kubectl get pods -o custom-columns=Namespace:..metadata.namespace,podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeIP:.status.hostIP,Pod_ID:.metadata.uid,ContainerName:.spec.containers[*].name
```



![1668051091904](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111709485.png)

当POD发生OOM时候，可以查看dmesg日志：

```bash
dmesg  | grep -A  20  7b5d76c8-a37b-4f1c-8db9-38301706324
```

![1668051203724](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111710803.png)



**如何通过oom kill日志反查对应的容器**

通过dmesg日志查到 有如下oom 信息：

```bash
dmesg  | grep -A  20 -i  killed
```

日志如下

```bash
[198542.576728] Task in /kubepods/burstable/pod0f5eb28f-6722-4cf5-ab0f-b62748c018d4/0dc1822caf9e150de25daa1cf572418426ce6b72757b5efdff52acef348368a3 killed as a result of limit of /kubepods/burstable/pod0f5eb28f-6722-4cf5-ab0f-b62748c018d4
[198542.576733] memory: usage 122880kB, limit 122880kB, failcnt 83
[198542.576734] memory+swap: usage 122880kB, limit 9007199254740988kB, failcnt 0
[198542.576734] kmem: usage 0kB, limit 9007199254740988kB, failcnt 0
[198542.576735] Memory cgroup stats for /kubepods/burstable/pod0f5eb28f-6722-4cf5-ab0f-b62748c018d4: cache:0KB rss:0KB rss_huge:0KB shmem:0KB mapped_file:0KB dirty:0KB writeback:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
```

可以查到POD的uuid：0f5eb28f-6722-4cf5-ab0f-b62748c018d4

通过kubelet 日志或者message日志过滤该uid

```bash
journalctl -u kubelet | grep "0f5eb28f-6722-4cf5-ab0f-b62748c018d4" #POD uuid
```

日志如下，可以看到具体POD信息 包括POD名称和所在namespace

```bash
Dec 11 17:29:21 VM-249-96-tlinux kubelet[10264]: E1211 17:29:21.519957   10264 pod_workers.go:191] Error syncing pod 0f5eb28f-6722-4cf5-ab0f-b62748c018d4 ("memory-request-limit-54ff657644-wm7pj_default(0f5eb28f-6722-4cf5-ab0f-b62748c018d4)"), skipping: failed to "StartContainer" for "memory-demo-ctr" with CrashLoopBackOff: "back-off 5m0s restarting failed container=memory-demo-ctr pod=memory-request-limit-54ff657644-wm7pj_default(0f5eb28f-6722-4cf5-ab0f-b62748c018d4)"
```

