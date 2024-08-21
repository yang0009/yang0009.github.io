---
layout: post
title:  "如何在容器内部使用宿主机代理端口"
date:   2024-08-21 18:12:10 +0630
categories: docker v2ray
---
背景:
需求：由于墙的关系，一些特殊apt包的源需要外网，服务是容器起的，所以构建docker镜像软件包需要翻墙，因此在构建服务器上起v2ray 代理端口监听0.0.0.0接口：端口

下面是v2ray 客户端的配置(服务端需要自己搭建这里不再多写)
```
[root@cicd ~]# cat /v2ray/config.json
{
  "log": {
    "access": "",
    "error": "",
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "socks",
      "port": 10808,
      "listen": "0.0.0.0",
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "settings": {
        "auth": "noauth",
        "udp": true,
        "allowTransparent": false
      }
    },
    {
      "tag": "http",
      "port": 10809,
      "listen": "0.0.0.0",
      "protocol": "http",
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      },
      "settings": {
        "udp": false,
        "allowTransparent": false
      }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "********",
            "port": ******,
            "users": [
              {
                "id": "********",
                "alterId": 0,
                "email": "t@t.tt",
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "allowInsecure": false,
          "serverName": "*****"
        },
        "wsSettings": {
          "path": "/stream",
          "headers": {
            "Host": "******"
          }
        }
      },
      "mux": {
        "enabled": false,
        "concurrency": -1
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "block",
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      }
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "domainMatcher": "linear",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
          "api"
        ],
        "outboundTag": "api",
        "enabled": true
      },
      {
        "type": "field",
        "outboundTag": "direct",
        "domain": [
          "domain:example-example.com",
          "domain:example-example2.com"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "outboundTag": "block",
        "domain": [
          "geosite:category-ads-all"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "outboundTag": "direct",
        "domain": [
          "geosite:cn"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "outboundTag": "direct",
        "ip": [
          "geoip:private",
          "geoip:cn"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "port": "0-65535",
        "outboundTag": "proxy",
        "enabled": true
      }
    ]
  }
}
```
上面的配置中，inbounds 块为主要的配置， 其中socks 和http 是v2ray 的代理端口，proxy 是v2ray 的代理服务器地址，direct 是直连地址，block 是黑名单地址，下面为主要的配置
    "tag": "socks",
    "port": 10808,
    "listen": "0.0.0.0",
    "protocol": "socks",
    这些的意思是v2ray 的代理端口，监听0.0.0.0 接口，协议为socks，端口为10808

启动v2ray代理
```
[root@cicd v2ray]# v2ray -config /v2ray/config.json

V2Ray 4.45.0 (V2Fly, a community-driven edition of V2Ray.) Custom (go1.18.1 linux/amd64)
A unified platform for anti-censorship.
2024/08/21 18:04:48 [Info] main/jsonem: Reading config: /root/v2ray/config.json
2024/08/21 18:04:49 [Warning] V2Ray 4.45.0 started

```
执行一下命令查看端口是否监听成功
```
[root@cicd v2ray]# ss -tlnp |grep v2ray
LISTEN     0      128         :::10808                   :::*                   users:(("v2ray",pid=17664,fd=7))
LISTEN     0      128         :::10809                   :::*                   users:(("v2ray",pid=17664,fd=3))
```
看见以上说明启动成功

接下来需要在容器内部执行指定的宿主机ip:端口的环境变量，注意这里的ip是宿主机的,不是容器内部的127.0.0.1
```
[root@cicd v2ray]# docker run -it wine:ubuntu20.04-python3.11 bash
root@c6012ee486df:/# export https_proxy=http://172.24.6.131:10808
root@c6012ee486df:/# export http_proxy=http://172.24.6.131:10808
root@c6012ee486df:/# apt-get update
```
以上说明，容器内部已经可以访问外网了，但有些人会遇到以下问题
1. 如果出现connection refused 说明代理端口没有监听成功, 检查一下listen 的ip 和是不是0.0.0.0 如果是127.0.0.1 则不通
2. 如果出现 OpenSSL SSL_connect 说明代理地址不对，检查https_proxy=http://172.24.6.131:10808 而不是 https://172.24.6.131:10808


