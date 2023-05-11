---
title: Kubernetes中的容器日志路径目录
abbrlink: c7ce75fa
date: 2023-04-18 14:00:21
tags:
---

## 容器日志文件路径

本文大概介绍kubernetes集群中POD容器日志存储位置及把运行日志记录至文件 ，主要包括标准输出日志，容器内文件日志，以及其他持久卷存储位置路径等，仅供参考

> 环境：
>
> 1，TKE集群
>
> 2，容器目录和kubelet目录都是默认配置
>
> 声明：以下仅个人经验总结，如果有不对的地方勿喷

## 标准输出

各种容器运行时都提供了对容器标准输出的处理。腾讯云TKE目前支持两种容器运行时——**docker**和**containerd**。

docker:

> 真实路径：/var/lib/docker/containers/$CONTAINERID
> 软连接：kubelet会在/var/log/pods和/var/log/containers创建软连接指向/var/lib/docker/containers/$CONTAINERID
> 配置文件：/etc/docker/daemon.json
> 参数：
> "log-driver": "json-file",
> "log-level": "warn",
> "log-opts": {
>  "max-file": "10",
>  "max-size": "100m"
> },

containerd :

> 日志存储路径：
> 真实路径：/var/log/pods/$CONTAINER_NAMES
> 软连接：同时kubelet也会在/var/log/containers目录下创建软链接指向    /var/log/pods/$CONTAINER_NAMES
> 日志配置参数：
> 配置文件：/etc/kubernetes/kubelet
> kubelet配置参数:
> –container-log-max-files=5 \
> –container-log-max-size=“100Mi” \
> –logging-format=“json” \



### docker运行时标准输出日志

如果Docker作为K8S容器运行时，容器日志的落盘将由docker来完成，处理的方式是logging driver，docker支持多种logging drivers，可以将日志以特定的格式输出到不同的目标，保存在/var/lib/docker/containers/<container-id> 目录下。Kubelet 会在 /var/log/pods 和 /var/log/containers 下面建立软链接，指向 /var/lib/docker/containers/<container-id> 该目录下的容器日志文件

TKE docker节点使用的logging driver是json-file，会将容器的**标准输出**以 json 的格式写到宿主机的文件里，文件路径为：

```bash
/var/lib/docker/containers/<container-id>/<container-id>-json.log  #其中容器ID可以通过下面命令查询
```

批量查下集群里面容器conatiner ID：

```bash
kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[0].containerID | awk -F 'docker://|| containerd://'  '{print $1,$2}'
```

### containerd运行时标准输出日志

如果Containerd作为K8 容器运行时， 容器标准输出日志的落盘由 Kubelet 来完成，保存至 /var/log/pods 目录下，同时在 /var/log/containers 目录下创建软链接，指向日志文件

文件路径为：

```
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/<container-name>/0.log
```

![1668063911037](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111839189.png)

批量查询集群POD的uid

```bash
kubectl get pods -o custom-columns=Namespace:..metadata.namespace,podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeIP:.status.hostIP,Pod_ID:.metadata.uid,ContainerName:.spec.containers[*].name
```



### kubelet处理运行时日志

无论使用那种容器运行时，kubelet都会在目录“/var/log/containers”和“/var/log/pods”下创建指向容器标准输出日志文件（目录）的软链接，文件路径如下：

```
/var/log/containers/<pod_name>_<pod_namespace>_<container_name>-<container_id>.log

/var/log/pods/<namespace>_<pod_name>_<pod_uid>/<container_name>/0.log
```

![1659871831763](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111839400.png)



## 容器文件路径

容器中的文件路径，是相对于容器的文件系统而言。容器中的文件日志也是落在宿主机node节点上的，可以通过宿主机目录查看到对应文件信息

### 没有挂载数据卷的目录

在这种情况下，可以根据storage driver（例如：aufs、overlay2）以及容器运行时（docker、containerd），查看容器根目录在宿主机上的挂载点，再根据日志文件在容器文件系统中的路径，找到日志文件在宿主机上的位置。 目前TKE节点主要安装的操作系统是ubuntu、CentOS和tlinux，使用的storage driver为 aufs或者overlay2。 获取容器根目录挂载点的方法如下：

#### docker运行时

|          | 容器根目录在宿主机的挂载点           | 如何查看id                                                   |
| :------- | :----------------------------------- | :----------------------------------------------------------- |
| aufs     | /var/lib/docker/aufs/mnt/<id>        | cat /var/lib/docker/image/aufs/layerdb/mounts/<container-id>/mount-id |
| overlay2 | /var/lib/docker/overlay2/<id>/merged | cat /var/lib/docker/image/overlay2/layerdb/mounts/<container-id>/mount-id |

查询示例：

```bash
[root@VM-249-47-tlinux ~]# kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[0].containerID | awk -F 'docker://|| containerd://'  '{print $1,$2}'   | grep nginx
nginx-0                         10.200.0.10   Running     172.30.249.47   172.30.249.47    a0d96814caa6248996dcbcea552fa5f405143500f417a764160010018411f368    #容器ID

[root@VM-249-47-tlinux ~]# cat /var/lib/docker/image/overlay2/layerdb/mounts/a0d96814caa6248996dcbcea552fa5f405143500f417a764160010018411f368/mount-id
dfdb4b151beca3e68047e748f2832a5acd04161beb4088ca454e7bdf0bdd7aa4

[root@VM-249-47-tlinux ~]# cd /var/lib/docker/overlay2/dfdb4b151beca3e68047e748f2832a5acd04161beb4088ca454e7bdf0bdd7aa4/merged
[root@VM-249-47-tlinux /var/lib/docker/overlay2/dfdb4b151beca3e68047e748f2832a5acd04161beb4088ca454e7bdf0bdd7aa4/merged]# ls 
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

![1659872541684](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111839568.png)

容器的根目录在宿主机上的挂载点

![1659873211290](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111839396.png)

####  容器的存储ID找到对应的容器ID和镜像ID 

1，进入overlay2目录

```bash
cd /var/lib/docker/overlay2

如果遇到日志处理时候可以查看1G以上的文件大小：du -sh * |grep ‘G’ | sort
```

2，查看占用空间的pid，以及对应的容器名称

```bash
docker ps -q | xargs docker inspect --format ‘{{.State.Pid}},{{.Name}},{{.GraphDriver.Data.WorkDir}}’ | grep 4d79558c06c01e7b078cf1829676a06f615371953b15b104cb83d94d4f9c2c43

如果运行的容器没有，可以在镜像里面查下
docker inspect -f $'{{.RepoTags}}\t{{.GraphDriver.Data.LowerDir}}' $(docker images -q)  | grep 4d79558c06c01e7b078cf1829676a06f615371953b15b104cb83d94d4f9c2c43
```



#### containerd运行时

```bash
/run/containerd/io.containerd.runtime.v1.linux/k8s.io/<container-id>/rootfs

/run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/rootfs  #最新containerd运行时目录
```

可以看到，容器根目录在宿主机的挂在点取决于容器的标识（id或者container-id）。

>  在默认配置/etc/containerd/config.toml 中还有两个关于存储的配置路径 :
>
>  root = "/var/lib/containerd" state = "/run/containerd" 
>
>  其中 root 是用来保存持久化数据，包括 Snapshots, Content, Metadata 以及各种插件的数据，每一个插件都有自己单独的目录，Containerd 本身不存储任何数据，它的所有功能都来自于已加载的插件。
>
>  而另外的 state 是用来保存运行时的临时数据的，包括 sockets、pid、挂载点、运行时状态以及不需要持久化的插件数据。

则容器里面的文件日志路径，可以通过宿主机的挂在点获取到，

| 容器中路径           | 宿主机路径(docker + overlay2)                                |
| :------------------- | :----------------------------------------------------------- |
| /data/logs/nginx.log | /run/containerd/io.containerd.runtime.v2.task/k8s.io/0b895cd3a0fb017337df1796530904b0185db7469f442b9413333f4c8c878fee/rootfs/data/logs/test.log |



### 挂载数据卷的目录日志

TKE支持以下几种挂载卷类型

#### hostpath

hostpath的处理比较简单，直接查看POD所在节点的挂载路径日志

#### emptyDir

emptyDir卷在宿主机上的路径为:

kubelet root-dir的默认值：/var/lib/kubelet

```bash
<kubelet root-dir>/pods/<pod uid>/volumes/kubernetes.io~empty-dir/<volume name>
```

#### 腾讯云CBS/CFS

cbs持久化卷在宿主机上的路径为：

```bash
#<kubelet root-dir>/pods/<pod uid>/volumes/kubernetes.io~qcloud-cbs/<volume name>/mount  #qcloud-cbs类型的CBS

#container节点和docker运行节点一样
# <kubelet root-dir>/plugins/kubernetes.io/csi/pv/<pvc-name>/globalmount    #cbs-csi类型的CBS
/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-a8c018b1-6e93-41e2-b620-9585d46d3619/globalmount
```

![image-20221211194126209](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111941327.png)

![image-20221211195129123](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212111951177.png)

cfs持久化卷在宿主机上的路径为：

```bash
#<kubelet root-dir>/pods/<pod uid>/volumes/kubernetes.io~csi/<pv name>/mount

/var/lib/kubelet/pods/4f65675c-ead4-4d75-b1b9-89784a332cf4/volumes/kubernetes.io~csi/pvc-24b1a5c5-f59a-4c5e-8ea8-bad84bacf9df/mount
```

![image-20221211200303472](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212112003529.png)

![image-20221211200418423](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202212112004478.png)

> 验证，如果是一个CFS类型的PVC挂载不同的POD上，节点上只有一个挂载点，
