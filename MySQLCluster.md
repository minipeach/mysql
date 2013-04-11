##简介

MySQL集群是一种在无共享架构系统里应用内存数据库集群的技术。这种无共享的架构可以使得系统使用非常便宜的并且是最小配置的硬件。

MySQL集群是一种分布式设计，目标是要达到没有任何单点故障点。因此，任何组成部分都应该拥有自己的内存和磁盘。任何共享存储方案如网络共享，网络文件系统和SAN设备是不推荐或不支持的。通过这种冗余设计，MySQL声称数据的可用度可以达到99.999%。

实际上，MySQL集群是把一个叫做NDB的内存集群存储引擎集成与标准的MySQL服务器集成。它包含一组计算机，每个都跑一个或者多个进程，这可能包括一个MySQL服务器，一个数据节点，一个管理服务器和一个专有的一个数据访问程序。它们之间的关系如下图所示：

<img width="589" height="378" alt="" src="http://i56.tinypic.com/296gn4k.jpg">

##存储引擎

MySQL Cluster 使用了一个专用的基于内存的存储引擎，这样做的好处是速度快， 没有磁盘I/O的瓶颈，但是由于是基于内存的，所以数据库的规模受系统总内存的限制， 如果运行NDB的MySQL服务器一定要内存够大，比如4G, 8G, 甚至16G。NDB引擎是分布式的，它可以配置在多台服务器上来实现数据的可靠性和扩展性，理论上 通过配置2台NDB的存储节点就能实现整个数据库集群的冗余性和解决单点故障问题。

该存储引擎有下列弊端：

- 基于内存，数据库的规模受集群总内存的大小限制
- 基于内存，断电后数据可能会有数据丢失，这点还需要通过测试验证。
- 多个节点通过网络实现通讯和数据同步、查询等操作，因此整体性受网络速度影响，
- 因此速度也比较慢

当然也有它的优点：

- 多个节点之间可以分布在不同的地理位置，因此也是一个实现分布式数据库的方案。
- 扩展性很好，增加节点即可实现数据库集群的扩展。
- 冗余性很好，多个节点上都有完整的数据库数据，因此任何一个节点宕机都不会造成服务中断。
- 实现高可用性的成本比较低，不象传统的高可用方案一样需要共享的存储设备和专用的软件才能实现，NDB 只要有足够的内存就能实现。


##体系结构

MySQL Cluster 由3个不同功能的服务构成，每个服务由一个专用的守护进程提供，一项 服务也叫做一个节点，下面来介绍每个节点的功能。

The management (MGM) node

管理节点，用来实现整个集群的管理，理论上一般只启动一个，而且宕机也不影响 cluster 的服务，这个进程只在cluster 启动以及节点加入集群时起作用， 所以这个节点不是很需要冗余，理论上通过一台服务器提供服务就可以了。

通过 ndb_mgmd 命令启动，使用 config.ini 配置文件

The storage or database (DB) node:

数据库节点，用来存储数据，可以和管理节点(MGM) , 用户端节点(API) 可以处在 不同的机器上，也可以在同一个机器上面，集群中至少要有一个DB节点，2个以上 时就能实现集群的高可用保证，DB节点增加时，集群的处理速度会变慢。

通过 ndbd 命令启动，第一次创建好cluster DB 节点时，需要使用 –init参数初始化。

例如： bin/ndbd –ndb-connectstring=ndb_mgmd.mysqlcluster.net –initial

The client (API) node:

客户端节点，通过他实现 cluster DB 的访问，这个节点也就是普通的 mysqld 进程， 需要在配置文件中配置ndbcluster 指令打开 NDB Cluster storage engine 存储引擎，增加 API 节点会提高整个集群的并发访问速度和整体的吞吐量，该节点 可以部署在Web应用服务器上，也可以部署在专用的服务器上，也开以和DB部署在 同一台服务器上。

通过 mysqld_safe 命令启动，

这3类节点可以分布在不同的主机上，比如 DB 可以是多台专用的服务器，也可以 每个DB都有一个API，当然也可以把API分布在Web前端的服务器上去，通常来说， API越多cluster的性能会越好。


##Mysql集群探索与实践

1.准备好3台机器，从官网下载最新的mysql集群版本，此处用到mysql-cluster-gpl-7.1.5.tar.gz源码包， 配置并安装，记得加上

–with-plugins=innobase,ndbcluster （innobase可选）

3台机器分别是192.168.207.153，192.168.208.3，192.168.208.9，具体分配如下

管理节点(ndb_mgmd):192.168.207.153

数据节点(ndbd): 192.168.208.3

数据节点(ndbd): 192.168.208.9

SQL节点(mysqld): 192.168.208.3

SQL节点(mysqld): 192.168.208.9

2.在mysql目录下新建mysql-cluster文件夹，切换到mysql-cluster，新建config.ini

```mysql
[NDBD DEFAULT]
NoOfReplicas=2       #备份，副本，这样的话2台数据节点的数据就会同步
DataMemory=200M
IndexMemory=100M
[TCP DEFAULT]
portnumber=2202
[NDB_MGMD]   #管理节点
id=1
hostname=192.168.207.153
datadir=/home/taozi/mysql/mysql-cluster
[NDBD]    #数据节点
id=2
hostname=192.168.208.3
datadir=/home/taozi/mysql/data
[NDBD]   #数据节点
id=3
hostname=192.168.208.9
datadir=/home/taozi/mysql/data
[MySQLD]   #sql节点
id=4
hostname=192.168.208.3
[MySQLD]    #sql节点
id=5
hostname=192.168.208.9
[MySQLD]     #sql节点
id=6
```

3.在管理节点服务器上启动管理节点服务 （如果不存在ndb_mgmd那么要从libexec下面复制过来）

```mysql
~/mysql/bin/ndb_mgmd -f ~/mysql/mysql-cluster/config.ini
```

4.进入2台数据节点服务器，分别启动数据节点服务

```mysql
~/mysql/bin/ndbd     （第一次启动使用  ~/mysql/bin/ndbd --initial）
```

5.最后分别进入sql节点服务器，修改my.cnf，加入

```mysql
[MYSQL_CLUSTER]
ndb-connectstring=192.168.207.153
[MYSQLD]
ndbcluster
ndb-connectstring=192.168.207.153
```

启动mysql服务

```mysql
/home/taozi/mysql/bin/mysqld_safe --ledir=/home/taozi/mysql/bin /
--log-error=/home/taozi/mysql/data/t.err --datadir=/home/taozi/mysql/data /
--socket=/home/taozi/mysql/tmp/mysql.sock --pid-file=/home/taozi/mysql/data/mysqld.pid &
```

6.此时回到管理节点

~/mysql/bin/ndb_mgm -e show

可以看到显示如下

```mysql
[taozi@search153 mysql]$ ./show.sh 
Connected to Management Server at: localhost:1186
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @192.168.208.3  (mysql-5.1.47 ndb-7.1.5, Nodegroup: 0, Master)
id=3    @192.168.208.9  (mysql-5.1.47 ndb-7.1.5, Nodegroup: 0)
[ndb_mgmd(MGM)] 1 node(s)
id=1    @192.168.207.153  (mysql-5.1.47 ndb-7.1.5)
[mysqld(API)]   3 node(s)
id=4    @192.168.208.3  (mysql-5.1.47 ndb-7.1.5)
id=5    @192.168.208.9  (mysql-5.1.47 ndb-7.1.5)
id=6 (not connected, accepting connect from any host)
```

7.进入sql节点，在test数据库创建表

```mysql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`)
) ENGINE=ndbcluster  DEFAULT CHARSET=gbk
```

切换到2台数据节点服务器~/mysql/data/ndb_2_fs和~/mysql/data/ndb_3_fs看看，
或者直接去数据库查，数据已经同步了！

8.关闭集群服务

关闭sql节点等同于停止mysql服务，此时外界数据不将再进来。然后关闭管理节点

```mysql
~/mysql/bin/ndb_mgm -e shutdown
rm ~/mysql/mysql-cluster/ndb_1_config.bin.1 #不是必须的，如果config.ini有改动则要加上
```

这样操作后，管理节点和数据节点都将停止服务

Notes:

1:如果发现关闭一台机器的ndbd进程，另一台机器的ndbd的进程也关闭，则需要修改参数NoOfReplicas。

2:./ndbd --initial 不能同时在所有数据节点机器上执行，如执行，会删除所有数据

3:可以像操作非簇类型的数据库那样，操作mysqld节点

4:每次修改config.ini文件，重启ndb_mgmd时，需要删除mysql-cluster文件下的ndb_1_config.bin.1文件，
因为他默认调用此文件 

5:NDB 簇不支持自动发现数据库的功能，这点很重要,一旦在一个数据节点上创建了世界（world）数据库和它的表，
在簇中的每个SQL节点上还需要发出命令 CREATE DATABASE world,后跟FLUSH TABLES。这样，节点就能
识别数据库并读取其表定义。（在本版本MySQL Cluster 7.1.5下数据库也会自动同步的）

6:如果在相关节点服务器启动时,注意查看~/mysql/mysql-cluster目录内的相关日志文件以获取错误信息.

7:在管理节点的配置文件里各[mysqld],[ndbd]和[ndb_mgmd]配置的选项值顺序应该如下:

[mysqld]

Id=4

HostName=192.168.208.3

Id在顶端紧跟其后的是HostName,如果顺序错了,当SQL或数据节点连接管理节点时,管理节点无法正确的定位
到其对应的节点配置上.
因为无法定位到对应的节点配置,当没有剩余的[空节点]时,客户端节点启动时(./mysqld or ./ndbd)
还会报:

Configuration error: Error : Could not alloc node id at 192.168.0.231 port 1186: No free 
node id found for mysqld 
(API).Failed to initialize consumers 

8:[空节点]是没有指定HostName选项的节点配置均为空节点,空节点可以用来动态配置一些动态IP的节点,
一般管理节点的 配置文件要预留3个以上的空节点,因为备份时需要连接一个节点,如下:

[mysqld]

Id=6
