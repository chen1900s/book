# 容器数据磁盘被写满

## 造成的危害

- 不能创建 Pod (一直 ContainerCreating)
- 不能删除 Pod (一直 Terminating)

## 判断是否被写满

容器数据目录大多会单独挂数据盘，路径一般是 `/var/lib/docker`，也可能是 `/data/docker` 或 `/opt/docker`，取决于节点被添加时的配置：

![图片描述](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141801.png)

可通过 `docker info` 确定：

```bash
$ docker info
...
Docker Root Dir: /var/lib/docker
...
```

如果没有单独挂数据盘，则会使用系统盘存储。判断是否被写满：

```bash
$ df
Filesystem     1K-blocks     Used Available Use% Mounted on
...
/dev/vda1       51474044  4619112  44233548  10% /
...
/dev/vdb        20511356 20511356         0 100% /var/lib/docker
```

## 解决方法

### 先恢复业务，清理磁盘空间

重启 dockerd (清理容器日志输出和可写层文件)

- 重启前需要稍微腾出一点空间，不然重启 docker 会失败，可以手动删除一些docker的log文件或可写层文件，通常删除log:

  ```bash
  $ cd /var/lib/docker/containers
  $ du -sh * # 找到比较大的目录
  $ cd dda02c9a7491fa797ab730c1568ba06cba74cecd4e4a82e9d90d00fa11de743c
  $ cat /dev/null > dda02c9a7491fa797ab730c1568ba06cba74cecd4e4a82e9d90d00fa11de743c-json.log.9 # 删除log文件
  ```

  注意:

  使用

  ```
  cat /dev/null > 式删除而不用
  ```

  ```
  rm 因为用 rm 删除的文件，docker 进程可能不会释放文件，空间也就不会释放；log 的后缀数字越大表示越久远，先删除旧日志。
  ```

- 将该 node 标记不可调度，并将其已有的 pod 驱逐到其它节点，这样重启dockerd就会让该节点的pod对应的容器删掉，容器相关的日志(标准输出)与容器内产生的数据文件(可写层)也会被清理：

  ```bash
  kubectl drain 10.179.80.31
  ```

- 重启 dockerd:

  ```bash
  systemctl restart dockerd
  ```

- 取消不可调度的标记:

  ```bash
  kubectl uncordon 10.179.80.31
  ```

### 定位根因，彻底解决

问题定位方法见附录，这里列举根因对应的解决方法：

- 日志输出量大导致磁盘写满:
  - 减少日志输出
  - 增大磁盘空间
  - 减小单机可调度的pod数量
- 可写层量大导致磁盘写满: 优化程序逻辑，不写文件到容器内或控制写入文件的大小与数量
- 镜像占用空间大导致磁盘写满:
  - 增大磁盘空间
  - 删除不需要的镜像

## 附录

### 查看docker的磁盘空间占用情况

```bash
$ docker system df -v
```

![图片描述](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141841.png)

### 定位容器写满磁盘的原因

进入容器数据目录(假设是 `/var/lib/docker`，并且存储驱动是 aufs):

```bash
$ cd /var/lib/docker
$ du -sh *
20K     builder
72K     buildkit
1.1G    containers
12M     image
100K    network
12G     overlay2
20K     plugins
4.0K    runtimes
4.0K    swarm
4.0K    tmp
4.0K    trust
28K     volumes
```

![image-20220119170006154](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141885.png)

- `containers` 目录: 体积大说明日志标准输出量大

  ```
  [root@v-cjweichen-mesh /var/lib/docker/containers]# du -sh *
  17M     01d8566ea348bd333b0d971a1751335b32f0d97396352e63a2dc25fdc7360a19
  32K     07533d0f9e45ee03f6f22117326de17e2c9e0712a1e265d3a778dec1329f25e7
  120K    076f585dc0f7532ab567743b7066d2803e2fbc698fa33f769e7f61002f450a3a
  28K     0777cff7c65a4594df7c612eca7dac0ec6ddea2ffe70a8b118830150b062a468
  28K     44565163715c8cdfe355892d7400f1836a86f9df27fb32adf32f5e41d7ea0531
  
  目录名即为容器id，可以根据前12位 进行模糊匹配，就等查到对应的POD 信息
  如：
  #docker ps  | grep 01d8566ea348
  01d8566ea348        ccr.ccs.tencentyun.com/tkeimages/tke-bridge-agent          "/install-cni.sh --c…"   2 weeks ago         Up 2 weeks                              k8s_tke-bridge-agent_tke-bridge-agent-qbbmn_kube-system_3b66c838-57c7-43a1-9281-8899c8b8b264_1
  #docker ps  | grep 445651637
  44565163715c        nginx     "/docker-entrypoint.…"   2 days ago          Up 2 days                               k8s_nginx_nginx-v1-6dc49fbd58-v2ntt_default_eed1f780-0d02-4788-9f45-8c94a2a58435_0
  ```

- overlay2    容器里面的文件系统rootfs在宿主机的路径规则

  ```
  [root@mesh /var/lib/docker/overlay2]# du -sh *
  36K     005fb0cee0261e7ab9201986ede339f5e1ea9f4d36efb5535c2451ecdb69488c
  91M     02973b91425e166ac4a6f096ab37e4af41fb2fcf7e658e809ddaf7698caa73b7
  32K     03cb6d1267a971930cdf50dcef86c69b3f92d16f0fc56a19b637e192c3bb57f7
  720K    07e9f8b0c251d3e6d01551622bd2c4994a638d65cba9a7073a61a3273b13a20a
  48K     07e9f8b0c251d3e6d01551622bd2c4994a638d65cba9a7073a61a3273b13a20a-init
  163M    09ae9b4457a0ed3eec58c129d524345855ab20bdb18560f9c7d506da3c48d121
  
  #目录名即为mount_id, 可以通过<docker_root>/image/overlay2/layerdb/mounts/container_id/mount-id查询
  cd  09ae9b4457a0ed3eec58c129d524345855ab20bdb18560f9c7d506da3c48d121
  [root@mesh /var/lib/docker/overlay2/09ae9b4457a0ed3eec58c129d524345855ab20bdb18560f9c7d506da3c48d121]# du -sh *
  100K    diff    #diff子目录: 容器可写层，体积大说明可写层数据量大(程序在容器里写入文件)
  4.0K    link
  4.0K    lower
  163M    merged   #联合挂载点，内容为容器里看到的内容，即包含镜像本身内容以及可写层内容
  8.0K    work
  ```

   ![image-20220119173509241](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141933.png)


### 找出日志输出量大的 pod

TKE 的 pod 中每个容器输出的日志最大存储 1G (日志轮转，最大10个文件，每个文件最大100m，可用 `docker inpect` 查看):

```bash
$ docker inspect fef835ebfc88
[
    {
         ...
        "HostConfig": {
            ...
            "LogConfig": {
                "Type": "json-file",
                "Config": {
                    "max-file": "10",
                    "max-size": "100m"
                }
            },
...
```

查看哪些容器日志输出量大：

```bash
$ cd /var/lib/docker/containers
$ du -sh *
```

![图片描述](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141977.png)

目录名即为容器id，使用前几位与 `docker ps` 结果匹配可找出对应容器，最后就可以推算出是哪些 pod 搞的鬼

### 找出可写层数据量大的 pod

可写层的数据主要是容器内程序自身写入的，无法控制大小，可写层越大说明容器写入的文件越多或越大，通常是容器内程序将log写到文件里了，查看一下哪个容器的可写层数据量大：

```bash
$ cd /var/lib/docker/overlay2/diff
$ du -sh *
```

![图片描述](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141024.png)

通过可写层目录(`diff`的子目录)反查容器id:

```bash
$ grep 834d97500892f56b24c6e63ffd4e520fc29c6c0d809a3472055116f59fb1d2be /var/lib/docker/image/aufs/layerdb/mounts/*/mount-id
/var/lib/docker/image/aufs/layerdb/mounts/eb76fcd31dfbe5fc949b67e4ad717e002847d15334791715ff7d96bb2c8785f9/mount-id:834d97500892f56b24c6e63ffd4e520fc29c6c0d809a3472055116f59fb1d2be
```

`mounts` 后面一级的id即为容器id: `eb76fcd31dfbe5fc949b67e4ad717e002847d15334791715ff7d96bb2c8785f9`，使用前几位与 `docker ps` 结果匹配可找出对应容器，最后就可以推算出是哪些 pod 搞的鬼

### 找出体积大的镜像

![图片描述](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042141069.png)



Docker 作为 K8S 容器运行时，容器日志的落盘将由 docker 来完成，保存在类似`/var/lib/docker/containers/$CONTAINERID` 目录下。Kubelet 会在 `/var/log/pods` 和 `/var/log/containers` 下面建立软链接，指向 `/var/lib/docker/containers/$CONTAINERID` 该目录下的容器日志文件

如果 Containerd 作为 K8S 容器运行时， 容器日志的落盘由 Kubelet 来完成，保存至 `/var/log/pods/$CONTAINER_NAME` 目录下，同时在 `/var/log/containers` 目录下创建软链接，指向日志文件