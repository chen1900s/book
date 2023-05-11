## 为什么要改变默认存储类？

 取决于安装模式，你的 Kubernetes 集群可能和一个被标记为默认的已有 StorageClass 一起部署。 这个默认的 StorageClass 以后将被用于动态的为没有特定存储类需求的 PersistentVolumeClaims 配置存储 

 预先安装的默认 StorageClass 可能不能很好的适应你期望的工作负载；例如，它配置的存储可能太过昂贵。 如果是这样的话，你可以改变默认 StorageClass，或者完全禁用它以防止动态配置存储 ， 删除默认 StorageClass 可能行不通，因为它可能会被你集群中的扩展管理器自动重建 

## 改变默认 StorageClass

1， 列出你的集群中的 StorageClasses： 

```
kubectl get storageclass

NAME            PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cbs (default)   cloud.tencent.com/qcloud-cbs   Delete          Immediate           false                  22d
cbs-snapclass   com.tencent.cloud.csi.cbs      Delete          Immediate           true                   18d
cbs-zone        com.tencent.cloud.csi.cbs      Delete          Immediate           true                   16d
cfs             com.tencent.cloud.csi.cfs      Delete          Immediate           false                  22d
```

![1628409827100](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042117697.png)

 默认 StorageClass 以 `(default)` 标记。 

2，标记默认 StorageClass 非默认：

默认 StorageClass 的注解 storageclass.beta.kubernetes.io/is-default-class:  设置为 `true`。 注解的其它任意值或者缺省值将被解释为 `false`。

 要标记一个 StorageClass 为非默认的，你需要改变它的值为 `false`： 

```
kubectl patch storageclass cbs -p '{"metadata": {"annotations":{"storageclass.beta.kubernetes.io/is-default-class":"false"}}}'

```

3， 标记一个 StorageClass 为默认的： 

```
kubectl patch storageclass cbs-snapclass -p '{"metadata": {"annotations":{"storageclass.beta.kubernetes.io/is-default-class": "true"}}}'
```

4，验证你选用的 StorageClass 为默认的： 

```
kubectl get storageclass
```

![1628410217004](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042117755.png)



参考链接：https://kubernetes.io/zh/docs/tasks/administer-cluster/change-default-storage-class/

