# kubernetes service模式分析

## 概述

Kubernetes service是为POD提供统一访问入口的，实现主要依靠kube-proxy实现，kube-proxy有三种模式userspace、iptables，ipvs，同时我们也知道service有三种类型clusterIP、nodeport，loadblance和三种端口类型port，targetport，nodeport。

## kube-proxy模式分析

### userspace

userspace为kube-proxy为早期的模式，Kubernetes1.2版本之前主要使用这个模式，转发原理参考

https://kubernetes.io/zh/docs/concepts/services-networking/service/

 这个模式最大缺点就是，所以端口请求都需要先经过kube-proxy然后在通过iptables转发，这样带来一个问题就是需要在用户态和内核态不断进行切换，效率低。 

### Iptables

​    Kubernetes在1.2版本开始将iptables做为kube-proxy的默认模式，iptables根之前userspace相比，完全工作在内核态而且不用在经过kube-proxy中转一次性能更强，下面介绍Kubernetes中iptables转发流程，

​    iptables有链和表的概念，链就相当于一道道关卡，表就是这个关卡上对应的规则总共有**四个表和五条链**，kube-proxy在这里就使用了两个表分别是**filter和nat表**，

也自定义了五个链

1. KUBE-SERVICES
2. KUBE-NODE-PORTS
3. KUBE-POSTROUTING
4. KUBE-MARK-MASQ
5. KUBE-MARK-DROP

目前kubernetes提供了两种负载分发策略：RoundRobin和SessionAffinity

RoundRobin：    轮询模式，即轮询将请求转发到后端的各个Pod上（默认是RoundRobin模式）。
SessionAffinity：基于客户端IP地址进行会话保持的模式，第一次客户端访问后端某个Pod，之后的请求都转发到这个Pod上
![1627478908337](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042149890.png)

iptables数据包转发流程
1、首先一个数据包经过网卡进来，先经过PREROUTING链

2、判定目的地址是否为主机，为本机就通过INPUT链转发

3、若不为本机通过FORWARDl链转发到POSTROUTING出去

以下展示一个实例展示kube-proxy是如何根据service不同类型生成对应规则

 我们创建名为nginx的deployment，镜像为nginx:latest，replicas为3个 

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
  replicas: 3
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
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 64Mi
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey

```

 在给这个deployment创建一个ClusterIP类型的service 

 

```
[root@VM-0-17-tlinux ~]# kubectl  expose deployment/nginx  --type=ClusterIP --port=80

[root@VM-0-17-tlinux ~]# kubectl  get deployment,pods,svc | grep nginx
deployment.apps/nginx                  3/3     3            3           3m23s
pod/nginx-754f548f9b-5rpzf    1/1     Running   0          3m23s
pod/nginx-754f548f9b-vfdjv    1/1     Running   0          3m23s
pod/nginx-754f548f9b-x6kv5    1/1     Running   0          3m23s
service/nginx        ClusterIP   172.18.255.127   <none>        80/TCP    44s
```

导出iptables规则：

##### ClusterIP类型

查看规则，首先Kubernetes会指对每个service在创建一些名为KUBE-SEP-xxx，KUBE-SVC-xxx的链

以刚刚创建的类型为Cluster-ip名为test这个service为例，创建了以下规则

```
[root@VM-0-17-tlinux ~]# iptables-save  | grep nginx | grep -v nginx-ingress  
-A KUBE-SEP-G4A7DIXDTUZ6AHE3 -s 172.18.0.5/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-G4A7DIXDTUZ6AHE3 -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 172.18.0.5:80
-A KUBE-SEP-J5YCYOGHESVN34H2 -s 172.18.0.6/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-J5YCYOGHESVN34H2 -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 172.18.0.6:80
-A KUBE-SEP-TPLSNUHWNBT6MT2B -s 172.18.0.7/32 -m comment --comment "default/nginx:" -j KUBE-MARK-MASQ
-A KUBE-SEP-TPLSNUHWNBT6MT2B -p tcp -m comment --comment "default/nginx:" -m tcp -j DNAT --to-destination 172.18.0.7:80
-A KUBE-SERVICES -d 172.18.255.127/32 -p tcp -m comment --comment "default/nginx: cluster IP" -m tcp --dport 80 -j KUBE-SVC-4N57TFCL4MD7ZTDA
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-G4A7DIXDTUZ6AHE3
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-J5YCYOGHESVN34H2
-A KUBE-SVC-4N57TFCL4MD7ZTDA -m comment --comment "default/nginx:" -j KUBE-SEP-TPLSNUHWNBT6MT2B
```

https://www.bladewan.com/2018/12/10/kubernetes_service_mode/