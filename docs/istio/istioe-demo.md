## 部署 demo 应用

### Demo 应用概览

Demo 应用是一个电商网站，基于 Istio 社区的官方样例 [bookinfo](https://raw.githubusercontent.com/istio/istio/release-1.12/samples/bookinfo/platform/kube/bookinfo.yaml) 改造，由 6 个服务组成：

- **frontend**：网站前端，调用 user、product、cart、order 服务。
- **product**：商品服务，提供商品信息。product 包含两个版本，版本一没有顶部广告 banner；版本二有顶部广告 banner。
- **user**：用户登录服务，提供登录功能。
- **cart**：购物车服务，提供添加、查看购物车功能，调用库存服务提供库存告警功能，需要登录才可以下单。
- **order**：订单结算服务，登录后点击 checkout 后可发起结算，结算时需要调用 stock 库存服务查询库存情况，库存不足会下单失败。order 包含两个版本，版本一无积分抵扣运费的功能，版本二有积分抵扣运费的功能。
- **stock**：库存服务，为 order 购物车服务的库存告警功能和 order 订单结算服务的库存查询提供库存信息。

#### Demo 应用架构

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108354.svg)

#### Demo 应用首页

 ![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108405.png) 

### 安装 Demo 应用

您可以在 [TCM Demo 仓库](https://github.com/Tencent-Cloud-Mesh/mesh-demo) 中获取 Demo 应用，由于 TCM 的 sidecar 自动注入需要标记 Istio 版本，您需要选择与您 Istio 版本一致的分支，或直接修改 master 分支，路径 mesh-demo/yamls/step01-apps-zone-a.yaml 中 base namespace 中的版本 label：

下载tcm-demo

```
[root@VM-249-47-tlinux ~]# git clone git@github.com:Tencent-Cloud-Mesh/mesh-demo.git
```

修改labels和服务网格版本保持一致

```
apiVersion: v1
kind: Namespace
metadata:
  name: base
  labels:
    istio.io/rev: 1-10-3
spec:
  finalizers:
    - kubernetes
```

例如，您的 istio 版本为 1.8.1，则需要将 `istio.io/rev: 1-10-3``istio.io/rev: 1-8-1`

使用如下命令可快速部署 Demo 应用：

```
kubectl apply -f yamls/step01-apps-zone-a.yaml
```

## 配置公网访问

### 1. 创建 Gateway 配置监听规则

首先需要创建 Gateway 资源，配置 istio-ingressgateway 的监听器规则，端口为 80，协议为 http。用户只需要配置 Gateway 规则，TCM 后台会自动实现 istio-ingressgateway 相关的 pod、service 和绑定的负载均衡器 CLB 的配置同步。

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gw
  namespace: base
spec:
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - '*'
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
```

### 2. 配置路由规则

监听器规则配置完成后，还需要通过 Virtual Service 资源配置路由规则，将来自 istio-ingressgateway 的流量路由至 frontend 服务。

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-vs
  namespace: base
spec:
  hosts:
    - '*'
  gateways:
    - base/frontend-gw
  http:
    - route:
        - destination:
            host: frontend.base.svc.cluster.local
```

 配置完成后，通过 istio-ingressgateway 的公网 IP 地址即可访问到 Demo 网站，当前部署的网站的结构如下图所示： ![1659511192806](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108451.png)

 单击链接访问网站后，可登录（1-5 均可登录，其中 1-3 为会员，4-5 为非会员）、添加购物车，下单，以产生调用完所有部署的服务的请求，网站界面右下角的悬浮窗展示了前端服务当前调用服务的名称、地域、版本、pod name 信息。悬浮窗信息展示如下图所示： 

![1659511448198](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108492.png)

账号直接输入1，或者2-5 任一数字就可以



 网格拓扑如下图所示： 

> > 需要客户使用prometheus监控才能看到

![1659515083747](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108530.png)

 链路追踪 :

> > 链路追踪需要将应用性能观测服务 APM 作为网格调用追踪消费端才能正常使用，您可以在网格基本信息-调用追踪-消费端设置中启用应用性能观测服务 APM 服务。 



## 多版本路由

### 1，操作场景

网站计划推出会员积分抵扣运费的活动以发展会员。电商网站策划了会员积分抵扣订单金额的新功能，当前部署的 order 服务由 v1 deployment 提供，没有运费抵扣的功能；网站新开发了 order 服务的 v2 版本，有积分抵扣运费的功能。网站希望可以于请求的 header 中是否会员的 cookie 信息进行路由，会员路由至 order v2（有运费抵扣功能），非会员路由至 order v1（无运费抵扣功能）。
服务多版本路由概览图如下所示![1659514009546](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108570.png)

 部署 order v2 至集群 :

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-v2
  namespace: base
  labels:
    app: order
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
      version: v2
  template:
    metadata:
      labels:
        app: order
        version: v2
    spec:
      containers:
        - name: order
          image: ccr.ccs.tencentyun.com/zhulei/testorder2:v1
          imagePullPolicy: Always
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: REGION
              value: "guangzhou-zoneA"
          ports:
            - containerPort: 7000
              protocol: TCP
```

 部署完成后，由于还未配置路由规则，此时访问 order 服务的流量会被随机路由至 v1 版本或 v2 版本。如下图所示： 

![1659514816598](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108609.png)

配置基于流量特征内容的路由规则前先需要通过 DestinationRule 定义 order 服务的两个版本。如下图所示：

定义 order 服务的版本如下图所示：

![1659515250606](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108648.png)

![1659515281387](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108685.png)

![1659515357769](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108724.png)

### 2，操作步骤

#### 创建DestinationRule

或者  YAML 文件至主集群完成 DestinationRule 的创建： 

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: order
  namespace: base
spec:
  host: order
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  exportTo:
    - '*'
```

#### 配置VirtualService 

两个版本定义完成后，通过 VirtualService 定义按流量特征进行路由，请求的 header-cookie 中 vip=false 时路由至 order 服务的 v1 subset，vip=true 时路由至 order 服务的 v2 subset。即会员的请求路由至 order v2，非会员的请求路由至 order v1。提交以下 yaml 资源至主集群，即可完成上述配置。 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-vs
  namespace: base
spec:
  hosts:
    - order.base.svc.cluster.local
  http:
    - match:
        - headers:
            cookie:
              exact: vip=false
      route:
        - destination:
            host: order.base.svc.cluster.local
            subset: v1
    - match:
        - headers:
            cookie:
              exact: vip=true
      route:
        - destination:
            host: order.base.svc.cluster.local
            subset: v2
```

 配置完成后，可登录会员帐号（ID：1-3）加购和买单，发现有运费抵扣功能，流量被路由到了 order v2 版本；登录非会员账号（ID：4-5）加购和买单，发现无运费抵扣功能，根据 header 中的 VIP 字段信息，请求被路由到了最初部署的 order v1 版本。版本信息也可通过左下角悬浮窗中的信息观察。
会员用户请求被路由到 v2 版本如下图所示： 

![1659515926869](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108759.png)

![1659516004008](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108793.png)



## 灰度发布

### 1，操作场景

随着网站流量的增加，网站开始有了广告投放的需求，广告投放需要在商品页面增加广告位。网站的开发人员新开发了 product 服务的 v2 版本，以 product v2 的 deployment 的形式提供，并希望对 product-v2 版本做灰度发布。
灰度发布概览图如下所示：![1659517102643](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108829.png)

### 2，操作步骤

#### 部署product v2 版本

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-v2
  namespace: base
  labels:
    app: product
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product
      version: v2
  template:
    metadata:
      labels:
        app: product
        version: v2
    spec:
      containers:
        - name: product
          image: ccr.ccs.tencentyun.com/zhulei/testproduct2:v1
          imagePullPolicy: Always
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: REGION
              value: "guangzhou-zoneA"
          ports:
            - containerPort: 7000
```

####  创建DR和VS

通过 DR 定义服务版本 + 通过 VS 定义权重路由来完成灰度发布的第一步，将部分流量（50%）路由至 product v2 subset 以验证新版本，剩余部分（50%）的流量仍然路由至 product v1 版本。将以下 YAML 文件提交至主集群即可完成以上设定。 

![1659517592217](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108871.png)

![1659517642602](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108910.png)

![1659517666483](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108957.png)

 product v2 版本验证通过后，即可修改关联 product 的 VirtualService 中路由规则目的端的权重，设置访问 product 服务的所有流量（100%）至 v2 版本，设置完成后可刷新商品列表页面验证。基于 virtual Service 更改权重如下图所示： ![1659517797794](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108019.png)

## 故障注入测试

 电商网站业务团队需要模拟访问库存服务存在延迟故障时网站系统的行为，以测试服务弹性，网站用户的优化访问体验。
stock 服务 fixed delay 7s 如下图所示： 

![1659519007980](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108066.png)

### 操作步骤

通过配置绑定 stock 服务的 VritualService，设置访问 stock 服务的故障注入策略：100%的请求会有 7 秒的固定延迟。

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: stock-vs
  namespace: base
spec:
  hosts:
    - stock.base.svc.cluster.local
  http:
    - route:
        - destination:
            host: stock.base.svc.cluster.local
      fault:
        delay:
          fixedDelay: 7000ms
          percentage:
            value: 100
```



 配置完成后，在 Demo 网站页面单击 “ADD TO CART” 加入购物车或者单击 “YOUR CART” 调用购物车服务，购物车服务会调用 stock 服务查询库存，访问 stock 服务有 7 秒的固定延迟故障注入策略，以及在购物车页面单击“CHECKOUT”发起结算会调用 order 服务，order 服务会调用 stock 服务查询库存，访问 stock 服务有 7 秒的固定延迟故障注入策略。此时页面处于加载中的状态会持续 7 秒一直等待故障结束，造成网站用户的不流畅浏览体验。
cart 服务调用 stock 服务的等待状态如下图所示： 

![1659519211132](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108100.png)





## 服务超时配置

 order 服务 timeout 3s 如下图所示： 

![1659519295490](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108135.png)

 通过对 stock 服务配置故障注入，发现由于故障会导致网站用户的请求一直处于等待状态，为优化网站用户的浏览体验，需要为服务配置 timeout。 

 应用以下 VS，为 order 服务配置 3 秒的超时时间，cart 服务不设置超时时间作为参照对比。 

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-vs
  namespace: base
spec:
  hosts:
    - order.base.svc.cluster.local
  http:
    - match:
        - headers:
            cookie:
              exact: vip=false
      route:
        - destination:
            host: order.base.svc.cluster.local
            subset: v1
      timeout: 3000ms
    - match:
        - headers:
            cookie:
              exact: vip=true
      route:
        - destination:
            host: order.base.svc.cluster.local
            subset: v2
      timeout: 3000ms
```

 配置完成后，选择商品加入购物车，此时访问 cart 服务会有 7 秒的故障注入访问延时，且没有 timeout 处理，点击“CHECKOUT”发起结算调用 order 服务，此时虽然访问 order 服务也会有 7 秒的故障注入访问延时，但是有 3 秒的 timeout 超时处理，在调用 order 服务 3 秒内没有反应会做超时处理。
cart 服务调用 order 服务 timeout 显示如下图所示： 

![1659519546951](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108175.png)

超时配置已完成，对于服务的故障注入测试已完成，可删除关联 stock 服务的 VirtualService 资源以解除对 stock 服务配置的故障注入策略。

删除 stock 服务关联 Virtual Service 操作如下图所示：

![1659519634038](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108216.png)

## 会话保持

 购物车服务由多个 pod 副本运行，需要会话保持功能，以保证同一用户请求被路由至同一个 pod，保证同一用户的购物车信息不会丢失。
会话保持如下图所示： 

![1659520453966](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108254.png)

### 操作步骤

会话保持功能可通过设置 cart 服务 DestinationRule 的负载均衡策略实现，以请求中 header 中的 UserID 做一致性 hash 负载均衡。

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: cart
  namespace: base
spec:
  host: cart
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: UserID
  exportTo:
    - '*'
```



 配置完成后，可在登录状态多次点击 “Your Cart” 或点击 “ADD TO CART” 调用 cart 服务验证会话保持功能，同一用户的多次请求会被路由至同一个 pod，左下角悬浮窗可查看提供 cart 服务的 pod name。同一用户多次请求的 pod name 不会变化。来自同一用户的多次请求被负载均衡至相同 pod 如下图所示： 

![1659521434416](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108290.png)



## 使用连接池限制并发

### 操作场景

随着电商网站业务规模的增大，对网站的访问请求并发量开始增加，网站业务人员计划限制服务最大并发数，保证服务运行健壮性。

### 操作步骤

为模拟“高并发”请求场景，首先通过提交以下 YAML 部署 client 服务（10 pods），模拟对 user 服务的高并发请求。



```
apiVersion: v1
kind: Namespace
metadata:
  name: test
  labels:
    istio-injection: enabled
spec:
  finalizers:
    - kubernetes
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: test
  labels:
    app: client
spec:
  replicas: 10
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
        - name: client
          image: ccr.ccs.tencentyun.com/zhulei/testclient:v1
          imagePullPolicy: Always
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: REGION
              value: "guangzhou-zoneA"
          ports:
            - containerPort: 7000
              protocol: TCP
---

apiVersion: v1
kind: Service
metadata:
  name: client
  namespace: test
  labels:
    app: client
spec:
  ports:
    - name: http
      port: 7000
      protocol: TCP
  selector:
    app: client
  type: ClusterIP
```

 此时对于访问 user 服务没有最大并发数限制，所有请求均可访问成功。通过 TKE 控制台 client deployment 查看 client pod 日志，所有的请求均返回了用户名 Kevin，证明访问请求成功。
高并发请求如下图所示： ![1659530540788](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108332.png)

 通过配置 user 服务的 Destination Rule 限制最大并发数为 1： 

 

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: user
  namespace: base
spec:
  host: user
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        http2MaxRequests: 1
        maxRequestsPerConnection: 1
  exportTo:
    - '*'
```

 此时查看 client pod 日志，部分请求开始出现异常，未返回用户名，请求失败，连接池起到了限制访问服务最大并发数的作用。
部分请求访问失败如下图所示： ![1659530666208](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108371.png)

 删除流量策略相关配置如下图所示：
连接池测试完成后，在 user 服务的详情页面删除连接池相关流量策略配置。 

![1659530746902](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042108407.png)