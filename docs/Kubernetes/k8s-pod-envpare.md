---
title: 通过环境变量将POD信息呈现给容器
abbrlink: 6c58efa9
date: 2022-10-23 15:46:12
tags:
  - TKE
  - Kubernetes
categories: TKE
keywords:
  - Kubernetes
  - TKE
description: 通过环境变量将POD信息呈现给容器
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202210232109389.jpeg
updated: 2022-10-23 15:46:12
---

本文通过环境变量将 Pod 信息呈现给容器

## 用 Pod 字段作为环境变量的值

这个配置文件中，你可以看到五个环境变量。`env` 字段是一个 [EnvVars](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#envvar-v1-core). 对象的数组。 数组中第一个元素指定 `MY_NODE_NAME` 这个环境变量从 Pod 的 `spec.nodeName` 字段获取变量值。 同样，其它环境变量也是从 Pod 的字段获取它们的变量值。

**说明：** 本示例中的字段是 Pod 字段，不是 Pod 中 Container 的字段。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: dapi-envars-fieldref
  name: dapi-envars-fieldref
  namespace: default
spec:
  selector:
    matchLabels:
      k8s-app: dapi-envars-fieldref
  template:
    metadata:
      labels:
        k8s-app: dapi-envars-fieldref
    spec:
      containers:
      - args:
        - while true; do echo -en '\n'; printenv MY_NODE_NAME MY_POD_NAME MY_POD_NAMESPACE; printenv MY_POD_IP MY_POD_SERVICE_ACCOUNT; sleep 10;done;
        command:
        - sh
        - -c
        env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIPs
        - name: MY_POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.serviceAccountName
        image: busybox
        imagePullPolicy: IfNotPresent
        name: dapi-envars-fieldref
        resources: {}

```

## 用 Container 字段作为环境变量的值

前面的练习中，你将 Pod 字段作为环境变量的值。 接下来这个练习中，你将用 Container 字段作为环境变量的值。这里是包含一个容器的 Pod 的配置文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: dapi-envars-container
  name: dapi-envars-container
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: dapi-envars-container
  template:
    metadata:
      labels:
        k8s-app: dapi-envars-container
    spec:
      containers:
      - args:
        - while true; do  echo -en '\n'; printenv MY_CPU_REQUEST MY_CPU_LIMIT;printenv MY_MEM_REQUEST MY_MEM_LIMIT;sleep 10;done;
        command:
        - sh
        - -c
        env:
        - name: MY_CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              divisor: "1"
              resource: requests.cpu
        - name: MY_CPU_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              divisor: "1"
              resource: limits.cpu
        - name: MY_MEM_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              divisor: "1"
              resource: requests.memory
        - name: MY_MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: test-container
              divisor: "1"
              resource: limits.memory
        image: busybox:1.24
        imagePullPolicy: IfNotPresent
        name: test-container
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 64Mi


```

这个配置文件中，你可以看到四个环境变量。`env` 字段是一个 [EnvVars](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#envvar-v1-core). 对象的数组。数组中第一个元素指定 `MY_CPU_REQUEST` 这个环境变量从 Container 的 `requests.cpu` 字段获取变量值。同样，其它环境变量也是从 Container 的字段获取它们的变量值。

**说明：** 本例中使用的是 Container 的字段而不是 Pod 的字段。

## 通过文件将 Pod 信息呈现给容器

此页面描述 Pod 如何使用 DownwardAPIVolumeFile 把自己的信息呈现给 Pod 中运行的容器。 DownwardAPIVolumeFile 可以呈现 Pod 的字段和容器字段

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kubernetes-downwardapi-volume-example
  name: kubernetes-downwardapi-volume-example
  namespace: default
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-downwardapi-volume-example
  template:
    metadata:
      labels:
        k8s-app: kubernetes-downwardapi-volume-example
    spec:
      containers:
      - args:
        - while true; do if [[ -e /etc/podinfo/labels ]]; then  echo -en '\n\n'; cat /etc/podinfo/labels; fi; if [[ -e /etc/podinfo/annotations ]]; then echo -en '\n\n'; cat /etc/podinfo/annotations; fi; sleep 5;done;
        command:
        - sh
        - -c
        image: busybox
        imagePullPolicy: IfNotPresent
        name: client-container
        resources: {}
        volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
      volumes:
      - name: podinfo
        downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "annotations"
              fieldRef:
                fieldPath: metadata.annotations
```

## Downward API支持的常用字段

**1，使用fieldRef可以声明使用:**

spec.nodeName - 宿主机名字

status.hostIP - 宿主机IP

metadata.name - Pod的名字

metadata.namespace - Pod的Namespace

status.podIP - Pod的IP

spec.serviceAccountName - Pod的Service Account的名字

metadata.uid - Pod的UID

metadata.labels['<KEY>'] - 指定<KEY>的Label值

metadata.annotations['<KEY>'] - 指定<KEY>的Annotation值

metadata.labels - Pod的所有Label

metadata.annotations - Pod的所有Annotation

 

**2，使用resourceFieldRef可以声明使用:**

limits.cpu-容器的CPU limit 

requests.cpu-容器的CPU request

limits.memory-容器的memory limit

requests.memory容器的memory request




完整示例：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: centos-env
    qcloud-app: centos-env
  name: centos-env
spec:
  selector:
    matchLabels:
      k8s-app: centos-env
  template:
    metadata:
      annotations:
        description: ddd
      labels:
        k8s-app: centos-env
        qcloud-app: centos-env
    spec:
      containers:
      - env:
        - name: pod-name
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: pod-namespacce
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: pod-labels
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.labels['k8s-app']
        - name: pod-annotations
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.annotations['description']
        - name: pod-uid
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.uid
        - name: pod-nodename
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        - name: pod-hostIP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.hostIP
        - name: pod-podIP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
        - name: pod-serviceAccountName
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.serviceAccountName
        image: ccr.ccs.tencentyun.com/chenjingwei/centos:latest
        imagePullPolicy: IfNotPresent
        name: centos-1
        resources:
          limits:
            cpu: 25m
            memory: 64Mi
          requests:
            cpu: 25m
            memory: 64Mi
        volumeMounts:
        - mountPath: /etc/podinfo
          name: podinfo
          readOnly: false
      imagePullSecrets:
      - name: qcloudregistrykey
      volumes:
      - name: podinfo
        projected:
          sources:
          - downwardAPI:
              items:
                - path: "labels"
                  fieldRef:
                    fieldPath: metadata.labels
                - path: "annotations"
                  fieldRef:
                    fieldPath: metadata.annotations

```

