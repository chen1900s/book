# CoreDns配置以及外部dns使用

## CoreDNS ConfigMap选项

先来看看默认的CoreDns的配置文件

```
  Corefile: |2-
        .:53 {
            errors
            health
            kubernetes cluster.local. in-addr.arpa ip6.arpa {
                pods insecure
                upstream
                fallthrough in-addr.arpa ip6.arpa
            }
            prometheus :9153
            forward . /etc/resolv.conf
            cache 30
            reload
            loadbalance
        }
```

Corefile 配置包括以下 CoreDNS [插件](https://coredns.io/plugins/)：

- [errors](https://coredns.io/plugins/errors/)：错误记录到标准输出。

- [health](https://coredns.io/plugins/health/)：在 http://localhost:8080/health 处提供 CoreDNS 的健康报告。

- [ready](https://coredns.io/plugins/ready/)：在端口 8181 上提供的一个 HTTP 末端，当所有能够 表达自身就绪的插件都已就绪时，在此末端返回 200 OK。

- [kubernetes](https://coredns.io/plugins/kubernetes/)：CoreDNS 将基于 Kubernetes 的服务和 Pod 的 IP 答复 DNS 查询。你可以在 CoreDNS 网站阅读[更多细节](https://coredns.io/plugins/kubernetes/)。 你可以使用 `ttl` 来定制响应的 TTL。默认值是 5 秒钟。TTL 的最小值可以是 0 秒钟， 最大值为 3600 秒。将 TTL 设置为 0 可以禁止对 DNS 记录进行缓存， 此行内的cluster.local处理custer.local区域中的所有查询（以及in-addr.arpa中的反向dns查找） 

  `pods insecure` 选项是为了与 kube-dns 向后兼容。你可以使用 `pods verified` 选项，该选项使得 仅在相同名称空间中存在具有匹配 IP 的 Pod 时才返回 A 记录。如果你不使用 Pod 记录，则可以使用 `pods disabled` 选项。

- [prometheus](https://coredns.io/plugins/prometheus/)：CoreDNS 的度量指标值以 [Prometheus](https://prometheus.io/) 格式在 http://localhost:9153/metrics 上提供。

- [forward](https://coredns.io/plugins/forward/): 不在 Kubernetes 集群域内的任何查询都将转发到 预定义的解析器 (/etc/resolv.conf).

  
  
  解析外部域名 coredns 默认会请求上游 DNS 来查询，这里的上游 DNS 默认是 coredns pod 所在宿主机的 `resolv.conf` 里面的 nameserver (coredns pod 的 dnsPolicy 为 “Default”，也就是会将宿主机里的 `resolv.conf` 里的 nameserver 加到容器里的 `resolv.conf`, coredns 默认配置 `proxy . /etc/resolv.conf`, 意思是非 service 域名会使用 coredns 容器中 `resolv.conf` 文件里的 nameserver 来解析)
  
   POD里面访问集群外的域名 走的是上游DNS服务器，也就是coredns所在的CVM节点DNS去做解析， 会先在coredns 的search域里面查找，查找不到走的是上游DNS服务器 
  
   外部域名请求的一个流程
  
   **路由请求流程** 
  
  未配置存根域：没有匹配上配置的集群域名后缀的任何请求，例如 “www.kubernetes.io”，将会被转发到继承自节点的上游域名服务器。
  
  已配置存根域：如果配置了存根域和上游DNS服务器，DNS查询将基于下面的流程对请求进行路由：
  \1. 查询首先被发送到coredns中的DNS缓存层。
  \2. 从缓存层，检查请求的后缀，并根据下面的情况转发到对应的DNS上：
    \- 具有集群后缀的名字（例如“.cluster.local”）：请求被发送到coredns。
    \- 具有存根域后缀的名字（例如“.acme.local”）：请求被发送到配置的自定义DNS解析器（例如：监听在 1.2.3.4）。
    \- 未能匹配上后缀的名字（例如“widget.com”）：请求被转发到上游DNS。 
  
  
  
   
  
  上游DNS解析是 随机的策略

https://github.com/coredns/coredns/blob/29f6d0a6b20e9f3e7509867d932f11efd443c72b/plugin/forward/README.md

默认是随机策略

![image-20211201151030828](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042140087.png)

- [cache](https://coredns.io/plugins/cache/)：启用前端缓存，需要指出的是，上文中的cache 30，表示 pttl 为30s，也就是说，一条域名解析记录在DNS缓存中的存留时间最长为30s。

- [loop](https://coredns.io/plugins/loop/)：检测到简单的转发环，如果发现死循环，则中止 CoreDNS 进程。

- [reload](https://coredns.io/plugins/reload)：允许自动重新加载已更改的 Corefile。 编辑 ConfigMap 配置后，请等待两分钟，以使更改生效。

- [loadbalance](https://coredns.io/plugins/loadbalance)：这是一个轮转式 DNS 负载均衡器， 它在应答中随机分配 A、AAAA 和 MX 记录的顺序。

   CoreDNS内置的两个健康检查插件`health`和`ready`的使用方式和适用场景 ：https://tinychen.com/20220728-dns-11-coredns-08-healthcheck/

 **路由请求流程** 

未配置存根域：没有匹配上配置的集群域名后缀的任何请求，例如 “www.kubernetes.io”，将会被转发到继承自节点的上游域名服务器。

已配置存根域：如果配置了存根域和上游DNS服务器，DNS查询将基于下面的流程对请求进行路由：

1. 查询首先被发送到coredns中的DNS缓存层。

2. 从缓存层，检查请求的后缀，并根据下面的情况转发到对应的DNS上：

   - 具有集群后缀的名字（例如“.cluster.local”）：请求被发送到coredns。

   - 具有存根域后缀的名字（例如“.acme.local”）：请求被发送到配置的自定义DNS解析器（例如：监听在 1.2.3.4）。
   - 未能匹配上后缀的名字（例如“widget.com”）：请求被转发到上游DNS。

## DNS域名解析原理

节点kubelet的启动参数有`--cluster-dns=<dns-service-ip>`、`--cluster-domain=<default-local-domain>`，这两个参数分别被用来设置集群DNS服务器的IP地址和主域名后缀。

Pod内的DNS域名解析配置文件为/etc/resolv.conf，文件内容如下。

```
earch default.svc.cluster.local svc.cluster.local cluster.local
nameserver 172.18.254.219
options ndots:5
```

**search**：设置域名的查找后缀规则，查找配置越多，说明域名解析查找匹配次数越多，集群匹配有`default.svc.cluster.local`、`svc.cluster.local`、`cluster.local`3个后缀，最多进行8次查询才能得到正确解析结果，因为集群里面进行IPV4和IPV6查询各四次

**nameserver**：定义DNS服务器的IP地址

**options**：定义域名解析配置文件选项，支持多个KV值。例如该参数设置成`ndots:5`，说明如果访问的域名字符串内的点字符数量超过`ndots`值，则认为是完整域名，并被直接解析；如果不足`ndots`值，则追加**search**段后缀再进行查询

## 集群dnsPolicy配置

Kubernetes持通过**dnsPolicy**字段为每个Pod配置不同的DNS策略。目前支持四种策略：

- **ClusterFirst**：通过CoreDNS来做域名解析，Pod内/etc/resolv.conf配置的DNS服务地址是集群DNS服务的kube-dns地址。该策略是集群工作负载的默认策略。
- **None**：忽略集群DNS策略，需要您提供**dnsConfig**字段来指定DNS配置信息。
- **Default**：Pod直接继承集群节点的域名解析配置。即在ACK集群直接使用ECS的/etc/resolv.conf文件（文件内配置的是阿里云DNS服务）。
- **ClusterFirstWithHostNet**：强制在hostNetWork网络模式下使用ClusterFirst策略（默认使用Default策略）

**Pod层面自定义DNS配置**

```
apiVersion: v1
kind: Pod
metadata:
  name: alpine
  namespace: default
spec:
  containers:
  - image: alpine
    command:
      - sleep
      - "10000"
    imagePullPolicy: Always
    name: alpine
  dnsPolicy: None
  dnsConfig:
    nameservers: ["169.254.xx.xx"]
    searches:
    - default.svc.cluster.local
    - svc.cluster.local
    - cluster.local
    options:
    - name: ndots
      value: "2"
```



## 集群CoreDNS扩展配置

### 场景一：开启日志服务

如果需将CoreDNS每次域名解析的日志打印出来，您可以开启log插件，在Corefile里加上log。示例配置如下：

```
  Corefile: |2-
        .:53 {
            errors
            log
            health
            kubernetes cluster.local. in-addr.arpa ip6.arpa {
                pods insecure
                upstream
                fallthrough in-addr.arpa ip6.arpa
            }
            prometheus :9153
            forward . /etc/resolv.conf
            cache 30
            reload
            loadbalance
        }
```



### 场景二：配置外部上游dns服务器

有些服务不在kubernetes内部，在内部环境内需要通过dns去访问,名称后缀为`carey.com` ，强制所有非集群 DNS 查找通过特定的域名服务器（位于10.150.0.1），将 proxy 和 forward 指向域名服务器，而不是 /etc/resolv.conf。 

```
carey.com:53 {
        errors
        cache 30
        forward . 10.150.0.1
    }
```

 要显式强制所有非集群 DNS 查找通过特定的域名服务器（位于 172.16.0.1），可将 `forward` 指向该域名服务器，而不是 `/etc/resolv.conf`。 

 完整的配置文件 

```
Corefile: |2-
  .:53 {
      errors
      health
      kubernetes cluster.local in-addr.arpa ip6.arpa {
         pods insecure
         upstream
         fallthrough in-addr.arpa ip6.arpa
      }
      prometheus :9153
      forward . /etc/resolv.conf
      cache 30
      loop
      reload
      loadbalance
  }
  domain-name:53 {
        errors 
        cache 30
        forward . custom-dns-server
        reload    #前面的域要加上reload
        }
  carey.com:53 {
      errors
      cache 30
      forward . 10.150.0.1
  }
```



### 场景三：通过coredns实现内外流量分离

1. **场景**
   旧业务固定了域名，无法通过内部service直接访问服务
   需要实现内部流量和外部流量自动拆分
2. **实现**
   通过coredns的rewrite功能实现以上能力,如以下内部访问tenant.msa.chinamcloud.com域名时，会将流量转发到 tenantapi.yunjiao.svc.cluster.local域名，实现内外域名访问一致。
   部分版本nginx配置时候可能遇见无法访问的情况

```
apiVersion: v1
data:
  Corefile: |2-
    .:53 {
        errors
        health
        rewrite name tenant.msa.chinamcloud.com tenantapi.yunjiao.svc.cluster.local
        rewrite name console.msa.chinamcloud.com console.yunjiao.svc.cluster.local
        rewrite name user.msa.chinamcloud.com userapi.yunjiao.svc.cluster.local
        rewrite name lims.msa.chinamcloud.com lims.yunjiao.svc.cluster.local
        rewrite name labapp.msa.chinamcloud.com limsapp.yunjiao.svc.cluster.local
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system

```

### 场景四：coredns的hosts特性声明

 **hosts** ：字段部分指明了三个域名的解析地址 ， hosts是 CoreDNS 的一个 plugin，这一节的意思是加载 /etc/hosts文件里面的解析信息。hosts 在最前面，则如果一个域名在 hosts 文件中存在，则优先使用这个信息返回； 

 **fallthrough** ：如果 hosts 中找不到，则进入下一个 plugin 继续。缺少这一个指令，后面的 plugins 配置就无意义了；

> **注意** 请配置**fallthrough**，否则会造成非定制hosts域名解析失败。

```
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        hosts {
            100.64.139.66 minio.chinamcloud.com
            100.64.139.66 registry.chinamcloud.com
            100.64.139.66 gitlab.chinamcloud.com
            fallthrough
        }
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

###  场景五：外部域名完全使用自建DNS服务器

如果您需要使用的自建DNS服务的域名没有统一的域名后缀，您可以选择所有集群外部域名走自建DNS服务器，例如，您自建的DNS服务器IP为10.10.0.10和10.10.0.20，可以更改**forward**参数进行配置。示例配置如下

```
  Corefile: |2-
        .:53 {
            errors
            health
            kubernetes cluster.local. in-addr.arpa ip6.arpa {
                pods insecure
                upstream
                fallthrough in-addr.arpa ip6.arpa
            }
            prometheus :9153
            forward . 10.10.0.10 10.10.0.20   ######
            cache 30
            reload
            loadbalance
        }
```

### 场景六：集群外部访问集群内服务

如果您希望运行在集群节点上的进程能够访问到集群内的服务，虽然可以通过将节点的/etc/resolv.conf文件内nameserver配置为集群kube-dns的ClusterIP地址来达到目的，但不推荐您直接更改节点的/etc/resolv.conf文件的方式来达到任何目的



### 场景七：禁止CoreDNS对IPv6类型的AAAA记录查询返回

当业务容器不需要AAAA记录类型时，可以在CoreDNS中将AAAA记录类型拦截，返回域名不存在，以减少不必要的网络通信。示例配置如下：

```yaml
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 15s
        }
        #新增以下一行Template插件，其它数据请保持不变。
        template IN AAAA .
    
    }
```



### 场景八：local dns配置外部域名

1. 修改kube-system下的local dns 的configmap

```
kubectl edit cm node-local-dns  -nkube-system
```



1. 增加11行：

```
    tencentyun.com:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.20.10  COREDNS_CLUSTER_IP
        forward . __PILLAR__CLUSTER__DNS__ {
                force_tcp
        }
        prometheus :9253
        }
```

​    ![image-20220217130218574](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202305042140131.png)         

ipvs在使用node-local-dns-dnscache缺陷导致访问链路异常
规避措施：

1. node-local-dns 不要绑定 kube-dns service IP
2. kubelet 增加命令行参数 --cluster-dns=169.254.20.10







参考文档：https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/#coredns

https://github.com/coredns/coredns/blob/master/plugin/kubernetes/README.md

https://kubernetes.io/zh/docs/tasks/administer-cluster/dns-custom-nameservers/

**Coredns+Nodelocaldns cache解决Coredns域名解析延迟** 

https://blog.51cto.com/u_14143894/2515451

  **CoreDNS**  详情

https://blog.51cto.com/u_14299052/3104596

