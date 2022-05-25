---
author: "Suika"
title: "虚拟机镜像转换"
date: "2020-05-08 11:00:23"
description: ""
categories: "Tech"
tags: 
  - "Virtualzation"
image: ""
---

总会遇到诡异的问题，比如要部署厂家打包好的定制化的镜像，发现这镜像只提供一种平台的格式，这时候就需要将虚拟磁盘做转换了。  
各个厂家提供一些定制化的工具并不是太通用，比如VMWare的vmware-vdiskmanager，好在这个世界有开源的qemu-img工具，可以方便的转换各种格式的镜像。

## 安装：
qemu-img提供多平台支持，常见的windows可以从[官网](https://qemu.weilnetz.de)下载安装；linux就更简单了，配源然后install，比如：`yum install qemu-img`
## 参数：
常用的转换参数如下：

convert：转换操作；  
-p：显示转换进度;  
-f xxx：指明被转换镜像的格式，可以留空，但hyperV的vhd最好写明vpc否则可能不识别；  
-O xxx：表明要转换成的格式；  
info：查询信息
## 实例：
vmdk转qcow2：
```bash
qemu-img convert -p -f vmdk -O qcow2 RHEL.vmdk RHEL.qcow2
```
查询镜像详细信息：
```bash
qemu-img info RHEL.qcow2
```
如果转vhd文件，注意`-f`后跟`vpc`而不是`vhd`

以上没有了……