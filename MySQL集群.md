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

Docker搭建Mysql集群：https://blog.csdn.net/qq_45562491/article/details/115335200

Docker安装Mysql，并搭建一主一从复制集群，一主双从，双主双从集群：https://www.cnblogs.com/tyhj-zxp/p/12719121.html



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

### 2.2 ShardingSphere分库分表

ShardingSphere包括两个核心产品：Sharding JDBC和Sharding Proxy

```markdown
Sharding JDBC：客户端分库分表
可以说是对JDBC的扩展，是个jar包，对我们的数据库操作进行了一个扩展，帮助我们将业务逻辑代码进行分库分表，我在程序中写了一个SQL，他会帮我们把SQL命令分发到不同的Databases中

Sharding Proxy：服务端分库分表
Sharding Proxy会作为一个单独的服务进行部署，在所有的数据库上面相当于做一层代理，他自己就会伪装成一个Mysql服务，然后呢，我的应用可以像连Mysql一样的去连这个Sharding Proxy。那这样我们就不需要考虑分库分表的逻辑了，所有的SQL就在这个Sharding Proxy这个虚拟的Mysql当中去执行，而Sharding Proxy呢，会把我们把这种SQL转换成一个分库分表的SQL往下面底层的各个数据库去分发，这是它们两者的区别，我们就能够知道他们两个两者有什么优点，对吧。你像客户端的分库分表Sharding JDBC，他是在业务端来帮我们做分库分表的，那是不是就更为灵活呀？对不对我代码要做什么调整？我直接在我的APP里面就可以可以完成我的调整，对不对？想怎么改就怎么改，所以呢这个Sharding JDBC最大的优点就是灵活，Sharding JDBC支持所有的JDBC产品。你以前JDBC能够连什么产品，那Sharding JDBC就能够连，而Sharding Proxy这种方式呢，他是不是跟下面数据库打交道的，所以逻辑都需要提前定义好对吧，一旦提前定义好，那肯定就丧失灵活性，所以像Sharding Proxy，它默认就只能够支持MySQL和PGSQL这样两个产品，它只支持这两个产品，其他产品支持不了，因为SQL的情况太多了，对吧，你全部要先给你想好很难很难。这是他们两者的区别。

Registry Center：
我们之前学微服务的时候是不是经常提到一个注册中心啊？通过注册中心可以统一的管理我们的集群对不对？那这个Sharding JDBC和ShardingProxy，他们是做分库分表的分库分表的，这个注册中心那怎么去理解呢？这是后面会带大家去玩的一个问题，就是分库分表的注册中心到底怎么用。
```

### 2.3 小案例

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



## 3 Mycat

Mycat是一个开源数据库中间件，是一个实现了MySQL协议的数据库中间件服务器，我们可以把它看做是一个数据库代理，用Mysql客户端工具和命令行访问Mycat，而Mycat再使用MySQL原生Native协议与多个MySQL服务器通信，也可以用JDBC协议与大多数主流数据库服务器通信，包括SQL Server、Oracle、DB2、PostgreSQL等主流数据库，也支持MongoDB这种新型NoSQL方式的存储，未来还会支持更多类型的存储；

一般的，Mycat主要用于代理Mysql数据库，虽然它也支持去访问其他类型的数据库；

Mycat默认端口是8066，一般的，我们可以使用常见的对象映射框架比如mybatis操作Mycat

### 3.1 Mycat主要能做什么

#### 3.1.1 数据库的读写分离

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

