---
title: "OpenWrt 添加 SSL 证书后 NetData 和 TTYD 拒绝请求"
date: 2022-04-10T21:06:37+08:00
categories: ["All In One Server"]
tags: ["OpenWrt"]
hiddenFromHomePage: false
---

原因是 iframe 中 hardcode 访问 http 协议。
直接通过 nginx 服务器反向代理添加于 OpenWrt 相同 SSL 证书解决。

## 添加 SSL 证书

> vim /etc/nginx/sites-available/netdata

```config
server {
    listen 19999 ssl;
    server_name domain;

    ssl_certificate 【pem 证书】;
    ssl_certificate_key 【key】;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
      proxy_pass http://192.168.50.1:19999/
      proxy_set_header Host $host;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
    }
}
```

添加 link 到 enabled 路径时注意不要使用相对路径：

> ln -s /etc/nginx/sites-available/netdata /etc/nginx/sites-enabled/netdata

`nginx -t` 验证后重启 nginx 服务

> service nginx restart

路由器映射 nginx 服务器 19999 端口后测试能否使用 https 协议域名访问 NetData。

## 修改静态 html
![](../media/16495959981816/16495984398450.jpg)

根目录下搜索路由 netdata 找到对应静态 html：

> find / -name netdata

搜索到有以下两个路径：

> /rom/usr/lib/lua/luci/view/netdata
> /usr/lib/lua/luci/view/netdata

rom 目录是用来 reset 使用的，如果想 reset 时候也保持修改，可以同时修改两处。

进入目录 `/usr/lib/lua/luci/view/netdata` 后就能看到静态的 'netdata.htm'。
有需求 http 访问的可以通过 `document.location.protocol` 判断协议，再给目标 url 赋值。
修改 iframe hardcode 的 protocol 和 端口号：

```js
        if(document.location.protocol.indexOf('https')>-1){
                document.getElementById("netdata").src = "https://" + window.location.hostname + ":8134";
        } else {
                document.getElementById("netdata").src = "http://" + window.location.hostname + ":19999";
        }
```
OK 刷新画面搞定。

## 最后
还用什么 uhttpd，直接全部 nginx 代理。