## 操作场景

Jenkins 是连接持续集成和持续交付的桥梁，采用 Jenkins Master/Slave pod 架构能够解决企业批量构建并发限制的痛点，实现和落地真正意义上持续集成。本文介绍了如何在腾讯云容器服务（TKE）中使用 Jenkins，以实现业务快速可持续性交付，减少资源及人力成本。

## 工作原理

本文采用基于 TKE 的 Jenkins 外网架构，即 Jenkins Master 在 TKE 集群外，slave pod 在集群内。该外网架构图如下所示：

![img](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119442.png)

- Jenkins Master 、TKE 集群位于同一 VPC 网络下。
- Jenkins Master 在 TKE 集群外，slave pod 在 TKE 集群的 node 节点上。
- 用户提交代码到 Gitlab，触发 Jenkins Master 调用 slave pod 进行构建打包并推送镜像到 TKE 镜像仓库，TKE 集群拉取镜像并触发滚动更新进行 Pod 部署。
- 多 slave pod 构建可满足批量并发构建的需求。

## 操作环境

本节介绍了该场景中的具体环境，如下：

### TKE 集群

| 角色         | Kubernetes 版本                 | 操作系统        |
| :----------- | :------------------------------ | :-------------- |
| TKE 托管集群 | 1.1.8  cls-32qxo736（重庆地域） | tlinux2.4x86_64 |

### Jenkins 配置

| 角色                    | 版本          |
| :---------------------- | :------------ |
| Jenins Master           | Jenkins 2.315 |
| Jenkins Kubernetes 插件 | 1.21.3        |

### 节点

| 角色           | 内网 IP      | 外网IP          | 操作系统        | CPU  | 内存 | 带宽   |
| :------------- | :----------- | --------------- | :-------------- | :--- | :--- | :----- |
| Jenkins Master | 192.168.2.42 | 139.186.160.196 | tlinux2.4x86_64 | 4核  | 8GB  | 10Mbps |
| Node           | 192.168.0.17 | 139.186.152.16  | tlinux2.4x86_64 | 4核  | 8GB  | 10Mbps |

## 注意事项

- 确保与 TKE 集群同 VPC 下已具备 Jenkins Master 节点，并且该节点已安装 Git。
- 确保操作步骤中用到的 gitlab 代码仓库里面已包含 Dockerfile 文件。





## 步骤1：TKE 集群配置

获取配置 Jenkins 时所需的集群访问地址、token 及集群 CA 证书信息

### 获取集群凭证

TKE集群开启RBAC控制后如何获取集群token

https://cloud.tencent.com/developer/article/1762567

**1，创建serviceAccount**

```
[root@VM-0-17-tlinux ~/jenkins]# kubectl create sa tke-admin
serviceaccount/tke-admin created
```

**2，为serviceAccount绑定集群角色**

```
[root@VM-0-17-tlinux ~/jenkins]# kubectl create clusterrolebinding tke-admin-binding --clusterrole=tke:admin --serviceaccount=default:tke-admin
clusterrolebinding.rbac.authorization.k8s.io/tke-admin-binding created
```

**3，获取serviceAccount对应的token**

```
[root@VM-0-17-tlinux ~/jenkins]# kubectl get sa tke-admin -o=jsonpath='{.secrets[0].name}'
tke-admin-token-7bvqr
[root@VM-0-17-tlinux ~/jenkins]# kubectl get secret  tke-admin-token-7bvqr  -o=jsonpath='{.data.token}' | base64 -d

eyJhbGciOiJSUzI1NiIsImtpZCI6IkVqUDlwYm5CMDBQSTJtclc2NzRZZV9ucnVwcy1FRkdvdHVSOW9rWlB3SGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InRrZS1hZG1pbi10b2tlbi03YnZxciIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJ0a2UtYWRtaW4iLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwMjg4M2I2Ny05NGNhLTRhZDUtOTBlNC01YzYyZjQ1MjQ0YzkiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDp0a2UtYWRtaW4ifQ.zh4kjnUqWTr9-MINcWvRnbbBbY-3aJCAuSzZgypY8tro-MTQymdjST1PdsmmqVbGnLaqErxcINaPTveOpebk3RjpgIoQ324UTolSVCf7b-TpuVY1Ixw2HjYeoPkXGkbyxO-1OAU2CoLItC8SK5pIjgfWtv8ogaQn8aDah8UzcS4KbUkumm28oFz9EAzcnoGbEjvJJqSGMF5ENiHQbdspeUdT2jkdAaM5dK6E5hGwAIEr5vBdlZO7NfuYyJpUMDrfiwaJg-ynYLPpk3Np4cy0CZItBUSSMnLsoRKsGtq1r0oH3awgZ9Fg8
```

通过上命令获取sa的token，然后进行base64解密就是你可以使用的token了

### 获取集群 CA 证书

1，登录目标集群的 node 节点，执行以下命令，查看集群 CA 证书，记录并保存查询所得证书信息

```
[root@VM-0-17-tlinux ~/jenkins]# cat /etc/kubernetes/cluster-ca.crt
```

![image-20211010170136258](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119495.png)

### 授权 docker.sock

TKE 集群中的每个 node 节点系统里都有一个 `docker.sock` 文件，slave pod 在执行 `docker build` 时将会连接该文件。在此之前，需逐个登录到每个节点上，依次执行以下命令对 `docker build` 进行授权

```
[root@VM-0-17-tlinux ~/jenkins]# ls -l /var/run/docker.sock 
srw-rw---- 1 root root 0 Sep 27 18:14 /var/run/docker.sock
[root@VM-0-17-tlinux ~/jenkins]# chmod 666 /var/run/docker.sock
[root@VM-0-17-tlinux ~/jenkins]# ls -l /var/run/docker.sock 
srw-rw-rw- 1 root root 0 Sep 27 18:14 /var/run/docker.sock
```





## 步骤2：Jenkins 安装和配置（CVM方式）

### 安装参考安装文档

【实用技巧】Jenkins 安装和配置.md

### 添加 TKE 内网访问地址

登录 Jenkins Master 节点，执行以下命令，配置访问域名。

```
#该命令可在集群开启内网访问后，从集群基本信息页面中的“集群APIServer” 中获取
sudo sed -i '$a 192.168.2.28 cls-32qxo736.ccs.tencent-cloud.com' /etc/hosts
```

### Jenkins 安装必备插件

- **Locale**：汉化语言插件，安装该插件可使 Jenkins 界面默认设置为中文版。
- **Kubernetes**：Kubernetes-plugin 插件。
- **Git Parameter** 和 **Extended Choice Parameter**：用于构建打包时传参。

### 开启 jnlp 端口

【系统管理】>【全局安全配置】>设置入站代理的 TCP 端口为“指定端口 50000”

![image-20211010194447573](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119546.png)

### 添加 TKE 集群 token

【用户列表】>【admin】>【凭据】>【全局凭据 (unrestricted)】

![image-20211010195240569](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119596.png)

### 添加 gitlab 认证

![image-20211010195659985](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119633.png)

### 配置 slave pod 模板

【系统管理】>【系统配置】>【新增一个云】>【Kubernetes】

![image-20211010195823471](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119670.png)

![image-20211010195837740](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119715.png)

![image-20211010200145031](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119761.png)

![image-20211010200635349](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119800.png)



![image-20211010220112175](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119842.png)



![image-20211010220147218](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119918.png)

![image-20211010220239576](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042119966.png)

