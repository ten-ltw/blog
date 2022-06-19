---
title: "OpenWrt 编译以及 ESXI 虚拟机宿主环境安装"
date: 2022-04-14T22:31:37+08:00
categories: ["All In One Server"]
---

# OpenWrt 编译以及 ESXI 虚拟机宿主环境安装
使用 LEDE 不使用原版包，嫌弃折腾插件费劲。
## 安装编译依赖
安装全部依赖
```shell
sudo apt-get -y install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch python3 python2.7 unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler g++-multilib antlr3 gperf wget curl swig rsync
```
找个编译目录拉取代码
```shell
git clone https://github.com/coolsnowwolf/lede.git openwrt
```
无脑给全部权限，注意不要使用 root 账号 make
```
chmod -R 777 openwrt
```

## 编译配置
工程根目录 `feeds.conf.default` 文件配置源，添加如下 git 源：
```
src-git kenzo https://github.com/kenzok8/openwrt-packages
src-git passwall https://github.com/xiaorouji/openwrt-passwall
```
执行：
```
./scripts/feeds update -a && ./scripts/feeds install -a
```
想要的京东签到包需要手动添加：
```shell
cd package/lean/  

git clone https://github.com/jerrykuku/luci-app-jd-dailybonus.git  
```
上面的 feeds 配置执行的脚本实际上就是将 github 包的源文件下载到这个目录中。
执行 `make menuconfig` 的时候就可以看到，他会将所有的包源文件集中到图形界面去配置。

## 镜像使用
编译生成的 vmdk 文件没有使用成功，所以还是使用 img 转成 vmdk 使用。
mac 安装 qemu 会带 qemu-img 工具包：
```
qemu-img convert -f raw openwrt_o.img -O vmdk openwrt_o.vmdk
```
将其转化为 vmdk 后上传到 ESXI 中，并通过 ssh 访问对应的实际路径找到该文件，使用 vmkfstools 将其再次转换为 ESXI 可以使用的虚拟磁盘：
```
vmkfstools -i openwrt_o.vmdk -d thin openwrt.vmdk
```
删除原来的 vmdk 磁盘就可以正常使用了。