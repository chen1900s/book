## 使用 CLB 实现简单的蓝绿发布和灰度发布

### 原理介绍

用户通常使用 Deployment、StatefulSet 等 Kubernetes 自带的工作负载来部署业务，每个工作负载管理一组 Pod。以 Deployment 为例，示意图如下：

![image-20211020214315751](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042121669.png)

通常还会为每个工作负载创建对应的 Service，Service 通过 selector 来匹配后端 Pod，其他服务或者外部通过访问 Service 即可访问到后端 Pod 提供的服务

### 蓝绿发布原理

以 Deployment 为例，集群中已部署两个不同版本的 Deployment，其 Pod 拥有共同的 label。但有一个 label 值不同，用于区分不同的版本。Service 使用 selector 选中了其中一个版本的 Deployment 的 Pod，此时通过修改 Service 的 selector 中决定服务版本的 label 的值来改变 Service 后端对应的 Deployment，即可实现让服务从一个版本直接切换到另一个版本。示意图如下：

![image-20211020214409761](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042121720.png)

### 灰度发布原理

用户通常会为每个工作负载创建一个 Service，但 Kubernetes 未限制 Servcie 需与工作负载一一对应。Service 通过 selector 匹配后端 Pod，若不同工作负载的 Pod 可被同一 selector 选中，即可实现一个 Service 对应多个版本工作负载。调整不同版本工作版本的副本数即调整不同版本服务的权重。示意图如下：

![image-20211020214429570](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042121768.png)



### 使用 YAML 创建资源

1. 在集群中部署第一个版本的 Deployment，本文以 nginx 为例。版本V1   YAML 示例如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
  namespace: green-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: "openresty/openresty:centos"
        ports:
        - name: http
          protocol: TCP
          containerPort: 80
        volumeMounts:
        - mountPath: /usr/local/openresty/nginx/conf/nginx.conf
          name: config
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-v1

---

apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: nginx
    version: v1
  name: nginx-v1
  namespace: green-blue
data:
  nginx.conf: |-
    worker_processes  1;

    events {
        accept_mutex on;
        multi_accept on;
        use epoll;
        worker_connections  1024;
    }

    http {
        ignore_invalid_headers off;
        server {
            listen 80;
            location / {
                access_by_lua '
                    local header_str = ngx.say("nginx-v1")
                ';
            }
        }
    }
```

2.再部署第二个版本的 Deployment，本文以 nginx 为例。YAML 示例如下：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
  namespace: green-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: "openresty/openresty:centos"
        ports:
        - name: http
          protocol: TCP
          containerPort: 80
        volumeMounts:
        - mountPath: /usr/local/openresty/nginx/conf/nginx.conf
          name: config
          subPath: nginx.conf
      volumes:
      - name: config
        configMap:
          name: nginx-v2
---

apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: nginx
    version: v2
  name: nginx-v2
  namespace: green-blue
data:
  nginx.conf: |-
    worker_processes  1;

    events {
        accept_mutex on;
        multi_accept on;
        use epoll;
        worker_connections  1024;
    }

    http {
        ignore_invalid_headers off;
        server {
            listen 80;
            location / {
                access_by_lua '
                    local header_str = ngx.say("nginx-v2")
                ';
            }
        }
    }
```

3，在集群的工作负载详情页查看部署情况

```
[root@VM-0-17-tlinux ~]# kubectl   get deployment -n green-blue
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
nginx-v1   3/3     3            3           10m
nginx-v2   3/3     3            3           9m8s
```

![image-20211020214738833](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042121809.png)



### 实现蓝绿发布

1. 为部署的 Deployment 创建 LoadBalancer（内外型） 类型的 Service 对外暴露服务，指定使用 v1 版本的服务。YAML 示例如下：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-mwkztq4u
  name: nginx
  namespace: green-blue
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: nginx
    version: v1
```

2，执行以下命令，测试访问，返回结果如下，均为 v1 版本的响应。

```
[root@VM-0-17-tlinux ~]# for i in {1..10}; do curl 192.168.3.13; done; 
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
```

3，通过控制台或 kubectl 方式修改 Service 的 selector，使其选中 v2 版本的服务：

```
selector:
  app: nginx
  version: v2
  
#通过 kubectl 修改：
kubectl patch service nginx -p '{"spec":{"selector":{"version":"v2"}}}' -n green-blue
```

4，执行以下命令，再次测试访问，返回结果如下，均为 v2 版本的响应，成功实现了蓝绿发布。

```
[root@VM-0-17-tlinux ~]# for i in {1..10}; do curl 192.168.3.13; done; 
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
```

### 实现灰度发布

1. 对比蓝绿发布，不指定 Service 使用 v1 版本服务。即从 selector 中删除 `version` 标签，让 Service 同时选中两个版本的 Deployment 的 Pod。YAML 示例如下

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-mwkztq4u
  name: nginx
  namespace: green-blue
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    name: http
  selector:
    app: nginx
```

2，执行以下命令，测试访问。

```
for i in {1..10}; do curl EXTERNAL-IP; done; # 替换 EXTERNAL-IP 为 Service 的 CLB IP 地址
```

```
[root@VM-0-17-tlinux ~]# for i in {1..10}; do curl 192.168.3.13; done; 
nginx-v1
nginx-v1
nginx-v2
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v1
nginx-v2
nginx-v2
#返回结果如下，一半是 v1 版本的响应，另一半是 v2 版本的响应。
```

3，通过控制台或 kubectl 方式调节 v1 和 v2 版本的 Deployment 的副本，将 v1 版本调至 1 个副本，v2 版本调至 4 个副本：

通过 kubectl 修改

```bash
kubectl scale deployment/nginx-v1 --replicas=1  -n green-blue
kubectl scale deployment/nginx-v2 --replicas=4   -n green-blue
```

![image-20211020220207578](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042121853.png)

4，执行以下命令，再次进行访问测试。

```bash
for i in {1..10}; do curl EXTERNAL-IP; done; # 替换 EXTERNAL-IP 为 Service 的 CLB IP 地址
```

返回结果如下，10次访问中仅2次返回了 v1 版本，v1 与 v2 的响应比例与其副本数比例一致，为 1:4。通过控制不同版本服务的副本数就实现了灰度发布。

```
[root@VM-0-17-tlinux ~]# for i in {1..10}; do curl 192.168.3.13; done; 
nginx-v1
nginx-v1
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
nginx-v2
```