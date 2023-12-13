---
layout: post
title:  "gitlab-runner cp命令使用需要注意的几个问题"
date:   2023-12-12 15:26:23 +0530
categories: gitlab-runner
---
背景:
gitlab-runner cp命令在gitlab-runner 使用中尽量不要直接cp 全部文件到部署目录下, 如果当前仓库文件是官网首页,如果是nginx部署代理静态页面,那么 root指令设置的是部署目录,那么当前的目录有可能会直接访问到.git/config 文件


原先gitlab-ci.yml目录部署代码
```
...
stage: deploy
tags:
	- deploy
script:
	cp -rf . /dst_path/
...
```


nginx.conf关键配置
```
listen 80;
server_name domain.example;
...
root /dst_path/;
index index.html;
...
```

改造后gitlab-ci.yml目录部署代码
```
...
stage: deploy
tags:
	- deploy
script:
	rsync -av --exclude=".git" --exclude=".gitlab-ci.yml" --delete ./ /dst_path/
...
```

# 问题严重性: 严重
总结:如果服务是对外公网可访问,那么通过http://domain.example/.git/config 可以直接拿到git仓库信息,如果是内网服务,那么通过http://domain.example/git/config 就无法访问到git仓库信息,所以需要在gitlab-runner rsync命令中增加 --exclude=".git" 等参数,这样就可以避免直接访问到.git/config 文件