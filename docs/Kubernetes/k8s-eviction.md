---
title: Kubernetes常见驱逐Eviction问题
tags:
  - Kubernetes
  - Docker
categories: Kubernetes
keywords: Kubernetes
description: Kubernetes常见驱逐Eviction问题
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210231937256.jpeg
abbrlink: d0686bc1
date: 2021-10-01 20:00:57
updated: 2021-10-01 23:58:58
---

> 在高可用的k8s集群中，当Node节点挂掉，kubelet无法提供工作的时候，pod将会驱逐并自动调度到其他的节点上去，或者在node节点的资源紧缺的条件下，kubelet为了保证node节点的稳定性，也回触发主动驱逐pod的机制

## POD驱逐

Eviction状态介绍：

- Eviction，即驱逐的意思，意思是当节点出现异常时，为了保证工作负载的可用性，kubernetes将有相应的机制驱逐该节点上的Pod。
  目前kubernetes中存在两种eviction机制，分别由kube-controller-manager和kubelet实现。
- kube-controller-manager的eviction机制是粗粒度的，即驱赶一个节点上的所有pod，而kubelet则是细粒度的，它驱赶的是节点上的某些Pod，驱赶哪些Pod与Pod的Qos机制有关。该Eviction会周期性检查本节点内存、磁盘等资源，当资源不足时，按照优先级驱逐部分pod

### kube-controller-manager实现的eviction

kube-controller-manager主要由多个控制器构成，而eviction的功能主要由node controller这个控制器实现。该Eviction会周期性检查所有节点状态，当节点处于**NotReady**状态超过一段时间后，驱逐该节点上所有pod

Kubelet 状态更新的基本流程：

- kubelet 自身会定期更新状态到 apiserver，通过参数--node-status-update-frequency指定上报频率，默认是 10s 上报一次。
- kube-controller-manager 会每隔--node-monitor-period时间去检查 kubelet 的状态，默认是 5s。
- 当 node 失联一段时间后，kubernetes 判定 node 为 notready 状态，这段时长通过--node-monitor-grace-period参数配置，默认 40s。
- 当 node 失联一段时间后，kubernetes 判定 node 为 unhealthy 状态，这段时长通过--node-startup-grace-period参数配置，默认 1m0s。
- 当 node 失联一段时间后，kubernetes 开始删除原 node 上的 pod，这段时长是通过--pod-eviction-timeout参数配置，默认 5m0s。

> kube-controller-manager 和 kubelet 是异步工作的，这意味着延迟可能包括任何的网络延迟、apiserver 的延迟、etcd 延迟，一个节点上的负载引起的延迟等等。因此，如果--node-status-update-frequency设置为5s，那么实际上 etcd 中的数据变化会需要 6-7s，甚至更长时间。

启动参数控制eviction：

- **pod-eviction-timeout：**即当节点宕机该时间间隔后，开始eviction机制，驱赶宕机节点上的Pod，默认为5min。
- **node-eviction-rate：**驱赶速率，即驱赶Node的速率，由令牌桶流控算法实现，默认为0.1，即每秒驱赶0.1个节点，注意这里不是驱赶Pod的速率，而是驱赶节点的速率。相当于每隔10s，清空一个节点。
- **secondary-node-eviction-rate：**二级驱赶速率，当集群中宕机节点过多时，相应的驱赶速率也降低，默认为0.01。
- **unhealthy-zone-threshold：**不健康zone阈值，会影响什么时候开启二级驱赶速率，默认为0.55，即当该zone中节点宕机数目超过55%，而认为该zone不健康。
- **large-cluster-size-threshold：**大集群阈值，当该zone的节点多余该阈值时，则认为该zone是一个大集群。大集群节点宕机数目超过55%时，则将驱赶速率降为0.01，假如是小集群，则将速率直接降为0。

社区默认的配置：

| 参数                          | 值   |
| ----------------------------- | ---- |
| –node-status-update-frequency | 10s  |
| –node-monitor-period          | 5s   |
| –node-monitor-grace-period    | 40s  |
| –pod-eviction-timeout         | 5m   |

### kubelet基于节点压力eviction机制

如果节点处于资源压力，那么kubelet就会执行驱逐策略。驱逐会考虑Pod的优先级，资源使用和资源申请。当优先级相同时，资源使用/资源申请最大的Pod会被首先驱逐，更多详情可以参考[官方文档](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

kubelet提供了以下参数控制eviction：

- **eviction-soft：**软驱逐阈值设置，具有一系列阈值，比如memory.available<1.5Gi时，它不会立即执行pod eviction，而会等待eviction-soft-grace-period时间，假如该时间过后，依然还是达到了eviction-soft，则触发一次pod eviction。
- **eviction-soft-grace-period：**默认为90秒，当eviction-soft时，终止Pod的grace的时间，即软驱逐宽限期，软驱逐信号与驱逐处理之间的时间差。
- **eviction-max-pod-grace-period：**最大驱逐pod宽限期，停止信号与kill之间的时间差。
- **eviction-pressure-transition-period：**默认为5分钟，脱离pressure condition的时间，超过阈值时，节点会被设置为memory pressure或者disk pressure，然后开启pod eviction。
- **eviction-minimum-reclaim：**表示每一次eviction必须至少回收多少资源。
- **eviction-hard****：**强制驱逐设置，也具有一系列的阈值，比如memory.available<1Gi，即当节点可用内存低于1Gi时，会立即触发一次pod eviction。

### 驱逐问题处理

1，使用如下命令发现很多pod的状态为Evicted：

```bash
kubectl get pods   -o wide --all-namespaces | grep -i Evicted
```

2，在节点的kubelet日志中会记录Evicted相关内容，搜索方法可参考如下命令：

```bash
journalctl  -u kubelet | grep -i Evicted -C3
```

3，查看节点监控是否有内存打满或者 节点磁盘

4，清理异常状态Evicted的POD

> <namespace>  为命名空间名称，请根据需要指定。

```bash
kubectl get pods <namespace> | grep Evicted | awk '{print $1}' | xargs kubectl delete pod <namespace> 
```

或者使用这个命令清理所有命名空间驱逐状态的POD

```bash
kubectl get pod -o wide --all-namespaces | awk '{if($4=="Evicted"){cmd="kubectl -n "$1" delete pod "$2; system(cmd)}}'
```

5，pod驱逐后，如果新调度到的节点也有驱逐情况，就会再次被驱逐；甚至出现pod不断被驱逐的情况,，需要确保集群资源充足

6，如果是由kube-controller-manager触发的驱逐，会留下一个状态为Terminating的pod；直到容器所在节点状态恢复后，pod才会自动删除。如果节点已经删除或者其他原因导致的无法恢复，可以使用“强制删除”删除pod，

7，如果是由kubelet触发的驱逐，会留下一个状态为Evicted的pod，此pod只是方便后期定位的记录，可以直接删除。









