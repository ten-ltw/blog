---
title: "Ubuntu server"
date: 2021-10-30T16:40:37+08:00
categories: ["All In One Server"]
tags: ["Ubuntu"]
---

# Ubuntu server
用于一些自己写的智能家居服务和开源项目。

## Ubuntu 安装
安装时以下个人配置

### 静态 IP
![](../media/16355575177750/16355579818615.jpg)

name servers 没有添加，导致系统无法解析域名。
`/etc/netplan/00-installer-config.yaml` 文件用来配置网络

### 分区
默认分配一半空间给 root 和 boot，留一半 free space，重新分区。
防止未来编译内核，给 boot 10 个 G。
![](../media/16355575177750/16355596791765.jpg)

### 额外服务
直接打开 open ssh server
安装时可以使用 snap 安装 nextcloud 和 docker，但是 docker 使用中有一些问题，留着自己安装。

## Samba service
安装
```cmd
sudo apt install samba samba-common
```
创建目录配置权限
```cmd
mkdir /home/volumes
chown nobody:nogroup /home/volumes
chmod 777 /home/volumes
```
配置 `/etc/samba/smb.conf`
```cmd
vim /etc/samba/smb.conf
```
```toml
[Volumes]
  comment = TimeCapsule Volumes
  path = /home/ten/openwrt/bin
  browseable = yes
  writable = yes
  available = yes
  valid users = [a user name]
```
添加用户
```cmd
smbpasswd -a [a user name]
```
重启服务
```cmd
sudo service smbd restart
```

## docker 安装
首先清理 docker
```cmd
sudo apt-get remove docker docker-engine docker.io containerd runc
```
安装
```cmd
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

```cmd
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
```cmd
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```cmd
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

## 安装 LMS 提供 airplay 给 HA
使用了别人的 docker compose
```
wget https://www.justplus.com.tw/uploads/7/6/7/0/76706939/docker-compose_lms.yml.zip
```
解压 安装 docker-compose
```
docker-compose up -d
```

## code server
以 code server 为例，部署 docker 项目的项目。
### docker compose 部署
docker-compose.yml
```
---
version: "2.1"
services:
  code-server:
    image: lscr.io/linuxserver/code-server
    container_name: code-server
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - PASSWORD= #optional
      - HASHED_PASSWORD= #optional
      - SUDO_PASSWORD= #optional
      - SUDO_PASSWORD_HASH= #optional
      - PROXY_DOMAIN=https:code.2019919.xyz #optional
    volumes:
      - /home/volumes/code_server:/config #直接将其文件放在SMB路径下，方便管理
    ports:
      - 8082:8443
    restart: unless-stopped
```
```
docker-compose up -d
```
### nginx 代理 https
#### 安装 nginx
```
apt-get install nginx
```
#### 上传证书
直接将证书目录指定到 SMB 共享目录的 ssl 下方便每年维护
#### 创建配置文件
进入可用网站配置目录 `/etc/nginx/sites-available`，新建一个code-server文件 `vi code-server`
添加以下内容
```
server {
    # 填写nginx向公网开放的端口
    listen 8083 ssl;
    # 填写SSL证书对应的域名
    server_name domain;

    # SSL证书文件，请替换成对应的绝对路径和文件名
    ssl_certificate /home/volumes/ssl/2021.pem;
    # SSL证书密钥文件，请替换成对应的绝对路径和文件名
    ssl_certificate_key /home/volumes/ssl/2021.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
      # 填写nginx代理的本地端口，请把端口替换成上面code-server配置的实际端口
      proxy_pass http://localhost:8082/;
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }

    # 使用497状态码检测HTTP请求，自动转换为HTTPS
    error_page 497 https://$host:$server_port$uri$is_args$args;
}
```
#### 应用配置
```
cd /etc/nginx

# 把可用网站配置目录中的 code-server 配置拷贝到已启用网站配置目录
cp sites-available/code-server sites-enabled/code-server

# 删除已启用网站配置目录中的默认网站配置
rm sites-enabled/default

# 检查配置文件是否存在错误，不知道为啥软连接过来的文件显示不存在
sudo nginx -t

# 重新加载配置文件
sudo nginx -s reload

# 重启 nginx 服务
sudo systemctl restart nginx
```
OK，Https 访问。
回头 electron 打包给 win mac ipadOS。

## mongoDB
```
docker pull mongo
```
将文件挂载到共享目录
```
docker run -p 27017:27017 -v /home/volumes/mongo:/data/db --name mongodb -d mongo
```

```yaml
# Use root/example as user/password credentials
version: '3.1'

services:

  mongo:
    image: mongo
    restart: always
    ports:
      - 27017:27017
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example
    volumes:
      # folder where lms stores its data (cache, logs, prefs)
      - /home/volumes/mongo:/data/db

  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8084:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: example
      ME_CONFIG_MONGODB_URL: mongodb://root:example@mongo:27017/
```