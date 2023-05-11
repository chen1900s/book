---
title: Kubernetes部署rabbitmq服务
abbrlink: 202f1e67
date: 2023-02-21 13:28:40
tags:
  - Kubernetes
  - Rabbitmq
categories: Kubernetes
keywords:
  - kubernetes
  - rabbitmq
  - tke
description: kubernetes部署rabbitmq服务
top_img: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202302211333655.png
updated: 2022-05-27 23:58:58
---

待补充，先上yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq
  namespace: default
secrets:
- name: rabbitmq



---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq-endpoint-reader
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq-endpoint-reader
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq-endpoint-reader
subjects:
- kind: ServiceAccount
  name: rabbitmq

```





```
apiVersion: v1
data:
  rabbitmq.conf: |-
    ## Username and password
    default_user = user
    default_pass = CHANGEME
    ## Clustering
    cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
    cluster_formation.node_cleanup.interval = 10
    cluster_formation.node_cleanup.only_log_warning = true
    cluster_partition_handling = autoheal
    # queue master locator
    queue_master_locator = min-masters
    # enable guest user
    loopback_users.guest = false
    #disk_free_limit.absolute = 50MB
    #management.load_definitions = /app/load_definition.json
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq-config
  namespace: default
  
---
apiVersion: v1
data:
  rabbitmq-erlang-cookie: b3hWVXpRQmlWWUt4VUR3bWRSbTlDRldseU01Ulg3SzM=
  rabbitmq-password: ZGtBU3o5ODZ2ZQ==
kind: Secret
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq
  namespace: default
type: Opaque

```







```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq-headless
  namespace: default
spec:
  clusterIP: None
  ports:
  - name: epmd
    port: 4369
    targetPort: epmd
  - name: amqp
    port: 5672
    targetPort: amqp
  - name: dist
    port: 25672
    targetPort: dist
  - name: stats
    port: 15672
    targetPort: stats
  selector:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/name: rabbitmq


---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq
  namespace: default
spec:
  ports:
  - name: amqp
    nodePort: null
    port: 5672
    targetPort: amqp
  - name: epmd
    nodePort: null
    port: 4369
    targetPort: epmd
  - name: dist
    nodePort: null
    port: 25672
    targetPort: dist
  - name: http-stats
    nodePort: null
    port: 15672
    targetPort: stats
  selector:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/name: rabbitmq
  type: ClusterIP

```



```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  annotations:
    meta.helm.sh/release-name: rabbitmq
    meta.helm.sh/release-namespace: cjweichen
  labels:
    app.kubernetes.io/instance: rabbitmq
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: rabbitmq
    helm.sh/chart: rabbitmq-7.5.7
  name: rabbitmq
  namespace: default
spec:
  podManagementPolicy: OrderedReady
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/instance: rabbitmq
      app.kubernetes.io/name: rabbitmq
  serviceName: rabbitmq-headless
  template:
    metadata:
      annotations:
        checksum/secret: c3d76de93266cdb4f6a35506a2a5e005e61c7074cd500d28f210e06a6043a0b7
      labels:
        app.kubernetes.io/instance: rabbitmq
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: rabbitmq
        helm.sh/chart: rabbitmq-7.5.7
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - eklet-subnet-fixt567x-ipcmleaw
      containers:
      - env:
        - name: BITNAMI_DEBUG
          value: "false"
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
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
        - name: K8S_SERVICE_NAME
          value: rabbitmq-headless
        - name: K8S_ADDRESS_TYPE
          value: hostname
        - name: RABBITMQ_FORCE_BOOT
          value: "no"
        - name: RABBITMQ_NODE_NAME
          value: rabbit@$(MY_POD_NAME).$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
        - name: K8S_HOSTNAME_SUFFIX
          value: .$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_MNESIA_DIR
          value: /bitnami/rabbitmq/mnesia/$(RABBITMQ_NODE_NAME)
        - name: RABBITMQ_LDAP_ENABLE
          value: "no"
        - name: RABBITMQ_LOGS
          value: '-'
        - name: RABBITMQ_ULIMIT_NOFILES
          value: "65536"
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: RABBITMQ_ERL_COOKIE
          valueFrom:
            secretKeyRef:
              key: rabbitmq-erlang-cookie
              name: rabbitmq
        - name: RABBITMQ_USERNAME
          value: user
        - name: RABBITMQ_PASSWORD
          valueFrom:
            secretKeyRef:
              key: rabbitmq-password
              name: rabbitmq
        - name: RABBITMQ_PLUGINS
          value: rabbitmq_management, rabbitmq_peer_discovery_k8s, rabbitmq_auth_backend_ldap
        image: ccr.ccs.tencentyun.com/tke-market/rabbitmq:3.8.5-debian-10-r38
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - bash
              - -ec
              - rabbitmqctl stop_app
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -ec
            - rabbitmq-diagnostics -q check_running
          failureThreshold: 6
          initialDelaySeconds: 120
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 20
        name: rabbitmq
        ports:
        - containerPort: 5672
          name: amqp
          protocol: TCP
        - containerPort: 25672
          name: dist
          protocol: TCP
        - containerPort: 15672
          name: stats
          protocol: TCP
        - containerPort: 4369
          name: epmd
          protocol: TCP
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -ec
            - rabbitmq-diagnostics -q check_running
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 20
        resources: {}
        volumeMounts:
        - mountPath: /bitnami/rabbitmq/conf
          name: configuration
        - mountPath: /bitnami/rabbitmq/mnesia
          name: data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1001
        runAsUser: 1001
      serviceAccount: rabbitmq
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: rabbitmq.conf
            path: rabbitmq.conf
          name: rabbitmq-config
        name: configuration
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      labels:
        app.kubernetes.io/instance: rabbitmq
        app.kubernetes.io/name: rabbitmq
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: cbs
      volumeMode: Filesystem
```



