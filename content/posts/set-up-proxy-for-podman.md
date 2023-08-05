---
author: "Suika"
title: "Set up proxy for podman"
date: "2020-07-20 14:15:12"
description: ""
categories: 
  - "Tech"
tags: 
  - "Proxy"
  - "Podman"
image: ""
---

背景如题，方案在/etc/profile.d/下配置：
```
# cat /etc/profile.d/http_proxy.sh
export HTTP_PROXY=http://192.168.0.1:8080
export HTTPS_PROXY=http://192.168.0.1:8080
```
或直接export，但前者对ansible也生效，后者只能是该终端的用户

貌似都是废话，其他程序配置Proxy，用常规的用export命令配置环境变量不生效的时候，也可以试试这样……