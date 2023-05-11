---
title: TKE集群中Nginx-ingress常用案例
abbrlink: 26d138ca
date: 2021-09-20 15:07:04
tags:
  - Nginx-ingress
  - Kubernetes
categories: Kubernetes
keywords:
  - Kubernetes
  - Nginx-ingress
description: NGINX Ingress Controller常用的案例介绍和使用
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181138830.jpeg
updated: 2022-09-06 23:58:58
---

##  一、Ingress简介 

> [nginx-ingress官方文档介绍](https://kubernetes.github.io/ingress-nginx/)

在Kubernetes中，服务和Pod的IP地址仅可以在集群网络内部使用，对于集群外的应用是不可见的。为了使外部的应用能够访问集群内的服务，在Kubernetes 目前 提供了以下几种方案：

- NodePort类型
- LoadBalancer类型
- Ingress

**1，Ingress组成**

- ingress controller：实际 Nginx 负载，会 watch kubernetes ingress 对象的变化更新在集群中，将新加入的Ingress转化成Nginx的配置文件并使之生效
- ingress服务：将Nginx的配置抽象成一个Ingress对象，每添加一个新的服务只需写一个新的Ingress的yaml文件即可

**2，Ingress工作原理**

1. ingress controller通过和kubernetes api交互，动态的去感知集群中ingress规则变化，
2. 然后读取它，按照自定义的规则，规则就是写明了哪个域名对应哪个service，生成一段nginx配置，
3. 再写到nginx-ingress-control的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器会把生成的nginx配置写入/etc/nginx.conf文件中，
4. 然后reload一下使配置生效，以此达到域名分配置和动态更新的问题。

**3，Ingress 可以解决什么问题**

- 动态配置服务

如果按照传统方式， 当新增加一个服务时，我们可能需要在流量入口加一个反向代理指向我们新的k8s服务. 而如果用了Ingress, 只需要配置好这个服务, 当服务启动时，自动注册到Ingress的中, 不需要额外的操作.

- 减少不必要的端口暴露

配置过k8s的都清楚，第一步是要关闭防火墙的, 主要原因是k8s的很多服务会以NodePort方式映射出去，这样就相当于给宿主机打了很端口， 既不安全也不优雅. 而Ingress可以避免这个问题, 除了Ingress自身服务可能需要映射出去，其他服务都不要用NodePort方式 

##  二、部署安装

见组件管理

##  三、相关案例

环境准备：

- 创建TKE集群
- 安装nginx-ingress组件
- 域名和证书

部署nginx服务用于测试和验证

```
[root@VM-249-130-tlinux ~]# kubectl -n  nginx-ingress get svc -o wide
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
nginx-v1   ClusterIP   172.16.252.224   <none>        80/TCP    13m     app=nginx,version=v1
nginx-v2   ClusterIP   172.16.252.208   <none>        80/TCP    13m     app=nginx,version=v2
nginx-v3   ClusterIP   172.16.254.220   <none>        80/TCP    7m40s   app=nginx,version=v3
nginx-v4   ClusterIP   172.16.255.165   <none>        80/TCP    6m13s   app=nginx,version=v4
[root@VM-249-130-tlinux ~]# curl http://172.16.252.224:80
nginx-v1
[root@VM-249-130-tlinux ~]# curl http://172.16.252.208:80
nginx-v2
[root@VM-249-130-tlinux ~]# curl http://172.16.254.220:80
nginx-v3
[root@VM-249-130-tlinux ~]# curl http://172.16.255.165:80
nginx-v4
```

表示访问对应的service 成功的

###  案例1 最简单基础配置

yaml示例如下：

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx
  namespace: nginx-ingress
spec:
  rules:
  - host: nginx.chen1900s.cn      #相当于定义了nginx的一个server_name
    http:
      paths:
      - backend:
          serviceName: nginx-v1  #定义后端的service
          servicePort: 80
        path: /                 #一个path就相当于一个location，path的值必须为“/”。这里为匹配的规则，根表示默认请求转发规则
        pathType: ImplementationSpecific
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /.*.(txt|css|doc)  #可以进入到ingress controller查看nginx的配置,这里相当于把结尾为txt,css,doc的url请求转发到nginx-v2 service
        pathType: ImplementationSpecific
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v3
          servicePort: 80
        path: /(api|app)/       #这里相当于将api和app开头的目录语法转发至nginx-v3 service
        pathType: ImplementationSpecific
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v4
          servicePort: 80
        path: /api           #这里相当于将api开头的url（可以是一个文件，也可以是一个目录）的请求，转发到nginx-4
        pathType: ImplementationSpecific
	#如果上面的都没匹配到，则默认转到“/”  也就是nginx-v1
```

执行以下命令，访问Nginx服务。

```bash
# curl nginx.chen1900s.cn    #默认转发规则
nginx-v1
# curl nginx.chen1900s.cn/nginx.txt  #结尾为txt,css,doc的url请求转发到nginx-v2 service
nginx-v2
# curl nginx.chen1900s.cn/nginx.css  #结尾为txt,css,doc的url请求转发到nginx-v2 service
nginx-v2
# curl nginx.chen1900s.cn/nginx.doc  #结尾为txt,css,doc的url请求转发到nginx-v2 service
nginx-v2
# curl nginx.chen1900s.cn/api/       #将api和app开头的目录语法转发至nginx-v3 service
nginx-v3
# curl nginx.chen1900s.cn/api/hello  #将api和app开头的目录语法转发至nginx-v3 service
nginx-v3
# curl nginx.chen1900s.cn/app/       #将api和app开头的目录语法转发至nginx-v3 service
nginx-v3
# curl nginx.chen1900s.cn/api        将api开头的url（可以是一个文件，也可以是一个目录）的请求，转发到nginx-4
nginx-v4
# curl nginx.chen1900s.cn/api111      将api开头的url（可以是一个文件，也可以是一个目录）的请求，转发到nginx-4
nginx-v4
# curl nginx.chen1900s.cn/app        #默认转发到/ nginx-v1上面
nginx-v1
```

 annotations配置作用于server 

> 指定了我们使用后端ingress controller的类别，如果后端有多个ingress controller的时候很重要
> kubernetes.io/ingress.class: "nginx"
> 指定我们的rules的path可以使用正则表达式，如果我们没有使用正则表达式，此项则可不使用
> nginx.ingress.kubernetes.io/use-regex: "true"

 说明：

> 上面定义的所有path到ingress controller都将会转换成nginx location规则，那么关于location的优先级与nginx一样，path转换到nginx后，会将path规则最长的排在最前面，最短的排在最后面。 

### 案例2 个性化配置

 在案例1的基础上面我们可以增加了annotations的一些配置 

```
kubernetes.io/ingress.class: "nginx"
nginx.ingress.kubernetes.io/use-regex: "true"

#连接超时时间，默认为5s
nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"

#后端服务器回转数据超时时间，默认为60s
nginx.ingress.kubernetes.io/proxy-send-timeout: "600"

#后端服务器响应超时时间，默认为60s
nginx.ingress.kubernetes.io/proxy-read-timeout: "600"

#客户端上传文件，最大大小，默认为20m
nginx.ingress.kubernetes.io/proxy-body-size: "10m"
```

###  案例3 配置URL重定向rewrite-target

使用Nginx Ingress Controller的时候，Nginx会将路径完整转发到后端（如，从Ingress访问的/service1/api路径会直接转发到后端Pod的/service1/api/路径）。如果您后端的服务路径为/api，则会出现路径错误，导致404的情况。该情况下，您可以通过配置`rewrite-target`的方式，来将路径重写至需要的目录

```
nginx.ingress.kubernetes.io/rewrite-target: /$2
```

完整配置如下

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: nginx
  namespace: nginx-ingress
spec:
  rules:
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v1
          servicePort: 80
        path: /svc(/|$)(.*)
        pathType: ImplementationSpecific

```



###  案例4 rewrite配置二

匹配请求头，主要用于根据请求头信息将用户请求转发到不同的应用，比如根据不同的客户端转发请求

- `nginx.ingress.kubernetes.io/server-snippet`：扩展配置到Server章节。
- `nginx.ingress.kubernetes.io/configuration-snippet`：扩展配置到Location章节。

```
annotations:
     nginx.ingress.kubernetes.io/server-snippet: |
         rewrite ^/v4/(.*)/card/query http://www.chen1900s.cn/v5/#!/card/query permanent;
     nginx.ingress.kubernetes.io/configuration-snippet: |
         rewrite ^/v6/(.*)/card/query http://www.chen1900s.cn/v7/#!/card/query permanent;
```

示例配置生成的nginx.conf如下所示

```
## start server www.chen1900s.cn
    server {
        server_name www.chen1900s.cn ;
        listen 80;
        listen [::]:80;
        set $proxy_upstream_name "-";
    ### server-snippet配置。
        rewrite ^/v4/(.*)/card/query http://www.chen1900s.cn/v5/#!/card/query permanent;
        ...
    ### configuration-snippet配置。
      rewrite ^/v6/(.*)/card/query http://www.chen1900s.cn/v7/#!/card/query permanent;
      ...
    }
    ## end server www.chen1900s.cn
```

这里直接使用了“nginx.ingress.kubernetes.io/server-snippet”来指定配置，这里可以直接写nginx的配置，通过这里可以不止是实现rewrite重写，还可以实现更多的功能需求，只要是作用于server的都可以 

### 案例5 域名登录认证

有时候我们的服务没有提供登录认证，但是有不希望将服务提供给所有的人都能访问，那么可以通过ingress上的认证控制访问，常用的2种认证方式。 

**1，基本身份认证**

创建secret 用于访问凭证

```bash
[root@VM-0-17-tlinux ~/nginx-ingress]# htpasswd -c auth admin
New password: 
Re-type new password: 
Adding password for user admin
#创建secret
[root@VM-0-17-tlinux ~/nginx-ingress]#  kubectl create secret generic basic-auth --from-file=auth -n nginx-ingress
secret/basic-auth created
```

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "true"
    nginx.ingress.kubernetes.io/auth-realm: Authentication Required - admin #请求用户名
    nginx.ingress.kubernetes.io/auth-secret: basic-auth  #对应的secret
    nginx.ingress.kubernetes.io/auth-type: basic  #认证方式
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx-basic-auth
  namespace: nginx-ingress
spec:
  rules:
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v1
          servicePort: 80
        path: /

```

验证

```bash
#再不输入认真情况下，访问会出现 401 Unauthorized
[root@VM-249-130-tlinux ~/nginx-ingress]# curl -kI https://nginx.chen1900s.cn/ 
HTTP/1.1 401 Unauthorized
Date: Sun, 18 Sep 2022 07:10:24 GMT
Content-Type: text/html
Content-Length: 172
Connection: keep-alive
WWW-Authenticate: Basic realm="Authentication Required - admin"
Strict-Transport-Security: max-age=15724800; includeSubDomains

#携带访问凭证访问
[root@VM-249-130-tlinux ~/nginx-ingress]# curl -kI https://nginx.chen1900s.cn/   -u 'admin:admin123'
HTTP/1.1 200 OK
Date: Sun, 18 Sep 2022 07:11:32 GMT
Content-Type: text/plain
Connection: keep-alive
Strict-Transport-Security: max-age=15724800; includeSubDomains
```

**2.2 外部身份验证**

有时候我们有自己的鉴权中心，也是可以使用外部身份进行认证的，这里我们采用https://httpbin.org/basic-auth/user/passwd这个作为外部身份，这个默认账号和密码user/passwd

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "true"
    nginx.ingress.kubernetes.io/auth-url: https://httpbin.org/basic-auth/user/passwd
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx-basic-auth-out
  namespace: nginx-ingress
spec:
  rules:
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v1
          servicePort: 80
        path: /
```

```
# curl -k https://nginx.chen1900s.cn/  -v -H 'Host: nginx.chen1900s.cn'  -u 'user:passwd'
```

![image-20220918151952442](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181519636.png)

### 案例6 访问白名单

有时候我们需要给域名配置下访问白名单，只希望部分ip可以访问我的服务，这时候需要用到ingress的whitelist-source-range，我们可以通过这个注解来配置我们希望放通访问的ip。下面我们只放通81.69.221.19 也可以指定某一网段 可以访问 

> 该案例需要nginx-ingress能够正常获取到客户端源IP，nginx-ingress-controller 对应的service需要是local模式 或者直连POD模式

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "true"
    nginx.ingress.kubernetes.io/whitelist-source-range: 81.69.221.19
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: nginx-whitelist-ip
  namespace: nginx-ingress
spec:
  rules:
  - host: nginx.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
```

源IP未81.69.221.19正常访问

```bash
[root@VM-0-33-tlinux ~]# curl myip.ipip.net
当前 IP：81.69.221.19  来自于：中国 上海 上海  电信
[root@VM-0-33-tlinux ~]# curl http://nginx.chen1900s.cn
nginx-v2
```

其他客户端禁止访问

```bash
[root@172-16-155-8 ~]# curl myip.ipip.net
当前 IP：121.5.26.195  来自于：中国 上海 上海  电信
[root@172-16-155-8 ~]# curl http://nginx.chen1900s.cn
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

![image-20220918153106012](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181531107.png)

### 案例7  永久重定向配置重定向错误码

 redirect主要用于域名重定向，比如访问a.com被重定向到b.com。 

```
nginx.ingress.kubernetes.io/permanent-redirect: https://www.baidu.com
nginx.ingress.kubernetes.io/permanent-redirect-code: "308"
```

### 案例8 客户端请求body的大小

如果遇到请求报错是 413 Request Entity Too Large 

可以配置客户端请求body的大小，创建 ingress 时添加 annotations（注释）

```
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
```

### 案例9  414 Request URI too large或400 bad request错

如遇到调用后端接口时候，需要在header中传一段很长的token，会报"414 Request URI too large"，可以登陆nginx-ingress-controller pod里查看配置

![image-20220918155331525](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181553609.png)

解决方法是修改两个参数

参数一：

```bash
#client_header_buffer_size：客户端请求头缓冲区大小，
client_header_buffer_size 128k;#如果请求头总长度大于小于128k，则使用此缓冲区
```

参数二：

```bash
#large_client_header_buffers：请求头总长度大于128k时使用large_client_header_buffers设置的缓存区
large_client_header_buffers 4 128k;
#large_client_header_buffers 指令参数4为个数，128k为大小，默认是8k。申请4个128k。
```



```yaml
apiVersion: v1
data:
  client-header-buffer-size: 128k      #注意参数是中横线
  large-client-header-buffers: 4 128k   #注意参数是中横线
  allow-backend-server-header: "true"
  enable-underscores-in-headers: "true"
```

![image-20220918155742919](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181557022.png)

### 案例10 上传超时 504：Gateway Timeout

```yaml
metadata:
  annotations:
　　 nginx.ingress.kubernetes.io/proxy-connect-timeout："300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    
    
#连接超时时间，默认为5s
nginx.ingress.kubernetes.io/proxy-connect-timeout: "300"
#后端服务器回转数据超时时间，默认为60s
nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
#后端服务器响应超时时间，默认为60s
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
```



### 案例11  白名单及请求速率限制

 可以限制速率来降低后端压力，比如如下配置： 

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-nginx
  namespace: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/limit-rate: "100K"
    nginx.ingress.kubernetes.io/limit-whitelist: 81.69.221.19
    nginx.ingress.kubernetes.io/limit-rps: "1"  #为每秒1个连接数
    nginx.ingress.kubernetes.io/limit-rpm: "5"  #单个IP每分钟的连接数
spec:
  rules:
  - host: nginx.chen1900s.cn 
    http:
      paths:
      - path: 
        backend:
          serviceName: nginx-v4
          servicePort: 80
```

用以设置基于流量、请求连接数、请求频率的访问控制。访问控制配置说明如下表所示。 

|                     注解                      | 类型/选项 |                           功能描述                           |
| :-------------------------------------------: | :-------: | :----------------------------------------------------------: |
|    nginx.ingress.kubernetes.io/limit-rate     |  number   |        访问流量速度限制，同 Nginx 配置指令 limit_rate        |
| nginx.ingress.kubernetes.io/limit-rate-after  |  number   | 启用访问流量速度限制的最大值，同 Nginx 配置指令 limit_rate_after |
| nginx.ingress.kubernetes.io/limit-connections |  number   |        节并发连接数限制，同 Nginx 配置指令 limit_conn        |
|     nginx.ingress.kubernetes.io/limit-rps     |  number   | 每秒请求频率限制，burst 参数为给定值的 5 倍，响应状态码由 ConfigMap 的 limit-req-status-code 设定 |
|     nginx.ingress.kubernetes.io/limit-rpm     |  number   | 每分钟请求频率限制，burst 参数为给定值的 5 倍，响应状态码由 ConfigMap 的 limit-req-status-code 设定 |
|  nginx.ingress.kubernetes.io/limit-whitelist  |   CIDR    |                对以上限制设置基于 IP 的白名单                |

![image-20220918162246526](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181622635.png)

### 案例12 配置URL重定向的路由服

通过以下命令创建一个简单的Ingress，所有对/svc路径的访问都会重新定向到后端服务能够识别的/路径上面。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /$1
  name: rewrite-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: rewrite-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v1
          servicePort: 80
        path: /svc/(.*)


```

执行以下命令，访问Nginx服务，替换**IP_ADDRESS**为Ingress对应的I

```bash
#curl -k -H "Host: rewrite-test-ingress.com"  http://<IP_ADDRESS>/svc/foo

# curl -k -H "Host: rewrite-nginx-ingress.com"  http://118.24.224.221/svc/foo
nginx-v1
```

![image-20220918162633115](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181626193.png)

### 案例13 配置安全的路由服务

支持多证书管理，为您的服务提供安全防护。

1，准备您的服务证书。如果没有证书，可以通过下面的方法生成测试证书。

```bash
  说明** 域名需要与您的Ingress配置保持一致。 **
```

- 执行以下命令，生成一个证书文件tls.crt和一个私钥文件tls.key

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=tls-nginx-ingress.com/O=tls-nginx-ingress.com"
```

- 执行以下命令，创建密钥。

  通过该证书和私钥创建一个名为tls-nginx-ingress的Kubernetes Secret。创建Ingress时需要引用这个Secret。

  ```bash
  kubectl create secret tls tls-nginx-ingress --key tls.key --cert tls.crt  -n nginx-ingress
  ```

2，执行以下命令，创建一个安全的Ingress服务。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: tls-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: tls-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
  tls:
  - hosts:
    - tls-nginx-ingress.com
    secretName: tls-nginx-ingress
```

3，执行以下命令，查询Ingress信息。

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  get ingress -n nginx-ingress | grep tls
tls-nginx-ingress       <none>   tls-nginx-ingress.com       114.117.219.97   80, 443   4m3s
```

4，配置`hosts`文件或者设置域名来访问该TLS服务

![image-20211021151855901](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130053.png)

![image-20211125174832750](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130095.png)

### 案例14 配置HTTPS双向认证

某些业务场景需要启用HTTPS双向验证，Ingress-Nginx支持该特性，配置步骤参考以下示例。

1，执行以下命令，创建自签的CA证书。 

```bash
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj '/CN=Fern Cert Authority'
```

2，执行以下命令，创建Server端证书。

执行以下命令，生成Server端证书的请求文件。

```bash
openssl req -new -newkey rsa:4096 -keyout server.key -out server.csr -nodes -subj '/CN=test.nginx.ingress.com'
```

 执行以下命令，使用根证书签发Server端请求文件，生成Server端证书

```bash
openssl x509 -req -sha256 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt
```

3，执行以下命令，创建Client端证书。

  生成Client端证书的请求文件

```bash
openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=Fern'
```

 执行以下命令，使用根证书签发Client端请求文件，生成Client端证书。

```bash
openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
```

4，执行以下命令，检查创建的证书

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# ls
ca.crt  ca.key  client.crt  client.csr  client.key  server.crt  server.csr  server.key
```

![image-20211021160303691](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130150.png)

5，执行以下命令，创建CA证书的Secret。

```bash
kubectl create secret generic ca-secret --from-file=ca.crt=ca.crt  -n nginx-ingress
```

6，执行以下命令，创建Server证书的Secret。

```bash
kubectl create secret tls  tls-secret --cert server.crt --key server.key  -n nginx-ingress

```

7，执行以下命令，创建测试用的Ingress用例。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    description: 配置HTTPS双向认证
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "false"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
    nginx.ingress.kubernetes.io/auth-tls-secret: nginx-ingress/ca-secret
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
  name: test-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: test.nginx.ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-a
          servicePort: 80
        path: /

```

8，执行以下命令，查看Ingress的IP地址

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  get ingress -n nginx-ingress | grep test
test-nginx-ingress      <none>   test.nginx.ingress.com      114.117.219.97   80        3m34s
```

9，执行以下命令，更新Hosts文件，替换下面的IP地址为真实获取的Ingress的IP地址。

```bash
echo "114.117.219.97 test.nginx.ingress.com" >> /etc/hosts
```

**结果验证**

客户端不传证书访问

```bash
curl --cacert ./ca.crt  https://test.nginx.ingress.com
#预期输出
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl --cacert ./ca.crt  https://test.nginx.ingress.com
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx</center>
</body>
</html>
```

客户端传证书访问

```bash
curl --cacert ./ca.crt --cert ./client.crt --key ./client.key https://test.nginx.ingress.com
#预期输出
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl --cacert ./ca.crt --cert ./client.crt --key ./client.key https://test.nginx.ingress.com
nginx-a hello world
```

![image-20211021163648495](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130187.png)



### 案例15 配置域名支持正则化

在Kubernetes集群中，Ingress资源不支持对域名配置正则表达式，但是可以通过`nginx.ingress.kubernetes.io/server-alias`注解来实现

1，创建Ingress，以正则表达式`~^www\.\d+\.example\.com`为例。

```bash
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    description: 配置域名支持正则化
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/server-alias: ~^www\.\d+\.example\.com$, abc.example.com
  name: regex-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: regex-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-b
          servicePort: 80
        path: /
```

2，执行以下命令，查看对应Nginx Ingress Controller的配置。

​      执行以下命令，查看部署Nginx Ingress Controller服务的Pod。

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  get pods -n kube-system  | grep nginx-ingress-nginxnginx-ingress-nginx-controller-5ddf7ccc4f-vss4f                   1/1     Running   0          5h13m
```

​    执行以下命令，查看对应Nginx Ingress Controller的配置，可以发现生效的配置（Server_Name字段）

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  -n kube-system exec nginx-ingress-ingress-nginx-controller-7dc5fd97f-t9l9l cat /etc/nginx/nginx.conf | grep -C3  "regex-nginx-ingress.com"
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
        }
        ## end server _

        ## start server regex-nginx-ingress.com
        server {
                server_name regex-nginx-ingress.com abc.example.com ~^www\.\d+\.example\.com$ ;

                listen 80  ;
                listen 443  ssl ;
--
                }

        }
        ## end server regex-nginx-ingress.com

        ## start server rewrite-nginx-ingress.com
        server {
```

3，执行以下命令，获取Ingress对应的IP。

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl get ingress -n nginx-ingress  | grep regex
regex-nginx-ingress     <none>   regex-nginx-ingress.com     114.117.219.97   80        12m
```

4，执行以下命令，进行不同规则下的服务访问测试，配置以下**IP_ADDRESS**为上一步获取的IP地址。

执行以下命令，通过`Host: regex-nginx-ingress.com `访问服务

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host: regex-nginx-ingress.com"  114.117.219.97/

nginx-b
```

执行以下命令，通过`Host: www.123.example.com`访问服务

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host: www.123.example.com"  114.117.219.97/
nginx-b
```

执行以下命令，通过`Host: www.321.example.com`访问服务

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host:  www.321.example.com"  114.117.219.97/
nginx-b
```

![image-20211021170658628](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130222.png)



### 案例16 配置域名支持泛化

在Kubernetes集群中，Ingress资源支持对域名配置泛域名，例如，可配置`*.ingress-regex.com`泛域名。

1，部署以下模板，创建Ingress

```bash
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: regex-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: '*.regex-ingress.com'
    http:
      paths:
      - backend:
          serviceName: nginx-c
          servicePort: 80
        path: /

```

2，执行以下命令，查看对应Nginx Ingress Controller的配置，可以发现生效的配置（Server_Name字段）

```bash
kubectl exec -n kube-system <ningx-ingress-pod-name> cat /etc/nginx/nginx.conf | grep -C3 "regex-ingress.com"

[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  -n kube-system exec nginx-ingress-ingress-nginx-controller-7dc5fd97f-t9l9l cat /etc/nginx/nginx.conf | grep -C3  "regex-ingress.com"
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

        # Global filters

        ## start server *.regex-ingress.com
        server {
                server_name ~^(?<subdomain>[\w-]+)\.regex-ingress\.com$ ;

--
                }

        }
        ## end server *.regex-ingress.com

        ## start server _
        server {
```

执行以下命令，获取Ingress对应的IP

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# kubectl  get ingress -nnginx-ingress | grep regex-ingress.com

regex-ingress           <none>   *.regex-ingress.com         114.117.219.97   80        10m
```

4，执行以下命令，进行不同规则下的服务访问测试，配置以下**IP_ADDRESS**为上一步获取的IP地址。

执行以下命令，通过Host: abc.regex-ingress.com 

访问服务。

```bash
# curl -H "Host: abc.regex-ingress.com" <IP_ADDRESS>/

[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host: abc.regex-ingress.com"  114.117.219.97/
nginx-c
```

预期输出：

```bash
nginx-c
```

执行以下命令，通过`Host: 123.regex-ingress.com `访问服务。 

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host: 123.regex-ingress.com"  114.117.219.97/
nginx-c
```

 执行以下命令，通过`Host: a1b1.regex-ingress.com`访问服务。

```bash
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -H "Host: ab1.regex-ingress.com"  114.117.219.97/
nginx-c
```

![image-20211021173344117](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130279.png)



### 案例17 cookie会话保持

nginx.ingress.kubernetes.io/session-cookie-name

默认为round-robin，在具体ingress资源中通过ingress metadata.annotations字段可具体设置

通过会话cookie进行一致性hash均衡算法

```yaml
ingress.kubernetes.io/affinity: "cookie"
ingress.kubernetes.io/session-cookie-name: "route"
ingress.kubernetes.io/session-cookie-hash: "sha1"
```

通过客户端ip进行一致性hash的均衡算法

```yaml
nginx.ingress.kubernetes.io/upstream-hash-by: "${remote_addr}"
```

 通过请求uri进行一致性hash的均衡算法

```yaml
nginx.ingress.kubernetes.io/upstream-hash-by: "${request_uri}"
```



### 案例18 nginx-ingress 开启跨域(CORS)

> [参考官方文档](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#enable-cors)

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress
    nginx.ingress.kubernetes.io/cors-allow-headers: >-
      DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization
    nginx.ingress.kubernetes.io/cors-allow-methods: 'PUT, GET, POST, OPTIONS'
    nginx.ingress.kubernetes.io/cors-allow-origin: '*'
    nginx.ingress.kubernetes.io/enable-cors: 'true'
    nginx.ingress.kubernetes.io/service-weight: ''
  name: cors-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: cors-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
  tls:
  - hosts:
    - tls-nginx-ingress.com
    secretName: tls-nginx-ingress
```

跨域访问功能配置说明如下表所示。

|                        注解                        |     类型      |                           功能描述                           |
| :------------------------------------------------: | :-----------: | :----------------------------------------------------------: |
|      nginx.ingress.kubernetes.io/enable-cors       | true 或 false |              是否启用跨域访问支持，默认为 false              |
|   nginx.ingress.kubernetes.io/cors-allow-origin    |    string     |     允许跨域访问的域名，默认为 *，表示接受任意域名的访问     |
|   nginx.ingress.kubernetes.io/cors-allow-methods   |    string     | 允许跨域访问方法，默认为 GET、PUT、POST、DELETE、PATCH、OPTIONS |
|   nginx.ingress.kubernetes.io/cors-allow-headers   |    string     | 允许跨域访问的请求头，默认为 DNT，X-CustomHeader、Keep-Alive、User-Agent、X-Requested-With、If-Modified-Since、Cache-Control、Content-Type、Authorization |
| nginx.ingress.kubernetes.io/cors-allow-credentials | true 或 false | 设置在响应头中 Access-Control-Allow-Credentials 的值，设置是否允许客户端携带验证信息，如 cookie 等，默认为 true |
|      nginx.ingress.kubernetes.io/cors-max-age      |    number     | 设置响应头中 Access-Control-Max-Age 的值，设置返回结果可以用于缓存的最长时间，默认为 1728000 秒 |



### 案例19 nginx-ingress关闭80强制跳转443

默认情况下，如果ingress对象入口启用了TLS，则ingress-controller将使用308永久重定向响应将HTTP客户端重定向到HTTPS端口443

1，执行以下命令，创建一个安全的Ingress服务(**带TLS证书** )

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress
  name: tls-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: tls-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
  tls:
  - hosts:
    - tls-nginx-ingress.com
    secretName: tls-nginx-ingress
```

2，执行以下命令，返回的是308信息。

```
[root@VM-0-17-tlinux ~/tls/nginx-ingress]# curl -I  http://tls-nginx-ingress.com
HTTP/1.1 308 Permanent Redirect
Date: Thu, 25 Nov 2021 10:07:30 GMT
Content-Type: text/html
Content-Length: 164
Connection: keep-alive
Location: https://tls-nginx-ingress.com
```

3，可以在特定ingress资源的metadata.annotations中通过配置nginx.ingress.[kubernetes](https://so.csdn.net/so/search?from=pc_blog_highlight&q=kubernetes).io/ssl-redirect: "false" 使用注释禁用此功能，关闭80强制跳转443

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'   #就可以禁止http强制跳转至https
  name: tls-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: tls-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
  tls:
  - hosts:
    - tls-nginx-ingress.com
    secretName: tls-nginx-ingress
```



4，添加annotations后再去访问，就可以正常访问80端口

```
[root@VM-0-17-tlinux ~]# curl -I  http://tls-nginx-ingress.com
HTTP/1.1 200 OK
Date: Thu, 25 Nov 2021 10:12:47 GMT
Content-Type: text/plain
Connection: keep-alive

[root@VM-0-17-tlinux ~]# curl   http://tls-nginx-ingress.com
nginx-v2
```

![image-20211125181623891](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130320.png)



### 案例20 nginx-ingress中ingress匹配优先级

首先在nginx中location的匹配优先级大致为：精准匹配 > 前缀匹配 > 正则匹配> /

其中，前缀匹配：^~，精准匹配 =，正则匹配细分为：

~ 区分大小写（大小写敏感）匹配成功；~* 不区分大小写匹配成功；!~ 区分大小写匹配失败；!~*  不区分大小写匹配失败

而ingress资源对象中，spec.rules.http.paths.path字段默认只支持不区分大小写的正则匹配，但前提需要设置nginx.ingress.kubernetes.io/use-regex注释设置

为true(默认值为false)来启用此功能

1，根据如下yaml文件创建ingress资源

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx-ingress"
    nginx.ingress.kubernetes.io/use-regex: "true"
  namespace: nginx-ingress
  name: use-regex-nginx-ingress
spec:
  rules:
    - host: use-regex-nginx-ingress.com
      http:
        paths:
          - path: /
            backend:
              serviceName: nginx-v1
              servicePort: 80
          - path: /wifi
            backend:
              serviceName: nginx-v2
              servicePort: 80

```

2，进入到ingress-controller的pod中，观察nginx配置文件，发现location的**正则匹配已生效**  

```
root@VM-0-17-tlinux ~]# kubectl  exec -it  nginx-ingress-ingress-nginx-controller-7dc5fd97f-t9l9l  -n  kube-system  /bin/bash
```

![image-20211125184440501](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130362.png)

![image-20211125184457459](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130393.png)

3，如果创建时候nginx.ingress.kubernetes.io/use-regex: "false"  或者不设置验证下效果

```
[root@VM-0-17-tlinux ~]# kubectl  exec -it  nginx-ingress-ingress-nginx-controller-7dc5fd97f-t9l9l  -n  kube-system  /bin/bash
```

![image-20211125185055225](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130429.png)

![image-20211125185129326](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130470.png)

### 案例21 **支持websocket配置** 

1，准备服务证书。如果没有证书，可以通过下面的方法生成测试证书。

      说明** 域名需要与您的Ingress配置保持一致。 **

执行以下命令，生成一个证书文件tls.crt和一个私钥文件tls.key

```
openssl req -x509 -nodes -days 1000 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=websocket-nginx-ingress.com/O=websocket-nginx-ingress.com"
```

执行以下命令，创建密钥。

通过该证书和私钥创建一个名为tls-nginx-ingress的Kubernetes Secret。创建Ingress时需要引用这个Secret。

```
kubectl create secret tls websocket-nginx-ingress --key tls.key --cert tls.crt  -n nginx-ingress
```

2，执行以下命令，创建一个安全的Ingress服务 (由于没有websocket，暂时没有做验证)

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-ingress
    nginx.ingress.kubernetes.io/configuration-snippet:  >-
    nginx.ingress.kubernetes.io/proxy-read-timeout 3600;
    nginx.ingress.kubernetes.io/proxy-send-timeout 3600;
  name: websocket-nginx-ingress
  namespace: nginx-ingress
spec:
  rules:
  - host: websocket-nginx-ingress.com
    http:
      paths:
      - backend:
          serviceName: nginx-v2
          servicePort: 80
        path: /
  tls:
  - hosts:
    - websocket-nginx-ingress.com
    secretName: websocket-nginx-ingress

```

### 案例22 通过configmap 定义全局常规参数 

```
data:
  multi_accept: on;
  use: epoll;
  user: www;
  worker_connections: 65535;
  worker_cpu_affinity: auto;
  worker_processes: auto;
  worker_rlimit_nofile: 300000;
  
  # 把真实IP地址传给后端
  compute-full-forwarded-for: "true"
  forwarded-for-header: "X-Forwarded-For"
  use-forwarded-headers: "true"
  
  # 关闭版本显示
  server-tokens: "false"
  
  # 客户端请求头的缓冲区大小 
  client-header-buffer-size: "512k"
  # 设置用于读取大型客户端请求标头的最大值number和size缓冲区
  large-client-header-buffers: "16 512k"
  
  # 读取客户端请求body的缓冲区大小
  client-body-buffer-size: "968k"
  # 代理缓冲区大小
  proxy-buffer-size: "1024k"
  # 代理body大小
  proxy-body-size: "50m"
  # 服务器名称哈希大小
  server-name-hash-bucket-size: "128"
  # map哈希大小
  map-hash-bucket-size: "128"
  # SSL加密套件
  ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA"
  # ssl 协议
  ssl-protocols: "TLSv1 TLSv1.1 TLSv1.2"
#定义json 访问日志格式
log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "x-forward-for": "$proxy_add_x_forwarded_for", "request_id": "$req_id", "remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status":$status, "vhost": "$host", "request_proto": "$server_protocol", "path": "$uri", "request_query": "$args", "request_length": $request_length, "duration": $request_time,"method": "$request_method", "http_referrer": "$http_referer", "http_user_agent": "$http_user_agent"}'

```

更多配置可以参考[文档](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/)

### 案例23 **获取真实IP地址配置**

nginx-ingress官方是通过修改容器的配置文件来配置，配置文件：ingress-nginx/ingress-nginx-controller

```
kubectl  edit   cm  -n kube-system nginx-ingress-nginx-controller -o yaml
```

```
compute-full-forwarded-for: "true"
forwarded-for-header: "X-Forwarded-For"
use-forwarded-headers: "true"
```

> compute-full-forwarded-for: "true"  //如果为真，NGINX 将传入的X-Forwarded-*标头传递给上游。当 NGINX 在设置这些标头的另一个 L7 代理/负载均衡器之后使用此选项
>
> forwarded-for-header: X-Forwarded-For //设置用于标识客户端的原始 IP 地址的标头字段。后端程序就可以在http 包的header里的X-Forwarded-For获取真实ip
>
> use-forwarded-headers: "true"   //将远程地址附加到 X-Forwarded-For 标头，而不是用pod或者其它cookie信息替换它。启用此选项后，应用程序负责根据自己的受信任代理列表来提取客户端 IP

保存后立即生效。随后ingress的添加真实的IP行为会与RFC一样都依次添加到X-Forwarded-For中了

> 在TKE集群环境中，想获取客户端源IP，需要nginx-ingress-controller 是local模式或者POD直连模式才可以，不需要修改配置文件

**local模式**

nginx-ingress-controller-service 配置

![image-20220918170155658](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181701765.png)



验证 ingress示例：

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.rule-mix: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
  name: whoami
  namespace: nginx-ingress
spec:
  rules:
  - host: whoami.chen1900s.cn
    http:
      paths:
      - backend:
          serviceName: whoami
          servicePort: 80
        path: /
        pathType: ImplementationSpecific

```

![image-20220918165856854](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181658981.png)

直连POD模式参考[TKE官方文档](https://cloud.tencent.com/document/product/457/48949)



### 案例24  nginx-ingress做tcp/udp4层网络转发

k8s集群通过nginx-ingress做tcp\udp 4层网络转发

1，检查nginx-ingress是否开启tcp\udp转发

```
      - args:
        - --tcp-services-configmap=kube-system/nginx-ingress-nginx-tcp
        - --udp-services-configmap=kube-system/nginx-ingress-nginx-udp
```

2，示例 kuard-demo.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  namespace: nginx-ingress
spec:
  selector:
    matchLabels:
      app: kuard
  replicas: 1
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: ccr.ccs.tencentyun.com/chenjingwei/kuard-amd64:v1
        imagePullPolicy: Always
        name: kuard
        ports:
        - containerPort: 8080
        
---
apiVersion: v1
kind: Service
metadata:
  name: kuard
  namespace: nginx-ingress
spec:
  ports:
  - port: 9527
    targetPort: 8080
    protocol: TCP
  selector:
    app: kuard
```

3，需要修改下configmap

```
# kubectl  -n kube-system get cm  | grep nginx-ingress-nginx
nginx-ingress-nginx-controller                      9      133d
nginx-ingress-nginx-tcp                             0      133d
nginx-ingress-nginx-udp                             0      133d

# kubectl  -n kube-system edit  cm nginx-ingress-nginx-tcp

[root@VM-0-17-tlinux ~]# kubectl  -n kube-system get  cm nginx-ingress-nginx-tcp -o yaml
apiVersion: v1
data:                                   #TKE默认么有data
  "9527": nginx-ingress/kuard:9527      #添加这个配置
kind: ConfigMap
metadata:
  labels:
    k8s-app: nginx-ingress-nginx-tcp
    qcloud-app: nginx-ingress-nginx-tcp
  name: nginx-ingress-nginx-tcp
  namespace: kube-system

```

4，进入nginx-ingress容器查看TCP services处会出现对应的负载配置



```
# kubectl  -n kube-system  exec -it nginx-ingress-nginx-controller-5ddf7ccc4f-v4pzp -- /bin/sh

vi  nginx.conf  镜像过滤
```

```
 
 
 # TCP services            
                                  
        server {
                preread_by_lua_block {
                        ngx.var.proxy_upstream_name="tcp-nginx-ingress-kuard-9527";
                }                                 

                listen                  9527;
                                                          
                listen                  [::]:9527;
                                             
                proxy_timeout           600s;  
                proxy_pass              upstream_balancer;
                                               
        }
```



5，编辑nginx-ingress-nginx-controller  svc 添加对应端口

![image-20220303172655193](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130616.png)



```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.cloud.tencent.com/direct-access: "false"
  labels:
    k8s-app: nginx-ingress-nginx-controller
    qcloud-app: nginx-ingress-nginx-controller
  name: nginx-ingress-nginx-controller
  namespace: kube-system
spec:
  clusterIP: 172.18.248.35
  externalTrafficPolicy: Cluster
  ports:
  - name: 80-80-tcp
    nodePort: 31899
    port: 80
    protocol: TCP
    targetPort: 80
  - name: 443-443-tcp
    nodePort: 32534
    port: 443
    protocol: TCP
    targetPort: 443
  - name: 9527-9527-tcp-5q8prs0zx68
    nodePort: 32677
    port: 9527
    protocol: TCP
    targetPort: 9527
  selector:
    k8s-app: nginx-ingress-nginx-controller
    qcloud-app: nginx-ingress-nginx-controller
  sessionAffinity: None
  type: LoadBalancer
,
```

然后通过nginx-ingress-nginx-controller   的svc  clb访问

```
[root@VM-0-17-tlinux ~]# kubectl  -n kube-system  get svc   | grep  nginx-ingress-nginx-controller
nginx-ingress-nginx-controller                     LoadBalancer   172.18.248.35    118.24.224.251   80:31899/TCP,443:32534/TCP     3m3s
nginx-ingress-nginx-controller-admission           ClusterIP      172.18.251.207   <none>           443/TCP                        133d
```

![image-20220303173434838](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209181130653.png)



### 案例25   Nginx-Ingress 实现grpc转发

参考文档：https://cloud.tencent.com/developer/article/1730604



### 案例26 配置HTTPS服务转发到后端容器为HTTPS协议

Nginx Ingress Controller默认使用HTTP协议转发请求到后端业务容器。当您的业务容器为HTTPS协议时，可以通过使用注解`nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`来使得Nginx Ingress Controller使用HTTP协议转发请求到后端业务容器。

Nginx Ingress配置示例如下：

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
  name: backend-protocol-https-ingress
  namespace: default
spec:
  rules:
  - host: example.chen1900s.cn
    http:
      paths:
      - backend:
          service:
            name: nginx
            port:
              number: 443
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - example.chen1900s.cn
    secretName: chen1900s

```

