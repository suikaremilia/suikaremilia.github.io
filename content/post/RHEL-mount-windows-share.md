---
author: "Suika"
title: "RHEL挂载windows共享"
date: "2020-07-13 13:50:07"
description: "Windows和Linux总是很难联系"
categories: "Tech"
tags: 
  - "SMB"
image: ""
---
莫名其妙的曾经能用的mount cifs的方式开始报错
```bash
mount error(112): Host is down
Refer to the mount.cifs(8) manual page (e.g. man mount.cifs)
```
查了一下原因是smb协议版本问题，RHEL5和6最高支持SMB 1.0协议，RHEL7.2以后支持到SMB3。
所以，RHEL5和6就不要想挂载最新的windows共享了，windows目前关闭了smb1，RHEL7如果想访问windows共享的话，则需要指定一下版本。
```bash
mount -t cifs -o vers=2.0,username=<win_share_user>,password=<win_share_password> //WIN_SHARE_IP/<share_name> /mnt/win_share
```
如果是域用户，则加domain信息：
```bash
mount -t cifs -o vers=2.0,username=<win_share_user>,domain=<win_domain>，password=<win_share_password> //WIN_SHARE_IP/<share_name> /mnt/win_share
```
或者更安全一点的做法，建个文件储存用户名和密码：
```bash
# cat /etc/win-credentials
username = user
password = password
domain = domain

设置权限：
chown root: /etc/win-credentials
chmod 600 /etc/win-credentials

挂载：
mount -t cifs -o credentials=/etc/win-credentials //WIN_SHARE_IP/<share_name> /mnt/win_share
```
也可以把信息写入/etc/fstab来实现自动挂载：
```bash
# cat /etc/fstab
# <file system>             <dir>          <type> <options>                                                   <dump>  <pass>
//WIN_SHARE_IP/share_name  /mnt/win_share  cifs  credentials=/etc/win-credentials,file_mode=0755,dir_mode=0755 0       0

mount /mnt/win_share