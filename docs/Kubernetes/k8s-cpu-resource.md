# 为容器和 Pods 分配 CPU 资源

 本章展示如何为容器设置 CPU *request（请求）* 和 CPU *limit（限制）*。 容器使用的 CPU 不能超过所配置的限制。 如果系统有空闲的 CPU 时间，则可以保证给容器分配其所请求数量的 CPU 资源 

## 创建一个名字空间

创建一个[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)，以便将 本练习中创建的资源与集群的其余部分资源隔离（本例和内存章节使用同一个命名空间验证）

```shell
kubectl create namespace cpu-example
```

## 指定 CPU 请求和 CPU 限制

要为容器指定 CPU 请求，请在容器资源清单中包含 `resources: requests` 字段。 要指定 CPU 限制，请包含 `resources:limits`。

在本练习中，你将创建一个具有一个容器的 Pod。容器将会请求 0.5 个 CPU，而且最多限制使用 1 个 CPU。 这是 Pod 的配置文件：cpu-request-limit.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: cpu-request-limit
    qcloud-app: cpu-request-limit
  name: cpu-request-limit
  namespace: mem-example
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: cpu-request-limit
      qcloud-app: cpu-request-limit
  template:
    metadata:
      labels:
        k8s-app: cpu-request-limit
        qcloud-app: cpu-request-limit
    spec:
      containers:
      - args:
        - -cpus
        - "2"
        image: vish/stress:latest
        imagePullPolicy: IfNotPresent
        name: cpu-demo-ctr
        resources:
          limits:
            cpu: "1"
          requests:
            cpu: 500m
```

 配置文件的 `args` 部分提供了容器启动时的参数。 `-cpus "2"` 参数告诉容器尝试使用 2 个 CPU。  创建 Pod

 验证所创建的 Pod 处于 Running 状态 

```
[root@VM-0-17-tlinux ~/memonry]# kubectl  get pod -n mem-example 
NAME                                      READY   STATUS    RESTARTS   AGE
cpu-request-limit-7bbdd6c6c8-789ks        1/1     Running   0          2m20s
```

![1628424589849](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042131869.png)

 查看显示关于 Pod 的详细信息： 

```
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 500m
```

![1628424615810](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042131917.png)

 使用 `kubectl top` 命令来获取该 Pod 的度量值数据： 

 此示例输出显示 Pod 使用的是 999milliCPU，即略低于 Pod 配置中指定的 1 个 CPU 的限制。 

```
[root@VM-0-17-tlinux ~/memonry]# kubectl  top pod cpu-request-limit-7bbdd6c6c8-789ks  -n mem-example 
NAME                                 CPU(cores)   MEMORY(bytes)   
cpu-request-limit-7bbdd6c6c8-789ks   999m         1Mi             
```

![1628424713167](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042131958.png)

 通过设置 `-cpu "2"`，你将容器配置为尝试使用 2 个 CPU， 但是容器只被允许使用大约 1 个 CPU。 容器的 CPU 用量受到限制，因为该容器正尝试使用超出其限制的 CPU 资源。 

 CPU 使用率低于 1.0 的另一种可能的解释是，节点可能没有足够的 CPU 资源可用。 回想一下，此练习的先决条件需要你的节点至少具有 1 个 CPU 可用。 如果你的容器在只有 1 个 CPU 的节点上运行，则容器无论为容器指定的 CPU 限制如何， 都不能使用超过 1 个 CPU。 

## CPU 单位

 小数值是可以使用的。一个请求 0.5 CPU 的容器保证会获得请求 1 个 CPU 的容器的 CPU 的一半。 你可以使用后缀 `m` 表示毫。例如 `100m` CPU、100 milliCPU 和 0.1 CPU 都相同。 精度不能超过 1m。 

 CPU 请求只能使用绝对数量，而不是相对数量。0.1 在单核、双核或 48 核计算机上的 CPU 数量值是一样的。 

## 设置超过节点能力的 CPU 请求

CPU 请求和限制与都与容器相关，但是我们可以考虑一下 Pod 具有对应的 CPU 请求和限制这样的场景。 Pod 对 CPU 用量的请求等于 Pod 中所有容器的请求数量之和。 同样，Pod 的 CPU 资源限制等于 Pod 中所有容器 CPU 资源限制数之和。

Pod 调度是基于资源请求值来进行的。 仅在某节点具有足够的 CPU 资源来满足 Pod CPU 请求时，Pod 将会在对应节点上运行