---
author: "Suika"
title: "ansible tower升级"
date: "2020-07-27 17:39:01"
description: "升级是一种常态"
categories: 
  - "Tech"
tags: 
  - "Ansible"
image: ""
---
Ansible Tower的版本更新的也是莫名的快，去年安装的3.5今年已经到3.7.1了，新的版本可以在线申请license，旧的license快过期了，所以就升个级吧。
## 步骤：
升级的时候不能只看升级文档，还要看新版本的support，一开始只看了升级文档，做的时候报操作系统版本不被支持，所以第一步配个较新的rhel源，然后update系统。
```bash
# yum update -y
# systemctl reboot 
```
下载要升级的版本，并解压：
```bash
# wget https://releases.ansible.com/ansible-tower/setup/ansible-tower-setup-3.7.1-1.tar.gz
# tar xvzf ansible-tower-setup-3.7.1-1.tar.gz
```
把曾经3.5时候安装的inventory拷贝一份到3.7,注意要把老版本的rabbitmq_host有关的变量更改为routable_hostname，因为新版本移除了rabbitmq：
```bash
# cp ansible-tower-bundle-3.5.1-1.el7/inventory ansible-tower-setup-3.7.1-1/
# cat inventory
[tower]
ansible01-ap
ansible02-ap
[database]
[all:vars]
admin_password='redhat'

pg_host='10.xx.xx.xx'
pg_port='5432'

pg_database='towerdata'
pg_username='postgres'
pg_password='redhat'

routable_username='tower'
routable_password='redhat'
routable_cookie=cookiemonster
```
因为涉及到clust所以用到了rhscl里的两个包，可以单独下下来建一个源，也可以直接陪rhscl的源，我enable了base和rhcsl两个源。
然后，在各节点上关闭服务：
```
ansible tower -m command -a "ansible-tower-service stop" -i inventory
```
拆除主备实例关系：
```
# awx-manage list_instances
[tower capacity=154]
ansible01-ap capacity=72 version=3.5.1
ansible02-ap capacity=72 version=3.5.1
# awx-manage deprovision_instance --hostname=ansible02-ap
# awx-manage list_instances
[tower capacity=0]
ansible01-ap capacity=0 version=3.5.1
```
最后就可以运行setup了：
```
# ./setup -i inventory
```
本来以为会很顺利，可是还是有点坑，不知道为什么ansible-tower的源通过proxy之后访问特别慢，而且还是总获取到ipv6的地址。所以找了一台可以连外网的机器，做了一个本地源：
```bash
# yum repolist
# yumdownloader --resolve --destdir /root/ansible ansible-tower
# createrepo -v -g /root/ansible-tower
```
然而，这只是把tower相关的包下载下来做了一个本地镜像，实际运行setup的时候还是去找tower的源。
尝试用下面的命令同步整个源到本地，发现速度太慢了
```
# reposync -l -p -m -g --download-metadata --repoid=ansible-tower --download_path=/root/ansible-tower
```
于是，翻了下playbook怎么写的，改了一下设置源的地方。有两个地方可以改，改一个就好了:
1、把源改成刚刚同步的最小的那个源的名：
```bash
cat role/packages_el/defaults/main
ansible_tower_repo: xxx
ansible_tower_dependency_repo: xxx
```
2、把install_tower.yml里检查repo的部分去掉，要改两部分一个是ansible_tower_repo，一个是ansible_tower_dependency_repo，把相关这两个tasks注释掉就好了

最后，等待setup完成就得到一个新版本的ansible tower
检查一下服务状态及实例情况，一切正常：
```bash
# ansible-tower-service status
# awx-manage list_instances
```
登录web，原来的project和inventory都在

## 参考文档：
https://docs.ansible.com/ansible-tower/3.7.1/html/installandreference/upgrade_tower.html#ir-upgrade-existing