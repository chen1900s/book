![image-20210825205632996](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042153162.png)





pod 状态变更：将 Pod 设置为 Terminating 状态，并从所有 Service 的 Endpoints 列表中删除。此时，Pod 停止获得新的流量，但在 Pod 中运行的容器不会受到影响；

- 执行 preStop Hook：Pod 删除时会触发 preStop Hook，preStop Hook 支持 bash 脚本、TCP 或 HTTP 请求；
- 发送 SIGTERM 信号：向 Pod 中的容器发送 SIGTERM 信号；
- 等待指定的时间：terminationGracePeriodSeconds 字段用于控制等待时间，默认值为 30 秒。该步骤与 preStop Hook 同时执行，因此 - - terminationGracePeriodSeconds 需要大于 preStop 的时间，否则会出现 preStop 未执行完毕，pod 就被 kill 的情况；
- 发送 SIGKILL 信号：等待指定时间后，向 pod 中的容器发送 SIGKILL 信号，删除 pod。



参考文档：https://www.sohu.com/a/400910043_612370



## pod的状态分析

Pod状态

| 状态      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| Running   | 该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。 |
| Pending   | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。创建pod的请求已经被k8s接受，但是容器并没有启动成功，可能处在：写数据到etcd，调度，pull镜像，启动容器这四个阶段中的任何一个阶段，pending伴随的事件通常会有：ADDED, Modified这两个事件的产生 |
| Succeeded | Pod中的所有的容器已经正常的自行退出，并且k8s永远不会自动重启这些容器，一般会是在部署job的时候会出现。 |
| Failed    | Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。 |
| Unknown   | 出于某种原因，无法获得Pod的状态，通常是由于与Pod主机通信时出错。 |

Pod 的详细的状态说明

| 状态                                  | 描述                          |
| ------------------------------------- | ----------------------------- |
| CrashLoopBackOff                      | 容器退出，kubelet正在将它重启 |
| InvalidImageName                      | 无法解析镜像名称              |
| ImageInspectError                     | 无法校验镜像                  |
| ErrImageNeverPull                     | 策略禁止拉取镜像              |
| ImagePullBackOff                      | 正在重试拉取                  |
| RegistryUnavailable                   | 连接不到镜像中心              |
| ErrImagePull                          | 通用的拉取镜像出错            |
| CreateContainerConfigError            | 不能创建kubelet使用的容器配置 |
| CreateContainerError                  | 创建容器失败                  |
| m.internalLifecycle.PreStartContainer | 执行hook报错                  |
| RunContainerError                     | 启动容器失败                  |
| PostStartHookError                    | 执行hook报错                  |
| ContainersNotInitialized              | 容器没有初始化完毕            |
| ContainersNotRead                     | 容器没有准备完毕              |
| ContainerCreating                     | 容器创建中                    |
| PodInitializing                       | pod 初始化中                  |
| DockerDaemonNotReady                  | docker还没有完全启动          |
| NetworkPluginNotReady                 | 网络插件还没有完全启动        |