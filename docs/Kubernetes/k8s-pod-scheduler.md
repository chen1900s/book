# Pod 调度策略

## 概述

**节点亲和性**：设置**节点亲和性**，通过Worker节点的Label标签进行设置。

节点调度支持硬约束和软约束（Required/Preferred），以及丰富的匹配表达式（In, NotIn, Exists, DoesNotExist. Gt, and Lt）：

- **必须满足**，即硬约束，一定要满足，对应**requiredDuringSchedulingIgnoredDuringExecution**，效果与`NodeSelector`相同。本例中Pod只能调度到具有对应标签的Worker节点。您可以定义多条硬约束规则，但只需满足其中一条。
- **尽量满足**，即软约束，不一定满足，对应**preferredDuringSchedulingIgnoredDuringExecution**。本例中，调度会尽量不调度Pod到具有对应标签的Node节点。您还可为软约束规则设定权重，具体调度时，若存在多个符合条件的节点，权重最大的节点会被优先调度。您可定义多条软约束规则，但必须满足全部约束，才会进行调度。

**POD亲和性**：决定应用的Pod可以和哪些Pod部署在同一拓扑域。例如，对于相互通信的服务，可通过应用亲和性调度，将其部署到同一拓扑域（如同一个主机）中，减少它们之间的网络延迟。

根据节点上运行的Pod的标签（Label）来进行调度，支持硬约束和软约束，匹配的表达式有：`In, NotIn, Exists, DoesNotExist`。

- 必须满足

  ，即硬约束，一定要满足，对应

  requiredDuringSchedulingIgnoredDuringExecution

  ，Pod的亲和性调度必须要满足后续定义的约束条件。

  - **命名空间**：该策略是依据Pod的Label进行调度，所以会受到命名空间的约束。
  - **拓扑域**：即TopologyKey，指定调度时作用域，这是通过Node节点的标签来实现的，例如指定为`kubernetes.io/hostname`，那就是以Node节点为区分范围；如果指定为`beta.kubernetes.io/os`，则以Node节点的操作系统类型来区分。
  - **选择器**：单击选择器右侧的加号按钮，您可添加多条硬约束规则。
  - **查看应用列表**：单击**应用列表**，弹出对话框，您可在此查看各命名空间下的应用，并可将应用的标签导入到亲和性配置页面。
  - 硬约束条件：设置已有应用的标签、操作符和标签值。本例中，表示将待创建的应用调度到该主机上，该主机运行的已有应用具有`app:nginx`标签。

- 尽量满足

  ，即软约束，不一定满足，对应

  preferredDuringSchedulingIgnoredDuringExecution

  。Pod的亲和性调度会尽量满足后续定义的约束条件。对于软约束规则，您可配置每条规则的权重，其他配置规则与硬约束规则相同。

  > **说明** **权重**：设置一条软约束规则的权重，介于1~100，通过算法计算满足软约束规则的节点的权重，将Pod调度到权重最大的节点上。

**POD反亲和性**：决定应用的Pod不与哪些Pod部署在同一拓扑域。应用非亲和性调度的场景包括：

- 将一个服务的Pod分散部署到不同的拓扑域（如不同主机）中，提高服务本身的稳定性。
- 给予Pod一个节点的独占访问权限来保证资源隔离，保证不会有其它Pod来分享节点资源。
- 把可能会相互影响的服务的Pod分散在不同的主机上。

> **说明** 应用非亲和性调度的设置方式与亲和性调度相同，但是相同的调度规则代表的意思不同，请根据使用场景进行选择。

**调度容忍**：容忍被应用于Pod，允许这个Pod被调度到相对应的污点上。



将 Pod 打散调度到不同地方，可避免因软硬件故障、光纤故障、断电或自然灾害等因素导致服务不可用，以实现服务的高可用部署。

Kubernetes 支持两种方式将 Pod 打散调度:

- Pod 反亲和 (Pod Anti-Affinity)
- Pod 拓扑分布约束 (Pod Topology Spread Constraints)

## 使用 podAntiAffinity

**将 Pod 强制打散调度到不同节点上(强反亲和)，以避免单点故障**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app: nginx
      containers:
      - name: nginx
        image: nginx
```

- `labelSelector.matchLabels` 替换成选中 Pod 实际使用的 label。

- `topologyKey`: 节点的某个 label 的 key，能代表节点所处拓扑域，可以用 [Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/#failure-domainbetakubernetesioregion)，常用的是 `kubernetes.io/hostname` (节点维度)、`topology.kubernetes.io/zone` (可用区/机房 维度)。也可以自行手动为节点打上自定义的 label 来定义拓扑域，比如 `rack` (机架维度)、`machine` (物理机维度)、`switch` (交换机维度)。

- 若不希望用强制，可以使用弱反亲和，让 Pod 尽量调度到不同节点:

  ```yaml
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - podAffinityTerm:
        topologyKey: kubernetes.io/hostname
      weight: 100
  ```

**将 Pod 强制打散调度到不同可用区(机房)，以实现跨机房容灾**:

将 `kubernetes.io/hostname` 换成 `topology.kubernetes.io/zone`，其余同上。

## 使用 topologySpreadConstraints

**将 Pod 最大程度上均匀的打散调度到各个节点上**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-topologyspreadconstraints
  namespace: podaffinity
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-topologyspreadconstraints
  template:
    metadata:
      labels:
        app: nginx-topologyspreadconstraints
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: nginx-topologyspreadconstraints
      containers:
      - name: nginx
        image: nginx
```

- `topologyKey`: 与 podAntiAffinity 中配置类似。
- `labelSelector`: 与 podAntiAffinity 中配置类似，只是这里可以支持选中多组 pod 的 label。
- `maxSkew`: 必须是大于零的整数，表示能容忍不同拓扑域中 Pod 数量差异的最大值。这里的 1 意味着只允许相差 1 个 Pod。
- `whenUnsatisfiable`: 指示不满足条件时如何处理。`DoNotSchedule` 不调度 (保持 Pending)，类似强反亲和；`ScheduleAnyway` 表示要调度，类似弱反亲和；

以上配置连起来解释: 将所有 nginx 的 Pod 严格均匀打散调度到不同节点上，不同节点上 nginx 的副本数量最多只能相差 1 个，如果有节点因其它因素无法调度更多的 Pod (比如资源不足)，那么就让剩余的 nginx 副本 Pending。

所以，如果要在所有节点中严格打散，通常不太可取，可以加下 nodeAffinity，只在部分资源充足的节点严格打散:

```yaml
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: io
                operator: In
                values:
                - high
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
        - matchLabels:
            app: nginx
```

或者类似弱反亲和， **将 Pod 尽量均匀的打散调度到各个节点上，不强制** (DoNotSchedule 改为 ScheduleAnyway):

```yaml
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx
```

如果集群节点支持跨可用区，也可以 **将 Pod 尽量均匀的打散调度到各个可用区** 以实现更高级别的高可用 (topologyKey 改为 `topology.kubernetes.io/zone`):

```yaml
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx
```

更进一步地，可以 **将 Pod 尽量均匀的打散调度到各个可用区的同时，在可用区内部各节点也尽量打散**:

```yaml
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
        - matchLabels:
            app: nginx
```

## 小结

从示例能明显看出，`topologySpreadConstraints` 比 `podAntiAffinity` 功能更强，提供了提供更精细的调度控制，我们可以理解成 `topologySpreadConstraints` 是 `podAntiAffinity` 的升级版。`topologySpreadConstraints` 特性在 K8S v1.18 默认启用，所以建议 v1.18 及其以上的集群使用 `topologySpreadConstraints` 来打散 Pod 的分布以提高服务可用性。

## 参考资料

- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/)

文档链接：https://imroc.cc/k8s/ha/pod-split-up-scheduling/

## 尽量调度到不同节点

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: k8s-app
            operator: In
            values:
            - kube-dns
      topologyKey: kubernetes.io/hostname
```

## 必须调度到不同节点

```yaml
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```

## 只调度到有指定 label 的节点

```yaml
affinity:
 nodeAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
     nodeSelectorTerms:
       matchExpressions:
       - key: ingress
         operator: In
         values:
         - true
```