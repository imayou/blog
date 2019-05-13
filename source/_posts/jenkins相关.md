---
layout: jenkins
title: jenkins相关
date: 2019-05-8 10:16:22
tags: jenkins
---
#### 1. maven打包
Maven Integration plugin
> 全局工具配置>Maven 安装>指定MAVEN_HOME和NAME>取消自动安装

#### 2. SSH
Publish Over SSH
> 发送文件到远程linux服务器并执行一个脚本

```java

SSH Server 
Name:	PITC
Hostname 10.196.20.159
Username user
Remote Directory：/home/user/
```

SSH plugin
> 链接远程服务器并执行命令

#### 3. Job
Multijob plugin
> 组合不同的Job构建，可实现模块化持续集成


```
#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

cd /home/user/project

if [ ! -d "back/`date +%Y-%m-%d`" ]; then
  mkdir -p back/`date +%Y-%m-%d`
fi

cp -r  print-*  back/`date +%Y-%m-%d`
```

#### jenkins启动

```bat
cls 
@ECHO OFF
title jenkins

set path=D:\jdk1.8\bin;%path%

set path=D:\jdk1.8\jre;%path%

java -Dnet.sf.ehcache.skipUpdateCheck=true -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+UseParNewGC -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m -jar jenkins.war &

pause...
```

#### 静态文件处理脚本
```
#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

lp='/home/user/project'
echo $lp/

chmod -R 777 *.zip

unzip -o $lp/print-admin-static-0.0.1-SNAPSHOT-static.zip -d $lp/print-temp
unzip -o $lp/print-external-static-0.0.1-SNAPSHOT-static.zip -d $lp/print-temp
unzip -o $lp/print-web-base-0.0.1-SNAPSHOT-static.zip -d $lp/print-temp

rm -rf $lp/print-admin-static
rm -rf $lp/print-external-static
rm -rf $lp/print-web-base

mkdir -p $lp/print-admin-static/webroot
mkdir -p $lp/print-external-static/webroot
mkdir -p $lp/print-web-base/webroot

mv $lp/print-temp/print-admin-static-0.0.1-SNAPSHOT/* $lp/print-admin-static/webroot
mv $lp/print-temp/print-external-static-0.0.1-SNAPSHOT/* $lp/print-external-static/webroot
mv $lp/print-temp/print-web-base-0.0.1-SNAPSHOT/* $lp/print-web-base/webroot

rm -rf $lp/print-temp
```

#### 重启脚本
```

#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

#SERVICES="print-admin-site-0.0.1-SNAPSHOT print-data-webservice-0.0.1-SNAPSHOT print-data-0.0.1-SNAPSHOT print-external-provider-0.0.1-SNAPSHOT print-destroy-provider-0.0.1-SNAPSHOT print-destroy-site-0.0.1-SNAPSHOT print-external-site-0.0.1-SNAPSHOT print-provider-0.0.1-SNAPSHOT"
SERVICES="print-external-provider-0.0.1-SNAPSHOT print-destroy-provider-0.0.1-SNAPSHOT print-destroy-site-0.0.1-SNAPSHOT print-external-site-0.0.1-SNAPSHOT"
#SERVICES="print-admin-site-0.0.1-SNAPSHOT print-data-webservice-0.0.1-SNAPSHOT print-data-0.0.1-SNAPSHOT print-provider-0.0.1-SNAPSHOT"

IP=`LC_ALL=C ifconfig|grep "inet addr:"|grep -v "127.0.0.1"|cut -d: -f2|awk '{print $1}'`
ENV='it'
FILE_PATH='/home/user/project'

for s in $SERVICES
do
    PID=`ps -ef|grep java|grep $s|awk '{print $2}'`
    if [ "$PID" != "" ]
    then
        echo -e "\033[35m KILLING PID ${PID} TO STOP server $s \033[m"
        kill -9 ${PID}
        sleep 2
    fi
    echo -e "\033[36m start server $s... \033[m"
    nohup java -jar -Dmyip="${IP}" $FILE_PATH/$s.jar --spring.profiles.active=$ENV >  $FILE_PATH/nohup/$s-nohup.out 2>&1 &
    sleep 20
done

exit 0
```
#### 备份脚本
```
#!/bin/bash
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

cd /home/user/project

if [ ! -d "back/`date +%Y-%m-%d`" ]; then
  mkdir -p back/`date +%Y-%m-%d`
fi

cp -r  print-*  back/`date +%Y-%m-%d`

```

#### WIN下copy文件脚本
```
set LOCAL=%cd%
cd ../PRINT

for /r %cd% %%a in (.) do @if exist %%a\print-*.jar xcopy %%a\print-*.jar %LOCAL%  /C /E /H /K /R /Y
```

### win下搜索拷贝jar和静态文件
```
set LOCAL=%cd%
cd ../PRINT
for /r %cd% %%a in (.) do @if exist %%a\*static*.zip xcopy %%a\*static*.zip %LOCAL%  /C /E /H /K /R /Y

for /r %cd% %%a in (.) do @if exist %%a\print-*.jar xcopy %%a\print-*.jar %LOCAL%  /C /E /H /K /R /Y
```

### SSH Publishers的配置
```
Source files: print-external-provider*.jar,print-external-site*.jar,print-destroy-provider*.jar,print-destroy-site*.jar
Remove prefix:
Remote directory: project
Exec command: /home/user/project/restart-all.sh


Source files: *static.zip
Remove prefix:
Remote directory: project
Exec command: /home/user/project/build.sh

```

> Source files还支持 `**/*/print-external-provider*.jar`查找当前`job`下的任何目录下包含`print-external-provider`的`jar`文件