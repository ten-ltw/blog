---
title: "ESXI 相关配置"
date: 2021-09-05T07:39:37+08:00
categories: ["All In One Server"]
tags: ["ESXI"]
hiddenFromHomePage: true
---

记录一些 ESXI 使用中遇到的问题。
## 虚拟交换机
![](../media/16307987538087/16482642367494.jpg)
![](../media/16307987538087/16482642538821.jpg)

## usb 硬件直通

lspci -v 获取所有芯片对应供应商 ID 和设备 ID

lspci -v | grep "Class 0c03"     这的0C03就是类ID0XC03

vi /etc/vmware/passthru.map
8086:7ae2
# Mass storage controller SATA controller: Intel Corporation Device 7ae2
8086  7ae2  d3d0     false
> Intel Corporation 8 Series/C220 Series Chipset Family USB xHCI
8086  8C31  d3d0     false
其中8086是供应商ID   8C31是设备ID    d3d0和false固定值

# Mass storage controller SATA controller: ASMedia Technology Inc. Device 1064 [vmhba2]
1b21  1064  d3d0     false
Class 0106: 1b21:1064

## SSL

上传改名后的 SSL

![](../media/16307987538087/16355165877304.jpg)

在根目录下搜索 rui.crt

> /vmfs/volumes/datastore1

路径下是上传的 SSL
备份 `/etc/vmware/ssl` 中的 SSL 证书，用上传的覆盖

```
/etc/init.d/hostd restart  #重启hostd服务
/etc/init.d/vpxa restart   #重启vpxa服务
/etc/init.d/vpxa start     #启动vpxa服务
/etc/init.d/hostd start    #启动hostd服务
```

## 许可证
JH09A-2YL84-M7EC8-FL0K2-3N2J2
[许可证可下载](https://customerconnect.vmware.com/cn/group/vmware/evalcenter?p=free-esxi6)
登录自己用户注册产品获取免费许可证。
![](../media/16307987538087/16355623064733.jpg)
分配许可证。