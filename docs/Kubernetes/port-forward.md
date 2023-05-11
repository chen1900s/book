# port-forward访问TKE集群中的应用程序

本文描述了如何使用 `kubectl port-forward` 访问 Kubernetes 集群中的 redis server。这种连接方式在实际进行Debug时非常有效。 

### 1，部署Redis服务

 YAML文件如下： 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: redis
    qcloud-app: redis
  name: redis
  namespace: default
spec:
  replicas:
  selector:
    matchLabels:
      k8s-app: redis
      qcloud-app: redis
  template:
    metadata:
      labels:
        k8s-app: redis
        qcloud-app: redis
    spec:
      containers:
      - image: redis:latest
        imagePullPolicy: Always
        name: redis
        resources:
          limits:
            cpu: 200m
            memory: 256Mi
          requests:
            cpu: 200m
            memory: 256Mi
        ports:
        - containerPort: 6379

```

 创建Redis服务，YAML文件如下所示： 

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: redis
    qcloud-app: redis
  name: redis
  namespace: default
spec:
  ports:
  - name: 6379-6379-tcp
    port: 6379
    protocol: TCP
    targetPort: 6379
  selector:
    k8s-app: redis
    qcloud-app: redis
  type: ClusterIP
```

 执行命令，检查POD 和 Service创建结果 

```
[root@VM-1-14-tlinux ~]# kubectl  get pods 
NAME                    READY   STATUS    RESTARTS   AGE
redis-7cdccd6f4-6mszh   1/1     Running   0          2m13s
[root@VM-1-14-tlinux ~]# kubectl  get deployment
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
redis                  1/1     1            1           2m24s
[root@VM-1-14-tlinux ~]# kubectl  get service  | grep redis
redis        ClusterIP   172.21.252.118   <none>        6379/TCP   7m10s
```

 验证 Redis Service已经运行，并监听了 6379 端口 ：

```
[root@VM-1-14-tlinux ~]# kubectl get pods  redis-7cdccd6f4-6mszh  --template='{{(index (index .spec.containers 0).ports 0).containerPort}}{{"\n"}}'
6379
```

### 2，转发本地端口到Pod的端口

 使用 `kubectl port-forward` 命令，用户可以使用资源的名称来进行端口转发。下面的命令中的任意一行，都可以实现端口转发的效果： 

```
# 这几个命令执行任意一个即可
kubectl port-forward redis-7cdccd6f4-6mszh 7000:6379
kubectl port-forward pods/redis-7cdccd6f4-6mszh 7000:6379
kubectl port-forward deployment/redis 7000:6379
kubectl port-forward rs/redis 7000:6379
kubectl port-forward svc/redis 7000:6379
```

 以上命令的输出结果类似： 

```
[root@VM-1-14-tlinux ~]# kubectl port-forward redis-7cdccd6f4-6mszh 7000:6379
Forwarding from 127.0.0.1:7000 -> 6379
Forwarding from [::1]:7000 -> 6379
```

## 3，redis-cli连接方法

安装redis-cli（如已安装，可跳过）。

1. 登录待安装redis-cli的设备，例如CVM机器

2. 执行下述命令下载Redis源码文件：

   ```javascript
   wget https://github.com/redis/redis/archive/7.0.2.tar.gz
   ```

   **说明**  具体操作，请参见[Redis官网](https://redis.io/download)。

3. 执行下述命令解压Redis源码文件：

   ```undefined
   tar xzf redis-6.0.9.tar.gz
   ```

4. 执行下述命令进入解压后的目录并编译安装Redis源码文件：

   ```go
   cd redis-6.0.9&&make
   ```

> > **说明** 编译安装需要一段时间（通常2分钟~3分钟）。 

5. 启动 Redis 命令行：     

```
[root@VM-1-14-tlinux ~/redis-7.0.2]# src/redis-cli  -p 7000
127.0.0.1:7000> ping 
PONG
```

 Redis 服务器将返回 `PONG` ，表示链接成功

> >由于一些限制，port-forward 目前只支持 TCP 协议，issue 47862 (opens new window)用来跟进对 UDP 协议的支持。
> >MySQL数据库等使用 TCP 协议的部署在K8S集群中的服务器，都可以使用此方式进行 DEBUG