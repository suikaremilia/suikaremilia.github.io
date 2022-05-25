---
author: "Suika"
title: "记一次博科光交排障"
date: "2020-08-03 19:24:01"
description: ""
categories: "Tech"
tags: 
  - "3PAR"
  - "Brocade"
image: ""
---
最近一台3PAR一直报CRC错误，3PAR的排障还算简单，就几条命令分析一下输出。  
然而，这次故障比较诡异：
```ini
% showport
N:S:P      Mode   State ----Node_WWN---- -Port_WWN/HW_Addr- Type Protocol Label Partner FailoverState
0:0:1    target   ready 2FF70002AC01AE3E   20010002AC01AE3E host       FC     -   1:0:1          none
0:0:2    target   ready 2FF70002AC01AE3E   20020002AC01AE3E host       FC     -   1:0:2          none
0:1:1 initiator   ready 50002ACFF701AE3E   50002AC01101AE3E disk      SAS  DP-1       -             -
0:1:2 initiator   ready 50002ACFF701AE3E   50002AC01201AE3E disk      SAS  DP-2       -             -
0:3:1      peer offline                -       3464A9EAFD3D free       IP   IP0       -             -
1:0:1    target   ready 2FF70002AC01AE3E   21010002AC01AE3E host       FC     -   0:0:1          none
1:0:2    target   ready 2FF70002AC01AE3E   21020002AC01AE3E host       FC     -   0:0:2          none
1:1:1 initiator   ready 50002ACFF701AE3E   50002AC11101AE3E disk      SAS  DP-1       -             -
1:1:2 initiator   ready 50002ACFF701AE3E   50002AC11201AE3E disk      SAS  DP-2       -             -
1:3:1      peer offline                -       94188246CFDD free       IP   IP1       -             -
-----------------------------------------------------------------------------------------------------

% showportlesb hist 1:0:1
ID    ALPA ----Port_WWN---- LinkFail LossSync LossSig PrimSeq InvWord InvCRC
<1:0:1> 0x15e00 21010002AC01AE3E        2        3       0       0     163  19870
host42  0x15300 51402EC000F79136        2        0       0       0       0      0
host43  0x15400 51402EC000F77CFA        2        0       0       0       0      0
host44  0x15500 51402EC000F79236        2        0       0       0       0      0
host45  0x15600 51402EC000F77D06        2        0       0       0       0      0
host48  0x15900 51402EC000F77F0E        2        0       0       0       0      0
```
一般情况3PAR端`showportlesb`会显示哪个host报crc错误，这次直接报接在同一交换机上的两个控制器端口<1:0:1>和<0:0:1>故障，这让人不得不怀疑控制器其实是没有故障的，故障在交换机侧，毕竟不同控制器的端口同时报错概率不大。

于是，清除3par侧的计数

```
showportlesb reset
```


继续排查交换机侧：
```ini
> porterrshow 
          frames      enc    crc    crc    too    too    bad    enc   disc   link   loss   loss   frjt   fbsy    c3timeout    pcs
       tx     rx      in    err    g_eof  shrt   long   eof     out   c3    fail    sync   sig                   tx    rx     err
  0:  463.2m 201.1m   0      0      0      0      0      0      0      0      0      0      0      0      0      0      0      0   
  1:  462.4m 200.9m   0      0      0      0      0      0      0      0      0      0      0      0      0      0      0      0
 85:   32.6m  56.0m 424    423    423      0      0      0      0      0      0      0      0      0      0      0      0      0   
 86:    4.6m   4.9m   0      0      0      0      0      0      0      0      0      0      0      0      0      0      0      0

```
果然一个主机的端口报错，考虑端口光衰
```bash
> sfpshow 85
Identifier:  3    SFP
Connector:   7    LC
Transceiver: 540c404000000000 2,4,8_Gbps M5,M6 sw Short_dist
Encoding:    1    8B10B
Baud Rate:   85   (units 100 megabaud)
Length 9u:   0    (units km)
Length 9u:   0    (units 100 meters)
Length 50u (OM2):  5    (units 10 meters)
Length 50u (OM3):  0    (units 10 meters)
Length 62.5u:2    (units 10 meters)
Length Cu:   0    (units 1 meter)
Vendor Name: BROCADE         
Vendor OUI:  00:05:1e
Vendor PN:   57-1000012-01   
Vendor Rev:  A   
Wavelength:  850  (units nm)
Options:     003a Loss_of_Sig,Tx_Fault,Tx_Disable
BR Max:      0   
BR Min:      0   
Serial No:   UAF109390000C3N 
Date Code:   090924  
DD Type:     0x68
Enh Options: 0xfa
Status/Ctrl: 0x82
Alarm flags[0,1] = 0x5, 0x40
Warn Flags[0,1] = 0x5, 0x40
                                           Alarm                  Warn
                                    low         high       low         high
Temperature: 42      Centigrade      -10        90         -5          85
Current:     7.140   mAmps           1.000      17.000     2.000       14.000 
Voltage:     3292.4  mVolts          2900.0     3700.0     3000.0      3600.0 
RX Power:    -2.3    dBm (582.9uW)   10.0   uW  1258.9 uW  15.8   uW   1000.0 uW
TX Power:    -3.3    dBm (263.1 uW)  125.9  uW  631.0  uW  158.5  uW   562.3  uW

State transitions: 1
Last poll time: 08-03-2020 UTC Thu 08:12:57
```
`TX Power`才200多，过低，换之，遂好……



## 附：博科交换机常用的内容如下：
**默认IP:**  
10.77.77.77  
**默认密码：**
admin/password  
root/fibrane  
**查看、设置网络：**  
ipaddrshow  
ipaddrset  
**显示、添加license：**  
licenseshow  
licenseadd  
**查看信息：**  
switchshow  
alishow  
zoneshow  
cfgshow  
fabrishow  
**配置时钟：**  
tsclockserver "ntp server"  
tstimezone --interactive  
**清除状态：**  
statsclear  

排障的话就需要用到`porterrshow`，上面有它的输出，配合statsclear会得到一定时间范围内所有端口错误统计信息：

**Frame(tx/rx)：** tx代表端口发送的数据帧，rx代表端口收到的数据帧。
 
**Enc_in：** 8b/10b或者64b/6bb数据帧帧内编码错误。在正常情况下20分钟会出现一次这个报错，交换机端口（offline/online）会产生这个错误。
 
**Crc_err：** 数据帧CRC校验错误。根据实际统计，如果crc_err和enc_out同时出现，通常代表GBIC/SFP有硬件问题。
 
**Crc_g_eof：** 数据帧CRC校验错误，但是数据帧EOF是正常的。
 
**Too_long：** 数据帧总长度超过2148字节或者workload长度超过2112字节。
 
**Too_short：** 小于36个字节长度的帧（workload字节长度等于0）。
 
**Bad_eof：** 数据帧EOF错误。
 
**Enc_out：** 8b/10b或者64b/66b数据帧帧外编码错误。在正常情况下20分钟会出现一次这个报错，交换机端口（offline/online）会产生这个报错，另外在HBA卡和交换机端口速率不同，而又使用的是静态配置端口速率的时候也会产生这个错误。单一的这个报错反映光纤线可能有问题；如果是Enc_out和crc_err同时报错代表GBIC/SFP有硬件问题。
 
**Disc c3：** Class 3被交换机丢弃的数据帧。常见情形帧的目标地址不可达或者源端口还没有FLOGI交换机。这个参数仅仅代表有丢包发生，不能用来判定问题的具体原因。
 
**Link-fail：** 当交换机端口在LR Receive State时间超过R_A_TOV就会产生这个错误。这个错误经常和loss of signal或者loss of sync同时出现。
 
**Loss sync：** bit或者transmission-word synchronization失败都会产生这个错误。当交换机端口（offline/online）会产生这个问题。
 
**Loss sig：** 链路收不到信号。当交换机端口（offline/online）会产生这个问题。
 
**Frjt：** 用于class 2。代表数据帧无法处理。
 
**Frbsy：** 用于class 2。数据帧无法在E_D_TOV时间内传输出去，超时后会产生这个问题。

一般情况下：  
如果仅是"enc out "单独报错主要是因为光纤线的问题。  
如果是"enc out"和"crc err"组合报错主要是GBIC/SFP的问题。

要确定是源端还是目标端SFP报错，需要再检查"portshow x" 的输出（x代表有问题端口号） 如果下面两对参数 "Lr_in " 和 "Ols_out " 以及 "Lr_out " 和"Ols_in " 的值相同，则表明SFP运行正常如果一个数值明显高于另一个, 连接问题可能出现在交换机连接的对端("in" > "out") 或是交换机本身("out" > "in").

如果”Ols_in”的值高于“Lr_out”的值，问题的根源大多数情况与连接的设备相关，(sending those offline sequences) 并且交换机通过"link reset"对此做出响应。

Loss sync，Loss sig，Link-fail这三个错误在链路初始化的过程中都会产生。当链路不稳定时候，通常这些错误计数器比较高。

Frjt，Frbsy用于class 2。SAN存储通常使用的是class 3，所以这两个错误很少见。

Enc_out和Crc_err两个计数器同时比较高，通常需要更换GBIC/SFP。

Disk c3只能代表链路有丢包现象。原因可能有很多种，具体问题具体分析。如果这个值过高，链路性能可能会受到影响。