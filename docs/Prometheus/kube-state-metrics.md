---
title: Kube-state-metrics部署安装
abbrlink: b9437839
date: 2021-09-09 21:58:58
tags:
  - Prometheus
  - Kubernetes
categories: Kubernetes
keywords:
  - Kubernetes
  - Prometheus
  - Kube-state-metrics
description: 在kubernetes集群中部署Kube-state-metrics
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209191619695.jpg
updated: 2022-09-17 20:29:11
---

## 背景

在kubernetes集群中已经有了 cadvisor、heapster、metric-server，基本上容器运行的所有指标都能拿到，但是针对下面这种情况获取不到：

- 比如调度了多少个 replicasset ，现在可用的有几个？
- 多少个 Pod 是 running/stopped/terminated 状态？
- Pod 重启了多少次？
- 有多少 job 在运行中

而这些指标采集则需要 kube-state-metrics 进行采集，它基于 client-go 开发，负责监听 K8s apiserver 从而生成metrics数据，指标数据通过 `/metrics` Endpoint 暴露，主要是适配 Prometheus

## 部署安装

**1，下载将 [kube-state-metrics examples](https://github.com/kubernetes/kube-state-metrics/blob/master/examples/standard) 几个文件，分别为**

```bash
 kube-state-metrics/    
 ├── cluster-role-binding.yaml  
 ├── cluster-role-binding.yaml   
 ├── deployment.yaml    
 ├── service-account.yaml   
 ├── service.yaml    
```

**2，修改镜像地址**（默认镜像地址k8s.gcr.io 在国外。可以找个海外机器拉取下来上传到国内镜像仓库进行拉取上传到自己的镜像仓库里面）

```bash
- image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.6.0

#本人已经拉取到自己镜像仓库 可以修改成
- image: ccr.ccs.tencentyun.com/chenjingwei/kube-state-metrics:v2.6.0
```

**3，安装kube-state-metrics**

```bash
kubectl apply -f ./
```

**4，查看是否安装成功**

```bash
# kubectl get pods -n kube-system -o wide | grep kube-state-metrics
kube-state-metrics-57768576b6-mh5d8   1/1     Running   0   2m19s   172.16.0.12      172.30.249.130   <none>      <none>

#通过 /healthz 健康检查端口查看Pod状态。
# curl  172.16.0.12:8080/healthz
OK


#通过 /metrics 接口可查看其采集的全量数据
# curl 172.16.0.12:8080/metrics

#登陆启动POD里面访问测试
[root@centos-777bdddd57-zv7bv /]# curl kube-state-metrics.kube-system.svc.cluster.local:8080
<html>
             <head><title>Kube Metrics Server</title></head>
             <body>
             <h1>Kube Metrics</h1>
                         <ul>
             <li><a href='/metrics'>metrics</a></li>
             <li><a href='/healthz'>healthz</a></li>
                         </ul>
             </body>
</html>
#表示安装成功
```

> 启动时候有可能端口冲突 可以按照如下文档 添加启动参数，[参考文档](https://github.com/bitnami/bitnami-docker-kube-state-metrics/blob/2.1.0-debian-10-r15/2/debian-10/Dockerfile)



## 与prometheus集成

修改prometheus配置文件，添加job_name

> kube-state-metrics是部署在kube-system命名空间下的，因此在正则匹配上，命名空间为kube-system，svc名称为kube-state-metrics，否则就不进行监控，k8s kube-state-metrics监控任务

```yaml
      job_name: "kube-state-metrics"
      kubernetes_sd_configs:
       - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_endpoints_name]
        regex: kube-system;kube-state-metrics
        action: keep
    - bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

![image-20220917220729053](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209172207163.png)

**6，查看prometheus的targets**

部署成功后，prometheus的target会出现如下标志

![image-20220917215311281](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209172153353.png)

可以看到kube-state-metrics监控任务下已经有两个Targets实例了。我们可以到grup界面通过PromQL语句查询相关监控数据。

示例： 查看当前K8s集群有多少configmap资源对象

```
count(kube_configmap_created)
```

![image-20220917215648632](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209172156719.png)

使用 kube-state-metrics 后的常用场景有：

- 存在执行失败的 Job

  ```yaml
  kube_job_status_failed{job="kubernetes-service-endpoints",k8s_app="kube-state-metrics"}==1
  ```

- 集群节点状态错误

  ```yaml
  kube_node_status_condition{condition="Ready",status!="true"}==1
  ```

- 集群中存在启动失败的 Pod

  ```yaml
   kube_pod_status_phase{phase=~"Failed|Unknown|Pending"}==1
  ```

  ![image-20220917221857125](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209172218318.png)

- 最近30分钟内有 Pod 容器重启

  ```yaml
  changes(kube_pod_container_status_restarts[30m])>0
  ```

- 查看当前集群有多少个Pod正在运行

  ```
  count (kube_pod_container_state_started)
  ```

  ![image-20220917220259158](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209172202253.png)

配合报警alertmanager可以更好地监控集群的运行



## 与metric-server的对比

- metric-server（或heapster）是从 api-server 中获取 cpu、内存使用率这种监控指标，并把他们发送给存储后端，如 influxdb 或云厂商，他当前的核心作用是：为 HPA 等组件提供决策指标支持。
- kube-state-metrics 关注于获取 k8s 各种资源的最新状态，如 deployment 或者 daemonset，之所以没有把kube-state-metrics 纳入到 metric-server 的能力中，是因为他们的关注点本质上是不一样的。metric-server仅仅是获取、格式化现有数据，写入特定的存储，实质上是一个监控系统。而 kube-state-metrics 是将 k8s 的运行状况在内存中做了个快照，并且获取新的指标，但他没有能力导出这些指标
- 换个角度讲，kube-state-metrics 本身是 metric-server 的一种数据来源，虽然现在没有这么做。
- 另外，像 Prometheus 这种监控系统，并不会去用 metric-server 中的数据，他都是自己做指标收集、集成的（Prometheus包含了metric-server的能力），但 Prometheus 可以监控 metric-server 本身组件的监控状态并适时报警，这里的监控就可以通过 kube-state-metrics 来实现，如 metric-server pod 的运行状态。

## 优化点和问题

- 因为 kube-state-metrics 是监听资源的 add、delete、update 事件，那么在 kube-state-metrics 部署之前已经运行的资源，岂不是拿不到数据？kube-state-metric 利用 client-go 可以初始化所有已经存在的资源对象，确保没有任何遗漏
- kube-state-metrics 当前不会输出 metadata 信息(如 help 和 description）
- 缓存实现是基于 golang 的 map，解决并发读问题当期是用了一个简单的互斥锁，可以解决问题，后续会考虑golang 的 sync.Map 安全 map。
- kube-state-metrics 通过比较 resource version 来保证 event 的顺序
- kube-state-metrics 并不保证包含所有资源

## 相关yaml文件

附上部署的yaml：

```yaml
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - serviceaccounts
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - list
  - watch
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs:
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  - ingresses
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - rolebindings
  - roles
  verbs:
  - list
  - watch
  
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/version: 2.6.0
    spec:
      automountServiceAccountToken: true
      containers:
      - image: ccr.ccs.tencentyun.com/chenjingwei/kube-state-metrics:v2.6.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsUser: 65534
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
      
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```

