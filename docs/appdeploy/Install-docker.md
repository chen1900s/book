# <center>CentOS Docker 安装

##### 1，使用 Docker 仓库进行安装

设置仓库

安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2

```
 sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

阿里源：

```
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

清华源：

```
sudo yum-config-manager \
    --add-repo \
    https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
```

安装 Docker Engine-Community：

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

##### 2，安装docker 二进制安装

二级制包下载地址：https://download.docker.com/linux/static/stable/x86_64/

```
tar -zxvf docker-18.09.6.tgz

mv docker/* /usr/bin/
```

```
vi /usr/lib/systemd/system/docker.service

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFLE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

##### 3，配置镜像加速源

```
vi /etc/docker/daemon.json
{
"registry-mirrors": [
  "https://mirror.ccs.tencentyun.com"
]
}
```

更多镜像加速源可以参考：https://www.runoob.com/docker/docker-mirror-acceleration.html

启动

```
systemctl  start docker
systemctl   enable docker
systemctl  status docker
```

 