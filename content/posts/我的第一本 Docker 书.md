---
title: "我的第一本 Docker 书"
date: 2021-10-30T12:12:31+08:00
categories: ["Reading"]
---

# 我的第一本 Docker 书
可以作为工具书使用
摘录一下目前我需要经常遇到的一些 cli
## docker 启动停止
### docker run
docker run 命令，并指定了 -i 和 -t 两个命令行参数。 -i 标志保证容器中 STDIN 是开启的，尽管我们并没有附着到容器中。持久的标准输入是交互式 shell 的“半边天”， -t 标志则是另外“半边天”，它告诉 Docker 为要创建的容器分配一个伪 tty 终端。
> docker run -i -t image_name /bin/bash
### docker start
开启容器
> docker start container_name
### docker attach
附着容器，就是退出容器后再进入
> docker attach container_name
### daemonized container
守护式容器（ daemonized container ）没有交互式会话，非常适合运行应用程序和服务。
> docker run --name daemon_dave -d ubuntu /bin/sh 
-d 会在后台运行
### docker stop
停止容器
docker stop 命令会向 Docker 容器进程发送 SIGTERM 信号。如果想快速停止某个容器，也可以使用 docker kill 命令来向容器进程发送 SIGKILL 信号。
> docker stop container_name
>
> docker kill container_name
### restart 设置重启条件
设置重启条件
> docker run --restart=always --name daemon_dave -d ubuntu /bin/bash -c "while true; do echo hello world; sleep 1; done"
## DockerFile
### DockerFile
- 每条指令都是大写字母，而且后面都会跟随一个参数
- EXPOSE 指令向外部开放多个端口
- RUN命令执行命令并创建新的镜像层，通常用于安装软件包
- CMD命令设置容器启动后默认执行的命令及其参数，但CMD设置的命令能够被`docker run`命令后面的命令行参数替换
- ENTRYPOINT配置容器启动时的执行命令（不会被忽略，一定会被执行，即使运行 `docker run`时指定了其他命令）
### 开放端口
用 docker ps 命令来看一下容器的端口分配情况，docker port static_web 80 0.0.0.0:49154。   
docker run -d -p 80:80 宿主机 端口在前。   Docker 还提供了一个更简单的方式，即 -P 参数，该参数可以用来对外公开在 Dockerfile 中通过 EXPOSE 指令公开的所有端口。