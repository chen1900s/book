---
title: Kubernetes部署主从架构Mysql集群
abbrlink: 24ba3d3c
date: 2020-05-28 17:46:52
tags: 
  - Mysql
  - Kubernetes
categories: Mysql
keywords:
  - kubernetes
  - Mysql
  - tke
description: 使用Kubernetes部署主从架构的Mysql集群
top_img: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041750814.jpeg
updated: 2020-05-28 23:58:58
---

## 概述

> 随着kubernetes发展，越来越多的人开始使用kubernetes部署自己应用，它是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能，然而很多应用也使用到数据库，下面通过kubernetes上部署个主从架构的Mysql集群供应用使用

## 准备环境

kubernetes集群（本次实验使用的是腾讯云TKE集群，版本1.18）

## 操作步骤

主要通过一下几个步骤完整的搭建一个MySQL集群

1. 搭建一个`主从复制`（Master-Slave）的MySQL集群
2. 从节点可以进行水平扩展，扩容多个节点
3. 所有的`写`操作只能在MySQL主节点上执行
4. 读操作可以在MySQL主从节点上执行
5. 从节点能自动同步主节点的数据

![image-20211125204741527](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041753863.png)



### 服务部署

1，创建mysql使用的Namespace（如果不创建可以使用默认命名空间，一般建议单独给数据创建个命名空间使用）

```
apiVersion: v1
kind: Namespace
metadata:
  name: mysql
  labels:
    app: mysql
```

2，创建数据库的配置文件configmap

使用ConfigMap为Master/Slave节点分配不同的配置文件

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  namespace: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # Master主节点配置
    [mysqld]
    log-bin=mysqllog
    skip-name-resolve
  slave.cnf: |
    # Slave从节点配置
    [mysqld]
    super-read-only
    skip-name-resolve
    log-bin=mysql-bin
    replicate-ignore-db=mysql
```

3，创建MySQL密码Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql
  labels:
    app: mysql
type: Opaque
data:
  password: MTIzNDU2   # echo -n "123456" | base64
```

4，使用Service为MySQL提供读写分离

- 用户所有写请求，必须以DNS记录的方式直接访问到Master节点，也就是mysql-0.mysql这条DNS记录。
- 用户所有读请求，必须访问自动分配的DNS记录可以被转发到任意一个Master或Slave节点上，也就是mysql-read这条DNS记录。

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

5，创建MySQL集群实例

使用StatefulSet搭建MySQL主从集群

整体的StatefulSet有两个Replicas，一个Master，一个Slave，然后使用`init-mysql`这个`initContainers`进行`配置文件的初始化`。接着使用`clone-mysql`这个`initContainers`进行`数据的传输`；同时使用`xtrabackup`这个**`sidecar`容器**进行`SQL初始化和数据传输功能`。

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
  labels:
    app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql    #注意这个千万别少些
  replicas: 2
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 从Pod的序号，生成server-id
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 由于server-id不能为0，因此给ID加100来避开它
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果Pod的序号为0，说明它是Master节点，从ConfigMap里把Master的配置文件拷贝到/mnt/conf.d目录下
          # 否则，拷贝ConfigMap里的Slave的配置文件
          if [[ ${ordinal} -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.tencentcloudcr.com/google-samples/xtrabackup:1.0   #使用腾讯镜像加速
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以数据已经存在则跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Master 节点（序号为 0）不需要这个操作
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal == 0 ]] && exit 0
          # 使用ncat指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
 #        - name: MYSQL_ALLOW_EMPTY_PASSWORD
 #          value: "1"
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.tencentcloudcr.com/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          # 从备份信息文件里读取MASTER_LOG_FILE和MASTER_LOG_POS这2个字段的值，用来拼装集群初始化SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果xtrabackup_slave_info文件存在，说明这个备份数据来自于另一个Slave节点
            # 这种情况下，XtraBackup工具在备份的时候，就已经在这个文件里自动生成了“CHANGE MASTER TO”SQL语句
            # 所以，只需要把这个文件重命名为change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着xtrabackup_binlog_info了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只是存在xtrabackup_binlog_info文件，说明备份来自于Master节点，就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ $(cat xtrabackup_binlog_info) =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成SQL，写入change_master_to.sql.in文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          # 如果存在change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等MySQL容器启动之后才能进行下一步连接MySQL的操作
            echo "Waiting for mysqld to be ready（accepting connections）"
            until mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            # 将文件change_master_to.sql.in改个名字
            # 防止这个Container重启的时候，因为又找到了change_master_to.sql.in，从而重复执行一遍初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用change_master_to.sql.orig的内容，也就是前面拼装的SQL，组成一个完整的初始化和启动Slave的SQL语句
            mysql -h 127.0.0.1 -uroot -p${MYSQL_ROOT_PASSWORD} << EOF
          $(< change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='${MYSQL_ROOT_PASSWORD}',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          # 使用ncat监听3307端口。
          # 它的作用是，在收到传输请求的时候，直接执行xtrabackup --backup命令，备份MySQL的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root --password=${MYSQL_ROOT_PASSWORD}"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - "ReadWriteOnce"
      storageClassName: cbs
      resources:
        requests:
          storage: 10Gi            #数据盘大小根据业务情况镜像修改，这个只做测试，只写了10Gi 
```

可以看到，StatefulSet启动成功后，会有两个Pod运行。接下来，我们可以尝试向这个MySQL集群发起请求，执行一些SQL操作来验证它是否正常。整个过程因为拉取mysql和一个`gcr.io/google-samples/xtrabackup:1.0`（使用腾讯云加速镜像地址gcr.tencentcloudcr.com）国外的镜像会很慢,但是在创建mysql-0拉取一次之后，后续创建mysql-1就相对很快了。

最后，容器检查pod的运行状态

```
[root@VM-0-17-tlinux ~]# kubectl  get all -n mysql  -o wide
NAME          READY   STATUS    RESTARTS   AGE    IP           NODE           NOMINATED NODE   READINESS GATES
pod/mysql-0   2/2     Running   0          108s   172.18.1.4   192.168.2.40   <none>           <none>
pod/mysql-1   2/2     Running   0          76s    172.18.1.5   192.168.2.40   <none>           <none>

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/mysql        ClusterIP   None             <none>        3306/TCP   2m24s   app=mysql
service/mysql-read   ClusterIP   172.18.253.108   <none>        3306/TCP   2m24s   app=mysql

NAME                     READY   AGE    CONTAINERS         IMAGES
statefulset.apps/mysql   2/2     108s   mysql,xtrabackup   mysql:5.7,gcr.tencentcloudcr.com/google-samples/xtrabackup:1.0
```

### 服务验证

1，验证主从关系

```
[root@VM-0-17-tlinux ~]# kubectl -n mysql exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'show slave status \G'"
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-0.mysql.mysql
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysqllog.000003
          Read_Master_Log_Pos: 154
               Relay_Log_File: mysql-1-relay-bin.000002
                Relay_Log_Pos: 319
        Relay_Master_Log_File: mysqllog.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 528
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 100
                  Master_UUID: f8d3bd9a-4df4-11ec-9930-52d15f478b07
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
```

2，接下来，我们通过Master容器创建数据库和表、插入数据库。

```
kubectl -n mysql exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'create database test'"
kubectl -n mysql exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;create table counter(c int);'"
kubectl -n mysql exec mysql-0 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;insert into counter values(123)'"
```

3，然后，我们观察Slave节点是否都同步到数据了

```
kubectl -n mysql exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;select * from counter'"  
```

执行返回结果是，当看到输出结果，主从同步正常了。

```
[root@VM-0-17-tlinux ~]# kubectl -n mysql exec mysql-1 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;select * from counter'" 
c
123
```



### 扩展从节点

在有了StatefulSet以后，你就可以像Deployment那样，非常方便地扩展这个MySQL集群，比如：

```
kubectl -n mysql scale statefulset mysql  --replicas=3
statefulset.apps/mysql scaled
[root@VM-0-17-tlinux ~]# kubectl  get pods -n mysql
NAME      READY   STATUS     RESTARTS   AGE
mysql-0   2/2     Running    0          10m
mysql-1   2/2     Running    0          10m
mysql-2   0/2     Init:1/2   0          24s
```

这时候，一个新的mysql-2就创建出来了，我们继续验证新扩容的节点是否都同步到主节点的数据。

```
kubectl -n mysql exec mysql-2 -c mysql -- bash -c "mysql -uroot -p123456 -e 'use test;select * from counter'"  
```

当看到输出结果，主从同步正常了。也就是说从StatefulSet为我们新创建的mysql-2上，同样可以读取到之前插入的记录。也就是说，我们的数据备份和恢复，都是有效的





