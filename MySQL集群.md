# MySQL集群

## 1 主从复制介绍

在实际生产中，数据的重要性不言而喻。

如果我们的数据库只有一台服务器，那么很容易产生单点故障的问题，比如这台服务器访问的压力过大而没有相应或者崩溃，那么这台服务器就不可用了，比如这台服务器的硬盘坏了，那么整个数据库的数据就全部丢失了，这是重大的安全事故。

为了避免服务的不可用以及提升数据的安全性和可靠性，我们至少需要部署两台或者两台以上服务器来存储数据库数据，也就是我们需要将数据复制多份部署在多台不同的服务器上，即使有一台服务器出现故障了，其他服务器依然可以继续提供服务。

MySQL提供了主从复制功能以提高服务的可用性和数据的安全性与可靠性。

主从复制是指服务器分为主服务器和从服务器，主服务器负责读和写，从服务器只负责读，主从复制也叫master/slave，master是主，slave是从，但是并没有强制，也就是说从也可以写，主也可以读，只不过一般我们不这么做。

主从复制可以实现对数据库的备份和读写分离



**主从的结构**

```markdown
1.一主多从
主数据库可以用来进行写入数据，所谓写数据包括：添加、删除、修改、建库建表等等的任何能够对数据库中的内容造成影响的操作；多从数据库服务器，可以用来进行数据的读操作，主要就是查询。
这种模式中，主库一旦崩溃，写数据就无法进行了，但是还可以进行数据的读取。

2.双主多从
主：3307 从：3309；主：3308 从：3310，他们相互会进行数据的同步。但是如果其中一个主从结构的主服务器坏掉了3307，那么其从服务器3309的数据也不可用，因为数据从另一个好的主服务器3308写入，无法同步到坏了的主服务器3307，也就无法将最新的数据同步到它的从库3309中。
```

![image-20210616211929009](/Users/liuxiangren/mysql-learning/master-slave.png)

**执行流程**

```markdown
1.从服务器轮询主服务器的二进制数据文件，如果这个数据有变动则发送请求，来获取主库中的更新的内容
2.向主库中写数据：包括添加、修改、删除、建库建表等等。
3.主库将写入的命令记录到二进制日志文件中并更新二进制日志文件的偏移量
4.如果从库轮询到主服务器的二进制数据文件，发现主库二进制文件偏移量与从服务器不一致，那么就启动IO线程，从主库中获取某个偏移量开始到二进制文件结束位置之间的所有数据
5.主库根据从库的请求，从偏移量位置来推送数据到从库中，然后从库接收到数据后，会更新从库记录的偏移量位置
6.从库获取完毕这些更新的数据之后，将这些数据写到中继日志中，然后唤醒SQL线程同时让当前IO线程挂起(休眠等待)
7.SQL线程根据中继日志的偏移量记录，读取中继日志中的命令
8.SQL线程拿到读取的命令之后，在本地的从数据库中进行回放，所谓回放就是在从库中执行这些更新的命令，回放完成之后，挂起当前的SQL线程(休眠等待)
```

如果对从库进行写入操作，那么将导致主从关系脱离，对主库进行读操作，那么对数据库性能会有影响

Docker搭建Mysql集群(主要参考)：https://blog.csdn.net/qq_45562491/article/details/115335200

Docker安装Mysql，并搭建一主一从复制集群，一主双从，双主双从集群：https://www.cnblogs.com/tyhj-zxp/p/12719121.html



### 1.1 Docker搭建MySQL集群

创建docker挂载卷

我默认是在/Users/username/docker_volume/mysql_cluster/目录下

```tex
master1/M1.cnf:
[mysqld]
#日志文件的名字
log-bin=master-a-bin
#日志文件的格式
binlog_format=ROW
#服务器的id(zk的集群),一定要是唯一的
server-id=1
# 在作为从数据库的时候，有写入操作也要更新二进制日志文件，和M2互为主备
log-slave-updates
 
m1slave1/M1S1.cnf:
[mysqld]
#服务器的id,一定要是唯一的
server-id=2
 
m1slave2/M1S2.cnf:
[mysqld]
#服务器的id,一定要是唯一的
server-id=3
 
master2/M2.cnf:
[mysqld]
#日志文件的名字
log-bin=master-a-bin
#日志文件的格式
binlog_format=ROW
#服务器的id,一定要是唯一的
server-id=4
#双主互相备份(表示从服务器可能是另外一台服务器的主服务器)
log-slave-updates=true
 
m2slave1/M2S1.cnf
[mysqld]
#服务器的id,一定要是唯一的
server-id=5
```



```shell
 docker run -id -p 3307:3306 --name=M1 -v /Users/liuxiangren/docker_volume/mysql_cluster/master1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
 
 docker run -id -p 3308:3306 --name=M1S1 -v /Users/liuxiangren/docker_volume/mysql_cluster/m1slave1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
 
 docker run -id -p 3309:3306 --name=M1S2 -v /Users/liuxiangren/docker_volume/mysql_cluster/m1slave2:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
 
 docker run -id -p 3310:3306 --name=M2 -v /Users/liuxiangren/docker_volume/mysql_cluster/master2:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7
 
 docker run -id -p 3311:3306 --name=M2S1 -v /Users/liuxiangren/docker_volume/mysql_cluster/m2slave1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7

```



设置主从复制

172.17.0.3  M1

172.17.0.2 M1S1

172.17.0.5 M1S2

172.17.0.4 M2

172.17.0.6 M2S1

```sql
M1： 
#添加完之后需要登录主服务器给从服务器授权(root ip地址 密码 按情况修改。)
 
grant replication slave on *.* to 'root'@'172.17.0.%' identified by '123456';
#刷新系统权限表
flush PRIVILEGES;
# 查看状态
show master status;
# 得到日志名和文件偏移量：File：master-a-bin.000003  Position：598， 用于后面从服务器的配置
 
 
M1S1 和 M1S2：
#登录从服务器，设置从服务器如何找到主服务器
change master to master_host='172.17.0.3',master_port=3307,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=598;
# 开启主从复制
start slave; 
 
 
 
M2(即是主服务器也是从服务器):
#添加完之后需要登录主服务器给从服务器授权
grant replication slave on *.* to 'root'@'172.17.0.%' identified by '123456';
#刷新系统权限表
flush PRIVILEGES;
 
# 设置从服务器如何找到主服务器
change master to master_host='172.17.0.3',master_port=3307,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=598;
# 开启主从复制
start slave; 
 
 
# 查看状态
show master status;
# 得到日志名和文件偏移量：File：master-a-bin.000002  Position：539， 用于后面从服务器的配置
 
M2S1：
登录从服务器，设置从服务器如何找到主服务器
change master to master_host='172.17.0.3',master_port=3310,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=598;
# 开启主从复制
start slave; 
```

### 1.2 Mysql集群网络问题



我他么服了哇，怎么看怎么一直Connecting，问题的[解决思路](https://blog.csdn.net/mbytes/article/details/86711508)

从数据库显示**Slave_IO_Running：Connecting； Slave_SQL_Running：Yes**的问题

```markdown
1.网络不通
2.账户密码错误
3.防火墙
4.mysql配置文件问题
5.连接服务器时语法
6.主服务器mysql权限
```

```shell
Slave_IO_Running: Connecting
Slave_SQL_Running: Yes
```

#### 1.2.1 方案一

怎么办呢？听人家说是因为防火墙没关闭，导致访问不到，那好吧，先关防火墙吧

```shell
systemctl status firewalld
bash: systemctl: command not found

service iptables stop
提示iptables：unrecognized service的错误。

yum install iptables  
bash: yum: command not found

apt-get install yum
E: Unable to locate package yum
```

最终

```shell
apt-get update
Reading package lists... Done

apt-get install yum

apt-get install iptables
1、查看系统是否安装防火墙
whereis iptables
iptables: /usr/sbin/iptables /sbin/iptables /usr/share/iptables #说明已经安装
2、查看防火墙的配置信息
iptables -L
iptables: Permission denied (you must be root).
#原因iptables必须是root用户执行
#iptables必须在容器的特权模式下执行
解决：
1.运行容器时添加privileged：docker run --privileged xxx
2.docker exec -it --privileged xxx
应该不需要了，具体看方案二吧
```

#### 1.2.2 方案二

使用docker自有网络

```shell
docker network create mysql_cluster
```

那么运行的时候使用

```shell
docker run -id -p 3307:3306 --name=M1 -v /Users/liuxiangren/docker_volume/mysql_cluster/master1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --network mysql_cluster mysql:5.7
 
 docker run -id -p 3308:3306 --name=M1S1 -v /Users/liuxiangren/docker_volume/mysql_cluster/m1slave1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --network mysql_cluster mysql:5.7
 
 docker run -id -p 3309:3306 --name=M1S2 -v /Users/liuxiangren/docker_volume/mysql_cluster/m1slave2:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --network mysql_cluster mysql:5.7
 
 docker run -id -p 3310:3306 --name=M2 -v /Users/liuxiangren/docker_volume/mysql_cluster/master2:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --network mysql_cluster mysql:5.7
 
 docker run -id -p 3311:3306 --name=M2S1 -v /Users/liuxiangren/docker_volume/mysql_cluster/m2slave1:/etc/mysql/conf.d  -e MYSQL_ROOT_PASSWORD=123456 --network mysql_cluster mysql:5.7
```

```sql
M1： 
#添加完之后需要登录主服务器给从服务器授权(root ip地址 密码 按情况修改。)
# % 表示啥都行
grant replication slave on *.* to 'root'@'%' identified by '123456';
#刷新系统权限表
flush PRIVILEGES;
# 查看状态
show master status;
# 得到日志名和文件偏移量：File：master-a-bin.000003  Position：589， 用于后面从服务器的配置
change master to master_host='192.168.43.116',master_port=3310,master_user='root',master_password='123456',master_log_file='master-a-bin.000005',master_log_pos=154;
 
 
 
M1S1 和 M1S2：
#登录从服务器，设置从服务器如何找到主服务器,master_host设置宿主机的IP
change master to master_host='192.168.43.116',master_port=3307,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=589;
# 开启主从复制
start slave; 
 
 
 
M2(即是主服务器也是从服务器):
#添加完之后需要登录主服务器给从服务器授权
grant replication slave on *.* to 'root'@'%' identified by '123456';
#刷新系统权限表
flush PRIVILEGES;
 
# 设置从服务器如何找到主服务器
change master to master_host='192.168.43.116',master_port=3307,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=589;
# 开启主从复制
start slave; 
 
 
# 查看状态
show master status;
# 得到日志名和文件偏移量：File：master-a-bin.000002  Position：539， 用于后面从服务器的配置
 
M2S1：
登录从服务器，设置从服务器如何找到主服务器
change master to master_host='192.168.43.116',master_port=3310,master_user='root',master_password='123456',master_log_file='master-a-bin.000003',master_log_pos=589;
# 开启主从复制
start slave; 

```

[Change Mater To](https://www.cnblogs.com/bdicaprio/articles/9877525.html) 

```markdown
发现slave还是connecting状态，想ping个看看
ping has no installation candidate
解决方法如下：
# apt-get update
# apt-get upgrade
# apt-get install <packagename>
这样就可以正常使用apt-get了～
#安装ping
apt-get install -y inetutils-ping

发现能ping的通
ping M1
PING M1 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: icmp_seq=0 ttl=64 time=0.127 ms
64 bytes from 172.20.0.2: icmp_seq=1 ttl=64 time=0.257 ms
64 bytes from 172.20.0.2: icmp_seq=2 ttl=64 time=0.148 ms
64 bytes from 172.20.0.2: icmp_seq=3 ttl=64 time=0.259 ms
64 bytes from 172.20.0.2: icmp_seq=4 ttl=64 time=0.287 ms

喔。。。 master_host要填的是宿主机的IP地址
得嘞，按照上面的配置写就运行好了，开心不啦？
```



### 1.3 集群主从复制验证

```shell
docker exec -it M1 bash

mysql -uroot -p123456

show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

#在M1上创建一个数据库试试

create database db03;

docker exec -it M2 bash
mysql -uroot -p123456
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| db03               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
同样的在M1S1,M1S2,M2S1上我都查过了，确实都有的这个db03这个库的没错的
```







## 2 分库分表

为什么要分库分表？

```markdown
磁盘IO瓶颈：热点数据太多，缓存放不下了，查询会直接打到数据库上了
网络IO瓶颈：请求的数据太多了，网络带宽不够
CPU瓶颈：单表数据量太大，扫描数据太多了，SQL效率低下，如果SQL在没有调优的情况下进行查询，很容易导致崩溃
```

怎么去解决呢？

```markdown
单机 --->  分布式集群
分库分表（ShardingSpehere、Mycat、DBLE...）
VS
NoSQL（postgresql、voltdb、tidb、oceanbase...）
```

### 2.1 分库分表方案

分库分表的方案大致有两种，一种是从分拆的维度来看：垂直拆分和水平拆分

```markdown
垂直拆分：简单来说就是按业务进行拆分，从业务的角度将数据分到不同的数据单元，比如一个商城系统，以往数据都是存放在一个数据库中的，现在按照他的业务不同把它分成”用户库“、”订单库“、”商品库“这样的一些数据库，这样就把以往存在一个数据库的数据分发到了不同的数据库。垂直拆分能够分担我们的磁盘IO以及网络IO的压力，和微服务的思想其实是有点像的，但是这种垂直拆分并不能够解决我们一个单表的数据量过大的一个问题，就是我们所要解决的根本问题。我一个应用中可能存在一个产品表，数据量太大了，进行了垂直拆分，可能只是将表移动到了某个业务库，但是数据量100w还是100w，所以就有另一个拆分维度，水平拆分。

水平拆分：拆分的思想就是把一个表，按照某一个规则拆分成多张表，以往我们的订单表，从单库中进行垂直拆分后，放到了Order业务库中，我们还可以对Order业务库中的订单表进行水平的拆分，比如就按照id%10的规则，分成10张表，拆分成多张表，你想想，加入你这张表之前有1000w张表，分到10张表的时候，每张表只有100w了，那是不是你查询的时候，查询的数据量是不是就会少很多？

还可以从分拆的结果上可以理解为，分库和分表。

我们将订单表中的订单数据，可以按照id来分，还可以按照日期来分
```

#### 2.1.1 什么时候要考虑分库分表

```markdown
分库分表势必会带来一些数据的复杂性，原本只需要管理一张表，一个库，现在要多个表多个库了，运维难度和业务难度肯定都会上升的，那他有这些代价会带来哪些好处呢？
那什么时候这个好处才会大于它带来的代价呢？关于什么时候考虑分库分表，我们最好最好的一个参考就是阿里给出一个开发手册，他可以上百度搜一下，阿里给出一个开发手册，里面列出了很多详细的一个建议，其中有一条当你的单表数据达到500万。
或者你的单表容量达到了2GB。达到这样一个标准的时候，你就开始要考虑分库分表了，而且这个规则是怎么算呢？一般来说需要预计你要去预估三年的业务量，三年的数据量，如果达到了这样一个级别就要考虑分库分表了，如果没有哒到尽量就不要分，因为分库分表固然他有好处，但是呢肯定也会带来一定的坏处，所以你想想吧，整个库和表都已经打散了，对不对？你就需要像我们微服务一样，由各种各样的组件的处理，会出现各种各样的分布式的问题。那了解这个呢，我们在看常用的这种分户表的策略有哪些？常用的分库分表策略，比如说我们这里有一个按ID取模的方式，对吧，这是一种最常用的方式。这种方式有什么好处呢？它的数分布的非常的均匀，那你这里你的原表里面如果是1000w条，那你这个数据分到每个表就固定，就是100万条。数据分布的非常均匀，但是有个缺点是不好扩容。因为现在如果我有十台机器，我现在十台机器的一个集群已经搭好了，表也分好了。当你再要加机器的时候，你的所有ID要这个取模的，这个数据跟着变对不对？所以整个数据的分配。这样的话，你扩展代价就会非常非常大，很难在去往往下面加其他的数据库了。

那有没有比较好的？至于扩展的这种策略呢，也有就是比如说按我的ID。同样以ID这个字段为例的话，我可以用ID，比如取一个范围对不对取一个范围，比如说ID。我配个小于1000。对吧，小于1000放一个库，然后呢，ID在2000到3000。这样一个范围，放一个库。我以后都按这样一个范围来分，这样的分的话，是不是就比较好地解决了扩容的问题。
以后新来一个库，我就这新来一个ID，当我的ID超过3000了之后。只要加一个库就可以了对不对？但是这个问题会引发你的数据分布会不够均匀，因为你的ID，你这种订单数据涉及到一些增删改查的操作之后，每个片儿里面的数据就会或多或少分布不均匀，这样对我的查询效率是不是达不到一个极限，同时还有一些我们业务上需要的一些策略，比如说有些场景，我们的ID呢，是比如说按地域来分的，对不对？按我的一些地市来分，那这事儿我可以按地市分片，有很多很多的规则都要需要去定义，要考虑的是两个方面，一个是你查询的效率，还一个是什么呢？你这个数据分布的均匀程度。因为只有分布均匀之后，你的整个查询才能够真正有效的去往各个片区分担。
```

#### 2.1.2 分库分表带来的问题

```markdown
一、主键避重问题
我需要这需要考虑最先需要考虑的这样两个问题，然后呢，我们还要理解一下，就是分库分表固然给我们带来了一些好处，你单表现在之前存不下，存不下一千万的数据，好，现在你分库分了十个步骤，你就存100万就可以存得下来，对不对？虽然看似很美好，但是要注意一下，一旦你选择分库分表，他就会带来一些固有的问题。会有哪些问题呢？一个主键避重的问题，我们之前在单机数据库当中，单机数据库当中，我的主键是不是可以有这些索引啊，主键策略进行管理对不对？但是一旦到了分布式之后，数据库就没法帮你做这个主键的补充了。这个时候需要你自己保证ID全局唯一，并且呢，能够像我们MySQL常用的这种自增一样的，能够带一定的这种业务信息，最好是能够进行递增，这个时候你的主键策略就需要考虑的比较多了。
二、公共表处理
然后还有公共表的处理，我们业务表分成了十份，分到了多个库，对不对？那你的这个公共表，包括像一些字典表这样的表。也需要跟着变，他就不可能再去分了对不对，所以其实也就是你的这个分片逻辑，一旦你改了一个业务表，需要改的表是很多的，就包括像公共表这些东西
三、事务一致性
还有呢，比如说哎，事务一致性，以往的事务是在一个数据库里面，对不对数据库就有一个事务机制，可以帮我们解决这种数据的事务问题，一旦你进行分库分表了之后，你的一个事务是分到了多个库上去执行的，分到多个数据片儿上去执行的，这个时候就会有分布式事务的问题，而分布式事务这个，我们在学微服务的时候遇到的，这个是一个令人头疼的问题。
四、跨节点关联查询
五、跨节点分页、排序函数
我们比如说像跨节点进行分页、排序，还有像查这种聚合。以往都是什么？我直接在一个数据库里面查出一个数据集，我进行处理就完了对吧，现在你的数据分成了多个片儿，那你的这个结果也要跟着去进行一个整合才行啊，要有良好的归并的策略对不对？而且这个归并很容易带来很多很多的问题，是什么有可能有一些什么奇怪的问题呢，比如说我们的往往一个数据库，以往是一个SQL对吧，以往是一个SQL，现在分成三个片儿，之后三个片儿之后你查出来的，查出来东西都要放内存吧，有可能会有奇怪的情况，比如说什么呢，我从每一个片里面查出来的数据，都没有超过内存的限制，但是呢最后，你把这三个查询结果整合到一起的时候。内存崩了，就会造成一些非常非常奇怪的问题，你都需要去处理。好，那其实这些呢，我们可以看到分库分表其实并不是一个容易的事情，它并不是简单的，你把表分完就完了。还需要处理的问题相当多，而这东西如果我们自己做肯定会很麻烦，对不对？
```

## 3 ShardingSphere分库分表

ShardingSphere包括两个核心产品：Sharding JDBC和Sharding Proxy

```markdown
Sharding JDBC：客户端分库分表
可以说是对JDBC的扩展，是个jar包，对我们的数据库操作进行了一个扩展，帮助我们将业务逻辑代码进行分库分表，我在程序中写了一个SQL，他会帮我们把SQL命令分发到不同的Databases中

Sharding Proxy：服务端分库分表
Sharding Proxy会作为一个单独的服务进行部署，在所有的数据库上面相当于做一层代理，他自己就会伪装成一个Mysql服务，然后呢，我的应用可以像连Mysql一样的去连这个Sharding Proxy。那这样我们就不需要考虑分库分表的逻辑了，所有的SQL就在这个Sharding Proxy这个虚拟的Mysql当中去执行，而Sharding Proxy呢，会把我们把这种SQL转换成一个分库分表的SQL往下面底层的各个数据库去分发，这是它们两者的区别，我们就能够知道他们两个两者有什么优点，对吧。你像客户端的分库分表Sharding JDBC，他是在业务端来帮我们做分库分表的，那是不是就更为灵活呀？对不对我代码要做什么调整？我直接在我的APP里面就可以可以完成我的调整，对不对？想怎么改就怎么改，所以呢这个Sharding JDBC最大的优点就是灵活，Sharding JDBC支持所有的JDBC产品。你以前JDBC能够连什么产品，那Sharding JDBC就能够连，而Sharding Proxy这种方式呢，他是不是跟下面数据库打交道的，所以逻辑都需要提前定义好对吧，一旦提前定义好，那肯定就丧失灵活性，所以像Sharding Proxy，它默认就只能够支持MySQL和PGSQL这样两个产品，它只支持这两个产品，其他产品支持不了，因为SQL的情况太多了，对吧，你全部要先给你想好很难很难。这是他们两者的区别。

Registry Center：
我们之前学微服务的时候是不是经常提到一个注册中心啊？通过注册中心可以统一的管理我们的集群对不对？那这个Sharding JDBC和ShardingProxy，他们是做分库分表的分库分表的，这个注册中心那怎么去理解呢？这是后面会带大家去玩的一个问题，就是分库分表的注册中心到底怎么用。
```

### 3.1 小案例

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-tet</artifactId>
</dependency>
<dependency>
  <groupId>org.apache.shardingsphere</groupId>
  <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
  <version>4.1.1</version>
</dependency>
<dependency>
  <groupId>org.alibaba</groupId>
  <artifactId>durid</artifactId>
  <version>1.1.22</version>
</dependency>
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
  <groupId>org.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
  <version>3.0.5</version>
</dependency>
```

配置文件

mysql数据库中coursedb库中有两张表：course_1、course_2

```java
@Test
public void addCourse(){
  for(int i = 0;i < 10;i++){
    Course c = new Course();
    c.setCname("java");
    c.setUserId(1001L);
    c.setCstatus("1")
      courseMapper.insert(c);
  }
}
//course_1和course_2中各5条数据
```



```properties
#垂直分表策略
#配置真实数据源
spring.shardingsphere.datasource.names=m1

#配置第1个数据源
spring.shardingsphere.datasource.m1.type=com.alibaba.druid.pool.DruidDataSource
spring.shardingsphere.datasource.m1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.m1.url=jdbc:mysql://localhost:3306/coursedb?serverTimezone=GMT%2B8
spring.shardingsphere.datasource.m1.username=root
spring.shardingsphere.datasource.m1.password=root

#指定表的分布情况，配置在哪个数据库里，表名是什么，水平分表，分两个表：m1.course_1，m1.course_2
spring.shardingsphere.sharding.tables.course.actual-data-nodes=m1.course_$->{1..2}

#指定表的主键生成策略
spring.shardingsphere.sharding.tables.course.key-generator.column=cid
spring.shardingsphere.sharding.tables.course.key-generator.type=SNOWFLAKE
#雪花算法的一个可选参数
spring.shardingsphere.sharding.tables.course.key-generator.props.worker.id=1

#使用自定义的主键生成策略
#spring.shardingsphere.sharding.tables.course.key-generator.type=MYKEY
#spring.shardingsphere.sharding.tables.course.key-generator.props.mykey-offset=88

#指定分片策略，约定cid值为偶数添加到course_1表，如果是奇数添加到course_2表
#选定计算的字段
spring.shardingsphere.sharding.tables.course.table-strategy.inline.sharding-column=cid
#根据计算的字段算出对应的表名
spring.shardingsphere.sharding.tables.course.table-strategy.inline.algorithm-expression=course_$->{cid%2+1}

#打开sql日志输出
spring.shardingsphere.props.sql.show=true
#区分内存限制模式与连接限制模式
spring.shardingsphere.props.max.connections.size.per.query=1

spring.main.allow-bean-definition-overriding=true
```

查询

```java
@Test
public void queryCourse(){
  QueryWrapper<Course> weapper = new QueryWrapper<>();
  wrapper.in("cid",16251452L,16251455L...);
  List<Course> courses = courseMapper.selectList(wrapper);
  courses.forEach(course -> System.out.println(course));
}
//帮我们做操作去不同表中去查询，cid只有一个的时候就只会去一张表进行查询，很智能

//但是between就不行了
wrapper.between("cid",16251417L,16251455L);
//会报错
/**
java.lang.IllegalStateException:Inline strategy cannot support this type sharding:RangeRouteValue(colunmName=cid,tableName=course,valueRange=[16251417L,16251455L])
*/
//那这个怎么办呢？
//上面的27行
/**
spring.shardingsphere.sharding.tables.course.table-strategy.[inline|standard|complex|hint|none]这些不同的策略可以选择
standard:可以针对范围查询，确定值的带分片键的查询，去进行分片
complex：有可能分片键不止一个，可能是cid+userId，inline就只能支持一个，complex就支持多个了
hint：分片不是由你的sql来决定了，我另外自己指定一个分片的值，那比如，我已经分好了，course_1表中只存放偶数cid的数据，course_2中只存放奇数cid的数据，那么我就想只查course_1中的数据，那怎么办呢？我们就需要hint，将其作为一个条件，这样就能查指定表的数据了
*/
```





数据分片

```properties
spring.shardingsphere.datasource.names= #数据源名称，多数据源以逗号分隔
 
spring.shardingsphere.datasource.data-source-name.type= #数据库连接池类名称
spring.shardingsphere.datasource.data-source-name.driver-class-name= #数据库驱动类名
spring.shardingsphere.datasource.data-source-name.url= #数据库url连接
spring.shardingsphere.datasource.data-source-name.username= #数据库用户名
spring.shardingsphere.datasource.data-source-name.password= #数据库密码
spring.shardingsphere.datasource.data-source-name.xxx= #数据库连接池的其它属性
 
spring.shardingsphere.sharding.tables.logic-table-name.actual-data-nodes= #由数据源名 + 表名组成，以小数点分隔。多个表以逗号分隔，支持inline表达式。缺省表示使用已知数据源与逻辑表名称生成数据节点。用于广播表（即每个库中都需要一个同样的表用于关联查询，多为字典表）或只分库不分表且所有库的表结构完全一致的情况
 
#分库策略，缺省表示使用默认分库策略，以下的分片策略只能选其一
 
#用于单分片键的标准分片场景
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.standard.sharding-column= #分片列名称
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.standard.precise-algorithm-class-name= #精确分片算法类名称，用于=和IN。该类需实现PreciseShardingAlgorithm接口并提供无参数的构造器
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.standard.range-algorithm-class-name= #范围分片算法类名称，用于BETWEEN，可选。该类需实现RangeShardingAlgorithm接口并提供无参数的构造器
 
#用于多分片键的复合分片场景
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.complex.sharding-columns= #分片列名称，多个列以逗号分隔
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.complex.algorithm-class-name= #复合分片算法类名称。该类需实现ComplexKeysShardingAlgorithm接口并提供无参数的构造器
 
#行表达式分片策略
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.inline.sharding-column= #分片列名称
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.inline.algorithm-expression= #分片算法行表达式，需符合groovy语法
 
#Hint分片策略
spring.shardingsphere.sharding.tables.logic-table-name.database-strategy.hint.algorithm-class-name= #Hint分片算法类名称。该类需实现HintShardingAlgorithm接口并提供无参数的构造器
 
#分表策略，同分库策略
spring.shardingsphere.sharding.tables.logic-table-name.table-strategy.xxx= #省略
 
spring.shardingsphere.sharding.tables.logic-table-name.key-generator.column= #自增列名称，缺省表示不使用自增主键生成器
spring.shardingsphere.sharding.tables.logic-table-name.key-generator.type= #自增列值生成器类型，缺省表示使用默认自增列值生成器。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
spring.shardingsphere.sharding.tables.logic-table-name.key-generator.props.property-name= #属性配置, 注意：使用SNOWFLAKE算法，需要配置worker.id与max.tolerate.time.difference.milliseconds属性。若使用此算法生成值作分片值，建议配置max.vibration.offset属性
 
spring.shardingsphere.sharding.binding-tables[0]= #绑定表规则列表
spring.shardingsphere.sharding.binding-tables[1]= #绑定表规则列表
spring.shardingsphere.sharding.binding-tables[x]= #绑定表规则列表
 
spring.shardingsphere.sharding.broadcast-tables[0]= #广播表规则列表
spring.shardingsphere.sharding.broadcast-tables[1]= #广播表规则列表
spring.shardingsphere.sharding.broadcast-tables[x]= #广播表规则列表
 
spring.shardingsphere.sharding.default-data-source-name= #未配置分片规则的表将通过默认数据源定位
spring.shardingsphere.sharding.default-database-strategy.xxx= #默认数据库分片策略，同分库策略
spring.shardingsphere.sharding.default-table-strategy.xxx= #默认表分片策略，同分表策略
spring.shardingsphere.sharding.default-key-generator.type= #默认自增列值生成器类型，缺省将使用org.apache.shardingsphere.core.keygen.generator.impl.SnowflakeKeyGenerator。可使用用户自定义的列值生成器或选择内置类型：SNOWFLAKE/UUID/LEAF_SEGMENT
spring.shardingsphere.sharding.default-key-generator.props.property-name= #自增列值生成器属性配置, 比如SNOWFLAKE算法的worker.id与max.tolerate.time.difference.milliseconds
 
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.master-data-source-name= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[0]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[1]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[x]= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.load-balance-algorithm-class-name= #详见读写分离部分
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.load-balance-algorithm-type= #详见读写分离部分
 
spring.shardingsphere.props.sql.show= #是否开启SQL显示，默认值: false
spring.shardingsphere.props.executor.size= #工作线程数量，默认值: CPU核数
```

读写分离

```properties
#省略数据源配置，与数据分片一致
 
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.master-data-source-name= #主库数据源名称
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[0]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[1]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.slave-data-source-names[x]= #从库数据源名称列表
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.load-balance-algorithm-class-name= #从库负载均衡算法类名称。该类需实现MasterSlaveLoadBalanceAlgorithm接口且提供无参数构造器
spring.shardingsphere.sharding.master-slave-rules.master-slave-data-source-name.load-balance-algorithm-type= #从库负载均衡算法类型，可选值：ROUND_ROBIN，RANDOM。若`load-balance-algorithm-class-name`存在则忽略该配置
 
spring.shardingsphere.props.sql.show= #是否开启SQL显示，默认值: false
spring.shardingsphere.props.executor.size= #工作线程数量，默认值: CPU核数
spring.shardingsphere.props.check.table.metadata.enabled= #是否在启动时检查分表元数据一致性，默认值: false
```

数据脱敏

```properties
#省略数据源配置，与数据分片一致
 
spring.shardingsphere.encrypt.encryptors.encryptor-name.type= #加解密器类型，可自定义或选择内置类型：MD5/AES 
spring.shardingsphere.encrypt.encryptors.encryptor-name.props.property-name= #属性配置, 注意：使用AES加密器，需要配置AES加密器的KEY属性：aes.key.value
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.plainColumn= #存储明文的字段
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.cipherColumn= #存储密文的字段
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.assistedQueryColumn= #辅助查询字段，针对ShardingQueryAssistedEncryptor类型的加解密器进行辅助查询
spring.shardingsphere.encrypt.tables.<table-name>.columns.<logic-column-name>.encryptor= #加密器名字
```

治理

```properties
#省略数据源、数据分片、读写分离和数据脱敏配置
 
spring.shardingsphere.orchestration.name= #治理实例名称
spring.shardingsphere.orchestration.overwrite= #本地配置是否覆盖注册中心配置。如果可覆盖，每次启动都以本地配置为准
spring.shardingsphere.orchestration.registry.type= #配置中心类型。如：zookeeper
spring.shardingsphere.orchestration.registry.server-lists= #连接注册中心服务器的列表。包括IP地址和端口号。多个地址用逗号分隔。如: host1:2181,host2:2181
spring.shardingsphere.orchestration.registry.namespace= #注册中心的命名空间
spring.shardingsphere.orchestration.registry.digest= #连接注册中心的权限令牌。缺省为不需要权限验证
spring.shardingsphere.orchestration.registry.operation-timeout-milliseconds= #操作超时的毫秒数，默认500毫秒
spring.shardingsphere.orchestration.registry.max-retries= #连接失败后的最大重试次数，默认3次
spring.shardingsphere.orchestration.registry.retry-interval-milliseconds= #重试间隔毫秒数，默认500毫秒
spring.shardingsphere.orchestration.registry.time-to-live-seconds= #临时节点存活秒数，默认60秒
spring.shardingsphere.orchestration.registry.props= #配置中心其它属性
```



## 4 Mycat

如今随着互联网的发展，数据的量级也是成指数式的增长，从GB到TB到PB。对数据的各种操作也是愈加的困难，传统的关系性数据库已经无法满足快速查询与插入数据的需求，这个时候NoSQI的出现暂时解决了这一危机。它通过降低教据的安全性，减少对事务的支持，减少对复杂查询的支持，来获取性能上的提升，但是，在有些场合NOSQL一些折衷是无法满足使用场景的，就比如有些使用场景是绝对要有事务与安全指标的。这个时候NoSQL肯定是无法满足的，所以还是需要使用关系型数据库。如何使用关系型数据库解决海量存储的问题呢?此时就需要做数据库集群，为了提高查询性能将一个数据库的数据分散到不同的数据库中存储，为应对此问题就出现了-Mycat。

Mycat的目标是:低成本的将现有的单机教据库和应用平滑迁移到"云“端解决海量数据存储和业务规模迅速增长情况下的数据存储和访问的瓶颈问题。

**MyCat历史**

1). Mycat背后是阿里曾经开源的知名产品Cobar。 Cobar的核心功能和优势是MySQL数据库分片，此产品曾经广为流传，据说最 早的发起者对Mysql很精通，后来从阿里跳槽了，阿里随后开源的Cobar ,并维持到2013年年初，然后，就没有然后了。Cobar的 思路和实现路径的确不错。基于Java开发的，实现了MySOL公开的二进制传输协议，巧妙地将自己伪装成一个MySQL Server，目前市面上绝大多数MySQL客户端工具和应用都能兼容。比自己实现一个新的数据库协议要明智的多，因为生态环境在哪里摆着。

2). Mycat是基于cobar演变而来，相对于cobar来说，有两个显著优势:①.对cobar 的代码进行了 彻底的重构，Mycat在 I/o方面进行了重大改进，将原来的BIO改成了NIO,并发量有大幅提高; ②.增加了对Order By、Group By、limit等聚合功能的支持，同时兼容绝大多数数据库成为通用的数据库中间件

3).简单的说，Mycat就是:一个新颖的数据库中间件产品支持mysql集群，或者mariadb cluster，提供高可用性数据分片集群。你 可以像使用mysql一样使用Mycat。对于开发人员来说根本感觉不到mycat的存在。

```markdown
mariadb可以理解为Mysql的一个分支，驱动和操作基本一致
```

**Mycat优势**

MyCat是一个彻底开源的，面向企业应用数据库中间件， 支持事务，可以视为MySQL集群的企业级数据库，用来替代昂贵的oracle集群，在MyCat中融合内存缓存技术、NoSQL技术、 HDFS大数据的新型SQL server, 并结合传统数据库和新型分布式数据仓库的新 代企业级数据库中间件产品。

并具有优势:

1).性能可靠稳定

基于阿里开源的Cobar产品而研发，Cobar的稳定性、可靠性、优秀的架构和性能以及众多成熟的使用案例使得Mycat-开始就拥有一个很好的起点，站在巨人的肩膀上，我们能看到更远。业界优秀的开源项目和创新思路被泛融入到Mycat的基因中，使得Mycat在很多方面都领先于目前其他一些同类的开源项目，甚至超越某些商业产品。

2).强大的技术团队

MyCat现在由一支强大的技术团队维护，吸引和聚集了一 大批业内大数据和云计算方面的资深 工程师、架构师、DBA ，优秀的团队保障了MyCat的稳定高效运行。而且MyCat不依托于任何商业公司，而且得到大批开源爱好者的支持。

3).体系完善

MyCat已经形成了一系列的周边产品比较有名的是Mycat-web、Mycat-NIO、 Mycat-Balance等 ,已经形成了一个比较完整的解决方案，而不仅仅是一个中间件。

4).社区活跃

与MyCat数据库中间件类似的产品还有TDDL、Amoeba、Cobar。

①. TDDL ( Taobao Distributed Data Layer)不同于其它几款产品，并非独立的中间件，只能算作中间层,是以Jar包方式提供给应用调用属于JDBC Shard的思想

②. Amoeba是作为一个真正的独立中间件提供服务，应用去连接Amoeba操作MySQL集群，就像操作单个MySQL一样。Amoeba算中间件中的早期产品，后端还在使用JDBCDriver。

③. Cobar是在Amoeba基础上进化的版本，一个显著变化是把后端JDBCDriver改为原生的MySQL通信协议层。 ④. MyCat又是在cobar基础上发展的版本，性能优良，功能强大，社区活跃

 **MyCat使用场合**

要想用好MyCat，就需要了解其适用场景，以下几个场景适合适用MyCat。

1) .高可用性与MySQL读写分离

高可用:利用MyCat可以轻松实现热备份，当台服务器停机时，可以由集群中的另一台服务器自动接管业务，无需人工干预，从而保证高可用。

读写分离:通过MySQI数据库的binlog日志完成主从复制，并可以通过MyCat轻松实现读写分离,实现insert、update、delete走主库，而在select时走从库，从而缓解单台服务器的访问压力。

2) .业务数据分级存储保障

企业的数据量总是无休止的增长,这些数据的格式不一样，访问效率不一样，重要性也不样。可以针对不同级别的数据，采用不同的存储设备，通过分级存储管理软件实现数据客体在存储设备之间自动迁移及自动访问切换。

3).大表水平拆分，集群并行计算

数据切分是Mycat的核心功能，是指通过某种特定的条件，将存放在同一个数据库的数据，分散在存储在多个数据库中，以达到分散单台设备负载的效果。当数据库量超过800万行且需要做分片时，就可以考虑使用Mycat实现数据切分。

4).数据库路由器

Mycat基于Mysql实例的连接池复用机制，可以让每个应用最大程度共享一个Mysql实例的所有连接池，让数据库的并发访问能力大大提高。

5).整合多种数据源

当一个项目中使用了多个数据库(Oracle,Mysql,SQL Server,PostgreSQL)，并配置了多个数据源，操作起来就比较繁琐，这时就可以使用Mycat进行整合，最终我们的应用程序只需要访问一个数据源即可。



**Mycat下载**

下载地址：http://dl.mycat.org.cn



Mycat是一个开源数据库中间件，是一个实现了MySQL协议的数据库中间件服务器，我们可以把它看做是一个数据库代理，用Mysql客户端工具和命令行访问Mycat，而Mycat再使用MySQL原生Native协议与多个MySQL服务器通信，也可以用JDBC协议与大多数主流数据库服务器通信，包括SQL Server、Oracle、DB2、PostgreSQL等主流数据库，也支持MongoDB这种新型NoSQL方式的存储，未来还会支持更多类型的存储；

一般的，Mycat主要用于代理Mysql数据库，虽然它也支持去访问其他类型的数据库；

Mycat默认端口是8066，一般的，我们可以使用常见的对象映射框架比如mybatis操作Mycat

### 4.1 Mycat主要能做什么

#### 4.1.1 数据库的读写分离

通过Mycat可以自动实现写数据时操作主数据库，读数据时操作从数据库，这样能有效地减轻数据库的压力，也能减轻IO压力。

实现读写分离，当主出现故障后，Mycat自动切换到另一个主上，进而提供高可用的数据库服务，当然我需要部署多主多从的模式

```markdown
主从复制+读写分离
客户端通过Master对数据库进行写操作，slave端进行读操作，并可进行备份。Master出现问题后，可以手动将应用切换到slave端，手动切换引发很多问题，主从分离，数据不一致。
```

```markdown
如果有了Mycat，客户端直接连接Mycat，可以实现读写分离，如果出现问题，会自动切换到从服务器上，这样就避免了人工操作的意外情况
```

Docker Mycat参考文章：https://www.cnblogs.com/niunafei/p/12823753.html



### 4.2 Docker安装Mycat

安装Mycat：https://github.com/MyCATApache/Mycat-Server/wiki/2.1-docker

```shell
创建文件目录 /Users/[username]/docker_volume/mycat
ca mycat/

#去网上下载mycat-server
http://dl.mycat.org.cn/1.6.7.6/20201126013625/Mycat-server-1.6.7.6-release-20201126013625-linux.tar.gz
#然后将下载来的tar包放到 /Users/[username]/docker_volume/mycat 目录下

#将配置文件单独拎出来
mycat % tar -zxvf Mycat-server-1.6.7.6-release-20210303094759-linux.tar  -C /Users/[username]/docker_volume/ mycat/conf

#官方没有提供Mycat镜像，要自己下载，这个Dockerfile中的内容不用改
Dockerfile:
FROM openjdk:8-jdk-stretch

ADD http://dl.mycat.org.cn/1.6.7.6/20201126013625/Mycat-server-1.6.7.6-release-20201126013625-linux.tar.gz /usr/local
RUN cd /usr/local && tar -zxvf Mycat-server-1.6.7.6-release-20201126013625-linux.tar.gz && ls -lna

ENV MYCAT_HOME=/usr/local/mycat
WORKDIR /usr/local/mycat

ENV TZ Asia/Shanghai

EXPOSE 8066 9066

CMD ["/usr/local/mycat/bin/mycat", "console","&"]

#看看目录结构吧
mycat % ls
Dockerfile	conf	Mycat-server-1.6.7.6-release-20210303094759-linux.tar

#创建mycat镜像与容器
docker build -t mycat:1.6.7.6 .
各种下载中......

#查看下镜像呗
docker images
REPOSITORY                        TAG       IMAGE ID       CREATED              SIZE
mycat                             1.6.7.6   88de409a9cf8   About a minute ago   544MB
别说还挺大的


#运行容器并挂载配置文件目录与日志目录
#-v /root/data/mycat/conf:/usr/local/mycat/conf 挂载配置文件目录
#-v /root/data/mycat/logs:/usr/local/mycat/logs 挂载日志目录
# --network=adnc_net --ip 172.20.0.16  adnc_net是自建的bridge网络，如果使用docker默认网络，不需要这段
这个地址要改一改喔,
1.--network后面的mysql_cluster是我之前自己建的:docker network create mysql_cluster
2.--ip我是感觉不需要，但是人家指定了，我就不指定了

docker run --privileged=true -p 8066:8066 -p 9066:9066 --name mycat -v /Users/[username]/docker_volume/mycat/conf:/usr/local/mycat/conf -v /Users/[username]/docker_volume/mycat/logs:/usr/local/mycat/logs --network=mysql_cluster [--ip 192.168.43.116] -d mycat:1.6.7.6
```



```markdown
小知识:
创建容器时，指定IP地址
docker run -it --network test_network --ip 172.18.0.101 alpine:latest sh
知识参考地址：https://www.jianshu.com/p/51b1e444871b
```

#### 4.2.1 mycat的schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

  <!-- MCLUB是我自定义的逻辑库名称 -->
	<schema name="MCLUB" checkSQLschema="true" sqlMaxLimit="100">
		<table name="user" dataNode="dn1" primaryKey="id"/>
	</schema>
  <!-- db03是上面通过主从复制创建的库名称，还记得吗create database db03;? -->
	<dataNode name="dn1" dataHost="localhost" database="db03" />
<!-- 192.168.43.116是宿主机的IP地址 -->
	<dataHost name="localhost" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" 	
				dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.43.116:3307" user="root" password="123456">
			<readHost host="hostM1S1" url="192.168.43.116:3308" user="root" password="123456" />
			<readHost host="hostM1S2" url="192.168.43.116:3309" user="root" password="123456" />
		</writeHost>
		
		<writeHost host="hostM2" url="192.168.43.116:3310" user="root" password="123456">
			<readHost host="hostM2S1" url="192.168.43.116:3311" user="root" password="123456" />
		</writeHost>
	</dataHost>

</mycat:schema>
```

#### 4.2.2 mycat的server.xml

```xml
<!-- 就改一下里面的user，shcemas对应着咱么上面创建的逻辑表 -->
<user name="root" defaultAccount="true">
		<property name="password">123456</property>
		<property name="schemas">MCLUB</property>
		<property name="defaultSchema">MCLUB</property>
		<!--No MyCAT Database selected 错误前会尝试使用该schema作为schema，不设置则为null,报错 -->
		
		<!-- 表级 DML 权限设置 -->
		<!-- 		
		<privileges check="false">
			<schema name="TESTDB" dml="0110" >
				<table name="tb01" dml="0000"></table>
				<table name="tb02" dml="1111"></table>
			</schema>
		</privileges>		
		 -->
	</user>

	<user name="user">
		<property name="password">user</property>
		<property name="schemas">MCLUB</property>
		<property name="readOnly">true</property>
		<property name="defaultSchema">MCLUB</property>
	</user>
```

让咱们重启mycat的docker服务吧，看看会出现什么问题！

```shell
#启动mycat docker容器
docker exec -it mycat bash;

#docker restart
cd /usr/local/
./bin/mycat restart
Stopping Mycat-server...

ps -ef | grep mycat
#查看了logs，好像没什么问题，(*^▽^*)
```

#### 4.2.3 连接咱们搭建的mycat吧

```shell
#在宿主机上食用，得使用mysql命令去连接喔 
mysql -h 192.168.43.116 -P 8066 -uroot -p123456

#没错我就死活都登录不上去，你看看
然后我看人家说是mycat和mysql的版本问题，这就尼玛离谱，需要修改
参考以上内容修改server.xml中的标签
<property name="nonePasswordLogin">1</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
重新启动mycat无密码登录，访问成功

果然，激动人心的时刻到了，我果然已经连接上了
mysql -h 192.168.43.116 -P 8066 -uroot -p123456
mysql> show databases;
+----------+
| DATABASE |
+----------+
| MCLUB    |
+----------+
1 row in set (0.01 sec)
牛批了兄弟
```

Access deined问题解决，[参考文章](https://blog.csdn.net/qq_45311824/article/details/106016696)

#### 4.2.4 使用mycat创建表

```shell
#连接上了咱们的mycat之后
create table user(
    -> id int(11) not null auto_increment,
    -> name varchar(50) not null,
    -> sex varchar(1),
    -> primary key (id)
    -> )engine=innodb default charset=utf8;
Query OK, 0 rows affected (0.06 sec)
创建user表，但是怎么都查询不到
select * from user;
ERROR 1146 (HY000): Table 'db03.user' doesn't exist

然后连进M1 Docker container之后查看？表名是大写的USER？
mysql> select * from USER;
Empty set (0.01 sec)

设置字符集：https://www.cnblogs.com/miclesvic/p/10345235.html

mysql> desc USER;
+-------+-------------+------+-----+---------+----------------+
| Field | Type        | Null | Key | Default | Extra          |
+-------+-------------+------+-----+---------+----------------+
| ID    | int(11)     | NO   | PRI | NULL    | auto_increment |
| NAME  | varchar(50) | NO   |     | NULL    |                |
| SEX   | varchar(1)  | YES  |     | NULL    |                |
+-------+-------------+------+-----+---------+----------------+
3 rows in set (0.01 sec)

mysql> insert into USER(NAME,SEX) values('小刘',1);
Query OK, 1 row affected (0.03 sec)

mysql> select * from USER;
+----+--------+------+
| ID | NAME   | SEX  |
+----+--------+------+
|  1 | 小刘   | 1    |
+----+--------+------+
1 row in set (0.00 sec)

还算顺利
```

#### 4.2.5 查看日志信息

看看这个查询语句从哪个库里面查询出来的

```shell
#进入mycat的logs目录
#修改日志信息配置文件
vim /usr/local/conf/log4j2.xml
#修改
<asyncRoot level="debug" includeLocation="true"> <!-- 要修改这个level为debug -->
```







### 4.3 Mycat目录结构

```markdown
bin：可执行文件的
catlet：~
cconf：非常重要的配置文件目录
lib：依赖包 java开发的用到的依赖包
logs：日志信息目录
verstion.txt：~
```

### 4.4 Mycat核心概念

#### 4.4.1 分片

简单来说，就是指通过某种特定的条件，将我们存放在同一个数据库中的数据分散存储到多个数据库(主机)上面，以达到分散单台设备负载的效果。

数据的切分(Sharding)根据其切分规则的类型，可以分为两种切分模式。

1)、一种是按照不同的表(或者schema)来切分到不同的数据库(主机上)，这种切分可以称之为数据的垂直(纵向)切分

2)、另外一种则是根据表中的数据的逻辑关系，将同一个表中的数据按照某种条件拆分到多台数据库(主机)上面，这种切分称之为数据的水平(横向)切分。

**Mycat分片策略**

![image-20210624154727613](/Users/liuxiangren/mysql-learning/mycat-partion-strategy.png)

虚线以上是逻辑结构，虚线以下是物理结构； 

```markdown
文字描述，参照：https://zhuanlan.zhihu.com/p/347770372
第一层：逻辑库(Schema)
MyCat是一个数据库中间件，通常对实际应用来说，并不需要知道中间件的存在，业务开发人员只需要知道数据库的概念，所以数据库中间件可以被看做是一个或多个数据库集群构成的逻辑库。

第二层：逻辑表(table)
既然有逻辑库，那么就会有逻辑表，分布式数据库中，对应用来说，读写数据的表就是逻辑表。逻辑表，可以是数据切分后，分布在一个或多个分片库中，也可以不做数据切分，不分片，只有一个表构成。

1.分片表
是指那些原有的很大数据的表，需要切分到多个数据库的表，这样，每个分片都有一部分数据，所有分片构成了完整的数据。 总而言之就是需要进行分片的表。如 ：tb_order 表是一个分片表, 数据按照规则被切分到dn1、dn2两个节点。
2.非分片表
一个数据库中并不是所有的表都很大，某些表是可以不用进行切分的，非分片是相对分片表来说的，就是那些不需要进行数据切分的表。如： tb_city是非分片表 , 数据只存于其中的一个节点 dn1 上。
3.ER表
关系型数据库是基于 实体关系模型(Entity Relationship Model) 的, MyCat中的ER表便来源于此。 MyCat提出了基于ER关系的数据分片策略 , 字表的记录与其所关联的父表的记录存放在同一个数据分片中, 通过 表分组(Table Group) 保证数据关联查询不会跨库操作。
4.全局表
在一个大型的项目中,会存在一部分 字典表(码表) , 在其中存储的是项目中的一些基础的数据 , 而这些基础的数据 , 数据量都不大 , 在各个业务表中可能都存在关联 。当业务表由于数据量大而分片后 ， 业务表与附属的数据字典表之间的关联查询就变成了比较棘手的问题 ， 在MyCat中可以通过数据冗余来解决这类表的关联查询 ， 即所有分片都复制这一份数据（数据字典表），因此可以把这些冗余数据的表定义为全局表。

第三层：分片节点(dataNode)
数据切分后，一个大表被分到不同的分片数据库上面，每个表分片所在的数据库就是 分片节点（dataNode）。

第四层：节点主机(dataHost)
数据切分后，每个分片节点（dataNode）不一定都会独占一台机器，同一机器上面可以有多个分片数据库，这样一个或多个分片节点（dataNode） 所在的机器就是节点主机（dataHost）,为了规避单节点主机并发数限制，尽量将读写压力高的分片节点（dataNode）均衡的放在不同的节点主机（dataHost）。

第五层：分片规则(rule)
前面讲了数据切分，一个大表被分成若干个分片表，就需要一定的规则，这样按照某种业务规则把数据分到某个分片的规则就是分片规则，数据切分选择合适的分片规则非常重要，将极大的避免后续数据处理的难度。
```

#### 4.4.2 分片配置测试

抛出需求

```markdown
由于TB_TEST表中的数据量很大，现在需要对TB_TEST表进行数据分片，分为三个数据节点，每个节点主机位于不同的服务器上，具体的结构：
				MCLUB(逻辑库)
							↓
				TB_TEST(逻辑表)
	↓						↓							↓			
DataNode1		DataNode2			DataNode3
	↓						↓							↓
	DB1					DB2						DB3
xx.xx.xx.1	xx.xx.xx.2		xx.xx.xx.3
```

##### 4.4.2.1 环境准备

准备三台虚拟机，且安装好Mysql，并配置好

```markdown
IP地址列表：
	192.168.192.157
	192.168.192.158
	192.168.194.159
```

##### 4.4.2.2 配置schema.xml

schema.xml作为mycat中最重要的配置文件之一，管理着Mycat的逻辑库，逻辑表以及对应的分片规则、DataNode以及DataSource。弄懂这些配置，是正确使用MyCat的前提。这里就一层层对该文件进行解析。

| 属性     | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| schema   | 标签用于定义Mycat实例中的逻辑库                              |
| table    | 标签定义了Mycat中的逻辑表，rule用于指定分片规则，auto-sharding-long的分片规则是按ID值的范围进行分片1-5000000为第一片，5000000-1000000为第二片...具体设置后面会讲到 |
| dataNode | 标签定义了Mycat中的数据节点，也就是我们通常所说的数据分片    |
| dataHost | 标签在Mycat逻辑库中也是作为最底层的标签存在，直接定义了具体的数据库实例、读写奋力配置和新条语句 |

在服务器上创建3个数据库，名为db1

修改schema.xml如下

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
  <!-- 配置逻辑库的信息 -->
	<schema name="MCLUB" checkSQLschema="true" sqlMaxLimit="100">
    <!-- 在这个逻辑库中只有一张逻辑表的信息，对其进行分片分为dn1、dn2、dn3 -->
  	<table name="TB_TEST" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"></table>
  </schema>
	
  <!-- TB_TEST这张表会在localhost1、2、3这三个库里面的db1 -->
	<dataNode name="dn1" dataHost="localhost1" database="db1" />
	<dataNode name="dn2" dataHost="localhost2" database="db1" />
	<dataNode name="dn3" dataHost="localhost3" database="db1" />

	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
    <!-- 数据源，url我就写宿主机IP地址了 -->
		<writeHost host="hostM1" url="192.168.43.116:3306" user="root" password="123456" >
		</writeHost>
	</dataHost>
  
  	<dataHost name="localhost2" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
    <!-- 数据源，url我就写宿主机IP地址了 -->
		<writeHost host="hostM1" url="192.168.43.116:3307" user="root" password="123456" >
		</writeHost>
	</dataHost>
  
  	<dataHost name="localhost3" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
    <!-- 数据源，url我就写宿主机IP地址了 -->
		<writeHost host="hostM1" url="192.168.43.116:3308" user="root" password="123456" >
		</writeHost>
	</dataHost>
 
</mycat:schema>
```

##### 4.4.2.3 配置server.xml

Server.xml几乎保存了所有mycat需要的系统配置信息，最常用的是在此配置用户名、密码及权限。在system中添加UTF-8字符集设置，否则存储中文会出现问号

```xml
<property name="charset">utf8</property>
```

修改user的设置，我们这里为MCLUB设置了两个用户

```xml
<user name="root">
  <property name="password">123456</property>
  <property name="schemas">MCLUB</property>
</user>
<user name="test">
  <property name="password">123456</property>
  <property name="schemas">MCLUB</property>
</user>
```

并且需要将原来的逻辑库的配置，替换为MCLUB逻辑库；

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License"); 
	- you may not use this file except in compliance with the License. - You 
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
	- - Unless required by applicable law or agreed to in writing, software - 
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
	License for the specific language governing permissions and - limitations 
	under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
	<system>
    <!-- 这个docker里面的已经帮我们配置好了 -->
	<property name="charset">utf8mb4</property>
	<property name="nonePasswordLogin">0</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
	<property name="ignoreUnknownCommand">1</property><!-- 0遇上没有实现的报文(Unknown command:),就会报错、1为忽略该报文，返回ok报文。
	在某些mysql客户端存在客户端已经登录的时候还会继续发送登录报文,mycat会报错,该设置可以绕过这个错误-->
	<property name="useHandshakeV10">1</property>
    <property name="removeGraveAccent">1</property>
	<property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
	<property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->
	<property name="sqlExecuteTimeout">300</property>  <!-- SQL 执行超时 单位:秒-->
    
    <!--
		 sequenceHandlerType:0 -- 使用本地方式作为数据库自增: 需设置 sequence_conf.properties配置文件
				 SCHEMATABLE.HISIDS=
         SCHEMATABLE.MINID=1001
         SCHEMATABLE.MAXID=100000000 最大值
         SCHEMATABLE.CURID=1000 当前数据库的索引值
重启mycat

			sequenceHandlerType:1 -- 使用的是本地数据库的方式，需要在一个分节点创建表和存储过程,其中MYCAT_SEQUENCE必须为大写

			sequnceHandlerType:2 -- 自动生成64为的时间戳，故设计主键是，要考虑其长度
-->
		<property name="sequenceHandlerType">2</property>
	<!--<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
	INSERT INTO `travelrecord` (`id`,user_id) VALUES ('next value for MYCATSEQ_GLOBAL',"xxx");
	-->
	<!--必须带有MYCATSEQ_或者 mycatseq_进入序列匹配流程 注意MYCATSEQ_有空格的情况-->
	<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
	<property name="subqueryRelationshipCheck">false</property> <!-- 子查询中存在关联查询的情况下,检查关联字段中是否有分片字段 .默认 false -->
	<property name="sequenceHanlderClass">io.mycat.route.sequence.handler.HttpIncrSequenceHandler</property>
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
	<!-- <property name="processorBufferChunk">40960</property> -->
	<!-- 
	<property name="processors">1</property> 
	<property name="processorExecutor">32</property> 
	 -->
        <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena | type 2 NettyBufferPool -->
		<property name="processorBufferPoolType">0</property>
		<!--默认是65535 64K 用于sql解析时最大文本长度 -->
		<!--<property name="maxStringLiteralLength">65535</property>-->
		<!--<property name="sequenceHandlerType">0</property>-->
		<!--<property name="backSocketNoDelay">1</property>-->
		<!--<property name="frontSocketNoDelay">1</property>-->
		<!--<property name="processorExecutor">16</property>-->
		<!--
			<property name="serverPort">8066</property> <property name="managerPort">9066</property> 
			<property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
			<property name="dataNodeIdleCheckPeriod">300000</property> 5 * 60 * 1000L; //连接空闲检查
			<property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
		<!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
		<property name="handleDistributedTransactions">0</property>
		
			<!--
			off heap for merge/order/group/limit      1开启   0关闭
		-->
		<property name="useOffHeapForMerge">0</property>

		<!--
			单位为m
		-->
        <property name="memoryPageSize">64k</property>

		<!--
			单位为k
		-->
		<property name="spillsFileBufferSize">1k</property>

		<property name="useStreamOutput">0</property>

		<!--
			单位为m
		-->
		<property name="systemReserveMemorySize">384m</property>


		<!--是否采用zookeeper协调切换  -->
		<property name="useZKSwitch">false</property>

		<!-- XA Recovery Log日志路径 -->
		<!--<property name="XARecoveryLogBaseDir">./</property>-->

		<!-- XA Recovery Log日志名称 -->
		<!--<property name="XARecoveryLogBaseName">tmlog</property>-->
		<!--如果为 true的话 严格遵守隔离级别,不会在仅仅只有select语句的时候在事务中切换连接-->
		<property name="strictTxIsolation">false</property>
		<!--如果为0的话,涉及多个DataNode的catlet任务不会跨线程执行-->
		<property name="parallExecute">0</property>
	</system>
	
	<!-- 全局SQL防火墙设置 -->
	<!--白名单可以使用通配符%或着*-->
	<!--例如<host host="127.0.0.*" user="root"/>-->
	<!--例如<host host="127.0.*" user="root"/>-->
	<!--例如<host host="127.*" user="root"/>-->
	<!--例如<host host="1*7.*" user="root"/>-->
	<!--这些配置情况下对于127.0.0.1都能以root账户登录-->
	<!--
	<firewall>
	   <whitehost>
	      <host host="1*7.0.0.*" user="root"/>
	   </whitehost>
       <blacklist check="false">
       </blacklist>
	</firewall>
	-->

  <user name="root" defaultAccount="true"> 
    <property name="password">123456</property>  
    <property name="schemas">MCLUB</property> 
  </user> 
  
  <user name="user"> 
    <property name="password">123456</property>  
    <property name="schemas">MCLUB</property> 
    <property name=≈"readOnly">true</property>
  </user> 

</mycat:server>
```

- sequenceHandlerType:[参考文章](https://blog.csdn.net/qq_29652199/article/details/81558545)

##### 4.4.2.4 Mysql环境

```markdown
linux虚拟机的话，应该是要把防火墙关掉，或者开放端口的，挨个关闭
service iptables status;
service iptables stop;
然后，数据库挨个执行
create database db1;
```

##### 4.4.2.5 启动Mycat

```shell
cd /usr/local/mycat/
./bin/mycat start
```

##### 4.4.2.6 测试

连接测试Mycat是否正常运行，两种方式：
1.通过命令行

```shell
mysql -h 127.0.0.1 -p 8086 -u root -p
通过mysql指令进行mycat的测试，mycat伪装成了mysql，模拟了mysql的协议
执行连接mycat命令：
mysql -h 192.168.192.157[安装mycat的机器] -p 8066 -u root -p
连接上之后，你会看到和mysql没太大区别，mycat伪装的很棒

#查看所有的database
show databases;
+---------+
| DATABSE |
+---------+
| MCLUB	  |
+---------+
这个MCLUB就是我们在schema.xml中配置的

#进入MCLUB
use MCLUB;

#查看MCLUB中的所有表
show tables;
+-------------------+
| TABLES IN DATABSE |
+-------------------+
| tb_test	          |
+-------------------+
```

2.使用工具进行连接

使用IDEA的DataGrip或者Navicat，直接配置就行了

##### 4.4.2.7 定义表结构

我们之前就在schema中只配置了MCLUB的schema还有里面的逻辑表TB_TEST，我们在上面的mycat中在MCLUB中能看到tb_test的表，但是在三个dataHost中去查看表，是空的啥都没有的，只有一个空库db1。

我们进入mycat的客户端内，进行实体表的创建

```sql
CREATE TABLE TB_TEST(
id BIGINT(20) NOT NULL,
title VARCHAR(100) NOT NULL,
PRIMARY KEY (id),
) ENGINE=INNODB DEFAULT CHARSET=utf8;
```

执行完成之后，所有的数据节点中的db1库中都创建了这个表结构。

```sql
show tables;
desc TB_TEST；
```

后面我们只针对mycat进行操作，不会直接操作实体库。

```sql
#插入数据，通过mycat
INSERT INTO TB_TEST(ID,TITLE) values(1,'goods1');
INSERT INTO TB_TEST(ID,TITLE) values(2,'goods2');
INSERT INTO TB_TEST(ID,TITLE) values(3,'goods3');

#数据查询
select * from TB_TEST;
#我们在Mycat中操作的，都是针对逻辑库、逻辑表中的操作，而Mycat会对底层实体数据库进行数据的影响，那我们上面的几条数据，插入到哪里去了呢？
我们挨个实体数据库进行数据的查询：select * from TB_TEST;

只有157里面有，158、159都没有，那么他们什么时候才会插入数据呢？那他们都不加入数据，这个分片操作有啥意义呢？这都没起到扩容的任务。
这次我给插一条数据，id比较大的！
insert into tb_test(ID,TITLE) VALUES(5000001,'good5000001');
唉！这个再到datanode2里面去找，就能找到了

id再加到10000000万之后，就到了datanode3里面了
insert into tb_test(ID,TITLE) VALUES(10000001,'good10000001');

那我id再加500万呢？！
insert into tb_test(ID,TITLE) VALUES(15000001,'good15000001');
完蛋了，报错了 ERROR 1064(HY000) : can't find any valid datanode : TB_TEST -> ID -> 150000001

那这些信息在哪配置呢？我怎么就知道分片规则是什么呢？其实啊，都跟这个schema里面table标签的rule相关的
```

##### 4.4.2.8 分片规则

```xml
<table name="TB_TEST" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"></table>
这个rule其实是和/usr/local/mycat/conf/rule.xml是相关的
里面有一个
<table>
	<rule>
  		<columns>id</columns>
    	<algorithm>rang-long</algorithm>
  </rule>
</table>
这个表示，会根据这个id字段进行分片，rang-long其实引用的是同样在/usr/local/mycat/conf/rule.xml的一个分片函数
<function name="rang-lang" class="io.mycat.route.function.AutoPartitionByLong">
  <property name="mapFile">autopartition-long.txt</property>
</function>
这个又关联是/usr/local/mycat/conf/autopartition-long.txt
打开这个文件可以看到
# range start-end ,data node index
# K=1000,M=10000.
0-500M=0
500M-1000M=1
1000M-1500M=2
这一看，就明了了~

那问题来了，我的数据就是这张TB_TEST表啊，数据就是超过1500万了怎么办呢？很简单只要修改配置，并且增加对应mysql节点就行了
```

### 4.5 Mycat原理介绍

Mycat原理中最重要的一个动词就是"拦截"，它拦截了用户发送过来的SQL语句，首先对SQL语句做一些特定的分析，如分片分析、路由分析、读写分离分析、缓存分析等，然后将此SQL语句发往后端的真实数据库，并将返回的结果做适当的处理，最终再返回给用户，如图所示。

![image-20210624234341782](/Users/liuxiangren/mysql-learning/mycat_sql_process.png)

在图中，user表被分为三个分片节点，DN1、DN2、DN3，他们分布式的在三个MySQL Server(dataHost)上，因此可以使用1-N台服务器来分片，分片规则(sharding rule)为典型的字符串枚举分片规则，一个规则的定义是分片字段+分片函数。这里的分片字段为status，分片函数则为字符串枚举方式。

Mycat收到一条SQL语句时，首先解析SQL语句涉及到的表，接着查看此表的定义，如果该表存在分片规则，则获取SQL语句里分片字段的值，并匹配分片函数，得到该SQL语句对应的分片列表，然后将SQL发送到相应的分片去执行，最后处理所有的分片返回的数据并返回给客户端。以"select * from user where status='0'"为例，查找status='0'，按照分片函数，'0'值存放在DN1中，于是SQL语句被发送到第一个节点中执行，然后再将查询的结果返回给用户。

### 4.6 Mycat配置详解

#### 4.6.1 server.xml

```markdown
主要是：<system>、<user>和<firewall>
```

##### 4.6.1.1 **system**

| 属性                      | 取值       | 含义                                                         |
| ------------------------- | ---------- | ------------------------------------------------------------ |
| charset                   | utf8       | 设置Mycat的字符集，字符集需要与MySQL的字符集保持一致         |
| nonePasswordLogin         | 0,1        | 0为需要密码登录、1不需要密码登录，默认为0，设置为1则需要制定默认账户 |
| useHandshakeV10           | 0,1        | 使用该选项主要的目的是为了能够兼容高版本的JDBC驱动，是否采用HandshakeV10Packet来与client进行通信，1：是，0：否 |
| useSqlStat                | 0,1        | 开启SQL实时统计，1为开启，0为关闭；<br/>开启之后，Mycat会自动统计SQL语句的执行情况；<br/>mysql -h 127.0.0.1 -P 9066 -u root -p<br/>查看Mycat执行的SQL，执行效率比较低的SQL，SQL的整体执行情况、读写比例等；<br/>show @@sql; show @@sql.slow; show @@sql.sum; |
| useGolbleTableCheck       | 0,1        | 是否开启全局表的一致性检测。1为开启，0为关闭。               |
| sqlExecuteTimeout         | 1000       | SQL语句执行的超时时间，单位为s；                             |
| sequnceHandlerType        | 0,1,2      | 用来指定Mycat全局序列类型，0为本地文件，1为数据库方式，2为时间戳列方式，默认使用本地文件方式，文件方式主要用于测试 |
| sequnceHandlerPattern     | 正则表达式 | 必须带有MYCATSEQ或者MYCATSEQ进入序列匹配流程，注意MYCATSEQ_有空格的情况 |
| subqueryRelationshipCheck | true,false | 子查询中存在关联查询的情况下，检查关联字段中是否有分片字段。默认false |
| useCompression            | 0,1        | 开启mysql压缩协议，0：关闭,1：开启                           |
| fakeMySQLVersion          | 5.5,5.6    | 设置模拟的MySQL版本号                                        |
| defaultSqlParset          |            | 由于MyCat的最初版本使用了FoundationDB的SQL解析器，在MyCat1.3之后性价了Druid解析器，所以要设置defaultSqlParser属性来指定默认的解析器;解析器有两个：druidpaeser和fdbparser，在Mycat1.4之后，默认是druidparser，fdbparser已经废除了 |
| processors                | 1,2....    | 指定系统可用的线程数量，默认值为CPU核心x每个核心运行线程数量；processors会影响peocessorBufferPool，processorBufferLocalPercent，processorExecutor属性，所有，在性能调优时，可以适当的修改processors的值 |
| processorBufferChunk      |            | 指定每次分配Socket Direct Buffer默认值为4096字节，也会影响BufferPool长度，如果一次性获取字节过多而导致buffer不够用，则会出现警告，可以调大该值 |
| processorExecutor         |            | 指定NIOProcessor上共享businessExecutor固定线程池的大小；Mycat把异步任务交给businessExecutor线程池中，在新版本的Mycat中这个连接池使用频次不高，可以适当的把该值调小 |
| packetHeaderSize          |            | 指定MySQL协议中的报文头长度，默认4个字节                     |
| maxPacketSize             |            | 指定MySQL协议可以携带的数据最大大小，默认为16M               |
| idleTimeout               | 30         | 指定连接的空闲时间的超市长度；如果超时，将关闭资源回收，默认30分钟 |
| txIsolation               | 1,2,3,4    | 初始化前端连接的事务隔离级别，默认为REPEATED_READ，对应数字为3<br/>READ_UNCOMMITED=1;<br/>READ_COMMITED=2;<br/>REPEATED_READ=3;<br/>SERIALIZABLE=4; |
| sqlExecuteTimeout         | 300        | 执行SQL的超时时间，如果SQL语句执行超时，将关闭连接；默认300秒； |
| serverPort                | 8066       | 定义Mycat的使用端口，默认8066                                |
| managerPort               | 9066       | 定义Mycat的管理端口，默认9066                                |

**useSqlStat**

```shell
#执行命令
cd /usr/local/mycat
#执行命令,重新启动mycat
./bin/mycat restart
#执行命令，连接mycat，装有mycat的ip地址，默认端口号为8066
mysql -h 192.168.192.157 -P 8066 -u root -p123456
#进入逻辑库
show databases;
use MCLUB;
#查询TB_TEST所有数据
select * from TB_TEST;
#我们执行这几个操作之后，我们就可以通过管理功能进行查看，那Mycat默认的管理端口为9066
mysql -h 192.168.192.157 -P 9066 -u root -p123456
#进入咱们这个管理客户端后，执行统计命令的展示
show @@sql;
Empty set(0.00 sec)


修改/usr/lcoal/mycat/conf/system.xml中的useSqlStat为1
#执行命令
cd /usr/local/mycat
#执行命令,重新启动mycat
./bin/mycat restart
#执行命令，连接mycat，装有mycat的ip地址，默认端口号为8066
mysql -h 192.168.192.157 -P 8066 -u root -p123456
#进入逻辑库
show databases;
use MCLUB;
#查询TB_TEST所有数据
select * from TB_TEST;
查询出了数据....
select * from TB_TEST where id in (1,2,3);
查询出了数据....

#这个时候再连接管理端口
mysql -h 192.168.192.157 -P 9066 -u root -p123456
#进入咱们这个管理客户端后，执行统计命令的展示
show @@sql;
+----+------+----------------+---------------+-----------------------------------------+
| ID | USER |	START_TIME     | EXECUTE_TIME  | SQL                                     |
+----+------+----------------+---------------+-----------------------------------------+
|   1| root |  157854455255  |            26 | select * from TB_TEST where id in(1,2,3)|
+----+------+----------------+---------------+-----------------------------------------+
|   2| root |  157854455287  |            4  | select * from TB_TEST                   |
+----+------+----------------+---------------+-----------------------------------------+
```

##### 4.6.1.2 user

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">MCLUB</property>
    <property name="readOnly">true</property>
    <property name="benchmark">1000</property>
    <property name="usingDecrypt">0</property>
    
    <!-- 表级 DML 权限设置 -->
    <!-- 		
    <privileges check="false">
        <schema name="TESTDB" dml="0110" >
            <table name="tb01" dml="0000"></table>
            <table name="tb02" dml="1111"></table>
        </schema>
    </privileges>		
    -->
</user>
```

user标签主要用于定义登录MyCat的用户和权限 :

```markdown
1). <user name="root" defaultAccount="true"> : name 属性用于声明用户名 ;
2). <property name="password">123456</property> : 指定该用户名访问MyCat的密码 ;
3). <property name="schemas">MCLUB</property> : 能够访问的逻辑库, 多个的话, 使用 "," 分割
4). <property name="readOnly">true</property> : 是否只读
5). <property name="benchmark">11111</property> : 指定前端的整体连接数量 , 0 或不设置表示不限制 
6). <property name="usingDecrypt">0</property> : 是否对密码加密默认 0 否 , 1 是
```

**readOnly小实验**

```shell
修改server.xml中的user的root用户，将readOnly由false改为true
#以root进行mycat的登录
mysql -h 192.168.192.157 -P 8066 -u root -p
use MCLUB;
insert into TB_TEST(id,title) values('5','goods5');
ERROR 1495 (HY000):User readonly

#只读了，只能进行数据的读取
```

**usingDecrypt小实验**

```shell
修改server.xml的user的root用户将usingDecrypt由0修改为1
那直接重启
./bin/mycat restart

mysql -h 192.168.192.157 -P 8066 -u root -p
输入密码还是123456，就登不上去了，也就是说不能再使用明文进行登陆了

那怎么生成这个密文呢？
进入到/usr/local/mycat/lib目录下
java -cp Mycat-server-1.6.7.3-release.jar io.mycat.util.DecryptUtil 0:root:123456
0:root:123456
	- 0：代表mycat用户登录密码加密
	- root：用户名
	- 123456：用户名的密码
执行完以后，会得到一个密文，修改user中对应的password属性值为密文，就能登陆上了
```

**privileges小实验**

```markdown
<privileges check="false">

A. 对用户的 schema 及 下级的 table 进行精细化的 DML 权限控制; 

B. privileges 节点中的 check 属性是用 于标识是否开启 DML 权限检查， 默认 false 标识不检查，当然 privileges 节点不配置，等同 check=false, 由于 Mycat 一个用户的 schemas 属性可配置多个 schema ，所以 privileges 的下级节点 schema 节点同样 可配置多个，对多库多表进行细粒度的 DML 权限控制;

C. 权限修饰符四位数字(0000 - 1111)，对应的操作是 IUSD ( 增，改，查，删 )。同时配置了库跟表的权限，就近原则。以表权限为准。


<privileges check="false">
	<schema name="MCLUB" dm1="1111"> <!-- 1111:对库有删除权限 -->
			<table name="TB_TEST" dm1="1110"></table> <!--表中配置了，就近原则，对库没有删除的权限-->
	</schema>
</privileges>

重启
cd /usr/local/mycat
./bin/mycat restart

mysql -h 192.168.192.157 -P 8066 -uroot -p
use MCLUB
#执行增改查删
select * from TB_TEST; ✅
insert into TB_TEST(id,title) values(6,'goods6');✅
update TB_TEST set title = 'goods001' where ids = 1;✅
delete from TB_TEST where id = 6;
Error 3012 (HY000) : The statement DML privilege check is not passed,reject for user 'root'
```



##### 4.6.1.3 firewall

firewall标签用来定义防火墙；firewall下whitehost标签用来定义 IP白名单 ，blacklist用来定义 SQL黑名单。

```xml
<firewall>
    <!-- 白名单配置 -->
    <whitehost>
        <host user="root" host="127.0.0.1"></host>
    </whitehost>
    <!-- 黑名单配置 -->
    <blacklist check="true">
        <property name="selelctAllow">false</property>
    </blacklist>
</firewall>
```

黑名单拦截明细配置:

| 配置项                      | 缺省值 | 描述                                                         |
| --------------------------- | ------ | ------------------------------------------------------------ |
| selelctAllow                | true   | 是否允许执行 SELECT 语句                                     |
| selectAllColumnAllow        | true   | 是否允许执行 SELECT * FROM T 这样的语句。如果设置为 false，不允许执行 select * from t，但可以select * from (select id, name from t) a。这个选项是防御程序通过调用 select * 获得数据表的结构信息。 |
| selectIntoAllow             | true   | SELECT 查询中是否允许 INTO 字句                              |
| deleteAllow                 | true   | 是否允许执行 DELETE 语句                                     |
| updateAllow                 | true   | 是否允许执行 UPDATE 语句                                     |
| insertAllow                 | true   | 是否允许执行 INSERT 语句                                     |
| replaceAllow                | true   | 是否允许执行 REPLACE 语句                                    |
| mergeAllow                  | true   | 是否允许执行 MERGE 语句，这个只在 Oracle 中有用              |
| callAllow                   | true   | 是否允许通过 jdbc 的 call 语法调用存储过程                   |
| setAllow                    | true   | 是否允许使用 SET 语法                                        |
| truncateAllow               | true   | truncate 语句是危险，缺省打开，若需要自行关闭                |
| createTableAllow            | true   | 是否允许创建表                                               |
| alterTableAllow             | true   | 是否允许执行 Alter Table 语句                                |
| dropTableAllow              | true   | 是否允许修改表                                               |
| commentAllow                | false  | 是否允许语句中存在注释，Oracle 的用户不用担心，Wall 能够识别 hints和注释的区别 |
| noneBaseStatementAllow      | false  | 是否允许非以上基本语句的其他语句，缺省关闭，通过这个选项就能够屏蔽 DDL。 |
| multiStatementAllow         | false  | 是否允许一次执行多条语句，缺省关闭                           |
| useAllow                    | true   | 是否允许执行 mysql 的 use 语句，缺省打开                     |
| describeAllow               | true   | 是否允许执行 mysql 的 describe 语句，缺省打开                |
| showAllow                   | true   | 是否允许执行 mysql 的 show 语句，缺省打开                    |
| commitAllow                 | true   | 是否允许执行 commit 操作                                     |
| rollbackAllow               | true   | 是否允许执行 roll back 操作                                  |
| 拦截配置－永真条件          |        |                                                              |
| selectWhereAlwayTrueCheck   | true   | 检查 SELECT 语句的 WHERE 子句是否是一个永真条件              |
| selectHavingAlwayTrueCheck  | true   | 检查 SELECT 语句的 HAVING 子句是否是一个永真条件             |
| deleteWhereAlwayTrueCheck   | true   | 检查 DELETE 语句的 WHERE 子句是否是一个永真条件              |
| deleteWhereNoneCheck        | false  | 检查 DELETE 语句是否无 where 条件，这是有风险的，但不是 SQL 注入类型的风险 |
| updateWhereAlayTrueCheck    | true   | 检查 UPDATE 语句的 WHERE 子句是否是一个永真条件              |
| updateWhereNoneCheck        | false  | 检查 UPDATE 语句是否无 where 条件，这是有风险的，但不是SQL 注入类型的风险 |
| conditionAndAlwayTrueAllow  | false  | 检查查询条件(WHERE/HAVING 子句)中是否包含 AND 永真条件       |
| conditionAndAlwayFalseAllow | false  | 检查查询条件(WHERE/HAVING 子句)中是否包含 AND 永假条件       |
| conditionLikeTrueAllow      | true   | 检查查询条件(WHERE/HAVING 子句)中是否包含 LIKE 永真条件      |
| 其他拦截配置                |        |                                                              |
| selectIntoOutfileAllow      | false  | SELECT ... INTO OUTFILE 是否允许，这个是 mysql 注入攻击的常见手段，缺省是禁止的 |
| selectUnionCheck            | true   | 检测 SELECT UNION                                            |
| selectMinusCheck            | true   | 检测 SELECT MINUS                                            |
| selectExceptCheck           | true   | 检测 SELECT EXCEPT                                           |
| selectIntersectCheck        | true   | 检测 SELECT INTERSECT                                        |
| mustParameterized           | false  | 是否必须参数化，如果为 True，则不允许类似 WHERE ID = 1 这种不参数化的 SQL |
| strictSyntaxCheck           | true   | 是否进行严格的语法检测，Druid SQL Parser 在某些场景不能覆盖所有的SQL 语法，出现解析 SQL 出错，可以临时把这个选项设置为 false，同时把 SQL 反馈给 Druid 的开发者。 |
| conditionOpXorAllow         | false  | 查询条件中是否允许有 XOR 条件。XOR 不常用，很难判断永真或者永假，缺省不允许。 |
| conditionOpBitwseAllow      | true   | 查询条件中是否允许有"&"、"~"、"\|"、"^"运算符。              |
| conditionDoubleConstAllow   | false  | 查询条件中是否允许连续两个常量运算表达式                     |
| minusAllow                  | true   | 是否允许 SELECT * FROM A MINUS SELECT * FROM B 这样的语句    |
| intersectAllow              | true   | 是否允许 SELECT * FROM A INTERSECT SELECT * FROM B 这样的语句 |
| constArithmeticAllow        | true   | 拦截常量运算的条件，比如说 WHERE FID = 3 - 1，其中"3 - 1"是常量运算表达式。 |
| limitZeroAllow              | false  | 是否允许 limit 0 这样的语句                                  |
| 禁用对象检测配置            |        |                                                              |
| tableCheck                  | true   | 检测是否使用了禁用的表                                       |
| schemaCheck                 | true   | 检测是否使用了禁用的 Schema                                  |
| functionCheck               | true   | 检测是否使用了禁用的函数                                     |
| objectCheck                 | true   | 检测是否使用了“禁用对对象”                                   |
| variantCheck                | true   | 检测是否使用了“禁用的变量”                                   |
| readOnlyTables              | 空     | 指定的表只读，不能够在 SELECT INTO、DELETE、UPDATE、INSERT、MERGE 中作为"被修改表"出现 |

**firewall小实验**

```markdown
官方示例已经说得较为详细了
<!-- 全局SQL防火墙设置 -->
	<!--白名单可以使用通配符%或着*-->
	<!--例如<host host="127.0.0.*" user="root"/>-->
	<!--例如<host host="127.0.*" user="root"/>-->
	<!--例如<host host="127.*" user="root"/>-->
	<!--例如<host host="1*7.*" user="root"/>-->
	<!--这些配置情况下对于127.0.0.1都能以root账户登录-->
	<!--
	<firewall>
	   <whitehost>
	      <host host="1*7.0.0.*" user="root"/>
	   </whitehost>
       <blacklist check="false">
       </blacklist>
	</firewall>
	-->
<firewall>
	 <whitehost>
	    <host host="1*7.0.0.*" user="root"/>
  </whitehost>
  <blacklist check="true">
  	<property name="selectAllow">true</property>
  	<property name="deleteAllow">false</property>
  </blacklist>
</firewall>

重启mycat
cd /usr/local/mycat
./bin/mycat restart

mysql -h 192.168.192.157 -P 8066 -u root -p
ERROR 1045 (HY000) : Access denied for user 'root' with host '192.168.192.157'
登录不上去了，因为我们配置了root用户，只能以"host="1*7.0.0.*"的IP进行mycat的登录

那这样登录就行了：
mysql -h 127.0.0.1 -P 8066 -u root -p #本机登录就行了

那咱么执行个查询和删除看看呢？
use MCLUB;
select * from TB_TEST;✅

delete from TB_TEST where id = 1;❌
ERROR 3012 (HY000) : The statment is unsafe SQL, reject for user 'root'
就没法删除了
```



#### 4.6.2 shcema.xml

schema.xml 作为MyCat中最重要的配置文件之一 , 涵盖了MyCat的逻辑库 、 表 、 分片规则、分片节点及数据源的配置。

##### 4.6.2.1 schema 标签

```xml
<schema name="MCLUB" checkSQLschema="false" sqlMaxLimit="100">
	<table name="TB_TEST" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
</schema>
```

schema 标签用于定义 MyCat实例中的逻辑库 , 一个MyCat实例中, 可以有多个逻辑库 , 可以通过 schema 标签来划分不同的逻辑库。MyCat中的逻辑库的概念 ， 等同于MySQL中的database概念 , 需要操作某个逻辑库下的表时, 也需要切换逻辑库:

```
use MCLUB;
```

**属性：name**

指定逻辑库的库名 , 可以自己定义任何字符串 ;



**属性：checkSQLschema**

```markdown
取值为 true / false ;

如果设置为true时 , 如果我们执行的语句为 "select * from MCLUB.TB_TEST;" , 则MyCat会自动把schema字符去掉, 把SQL语句修改为 "select * from TB_TEST;" 可以避免SQL发送到后端数据库执行时, 报table不存在的异常 。

不过当我们在编写SQL语句时, 指定了一个不存在schema, MyCat是不会帮我们自动去除的 ,这个时候数据库就会报错, 所以在编写SQL语句时,最好不要加逻辑库的库名, 直接查询表即可。
```



**属性：sqlMaxLimit**

```markdown
当该属性设置为某个数值时,每次执行的SQL语句如果没有加上limit语句, MyCat也会自动在limit语句后面加上对应的数值 。也就是说， 如果设置了该值为100，则执行 select * from TB_TEST 与 select * from TB_TEST limit 100 是相同的效果 。

所以在正常的使用中, 建立设置该值 , 这样就可以避免每次有过多的数据返回。
```

```markdown
怎么能看到效果呢？
cd /usr/local/mycat/conf/log4j2.xml
修改里面的
<Loggers>将其中的<asyncRoot level="info">的级别改成debug，然后重启mycat
./bin/mycat restart

然后新开一个窗口连接到mycat服务器，到/usr/local/mycat/logs目录下执行，追加打印日志信息到控制台
tailf -f mycat.log

连接上mycat
mysql -h 192.168.192.157 -P 8066 -uroot -p
use MCLUB;
select * from TB_TEST;

然后我们再去查看日志 就能看到，mycat在我们的SQL都加上limit 100这个限制语句
```

##### 4.6.2.2 schama子标签table

table 标签定义了MyCat中逻辑库schema下的逻辑表 , 所有需要拆分的表都需要在table标签中定义 。

```xml
<table name="TB_TEST" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
```

属性如下 ： 

![image-20210625111428585](/Users/liuxiangren/mysql-learning/image/schema-table.png)



**属性name**

定义逻辑表的表名 , 在该逻辑库下必须唯一。



**属性dataNode**

定义的逻辑表所属的dataNode , 该属性需要与dataNode标签中的name属性的值对应。 如果一张表拆分的数据，存储在多个数据节点上，多个节点的名称使用","分隔 。

![image-20210625111553355](/Users/liuxiangren/mysql-learning/image/schema-datanode.png)



**属性rule**

该属性用于指定逻辑表的分片规则的名字, 规则的名字是在rule.xml文件中定义的, 必须与tableRule标签中name属性对应。

![image-20210625111807615](/Users/liuxiangren/mysql-learning/image/schema-rule.png)



**属性ruleRequired**

该属性用于指定表是否绑定分片规则, 如果配置为true, 但是没有具体的rule, 程序会报错。



**属性primaryKey**

逻辑表对应真实表的主键

如: 分片规则是使用主键进行分片, 使用主键进行查询时, 就会发送查询语句到配置的所有的datanode上; 如果使用该属性配置真实表的主键, 那么MyCat会缓存主键与具体datanode的信息, 再次使用主键查询就不会进行广播式查询了, 而是直接将SQL发送给具体的datanode。



**属性type**

该属性定义了逻辑表的类型，目前逻辑表只有全局表和普通表。

全局表：type的值是 global , 代表 全局表 。

普通表：无



**属性autoIncrement**

mysql对非自增长主键，使用last_insert_id() 是不会返回结果的，只会返回0。所以，只有定义了自增长主键的表，才可以用last_insert_id()返回主键值。
mycat提供了自增长主键功能，但是对应的mysql节点上数据表，没有auto_increment,那么在mycat层调用last_insert_id()也是不会返回结果的。

如果使用这个功能， 则最好配合数据库模式的全局序列。使用 autoIncrement="true" 指定该表使用自增长主键,这样MyCat才不会抛出 "分片键找不到" 的异常。 autoIncrement的默认值为 false。



**属性needAddLimit**

指定表是否需要自动在每个语句的后面加上limit限制, 默认为true。



##### 4.6.2.3 schema子标签dataNode

#### 

```xml
<dataNode name="dn1" dataHost="host1" database="db1" />
```

dataNode标签中定义了MyCat中的数据节点, 也就是我们通常说的数据分片。一个dataNode标签就是一个独立的数据分片。



具体的属性 ： 

| 属性     | 含义                 | 描述                                                         |
| -------- | -------------------- | ------------------------------------------------------------ |
| name     | 数据节点的名称       | 需要唯一 ; 在table标签中会引用这个名字, 标识表与分片的对应关系 |
| dataHost | 数据库实例主机名称   | 引用自 dataHost 标签中name属性                               |
| database | 定义分片所属的数据库 |                                                              |



##### 4.6.2.4 schema子标签dataHost

```xml
<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <writeHost host="hostM1" url="192.168.192.147:3306" user="root" password="itcast"></writeHost>
</dataHost>	
```

该标签在MyCat逻辑库中作为底层标签存在, 直接定义了具体的数据库实例、读写分离、心跳语句。

**相关属性**

| 属性       | 含义           | 描述                                                         |
| ---------- | -------------- | ------------------------------------------------------------ |
| name       | 数据节点名称   | 唯一标识， 供上层标签使用                                    |
| maxCon     | 最大连接数     | 内部的writeHost、readHost都会使用这个属性                    |
| minCon     | 最小连接数     | 内部的writeHost、readHost初始化连接池的大小                  |
| balance    | 负载均衡类型   | 取值0,1,2,3 ; 后面章节会详细介绍;                            |
| writeType  | 写操作分发方式 | 0 : 写操作都转发到第1台writeHost, writeHost1挂了, 会切换到writeHost2上; <br />1 : 所有的写操作都随机地发送到配置的writeHost上 ; |
| dbType     | 后端数据库类型 | mysql, mongodb , oracle                                      |
| dbDriver   | 数据库驱动     | 指定连接后端数据库的驱动,目前可选值有 native和JDBC。native执行的是二进制的MySQL协议，可以使用MySQL和MariaDB。其他类型数据库需要使用JDBC（需要在MyCat/lib目录下加入驱动jar） |
| switchType | 数据库切换策略 | 取值 -1,1,2,3 ; 后面章节会详细介绍;                          |

**子标签heartbeat**

配置MyCat与后端数据库的心跳，用于检测后端数据库的状态。heartbeat用于配置心跳检查语句。例如 ： MySQL中可以使用 select user(), Oracle中可以使用 select 1 from dual等。



**子标签writeHost、readHost**

指定后端数据库的相关配置， 用于实例化后端连接池。 writeHost指定写实例， readHost指定读实例。

在一个dataHost中可以定义多个writeHost和readHost。但是，如果writeHost指定的后端数据库宕机， 那么这个writeHost绑定的所有readHost也将不可用。



**子标签相关属性**

| 属性名       | 含义               | 取值                                                         |
| ------------ | ------------------ | ------------------------------------------------------------ |
| host         | 实例主机标识       | 对于writeHost一般使用 *M1；对于readHost，一般使用 *S1；      |
| url          | 后端数据库连接地址 | 如果是native，一般为 ip:port ; 如果是JDBC, 一般为jdbc:mysql://ip:port/ |
| user         | 数据库用户名       | root                                                         |
| password     | 数据库密码         | itcast                                                       |
| weight       | 权重               | 在readHost中作为读节点权重                                   |
| usingDecrypt | 密码加密           | 默认 0 否 , 1 是                                             |



#### 4.6.3 rule.xml

rule.xml中定义所有拆分表的规则, 在使用过程中可以灵活的使用分片算法, 或者对同一个分片算法使用不同的参数, 它让分片过程可配置化。



##### 4.6.3.1 标签tableRule

```xml
<tableRule name="auto-sharding-long">
    <rule>
        <columns>id</columns>
        <algorithm>rang-long</algorithm>
    </rule>
</tableRule>
```

A. name : 指定分片算法的名称

B. rule : 定义分片算法的具体内容 

C. columns : 指定对应的表中用于分片的列名

D. algorithm : 对应function中指定的算法名称



##### 4.6.3.2 Function标签

```xml
<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
</function>
```

A. name : 指定算法名称, 该文件中唯一 

B. class : 指定算法的具体类

C. property : 根据算法的要求执行 



#### 4.6.4 sequence配置文件

在分库分表的情况下 , 原有的自增主键已无法满足在集群中全局唯一的主键 ,因此, MyCat中提供了全局sequence来实现主键 , 并保证全局唯一。那么在MyCat的配置文件 sequence_conf.properties 中就配置的是序列的相关配置。

主要包含以下几种形式：

1). 本地文件方式

2). 数据库方式

3). 本地时间戳方式

4). 其他方式

5). 自增长主键

在/usr/local/mycat里面就有一些像是sequence_db_conf.properties、sequence_distributed_conf.properties、sequence_time_conf.properties这些配置文件，配置主键自增机制的



## 5. MyCat分片

### 5.1 垂直拆分

一个数据库中存储这整个应用系统的所有业务逻辑库表，那会出现性能的瓶颈，所以我们要按业务逻辑，将不同的子系统拆分到不同的数据库中。

#### 5.1.1 概述

![1573622314361](/Users/liuxiangren/mysql-learning/image/v_split.png) 

一种是按照不同的表（或者Schema）来切分到不同的数据库（主机）之上，这种切分可以称之为数据的垂直（纵向）切分。



#### 4.1.2 案例场景

![1575725341210](/Users/liuxiangren/mysql-learning/image/v_split_case_arch.png) 

在业务系统中, 有以下表结构 ,但是由于用户与订单每天都会产生大量的数据, 单台服务器的数据存储及处理能力是有限的, 可以对数据库表进行拆分, 原有的数据库表: 

![1575726526012](/Users/liuxiangren/mysql-learning/image/v_split_case_db.png) 



#### 5.1.3 准备工作

1). 准备三台数据库实例

```
192.168.192.157
192.168.192.158
192.168.192.159
```



2). 在三台数据库实例中建库建表

将准备好的三个SQL脚本, 分别导入到三台MySQL实例中 ;(实体数据库中，并不是Mycat)

![1575730157557](/Users/liuxiangren/mysql-learning/image/v_split_import_data.png) 

登录MySQL数据库之后, 使用source命令导入 ;



#### 5.1.4 schema.xml的配置

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	
  <!-- 配置一个逻辑库，库名叫做ITCAST_DB -->
	<schema name="ITCAST_DB" checkSQLschema="false" sqlMaxLimit="100">
		<table name="tb_areas_city" dataNode="dn1" primaryKey="id" />
		<table name="tb_areas_provinces" dataNode="dn1" primaryKey="id" />
		<table name="tb_areas_region" dataNode="dn1" primaryKey="id" />
		<table name="tb_user" dataNode="dn1" primaryKey="id" />
		<table name="tb_user_address" dataNode="dn1" primaryKey="id" />
		
		<table name="tb_goods_base" dataNode="dn2" primaryKey="id" />
		<table name="tb_goods_desc" dataNode="dn2" primaryKey="goods_id" />
		<table name="tb_goods_item_cat" dataNode="dn2" primaryKey="id" />
		
		<table name="tb_order_item" dataNode="dn3" primaryKey="id" />
		<table name="tb_order_master" dataNode="dn3" primaryKey="order_id" />
		<table name="tb_order_pay_log" dataNode="dn3" primaryKey="out_trade_no" />
	</schema>
	
	
	<dataNode name="dn1" dataHost="host1" database="user_db" />
	<dataNode name="dn2" dataHost="host2" database="goods_db" />
	<dataNode name="dn3" dataHost="host3" database="order_db" />
	
    
	<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.157:3306" user="root" password="123456"></writeHost>
	</dataHost>	
    
    <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM2" url="192.168.192.158:3306" user="root" password="123456"></writeHost>
	</dataHost>	
    
    <dataHost name="host3" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM3" url="192.168.192.159:3306" user="root" password="123456"></writeHost>
	</dataHost>	
    
</mycat:schema>
```



#### 5.1.5 server.xml的配置

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">ITCAST_DB</property>
    <property name="readOnly">false</property>
    <property name="benchmark">1000</property>
    <property name="usingDecrypt">1</property>
</user>

<user name="test">
    <property name="password">123456</property>
    <property name="schemas">ITCAST_DB</property>
</user>

<user name="user">
    <property name="password">123456</property>
    <property name="schemas">ITCAST_DB</property>
    <property name="readOnly">true</property>
</user>
```



#### 5.1.6 测试

准备，配置文件修改后，重启mycat

```shell
cd /usr/local/mycat
./bin/mycat

mysql -h 127.0.0.1 -P 8066 -u root -p

show databases;
+-----------+
| DATABASE  |
+-----------+
| ITCAST_DB |
+-----------+
#那这个DATABSE是不是就是我们在schema.xml中配置的逻辑库名啊

use ITCAST_DB;

#看看逻辑表，啧啧啧，都能看得到嘛
show tables;
+----------------------+
| Tables in ITCAST_DB  |
+----------------------+
| tb_areas_city        |
+----------------------+
| tb_areas_provinces   |
+----------------------+
| tb_areas_region      |
+----------------------+
| tb_goods_base        |
+----------------------+
| tb_goods_desc        |
+----------------------+
| tb_goods_item_cat    |
+----------------------+
| tb_goods_item        |
+----------------------+
| tb_order_item        |
+----------------------+
| tb_order_master      |
+----------------------+
| tb_order_pay_log     |
+----------------------+
| tb_user              |
+----------------------+
| tb_user_address      |
+----------------------+
```



1). 查询数据

```
select * from tb_goods_base;		#158上
select * from tb_user;					#157上
select * from tb_order_master;	#159上
都能查的到的，没问题的
```



2). 插入数据

```sql
insert  into tb_user_address(id,user_id,province_id,city_id,town_id,mobile,address,contact,is_default,notes,create_date,alias) values (null,'java00001',NULL,NULL,NULL,'13900112222','钟楼','张三','0',NULL,NULL,NULL)
插入成功
```



```sql
insert  into tb_order_item(id,item_id,goods_id,order_id,title,price,num,total_fee,pic_path,seller_id) values (100,19,149187842867954,3,'3G 6','1.00',5,'5.00',NULL,'qiandu')
插入成功
```



3). 测试跨分片的查询

```sql
SELECT order_id , payment ,receiver, province , city , area FROM tb_order_master o , tb_areas_provinces p , tb_areas_city c , tb_areas_region r
WHERE o.receiver_province = p.provinceid AND o.receiver_city = c.cityid AND o.receiver_region = r.areaid ;
ERROR 1064 (HY000): invalid route in sql, multi tablses found but datanode has no intersection sql:SELECT order_id.....
```

当运行上述的SQL语句时, MyCat会报错, 原因是因为当前SQL语句涉及到跨域的join操作 ;



#### 5.1.7 全局表配置

全局表，数据量不大，在很多库中需要进行关联查询的表

1). 将数据节点user_db中的关联的字典表 tb_areas_provinces , tb_areas_city , tb_areas_region中的数据备份 ;

```sql
直接在存有这三张字典表的linux命令行执行这三条语句就行了
mysqldump -uroot -pitcast user_db tb_areas_provinces  > provinces;
mysqldump -uroot -pitcast user_db tb_areas_city > city;
mysqldump -uroot -pitcast user_db tb_areas_region > region;

province city region都存放在当前执行命令的目录下
```



2). 将备份的表结构及数据信息, 远程同步到其他两个数据节点的数据库中;

```sql
scp远程拷贝：
scp city root@192.168.192.158:/root
scp city root@192.168.192.159:/root

scp provinces root@192.168.192.158:/root
scp provinces root@192.168.192.159:/root

scp region root@192.168.192.158:/root
scp region root@192.168.192.159:/root
```



3). 导入到对应的数据库中

```
mysql -uroot -p goods_db < city
mysql -uroot -p goods_db < provinces 
mysql -uroot -p goods_db < region 
```

或者

```markdown
先登录158、159的mysql，然后导入
source /root/provinces
source /root/city
source /root/region
```



4). MyCat逻辑表中的配置

```xml
<table name="tb_areas_city" dataNode="dn1,dn2,dn3" primaryKey="id" type="global"/>
<table name="tb_areas_provinces" dataNode="dn1,dn2,dn3" primaryKey="id"  type="global"/>
<table name="tb_areas_region" dataNode="dn1,dn2,dn3" primaryKey="id"  type="global"/>
```



5). 重启MyCat

```
bin/mycat restart
```



6). 测试

再次执行相同的连接查询 , 是可以正常查询出对应的数据的 ;

```
SELECT order_id , payment ,receiver, province , city , area FROM tb_order_master o , tb_areas_provinces p , tb_areas_city c , tb_areas_region r
WHERE o.receiver_province = p.provinceid AND o.receiver_city = c.cityid AND o.receiver_region = r.areaid ;
```



当我们对Mycat全局表进行增删改的操作时, 其他节点主机上的后端MySQL数据库中的数据时会同步变化的;

```
update tb_areas_city set city = '石家庄' where id = 5;
```



### 5.2 水平拆分

#### 5.2.1 概述

根据表中的数据的逻辑关系，将同一个表中的数据按照某种条件拆分到多台数据库（主机）上面，这种切分称之为数据的水平（横向）切分。

![1577151764698](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/1577151764698.png) 



#### 5.2.2 案例场景

![1577152000136](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/1577152000136.png) 

在业务系统中, 有一张表(日志表), 业务系统每天都会产生大量的日志数据 , 单台服务器的数据存储及处理能力是有限的, 可以对数据库表进行拆分, 原有的数据库表拆分成以下表 : 

![1577152168606](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/1577152168606.png) 



#### 5.2.3 准备工作

1). 准备三台数据库实例

```
192.168.192.157
192.168.192.158
192.168.192.159
```

2). 在三台数据库实例中创建数据库

```sql
#分别登录157、158、159三台服务器连接实体数据库进行database的创建
create database log_db DEFAULT CHARACTER SET utf8mb4;
```



#### 5.2.4 schema.xml的配置

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	
	<schema name="LOG_DB" checkSQLschema="false" sqlMaxLimit="100">
    <!-- mod-long取模分片算法，对应的rule.xml可以看得到，默认的就分三个库，所以rule.xml不用改 -->
		<table name="tb_log" dataNode="dn1,dn2,dn3" primaryKey="id"  rule="mod-long" />
	</schema>
	
	<dataNode name="dn1" dataHost="host1" database="log_db" />
	<dataNode name="dn2" dataHost="host2" database="log_db" />
	<dataNode name="dn3" dataHost="host3" database="log_db" />
	
    
	<dataHost name="host1" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.157:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
    
    <dataHost name="host2" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM2" url="192.168.192.158:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
    
    <dataHost name="host3" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM3" url="192.168.192.159:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
    
</mycat:schema>
```



#### 5.2.5 server.xml的配置

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">LOG_DB</property>
</user>

<user name="test">
    <property name="password">123456</property>
    <property name="schemas">LOG_DB</property>
</user>

<user name="user">
    <property name="password">123456</property>
    <property name="schemas">LOG_DB</property>
    <property name="readOnly">true</property>
</user>
```



#### 5.2.6 测试

```shell
配置完成了重启mycat
cd /usr/local/mycat/
./bin/mycat restart

mysql -h 127.0.0.1 -P 8066 -u root -p

#查看逻辑库
show databases;
+-----------+
| DATABASE  |
+-----------+
| LOG_DB    |
+-----------+

use LOG_DB;
#查看逻辑表
show tables;
+-------------------+
| Tables in LOG_DB  |
+-------------------+
| tb_log            |
+-------------------+

那咱们的底层实体数据库中，有没有这张库呢？
use LOG_DB;
show tables;
三台服务器上都是Empty~
```



1). 在MyCat数据库中执行建表语句，注意是在mycat中执行创建表的语句

```sql
CREATE TABLE `tb_log` (
  `id` bigint(20) NOT NULL COMMENT 'ID',
  `model_name` varchar(200) DEFAULT NULL COMMENT '模块名',
  `model_value` varchar(200) DEFAULT NULL COMMENT '模块值',
  `return_value` varchar(200) DEFAULT NULL COMMENT '返回值',
  `return_class` varchar(200) DEFAULT NULL COMMENT '返回值类型',
  `operate_user` varchar(20) DEFAULT NULL COMMENT '操作用户',
  `operate_time` varchar(20) DEFAULT NULL COMMENT '操作时间',
  `param_and_value` varchar(500) DEFAULT NULL COMMENT '请求参数名及参数值',
  `operate_class` varchar(200) DEFAULT NULL COMMENT '操作类',
  `operate_method` varchar(200) DEFAULT NULL COMMENT '操作方法',
  `cost_time` bigint(20) DEFAULT NULL COMMENT '执行方法耗时, 单位 ms',
  `source` int(1) DEFAULT NULL COMMENT '来源 : 1 PC , 2 Android , 3 IOS',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

#执行完成之后，157、158、159三个库中都被创建了这张表
```

2). 插入数据，也是在Mycat中执行的

```sql
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('1','user','insert','success','java.lang.String','10001','2020-02-26 18:12:28','{\"age\":\"20\",\"name\":\"Tom\",\"gender\":\"1\"}','cn.itcast.controller.UserController','insert','10',1);
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('2','user','insert','success','java.lang.String','10001','2020-02-26 18:12:27','{\"age\":\"20\",\"name\":\"Tom\",\"gender\":\"1\"}','cn.itcast.controller.UserController','insert','23',1);
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('3','user','update','success','java.lang.String','10001','2020-02-26 18:16:45','{\"age\":\"20\",\"name\":\"Tom\",\"gender\":\"1\"}','cn.itcast.controller.UserController','update','34',1);
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('4','user','update','success','java.lang.String','10001','2020-02-26 18:16:45','{\"age\":\"20\",\"name\":\"Tom\",\"gender\":\"1\"}','cn.itcast.controller.UserController','update','13',2);
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('5','user','insert','success','java.lang.String','10001','2020-02-26 18:30:31','{\"age\":\"200\",\"name\":\"TomCat\",\"gender\":\"0\"}','cn.itcast.controller.UserController','insert','29',3);
INSERT INTO `tb_log` (`id`, `model_name`, `model_value`, `return_value`, `return_class`, `operate_user`, `operate_time`, `param_and_value`, `operate_class`, `operate_method`, `cost_time`，`source`) VALUES('6','user','find','success','java.lang.String','10001','2020-02-26 18:30:31','{\"age\":\"200\",\"name\":\"TomCat\",\"gender\":\"0\"}','cn.itcast.controller.UserController','find','29',2);
```

```markdown
那上面咱们插入的数据都插入到哪个实体库中了呢？
157：select * from tb_log;
查到了id[3,6]

158：select * from tb_log;
查到了id[1,4]

159：select * from tb_log;
查到了id[2,5]
```

那这个是怎么进行分片的呢？

```markdown
我们用的是mod-long分片规则
进行取模：
	1%3 = 1 : 第二个节点 158
	2%3 = 2 : 第三个节点 159
	3%3 = 0 : 第一个节点 157
	...
```





### 5.3 分片规则

MyCat的分片规则配置在conf目录下的rule.xml文件中定义 ; 

环境准备 : 

1). schema.xml中的内容做好备份 , 并配置逻辑库; 

```xml
<schema name="PARTITION_DB" checkSQLschema="false" sqlMaxLimit="100">
	<table name="" dataNode="dn1,dn2,dn3" rule=""/>
</schema>


<dataNode name="dn1" dataHost="host1" database="partition_db" />
<dataNode name="dn2" dataHost="host2" database="partition_db" />
<dataNode name="dn3" dataHost="host3" database="partition_db" />


<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <writeHost host="hostM1" url="192.168.192.157:3306" user="root" password="itcast"></writeHost>
</dataHost>	

<dataHost name="host2" maxCon="1000" minCon="10" balance="0"
writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <writeHost host="hostM2" url="192.168.192.158:3306" user="root" password="itcast"></writeHost>
</dataHost>	

<dataHost name="host3" maxCon="1000" minCon="10" balance="0"
writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <writeHost host="hostM3" url="192.168.192.159:3306" user="root" password="itcast"></writeHost>
</dataHost>	
```

server.xml中的用户也需要进行修改

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">PARTITION_DB</property>
    <property name="readOnly">false</property>
    <property name="benchmark">1000</property>
    <property name="usingDecrypt">1</property>
</user>

<user name="test">
    <property name="password">123456</property>
    <property name="schemas">PARTITION_DB</property>
</user>

<user name="user">
    <property name="password">123456</property>
    <property name="schemas">PARTITION_DB</property>
    <property name="readOnly">true</property>
</user>
```



2). 在MySQL的三个节点的数据库中 , 创建数据库partition_db，是在实体库中创建的喔

```
create database partition_db DEFAULT CHARACTER SET utf8mb4;
```



#### 5.3.1 取模分片

```xml
<tableRule name="mod-long">
    <rule>
        <columns>id</columns>  <!-- 按照哪个字段进行求模运算 -->
        <algorithm>mod-long</algorithm>  <!-- 分片算法 -->
    </rule>
</tableRule>

<function name="mod-long" class="io.mycat.route.function.PartitionByMod">
    <property name="count">3</property>
</function>
```

配置说明 : 

| 属性      | 描述                             |
| --------- | -------------------------------- |
| columns   | 标识将要分片的表字段             |
| algorithm | 指定分片函数与function的对应关系 |
| class     | 指定该分片算法对应的类           |
| count     | 数据节点的数量                   |



#### 5.3.2 范围分片

根据指定的字段及其配置的范围与数据节点的对应情况， 来决定该数据属于哪一个分片 ， 配置如下： 

```xml
<tableRule name="auto-sharding-long">
	<rule>
		<columns>id</columns>
		<algorithm>rang-long</algorithm>
	</rule>
</tableRule>

<function name="rang-long" class="io.mycat.route.function.AutoPartitionByLong">
	<property name="mapFile">autopartition-long.txt</property>
    <property name="defaultNode">0</property> <!-- 这个的意思就是，如果插入的ID，没有被接地那覆盖，默认插入的节点序号 -->
</function>
```

autopartition-long.txt 配置如下： 

```properties
# range start-end ,data node index
# K=1000,M=10000.
0-500M=0
500M-1000M=1
1000M-1500M=2
```

含义为 ： 0 - 500 万之间的值 ， 存储在0号数据节点 ； 500万 - 1000万之间的数据存储在1号数据节点 ； 1000万 - 1500 万的数据节点存储在2号节点 ；



配置说明: 

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| columns     | 标识将要分片的表字段                                         |
| algorithm   | 指定分片函数与function的对应关系                             |
| class       | 指定该分片算法对应的类                                       |
| mapFile     | 对应的外部配置文件                                           |
| type        | 默认值为0 ; 0 表示Integer , 1 表示String                     |
| defaultNode | 默认节点 <br />默认节点的所用:枚举分片时,如果碰到不识别的枚举值, 就让它路由到默认节点 ; 如果没有默认值,碰到不识别的则报错 。 |

**测试: **

配置

```xml
<table name="tb_log" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"/>
```

数据

```sql
重启mycat
cd /usr/local/mycat/
./bin/mycat restart

show databases;
+-----------------+
| DATABASE        |
+-----------------+
| PARTITION_DB    |
+-----------------+

use partition_db;
#仅仅是有了这个逻辑表，但是实体库中并不存在实体表
show tables;
+------------------------+
| Tables in PARTITION_DB |
+------------------------+
| tb_log                 |
+------------------------+


1). 创建实体表，在Mycat当中执行这条语句
CREATE TABLE `tb_log` (
  id bigint(20) NOT NULL COMMENT 'ID',
  operateuser varchar(200) DEFAULT NULL COMMENT '姓名',
  operation int(2) DEFAULT NULL COMMENT '1: insert, 2: delete, 3: update , 4: select',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
insert into tb_log (id,operateuser ,operation) values(1,'Tom',1);
insert into tb_log (id,operateuser ,operation) values(2,'Cat',2);
insert into tb_log (id,operateuser ,operation) values(3,'Rose',3);
insert into tb_log (id,operateuser ,operation) values(4,'Coco',2);
insert into tb_log (id,operateuser ,operation) values(5,'Lily',1);

数据插入到哪里去了呢？
use partition_db;
node0 - 157 : 全部五条数据 ✅
node1 - 158 : 没有
node1 - 159 : 没有

那再插入一条数据
insert into tb_log (id,operateuser ,operation) values(5000001,'TomCat',4);
use partition_db;
node0 - 157 : 没有
node1 - 158 : 有的 ✅
node1 - 159 : 没有


insert into tb_log (id,operateuser ,operation) values(10000001,'RoseCat',1);
use partition_db;
node0 - 157 : 没有
node1 - 158 : 没有
node1 - 159 : 有的 ✅

insert into tb_log (id,operateuser ,operation) values(15000001,'CocoCat',1);
use partition_db;
node0 - 157 : 有的 ✅   这就是配置了defaultNode的结果
node1 - 158 : 没有
node1 - 159 : 没有 
```



#### 5.3.3 枚举分片

通过在配置文件中配置可能的枚举值, 指定数据分布到不同数据节点上, 本规则适用于按照省份、性别或状态拆分数据等业务 , 配置如下: 

```xml
<tableRule name="sharding-by-intfile">
    <rule>
        <columns>status</columns>
        <algorithm>hash-int</algorithm>
    </rule>
</tableRule>

<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
    <property name="mapFile">partition-hash-int.txt</property>
    <property name="type">0</property>
    <property name="defaultNode">0</property>
</function>
```

partition-hash-int.txt ，内容如下 :

```tex
#前面的这个1、2、3就是上面rule标签中的<columns>status</columns>
#后面的0、1、2就是对应的实体库节点
1=0
2=1
3=2
```

配置说明: 

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| columns     | 标识将要分片的表字段                                         |
| algorithm   | 指定分片函数与function的对应关系                             |
| class       | 指定该分片算法对应的类                                       |
| mapFile     | 对应的外部配置文件                                           |
| type        | 默认值为0 ; 0 表示Integer , 1 表示String                     |
| defaultNode | 默认节点 ; 小于0 标识不设置默认节点 , 大于等于0代表设置默认节点 ; <br />默认节点的所用:枚举分片时,如果碰到不识别的枚举值, 就让它路由到默认节点 ; 如果没有默认值,碰到不识别的则报错 。 |

有个问题，我TABLE_1对应的枚举分片字段是COLUMN_1，TABLE_2对应的枚举分片字段是COLUMN_2，那该怎么处理呢？

```xml
<!-- tableRule定义多个 -->
<tableRule name="sharding-by-enum-status">
    <rule>
        <columns>status</columns>
        <algorithm>hash-int</algorithm>
    </rule>
</tableRule>

<tableRule name="sharding-by-intfile">
    <rule>
        <columns>sharding_id</columns>
        <algorithm>hash-int</algorithm>
    </rule>
</tableRule>

<!-- function 也可以定义多个 -->
<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
    <property name="mapFile">partition-hash-int.txt</property> <!-- 主要是这个配置 -->
    <property name="type">0</property>
    <property name="defaultNode">0</property>
</function>
```



测试: 

配置，schema中添加逻辑表配置

```xml
<table name="tb_user" dataNode="dn1,dn2,dn3" rule="sharding-by-enum-status"/>
```

数据

```sql
#重启mycat
cd /usr/local/mycat/
./bin/mycat restart

mysql -h 127.0.0.1 -P 8066 -u root -p

show databases;逻辑库实体库都存在
use partition_db;
show tables;只有逻辑表


1). 创建表
CREATE TABLE `tb_user` (
  id bigint(20) NOT NULL COMMENT 'ID',
  username varchar(200) DEFAULT NULL COMMENT '姓名',
  status int(2) DEFAULT '1' COMMENT '1: 未启用, 2: 已启用, 3: 已关闭',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
insert into tb_user (id,username ,status) values(1,'Tom',1);
insert into tb_user (id,username ,status) values(2,'Cat',2);
insert into tb_user (id,username ,status) values(3,'Rose',3);
insert into tb_user (id,username ,status) values(4,'Coco',2);
insert into tb_user (id,username ,status) values(5,'Lily',1);


#看看数据在哪里
1=0 status:1 - 192.168.192.157  id:[1,5] 
2=1 status:2 - 192.168.192.158  id:[2,4]
3=2 status:3 - 192.168.192.159  id:[3]

那，插入个statu:4会怎么样呢？
insert into tb_user (id,username ,status) values(6,'WXX',4);
报错了，但是可以配置defaultNode，那么这样就有默认的存储库了。
```





#### 5.3.4 范围求模算法

该算法为先进行范围分片, 计算出分片组 , 再进行组内求模。

**优点**： 综合了范围分片和求模分片的优点。 分片组内使用求模可以保证组内的数据分布比较均匀， 分片组之间采用范围分片可以兼顾范围分片的特点。

**缺点**： 在数据范围时固定值（非递增值）时，存在不方便扩展的情况，例如将 dataNode Group size 从 2 扩展为 4 时，需要进行数据迁移才能完成 ； 如图所示： 

![image-20200110193319982](/Users/liuxiangren/mysql-learning/image/rule_slice_scpoe.png) 

因为128和129数据都存满了，所以要先将两个库的数据分一部分出来给到130这个服务器

配置如下： 

```xml
<tableRule name="auto-sharding-rang-mod">
	<rule>
		<columns>id</columns>
		<algorithm>rang-mod</algorithm>
	</rule>
</tableRule>

<function name="rang-mod" class="io.mycat.route.function.PartitionByRangeMod">
	<property name="mapFile">autopartition-range-mod.txt</property>
    <property name="defaultNode">0</property>
</function>
```

autopartition-range-mod.txt 配置格式 :

```
#range  start-end , data node group size
0-500M=1 #1个数据节点
500M1-2000M=2  #2个数据节点
```

在上述配置文件中, 等号前面的范围代表一个分片组 , 等号后面的数字代表该分片组所拥有的分片数量;



配置说明: 

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| columns     | 标识将要分片的表字段名                                       |
| algorithm   | 指定分片函数与function的对应关系                             |
| class       | 指定该分片算法对应的类                                       |
| mapFile     | 对应的外部配置文件                                           |
| defaultNode | 默认节点 ; 未包含以上规则的数据存储在defaultNode节点中, 节点从0开始 |



**测试:**

配置逻辑表，也是在schema.xml中 

```xml
<table name="tb_stu" dataNode="dn1,dn2,dn3" rule="auto-sharding-rang-mod"/>
```

数据

```sql
#分别连接到三台机器的mysql实体库中
都进入到partition_db中
use partition_db;

进入到192.168.192.157，mycat安装的机器重新启动mycat
cd /usr/local/mycat/
./bin/mycat restart

#查看mycat进程
ps -ef | grep -i mycat
#连接mycat
mysql -h 127.0.0.1 -P 8066 -u root -p
show datatbases;
+-----------------+
| DATABASE        |
+-----------------+
| PARTITION_DB    |
+-----------------+
use partition_db;
#下面这三个都是逻辑表
show tables;
+------------------------+
| Tables in PARTITION_DB |
+------------------------+
| tb_log                 |
+------------------------+
| tb_stu                 |
+------------------------+
| tb_user                |
+------------------------+


1). 创建表，在mycat中创建，他会自动同步到对应数据节点库中
    CREATE TABLE `tb_stu` (
      id bigint(20) NOT NULL COMMENT 'ID',
      username varchar(200) DEFAULT NULL COMMENT '姓名',
      status int(2) DEFAULT '1' COMMENT '1: 未启用, 2: 已启用, 3: 已关闭',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
    insert into tb_stu (id,username ,status) values(1,'Tom',1);
    insert into tb_stu (id,username ,status) values(2,'Cat',2);
    insert into tb_stu (id,username ,status) values(3,'Rose',3);
    insert into tb_stu (id,username ,status) values(4,'Coco',2);
    insert into tb_stu (id,username ,status) values(5,'Lily',1);
    
    insert into tb_stu (id,username ,status) values(5000001,'Roce',1);
    insert into tb_stu (id,username ,status) values(5000002,'Jexi',2);
    insert into tb_stu (id,username ,status) values(5000003,'Mini',1);

#那么前面的五条数据是在哪呢？
都存到了157上面
#再插入一条数据，看看是在哪存储呢？
insert into tb_stu (id,username ,status) values(6,'LilyXX',1);
查看还是在157上，因为我们刚才进行了范围的分片，小于500万的数据都在第一个节点中

执行插入500万以上的数据
158 第二组的第一个节点中有5000002这个节点
159 第二组的第二个节点中有5000001和5000003这两个节点

那再插入一条数据
insert into tb_stu (id,username ,status) values(5000004,'Mini',1);
158 第二组的第一个节点中有5000002和5000004这个节点
```

**总结就是组外”范围“，组内”取模“**





#### 5.3.5 固定分片hash算法

该算法类似于十进制的求模运算，但是为二进制的操作，例如，取 id 的二进制低 10 位 与 1111111111 进行位 & 运算。

最小值：

![image-20200112180630348](/Users/liuxiangren/mysql-learning/image/rule_slice_fixed_1.png) 

最大值：

![image-20200112180643493](/Users/liuxiangren/mysql-learning/image/rule_slice_fixed_2.png) 



**优点**： 这种策略比较灵活，可以均匀分配也可以非均匀分配，各节点的分配比例和容量大小由partitionCount和partitionLength两个参数决定

**缺点**：和取模分片类似。

配置如下 ： 

```xml
<tableRule name="sharding-by-long-hash">
    <rule>
        <columns>id</columns>
        <algorithm>func1</algorithm>
    </rule>
</tableRule>

<function name="func1" class="org.opencloudb.route.function.PartitionByLong">
    <property name="partitionCount">2,1</property>
    <property name="partitionLength">256,512</property>
</function>
```

在示例中配置的分片策略，希望将数据水平分成3份，前两份各占 25%，第三份占 50%。



**配置说明:** 

| 属性            | 描述                             |
| --------------- | -------------------------------- |
| columns         | 标识将要分片的表字段名           |
| algorithm       | 指定分片函数与function的对应关系 |
| class           | 指定该分片算法对应的类           |
| partitionCount  | 分片个数列表                     |
| partitionLength | 分片范围列表                     |

**约束 :** 

1). 分片长度 : 默认最大2^10 , 为 1024 ;

2). count, length的数组长度必须是一致的 ;

3). 两组数据的对应情况: (partitionCount[0]partitionLength[0])=(partitionCount[1]partitionLength[1])

以上分为三个分区:0-255,256-511,512-1023



**测试:**

配置

```xml
<table name="tb_brand" dataNode="dn1,dn2,dn3" rule="sharding-by-long-hash"/>
```

这个sharding-by-long-hash，在rule.xml并没有定义的，打开rule.xml

```xml
<tableRule name="sharding-by-long-hash">
  <rule>
  	<columns>id</columns>
    <algorithm>func1</algorithm>
  </rule>
</tableRule>
<!-- 这个rule.xml中其实应该是有的，有的话就进行修改 -->
<function name="func1" class="io.mycat.route.function.PartitionByLong">
	<property name="partitionCount">2,1</property>
  <property name="partitionLength">256,512</property>
</function>
```





数据

```sql
#重启mycat
cd /usr/local/mycat
./bin/mycat restart

#重新连接mycat
mysql -h 127.0.0.1 -P 8066 -u root -p

show datatbases;
+-----------------+
| DATABASE        |
+-----------------+
| PARTITION_DB    |
+-----------------+
use partition_db;
#下面这个tb_brand是逻辑表
show tables;
+------------------------+
| Tables in PARTITION_DB |
+------------------------+
| tb_brand               |
+------------------------+
| tb_log                 |
+------------------------+
| tb_stu                 |
+------------------------+
| tb_user                |
+------------------------+

1). 创建表
    CREATE TABLE `tb_brand` (
      id int(11) NOT NULL COMMENT 'ID',
      name varchar(200) DEFAULT NULL COMMENT '名称',
      firstChar char(1)  COMMENT '首字母',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
    insert into tb_brand (id,name ,firstChar) values(1,'七匹狼','Q');
    insert into tb_brand (id,name ,firstChar) values(529,'八匹狼','B');
    insert into tb_brand (id,name ,firstChar) values(1203,'九匹狼','J');
    insert into tb_brand (id,name ,firstChar) values(1205,'十匹狼','S');
    insert into tb_brand (id,name ,firstChar) values(1719,'六匹狼','L');
 
那这次数据又插入到哪些节点当中了呢？
157  :  id[1,1203,1205]
158  :  id[Empty]
159  :  id[529,1719]
```





#### 5.3.6 取模范围算法

该算法先进行取模，然后根据取模值所属范围进行分片。

**优点**：可以自主决定取模后数据的节点分布

**缺点**：dataNode 划分节点是事先建好的，需要扩展时比较麻烦。

 配置如下:

```xml
<tableRule name="sharding-by-pattern">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-pattern</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-pattern" class="io.mycat.route.function.PartitionByPattern">
	<property name="mapFile">partition-pattern.txt</property>
    <property name="defaultNode">0</property>
    <property name="patternValue">96</property>
</function>
```

partition-pattern.txt 配置如下: 

```
0-32=0
33-64=1
65-96=2
```

在mapFile配置文件中, 1-32即代表id%96后的分布情况。如果在1-32, 则在分片0上 ; 如果在33-64, 则在分片1上 ; 如果在65-96, 则在分片2上。



配置说明: 

| 属性         | 描述                                                       |
| ------------ | ---------------------------------------------------------- |
| columns      | 标识将要分片的表字段                                       |
| algorithm    | 指定分片函数与function的对应关系                           |
| class        | 指定该分片算法对应的类                                     |
| mapFile      | 对应的外部配置文件                                         |
| defaultNode  | 默认节点 ; 如果id不是数字, 无法求模, 将分配在defaultNode上 |
| patternValue | 求模基数                                                   |



**测试:**

配置

```xml
<table name="tb_mod_range" dataNode="dn1,dn2,dn3" rule="sharding-by-pattern"/>

<!--在rule.xml中进行配置-->
<table>
	<rule>
  	<columns>id</columns>
    <algorithm>sharding-by-pattern</algorithm>
  </rule>
</table>

<function name="sharding-by-pattern" class="io.mycat.route.function.PartitionByPattern">
	<property name="mapFile">partition-pattern.txt</property>
  <property name="defaultNode">0</property>
  <property name="patternValue">96</property>
</function>

在 rule.xml同级目录创建文件partition-pattern.txt，内容：
0-32=0
33-64=1
65-96=2  

```

数据

```sql
#重启mycat
cd /usr/local/mycat
./bin/mycat restart

#重新连接mycat
mysql -h 127.0.0.1 -P 8066 -u root -p

show datatbases;
+-----------------+
| DATABASE        |
+-----------------+
| PARTITION_DB    |
+-----------------+
use partition_db;
#下面这个tb_mod_range是逻辑表
show tables;
+------------------------+
| Tables in PARTITION_DB |
+------------------------+
| tb_brand               |
+------------------------+
| tb_log                 |
+------------------------+
| tb_mod_range           |
+------------------------+
| tb_stu                 |
+------------------------+
| tb_user                |
+------------------------+

#在Mycat中执行创建表的语句
1). 创建表
    CREATE TABLE `tb_mod_range` (
      id int(11) NOT NULL COMMENT 'ID',
      name varchar(200) DEFAULT NULL COMMENT '名称',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
    insert into tb_mod_range (id,name) values(1,'Test1');
    insert into tb_mod_range (id,name) values(2,'Test2');
    insert into tb_mod_range (id,name) values(3,'Test3');
    insert into tb_mod_range (id,name) values(4,'Test4');
    insert into tb_mod_range (id,name) values(5,'Test5');
    
#那么问题来了，这些数据到哪个实体库了哇？
157 ： 五条数据都在

insert into tb_mod_range (id,name) values(34,'Test34');
这个就插入到158了
```



<font color='red'>注意 : 取模范围算法只能针对于数字类型进行取模运算 ; 如果是字符串则无法进行取模分片 ;</font>





#### 5.3.7 字符串hash求模范围算法

与取模范围算法类似, 该算法支持数值、符号、字母取模，首先截取长度为 prefixLength 的子串，在对子串中每一个字符的 ASCII 码求和，然后对求和值进行取模运算（sum%patternValue），就可以计算出子串的分片数。

**优点**：可以自主决定取模后数据的节点分布

**缺点**：dataNode 划分节点是事先建好的，需要扩展时比较麻烦。



配置如下：

```xml
<tableRule name="sharding-by-prefixpattern">
	<rule>
	<!--	<columns>id</columns> -->
    <columns>username</columns>
		<algorithm>sharding-by-prefixpattern</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-prefixpattern" class="io.mycat.route.function.PartitionByPrefixPattern">
	<property name="mapFile">partition-prefixpattern.txt</property>
    <property name="prefixLength">5</property>
    <property name="patternValue">96</property>
</function>
```

partition-prefixpattern.txt 配置如下: 

```
# range start-end ,data node index
# ASCII
# 48-57=0-9
# 64、65-90=@、A-Z
# 97-122=a-z
###### first host configuration
0-32=0
33-64=1
65-96=2
```



配置说明: 

| 属性         | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| columns      | 标识将要分片的表字段                                         |
| algorithm    | 指定分片函数与function的对应关系                             |
| class        | 指定该分片算法对应的类                                       |
| mapFile      | 对应的外部配置文件                                           |
| prefixLength | 截取的位数; 将该字段获取前prefixLength位所有ASCII码的和, 进行求模sum%patternValue ,获取的值，在通配范围内的即分片数 ; |
| patternValue | 求模基数                                                     |

如 : 

```
字符串 :
	gf89f9a
	
截取字符串的前5位进行ASCII的累加运算 : 
	g - 103
	f - 102
	8 - 56
	9 - 57
	f - 102
	
    sum求和 : 103 + 102 + + 56 + 57 + 102 = 420
    求模 : 420 % 96 = 36
    
```

附录 ASCII码表 : 

![1577267028771](/Users/liuxiangren/mysql-learning/image/rule_slice_hash.png) 



**测试:**

配置

```xml
<table name="tb_u" dataNode="dn1,dn2,dn3" rule="sharding-by-prefixpattern"/>

在rule.xml中配置rule = sharding-by-prefixpattern
别忘了创建partition-prefixpattern.txt

```

数据

```sql
#重启mycat
#在mycat中执行下面的建表语句，三个库中都同步了

1). 创建表
    CREATE TABLE `tb_u` (
      username varchar(50) NOT NULL COMMENT '用户名',
      age int(11) default 0 COMMENT '年龄',
      PRIMARY KEY (`username`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
    insert into tb_u (username,age) values('Test100001',18);
    insert into tb_u (username,age) values('Test200001',20);
    insert into tb_u (username,age) values('Test300001',19);
    insert into tb_u (username,age) values('Test400001',25);
    insert into tb_u (username,age) values('Test500001',22);
    
#那问题来了，这五条数据插入到哪里去了呢？    
都在159节点里了
```







#### 5.3.8 应用指定算法

由运行阶段由应用自主决定路由到那个分片 , 直接根据字符子串（必须是数字）计算分片号 , 配置如下 : 

```xml
<tableRule name="sharding-by-substring">
	<rule>
		<columns>id</columns>
		<algorithm>sharding-by-substring</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-substring" class="io.mycat.route.function.PartitionDirectBySubString">
	<property name="startIndex">0</property> <!-- zero-based -->
	<property name="size">2</property>
	<property name="partitionCount">3</property>
	<property name="defaultPartition">0</property>
</function>
```

配置说明: 

| 属性             | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| columns          | 标识将要分片的表字段                                         |
| algorithm        | 指定分片函数与function的对应关系                             |
| class            | 指定该分片算法对应的类                                       |
| startIndex       | 字符子串起始索引                                             |
| size             | 字符长度                                                     |
| partitionCount   | 分区(分片)数量                                               |
| defaultPartition | 默认分片(在分片数量定义时, 字符标示的分片编号不在分片数量内时,使用默认分片) |

示例说明 : 

id=05-100000002 , 在此配置中代表根据id中从 startIndex=0，开始，截取siz=2位数字即05，05就是获取的分区，如果没传默认分配到defaultPartition 。



**测试:**

配置

```xml
<table name="tb_app" dataNode="dn1,dn2,dn3" rule="sharding-by-substring"/>
```

数据

```sql
1). 创建表
    CREATE TABLE `tb_app` (
      id varchar(10) NOT NULL COMMENT 'ID',
      name varchar(200) DEFAULT NULL COMMENT '名称',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


2). 插入数据
	insert into tb_app (id,name) values('00-00001','Testx00001');
    insert into tb_app (id,name) values('01-00001','Test100001');
    insert into tb_app (id,name) values('01-00002','Test200001');
    insert into tb_app (id,name) values('02-00001','Test300001');
    insert into tb_app (id,name) values('02-00002','TesT400001');
    
00就是第一个
01就是第二个
02就是第三个
```







#### 5.3.9 字符串hash解析算法

截取字符串中的指定位置的子字符串, 进行hash算法， 算出分片 ， 配置如下： 

```xml
<tableRule name="sharding-by-stringhash">
	<rule>
		<columns>user_id</columns>
		<algorithm>sharding-by-stringhash</algorithm>
	</rule>
</tableRule>

<function name="sharding-by-stringhash" class="io.mycat.route.function.PartitionByString">
	<property name="partitionLength">512</property> <!-- zero-based -->
	<property name="partitionCount">2</property>
	<property name="hashSlice">0:2</property>
</function>
```

配置说明: 

| 属性            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| columns         | 标识将要分片的表字段                                         |
| algorithm       | 指定分片函数与function的对应关系                             |
| class           | 指定该分片算法对应的类                                       |
| partitionLength | hash求模基数 ; length*count=1024 (出于性能考虑)              |
| partitionCount  | 分区数                                                       |
| hashSlice       | hash运算位 , 根据子字符串的hash运算 ; 0 代表 str.length() , -1 代表 str.length()-1 , 大于0只代表数字自身 ; 可以理解为substring（start，end），start为0则只表示0 |



**测试:** 

配置

```xml
<table name="tb_strhash" dataNode="dn1,dn2,dn3" rule="sharding-by-stringhash"/>
```

数据

```sql
1). 创建表
create table tb_strhash(
	name varchar(20) primary key,
	content varchar(100)
)engine=InnoDB DEFAULT CHARSET=utf8mb4;

2). 插入数据
INSERT INTO tb_strhash (name,content) VALUES('T1001', UUID());
INSERT INTO tb_strhash (name,content) VALUES('ROSE', UUID());
INSERT INTO tb_strhash (name,content) VALUES('JERRY', UUID());
INSERT INTO tb_strhash (name,content) VALUES('CRISTINA', UUID());
INSERT INTO tb_strhash (name,content) VALUES('TOMCAT', UUID());
```



原理: 

会创建一个数组，数组中前512个数都为0后512个数都为1，这就是（partition=512+partitionCount=2）组合后，这个数组的组成。

![image-20200112234530612](/Users/liuxiangren/mysql-learning/image/rule_slice_hash_detail.png) 





#### 5.3.10 一致性hash算法

一致性Hash算法有效的解决了分布式数据的拓容问题 , 配置如下: 



```xml
<tableRule name="sharding-by-murmur">
    <rule>
        <columns>id</columns>
        <algorithm>murmur</algorithm>
    </rule>
</tableRule>

<function name="murmur" class="io.mycat.route.function.PartitionByMurmurHash">
    <property name="seed">0</property>
    <property name="count">3</property><!--  -->
    <property name="virtualBucketTimes">160</property>
    <!-- <property name="weightMapFile">weightMapFile</property> -->
    <!-- <property name="bucketMapPath">/etc/mycat/bucketMapPath</property> -->
</function>
```

配置说明: 

| 属性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| columns            | 标识将要分片的表字段                                         |
| algorithm          | 指定分片函数与function的对应关系                             |
| class              | 指定该分片算法对应的类                                       |
| seed               | 创建murmur_hash对象的种子，默认0                             |
| count              | 要分片的数据库节点数量，必须指定，否则没法分片               |
| virtualBucketTimes | 一个实际的数据库节点被映射为这么多虚拟节点，默认是160倍，也就是虚拟节点数是物理节点数的160倍;virtualBucketTimes*count就是虚拟结点数量 ; |
| weightMapFile      | 节点的权重，没有指定权重的节点默认是1。以properties文件的格式填写，以从0开始到count-1的整数值也就是节点索引为key，以节点权重值为值。所有权重值必须是正整数，否则以1代替 |
| bucketMapPath      | 用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的murmur hash值与物理节点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 |



**测试：**

配置

```xml
<table name="tb_order" dataNode="dn1,dn2,dn3" rule="sharding-by-murmur"/>
```

数据

```sql
1). 创建表
create table tb_order(
	id int(11) primary key,
	money int(11),
	content varchar(200)
)engine=InnoDB ;

2). 插入数据
INSERT INTO tb_order (id,money,content) VALUES(1, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(212, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(312, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(412, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(534, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(621, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(754563, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(8123, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(91213, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(23232, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(112321, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(21221, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(112132, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(12132, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(124321, 100 , UUID());
INSERT INTO tb_order (id,money,content) VALUES(212132, 100 , UUID());
```





#### 5.3.11 日期分片算法

按照日期来分片

```xml
<tableRule name="sharding-by-date">
    <rule>
        <columns>create_time</columns>
        <algorithm>sharding-by-date</algorithm>
    </rule>
</tableRule>

<function name="sharding-by-date" class="io.mycat.route.function.PartitionByDate">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2020-01-01</property>
	<property name="sEndDate">2020-12-31</property>
    <property name="sPartionDay">10</property>
</function>
```

配置说明: 

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| columns     | 标识将要分片的表字段                                         |
| algorithm   | 指定分片函数与function的对应关系                             |
| class       | 指定该分片算法对应的类                                       |
| dateFormat  | 日期格式                                                     |
| sBeginDate  | 开始日期                                                     |
| sEndDate    | 结束日期，如果配置了结束日期，则代码数据到达了这个日期的分片后，会重复从开始分片插入 |
| sPartionDay | 分区天数，默认值 10 ，从开始日期算起，每个10天一个分区       |

注意：配置规则的表的 dataNode 的分片，必须和分片规则数量一致，例如 2020-01-01 到 2020-12-31 ，每10天一个分片，一共需要37个分片。





#### 5.3.12 单月小时算法

单月内按照小时拆分, 最小粒度是小时 , 一天最多可以有24个分片, 最小1个分片, 下个月从头开始循环, 每个月末需要手动清理数据。

配置如下 ： 

```xml
<tableRule name="sharding-by-hour">
    <rule>
        <columns>create_time</columns>
        <algorithm>sharding-by-hour</algorithm>
    </rule>
</tableRule>

<function name="sharding-by-hour" class="io.mycat.route.function.LatestMonthPartion">
	<property name="splitOneDay">24</property>
</function>
```

配置说明: 

| 属性        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| columns     | 标识将要分片的表字段 ； 字符串类型（yyyymmddHH）， 需要符合JAVA标准 |
| algorithm   | 指定分片函数与function的对应关系                             |
| splitOneDay | 一天切分的分片数                                             |





#### 5.3.13 自然月分片算法

使用场景为按照月份列分区, 每个自然月为一个分片, 配置如下: 

```xml
<tableRule name="sharding-by-month">
    <rule>
        <columns>create_time</columns>
        <algorithm>sharding-by-month</algorithm>
    </rule>
</tableRule>

<function name="sharding-by-month" class="io.mycat.route.function.PartitionByMonth">
	<property name="dateFormat">yyyy-MM-dd</property>
	<property name="sBeginDate">2020-01-01</property>
	<property name="sEndDate">2020-12-31</property>
</function>
```

配置说明: 

| 属性       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| columns    | 标识将要分片的表字段                                         |
| algorithm  | 指定分片函数与function的对应关系                             |
| class      | 指定该分片算法对应的类                                       |
| dateFormat | 日期格式                                                     |
| sBeginDate | 开始日期                                                     |
| sEndDate   | 结束日期，如果配置了结束日期，则代码数据到达了这个日期的分片后，会重复从开始分片插入 |



#### 5.3.14 日期范围hash算法

其思想和范围取模分片一样，先根据日期进行范围分片求出分片组，再根据时间hash使得短期内数据分布的更均匀 ;

优点 : 可以避免扩容时的数据迁移，又可以一定程度上避免范围分片的热点问题

注意 : 要求日期格式尽量精确些，不然达不到局部均匀的目的

```xml
<tableRule name="range-date-hash">
    <rule>
        <columns>create_time</columns>
        <algorithm>range-date-hash</algorithm>
    </rule>
</tableRule>

<function name="range-date-hash" class="io.mycat.route.function.PartitionByRangeDateHash">
	<property name="dateFormat">yyyy-MM-dd HH:mm:ss</property>
	<property name="sBeginDate">2020-01-01 00:00:00</property>
	<property name="groupPartionSize">6</property>
    <property name="sPartionDay">10</property>
</function>
```

配置说明: 

| 属性             | 描述                                   |
| ---------------- | -------------------------------------- |
| columns          | 标识将要分片的表字段                   |
| algorithm        | 指定分片函数与function的对应关系       |
| class            | 指定该分片算法对应的类                 |
| dateFormat       | 日期格式 , 符合Java标准                |
| sBeginDate       | 开始日期 , 与 dateFormat指定的格式一致 |
| groupPartionSize | 每组的分片数量                         |
| sPartionDay      | 代表多少天为一组                       |





## 6 MyCat高级

### 6.1 MyCat 性能监控

#### 6.1.1 MyCat-web简介

Mycat-web 是 Mycat 可视化运维的管理和监控平台，弥补了 Mycat 在监控上的空白。帮 Mycat 分担统计任务和配置管理任务。Mycat-web 引入了 ZooKeeper 作为配置中心，可以管理多个节点。Mycat-web 主要管理和监控 Mycat 的流量、连接、活动线程和内存等，具备 IP 白名单、邮件告警等模块，还可以统计 SQL 并分析慢 SQL 和高频 SQL 等。为优化 SQL 提供依据。

![1577358192118](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-brief.png) 



#### 6.1.2 MyCat-web下载

下载地址 : http://dl.mycat.io/、http://dl.mycat.org.cn

![1577369790112](/Users/liuxiangren/mysql-learning/image/mycat-senior-mycat-web-dowload.png) 



#### 6.1.3 Mycat-web安装配置

##### 6.1.3.1 安装

1). 安装Zookeeper(保证JDK已经安装)，直接安装在157上

```
A. 上传安装包 
	alt + p -----> put D:\tmp\zookeeper-3.4.11.tar.gz
	
B. 解压
	tar -zxvf zookeeper-3.4.11.tar.gz -C /usr/local/

C. 创建数据存放目录
	mkdir data

D. 修改配置文件名称并配置
	cd /usr/local/zookeeper-3.4.11/conf/ 
	mv zoo_sample.cfg zoo.cfg

E. 配置数据存放目录
	vim zoo.cfg
	#修改其中的dataDir，就是上面创建的data目录
	dataDir=/usr/local/zookeeper-3.4.11/data
	
F. 启动Zookeeper
	bin/zkServer.sh start
```



2). 安装Mycat-web

```
A. 上传安装包 
	alt + p --------> put D:\tmp\Mycat-web-1.0-SNAPSHOT-20170102153329-linux.tar.gz
	
B. 解压
	tar -zxvf Mycat-web-1.0-SNAPSHOT-20170102153329-linux.tar.gz -C /usr/local/

C. 目录介绍
		cd /usr/local/mycat-web/
		ll
    drwxr-xr-x. 2 root root  4096 Oct 19  2015 etc         ----> jetty配置文件
    drwxr-xr-x. 3 root root  4096 Oct 19  2015 lib         ----> 依赖jar包
    drwxr-xr-x. 7 root root  4096 Jan  1  2017 mycat-web   ----> mycat-web项目
    -rwxr-xr-x. 1 root root   116 Oct 19  2015 readme.txt
    -rwxr-xr-x. 1 root root 17125 Oct 19  2015 start.jar   ----> 启动jar
    -rwxr-xr-x. 1 root root   381 Oct 19  2015 start.sh    ----> linux启动脚本

D. 启动
	sh start.sh
	
E. 访问
	http://192.168.192.157:8082/mycat
```

如果Zookeeper与Mycat-web不在同一台服务器上 , 需要设置Zookeeper的地址 ; 在/usr/local/mycat-web/mycat-web/WEB-INF/classes/mycat.properties文件中配置,localhost改为具体的IP地址 : 

![1577370960657](/Users/liuxiangren/mysql-learning/image/mycat-senior-zk-prop.png) 



##### 6.1.3.2 配置

刚一进去并看不到什么有价值的信息，因为我们还没进行mycat-web的配置，mycat-web甚至都不知道mycat的信息，IP地址改为自己的IP地址，数据库名称就是我们在schema.xml中配置的逻辑库名称，用户名就是在server.xml中配置的用户名密码

![1577372353498](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-config.png) 

![1577372371549](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-config-manager.png) 



#### 6.1.4 Mycat-web之MyCat性能监控

在 Mycat-web 上可以进行 Mycat 【性能监控】，例如：内存分享、流量分析、连接分析、活动线程分析等等。 如下图: 

A. MyCat内存分析: 

![1577373437531](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-memory.png)  

MyCat的内存分析 , 反映了当前的内存使用情况与历史时间段的峰值、平均值。



B. MyCat流量分析: 

![1577373861622](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-flow.png) 

MyCat流量分析统计了历史时间段的流量峰值、当前值、平均值，是MyCat数据传输的重要指标， In代表输入， Out代表输出。



C. MyCat连接分析

![1577374030291](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-connections.png) 

MyCat连接分析, 反映了MyCat的连接数 



D. MyCat TPS分析

![1577374126073](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-tps.png) 

MyCat TPS 是并发性能的重要参数指标, 指系统在每秒内能够处理的请求数量。 MyCat TPS的值越高 , 代表MyCat单位时间内能够处理的请求就越多, 并发能力也就越高。



E. MyCat活动线程分析反映了MyCat线程的活动情况。

F. MyCat缓存队列分析, 反映了当前在缓存队列中的任务数量。



还有Mycat物理节点，其中物理节点的名字就是schema.xml中dataHost的writeHost中的host属性的值



#### 6.1.5 Mycat-web之MySQL性能监控指标

1). MySQL配置

![1577374634946](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-mysql-nodes.png) 

2). MySQL监控指标

![1577374588708](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-mysql-services.png) 

可以通过MySQL服务监控, 检测每一个MySQL节点的运行状态, 包含缓存命中率 、增删改查比例、流量统计、慢查询比例、线程、临时表等相关性能数据。



#### 6.1.6 Mycat-web之SQL监控

1). SQL 统计

![1577374982024](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql.png) 



2). SQL表分析

![1577375016852](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql-statistic.png) 



3). SQL监控

![1577375043787](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql-count.png)  



4). 高频SQL

![1577375072881](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql-fast.png) 



5). 慢SQL统计

![1577375100383](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql-slow.png) 



6). SQL解析

![1577375162928](/Users/liuxiangren/mysql-learning/image/mycat-senior-monitor-sql-route.png) 





### 6.2 MyCat 读写分离

#### 6.2.1 MySQL主从复制原理

复制是指将主数据库的DDL 和 DML 操作通过二进制日志传到从库服务器中，然后在从库上对这些日志重新执行（也叫重做），从而使得从库和主库的数据保持同步。

MySQL支持一台主库同时向多台从库进行复制， 从库同时也可以作为其他从服务器的主库，实现链状复制。



MySQL主从复制的原理如下 : 

![image-20200103093716416](/Users/liuxiangren/mysql-learning/image/mysql-cluster-rw-basic.png) 

从上层来看，复制分成三步：

- Master 主库在事务提交时，会把数据变更作为时间 Events 记录在二进制日志文件 Binlog 中。
- 主库推送二进制日志文件 Binlog 中的日志事件到从库的中继日志 Relay Log 。

- slave重做中继日志中的事件，将改变反映它自己的数据。



MySQL 复制的优点：

- 主库出现问题，可以快速切换到从库提供服务。
- 可以在从库上执行查询操作，从主库中更新，实现读写分离，降低主库的访问压力。
- 可以在从库中执行备份，以避免备份期间影响主库的服务。





#### 6.2.2 MySQL一主一从搭建

准备的两台机器: 

| MySQL  | IP              | 端口号 |
| ------ | --------------- | ------ |
| Master | 192.168.192.157 | 3306   |
| Slave  | 192.168.192.158 | 3306   |



##### 6.2.2.1 master

1） 在master 的配置文件（/usr/my.cnf）中，配置如下内容：

```properties
#mysql 服务ID,保证整个集群环境中唯一
server-id=1

#mysql binlog 日志的存储路径和文件名
log-bin=/var/lib/mysql/mysqlbin

#设置logbin格式
binlog_format=STATEMENT

#是否只读,1 代表只读, 0 代表读写
read-only=0

#忽略的数据, 指不需要同步的数据库
#binlog-ignore-db=mysql

#指定同步的数据库
binlog-do-db=db01
```



2） 执行完毕之后，需要重启Mysql：

```sql
service mysql restart;
```



3） 创建同步数据的账户，并且进行授权操作：

```sql
mysql -uroot -p;
#主库创建，从库使用
grant replication slave on *.* to 'mclub'@'192.168.192.158' identified by '123456';	

flush privileges;
```



4） 查看master状态：

```sql
show master status;
```



![image-20200103102209631](/Users/liuxiangren/mysql-learning/image/mycat-cluster-rw-master-status.png) 

字段含义:

```
File : 从哪个日志文件开始推送日志文件 
Position ： 从哪个位置开始推送日志
Binlog_Ignore_DB : 指定不需要同步的数据库
```





##### 6.2.2.2 slave

1） 在 slave 端配置文件/usr/my.cnf中，配置如下内容：

```properties
#mysql服务端ID,唯一
server-id=2

#指定binlog日志
log-bin=/var/lib/mysql/mysqlbin

#启用中继日志
relay-log=mysql-relay
```



2）  执行完毕之后，需要重启Mysql：

```
service mysql restart;
```



3） 执行如下指令 ：

```sql
change master to master_host= '192.168.192.157', master_user='mclub', master_password='123456', master_log_file='mysqlbin.000001', master_log_pos=413;
```

指定当前从库对应的主库的IP地址，这个用户名，密码就是我们上面创建的用户名和密码，从哪个日志文件开始的那个位置开始同步推送日志。



4） 开启同步操作

```
start slave;

show slave status \G;
```

 ![image-20200103144903105](/Users/liuxiangren/mysql-learning/image/mysql-cluster-build-check.png)  



5） 停止同步操作

```
stop slave;
```



##### 6.2.2.3 验证主从同步

1） 在主库中创建数据库，创建表，并插入数据 ：

```sql
create database db01;

user db01;

create table user(
	id int(11) not null auto_increment,
	name varchar(50) not null,
	sex varchar(1),
	primary key (id)
)engine=innodb default charset=utf8;

insert into user(id,name,sex) values(null,'Tom','1');
insert into user(id,name,sex) values(null,'Trigger','0');
insert into user(id,name,sex) values(null,'Dawn','1');
```



2） 在从库中查询数据，进行验证 ：

在从库中，可以查看到刚才创建的数据库：

![image-20200103103029311](/Users/liuxiangren/mysql-learning/image/mysql-cluster-slave-db-check.png)  

在该数据库中，查询user表中的数据：

![image-20200103103049675](/Users/liuxiangren/mysql-learning/image/mysql-cluster-slave-tb-check.png)  



#### 6.2.3 MyCat一主一从读写分离

##### 6.2.3.1 读写分离原理

![image-20200103140249789](/Users/liuxiangren/mysql-learning/image/mysql-cluster-rw-arch.png) 

读写分离,简单地说是把对数据库的读和写操作分开,以对应不同的数据库服务器。主数据库提供写操作，从数据库提供读操作，这样能有效地减轻单台数据库的压力。

通过MyCat即可轻易实现上述功能，不仅可以支持MySQL，也可以支持Oracle和SQL Server。

MyCat控制后台数据库的读写分离和负载均衡由schema.xml文件datahost标签的balance属性控制。



##### 6.2.3.2 读写分离配置

配置如下： 

1). 检查MySQL的主从复制是否运行正常 .

2). 修改MyCat 的conf/schema.xml 配置如下:

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<schema name="ITCAST" checkSQLschema="true" sqlMaxLimit="100">
		<table name="user" dataNode="dn1" primaryKey="id"/>
	</schema>
   
  <!-- 主从搭建时创建的database就是db01 -->
	<dataNode name="dn1" dataHost="localhost1" database="db01" />
   <!-- 主从复制重要属性balance -->
	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" 	
				dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.157:3306" user="root" password="itcast">
			<readHost host="hostS1" url="192.168.192.158:3306" user="root" password="itcast" />
		</writeHost>
	</dataHost>  
</mycat:schema>
```

3). 修改conf/server.xml

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">ITCAST</property>
</user>

<user name="test">
    <property name="password">123456</property>
    <property name="schemas">ITCAST</property>
</user>

<user name="user">
    <property name="password">123456</property>
    <property name="schemas">ITCAST</property>
    <property name="readOnly">true</property>
</user>
```



4). 配置完毕之后, 重启MyCat服务;

dataHost中的balance设置了0，不开启读写分离，所有的请求都发送到了主节点

```shell
cd /usr/local/mycat/
./bin/mycat restart

mysql -h 127.0.0.1 -P 8066 -u root -p

show databases;
+---------+
| DATABSE |
+---------+
| ITCAST  |
+---------+

use ITCAST;
show tablse;

+------------------+
| Tables in ITCAST |
+------------------+
| user             |
+------------------+

select * from user;
+----+---------+-----+
| id | name    | sex |
+----+---------+-----+
|  1 | Tom     |  1  |
+--------------+-----+
|  2 | Trigger |  0  |
+--------------+-----+
|  3 | Dawn    |  1  |
+--------------+-----+

那我想看看这个select的SQL由没有走这个从节点啊？
#修改日志信息配置文件
vim /usr/local/conf/log4j2.xml
#修改
<asyncRoot level="debug" includeLocation="true"> <!-- 要修改这个level为debug -->

cd /usr/local/logs/
tail -f mycat.log

再执行一次查询，再看看日志文件，唉？怎么走的是157啊？
那么修安排将dataHost的balance配置为1
然后查询就走158了

那么insert、update都在157上
select都在158上了
```



属性含义说明: 

```
checkSQLschema
	当该值设置为true时, 如果我们执行语句"select * from test01.user ;" 语句时, MyCat则会把schema字符去掉 , 可以避免后端数据库执行时报错 ;
	
	
balance
	负载均衡类型, 目前取值有4种:
	
	balance="0" : 不开启读写分离机制 , 所有读操作都发送到当前可用的writeHost上.
	
	balance="1" : 全部的readHost 与 stand by writeHost (备用的writeHost) 都参与select 语句的负载均衡,简而言之,就是采用双主双从模式(M1 --> S1 , M2 --> S2, 正常情况下，M2,S1,S2 都参与 select 语句的负载均衡。)，单主单从也可以为1;
    
    balance="2" : 所有的读写操作都随机在writeHost , readHost上分发，这就读写不分离了啊
    
    balance="3" : 所有的读请求随机分发到writeHost对应的readHost上执行, writeHost不负担读压力 ;balance=3 只在MyCat1.4 之后生效，单主单从也可以设置3.
```

##### 6.2.3.3 验证读写分离

修改balance的值, 查询MyCat中的逻辑表中的数据变化; 

#### 6.2.4 MySQL双主双从搭建

##### 6.2.4.1 架构

一个主机 Master1 用于处理所有写请求，它的从机 Slave1 和另一台主机 Master2 还有它的从机 Slave2 负责所有读请求。当 Master1 主机宕机后，Master2 主机负责写请求，Master1 、Master2 互为备机。架构图如下: 

 ![image-20200103170452653](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw.png) 





##### 6.2.4.2 双主双从配置

准备的机器如下: 

| 编号 | 角色    | IP地址          | 端口号 |
| ---- | ------- | --------------- | ------ |
| 1    | Master1 | 192.168.192.157 | 3306   |
| 2    | Slave1  | 192.168.192.158 | 3306   |
| 3    | Master2 | 192.168.192.159 | 3306   |
| 4    | Slave2  | 192.168.192.160 | 3306   |



之前157和158是有主从关系的，先清除掉

```shell
#清除主从关系的话，只需要在158上(也就是slave上)进行操作
mysql -uroot -p

show slave status \G;

#停止主从复制的话，就先执行stop slave;
stop slave;

#上面的指令只是停了从节点，要重置主从之间的关系，那还要再执行一条指令，同样也是在158上执行
reset master;
```







**1). 双主机配置**

Master1配置: 

```properties
#主服务器唯一ID
server-id=1

#启用二进制日志
log-bin=mysql-bin

# 设置不要复制的数据库(可设置多个)
# binlog-ignore-db=mysql
# binlog-ignore-db=information_schema

#设置需要复制的数据库
binlog-do-db=db02
binlog-do-db=db03
binlog-do-db=db04

#设置logbin格式
binlog_format=STATEMENT

# 在作为从数据库的时候，有写入操作也要更新二进制日志文件
log-slave-updates
```

Master2配置: 

```properties
#主服务器唯一ID
server-id=3

#启用二进制日志
log-bin=mysql-bin

# 设置不要复制的数据库(可设置多个)
#binlog-ignore-db=mysql
#binlog-ignore-db=information_schema

#设置需要复制的数据库
binlog-do-db=db02
binlog-do-db=db03
binlog-do-db=db04

#设置logbin格式
binlog_format=STATEMENT

# 在作为从数据库的时候，有写入操作也要更新二进制日志文件
log-slave-updates
```



**2). 双从机配置**

Slave1配置: 

```properties
#从服务器唯一ID
server-id=2

#启用中继日志
relay-log=mysql-relay
```



Salve2配置:

```properties
#从服务器唯一ID
server-id=4

#启用中继日志
relay-log=mysql-relay
```



**3). 双主机、双从机重启 mysql 服务**

```shell
service mysql restart
```



**4). 主机从机都关闭防火墙**

```shell
service iptables status;
service iptables stop;
```



**5). 在两台主机上建立帐户并授权 slave**

```sql
#在主机MySQL里执行授权命令，前itcast是用户名后itcast是密码
GRANT REPLICATION SLAVE ON *.* TO 'itcast'@'%' IDENTIFIED BY 'itcast';

flush privileges;
```

查询Master1的状态 : 

![image-20200104090901765](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-master-check.png) 

查询Master2的状态 :

![image-20200104090922386](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-master-status.png) 



**6). 在从机上配置需要复制的主机**

Slave1 复制 Master1，Slave2 复制 Master2

slave1 指令: 

```sql
#下面的MASTER_LOG_FILE和MASTER_LOG_POS都是上面157的status中对应的File和Position
CHANGE MASTER TO MASTER_HOST='192.168.192.157',
MASTER_USER='itcast',
MASTER_PASSWORD='itcast',
MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=409;

如有报错：
Error 1198 (HY000): This operation cannot be performed with a running slave; run STOP SLAVE first

#那么就要执行
stop slave;
reset master;
#然后再执行上面的CHANGE MASTER TO ...
```



slave2 指令:

```sql
#下面的MASTER_LOG_FILE和MASTER_LOG_POS都是上面159的status中对应的File和Position
CHANGE MASTER TO MASTER_HOST='192.168.192.159',
MASTER_USER='itcast',
MASTER_PASSWORD='itcast',
MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=409;
```

**7). 启动两台从服务器复制功能 , 查看主从复制的运行状态**

```
start slave;

show slave status\G;
```

![image-20200104091917814](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-slave-check-1.png) 



![image-20200104091948213](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-slave-check-2.png) 



**8). 两个主机互相复制**

Master2 复制 Master1，Master1 复制 Master2

Master1 执行指令: 

```sql
CHANGE MASTER TO MASTER_HOST='192.168.192.159',
MASTER_USER='itcast',
MASTER_PASSWORD='itcast',
MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=409;
```



Master2 执行指令:

```sql
CHANGE MASTER TO MASTER_HOST='192.168.192.157',
MASTER_USER='itcast',
MASTER_PASSWORD='itcast',
MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=409;
```



**9). 启动两台主服务器复制功能 , 查看主从复制的运行状态**

```
start slave;

show slave status\G;
```

![image-20200104092654432](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-slave-check-3.png) 



![image-20200104092741892](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-slave-check-4.png) 



**10). 验证**

```sql
create database db03;

use db03;

create table user(
	id int(11) not null auto_increment,
	name varchar(50) not null,
	sex varchar(1),
	primary key (id)
)engine=innodb default charset=utf8;

insert into user(id,name,sex) values(null,'Tom','1');
insert into user(id,name,sex) values(null,'Trigger','0');
insert into user(id,name,sex) values(null,'Dawn','1');


insert into user(id,name,sex) values(null,'Jack Ma','1');
insert into user(id,name,sex) values(null,'Coco','0');
insert into user(id,name,sex) values(null,'Jerry','1');
```



在Master1上创建数据库: 

![image-20200104095232047](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-all-test-1.png) 



在Master1上创建表 :

![image-20200104095521070](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-all-test-2.png) 



**11). 停止从服务复制功能**

```
stop slave;
```



**12). 重新配置主从关系**

```sql
stop slave;
reset master;
```





#### 6.2.5 MyCat双主双从读写分离

##### 6.2.5.1 配置

修改\<dataHost>的 balance属性，通过此属性配置读写分离的类型 ; 

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	
	<schema name="ITCAST" checkSQLschema="true" sqlMaxLimit="100">
		<table name="user" dataNode="dn1" primaryKey="id"/>
	</schema>
	
	<dataNode name="dn1" dataHost="localhost1" database="db03" />

	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" 	
				dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.147:3306" user="root" password="itcast">
			<readHost host="hostS1" url="192.168.192.149:3306" user="root" password="itcast" />
		</writeHost>
		
		<writeHost host="hostM2" url="192.168.192.150:3306" user="root" password="itcast">
			<readHost host="hostS2" url="192.168.192.151:3306" user="root" password="itcast" />
		</writeHost>
	</dataHost>
    
</mycat:schema>
```

1). balance

1 : 代表 全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下，M2,S1,S2 都参与 select 语句的负载均衡 ;

2). writeType

0 : 写操作都转发到第1台writeHost, writeHost1挂了, 会切换到writeHost2上;

1 : 所有的写操作都随机地发送到配置的writeHost上 ;

3). switchType

-1 : 不自动切换

1 : 默认值, 自动切换

2 : 表示基于MySQL的主从同步状态决定是否切换, 心跳语句 : show slave status



##### 6.2.5.2 读写分离验证

查询数据 : select * from user;

![image-20200104101106144](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-all-assert-1.png) 

插入数据 : insert into user(id,name,sex) values(null,'Dawn','1');

![image-20200104100956216](/Users/liuxiangren/mysql-learning/image/mysql-cluster-double-rw-all-assert-2.png) 





##### 6.2.5.3 可用性验证

关闭Master1 , 然后再执行写入的SQL语句 , 通过日志查询当前写入操作, 操作的是那台服务器 ;









## Linux命令

### 上传文件到Linux

rz -y和rz -E的共同点和区别：

共同点：

都可以把文件上传到Linux中

区别：

rz -y： 把文件上传到Linux中，如果有相同文件名的文件，会将其覆盖。
（比如：所上传的路径中有一个hbase-env.sh文件，这时使用rz -y再上传一个hbase-env.sh文件，它会把原来的hbase-env.sh文件覆盖掉）

rz -E： 把文件上传到Linux中，如果有相同文件名的文件，不会将其覆盖，而是会在所上传文件后面加上 .0 ，两个文件都会存在与此目录中，再次上传则会在文件名后加上 .1，以此类推。
（比如：所上传路径中有一个hbase-env.sh文件，这时使用rz -E再上传一个hbase-env.sh文件，原来的hbase-env.sh不会受影响，所上传的文件会变成hbase-env.sh.0，与原来的hbase-env.sh文件并存，再次上传会再多一个hbase-env.sh.1，再上传会再多一个hbase-env.sh.2、hbase-env.sh.3，以此类推）

### 下载Linux文件到本地

sz的用法
sz命令可以单下载一个文件，也可以多个文件同时下载

```shell
[root@vdedu vastedu]# sz ashrpt_1_1223_1334.html awrrpt_1_9112_9113.html 
rz
Starting zmodem transfer.  Press Ctrl+C to cancel.
Transferring ashrpt_1_1223_1334.html...
  100%      45 KB      45 KB/sec    00:00:01       0 Errors  
Transferring awrrpt_1_9112_9113.html...
  100%     699 KB     699 KB/sec    00:00:01       0 Errors  
[root@vdedu vastedu]#
```



## MySQL命令

### grant

```markdown
用户权限管理主要有以下作用： 
1. 可以限制用户访问哪些库、哪些表 
2. 可以限制用户对哪些表执行SELECT、CREATE、DELETE、DELETE、ALTER等操作 
3. 可以限制用户登录的IP或域名 
4. 可以限制用户自己的权限是否可以授权给别的用户
```

#### 一、用户授权

mysql> grant all privileges on *.* to 'yangxin'@'%' identified by 'yangxin123456' with grant option;

添加权限（和已有权限合并，不会覆盖已有权限）

GRANT Insert ON `your database`.* TO `user`@`host`;

删除权限

REVOKE Delete ON `your database`.* FROM `user`@`host`;

all privileges：表示将所有权限授予给用户。也可指定具体的权限，如：SELECT、CREATE、DROP等。
on：表示这些权限对哪些数据库和表生效，格式：数据库名.表名，这里写“*”表示所有数据库，所有表。如果我要指定将权限应用到test库的user表中，可以这么写：test.user
to：将权限授予哪个用户。格式：”用户名”@”登录IP或域名”。%表示没有限制，在任何主机都可以登录。比如：”yangxin”@”192.168.0.%”，表示yangxin这个用户只能在192.168.0IP段登录
identified by：指定用户的登录密码
with grant option：表示允许用户将自己的权限授权给其它用户
可以使用GRANT给用户添加权限，权限会自动叠加，不会覆盖之前授予的权限，比如你先给用户添加一个SELECT权限，后来又给用户添加了一个INSERT权限，那么该用户就同时拥有了SELECT和INSERT权限。

#### 二、刷新权限

对用户做了权限变更之后，一定记得重新加载一下权限，将权限信息从内存中写入数据库。

mysql> flush privileges;

#### 三、查看用户权限

mysql> grant select,create,drop,update,alter on *.* to 'yangxin'@'localhost' identified by 'yangxin0917' with grant option;
mysql> show grants for 'yangxin'@'localhost';

#### 四、回收权限

删除yangxin这个用户的create权限，该用户将不能创建数据库和表。

mysql> revoke create on *.* from 'yangxin@localhost';
mysql> flush privileges;

MySQL grant命令：https://www.cnblogs.com/felix-h/p/11072743.html











## MySQL .cnf配置详解

https://www.cnblogs.com/langdashu/p/5889352.html

## MYSQL 字符集问题

设置字符集：https://www.cnblogs.com/miclesvic/p/10345235.html

## XA事务

参考文章XA事务：https://blog.csdn.net/hengyunabc/article/details/19433947

## Docker搭建Mycat Cluster

Docker Mysql Mycat搭建（主要参考）：https://blog.csdn.net/weixin_41869700/article/details/105271813

## Docker-Compose build Mycat Cluster

Docker-Compose Mysql Cluster :https://blog.csdn.net/qq_41967899/article/details/104261198

### Mysql Cluster优劣势

Mysql集群架构优劣：https://www.cnblogs.com/wuxu/p/13161438.html
