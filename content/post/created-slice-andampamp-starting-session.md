---
author: "Suika"
title: "Created slice &amp;amp; Starting Session"
date: "2020-02-26 15:40:16"
description: "Logs should only record what is useful"
categories: 
  - "Tech"
tags: 
  - "log"
  - "RHEL"
image: ""
---

## 背景：
RHEL7的日志（/var/log/message）里总有如下log，甚至很长一段时间内整个log都是这些内容：
```
example.com systemd: Created slice user-0.slice.
example.com systemd: Starting Session 150 of user root.
example.com systemd: Started Session 150 of user root.
example.com systemd: Created slice user-0.slice.
example.com systemd: Starting Session 151 of user root.
example.com systemd: Started Session 151 of user root.
```
## 过滤：
照红帽的官方说法，这是居然是正常情况，只要用户登录就会记录这些日志，如果不想记录这些内容需要主动将其过滤掉，方法如下：
```
echo 'if $programname == "systemd" and ($msg contains "Starting Session" or $msg contains "Started Session" or $msg contains "Created slice" or $msg contains "Starting user-" or $msg contains "Starting User Slice of" or $msg contains "Removed session" or $msg contains "Removed slice User Slice of" or $msg contains "Stopping User Slice of") then stop' >/etc/rsyslog.d/ignore-systemd-session-slice.conf
```
然后，重启rsyslog服务：
```
systemctl restart rsyslog
```
## 其他：
以上方法只对本地日志起作用，远程日志系统不适用。
如果远程日志要过滤此类日志，需要把上述规则写进/etc/rsyslog.conf，此时要写在发送到日志服务器之前，即. @@sys-log server这一行前。