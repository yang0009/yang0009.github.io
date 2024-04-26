---
layout: post
title:  "openvpn client后台守护进程过程需要输入密码的问题处理"
date:   2024-04-26 16:29:23 +0529
categories: openvpn
---
## 背景:
最近在使用openvpn 客户端,发现后台运行的openvpn 客户端会出现输入密码的问题, 导致无法正常启动.后面查找文档,可以指定秘钥文件来运行,但是命令也有区别，其中启动命令取决于创建客户端时使用的是方式

## 情况1:
例如   ./easyrsa gen-req user1 nopass 命令生成的是没有密码的共享秘钥, 那么使用这个秘钥启动openvpn 客户端,就不会出现输入密码的情况.所以启动后台是命令是 
```
openvpn --config client.ovpn --daemon
```


## 情况2:
例如   ./easyrsa gen-req user1  命令会提示输入PEM pass 密码, 那么使用这个秘钥启动openvpn 客户端,就会出现输入密码的情况.所以启动后台是命令是 
```
openvpn --config client.ovpn  --askpass pass.txt --daemon #pass.txt 里面第一行输入密码
```

## 情况3:
例如 创建用户时使用的是账号密码的方式那么使用这个秘钥启动openvpn 客户端,就会出现输入密码的情况.所以启动后台是命令是
```
openvpn --config client.ovpn  --auth-user-pass pass.txt --daemon #pass.txt 里面第一行输入用户,第二行输入密码
```
或者在配置文件中写上 auth-user-pass字段并且指定密码文件
```
[root@iZwz99r04mk0fbt6slyc27Z client]# cat client.ovpn 
client
dev tun
proto tcp 
remote *** 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
remote-cert-tls server
cipher AES-256-CBC
verb 3
comp-lzo
auth-user-pass pass.txt
```

如果你想在 OpenVPN 中使用账号密码认证，而不是基于证书的方式，可以配置 OpenVPN 服务器以使用一个用户名和密码列表来认证客户端。这通常涉及以下几个步骤：

1 配置 OpenVPN 服务器：在 OpenVPN 服务器的配置文件中，你需要启用用户名和密码认证。这可以通过在配置文件中添加如下指令实现：
```
auth-user-pass-verify /etc/openvpn/auth_script.sh via-env
script-security 3
client-cert-not-required
username-as-common-name
```
这里的 /etc/openvpn/auth_script.sh 是一个脚本，你需要创建它来验证用户名和密码

2 创建认证脚本：创建一个脚本（例如 /etc/openvpn/auth_script.sh），该脚本将会被 OpenVPN 调用来验证客户端提供的用户名和密码。这个脚本可以使用任何你喜欢的方法来验证凭据，比如检查一个静态文件、查询数据库或者与 LDAP 服务器通信。
这是一个简单的认证脚本示例，只是检查用户名和密码是否匹配预定义的值：
```
#!/bin/sh
# auth_script.sh
USERNAME="user1"
PASSWORD="pass1"

if [ "$username" == "$USERNAME" ] && [ "$password" == "$PASSWORD" ]; then
    exit 0
fi
exit 1

```
3 配置客户端：在客户端配置文件中，添加以下指令以提示输入用户名和密码：
```
auth-user-pass

```
这会让 OpenVPN 客户端在建立连接时提示用户输入用户名和密码。