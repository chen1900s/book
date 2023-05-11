## 操作场景

实例（Pod）自动扩缩容功能（Horizontal Pod Autoscaler，HPA）可以根据目标实例 CPU 利用率的平均值等指标自动扩展、缩减服务的 Pod 数量

## 工作原理

HPA 后台组件会每隔15秒向腾讯云云监控拉取容器和 Pod 的监控指标，然后根据当前指标数据、当前副本数和该指标目标值进行计算，计算所得结果作为服务的期望副本数。当期望副本数与当前副本数不一致时，HPA 会触发 Deployment 进行 Pod 副本数量调整，从而达到自动伸缩的目的。
以 CPU 利用率为例，假设当前有2个实例， 平均 CPU 利用率（当前指标数据）为90%，自动伸缩设置的目标 CPU 为60%， 则自动调整实例数量为：90% × 2 / 60% = 3个。

```
注意：
如果用户设置了多个弹性伸缩指标，HPA 会依据各个指标，分别计算出目标副本数，取最大值进行扩缩容操作。
```

Pod 水平自动扩缩原理：https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/

## 注意事项

- 当指标类型选择为 **CPU 利用率（占 Request）**时，必须为容器设置 CPU Request。
- 策略指标目标设置合理，例如设置70%给容器和应用，预留30%的余量。
- 保持 Pod 和 Node 健康（避免 Pod 频繁重建）。
- 保证用户请求的负载均衡稳定运行。
- HPA 在计算目标副本数时会有一个10%的波动因子。如果在波动范围内，HPA 并不会调整副本数目。
- 如果服务对应的 Deployment.spec.replicas 值为0，HPA 将不起作用。
- 如果对单个 Deployment 同时绑定多个 HPA ，则创建的 HPA 会同时生效，会造成工作负载的副本重复扩缩。

## 演练

### 运行 php-apache 服务器并暴露服务

为了演示 Horizontal Pod Autoscaler，我们将使用一个基于 php-apache 镜像的 定制 Docker 镜像。Dockerfile 内容如下：

```
FROM php:5-apache
COPY index.php /var/www/html/index.php
RUN chmod a+rx index.php
```

该文件定义了一个 index.php 页面来执行一些 CPU 密集型计算：

```
<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) {
    $x += sqrt($x);
  }
  echo "OK!";
?>
```

```
#镜像构建
docker build -t tcr-chen.tencentcloudcr.com/docker/hpa-example  .
```

使用下面的配置启动一个 Deployment 来运行这个镜像并暴露一个服务：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: tcr-chen.tencentcloudcr.com/docker/hpa-example:latest
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m

---

apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

### 创建 Horizontal Pod Autoscaler

现在，php-apache 服务器已经运行，我们将通过 [kubectl autoscale](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#autoscale) 命令创建 Horizontal Pod Autoscaler。 以下命令将创建一个 Horizontal Pod Autoscaler 用于控制我们上一步骤中创建的 Deployment，使 Pod 的副本数量维持在 1 到 10 之间。 大致来说，HPA 将（通过 Deployment）增加或者减少 Pod 副本的数量以保持所有 Pod 的平均 CPU 利用率在 50% 左右（由于每个 Pod 请求 200 毫核的 CPU，这意味着平均 CPU 用量为 100 毫核）

```
[root@VM-0-17-tlinux ~/hpa/php-apache]# kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled

```

我们可以通过以下命令查看 Autoscaler 的状态：

```
[root@VM-0-17-tlinux ~/hpa/php-apache]# kubectl  get hpa
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          2m21s
```

请注意当前的 CPU 利用率是 0%，这是由于我们尚未发送任何请求到服务器 （`CURRENT` 列显示了相应 Deployment 所控制的所有 Pod 的平均 CPU 利用率）

### 增加负载

我们将看到 Autoscaler 如何对增加负载作出反应。 我们将启动一个容器，并通过一个循环向 php-apache 服务器发送无限的查询请求 （请在另一个终端中运行以下命令）

```
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

一分钟时间左右之后，通过以下命令，我们可以看到 CPU 负载升高了：

```
[root@VM-0-17-tlinux ~/hpa/php-apache]# kubectl  get hpa -w
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   249%/50%   1         10        5          8m4s
```

这时，由于请求增多，CPU 利用率已经升至请求值的 249%。 可以看到，Deployment 的副本数量已经增长到了 5：

```
[root@VM-0-17-tlinux ~/hpa/php-apache]# kubectl  get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   16%/50%   1         10        8          19m
```

**说明：** 有时最终副本的数量可能需要几分钟才能稳定下来。由于环境的差异， 不同环境中最终的副本数量可能与本示例中的数量不同

### 停止负载

我们将通过停止负载来结束我们的示例。

在我们创建 busybox 容器的终端中，输入`<Ctrl> + C` 来终止负载的产生。

然后我们可以再次检查负载状态（等待几分钟时间）：

![image-20211012164607241](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145436.png)



### 根据不同业务场景调节 HPA 扩缩容灵敏度

#### 使用场景

在 K8S 1.18之前，HPA 扩缩容无法调整灵敏度：

- 对于缩容，由 `kube-controller-manager` 的 `--horizontal-pod-autoscaler-downscale-stabilization-window` 参数控制缩容时间窗口，默认为5分钟，即负载减小后至少需要5分钟后才开始缩容。
- 对于扩容，由 hpa controller 固定的算法、硬编码的常量因子来控制扩容速度，无法自定义。

K8S 设计逻辑导致用户无法自定义 HPA 的扩缩容灵敏度，不同的业务场景对于扩缩容灵敏度要求并不一样，例如：

- 对于有流量突发的关键业务，需要快速扩容（防止流量瓶颈）、缓慢缩容（防止另一个流量高峰）。
- 对于需要处理大量数据的离线业务，在处理高峰期时应尽快扩容以减少处理时间，高峰期后应尽快缩容以节约成本。
- 对于处理常规数据、网络流量的业务，需要以正常速度扩大和缩小规模，以减少抖动。

K8S 1.18在 HPA Spec 下新增了 `behavior` 字段，该字段提供 `scaleUp` 和 `scaleDown` 两个字段分别控制扩容和缩容行为

##### 示例1：快速扩容

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  labels:
    qcloud-app: hpa-php-apache
  name: hpa-php-apache
  namespace: default
spec:
  maxReplicas: 1000
  metrics:
  - pods:
      metricName: k8s_pod_rate_cpu_core_used_limit
      targetAverageValue: "80"
    type: Pods
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  behavior:
    scaleUp:
      policies:
      - type: Percent
        value: 300
        periodSeconds: 15

```



ke控制台通过v2beta1的接口去获取的，因此展示的是v2beta1版本；kubectl 命令默认用的是v1的版本去获取的这个资源对象信息，所以展示的v1
解决：执行这个命令：
kubectl edit HorizontalPodAutoscaler.v2beta2.autoscaling/<hpa的名称> -n <命名空间>



kubectl get HorizontalPodAutoscaler.v2beta2.autoscaling/<hpa的名称> -n <命名空间> //查看

```
kubectl  get   HorizontalPodAutoscaler.v2beta2.autoscaling/php-apache  -o yaml

#######
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  behavior:
    scaleDown:
      policies:
      - periodSeconds: 15
        type: Percent
        value: 100
      selectPolicy: Max
      stabilizationWindowSeconds: null
    scaleUp:
      policies:
      - periodSeconds: 15
        type: Percent
        value: 300
      selectPolicy: Max
      stabilizationWindowSeconds: 0
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
```





![1656151060456](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042145484.png)