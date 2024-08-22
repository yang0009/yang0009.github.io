---
layout: post
title:  "关于通过openvpn client-config-dir打通客户端内网的ccd设置"
date:   2024-08-14 16:02:10 +0430
categories: openvpn
---
背景:
最近在使用openvpn 客户端,碍于无法直接在客户端之间访问其内部设备，进行了一些尝试,记录一下

下面是openvpn server 端的配置
```
[root@cicd ~]# cat /etc/openvpn/server.conf
local <ip地址>
remote <ip地址>
port 1194
proto tcp
dev tun

ca /etc/openvpn/certs/ca.crt
cert /etc/openvpn/certs/server.crt
key /etc/openvpn/certs/server.key
dh /etc/openvpn/certs/dh.pem
server 10.8.0.0 255.255.255.0

# 固定ip地址的数据文件，确保客户端不会每次重新登陆获取的ip不变
ifconfig-pool-persist /etc/openvpn/ipp.txt

# 核心配置
client-config-dir /etc/openvpn/ccd

# client-config-dir 字段需要配合以下route 进行
route 10.10.100.0 255.255.255.0   # 服务器静态路由 告诉OpenVPN服务器，要访问10.10.100.0/24网段时，流量应该由ccd目录中记录的客户端发送
route 10.10.88.0 255.255.255.0   # 服务器静态路由 告诉OpenVPN服务器，要访问10.10.88.0/24网段时，流量应该由ccd目录中记录的客户端发送
route 172.29.2.0 255.255.255.0 # 服务器静态路由 告诉OpenVPN服务器，要访问172.29.2.0/24网段时，流量应该由ccd目录中记录的客户端发送
route 172.29.4.0 255.255.255.0 # 服务器静态路由 告诉OpenVPN服务器，要访问172.29.4.0/24网段时，流量应该由ccd目录中记录的客户端发送
route 172.29.1.0 255.255.255.0 # 服务器静态路由 告诉OpenVPN服务器，要访问172.29.1.0/24网段时，流量应该由ccd目录中记录的客户端发送
route 172.28.10.0 255.255.255.0 # 服务器静态路由 告诉OpenVPN服务器，要访问172.28.10.0/24网段时，流量应该由ccd目录中记录的客户端发送

# server 端需要push 给客户端的路由
push "route 172.24.0.0 255.255.0.0" 

#
client-to-client

#route 10.150.15.0 255.255.255.0
#crl-verify /etc/openvpn/easy-rsa/3.0.3/pki/crl.pem
crl-verify /etc/openvpn/easy-rsa/3.0.3/pki/crl.pem

keepalive 20 120
#tls-auth ta.key 0
#client-config-dir ccd
#cipher AES-256-CBC

user openvpn
group openvpn


comp-lzo
persist-key
persist-tun
status openvpn-status.log
verb 1
mute 20
log-append /var/log/openvpn.log
management 0.0.0.0 5555
```
上面的配置中，client-config-dir 字段为主要的配置，用于指定客户端配置文件的目录。在这个目录中，每个客户端的配置文件都对应一个客户端的配置。

在客户端配置文件中，你可以指定客户端的IP地址、路由信息。例如，你可以创建一个名为 "aliyun" 的文件，内容如下：
```
[root@cicd ccd]# cat /etc/openvpn/ccd/aliyun 
iroute 10.10.100.0 255.255.255.0
iroute 10.10.88.0 255.255.255.0

```

这样的话，在任何客户端连接到 OpenVPN 服务器时，OpenVPN 服务器会读取 ccd 目录中的配置文件，并根据这些配置文件中的信息来配置客户端的网络设置进行路由,假如我在客户端中访问10.10.100.0/24网段时，流量应该由ccd目录中记录的客户端发送
这样就实现了客户端之间通过openvpn 打通内网
于此同时，内网的客户端也可以不通过openvpn打通其他客户端的内网，只需要在路由器配置路由表即可
例如：
![alt text](route.png)
这样就实现了免登录openvpn 打通内网，一般在办公司路由器配置可以实现整个公司实现网络打通

