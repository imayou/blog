---
title: CentOS7挂载硬盘（多分区）
date: 2018-05-25 23:30:43
tags:
---
###分区工具fdisk用法介绍
fdisk命令参数介绍
p、打印分区表。
n、新建一个新分区。
d、删除一个分区。
q、退出不保存。
w、把分区写进分区表，保存并退出。
看一眼硬盘使用情况
```bash
# df -h
Filesystem Size Used Avail Use% Mounted on
/dev/vda1 7.8G 1.1G 6.3G 15% /
devtmpfs 1.9G 0 1.9G 0% /dev
tmpfs 1.9G 24K 1.9G 1% /dev/shm
tmpfs 1.9G 8.3M 1.9G 1% /run
tmpfs 1.9G 0 1.9G 0% /sys/fs/cgroup
```bash

看一下各个硬盘
```bash
# fdisk -l
Disk /dev/vda: 8589 MB, 8589934592 bytes, 16777216 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000c6178
Device Boot Start End Blocks Id System
/dev/vda1 * 2048 16777215 8387584 83 Linux
Disk /dev/vdb: 107.4 GB, 107374182400 bytes, 26214400 sectors
Units = sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk /dev/vdc: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```bash
由于腾讯云应该用的是跨盘lvm所以我看不出未分区，所以判断上面我红色标注的为未分区(系统盘8G数据盘100G)

###开始分区
```bash
# fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xd2faa0a2.
```bash
输入`p`打印分区表
```bash
Disk /dev/vdb: 107.4 GB, 107374182400 bytes, 26214400 sectors
Units = sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0xd2faa0a2
Device Boot Start End Blocks Id System
```bash
按`n`键新建一个分区
```bash
Partition type:
p primary (0 primary, 0 extended, 4 free)
e extended
Select (default p):
```bash
出现两个菜单e表示扩展分区，p表示主分区

按`p`建一个主分区
然后出现
```bash
Partition number (1-4, default 1):
```bash
按`1`表示第一个主分区
```bash
First sector (256-26214399, default 256):
```bash
选择扇区大小，直接回车
再次提示
```bash
Last sector, +sectors or +size{K,M,G} (256-26214399, default 26214399):
```bash
输入5240000将近20G（如果只打算分一个区，那么这里直接回车默认就可以，然后按w保存分区表，如果是还要分区就先不要按w保存）
提示
```bash
Partition 1 of type Linux and of size 20 GiB is set
```bash
表示第一个主分区已经创建好了有20G

接下来再加一个分区
输入`n`
输入`p`
输入`2`
然后一直回车就好了，如果是需要分3个分区，那么计算一下在Last sector输入就行了

再看一下分区表输入`p`
```bash
Disk /dev/vdb: 107.4 GB, 107374182400 bytes, 26214400 sectors
Units = sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0x6f1ec174
Device Boot Start End Blocks Id System
/dev/vdb1 256 5240000 20958980 83 Linux
/dev/vdb2 5240064 26214399 83897344 83 Linux
```bash
后面两行我用红色标注的就是刚刚分好的vdb1和vdb2两个分区
最后的关键就是输入w保存分区！

###接下来就是格式化
```bash
#mkfs.ext4 /dev/vdb1（ext4和ext3的区别是ext3最大约等于4T，而ext4使用了48位的数据块索引空间，在大文件的时候，磁盘io效率以及查找数据块效率上都有很大的提高）
```bash
回车/dev/vdb1就格式化好了
###格式化第二块
```bash
#mkfs.ext4 /dev/vdb2回车
```bash
两块都分好区格式化好了
再来一步就可以使用了

###挂载分区
```bash
# cd /（我习惯把分区挂载到根目录下的目录）
# mkdir tools创建一个目录tools用来放我的工具，比如lnmp包呀jdk呀tomcat的呀…等等
# mkdir ayou再建一个目录用来放网站各种…名称随便，要多骚有多骚…
# mount /dev/vdb1 /tools/将刚刚在根目录创建的tools目录挂载到/dev/vdb1分区
# mount /dev/vdb1 /ayou/将刚刚在根目录创建的ayou目录挂载到/dev/vdb2分区
```bash
在看一眼磁盘
```bash
# df -h
Filesystem Size Used Avail Use% Mounted on
/dev/vda1 7.8G 1.2G 6.3G 15% /
devtmpfs 1.9G 0 1.9G 0% /dev
tmpfs 1.9G 24K 1.9G 1% /dev/shm
tmpfs 1.9G 8.3M 1.9G 1% /run
tmpfs 1.9G 0 1.9G 0% /sys/fs/cgroup
/dev/vdb1 20G 45M 19G 1% /tools
/dev/vdb1 79G 57M 75G 1% /ayou
```bash
tools有20G，ayou有79G将近80G行了，证明分区挂载都好了

但是你重启就不见了，因为linux有一种叫自动挂载…
好吧我觉得很贱，那就自动挂载一下吧，
首先第一种方法直接echo进/etc/fstab文件
```bash
# echo '/dev/vdb1 /tools ext4 defaults 0 0' >> /etc/fstab（自动挂载/dev/vdb1到tools）
# echo '/dev/vdb2 /ayou ext4 defaults 0 0' >> /etc/fstab（自动挂载/dev/vdb2到ayou）
```bash
第二种方法直接修改/etc/fstab文件
```bash
# vi /etc/fstab 
```bash
然后加入下面这两句就行了，结果都一样都是要把这两句加到/etc/fstab文件里
```bash
/dev/vdb1 /tools ext4 defaults 0 0
/dev/vdb2 /ayou ext4 defaults 0 0
```bash