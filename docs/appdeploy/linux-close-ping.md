---
title: Linux关闭/开启ICMP协议ping的方法
tags: Linux
abbrlink: 2fe3bcdf
date: 2020-04-30 13:22:07
---

### Linux服务器关闭/开启ICMP协议(ping)的方法

一、内核参数设置
1、允许ping设置\

临时

```shell
echo 0 >/proc/sys/net/ipv4/icmp_echo_ignore_all
```

永久

```shell
echo net.ipv4.icmp_echo_ignore_all=0 >> /etc/sysctl.conf
sysctl -p
```

> \# 执行这条命令使更改后的 /etc/sysctl.conf 配置文件生效
> 注意：如果 /etc/sysctl.conf 配置文件里已经有 net.ipv4.icmp_echo_ignore_all 字段了，那么直接用 vim 进去更改对应的值即可。

2、禁止ping设置

临时

```shell
echo 1 >/proc/sys/net/ipv4/icmp_echo_ignore_all
```


永久

```shell
echo net.ipv4.icmp_echo_ignore_all=1 >> /etc/sysctl.conf
sysctl -p
```

> \# 执行这条命令使更改后的 /etc/sysctl.conf 配置文件生效
> 注意：如果 /etc/sysctl.conf 配置文件里已经有 net.ipv4.icmp_echo_ignore_all 字段了，那么直接用 vim 进去更改对应的值即可。

二、防火墙设置

> 注：使用以下方法的前提是内核配置是默认值，也就是内核没有禁ping

1、允许PING设置

```shell
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
```

2、禁止PING设置

```shell
iptables -A INPUT -p icmp --icmp-type 8 -s 0/0 -j DROP

--icmp-type 8 echo request 表示回显请求（ping请求）

0/0 表示所有 IP
```

