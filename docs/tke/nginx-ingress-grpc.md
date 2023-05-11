---
title: Nginx-Ingress实现gRPC服务访问
abbrlink: 450ae69a
date: 2022-10-23 14:26:43
tags:
  - Nginx-ingress
  - Kubernetes
categories: Kubernetes
keywords:
  - Kubernetes
  - Nginx-ingress
description: 在TKE集群中使用Nginx-Ingress实现gRPC服务访问
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210231529348.jpeg
updated: 2022-10-23 14:26:43
---

这里验证下如何通过Ingress-NGINX控制器将流量路由到gRPC服务

> 前提条件：
>
> 1，已创建或者自建kubernetes集群
>
> 2，已经申请对应域名
>
> 3，已安装了ingress-nginx-controller
>
> 4，GRPC服务 可以使用[go-grpc-greeter-server](https://github.com/grpc/grpc-go/blob/91e0aeb192456225adf27966d04ada4cf8599915/examples/features/reflection/server/main.go)作为示例
>
> 5，和域名对应的SSL证书

## 部署gRPC应用

确保gRPC应用程序正在运行并监听连接，本示例使用镜像`ccr.ccs.tencentyun.com/v_cjweichen/grpc-server:latest`创建gRPC服务

复制以下YAML内容创建grpc.yaml文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grpc-service
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      run: grpc-service
  template:
    metadata:
      labels:
        run: grpc-service
    spec:
      containers:
      - image: ccr.ccs.tencentyun.com/chenjingwei/grpc-server:latest
        imagePullPolicy: Always
        name: grpc-service
        ports:
        - containerPort: 50051
          protocol: TCP
      imagePullSecrets:
      - name: qcloudregistrykey 
      restartPolicy: Always
```



## 为gRPC应用创建Service

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    run: grpc-service
  name: grpc-service
  namespace: default
spec:
  ports:
  - port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    run: grpc-service
  sessionAffinity: None
  type: ClusterIP
```

执行以下命令查看Pod

```bash
[root@VM-0-16-tlinux ~/grpc]# kubectl  get pods,svc -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE        NOMINATED NODE   READINESS GATES
pod/grpc-service-5d48b78787-hcfhf   1/1     Running   0          3m57s   172.16.0.5   10.0.0.16   <none>           <none>

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE     SELECTOR
service/grpc-service   ClusterIP   172.16.255.180   <none>        50051/TCP   2m41s   run=grpc-service
```



## 为gRPC应用创建ingress路由

复制以下YAML内容创建grpc-ingress.yaml文件。

> **注意**
>
> - 部署gRPC服务所使用的Ingress需要在**annotation**中加入`nginx.ingress.kubernetes.io/backend-protocol`，值为**GRPC**。
> - 本示例使用的域名为`grpc.example.com`，请根据实际情况修改。

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: GRPC   # 注意这里：必须要配置以指明后端服务为gRPC服务
  name: grpc-ingress
  namespace: default
spec:
  rules:
  - host: grpc.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: grpc-service
          servicePort: 50051
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - grpc.chen1900s.cn
    secretName: grpc-secret

```

## 测试连接

一旦我们将配置应用到Kubernetes上，就该测试我们是否可以实际与后端通信了。为此，我们将使用grpcurl实用测试程序:

需要提前安装grpcurl工具，grpcurl是Go语言开源社区开发的工具，需要手工安装：

```
wget https://github.com/fullstorydev/grpcurl/releases/download/v1.7.0/grpcurl_1.7.0_linux_x86_64.tar.gz
tar -xvf grpcurl_1.7.0_linux_x86_64.tar.gz
chmod +x grpcurl
./grpcurl -help
```

本示例中使用域名`grpc.chen1900s.cn`以及自签证书，执行以下命令验证请求是否成功转发到后端服务。

```
##grpcurl -insecure -authority grpc.example.com <ip_address>:443 list
##ip_address为Nginx Ingress Controller的Service外部IP


[root@VM-0-16-tlinux ~/grpc]# ./grpcurl -insecure -authority grpc.chen1900s.cn  170.106.134.2:443 list
greet.GrpcService
grpc.reflection.v1alpha.ServerReflection
```

从预期输出可得，流量被Ingress成功转发到后端gRPC服务

注意点：由于Nginx grpc_pass的限制，目前对于gRPC服务，暂不支持service-weight的配置
