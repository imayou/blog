---
title: 腾讯云更换yum源为阿里云的yum源
date: 2018-05-25 23:31:19
tags:
---
备份base.repo epel.repo
```bash
# cd /etc/yum.repos.d/
# ls
2016-01-19 base.repo epel.repo
# mv base.repo base.repo.back
# mv epel.repo epel.repo.back
```bash
下载阿里的repo，Centos-7代表的为CentOS7，Centos-6代表的为CentOS6，Centos-5代表的为CentOS5…
```bash
# wget -O /etc/yum.repos.d/base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```bash
清空yum缓存
```bash
# yum clean all
```bash
如果出现以下提示
```shell
Another app is currently holding the yum lock; waiting for it to exit…
The other application is: yum
Memory : 34 M RSS (252 MB VSZ)
Started: Wed Jan 20 00:10:40 2016 – 14:53 ago
State : Traced/Stopped, pid: 21527
```shell
则杀掉`pid`为`21527`（不同机器可能不同）的进程
```bash
# kill -9 21527
```bash
然后再清理
```bash
# yum clean all
```bash
最后重新生成缓存
```bash
# yum makecache
```bash