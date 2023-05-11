## 在kubernetes中搭建LNMP环境，并安装Discuzx

本实验，需要已经搭建好kubernetes集群和harbor服务（使用腾讯云镜像仓库）。

首先克隆本项目：git clone https://github.com/donxan/k8s_lnmp_discuzx.git

#### 下载镜像

```
docker pull mysql:5.7
docker pull richarvey/nginx-php-fpm
```

#### 用dockerfile重建nginx-php-fpm镜像

```
cd k8s_discuz/dz_web_dockerfile/
docker build -t ccr.ccs.tencentyun.com/v_cjweichen/nginx-php .
```

#### 将镜像push到镜像仓库

```
docker login ccr.ccs.tencentyun.com
 
docker push ccr.ccs.tencentyun.com/v_cjweichen/nginx-php:latest

docker tag mysql:5.7 ccr.ccs.tencentyun.com/v_cjweichen/mysql:5.7

docker push ccr.ccs.tencentyun.com/v_cjweichen/mysql:5.
 
```

#### 搭建MySQL服务

- 创建secret (设定mysql的root密码)

```
kubectl create secret generic mysql-pass --from-literal=password=DzPasswd1
```

- 创建pv

```
cd ../../k8s_discuz/mysql
kubectl apply -f mysql-pv.yaml

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: 192.168.1.6
      path: /k8s/discuz/db
    volumeHandle: mysql-pv
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cfs
  volumeMode: Filesystem

```

- 创建pvc

```
kubectl apply -f mysql-pvc.yaml

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-claim
  namespace: lnmp
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs
  volumeMode: Filesystem
  volumeName: mysql-pv
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi

```

- 创建deployment

```
kubectl apply -f mysql-dp.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dz-mysql
  labels:
    app: discuz
spec:
  selector:
    matchLabels:
      app: discuz
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: discuz
        tier: mysql
    spec:
      imagePullSecrets:
       - name: harbor-secret
      containers:
      - image: ccr.ccs.tencentyun.com/v_cjweichen/mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: dz-mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /data
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-claim
```

- 创建service

```
kubectl apply -f mysql-svc.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: dz-mysql
  labels:
    app: discuz
spec:
  ports:
    - port: 3306
  selector:
    app: discuz
    tier: mysql
```

#### 搭建Nginx+php-fpm服务

注意搭建步骤，在部署mysql时，不能deploy，svc一起执行，需要一步一步来操作。

- 搭建pv

```
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-pv.yaml
----
apiVersion: v1
kind: PersistentVolume
metadata:
  name: web-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: 192.168.1.6
      path: /k8s/discuz/web
    volumeHandle: mysql-pv
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cfs
  volumeMode: Filesystem

```

- 创建pvc

```
kubectl apply -f web-pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-claim
  namespace: lnmp
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs
  volumeMode: Filesystem
  volumeName: web-pv
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi

```

- 创建deployment

```
kubectl apply -f web-dp.yaml 

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dz-web
  labels:
    app: discuz
spec:
  replicas: 1
  selector:
    matchLabels:
      app: discuz
      tier: nginx-php
  template:
    metadata:
      labels:
        app: discuz
        tier: nginx-php
    spec:
      imagePullSecrets:
      - name: harbor-secret
      containers:
      - image: ccr.ccs.tencentyun.com/v_cjweichen/nginx-php:latest
        name: dz-web
        ports:
        - containerPort: 9000
        - containerPort: 80
          name: dz-web
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/www/html/
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: web-claim
```

- 创建service

```
kubectl apply -f web-svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: dz-web
  labels:
    app: discuz
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30001
  selector:
    app: discuz
    tier: nginx-php
```



#### 安装Discuz

- 下载dz代码 (到NFS服务器上)

```
cd /tmp/
git clone https://gitee.com/Discuz/DiscuzX.git
cd /nfsroot/k8s/discuz/web/
mv /tmp/DiscuzX/upload/* .
chown -R 100 data uc_server/data/ uc_client/data/ config/
```

- 设置MySQL普通用户

```
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -pDzPasswd1  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by 'dz-passwd-123';
```

#### 部署traefik ingress

```
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-ingress.yaml
```

 	如果没有部署ingress，可以使用安装nginx,配置nginx反向代理。 参考如下。

- 设置Nginx代理

```
注意：目前nginx服务是运行在kubernetes集群里，node节点以及master节点上是可以通过cluster-ip访问到，但是外部的客户端就不能访问了。
      所以，可以在任意一台node或者master上建一个nginx反向代理即可访问到集群内的nginx。
kubectl get svc dz-web //查看cluster-ip，我的ip是10.68.190.99
nginx代理配置文件内容如下：
server {
            listen 80;
            server_name dz.abcgogo.com;

            location / {
                proxy_pass      http://10.68.190.99:80;
                proxy_set_header Host   $host;
                proxy_set_header X-Real-IP      $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}

做完Nginx代理，就可以通过node的IP来访问discuz了。
```

#### 安装Discuz

按照提示配置即可。 