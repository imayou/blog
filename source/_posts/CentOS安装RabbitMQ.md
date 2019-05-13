---
title: CentOS安装RabbitMQ
date: 2018-05-25 23:35:12
tags:
---
下载`Erlang linux`版选择`source file`   我选的是[http://erlang.org/download/otp_src_19.0.tar.gz](http://erlang.org/download/otp_src_19.0.tar.gz)

解压
```bash
tar -vxf otp_src_19.0.tar.gz 
```
配置安装，去掉`java`编译

```bash
./configure --prefix=/usr/local/erlang --with-ssl --enable-threads --enable-smp-support --enable-kernel-poll --enable-hipe --without-javac
```
如果提示
```bash
checking for perl... no_perl
configure: error: Perl is required to generate v2 to v1 mib converter script
```
安装`perl`
```bash
yum install perl
```
如果报`No curses library functions found`
```bash
yum install ncurses-devel
```
如果提示`No C++ compiler found `如下：
```bash
*********************************************************************
**********************  APPLICATIONS DISABLED  **********************
*********************************************************************

jinterface     : Java compiler disabled by user
odbc           : ODBC library - link check failed
orber          : No C++ compiler found

*********************************************************************
*********************************************************************
**********************  APPLICATIONS INFORMATION  *******************
*********************************************************************

wx             : wxWidgets not found, wx will NOT be usable

*********************************************************************
*********************************************************************
**********************  DOCUMENTATION INFORMATION  ******************
*********************************************************************

documentation  : 
                 xsltproc is missing.
                 fop is missing.
                 The documentation can not be built.

*********************************************************************
```
安装`gcc`
```bash
yum install gcc gcc-c++
```

如果不需要安装`wxWidgets`的话，`Erlang`的`debugger`无法启动，显示`“ERROR: Could not find 'wxe_driver.so' in: /usr/local/lib/erlang/lib/wx-0.99.1/priv”`，
虽说不使用`debugger`也问题不大，不过有时候调个小程序什么的还是不方便
```bash
yum list *gtk+*    
yum install gtk+extra
```
```bash
yum install bzip2
```
安装`gtk`
```bash
yum install gtk2 gtk2-devel gtk2-devel-docs
```
安装`OpenGL`
```bash
yum install mesa*
yum install freeglut*
```

下载：[http://www.wxwidgets.org/downloads/](http://www.wxwidgets.org/downloads/)
```bash
tar -jxvf wxWidgets-3.1.0.tar.bz2
./configure --with-opengl --enable-debug --enable-unicod
```


然后再去安装`erlang`
```bash
./configure --prefix=/usr/local/erlang --with-ssl --enable-threads --enable-smp-support --enable-kernel-poll --enable-hipe --without-javac

*********************************************************************
**********************  APPLICATIONS DISABLED  **********************
*********************************************************************

jinterface     : Java compiler disabled by user
odbc           : ODBC library - link check failed

*********************************************************************
*********************************************************************
**********************  APPLICATIONS INFORMATION  *******************
*********************************************************************

wx             : Can not link the wx driver, wx will NOT be useable

*********************************************************************
*********************************************************************
**********************  DOCUMENTATION INFORMATION  ******************
*********************************************************************

documentation  : 
                 xsltproc is missing.
                 fop is missing.
                 The documentation can not be built.

*********************************************************************
```


然后就`make`，报错的话就`make clean`一下然后再`make`
```bash
make && make install
```


安装`rabbitmq`
```bash
rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
yum install rabbitmq-server-3.6.5-1.noarch.rpm
```
`erlang`版本问题忽略
```bash
rpm -ivh --nodeps rabbitmq-server-3.6.5-1.noarch.rpm
```