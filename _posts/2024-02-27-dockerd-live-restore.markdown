---
layout: post
title:  "dockerd 如何理解live restore设置"
date:   2024-02-27 15:26:23 +0530
categories: dockerd
---
背景:
A100机器是由docker 启动，，其中挂载的驱动会被dockerd 重启,导致挂载的驱动不能被正常运行,导致服务异常


根据官方文档设置live restore 为true,即可解决问题 加入一下配置到 /etc/docker/daemon.json
```
{
  "live-restore": true
}
```

reference: https://docs.docker.com/config/containers/live-restore/
