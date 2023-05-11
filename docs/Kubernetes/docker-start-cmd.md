## 创建 Pod 时设置命令及参数

创建 Pod 时，可以为其下的容器设置启动时要执行的命令及其参数。如果要设置命令，就填写在配置文件的 `command` 字段下，如果要设置命令的参数，就填写在配置文件的 `args` 字段下。一旦 Pod 创建完成，该命令及其参数就无法再进行更改了。

如果在配置文件中设置了容器启动时要执行的命令及其参数，那么容器镜像中自带的命令与参数将会被覆盖而不再执行。如果配置文件中只是设置了参数，却没有设置其对应的命令，那么容器镜像中自带的命令会使用该新参数作为其执行时的参数。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: command-demo
  name: command-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: command-demo
  template:
    metadata:
      labels:
        k8s-app: command-demo
    spec:
      containers:
      - args:
        - HOSTNAME
        - KUBERNETES_PORT
        command:
        - printenv
        image: debian
        imagePullPolicy: IfNotPresent
        name: command-demo-container
        resources: {}
        securityContext:
          privileged: false
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Alway
```

## 在 Shell 来执行命令

有时候，你需要在 Shell 脚本中运行命令。 例如，你要执行的命令可能由多个命令组合而成，或者它就是一个 Shell 脚本。 这时，就可以通过如下方式在 Shell 中执行命令：

```
command: ["/bin/sh"]
args: ["-c", "while true; do echo hello; sleep 10;done"]
```

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: command-demo
  name: command-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: command-demo
  template:
    metadata:
      labels:
        k8s-app: command-demo
    spec:
      containers:
      - args: ["-c", "while true; do echo hello; sleep 10;done"]
        command: ["/bin/sh"]
        image: debian
        imagePullPolicy: IfNotPresent
        name: command-demo-container
        resources: {}
        securityContext:
          privileged: false
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
```

## 说明事项

下表给出了 Docker 与 Kubernetes 中对应的字段名称。

| 描述           | Docker 字段名称 | Kubernetes 字段名称 |
| -------------- | --------------- | ------------------- |
| 容器执行的命令 | Entrypoint      | command             |
| 传给命令的参数 | Cmd             | args                |

如果要覆盖默认的 Entrypoint 与 Cmd，需要遵循如下规则：

- 如果在容器配置中没有设置 `command` 或者 `args`，那么将使用 Docker 镜像自带的命令及其参数。
- 如果在容器配置中只设置了 `command` 但是没有设置 `args`，那么容器启动时只会执行该命令， Docker 镜像中自带的命令及其参数会被忽略。
- 如果在容器配置中只设置了 `args`，那么 Docker 镜像中自带的命令会使用该新参数作为其执行时的参数。
- 如果在容器配置中同时设置了 `command` 与 `args`，那么 Docker 镜像中自带的命令及其参数会被忽略。 容器启动时只会执行配置中设置的命令，并使用配置中设置的参数作为命令的参数。

下面是一些例子：

| 镜像 Entrypoint | 镜像 Cmd    | 容器 command | 容器 args   | 命令执行         |
| --------------- | ----------- | ------------ | ----------- | ---------------- |
| `[/ep-1]`       | `[foo bar]` | <not set>    | <not set>   | `[ep-1 foo bar]` |
| `[/ep-1]`       | `[foo bar]` | `[/ep-2]`    | <not set>   | `[ep-2]`         |
| `[/ep-1]`       | `[foo bar]` | <not set>    | `[zoo boo]` | `[ep-1 zoo boo]` |
| `[/ep-1]`       | `[foo bar]` | `[/ep-2]`    | `[zoo boo]` | `[ep-2 zoo boo]` |

参考链接：https://kubernetes.io/zh/docs/tasks/inject-data-application/define-command-argument-container/#notes