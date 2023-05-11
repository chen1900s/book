### Network Policy

[Network Policy ](https://kubernetes.io/docs/concepts/services-networking/network-policies/)是 Kubernetes 提供的一种资源，用于定义基于 Pod 的网络隔离策略。描述了一组 Pod 是否可以与其他组 Pod，以及其他 network endpoints 进行通信。

### 在 TKE 上启用 NetworkPolicy 扩展组件

具体操作步骤可参见 [NetworkPolicy 说明](https://cloud.tencent.com/document/product/457/50841)。

本次测试环境：集群版本1.18 

容器网络 ：  172.16.0.0/16 

集群网络：   172.30.0.0/16 

### NetworkPolicy 配置示例

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: nginx
    qcloud-app: nginx
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: nginx
      qcloud-app: nginx
  template:
    metadata:
      labels:
        k8s-app: nginx
        qcloud-app: nginx
    spec:
      containers:
      - image: nginx:stable
        imagePullPolicy: IfNotPresent
        name: nginx
        resources: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: centos
    qcloud-app: centos
  name: centos
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: centos
      qcloud-app: centos
  template:
    metadata:
      labels:
        k8s-app: centos
        qcloud-app: centos
    spec:
      containers:
      - image: ccr.ccs.tencentyun.com/v_cjweichen/centos:latest
        imagePullPolicy: IfNotPresent
        name: centos
        resources: {}

```

```
[root@VM-249-8-tlinux ~]# kubectl  get  pods -A  -o wide  | grep -v kube-system
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
default       centos-59cd95cbbf-jjm6j            1/1     Running   0          3m15s   172.16.0.5     172.30.249.8   <none>           <none>
default       nginx-686bf668dc-vgndc             1/1     Running   0          2m38s   172.16.0.6     172.30.249.8   <none>           <none>
nsa           centos-f47c886c9-l2tk2             1/1     Running   0          3m51s   172.16.0.4     172.30.249.8   <none>           <none>
nsa           nginx-686bf668dc-kckln             1/1     Running   0          2m4s    172.16.0.7     172.30.249.8   <none>           <none>
```

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151550.png) 



**案例一：nsa namespace 下的 Pod 可互相访问，而不能被其他任何 Pod 访问。**

为 部署network policy 之前是可以正常访问

![1653390992703](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151601.png)

 

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
     name: npa
     namespace: nsa
spec:
     ingress: 
     - from:
       - podSelector: {} 
     podSelector: {} 
     policyTypes:
     - Ingress
```

在nsa 命名空间下的centos POD访问同一个命名空间（nsa）下的nginx服务

![1653391215925](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151655.png)

在default 命名空间下的centos 访问nsa命名空间下nginx服务，无法访问

或使用命令行

```
[root@VM-249-8-tlinux ~]# kubectl  exec -it centos-66dbbdc5b6-8cbk4  -- curl -I 172.16.0.7
```

![1653391278777](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151691.png)



**案例二： nsa namespace 下的 Pod 不能被任何 Pod 访问** 

```
[root@VM-249-8-tlinux ~]# kubectl   -n nsa   delete NetworkPolicy npa
networkpolicy.networking.k8s.io "npa" deleted
```

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
     name: npa
     namespace: nsa
spec:
     podSelector: {}
     policyTypes:
     - Ingress
```

```
[root@VM-249-8-tlinux ~]# kubectl   -n nsa get NetworkPolicy
NAME   POD-SELECTOR   AGE
npa    <none>         46s
```

在nsa 命名空间下的centos POD访问同一个命名空间（nsa）下的nginx服务，无法访问

![1653391994712](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151740.png)

在default 命名空间下的centos 访问nsa命名空间下nginx服务，无法访问

![1653392054013](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151779.png)



**案例三： nsa namespace 下的 Pod 只在 80/TCP 端口可以被带有标签 app: nsb 的 namespace 下的 Pod 访问，而不能被其他任何 Pod 访问。** 

```
root@VM-249-8-tlinux ~]# kubectl   -n nsa   delete NetworkPolicy npa
networkpolicy.networking.k8s.io "npa" deleted
```

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: npa
  namespace: nsa
spec:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          app: nsb
    ports:
    - port: 80
      protocol: TCP
  podSelector: {}
  policyTypes:
  - Ingress
```

目前集群内所有的namespace 都没有app: nsb 标签，访问失败

![1653392745448](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151817.png)

![1653392755880](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151857.png)

给default命名空间打labels 

```
[root@VM-249-8-tlinux ~]# kubectl  label ns default app=nsb
namespace/default labeled
[root@VM-249-8-tlinux ~]# kubectl  get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   52m   app=nsb
```

![1653393017565](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042151914.png)

