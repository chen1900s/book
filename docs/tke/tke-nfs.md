## 一，背景

 CFS 文件存储支持 NFS v3.0 及 NFS v4.0 协议， 其中 NFS v3.0 是 NFS 协议较早期版本，兼容 Windows 客户端；NFS v4.0 协议为稍后期版本，支持文件锁等功能 

 在大量小文件或者大小文件混合场景下，用户在 容器 TKE 或者云服务器 CVM 等客户端使用 NFS v4.0 协议挂载 CFS 文件系统，可能在应用运行一段时候后出现：客户端负载居高不下，无限累加，业务读取数据慢或无响应，但是业务进程的 CPU 使用率并不是很高的情况。 

## **二，优化**

 如果业务应用存在大量小文件的场景，或者并发操作文件数量巨大，推荐客户端使用 NFS v3.0 协议挂载。以下为规避大量小文件及大并发请求下客户端高负载的问题的方法

## 三，部署安装

**方式一： NFS 4.0 挂载根目录** 

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
  name: cfs
parameters:
  pgroupid: pgroup-g6hwk27v
  storagetype: SD
  subnetid: subnet-ge8hhr3e
  vpcid: vpc-bkoyzvar
  zone: ap-chongqing-1
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv-root
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: 192.168.1.6
      path: /
    volumeHandle: cfs-pv-root
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cfs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc-root
  namespace: storage
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs
  volumeMode: Filesystem
  volumeName: cfs-pv-root

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: centos-cfs-root
    qcloud-app: centos-cfs-root
  name: centos-cfs-root
  namespace: storage
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: centos-cfs-root
      qcloud-app: centos-cfs-root
  template:
    metadata:
      labels:
        k8s-app: centos-cfs-root
        qcloud-app: centos-cfs-root
    spec:
      containers:
      - env:
        - name: PATH
          value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        image: ccr.ccs.tencentyun.com/v_cjweichen/centos:v3
        imagePullPolicy: Always
        name: centos
        resources:
          limits:
            cpu: 100m
            memory: 64Mi
          requests:
            cpu: 100m
            memory: 64Mi
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /mnt
          name: cfs-vol
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      volumes:
      - name: cfs-vol
        persistentVolumeClaim:
          claimName: cfs-pvc-root

```

创建：

```
[root@VM-0-17-tlinux ~]# kubectl  apply -f centos-pvc-cfs.yaml 
persistentvolume/cfs-pv-root created
persistentvolumeclaim/cfs-pvc-root created
deployment.apps/centos-cfs-root created
[root@VM-0-17-tlinux ~]# 
[root@VM-0-17-tlinux ~]# kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
cfs-pv-root   10Gi       RWX            Retain           Bound    storage/cfs-pvc-root   cfs                     14s
[root@VM-0-17-tlinux ~]# kubectl get pvc -n storage
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cfs-pvc-root   Bound    cfs-pv-root   10Gi       RWX            cfs            29s
[root@VM-0-17-tlinux ~]# kubectl get pod -n storage
NAME                               READY   STATUS    RESTARTS   AGE
centos-cfs-root-6955f4656f-t4bhp   1/1     Running   0          35s
```



查看挂载情况

pod内：

```
[root@centos-cfs-root-7775cc5fbf-jgff8 /]# nfsstat -m
/mnt from 192.168.1.6:/
 Flags: rw,relatime,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.0.17,local_lock=none,addr=192.168.1.6
```

```
[root@centos-cfs-root-7775cc5fbf-jgff8 /]# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          50G  7.3G   40G  16% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
192.168.1.6:/    10G   32M   10G   1% /mnt
```

node节点上对应的命令如下：

```
[root@VM-0-17-tlinux ~]# df -h | grep cfs-pv-root
192.168.1.6:/    10G   32M   10G   1% /var/lib/kubelet/pods/e7b28b08-1322-4f8c-aefc-f3892f29b652/volumes/kubernetes.io~csi/cfs-pv-root/mount
```

![1626507030089](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156658.png)

**方式二  TKE POD 使用 NFS 3.0 挂载子目录** 

唯一不同点是 其中挂载参数要写在volumeAttributes：

示例：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv-root
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
      host: 192.168.1.6
      path: /bbs2yahv   ##这个目录 必须是CFS那边给的目录 自建目录不行
      vers: "3"         ##在这里声明NFSV3协议挂载，注意双引号
    volumeHandle: cfs-pv-root
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cfs
  volumeMode: Filesystem
```

 `总结来说 用v3挂载一定要加上fsid，在本文中对应为bbs2yahv，具体的还得参照你cfs中对应的fsid` 

```
[root@centos-cfs-root-6955f4656f-2blh4 /]# nfsstat -m
/mnt from 192.168.1.6://bbs2yahv
 Flags: rw,relatime,vers=3,rsize=1048576,wsize=1048576,namlen=255,hard,nolock,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.6,mountvers=3,mountport=892,mountproto=tcp,local_lock=all,addr=192.168.1.6
```

https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CFS.md

## 四，权限

**1，目录权限**

cfs挂载目录权限 是由CFS那边权限组设置

使用说明：

all_squash：所有访问用户都会被映射为匿名用户或用户组；

no_all_squash：访问用户会先与本机用户匹配，匹配失败后再映射为匿名用户或用户组；

root_squash：将来访的root用户映射为匿名用户或用户组；

no_root_squash：来访的root用户保持root帐号权限 （**默认是这个**）

**使用：no_root_squash这个权限 普通用户没有权限写 只有读的权限**

![1664267579935](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156703.png)

```
[chen@centos-cfs-root-6955f4656f-pvnxs mnt]$ cat 1111.txt 
hello.txt
[chen@centos-cfs-root-6955f4656f-pvnxs mnt]$ ls -lrt
total 4
drwxr-xr-x 2 root root  6 Jul 17 16:06 1111
-rw-r--r-- 1 root root 10 Jul 17 16:08 1111.txt
[chen@centos-cfs-root-6955f4656f-pvnxs mnt]$ echo 1111 >> 1111.txt 
-bash: 1111.txt: Permission denied
[chen@centos-cfs-root-6955f4656f-pvnxs mnt]$ mkdir data
mkdir: cannot create directory 'data': Permission denied
[chen@centos-cfs-root-6955f4656f-pvnxs mnt]$ cd 1111
[chen@centos-cfs-root-6955f4656f-pvnxs 1111]$ ls
[chen@centos-cfs-root-6955f4656f-pvnxs 1111]$ mkdr 111
```

## 五，部署mysql使用CFS存储

前置条件

1，创建CFS实例

2，权限组  来访地址： *  用户权限是：no_root_squash    **读写权限**：读写  



/var/lib/mysql 目录下的文件属性 都是mysql

```
root@mysql-cfs-data-66c5dc7875-fnm22:/var/lib/mysql# pwd
/var/lib/mysql
root@mysql-cfs-data-66c5dc7875-fnm22:/var/lib/mysql# ls -lrt 
total 198064
-rw-r----- 1 mysql mysql 50331648 Jul 17 08:22  ib_logfile1
-rw-r----- 1 mysql mysql  8585216 Jul 17 08:22 '#ib_16384_1.dblwr'
drwxr-x--- 2 mysql mysql     4096 Jul 17 08:23  performance_schema
root@mysql-cfs-data-66c5dc7875-fnm22:/var/lib/mysql# id mysql
uid=999(mysql) gid=999(mysql) groups=999(mysql）
```

![1626510303615](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156751.png)

需要将文件系统的权限组修改成 **no_all_squash**  访问用户会先与本机用户匹配，匹配失败后再映射为匿名用户或用户组；

![1626513815864](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156792.png)

yaml文件：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: mysql-cfs-data
    qcloud-app: mysql-cfs-data
  name: mysql-cfs-data
  namespace: storage
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: mysql-cfs-data
      qcloud-app: mysql-cfs-data
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: mysql-cfs-data
        qcloud-app: mysql-cfs-data
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
        image: mysql:5.7.16
        imagePullPolicy: IfNotPresent
        name: mysql
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 64Mi
        securityContext:
          privileged: false
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: mysql-vol
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      volumes:
      - name: mysql-vol
        persistentVolumeClaim:
          claimName: cfs-pvc-root
---
##内网型CLB
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-ge8hhr3e
  name: mysql-cfs-data
  namespace: storage
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: 3306-3306-tcp
    nodePort: 31761
    port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    k8s-app: mysql-cfs-data
    qcloud-app: mysql-cfs-data
  sessionAffinity: None
  type: LoadBalancer

```

查看部署结果

```
root@mysql-cfs-data-b58fc4767-7lzqp:/var/lib/mysql# df -h | grep /var         
192.168.1.6:/111   10G  241M  9.8G   3% /var/lib/mysql
root@mysql-cfs-data-b58fc4767-7lzqp:/var/lib/mysql# ls -lt
total 188452
drwxr-x--- 2 mysql mysql       19 Jul 17 09:28 chen
-rw-r----- 1 mysql mysql 12582912 Jul 17 09:26 ibtmp1
-rw-r----- 1 mysql mysql 50331648 Jul 17 09:25 ib_logfile0
-rw-r----- 1 mysql mysql 79691776 Jul 17 09:25 ibdata1
-rw-r----- 1 mysql mysql     1325 Jul 17 09:25 ib_buffer_pool
drwxr-x--- 2 mysql mysql     8192 Jul 17 09:25 sys
drwxr-x--- 2 mysql mysql     4096 Jul 17 09:25 mysql
drwxr-x--- 2 mysql mysql     8192 Jul 17 09:25 performance_schema
-rw-r----- 1 mysql mysql       56 Jul 17 09:25 auto.cnf
-rw-r----- 1 mysql mysql 50331648 Jul 17 09:25 ib_logfile1
```

![1626514270723](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156839.png)

 StatefulSet方式部署：

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: mysql
    qcloud-app: mysql
  name: mysql
  namespace: cjweichen
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: mysql
      qcloud-app: mysql
  serviceName: ""
  template:
    metadata:
      labels:
        k8s-app: mysql
        qcloud-app: mysql
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
        image: mysql:5.7
        imagePullPolicy: IfNotPresent
        name: mysql
        resources: {}
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: nfs
          subPath: mysql_docker/data
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      volumes:
      - name: nfs
        nfs:
          path: /
          server: 172.16.3.7

```

![1650375717677](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156887.png)

![1650375918353](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156939.png)

![1650375966195](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156992.png)





## 六，弹性集群EKS使用 NFS 3.0 挂载子目录

目前控制台没有CFS相关存储组件，可以创建静态CFS pv ,然后静态PVC绑定，以下是nfs 3.0协议挂载

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv
spec:
  mountOptions:
  - vers=3
  - nolock
  - proto=tcp
  - noresvport
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    server: 10.2.2.66   ########cfs地址，需确保网络能通
    path: /vshm4707        ########cfs侧拿到的路径
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: cfs-pv  
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    k8s-app: centos-cfs
    qcloud-app: centos-cfs
  name: centos-cfs
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: centos-cfs
      qcloud-app: centos-cfs
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      annotations:
        eks.tke.cloud.tencent.com/security-group-id: sg-7ad888n4
      labels:
        k8s-app: centos-cfs
        qcloud-app: centos-cfs
    spec:
      containers:
      - env:
        - name: PATH
          value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        image: ccr.ccs.tencentyun.com/v_cjweichen/centos:latest
        imagePullPolicy: Always
        name: centos
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 64Mi
        volumeMounts:
        - mountPath: /mnt
          name: cfs-vol
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      volumes:
      - name: cfs-vol
        persistentVolumeClaim:
          claimName: cfs-pvc
```

![1627544499622](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156038.png)

参考官方文档

https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/



## 七，跨云联网挂载CFS

目前TKE使用PVC和PV创建CFS，只能选择当前VPC下的CFS实例。有时用户有诉求，想挂载其他VPC下的CFS实例可以参考如下:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ccn
spec:
  accessModes:
     - ReadWriteMany
  capacity:
     storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cfs
    volumeAttributes:
       host: 10.2.2.66
       path: /
    volumeHandle: cfs-pv2
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-operationdata
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: pv-operationdata
```

```
[root@tke-node /data]# kubectl  exec -it nginx-d98d55878-445cv bash
root@nginx-d98d55878-445cv:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          50G  7.8G   39G  17% /
tmpfs            64M     0   64M   0% /dev
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
10.2.2.66:/      10G   32M   10G   1% /mnt
/dev/vda1        50G  9.6G   38G  21% /etc/hosts
/dev/vdb         50G  7.8G   39G  17% /etc/hostname
shm              64M     0   64M   0% /dev/shm
tmpfs           3.9G   12K  3.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs           3.9G     0  3.9G   0% /proc/acpi
tmpfs           3.9G     0  3.9G   0% /proc/scsi
tmpfs           3.9G     0  3.9G   0% /sys/firmware
root@nginx-d98d55878-445cv:/# cd /mnt/
root@nginx-d98d55878-445cv:/mnt# ls
123  aa  bb  data  etc  vshm4707
```



## 八，CFS V3挂载示例

小文件及高并发场景下客户端使用卡顿是已知问题

https://cloud.tencent.com/document/product/582/46359

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cfs-pv
spec:
  mountOptions:
  - vers=3
  - nolock
  - proto=tcp
  - noresvport
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    server: 10.0.0.11 
    path: /mu3qj5m4     
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem


---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: cfs-pv
```



## 九，CFS 动态创建不同子目录的 PVC

参考：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

https://cloud.tencent.com/developer/article/1839599

#### 使用场景

目前TKE 使用 StorageClass 自动创建 CFS 类型 PVC 和 PV，每个 PV 都需要对应一个文件系统（CFS 实例），如果想要多个 PV（不同子路径） 使用同一个文件系统，就需要手动创建 PV 时指定 CFS 文件系统的具体的路径然后绑定 PVC 使用，这是一种办法，但是当需要的 PV 数量多了就会非常繁琐， 对于此使用场景我们可以使用社区的 [nfs-client-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) 项目来实现动态创建 CFS 文件系统中的子路径，接下来我们来介绍下如何在 TKE 中使用`nfs-client-provisioner`

#### 操作步骤

**1，准备CFS实例**

使用cfs-csi插件 创建或者通过cfs 控制台创建，更或者是自建NFS文件系统实例

**2，安装部署**

官方提供两种安装方式， [helm 安装](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner#with-helm) 和 [手动部署 YAML 安装](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner#without-helm)，这里我们提供两种的安装方式,

**3，helm安装**（helm客户端安装可以参考另外文档）安装 helm3 参考 [helm 安装](https://helm.sh/zh/docs/intro/install/)

1，可以根据需求修改指定参数后部署：

```js
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# helm repo list  #查看添加helm repo仓库情况
NAME                            URL                                                               
tcr-chen-helm                   https://tcr-chen.tencentcloudcr.com/chartrepo/helm                
nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# 【可选步骤，可以将helm chart 包下载下来 上传到自己的镜像仓库，方便后续其他集群安装】
# 下载 helm chart 文件至本地目录，查看可以指定的 values 选项（可选）
$ helm pull nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --untar
$ tar  zcvf nfs-subdir-external-provisioner-1.0.0.tgz nfs-subdir-external-provisioner/
$ helm cm-push nfs-subdir-external-provisioner-1.0.0.tgz  tcr-chen-helm
# 默认镜像是国外镜像，可以下载下来上传到自己的镜像仓库里面
$ docker pull k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
$ docker tag 932b0bface75 tcr-chen.tencentcloudcr.com/docker/nfs-subdir-external-provisioner:v4.0.2
$ docker push tcr-chen.tencentcloudcr.com/docker/nfs-subdir-external-provisioner:v4.0.2



# 使用类似下面命令安装 nfs-subdir-external-provisioner 资源
#  helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner  \
--set nfs.server=10.2.2.45  \
--set nfs.path=/tke_path \
--set image.repository=tcr-chen.tencentcloudcr.com/docker/nfs-subdir-external-provisioner \
--set image.tag=v4.0.2

#出现如下提示 表示安装成功
NAME: nfs-subdir-external-provisioner
LAST DEPLOYED: Tue Feb 15 13:22:03 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: Non
```

2，配置使用 CFS 文件系统子目录的 PVC 。

使用上一步部署的`nfs-subdir-external-provisioner`动态创建存储卷。

部署后会默认生成一个存储类资源，默认存储类名是"nfs-client"(也可以在部署时自定义指定)，如下：

```
# kubectl  get sc | grep nfs-client
nfs-client          cluster.local/nfs-subdir-external-provisioner   4m8s
```

3，使用上面的**StorageClass**创建PVC

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```
# kubectl  get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-c0ee60d2-a0b8-4902-a4c1-5145f78e5606   1Mi        RWX            nfs-client     11s
```

![image-20220215133515413](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156083.png)

会自动创建命令，命令命名方式是 namespace-pvc名称-pv名称

4，创建测试POD 挂载对应PVC

```
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox:stable
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

```
# pwd
/localfolder/tke_path/default-test-claim-pvc-c0ee60d2-a0b8-4902-a4c1-5145f78e5606
# ls -lrt
total 0
-rw-r--r-- 1 root root 0 Feb 15 13:47 SUCCESS
```

**4，手动部署 YAML 安装**

1，创建账号ServiceAccount

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

2，部署应用

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: tcr-chen.tencentcloudcr.com/docker/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: cluster.local/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 10.2.2.45
            - name: NFS_PATH
              value: /tke_path
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.2.2.45
            path: /tke_path
```

3，创建storageClass

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

4，创建PVC和POD验证

![image-20220215142033057](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156123.png)



## 十，使用自建NFS挂载 

#### TKE使用自建NFS挂载POD

NFS基本配置：

```
[root@VM-55-9-tlinux ~]# showmount -e 172.16.55.9
Export list for 172.16.55.9:
/data/saas 172.16.0.0/16,10.0.0.0/8
[root@VM-55-9-tlinux ~]# cat /etc/exports
/data/saas  10.0.0.0/8(rw,async,no_root_squash) 
/data/saas  172.16.0.0/16(rw,async,no_root_squash) 

[root@VM-55-9-tlinux ~]# ls -l /data/saas/
total 4
drwxr-xr-x 3 root root 4096 Apr 19 18:42 mysql_docker

[root@VM-55-9-tlinux /data/saas/mysql_docker/data]# ls -lrt
total 0
```

部署yaml文件

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: mysql-nfs
    qcloud-app: mysql-nfs
  name: mysql-nfs
  namespace: cjweichen
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: mysql-nfs
      qcloud-app: mysql-nfs
  serviceName: ""
  template:
    metadata:
      labels:
        k8s-app: mysql-nfs
        qcloud-app: mysql-nfs
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: "12345"
        image: mysql:5.7
        imagePullPolicy: IfNotPresent
        name: mysql
        resources: {}
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: nfs-vol
          subPath: mysql_docker/data
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      volumes:
      - name: nfs-vol
        nfs:
          path: /data/saas
          server: 172.16.55.9

```

![1650376696237](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156172.png)

对应目录下也有相应文件

![1650376734862](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156209.png)



#### EKS使用自建NFS部署mysql

相同nfs客户端

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    k8s-app: mysql-nfs-eks
    qcloud-app: mysql-nfs-eks
  name: mysql-nfs-eks
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: mysql-nfs-eks
      qcloud-app: mysql-nfs-eks
  serviceName: ""
  template:
    metadata:
      labels:
        k8s-app: mysql-nfs-eks
        qcloud-app: mysql-nfs-eks
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          value: "12345"
        image: mysql:5.7
        imagePullPolicy: IfNotPresent
        name: mysql
        resources: {}
        volumeMounts:
        - mountPath: /var/lib/mysql
          name: nfs-vol
          subPath: mysql_docker/data
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      volumes:
      - name: nfs-vol
        nfs:
          path: /data/saas
          server: 172.16.55.9


```





![1650377776203](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156258.png)

![1650377798650](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156296.png)

![1650378336794](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156343.png)



## 十一CFS类型PVC删除策略

 1，**StorageClass** 回收策略是reclaimPolicy: Delete

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cfs
parameters:
  pgroupid: pgroup-jsgxkfjl
  storagetype: SD
  subnetid: subnet-qtlzyxzt
  vers: "3"
  vpcid: vpc-a19fq3f0
  zone: ap-shanghai-2
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

2，使用这个**StorageClass** 自动创建CFS类型pvc

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: v-cjwiechen-cfs
  namespace: cjweichen
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: cfs
  volumeMode: Filesystem
```

会自动创建pv和一个CFS实例

3，创建POD并且挂载，这个时候去删除对应的pvc，会提示该pvc上时候操作的删除，待POD删除时候才删除pvc

如果**StorageClass** 回收策略是reclaimPolicy: Delete，会自动删除pv 和PV对应的 cfs实例的（测试验证过）

4，创建成功收，手动去修改PV的保留策略（控制台方式修改，编辑yaml文件）

![1659195066464](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156391.png)

 4.1，命令方式修改

选择你的 PersistentVolumes 中的一个并更改它的回收策略：  这里的 `` 是你选择的 PersistentVolume 的名字 

```
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

4.2，验证你选择的 PersistentVolume 拥有正确的策略： 如上

![1659195298446](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156437.png)



5，然后去删除pvc，对应的pv不会删除，为Released状态

![1659195593447](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042156478.png)

6，然后删除PV，看下cfs实例是否被删除，验证对应的cfs实例 不会被删除

