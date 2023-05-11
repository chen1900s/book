## CBS盘基本使用

由于CBS只支持单机读写，测试如果多个POD使用一个CBS盘调度到不同节点和调度到同一个节点场景

### 案例一：多POD挂载单个CBS PV

**1，先创建一个CBS类型的PVC和PV**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: multpod-onenode
  namespace: storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs-snapclass
  volumeMode: Filesystem

```

**2，部署工作负载，调度在同一个节点上**

![image-20210812124935861](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155143.png)

**3，分别登录POD 到挂载点里面创建文件**

![image-20210812125401627](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155195.png)

**4，查看CBS在节点上的映射关系**

```
[root@VM-2-36-tlinux ~]# df -h | grep /dev
devtmpfs        3.9G     0  3.9G   0% /dev
tmpfs           3.9G   24K  3.9G   1% /dev/shm
/dev/vda1        50G   11G   37G  24% /
/dev/vdb         30G   57M   30G   1% /var/lib/kubelet/plugins/kubernetes.io/qcloud-cbs/mounts/disk-632p551w
/dev/vdc        9.8G   37M  9.7G   1% /var/lib/kubelet/pods/35ad6d05-5220-4a6b-a687-0dfdee0dc742/volumes/kubernetes.io~csi/pvc-384c96ec-91f2-4e49-8918-c1c53ebbb3b2/mount
```

![image-20210812125716434](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155241.png)

**5，封锁当前节点，销毁重建一个POD，让其调度到其他节点上**

```
 kubectl  cordon 192.168.2.36 
 kubectl  delete pod multpod-onenode-679c6bcc84-rxcgp 
```

![image-20210812130349048](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155299.png)

这个时候 就会有一个POD一直是Pending 状态，无法调度

![image-20210812130812736](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155343.png)





### 案例二：CBS类型PVC删除策略

 **StorageClass** 回收策略是reclaimPolicy: Delete

**1，静态创建PV， 不指定StorageClass** 

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cbs-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cbs
    volumeHandle: disk-h2v5jzxy
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem
```

创建PVC时候，也需要不指定 StorageClass 才能选择到这个pv，如果使用指定了会无法选择这个pv（提示是： 当前PersistentVolume与PersistentVolumeClaim所指定的StorageClass不一致 ）

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ""
  volumeMode: Filesystem
  volumeName: cbs-pv
  
```

> > 删除PVC和PV时候，对应的CBS盘不会回收删除



**2，静态创建PV， 指定StorageClass** 

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cbs-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  csi:
    driver: com.tencent.cloud.csi.cbs
    volumeHandle: disk-h2v5jzxy
  persistentVolumeReclaimPolicy: Retain
  storageClassName: cbs
  volumeMode: Filesystem
```

创建PVC，需要选择对应的storageclass

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs
  volumeMode: Filesystem
  volumeName: cbs-pv
```

> > 模拟创建一定大小文件 ： dd if=/dev/zero of=hello.txt bs=100M count=1 
> >
> > 删除PVC和PV时候，对应的CBS盘不会回收删除



**3，动态创建pvc ，不指定pv，会自动创建pv**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs
```

> > 注意：由于cbs **回收策略** 是 Delete ，所以删除pvc时候，对应的PV和CBS盘也会自动删除



 **StorageClass** 回收策略是reclaimPolicy: Retain

**4，动态创建PVC，保留策略**

1）创建reclaimPolicy: Retain 类型的storgeclass 和PVC

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cbs-retain
parameters:
  diskChargeType: POSTPAID_BY_HOUR
  diskType: CLOUD_PREMIUM
provisioner: com.tencent.cloud.csi.cbs
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: cbs-retain
  volumeMode: Filesystem

```

2）然后删除对应的PVC，而对应的PV并没有删除，PV状态是 Released 

```
[root@VM-249-33-tlinux ~]# kubectl   get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM             STORAGECLASS   REASON   AGE
persistentvolume/pvc-ae7a89e4-3a3c-4f87-b0c9-fffaf91559d2   10Gi       RWO            Retain           Released   default/cbs-pvc   cbs-retain              2m47s
```

3）单独删除PV时候，对应的CBS盘也是不会删除的

```
[root@VM-249-33-tlinux ~]# kubectl  delete pv pvc-ae7a89e4-3a3c-4f87-b0c9-fffaf91559d2
persistentvolume "pvc-ae7a89e4-3a3c-4f87-b0c9-fffaf91559d2" deleted
```

![1657345956570](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155408.png)

4）修改对应的PV配置，删除spec.claimRef部分，对应PV可以和其他pvc绑定

![1657344840364](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155449.png)



### 案例二：CBS类型PV挂载目录权限问题

**方案一：**

先把pvc 先挂载到一个非业务使用的目录下，手动chmod -R 修改目录权限为777，然后再 挂载到业务使用的目录

如下，

```
        volumeMounts:
        - mountPath: /mnt/data
          name: vol
```

![1657348794636](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155494.png)

然后再将这块数据盘挂载到应用目录，查看目录权限是777

```
        volumeMounts:
        - mountPath: /var/lib/grafana/
          name: vol
```

![1657349124914](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155530.png)



**方案二：使用init 容器修改对应目录权限**

业务容器可以使用普通用户启动服务，如果在启动过程中遇到您的普通用户对某些目录没有操作权限，这个时候您可以使用Init 容器去修改你对应目录的权限，达到普通用户能够读写，例如如下方式（示例，具体根据业务进行修改）

init 容器 使用可以参考：https://kubernetes.io/zh/docs/concepts/workloads/pods/init-containers/

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155568.png)

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042155615.png)