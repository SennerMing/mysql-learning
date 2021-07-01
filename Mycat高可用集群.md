# Mycat高可用集群

## 1.1 集群架构

### 1.1.1 MyCat实现读写分离架构

在上面的章节, 我们已经讲解过了通过MyCat来实现MySQL的读写分离, 从而完成MySQL集群的负载均衡 , 如下面的结构图: 

![image-20200104144550132](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200104144550132.png) 

但是以上架构存在问题 , 由于MyCat中间件是单节点的服务, 前端客户端所有的压力过来都直接请求这一台MyCat , 存在单点故障。所以这个时候， 我们就需要考虑MyCat的集群 ；





### 1.1.2 MyCat集群架构

通过MyCat来实现后端MySQL的负载均衡 ， 通过HAProxy再实现MyCat集群的负载均衡 ; 

![image-20200104151016408](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200104151016408.png) 

HAProxy 负责将请求分发到 MyCat 上，起到负载均衡的作用，同时 HAProxy 也能检测到 MyCat 是否存活，HAProxy 只会将请求转发到存活的 MyCat 上。如果一台 MyCat 服务器宕机，HAPorxy 转发请求时不会转发到宕机的 MyCat 上，所以 MyCat 依然可用。



**HAProxy介绍:**

HAProxy 是一个开源的、高性能的基于TCP(第四层)和HTTP(第七层)应用的负载均衡软件。 使用HAProxy可以快速、可靠地实现基于TCP与HTTP应用的负载均衡解决方案。

具有以下优点： 

①. 可靠性和稳定性好, 可以与硬件级的F5负载均衡服务器媲美 ;

②. 处理能力强, 最高可以通过维护4w-5w个并发连接, 单位时间处理的最大请求数达到2w个 ;

③. 支持多种负载均衡算法 ;

④. 有功能强大的监控界面, 通过此页面可以实时了解系统的运行情况 ;



但是， 上述的架构也是存在问题的， 因为所以的客户端请求都是先到达HAProxy, 由HAProxy再将请求再向下分发, 如果HAProxy宕机的话, 就会造成整个MyCat集群不能正常运行, 依然存在单点故障。





### 1.1.3 MyCat的高可用集群

![image-20200104153537319](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200104153537319.png) 



**图解说明：**
1). HAProxy 实现了 MyCat 多节点的集群高可用和负载均衡，而 HAProxy 自身的高可用则可以通过Keepalived 来实现。因此，HAProxy 主机上要同时安装 HAProxy 和 Keepalived，Keepalived 负责为该服务器抢占 vip（虚拟 ip），抢占到 vip 后，对该主机的访问可以通过原来的 ip访问，也可以直接通过 vip访问。

2). Keepalived 抢占 vip 有优先级，在 keepalived.conf 配置中的 priority 属性决定。但是一般哪台主机上的Keepalived服务先启动就会抢占到vip，即使是slave，只要先启动也能抢到（要注意避免Keepalived的资源抢占问题）。

3). HAProxy 负责将对 vip 的请求分发到 MyCat 集群节点上，起到负载均衡的作用。同时 HAProxy 也能检测到 MyCat 是否存活，HAProxy 只会将请求转发到存活的 MyCat 上。

4). 如果 Keepalived+HAProxy 高可用集群中的一台服务器宕机，集群中另外一台服务器上的 Keepalived会立刻抢占 vip 并接管服务，此时抢占了 vip 的 HAProxy 节点可以继续提供服务。

5). 如果一台 MyCat 服务器宕机，HAPorxy 转发请求时不会转发到宕机的 MyCat 上，所以 MyCat 依然可用。



综上：MyCat 的高可用及负载均衡由 HAProxy 来实现，而 HAProxy 的高可用，由 Keepalived 来实现。



**keepalived介绍:**

Keepalived是一种基于VRRP协议来实现的高可用方案,可以利用其来避免单点故障。 通常有两台甚至多台服务器运行Keepalived，一台为主服务器(Master), 其他为备份服务器, 但是对外表现为一个虚拟IP(VIP), 主服务器会发送特定的消息给备份服务器, 当备份服务器接收不到这个消息时, 即认为主服务器宕机, 备份服务器就会接管虚拟IP, 继续提供服务, 从而保证了整个集群的高可用。
VRRP(虚拟路由冗余协议-Virtual Router Redundancy Protocol)协议是用于实现路由器冗余的协议，VRRP 协议将两台或多台路由器设备虚拟成一个设备，对外提供虚拟路由器 IP(一个或多个)，而在路由器组内部，如果实际拥有这个对外 IP 的路由器如果工作正常的话就是 MASTER，或者是通过算法选举产生。MASTER 实现针对虚拟路由器 IP 的各种网络功能，如 ARP 请求，ICMP，以及数据的转发等；其他设备不拥有该虚拟 IP，状态是 BACKUP，除了接收 MASTER 的VRRP 状态通告信息外，不执行对外的网络功能。当主机失效时，BACKUP 将接管原先 MASTER 的网络功能。VRRP 协议使用多播数据来传输 VRRP 数据，VRRP 数据使用特殊的虚拟源 MAC 地址发送数据而不是自身网卡的 MAC 地址，VRRP 运行时只有 MASTER 路由器定时发送 VRRP 通告信息，表示 MASTER 工作正常以及虚拟路由器 IP(组)，BACKUP 只接收 VRRP 数据，不发送数据，如果一定时间内没有接收到 MASTER 的通告信息，各 BACKUP 将宣告自己成为 MASTER，发送通告信息，重新进行 MASTER 选举状态。







## 1.2 高可用集群搭建

### 1.2.1 部署环境规划

| 名称                      |       IP        | 端口 | 用户名/密码 |
| :------------------------ | :-------------: | :--: | :---------: |
| MySQL Master              | 192.168.192.157 | 3306 | root/itcast |
| MySQL Slave               | 192.168.192.158 | 3306 | root/itcast |
| MyCat节点1                | 192.168.192.157 | 8066 | root/123456 |
| MyCat节点2                | 192.168.192.158 | 8066 | root/123456 |
| HAProxy节点1/keepalived主 | 192.168.192.159 |      |             |
| HAProxy节点2/keepalived备 | 192.168.192.160 |      |             |

![image-20200104153537319](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200104153537320.png) 



### 1.2.2 MySQL主从复制搭建

#### 1.2.2.1 master

准备工作

```shell
#删除之前的库，在master节点上进行操作
docker exec -it M1 bash
mysql -uroot -p123456

show databases;
drop database db01;
drop database db02;
drop database db03;
```



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
#指定同步的数据库
binlog-do-db=db01
binlog-do-db=db02
binlog-do-db=db03
```



2） 执行完毕之后，需要重启Mysql：

```sql
service mysql restart;
```



3） 创建用于主从复制(同步数据)的账户，并且进行授权操作，之前创建过，这步可以省略的：

```sql
grant replication slave on *.* to 'root'@'%' identified by '123456';	

flush privileges;
```



4） 查看master状态：

```sql
show master status;
```

![image-20200104171647471](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200104171647471.png)  

字段含义:

```
File : 从哪个日志文件开始推送日志文件 
Position ： 从哪个位置开始推送日志
Binlog_Do_DB : 指定需要同步的数据库
```



#### 1.2.2.2 slave

1） 在 slave 端配置文件中，配置如下内容：

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
#master_host 主节点的IP;
#master_user用于主从复制的用户名;
#master_log_file在master中执行show master status;查看
#master_log_pos在master中执行show master status;查看
change master to master_host= '192.168.192.157', master_user='root', master_password='123456', master_log_file='mysqlbin.000002', master_log_pos=120;

#执行这个可能报错
ERROR 1198 (HY000): This operation cannot be performed with a running salve;run STOP SLAVE first;

stop slave;
reset master;
然后再执行
change master to...
```

指定当前从库对应的主库的IP地址，用户名，密码，从哪个日志文件开始的那个位置开始同步推送日志。



4） 开启同步操作

```
start slave;
show slave status;
```

![image-20200103144903105](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200103144903105.png)  



5） 停止同步操作

```
stop slave;
```



#### 1.2.2.3 测试验证

```java
#在M1(mysql物理节点)上执行
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



### 1.2.3 MyCat安装配置

```shell
cd /usr/local/mycat/
我这个是docker搭建的直接在本地就行
cd ~/docker_volume/mysql_cluster/conf/
#直接配置读写分离
```



#### 1.2.3.1 schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<schema name="MCLUB" checkSQLschema="true" sqlMaxLimit="100">
		<table name="user" dataNode="dn1" primaryKey="id"/>
	</schema>
	<dataNode name="dn1" dataHost="localhost1" database="db01" />
	<dataHost name="localhost1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" 	
				dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.157:3306" user="root" password="123456">
			<readHost host="hostS1" url="192.168.192.158:3306" user="root" password="123456" />
		</writeHost>
	</dataHost>  
</mycat:schema>
```



#### 1.2.3.2 server.xml

```xml
<user name="root" defaultAccount="true">
    <property name="password">123456</property>
    <property name="schemas">ITCAST</property>
</user>

<user name="test">
    <property name="password">123456</property>
    <property name="schemas">ITCAST</property>
</user>
```



两台MyCat服务, 做相同的配置 ;

```shell
我使用的是docker进行部署的，所以

1.创建挂载卷，将之前的mycat容器挂载的~/docker_volume/conf、~/docker_vloume/logs内容复制到新创建的宿主机挂载卷中
mkdir ~/docker_volume/mycat_1
cd ~/docker_volume/mycat
cp -r conf ../mycat_1
cp -r logs ../mycat_1

2.启动一个新的mycat容器，同时改一下端口号，因为之前的mycat已经占了8066和9066
docker run --privileged=true -p 8067:8066 -p 9067:9066 --name mycat_1 -v /Users/[username]/docker_volume/mycat_1/conf:/usr/local/mycat/conf -v /Users/[username]/docker_volume/mycat_1/logs:/usr/local/mycat/logs --network=mysql_cluster [--ip 192.168.43.116] -d mycat:1.6.7.6

3.修改日志级别，由info --> debug
cd ~/docker_volume/mycat/conf/
vim log4j2.xml
下面有个标签里属性值为info --> 改为debug

4.重新启动mycat
docker exec -it mycat bash
./bin/mycat restart
#mycat_1也是相同的操作

```









### 1.2.4 HAProxy安装配置

#### 1.2.4.1 安装

1). 准备好HAProxy安装包，传到/root目录下

```
haproxy-1.5.16.tar.gz
```



2). 解压到/usr/local/src目录下

```
tar -zxvf haproxy-1.5.16.tar.gz -C /usr/local/src
```



3). 进入解压后的目录，查看内核版本，进行编译

```
cd /usr/local/src/haproxy-1.5.16
uname -r
make TARGET=linux2632 PREFIX=/usr/local/haproxy ARCH=x86_64

# TARGET=linux310，内核版本，使用uname -r查看内核，如：2.6.32-431.el6.x86_64，此时该参数就为linux2632；
# ARCH=x86_64，系统位数；
# PREFIX=/usr/local/haprpxy #/usr/local/haprpxy，为haprpxy安装路径。
```



4). 编译完成后，进行安装

```
make install PREFIX=/usr/local/haproxy
```



5). 安装完成后，创建目录

```
mkdir -p /usr/data/haproxy/
```



6). 创建HAProxy配置文件

vim /usr/local/haproxy/haproxy.conf

```yml
global
	log 127.0.0.1 local0 
	maxconn 4096 
	chroot /usr/local/haproxy 
	pidfile /usr/data/haproxy/haproxy.pid
	uid 99
	gid 99
	daemon
	node mysql-haproxy-01
	description mysql-haproxy-01
defaults
	log global
	mode tcp
	option abortonclose
	option redispatch
	retries 3
	maxconn 2000
	timeout connect 50000ms
	timeout client 50000ms
	timeout server 50000ms
listen proxy_status
	bind 0.0.0.0:48066
		mode tcp
		balance roundrobin
		server mycat_1 192.168.192.157:8066 check
		server mycat_2 192.168.192.158:8066 check
frontend admin_stats
	bind 0.0.0.0:8888
		mode http
		stats enable
		option httplog
		maxconn 10
		stats refresh 30s
		stats uri /admin
		stats auth admin:123123
		stats hide-version
		stats admin if TRUE
```



**内容解析如下** : 

```yml
#global 配置中的参数为进程级别的参数，通常与其运行的操作系统有关
global
	#定义全局的syslog服务器, 最多可定义2个; local0 是日志设备, 对应于/etc/rsyslog.conf中的配置 , 默认收集info级别日志
	log 127.0.0.1 local0 
	#log 127.0.0.1 local1 notice
	#log loghost local0 info
	#设定每个haproxy进程所接受的最大并发连接数 ;
	maxconn 4096 
	#修改HAproxy工作目录至指定的目录并在放弃权限之前执行chroot操作, 可以提升haproxy的安全级别
	chroot /usr/local/haproxy 
	#进程ID保存文件
	pidfile /usr/data/haproxy/haproxy.pid
	#指定用户ID
	uid 99
	#指定组ID
	gid 99
	#设置HAproxy以守护进程方式运行
	daemon
	#debug
	#quiet
	node mysql-haproxy-01  ## 定义当前节点的名称，用于 HA 场景中多 haproxy 进程共享同一个 IP 地址时
	description mysql-haproxy-01 ## 当前实例的描述信息
	
#defaults：用于为所有其他配置段提供默认参数，这默认配置参数可由下一个"defaults"所重新设定
defaults
	#继承global中的log定义
	log global
	#所使用的处理模式(tcp:四层 , http:七层, health:状态检查,只返回OK)
	### tcp: 实例运行于纯 tcp 模式，在客户端和服务器端之间将建立一个全双工的连接，且不会对 7 层报文做任何类型的检查，此为默认模式
	### http:实例运行于 http 模式，客户端请求在转发至后端服务器之前将被深度分析，所有不与 RFC 模式兼容的请求都会被拒绝
	### health：实例运行于 health 模式，其对入站请求仅响应“OK”信息并关闭连接，且不会记录任何日志信息 ，此模式将用于相应外部组件的监控状态检测请求
	mode tcp
	#当服务器负载很高的时候，自动结束掉当前队列处理时间比较长的连接
	option abortonclose
		
	#当使用了cookie时，haproxy将会将请求的后端服务器的serverID插入到cookie中，以保证会话的session持久性，而此时，后端服务器宕机，但是客户端的cookie不会刷新，设置此参数，将会将客户请求强制定向到另外一个后端server上，以保证服务的正常。
	option redispatch
	retries 3
	# 前端的最大并发连接数（默认为 2000）
	maxconn 2000
	# 连接超时(默认是毫秒,单位可以设置 us,ms,s,m,h,d)
	timeout connect 5000
	# 客户端超时时间
	timeout client 50000
	# 服务器超时时间
	timeout server 50000

#listen: 用于定义通过关联“前端”和“后端”一个完整的代理，通常只对 TCP 流量有用
listen proxy_status
	bind 0.0.0.0:48066 # 绑定端口
		mode tcp
		balance roundrobin # 定义负载均衡算法，可用于"defaults"、"listen"和"backend"中,默认为轮询
		#格式: server <name> <address> [:[port]] [param*]
		# weight : 权重，默认为 1，最大值为 256，0 表示不参与负载均衡
        # backup : 设定为备用服务器，仅在负载均衡场景中的其他 server 均不可以启用此 server
        # check  : 启动对此 server 执行监控状态检查，其可以借助于额外的其他参数完成更精细的设定
        # inter  : 设定监控状态检查的时间间隔，单位为毫秒，默认为 2000，也可以使用 fastinter 和 downinter 来根据服务器端专题优化此事件延迟
        # rise   : 设置 server 从离线状态转换至正常状态需要检查的次数（不设置的情况下，默认值为 2）
        # fall   : 设置 server 从正常状态转换至离线状态需要检查的次数（不设置的情况下，默认值为 3）
        # cookie : 为指定 server 设定 cookie 值，此处指定的值将会在请求入站时被检查，第一次为此值挑选的 server 将会被后续的请求所选中，其目的在于实现持久连接的功能
        # maxconn: 指定此服务器接受的最大并发连接数，如果发往此服务器的连接数目高于此处指定的值，其将被放置于请求队列，以等待其他连接被释放
		server mycat_1 192.168.192.157:8066 check inter 10s
		server mycat_2 192.168.192.158:8066 check inter 10s

# 用来匹配接收客户所请求的域名，uri等，并针对不同的匹配，做不同的请求处理
# HAProxy 的状态信息统计页面
frontend admin_stats
	bind 0.0.0.0:8888
		mode http
		stats enable
		option httplog
		maxconn 10
		stats refresh 30s
		stats uri /admin
		stats auth admin:123123
		stats hide-version
		stats admin if TRUE
```



HAProxy的负载均衡策略: 

| 策略               | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| roundrobin         | 表示简单的轮循，即客户端每访问一次，请求轮循跳转到后端不同的节点机器上 |
| static-rr          | 基于权重轮循，根据权重轮循调度到后端不同节点                 |
| leastconn          | 加权最少连接，表示最少连接者优先处理                         |
| source             | 表示根据请求源IP，这个跟Nginx的IP_hash机制类似，使用其作为解决session问题的一种方法 |
| uri                | 表示根据请求的URL，调度到后端不同的服务器                    |
| url_param          | 表示根据请求的URL参数来进行调度                              |
| hdr（name）        | 表示根据HTTP请求头来锁定每一次HTTP请求                       |
| rdp-cookie（name） | 表示根据cookie（name）来锁定并哈希每一次TCP请求              |





#### 1.2.4.2 启动访问

1). 启动HAProxy

```
/usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.conf
```



2). 查看HAProxy进程

```
ps -ef|grep haproxy
```



3). 访问

http://192.168.192.162:8888/admin



界面: 

![image-20200202214408231](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200202214408231.png)  





### 1.2.5 Docker HAProxy

#### 1.2.5.1 准备image

```shell
拉取镜像
docker pull haproxy:1.9.6

创建目录，用于存放配置文件
mkdir /User/[username]/docker_volume/haproxy_1/conf
mkdir /User/[username]/docker_volume/haproxy_2/conf

```



#### 1.2.5.2 配置HAProxy

编写配置文件
vim /Users/[username]/docker_volume/haproxy_1/conf/haproxy.cfg

**注意：换行和缩进**

```yml
global
    daemon
    # nbproc 1
    # pidfile /var/run/haproxy.pid
    # 工作目录
    #修改HAproxy工作目录至指定的目录并在放弃权限之前执行chroot操作, 可以提升haproxy的安全级别
    chroot /usr/local/etc/haproxy
    #设定每个haproxy进程所接受的最大并发连接数 ;
    maxconn 4096 
    #进程ID保存文件
    # pidfile /usr/local/etc/haproxy/haproxy.pid
    #指定用户ID
    uid 99
    #指定组ID
    gid 99
    #设置HAproxy以守护进程方式运行
    daemon
    #debug
    #quiet
    node mysql-haproxy-01  ## 定义当前节点的名称，用于 HA 场景中多 haproxy 进程共享同一个 IP 地址时
    description mysql-haproxy-01 ## 当前实例的描述信息
defaults
    log global
    mode tcp
    # 当服务器负载很高的时候，自动结束掉当前队列处理时间比较长的连接
    option abortonclose
    #当使用了cookie时，haproxy将会将请求的后端服务器的serverID插入到cookie中，以保证会话的session持久性，而此时，后端服务器宕机，但是客户端的cookie不会刷新，设置此参数，将会将客户请求强制定向到另外一个后端server上，以保证服务的正常。
	  option redispatch
    retries 3
    timeout connect 3000
    timeout server 50000
    timeout client 50000
listen mysql-cluster
    bind 0.0.0.0:48067
        mode tcp
        #option mysql-check user haproxy_check  (This is not needed as for Layer 4 balancing)
        option tcp-check
        balance roundrobin
        # The below nodes would be hit on 1:1 ratio. If you want it to be 1:2 then add 'weight 2' just after the line.
        server mysql1 192.168.43.116:8066 check
        server mysql2 192.168.43.116:8067 check
# Enable cluster status
listen mysql-clusterstats
    bind 0.0.0.0:8888
      mode http
			stats enable
			option httplog
			maxconn 10
			stats refresh 30s
			stats uri /admin
			stats auth admin:123123
			stats hide-version
			stats admin if TRUE
```



#### 1.2.5.3 启动容器

```shell
docker run -d --name haproxy_1 -p 8888:8888  -p 48067:48067   -v /Users/[username]/docker_volume/haproxy_1/conf:/usr/local/etc/haproxy:ro --network mysql_cluster --privileged=true haproxy:1.9.6
```

采用host模式进行创建，使用宿主机的网卡，否则KP创建的VIP是容器内VIP而不是容器外VIP

```shell
#-v 中的参数:ro表示read only，宿主文件为只读。如果不加此参数默认为rw，即允许容器对宿主文件的读写
#一定要添加--privileged参数,使用该参数，container内的root拥有真正的root权限。
#否则，container内的root只是外部的一个普通用户权限（无法创建网卡）
docker run -d --name cluster-rabbit-haproxy --privileged --net host -v /home/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy

docker run -d --name haproxy_1 -v /Users/[username]/docker_volume/haproxy_1/conf:/usr/local/etc/haproxy:ro --privileged=true --net host haproxy:1.9.6
```





第二台 HAProxy

```yml
global
    daemon
    # nbproc 1
    # pidfile /var/run/haproxy.pid
    # 工作目录
    #修改HAproxy工作目录至指定的目录并在放弃权限之前执行chroot操作, 可以提升haproxy的安全级别
    chroot /usr/local/etc/haproxy
    #设定每个haproxy进程所接受的最大并发连接数 ;
    maxconn 4096 
    #进程ID保存文件
    # pidfile /usr/local/etc/haproxy/haproxy.pid
    #指定用户ID
    uid 99
    #指定组ID
    gid 99
    #设置HAproxy以守护进程方式运行
    daemon
    #debug
    #quiet
    node mysql-haproxy-02  ## 定义当前节点的名称，用于 HA 场景中多 haproxy 进程共享同一个 IP 地址时
    description mysql-haproxy-02 ## 当前实例的描述信息
defaults
    log global
    mode tcp
    # 当服务器负载很高的时候，自动结束掉当前队列处理时间比较长的连接
    option abortonclose
    #当使用了cookie时，haproxy将会将请求的后端服务器的serverID插入到cookie中，以保证会话的session持久性，而此时，后端服务器宕机，但是客户端的cookie不会刷新，设置此参数，将会将客户请求强制定向到另外一个后端server上，以保证服务的正常。
	  option redispatch
    retries 3
    timeout connect 3000
    timeout server 50000
    timeout client 50000
listen mysql-cluster
    bind 0.0.0.0:48067
        mode tcp
        #option mysql-check user haproxy_check  (This is not needed as for Layer 4 balancing)
        option tcp-check
        balance roundrobin
        # The below nodes would be hit on 1:1 ratio. If you want it to be 1:2 then add 'weight 2' just after the line.
        server mysql1 192.168.43.116:8066 check
        server mysql2 192.168.43.116:8067 check
# Enable cluster status
listen mysql-clusterstats
    bind 0.0.0.0:8888
      mode http
			stats enable
			option httplog
			maxconn 10
			stats refresh 30s
			stats uri /admin
			stats auth admin:123123
			stats hide-version
			stats admin if TRUE
```

启动容器

```shell
docker run -d --name haproxy_2 -p 8889:8888  -p 48068:48067   -v /Users/liuxiangren/docker_volume/haproxy_2/conf:/usr/local/etc/haproxy:ro --network mysql_cluster --privileged=true haproxy:1.9.6
```

采用host模式进行创建，使用宿主机的网卡，否则KP创建的VIP是容器内VIP而不是容器外VIP

```shell
#-v 中的参数:ro表示read only，宿主文件为只读。如果不加此参数默认为rw，即允许容器对宿主文件的读写
#一定要添加--privileged参数,使用该参数，container内的root拥有真正的root权限。
#否则，container内的root只是外部的一个普通用户权限（无法创建网卡）
docker run -d --name cluster-rabbit-haproxy --privileged --net host -v /home/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro haproxy

docker run -d --name haproxy_2 -p 8889:8888  -p 48068:48067   -v /Users/liuxiangren/docker_volume/haproxy_2/conf:/usr/local/etc/haproxy:ro --privileged=true haproxy:1.9.6
```





访问

http://192.168.43.116:8888/admin

输入上面配置的用户名和密码

![image-20210627171412862](/Users/liuxiangren/mysql-learning/mycat-img/haproxy-web-manager.png)

#### 1.2.5.4 使用HAProxy

```shell
#同样的使用HAProxy和mycat相同的，这个48067是 HAProxy_1 的服务端口号
mysql -h 192.168.43.116 -P 48067 -uroot -p

show databases;
+----------+
| DATABASE |
+----------+
| MCLUB    |
+----------+
1 row in set (0.02 sec)

select * from USER;
+----+--------+------+
| ID | NAME   | SEX  |
+----+--------+------+
|  1 | 小刘   | 1    |
+----+--------+------+
1 row in set (0.04 sec)

#很nice的
```





### 1.2.6 Keepalived安装配置

参考文章：

docker下用keepalived+Haproxy实现高可用负载均衡集群：https://blog.csdn.net/qq_21108311/article/details/82973763

docker 部署高可用 HAProxy Mysql 双主方案：https://blog.csdn.net/warrior_0319/article/details/80805030

docker下使用HAProxy：https://blog.csdn.net/inthat/article/details/88928749

Docker搭建多机多节点haproxy+keepalived负载均衡的高可用RabbitMQ集群:https://www.jianshu.com/p/42f6d3e9f55b

上面我们安装的HAProxy是单节点的，也就是说如果HAProxy_1挂了，HAProxy是不会自动顶上来的，如果没有这个KeepAlive你连接哪个HAProxy合适？连接哪个都不合适嘛，还是要通过KeepAlive虚拟出来的IP，然后用户去请求这个VIP节点，由VIP节点进行请求的转发，下图，这两个KeepAlive节点会共同虚拟出这个192.168.192.200这个VIP，那么以后用户就只要访问这一个VIP192.168.192.200这个地址就行了，如果159的KeepAlive挂了，其实这两个KeepAlive会发送心跳，159的KeepAlive挂了的话，就获得不到反映了，那就自动进行下一个存活的Haproxy进行绑定。

```markdown
注意！
有一种情况是，159上HAProxy出现问题，但KeepAlive在一次心跳中，还是存活的状态，给160发送了应答信息，但是用户请求来了访问了159上的HAProxy，那么就会产生问题了！
我们要在KeepAlived中嵌入一个脚本，这个脚本就是来监测HAProxy的(代表这个下图中的check蓝色箭头)，脚本会监测这个HAProxy，如果脚本监测到HAProxy挂掉，会自动重新启动该HAProxy，如果启动命令执行后，监测HAProxy服务还是不存在，则这个脚本会干掉KeepAlive
```







![image-20200203010953404](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200203010953404.png) 



#### 1.2.6.1 安装配置

keepalived官方下载地址：https://keepalived.org/download.html

1). 上传安装包到Linux

```
alt + p --------> put D:/tmp/keepalived-1.4.5.tar.gz
```



2). 解压安装包到目录 /usr/local/src

```
tar -zxvf keepalived-1.4.5.tar.gz -C /usr/local/src
```



3). 安装依赖插件

```
yum install -y gcc openssl-devel popt-devel

#安装gcc
apt-get  install  build-essential

安装openssl-dev 报错E: Unable to locate package openssl-dev
sudo apt-get install openssl
sudo apt-get install libssl-dev
RedHat、centos才是openssl-devel

E: Unable to locate package popt-devel
apt-get install aptitude
aptitude install popt-devel
```

```shell
apt-get update:更新安装列表 
apt-get upgrade:升级软件 
apt-get install software_name :安装软件 
apt-get --purge remove  software_name :卸载软件及其配置 
apt-get autoremove software_name:卸载软件及其依赖的安装包 
```



4). 进入解压后的目录，进行配置，进行编译

```
 cd /usr/local/src/keepalived-1.4.5
 
 ./configure --prefix=/usr/local/keepalived
```



5). 进行编译，完成后进行安装

```shell
make && make install
```



6). 运行前配置

```
cp /usr/local/src/keepalived-2.0.20/keepalived/etc/init.d/keepalived /etc/init.d/
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/src/keepalived-2.0.20/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
```



7). 修改配置文件 /etc/keepalived/keepalived.conf

Master: 

```
global_defs {
	notification_email {
		javadct@163.com
	}
	notification_email_from keepalived@showjoy.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	router_id haproxy01
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}

vrrp_script chk_haproxy {
	script "/etc/keepalived/haproxy_check.sh"
	interval 2
	weight 2
}

vrrp_instance VI_1 {
	#主机配MASTER，备机配BACKUP
	state MASTER
	#所在机器网卡
	interface eth1
	virtual_router_id 51
	#数值越大优先级越高
	priority 120
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	## 将 track_script 块加入 instance 配置块
    track_script {
    	chk_haproxy ## 检查 HAProxy 服务是否存活
    }
	virtual_ipaddress {
		#虚拟IP
		192.168.192.200
	}
}
```



BackUP:

```conf
global_defs {
	notification_email {
		javadct@163.com
	}
	notification_email_from keepalived@showjoy.com
	smtp_server 127.0.0.1
	smtp_connect_timeout 30
	#标识本节点
	router_id haproxy02
	vrrp_skip_check_adv_addr
	vrrp_garp_interval 0
	vrrp_gna_interval 0
}

# keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级
vrrp_script chk_haproxy {
	# 检测 haproxy 状态的脚本路径
	script "/etc/keepalived/haproxy_check.sh"
	#检测时间间隔
	interval 2
	#如果条件成立，权重+2
	weight 2
}

vrrp_instance VI_1 {
	#主机配MASTER，备机配BACKUP
	state BACKUP
	#所在机器网卡
	interface eth1
	virtual_router_id 51
	#数值越大优先级越高
	priority 100
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	## 将 track_script 块加入 instance 配置块
    track_script {
    	chk_haproxy ## 检查 HAProxy 服务是否存活
    }
	virtual_ipaddress {
		#虚拟IP
		192.168.192.200
	}
}
```



8). 编写检测haproxy的shell脚本 haproxy_check.sh

```shell
#!/bin/bash

A=`ps -C haproxy --no-header | wc -l`

if [ $A -eq 0 ];then

  /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.conf

  echo "haproxy restart ..." &> /dev/null

  sleep 1

  if [ `ps -C haproxy --no-header | wc -l` -eq 0 ];then

    /etc/init.d/keepalived stop

    echo "stop keepalived" &> /dev/null

  fi

fi
```



#### 1.2.6.2 启动测试

1). 启动Keepalived

```
service keepalived start
```

2). 登录验证

```
mysql -uroot -p123456 -h 192.168.192.200 -P 48066
```



![image-20200202193227448](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200202193227448.png) 







### 1.2.7 Docker KeppAlived

参考文章：https://www.jianshu.com/p/42f6d3e9f55b

https://www.jb51.cc/docker/990294.html

主要参考：https://blog.csdn.net/wwxuelei/article/details/89471839

#### 1.2.7.1 准备工作

```shell
1、前往官网下载所需版本https://www.keepalived.org/
docker cp [/path/to/keepalive] haproxy_1:/usr/local/src

解压
tar -zxvf keepalived-2.0.15.tar
 tar包压缩的时候用cvf参数，解压的时候用xvf参数
或压缩的时候用czvf参数，解压的时候用xzvf参数


2、编译扮装
1)检查环境
./configure --prefix=/usr/local/keepalived

报错1：configure: error: in `/usr/local/src/keepalived-2.0.20':
configure: error: no acceptable C compiler found in $PATH
apt-get update
apt-get  install  build-essential  


报错2：Can not include OpenSSL headers files
apt-get -y install openssl libssl-dev
注意：redhat和centos中是需要安装openssl和openssl-devel的，在ubuntu中，openssl-devel被libssl-dev所代替，安装libssl-dev即可
再重新检查环境~，ok，没问题，警告忽视

2)编译、编译安装
make && make install

3、编辑配置文件
v1.0
cp /usr/local/keepalived-2.0.15/etc/keepalived/keepalived.conf /etc/keepalived/  #复制配置文件
cp /usr/local/keepalived-2.0.15/sbin/keepalived /usr/local/sbin/
cp /usr/local/keepalived-2.0.15/etc/rc.d/init.d/keepalived /etc/init.d/  #复制服务启动文件
chmod +x /etc/init.d/keepalived
v1.1
#使keepalived命令能直接使用
cp /usr/local/keepalived/sbin/keepalived /sbin/
 # 创建配置文件并修改
mkdir -p /etc/keepalived
#把宿主机写好的keepalived.conf和check_haproxy.sh通过docker cp命令复制到haproxy_1机器中


#以下信息可忽略，记录用
apt-get update
#安装ifconfig、ping、vim还有ps(早知道这样，之前就应该挂载个目录卷)
apt-get install net-tools
apt-get install iputils-ping
apt-get -y install openssl libssl-dev

#创建keepalive配置文件
vim 竟然还是不能用，这
唉，在宿主机创建出来，复制到本机算嘞
```

#### 1.2.7.2 配置KeepAlived

```json
#keepalived配置文件
global_defs {
    router_id haproxy01                 #路由ID, 主备的ID不能相同
    notification_email {
        xxx@qq.com
    }
    notification_email_from keepalived@showjoy.com
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    vrrp_skip_check_adv_addr
    #在keepalived的服务器上配合使用nginx或haproxy时，需要把这一项注掉，否则VIP ping不通，80端口也无法正常访问
    # vrrp_strict 
    vrrp_garp_interval 0
    vrrp_gna_interval 0
}

#自定义监控脚本
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER #Master为主机，备机设为BACKUP
    interface en0        #指定网卡（宿主机真实网卡，ip a查看）
    virtual_router_id 1   #在同一局或网内如果有多个keepalived 的话 virtuall_router_id 44 (不能相同，但同一对，是一定相同)
    priority 100            #优先级，BACKUP机器上的优先级要小于这个值
    advert_int 1            #设置主备之间的检查时间，单位为s
    authentication {        #定义验证类型和密码，主备需相同
            auth_type PASS
            auth_pass 1111
    }
    track_script {
            chk_haproxy     #ha存活监控脚本
    }
    virtual_ipaddress {     #VIP地址，可为多个。如果有需要可以部署双机双VIP
        172.17.0.200
    }
}
```

```markdown
#Mac OS查看自己的网卡
ifconfig
找到带自己公网ip就是啦
```

复制到haproxy_1的容器里面

```shell
docker cp /Users/[username]/docker_volume/keepalived.conf haproxy_1:/etc/keepalived/ 
```

#### 1.2.7.3 keepalive引用脚本

先在宿主机中进行创建，然后复制到docker容器中

```shell
#!/bin/bash
if [ $(ps -C haproxy --no-header | wc -l) -eq 0 ];then
        haproxy -f /usr/local/etc/haproxy/haproxy.cfg
fi
sleep 2
if [ $(ps -C haproxy --no-header | wc -l) -eq 0 ];then
        #service keepalived stop
        /etc/init.d/keepalived stop
fi
```

```shell
chmod +x check_haproxy.sh #给个执行权限
```



复制到haproxy_1容器中

```shell
docker cp /Users/[username]/docker_volume/check_haproxy.sh haproxy_1:/etc/keepalived/check_haproxy.sh

docker cp /Users/liuxiangren/docker_volume/check_haproxy.sh haproxy_1:/etc/keepalived/
docker cp /Users/liuxiangren/docker_volume/keepalived.conf haproxy_1:/etc/keepalived/
```

设置服务启动

```shell
一、复制服务脚本
cp /usr/local/src/keepalived-2.0.20/keepalived/etc/init.d/keepalived /etc/init.d/
#注意：上面的服务启动脚本是在源文件目录，而不是安装目录（/usr/local/keepalived）下，这是2.0之后的变化。

二、修正相关配置问题
vim /etc/init.d/keepalived
#下图三个地方有问题
```

![image-20210628121157817](/Users/liuxiangren/mysql-learning/mycat-img/keepalived-problems.png)

```shell
图中1:  由于 ubuntu下没有 /etc/rc.d/init.d/functions，需要为其建立软链接

mkdir -p /etc/rc.d/init.d
ln -s /lib/lsb/init-functions /etc/rc.d/init.d/functions

图中2：拷贝相应文件的源配置文件
注释内容有介绍 ,这个源配置文件（在里面设置keepalived启动参数）

mkdir /etc/sysconfig
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  

图中3：安装daemon,并修改命令
 1. 安装 daemon:  apt install daemon
 2. 图中3的命令 修改为daemon -- keepalived ${KEEPALIVED_OPTIONS}  #加了一个“--”

说明：命令里的变量 ${KEEPALIVED_OPTIONS} 是在/etc/sysconfig/keepalived里设置的
内容是：KEEPALIVED_OPTIONS="-D"。
这里的-D 代表记录详细日志。那么命令 daemon keepalived ${KEEPALIVED_OPTIONS}的结果是
daemon keepalived -D
这个命令是有问题的，其中的-D本来是给keepalived用的，但这样组合后被认为是daemon命令的参数。这会导致服务不能启动。  如果不修改，会提示启动失败，但却不输出具体信息。但可以通过查看 /var/log/syslog  找到错误信息
执行  daemon --help， 可以看到帮助信息
可以看出， daemon命令的-D参数是需要一个path参数的，所以会出现系统日志里的错误。
由 usage: daemon [options] [--] [cm arg...]，可知正确的命令格式应该是:daemon -- keepalived -D
所以上面力中标示的第3处，应该修改为 daemon -- keepalived ${KEEPALIVED_OPTIONS}
注意：每次修改/etc/init.d/keepalived后，需要重新运行 systemctl daemon-reload 重新加载服务脚本

```



#### 1.2.7.4 启动keepalived

参考启动：https://blog.csdn.net/wwxuelei/article/details/89471839

```shell
#注意
systemctl daemon-reload
bash: systemctl: command not found

#然后重新 daemon-reload
systemctl daemon-reload
又报错了：
Failed to connect to bus: No such file or directory
解决方案1：
参考：https://blog.csdn.net/rznice/article/details/52253114，说是在启动的时候后面加上/usr/sbin/init，例如：docker run -tid --name hadoopbase centos/hadoopbase:v001 /usr/sbin/init

解决方案2：
参考：https://stackoverflow.com/questions/49285658/how-to-solve-docker-issue-failed-to-connect-to-bus-no-such-file-or-directory

apt-get install wget

wget https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py -O /usr/local/bin/systemctl

#然后重新 daemon-reload
又报错了兄弟！
systemctl daemon-reload
bash: /usr/local/bin/systemctl: /usr/bin/python2: bad interpreter: No such file or directory

pat-get updata
apt-get install yum
正准备安装python2呢，我看网上人说，安装yum自带python2，啧啧啧
#然后重新 daemon-reload
啧啧，竟然没问题了

apt-get install --reinstall systemd
```





```shell
一、修改其权限并开机启动
修改权限：chmod 755 /etc/init.d/keepalived
加为系统服务：chkconfig –add keepalived
开机启动：chkconfig keepalived on
查看开机启动的服务：chkconfig –list

二、备注：keepalived服务控制
systemctl enable keepalived.service #设置开机自动启动
systemctl disable keepalived.service #取消开机自动启动
systemctl start keepalived.service #启动服务
systemctl restart keepalived.service #重启服务
systemctl stop keepalived.service #停止服务
systemctl status keepalived.service #查看服务状态


service keepalived start
touch: cannot touch '/var/lock/subsys/keepalived': No such file or directory
问题方案：
参考地址：https://blog.csdn.net/weixin_30166297/article/details/78040444?locationNum=10&fps=1
错误原因：
不能在创建/var/lock/subsys/keepalived文件。正常Ubuntu或者CentOS系统/var/lock下都有/subsys文件夹。这个文件夹下创建的文件是正在运行的程序或者服务的标识符，例如keepalived文件如果存在，证明keepalived正在运行。
mkdir /var/lock/subsys
chmod +x /var/lock/subsys

ip a
bash:ip command not found

service keepalived stop
/etc/init.d/keepalived: 12: .: Can't open /etc/rc.d/init.d/functions

启动文件中某些文件不存在，需要手动链接一下
ln -s /lib/lsb/init-functions /etc/init.d/functions

mkdir /etc/rc.d

ln -s /etc/init.d /etc/rc.d/

cp /usr/local/src/keepalived-2.0.20/keepalived/etc/sysconfig/keepalived /etc/sysconfig/

```

#### 1.2.7.5 检查启动情况

```shell
ping 172.17.0.200
PING 172.17.0.200 (172.17.0.200) 56(84) bytes of data.
From 172.17.0.2 icmp_seq=1 Destination Host Unreachable
From 172.17.0.2 icmp_seq=2 Destination Host Unreachable
From 172.17.0.2 icmp_seq=3 Destination Host Unreachable
```

#### 1.2.7.6 最新Docker安装Keepalived方式

```shell
参考文章：https://blog.csdn.net/mhdp820121/article/details/84588126

apt-get update

apt-get install libssl-dev

apt-get install openssl

apt-get install libpopt-dev

apt-get install keepalived

docker cp ~/docker_volume/keepalived.conf haproxy_1:/etc/keepalived/ 
docker cp ~/docker_volume/check_haproxy.sh haproxy_1:/etc/keepalived/

service keepalived start
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1000
    link/tunnel6 :: brd ::
157: eth0@if158: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:14:00:09 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.9/16 brd 172.20.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 172.17.0.200/32 scope global eth0
       valid_lft forever preferred_lft forever
       
apt install iputils-ping

```

```shell
#mac os 宿主机访问docker container，参考

https://www.cnblogs.com/lucky9322/p/13648282.html(重点参照，使用已成功解决)

https://www.jianshu.com/p/0e7aab7cab89（这个相对简单些）

```

记录下mac os ping通 docker 容器

```shell
1.下载tunnelblick
tunnelblick的地址 https://github.com/Tunnelblick/Tunnelblick/releases
然后克隆docker-mac-network项目
git clone https://github.com/wojas/docker-mac-network.git
2.就在当前执行git命令路径下
cd docker-mac-network

3.vim helpers/run.sh

4.修改文件里的ip和子网掩码，改为你容器的(在容器里ifconfig查询）
route -n #查询ip地址和子网掩码
bash: route: command not found
apt-get install net-tools

route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.20.0.1      0.0.0.0         UG    0      0        0 eth0
172.20.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```

![img](/Users/liuxiangren/mysql-learning/mycat-img/docker-mac-network.png)

```shell
5.在刚刚克隆下的目录中执行 ，注意因为是后台执行所以你要等看到当前目录生成docker-for-mac.ovpn这个文件为止
docker-compose up -d

6.在docker-for-mac.ovpn文件中添加一行
comp-lzo yes

7.双击docker-for-mac.ovpn这个文件，然后跟着tunnelblick提示一直点就行了
注意：出现了提示说comp-lzo yes这个之后会被废弃，不用管直接忽略
我的会出现一个掩码的提示警告，直接忽略

8.测试,ping 容器ip
ping 172.20.0.200 
PING 172.20.0.200 (172.20.0.200): 56 data bytes
64 bytes from 172.20.0.200: icmp_seq=0 ttl=63 time=2.796 ms
64 bytes from 172.20.0.200: icmp_seq=1 ttl=63 time=3.560 ms
64 bytes from 172.20.0.200: icmp_seq=2 ttl=63 time=3.613 ms
64 bytes from 172.20.0.200: icmp_seq=3 ttl=63 time=3.243 ms
64 bytes from 172.20.0.200: icmp_seq=4 ttl=63 time=2.484 ms
64 bytes from 172.20.0.200: icmp_seq=5 ttl=63 time=2.514 ms

9.重新生成
如果要重新生成的话，把目录下这些文件删除，然后再执行一次 docker-compose up -d
rm -rf conf/*
rm -rf docker-for-mac.ovpn
```

#### 1.2.7.8 通过VIP进行连接测试

```shell
#在宿主机中进行连接测试
mysql -h172.20.0.200  -P48067 -uroot -p123456
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1881
Server version: 5.6.29-mycat-1.6.7.6-release-20201126013625 MyCat Server (OpenCloudDB)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+----------+
| DATABASE |
+----------+
| MCLUB    |
+----------+
1 row in set (0.06 sec)
```

#### 1.2.7.9 请求路由验证

当haproxy_1被关闭，

```shell
#宿主机中使用
mysql -h 172.20.0.200 -P 48067 -u root -p
mysql -h 172.20.0.200 -P 48068 -u root -p #这个就不行了，通过VIP只能访问一台服务
#还是访问的到的，但是下面的指令就访问不到数据库了
mysql -h 192.168.43.116 -P 48067 -u root -p
#那上面这个是访问谁的呢，其实是haproxy_2上的服务
```



## 2. MyCat架构剖析

### 2.1 MyCat总体架构介绍

#### 2.1.1 源码下载及导入

![image-20200202220149279](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200202220149279.png) 

导入Idea

![image-20200202220220682](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200202220220682.png) 



#### 2.1.2 总体架构

MyCat在逻辑上由几个模块组成: 通信协议、路由解析、结果集处理、数据库连接、监控等模块。如图所示： 

![image-20200107230122662](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200107230122662.png) 

1). 通信协议模块： 通信协议模块承担底层的收发数据、线程回调处理工作， MyCat通信协议默认采用Reactor模式，在协议层采用MySQL协议；

2). 路由解析模块: 负责对传入的SQL语句进行语法解析, 解析语句的条件、类型、关键字等，并进行优化；

3). SQL执行模块: 负责从连接池中获取连接, 再根据路由解析的结果, 把SQL语句分发到相应的节点执行;

4). 数据库连接模块: 负责创建、管理、维护后端的连接池。为减少每次建立数据库连接的开销，数据库使用连接池机制对连接声明周期进行管理；

5). 结果集处理模块: 负责对跨分片的查询结果进行汇聚、排序、截取等；

6). 监控管理模块: 负责MyCat的连接、内存等资源进行监控和管理。监控主要通过管理指令及监控服务展现一些监控数据； 管理则主要通过轮询事件来检测和释放不适用的资源；





#### 2.1.3 总体执行流程

![image-20200107233001248](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200107233001248.png) 





### 2.2 MyCat网络I/O架构及实现

#### 2.2.1 BIO、NIO与AIO

1). BIO

BIO(同步阻塞I/O) 通常由一个单独的Acceptor线程负责监听客户端的连接, 接收到客户端的连接请求后, 会为每个客户端创建一个新的线程进行处理, 处理完成之后, 再给客户端返回结果, 销毁线程 。

每个客户端请求接入时， 都需要开启一个线程进行处理， 一个线程只能处理一个客户端连接。 当客户端变多时，会创建大量的处理线程， 每个线程都需要分配栈空间和CPU， 并且频繁的线程上下文切换也会造成性能的浪费。所以该模式， 无法满足高性能、高并发接入的需求。



2). NIO

NIO(同步非阻塞I/O)基于Reactor模式作为底层通信模型，Reactor模式可以将事件驱动的应用进行事件分派, 将客户端发送过来的服务请求分派给合适的处理类(handler)。当Socket有流可读或可写入Socket时, 操作系统会通知相应的应用程序进行处理, 应用程序再将流读取到缓冲区或写入操作系统。 这时已经不是一个连接对应一个处理线程了， 而是一个有效的请求对应一个线程， 当没有数据时， 就没有工作线程来处理。

NIO 的最大优点体现在线程轮询访问Selector, 当read或write到达时则处理, 未到达时则继续轮询。



3). AIO

AIO，全程 Asynchronous IO(异步非阻塞的IO), 是一种非阻塞异步的通信模式。在NIO的基础上引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现。AIO中客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。

AIO与NIO的主要区别在于回调与轮询, 客户端不需要关注服务处理事件是否完成, 也不需要轮询, 只需要关注自己的回调函数。



#### 2.2.2 通信架构

在MyCat中实现了NIO与AIO两种I/O模式, 可以通过配置文件server.xml进行指定 : 

```xml
<property name="usingAIO">1</property>
```

usingAIO为1代表使用AIO模型 , 为0表示使用NIO模型;



**MyCat的AIO架构**

![image-20200108103954458](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108103954458.png) 

1). MyCatStartUp是整个MyCat服务启动的入口;

2). 在获取到MyCat的home目录后, 把主要的任务交给MyCatServer , 并调用其startup方法;

3). 初始化系统配置, 获取配置文件中的usingAIO的配置, 如果配置为1, 说明使用AIO模型 , 进入到AIO的分支, 并创建两个连接, 一个是管理后台连接(9066), 一个server的连接(8066);

4). 进入AIO分支 , 主要有AIOAcceptor接收客户端请求, 绑定端口, 创建服务端的异步Socket ;在accept方法中完成两件事: ①. FrontedConnection的创建, 这是前段连接的关键; ②. register注册事件, MySQL协议握手包就在此时发送;

![image-20200108111012502](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108111012502.png) 



**MyCat的NIO架构**

如果设置的usingAIO为0 ,那么将走NIOAcceptor通道 , 流程如下: 

![image-20200108111153230](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108111153230.png) 

1). 如果走NIO分支 , 将首先创建NIOAcceptor对象, 并调用其start方法;

2). NIOAcceptor 负责处理Accept事件, 服务端接收客户端的连接事件, 就是MyCat作为服务端去处理前端业务程序发过来的连接请求, 建立链接后, 调用NIOAcceptor的 NIOReactor.postRegister方法进行注册（并没有注解注册， 而是放入缓冲队列， 避免加锁的竞争）。 

NIOAcceptor的accept方法 ： 

![image-20200108112521438](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108112521438.png) 

NIOReactor的postRegister方法： 

![image-20200108112959564](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108112959564.png) 





### 2.3 Mycat实现MySQL协议

#### 2.3.1 MySQL协议简介

##### 2.3.1.1 概述

MySQL协议处于应用层之下、TCP/IP之上, 在MySQL客户端和服务端之间使用。包含了链接器、MySQL代理、主从复制服务器之间通信，并支持SSL加密、传输数据的压缩、连接和身份验证及数据交互等。其中，握手认证阶段和命令执行阶段是MySQL协议中的两个重要阶段。



##### 2.3.1.2 握手认证阶段

![image-20200109113445831](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200109113445831.png) 

A. 握手认证阶段是客户端连接服务器的必经之路, 客户端与服务端完成TCP的三次握手以后, 服务端会向客户端发送一个初始化握手包, 握手包中包含了协议版本、MySQLServer版本、线程ID、服务器的权能标识和字符集等信息。

B. 客户端在接收到服务端的初始化握手包之后， 会发送身份验证包给服务端（AuthPacket）, 该包中包含用户名、密码等信息。

C. 服务端接收到客户端的登录验证包之后，需要进行逻辑校验，校验该登录信息是否正确。如果信息都符合，则返回一个OKPacket，表示登录成功,否则返回ERR_Packet，表示拒绝。

Wireshark抓包如下:

![image-20200127165109223](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127165109223.png) 



报文分析如下： 

1). 初始化握手包

![image-20200109133647751](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200109133647751.png) 

通过抓包工具Wireshark抓取到的握手包信息如下, 握手包格式:

![image-20200127162616334](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127162616334.png)  

说明: 

Packet Length : 包的长度;

Packet Number : 包的序号;

Server Greeting : 消息体, 包含了协议版本、MySQLServer版本、线程ID和字符集等信息。



2). 登录认证包

客户端在接收到服务端发来的初始握手包之后， 向服务端发出认证请求， 该请求包含以下信息（由Wireshark抓获） ： 

![image-20200127163702804](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127163702804.png) 



3). OK包或ERROR包

服务端接收到客户端的登录认证包之后，如果通过认证，则返回一个OKPacket，如果未通过认证，则返回一个ERROR包。

OK报文如下： 

![image-20200127163957990](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127163957990.png) 

ERROR报文如下 :

![image-20200127165156952](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127165156952.png) 



##### 2.3.1.3 命令执行阶段

在握手认证阶段通过并完成以后, 客户端可以向服务端发送各种命令来请求数据, 此阶段的流程是: 命令请求->返回结果集。

Wireshark 捕获的数据包如下： 

![image-20200127170112968](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127170112968.png) 

1). 命令包

![image-20200127170235143](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127170235143.png) 

2). 结果集包

![image-20200127170823882](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127170823882.png) 



#### 2.3.2 MySQL协议在MyCat中实现

##### 2.3.2.1 握手认证实现

在MyCat中同时实现了NIO和AIO, 通过配置可以选择NIO和AIO。MyCat Server在启动阶段已经选择好采用NIO还是AIO，因此建立I/O通道后,MyCat服务端一直等待客户端端的连接,当有连接到来的时候,MyCat首先发送握手包。 



1). 握手包源码实现

MyCat中的源码中io.mycat.net.FrontendConnection类的实现如下:

![image-20200127183259378](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127183259378.png) 

握手包信息组装完毕后, 通过FrontedConnection写回客户端。



2). 认证包源码实现

客户端接收到握手包后, 紧接着向服务端发起一个认证包, MyCat封装为类 AuthPacket:

![image-20200127231628215](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127231628215.png) 

客户端发送的认证包转由 FrontendAuthenticator 的Handler来处理, 主要操作就是 拆包, 检查用户名、密码合法性， 检查连接数是够超出限制。源码实现如下： 

![image-20200127232022594](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127232022594.png) 

认证失败， 调用failure方法， 认证成功调用success方法。

failure方法源码： 

![image-20200127232344040](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127232344040.png) 

success方法源码： 

![image-20200127232422887](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127232422887.png) 



##### 2.3.2.2 命令执行实现

命令执行阶段就是SQL命令和SQL语句执行阶段， 在该阶段MyCat主要需要做的事情， 就是对客户端发来的数据包进行拆包， 并判断命令的类型， 并解析SQL语句， 执行响应的SQL语句， 最后把执行结果封装在结果集包中， 返回给客户端。

从客户端发来的命令交给 FrontendCommandHandler 中的handle方法处理:

![image-20200127235140959](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200127235140959.png) 

处理具体的请求, 返回客户端结果集数据包: 

![image-20200128000050787](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200128000050787.png) 





### 2.4 MyCat线程架构与实现

#### 2.4.1 MyCat线程池实现

在MyCat中大量用到了线程池， 通过线程池来避免频繁的创建和销毁线程而造成的系统性能的浪费。在MyCat中使用的线程池是JDK中提供的线程池 ThreadPoolExecutor 的子类 NameableExecutor ， 构造方法如下： 

![image-20200108114506434](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108114506434.png) 

父类构造为： 

![image-20200108114611505](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108114611505.png) 

构造参数含义: 

corePoolSize : 核心池大小

maximumPoolSize : 最大线程数

keepAliveTime: 线程没有任务执行时, 最多能够存活多久

timeUnit: 时间单位

workQueue: 阻塞任务队列

threadFactory: 线程工厂, 用来创建线程



#### 2.4.2 MyCat线程架构

![image-20200108114952672](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108114952672.png) 

在MyCat中主要有两大线程池: timerExecutor 和 businessExecutor。

1). timerExecutor 线程池主要完成系统时间定时更新、处理器定时检查、数据节点定时连接空闲超时检查、数据节点定时心跳检测等任务。

2). businessExecutor是MyCat最重要的线程资源池, 该资源池的线程使用的范围非常广, 涵盖以下方面: 

A. 后端用原生协议连接数据

B. JDBC执行SQL语句

C. SQL拦截

D. 数据合并服务

E. 批量SQL作业

F. 查询结果的异步分发

G. 基于guava实现异步回调

![image-20200108141645417](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108141645417.png) 



### 2.5 MyCat内存管理及缓存框架与实现

这里所提到的内存管理指的是MyCat缓冲区管理, 众所周知设置缓冲区的唯一目的是提高系统的性能, 缓冲区通常是部分常用的数据存放在缓冲池中以便系统直接访问, 避免使用磁盘IO访问磁盘数据, 从而提高性能。

#### 2.5.1 内存管理

1). 缓冲池组成

缓冲池的最小单位为chunk, 默认的chunk大小为4096字节(DEFAULT_BUFFER_CHUNK_SIZE), BufferPool的总大小为4096 x processors x 1000(其中processors为处理器数量)。对I/O进程而言, 他们共享一个缓冲池。缓冲池有两种类型： 本地缓存线程（以$_开头的线程）缓冲区和其他缓冲区， 分配buffer时, 优先获取ThreadLocalPool中的buffer, 没有命中时会获取BufferPool中的buffer。



2). 分配MyCat缓冲池

分配缓冲池时, 可以指定大小, 也可以用默认值。

A. allocate(): 先检测是否为本地线程， 当执行线程为本地缓存线程时， localBufferPool取出一个可用的buffer。如果不是， 则从ConcurrentLinkedQueue队列中取出一个buffer进行分配, 如果队列没有可用的buffer, 则创建一个直接缓冲区。

B. allocate(size): 如果用户指定的size不大于chunkSize, 则调用allocate()进行分配; 反之则调用createTempBuffer(size)创建临时非直接缓冲区。



3). MyCat缓冲池的回收

回收时先判断buffer是否有效, 有如下情况时缓冲池不回收。

A. 不是直接缓冲区

B. buffer是空的

C. buffer的容量大于chunkSize



#### 2.5.2 MyCat缓存架构

1). 缓存框架选择

MyCat支持ehcache、mapdb、leveldb缓存, 可通过配置文件cacheserver.properties来进行配置;

![image-20200108154627518](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108154627518.png) 



2). 缓存内容

MyCat有路由缓存、表主键到datanode缓存、ER关系缓存。

A. 路由缓存: 即SQLRouteCache, 根据SQL语句查找路由信息的缓存, 该缓存只是针对select语句, 如果执行了之前已经执行过的某个SQL语句(缓存命中), 那么路由信息就不需要重复计算了, 直接从缓存中获取。

B. 表主键到datanode缓存: 当分片字段与主键字段不一致时, 直接通过主键值查询时无法定位具体分片的(只能全分片下发), 所以设置该缓存之后, 就可以利用主键值查找到分片名, 缓存的key是ID值, value是节点名。

C. ER关系缓存: 在ER分片时使用, 而且在insert查询中才会使用缓存, 当字表插入数据时, 根据父子关联字段确定子表分片, 下次可以直接从缓存中获取所在的分片。 

查看缓存指令： show @@cache；

![image-20200108155642414](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108155642414.png) 



### 2.6 MyCat连接池架构与实现

这里我们所讨论的连接池是MyCat的后端连接池， 也就是MyCat后端与各个数据库节点之间的连接架构。

1). 连接池创建

MyCat按照每个dataHost创建一个连接池, 根据schema.xml文件的配置取得最小的连接数minCon,  并初始化minCon个连接。在初始化连接时， 还需要判定用户选择的是JDBC还是原生的MySQL协议， 以便于创建对应的连接。

2). 连接池分配

分配连接就是从连接池队列中取出一个连接， 在取出一个连接时， MyCat需要根据负载均衡（balance属性）的类型选择不同的数据源， 因为连接和数据源绑在一起，所以需要知道MyCat读写的是那些数据源， 才能分配响应的连接。 

3). 架构

![image-20200108162456464](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200108162456464.png) 



### 2.7 MyCat主从切换架构与实现

#### 2.7.1 MyCat主从切换概述

MyCat实现MySQL读写分离的目的在于降低单节点数据库的访问压力,  原理就是让主数据库执行增删改操作, 从数据库执行查询操作, 利用MySQL数据库的复制机制将Master的数据同步到slave上。

当master宕机后，slave承载的业务如何切换到master继续提供服务，以及slave宕机后如何将master切换到slave上。手动切换数据源很简单， 但不是运维工作的首选，本节重点就是讲解如何实现自动切换。

MyCat的读写分离依赖于MySQL的主从同步, 也就是说MyCat没有实现数据的主从同步功能, 但是实现了自动切换功能。

**1). 自动切换**

switchtype：1

自动切换是MyCat主从复制的默认配置 , 当主机或从机宕机后, MyCat自动切换到可用的服务器上。 假设写服务器为M， 读服务器为S， 则： 

正常时， 写M读S；

当M宕机后， 读写S ； 恢复M后， 写S， 读M ；

当S宕机后， 读写M ； 恢复S后， 写M， 读S ；

 

**2). 基于MySQL主从同步状态的切换**

switchtype：2

这种切换方式与自动切换不同， MyCat检测到主从数据同步延迟时， 会自动切换到拥有最新数据的MySQL服务器上， 防止读到很久以前的数据。

原理就是通过检查MySQL的主从同步状态（show slave status）中的Seconds_Behind_Master、Slave_IO_Running、Slave_SQL_Running三个字段,来确定当前主从同步的状态以及主从之间的数据延迟。 Seconds_Behind_Master为0表示没有延迟，数值越大，则说明延迟越高。



#### 2.7.2 MyCat主从切换实现

基于延迟的切换， 则判断结果集中的Slave_IO_Running、Slave_SQL_Running两个个字段是否都为yes，以及Seconds_Behind_Master 是否小于配置文件中配置的 slaveThreshold的值, 如果有其中任何一个条件不满足, 则切换。

主要流程如下:

![image-20200128005840029](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200128005840029.png) 





### 2.8 MyCat核心技术

#### 2.2.1 MyCat分布式事务实现

MyCat在1.6版本以后已经支持XA分布式事务类型了。具体的使用流程如下：

1). 在应用层需要设置事务不能自动提交

```
set autocommit=0;
```

2). 在SQL中设置XA为开启状态

```
set xa = on;
```

3). 执行SQL

```
insert into user(id,name,sex) values(1,'Tom','1'),(2,'Rose','2'),(3,'Leo','1'),(4,'Lee','1');
```

4). 对事务进行提交或回滚

```
commit/rollback
```



完整流程如下: 

![image-20200129223657058](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200129223657058.png) 





#### 2.2.2 MyCat SQL路由实现

MyCat的路由是和SQL解析组件息息相关的, SQL路由模块是MyCat数据库中间件最重要的模块之一, 使用MyCat主要是为了分库分表, 而分库分表的核心就是路由。

##### 2.2.2.1 路由的作用

![image-20200113225535847](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200113225535847.png) 

如图所示， MyCat接收到应用系统发来的查询语句， 要将其发送到后端连接的MySQL数据库去执行， 但是后端有三个数据库服务器，具体要查询那一台数据库服务器呢， 这就是路由需要实现的功能。

SQL的路由既要保证数据的完整 ， 也不能造成资源的浪费， 还要保证路由的效率。



##### 2.2.2.2 SQL解析器

Mycat1.3版本之前模式使用的是Fdbparser的foundationdb的开源SQL解析器，在2015年被apple收购后，从开源变为闭源了。

目前版本的MyCat采用的是Druid的SQL解析器， 性能比采用Fdbparser整体性能提高20%以上。





#### 2.2.3 MyCat跨库Join

##### 2.2.3.1 全局表

每个企业级的系统中, 都会存在一些系统的基础信息表, 类似于字典表、省份、城市、区域、语言表等， 这些表与业务表之间存在关系， 但不是业务主从关系，而是一种属性关系。

当我们对业务表进行分片处理时， 可以将这些基础信息表设置为全局表， 也就是在每个节点中都存在该表。

全局表的特性如下： 

A. 全局表的insert、update、delete操作会实时地在所有节点同步执行, 保持各个节点数据的一致性

B. 全局表的查询操作会从任意节点执行,因为所有节点的数据都一致

C. 全局表可以和任意表进行join操作

![image-20200128013501684](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200128013501684.png) 



##### 2.2.3.2 ER表

关系型数据库是基于实体关系模型(Entity Relationship Model)的, MyCat中的ER表便来源于此。 MyCat提出了基于ER关系的数据分片策略 , 子表的记录与其所关联的父表的记录存放在同一个数据分片中, 通过表分组(Table Group)保证数据关联查询不会跨库操作。

![image-20200129101108379](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200129101108379.png) 

##### 2.2.3.3 catlet

catlet是MyCat为了解决跨分片Join提出的一种创新思路, 也叫做人工智能(HBT)。MyCat参考了数据库中存储过程的实现方式，提出类似的跨库解决方案，用户可以根据系统提供的API接口实现跨分片Join。

采用这种方案开发时,必须要实现Catlet接口的两个方法 :

![image-20200129104415975](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200129104415975.png) 

route 方法: 路由的方法, 传递系统配置和schema配置等 ;

processSQL方法: EngineCtx执行SQL并给客户端返回结果集 ;

当我们自定义Catlet完成之后, 需要将Catlet的实现类进行编译,并将其字节码文件 XXXCatlet.class存放在mycat_home/catlet目录下, 系统会加载相关Class, 而且每隔1分钟扫描一次文件是否更新, 若更新则自动重新加载,因此无需重启服务。



**ShareJoin**

ShareJoin 是Catlet的实现， 是一个简单的跨分片Join， 目前支持两个表的Join，原理就是解析SQL语句， 拆分成单表的语句执行， 单后把各个节点的数据进行汇集。



要想使用Catlet完成join， 还需要借助于MyCat中的注解， 在执行SQL语句时，使用catlet注解:

```sql
/*!mycat:catlet=demo.catlets.ShareJoin */ select a.id as aid , a.id , b.id as bid , b.name as name from customer a, company b where a.company_id=b.id and a.id = 1;
```



#### 2.2.4 MyCat数据汇聚与排序

通过MyCat实现数据汇聚和排序,不仅可以减少各分片与客户端之间的数据传输IO, 也可以帮助开发者总复杂的数据处理中解放出来,从而专注于开发业务代码。



在MySQL中存在两种排序方式： 一种利用有序索引获取有序数据， 另一种通过相应的排序算法将获取到的数据在内存中进行排序。 而MyCat中数据排序采用堆排序法对多个分片返回有序数据，并在合并、排序后再返回给客户端。

![image-20200129113055429](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200129113055429.png) 

## 3. MyCat综合案例

### 3.1 案例概述

#### 3.1.1 案例介绍

本案例将模拟电商项目中的商品管理、订单管理、基础信息管理、日志管理模块，对整个系统中的数据表进行分片操作，将根据不同的业务需求，采用不同的分片方式 。



#### 3.1.2 系统架构

![image-20200201153127417](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200201153127417.png) 

本案例涉及到的模块： 

1). 商品微服务

2). 订单微服务

3). 日志微服务





#### 3.1.3 技术选型

- SpringBoot
- SpringCloud
- SpringMVC
- Mybatis
- SpringDataRedis
- MySQL
- Redis

- Lombok









### 3.2 案例需求

1). 商品管理

A. 添加商品

B. 查询商品

![image-20200201194027874](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200201194027874.png) 



2). 订单管理

A. 下订单

B. 查询订单

![image-20200201194121792](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200201194121792.png) 



3). 日志管理

A. 日志记录

B. 日志查询

![image-20200201194159102](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200201194159102.png) 

















### 3.3 案例环境搭建

#### 3.3.1 数据库

1). 省份表 tb_provinces

| Field      | Type        | Comment  |
| ---------- | ----------- | -------- |
| provinceid | varchar(20) | 省份ID   |
| province   | varchar(50) | 省份名称 |



2). 市表 tb_cities

| Field      | Type        | Comment  |
| ---------- | ----------- | -------- |
| cityid     | varchar(20) | 城市ID   |
| city       | varchar(50) | 城市名称 |
| provinceid | varchar(20) | 省份ID   |



3). 区县表 tb_areas

| Field  | Type        | Comment  |
| ------ | ----------- | -------- |
| areaid | varchar(20) | 区域ID   |
| area   | varchar(50) | 区域名称 |
| cityid | varchar(20) | 城市ID   |



4). 商品分类表 tb_category

| Field     | Type        | Comment  |
| --------- | ----------- | -------- |
| id        | int(20)     | 分类ID   |
| name      | varchar(50) | 分类名称 |
| goods_num | int(11)     | 商品数量 |
| is_show   | char(1)     | 是否显示 |
| is_menu   | char(1)     | 是否导航 |
| seq       | int(11)     | 排序     |
| parent_id | int(20)     | 上级ID   |



5). 品牌表 tb_brand

| Field  | Type          | Comment      |
| ------ | ------------- | ------------ |
| id     | int(11)       | 品牌id       |
| name   | varchar(100)  | 品牌名称     |
| image  | varchar(1000) | 品牌图片地址 |
| letter | char(1)       | 品牌的首字母 |
| seq    | int(11)       | 排序         |



6). 商品SPU表 tb_spu

| Field          | Type          | Comment      |
| -------------- | ------------- | ------------ |
| id             | varchar(20)   | 主键         |
| sn             | varchar(60)   | 货号         |
| name           | varchar(100)  | SPU名        |
| caption        | varchar(100)  | 副标题       |
| brand_id       | int(11)       | 品牌ID       |
| category1_id   | int(20)       | 一级分类     |
| category2_id   | int(10)       | 二级分类     |
| category3_id   | int(10)       | 三级分类     |
| template_id    | int(20)       | 模板ID       |
| freight_id     | int(11)       | 运费模板id   |
| image          | varchar(200)  | 图片         |
| images         | varchar(2000) | 图片列表     |
| sale_service   | varchar(50)   | 售后服务     |
| introduction   | text          | 介绍         |
| spec_items     | varchar(3000) | 规格列表     |
| para_items     | varchar(3000) | 参数列表     |
| sale_num       | int(11)       | 销量         |
| comment_num    | int(11)       | 评论数       |
| is_marketable  | char(1)       | 是否上架     |
| is_enable_spec | char(1)       | 是否启用规格 |
| is_delete      | char(1)       | 是否删除     |
| status         | char(1)       | 审核状态     |



7). 商品SKU表 tb_sku

| Field         | Type          | Comment                         |
| ------------- | ------------- | ------------------------------- |
| id            | varchar(20)   | 商品id                          |
| sn            | varchar(100)  | 商品条码                        |
| name          | varchar(200)  | SKU名称                         |
| price         | int(20)       | 价格（分）                      |
| num           | int(10)       | 库存数量                        |
| alert_num     | int(11)       | 库存预警数量                    |
| image         | varchar(200)  | 商品图片                        |
| images        | varchar(2000) | 商品图片列表                    |
| weight        | int(11)       | 重量（克）                      |
| create_time   | datetime      | 创建时间                        |
| update_time   | datetime      | 更新时间                        |
| spu_id        | varchar(20)   | SPUID                           |
| category_id   | int(10)       | 类目ID                          |
| category_name | varchar(200)  | 类目名称                        |
| brand_name    | varchar(100)  | 品牌名称                        |
| spec          | varchar(200)  | 规格                            |
| sale_num      | int(11)       | 销量                            |
| comment_num   | int(11)       | 评论数                          |
| status        | char(1)       | 商品状态 1-正常，2-下架，3-删除 |
| version       | int(255)      |                                 |



8). 订单表 tb_order

| Field             | Type          | Comment                                                      |
| ----------------- | ------------- | ------------------------------------------------------------ |
| id                | varchar(200)  | 订单id                                                       |
| total_num         | int(11)       | 数量合计                                                     |
| total_money       | int(11)       | 金额合计                                                     |
| pre_money         | int(11)       | 优惠金额                                                     |
| post_fee          | int(11)       | 邮费                                                         |
| pay_money         | int(11)       | 实付金额                                                     |
| pay_type          | varchar(1)    | 支付类型，1、在线支付、0 货到付款                            |
| create_time       | datetime      | 订单创建时间                                                 |
| update_time       | datetime      | 订单更新时间                                                 |
| pay_time          | datetime      | 付款时间                                                     |
| consign_time      | datetime      | 发货时间                                                     |
| end_time          | datetime      | 交易完成时间                                                 |
| close_time        | datetime      | 交易关闭时间                                                 |
| shipping_name     | varchar(20)   | 物流名称                                                     |
| shipping_code     | varchar(20)   | 物流单号                                                     |
| username          | varchar(50)   | 用户名称                                                     |
| buyer_message     | varchar(1000) | 买家留言                                                     |
| buyer_rate        | char(1)       | 是否评价                                                     |
| receiver_contact  | varchar(50)   | 收货人                                                       |
| receiver_mobile   | varchar(12)   | 收货人手机                                                   |
| receiver_province | varchar(200)  | 收货人省份                                                   |
| receiver_city     | varchar(200)  | 收货人市                                                     |
| receiver_area     | varchar(200)  | 收货人区/县                                                  |
| receiver_address  | varchar(200)  | 收货人具体街道地址                                           |
| source_type       | char(1)       | 订单来源：1:web，2：app，3：微信公众号，4：微信小程序 5 H5手机页面 |
| transaction_id    | varchar(30)   | 交易流水号                                                   |
| order_status      | char(1)       | 订单状态                                                     |
| pay_status        | char(1)       | 支付状态 0:未支付 1:已支付                                   |
| consign_status    | char(1)       | 发货状态 0:未发货 1:已发货 2:已送达                          |
| is_delete         | char(1)       | 是否删除                                                     |



9). 订单明细表 tb_order_item

| Field        | Type         | Comment  |
| ------------ | ------------ | -------- |
| id           | varchar(200) | ID       |
| category_id1 | int(11)      | 1级分类  |
| category_id2 | int(11)      | 2级分类  |
| category_id3 | int(11)      | 3级分类  |
| spu_id       | varchar(200) | SPU_ID   |
| sku_id       | varchar(200) | SKU_ID   |
| order_id     | varchar(200) | 订单ID   |
| name         | varchar(200) | 商品名称 |
| price        | int(20)      | 单价     |
| num          | int(10)      | 数量     |
| money        | int(20)      | 总金额   |
| pay_money    | int(11)      | 实付金额 |
| image        | varchar(200) | 图片地址 |
| weight       | int(11)      | 重量     |
| post_fee     | int(11)      | 运费     |
| is_return    | char(1)      | 是否退货 |



10). 订单日志表 tb_order_log 

| Field          | Type         | Comment  |
| -------------- | ------------ | -------- |
| id             | varchar(20)  | ID       |
| operater       | varchar(50)  | 操作员   |
| operate_time   | datetime     | 操作时间 |
| order_id       | bigint(20)   | 订单ID   |
| order_status   | char(1)      | 订单状态 |
| pay_status     | char(1)      | 付款状态 |
| consign_status | char(1)      | 发货状态 |
| remarks        | varchar(100) | 备注     |



11). 操作日志表 tb_operatelog

| Field           | Type         | Comment               |
| --------------- | ------------ | --------------------- |
| id              | bigint(20)   | ID                    |
| model_name      | varchar(200) | 模块名                |
| model_value     | varchar(200) | 模块值                |
| return_value    | varchar(200) | 返回值                |
| return_class    | varchar(200) | 返回值类型            |
| operate_user    | varchar(20)  | 操作用户              |
| operate_time    | varchar(20)  | 操作时间              |
| param_and_value | varchar(500) | 请求参数名及参数值    |
| operate_class   | varchar(200) | 操作类                |
| operate_method  | varchar(200) | 操作方法              |
| cost_time       | bigint(20)   | 执行方法耗时, 单位 ms |



12). 字典表 tb_dictionary

| Field       | Type         | Comment       |
| ----------- | ------------ | ------------- |
| id          | int(11)      | 主键ID , 自增 |
| codeid      | int(11)      | 码表ID        |
| codetype    | varchar(2)   | 码值类型      |
| codename    | varchar(50)  | 名称          |
| codevalue   | varchar(50)  | 码值          |
| description | varchar(100) | 描述          |
| createtime  | datetime     | 创建时间      |
| updatetime  | datetime     | 修改时间      |
| createuser  | int(11)      | 创建人        |
| updateuser  | int(11)      | 修改人        |



#### 3.3.2 工程预览

![image-20200201201704741](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200201201704741.png) 

```
spring-boot-starter-parent
    |- v_parent	--------------------> 父工程, 统一管理依赖版本
        |- v_common ----------------> 通用工程, 存放通用的工具类及组件
        |- v_model -----------------> 实体类
        |- v_eureka ----------------> 注册中心
        |- v_feign_api -------------> feign远程调用的客户端接口
        |- v_gateway ---------------> 网关工程
        |- v_manage_web ------------> 模拟前端工程
        |- v_service_goods ---------> 商品微服务
        |- v_service_log -----------> 日志微服务
        |- v_service_order ---------> 订单微服务
```



#### 3.3.3 工程层级关系

![image-20200205010155489](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200205010155489.png) 



#### 3.3.4 父工程搭建

工程名: v_parent

pom.xml

```xml
<!-- springBoot项目需要集成自父工程 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.4.RELEASE</version>
</parent>

<properties>
    <skipTests>true</skipTests>
</properties>

<!--依赖包-->
<dependencies>
    <!--测试包-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Greenwich.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <!--MySQL数据库驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>

        <!--mybatis分页插件-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.3</version>
        </dependency>

        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.51</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.6</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```



#### 3.3.5 基础工程搭建

1). v_model

该基础工程中存放的是与数据库对应的实体类 ;

A. pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```



B. 导入实体类



2). v_common

该基础工程中存放的是通用的组件及工具类 , 比如 分页实体类, 结果实体类, 状态码 等

直接导入资料中提供的基础组件和工具类 ;



3). v_feign_api

该工程中, 主要存放的是Feign远程调用的客户端接口;

pom.xml

```xml
<dependencies>
    <!--web起步依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Feign起步依赖 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <dependency>
        <groupId>cn.itcast</groupId>
        <artifactId>v_common</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>

    <dependency>
        <groupId>cn.itcast</groupId>
        <artifactId>v_model</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>

</dependencies>
```



#### 3.3.6 Eureka Server搭建

1). pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```



2). 引导类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class,args);
    }
}
```



3). application.yml

```yml
spring:
  application:
    name: eureka
server:
  port: 8161
eureka:
  client:
    register-with-eureka: false #是否将自己注册到eureka中
    fetch-registry: false #是否从eureka中获取信息
    service-url:
      defaultZone: http://127.0.0.1:${server.port}/eureka/
  server:
    enable-self-preservation: true
```



#### 3.3.7 GateWay 网关搭建

1). pom.xml

```xml
<!--网关依赖-->
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```



2). 引导类

```java
@SpringBootApplication
@EnableEurekaClient
public class GateWayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GateWayApplication.class,args);
    }
}
```



3). application.yml

```yml
server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:8161/eureka
  instance:
    prefer-ip-address: true
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: v_goods_route
          uri: lb://goods
          predicates:
            - Path=/goods/**
          filters:
            - StripPrefix=1

        - id: v_order_route
          uri: lb://order
          predicates:
            - Path=/order/**
          filters:
            - StripPrefix=1
```



4). Cors配置类

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsWebFilter;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;
import org.springframework.web.util.pattern.PathPatternParser;

@Configuration
public class CorsConfig {

    @Bean
    public CorsWebFilter corsFilter(){
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", buildConfig());
        return new CorsWebFilter(source);
    }

    private CorsConfiguration buildConfig(){
        CorsConfiguration corsConfiguration = new CorsConfiguration();
		//在生产环境上最好指定域名，以免产生跨域安全问题
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        return corsConfiguration;
    }
}
```





### 3.4 功能开发

#### 3.4.1 商品管理模块

需求 : 

1). 根据ID查询商品SPU信息;

2). 根据条件查询商品SPU列表;

3). 根据ID查询商品SKU信息;

![image-20200216225916691](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200216225916691.png) 



概念: 

1). SPU = Standard Product Unit  （标准产品单位）

概念 : SPU 是商品信息聚合的最小单位，是一组可复用、易检索的标准化信息的集合，该集合描述了一个产品的特性。
通俗点讲，属性值、特性相同的货品就可以称为一个 SPU

例如：华为P30 就是一个 SPU



2). SKU=stock keeping unit( 库存量单位)

SKU  即库存进出计量的单位， 可以是以件、盒、托盘等为单位。
SKU  是物理上不可分割的最小存货单元。在使用时要根据不同业态，不同管理模式来处理。在服装、鞋类商品中使用最多最普遍。

例如：红色 64G 全网通 的华为P30 就是一个 SKU



##### 3.4.1.1 创建工程

pom.xml

```xml
 <dependencies>
        <!-- Eureka客户端依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <!--MySQL数据库驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--mybatis分页插件-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
        </dependency>

        <!--web起步依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- redis 使用-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- fastJson依赖 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
        </dependency>

        <!-- Feign依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_model</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_feign_api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

    </dependencies>
```

application.yml

```yml
server:
  port: 9001
spring:
  application:
    name: goods
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/v_shop?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: 2143
  main:
    allow-bean-definition-overriding: true #当遇到同样名字的时候，是否允许覆盖注册
eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://127.0.0.1:8161/eureka
  instance:
    prefer-ip-address: true
```



引导类

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = "cn.itcast.feign")
@MapperScan("cn.itcast.goods.mapper")
public class GoodsApplication {
    public static void main(String[] args) {
        SpringApplication.run(GoodsApplication.class,args);
    }
}
```



##### 3.4.1.2 Mapper

1). mapper接口定义

```java
public interface SpuMapper {

    public TbSpu findById(String spuId);

    public List<TbSpu> search(Map<String,Object> searchMap);

}
```

```java
public interface SkuMapper  {
    //根据ID查询SKU
    public TbSku findById(String skuId);

}
```



2). mapper映射配置文件

SpuMapper.xml

```xml
<mapper namespace="cn.itcast.goods.mapper.SpuMapper" >

    <resultMap id="spuResultMap" type="cn.itcast.model.TbSpu">
        <id column="id" jdbcType="VARCHAR" property="id" />
        <result column="sn" jdbcType="VARCHAR" property="sn" />
        <result column="name" jdbcType="VARCHAR" property="name" />
        <result column="caption" jdbcType="VARCHAR" property="caption" />
        <result column="brand_id" jdbcType="INTEGER" property="brandId" />
        <result column="category1_id" jdbcType="INTEGER" property="category1Id" />
        <result column="category2_id" jdbcType="INTEGER" property="category2Id" />
        <result column="category3_id" jdbcType="INTEGER" property="category3Id" />
        <result column="template_id" jdbcType="INTEGER" property="templateId" />
        <result column="freight_id" jdbcType="INTEGER" property="freightId" />
        <result column="image" jdbcType="VARCHAR" property="image" />
        <result column="images" jdbcType="VARCHAR" property="images" />
        <result column="sale_service" jdbcType="VARCHAR" property="saleService" />
        <result column="spec_items" jdbcType="VARCHAR" property="specItems" />
        <result column="para_items" jdbcType="VARCHAR" property="paraItems" />
        <result column="sale_num" jdbcType="INTEGER" property="saleNum" />
        <result column="comment_num" jdbcType="INTEGER" property="commentNum" />
        <result column="is_marketable" jdbcType="CHAR" property="isMarketable" />
        <result column="is_enable_spec" jdbcType="CHAR" property="isEnableSpec" />
        <result column="is_delete" jdbcType="CHAR" property="isDelete" />
        <result column="status" jdbcType="CHAR" property="status" />
    </resultMap>

    <select id="findById" parameterType="java.lang.String" resultMap="spuResultMap">
        select
            *
        from
            tb_spu
        where
            id = #{spuId}
    </select>

    <select id="search" resultMap="spuResultMap">
        select * from tb_spu
        <where>
            <if test="name != null and name != ''">
                and name like '%${name}%'
            </if>
            <if test="caption != null and caption != ''" >
                and caption like '%${caption}%'
            </if>
            <if test="brandId != null">
                and brand_id = #{brandId}
            </if>
            <if test="status != null and status != ''">
                and status = #{status}
            </if>
        </where>
    </select>

</mapper>
```



SkuMapper.xml

```xml
<mapper namespace="cn.itcast.goods.mapper.SkuMapper" >

    <resultMap id="skuResultMap" type="cn.itcast.model.TbSku">
        <id column="id" jdbcType="VARCHAR" property="id" />
        <result column="sn" jdbcType="VARCHAR" property="sn" />
        <result column="name" jdbcType="VARCHAR" property="name" />
        <result column="price" jdbcType="INTEGER" property="price" />
        <result column="num" jdbcType="INTEGER" property="num" />
        <result column="alert_num" jdbcType="INTEGER" property="alertNum" />
        <result column="image" jdbcType="VARCHAR" property="image" />
        <result column="images" jdbcType="VARCHAR" property="images" />
        <result column="weight" jdbcType="INTEGER" property="weight" />
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime" />
        <result column="update_time" jdbcType="TIMESTAMP" property="updateTime" />
        <result column="spu_id" jdbcType="VARCHAR" property="spuId" />
        <result column="category_id" jdbcType="INTEGER" property="categoryId" />
        <result column="category_name" jdbcType="VARCHAR" property="categoryName" />
        <result column="brand_name" jdbcType="VARCHAR" property="brandName" />
        <result column="spec" jdbcType="VARCHAR" property="spec" />
        <result column="sale_num" jdbcType="INTEGER" property="saleNum" />
        <result column="comment_num" jdbcType="INTEGER" property="commentNum" />
        <result column="status" jdbcType="CHAR" property="status" />
        <result column="version" jdbcType="INTEGER" property="version" />
    </resultMap>

    <select id="findById" parameterType="java.lang.String" resultMap="skuResultMap">
        select * from tb_sku where id = #{skuId}
    </select>
</mapper>
```



##### 3.4.1.3 Service

1). 接口定义 

```java
public interface SkuService {
    /**
     * 根据ID查询SKU
     * @param skuId
     * @return
     */
    public TbSku findById(String skuId);
}
```

```java
public interface SpuService {
    /**
     * 根据ID查询
     * @param id
     * @return
     */
    TbSpu findById(String id);
    /***
     * 多条件分页查询
     * @param searchMap
     * @param page
     * @param size
     * @return
     */
    Page<TbSpu> findPage(Map<String, Object> searchMap, int page, int size);
}
```



2).接口实现

```java
@Service
public class SkuServiceImpl implements SkuService {
    @Autowired
    private SkuMapper skuMapper;
    @Autowired
    private RedisTemplate redisTemplate;
    
    @Override
    public TbSku findById(String skuId) {
        return skuMapper.findById(skuId);
    }
    
}
```

```java
@Service
public class SpuServiceImpl implements SpuService {

    @Autowired
    private SpuMapper spuMapper;

    /**
     * 根据ID查询
     * @param id
     * @return
     */
    @Override
    public TbSpu findById(String id){
        return  spuMapper.findById(id);
    }

    /**
     * 条件+分页查询
     * @param searchMap 查询条件
     * @param page 页码
     * @param size 页大小
     * @return 分页结果
     */
    @Override
    public Page<TbSpu> findPage(Map<String,Object> searchMap, int page, int size){
        PageHelper.startPage(page,size);
        return (Page<TbSpu>) spuMapper.search(searchMap);
    }

}
```



##### 3.4.1.4 Controller

```java
@RestController
@CrossOrigin(origins = "*")
@RequestMapping("/sku")
public class SkuController {
    @Autowired
    private SkuService skuService;
    /***
     * 根据ID查询数据
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    public Result<TbSku> findById(@PathVariable("id") String id){
        TbSku sku = skuService.findById(id);
        return new Result(true,StatusCode.OK,"查询成功",sku);
    }
}
```

```java
@RestController
@CrossOrigin(origins = "*")
@RequestMapping("/spu")
public class SpuController {

    @Autowired
    private SpuService spuService;

    /***
     * 根据ID查询数据
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    public Result<TbSpu> findById(@PathVariable("id") String id){
        TbSpu spu = spuService.findById(id);
        return new Result(true,StatusCode.OK,"查询成功",spu);
    }
    
    /***
     * 分页搜索实现
     * @param searchMap
     * @param page
     * @param size
     * @return
     */
    @PostMapping(value = "/search/{page}/{size}" )
    public Result<TbSpu> findPage(@RequestBody Map searchMap, @PathVariable  Integer page, @PathVariable  Integer size){
        com.github.pagehelper.Page<TbSpu> pageList = spuService.findPage(searchMap, page, size);
        PageResult pageResult=new PageResult<TbSpu>(pageList.getTotal(),pageList.getResult());
        return new Result<TbSpu>(true,StatusCode.OK,"查询成功",pageResult);
    }
}
```





#### 3.4.2 订单模块

需求:

1). 下单业务分析

2). 根据条件分页查询订单

表结构: 

tb_order , tb_order_item , tb_order_log



##### 3.4.2.1 创建工程

1).pom.xml

```xml
 <dependencies>
     <dependency>
         <groupId>cn.itcast</groupId>
         <artifactId>v_common</artifactId>
         <version>1.0-SNAPSHOT</version>
     </dependency>

     <dependency>
         <groupId>cn.itcast</groupId>
         <artifactId>v_model</artifactId>
         <version>1.0-SNAPSHOT</version>
     </dependency>

     <dependency>
         <groupId>cn.itcast</groupId>
         <artifactId>v_feign_api</artifactId>
         <version>1.0-SNAPSHOT</version>
     </dependency>

     <!-- Eureka客户端依赖 -->
     <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
     </dependency>
	
	<!-- springboot - Mybatis 起步依赖 -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.1.0</version>
    </dependency>

     <!--MySQL数据库驱动-->
     <dependency>
         <groupId>mysql</groupId>
         <artifactId>mysql-connector-java</artifactId>
     </dependency>

     <!--mybatis分页插件-->
     <dependency>
         <groupId>com.github.pagehelper</groupId>
         <artifactId>pagehelper-spring-boot-starter</artifactId>
     </dependency>

     <!--web起步依赖-->
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-web</artifactId>
     </dependency>

     <!-- redis 使用-->
     <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-data-redis</artifactId>
     </dependency>

     <!-- fastJson依赖 -->
     <dependency>
         <groupId>com.alibaba</groupId>
         <artifactId>fastjson</artifactId>
     </dependency>

     <!-- Feign依赖 -->
     <dependency>
         <groupId>org.springframework.cloud</groupId>
         <artifactId>spring-cloud-starter-openfeign</artifactId>
     </dependency>

</dependencies>
```



2). application.yml

```yml
server:
  port: 9002
spring:
  application:
    name: order
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/v_shop?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: 2143
  main:
    allow-bean-definition-overriding: true #当遇到同样名字的时候，是否允许覆盖注册
eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://127.0.0.1:8161/eureka
  instance:
    prefer-ip-address: true
feign:
  client:
    config:
      default:   #配置全局的feign的调用超时时间  如果 有指定的服务配置 默认的配置不会生效
        connectTimeout: 60000 # 指定的是 消费者 连接服务提供者的连接超时时间 是否能连接  单位是毫秒
        readTimeout: 20000  # 指定的是调用服务提供者的 服务 的超时时间（）  单位是毫秒
```



3). 引导类

```java
@SpringBootApplication
@EnableEurekaClient
@MapperScan(basePackages = "cn.itcast.order.mapper")
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class,args);
    }
}
```



##### 3.4.2.2 下单业务分析

![image-20200217102815624](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200217102815624.png)



##### 3.4.2.3 查询订单

![image-20200217141007643](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200217141007643.png) 

###### 3.4.2.3.1 Mapper

1). mapper接口

```java
public interface OrderMapper  {
    public List<TbOrder> search(Map<String,Object> searchMap);
}
```





2). mapper映射配置文件

OrderMapper.xml

```xml
<mapper namespace="cn.itcast.order.mapper.OrderMapper" >
    <resultMap id="orderResultMap" type="cn.itcast.model.TbOrder">
        <id column="id" jdbcType="VARCHAR" property="id" />
        <result column="total_num" jdbcType="INTEGER" property="totalNum" />
        <result column="total_money" jdbcType="INTEGER" property="totalMoney" />
        <result column="pre_money" jdbcType="INTEGER" property="preMoney" />
        <result column="post_fee" jdbcType="INTEGER" property="postFee" />
        <result column="pay_money" jdbcType="INTEGER" property="payMoney" />
        <result column="pay_type" jdbcType="VARCHAR" property="payType" />
        <result column="create_time" jdbcType="TIMESTAMP" property="createTime" />
        <result column="update_time" jdbcType="TIMESTAMP" property="updateTime" />
        <result column="pay_time" jdbcType="TIMESTAMP" property="payTime" />
        <result column="consign_time" jdbcType="TIMESTAMP" property="consignTime" />
        <result column="end_time" jdbcType="TIMESTAMP" property="endTime" />
        <result column="close_time" jdbcType="TIMESTAMP" property="closeTime" />
        <result column="shipping_name" jdbcType="VARCHAR" property="shippingName" />
        <result column="shipping_code" jdbcType="VARCHAR" property="shippingCode" />
        <result column="username" jdbcType="VARCHAR" property="username" />
        <result column="buyer_message" jdbcType="VARCHAR" property="buyerMessage" />
        <result column="buyer_rate" jdbcType="CHAR" property="buyerRate" />
        <result column="receiver_contact" jdbcType="VARCHAR" property="receiverContact" />
        <result column="receiver_mobile" jdbcType="VARCHAR" property="receiverMobile" />
        <result column="receiver_province" jdbcType="VARCHAR" property="receiverProvince" />
        <result column="receiver_city" jdbcType="VARCHAR" property="receiverCity" />
        <result column="receiver_area" jdbcType="VARCHAR" property="receiverArea" />
        <result column="receiver_address" jdbcType="VARCHAR" property="receiverAddress" />
        <result column="source_type" jdbcType="CHAR" property="sourceType" />
        <result column="transaction_id" jdbcType="VARCHAR" property="transactionId" />
        <result column="order_status" jdbcType="CHAR" property="orderStatus" />
        <result column="pay_status" jdbcType="CHAR" property="payStatus" />
        <result column="consign_status" jdbcType="CHAR" property="consignStatus" />
        <result column="is_delete" jdbcType="CHAR" property="isDelete" />
    </resultMap>


    <select id="search" resultType="cn.itcast.model.TbOrder">
        SELECT
            o.id ,
            o.`create_time` createTime,
            o.username ,
            o.`total_money` totalMoney,
            o.`total_num` totalNum,
            o.`pay_type` payType,
            o.`pay_status` payStatus,

            p.`province` receiverProvince
        FROM
            tb_order o , tb_provinces p
        WHERE
            o.receiver_province = p.provinceid
        <if test="orderId != null and orderId != ''">
            and o.id = #{orderId}
        </if>

        <if test="payType != null and payType != ''">
            and o.pay_type = #{payType}
        </if>

        <if test="username != null and username != ''">
            and o.username = #{username}
        </if>

        <if test="payStatus != null and payStatus != ''">
            and o.order_status = #{payStatus}
        </if>
    </select>
</mapper>
```



###### 3.4.2.3.2 Service

service接口

```java
public interface OrderService {

    /***
     * 新增
     * @param order
     */
    void add(TbOrder order);

    /***
     * 多条件分页查询
     * @param searchMap
     * @param page
     * @param size
     * @return
     */
    Page<TbOrder> findPage(Map<String, Object> searchMap, int page, int size);

}
```

service实现

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private OrderItemMapper orderItemMapper;
    @Autowired
    private IdWorker idWorker;

    /**
     * 增加
     * @param order
     */
    @Override
    public void add(TbOrder order){
        //1.获取购物车的相关数据(redis)
        
        //2.统计计算:总金额,总数量
        
        //3.填充订单数据并保存到tb_order

        //4.填充订单项数据并保存到tb_order_item

        //5.记录订单日志
        
        //6.扣减库存并增加销量

        //7.删除购物车数据(redis)

      
    }
    
    
    /**
     * 条件+分页查询
     * @param searchMap 查询条件
     * @param page 页码
     * @param size 页大小
     * @return 分页结果
     */
    @Override
    public Page<TbOrder> findPage(Map<String,Object> searchMap, int page, int size){
        PageHelper.startPage(page,size);
        return (Page<TbOrder>)orderMapper.search(searchMap);
    }
}
```





###### 3.4.2.3.3 Controller

```java
@RestController
@CrossOrigin(value = {"*"})
@RequestMapping("/order")
public class OrderController {

    @Autowired
    private OrderService orderService;


    @PostMapping
    @OperateLog
    public Result add(@RequestBody TbOrder order){
        //获取登录人名称
        orderService.add(order);
        return new Result(true,StatusCode.OK,"提交成功");
    }


    /***
     * 分页搜索实现
     * @param searchMap
     * @param page
     * @param size
     * @return
     */
    @PostMapping(value = "/search/{page}/{size}" )
    @OperateLog
    public Result findPage(@RequestBody Map searchMap, @PathVariable  Integer page, @PathVariable  Integer size){
        Page<TbOrder> pageList = orderService.findPage(searchMap, page, size);
        PageResult pageResult=new PageResult(pageList.getTotal(),pageList.getResult());
        return new Result(true,StatusCode.OK,"查询成功",pageResult);
    }
}
```



#### 3.4.3 日志模块

表结构: 

tb_operatelog



需求: 

1). 记录日志

2). 查询日志

![image-20200205224758890](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200205224758890.png) 

![image-20200218031150324](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200218031150324.png) 

##### 3.4.3.1 创建工程

1). pom.xml

```xml
    <dependencies>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_model</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>cn.itcast</groupId>
            <artifactId>v_feign_api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>


        <!-- Eureka客户端依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!--MySQL数据库驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!--mybatis分页插件-->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
        </dependency>

        <!--web起步依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- fastJson依赖 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
        </dependency>

        <!-- Feign依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>

    </dependencies>
```



2). application.yml

```yml
server:
  port: 9003
spring:
  application:
    name: log
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/v_shop?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: 2143
  main:
    allow-bean-definition-overriding: true #当遇到同样名字的时候，是否允许覆盖注册
eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://127.0.0.1:8161/eureka
  instance:
    prefer-ip-address: true
```



3). 引导类

```java
@SpringBootApplication
@EnableEurekaClient
@MapperScan(basePackages = "cn.itcast.log.mapper")
public class LogApplication {
    public static void main(String[] args) {
        SpringApplication.run(LogApplication.class,args);
    }
   
    @Bean
    public IdWorker idworker(){
        return new IdWorker(0,0);
    }
}
```





**分布式ID生成**

snowflake是 Twitter 开源的分布式ID生成算法，结果是一个long型的ID。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中心，5个bit的机器ID），12bit作为毫秒内的流水号（意味着每个节点在每毫秒可以产生 4096 个 ID），最后还有一个符号位，永远是0 ;

![image-20200218010603410](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200218010603410.png) 

使用方式: 

```java
IdWorker idWorker=new IdWorker(1,1);//0-31 , 0-31

for(int i=0;i<10000;i++){
	long id = idWorker.nextId();
	System.out.println(id);
}
```







##### 3.4.3.2 Mapper

mapper接口

```java
public interface OperateLogMapper {

    public void insert(TbOperatelog operationLog);

    public List<TbOperatelog> search(Map searchMap);

}
```

OperateLogMapper.xml

```xml
<mapper namespace="cn.itcast.log.mapper.OperateLogMapper" >

    <resultMap id="operateLogResultMap" type="cn.itcast.model.TbOperatelog">
        <id column="id" jdbcType="BIGINT" property="id" />
        <result column="model_name" jdbcType="VARCHAR" property="modelName" />
        <result column="model_value" jdbcType="VARCHAR" property="modelValue" />
        <result column="return_value" jdbcType="VARCHAR" property="returnValue" />
        <result column="return_class" jdbcType="VARCHAR" property="returnClass" />
        <result column="operate_user" jdbcType="VARCHAR" property="operateUser" />
        <result column="operate_time" jdbcType="VARCHAR" property="operateTime" />
        <result column="param_and_value" jdbcType="VARCHAR" property="paramAndValue" />
        <result column="operate_class" jdbcType="VARCHAR" property="operateClass" />
        <result column="operate_method" jdbcType="VARCHAR" property="operateMethod" />
        <result column="cost_time" jdbcType="BIGINT" property="costTime" />
    </resultMap>


    <insert id="insert" parameterType="cn.itcast.model.TbOperatelog">
    insert into tb_operatelog (id, model_name, model_value, 
      return_value, return_class, operate_user, 
      operate_time, param_and_value, operate_class, 
      operate_method, cost_time)
    values (#{id}, #{modelName}, #{modelValue}, 
      #{returnValue}, #{returnClass}, #{operateUser}, 
      #{operateTime}, #{paramAndValue}, #{operateClass}, 
      #{operateMethod}, #{costTime})
  </insert>


    <select id="search" resultMap="operateLogResultMap">
        select * from tb_operatelog
        <where>
            <if test="operateUser != null and operateUser != ''">
                and operate_user = #{operateUser}
            </if>
            <if test="operateMethod != null and operateMethod != ''">
                and operate_method = #{operateMethod}
            </if>
            <if test="returnClass != null and returnClass != ''">
                and return_class = #{returnClass}
            </if>
            <if test="costTime != null and costTime != '' ">
                and cost_time = #{costTime}
            </if>
        </where>
    </select>

</mapper>
```



##### 3.4.3.3 Service

接口

```
public interface OperateLogService {
    public void insert(TbOperatelog operationLog);

    public Page<TbOperatelog> findPage(Map searchMap, Integer pageNum , Integer pageSize);
}
```

实现

```java
@Service
@Transactional
public class OperateLogServiceImpl implements OperateLogService {


    @Autowired
    private OperateLogMapper operateLogMapper;

    public void insert(TbOperatelog operationLog){
        long id = idworker.nextId();
        operationLog.setId(id);
        operateLogMapper.insert(operationLog);
    }

    public Page<TbOperatelog> findPage(Map searchMap, Integer pageNum , Integer pageSize){
        System.out.println(searchMap);
	
        PageHelper.startPage(pageNum,pageSize);
        List<TbOperatelog> list = operateLogMapper.search(searchMap);

        return (Page<TbOperatelog>) list;
    }

}
```





##### 3.4.3.4 Controller

```java
@RestController
@RequestMapping("/operateLog")
public class OperateLogController {

    @Autowired
    private OperateLogService operateLogService;

    @RequestMapping("/search/{page}/{size}")
    public Result findList(@RequestBody Map dataMap, @PathVariable Integer page, @PathVariable  Integer size){

        Page<TbOperatelog> pageList = operateLogService.findPage(dataMap, page, size);
        PageResult pageResult=new PageResult(pageList.getTotal(),pageList.getResult());

        return new Result(true, StatusCode.OK,"查询成功",pageResult);
    }


    @RequestMapping("/add")
    public Result add(@RequestBody TbOperatelog operatelog){
        operateLogService.insert(operatelog);
        return new Result(true,StatusCode.OK,"添加成功");
    }

}
```



##### 3.4.3.5 AOP记录日志

在需要记录操作日志的微服务中, 引入AOP记录日志的类 : 

1). 自定义注解

```java
@Inherited
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OperateLog {
}
```

2). AOP通知类

```java
@Component
@Aspect
public class OperateAdvice {

	@Autowired
	private OperateLogFeign operateLogFeign;

	@Around("execution(* cn.itcast.goods.controller.*.*(..)) && @annotation(operateLog)")
	public Object insertLogAround(ProceedingJoinPoint pjp , OperateLog operateLog) throws Throwable{
		System.out.println(" *********************************** 记录日志 [start]  ****************************** ");

		TbOperatelog op = new TbOperatelog();

		DateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

		op.setOperateTime(sdf.format(new Date()));
		op.setOperateUser("10000");
		
		op.setOperateClass(pjp.getTarget().getClass().getName());
		op.setOperateMethod(pjp.getSignature().getName());

		String paramAndValue = "";
		Object[] args = pjp.getArgs();
		if(args != null){
			for (Object arg : args) {
				if(arg instanceof String || arg instanceof Integer || arg instanceof Long){
					paramAndValue += arg +",";
				}else{
					paramAndValue += JSON.toJSONString(arg)+",";
				}
			}
			op.setParamAndValue(paramAndValue);
		}

		long start_time = System.currentTimeMillis();

		//放行
		Object object = pjp.proceed();

		long end_time = System.currentTimeMillis();
		op.setCostTime(end_time - start_time);

		if(object != null){
			op.setReturnClass(object.getClass().getName());
			op.setReturnValue(object.toString());
		}else{
			op.setReturnClass("java.lang.Object");
			op.setParamAndValue("void");
		}

		operateLogFeign.add(op);

		System.out.println(" *********************************** 记录日志 [end]  ****************************** ");

		return object;
	}
}
```



### 3.5 MyCat分片

当前数据库的情况 :

![image-20200209111951501](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200209111951501.png) 

由于当前项目是一个电商项目，项目上线后，随着项目的运营，业务系统的数据库中的数据与日俱增，特别是订单、日志等数据，如果数据量过大，这个时候就需要考虑通过MyCat分库分表。



#### 3.5.1 分片分析

1). 垂直拆分

数据量过大，需要考虑扩容，可以通过MyCat来实现数据库表的垂直拆分，将同一块业务的数据库表拆分到同一个数据库服务中。拆分方式如下： 

![image-20200209110344132](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200209110344132.png) 



2). 全局表

按照上述的方式进行表结构的拆分，可以解决扩容的问题，但是存在另一个问题：由于省、市、区县、数据字典表，在订单及商品等模块中都需要用到，还会涉及到多表连接查询，那么这个时候涉及到跨库的join操作，可以使用全局表来解决。结构图如下： 

![image-20200209111118652](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200209111118652.png) 



3). 水平拆分

即使我们在上述的方案中使用垂直拆分，将系统中的表结构拆分到了三个数据库服务器中，但是对于当前这个比较繁忙的业务系统来说，每天都会产生大量的用户操作日志，长年累月，这张表的数据在单台服务器中已经存储不下了，这个时候，我们就可以使用MyCat的水平拆分来解决这个问题。

![image-20200209171203841](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200209171203841.png) 



#### 3.5.2 服务器配置

| 名称         |       IP        | 端口 | 用户名/密码 |
| :----------- | :-------------: | :--: | :---------: |
| MyCat-Server | 192.168.192.157 | 8066 | root/123456 |
| MySQL-1      | 192.168.192.158 | 3306 | root/itcast |
| MySQL-2      | 192.168.192.159 | 3306 | root/itcast |
| MySQL-3      | 192.168.192.160 | 3306 | root/itcast |
| MySQL-4      | 192.168.192.161 | 3306 | root/itcast |



#### 3.5.3 schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	
	<schema name="V_SHOP" checkSQLschema="false" sqlMaxLimit="100">
       
	   <table name="tb_areas" dataNode="dn1,dn2,dn3,dn4" primaryKey="areaid" type="global"/>
	   <table name="tb_provinces" dataNode="dn1,dn2,dn3,dn4" primaryKey="provinceid" type="global"/>
	   <table name="tb_cities" dataNode="dn1,dn2,dn3,dn4" primaryKey="cityid"  type="global"/>
	   <table name="tb_dictionary" dataNode="dn1,dn2,dn3,dn4" primaryKey="id"  type="global"/>
		
		
		<table name="tb_brand" dataNode="dn1" primaryKey="id" />
		<table name="tb_category" dataNode="dn1" primaryKey="id" />
		<table name="tb_sku" dataNode="dn1" primaryKey="id" />
		<table name="tb_spu" dataNode="dn1" primaryKey="id" />
		
        
		<table name="tb_order" dataNode="dn2" primaryKey="id" />
		<table name="tb_order_item" dataNode="dn2" primaryKey="id" />
		<table name="tb_order_log" dataNode="dn2" primaryKey="id" />
		
        
		<table name="tb_operatelog" dataNode="dn3,dn4" primaryKey="id" rule="log-sharding-by-murmur"/>
	</schema>
    
	
	<dataNode name="dn1" dataHost="host1" database="v_goods" />
	<dataNode name="dn2" dataHost="host2" database="v_order" />
	<dataNode name="dn3" dataHost="host3" database="v_log" />
	<dataNode name="dn4" dataHost="host4" database="v_log" />
    
	
	<dataHost name="host1" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM1" url="192.168.192.158:3306" user="root" password="itcast">	</writeHost>
	</dataHost>	
    
    <dataHost name="host2" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM2" url="192.168.192.159:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
    
    <dataHost name="host3" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM3" url="192.168.192.160:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
	
	<dataHost name="host4" maxCon="1000" minCon="10" balance="0"
		writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="hostM4" url="192.168.192.161:3306" user="root" password="itcast"></writeHost>
	</dataHost>	
</mycat:schema>
```





#### 3.5.4 分片配置

1). 配置Mycat的schema.xml



2). 配置rule.xml

```xml
<tableRule name="log-sharding-by-murmur">
    <rule>
        <columns>id</columns>
        <algorithm>log-murmur</algorithm>
    </rule>
</tableRule>

<function name="log-murmur" class="io.mycat.route.function.PartitionByMurmurHash">
    <property name="seed">0</property>
    <property name="count">2</property>
    <property name="virtualBucketTimes">160</property>
</function>
```



3). 配置server.xml

```xml
<user name="root" defaultAccount="true">
    <property name="password">GO0bnFVWrAuFgr1JMuMZkvfDNyTpoiGU7n/Wlsa151CirHQnANVk3NzE3FErx8v6pAcO0ctX3xFecmSr+976QA==</property>
    <property name="schemas">V_SHOP</property>
    <property name="readOnly">false</property>
    <property name="benchmark">1000</property>
    <property name="usingDecrypt">1</property>

    <!-- 表级 DML 权限设置

  <privileges check="true">
   <schema name="ITCAST" dml="1111" >
    <table name="TB_TEST" dml="1110"></table>
   </schema>
  </privileges>	
  -->		
</user>
```



4). 在各个MySQL数据库实例中创建数据库

```
MySQL-1 : v_goods
MySQL-2 : v_order
MySQL-3 : v_log
MySQL-4 : v_log
```



5). 导出本地的SQL脚本 , 在MyCat中执行SQL脚本 , 创建数据表 ,并导入数据



6). 连接测试



#### 3.5.5 微服务连接MyCat

```yml
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://192.168.192.157:8066/V_SHOP?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: root
    password: 123456
```



#### 3.5.6 配置MyCat-Web监控

1). 启动Zookeeper

2). 启动MyCat-Web

3). 访问 

http://192.168.192.157:8082/mycat

界面: 

![image-20200219231604626](/Users/liuxiangren/Downloads/教程相关/资料-全面讲解开源数据库中间件MyCat使用及原理/MyCat资料/文档/文档/assets/image-20200219231604626.png) 





## Docker HAProxy MySQL

HAProxy配置主要参考：https://blog.csdn.net/inthat/article/details/88928749

完整版：https://blog.csdn.net/warrior_0319/article/details/80805030



