## 删除Terminating状态的pv

正常删除步骤为：先删pod   再删pvc   最后删pv

平时在处理工作时候，会遇到“Terminating”状态的PV，而且delete不掉。如下图：

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042124727.png)



解决方案：

直接删除k8s中的记录：

```
kubectl patch pv xxx -p '{"metadata":{"finalizers":null}}'
```

参考信息：

This happens when persistent volume is protected. You should be able to cross verify this:

Command:

```
kubectl describe pvc PVC_NAME | grep Finalizers
```

Output:

```
Finalizers: [kubernetes.io/pvc-protection]
```

You can fix this by setting finalizers to null using `kubectl patch`:

```rust
kubectl patch pvc PVC_NAME -p '{"metadata":{"finalizers": []}}' --type=merge
```