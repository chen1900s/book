# ConfigMap 管理

## 基本概念

Kubernetes原生提供的 ConfigMap(配置集)，可以将配置数据，与业务镜像解耦分离， 以使容器化的应用程序具有可移植性 ，ConfigMap 包含配置数据，以供pod运行时使用，其核心数据结构，可以理解为一个 key-value 映射

> 说明：
> 配置集的key，value 均为string 类型。其中，key创建时有语法要求。

根据业务需要，业务Pod可以引用 ConfigMap 对象，作为容器的环境变量；或者以数据卷方式（后续表现为一个文件），挂载到 Pod 内部，作为配置文件使用。

## 使用场景

### 使用ConfigMap数据定义容器环境变

configmap作为环境变量使用时，Pod 可以挂载一个 ConfigMap key，也可以挂载多个 ConfigMap key:

> 说明：
> 此时 ConfigMap 的 key 为环境变量名称，对应的 Value 为环境变量值。



### 以数据卷方式挂载到Pod

同理，作为数据卷使用时，Pod 可以挂载多个 ConfigMap key 也可以挂载一个ConfigMap key

> 说明：
>
> 1. 此时 ConfigMap Key 映射为容器内部的配置文件名，对应的 ConfigMap Value 表现为配置文件内容。
>
> 2. 文件覆盖：
> 数据卷形式挂载，会覆盖对应挂载目录，使用时需注意路径信息。
>
> 3. 实时更新：
> 本地小规模集群测试发现，ConfigMap 对象更新后，Pod 内部，ConfigMap 挂载对象几乎实时更新。
> 部分业务应用如果支持热加载，可以考虑使用该特性 reload 进程。社区开源的 nginx-ingress-controller，就包含类似机制（cm 对象更新，nginx 进程 reload）。
>
> kubelet 在每次周期性同步时都会检查已挂载的 ConfigMap 是否是最新的。 但是，它使用其本地的基于 TTL 的缓存来获取 ConfigMap 的当前值。 因此，从更新 ConfigMap 到将新键映射到 Pod 的总延迟可能等于 kubelet 同步周期 （默认 1 分钟） + ConfigMap 在 kubelet 中缓存的 TTL（默认 1 分钟）。 
>
> 4. Subpath volume：
>     业务容器，如果使用ConfigMap作为subPath 卷，将不会感知到ConfigMap 更新。
>     详情请参考：https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath
>
>   同理secret也是一样





### 使用单个 ConfigMap 中的数据定义容器环境变量

1，在 ConfigMap 中将环境变量定义为键值对:

```
kubectl create configmap special-config --from-literal=special.how=very -n m-configmap 
```

2，将 ConfigMap 中定义的 `special.how` 值分配给 Pod 规范中的 `SPECIAL_LEVEL_KEY` 环境变量。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    description: 将 ConfigMap 中定义的 special.how 值分配给 Pod 规范中的 SPECIAL_LEVEL_KEY 环境变量
  labels:
    k8s-app: configmap-env-key
    qcloud-app: configmap-env-key
  name: configmap-env-key
  namespace: m-configmap
spec:
  selector:
    matchLabels:
      k8s-app: configmap-env-key
      qcloud-app: configmap-env-key
  template:
    metadata:
      labels:
        k8s-app: configmap-env-key
        qcloud-app: configmap-env-key
    spec:
      containers:
      - env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              key: special.how
              name: special-config
              optional: false
        image: ccr.ccs.tencentyun.com/v_cjweichen/centos:latest
        imagePullPolicy: IfNotPresent
        name: centos
```

![image-20210815220843445](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042139971.png)

### 使用来自多个 ConfigMap 的数据定义容器环境变量

 使用 ConfigMap 作为 [subPath](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes#using-subpath) 卷挂载的容器将不会收到 ConfigMap 的更新。











### ConfigMap或Secret挂载到特定目录的特定路径

有时候挂载希望将`setenv.sh`这样的一个初始化配置环境变量的脚本挂载到tomcat的bin目录: `/usr/local/tomcat/bin`下.

如果不使用subpath, 直接将该ConfigMap 挂载到`/usr/local/tomcat/bin`目录下, **那么该目录下已有的文件全部被覆盖.**

所以正确的做法是使用`Subpath`进行挂载:

> >特别注意`mountPath`和`subPath`的写法, 最后的path要保持一致.
> >
> >如`mountPath`是: /usr/local/tomcat/bin/setenv.sh`; `subPath`是: `setenv.sh`.
> >
> >```bash
> >mountPath`不要漏写为: `/usr/local/tomcat/bin/
> >```

```bash
        volumeMounts:
        - mountPath: /usr/local/tomcat/bin/setenv.sh
          subPath: setenv.sh
          #name: config
          #readOnly: true
```

![1664015882747](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042139021.png)

![1664015850024](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042139064.png)