---
author: "Suika"
title: "Ubuntu删除snap"
date: "2020-07-07 09:25:12"
description: "Snap虽方便，但邪教"
categories: "Tech"
tags: 
  - "Snap"
image: ""
---

Snap在ubuntu里简直是病毒一样的存在，一定要用`purge`才能删掉……  
~~顽固病毒也不过如此~~
```bash
# apt autoremove --purge snapd
```
更彻底一点：
```bash
# rm -rf /var/cache/snapd
# rm -rf ~/snap
```