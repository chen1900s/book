---
title: 在TKE里安装和使用Traefik Ingress
abbrlink: 94ad787b
date: 2020-06-07 16:14:33
tags:
  - TKE
  - Kubernetes
categories: TKE
keywords:
  - Kubernetes
  - TKE
  - Traefik Ingress
description: 在TKE里安装和使用Traefik Ingress
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171611176.png
updated: 2020-06-07 23:58:58
---

## 在TKE里安装和使用Traefik Ingress

> [Traefik 官方文档](https://doc.traefik.io/traefik/)

Traefik 是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具

-  Traefik的与众不同之处在于，除了它的许多特性之外，它还能自动发现服务的正确配置。当Traefik检查您的基础设施时，它会发现相关信息并发现哪个服务服务于哪个请求。
- Traefik从根本上兼容所有主要的集群技术，如Kubernetes、Docker、Docker Swarm、AWS、Mesos、Marathon等，并且可以同时处理很多问题。

- 使用Traefik，不需要维护和同步单独的配置文件，所有事情都是自动实时发生的(没有重启，没有连接中断)。使用Traefik，用户可以将时间花在开发和部署系统新功能上，而不是配置和维护系统的工作状态上。

**环境准备**

- TKE集群，版本1.18

### 安装Traefik方式一 YAML

> 本文介绍两种安装方式，yaml和helm chart方式，根据自己情况 只需要选一种即可

本文介绍如何在 TKE 环境中使用Traefik Ingress，目标是学习如何在Kubernetes中运行Traefik反向代理后的应用程序。它介绍并解释了开始使用Traefik所需的基本块，如入口控制器、入口、部署、静态和动态配置。

Traefik使用Kubernetes API来发现正在运行的服务，为了使用Kubernetes API, Traefik需要一些权限。此权限机制基于集群管理员定义的角色。然后将角色绑定到应用程序使用的帐户

**创建角色** 

traefik-role.yaml示例：

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role
  namespace: kube-system
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
```

创建服务账号：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-account
  namespace: kube-system
```

创建ClusterRoleBinding

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: traefik-role-binding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-role
subjects:
  - kind: ServiceAccount
    name: traefik-account
    namespace: kube-system  # Using "default" because we did not specify a namespace when creating the ClusterAccount.
```

**部署 ingress controller**

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik-ingress-controller
  labels:
    app: traefik
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-account
      containers:
        - name: traefik
          image: traefik:v2.8
          args:
            - --api.insecure
            - --providers.kubernetesingress.ingressclass=traefik
            - --log.level=DEBUG
          ports:
            - name: web
              containerPort: 80
            - name: dashboard
              containerPort: 8080
```

一个workload可以运行多个Traefik代理Pods，需要一个片段将流量转发到任何实例： 即一个服务。创建名为traefik-ingress-controller-services的文件。yaml并插入两个Service资源:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-dashboard-service
  namespace: kube-system
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: dashboard
  selector:
    app: traefik
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-ingress-controller-services
  namespace: kube-system
spec:
  type: LoadBalancer
  ports:
    - targetPort: web
      port: 80
  selector:
    app: traefik
```

执行以下命令将 Traefik 安装到 TKE 集群

```bash
[root@VM-249-130-tlinux ~/traefik]# kubectl  apply -f .
serviceaccount/traefik-account created
service/traefik-dashboard-service created
service/traefik-ingress-controller-services created
deployment.apps/traefik-ingress-controller created
clusterrolebinding.rbac.authorization.k8s.io/traefik-role-binding created
clusterrole.rbac.authorization.k8s.io/traefik-role created
```

获取流量入口的 IP 地址（如下为 EXTERNAL-IP 字段），以及Dashboard,示例如下

```bash
[root@VM-249-130-tlinux ~/traefik]# kubectl -n kube-system get pods,svc  | grep traefik
pod/traefik-ingress-controller-5b6dd45cfb-pvddr   1/1     Running            0          3m29s
service/traefik-dashboard-service             LoadBalancer   172.16.255.160   118.24.226.193    8080:30299/TCP   3m29s
service/traefik-ingress-controller-services   LoadBalancer   172.16.253.56    114.117.220.253   80:31292/TCP     3m29s
```

### 安装Traefik方式二 Helm

> Traefik可以安装在Kubernetes使用Helm 仓库从https://github.com/traefik/traefik-helm-chart。

确保满足以下要求:

- 已创建TKE集群 Kubernetes 1.14 +

- 已安装helm客户端  版本3.x以上 ，[helm安装参考](https://blog.chen1900s.cn/post/e3ad72ca.html)


添加Traefik的helm chart 仓库

```bash
helm repo add traefik https://helm.traefik.io/traefik
```

准备配置文件values.yaml：

```yaml
providers: 
  kubernetesIngress: 
    publishedService: 
      enabled: true   #让 Ingress的外部 IP 地址状态显示为 Traefik 的 LB IP 地址
additionalArguments: 
- "--providers.kubernetesingress.ingressclass=traefik" # 指定 ingress class 名称 
- "--log.level=DEBUG" 
ports: 
  web: 
    expose: true
    exposedPort: 80 # 对外的 HTTP 端口号，使用标准端口号在国内需备案
  websecure: 
    expose: true
    exposedPort: 443 # 对外的 HTTPS 端口号，使用标准端口号在国内需备案
  traefik:
    expose: true
    exposedPort: 9000
deployment: 
  enabled: true
  replicas: 1
```

> 完整的默认配置可执行 `helm show values traefik/traefik` 命令查看

用helm命令行安装：

```bash
helm install traefik -f values.yaml traefik/traefik  #默认是安装在default命名空间下的

#可以指定命名命名空间进行安装
kubectl create ns traefik-v2
# Install in the namespace "traefik-v2"
helm install --namespace=traefik-v2 -f values.yaml    traefik traefik/traefik

#后续如果修改配置使用如下命令进行更新
helm upgrade --install traefik -f values.yaml traefik/traefik
```

查看部署的资源

```bash
[root@VM-249-130-tlinux ~]# kubectl   get pods,svc
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-7bb76c56fb-skzvq   1/1     Running   0          5m22s

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                      AGE
service/kubernetes   ClusterIP      172.16.252.1     <none>          443/TCP                      95m
service/traefik      LoadBalancer   172.16.252.238   118.24.225.19   80:32467/TCP,443:30353/TCP   5m22s
```



### 使用 Ingress

我们使用示例应用程序traefik/whoami，但其原则适用于任何其他应用程序。

whoami应用程序是一个简单的HTTP服务器，运行在端口80上，它向传入的请求应答与主机相关的信息。和往常一样，首先创建一个名为whoami的文件。yml并粘贴以下部署资源:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - name: web
              containerPort: 80
```

创建service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  ports:
    - name: web
      port: 80
      targetPort: web
  selector:
    app: whoami
```

Traefik 支持使用 Kubernetes 的 Ingress 资源作为动态配置，可直接在集群中创建 Ingress 资源用于对外暴露集群，需要加上指定的 Ingress class（安装 Traefik 时定义）。示例如下

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata: 
  name: whoami-ingress
  annotations: 
    kubernetes.io/ingress.class: traefik # 这里指定 ingress class 名称
spec: 
  rules: 
  - host: traefik.chen1900s.cn
    http: 
      paths: 
      - path: /
        backend: 
          serviceName: whoami
          servicePort: 80
```

> TKE 暂未将 Traefik 产品化，无法直接在 TKE 控制台进行可视化创建 Ingress，需要使用 YAML 进行创建。

**使用安装方式一 部署后访问测试如下**

```bash
[root@VM-249-130-tlinux ~/traefik]# nslookup traefik.chen1900s.cn  #解析到traefik-ingress-controller的service VIP上
Server:         183.60.82.98
Address:        183.60.82.98#53

Non-authoritative answer:
Name:   traefik.chen1900s.cn
Address: 114.117.220.253

[root@VM-249-130-tlinux ~/traefik]# curl http://traefik.chen1900s.cn/
Hostname: whoami-5db58df676-vwvzv
IP: 127.0.0.1
IP: 172.16.0.5
RemoteAddr: 172.16.0.7:49846
GET / HTTP/1.1
Host: traefik.chen1900s.cn
User-Agent: curl/7.29.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.16.0.1
X-Forwarded-Host: traefik.chen1900s.cn
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-ingress-controller-745b557778-jrjzn
X-Real-Ip: 172.16.0.1
```

**使用安装方式二  部署后访问测试如下：**

```bash
[root@VM-0-33-tlinux ~]# curl http://traefik.chen1900s.cn/
Hostname: whoami-5db58df676-856ss
IP: 127.0.0.1
IP: 172.16.0.10
RemoteAddr: 172.16.0.9:58498
GET / HTTP/1.1
Host: traefik.chen1900s.cn
User-Agent: curl/7.29.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.16.0.1
X-Forwarded-Host: traefik.chen1900s.cn
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-7bb76c56fb-skzvq
X-Real-Ip: 172.16.0.1

[root@VM-0-33-tlinux ~]# nslookup traefik.chen1900s.cn   #正常解析到 方式二 对应的service VIP上
Server:         183.60.83.19
Address:        183.60.83.19#53

Non-authoritative answer:
Name:   traefik.chen1900s.cn
Address: 118.24.225.19
```



### 使用Traefik Dashboard

使用第一种方式安装时候默认安装了Traefik Dashboard，使用方式二Helm chart 方式安装，出于安全考虑，这个Helm Chart默认情况下不公开Traefik Dashboard。因此，如果需要暴露Dashboard端口，需要在values.yaml里面添加

```yaml
  traefik:
    expose: true
    exposedPort: 9000
```

> 还有更多参数可以通过`helm show values traefik/traefik` 命令查看， 比如暴露metrics端口等等

或者使用IngressRoute CRD

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dashboard
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`traefikdashboard.chen1900s.cn`) && (PathPrefix(`/dashboard`) || PathPrefix(`/api`))
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
```

方式一安装默认是8080端口

![image-20220917171631043](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171716193.png)



方式二helm方式安装，端口设置的是9000

http://EXTERNAL-IP:9000/dashboard/#/

![image-20220917180436515](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209171804625.png)



### 使用 IngressRoute

Traefik 不仅支持标准的 Kubernetes Ingress 资源，也支持 Traefik 特有的 CRD 资源，例如 IngressRoute，可以支持更多 Ingress 不具备的高级功能。IngressRoute 使用示例如下：

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata: 
  name: whoami-ingressroute
spec: 
  entryPoints: 
    - web
  routes: 
    - match: Host(`traefikroute.chen1900s.cn`) && PathPrefix(`/`)
      kind: Rule
      services: 
        - name: whoami
          port: 80
```

创建ingressroute

```bash
#kubectl  apply -f whoami-ingressroute.yaml 
ingressroute.traefik.containo.us/whoami-ingressroute created

```

查看是否创建成功

```bash
# kubectl   get ingressroute
NAME                  AGE
whoami-ingressroute   48s
```

访问验证

```bash
[root@VM-249-130-tlinux ~]# curl  http://traefikroute.chen1900s.cn
Hostname: whoami-5db58df676-856ss
IP: 127.0.0.1
IP: 172.16.0.10
RemoteAddr: 172.16.0.9:57430
GET / HTTP/1.1
Host: traefikroute.chen1900s.cn
User-Agent: curl/7.29.0
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 172.16.0.1
X-Forwarded-Host: traefikroute.chen1900s.cn
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-7bb76c56fb-skzvq
X-Real-Ip: 172.16.0.1
```

