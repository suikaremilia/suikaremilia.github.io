---
author: "Suika"
title: "Arch Linux instal"
date: "2020-03-08 17:48:42"
description: "Arch is great"
categories: "Tech"
tags: 
  - "Arch"
image: ""
---

主力的本子用ARCH工作已经几年了，一直没有问题，有问题也都是能解决的问题。所以是出于莫名其妙的原因吧，在这个假期把arch重装了一下。跟几年前有些许变化，趁还有日志，记录一下过程。

### 一、环境准备：
设置bios为UEFI启动，各主机情况不同，一般是开机按F2进主板进行设置，设置的同时在boot里将从其他设备启动的选项打开；  
现在没有本子不是UEFI启动了吧，如果有就去看官方文档吧，这篇不适合做参考。  
下载最新的arch linux的[ISO镜像](https://www.archlinux.org/download/)，然后制作启动U盘。  
我是在现有的arch上用dd做的：  
```
dd if=archlinux.iso of=/dev/sdb bs=1M
```
然后，用U盘引导启动，进入live CD的系统后再确认一下启动方式是什么：
```
# ls  /sys/firmware/efi/efivars
```
如果此目录下有文件则为UEFI，否则请重新检查bios设置。  
国内的主机默认美国键盘就好，不需要做任何更改。  
### 二、连接到网络：
有线默认是dhcp的，插上网线自动获取地址即可，无线稍微复杂一点，可选的工具也很多，我习惯用netctl这工具。  
检查网卡名称：  
```
ip link
```
拷贝并修改配置文件：  
```
# cp /etc/netctl/examples/wireless-wpa /etc/netctl/ihangzhou
# vim /etc/netctl/ihangzhou
```

修改了以下标记了的内容：  
```bash
Description='A simple WPA encrypted wireless connection'
Interface=~~wlp2s0~~
Connection=wireless
Security=wpa
IP=dhcp
ESSID='~~i-hangzhou~~'
Key='~~xxxxxx~~'   
wifi的密码
# Uncomment this if your ssid is hidden
Hidden=*~~yes~~*         
是否是隐藏的网络，如不是则在此行前面加#将其注释掉，默认注就是注释的
# Set a priority for automatic profile selection
Priority=10
```

启用配置：
```
cd /etc/netctl/
netctl start ihangzhou
```
如果没有报错应该就会获取到地址，有报错就检查一下哪里错了。  
确认一下地址，并看下联通性，这时候百度有了它唯一的用处：
```
# ip a
# ping www.baidu.com
```
### 三、磁盘分区：
磁盘分区和文件系统都算常识吧，有兴趣的去看wiki或google，此次只是记录下简单的使用GPT分区和brtfs的过程。  
```bash
# parted /dev/sdx
(parted) mklabel gpt
(parted) mkpart ESP fat32 1M 150M 
(parted) set 1 boot on
(parted) mkpart primary brtfs 151M 15G 
(parted) mkpart primary brtfs 24.5G 100%
```
**将分区格式化：**  
```
# lsblk /dev/sdx
# mkfs.vfat -F32 /dev/sdxY
# mkfs.brtfs /dev/sdxZ
```
**创立挂载点并挂载分区：**  
```
# mount /dev/sdxZ /mnt
根分区要先挂载，然后再创建其他挂载点
# mkdir /mnt/home
# mount /dev/sdxW /mnt/home
# mkdir -p /mnt/boot
# mount /dev/sdXY /mnt/boot
```
我没有创建swap分区，我觉得内存足够大够用了。

### 四、安装系统：
**选用合适的安装镜像地址**  
要根据网络情况来调整，列表中排的越靠前的镜像站优先级越高，同时理论地理位置越近的镜像站速度越快，国内墙是主要考虑因素，按实际情况调整即可，以获得尽量大的下载速度为主要目标。  
更改镜像列表后请务必强制刷新。  
```
# vim /etc/pacman.d/mirrorlist
# pacman -Syy
```
**安装基本软件包**  
```
# pacstrap /mnt base linux linux-firmware
```
此处也可以同时安装其他的软件包，把包名加在后面即可

**生成 fstab：**  
最好用-U参数，使用设备UUID而不是别名进行挂载  
```
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

### 五、配置环境：
进入安装好的系统：  
```
# arch-chroot /mnt
```
**调整时间：**  
开启NTP同步：
```
# timedatectl set-ntp true
```
选择时区：  
```
# tzselect
# ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
同步硬件时间：  
```
# hwclock --systohc
```
**本地化设置：**  
locale.gen 与 locale.conf两个文件，规定地域、货币、时区日期的格式、字符排列方式和其他本地化标准等等，非常重要。  

`/etc/locale.gen`是一个仅包含注释文档的文本文件。指定需要的本地化类型，只需移除对应行前面的注释符号（＃）即可，建议选择带UTF-8的选项，之后再生成locale信息：
```bash
# vim /etc/locale.gen
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
zh_TW.UTF-8 UTF-8
ja_JP.UTF-8 UTF-8

# locale-gen
```

创建 locale.conf 并提本地化选项：
```
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```
**创建初始ramdisk环境:**  
打开需要的钩子，比如lvm什么的，然后生成Initramfs
```
# vim /etc/mkinitcpio.conf 
# mkinitcpio -p linux
```
**设置root密码：**  
```
# passwd
```
**安装启动器：**  
再次确认一下分区挂载情况，尤其是boot分区，不正确还可以调整：  
```
# lsblk
```
启动器有很多可选，grub2太传统，这次选择了syslinux，如果要好看可以选rEFInd。  
~~我才不会说是搞了半天搞不定rEFInd才换syslinux的，逃~~  
```
# pacman -S syslinux
# syslinux-install_update -i -a -m
```

**安装其他包：**  
```
# pacman -S networkmanager wpa_supplicant dialog bash-completion dhcpd
```

**重启进入系统：**  
```
# exit
# reboot
```

**其他设置：**  
联网：  
```
# nmtui
```
设置主机名：  
```
# hostnamectl set-hostname XXX
```
添加管理员帐号：  
```
# useradd -m -g users -G wheel -s /bin/bash xxx
# passwd xxx
```
**更新一下系统：**  
```
# pacman -Syyu
```
然后就完成了，后面图形什么的是另外的事。  
我用了xfce4做图形桌面，fcitx做输入法，挺好用的。  

### 六、其他工具：
如今arch的安装其实很简单，手操一遍有助于对linux工作方式和启动的理解。如果不想手动的操作，github上有专门的辅助部署的项目，比较完善的有[helmuthdu/aui](https://github.com/helmuthdu/aui)连后面的图形和软件一起都搞定了，只有你能进入live CD环境并联网。

> [https://wiki.archlinux.org/index.php/Installation_guide](https://wiki.archlinux.org/index.php/Installation_guide)  
> [https://wiki.archlinux.org/index.php/General_recommendations](https://wiki.archlinux.org/index.php/General_recommendations)  
> [https://wiki.archlinux.org/index.php/List_of_applications](https://wiki.archlinux.org/index.php/List_of_applications)  
> [https://telegra.ph/%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%85ARCH-LINUX-07-18](https://telegra.ph/%E5%A6%82%E4%BD%95%E5%AE%89%E8%A3%85ARCH-LINUX-07-18)
