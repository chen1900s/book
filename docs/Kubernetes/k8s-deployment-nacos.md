---
title: Kubernetes部署nacos服务
tags:
  - Kubernetes
  - Nacos
categories: Kubernetes
keywords:
  - kubernetes
  - nacos
  - tke
description: kubernetes部署nacos服务
top_img: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/微信图片_20220828001026.jpg
abbrlink: 7af4624e
date: 2020-05-27 00:40:33
updated: 2022-05-27 23:58:58
---

# Kubernetes 部署 Nacos 集群

> 官网文档：https://nacos.io/zh-cn/docs/use-nacos-with-kubernetes.html

本文是基于腾讯云TKE容器服务集群搭建

## 部署数据库

**1，数据库是NFS做数据化存储**（或者CBS都可以）

需要注意： 镜像要使用nacos提供的数据库nacos/nacos-mysql:5.7，自带的数据库创建完成后相关库和数据都已经导入，其中NFS替换成自己NFS实例，自建或者云上CFS

```
apiVersion: apps/v1
kind: Deployment
metadata:
  generation: 1
  labels:
    k8s-app: nacos-mysql
    qcloud-app: nacos-mysql
  name: nacos-mysql
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: nacos-mysql
      qcloud-app: nacos-mysql
  template:
    metadata:
      labels:
        k8s-app: nacos-mysql
        qcloud-app: nacos-mysql
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: root
        - name: MYSQL_DATABASE
          value: nacos
        - name: MYSQL_USER
          value: nacos
        - name: MYSQL_PASSWORD
          value: nacos
        image: nacos/nacos-mysql:5.7
        imagePullPolicy: IfNotPresent
        name: mysql
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 256Mi
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-data
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      volumes:
      - name: mysql-data
        nfs:
          path: /nacos
          server: 192.168.1.6

---
apiVersion: v1
kind: Service
metadata:
  name: nacos-mysql
  namespace: default
  labels:
    name: nacos-mysql
spec:
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    k8s-app: nacos-mysql
    qcloud-app: nacos-mysql
```

验证数据库可用性：

```
[root@VM-0-17-tlinux ~/nacos]# mysql -h172.18.250.54 -unacos -pnacos
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nacos              |
+--------------------+
2 rows in set (0.01 sec)

MySQL [(none)]> use nacos
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [nacos]> show tables;
+----------------------+
| Tables_in_nacos      |
+----------------------+
| config_info          |
| config_info_aggr     |
| config_info_beta     |
| config_info_tag      |
| config_tags_relation |
| group_capacity       |
| his_config_info      |
| permissions          |
| roles                |
| tenant_capacity      |
| tenant_info          |
| users                |
+----------------------+
12 rows in set (0.00 sec)
```

## 部署nacos

**1，创建链接mysql的配置文件**

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-cm
  namespace: nacos
data:
  mysql.host: "172.18.253.36"  #如果是K8S集群内数据库，可以使用服务名称
  mysql.db.name: "nacos"      #上面创建数据库是指的的库名称
  mysql.port: "3306"           #端口
  mysql.user: "nacos"          #用户
  mysql.password: "nacos"      #用户密码
```

**2，创建nacos-headless 用于集群之间的链接**

```
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: nacos
  labels:
    app: nacos
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  ports:
    - port: 8848
      name: server
      targetPort: 8848
  clusterIP: None
  selector:
    app: nacos
    
   
```

**3，部署nacos服务**

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: nacos
spec:
  serviceName: nacos-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                      - nacos
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: k8snacos
          imagePullPolicy: Always
          image: nacos/nacos-server:latest
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
          ports:
            - containerPort: 8848
              name: client
          env:
            - name: NACOS_REPLICAS
              value: "3"
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.password
            - name: MODE
              value: "cluster"
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: PREFER_HOST_MODE
              value: "hostname"
            - name: NACOS_SERVERS
              value: "nacos-0.nacos-headless.nacos.svc.cluster.local:8848 nacos-1.nacos-headless.nacos.svc.cluster.local:8848 nacos-2.nacos-headless.nacos.svc.cluster.local:8848"
  selector:
    matchLabels:
      app: nacos
      
      
```

**4，创建公网类型CLB的service（可选）**

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/service.extensiveParameters: '{"AddressIPVersion":"IPV4","ZoneId":"ap-chongqing-1"}'
  name: nacos
  namespace: nacos
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: 8848-8848-tcp
    port: 8848
    protocol: TCP
    targetPort: 8848
  selector:
    app: nacos
  sessionAffinity: None
  type: LoadBalancer

```

**5，创建公网访问的ingress（可选）**

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.cloud.tencent.com/direct-access: "false"
  name: nacos-ingress
  namespace: nacos
spec:
  rules:
  - host: nacos.dev.cc
    http:
      paths:
      - backend:
          serviceName: nacos
          servicePort: 8848
        path: /nacos
```

## nacos使用

**1，使用service访问 或者ingress进行访问**

![image-20220826004712494](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/image-20220826004712494.png)

## 【附录事项】：

附1，如果对应数据库配置没有问题，但是nacos启动一直报如下错，找不到数据源

```
nacos server did not start because dumpservice bean construction failure. errMsg:102, dataSource or tableName is null
```

导致的原因是，需要额外加一下这个环境变量

SPRING_DATASOURCE_PLATFORM

值为：mysql

```
- name: SPRING_DATASOURCE_PLATFORM  #最新的版本一定要这个，nacos官方的yaml漏了这个（2.2版本，2.0版本是不需要加这个）
  value: "mysql"
```

附2，使用CBS做数据库的持久化存储，实例yaml: 仅参考

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: nacos-mysql
    qcloud-app: nacos-mysql
  name: nacos-mysql
  namespace: default
spec:
  podManagementPolicy: OrderedReady
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: nacos-mysql
      qcloud-app: nacos-mysql
  serviceName: ""
  template:
    metadata:
      annotations:
        eks.tke.cloud.tencent.com/retain-ip: "true" #EKS集群需要，TKE集群不需要这个
        eks.tke.cloud.tencent.com/root-cbs-size: "20"  #EKS集群需要，TKE集群不需要这个
      labels:
        k8s-app: nacos-mysql
        qcloud-app: nacos-mysql
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: root
        - name: MYSQL_DATABASE
          value: nacos
        - name: MYSQL_USER
          value: nacos
        - name: MYSQL_PASSWORD
          value: nacos
        image: nacos/nacos-mysql:5.7
        imagePullPolicy: IfNotPresent
        name: nacos-mysql
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 256Mi
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: nacos-mysql
          subPath: mysql
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nacos-mysql
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: cbs
      volumeMode: Filesystem

```

