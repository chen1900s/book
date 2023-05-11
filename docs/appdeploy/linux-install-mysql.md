---
title: Linux安装mysql数据库
tags:
  - Mysql
categories: Mysql
keywords:
  - Mysql
description: Linux安装mysql数据库
top_img: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209031854794.jpg
cover: >-
  https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202209041750814.jpeg
abbrlink: 3b301983
date: 2019-03-14 10:40:17
updated: 2019-05-28 23:58:58
---

## Linux安装MySQL数据库

### 1. 环境准备

- 云服务器或者虚拟机

- Linux的版本为CentOS7或者Redhat;


### 2. 下载安装包

​        [下载地址](https://downloads.mysql.com/archives/community/)

![image-20230314104539009](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202303141045132.png) 

### 3. 上传安装包

 

### 4. 创建目录&并解压

```bash
mkdir mysql

tar -xvf mysql-8.0.26-1.el7.x86_64.rpm-bundle.tar -C mysql
```

![image-20230314104754614](https://chen1900s-1257020962.cos.ap-chongqing.myqcloud.com/my-blog/image/202303141047704.png)

### 5. 安装mysql的安装包

```bash
cd mysql

rpm -ivh mysql-community-common-8.0.26-1.el7.x86_64.rpm 
rpm -ivh mysql-community-client-plugins-8.0.26-1.el7.x86_64.rpm 
rpm -ivh mysql-community-libs-8.0.26-1.el7.x86_64.rpm 
rpm -ivh mysql-community-libs-compat-8.0.26-1.el7.x86_64.rpm
yum install openssl-devel
rpm -ivh  mysql-community-devel-8.0.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.26-1.el7.x86_64.rpm
rpm -ivh  mysql-community-server-8.0.26-1.el7.x86_64.rpm

#安装时候 可能需要有些依赖安装包，可以直接执行如下命令
rpm -Uvh *.rpm --nodeps --force
```

### 6. 启动MySQL服务

```bash
#启动
systemctl start mysqld
#重启
systemctl restart mysqld
#停服务
systemctl stop mysqld
#查询mysql状态
systemctl status mysqld

```

### 7. 查询自动生成的root用户密码

使用rpm安装时候，会自动给root生成密码，我们可以通过查询mysql启动日志，查看自动生成的密码

```bash
cat  /var/log/mysqld.log  | grep -C 2 'temporary password'
```

命令行执行指令 :

```bash
mysql -u root -p
```

然后输入上述查询到的自动生成的密码, 完成登录 

### 8. 修改root用户密码

登录到MySQL之后，需要将自动生成的不便记忆的密码修改了，修改成自己熟悉的便于记忆的密码。

```bash
ALTER  USER  'root'@'localhost'  IDENTIFIED BY '1234';
```

执行上述的SQL会报错，原因是因为设置的密码太简单，密码复杂度不够。我们可以设置密码的复杂度为简单类型，密码长度为4。

```bash
set global validate_password.policy = 0;
set global validate_password.length = 4;
```

降低密码的校验规则之后，再次执行上述修改密码的指令。

### 9. 创建用户

默认的root用户只能当前节点localhost访问，是无法远程访问的，我们还需要创建一个root账户，用户远程访问

```bash
create user 'root'@'%' IDENTIFIED WITH mysql_native_password BY '1234';
```

### 10. 并给root用户分配权限

```bash
grant all on *.* to 'root'@'%';
```

### 11. 重新连接MySQL

```bash
mysql -u root -p
```

然后输入密码









