

# HPA 扩缩容灵敏度

```
--horizontal-pod-autoscaler-downscale-stabilization-window  ：参数控制hpa缩容时间窗口  默认 5 分钟**
```

备注：

在 K8S 1.18 之前，HPA 扩容是无法调整灵敏度的:

1. 对于缩容，由 `kube-controller-manager` 的 `--horizontal-pod-autoscaler-downscale-stabilization-window` 参数控制缩容时间窗口，默认 5 分钟，即负载减小后至少需要等 5 分钟才会缩容。
2. 对于扩容，由 hpa controller 固定的算法、硬编码的常量因子来控制扩容速度，无法自定义。

```
terminationGracePeriodSeconds   
```

备注：等待容器进程完全停止，如果在 terminationGracePeriodSeconds 内 (默认30s) 还未完全停止，将发送 SIGKILL 信号强制停止进程

```
--node-monitor-period=5s   
```

在NodeController中同步节点状态的周期。默认5s

```
--node-monitor-grace-period=40s
```

 我们允许运行的节点在标记为不健康之前没有响应的时间。必须是kubelet的nodeStatusUpdateFrequency的N倍，其中N表示允许kubelet发布节点状态的重试次数默认40s。

```
--node-startup-grace-period=1m0s
```

   我们允许启动节点在标记为不健康之前没有响应的时间。默认1m0s。(这个其实不需要调整因为是启动节点的时间，业务并没有要求启动时间)

```
--pod-eviction-timeout=5m
```

 #删除失败节点上的pods的宽限期。默认5m 失败立即删除

```
--node-status-update-frequency: 10s
```

指定的频率连续报告节点状态更新，其默认值为 10s。指定kubelet多长时间向master发布一次节点状态。注意: 它必须与kube-controller中的`nodeMonitorGracePeriod`一起协调工作。(默认 10s)



**TEK控制台webconsole  默认3分钟不进行任何操作，就会断开**

