---
title: Linux安装NodeJS
tags: Linux
categories: Linux
keywords: nodejs
description: Linux安装NodeJS
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031801736.jpeg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031801736.jpeg
abbrlink: fe4ed47c
date: 2018-05-25 20:10:42
updated: 2018-05-25 23:58:58
---

## Linux安装NodeJS

> Node.js  是一个基于Chrome V8 引擎 JavaScript 运行时环境， [nodejs 官网](https://nodejs.org/zh-cn/download/)

### 下载安装包

**1，可以在官网控制台下载上传至服务器**

![image-20220409115051104](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031802065.png)

**2，或者使用命令直接下载**

```
[root@xxx ~/nodejs]# wget https://nodejs.org/dist/v16.14.2/node-v16.14.2-linux-x64.tar.xz   // 下载
[root@xxx ~/nodejs]# tar xf  node-v16.14.2-linux-x64.tar.xz        // 解压
[root@xxx ~/nodejs]# cd nnode-v16.14.2-linux-x64/                  // 进入解压目录
```

>  [历史版本下载]( https://nodejs.org/dist/)  

### 使用RZ上传到服务器并解压

> Linux的sz和rz命令，可使用yum命令安装：yum install -y lrzsz

```
[root@xxx ~/nodejs]# pwd
/root/nodejs
[root@xxx ~/nodejs]# ls -lrt
total 21428
-rw-rw-rw- 1 root root 21941244 Apr  9 11:52 node-v16.14.2-linux-x64.tar.xz
[root@xxx ~/nodejs]# tar -xvf node-v16.14.2-linux-x64.tar.xz 
[root@xxx ~/nodejs]# cd /usr/local/
[root@xxx /usr/local]# ls
bin  etc  games  include  lib  lib64  libexec  lost+found  qcloud  sa  sbin  share  src
[root@xxx /usr/local]# mv /root/nodejs/node-v16.14.2-linux-x64 .
[root@xxx /usr/local]# mv node-v16.14.2-linux-x64/ nodejs 
```

### 配置环境变量

**方式一：环境变量**

　1）、加入环境变量，在 /etc/profile 文件末尾增加配置

```
vi /etc/profile
export PATH=$PATH:/usr/local/nodejs/bin
```

　2）、执行命令使配置文件生效

```
source /etc/profile
```

**方式二：软链接方式**

```
ln -s /usr/local/nodejs/bin/npm /usr/local/bin/
ln -s /usr/local/nodejs/bin/node /usr/local/bin/
```

### 验证

```
[root@xxx /usr/local]# node -v
v16.14.2
[root@xxx /usr/local]# npm -v
8.5.0
```

### Npm 更换淘宝镜像

```
npm config set registry https://registry.npm.taobao.org
npm install
```

