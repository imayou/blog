---
title: CentOS安装最新版docker
date: 2018-05-25 23:32:47
tags:
---
### 1. 卸载当前版本

```shell
#当前版本(通过yum install docker -y 安装)
[root@aerver docker]# docker -v
Docker version 1.10.3, build cb079f6-unsupported
```
卸载
```shell
[root@aerver docker]# yum remove docker -y
#此处省略...
```
清理

```shell
[root@aerver docker]#rpm -qa|grep docker-engine-selinux.noarch 
[root@aerver docker]#rpm -qa|grep docker-engine-selinux
[root@aerver docker]#rpm -qa|grep docker
docker-common-1.10.3-46.el7.centos.14.x86_64
docker-selinux-1.10.3-46.el7.centos.14.x86_64

[root@aerver docker]# rpm -e docker-common && rpm -e docker-selinux
```

### 2. 配置docker yum repo
```bash
#直接粘贴到命令行
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

```

### 3. 安装docker软件包
```shell
[root@aerver docker]#yum install docker-engine -y

[root@aerver docker]#docker -v
Docker version 1.12.3, build 6b644ec

#开机启动
[root@aerver docker]#systemctl enable docker.service
```
