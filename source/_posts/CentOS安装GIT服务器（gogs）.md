---
title: CentOS安装GIT服务器（gogs）
date: 2018-05-25 23:34:39
tags:
---

`Gogs`下载[https://gogs.io/](https://gogs.io/)  我选的是[https://dl.gogs.io/gogs_v0.9.97_linux_amd64.tar.gz](https://dl.gogs.io/gogs_v0.9.97_linux_amd64.tar.gz)

```shell
#安装go
yum install -y go

#下载
wget https://dl.gogs.io/gogs_v0.9.97_linux_amd64.tar.gz

#解压
tar -zvxf gogs_v0.9.97_linux_amd64.tar.gz

#给执行权限
chmod -R 755 gogs

#编辑脚本(pwd获得gogs的路径/home/git/gogs)
#找到
GOGS_HOME=/home/git/gogs
#修改为你的gogs路径
GOGS_HOME=/home/git/gogs

#cp脚本文件到init
cp gogs/scripts/init/centos/gogs /etc/rc.d/init.d/

#添加git用户和组
adduser git
groupadd git

#设置权限
chown -R git:git /home/git/

#大功告成
service gogs start/stop/status/restart

```

如果起不来进入`gogs`目录运行 `./gogs web` [查看官网文档](https://gogs.io/docs/installation/configuration_and_run)

` Gitlab`安装方式 [http://www.saunix.cn/2415.html](http://www.saunix.cn/2415.html)

[Git使用教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
