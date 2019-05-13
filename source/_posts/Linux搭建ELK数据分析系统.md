---
title: Linux搭建ELK数据分析系统
date: 2018-05-25 23:32:22
tags:
---

> log4j-->logstash-->elasticsearch 然后kibana分析elasticsearch

## 一、ElasticSearch
### 下载 
[ https://www.elastic.co/downloads](https://www.elastic.co/downloads)

解压

### 安装head插件和nodejs(可以忽略)
[https://github.com/mobz/elasticsearch-head](https://github.com/mobz/elasticsearch-head)

1. 依赖
```shell
yum -y install gcc make gcc-c++ openssl-devel wget
```

2. 下载解压：
```shell
http://nodejs.cn/download/
xd -d node-v6.10.0-linux-x64.tar.xz
tar -vxf node-v6.10.0-linux-x64.tar
```

3. 环境变量
```shell
#node
PATH=$PATH:/tools/node-v6.10.0-linux-x64/bin
export PATH
```

4. 安装grunt
```shell
npm install -g grunt-cli
```

5. 启动head
```shell
Running with built in server
git clone git://github.com/mobz/elasticsearch-head.git
cd elasticsearch-head
npm install
grunt server
open http://localhost:9100/
```

### 安装插件x-pack
```shell
bin/elasticsearch-plugin install x-pack
```
> 替代Hand及Marvel,安装后，访问es有密码（默认：elastic  changeme）

### ES配置文件
```shell
vi config/elasticsearch.yml
```
修改
```shell
cluster.name: es1
node.name: node-1
path.data: /tools/tmp/es/data
path.logs: /tools/tmp/es/logs
network.host: 192.168.1.117 #不配ip不能用ip访问
network.port: 9200
```

### 启动
```shell
adduser elasticsearch
chown -R elasticsearch elasticsearch目录
chmod -R 777 elasticsearch目录
su elasticsearch
./bin/elasticsearch #必须不能是root用户
```

## 二、Logstash
### 下载 
[https://www.elastic.co/downloads](https://www.elastic.co/downloads)

### 配置
```shell
touch config/log4j_to_es.conf
```
加入
```shell
# For detail structure of this file
# Set: https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html
input {
  # For detail config for log4j as input,
  # See: https://www.elastic.co/guide/en/logstash/current/plugins-inputs-log4j.html
  log4j {
    mode => "server"
    host => "192.168.1.117"  #localhost
    port => "4567"
    type => "log4j"
  }
  file {
     path => "/tools/tmp/logstash/log/t.log"
     type => "syslog"
  }
}
filter {
    #Only matched data are send to output.
    if [priority]=="DEBUG" { #filter debug log
        drop{}
    }

}
output {
  # For detail config for elasticsearch as output,
  # See: https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html
  elasticsearch {
    action => "index"                #The operation on ES
    hosts  => "192.168.1.117:9200"   #ElasticSearch host, can be array.
    index  => ".applog"              #The index to write data to.
    user   => "elastic"
    password => "changeme"
  }
  file {
     path           => "/tools/tmp/logstash/log/all.log"
     codec          => line { format => "custom format: %{message}"}
     flush_interval => 0
  }
}
```
### 启动
```shell
 /tools/logstash-5.2.1/config/log4j_to_es.conf -w 2 &
```

## 三、Kibana
### 下载 
[https://www.elastic.co/downloads](https://www.elastic.co/downloads)
安装插件x-pack
```shell
bin/elasticsearch-plugin install x-pack
```
> 安装后，默认用户/密码:elastic/changeme

修改配置
```shell
vi /tools/kibana-5.2.1-linux-x86_64/config/kibana.yml
```
直接加入或修改
```shell
server.port: 5601
server.host: "es1"
elasticsearch.url: http://es1:9200
kibana.index: ".kibana"
```
启动
```shell
/tools/kibana-5.2.1-linux-x86_64/bin/kibana &
```
## 四、Java

### 配置 log4j.properties
```shell
# console
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p %t %c : %m%n

# file
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.file=E:/JAVA/servicer/ELK/tmp/logstash/log/filelog.log
log4j.appender.file.maxFileSize=1024
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d %p %t %c : %m%n

# logstash
log4j.appender.logstash=org.apache.log4j.net.SocketAppender
log4j.appender.logstash.port=4567
log4j.appender.logstash.remoteHost=192.168.1.117

log4j.rootLogger=debug,stdout,file,logstash
```
> 注意：这里的端口号需要跟Logstash监听的端口号一致，这里是4567。


### java代码
```java
package com.demo.elk;

import org.apache.log4j.Logger;

public class Application {
    private static final Logger LOGGER = Logger.getLogger(Application.class);
    public static void main(String[] args) throws Exception {
		for (int i = 0; i < 10; i++) {
			LOGGER.info("UUID TO LOG [" + UUID.randomUUID() + i + "].");
		}
	}
}
```

### pom.xml
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.4.1.RELEASE</version>
		<relativePath />
	</parent>
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.ayou.spring</groupId>
	<artifactId>spring-cloud-elk</artifactId>
	<packaging>war</packaging>
	<version>1.0.0-SNAPSHOT</version>
	<name>spring-cloud-elk</name>
	<url>https://www.qtdu.com</url>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
			<exclusions>
				<exclusion>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-starter-logging</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-log4j</artifactId>
			<version>1.3.8.RELEASE</version>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Camden.SR2</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<finalName>spring-cloud-elk</finalName>
	</build>
</project>
```


## 五、查询
打开 http://192.168.1.117:5601
用户/密码（elastic/changeme）
登录后选择左边菜单Dev Tools

> 查询message包含UUID的数据

```shell
GET _search
{
  "query": {
     "match": {
        "message": "UUID"
    }
  }
}
```
更多基础查询语句[http://www.open-open.com/lib/view/open1455326349058.html](http://www.open-open.com/lib/view/open1455326349058.html)


## 六、ES错误处理

> elasticsearch 5.0 安装过程中遇到了一些问题，通过查找资料几乎都解决掉了，这里简单记录一下 ，供以后查阅参考，也希望可以帮助遇到同样问题的你。

### 问题一：警告提示

> [2016-11-06T16:27:21,712][WARN ][o.e.b.JNANatives ] unable to install syscall filter: 
java.lang.UnsupportedOperationException: seccomp unavailable: requires kernel 3.5+ with CONFIG_SECCOMP and CONFIG_SECCOMP_FILTER compiled in
at org.elasticsearch.bootstrap.Seccomp.linuxImpl(Seccomp.java:349) ~[elasticsearch-5.0.0.jar:5.0.0]
at org.elasticsearch.bootstrap.Seccomp.init(Seccomp.java:630) ~[elasticsearch-5.0.0.jar:5.0.0]

报了一大串错误，其实只是一个警告。
解决：使用心得linux版本，就不会出现此类问题了。

 

### 问题二：ERROR: bootstrap checks failed

> max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]
max number of threads [1024] for user [lishang] likely too low, increase to at least [2048]

解决：切换到root用户，编辑limits.conf 添加类似如下内容
```shell
vi /etc/security/limits.conf 
```
添加如下内容:
```shell
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```
 

### 问题三
> max number of threads [1024] for user [lish] likely too low, increase to at least [2048]

解决：切换到root用户，进入limits.d目录下修改配置文件。
```shell
vi /etc/security/limits.d/90-nproc.conf 
```
修改如下内容：
```shell
* soft nproc 1024
```
修改为
```shell
* soft nproc 2048
```
 

### 问题四
> max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]

解决：切换到root用户修改配置sysctl.conf
```shell
vi /etc/sysctl.conf 
```
添加下面配置：
```shell
vm.max_map_count=655360
```
并执行命令：
```shell
sysctl -p
```
然后，重新启动elasticsearch，即可启动成功。


> 报错 npm install phantomjs-prebuilt@2.1.14 --ignore-scripts
