### Service Mesh  

Service Mesh 的中文译为 “服务网格” ，是一个用于处理服务和服务之间通信的基础设施层，它负责为构建复杂的云原生应用传递可靠的网络请求，并为服务通信实现了微服务所需的基本组件功能，例如服务发现、负载均衡、监控、流量管理、 访问控制等。在实践中，服务网格通常实现为一组和应用程序部署在一起的轻量级的网络代理，但对应用程序来说是透明的。右图， 绿色方块为应用服务，蓝色方块为 Sidecar Proxy，应用服务之间通过 Sidecar Proxy 进行通信，整个服务通信形成图中的蓝色网络连线，图中所有蓝色部分就形成一个网络，这个就是服务网格名字的由来  

![image-20220102142536312](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107153.png)

### Service Mesh特点  

Service Mesh有以下特点：
• 治理能力独立（Sidecar）
• 应用程序无感知
• 服务通信的基础设施层
• 解耦应用程序的重试/超时、监控、追踪和服务发现  

![image-20220102142655219](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107541.png)

### Istio概述  

Isito是Service Mesh的产品化落地，是目前最受欢迎的服务网格，功能丰富、成熟度高。

Linkerd是世界上第一个服务网格类的产品 

• 连接（Connect）
\- 流量管理
\- 负载均衡
\- 灰度发布
• 安全（Secure）
\- 认证
\- 鉴权
• 控制（Control）
\- 限流
\- ACL
• 观察（Observe）
\- 监控
\- 调用链  

![image-20220102142845281](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107626.png)

![image-20220102142901068](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107659.png)





### Istio版本变化 

在Istio1.5版本发生了一个重大变革，彻底推翻原有控制平面的架构，将有原有多个组件整合为单体结构
“istiod”， 同时废弃了Mixer 组件，如果你正在使用之前版本，必须了解这些变化  

![image-20220102143026421](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107697.png)

### Istio架构与组件  

Istio服务网格在逻辑上分为数据平面和控制平面。
**• 控制平面**： 使用全新的部署模式： istiod，这个组件负责处理Sidecar注入、证书分发、配置管理等功能，替
代原有组件，降低复杂度，提高易用性。
    • Pilot：策略配置组件，为Proxy提供服务发现、智能路由、错误处理等。
    • Citadel： 安全组件，提供证书生成下发、加密通信、访问控制。
    • Galley： 配置管理、验证、分发。
• **数据平面**： 由一组Proxy组成， 这些Proxy负责所有微服务网络通信，实现高效转发和策略。使用envoy实现，
envoy是一个基于C++实现的L4/L7 Proxy转发器，是Istio在数据平面唯一的组件。  

### Istio基本概念  

Istio 有 4 个配置资源，落地所有流量管理需求：
• VirtualService（虚拟服务）：实现服务请求路由规则的功能。
• DestinationRule（目标规则）：实现目标服务的负载均衡、服务发现、故障处理和故障注入的功能。
• Gateway（网关）：让服务网格内的服务，可以被全世界看到。
• ServiceEntry（服务入口） ：允许管理网格外的服务的流量  

### 部署Istio  

```
tar zxvf istio-1.8.2-linux.tar.gz
cd  istio-1.8.2
mv  bin/istioctl /usr/bin
istioctl  install
kubectl  get pods -n istio-system
kubectl  get svc -n istio-system
卸载：
istioctl manifest generate | kubectl delete -f -
```

### Sidercar注入

```
部署httpbin Web示例：
cd istio-1.8.2/samples/httpbin
# 手动注入
kubectl apply -f <(istioctl kube-inject -f httpbin-nodeport.yaml)
或者
istioctl kube-inject -f httpbin-nodeport.yaml |kubectl apply -f -
# 自动注入（给命名空间打指定标签，启用自动注入）
kubectl label namespace default istio-injection=enabled
kubectl apply -f httpbin-gateway.yaml
IngressGateway NodePort访问地址： http://81.69.167.80:80
```

![image-20220102143706176](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107734.png)

![image-20220102144036432](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107792.png)

### Istio与K8s集成流程  

![image-20220102144130551](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107832.png)







### Istio 流量管理核心资源  

**核心资源**：
• VirtualService（虚拟服务）
• DestinationRule（目标规则）
• Gateway（网关）
• ServiceEntry（服务入口）  



#### VirtualService

VirtualService（虚拟服务）：
• 定义路由规则
• 描述满足条件的请求去哪里  

![image-20220102144310931](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107875.png)

#### DestinationRule

DestinationRule（目标规则）：定义虚拟服务路由目标地址
真实地址，即子集（subset），支持多种负载均衡策略：
• 随机
• 权重
• 最小请求数  

#### Gateway

Gateway（网关）：为网格内服务对外访问入口，管理进出网格的流量，根据流入流出方向分为：
• IngressGateway：接收外部访问，并将流量转发到网格内的服务。
• EgressGateway：网格内服务访问外部应用。  

![image-20220102145508447](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107915.png)

**Gateway（网关）与Kubernetes Ingress有什么区别？**
Kubernetes Ingress与Getaway都是用于为集群内服务提供访问入口，
但Ingress主要功能比较单一，不易于Istio现有流量管理功能集成。
目前Gateway支持的功能：
• 支持L4-L7的负载均衡
• 支持HTTPS和mTLS
• 支持流量镜像、熔断等  

![image-20220102151816275](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107949.png)

#### ServiceEntry

ServiceEntry（服务入口）： 将网格外部服务添加到网格内，像网格内其他服务一样管理。  

![image-20220102151901001](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107988.png)



### Istio 流量管理案例 

#### 主流发布方案

• 蓝绿发布
• 滚动发布
• 灰度发布（金丝雀发布）
• A/B Test  

**蓝绿发布**
项目逻辑上分为AB组，在项目升级时，首先把A组从负载均衡
中摘除，进行新版本的部署。 B组仍然继续提供服务。 A组升级
完成上线， B组从负载均衡中摘除。
特点：
• 策略简单
• 升级/回滚速度快
• 用户无感知，平滑过渡
缺点：
• 需要两倍以上服务器资源
• 短时间内浪费一定资源成本
• 有问题影响范围大  

![image-20220102153923205](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107031.png)



**滚动发布**
每次只升级一个或多个服务，升级完成后加入生产环境，不断执行这个过程，直到集群中的全部旧版升级新版本。Kubernetes的默认发布策略。
特点：
• 用户无感知，平滑过渡
缺点：
• 部署周期长
• 发布策略较复杂
• 不易回滚
• 有影响范围较大  

![image-20220102153951015](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107069.png)

![image-20220102153959051](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107109.png)

**灰度发布（金丝雀发布）**
只升级部分服务，即让一部分用户继续用老版本，一部分用户，开始用新版本，如果用户对新版本没有什么意见，那么逐步扩大范围，把所有用户都迁移到新版本上面来。
特点：
• 保证整体系统稳定性
• 用户无感知，平滑过渡
缺点：
• 自动化要求高  

![image-20220102154026132](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107141.png)

**A/B Test**
灰度发布的一种方式，主要对特定用户采样后，对收集到的反馈数据做相关对比，然后根据比对结果作出决策。用来测试应
用功能表现的方法，侧重应用的可用性，受欢迎程度等， 最后决定是否升级  

![image-20220102154045168](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107204.png)

#### 部署Bookinfo微服务项目  

Bookinfo 是官方提供一个图书评测系统微服务项目示例，
分为四个微服务：  

| 服务        | 说明     | 调用服务          |
| ----------- | -------- | ----------------- |
| productpage | 主页     | reviews、 details |
| reviews     | 评论内容 | ratings           |
| details     | 详细内容 |                   |
| ratings     | 评分     |                   |

![image-20220102154221448](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/image-20220102154221448.png)



1、创建命名空间并开启自动注入  

```
kubectl  create ns bookinfo
kubectl  label ns bookinfo  istio-injection=enabled
kubectl  get ns --show-labels

```

2、部署应用YAML

```
cd istio-1.8.2/samples/bookinfo
kubectl apply -f platform/kube/bookinfo.yaml -n bookinfo
kubectl get pod -n bookinfo
```

![image-20220102154922073](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042107290.png)

3、创建Ingress网关

```
kubectl apply -f networking/bookinfo-gateway.yaml -n book
```

4、确认网关和访问地址，访问应用页面

```
kubectl get pods -n istio-system
```

访问地址： http://192.168.31.62:31928/productpage  

