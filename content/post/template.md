---
author: "Suika"
title: "封装虚拟机"
date: "2020-05-12 11:19:16"
description: ""
categories: "Tech"
tags: 
  - "Virtualzation"
image: ""
---

很多时候我们需要重复创建虚机，模板就是方便此类重复操作的一个虚机副本。模板不能拥有任何特定的信息，否则依此建立的虚机将都带有特定信息。所以，在建立模板前需要对源虚机进行封装，所谓封装就是在保证其他功能正常的同时，去除掉特点的功能。  
以前linux通用的封装方式比较烦，需要手动删除如下：  
1. ssh主机密钥：
 ```bash
rm -rf /etc/ssh/ssh_host_*
  ```
2. 设置hostname为localhost
  ```bash
  cat /etc/sysconfig/network
  HOSTNAME=localhost.localdomain
  ```
3. 从/var/log中删除所有日志：
```bash
for i in `find .-name "*.log"`;do cat /dev/null>$i;done
```
4. /root中删除build日志：  
5. 关闭虚机

**现在rhel提供了更简便的封装方法：**
```bash
yum install -y libguestfs-tools
virt-sysprep
```

**windows的封装一向很简单：**  
运行sysprep.exe  
一般在C:\Windows\System32\sysprep\sysprep.exe  

此类是通用的封装方法，有虚拟化平台提供自己的封装工具，那么以平台为准。