---
title: CentOS安装zookeeper与集群
date: 2018-05-25 23:35:02
tags:
---
1.解压
```bash
#tar -zxvf zookeeper-3.4.8.tar.gz
```
2.复制
```bash
#cp -r zookeeper-3.4.8/* /usr/local/zookeeper #复制
#cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg #复制配置文件
```bash
3.修改配置文件
```bash
#vi /usr/local/zookeeper/conf/zoo.cfg #修改配置文件
```

如下
```shell
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/usr/local/zookeeper/data
# 错误日志的存放位置
dataLogDir=/usr/local/zookeeper/logs
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

#server.1=192.168.1.106:2888:3888
server.2=192.168.1.111:2888:3888
server.3=192.168.1.114:2888:3888
#server.3=103.51.145.246:2888:3888
```

创建文件并标识id，必须用id来标识不同的zookeeper，server.2中的2就是zookeeper中的myid下的id
```bash
#mkdir /usr/local/zookeeper/data #创建data文件
#mkdir /usr/local/zookeeper/data/myid #创建myid文件
#vi /usr/local/zookeeper/data/myid #内容为 1，对应server.1=127.0.0:2888:3888中的1，不同节点不同，server.2节点则myid为2
#mkdir /usr/local/zookeeper/logs #创建日志文件
```

4.配置JDK和ZooKeeper环境变量
```bash
#vi /etc/profile
```
加入
```shell

#jdk
JAVA_HOME=/usr/local/jdk1.8.0_92
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH
#zookeeper
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
export PATH
```

5.启动和链接
```bash
/usr/local/zookeeper/bin/zkServer.sh start #start启动 restart重启 stop停止
/usr/local/zookeeper/bin/zkCli.sh #链接 链接上了就OK了
```

6.其他节点一模一样的配置，注意myid和 有x台就x个server.x `本站就用这样的方式在跑dubbo`
