

## 应用场景

当需明确服务请求来源以满足业务需求时，则需后端服务能够准确获取请求客户端的真实源 IP。例如以下场景：

- 具有对服务请求的来源进行审计的需求，例如异地登录告警。
- 具有针对安全攻击或安全事件溯源的需求，例如 APT 攻击及 DDoS 攻击等。
- 业务场景具有数据分析的需求，例如业务请求区域统计。
- 其他需获取客户端地址的需求。

## 实现方法

在 TKE 中默认的外部负载均衡器为 [腾讯云负载均衡](https://cloud.tencent.com/product/clb) 作为服务流量的访问首入口，腾讯云负载均衡器会将请求流量负载转发到 Kubernetes 工作节点的 Kubernetes Service（默认）。此负载均衡过程会保留客户端真实源 IP（透传转发），但在 Kubernetes Service 转发场景下，无论使用 iptbales 或 ipvs 的负载均衡转发模式，转发时都会对数据包做 SNAT，即不会保留客户端真实源 IP。在 TKE 使用场景下，本文提供以下4种方式获取客户端真实源 IP，请参考本文按需选择适用方式。

## 实践验证



### 一 GR网络模式的集群

#### 1，通过 Service 资源的配置选项保留客户端源 IP

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: nginx
  name: nginx
  namespace: service7
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: nginx
  template:
    metadata:
      labels:
        k8s-app: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx

```

#### 2，查看POD状态及调度节点

调度在 192.168.0.11

```
[root@VM-0-17-tlinux ~]# kubectl  get pods -nservice -owide | grep nginx
nginx-56745866db-hcc55              1/1     Running   0          35d   172.18.0.80      192.168.0.11   <none>           <none>
```

#### 3，直接在集群内节点访问POD IP

1） 在集群另外一个节点17上访问，获取的还是192.168.0.17 节点IP，因为在集群网络内不做Snat ,POD看到的就是真实IP

![image-20211205104053996](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137032.png)

![image-20211205104124012](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137085.png)

2） 在POD所在节点上去访问如下172.18.0.65 ,有些人会问，这个IP是什么IP呢，**其实是POD所在节点的网桥cbr0**的IP

![image-20211205104318096](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137138.png)

![image-20211205104337577](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137186.png)

![image-20211205104603088](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137240.png)

3） 在集群内其他节点上POD直接访问，获取到的是POD真实IP

![image-20211205104825738](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137281.png)

![image-20211205104932755](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137322.png)

![image-20211205104959158](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137359.png)

4） 在相同节点上POD去访问

![image-20211205120215853](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137402.png)

![image-20211205120307799](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137442.png)

![image-20211205120333177](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137488.png)



#### 4，非local模式CLB类型的service

```
apiVersion: v1
kind: Service
metadata
  labels:
    k8s-app: nginx
  name: nginx
  namespace: service
spec:
  clusterIP: 172.18.251.84
  externalTrafficPolicy: Cluster
  ports:
  - name: 80-80-tcp
    nodePort: 32197
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    k8s-app: nginx
  sessionAffinity: None
  type: LoadBalancer


```

```
[root@VM-0-17-tlinux ~]# kubectl  get svc -n service | grep nginx
nginx   LoadBalancer   172.18.251.84    114.117.221.190   80:32197/TCP   5m55s

#集群外节点去访问：
[root@k8s-node01 ~]# for i in {1..12};do curl -I 114.117.221.190 ;done
```

1）其他客户端访问公网CLB 查看发现，访问客户端IP全部Snat为节点IP

![image-20211205121632397](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137530.png)

![image-20211205121451546](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137570.png)

2）集群内节点通过公网去访问 

![image-20211205122125282](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137611.png)

![image-20211205122232015](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137658.png)

![image-20211205123142620](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137720.png)



#### 5，service 使用externalTrafficPolicy：Local模式

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "false"
    service.cloud.tencent.com/local-svc-weighted-balance: "true"
    service.kubernetes.io/local-svc-only-bind-node-with-pod: "true"
  labels:
    k8s-app: nginx
  name: nginx
  namespace: service
spec:
  clusterIP: 172.18.251.84
  externalTrafficPolicy: Local
  healthCheckNodePort: 32699
  ports:
  - name: 80-80-tcp
    nodePort: 32197
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    k8s-app: nginx
  sessionAffinity: None
  type: LoadBalancer

```

1）CLB只绑定POD所在节点

![image-20211205123747582](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137759.png)

2） 其他客户端访问公网CLB 查看发现，可以获取到客户端真实IP

![image-20211205124121281](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137805.png)

![image-20211205124143154](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137843.png)

3） 在集群内节点访问公网CLB 查看，还是节点的IP

![image-20211205124415862](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137895.png)



#### 6，后端service是local模式的clb 类型的ingress

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: qcloud
    kubernetes.io/ingress.rule-mix: "false"
  name: nginx
  namespace: service
spec:
  rules:
  - host: chen.nginx.cn
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /

```

```
[root@VM-0-17-tlinux ~]# kubectl  get ingress -n service
NAME    CLASS    HOSTS           ADDRESS           PORTS   AGE
nginx   <none>   chen.nginx.cn   139.186.101.117   80      2m35s
```

**1）集群外节点访问**

![image-20211205125628466](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137933.png)

![image-20211205130043717](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137980.png)

**2） 集群内节点去访问**

![image-20211205130243510](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137020.png)

![image-20211205130230561](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137060.png)



### 二  VPC-CNI网络模式 CLB 直通 Pod 转发模式获取

该方式优缺点分析如下：

- **优点**：为 TKE 原生支持的功能特性，只需在控制台参考对应文档配置即可。
- **缺点**：集群需开启 VPC-CNI 网络模式，详情请参见 [VPC-CNI 模式说明](https://cloud.tencent.com/document/product/457/34993)。

使用 TKE 原生支持的 CLB 直通 Pod 的转发功能（CLB 透传转发，并绕过 Kubernetes Service 流量转发），后端 Pods 收到的请求的源 IP 即为客户端真实源 IP，此方式适用于四层及七层服务的转发场景。转发原理如下图

#### 1，部署POD资源对象

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-deployment-eni
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        tke.cloud.tencent.com/networks: tke-route-eni
      labels:
        app: nginx
    spec:
      containers:
        - image: nginx
          name: nginx
          resources:
            requests:
              tke.cloud.tencent.com/eni-ip: "1"
            limits:
              tke.cloud.tencent.com/eni-ip: "1"
```

```
[root@VM-0-17-tlinux ~]# kubectl  get pod -n service -o wide  | grep eni
nginx-deployment-eni-bc566bd66-2gfdz   1/1     Running   0          68s   192.168.253.9    192.168.0.17   <none>           1/1
```

#### 2，直接在集群内访问POD IP

![image-20211205132249641](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137106.png)

![image-20211205132400042](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137149.png)

#### 3，service直连POD模式

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"
  labels:
    app: nginx
  name: nginx-eni
  namespace: service
spec:
  clusterIP: 172.18.253.247
  externalTrafficPolicy: Cluster
  ports:
  - name: 80-80-tcp
    nodePort: 30240
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: LoadBalancer


```

**1）集群外节点通过公网VIP访问**

![image-20211205132900857](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137194.png)

能够正常获取到

![image-20211205132928137](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137235.png)



**2）在集群内节点访问**

看到是节点内外IP

![image-20211205133045364](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137280.png)

#### 4，service非直连POD模式

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "false"
  labels:
    app: nginx
  name: nginx-eni
  namespace: service
spec:
  clusterIP: 172.18.253.247
  externalTrafficPolicy: Cluster
  ports:
  - name: 80-80-tcp
    nodePort: 30240
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: LoadBalancer

```

及时后端POD使用的是eni模式，但是service如果不选择直连POD模式，还是通过NodePort模式转发的

![image-20211205134857124](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137324.png)

#### 5，ingress直连POD模式

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/direct-access: "true"
    kubernetes.io/ingress.class: qcloud
  name: nginx-eni
  namespace: service
spec:
  rules:
  - host: chen.nginx-eni.cn
    http:
      paths:
      - backend:
          serviceName: nginx-eni
          servicePort: 80
        path: /

```

**1）集群外节点去访问**

![image-20211205135630739](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137367.png)

**2）集群内节点访问 走的是公网IP**

![image-20211205135723134](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137411.png)



#### 6，ingress非直连POD模式

![image-20211205140607074](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137452.png)

### 三  VPC-CNI网络模式nginx-ingress获取客户端源IP



```
root@VM-0-17-tlinux ~]# kubectl get pod -n kube-system -owide |grep nginx-ingress-eni
nginx-ingress-eni-ingress-nginx-controller-744d9ff489-649wp   1/1     Running   0          18m     192.168.253.10   192.168.2.36   <none>           1/1
```

![image-20211205162013403](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137507.png)



#### 1，后端POD使用的是GR模式，nginx-ingress-controller使用的是VPC-CNI直连模式（正常获取）

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress-eni
    kubernetes.io/ingress.rule-mix: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx
  namespace: service
spec:
  rules:
  - host: chen.nginx.cn
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /
```

测试命令：

```
for i in {1..4};do curl -H "Host: chen.nginx.cn" -s http://114.117.221.188/  ;done
```

1）VPC外其他节点访问

![image-20211205162049073](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137548.png)

2）集群内节点访问

![image-20211205162317704](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137587.png)



#### 2，后端POD使用的是VPC-CNI模式，nginx-ingress-controller使用的是VPC-CNI直连模式（正常获取）

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress-eni
    kubernetes.io/ingress.rule-mix: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx-eni
  namespace: service
spec:
  rules:
  - host: chen.nginx-eni.cn
    http:
      paths:
      - backend:
          serviceName: nginx-eni
          servicePort: 80
        path: /
```

测试命令：

```
for i in {1..4};do curl -H "Host: chen.nginx-eni.cn" -s http://114.117.221.188/  ;done
```

1）VPC外其他节点访问

![image-20211205162738016](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137629.png)



2）集群内节点访问

![image-20211205162951204](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137674.png)



#### 3，后端POD使用的是GR模式，nginx-ingress-controller使用的是local模式（正常获取）

![image-20211205164454396](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137726.png)

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: nginx-local
  namespace: service
spec:
  rules:
  - host: chen.nginx-local.cn
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /
```

测试命令：

```
for i in {1..4};do curl -H "Host: chen.nginx-local.cn" -s http://118.24.224.103/  ;done
```

1）VPC外节点访问

![image-20211205164907296](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137768.png)

![image-20211205165006929](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137808.png)

#### 4，后端POD使用的是GR模式，nginx-ingress-controller使用默认模式（无法获取）

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress
  name: nginx-cluster
  namespace: service
spec:
  rules:
  - host: chen.nginx-cluster.cn
    http:
      paths:
      - backend:
          serviceName: nginx
          servicePort: 80
        path: /
```

测试命令

```
for i in {1..4};do curl -H "Host: chen.nginx-cluster.cn" -s http://114.117.219.97/  ;done
```

![image-20211205165430446](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137856.png)





### 四 k8s nginx-ingress获取真实IP地址配置

https://developer.aliyun.com/article/699074

https://www.imooc.com/article/303503

#### GR模式+非local模式

集群是GR模式，创建的nginx-ingress实例默认使用GR网络模式，并且service使用的是非local模式

**nginx-ingress-controller 配置：**

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  name: realip-nginx-ingress-nginx-controller
  namespace: kube-system
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 31827
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    nodePort: 31404
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  sessionAffinity: None
  type: LoadBalancer


```

**部署测试案例**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: cjweichen
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
      - image: ccr.ccs.tencentyun.com/chenjingwei/whoami:latest
        imagePullPolicy: Always
        name: whoami
        ports:
        - containerPort: 80
          name: 80tcp02
          protocol: TCP
      dnsPolicy: ClusterFirst
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: whoami
  name: whoami
  namespace: cjweichen
spec:
  ports:
  - name: 80-80-tcp
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: whoami
  sessionAffinity: None
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: whoami
  namespace: cjweichen
spec:
  rules:
  - host: whoami.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: whoami
          servicePort: 80
        path: /
        pathType: ImplementationSpecific


```

访问服务 其中X-Forwarded-For 会SNAT 成集群节点IP

```
[root@172-16-155-8 ~]# curl http://whoami.chen1900s.cn/
Hostname: whoami-5696f977bb-5xks6
IP: 127.0.0.1
IP: 10.55.1.14
RemoteAddr: 10.55.1.17:53128
GET / HTTP/1.1
Host: whoami.chen1900s.cn
User-Agent: curl/7.29.0
Accept: */*
X-Forwarded-For: 172.16.55.9
X-Forwarded-Host: whoami.chen1900s.cn
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Real-Ip: 172.16.55.9
X-Request-Id: c3e95454d539564bb547352431464788
X-Scheme: http

```

![1654776033036](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137898.png)

![1654776183384](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137943.png)

其中10.55.1.1 为pod所在CVM节点的cbr0  IP

#### GR模式+local模式

nginx-ingress-controller  service 配置

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "false"
    service.cloud.tencent.com/local-svc-weighted-balance: "false"
    service.kubernetes.io/local-svc-only-bind-node-with-pod: "true"
  labels:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  name: realip-nginx-ingress-nginx-controller
  namespace: kube-system
spec:
  externalTrafficPolicy: Local
  healthCheckNodePort: 30186
  ports:
  - name: http
    nodePort: 31827
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 31404
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  sessionAffinity: None
  type: LoadBalancer

```

通过客户端curl 可以看到，能够正常获取到客户端源IP

![1654776729696](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137997.png)

![1654777235123](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042137052.png)

#### 直连POD模式+非local模式

nginx-ingress 配置（注意点：需要后端POD也是使用eni模式 ）

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "true"
  labels:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  name: realip-nginx-ingress-nginx-controller
  namespace: kube-system
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 31827
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 31404
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    k8s-app: realip-nginx-ingress-nginx-controller
    qcloud-app: realip-nginx-ingress-nginx-controller
  sessionAffinity: None
  type: LoadBalancer

```









修改configmap nginx-configuration配置

```
  compute-full-forwarded-for: "true"
  forwarded-for-header: "X-Forwarded-For"
  use-forwarded-headers: "true"

```

