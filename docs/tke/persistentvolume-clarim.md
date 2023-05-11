# 更改 PersistentVolume 的回收策略

 本文展示了如何更改 Kubernetes PersistentVolume 的回收策略 

## 为什么要更改 PersistentVolume 的回收策略

​     PersistentVolumes 可以有多种回收策略，包括 "Retain"、"Recycle" 和 "Delete"。 对于动态配置的 PersistentVolumes 来说，默认回收策略为 "Delete"。 这表示当用户删除对应的 PersistentVolumeClaim 时，动态配置的 volume 将被自动删除。 如果 volume 包含重要数据时，这种自动行为可能是不合适的。 那种情况下，更适合使用 "Retain" 策略。 使用 "Retain" 时，如果用户删除 PersistentVolumeClaim，对应的 PersistentVolume 不会被删除。 相反，它将变为 Released 状态，表示所有的数据可以被手动恢复 

## 更改 PersistentVolume 的回收策略

 1，列出你集群中的 PersistentVolumes 

```
 kubectl  get pv 
```

![1628411191819](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042118633.png)

 这个列表同样包含了绑定到每个卷的 claims 名称，以便更容易的识别动态配置的卷。 

 2，选择你的 PersistentVolumes 中的一个并更改它的回收策略：  这里的 `` 是你选择的 PersistentVolume 的名字 

```
kubectl patch pv <your-pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```



3，验证你选择的 PersistentVolume 拥有正确的策略： 如上

![1628411299123](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042118686.png)

4，也可以通过edit方式修改

```
kubectl  edit pv pvc-c1b6f3b9-34b6-4958-b3ef-963b3771686b 
```

![1628411506589](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042118730.png)



参考文档：https://kubernetes.io/zh/docs/tasks/administer-cluster/change-pv-reclaim-policy/