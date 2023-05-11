

1， 查看命名空间  发现istio-system 一直处于Terminating 状态。无法删除命名空间！！ 

```
[root@VM-2-45-tlinux ~]# kubectl  get ns -o wide
NAME              STATUS        AGE
default           Active        100d
istio-system      Terminating   64d
kube-node-lease   Active        100d
kube-public       Active        100d
kube-system       Active        100d
prom-ktn7d2ot     Active        84d
```

2，解决方法

 1）查看istio-system的namespace描述 

```
[root@VM-2-45-tlinux ~]# kubectl get ns istio-system -o json > istio-system.json
```

 2）编辑json文件，删除spec字段的内存，因为k8s集群时需要认证的。 

 将finalizers 部分删除

```
    "spec": {
        "finalizers": [
            "kubernetes"
        ]
    },
```

```
"spec": {
    
  },
```



3）（**注意**） 新开一个窗口运行kubectl proxy跑一个API代理在本地的8081端口 

```
[root@VM-2-45-tlinux ~]# kubectl proxy --port=8081
Starting to serve on 127.0.0.1:8081

```

4） 最后运行curl命令进行删除 ：

```
curl -k -H "Content-Type:application/json" -X PUT --data-binary @istio-system.json http://127.0.0.1:8081/api/v1/namespaces/istio-system/finalize
```

5）出现如下表示删除成功

```
[root@VM-2-45-tlinux ~]# curl -k -H "Content-Type:application/json" -X PUT --data-binary @istio-system.json http://127.0.0.1:8081/api/v1/namespaces/istio-system/finalize
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "istio-system",
    "selfLink": "/api/v1/namespaces/istio-system/finalize",
    "uid": "59b8d678-be37-4dbf-952e-8c1051bd85af",
    "resourceVersion": "5817056800",
    "creationTimestamp": "2021-05-15T12:07:05Z",
    "deletionTimestamp": "2021-07-19T08:30:30Z"
  },
  "spec": {
    
  },
  "status": {
    "phase": "Terminating",
    "conditions": [
      {
        "type": "NamespaceDeletionDiscoveryFailure",
        "status": "False",
        "lastTransitionTime": "2021-07-19T08:30:39Z",
        "reason": "ResourcesDiscovered",
        "message": "All resources successfully discovered"
      },
      {
        "type": "NamespaceDeletionGroupVersionParsingFailure",
        "status": "False",
        "lastTransitionTime": "2021-07-19T08:30:39Z",
        "reason": "ParsedGroupVersions",
        "message": "All legacy kube types successfully parsed"
      },
      {
        "type": "NamespaceDeletionContentFailure",
        "status": "False",
        "lastTransitionTime": "2021-07-19T08:30:39Z",
        "reason": "ContentDeleted",
        "message": "All content successfully deleted"
      }
    ]
  }
```

 本文参考链接： https://cloud.tencent.com/developer/article/1678604

https://www.cnblogs.com/zhangmingcheng/p/13987093.html

https://support.huaweicloud.com/cce_faq/cce_faq_00277.html



### EKS集群删除Terminating状态的命名空间

参考文档：https://cloud.tencent.com/developer/article/1678604

**1，开启集群内网或者外网访问**

![image-20211216204149898](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123086.png)

**2，配置kubectl 的kubeconfig文件**

```
vi /root/.kube/config
将控制台获取的KubeConfig文件复制到config文件内

执行kubectl  get nodes 命令确认当前集群
```

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123140.png)



**3，删除Terminating状态的命名空间**

```
[root@chen ~]# kubectl  get ns prom-d5d7opjl -o json > prom-d5d7opjl.json
[root@chen ~]# vi prom-d5d7opjl.json 
```

编辑json文件，删除spec字段的内容

![image-20211216204831803](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123187.png)

更改为：

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123229.png)



新开一个窗口运行kubectl proxy跑一个API代理在本地的8081端口

```
[root@chen ~]#  kubectl proxy --port=8081
```

![image-20211216205003957](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123285.png)

最后运行curl命令进行删除：

```
[root@chen ~]# curl -k -H "Content-Type:application/json" -X PUT --data-binary @prom-d5d7opjl.json   http://127.0.0.1:8081/api/v1/namespaces/prom-d5d7opjl/finalize
```

![image-20211216205106917](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123324.png)



最后确认是否删除：

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123380.png)





### 命名空间因APIService对象访问失败无法删除

#### 问题现象

删除命名空间时，命名空间一直处“删除中”状态，无法删除。查看命名空间yaml配置，status中有报错“DiscoveryFailed”，示例如下：

![点击放大](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123423.png)

上图中报错信息为：Discovery failed for some groups, 1 failing: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: the server is currently unable to handle the request

表示当前删除命名空间动作阻塞在kube-apiserver访问metrics.k8s.io/v1beta1 接口的APIService资源对象。

#### 问题根因

当集群中存在APIService对象时，删除命名空间会先访问APIService对象，若APIService资源无法正常访问，会阻塞命名空间删除。除用户创建的APIService对象资源外，K8S集群部分插件也会自动创建APIService资源，如metrics-server, prometheus插件。

> > ![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042123469.svg)说明：APIService使用介绍请参考：https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/

#### 解决方法

可以采用如下两种方法解决：

- 修复报错信息中的APIService对象，使其能够正常访问，如果是插件中的APIService，请确保插件的Pod正常运行
- 删除报错信息中的APIService对象，如果是插件中的APIService, 可从页面卸载该插件