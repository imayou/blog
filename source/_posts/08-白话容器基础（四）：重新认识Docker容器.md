---
layout: docker、kubernate
title: 08-白话容器基础（四）：重新认识Docker容器
date: 2019-05-8 10:04:42
tags: docker、kubernate
---
[原文](https://time.geekbang.org/column/article/23132)
```shell
# 使用官方提供的 Python 开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为 /app
WORKDIR /app

# 将当前目录下的所有内容复制到 /app 下
ADD . /app

# 使用 pip 命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的 80 端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个 Python 应用的启动命令
CMD ["python", "app.py"]

```
FORM指定了`python:2.7-slim`这个官方基础镜像，如果不这么写得这么写：
```shell
FROM ubuntu:latest
RUN apt-get update -yRUN apt-get install -y python-pip python-dev build-essential
...
```
其中，RUN原语就是在容器里执行shell命令的意思，而WORKDIR，意思是在这一句之后，Dockerfile后面的操作都以这一句指定的/app目录作为当前目录。

CMD ["python","app.py"]=="docker run python app.py"

`ENTRYPOINT`和`CMD`都是Docker容器进程启动所必须的参数，完整格式是：`ENTRYPOINT COM`， 默认情况下Docker会为你提供一个隐含的`ENTRYPOINT`即：/bin/sh -c，所以在不指定`ENTRPOINT`时，实际运行在容器里的完整进程是：/bin/sh -c "python app.py"，即CMD的内容就是ENTRYPOINT的参数
> so 称Docker容器的启动进程为ENTRYPOINT，而不是CMD

```shell
docker build -t helloworld .
```

`-t` 的作用是给这个镜像加一个Tag，即：起一个好听的名字，`docker build` 会自动加载当前目录下的Dockerfile文件，然后按照顺序，执行文件中的原语，实际上可以等同于Docker使用基础镜像启动了一个容器，然后在容器中一次执行`Dockerfile`中的原语

> Dockerfile中的每个原语执行后，都会生成一个对应的镜像层，即使原语本身并没有明显的修改文件的操作（比如，ENV原语），它对应的层也会存在，只不过在外界看来，这个层是空的。

上传镜像需要注册Docker Hub帐号 然后docker login登录，用docker tag命令给容器起一个完整的名字：
```shell
docker tag helloworld ayou/helloworld:v1
```
然后执行`docker push`把镜像上传到Docker Hub上

可以用docker commit把一个正在运行的容器直接提交为一个镜像，可以在容器运行起来后，又对容器做了一些操作，并且要把操作保存到镜像里：
```shell
$ docker exec -it 4ddf4638572d /bin/sh
# 在容器内部新建了一个文件
root@4ddf4638572d:/app# touch test.txt
root@4ddf4638572d:/app# exit

# 将这个新建的文件提交到镜像中保存
$ docker commit 4ddf4638572d geektime/helloworld:v2
```

> docker exec 是怎么做到进入容器里的呢？

Linux Namespace创建的隔离空间虽然看不见摸不着，但一个进程的Namespace信息在宿主机上是以一个真实的文件存在的。

比如查看docker容器的PID：
```shell
$ docker inspect --format '{{ .State.Pid }}'  4ddf4638572d
25686
```
通过PID查看宿主机的proc文件
```shell
$ ls -l  /proc/25686/ns
total 0
lrwxrwxrwx 1 root root 0 Aug 13 14:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 ipc -> ipc:[4026532278]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 mnt -> mnt:[4026532276]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 net -> net:[4026532281]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid_for_children -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 uts -> uts:[4026532277]
```

可见每种Linux Namespace，都在对应的/proc/[PID]/ns下有一个对应的虚拟文件，并且链接到一个真实的Namespace文件上。这样我们就可以对Namespace搞事情了，比如加入到一个已经存在的Namespace当中，也就意味着：一个进程，可以选择加入到某个进程已有的Namespace当中，从而达到“进入”这个进程所在容器的目的，这正是docker exec的实现原理。依赖于一个名叫setns()的linux系统调用。如下：
```shell
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```
共2参数
- argv[1]是当前要加入的Namespace文件的路径，比如/prod/25686/hs/net
- argv[2]是你要在这个Namespace里运行的进程，比如/bin/bash

> 代码核心操作，通过open()系统调用打开指定的Namespace文件，并把这个文件的描述符fd传给setns()，setns()执行后，当前进程就加入了这个文件对应的Linux Namespace 当中了

例：编译并加入到容器进程PID=25686的Network Namespace中
```shell
$ gcc -o set_ns set_ns.c 
$ ./set_ns /proc/25686/ns/net /bin/bash 
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02  
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
	   collisions:0 txqueuelen:0 
          RX bytes:976 (976.0 B)  TX bytes:796 (796.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	  collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
在宿主机上可以找到到这个set_ns程序执行的/bin/bash真实的PID是28499
```shell
# 在宿主机上
ps aux | grep /bin/bash
root     28499  0.0  0.0 19944  3612 pts/0    S    14:15   0:00 /bin/bash
```
查看PID=28499进程的Namespace
```shell
$ ls -l /proc/28499/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:18 /proc/28499/ns/net -> net:[4026532281]

$ ls -l  /proc/25686/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:05 /proc/25686/ns/net -> net:[4026532281]
```
在/proc/[PID]/ns/net目录下，这个PID=28499进程与我们之前的docker容器进程（PID=25686）指向的Network Namespace文件完全一样，说明这个两个进程共享了这个net:[4026532281]的Network Namespace

`--net` 参数可以让你在启动一个容器的是并`加入`到另一个容器的Network Namespace里
```shell
$ docker run -it --net container:4ddf4638572d busybox ifconfig
```
这样容器启动的时候就会直接加入到ID=4ddf4638572d的容器，也就是前面的python容器应用容器（PID=25686）的Network Namespace中，所以ifconfig返回的网卡信息和前面的小程序信息一模一样。

`-net-host`不会为进程启动Network Namespace，意味着这个容器拆除了Network Namespace的`隔离墙`，就和普通进程已有共享宿主机的网络栈。
