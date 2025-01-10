---
layout: post
title:  "gitlab-runner shell executer指定默认shell为bash"
date:   2025-01-10 15:22:10 +0525
categories: gitlab-runner
---

背景：gitlab-runner注册为shell executer时执行shell，默认使用的是sh，但shell执行需要安装nodejs，而nodejs 是通过nvm实现版本控制，初始化过程就是在bash下面执行的，需要使用bash，比如使用source命令，所以需要将默认shell指定为bash。


按照以下步骤设置即可，先查看gitlab-runner systemd配置文件，通过启动命令查看配置路径
![alt text](/assets/gitlab-runner-systemd.png)

配置如下
```
root@iZeb3e3lhj5yxnnt3r5o4iZ:~# cat /etc/gitlab-runner/config.toml 
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "test"
  url = "https://************"
  id = 45
  token = "***********"
  token_obtained_at = 2025-01-06T07:21:13Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "shell"
  shell = "bash"
  [runners.custom_build_dir]
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
```
添加shell = "bash"即可 ，之后systemctl daemon-reload && systemctl restart gitlab-runner 重新加载。如果不添加，默认是sh，执行npm 时会提示npm not found等其他一些问题，为了环境初始化彻底，还需要显示指定 source ~/.bashrc , 为什么？
因为gitlab-runner 默认直接执行bash 是非交互不登录shell 不会执行~/.bashrc ，这样会导致环境变量丢失，同样会提示npm not found
例如：gitlab-ci.yml内容如下

```
stages:
  - build&deploy

build:
  stage: build&deploy
  tags:
    - dubai
  script:
    - source ~/.bashrc
    - cp ~/deploy.sh ./
    - bash deploy.sh
```

参考：[Types of shells supported by GitLab Runner](https://docs.gitlab.com/runner/shells/)

