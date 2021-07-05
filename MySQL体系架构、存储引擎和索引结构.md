

# MySQL体系架构、存储引擎和索引结构

对某项技术进行系统性的学习，始终离不开对该项技术的整体认知。只有领略其全貌，方可将各块知识点更好的串联起来。为了进一步理解和学习MySQL，我们有必要了解一下MySQL的体系构架、存储引擎和索引结构。

## 1 MySQL体系构架

以下是官网MySQL体系构架图，我们稍微对其进行了层级划分。

![Mysql-struct](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-en.png)

英文不好的同学可以看下中文版的：

![mysql-struct-cn](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-ch.png)

整个MySQL Server由一下组成

- Connection Pool：连接池组件
- Management Service & Utilities：管理服务和工具组件
- SQL Interface：SQL接口组件
- Parser：查询分析器组件
- Optimizer：优化器组件
- Caches & Buffers：缓冲池组件
- Pluggable Storage Engines：存储引擎
- File System：文件系统

1）连接层

最上层是一些客户端和链接服务 ，包含本地sock通信和大多数基于客户端/服务端工具实现的类似于TCP/IP的通信。主要完成些类似于连接处 理、授权认证、及相关的安全方案。在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。 同样在该层上可以实现基于SSL的安全链接。服务器也会为安全接入的每个客户端验证它所具有的操作权限。

2）服务层

第二层架构主要完战大多数的核心服务功能，如SQL接口，并完成缓存的查询，SQL的分析和优化.部分内函数的执行。所有跨存储引擎的功能也在这一层实现，如过程、函数等。在该层，服务器会解析查询并创建相应的内部解析树，并对其完成相应的优化如确定表的查询的顺序，是否利 用索引等，最后生成相应的执行操作。如果是select语句 ，服务器还会查询内部的缓存，如果缓存空间足够大，这样在解决大量读操作的环境中能够很好的提升系统的性能。

3）引擎层

存储引擎层，存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API和存储引擎进行通信，不同的存储引擎具有不同的功能，这样我们可以根据自己的需要.来选取合适的存储引擎。

4）存储层

数据存储层，主要是将数据存储在文件系统之上，并完成与存储引掌的交互。



和其他数据库相比，MySQL有点与众不同，它的架构可以在多种不同场景中应用并发挥良好作用。主要体现在存储引擎上，插件式的存储引擎架构，将查询处理和其他的系统任务以及数据的存储提取分离。这种架构可以根据业务的需求和实际需要选择合适的存储引擎。



由上至下，我们可以MySQL的体系构架划分为：1.网络接入层 2.服务层 3.存储引擎层 4.文件系统层

### 1.1 网络接入层

提供了应用程序接入MySQL服务的接口。客户端与服务端建立连接，客户端发送SQL到服务端。

### 1.2 服务层

#### 1.2.1 管理工具和服务

系统管理和控制工具，例如备份恢复、Mysql复制、集群等

#### 1.2.2 连接池

主要负责连接管理、授权认证、安全等等。每个客户端连接都对应着服务器上的一个线程。服务器上维护了一个线程池，避免为每个连接都创建销毁一个线程。当客户端连接到MySQL服务器时，服务器对其进行认证。可以通过用户名与密码认证，也可以通过SSL证书进行认证。登录认证后，服务器还会验证客户端是否有执行某个查询的操作权限。

由于每次建立连接需要消耗很多时间，连接池的作用就是将这些连接缓存下来，下次可以直接用已经建立好的连接，提升服务器性能。

#### 1.2.3 SQL接口

接受用户的SQL命令，并且返回用户操作的结果。

查询解析器

SQL命令传递到解析器的时候会被解析器验证和解析。

MySQL是一个DBMS（数据库管理系统），没法直接理解SQL语句。Parser负责对SQL语句进行解析好让DBMS知道该怎么做。

#### 1.2.4 查询优化器

SQL语句在查询之前会使用查询优化器对查询进行优化。它使用的是“选取-投影-联接”策略进行查询以此选择一个最优的查询路径。

```sql
select uid,name from user where gender = 1;
```

select 查询先根据 where 语句进行选取，而不是先将表全部查询出来以后再进行条件过滤
select查询先根据 uid 和 name 进行属性投影，而不是将属性全部取出以后再进行过滤
将这两个查询条件联接起来生成最终查询结果

```markdown
缓存（8.0版本之前支持查询缓存，8.0之后不支持了）
```

查询缓存，如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。
通过LRU算法将数据的冷端溢出，未来得及时刷新到磁盘的数据页，叫脏页。
这个缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，key缓存，权限缓存等

### 1.3 存储引擎层

负责数据的存储和读取，与数据库文件打交道。 服务器中的查询执行引擎通过API与存储引擎进行通信，通过接口屏蔽了不同存储引擎之间的差异。


MySQL采用插件式的存储引擎。MySQL为我们提供了许多存储引擎，每种存储引擎有不同的特点。我们可以根据不同的业务特点，选择最适合的存储引擎。

MySQL区别于其他数据库的最重要的一个特点就是插件式的表存储引擎，注意：存储引擎是基于表的。

### 1.4 系统文件层

该层主要是将数据库的数据存储在文件系统之上，并完成与存储引擎的交互。

存储引擎是基于表的，以下分别使用MyISAM和InnoDB存储引擎建立两张表，看看其在文件系统中对应的文件存储格式。



![file-system](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-file-system.png)

**存储引擎为MyISAM：**

- .frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等

- *.MYD：MyISAM DATA，用于存储MyISAM表的数据

- *.MYI：MyISAM INDEX，用于存储MyISAM表的索引相关信息

  

**存储引擎为InnoDB：**

- *.frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
- *.ibd：InnoDB DATA，表数据和索引的文件。该表的索引(B+树)的每个非叶子节点存储索引，叶子节点存储索引和索引对应的数据

除了.ibd文件InnoDB还有一种文件的存储格式为.ibdata文件，那么他们之间有什么区别呢？

```markdown
InnoDB的数据存储方式能够通过配置来决定是使用共享表空间存放存储数据，还是用独享表空间存放存储数据。独享表空间存储方式使用.ibd文件，并且每个表为一个ibd文件。共享表空间存储方式采用.ibdata文件，所有的表共同使用一个ibdata文件，即所有的数据文件都存在一个文件中。决定使用哪种表的存储方式可以通过mysql的配置文件中innodb_file_per_table选项来指定。
InnoDB默认使用的是独享表的存储方式，这种方式的好处是当数据库产生大量文件碎片的时，整理磁盘碎片对线上运行环境的影响较小。
```

### 1.5【拓展】一个SQL语句在MySQL中的整体流程

用户使用mysql查询的一个整体流程如下（[原图链接](https://www.cnblogs.com/klvchen/articles/12809342.html)）：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-extend-flow.png)

简化版：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-extend-flow-simple.png)

## 2 存储引擎

### 2.1 了解存储引擎

MySQL中的数据用各种不同的技术存储在文件（或者内存）中。每一种技术都使用不同的存储机制、索引技巧、锁定水平并且最终提供广泛的不同的功能和能力。通过选择不同的技术，能够获得额外的速度或者功能，从而改善应用的整体功能。 这些不同的技术以及配套的相关功能在MySQL中被称作存储引擎(也称作表类型)。

MySQL区别于其他数据库的最重要的一个特点就是插件式的表存储引擎，也就是说存储引擎是基于表的。

存储引擎的概念是MySQL里面才有的，不是所有的关系型数据库都有存储引擎这个概念 。其它数据库系统 (包括大多数商业选择)仅支持一种类型的数据存储， 也就是说采用“ 一个尺码满足一切需求 ”的存储方式，也意味着“功能强大，性能平庸”。而MySQL默认配置了许多不同的存储引擎，你可以根据业务需求选取一种最适配最高效的存储引擎。这也是为什么MySQL为何如此受欢迎的主要原因之一。

**存储引擎新概述**

和大多数的数据库不同，MySQL中有一个存储引擎的概念，针对不同的存储需求可以选择最优的存储引擎。

存储引擎就是存储数据，建立索引，更新查询数据等等技术的实现方式。存储引擎是基于表的，而不是基于库的。所以存储引擎也可被称为表类型。

Oracle，SqlServer等数据库只有一种存储引擎，MySQL提供了插件式的存储引擎架构。所以MySQL存在多种存储引擎，可以根据需要使用相应引擎，或者编写存储引擎。

MySQL5.0支持的存储引擎包含: InnoDB、 MyISAM、BDB、MEMORY MERGE EXAMPLE、NDB Cluster、ARCHIVE、 CSV、BLACKHOLE、FEDERATED等 ，其中InnoDB和BDB提供事务安全表，其他存储引擎是非事务安全表。

可以通过指定show engines，来查询当前数据库支持的存储引擎

创建新表时如果不指定存储引擎，那么系统就会使用默认的存储引擎，MySQL5.5之前的默认存储引擎是MylSAM，5.5之后就改为了InnoDB。

### 2.2 存储引擎分类

查看当前安装的MySQL版本支持的存储引擎

```sql
-- 查看MySQL版本
select version();

-- 查看版本支持的存储引擎
show engines;

-- 查看存储引擎
show variables like '%storage_engine%';
+---------------------------------+-----------+
| Variable_name                   | Value     |
+---------------------------------+-----------+
| default_storage_engine          | InnoDB    |
| default_tmp_storage_engine      | InnoDB    |
| disabled_storage_engines        |           |
| internal_tmp_mem_storage_engine | TempTable |
+---------------------------------+-----------+
4 rows in set (0.01 sec)

```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-struct-version.png)


我本地安装的社区版MySQL，版本号为5.7.23，支持9种存储引擎或者说是8种（FEDERATED NO SUPPORT 不支持FEDERATED），而官网提供了10种存储引擎。本地与官网支持的存储引擎略微不同，个人估计是社区版和商用版的差别的缘故，或者是安装时候配置项设置导致的差异，有清楚的小伙伴还望告知一下。

官网5.7版本支持的10种存储引擎：

- MyISAM： 拥有较高的插入，查询速度，但不支持事务
- InnoDB ：5.5.8版本后Mysql的默认数据库引擎，支持ACID事务，支持行级锁定和外键
- BDB： 源自Berkeley DB，事务型数据库的另一种选择，支持COMMIT和ROLLBACK等其他事务特性
- Memory ：所有数据置于内存的存储引擎，拥有极高的插入，更新和查询效率。但是会占用和数据量成正比的内存空间。并且其内容会在Mysql重新启动时丢失
- Merge ：将一定数量的MyISAM表联合而成一个整体，在超大规模数据存储时很有用
- Archive ：非常适合存储大量的独立的，作为历史记录的数据。因为它们不经常被读取。Archive拥有高效的插入速度，但其对查询的支持相对较差
- Federated： 将不同的Mysql服务器联合起来，逻辑上组成一个完整的数据库。非常适合分布式应用
- Cluster/NDB ：高冗余的存储引擎，用多台数据机器联合提供服务以提高整体性能和安全性。适合数据量大，安全和性能要求高的应用
- CSV： 逻辑上由逗号分割数据的存储引擎。它会在数据库子目录里为每个数据表创建一个.CSV文件。这是一种普通文本文件，每个数据行占用一个文本行。CSV存储引擎不支持索引。
- BlackHole ：黑洞引擎，写入的任何数据都会消失，一般用于记录binlog做复制的中继
  常用存储引擎特性

### 2.3 存储特性要求

存储引擎常见的目标特性要求

```markdown
并发性：某些应用程序比其他应用程序具有更高的颗粒级锁定要求（如行级锁定）。

事务支持：并非所有的应用程序都需要事务，但对的确需要事务的应用程序来说，有着定义良好的需求，如ACID兼容等。

引用完整性：通过DDL定义的外键，服务器需要强制保持关联数据库的引用完整性。

物理存储：它包括各种各样的事项，从表和索引的总的页大小，到存储数据所需的格式，到物理磁盘。

索引支持：不同的应用程序倾向于采用不同的索引策略，每种存储引擎通常有自己的编制索引方法，但某些索引方法（如B-tree索引）对几乎所有的存储引擎来说是共同的。

内存高速缓冲：与其他应用程序相比，不同的应用程序对某些内存高速缓冲策略的响应更好，因此，尽管某些内存高速缓冲对所有存储引擎来说是共同的（如用于用户连接的高速缓冲，MySQL的高速查询高速缓冲等），其他高速缓冲策略仅当使用特殊的存储引擎时才唯一定义。

性能帮助：包括针对并行操作的多I/O线程，线程并发性，数据库检查点，成批插入处理等。

其他目标特性：可能包括对地理空间操作的支持，对特定数据处理操作的安全限制等。
```

以上特性很多是互斥的，一个存储引擎只能具备其中某些要求。

具体查考官网[Storage Engines Feature Summary](https://dev.mysql.com/doc/refman/5.7/en/storage-engines.html)

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/mysql-store-engines-fetures.png)

Notes:

```markdown
在服务器中实现，而不是在存储引擎中。

只有使用压缩行格式时，才支持压缩的MyISAM表。使用MyISAM压缩行格式的表是只读的。

在服务器端通过加密功能实现。

在服务器端通过加密功能实现;在MySQL 5.7和更高版本中，支持数据静止表空间加密。

在MySQL Cluster NDB 7.3和更高版本中支持外键。

InnoDB在MySQL 5.6和更高版本中提供对 全文索引（FULLTEXT ） 的支持。

InnoDB在MySQL 5.7和更高版本中提供对地理空间索引的支持。

InnoDB内部利用哈希索引来实现自适应哈希索引特性。
```

下文主要介绍InnoDB MyISAM Memory三种存储引擎，以下是三者简要特性对比

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-engines-compare.png)

**存储引擎对比**

| 特点         | InnoDB           | MyISAM | Memory | Merge | NDB  |
| ------------ | ---------------- | ------ | ------ | ----- | ---- |
| 存储限制     | 64TB             | 有     | 有     | 没有  | 有   |
| 事务安全     | 支持             |        |        |       |      |
| 锁机制       | 行锁(适合高并发) | 表锁   | 表锁   | 表锁  | 行锁 |
| B树索引      | 支持             | 支持   | 支持   | 支持  | 支持 |
| 哈希索引     |                  |        | 支持   |       |      |
| 全文索引     | 支持(5.6之后)    | 支持   |        |       |      |
| 集群索引     | 支持             |        |        |       |      |
| 数据索引     | 支持             |        | 支持   |       | 支持 |
| 索引缓存     | 支持             | 支持   | 支持   | 支持  | 支持 |
| 数据可压缩   |                  | 支持   |        |       |      |
| 空间使用     | 高               | 低     | N/A    | 低    | 低   |
| 内存使用     | 高               | 低     | 中等   | 低    | 高   |
| 批量插入速度 | 低               | 高     | 高     | 高    | 高   |
| 支持外键     | 支持             |        |        |       |      |

重点是InnoDB、MyISAM，另外Memory和Merge了解即可



#### 2.3.1 InnoDB引擎

InnoDB 是一个事务安全的存储引擎，它具备提交、回滚以及崩溃恢复的功能以保护用户数据。InnoDB 的行级别锁定保证数据一致性提升了它的多用户并发数以及性能。InnoDB 将用户数据存储在聚集索引中以减少基于主键的普通查询所带来的 I/O 开销。为了保证数据的完整性，InnoDB 还支持外键约束。默认使用B+TREE数据结构存储索引。

InnoDB存储引擎是Mysql的默认存储引擎。InnoDB存储引擎提供了具有提交、回滚、崩溃恢复能力的事务安全。但是对比MyISAM的存储引擎，InnoDB写的处理效率差一些，并且会占用更多的磁盘空间以保留数据和索引。

##### 2.3.1.1 特点

- 支持事务，支持4个事务隔离（ACID）级别
- 行级锁定（更新时锁定当前行）
- 读写阻塞与事务隔离级别相关
- 既能缓存索引又能缓存数据
- 支持外键
- InnoDB更消耗资源，读取速度没有MyISAM快
- 在InnoDB中存在着缓冲管理，通过缓冲池，将索引和数据全部缓存起来，加快查询的速度；
- 对于InnoDB类型的表，其数据的物理组织形式是聚簇表。所有的数据按照主键来组织。数据和索引放在一块，都位于B+数的叶子节点上；

##### 2.3.1.2 业务场景

- 需要支持事务的场景（银行转账之类）

- 适合高并发，行级锁定对高并发有很好的适应能力，但需要确保查询是通过索引完成的

- 数据修改较频繁的业务

  

##### 2.3.1.3 InnoDB引擎调优

- 主键尽可能小，否则会给Secondary index带来负担

- 避免全表扫描，这会造成锁表

- 尽可能缓存所有的索引和数据，减少IO操作

- 避免主键更新，这会造成大量的数据移动

  

##### 2.3.1.4 补充：事务（ACID）

```markdown
A 事务的原子性(Atomicity)：指一个事务要么全部执行,要么不执行.也就是说一个事务不可能只执行了一半就停止了.比如你从取款机取钱,这个事务可以分成两个步骤:1划卡,2出钱.不可能划了卡,而钱却没出来.这两步必须同时完成.要么就不完成.
C 事务的一致性(Consistency)：指事务的运行并不改变数据库中数据的一致性.例如,完整性约束了a+b=10,一个事务改变了a,那么b也应该随之改变.
I 独立性(Isolation）:事务的独立性也有称作隔离性,是指两个以上的事务不会出现交错执行的状态.因为这样可能会导致数据不一致.
D 持久性(Durability）:事务的持久性是指事务执行成功以后,该事务所对数据库所作的更改便是持久的保存在数据库之中，不会无缘无故的回滚.
```



#### 2.3.2 MyISAM引擎

MyISAM既不支持事务、也不支持外键、其优势是访问速度快，但是表级别的锁定限制了它在读写负载方面的性能，因此它经常应用于只读或者以读为主的数据场景。默认使用B+TREE数据结构存储索引。

##### 2.3.2.1 特点

- 不支持事务
- 表级锁定（更新时锁定整个表）
- 读写互相阻塞（写入时阻塞读入、读时阻塞写入；但是读不会互相阻塞）
- 只会缓存索引（通过key_buffer_size缓存索引，但是不会缓存数据）
- 不支持外键
- 读取速度快

##### 2.3.2.2 业务场景

- 不需要支持事务的场景（像银行转账之类的不可行）
- 一般读数据的较多的业务
- 数据修改相对较少的业务
- 数据一致性要求不是很高的业务

##### 2.3.2.3 MyISAM引擎调优

- 设置合适索引
- 启用延迟写入，尽量一次大批量写入，而非频繁写入
- 尽量顺序insert数据，让数据写入到尾部，减少阻塞
- 降低并发数，高并发使用排队机制
- MyISAM的count只有全表扫描比较高效，带有其它条件都需要进行实际数据访问

#### 2.3.3 Memory引擎

在内存中创建表。每个MEMORY表只实际对应一个磁盘文件(frm 表结构文件)。MEMORY类型的表访问非常得快，因为它的数据是放在内存中的，并且默认使用HASH索引。要记住，在用完表格之后就删除表格，不然一直占据内存空间。

##### 2.3.3.1 特点

- 支持的数据类型有限制，比如：不支持TEXT和BLOB类型（长度不固定），对于字符串类型的数据，只支持固定长度的行，VARCHAR会被自动存储为CHAR类型；
- 支持的锁粒度为表级锁。所以，在访问量比较大时，表级锁会成为MEMORY存储引擎的瓶颈；
- 由于数据是存放在内存中，一旦服务器出现故障，数据都会丢失；
- 查询的时候，如果有用到临时表，而且临时表中有BLOB，TEXT类型的字段，那么这个临时表就会转化为MyISAM类型的表，性能会急剧降低；
- 默认使用hash索引。
- 如果一个内部表很大，会转化为磁盘表。

##### 2.3.3.2 业务场景

- 那些内容变化不频繁的代码表，或者作为统计操作的中间结果表，便于高效地堆中间结果进行分析并得到最终的统计结果。
- 目标数据比较小，而且非常频繁的进行访问，在内存中存放数据，如果太大的数据会造成内存溢出。可以通过参数max_heap_table_size控制Memory表的大小，限制Memory表的最大的大小。
- 数据是临时的，而且必须立即可用得到，那么就可以放在内存中。
- 存储在Memory表中的数据如果突然间丢失的话也没有太大的关系。

### 2.4 存储引擎构架

为了进一步深入理解MySQL存储引擎，我们有必要了解一下存储引擎的数据存储结构，在此之前，我们得先了解下数据在文件系统中的存储。

#### 2.4.1 磁盘基本知识

数据库的数据存储在文件系统中。文件系统是操作系统用来 明确 存储设备（常见的是磁盘，也有基于NAND Flash的固态硬盘）或分区上的文件 的方法和数据结构。磁盘上数据必须用一个三维地址唯一标示：柱面号、盘面号、块号(磁道上的盘块)。

硬盘只是磁盘的一种，或说是经典代表，以下通过硬盘模型图讲解磁盘中的各个概念。

硬盘整体模型图

![disk-model](/Users/liuxiangren/mysql-learning/mysql-basic-img/disk-model.png)

硬盘模型图

![disk-model-detail](/Users/liuxiangren/mysql-learning/mysql-basic-img/disk-model-detail.png)

#### 2.4.2 磁盘重点概念

- 盘片（platter）：硬盘中承载数据存储的介质
  硬盘一般由多个盘片组成，每个盘片包含两个面，每个盘面都对应地有一个读/写磁头。受到硬盘整体体积和生产成本的限制，盘片数量都受到限制，一般都在5片以内。盘片的编号自下向上从0开始，如最下边的盘片有0面和1面，再上一个盘片就编号为2面和3面。
- 磁头（head）：通过磁性原理读取磁性介质上数据的部件
- 磁道（track）：当磁盘旋转时，磁头若保持在一个位置上，则每个磁头都会在磁盘表面划出一个圆形轨迹，这些圆形轨迹就叫做磁道
- 扇区（sector）：磁盘上的每个磁道被等分为若干个弧段，这些弧段便是硬盘的扇区，同一块硬盘上的扇区大小是一致的
  "每个磁道的扇区数一样的"说的是老的硬盘，外圈的密度小，内圈的密度大（简单理解就是，磁盘存储媒介为
  磁性记忆材料，在内圈涂的密度高），故每圈可存储的数据量是一样的。新的硬盘数据的密度都一致，这样磁道的周长越长，扇区就越多，存储的数据量就越大。
- 柱面（cylinder）：在有多个盘片构成的盘组中，由不同盘片的面，但处于同一半径圆的多个磁道组成的一个圆柱面

##### 2.4.2.1 物理扇区（physical sector）与逻辑扇区（logical sector）

近年来，为了最求更高的硬盘容量，便出现了扇区存储容量为2048、4096等字节的硬盘，我们称这样的扇区为"物理扇区"。这样的大扇区会导致许多兼容性问题，有的系统或软件无法适应。为了解决这个问题，硬盘内部将物理扇区在逻辑上划分为多个扇区片段并将其作为普通的扇区（一般为512字节大小）报告给操作系统及应用软件。这样的扇区片段我们称之为“逻辑扇区”。实际读写时由硬盘内的程序（固件）负责在逻辑扇区与物理扇区之间进行转换，上层程序“感觉”不到物理扇区的存在。

逻辑扇区是硬盘可以接受读写指令的最小操作单元，是操作系统及应用程序可以访问的扇区，多数情况下其大小为512字节。我们通常所说的扇区一般就是指的逻辑扇区。物理扇区是硬盘底层硬件意义上的扇区，是实际执行读写操作的最小单元。是只能由硬盘直接访问的扇区，操作系统及应用程序一般无法直接访问物理扇区。当要读写某个逻辑扇区时，硬盘底层在实际操作时都会读写逻辑扇区所在的整个物理扇区。

##### 2.4.2.2 磁盘容量计算

- 旧式——非ZBR区位记录（不同磁道扇区数相同）
  存储容量 ＝ 磁头数 × 磁道(柱面)数 × 每道扇区数 × 每扇区字节数
  比如上图最右边硬盘容量：6 * 7 * 12 * 512 = 258048 byte
- 新式——ZBR区位记录（不同磁道扇区数不同）

##### 2.4.2.3 块（Block）/簇（Cluster）

块/簇两者指的是同一个逻辑上的概念，只是在Linux与Windows中的称呼不同。

- 块/簇 是操作系统中最小的逻辑存储单位。操作系统与磁盘打交道的最小单位是块/簇。
- 在Windows下如NTFS等文件系统中叫做簇；在Unix和Linux下如Ext4等文件系统中叫做块（block）。
- 每个簇或者块可以包括2、4、8、16、32、64…2的n次方个扇区。

##### 2.4.2.4 块/簇 用来干什么的

磁盘的最小单位是扇区，操作系统使用的是 块/簇 作为IO的基本单位。

- 读取方便：扇区容量小，数据多会加大寻址难度。操作系统将相邻的扇区组合一起形成块，再对块整体操作
- 分离对底层的依赖：操作系统忽略对底层物理存储结构的设计。通过虚拟出来磁盘块的概念，在系统中认为块是最小的单位

扇区是对硬盘而言，块是对文件系统而言，出于不同的需要。

##### 2.4.2.5 查看块/簇的大小

不同文件系统中block的大小不一样。

```shell
Windows:（使用管理员命令提示行）
fsutil fsinfo ntfsinfo E:

Linux：
stat /home | grep "IO Block"
```

如下所示，Windows下E盘的Cluster的大小为4Kb大小，如下所示：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/window-edisk-cluster.png)

##### 2.4.2.6 页（Page）

操作系统经常与内存和硬盘这两种存储设备进行通信，类似于“块”的概念，都需要一种虚拟的基本单位。与内存操作，是虚拟一个页的概念来作为最小单位。与硬盘打交道，就是以块为最小单位。

##### 2.4.2.7 扇区、块/簇、页的关系

- 扇区： 硬盘的最小读写单元
- 块/簇： 是操作系统针对硬盘读写的最小单元
- 页： 是内存与操作系统之间操作的最小单元。
- 扇区 <= 块/簇 <= 页

#### 2.4.3 MySQL的InnoDB数据存储结构

MySQL的InnoDB数据存储结构可以划分为逻辑存储结构和物理存储结构。

**前置：数据库磁盘读取与系统磁盘读取**

- 系统从磁盘中读取数据到内存时是以磁盘块（block）为基本单位，位于同一个磁盘块中的数据会被一次性读取出来。
- InnoDB存储引擎中有页（Page）的概念，页是数据库管理磁盘的最小单位，InnoDB存储引擎中默认每个页的大小为16kb，每次读取磁盘时都将页载入内存中。
- 系统一个磁盘块的大小空间往往没有16kb这么大，因此InnoDB每次io操作时都会将若干个地址连续的磁盘块的数据读入内存，从而实现整页读入内存。

##### 2.4.3.1 **物理存储结构**

从物理意义上来看，InnoDB表由共享表空间、日志文件组（更准确地说，应该是Redo文件组）、表结构定义文件组成。若将innodb_file_per_table设置为on，则每个表将独立地产生一个表空间文件，以ibd结尾，数据、索引、表的内部数据字典信息都将保存在这个单独的表空间文件中。表结构定义文件以frm结尾，这个是与存储引擎无关的，任何存储引擎的表结构定义文件都一样，为.frm文件。

##### 2.4.3.2 **逻辑存储结构**

InnoDB存储引擎的逻辑存储结构和Oracle大致相同，所有数据都被逻辑地存放在一个空间中，我们称之为表空间。表空间又由段、区、页组成。1 extent = 64 pages，InnoDB存储引擎的逻辑存储结构大致如图所示。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/table-sapce-desc.png)

##### 2.4.3.3 表空间（tablespace）

表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都是存放在表空间中。默认情况下InnoDB存储引擎有一个共享表空间ibdata1，即所有数据都放在这个表空间内。如果我们启用了参数innodb_file_per_table，则每张表内的数据可以单独放到一个表空间内。

对于启用了innodb_file_per_table的参数选项，需要注意的是，每张表的表空间内存放的只是数据、索引和插入缓冲，其他类的数据，如撤销（Undo）信息、系统事务信息、二次写缓冲（double write buffer）等还是存放在原来的共享表空间内。这也就说明了另一个问题：即使在启用了参数innodb_file_per_table之后，共享表空间还是会不断地增加其大小。

![tablespace-real](/Users/liuxiangren/mysql-learning/mysql-basic-img/table-space-real.png)

##### 2.4.3.4 段（segment）

表空间是由各个段组成的，常见的段有数据段、索引段、回滚段等。

InnoDB存储引擎表是由索引组织的（index organized），因此数据即索引，索引即数据。InnoDB采取B+树作为存储数据的结构，数据段即为B+树的叶节点（上图的leaf node segment），索引段即为B+树的非叶子节点（上图的non-leaf node segment）。

InnoDB存储引擎对于段的管理是由引擎本身完成。

##### 2.3.4.5 区（extent）

一个区是由64个连续的页组成的，每个页大小为16KB，即每个区的大小为1MB。对于大的数据段，InnoDB存储引擎最多每次可以申请4个区，以此来保证数据的顺序性能。

在我们启用了参数innodb_file_per_talbe后，创建的表默认大小是96KB，新建的InnoDB表就是一个区。区是64个连续的页，那创建的表的大小至少是1MB才对啊？其实这是因为在每个段开始时，先有32个页大小的碎片页（fragment page）来存放数据，当这些页使用完之后才是64个连续页的申请。

```sql
create table innodb_table(
	id int primary key
)engine=innodb default charset=utf8;
```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/table-extent.png)

##### 2.3.4.6 页（page）

每个页大小为16KB，页是InnoDB磁盘管理的最小单位，整页整页的读取。

InnoDB中主要的页类型：

- 数据页（BTreeNode）
- Undo页（undo Log page）
- 系统页（System page）
- 事务数据页（Transaction SystemPage）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/table-page-detail.png)

1. 0-38：页头占据38位字节，页面id（32位的整数），页面类型，以及两个分别指向前一个page和后一个page的指针（page是一个双向列表）等信息
2. 38-16376：不同的类型页所含的数据不同，这部分空间包含系统记录（SystemRecord）和用户记录（UserRecord），我们表中的一条条记录就放在UserRecord部分
3. 16376-16384：页面结束标识

由页组成的链表，页之间是双向列表，页里面的数据是单向链表，这种结构组成了主键索引B+树，组成了叶子节点数据。



![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/page-struct-detail.png)



#### 2.4.4 拓展：定位一条表记录的过程

```sql
select * from user where id = 29
```

这里id是主键,我们通过这棵B+树来查找，首先找到根页，你怎么知道user表的根页在哪呢？

其实每张表的根页位置在表空间文件中是固定的。系统经过解析sql语句，首先从找到user表的跟页面（一个表通常需要多个页面组成，跟页面就是起始页），层级遍历非叶子节点页（索引）读取到key值为29的指针（遍历非叶子节点的过程随着节点的遍历会将一个或多个页加载到内存），最后到指针指向的叶子节点所在的页中，然后遍历找出该条记录。

如果使用了二级索引则先读取二级索引page遍历这个二级索引，找到装有主键信息叶子节点page页，遍历找到该主键。然后再根据主键索引寻找到该条记录

## 3 索引结构

### 3.1 常见的索引结构

Mysql数据库中的常见索引结构有多种，常用Hash，B-树，B+树等数据结构来进行数据存储。树的深度加深一层，意味着多一次查询，对于数据库磁盘而言，就是多一次IO操作，导致查询效率低下。

#### 3.1.1 前置：二叉搜索树

了解下二叉搜索树有助于我们理解B-树、B+树，二叉搜索树的特点是：

- 所有非叶子结点至多拥有两个儿子（Left和Right）；
- 所有结点存储一个关键字；
- 非叶子结点的左指针指向小于其关键字的子树，右指针指向大于其关键字的子树；

以下都是二叉搜索树：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/binary-tree.png)

如果要找到65，左边的二叉树需要扫描3层（3次IO），而右边的却需要6层。

#### 3.1.2 B-Tree（B树）

```markdown
B-tree树即B树，B即Balanced，平衡的意思。因为B树的原英文名称为B-tree，而国内很多人喜欢把B-tree译作B-树，其实，这是个非常不好的直译，很容易让人产生误解。事实上，B-tree就是指的B树。
```

B树是一种多路搜索树，一棵m阶的B树满足下列条件：

- 树中每个结点至多有m个孩子

- 根结点的儿子数为[2, M]；

- 除根结点以外的非叶子结点的儿子数为[M/2, M]；

- 每个结点存放至少M/2-1（取上整）和至多M-1个关键字；（至少2个关键字）

- 非叶子结点的关键字个数 = 指向子节点的指针个数-1；

- 非叶子结点的关键字：K[1], K[2], …, K[M-1]；且K[i] < K[i+1]；

- 非叶子结点的指针：P[1], P[2], …, P[M]；其中P[1]指向关键字小于K[1]的子树，P[M]指向关键字大于K[M-1]的

  子树，其它P[i]指向关键字属于(K[i-1], K[i])的子树；

- 所有叶子结点位于同一层；

以下是3阶B树

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/triple-binary-tree.png)

磁盘读取数据是以盘块(block)为基本单位的。

以下结合磁盘块作图

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/triple-binary-tree-with-disk.png)




B树的特征：

- 关键字集合分布在整颗树中；
- 任何一个关键字出现且只出现在一个结点中；
- 搜索有可能在非叶子结点结束；
- 其搜索性能等价于在关键字全集内做一次二分查找；
- 自动层次控制；

B树的搜索，从根结点开始，对结点内的关键字（有序）序列进行二分查找，如果命中则结束，否则进入查询关键字所属范围的儿子结点；重复，直到所对应的儿子指针为空，或已经是叶子结点；

#### 3.1.3 B+ Tree

B+树是B-树的变体，也是一种多路搜索树：（❀ 表示两者间的不同点）

```markdown
树中每个结点至多有m个孩子

根结点的儿子数为[2, M]；

除根结点以外的非叶子结点的儿子数为[M/2, M]；

每个结点存放至少M/2-1（取上整）和至多M-1个关键字；（至少2个关键字）

非叶子结点的关键字：K[1], K[2], …, K[M-1]；且K[i] < K[i+1]；

❀ 非叶子结点的子树指针与关键字个数相同；

❀ 非叶子结点的子树指针P[i]，指向关键字值属于[K[i], K[i+1])的子树；（B树是开区间）；

❀ 为所有叶子结点增加一个链指针；

❀ 所有关键字都在叶子结点出现；
```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/b+tree-desc.png)


B+树的特征：

- 所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
- 不可能在非叶子结点命中；
- 非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
- 每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。
- 更适合文件索引系统；

B+树的搜索与B-树也基本相同，区别是B+树只有达到叶子结点才命中（B-树可以在非叶子结点命中），其性能也等价于在关键字全集做一次二分查找；

#### 3.1.4 为什么B+ 树比B 树更适合作为索引？

1. B+ 树的磁盘读写代价更低
   B+ 树的数据都集中在叶子节点，分支节点 只负责指针（索引）；B 树的分支节点既有指针也有数据 。这将导致B+ 树的层高会小于B 树的层高，也就是说B+ 树平均的Io次数会小于B 树。
2. B+ 树的查询效率更加稳定
   B+ 树的数据都存放在叶子节点，故任何关键字的查找必须走一条从根节点到叶子节点的路径。所有关键字的查询路径相同，每个数据查询效率相当。
3. B+树更便于遍历
   由于B+树的数据都存储在叶子结点中，分支结点均为索引，遍历只需要扫描一遍叶子节点即可；B树因为其分支结点同样存储着数据，要找到具体的数据，需要进行一次中序遍历按序来搜索。
4. B+树更擅长范围查询
   B+树叶子节点存放数据，数据是按顺序放置的双向链表。B树范围查询只能中序遍历。
5. B+ 树占用内存空间小
   B+ 树索引节点没有数据，比较小。在内存有限的情况下，相比于B树索引可以加载更多B+ 树索引。

#### 3.1.5 Hash

哈希索引就是采用一定的哈希算法，把键值换算成新的哈希值，检索时不需要类似B+树那样从根节点到叶子节点逐级查找，只需一次哈希算法即可立刻定位到相应的位置，速度非常快。Memory存储引擎使用Hash。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/hash-index-desc.png)

Hash索引仅仅能满足"=",“IN"和”<=>"查询，不能使用范围查询。也不支持任何范围查询，例如WHERE price > 100。
　　
由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样。

从上面的图来看，B+树索引和哈希索引的明显区别是：

```markdown
如果是等值查询，那么哈希索引明显有绝对优势，因为只需要经过一次算法即可找到相应的键值；这有个前提，键值都是唯一的。如果键值不是唯一的，就需要先找到该键所在位置，然后再根据链表往后扫描，直到找到相应的数据；

如果是范围查询检索，这时候哈希索引就毫无用武之地了，因为原先是有序的键值，经过哈希算法后，有可能变成不连续的了，就没办法再利用索引完成范围查询检索；

哈希索引也没办法利用索引完成排序，以及like ‘xxx%’ 这样的部分模糊查询（这种部分模糊查询，其实本质上也是范围查询）；

哈希索引也不支持多列联合索引的最左匹配规则；

B+树索引的关键字检索效率比较平均，不像B树那样波动幅度大，在有大量重复键值情况下，哈希索引的效率也是极低的，因为存在所谓的哈希碰撞问题。
```



### 3.2 InnoDB B+Tree结构来存储索引

InnoDB使用B+Tree数据结构存储索引，根据索引物理结构可将索引划分为聚簇索引和非聚簇索引（也可称辅助索引或二级索引）。一个表中只能存在一个聚簇索引（主键索引），但可以存在多个非聚簇索引。

B+树 叶子节点包含数据表中行记录就是聚簇索引（索引和数据是一块的）。

![](/Users/liuxiangren/mysql-learning/mysql-basic-img/innodb-b+tree-index.png)

B+树 叶子节点没包含数据表中行记录就是非聚簇索引（索引和数据是分开的）。

![](/Users/liuxiangren/mysql-learning/mysql-basic-img/innodb-b+tree-index-detail.png)

#### 3.2.1 B+ 树可以存储多少行数据

InnoDB存储引擎也有自己的最小储存单元——页（Page），一个页的大小默认是16K。

```sql
mysql> show variables like 'innodb_page_size';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_page_size | 16384 |
+------------------+-------+
```

磁盘扇区、文件系统、InnoDB存储引擎都有各自的最小存储单元

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/b+tree-max-store-capability.png)


数据表中的数据都是存储在页中的，所以一个页中能存储多少行数据呢？假设一行数据的大小是1k，那么一个页可以存放16行这样的数据。

如果数据库只按这样的方式存储，那么如何查找数据就成为一个问题？
因为我们不知道要查找的数据存在哪个页中，也不可能把所有的页遍历一遍，那样太慢了。

于是人们想到了用B+ 树的方式组织这些数据，下图以InnoDB为例。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/innodb-b+tree-pointer.png)

pointer往往是6个字节，指明对应key值的页面位置信息。key一般为索引主键，如果为单字段 bigint 类型，则为8字节。如此可计算一个页大概可以存放16 * 1024/（6+8）=1170行数据。假设一行数据1k，那么2层B+ 树（第一层索引，第二层叶子节点 存数据）就可以存储1170 * 16 = 18 720行；三层则可以存储1170 * 1170 * 16=21902400行。

#### 3.2.2 MyISAM B+Tree结构来存储索引

MyISAM也使用B+Tree数据结构存储索引，但都是非聚簇索引。

以下是MyISAM主键索引存储图

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/myisam-pk-store-desc.png)

可见，索引和数据是分开的 索引的data部分只是索引的地址值。其实上文也提到过，.MYI就是MyISAM表的索引文件，MYD是MyISAM表的数据文件。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/mysql-basic-img/myisam-store-file-sytem.png)

## 4 InnoDB 最新版

InnoDB存储引擎是MyISAM的默认存储引擎。InnoDB存储引擎提供了具有提交、回滚、崩溃恢复能力的事务安全。但是对比MyISAM的存储引擎，InnoDB写的处理效率差一些，并且会占用更多的磁盘空间以保留数据和索引。

InnoDB存储引擎不同与其他存储引擎的特点

### 4.1 事务控制

```sql
create table goods_innodb(
  id int not null auto_increment,
  name varchar(20) not null,
  primary key (id)
)engine=innodb default charset=utf8;
```

```sql
start transaction;
insert into goods_innodb(id,name) values(null,'HuaWei Mate 40 PRO');
commit; #未提交，别的连接就查不到，默认隔离级别，REPETABLE_READ
```

### 4.2 外键约束

唯一支持外键约束的存储引擎

MySQL支持外键的存储引擎只有InnoDB，在创建外键的时候，要求父表必须有对应的索引，子表在创建外键的时候，也会自动的创建对应的索引

下面两张表中，country_innodb是父表，country_id为主键索引，city_innodb表是子表，country_id字段为外键，对应于country_innodb表的主键country_id

```sql
create table country_innodb(
  country_id int not null auto_increment,
  country_name varchar(100) not null,
  primary key(country_id)
)engine=innodb default charset=utf8;

create table city_innodb(
  city_id int not null auto_increment,
  city_name varchar(50) not null,
  country_id int not null,
  primary key (city_id),
  key fk_idx_country_id(country_id),
  constraint `fk_city_country` foreign key (country_id) references country_innodb(country_id) on delete restrict on update cascade
)engine=innodb default charset=utf8;

desc city_innodb;
+------------+-------------+------+-----+---------+----------------+
| Field      | Type        | Null | Key | Default | Extra          |
+------------+-------------+------+-----+---------+----------------+
| city_id    | int         | NO   | PRI | NULL    | auto_increment |
| city_name  | varchar(50) | NO   |     | NULL    |                |
| country_id | int         | NO   | MUL | NULL    |                |
+------------+-------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)

 desc country_innodb;
+--------------+--------------+------+-----+---------+----------------+
| Field        | Type         | Null | Key | Default | Extra          |
+--------------+--------------+------+-----+---------+----------------+
| country_id   | int          | NO   | PRI | NULL    | auto_increment |
| country_name | varchar(100) | NO   |     | NULL    |                |
+--------------+--------------+------+-----+---------+----------------+
2 rows in set (0.00 sec)


insert into country_innodb values(null,'China'),(null,'America'),(null,'Japan');

insert into city_innodb values(null,'西安市',1),(null,'NewYork',2),(null,'BeiJing',1);

```

```sql
#上面的
on delete restrict -----> 删除主表数据时，如果有关联记录，则不删除

on update cascade  -----> 更新主表时，如果子表有关联记录，也会更新子表记录 
```

### 4.3 存储方式

InnoDB存储表和索引有以下两种方式：

1. 使用共享表空间存储，这种方式创建的表结构保存.frm文件中，数据和索引保存在Innodb_data_home_dir和innodb_data_file_path定义的表空间中，可以是多个文件
2. 使用多表空间存储，这种方式创建的表的表结构仍然在.frm文件中，但是每个表的数据和索引单独保存在.ibd中

```markdown
.frm总是存着表结构，.ibd存着数据还有索引
```

## 5 MyISAM

MyISAM不支持事务、也不支持外键，其优势是访问的速度快，对事务的完整性没有要求或者以SELECT、INSERT为主的应用基本上都可以使用这个引擎来创建表。有以下两个比较重要的特点

### 5.1 不支持事务

```sql
create table goods_myisam(
  id int not null auto_increment,
  name varchar(50) not null,
  primary key (id)
)engine=myisam default charset=utf8;
```



```sql
start transaction;
insert into goods_myisam values(null,'电脑');

rollback;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

通过测试，我们发现，在MyISAM存储引擎中，是没有事务控制的;

### 5.2 文件存储方式

每个MyISAM在磁盘上存储成3个文件，其文件名都和表名相同，但拓展名分别是：

.frm(存储表结构)

.MYD(MyData，存储数据);

.MYI(MyIndex，存储索引);

## 6 Memory和Merge

### 6.1 Memory

Memory存储引擎将表的数据存放在内存中。每个MEMORY表实际对应一个磁盘文件，格式是.frm， 该文件中只存储表的结构，而其数据文件，都是存储在内存中，这样有利于数据的快速处理，提高整个表的效率。MEMORY类型的表访问非常地快，因为他的数据是存放在内存中的， 并且默认使用HASH索引，但是服务-旦关闭 ，表中的数据就会丢失。

### 6.2 Merge

MERGE存储引擎是一组MyISAM表的组合，这些MyISAM表必须结构完全相同，MERGE表本身并没有存储数据，对MERGE类型的表可以进行查询、更新、删除操作，这些操作实际上是对内部的MyISAM表进行的。

对于MERGE类型表的插入操作，是通过INSERT METHOD子句定义插入的表，可以有3个不同的值，使用FIRST或LAST值使得插入操作被相应地作用在第一或者最后一个表上，不定义这个子句或者定义为NO，表示不能对这个MERGE表执行插入操作。

可以对MERGE表进行DROP操作，但是这个操作只是删除MERGE表的定义，对内部的表是没有任何影响的。

![image-20210703125659238](/Users/liuxiangren/mysql-learning/senior-img/mysql-merge-engine.png)

下面是一个创建和使用MERGE表的实例：

1)创建3个测试表，payment_2006、payment_2007、peyment_all，其中payment_all是前两个表的MERGE表

```sql
create table order_1990(
 order_id int,
 order_momory double(10,2),
 order_address varchar(50),
 primary key (order_id)
)engine=myisam default charset=utf8;

create table order_1991(
  order_id int,
  order_money double(10,2),
  order_address varchar(50),
  primary key (order_id)
)engine=myisam default charset=utf8;

create table order_all(
  order_id int,
  order_money double(10,2),
  order_address varchar(50),
  primary key (order_id)
)engine=merge union=(order_1990,order_1991) INSERT_METHOD=LAST default charset=utf8;
```

2)分别向两张表中插入记录

```sql
insert into order_1990 values(1,100.0,'北京');
insert into order_1990 values(2,100.0,'上海');

insert into order_1991 values(10,200.0,'北京');
insert into order_1991 values(11,200.0,'上海');
```

3)查询3张表中的数据

order_1990中的数据

```sql
select * from order_1990;
+----------+--------------+---------------+
| order_id | order_momory | order_address |
+----------+--------------+---------------+
|        1 |       100.00 | 北京          |
|        2 |       100.00 | 上海          |
+----------+--------------+---------------+
2 rows in set (0.00 sec)
```

order_1991中的数据

```sql
select * from order_1991;
+----------+-------------+---------------+
| order_id | order_money | order_address |
+----------+-------------+---------------+
|       10 |      200.00 | 北京          |
|       11 |      200.00 | 上海          |
+----------+-------------+---------------+
2 rows in set (0.00 sec)
```

order_all中的数据

```sql
select * from order_all;

+----------+-------------+---------------+
| order_id | order_money | order_address |
+----------+-------------+---------------+
|        1 |      100.00 | 北京          |
|        2 |      100.00 | 上海          |
|       10 |      200.00 | 北京          |
|       11 |      200.00 | 上海          |
+----------+-------------+---------------+
4 rows in set (0.00 sec)
```

4)往order_all中插入一条数据，由于在MERGE表定义时，INSERT_METHOD选择是LAST，那么插入的数据会最后

一个表

## 7 存储引擎的选择

在选择存储引擎时，应该根据应用系统的特点选择合适的存储引擎。对于杂的应用系统，还可以根据实际情况选择多种存储引擎进行组合。以下是几种常用的存储引擎的使用环境。

●InnoDB :是Mysq|的默认存储引擎，用于事务处理应用程序，支持外键。如果应用对事务的完整性有比较高的要求，在并发条件下要求数据的 一致性， 数据操作除了插入和查询意外，还包含很多的更新、删除操作，那么InnoDB存储引擎是比较合适的选择。InnoDB存储引擎除了有效的降低由于删除和更新导致的锁定，还可以确保事务的完整提交和回滚，对于类似于计费系统或者财务系统等对数据准确性要求比较高的系统，InnoDB是最合适的选择。

●MyISAM :如果应用是以读操作和插入操作为主，只有很少的更新和删除操作，并且对事务的完整性、并发性要求不是很高，那么选择这个存储引擎是非常合适的。

●MEMORY :将所有数据保存在RAM中，在需要快速定位记录和其他类似数据环境下，可以提供几块的访问。MEMORY的缺陷就是对表的大小有限制，太大的表无法缓存在内存中，其次是要确保表的数据可以恢复，数据库异常终止后表中的数据是可以恢的。MEMORY表通常用于更新不太频繁的小表，用以快速得到访问结果。

●MERGE :用于将一系列等同的MyISAM表以逻辑方式组合在 起，并作为一个对象引用他们。MERGE表的优点在于可以突破对单个MyISAM表的大小限制，并且通过将不同的表分布在多个磁盘上，可以有效的改善MERGE表的访问效率。这对于存储诸如数据仓储等VLDB环境十分合适。

## 8 优化SQL步骤

在应用的的开发过程中，由于初期数据量小，开发人员写SQL语句时更视功能上的实现，但是当应用系统正式上线后，随着生产数据的急剧增长，很多SQL语句开始逐渐显露出性能问题，对生产的影响也越来越大，此时这些有问题的SQL语句就成为整个系统性能的瓶颈，因此我们必须要对它们进行优化，本章将详细介绍在MySQL中优化SQL语句的方法。

当面对一个有SQL性能问题的数据库时，我们应该从何处入手来进行系统的分析，使得能够尽快定位问题SQL并尽快解决问题。

### 8.1 查看SQL执行频率

MySQL客户端连接成功后，通过show [ session | global ] status命令可以提供服务状态信息。show [ session | global ] status可以根据需要加上参数"session"或者"global"来显示session级(当前连接)的统计结果和global级(自数据库上次启动至今)的统计结果。如果不写，默认使用参数是"session"。

下面的命令显示了当前session中多有统计参数的值，频次

```sql
show status like 'Com_______';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_binlog    | 0     |
| Com_commit    | 1     |
| Com_delete    | 1     |
| Com_import    | 0     |
| Com_insert    | 26    |
| Com_repair    | 0     |
| Com_revoke    | 0     |
| Com_select    | 46    |
| Com_signal    | 0     |
| Com_update    | 2     |
| Com_xa_end    | 0     |
+---------------+-------+
11 rows in set (0.00 sec)
```

```sql
show status like 'Innodb_rows_%';
+----------------------+--------+
| Variable_name        | Value  |
+----------------------+--------+
| Innodb_rows_deleted  | 1      |
| Innodb_rows_inserted | 103364 |
| Innodb_rows_read     | 36913  |
| Innodb_rows_updated  | 2      |
+----------------------+--------+
4 rows in set (0.00 sec)
```

Com_xxx表示每个xxx语句执行的次数，我们通常比较关心的是一下几个统计参数

| 参数                 | 含义                                                       |
| -------------------- | ---------------------------------------------------------- |
| Com_select           | 执行select操作的次数，一次查询只累加1                      |
| Com_insert           | 执行insert操作的次数，对于批量插入的insert操作，只累加一次 |
| Com_update           | 执行update操作的次数                                       |
| Com_delete           | 执行delete操作的次数                                       |
| Innodb_rows_read     | select查询返回的行数                                       |
| Innodb_rows_inserted | 执行insert操作，插入的行数                                 |
| Innodb_rows_updated  | 执行update操作，更新的行数                                 |
| Innodb_rows_deleted  | 执行delete操作，删除的行数                                 |
| Connections          | 视图连接MySQL服务器的次数                                  |
| Uptime               | 服务器的工作时间                                           |
| Slow_queries         | 慢查询的次数                                               |

Com_***：这些参数对于所有存储引擎的表操作都会进行累计

Innodb_***:这几个参数只是针对InnoDB存储引擎的，累加的算法也略有不同

### 8.2 定位低效率执行的SQL

可以通过以下两种方式定位执行效率较低的SQL语句

- 慢查询日志：通过慢查询日志定位哪些执行效率较低的SQL语句，用--log--slow--queries[=file_name]选项启动时，mysqld写一个包含所有执行时间超过long_query_time秒的SQL语句的日志文件。具有可以查看本书第26章中日志管理的相关部分。
- show processlist：慢查询日志在查询结束以后才记录，所以在应用反应执行效率出现问题的时候查询日志并不能定位问题，可以使用show processlist命令查看MySQL在进行的线程，包括线程的状态，是否锁表等，可以实时的查看SQL的执行情况，同时对一些锁表操作进行优化。

```sql
show processlist;

+-----+-----------------+-----------------+---------+---------+--------+------------------------+------------------+
| Id  | User            | Host            | db      | Command | Time   | State                  | Info             |
+-----+-----------------+-----------------+---------+---------+--------+------------------------+------------------+
|   5 | event_scheduler | localhost       | NULL    | Daemon  | 420545 | Waiting on empty queue | NULL             |
| 344 | root            | localhost:59122 | demo_01 | Query   |      0 | init                   | show processlist |
+-----+-----------------+-----------------+---------+---------+--------+------------------------+------------------+
2 rows in set (0.01 sec)
```

```sql
1.id列，用户登录mysql时，系统分配的"connection_id"，可以使用函数connedction_id()查看
2.user列，显示当前用户，如果不是root，这个命令就只显示用户权限范围的SQL语句
3.host列，显示这个语句是从哪个IP的哪个端口上发的，可以又拿过来跟踪出现问题语句的用户
4.db列，显示这个进行目前连接的是哪个数据库
5.command列，显示当前连接的执行命令，一般取值为休眠(sleep)，查询(query)，连接(connect)等
6.time列，显示这个状态持续的时间，单位是秒
7.state列，显示使用当前连接的sql语句的状态，很重要的列。state描述的是语句执行中的某一个状态，一个sql语句，以查询为例，可能需要经过copying to tmp table、sorting result、sending data等状态才可以完成
8.info列，执行的SQL语句，是判断问题的重要依据
```

### 8.3 Explain分析执行计划

通过以上步骤查询到效率低下的SQL语句后，可以通过EXPLAIN或者DESC命令获取MySQL如何执行SELECT语句的信息，包括在SELECT语句执行过程中表如何连接和连接的顺序

查询SQL语句的执行计划

```sql
explain select * from tb_item where id=1;

explain select * from tb_item where title='华为 (P-40) 冰川白 联通5G手机';
```

| 字段          | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| id            | select查询的序列号，是一组数字，表示的是查询中执行select子句或者是操作表的顺序 |
| select_type   | 表示select的类型，常见的取值有simple(简单表，即不适用表连接或者子查询)、primary(主查询，即外层的查询)、union(union中的第二个或者后面的查询语句)、subquery(子查询中的第一个select)等 |
| table         | 输出结果集的表                                               |
| type          | 表示表的连接类型，性能由好到差的连接类型为(system --> cost --> eq_ref --> ref --> ref_or_null --> index_merge --> index_subquery --> range --> index --> all) |
| possible_keys | 表示查询时，可能使用的索引                                   |
| key           | 表示实际使用的索引                                           |
| key_len       | 索引字段的长度                                               |
| rows          | 扫描行的数量                                                 |
| Extra         | 执行情况的说明和描述                                         |

#### 8.3.1 环境准备

```sql
create table t_role(
  id varchar(32) not null,
  role_name varchar(255) default null,
  role_code varchar(255) default null,
  description varchar(255) default null,
  primary key (id),
  unique key unique_role_name(role_name)
)engine=innodb default charset=utf8;

create table t_user(
  id varchar(32) not null,
  username varchar(45) not null,
  password varchar(96) not null,
  name varchar(45) not null,
  primary key (id),
  unique key unique_user_username(username)
)engine=innodb default charset=utf8;

create table t_user_role(
  id int(11) not null auto_increment,
  user_id varchar(32) default null,
  role_id varchar(32) default null,
  primary key (id),
  key fk_ur_user_id(user_id),
  key fk_ur_role_id(role_id),
  constraint fk_ur_user_id foreign key (user_id) references t_user (id) on delete no  action on update no action,
  constraint fk_ur_role_id foreign key (role_id) references t_role (id) on delete no  action on update no action
)engine=innodb default charset=utf8;


insert into t_user values('1','super','123465','超级管理员');
insert into t_user values('2','admin','123465','系统管理员');
insert into t_user values('3','senior','123465','test02');
insert into t_user values('4','stu1','123465','学生1');
insert into t_user values('5','stu2','123465','学生2');
insert into t_user values('6','teac1','123465','老师1');

insert into t_role values('5','学生','studnet','学生');
insert into t_role values('7','老师','teacher','老师');
insert into t_role values('8','教学管理员','teachermanager','教学管理员');
insert into t_role values('9','管理员','admin','管理员');
insert into t_role values('10','超级管理员','super','超级管理员');


insert into t_user_role(id,user_id,role_id) values(null,'1','5'),(null,'1','7'),(null,'2','8'),(null,'3','9'),(null,'4','8'),(null,'5','10');
```





#### 8.3.2 explain之id

id字段是select查询的序列号，是一组数字，表示的是查询中执行select子句或者是操作表的顺序。id情况有三种：

1. id相同表示加载表的顺序是从上到下

   ```sql
   explain select * from t_role r,t_user u,t_user_role ur where r.id=ur.role_id and u.id=ur.user_id;
   +----+-------------+-------+------------+--------+-----------------------------+---------------+---------+--------------------+------+----------+-------------+
   | id | select_type | table | partitions | type   | possible_keys               | key           | key_len | ref                | rows | filtered | Extra       |
   +----+-------------+-------+------------+--------+-----------------------------+---------------+---------+--------------------+------+----------+-------------+
   |  1 | SIMPLE      | r     | NULL       | ALL    | PRIMARY                     | NULL          | NULL    | NULL               |    5 |   100.00 | NULL        |
   |  1 | SIMPLE      | ur    | NULL       | ref    | fk_ur_user_id,fk_ur_role_id | fk_ur_role_id | 99      | demo_01.r.id       |    1 |   100.00 | Using where |
   |  1 | SIMPLE      | u     | NULL       | eq_ref | PRIMARY                     | PRIMARY       | 98      | demo_01.ur.user_id |    1 |   100.00 | NULL        |
   +----+-------------+-------+------------+--------+-----------------------------+---------------+---------+--------------------+------+----------+-------------+
   3 rows in set, 1 warning (0.00 sec)
   ```

2. id不同id值越大，优先级越高，越先被执行

   ```sql
   explain select * from t_role where id =(select role_id from t_user_role where user_id = (select id from t_user where username='stu1'));
   +----+-------------+-------------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------------+
   | id | select_type | table       | partitions | type  | possible_keys        | key                  | key_len | ref   | rows | filtered | Extra       |
   +----+-------------+-------------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------------+
   |  1 | PRIMARY     | t_role      | NULL       | const | PRIMARY              | PRIMARY              | 98      | const |    1 |   100.00 | NULL        |
   |  2 | SUBQUERY    | t_user_role | NULL       | ref   | fk_ur_user_id        | fk_ur_user_id        | 99      | const |    1 |   100.00 | Using where |
   |  3 | SUBQUERY    | t_user      | NULL       | const | unique_user_username | unique_user_username | 137     | const |    1 |   100.00 | Using index |
   +----+-------------+-------------+------------+-------+----------------------+----------------------+---------+-------+------+----------+-------------+
   
   ```

3. id有相同，也有不同，同时存在。id相同的可以认为是一组，从上往下顺序执行；在所有的组中，id的值越大，优先级越高，越先执行。

   ```sql
   explain select * from t_role r,(select * from t_user_role ur where ur.user_id='2') a where r.id=a.role_id;
   ```

#### 8.3.3 explain之select_type

表示select的类型，常见的取值，如下表所示：

| Select_type  | 含义                                                         |
| ------------ | ------------------------------------------------------------ |
| simple       | 简单的select查询，查询中不包含子查询或者union                |
| primary      | 查询中若包含任何复杂的子查询，最外层查询标记为该标识         |
| subquery     | 在select或者where列表中包含了子查询                          |
| derived      | 在form列表中包含的子查询，被标记为derived(衍生)Mysql会递归执行这些子查询，把结果放在临时表汇总 |
| union        | 若第二个select出现在union之后，则标记为union；若union包含在form子句的子查询中，外层的select将被标记为：DERIVED |
| union result | 从union表获取结果的select                                    |

#### 8.3.4 explain之table

展示这一行的数据是关于哪一张表的

#### 8.3.5 explain之type

type显示的是访问类型，是较为重要的一个指标，可取值为：

| type   | 含义                                                         |
| ------ | ------------------------------------------------------------ |
| NULL   | MySQL不访问任何表，索引，直接返回结果                        |
| system | 表只有一行记录(等于系统表)，这是const类型的特例，一般不会出现 |
| const  | 表示通过索引一次就找到了，const用于比较primary key或者unique索引，因为只匹配一行数据，所以很快，如将主键置于where列表中，mysql就能将该查询转换为一个常量，const于将主键或者唯一索引的所有部分与常量值进行比较 |
| eq_ref | 类似ref，区别在于使用的是唯一索引，使用主键的关联查询，关联查询出记录只有一条，常见于主键或唯一索引扫描 |
| ref    | 非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，返回所有匹配某个单独值的所有行(多个) |
| range  | 只检索给定返回的行，使用一个索引来选择行，where之后出现between,<,>,in等操作 |
| index  | index与ALL的区别为index类型只是遍历了索引树，通常比ALL块，ALL是遍历数据文件 |
| all    | 将遍历全表以找到匹配的行                                     |

结果值从最好到最坏依次是

```sql
NULL > system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL
```

#### 8.3.6 explain之key

```sql
possible_keys : 显示可能应用在这张表的索引，一个或者多个
key : 实际使用的索引，如果为NULL，则没有使用索引
key_len : 表示索引中使用的字节数，该值为索引字段最大可能长度，并非实际使用长度，在不损失精确性的前提下，长度越短越好
```

#### 8.3.7 explain之rows

扫描的行数

#### 8.3.8 explain之extra

其他额外的执行计划信息，在该列展示

| extra           | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| using filesort  | 说明mysql 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取，成为"文件排序" |
| using remporary | 使用了临时表保存中间结果，mysql在对查询结果排序时使用临时表，常见于order by 和group by |
| using index     | 表示相应的select操作使用了覆盖索引，避免访问表的数据行，效率不错 |

### 8.4 show profile分析sql

mysql从5.0.37版本开始增加了对show profile和show profile语句的支持，show profiles能够在做sql优化时帮助我们了解时间都耗费到哪里去了。

通过have_profiling参数，能狗看到当前Mysql是否支持profile

```sql
select @@have_profiling;
```

默认profiling是关闭的，可以通过set语句在session级别开启profiling;

```sql
select @@profiling;

set profiling=1; //开启profiling开关
```

通过profile，我们能够更清楚的了解SQL执行的过程

首先，我们可以执行一系列的操作，如下图所示：

```sql
show databases;

use demo_01;

#一通查询操作什么的...

#可以进行SQL耗时的查询
show profiles;
可以列出 query_id、duration(耗时)、query(sql语句)
#可以根据query_id查看更多SQL执行的详细信息
show profile for query [query_id];
+-----------------------+----------+
| starting							| 0.000057 |
+-----------------------+----------+
| checking permissions  | 0.000006 |
+-----------------------+----------+
| opening tables				| 0.000020 |
+-----------------------+----------+
| init									| 0.000016 |
+-----------------------+----------+
| System lock						| 0.000010 |
+-----------------------+----------+
| optimizing						| 0.000010 |					 
+-----------------------+----------+
| statistics						|	0.000014 |				
+-----------------------+----------+
| preparing							| 0.000015 |
+-----------------------+----------+
| executing							| 0.000003 |
+-----------------------+----------+
| Sending data					| 3.318910 |
+-----------------------+----------+
| end										| 0.000020 |
+-----------------------+----------+
| query end							| 0.000008 |
+-----------------------+----------+
| closing tables				| 0.000018 |
+-----------------------+----------+
| freeing items					| 0.000029 |
+-----------------------+----------+
| cleaning up						| 0.000014 |
+-----------------------+----------+
```



```sql
Tip:
 Sending data 状态表示mysql线程开始访问数据行并把结果返回给客户端，而不仅仅是返回给客户端。由于Sending data 状态下，Mysql线程往往需要做大量的磁盘读取操作，所以经常是整个查询中耗时最长的状态。
```

在获取到最消耗时间的线程状态后，Mysql支持进一步选择all、cpu、block io、context switch、page faults等明细类型类查看Mysql在使用什么资源上耗费了过高的时间。例如，选择查看CPU的耗费时间：

```sql
show profile cpu for query 5;

#查看所有
show profile all for query 5;
```



### 8.5 trace分析优化器执行计划

mysql5.6提供了对SQL的跟踪trace，通过trace文件能够进一步了解为什么优化器选择A计划，而不是B计划。

打开trace，设置格式为JSON，并设置trace最大能够使用的内存大小，避免解析过程中因为内存过小而不能够完整展示。

```sql
set optimizer_trace='enable=on',end_markers_in_json=on;
set optimizer_trace_max_mem_size=100000;
```

执行sql语句

```sql
select * from tb_item where id<4;
```

最后检查information_schema.optimizer_trace就可以知道Mysql是如何执行SQL的

```sql
select * from information_schema.optimizer_trace\G;
```

```json
************************1. row **************************
QUERY: select * from tb_item where id < 4
TRACE: {
  "steps:"[
  	{
  		"join_preparation":{
  			"select#":1,
  			"steps":[
  				{
  					"expanded_query":"/* seelct#1 */ select `tb_item`.`id` as ``id,`tb_item`.`title` as `title`,`tb_item`.`price` as `price`,`tb_item`.`num` as `num`,`tb_item`.`categoryid` as `categoryid`...."
					}
  			]
			}....
		}....
  ]
}
```

### 8.6 索引的使用

索引是数据库优化最常用也是最重要的手段之一，通过索引通常可以帮助用户解决大多数的Mysql的性能优化问题。

#### 8.6.1 验证索引提升效率查询

在我们准备的表结构tb_item中，一共存储了300万条记录；

A.根据ID查询

```sql
select * from tb_item where id = 1999;
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| id   | title         | price    | num   | categoryid | status | sellerid   | createtime          | updatetime          |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| 1999 | 货物1999号    | 84192.28 | 74652 |          2 | 1      | 5435343235 | 2019-04-20 22:37:15 | 2019-04-20 22:37:15 |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

这样根据主键查询的速度是非常快的

B.那我们根据title进行查询

```sql
select * from tb_item where title = '货物1999号';
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| id   | title         | price    | num   | categoryid | status | sellerid   | createtime          | updatetime          |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| 1999 | 货物1999号    | 84192.28 | 74652 |          2 | 1      | 5435343235 | 2019-04-20 22:37:15 | 2019-04-20 22:37:15 |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
1 row in set (0.97 sec)
```

处理方案，针对title字段，创建索引

```sql
create index idx_item_title on tb_item(title);
Query OK, 0 rows affected (4.59 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

针对title字段创建索引之后，再次进行查询

```sql
select * from tb_item where title = '货物1999号';
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| id   | title         | price    | num   | categoryid | status | sellerid   | createtime          | updatetime          |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
| 1999 | 货物1999号    | 84192.28 | 74652 |          2 | 1      | 5435343235 | 2019-04-20 22:37:15 | 2019-04-20 22:37:15 |
+------+---------------+----------+-------+------------+--------+------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

可以说是相当的快了

#### 8.6.2 索引的使用

##### 8.6.2.1 环境准备

```sql
create table tb_seller(
  sellerid varchar(100),
  name varchar(100),
  nickname varchar(50),
  password varchar(60),
  status varchar(1),
  address varchar(100),
  createtime datetime,
  primary key(sellerid)
)engine=innodb default charset=utf8mb4;

insert into tb_seller values('alibaba','阿里巴巴','阿里小店','123456','1','北京市','2088-01-01 12:00:00');
insert into tb_seller values('baidu','百度','阿波罗','123456','1','北京市','2038-01-01 12:00:00');
insert into tb_seller values('huawei','华为','鸿蒙','123456','1','北京市','2048-01-01 12:00:00');
insert into tb_seller values('bytedance','字节跳动','抖音','123456','1','北京市','2058-01-01 12:00:00');
insert into tb_seller values('mclub','mc','音乐人俱乐部','123456','1','北京市','2028-01-01 12:00:00');
insert into tb_seller values('jingdong','京东','京东商城','123456','1','北京市','2018-01-01 12:00:00');

insert into tb_seller values('oppo','欧珀','手机','123456','0','北京市','2088-01-01 12:00:00');
insert into tb_seller values('vivo','步步高','手机','123456','0','北京市','2038-01-01 12:00:00');
insert into tb_seller values('logic','逻辑','鼠标','123456','1','北京市','2048-01-01 12:00:00');
insert into tb_seller values('sina','新浪','微博','123456','0','南京市','2058-01-01 12:00:00');
insert into tb_seller values('ctln','宁德时代','三元锂电池','123456','1','北京市','2028-01-01 12:00:00');
insert into tb_seller values('byd','比亚迪','车载芯片','123456','1','北京市','2018-01-01 12:00:00');

insert into tb_seller values('xiaomi','小米科技','小米手机','123456','1','北京市','2018-01-01 12:00:00');
```

##### 8.6.2.2 **创建组合索引**

```sql
create index idx_seller_name_sta_addr on tb_seller(name,status,address);
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

##### 8.6.2.3 避免索引失效

1)全值匹配，对索引中所有列都指定具体值

该情况下，索引生效，执行效率高

```sql
explain select * from tb_seller where name='宁德时代' and status='1' and address='北京市';
```



2)最左前缀法则

如果索引了多列，要遵守最左前缀法则，指的是查询从索引的最左前列开始，并且不跳过索引中的列。

匹配最左前缀法则，走索引：

```sql
explain select * from tb_seller where name='小米科技';
走索引了
```

```sql
explain select * from tb_seller where name='小米科技' and status='1';
走索引了
```

```sql
explain select * from tb_seller where name='小米科技' and address='北京市';
走索引了,但是只匹配了name的索引，没有用到address的索引
```

```sql
explain select * from tb_seller where name='小米科技' and status='1' and address='北京市';
走索引了
```

```sql
explain select * from tb_seller where address='北京市' and name='小米科技';
走索引了,说明先后顺序是没有关系的
```

没走索引：

```sql
explain select * from tb_seller where status='1';
```

```sql
explain select * from tb_seller where address='北京市';
```



3)范围查询右边的列，不能使用索引

```sql
explain select * from tb_seller where name='小米科技' and status='1' and address='北京市';

explain select * from tb_seller where name='小米科技' and status>'1' and address='北京市';
根据前面的两个字段的name，status查询是走索引的，但是最后一个条件address没有用到索引，使用范围查询后，右边的列是无法再使用索引的

放在后面也一样的
explain select * from tb_seller where name='小米科技' and address='北京市' and status>'1';
```



4)不要在索引列上进行运算操作，索引将失效

```sql
select * from tb_seller where substring(name,3,2)='科技';
```

```sql
explain select * from tb_seller where name='小米科技' and status ='1' and address='北京市';
```



5)字符串不加单引号，造成索引失效，也就是输入的类型和要查询字段的类型不匹配

```sql
explain select * from tb_seller where name='小米科技' and status=1;
只用到了name
```



6)尽量使用覆盖索引，避免select *

尽量使用覆盖索引(只访问索引的查询(索引列完全包含查询列))，减少select *。

如果查询列，超出索引列，也会降低性能

```sql
explain select status,address,password from tb_seller where name='科技' and status='0' and address='北京市';
#这个password就超出了组合索引列
```

```markdown
TIP：
	using index : 使用覆盖索引的时候就会出现
	using where : 在查找使用索引的情况下，需要回表去查询所需的数据
	using index condition : 查找使用了索引，但是需要回表查询数据
	using index;using where : 查找使用了索引，但是需要的数据都在索引列中能找到，所以不需要回表查询数据

最后这个using index;using where:这个情况，假设组合索引abc,只要查询的字段和查询条件为(a,b,c),那么就可以通过索引进行数据的获取，不需要再回表查了。
```



7)in走索引，not in索引失效

In 在5.7以前，如果是小范围的查询，还是走索引的，type属于range，在随着数据量的增大时会自动进行全表的扫描（并且与要查询的结果是否包含在索引树中决定走index还是all）；not in则不走索引；

**目前在8.0以后验证，发现无论是in not 或者<>,都会走索引；**

```sql
explain select * from tb_seller where sellerid in('oppo','xiaomi','sina'); #走主键索引

explain select * from tb_seller where sellerid not in('oppo','xiaomi','sina'); #mysql8走主键索引
```





8)用or分割开的条件，如果or前的条件有索引，而or后面的列没有索引，那么设计的索引都不会被使用到。

实例：

```sql
explain select * from tb_seller where name='宁德时代' or createtime='2028-01-01 12:00:00';

explain select * from tb_seller where name='宁德时代' or nickname='三元锂电池';

key的列全都为NULL，不会使用到索引

explain select * from tb_seller where name='宁德时代' and nickname='三元锂电池';
就走索引啦但是只有name，肯定还要回表
```



9)以%开头的Like模糊查询，索引失效

如果仅仅是尾部模糊匹配，索引不会失效，如果是头部模糊匹配，索引失效

```sql
explain select * from tb_seller where name like '比亚%';
```



10)如果myslq评估使用索引比全表更慢，则不适用索引

```sql
explain select * from tb_seller where address='北京市'; #走不了索引
explain select name from tb_seller where address='北京市'; #走的了索引

create index idx_address on tb_seller(address);

explain select * from tb_seller where address='北京市'; #没走索引
explain select * from tb_seller where address='南京市'; #走了索引
为啥？因为已有一条addres是南京市，其他的全是北京市，也正因此，所以搜索address='北京市'的时候才没走索引，因为北京市的数据在表中占比太大了。
```



11) is null,is not null有时索引失效

```sql
情况一：
explain select * from tb_seller where name is null; #走索引，因为address为null的数据在表中就没有，所以就走了索引

explain select * from tb_seller where name is not null; #mysql8没走索引，is not null的话不如全表扫描快

情况二：
select * from t_user;
+----+----------+----------+-----------------+
| id | username | password | name            |
+----+----------+----------+-----------------+
| 1  | super    | 123465   | 超级管理员      |
| 2  | admin    | 123465   | 系统管理员      |
| 3  | senior   | 123465   | test02          |
| 4  | stu1     | 123465   | 学生1           |
| 5  | stu2     | 123465   | 学生2           |
| 6  | teac1    | 123465   | 老师1           |
+----+----------+----------+-----------------+
6 rows in set (0.00 sec)

update t_user set name = null where id <> 1; #只有id为1的name不为null

explain select * from t_user where name is null;  #那么is null就不走索引了，绝大多数都是null，所以走了全表扫描

explain select * from t_uer where name is not null; #那is not null就走索引了
```



12)单列索引和复合索引的选择问题

尽量使用复合索引，而少使用单列索引

```sql
#创建复合索引
create index idx_name_status_address on tb_seller(name,status,address);

就相当于创建了三个索引:
	name
	name + status;
	name + status + address;
```

创建单列索引

```sql
create index ids_seller_name on tb_seller(name);
create index ids_seller_status on tb_seller(status);
create index ids_seller_address on tb_seller(address);
```

数据库会选择一个最优的索引来使用，并不会使用全部索引



### 8.7 查看索引使用情况

```sql
show status like 'Handler_read%'; #当前会话的索引使用情况

show global status like 'Hnadler_read%';
```

```markdown
Handler_read_first:索引中第一条被读的次数。如果较高，表示服务器正在执行大量全索引扫描(这个值越低越好)

Handler_read_key:如果索引正在工作，这个值代表一个行被索引值读的次数，如果值越低，表示索引得到的性能改善不高，因为索引不经常使用(这个值越高越好)

Handler_read_next:按照键顺序读下一行的请求数，如果你用范围约束或如果执行索引扫描来查询索引列，该值增加

Handler_read_prev:按照键顺序读前一行的请求数，该读方法主要用于优化ORDER BY ... DESC

Handler_read_rnd:根据固定位置读一行的请求数，如果你正执行大量查询并需要对结果进行排序该值较高，你可能使用了大量需要MySQL扫描整个表的查询或你的连接没有正确使用键。这个值越高意味着效率低，应该建立索引来补救。

Handler_read_next:在数据文件中读下一行的请求数，如果你正在进行大量的表扫描，该值越高，通常说明你的表索引不正确或者写入的查询没有利用索引。
```





## 9 SQL优化

### 9.1 准备环境

```sql
create table tb_user_2(
  id int(11) not null auto_increment,
  username varchar(45) not null,
  password varchar(96) not null,
  name varchar(45) not null,
  birthday datetime default null,
  sex char(1) default null,
  email varchar(45) default null,
  phone varchar(45) default null,
  qq varchar(32) default null,
  status varchar(32) not null comment '用户状态',
  create_time datetime not null,
  update_time datetime default null,
  primary key (id),
  unique key unique_user_username (username)
)engine=innodb default charset=utf8;

tb_user_1,tb_user_2两张表结构一样的
```

当使用load 命令导入数据的时候，适当的设置可以提高导入的效率。

对于 InnoDB 类型的表，有以下几种方式可以提高导入的效率：

1） 主键顺序插入

因为InnoDB类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺序排列，可以有效的提高导入数据的效率。如果InnoDB表没有主键，那么系统会自动默认创建一个内部列作为主键，所以如果可以给表创建一个主键，将可以利用这点，来提高导入数据的效率。

```
脚本文件介绍 :
	sql1.log  ----> 主键有序
	sql2.log  ----> 主键无序
```

插入ID顺序排列数据：

```sql
load data local infile '/Users/xxx/Downloads/sql1.log' into table `tb_user_1` fields terminated by ',' lines terminated by '\n';
ERROR 3948 (42000): Loading local data is disabled; this must be enabled on both the client and server sides
提示是限制了本地文件加载：修改本地加载功能： set global local_infile = 1;没得用

mysql --local-infile=1 -h127.0.0.1 -uroot -p #--local-infile=1这个管用

导入：tb_user_1
Query OK, 1000000 rows affected, 65535 warnings (14.05 sec)
Records: 1000000  Deleted: 0  Skipped: 0  Warnings: 4000000

load data local infile '/Users/xxx/Downloads/sql2.log' into table `tb_user_2` fields terminated by ',' lines terminated by '\n';
Query OK, 1000000 rows affected, 65535 warnings (23.55 sec)
Records: 1000000  Deleted: 0  Skipped: 0  Warnings: 4000000
```

**有序主键插入效率高**

2） 关闭唯一性校验

在导入数据前执行 SET UNIQUE_CHECKS=0，关闭唯一性校验，在导入结束后执行SET UNIQUE_CHECKS=1，恢复唯一性校验，可以提高导入的效率。

```sql
set unique_check=0;
load data.....
set unique_check=1;
```

3） 手动提交事务

如果应用使用自动提交的方式，建议在导入前执行 SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行 SET AUTOCOMMIT=1，打开自动提交，也可以提高导入的效率。

```sql
set autocommit=0;
load data.....
set autocommit=1;
```

### 9.2 优化insert语句

当进行数据的insert操作的时候，可以考虑采用以下几种优化方案。

- 如果需要同时对一张表插入很多行数据时，应该尽量使用多个值表的insert语句，这种方式将大大的缩减客户端与数据库之间的连接、关闭等消耗。使得效率比分开执行的单个insert语句快。

  示例， 原始方式为：

  ```sql
  insert into tb_test values(1,'Tom');
  insert into tb_test values(2,'Cat');
  insert into tb_test values(3,'Jerry');
  ```

  优化后的方案为 ： 

  ```sql
  insert into tb_test values(1,'Tom'),(2,'Cat')，(3,'Jerry');
  ```

- 在事务中进行数据插入。

  ```sql
  start transaction;
  insert into tb_test values(1,'Tom');
  insert into tb_test values(2,'Cat');
  insert into tb_test values(3,'Jerry');
  commit;
  ```

- 数据有序插入

  ```sql
  insert into tb_test values(4,'Tim');
  insert into tb_test values(1,'Tom');
  insert into tb_test values(3,'Jerry');
  insert into tb_test values(5,'Rose');
  insert into tb_test values(2,'Cat');
  ```

  优化后

  ```sql
  insert into tb_test values(1,'Tom');
  insert into tb_test values(2,'Cat');
  insert into tb_test values(3,'Jerry');
  insert into tb_test values(4,'Tim');
  insert into tb_test values(5,'Rose');
  ```



### 9.3 优化order by语句

```sql
CREATE TABLE `emp` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `age` int(3) NOT NULL,
  `salary` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4;

insert into `emp` (`id`, `name`, `age`, `salary`) values('1','Tom','25','2300');
insert into `emp` (`id`, `name`, `age`, `salary`) values('2','Jerry','30','3500');
insert into `emp` (`id`, `name`, `age`, `salary`) values('3','Luci','25','2800');
insert into `emp` (`id`, `name`, `age`, `salary`) values('4','Jay','36','3500');
insert into `emp` (`id`, `name`, `age`, `salary`) values('5','Tom2','21','2200');
insert into `emp` (`id`, `name`, `age`, `salary`) values('6','Jerry2','31','3300');
insert into `emp` (`id`, `name`, `age`, `salary`) values('7','Luci2','26','2700');
insert into `emp` (`id`, `name`, `age`, `salary`) values('8','Jay2','33','3500');
insert into `emp` (`id`, `name`, `age`, `salary`) values('9','Tom3','23','2400');
insert into `emp` (`id`, `name`, `age`, `salary`) values('10','Jerry3','32','3100');
insert into `emp` (`id`, `name`, `age`, `salary`) values('11','Luci3','26','2900');
insert into `emp` (`id`, `name`, `age`, `salary`) values('12','Jay3','37','4500');

create index idx_emp_age_salary on emp(age,salary);
```

```sql
show index from emp;
```

#### 9.3.1 FileSort 排序

1). 第一种是通过对返回数据进行排序，也就是通常说的 filesort 排序，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。

```sql
#不走主键索引，也不走组合索引
explain select * from emp order by age desc;
explain select * from emp order by age asc;
+-----+---------+------+------+----------+----------------+
| key | key_len | ref  |  12  |  100.00  | Using filesort |
+-----+---------+------+------+----------+----------------+
```



#### 9.3.2 using index 排序

2). 第二种通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要额外排序，操作效率高。

```sql
#走了联合索引
explain select id from emp order by age asc;
explain select id,age from emp order by age asc;
explain select id,age,salary from emp order by age asc;

+--------------------+---------+------+------+----------+-------------+
| idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index |
+--------------------+---------+------+------+----------+-------------+
```



#### 9.3.3 多字段排序

了解了MySQL的排序方式，优化目标就清晰了：尽量减少额外的排序，通过索引直接返回有序数据。where 条件和Order by 使用相同的索引，并且Order By 的顺序和索引顺序相同， 并且Order  by 的字段都是升序，或者都是降序。否则肯定需要额外的操作，这样就会出现FileSort。

```sql
#走了联合索引
explain select id,age,salary from emp order by age,salary;
+--------------------+---------+------+------+----------+-------------+
| idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index |
+--------------------+---------+------+------+----------+-------------+

explain select id,age,salary from emp order by age desc,salary desc;
+--------------------+---------+------+------+----------+----------------------------------+
| idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Backward index scan; Using index |
+--------------------+---------+------+------+----------+----------------------------------+
explain select id,age,salary from emp order by age desc,age desc;
+--------------------+---------+------+------+----------+----------------------------------+
| idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Backward index scan; Using index |
+--------------------+---------+------+------+----------+----------------------------------+
explain select id,age,salary from emp order by age desc,salary asc;
+--------------------+---------+------+------+----------+-----------------------------+
| key                | key_len | ref  | rows | filtered | Extra                       |
+--------------------+---------+------+------+----------+-----------------------------+
| idx_emp_age_salary | 9       | NULL |   12 |   100.00 | Using index; Using filesort |
+--------------------+---------+------+------+----------+-----------------------------+
```



#### 9.3.4 Filesort 的优化

通过创建合适的索引，能够减少 Filesort 的出现，但是在某些情况下，条件限制不能让Filesort消失，那就需要加快 Filesort的排序操作。对于Filesort ， MySQL 有两种排序算法：

1） 两次扫描算法 ：MySQL4.1 之前，使用该方式排序。首先根据条件取出排序字段和行指针信息，然后在排序区 sort buffer 中排序，如果sort buffer不够，则在临时表 temporary table 中存储排序结果。完成排序之后，再根据行指针回表读取记录，该操作可能会导致大量随机I/O操作。

2）一次扫描算法：一次性取出满足条件的所有字段，然后在排序区 sort  buffer 中排序后直接输出结果集。排序时内存开销较大，但是排序效率比两次扫描算法要高。



MySQL 通过比较系统变量 max_length_for_sort_data 的大小和Query语句取出的字段总大小， 来判定是否那种排序算法，如果max_length_for_sort_data 更大，那么使用第二种优化之后的算法；否则使用第一种。

可以适当提高 sort_buffer_size  和 max_length_for_sort_data  系统变量，来增大排序区的大小，提高排序的效率。

```sql
show variables like 'max_length_for_sort_data';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| max_length_for_sort_data | 4096  |
+--------------------------+-------+
1 row in set (0.01 sec)

show variables like 'sort_buffer_size';
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| sort_buffer_size | 262144 |
+------------------+--------+
1 row in set (0.00 sec)
```



### 9.4 优化group by 语句

由于GROUP BY 实际上也同样会进行排序操作，而且与ORDER BY 相比，GROUP BY 主要只是多了排序之后的分组操作。当然，如果在分组的时候还使用了其他的一些聚合函数，那么还需要一些聚合函数的计算。所以，在GROUP BY 的实现过程中，与 ORDER BY 一样也可以利用到索引。

如果查询包含 group by 但是用户想要避免排序结果的消耗， 则可以执行order by null 禁止排序。如下 ：

```SQL
drop index idx_emp_age_salary on emp;

explain select age,count(*) from emp group by age;
```

优化后

```sql
explain select age,count(*) from emp group by age order by null;
```

**mysql8实测，优化前优化后都一样的Using temporary**

创建索引 ：

```SQL
create index idx_emp_age_salary on emp(age,salary);
```



### 9.5 优化嵌套查询

Mysql4.1版本之后，开始支持SQL的子查询。这个技术可以使用SELECT语句来创建一个单列的查询结果，然后把这个结果作为过滤条件用在另一个查询中。使用子查询可以一次性的完成很多逻辑上需要多个步骤才能完成的SQL操作，同时也可以避免事务或者表锁死，并且写起来也很容易。但是，有些情况下，子查询是可以被更高效的连接（JOIN）替代。

示例 ，查找有角色的所有的用户信息 : 

```SQL
 explain select * from t_user where id in (select user_id from user_role );
```

执行计划为 : 

优化后 :

```SQL
explain select * from t_user u , user_role ur where u.id = ur.user_id;
```

连接(Join)查询之所以更有效率一些 ，是因为MySQL不需要在内存中创建临时表来完成这个逻辑上需要两个步骤的查询工作。

#### 9.6 优化OR条件

对于包含OR的查询子句，如果要利用索引，则OR之间的每个条件列都必须用到索引 ， 而且不能使用到复合索引； 如果没有索引，则应该考虑增加索引。

获取 emp 表中的所有的索引 ： 

```sql
show from emp;
```

示例 ： 

```SQL
explain select * from emp where id = 1 or age = 30;
```

建议使用 union 替换 or ： 

```sql
explain select * from emp where id=1 uion select * from emp where id=10;
```

我们来比较下重要指标，发现主要差别是 type 和 ref 这两项

type 显示的是访问类型，是较为重要的一个指标，结果值从好到坏依次是：

```
system > const > eq_ref > ref > fulltext > ref_or_null  > index_merge > unique_subquery > index_subquery > range > index > ALL
```

UNION 语句的 type 值为 ref，OR 语句的 type 值为 range，可以看到这是一个很明显的差距

UNION 语句的 ref 值为 const，OR 语句的 type 值为 null，const 表示是常量值引用，非常快

这两项的差距就说明了 UNION 要优于 OR 。



#### 9.7 优化分页查询

一般分页查询时，通过创建覆盖索引能够比较好地提高性能。一个常见又非常头疼的问题就是 limit 2000000,10  ，此时需要MySQL排序前2000010 记录，仅仅返回2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大 。

```sql
explain select * from tb_item limit 200000,10;
```



##### 9.7.1 优化思路一

在索引上完成排序分页操作，最后根据主键关联回原表查询所需要的其他列内容。

```sql
explain select * from tb_item t,(select id from tb_item order by id limit 200000,10) a where t.id=a.id;
```



##### 9.7.2 优化思路二

该方案适用于主键自增的表，可以把Limit 查询转换成某个位置的查询 。

```sql
explain select * from tb_item where id>1000000 limit 10;
```



#### 9.8 使用SQL提示

SQL提示，是优化数据库的一个重要手段，简单来说，就是在SQL语句中加入一些人为的提示来达到优化操作的目的。

##### 9.8.1 USE INDEX

在查询语句中表名的后面，添加 use index 来提供希望MySQL去参考的索引列表，就可以让MySQL不再考虑其他可用的索引。

```sql
create index idx_seller_name on tb_seller(name);
```

```sql
explain select * from tb_seller where name='小米科技';

explain select * from tb_seller use index(idx_seller_name) where name='小米科技';
```

##### 9.8.2 IGNORE INDEX

如果用户只是单纯的想让MySQL忽略一个或者多个索引，则可以使用 ignore index 作为 hint 。

```sql
explain select * from tb_seller ignore index(idx_seller_name) where name = '小米科技';
```

```sql
explain select * from tb_seller ignore index(idx_seller_name) where name='小米科技';

explain select * from tb_seller ignore index(idx_seller_name) where name='小米科技';
```



##### 9.8.3 FORCE INDEX

为强制MySQL使用一个特定的索引，可在查询中使用 force index 作为hint 。 

``` SQL
create index idx_seller_address on tb_seller(address);
```

```sql
explain select * from tb_seller where address='北京市';

explain select * from tb_seller use index(idx_seller_address) where address='北京市';

explain select * from tb_seller force index(idx_seller_address) where address='北京市';
```





## 20 创建1000万数据

数据随意加的
建表：

```sql
CREATE TABLE `tb_item` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '商品id',
  `title` varchar(100) NOT NULL COMMENT '商品标题',
  `price` decimal(20,2) NOT NULL COMMENT '商品价格,单位为:元',
  `num` int(10) NOT NULL COMMENT '库存数量',
  `categoryid` bigint(10) NOT NULL COMMENT '所属类目,叶子类目',
  `status` varchar(1) DEFAULT NULL COMMENT '商品状态,1-正常,2-下架,3-删除',
  `sellerid` varchar(50) DEFAULT NULL COMMENT '商家ID',
  `createtime` datetime DEFAULT NULL COMMENT '创建时间',
  `updatetime` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='商品表';
```

创建存储过程：

```sql
delimiter $

create procedure insert_tb_item(num int)
begin
while num<=3000000 do
insert into tb_item values(num,concat('货物',num,'号'),round(RAND()*100000,2),FLOOR(RAND()*100000),FLOOR(RAND()*10),1,5435343235,'2019-04-20 22:37:15','2019-04-20 22:37:15');
set num=num+1;
end while;
end$

delimiter ;
```

直接执行存储过程：

```sql
call insert_tb_item(1);
Query OK, 1 row affected (12 min 48.67 sec)
```















































