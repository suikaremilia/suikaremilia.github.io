---
author: "Suika"
title: "Too many open files导致登录失败"
date: "2021-06-24 15:42:08"
description: "有人的地方就有人挖坑"
categories: 
  - "Tech"
tags: 
  - "SSH"
image: ""
---

发生个诡异的故障，用户ssh登录可以成功，但是马上session就被关闭了。
```
ssh root@xxxx
Password:
Last Login: Thu Jun 24 14:26:00 2021 from xxxxx
Connection to xxxx closed.
```
就是ssh 加上 -vvv看到的也是这种现象，既明显登录成功了，但立即被关闭了。

好在有其他人还有个session连在上面，看了日志有类似的报错：
```
Accepted keyboard-interactive/pam for usr_ins from xxx port xxx ssh2
pam_limits(sshd:session): Could not set limit for 'nofile': Operation not permitted
error: PAM: pam_open_session(): Permission denied
```

几经排查，发现是fs.nr_open的锅。有人改了/etc/security/limits.conf中的hard nofile，改的这个值比fs.nr_open大了，改小hard nofile或改大fs.nr_open就解决了。
```
sysctl -w fs.nr_open = xxxx
```
好在有个高权限的session还活着，不然要重启挂光盘修改了。

**总结：**
只要在/var/log/secure中看到如下类似报错：
```
login: pam_unix(remote:session): session opened for user root by (uid=0)
login: Permission denied
login: pam_limits(remote:session): Could not set limit for 'nofile': Operation not permitted

su: pam_limits(su:session): Could not set limit for 'nofile': Operation not permitted
su: pam_unix(su:session): session opened for user root by test(uid=501)

pam: gdm-password: pam_limits(gdm-password:session): Could not set limit for 'nofile': Operation not 
permitted
pam: gdm-password: pam_unix(gdm-password:session): session opened for user root by (uid=0)
```
那么就是此类问题引起的。

另外，fs.nr_open的默认值是1024*1024 = 1048576，最大值是由sysctl_nr_open_max在内核控制的，在X86服务器上是2147483584

这又增强了对linux一切皆文件的理解，系统文件不能打开了session当然无法创建。