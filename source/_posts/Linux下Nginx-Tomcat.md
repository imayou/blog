---
title: Linux下Nginx+Tomcat
date: 2018-05-25 23:35:25
tags:
---
### 一、Linux安装软件常用方法
  1 `rpm`（或pkg）安装，类似于Windows安装程序，是预编译好的程序。
  2 使用的是通用参数编译，配置参数不是最佳
  3 可控制性不强，比如对程序特定组件的定制性安装
  4 通常安装包间有复杂依赖关系，操作比较复杂
  5 安装简单，出错机率低
  6 `yum`（或apt-get)安装，改良版的rpm，自动联网下载安装包，自动管理依赖关系
  7 编译安装（方式在各类Linux发行版中差异不大)
  8 可控性强，config时可根据当前系统环境优化参数，可定制组件及安装参数
  9 易出错，难度略高

### 二、Nginx编译安装
1、检查和安装依赖项
```bash
yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```
2、配置安装信息`./configure --prefix=/ayou/service/nginx`(我只加了一个安装目录)
3、编译并安装`make && make install`

查看nginx进程
```bash
ps aux|grep nginx
```
进入`nginx`安装目录`/ayou/service/nginx`下的`sbin`目录
启动`./nginx`
停止 `./nginx -s stop`
重启`./nginx -s reload`

4、配置开机启动

```bash
#vi /etc/rc.d/rc.local
```
加入

```bash
/ayou/service/nginx/sbin/nginx
```

### 三、配置JDK
解压
```bash
#tar -zvxf jdc.tar.gz
```
配置环境变量
```bash
#vi /etc/profile
```
加入
```shell
#JDK
JAVA_HOME=/ayou/service/jdk1.8.0_77
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
```
使环境生效
```bash
source /etc/profile
```

### 四、配置Tomcat
解压
```bash
tar -zvxf tomcat
```
启动
```bash
bin/startup.sh
```
实时查看日志
```bash
tail -f catalina.out
```
停止
```bash
bin/shutdown.sh
```
开机启动
```bash
vi /etc/rc.d/rc.local加入
```
```shell
#tomcat
export JAVA_HOME=/ayou/service/jdk1.8.0_77
$JAVA_HOME/bin/startup.sh
```
注：在`linux`开机时先加载的`rc.d`文件再加载的`profile`文件所以这里必须再次设置`jdk`环境变量`JAVA_HOME`

### 五、防火墙相关(`centos7`使用的`firewall步`骤与之不同)
编辑防火墙
```bash
#vi /etc/sysconfig/iptables
```
检查端口是否开启
```shell
-A INPUT -p tcp -m state –state NEW -m tcp –dport 22 -j ACCEPT
-A INPUT -p tcp -m state –state NEW -m tcp –dport 3306 -j ACCEPT
-A INPUT -p tcp -m state –state NEW -m tcp –dport 8080 -j ACCEPT
-A INPUT -p tcp -m state –state NEW -m tcp –dport 80 -j ACCEPT
```
重启防火墙
```bash
#service iptables restart
```