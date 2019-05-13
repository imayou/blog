---
title: 编译OpenJdk
categories: "java" 
date: 2018-05-25 22:40:08
tags:
---

### 环境
1. `CentOS7`
2. `jdk-7u79`
3. `ccach`(`可不要`，[下载](https://ccache.samba.org/download.html)，下载后`编译安装`即可)

### 准备
1. 下载源码[http://hg.openjdk.java.net/jdk8/jdk8](http://hg.openjdk.java.net/jdk8/jdk8)左侧的`gz`压缩包
2. 安装`hg`
```bash
yum install hg -y
```
  下载安装
```linux
# wget https://www.mercurial-scm.org/release/centos7/RPMS/x86_64/mercurial-3.9.2-1.x86_64.rpm
# chmod 755 mercurial-3.9.2-1.x86_64.rpm
# rpm -ivh mercurial-3.9.2-1.x86_64.rpm
# hg version
Mercurial Distributed SCM (version 3.9.2)
(see https://mercurial-scm.org for more information)
Copyright (C) 2005-2016 Matt Mackall and others
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
3. 解压上面的gz包，并下载源代码
```linux
tar -zvxf jdk-935758609767.tar.gz
mv jdk-935758609767 jdk8
cd jdk8
chmod -R 755 *
 ./get_source.sh #下载需要一段时间
```

### 编译
构建相关说明在`根目录`下的`README-builds.html`
根据说明看`配置`是否缺少相关`依赖`
```bash
./configure --enable-debug --with-target-bits=64 --with-boot-jdk=/tools/jdk/jdk1.7.0_79/
```
根据`提示`安装相关依赖，比如我安装缺少了以下相关`依赖包`（每安装一条然后再执行上面`./configure`便会继续提示你还需要的依赖包）
```bash
yum install libXtst-devel libXt-devel libXrender-devel -y
yum install cups-devel -y
yum install freetype-devel -y
yum install alsa-lib-devel -y
```

`新建一个脚本`贴入以下`代码`（我从书里抄下来的`纯手打`）
```shell
#本脚本摘自《深入理解JAVA虚拟机》

#语言选项，必须设置，否则编译好后会出现HashTable的NPE错
export LANG=C
#Bootstrap JDK的安装路径，必须设置
export ALT_BOOTDIR=/tools/jdk/jdk1.7.0_79/

#允许自动下载依赖
export ALLOW_DOWNLOADS=true

#并行编译的线程数，设置为和CPU内核数量一致即可
export HOTSPOT_BUILD_JOBS=8
export ALT_PARALLEL_COMPILE_JOBS=8

#比较本次build出来的映像与先前版本的差异，这对我们来说没有意义
#必须设置为false，否则sanity检查会报缺少先前版本JDK映射错误提示
#如果已经设置dev或者DEV_ONLY=true，这个不显式设置也行
export SKIP_COMPARE_IMAGES=true

#使用编译头文件，不加这个编译会更慢一些
export USE_PRECOMPILED_HEADER=true

#编译的内容
export BUILD_LANGTOOLS=true
#export BUILD_JAXP=false
#export BUILD_JAXWS=false
#export BUILD_CORBA=false
export BUILD_HOTSPOT=true
export BUILD_JDK=true

#要编译的版本
#export SKIP_DEBUG_BUILD=false
#export SKIP_FASTDEBUG_BUILD=false
#export DEBUG_NAME=debug

#把它设置为false可以避开javaws和浏览器Java插件之类的部分build
BUILD_DEPLOY=false

#把它设置为false就不会build出安装包。因为安装包有些奇怪的依赖；
#但即便不build出它也已经能得到完整的JDK映像
BUILD_INSTALL=false

#编译结果存放路径
export ALT_OUTPUTDIR=/tools/jdk/build/

#这两个变量必须去掉
unset JAVA_HOME
unset CLASSPATH

make 2>&1 | tee $ALT_OUTPUTDIR/build.log

```

给这个脚本`权限`然后`运行`脚本
```bash
chmod 755 ayou-build-openjdk.sh
./ayou-build-openjdk.sh
```

大概跑了`22分钟`左右，如下提示说忽略了一些配置，到此已经`编译`完成了。
```shell
----- Build times -------
Start 2016-11-16 00:04:18
End   2016-11-16 00:26:46
00:00:34 corba
00:15:24 hotspot
00:00:20 jaxp
00:00:31 jaxws
00:05:05 jdk
00:00:32 langtools
00:22:28 TOTAL
-------------------------
Finished building OpenJDK for target 'default'

WARNING: You have the following ALT_ variables set:
ALT_PARALLEL_COMPILE_JOBS=8 ALT_BOOTDIR=/tools/jdk/jdk1.7.0_79/ ALT_OUTPUTDIR=/tools/jdk/build/
ALT_ variables are deprecated and will be ignored. Please clean your environment.
```


### 使用
进入`jdk`根目录下的`build`目录下的`linux-x86_64-normal-server-fastdebug`目录，然后目录jdk加到path就可以了
```shell
JAVA_HOME=/tools/jdk/jdk8/build/linux-x86_64-normal-server-fastdebug/jdk
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
```

然后查看版本
```shell
#使刚加的`环境变量`生效
source /etc/profile

#查看版本
java -version
openjdk version "1.8.0-internal-fastdebug"
OpenJDK Runtime Environment (build 1.8.0-internal-fastdebug-root_2016_11_16_00_03-b00)
OpenJDK 64-Bit Server VM (build 25.0-b70-fastdebug, mixed mode)

```
