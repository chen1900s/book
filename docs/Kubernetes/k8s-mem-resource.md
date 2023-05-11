# 为容器和 Pod 分配内存资源

此页面展示如何将内存 *请求* （request）和内存 *限制* （limit）分配给一个容器。 我们保障容器拥有它请求数量的内存，但不允许使用超过限制数量的内存。

## 1，创建命名空间

```
kubectl create namespace mem-example
```

## 2，指定内存请求和限制

要为容器指定内存请求，请在容器资源清单中包含 `resources：requests` 字段。 同理，要指定内存限制，请包含 `resources：limits`。

在本练习中，你将创建一个拥有一个容器的 Pod。 容器将会请求 100 MiB 内存，并且内存会被限制在 200 MiB 以内。 这是 Pod 的配置文件：后续全部以deployment部署

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: memory-request-limit
    qcloud-app: memory-request-limit
  name: memory-request-limit
  namespace: mem-example
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: memory-request-limit
      qcloud-app: memory-request-limit
  template:
    metadata:
      labels:
        k8s-app: memory-request-limit
        qcloud-app: memory-request-limit
    spec:
      containers:
      - args:
        - --vm
        - "1"
        - --vm-bytes
        - 150M
        - --vm-hang
        - "1"
        command:
        - stress
        image: polinux/stress:latest
        imagePullPolicy: IfNotPresent
        name: memory-demo-ctr
        resources:
          limits:
            cpu: 500m
            memory: 200Mi
          requests:
            cpu: 250m
            memory: 100Mi

```

 配置文件的 `args` 部分提供了容器启动时的参数。 `"--vm-bytes", "150M"` 参数告知容器尝试分配 150 MiB 内存。 

 开始创建 Pod： 

```
 kubectl  apply -f memory-request-limit.yaml 
```

 验证 Pod 中的容器是否已运行： 

```
[root@VM-0-17-tlinux ~/memonry]# kubectl  get deployments,pods -n mem-example 
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/memory-request-limit   1/1     1            1           4m52s

NAME                                       READY   STATUS    RESTARTS   AGE
pod/memory-request-limit-585f76b89-sx7v8   1/1     Running   0          4m52s
```

 输出结果显示：该 Pod 中容器的内存请求为 100 MiB，内存限制为 200 MiB。 

![1628420380965](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042130260.png)

 运行 `kubectl top` 命令，获取该 Pod 的指标数据： 

```
root@VM-0-17-tlinux ~/memonry]# kubectl  top pod -n mem-example 
NAME                                    CPU(cores)   MEMORY(bytes)   
memory-request-limit-599b8774d8-dktwv   81m          151Mi           
```

![1628420802978](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042130312.png)

## 3，超过容器限制的内存

 当节点拥有足够的可用内存时，容器可以使用其请求的内存。 但是，容器不允许使用超过其限制的内存。 如果容器分配的内存超过其限制，该容器会成为被终止的候选容器。 如果容器继续消耗超出其限制的内存，则终止容器。 如果终止的容器可以被重启，则 kubelet 会重新启动它，就像其他任何类型的运行时失败一样。 

 在本练习中，你将创建一个 Pod，尝试分配超出其限制的内存。 这是一个 Pod 的配置文件，其拥有一个容器，该容器的内存请求为 50 MiB，内存限制为 100 MiB： 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: memory-request-limit-2
    qcloud-app: memory-request-limit-2
  name: memory-request-limit-2
  namespace: mem-example
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: memory-request-limit-2
      qcloud-app: memory-request-limit-2
  template:
    metadata:
      labels:
        k8s-app: memory-request-limit-2
        qcloud-app: memory-request-limit-2
    spec:
      containers:
      - args:
        - --vm
        - "1"
        - --vm-bytes
        - 250M
        - --vm-hang
        - "1"
        command:
        - stress
        image: polinux/stress:latest
        imagePullPolicy: IfNotPresent
        name: memory-demo-ctr
        resources:
          limits:
            memory: 100Mi
          requests:
            memory: 50Mi

```

 在配置文件的 `args` 部分中，你可以看到容器会尝试分配 250 MiB 内存，这远高于 100 MiB 的限制。 

 此时，容器可能正在运行或被杀死。重复前面的命令，直到容器被杀掉： 查看yaml文件

```
[root@VM-0-17-tlinux ~/memonry]# kubectl  get pods  memory-request-limit-2-7f6548fd7b-n8smk   -n mem-example -o yaml
```

![1628421273729](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042130356.png)

![1628421317364](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042130402.png)



裸POD模式：

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

![1628421101365](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042130449.png)



```
kubectl  get pod -n mem-example -o yaml
```

```
    containerStatuses:
    - containerID: docker://e2a137f7d65e0e4f2ec7da7e036c2e1085b00f1145cccf92cc5169c9a1ee0c48
      image: polinux/stress
      imageID: docker-pullable://polinux/stress@sha256:b6144f84f9c15dac80deb48d3a646b55c7043ab1d83ea0a697c09097aaad21aa
      lastState:
        terminated:
          containerID: docker://e2a137f7d65e0e4f2ec7da7e036c2e1085b00f1145cccf92cc5169c9a1ee0c48
          exitCode: 1
          finishedAt: "2021-08-08T11:11:33Z"
          reason: OOMKilled
          startedAt: "2021-08-08T11:11:33Z"
```

 本练习中的容器可以被重启，所以 kubelet 会重启它。 多次运行下面的命令，可以看到容器在反复的被杀死和重启： 输出结果显示：容器被杀掉、重启、再杀掉、再重启……： 

## 4，超过整个节点容量的内存

内存请求和限制是与容器关联的，但将 Pod 视为具有内存请求和限制，也是很有用的。 Pod 的内存请求是 Pod 中所有容器的内存请求之和。 同理，Pod 的内存限制是 Pod 中所有容器的内存限制之和。

Pod 的调度基于请求。只有当节点拥有足够满足 Pod 内存请求的内存时，才会将 Pod 调度至节点上运行。

在本练习中，你将创建一个 Pod，其内存请求超过了你集群中的任意一个节点所拥有的内存。 这是该 Pod 的配置文件，其拥有一个请求 1000 GiB 内存的容器，这应该超过了你集群中任何节点的容量。

 Pod 处于 PENDING 状态。 这意味着，该 Pod 没有被调度至任何节点上运行，并且它会无限期的保持该状态 

## 5，内存单位

 内存资源的基本单位是字节（byte）。你可以使用这些后缀之一，将内存表示为 纯整数或定点整数：E、P、T、G、M、K、Ei、Pi、Ti、Gi、Mi、Ki。 例如，下面是一些近似相同的值： 

```
128974848, 129e6, 129M , 123Mi
```

## 6，如果你没有指定内存限制

如果你没有为一个容器指定内存限制，则自动遵循以下情况之一：

- 容器可无限制地使用内存。容器可以使用其所在节点所有的可用内存， 进而可能导致该节点调用 OOM Killer。 此外，如果发生 OOM Kill，没有资源限制的容器将被杀掉的可行性更大。
- 运行的容器所在命名空间有默认的内存限制，那么该容器会被自动分配默认限制。 集群管理员可用使用 [LimitRange](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#limitrange-v1-core) 来指定默认的内存限制。

## 7，内存请求和限制的目的

通过为集群中运行的容器配置内存请求和限制，你可以有效利用集群节点上可用的内存资源。 通过将 Pod 的内存请求保持在较低水平，你可以更好地安排 Pod 调度。 通过让内存限制大于内存请求，你可以完成两件事：

- Pod 可以进行一些突发活动，从而更好的利用可用内存。
- Pod 在突发活动期间，可使用的内存被限制为合理的数量。

## 清理

删除命名空间。下面的命令会删除你根据这个任务创建的所有 Pod：

```shell
kubectl delete namespace mem-example
```



参考链接：https://kubernetes.io/zh/docs/tasks/configure-pod-container/assign-memory-resource/

