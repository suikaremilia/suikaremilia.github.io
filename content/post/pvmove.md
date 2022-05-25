---
author: "Suika"
title: "pvmove迁移数据"
date: "2020-04-12 15:29:48"
description: "别怕，这是个安全操作"
categories: "Tech"
tags: 
  - "pvmove"
  - "LVM"
image: ""
---

使用LVM的好处之一就是可以用pvmove来方便的在线迁移数据。用pvmove来迁移、合并数据比创建mirorr的方式要安全而方便，它会把要迁移的数据拆分成多分，并通过建立临时镜像的方式来移动数据，即使中断也有全部的数据，还能很方便的再次开始。

## 简单的用法：

```
pvmove /dev/sdc1
```
将/dev/sdc1上的数据迁移到vg中的其他物理卷

```
pvmove -n MYLV /dev/sdc1
```
将MYLV迁移到物理卷/dev/sdc1

```
pvmove -b /dev/sdc1 /dev/sdd1
```
pvmove的最常用用法，在后台迁移/dev/sdc1的数据到/dev/sdd1

```
pvmove -i5 /dev/sdc1
```
查询pvmove的进度，每5s输出一次结果。可以随时执行以查询迁移进度，后面不指定迁移的磁盘则会显示所有pvmove的进度
```
lvs -a -o +devices
```
另一种查询迁移进度的方式

## 迁移完成后：

```
vgreduce /dev/sda
pvremove /dev/sda
```
迁移完成后即可将不用的物理卷从vg中剔除，把它的PV属性去掉

## 注意：
虽然可以在线迁移，但仍会增大CPU、MEMORY、IO的开销，所以尽量在不忙的时候做操作。

如果要迁移clvmd管理的集群LVM，则需要确保cmirrord服务安装并开启。