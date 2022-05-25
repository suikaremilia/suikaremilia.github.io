---
author: "Suika"
title: "Commvault修改介质信息"
date: "2020-04-27 16:53:56"
description: "这年头还有用磁带的"
categories: "Tech"
tags: 
  - "Backup"
  - "tape"
image: ""
---
做磁带转储的时候发生了一件很奇怪的事情，一盘LTO3的磁带被认成了LTO2，在LTO5的驱动器里读不出来。  
当然读不出来很正常，毕竟只向下兼容两代读，可认成LTO2就很诡异了。  
于是找了一下方法把它修改过来：  

1、CV管理员登录：  
cd到安装目录然后qlogin，按提示输入用户名和密码；

2、对要操作的介质执行：  
QOperation.exe execscript -sn changeMediaType -si media -si "000231" -si "ULTRIUM V3"
