---
author: "Suika"
title: "RHEL8加密策略改变"
date: "2021-06-25 17:27:15"
description: "这谁知道啊……"
categories: "Tech"
tags: 
  - "RHEL"
image: ""
---

升级了rhel8.4，发现AD域的账户登不上去了。  
日志显示的问题出在：
```ini
(2021-06-24 17:34:18): [krb5_child[39535]] [get_and_save_tgt] (0x0020): 1757: [-1765328370][KDC has no support for encryption type]
(2021-06-24 17:34:18): [krb5_child[39535]] [map_krb5_error] (0x0020): 1849: [-1765328370][KDC has no support for encryption type]
(2021-06-24 17:35:46): [krb5_child[39555]] [validate_tgt] (0x0020): TGT failed verification using key for [host/xxxxx@xxxx.COM].
(2021-06-24 17:35:46): [krb5_child[39555]] [get_and_save_tgt] (0x0020): 1757: [-1765328370][KDC has no support for encryption type]
(2021-06-24 17:35:46): [krb5_child[39555]] [map_krb5_error] (0x0020): 1849: [-1765328370][KDC has no support for encryption type]

```
这是什么鬼？KDC不支持的加密类型？？？可是加密类型是default啊。

```
# cat etc/crypto-policies/config
DEFAULT
```
于是去查了一下，发现真的是加密类型变了，redhat是这么写的：
```
 By default, SSSD supports RC4, AES-128, and AES-256 Kerberos encryption types.

RC4 encryption has been deprecated and disabled by default in RHEL 8, as it is considered less secure than the newer AES-128 and AES-256 encryption types. In contrast, Active Directory (AD) user credentials and trusts between AD domains support RC4 encryption and they might not support AES encryption types. 
```
[Ensuring support for common encryption types in AD and RHEL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/integrating_rhel_systems_directly_with_windows_active_directory/connecting-rhel-systems-directly-to-ad-using-sssd_integrating-rhel-systems-directly-with-active-directory#ensuring-support-for-common-encryption-types-in-ad-and-rhel_connecting-rhel-systems-directly-to-ad-using-sssd)  
[ Identity Management](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/8.3_release_notes/rhel-8-3-0-release#enhancement_identity-management)

既然找到原因，那就好解决了：  
RHEL8.3及以后比较简单，一条命令：  
```bash
# update-crypto-policies --set DEFAULT:AD-SUPPORT
```
提示要重启，实际不重启也可以。


之前的版本虽然不存在这个问题，但要修改默认加密策略的话要麻烦一点：
```ini
# vim /etc/crypto-policies/back-ends/krb5.config
添加+rc4在后面
[libdefaults]
permitted_enctypes = aes256-cts-hmac-sha1-96 aes256-cts-hmac-sha384-192 camellia256-cts-cmac aes128-cts-hmac-sha1-96 aes128-cts-hmac-sha256-128 camellia128-cts-cmac +rc4
```
具体见这里：[修改方法](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/integrating_rhel_systems_directly_with_windows_active_directory/connecting-rhel-systems-directly-to-ad-using-sssd_integrating-rhel-systems-directly-with-active-directory#ensuring-support-for-common-encryption-types-in-ad-and-rhel_connecting-rhel-systems-directly-to-ad-using-sssd)
文档中说是8.2的最麻烦，但实际用8.0和8.1的方法改了也OK

节末的教训是，多看文档总是没有坏处的。