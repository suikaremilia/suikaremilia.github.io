---
author: "Suika"
title: "Python 更改pip源"
date: "2020-03-25 10:08:05"
description: ""
categories: 
  - "Tech"
tags: 
  - "PIP"
image: ""
---

内网环境往往需要改pip为内网源地址，否则要配Proxy。
## 临时更改：  
```bash
pip install -i <server IP> --trusted-host <server IP> <package_name>
```
## 永久修改：
linux修改 ~/.pip/pip.conf中的index-url和trusted-host，windows修改C:\Users\xxx\pip\pip.ini (没有就创建一个)  
```bash
[global]
index-url = pypi.python.org
trusted-host = pypi.python.org
               pypi.org
               files.pythonhosted.org
```