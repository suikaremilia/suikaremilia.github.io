---
author: "Suika"
title: "Opne Shift 4.6在线安装"
date: "2020-12-12 09:14:40"
description: "OCP是个好东西，除了贵"
categories: 
  - "Tech"
tags: 
  - "Openshift"
image: ""
---

## 规划
### 架构：

### 服务器信息：

| 用途 | 主机名 | IP |
| -------- | -------- | -------- |
| Master1| ocpmaster01.suika.com | 10.90.16.21|
| Work1| ocpwork01.suika.com | 10.90.16.22|
| Work2| ocpwork02.suika.com | 10.90.16.23|
| bootstrap| bootstrap.ocpsuika.com |10.90.18.19|
| bastion| bastion.suika.com |10.90.18.16|

### 域名信息：

| 用途           | 域名                | 地址 | 端口 |
| -------------- | ------------------- | ---- | ---- |
| Kubernetes API | api.ocp.suika.com | 10.90.18.19 | 6443 |
| Kubernetes API | api-int.suika.com | 10.90.18.19 |  443    |
| Routes| *.apps.suika.com | 10.90.18.19 | 443  |


| 域名 | IP | 端口 |用途|
| -------- | -------- | -------- |-------- |
| console-openshift-console.apps.ocp.suika.com| 10.90.16.21|443 |OCP控制台页面|
| oauth-openshift.apps.ocp.suika.com| 10.90.16.21|443 |OCP oauth登录跳转页面|
| prometheus-k8s-openshift-monitoring.apps.ocp.suika.com| 10.90.16.21|443 |Prometheus|
| kibana-openshift-logging.apps.ocp.suika.com| 10.90.16.21|443 |Kibana|
| api.ocp.suika.com| 10.90.16.21|6443 |API|
### 端口列表：

| 协议 | 端口 | 用途 |
| -------- | -------- | -------- |
| TCP| 2379-2380 | etcd server, peer, and metrics ports|
| TCP| 6443 | Kubernetes API|
| TCP| 10249-10259| The default ports that Kubernetes reserves|
| TCP| 10256 |openshift-sdn|
| TCP| 9000-9999 |Host level services, including the node exporter on ports 9100-9101 and the Cluster Version Operator on port 9099.|
| UDP| 4789、6081 |VXLAN and GENEVE|
| UDP| 9000-9999 |Host level services, including the node exporter on ports 9100-9101.|
| TCP/UDP| 30000-32767 |Kubernetes NodePort|

## 环境准备：
### 离线资源：  
pull secret [下载地址](https://cloud.redhat.com/openshift/install/metal/installer-provisioned)  
OpenShift CLI（OC）[下载地址](https://access.redhat.com/downloads/content/290/ver=4.6/rhel---8/4.6.3/x86_64/product-software)  
Openshift-install CLI [下载地址](https://access.redhat.com/downloads/content/290/ver=4.6/rhel---8/4.6.3/x86_64/product-software)  
CoreOS image [下载地址](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/4.6.1/)  

### 安装OpenShift CLI并创建目录：
```bash
# tar xvzf oc-4.6.3-linux.tar.gz
# tar xvzf openshift-install-linux-4.6.3.tar.gz
# mv oc kubectl openshift-install /usr/local/bin
# mkdir -p /opt/install
# touch /opt/install/install-config.yaml
```
### DNS配置：
```bash
[root@bastion ~]# yum install dnsmasq -y

[root@bastion ~]# cat /etc/dnsmasq.d/ocp.conf 
address=/bastion.suika.com/10.90.18.16
address=/api.ocp.suika.com/10.90.18.16
address=/.apps.ocp.suika.com/10.90.18.16
address=/api-int.ocp.suika.com/10.90.18.16
address=/ocpmaster01.ocp.suika.com/10.90.16.21
address=/ocpmaster02.ocp.suika.com/10.90.16.22
address=/ocpmaster03.ocp.suika.com/10.90.16.23
address=/bootstrap.ocp.suika.com/10.19.18.19
address=/registry.suika.com/10.90.18.9
address=/oauth-openshift.ocp.suika.com/10.90.18.16
ptr-record=21.16.90.10.in-addr.arpa,ocpmaster01.ocp.suika.com
ptr-record=22.16.90.10.in-addr.arpa,ocpmaster02.ocp.suika.com
ptr-record=23.16.90.10.in-addr.arpa,ocpmaster03.ocp.suika.com
ptr-record=19.18.90.10.in-addr.arpa,bootstrap.ocp.suika.com
srv-host=_etcd-server-ssl._tcp.ocp.suika.com,etcd-0.ocp.suika.com,2380,10
srv-host=_etcd-server-ssl._tcp.ocp.suika.com,etcd-1.ocp.suika.com,2380,10
srv-host=_etcd-server-ssl._tcp.ocp.suika.com,etcd-2.ocp.suika.com,2380,10
```

启动服务：  
```bash
[root@bastion ~]# systemctl start dnsmasq.service
[root@bastion ~]# systemctl status dnsmasq.service 
● dnsmasq.service - DNS caching server.
   Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2020-12-11 23:03:19 CST; 2 weeks 3 days ago
 Main PID: 46638 (dnsmasq)
   CGroup: /system.slice/dnsmasq.service
           └─46638 /usr/sbin/dnsmasq -k

```
### 配置haproxy：
```bash
[root@bastion ~]# yum install haproxy -y
[root@bastion ~]# vim /etc/haproxy/haproxy.cfg
global

    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /
    monitor-uri /healthz

frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server bootstrap 10.90.18.19:6443 check
    server master-0 10.90.16.21:6443 check
    server master-1 10.90.16.22:6443 check
    server master-2 10.90.16.23:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap 10.90.18.19:22623 check
    server master-0 10.90.16.21:22623 check
    server master-0 10.90.16.22:22623 check
    server master-0 10.90.16.23:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker-1 10.90.16.22:80 check
    server worker-2 10.90.16.23:80 check
    server worker-3 10.90.16.21:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker-1 10.90.16.22:443 check
    server worker-2 10.90.16.23:443 check
    server worker-3 10.90.16.21:443 check
	
```
查看服务状态：
```
[root@bastion ~]# systemctl start haproxy.service

[root@bastion ~]# systemctl status haproxy.service 
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2020-12-11 22:55:58 CST; 2 weeks 3 days ago
 Main PID: 46295 (haproxy-systemd)
   CGroup: /system.slice/haproxy.service
           ├─46295 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─46296 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─46297 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
```
### 配置http服务器：
```
[root@bastion ~]# yum install httpd -y
[root@bastion ~]# mkdir -p /var/www/html/install/
[root@bastion ~]# ln -s /opt/install/ /var/www/html/install
```
同样启动服务并检查状态：
```
[root@bastion ~]# systemctl start httpd.service；systemctl status httpd.service
```
## 安装过程：
### 生成密钥待用：  
```
 ssh-keygen -t rsa -b 2048 -N "" -f /root/.ssh/id_rsa ；cat /root/.ssh/id_rsa.pub 
```
### 编辑install文件：  
把下载的pull-secret.txt及上面的文件加到默认的install-config.yaml后面，并且配置proxy  
注意：noproxy中按“，”分隔
```yaml
[root@bastion ~]# cat /opt/install-config.yaml 
apiVersion: v1
baseDomain: suika.com 
proxy:
  httpProxy: http://suika:suika@proxy@suika.com:8080 
  noProxy: .suika.com,.suika.com,.ocp.suika.com,10.90.16.0/22
compute:
- hyperthreading: Enabled   
  name: worker
  replicas: 0 
controlPlane:
  hyperthreading: Enabled   
  name: master 
  replicas: 3
metadata:
  name: ocp
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OpenShiftSDN
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K2N1aXNvbmd0YW9oM2Njb20xdnY2YjdzZ3FodWxyYjBiYXhucThxbW51b2s6WUYyQ1I1UTNNV1NFU1dLVVNEME8zNUo4UkVRUjI3QVBURDVIUkVWRVpEUExRSUJFOE85U0dFWE81QlRQNVo2VQ==","email":"suika@suika.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K2N1aXNvbmd0YW9oM2Njb20xdnY2YjdzZ3FodWxyYjBiYXhucThxbW51b2s6WUYyQ1I1UTNNV1NFU1dLVVNEME8zNUo4UkVRUjI3QVBURDVIUkVWRVpEUExRSUJFOE85U0dFWE81QlRQNVo2VQ==","email":"suika@suika.com"},"registry.connect.redhat.com":{"auth":"NzYxMDE1Mnx1aGMtMVZWNmI3c0dxaFVMcmIwYkF4blE4cU1OVU9LOmV5SmhiR2NpT2lKU1V6VXhNaUo5LmV5SnpkV0lpT2lJMU1tWTROR05tWm1Nd01EQTBOelV5WVdVNU16WmxaVGhrTkROaFlUbGhNeUo5LlJiT1g3Ml9qY21fVTZLSFdVMS1VT19YTUk0aXVuSXZ3SVRWa1dsUDZ2QlU0Qzl0U2pickltM1FuY2Q4WENjeUVoQWY5cUoxVXRtWHlVWk5JODZ3dEo1eVE1STZnU3FlZ2RBMEZ2ek1tQk9qUzZ6eHAxVm83ZmNfVG1oYzBQSERhSWVudlhTZ3BscnNaUWpRckllVUNlUlNuaGRGdDFSdHU0djFaMDh1V1N4Y3k4VGI0dXIyNmRXZWhUSUF6Q0R2aFNkRzBhUVZaRExOYVd5QTE2YjIwSEdFYVNRNmlXUDNQZ05tUWFmTkdyUnNOTFZYd01Qd0ZQU01YMkdsVGhmMXludEpiTXl3UEx1a3pzaEozeFJ3bkFFV0JIdGd2dVlDNUN2YjB1SFlUMDJWVUV1Q21hbDBBQkppUlo4LVdiOV9RZEgxcXNKQzBGRGFUa2dhSjhzSWpleE9qTWtRWVZCeHI4OGpNNVcta0xlbmhLQmRxMTladHFiWTdwMGd0S0JjWjFQczlkSWJRRzJKSTVUZEZza2suika1VDeHdYM1o2QVlBeGJyR2ItSHVBRi1PT0pTdVBKU3RNMDJPblBHdGVRaUVSd0ZOMUZNZjEwRmxjQktlRTIyUGR1SU1sN3gyWTIxalJMR3VZdHBZa0FWVUZ1UEdMM3l3ZVFsWGk4RWYxVG9FV2kwU0dpWXFaNEt1ZWNLVjhUWFhWNHJCUTJSNHhZQmVwZ1F5d2xPNWdiU0xNQ0IwNzkxUlAtRXFyUlNadkJhbERjSlRfbzNxNlk5VWYtYnRuaVBjeTY0bkIxUHZmakxwazNqY0tBdWtBVHZDZjhrUmJKRVkwOFVWQXVpZENadFZiZDByWXpXNS1TUDN3VC10ZGhTb2h5OWNvZ2xTWkZ6aXhlYmJFdkNmalBJ","email":"suika@suika.com"},"registry.redhat.io":{"auth":"NzYxMDE1Mnx1aGMtMVZWNmI3c0dxaFVMcmIwYkF4blE4cU1OVU9LOmV5SmhiR2NpT2lKU1V6VXhNaUo5LmV5SnpkV0lpT2lJMU1tWTROR05tWm1Nd01EQTBOelV5WVdVNU16WmxaVGhrTkROaFlUbGhNeUo5LlJiT1g3Ml9qY21fVTZLSFdVMS1VT19YTUk0aXVuSXZ3SVRWa1dsUDZ2QlU0Qzl0U2pickltM1FuY2Q4WENjeUVoQWY5cUoxVXRtWHlVWk5JODZ3dEo1eVE1STZnU3FlZ2RBMEZ2ek1tQk9qUzZ6eHAxVm83ZmNfVG1oYzBQSERhSWVudlhTZ3BscnNaUWpRckllVUNlUlNuaGRGdDFSdHU0djFaMDh1V1N4Y3k4VGI0dXIyNmRXZWhUSUF6Q0R2aFNkRzBhUVZaRExOYVd5QTE2YjIwSEdFYVNRNmlXUDNQZ05tUWFmTkdyUnNOTFZYd01Qd0ZQU01YMkdsVGhmMXludEpiTXl3UEx1a3pzaEozeFJ3bkFFV0JIdGd2dVlDNUN2YjB1SFlUMDJWVUV1Q21hbDBBQkppUlo4LVdiOV9RZEgxcXNKQzBGRGFUa2dhSjhzSWpleE9qTWtRWVZCeHI4OGpNNVcta0xlbmhLQmRxMTladHFiWTdwMGd0S0JjWjFQczlkSWJRRzJKSTVUZEZza2suika1VDeHdYM1o2QVlBeGJyR2ItSHVBRi1PT0pTdVBKU3RNMDJPblBHdGVRaUVSd0ZOMUZNZjEwRmxjQktlRTIyUGR1SU1sN3gyWTIxalJMR3VZdHBZa0FWVUZ1UEdMM3l3ZVFsWGk4RWYxVG9FV2kwU0dpWXFaNEt1ZWNLVjhUWFhWNHJCUTJSNHhZQmVwZ1F5d2xPNWdiU0xNQ0IwNzkxUlAtRXFyUlNadkJhbERjSlRfbzNxNlk5VWYtYnRuaVBjeTY0bkIxUHZmakxwazNqY0tBdWtBVHZDZjhrUmJKRVkwOFVWQXVpZENadFZiZDByWXpXNS1TUDN3VC10ZGhTb2h5OWNvZ2xTWkZ6aXhlYmJFdkNmalBJ","email":"suika@suika.com"}}}' 
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDnfEBthXuNhuE/3dnGHEfzB9I2aYVmJUMe25sxL9BsX02tj+OA4H30j30tB4hD+AWrP3PEnt8zdD25ggGu1lGE4Ckt38uhMYud/a+amr5oTcIfIYlj/f9mo8VwKzehx7K7G8JWnFXwPjln97MuM/QVg6S+DY2wvGCSG+6MFwrq4/OCBlu1qjyoaE6yMl510n3rOB1WrUcj/LCiHBTsCsytx4fmtbSLxUxMyl3nfA83zhATg+meHKFtLpIEGZbMib7diI8APcOlEX4lOOwqf0lI6bFfTy2pRJ2cLAJXLvQaJL4eOhTOMqzbquj3g+gzkqpMf2R9L/fDTP5pwQqIkJjd root@bastion' 
[root@bastion ~]# cp /opt/install-config.yaml /opt/install/install-config.yaml 
```
### 创建配置：
```bash
[root@bastion ~]# openshift-install create manifests --dir=/opt/install
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 
INFO Manifests created in: /opt/install/manifests and /opt/install/openshift 

[root@bastion ~]# openshift-install create ignition-configs --dir=/opt/install/
INFO Consuming Common Manifests from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming OpenShift Install (Manifests) from target directory 
INFO Consuming Openshift Manifests from target directory 
INFO Consuming Worker Machines from target directory 
INFO Ignition-Configs created in: /opt/install and /opt/install/auth
```
现在的目录结构是这样：  
```
[root@bastion ~]# tree /opt/install
/opt/install
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── bootstrap.sh
├── master.ign
├── master.sh
├── metadata.json
└── worker.ign

1 directory, 8 files
```
### 创建安装脚本：
环境比较特殊，不能搭DHCP所以无法pxe安装，只能配置静态IP，又懒得改镜像，所以就琢磨出这么个办法安装。  
注意脚本中的`'rd.neednet=1'`否则启动的时候有可能主机名变为localhost
```
[root@bastion ~]# cat /var/www/html/install/bootstrap.sh
#!/bin/bash
/bin/coreos-installer install --copy-network --insecure-ignition --ignition-url=http://10.90.18.16:8080/install/bootstrap.ign --firstboot-args 'rd.neednet=1' /dev/vda

[root@bastion ~]# cat /var/www/html/install/master.sh
#!/bin/bash
/bin/coreos-installer install --copy-network --insecure-ignition --ignition-url=http://10.90.18.16:8080/install/master.ign --firstboot-args 'rd.neednet=1' /dev/sda

```
### 启动设备：
打开设备电源并在ilo挂载rhcos镜像：  
![](1609291406688.png)  
![](1609291424913.png)  
![](1609291432187.png)  
用nmtui配置网络，生效后检查是否获取到主机名  
下载上面的安装脚本进行安装：  
![](1609291401163.png)

### 重启设备进行安装：
安装完成后先重启bootstrap，待其重启完成后，并检查有pod后重启三台master主机。  
```bash
[core@bootstrap ~]$ sudo crictl pods
POD ID              CREATED              STATE               NAME                                                           NAMESPACE                             ATTEMPT
519ca2107ede7       33 seconds ago       Ready               bootstrap-kube-scheduler-bootstrap.ocp.suika.com             kube-system                           0
ee3682d35720d       33 seconds ago       Ready               bootstrap-kube-controller-manager-bootstrap.ocp.suika.com    kube-system                           0
4832d0632a339       33 seconds ago       Ready               bootstrap-kube-apiserver-bootstrap.ocp.suika.com             kube-system                           0
1e6aa212b904d       33 seconds ago       Ready               cloud-credential-operator-bootstrap.ocp.suika.com            openshift-cloud-credential-operator   0
4fc5ec6c6e443       33 seconds ago       Ready               bootstrap-cluster-version-operator-bootstrap.ocp.suika.com   openshift-cluster-version             0
fee62b862de12       50 seconds ago       Ready               bootstrap-machine-config-operator-bootstrap.ocp.suika.com    default                               0
f7b83eedcfbc6       About a minute ago   Ready               etcd-bootstrap-member-bootstrap.ocp.suika.com                openshift-etcd                        0
[core@bootstrap ~]$ ss -tulnp|grep 6443
tcp     LISTEN   0        128                    *:6443                 *:*     
[core@bootstrap ~]$ ss -tulnp|grep 22623
tcp     LISTEN   0        128                    *:22623                *:*   
```
重启master的时候需要盯着点ilo，启动完成后虽然可以获取到正确的主机名，但还是要把主机名修改一下，不然后面再重启主机名可能又变回localhost了。  
```
[root@bastion ~]# ssh core@10.90.16.23
The authenticity of host '10.90.16.23 (10.90.16.23)' can't be established.
ECDSA key fingerprint is SHA256:9ZehCh3XkLlk86xu43DiSnIk8bT14UfPGc+dlxX6qzg.
ECDSA key fingerprint is MD5:a7:d7:2c:58:5d:8c:42:44:ec:fa:b0:b8:76:86:7f:14.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.90.16.23' (ECDSA) to the list of known hosts.
Red Hat Enterprise Linux CoreOS 46.82.202010091720-0
  Part of OpenShift 4.6, RHCOS is a Kubernetes native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.6/architecture/architecture-rhcos.html

---
[core@ocpmaster03 ~]$ hostname
ocpmaster03.ocp.suika.com
[core@ocpmaster03 ~]$ sudo hostnamectl set-hostname ocpmaster03.ocp.suika.com

```
### 等待安装完成：
此后剩下的就是等待安装完成，大概要一杯咖啡的时间：  
```
[root@bastion ~]# openshift-install --dir=/opt/install wait-for bootstrap-complete --log-level=debug
DEBUG OpenShift Installer 4.6.4                    
DEBUG Built from commit 6e02d049701437fa81521fe981405745a62c86c5 
INFO Waiting up to 20m0s for the Kubernetes API at https://api.ocp.suika.com:6443... 
INFO API v1.19.0+9f84db3 up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
```
至此可以移除bootstrap节点，并把haproxy中关于bootstrap的信息注释掉，并重启haproxy，然后继续等待安装完成：  
```
[root@bastion ~]# oc get node
NAME                          STATUS   ROLES           AGE    VERSION
ocpmaster01.ocp.suika.com   Ready    master,worker   2m     v1.19.0+9f84db3
ocpmaster02.ocp.suika.com   Ready    master,worker   5m9s   v1.19.0+9f84db3
ocpmaster03.ocp.suika.com   Ready    master,worker   10m    v1.19.0+9f84db3
[root@bastion install]# openshift-install --dir=/opt/install wait-for install-complete --log-level debug
DEBUG OpenShift Installer 4.6.4                    
DEBUG Built from commit 6e02d049701437fa81521fe981405745a62c86c5 
DEBUG Loading Install Config...                    
DEBUG   Loading SSH Key...                         
DEBUG   Loading Base Domain...                     
DEBUG     Loading Platform...                      
DEBUG   Loading Cluster Name...                    
DEBUG     Loading Base Domain...                   
DEBUG     Loading Platform...                      
DEBUG   Loading Pull Secret...                     
DEBUG   Loading Platform...                        
DEBUG Using Install Config loaded from state file  
INFO Waiting up to 40m0s for the cluster at https://api.ocp.suika.com:6443 to initialize... 
DEBUG Cluster is initialized                       
INFO Waiting up to 10m0s for the openshift-console route to be created... 
DEBUG Route found in openshift-console namespace: console 
DEBUG Route found in openshift-console namespace: downloads 
DEBUG OpenShift console route is created           
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/opt/install/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp.suika.com 
INFO Login to the console with user: "kubeadmin", and password: "XviuD-bnp8Y-RNYxL-zcp8R" 
INFO Time elapsed: 0s  
```
注意这个输出的kubeadmin用户及其密码，后面console登录会用到  
## 安装后的工作：
到上面那个输出openshift就已经安装完成了，做一点额外的工作  
### 检查状态：
```
[root@bastion ~]# oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.6.4     True        False         False      90s
cloud-credential                           4.6.4     True        False         False      37m
cluster-autoscaler                         4.6.4     True        False         False      21m
config-operator                            4.6.4     True        False         False      22m
console                                    4.6.4     True        False         False      8m50s
csi-snapshot-controller                    4.6.4     True        False         False      22m
dns                                        4.6.4     True        False         False      20m
etcd                                       4.6.4     True        False         False      15m
image-registry                             4.6.4     True        False         False      12m
ingress                                    4.6.4     True        False         False      12m
insights                                   4.6.4     True        False         False      22m
kube-apiserver                             4.6.4     True        False         False      15m
kube-controller-manager                    4.6.4     True        False         False      20m
kube-scheduler                             4.6.4     True        False         False      16m
kube-storage-version-migrator              4.6.4     True        False         False      20m
machine-api                                4.6.4     True        False         False      21m
machine-approver                           4.6.4     True        False         False      21m
machine-config                             4.6.4     True        False         False      20m
marketplace                                4.6.4     True        False         False      20m
monitoring                                 4.6.4     True        False         False      10m
network                                    4.6.4     True        False         False      23m
node-tuning                                4.6.4     True        False         False      22m
openshift-apiserver                        4.6.4     True        False         False      11m
openshift-controller-manager               4.6.4     True        False         False      19m
openshift-samples                          4.6.4     True        False         False      12m
operator-lifecycle-manager                 4.6.4     True        False         False      21m
operator-lifecycle-manager-catalog         4.6.4     True        False         False      21m
operator-lifecycle-manager-packageserver   4.6.4     True        False         False      12m
service-ca                                 4.6.4     True        False         False      22m
storage                                    4.6.4     True        False         False      22m
```
### 获取console地址：
```
[root@bastion ~]# oc get route -A |grep openshift
openshift-authentication   oauth-openshift     oauth-openshift.apps.ocp.suika.com                                 oauth-openshift     6443       passthrough/Redirect   None
openshift-cnv              test-maven-app      test-maven-app-openshift-cnv.apps.ocp.suika.com                    test-maven-app      8080-tcp                          None
openshift-console          console             console-openshift-console.apps.ocp.suika.com                       console             https      reencrypt/Redirect     None
openshift-console          downloads           downloads-openshift-console.apps.ocp.suika.com                     downloads           http       edge/Redirect          None
openshift-monitoring       alertmanager-main   alertmanager-main-openshift-monitoring.apps.ocp.suika.com          alertmanager-main   web        reencrypt/Redirect     None
openshift-monitoring       grafana             grafana-openshift-monitoring.apps.ocp.suika.com                    grafana             https      reencrypt/Redirect     None
openshift-monitoring       prometheus-k8s      prometheus-k8s-openshift-monitoring.apps.ocp.suika.com             prometheus-k8s      web        reencrypt/Redirect     None
openshift-monitoring       thanos-querier      thanos-querier-openshift-monitoring.apps.ocp.suika.com             thanos-querier      web        reencrypt/Redirect     None
openshift                  nodejs-sample       nodejs-sample-openshift.apps.ocp.suika.com                         nodejs-sample       8080-tcp                          None
```
添加上面的域名解析到任意一个master或haproxy的虚地址，然后就可以用web登录了，用前面的kubeadmin用户及密码


### 配置时钟同步服务：
通过浏览器登录到open shift的console之后，就会发现一个告警，集群没有配置NTP服务，然而又翻了一遍安装文档还是没地方讲这玩意怎么配。  
找了一下红帽的官网，发现解决的办法有两个，一个就是在主机启动后尽快登录，然后配置ntp服务，就是在重启后修改主机名的时候一并修改了NTP，然而这是什么鬼方案，如果不是环境不允许谁没事盯着ilo看启动到哪一步了……  
合理的是采用第二种方案：  
1、创建base64格式的chrony.conf  
```
cat << EOF | base64 -w 0
server suikantp02-in.suika.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
```
2、创建99_masters-chrony-configuration.yaml文件，并写入上面的信息：  
```yaml
[root@bastion opt]# cat 99_masters-chrony-configuration.yaml 
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: masters-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,c2VydmVyIGgzY250cDAyLWluLmgzYy5jb20gaWJ1cnN0CmRyaWZ0ZmlsZSAvdmFyL2xpYi9jaHJvbnkvZHJpZnQKbWFrZXN0ZXAgMS4wIDMKcnRjc3luYwpsb2dkaXIgL3Zhci9sb2cvY2hyb255Cg==
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
```
如果有worker节点创建一个类似的99_workers-chrony-configuration.yaml的文件。

3、应用文件：
```
oc apply -f 99_masters-chrony-configuration.yaml
```
等一会，登录到任意主机，检查一下chrony的状态已是修改后的状态：
```
[core@ocpmaster03 ~]$ cat /etc/chrony.conf 
server suikantp02-in.suika.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
[core@ocpmaster03 ~]$ timedatectl 
               Local time: Mon 2020-12-14 08:12:01 UTC
           Universal time: Mon 2020-12-14 08:12:01 UTC
                 RTC time: Mon 2020-12-14 08:12:01
                Time zone: UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

至此，集群搭建完毕。

## 参考文档：
https://access.redhat.com/solutions/4906341  
https://misa.gitbook.io/k8s-ocp-yaml/openshift-docs/  
https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal.html