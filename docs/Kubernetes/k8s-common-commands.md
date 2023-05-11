> 仅限腾讯云TKE集群中使用，其他容器集群或者自建仅参考，依赖于环境

### 1，节点相关

```
表格输出各节点占用的 podCIDR

kubectl get no -o=custom-columns=INTERNAL-IP:.metadata.name,EXTERNAL-IP:.status.addresses[1].address,CIDR:.spec.podCIDR
INTERNAL-IP             EXTERNAL-IP     CIDR
172.30.2.5              139.186.202.9   172.16.0.0/26

表格输出各节点总可用资源 (Allocatable)
kubectl get no -o=custom-columns="NODE:.metadata.name,ALLOCATABLE CPU:.status.allocatable.cpu,ALLOCATABLE MEMORY:.status.allocatable.memory"
NODE                    ALLOCATABLE CPU   ALLOCATABLE MEMORY
172.30.2.5              1930m             1347064Ki

输出各节点已分配资源的情况
kubectl get nodes --no-headers | awk '{print $1}' | xargs -I {} sh -c "echo {} ; kubectl describe node {} | grep Allocated -A 5 | grep -ve Event -ve Allocated -ve percent -ve --;"
```

### 2，查询所有节点挂载卷信息

```bash
kubectl  get node  -ocustom-columns='节点名称:.metadata.name,容器网段:.spec.podCIDR,eni-ip配额:.status.capacity."tke\.cloud\.tencent\.com\/eni-ip",挂卷:.status.volumesInUse'
```

### 3，查询所有节点IP和实例ID

```bash
##
kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点IP:节点IP.status.addresses[0].address,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id",providerID:.spec.providerID
##查看节点的内核版本
kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点内核:.status.nodeInfo.kernelVersion,节点UUID:.status.nodeInfo.machineID,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id"
##全部信息
kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点IP:.status.addresses[0].address,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id",providerID:.spec.providerID,节点内核:.status.nodeInfo.kernelVersion,节点UUID:.status.nodeInfo.machineID,可用区:.metadata.labels."failure-domain\.beta\.kubernetes\.io\/zone"
```

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name} {.status.addresses[?(@.type=="ExternalIP")].address}{"\n"}'
#获取节点IP：
kubectl get node -ocustom-columns=name:.metadata.name,IP:status.addresses[0].address
```

总结如上命令

```bash
kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点IP:.status.addresses[0].address,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id",providerID:.spec.providerID,节点内核:.status.nodeInfo.kernelVersion,节点UUID:.status.nodeInfo.machineID,可用区:.metadata.labels."failure-domain\.beta\.kubernetes\.io\/zone"
```

查看容量：

```bash
 kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点IP:.status.addresses[0].address,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id",providerID:.spec.providerID,节点内核:.status.nodeInfo.kernelVersion,Allocatable:.status.allocatable."nvidia\.com\/gpu"


kubectl  get node  -ocustom-columns=节点名称:.metadata.name,节点IP:.status.addresses[0].address,实例ID:.metadata.labels."cloud\.tencent\.com\/node-instance-id",providerID:.spec.providerID,节点内核:.status.nodeInfo.kernelVersion,GPU-Allocatable:.status.allocatable."nvidia\.com\/gpu",CPU-Allocatable:.status.allocatable.cpu,MEM-Allocatable:.status.allocatable.memory,eniIP-Allocatable:.status.allocatable."tke\.cloud\.tencent\.com\/eni-ip"
```

查看节点当前所在的节点池

```bash
kubectl get node -ocustom-columns='Name:.metadata.name,节点池:.metadata.labels.tke\.cloud\.tencent\.com\/nodepool-id,伸缩组:.metadata.labels.cloud\.tencent\.com\/auto-scaling-group-id,createTime:.metadata.creationTimestamp'
```



### 4，查询所有节点eni-ip信息

```bash
#查下辅助网卡eniInfo详细信息
kubectl  get nec -ocustom-columns=Name:.metadata.name,cvmID:.spec.providerID,eni:.status.eniInfos,ip:.spec.maxIPPerENI

#查看集群节点vpc-cni配额
kubectl get node -ocustom-columns=Name:.metadata.name,capcIP:.status.capacity."tke\.cloud\.tencent\.com/eni-ip",allocIP:.status.allocatable."tke\.cloud\.tencent\.com/eni-ip"

#查询VIP和VIPC绑定关系
kubectl  get vip  -ocustom-columns=Name:.metadata.name,vipcNamespace:.spec.claimRef.namespace,type:spec.type,vipcName:.spec.claimRef.name
```

### 5，批量删除驱逐状态POD

查寻确认没问题后再做删除，需要把NameSpace替换成用户的命名空间名称

```bash
kubectl get pods -n NameSpace |grep  Evicted
然后是批量删除
kubectl get pods -n NameSpace | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n NameSpace


#所有命名空间
kubectl get pods --all-namespaces |grep  Evicted
然后是批量删除
kubectl get pods --all-namespaces  | grep Evicted | awk '{print $1}' | xargs kubectl delete pod --all-namespaces

```

### 6，批量删除UnexpectedAdmissionError状态pod 

```sh
#指定命名空间去删除
kubectl get pods  -n  NAMESPACE   | grep  UnexpectedAdmissionError | awk '{print $1}'  | xargs kubectl delete pod -n NAMESPACE
```

### 7，批量查看集群中LB类型的service信息

```bash
#可以打印的第一行，然后匹配的其他行
kubectl  get svc -ocustom-columns=namespace:.metadata.namespace,serviceName:.metadata.name,type:.spec.type,lbID:.metadata.annotations."service\.kubernetes\.io\/loadbalance-id",subnet:.metadata.annotations."service\.kubernetes\.io\/qcloud-loadbalancer-internal-subnetid" -A | grep -E  "serviceName|LoadBalancer"


#查看LBid & VIP信息
kubectl  get svc -ocustom-columns=namespace:.metadata.namespace,serviceName:.metadata.name,type:.spec.type,lbID:.metadata.annotations."service\.kubernetes\.io\/loadbalance-id",subnet:.metadata.annotations."service\.kubernetes\.io\/qcloud-loadbalancer-internal-subnetid",vip:.status.loadBalancer.ingress[0].ip -A | grep -E  "serviceName|LoadBalancer"

```

### 8，批量查看集群中ingress信息

```bash
kubectl  get ingress -ocustom-columns=namespace:.metadata.namespace,ingressName:.metadata.name,ingressType:.metadata.annotations."kubernetes\.io\/ingress\.class",lbID:.metadata.annotations."kubernetes\.io\/ingress\.qcloud-loadbalance-id",vip:.status.loadBalancer.ingress[0].ip,uuid:.metadata.uid  -A
```

### 9，批量查询PV和PVC 绑定关系

```bash
 kubectl  get pv  -ocustom-columns='pvName:.metadata.name,storageClassName:.spec.storageClassName,pvcNamespace:.spec.claimRef.namespace,pvcName:.spec.claimRef.name,volumeHandle:.spec.csi.volumeHandle,driver:.spec.csi.driver'
 
 批量查看CBS盘信息
 kubectl  get pv  -ocustom-columns='pvName:.metadata.name,storageClassName:.spec.storageClassName,pvcNamespace:.spec.claimRef.namespace,pvcName:.spec.claimRef.name,volumeHandle:.spec.csi.volumeHandle,driver:.spec.csi.driver,CBSid:.spec.qcloudCbs.cbsDiskId'

```

### 10，显示Pod的容器镜像

```bash
kubectl get pods -o custom-columns='NAME:metadata.name,IMAGES:spec.containers[*].image'


kubectl  get deployments -o custom-columns='DeploymentName:metadata.name,Namespace:.metadata.namespace,image:.spec.template.spec.containers[*].image' --all-namespaces

kubectl  get statefulset  -o custom-columns='DeploymentName:metadata.name,Namespace:.metadata.namespace,image:.spec.template.spec.containers[*].image' --all-namespaces

kubectl get pods -o custom-columns='NAME:metadata.name,security-group-id:.metadata.annotations.eks\.tke\.cloud\.tencent\.com\/security-group-id'

kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}'
```

### 11，批量查询POD的里面容器ID

```bash
kubectl get pods -o custom-columns='podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[0].containerID'

#kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[*].containerID


#kubectl get pods -o custom-columns=podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[0].containerID | awk -F 'docker://' '{print $1,$2}'

#kubectl get pods -o custom-columns='podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeName:.spec.nodeName,nodeIP:.status.hostIP,containerID:.status.containerStatuses[0].containerID' | awk -F 'docker://|| containerd://'  '{print $1,$2}'

##批量查下POD的 POD_ID 可以对照查看对应日志命令文件
kubectl get pods -o custom-columns='Namespace:..metadata.namespace,podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,nodeIP:.status.hostIP,Pod_ID:.metadata.uid,ContainerName:.spec.containers[*].name'


查看POD创建时间：
kubectl get pods -o custom-columns='podName:.metadata.name,podIP:.status.podIP,podStatus:.status.phase,startTime:.status.startTime'
```

### 12，批量查看容器ID

```bash
docker ps | awk '{print $1}' | xargs -n 1 -I {} -- sh -c "echo {} && docker inspect {} |grep 27c0e22fc5ffa8aa540b68182672ea61fd2bb701740926025ba77a66b4428341 -C 1"
```

### 13，EKS获取所有EIP列表

```bash
kubectl get pods -A -o=jsonpath='{range .items[*]}{"pod_name:"}{.metadata.name}{"\teip-id:"}{.metadata.annotations.tke\.cloud\.tencent\.com\/eip-id}{"\teip-ip:"}{.metadata.annotations.tke\.cloud\.tencent\.com\/eip\-public\-ip}{"\n"}{end}'


或者使用这个命令
kubectl  get  pod -ocustom-columns=pod-name:.metadata.name,eip-id:.metadata.annotations."tke\.cloud\.tencent\.com\/eip\-id",eip-public-ip:.metadata.annotations."tke\.cloud\.tencent\.com\/eip\-public\-ip"
```

### 14，批量查看集群里面有那些RS使用了 csi inline模式

```bash
kubectl  get rs -o custom-columns=ReplicaSetsName:.metadata.name,CurrentPOD:.spec.replicas,CBSid:.spec.template.spec.volumes[*].qcloudCbs.cbsDiskId  -A

#查看当前有哪些POD在使用csi inline

kubectl  get pods -o custom-columns=Namespace:.metadata.namespace,PodName:.metadata.name,Status:.status.phase,CBSid:.spec.volumes[*].qcloudCbs.cbsDiskId

#通过这个简单查询下 有那些rs使用了
kubectl  get rs -o custom-columns=namespace:.metadata.namespace,ReplicaSetsName:.metadata.name,CurrentPOD:.spec.replicas,CBSid:.spec.template.spec.volumes[*].qcloudCbs.cbsDiskId -A  |  grep disk

查看 StatefulSet类型是否使用csi inline
kubectl  get sts -o custom-columns=namespace:.metadata.namespace,StatefulSetName:.metadata.name,CBSid:.spec.template.spec.volumes[*].qcloudCbs.cbsDiskId -A    | grep disk


```

### 15，批量查看pod资源设置情况limit和request

```bash

#调整显示顺序
kubectl  get pod  -o custom-columns='namespace:.metadata.namespace,PodName:.metadata.name,ContainerName:.spec.containers[*].name,CPURequest:.spec.containers[*].resources.requests.cpu,CPULimit:.spec.containers[*].resources.limits.cpu,MemRequest:.spec.containers[*].resources.requests.memory,MemLimit:.spec.containers[*].resources.limits.memory,Node:.status.hostIP'


kubectl get pod -n kube-system  -o=custom-columns='NAME:.metadata.name,NAMESPACE:.metadata.namespace,PHASE:.status.phase,Request-cpu:.spec.containers\[0\].resources.requests.cpu,Request-memory:.spec.containers\[0\].resources.requests.memory,Limit-cpu:.spec.containers\[0\].resources.limits.cpu,Limit-memory:.spec.containers\[0\].resources.limits.memory,Node:.status.hostIP'

```

### 16，批量查看集群容器网段

```bash
 kubectl get node -ocustom-columns='Name:.metadata.name,cidr:.spec.podCIDR,annotations:.metadata.annotations."tke\.cloud\.tencent\.com\/pod-cidrs"'
```

### 17，批量查看日志采集配置

```bash
kubectl get  logconfig  -ocustom-columns='Name:.metadata.name,ClsTopic:.spec.clsDetail.topicId,ClsType:.spec.inputDetail.type'

查看采集对象：
kubectl get  logconfig  -ocustom-columns='Name:.metadata.name,ClsTopic:.spec.clsDetail.topicId,ClsType:.spec.inputDetail.type,Namespace:.spec.inputDetail.containerStdout.workloads[*].namespace,Workloads:.spec.inputDetail.containerStdout.workloads[*].name'
```

### 18，批量删除多个job

```bash
#删除成功的作业: 
kubectl delete jobs --field-selector status.successful=1 
#删除失败或长时间运行的作业 
kubectl delete jobs --field-selector status.successful=0

```

### 19，批量查看集群有没有POD使用GPU

```bash
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' | while read line; do pod=$(echo $line | awk '{print $1}'); node=$(echo $line | awk '{print $2}'); echo -n "${pod} "; kubectl describe pod $pod | grep -q "nvidia.com/gpu" && echo "is using GPU on ${node}" || echo "is not using GPU"; done
```

































































































































































































