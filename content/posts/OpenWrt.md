---
title: "OpenWrt 初识记录"
date: 2021-10-14T22:12:37+08:00
categories: ["All In One Server"]
tags: ["OpenWrt"]
hiddenFromHomePage: true
---

ESXI 不要修改 dispaly name，防止重启找不到 vmdk

管理口注意需要在 ESXI 中设置混淆

![](../media/16342135727671/16343547085004.jpg)

## 安装
1. 创建 3.x Linux 64位 虚拟机
2. 给预留内存以便于硬件直通
> 对应的网口硬件号是 2 3 4 5
> 网口直通：
> - eth0：Control
> - eth1：NAS
> - eth2：LAN
> - eth3：WAN

3. 创建目录上传 vmdk，使用 self 精简版镜像。
4. 选择 BIOS 引导镜像，防止 UEFI 未来需要修改硬盘大小。

启动设置密码。
{{< admonition type=tip title="注意" open=true >}}
设置 ESXI 启动项添加 OpenWrt。
{{< /admonition >}}

## 配置
文档配置会更快更方便，大部分都在 `etc/config`。

### network
设置网口和固定 IP
network 路径：
```cmd
vi /etc/config/network
```
设置 LAN 口固定 IP 后可以使用 `ifdown lanName` 和 `ifup lanName` 重启接口。

### DHCP
dhcp 文件路径：
```cmd
vi /etc/config/dhcp
```
配置格式：
```yaml
config host
	option name 'HostName'
	option dns '1'
	option mac '12:d3:3a:43:bc:54'
	option ip '192.168.50.12'
	option leasetime 'infinite'
```
{{< admonition type=tip title="注意" open=true >}}
**host name** 不可以有空格，否则导致 dhcp server 无法启动。
{{< /admonition >}}

{{< admonition type=note title="提示" open=true >}}
本地无线设备在重启软路由 dhcp 依然无变化，尝试重启 wifi 转发路由器，wifi 设备的 ip 因为保持与路由连接，没有更新。
{{< /admonition >}}

### 动态 DNS
ddns 文件路径：
```cmd
vi /etc/config/ddns
```
创建 [token](https://console.dnspod.cn/account/token/token)
```yaml
config service 'DNSPod'
        option service_name 'dnspod.cn'
        option enabled '1'
        option lookup_host 'domain name'
        option domain 'domain name'
        option username 'ID'
        option password 'Token'
```
不同服务商的 ddns 配置不一样，可以 luci UI 配置。
    
### ShadowSocksR Plus+
服务器节点

订阅 URL 添加

更新所有订阅服务节点

应用节点

保存

为保证 ddns 好用，记得清理缓存。

### 端口映射
配置在防火墙里：
```cmd
vi /etc/config/ddns
```
配置如下：
```yaml
config redirect                                    
        option target 'DNAT'                    
        option src 'wan'                           
        option dest 'lan'                                  
        option proto 'tcp udp'                  
        option src_dport '80'                 
        option dest_ip '192.168.50.1'           
        option dest_port '80'                   
        option name 'Router'
```

### HTTPS
luci UI 配置
```cmd
opkg update && opkg install openssl-util luci-app-uhttpd
```
或者也是在 config 里面
```cmd
vi /etc/config/uhttpd
```
添加 https 监听和证书地址，证书使用 nigix 的。
注意重启 uhttpd
```cmd
/etc/init.d/uhttpd restart
```