 

### 安装 CRI 客户端 crictl

```
# https://github.com/kubernetes-sigs/cri-tools/releases/ 选择版本
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz
sudo tar crictl-v1.24.2-linux-amd64.tar.gz   -C /usr/local/bin

vi /etc/crictl.yaml 
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false

# 验证是否可用
crictl  pull  centos:latest
crictl  image
crictl   rmi  centos:latest

```



```
containerd节点如何使用密码拉取镜像
crictl pull --creds USERNAME[:PASSWORD]  镜像地址

示例：
[root@VM-155-12-tlinux ~]#crictl pull --creds  100006305462:chen188289  ccr.ccs.tencentyun.com/chenjingwei/goproxy:latest
Image is up to date for sha256:ca30c529755f98c53f660f86fe42f1b38f19ca6127c57aa6025afaf9a016742a
```

