---
author: "Suika"
title: "Caddy翻墙"
date: "2020-04-06 10:09:28"
description: "Fuck GFW"
categories: 
  - "Tech"
tags: 
  - "Proxy"
image: ""
---

Caddy有个模块很好用，经过简单的配置即可以https或http2的方式跨越防火墙，不需要单独申请证书同时还有正常网站做伪装，配合浏览器switch omega插件，方便快捷的访问被墙的资源。
## 安装：
```
curl https://getcaddy.com | bash -s personal http.forwardproxy
```
## 配置一个正常网站：
创建文件并赋权等杂项：  
```bash
setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy
useradd -r -d /var/www -M -s /sbin/nologin caddy
chown -R caddy:caddy /var/www
mkdir /etc/caddy
chown -R root:caddy /etc/caddy
touch /etc/caddy/Caddyfile
chown caddy:caddy /etc/caddy/Caddyfile
mkdir /var/log/caddy
chown caddy:caddy /var/log/caddy/
mkdir -p /var/www/xxx.yyy.zzz
chown -R caddy:caddy /var/www
mkdir -p /etc/ssl/caddy
chown -R caddy:caddy /etc/ssl/caddy
```
编辑配置文件：  
```json
vim /etc/caddy/Caddyfile

https://xxx.yyy.zzz {           //xxx.yyy.zzz为可以解析到本机的域名，下同
    log /var/log/caddy/xxx.log
    errors /var/log/caddy/xxx.error
    root /var/www/xxx.yyy.zzz
    gzip
 
    tls <xxx@xxxxx.xxx> {        //随便一个邮箱
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }

    header / {
        Strict-Transport-Security "max-age=31536000;"
        X-XSS-Protection "1; mode=block"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }
}
```
编辑一个简单的网页：  
```html
vim /var/www/xxx.yyy.zzz/index.html

<!DOCTYPE html>
<html>
  <head>
      <title>Hello from Caddy!</title>
  </head>
  <body>
      <h1 style="font-family: sans-serif">This page is being served via Caddy!!!</h1>
  </body>
</html>

```

测试是否可以正常建立网站：  
```bash
caddy -conf /etc/caddy/Caddyfile
```
按提示填写信息生成证书，如果没有报错，至此，应该可以打开一个网页，内容是加黑的This page is being served via Caddy!!!。

## 配置代理：
在caddy的配置文件中加入forwardproxy信息，加入后内容如下：  
```json
https://xxx.yyy.zzz {           //xxx.yyy.zzz为可以解析到本机的域名，下同
    log /var/log/caddy/xxx.log
    errors /var/log/caddy/xxx.error
    root /var/www/xxx.yyy.zzz
    gzip
 
    tls <xxx@xxxxx.xxx> {        //随便一个邮箱
        ciphers ECDHE-ECDSA-WITH-CHACHA20-POLY1305 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES256-CBC-SHA
        curves p384
        key_type p384
    }

    forwardproxy {
	basicauth xxx yyy      //xxx yyy为用户及密码
	hide_ip
	hide_via
	probe_resistance caddyserver.com
    }

    header / {
        Strict-Transport-Security "max-age=31536000;"
        X-XSS-Protection "1; mode=block"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
    }
}
```
其中probe_resistance可以隐藏代理类型，是个实验功能，可加可不加
## 证书：
caddy默认生成的证书在/root/.caddy下，ln它到/etc/ssl/caddy  
```bash
ln /root/.caddy/acme/acme-v02.api.letsencrypt.org/sites/xxx.yyy.zzz/xxx.yyy.zzz.key /etc/ssl/caddy/
ln /root/.caddy/acme/acme-v02.api.letsencrypt.org/sites/xxx.yyy.zzz/xxx.yyy.zzz.crt /etc/ssl/caddy/
```
## 日志权限：
```bash
chown caddy:caddy /var/log/caddy/*
```

## 管理服务：

配置systemd管理：  
```ini
vim /etc/systemd/system/caddy.service

[Unit]
Description=Caddy HTTP/2 web server
Documentation=https://caddyserver.com/docs
After=network-online.target
Wants=network-online.target systemd-networkd-wait-online.service

[Service]
Restart=on-abnormal

; User and group the process will run as.
User=caddy
Group=caddy

; Letsencrypt-issued certificates will be written to this directory.
Environment=CADDYPATH=/etc/ssl/caddy

; Always set "-root" to something safe in case it gets forgotten in the Caddyfile.
ExecStart=/usr/local/bin/caddy -log stdout -agree=true -conf=/etc/caddy/Caddyfile -root=/var/tmp
ExecReload=/bin/kill -USR1 $MAINPID

; Use graceful shutdown with a reasonable timeout
KillMode=mixed
KillSignal=SIGQUIT
TimeoutStopSec=5s

; Limit the number of file descriptors; see `man systemd.exec` for more limit settings.
LimitNOFILE=1048576
; Unmodified caddy is not expected to use more than that.
LimitNPROC=512

; Use private /tmp and /var/tmp, which are discarded after caddy stops.
PrivateTmp=true
; Use a minimal /dev
PrivateDevices=true
; Hide /home, /root, and /run/user. Nobody will steal your SSH-keys.
ProtectHome=true
; Make /usr, /boot, /etc and possibly some more folders read-only.
ProtectSystem=full
; … except /etc/ssl/caddy, because we want Letsencrypt-certificates there.
;   This merely retains r/w access rights, it does not add any new. Must still be writable on the host!
ReadWriteDirectories=/etc/ssl/caddy

; The following additional security directives only work with systemd v229 or later.
; They further retrict privileges that can be gained by caddy. Uncomment if you like.
; Note that you may have to add capabilities required by any plugins in use.
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target

```

至此，可以通过systemd统一管理caddy服务：  
```bash
systemctl status caddy.service -l
```
## 配置客户端：
PC端firefox及chrome可以安装Switch Omega，然后添加一个https代理即可，注意添加用户名和密码否则每次都要输
IOS端可以shadowrocket，配置http2代理和https代理均可。  

然后，然后就没有然后了……