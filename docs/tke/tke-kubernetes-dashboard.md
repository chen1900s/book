---
title: 部署kubernetes-dashboard服务
abbrlink: 1814e34b
date: 2022-10-23 13:55:09
tags:
  - TKE
  - Kubernetes
categories: TKE
keywords:
  - Kubernetes
  - TKE
  - Kubernetes-dashboard
description: 在TKE集群部署kubernetes-dashboard
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209022219214.jpeg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210231932160.jpeg
updated: 2022-10-23 13:55:09
---

## 在TKE集群部署kubernetes-dashboard服务

[具体参考](https://github.com/kubernetes/dashboard)

> 基于已经创建好的Kubernetes集群进行部署Kubernetes-dashboard

### 下载部署yaml文件

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

### 下载镜像

由于镜像是国外镜像，国内集群部署有可能会拉取镜像失败，可以先手动拉取下来，然后修改对应的yaml文件

```bash
docker pull kubernetesui/dashboard:v2.0.4
docker pull kubernetesui/metrics-scraper:v1.0.4

docker tag 46d0a29c3f61 ccr.ccs.tencentyun.com/chenjingwei/dashboard:v2.0.4  #重新打tag, 对应到自己账号下个人镜像参考，我这里上传到我自己的镜像仓库，如果有需要 也可以从这个地址下载
docker tag 86262685d9ab  ccr.ccs.tencentyun.com/chenjingwei/metrics-scraper:v1.0.4

docker push ccr.ccs.tencentyun.com/chenjingwei/dashboard:v2.0.4
docker push ccr.ccs.tencentyun.com/chenjingwei/metrics-scraper:v1.0.4
```

### 访问方式配置（可选）

对应的service 默认是clusterIP方式，如果需要CLB去访问 或者nodePort方式去访问，可以修改对应的yaml文件，也可以部署完后修改

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort   #添加这个 service类型  如果需要是CLB类型 则修改成  LoadBalancer
```

### 部署kubernetes-dashboard

```bash
kubectl apply -f recommended.yaml
```

查看是否部署成功

```bash
[root@VM-249-41-tlinux ~]# kubectl  -n kubernetes-dashboard get pod,svc
NAME                                            READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-7748f84fc-8ldkn   1/1     Running   0          92m
pod/kubernetes-dashboard-6fbb5497cf-qscnk       2/2     Running   1          29m

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   172.16.253.139   <none>        8000/TCP        92m
service/kubernetes-dashboard        NodePort    172.16.252.218   <none>        443:31778/TCP   92m
```

### 访问kubernetes-dashboard

由于上面service使用的是NodePort类型，可以通过nodeIP+NodePort端口去访问，kubernetes-dashboard后端服务是https协议的，则需要通过https://节点IP:NodePort

![image-20221018002131961](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180021047.png)

验证方式选择token

### TKE集群获取Token认证方式

```
# 创建serviceaccount
kubectl create serviceaccount dashboard-serviceaccount -n kubernetes-dashboard
# 创建clusterrolebinding
kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-serviceaccount
```

获取token

```
[root@VM-249-41-tlinux ~]# kubectl get secret -n kubernetes-dashboard | grep dashboard-serviceaccount-token
dashboard-serviceaccount-token-pddv4   kubernetes.io/service-account-token   3      98m
[root@VM-249-41-tlinux ~]# kubectl describe secret dashboard-serviceaccount-token-pddv4 -n kubernetes-dashboard
```

![image-20221018002432661](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180024730.png)

复制token 到控制台

![image-20221018002536365](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180025468.png)



## 基于Istio的访问kubernetes-dashboard配置

参考[istio官方文档](https://preliminary.istio.io/latest/docs/tasks/traffic-management/ingress/ingress-sni-passthrough/)

> 前提条件：
>
> 1，服务网格已经关联集群
>
> 2，已经创建边缘代理网关**istio-ingressgateway**

首先需要开启 Sidecar 自动注入配置，命名空间选择kubernetes-dashboard，然后销毁重建kubernetes-dashboard的POD

```bash
#这边是基于TKE的容器服务网格1.12.5版本的，其他版本需要修改成对应版本

kubectl label namespace kubernetes-dashboard istio.io/rev=1-12-5

#如果是使用的是自建sidecar 则需要使用下面这个命令开启自动注入
#kubectl label namespace xxx istio-injection=enalbed 为某个 namespace 开启 sidecar 自动注入
```

### 新建Gateway

这里使用自定义的域名kubernetes-dashboard.chen1900s.com

![image-20221018003822597](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180038668.png)

对应yaml:

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kubernetes-dashboard-gateway
  namespace: kubernetes-dashboard
spec:
  servers:
    - port:
        number: 443
        name: HTTPS-1-m00g
        protocol: HTTPS
      hosts:
        - kubernetes-dashboard.chen1900s.com
      tls:
        mode: PASSTHROUGH
  selector:
    app: istio-ingressgateway
    istio: ingressgateway
```

### 创建Virtual Service



![image-20221018004125714](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180041789.png)

yaml文件

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kubernetes-dashboard-vs
  namespace: kubernetes-dashboard
spec:
  hosts:
    - kubernetes-dashboard.chen1900s.com
  gateways:
    - kubernetes-dashboard/kubernetes-dashboard-gateway
  tls:
    - match:
        - sniHosts:
            - kubernetes-dashboard.chen1900s.com
      route:
        - destination:
            host: kubernetes-dashboard.kubernetes-dashboard.svc.cluster.local
            port:
              number: 443
          weight: 100
```

设置完成后，配置如下本地host映射，即可通过域名 https://kubernetes-dashboard.chen1900s.com 访问，效果与之前的NodePort一致

```
[root@VM-249-41-tlinux ~]# kubectl  -n istio-system get svc  istio-ingressgateway 
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)         AGE
istio-ingressgateway   LoadBalancer   172.16.255.54   114.117.219.33   443:32430/TCP   3h47m

#114.117.219.33 kubernetes-dashboard.chen1900s.com
```

![image-20221018004522135](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180045224.png)

输入上面查询到的token 可以正常登陆

![image-20221018004606746](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180046871.png)

Istio中基于Secure Ingress的访问方式还有多种



## 基于nginx-ingress方式访问kubernetes-dashboard

Nginx Ingress Controller默认使用HTTP协议转发请求到后端业务容器。当后端业务容器为HTTPS协议时，可以通过使用注解`nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`来使得Nginx Ingress Controller使用HTTP协议转发请求到后端业务容器

kubernetes-dashboard服务正是HTTPS协议服务，则需要使用这个annotations

> 环境准备：
>
> 1，已经创建nginx-ingress-controller 实例
>
> 2，已经创建TLS证书

首先是如果不添加这个annotations 访问会报 Client sent an HTTP request to an HTTPS server.

![image-20221018010208941](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180102998.png)

添加nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" 后访问正常

![image-20221018010345523](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180103610.png)

yaml文件：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "true"
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS    #重点这个annotations
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: kubernetes-dashboard.chen1900s.cn     #域名替换成自己的域名
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - kubernetes-dashboard.chen1900s.cn
    secretName: chen1900s                          #替换成自己的证书secret
```



## 基于CLB类型ingress方式访问kubernetes-dashboard配置

CLB类型ingress，对应后端服务协议是默认HTTP的，后端协议是指 CLB 与后端服务之间协议，后端协议选择 HTTP 时，后端服务需部署 HTTP 服务。后端协议选中 HTTPS 时，后端服务需部署 HTTPS 服务，HTTPS 服务的加解密会让后端服务消耗更多资源

如果需要后端协议为HTTPS 则需要使用TkeServiceConfig来配置ingress自动创建的CLB

如果不通过TkeServiceConfig配置后端是HTTPS服务时候，访问会异常，如下：

![image-20221018011724396](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180117469.png)

### 使用 TkeServiceConfig 配置 CLB

创建 Ingress 时，设置 **ingress.cloud.tencent.com/tke-service-config-auto:<true>** ，将自动创建 <IngressName>-auto-ingress-config

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/tke-service-config-auto: "true"   #自动创建TkeServiceConfig
    kubernetes.io/ingress.class: qcloud
    kubernetes.io/ingress.rule-mix: "true"
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: kubernetes-dashboard.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - kubernetes-dashboard.chen1900s.cn
    secretName: chen1900s-ye4ubdzo

```

查看TkeServiceConfig配置

```bash
[root@VM-249-41-tlinux ~]# kubectl  -n  kubernetes-dashboard  get TkeServiceConfig   kubernetes-dashboard-auto-ingress-config 
NAME                                       AGE
kubernetes-dashboard-auto-ingress-config   66s
```

编辑这个配置文件，修改后端协议 `spec.loadBalancer.l7listeners.protocol.domain.rules.url.forwardType`: 指定后端协议

```bash
kubectl  -n  kubernetes-dashboard  edit  TkeServiceConfig   kubernetes-dashboard-auto-ingress-config 
```

![image-20221018012329401](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180123467.png)

然后使用这个CLB对应的域名进行访问

![image-20221018012426758](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180124832.png)

![image-20221018012544266](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210180125382.png)

附：

```yaml
apiVersion: cloud.tencent.com/v1alpha1
kind: TkeServiceConfig
metadata:
  name: kubernetes-dashboard-auto-ingress-config
  namespace: kubernetes-dashboard
spec:
  loadBalancer:
    l7Listeners:
    - defaultServer: kubernetes-dashboard.chen1900s.cn
      domains:
      - domain: kubernetes-dashboard.chen1900s.cn
        rules:
        - forwardType: HTTPS                  #主要修改这个   指定后端协议
          healthCheck:
            enable: true
            healthNum: 3
            httpCheckDomain: kubernetes-dashboard.chen1900s.cn
            httpCheckMethod: HEAD
            httpCheckPath: /
            httpCode: 31
            intervalTime: 5
            sourceIpType: 0
            timeout: 5
            unHealthNum: 3
          scheduler: WRR
          session:
            enable: false
          url: /
      keepaliveEnable: 0
      port: 443
      protocol: HTTPS
    - keepaliveEnable: 0
      port: 80
      protocol: HTTP
```

