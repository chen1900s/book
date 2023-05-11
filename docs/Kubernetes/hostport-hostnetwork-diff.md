##  hostPort示例

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: nginx-hostport
    qcloud-app: nginx-hostport
  name: nginx-hostport
  namespace: cjweichen
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: nginx-hostport
      qcloud-app: nginx-hostport
  template:
    metadata:
      labels:
        k8s-app: nginx-hostport
        qcloud-app: nginx-hostport
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          hostPort: 8000
          name: http
          protocol: TCP
        - containerPort: 443
          hostPort: 44300
          name: https
          protocol: TCP


```

![1652866958008](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145450.png)

![1652867256089](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145493.png)





## hostPort 与 hostNetwork 异同 

### 相同点

​    hostPort 与 hostNetwork 本质上都是暴露 pod 宿主机 IP 给终端用户，因为 pod 生命周期并不固定，随时都有可能异常重建，故 IP 的不确定最终导致用户使用上的不方便；此外宿主机端口占用也导致不能在同一台机子上有多个程序使用同一端口。因此一般情况下，不要使用 hostPort 方式。

### 不同点

​    使用 hostNetwork，pod 实际上用的是 pod 宿主机的网络地址空间：即 pod  IP 是宿主机 IP，而非 cni 分配的 pod IP，端口是宿主机网络监听接口。

使用 hostPort，pod IP 并非宿主机 IP，而是 cni 分配的 pod IP，跟其他普通的 pod 使用一样的 ip 分配方式，端口并非宿主机网络监听端口，只是使用了 DNAT 机制将 hostPort 指定的端口映射到了容器的端口之上（可以通过 iptables 命令进行查看）。外部访问此 pod 时，仍然使用宿主机和 hostPort 方式。pod ip 跟宿主机 ip 截图如

![1652867334640](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145534.png)

 有关端口 DNAT 通过 iptables 命令进行查看，如下截图所示： 

```
iptables -nvL -t nat  | grep -C 5 DNAT
```

![1653016603475](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145588.png)

 由上图可知，pod 所在宿主机上的 iptables nat 表流向如下： 

1 当客户端发起 pod 访问时，比如 curl http://pod_in_host:hostPort

2 网络包会流经 pod 宿主机的 prerouting chain，会命中 CNI-HOSTPORT-DNAT 链

3 网络包会流经 CNI-HOSTPORT_DNAT 链中的第 3 条规则，即 DNAT 目标，此时会将 9998 端口访问的流量路由到 80 端口去





 基于此，当客户端访问 pod 所在主机的 8000 端口时，流量会自动被路由到 IP 为  10.55.3.5（也就是 pod ip）的 80 端口上。 



​     当 pod 同时使用了 hostNetwork 和 hostPort，那么 hostNetwork 将会直接使用宿主机网络命名空间，hostPort 其实就形同虚设了。可以认为 hostNetwork 选项优先级要高于 hostPort。 

参考文档： https://blog.51cto.com/u_14625168/2489160