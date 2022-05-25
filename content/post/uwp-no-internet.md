---
title: '解决UWP应用无法联网的三种方法'
date: 2020-03-12 14:07:17
tags: [UWP]
published: true
hideInList: false
feature: 
isTop: false
---
---
author: "Suika"
title: "解决UWP应用无法联网的三种方法"
date: "2020-03-12 14:07:17"
description: "怎么会有UWP这种东西"
categories: "Tech"
tags: 
  - "UWP"
  - “Proxy”
image: ""
---
surface Pro 6又坏了，微软的硬件好垃圾，一年多坏了3次，再不会买了，影响工作效率。每次送修的结果就是所有东西都要重配一遍，这次是换机，更彻底一点。然后，又遇到了当初第一次用surface的时候遇到的问题——UWP应用联网。于是把备份的一篇文字直接拿来解决这个问题，再发一遍方便以后找。


---

从Arch到Win10各种不适应，最近这种不适应在一个始料未及的事情上表现的尤为明显——只要开了全局代理，微软自家的UWP程序就无法联网了。

对此，在下也是服气。

然而，发现问题不解决不是吾辈的作风，于是开始找各种解决方案，顺带了解了一下UWP。

这玩意出来也好几年了，一直不温不火的，以至于我从来没有注意到还有这么个玩意。据说所有UWP应用均运行在被称为App Container的虚拟沙箱环境中，其安全性胜于传统的EXE应用。但问题也出在这里，安全的同时它阻止了网络流量发送到本机（即loopback），即阻止了UWP应用访问 localhost。没有访问回环地址的权限，也意味着即便系统设置中启用了本地代理，UWP应用也无法访问本地代理服务器。

按照官方的解决方案，貌似可以用CheckNetIsolation命令来解决这个问题，MSDN轻描淡写的写着命令：
```
checknetisolation loopbackexempt -a -n= UWP应用容器名

checknetisolation loopbackexempt -a -p= UWP应用SID
```
可是谁告诉我UWP的SID去哪里找？UWP的应用容器名称又是啥？

有一种找SID的方法，通过注册表获取：

win+r的运行里输入Regedit打开注册表编辑器，然后定位到
```
HKEY_CURRENT_USERSoftwareClassesLocal 
Settings
Software
MicrosoftWindows
CurrentVersion
App
Container
Mappings 
```
接着在左边的注册表项中找到你想解除网络隔离的应用，右边的 DisplayName 就是应用名称，而左边那一大串字符就是应用的SID值了。  
SID值是人类可以理解的么？对此我可以骂人么？这是人干的事？我不能骂人么？那我还有什么话好说。

这方法还是放弃的好！

官方的解决方案如此复杂，按照问题发生后最简单的思路去解决，那就是关闭代理。

这简单粗暴的方法确实可行，唯一的问题是关闭代理这事在国内是我不能接受的。

接着搜索，互联网给出的答案是网络调试工具Fiddler，然后左上角的WinConfig按钮，排除不要的UWP应用。

可是，Fiddler这工具用来干这个岂不是太大材小用了？

最终，我找到了这个——**Windows-Loopback-Exemption-Manager**

Github上的项目，看来遭这问题困扰的人好多啊

[https://github.com/tiagonmas/Windows-Loopback-Exemption-Manager](https://github.com/tiagonmas/Windows-Loopback-Exemption-Manager)

好了，这玩意跟Fiddler用法差不多，功能更单一，仅此而已。

哦，这东西还有一个坑，就是邮件，用这种方法邮件还是没办法添加Gmail不知道是为啥。

以上就是三种解决方案，其实只能算两种:

如果**想简单粗暴**，那么关闭代理是最直接的,这简直不能算解决方案；  
不然，**就Fiddle或是Windows-Loopback-Exemption-Manager**这两者虽然有坑都还可以接受；  

我想没有人愿意手动去一条条命令敲吧，光是找SID的工作就是非人类的。