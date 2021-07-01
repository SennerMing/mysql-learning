# MySQL高级

## 1. 索引

### 1.1 索引概述

MySQL官方对索引的定义为：索引(index)是帮助MySQL高效获取数据的数据结构(有序)。在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用(指向)数据，这样就可以在这些数据结构上实现高级查找算法，这种数据结构就是索引。如下面的示意图所示：

![image-20210701094359037](/Users/liuxiangren/mysql-learning/senior-img/index-brief.png)

```markdown
#左侧是没有建立索引的数据，Col1为序号，Col2为ID
问题1：假设我们要查找ID为34的这条数据
那么，搜索是自上而下的，第一次就搜索到了要找的数据
问题2：假设我们要查找ID为91的这条数据
那么，搜索是自上而下的，第四次才能搜索到要找的数据
问题3：假设我们要查找ID为3的这条数据
那么我们得一直搜索到最底部才能找到我们想要的数据
问题4：假设我们一张表中有500W条数据，那假设我们要找的数据就是在最后一个
可想而知，这样查找的话，效率是非常慢的

#右侧是建立了索引的数据
问题1：那么当数据插入的时候，就会按照ID值进行排序插入到二叉树中，也就是会进行对比。查找的时候，就会进行快速查找(二分查找)
```

左边是数据表，一共有两列七条记录，最左边的是数据记录的物理地址(注意逻辑上相邻的记录在磁盘上也并不是一定物理相邻的)。为了加快Col2的查找，可以维护一个右边所示的二叉查找树，每个节点分别包含索引值和一个指向对应数据记录物理地址的指针，这样就可以运用二叉查找快速获取到相应的数据。

一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上。索引是数据库中用来提高性能的最常用的工具。

### 1.2 索引优势劣势

优势

- 索引大大减小了服务器需要扫描的数据量，从而大大加快数据的检索速度，这也是创建索引的最主要的原因。
- 索引可以帮助服务器避免排序和创建临时表
- 索引可以将随机IO变成顺序IO
- 索引对于InnoDB（对索引支持行级锁）非常重要，因为它可以让查询锁更少的元组，提高了表访问并发性
- 关于InnoDB、索引和锁：InnoDB在二级索引上使用共享锁（读锁），但访问主键索引需要排他锁（写锁）
- 通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。
- 可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。
- 在使用分组和排序子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。
- 通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。
  

劣势

- 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加

- 索引需要占物理空间，除了数据表占用数据空间之外，每一个索引还要占用一定的物理空间，如果需要建立聚

  簇索引，那么需要占用的空间会更大

- 对表中的数据进行增、删、改的时候，索引也要动态的维护，这就降低了整数的维护速度

- 如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。

- 对于非常小的表，大部分情况下简单的全表扫描更高效；



**索引使用准则**

索引是建立在数据库表中的某些列的上面。因此，在创建索引的时候，应该仔细考虑在哪些列上可以创建索引，在哪些列上不能创建索引。

**应该创建索引的列**

- 在经常需要搜索的列上，可以加快搜索的速度

- 在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构

- 在经常用在连接（JOIN）的列上，这些列主要是一外键，可以加快连接的速度

- 在经常需要根据范围（<，<=，=，>，>=，BETWEEN，IN）进行搜索的列上创建索引，因为索引已经排序，

  其指定的范围是连续的

- 在经常需要排序（order by）的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序

  查询时间；

- 在经常使用在WHERE子句中的列上面创建索引，加快条件的判断速度。
  

**不该创建索引的列**

- 对于那些在查询中很少使用或者参考的列不应该创建索引。
  若列很少使用到，因此有索引或者无索引，并不能提高查询速度。相反，由于增加了索引，反而降低了系统的维护速度和增大了空间需求。
- 对于那些只有很少数据值或者重复值多的列也不应该增加索引。
  这些列的取值很少，例如人事表的性别列，在查询的结果中，结果集的数据行占了表中数据行的很大比例，即需要在表中搜索的数据行的比例很大。增加索引，并不能明显加快检索速度。
- 对于那些定义为text, image和bit数据类型的列不应该增加索引。
  这些列的数据量要么相当大，要么取值很少。
- 当该列修改性能要求远远高于检索性能时，不应该创建索引。（修改性能和检索性能是互相矛盾的）
  



### 1.3 索引结构

索引是在MySQL的存储引擎层中实现的，而不是在服务器层实现的。所以每种存储引擎的索引都不一定完全相同，也不是所有的存储引擎都支持所有的索引类型的。MySQL目前提供一下四种索引：

- BTREE索引：最想见的索引类型，大部分索引都支持B树索引
- HASH索引：只有Memory引擎支持，使用场景简单
- R-tree索引（空间索引）：空间索引是MyISAM引擎的一个特殊索引类型，主要用于地理空间的数据类型，通常使用较少
- FULL-TEXT（全文索引）：全文索引也是MyISAM的一个特殊索引类型，主要用于全文索引，InnoDB从MySQL5.6版本开始支持全文索引

**MyISAM、InnoDB、Memory三种存储引擎对各种索引类型的支持**

| 索引      | InnoDB引擎      | MyISAM引擎 | Memory引擎 |
| --------- | --------------- | ---------- | ---------- |
| BTREE     | 支持            | 支持       | 支持       |
| HASH      | 不支持          | 不支持     | 支持       |
| R-TREE    | 不支持          | 支持       | 不支持     |
| FULL-TEXT | 5.6版本之后支持 | 支持       | 不支持     |

我们平常所说的索引，如果没有特别指明，都是指B+树（多路搜索树，并不一定是二叉树）结构组织的索引。其中聚集索引、复合索引、前缀索引、唯一索引默认都是使用B+Tree树索引，统称为索引。



#### 1.3.1 B-TREE

B-TREE就是B树，也叫多路搜索树，树高一层意味着多以次的磁盘I/O，下图是3阶B数

![image-20210701112432690](/Users/liuxiangren/mysql-learning/senior-img/B-TREE-example.png)

B树的特征：

- 关键字集合分布在整颗树中；
- 任何一个关键字出现且只出现在一个结点中；
- 搜索有可能在非叶子结点结束；
- 其搜索性能等价于在关键字全集内做一次二分查找；
- 自动层次控制；

#### 1.3.2 B+TREE

B+树是B-树的变体，也是一种多路搜索树

![image-20210701114751867](/Users/liuxiangren/mysql-learning/senior-img/B+TREE.png)



B+树的特征：

- 所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
- 不可能在非叶子结点命中；
- 非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
- 每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。
- 更适合文件索引系统；



#### 1.3.3 HASH

哈希索引就是采用一定的哈希算法，把键值换算成新的哈希值，检索时不需要类似B+树那样从根节点到叶子节点逐级查找，只需一次哈希算法即可立刻定位到相应的位置，速度非常快。

![image-20210701121406177](/Users/liuxiangren/mysql-learning/senior-img/HASH-Example.png)



Hash索引仅仅能满足"=",“IN"和”<=>"查询，不能使用范围查询。也不支持任何范围查询，例如WHERE price > 100。
　　
由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样。

**补充：索引存储在文件系统中**

索引是占据物理空间的，在不同的存储引擎中，索引存在的文件也不同。存储引擎是基于表的，以下分别使用MyISAM和InnoDB存储引擎建立两张表。

![存储引擎是基于表的，以下建立两张别使用MyISAM和InnoDB引擎的表，看看其在文件系统中对应的文件存储格式。](/Users/liuxiangren/mysql-learning/senior-img/mysql-engine-store-file-struct.png)

存储引擎为MyISAM：

- .frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等

- .MYD：MyISAM DATA，用于存储MyISAM表的数据

- .MYI：MyISAM INDEX，用于存储MyISAM表的索引相关信息



存储引擎为InnoDB：

- .frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
- .ibd：InnoDB DATA，表数据和索引的文件。该表的索引(B+树)的每个非叶子节点存储索引，叶子节点存储索引和索引对应的数据



#### 1.3.4 索引分类

MySQL 的索引有两种分类方式：逻辑分类和物理分类。

##### 1.3.4.5 逻辑分类

有多种逻辑划分的方式，比如按功能划分，按组成索引的列数划分等

###### 1.3.4.5.1 **按功能划分**

- 主键索引：一张表只能有一个主键索引，不允许重复、不允许为 NULL；

```sql
ALTER TABLE TableName ADD PRIMARY KEY(column_list); 
```

- 唯一索引：数据列不允许重复，允许为 NULL 值，一张表可有多个唯一索引，索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。

```sql
CREATE UNIQUE INDEX IndexName ON `TableName`(`字段名`(length));
# 或者
ALTER TABLE TableName ADD UNIQUE (column_list); 
```

- 普通索引：一张表可以创建多个普通索引，一个普通索引可以包含多个字段，允许数据重复，允许 NULL 值插入；

```sql
CREATE INDEX IndexName ON `TableName`(`字段名`(length));
# 或者
ALTER TABLE TableName ADD INDEX IndexName(`字段名`(length));
```

- 全文索引：它查找的是文本中的关键词，主要用于全文检索。（篇幅较长，下文有独立主题说明）

###### 1.3.4.5.2 按列数划分

- 单例索引：一个索引只包含一个列，一个表可以有多个单例索引。

- 组合索引：一个组合索引包含两个或两个以上的列。查询的时候遵循 mysql 组合索引的 “最左前缀”原则，即使用 where 时条件要按照建立索引的时候字段的排列方式放置索引才会生效。



##### 1.3.4.6 物理分类

分为聚簇索引和非聚簇索引（有时也称辅助索引或二级索引）

###### 1.3.4.6.1 聚簇索引和非聚簇索引

```markdown
聚簇是为了提高某个属性(或属性组)的查询速度，把这个或这些属性(称为聚簇码)上具有相同值的元组集中存放在连续的物理块。
```

聚簇索引（clustered index）不是单独的一种索引类型，而是一种数据存储方式。这种存储方式是依靠B+树来实现的，根据表的主键构造一棵B+树且B+树叶子节点存放的都是表的行记录数据时，方可称该主键索引为聚簇索引。聚簇索引也可理解为将数据存储与索引放到了一块，找到索引也就找到了数据。

非聚簇索引：数据和索引是分开的，B+树叶子节点存放的不是数据表的行记录。

虽然InnoDB和MyISAM存储引擎都默认使用B+树结构存储索引，但是只有InnoDB的主键索引才是聚簇索引，InnoDB中的辅助索引以及MyISAM使用的都是非聚簇索引。每张表最多只能拥有一个聚簇索引。

拓展：聚簇索引优缺点

优点：

- 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快
- 聚簇索引对于主键的排序查找和范围查找速度非常快

缺点：

- 插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式，否则将会出现页分裂，严重影响性能。因此，对于InnoDB表，我们一般都会定义一个自增的ID列为主键（主键列不要选没有意义的自增列，选经常查询的条件列才好，不然无法体现其主键索引性能）
- 更新主键的代价很高，因为将会导致被更新的行移动。因此，对于InnoDB表，我们一般定义主键为不可更新。
- 二级索引访问需要两次索引查找，第一次找到主键值，第二次根据主键值找到行数据。



【补充】Mysql中key 、primary key 、unique key 与index区别，以下为补充内容

```markdown
一、key 与 index 含义
key具有两层含义：1.约束（约束和规范数据库的结构完整性）2.索引
index：索引

二、key种类
key:等价普通索引 key 键名(列)

primary key：
	-	约束作用(constraint)，主键约束(unique and not null，一表一主键，唯一标识记录)，规范存储主键和强调唯一性
	- 为这个key建立主键索引

unique key：
	- 约束作用(constraint)，unique约束（保证列或集合提供了唯一性）
	- 为这个key建立一个唯一索引

foreign key：
	-	约束作用(constraint)，外键约束，规范数据的引用完整性
	- 为这个key建立一个普通索引；

三、实战分析
建立一张表
```

```sql
create table user(
	id int auto_increment,
	username varchar(100) not null,
	user_id int(8) primary key,
	depart_no int not null,
	corp varchar(100),
	phone char(11),
	key auto_id (id),
	unique key phone (phone),
	index username_depart_corp (username,depart_no,corp),
	constraint fk_user_depart foreign key(depart_no) references depart(id);
)engine=innodb charset=utf8mb4;
```

auto_increment修饰的字段需要是一个候选键，需要用key指定，否则报错。我们看下表的结构：

![key-types-table-example](/Users/liuxiangren/mysql-learning/senior-img/key-types-table-example.png)

查看表的索引

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/key-types-table-desc-example.png)

可见key也会生成索引

**key值类型**

- PRI 主键约束
- UNI 唯一约束
- MUL 可以重复

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/key-types-value.png)

如果一个Key有多个约束，将显示约束优先级最高的， PRI>UNI>MUL

#### 1.3.5 InnoDB和MyISAM实现

##### 1.3.5.1 InnoDB索引实现

InnoDB使用B+TREE存储数据，除了主键索引为聚簇索引，其它索引均为非聚簇索引。

一个表中只能存在一个聚簇索引（主键索引），但可以存在多个非聚簇索引。

InnoDB表的索引和数据是存储在一起的，`.idb`表数据和索引的文件

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/Innodb-index-file-struct-example.png)

###### 1.3.5.1.1 聚簇索引（主键索引）

B+树 叶子节点包含数据表中行记录就是聚簇索引（索引和数据是存放在一块的）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/Innodb-pk-example.png)

可以看到叶子节点包含了完整的数据记录，这就是聚簇索引。因为InnoDB的数据文件（.idb）按主键聚集，所以InnoDB必须有主键（MyISAM可以没有），如果没有显示指定主键，则选取首个为唯一且非空的列作为主键索引，如果还没具备，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形。

**主键索引结构分析：**

- B+树单个叶子节点内的行数据按主键顺序排列，物理空间是连续的（聚簇索引的数据的物理存放顺序与索引顺序是一致的）；
- 叶子节点之间是通过指针连接，相邻叶子节点的数据在逻辑上也是连续的（根据主键值排序），实际存储时的数据页（叶子节点）可能相距甚远。

**非聚簇索引（辅助索引或二级索引）**

在聚簇索引之外创建的索引（不是根据主键创建的）称之为辅助索引，辅助索引访问数据总是需要二次查找。辅助索引叶子节点存储的不再是行数据记录，而是主键值。首先通过辅助索引找到主键值，然后到主键索引树中通过主键值找到数据行。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/non-cluster-index-example.png)

**拓展：InnoDB索引优化**

InnoDB中主键不宜定义太大，因为辅助索引也会包含主键列，如果主键定义的比较大，其他索引也将很大。如果想在表上定义 、很多索引，则争取尽量把主键定义得小一些。InnoDB 不会压缩索引。

InnoDB中尽量不使用非单调字段作主键（不使用多列），因为InnoDB数据文件本身是一颗B+Tree，非单调的主键会造成在插入新记录时数据文件为了维持B+Tree的特性而频繁的分裂调整，十分低效，而使用自增字段作为主键则是一个很好的选择。

##### 1.3.5.2 MyISAM索引实现

MyISAM也使用B+Tree作为索引结构，但具体实现方式却与InnoDB截然不同。MyISAM使用的都是非聚簇索引。

MyISAM表的索引和数据是分开存储的，`.MYD`表数据文件 `.MYI`表索引文件

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/MyISAM-index-struct.png)

**MyISAM主键索引**

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/myisam-index-pk-example.png)

可以看到叶子节点的存放的是数据记录的地址。也就是说索引和行数据记录是没有保存在一起的，所以MyISAM的主键索引是非聚簇索引。

**MyISAM辅助索引**

在MyISAM中，主键索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。 MyISAM辅助索引也是非聚簇索引。

###### 1.3.5.2.1 InnoDB和MyISAM的索引检索过程

对于InnoDB和MyISAM而言，主键索引是根据主关键字来构建的B+树存储结构，辅助索引则是根据辅助键来构造的B+树存储结构，彼此的索引树都是相互独立的。

InnoDB辅助索引的访问需要两次索引查找，第一次从辅助索引树找到主键值，第二次根据主键值到主键索引树中找到对应的行数据。

MyISM使用的是非聚簇索引，表数据存储在独立的地方，这两棵（主键和辅助键）B+树的叶子节点都使用一个地址指向真正的表数据。由于索引树是独立的，通过辅助键检索无需访问主键的索引树。

假想一个表如下图存储了4行数据。其中Id作为主索引，Name作为辅助索引。图示清晰的显示了聚簇索引和非聚簇索引的差异。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/myisam-vice-index-example.png)



##### 1.3.5.3 聚簇索引和非聚簇索引的区别

- 聚簇索引的叶子节点存放的是数据行（主键值也是行内数据），支持覆盖索引；而二级索引的叶子节点存放的是主键值或指向数据行的指针。
- 由于叶子节点(数据页)只能按照一棵B+树排序，故一张表只能有一个聚簇索引。辅助索引的存在不影响聚簇索引中数据的组织，所以一张表可以有多个辅助索引。

#### 1.3.6 索引操作

##### 1.3.6.1 创建索引

索引名称 index_name 是可以省略的，省略后，索引的名称和索引列名相同。

```sql
-- 创建普通索引 
CREATE INDEX index_name ON table_name(col_name);

-- 创建唯一索引
CREATE UNIQUE INDEX index_name ON table_name(col_name);

-- 创建普通组合索引
CREATE INDEX index_name ON table_name(col_name_1,col_name_2);

-- 创建唯一组合索引
CREATE UNIQUE INDEX index_name ON table_name(col_name_1,col_name_2);
```

##### 1.3.6.2 修改表结构创建索引

```sql
ALTER TABLE table_name ADD INDEX index_name(col_name);
```

##### 1.3.6.3 创建表时直接指定索引

```sql
CREATE TABLE table_name (
    ID INT NOT NULL,
    col_name VARCHAR (16) NOT NULL,
    INDEX index_name (col_name)
);
```

##### 1.3.6.4 删除索引

```sql
-- 直接删除索引
DROP INDEX index_name ON table_name;

-- 修改表结构删除索引
ALTER TABLE table_name DROP INDEX index_name;
```

##### 1.3.6.5 其他相关命令

```sql
-- 查看表结构
desc table_name;

-- 查看生成表的SQL
show create table table_name;

-- 查看索引信息（包括索引结构等）
show index from table_name;

-- 查看SQL执行时间（精确到小数点后8位）
set profiling = 1;
SQL...
show profiles;
```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/mysql-some-skills-1365.png)



#### 1.3.7 索引实战

索引实战学习的基础，首先应该学会分析SQL的执行，使用EXPLAIN关键字可以模拟优化器执行SQL查询语句，下面我们学习下EXPLAIN。

##### 1.3.7.1 explain

使用EXPLAIN关键字可以模拟优化器执行SQL查询语句，从而知道MySQL是如何处理SQL语句。不展开讲解，大家可自行百度这块知识点。

使用格式：`EXPLAIN SQL...;`

看一下EXPLAIN 查询结果包含的字段（v5.7）

```sql
mysql> explain select * from student;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    2 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------+
```

- id:选择标识符
- select_type:表示查询的类型。
- table:输出结果集的表
- partitions:匹配的分区
- type:表示表的连接类型
- possible_keys:表示查询时，可能使用的索引
- key:表示实际使用的索引
- key_len:索引字段的长度
- ref:列与索引的比较
- rows:扫描出的行数(估算的行数)
- filtered:按表条件过滤的行百分比
- Extra:执行情况的描述和说明
  

##### 1.3.7.2 Extra

本打算展开讲一下Extra的常见的几个值：Using index，Using index condition，Using where，其中Using index 表示使用了覆盖索引，其它的都不好总结。翻阅网上众多博文，目前暂未看到完全总结到位的，且只是简单的查询条件下也是如此。我本打算好好总结一番，发现半天过去了，坑貌似越来越大，先打住，因为我目前没太多时间研究…

我这有个简单的表结构，有兴趣的同学可以多尝试总结（注意 玩总结的，不能只考虑简单查询的情况）

```sql
create table student(
 id int auto_increment primary key,
 name varchar(255) not null,
 c_id int,
 phone char(11),
 guardian varchar(50) not null,
 qq varchar(20) not null,
 index stu_class_phone (name,c_id,phone),
 index qq (qq)
)engine=innodb charset=utf8mb4;
```

这里有我的一个比对关键项表，或许对有心探索的同学有点帮助，估计看了也有点懵，建议先尝试后再回头看我这个表。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/extra-relevant-explain.png)



##### 1.3.7.3 key_len

我也稍微讲解一下网文中鲜有提及key_len字节长度计算规则

![key-len](/Users/liuxiangren/mysql-learning/senior-img/key-_len-example.png)

```sql
mysql> explain select * from student where name='Joe';
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys   | key             | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ref  | stu_class_phone | stu_class_phone | 767     | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from student where name='Joe' and c_id=2;
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys   | key             | key_len | ref         | rows | filtered | Extra |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ref  | stu_class_phone | stu_class_phone | 772     | const,const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from student where name='Joe' and c_id=2 and phone='13500000000';
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys   | key             | key_len | ref               | rows | filtered | Extra |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ref  | stu_class_phone | stu_class_phone | 806     | const,const,const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------------------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

```markdown
如果表结构未限制某列为非空，那么MySQL将会使用一个字节来标识该列对应的值是否为NULL；限定非空，不止not null，还有primary key等隐含非空约束。

字符串类型括号内的数字并不是字节数，而是字符长度，一个字符占几个字节与建表选用的字符集有关，如果表使用的是utf8字符集，那么一个字符占3个字节；注意，对于可变长字符串类（varchar）型的实际占用字节数，除了需要考虑设置了非空与否的那个字节，还要使用2个字节来记录字符串的长度。定长字符串类型（char）则不用额外字节记录长度

整数类型括号内的数字无论是什么，都不影响它实际的字节数，int就是4个字节。int(xx)，xx只是填充占位符，一般配合zerofill使用，只是一种标识，没有多大用处。
```



观察三次Explain 的查询结果，留意key_len与where搜索键的微妙关系，如果type列的值是ref时，ref列的值标识索引参考列的形参。

首先，我们看到key列为stu_class_phone ，说明该查询使用了stu_class_phone索引，这是一个组合索引（name,c_id,phone）。看下这三个字段的结构声明与实际字节计算：

- name varchar(255) not null, （占767字节）
  - ①255字长（utf8字符集，一个字长3字节 ）255*3=765 √
  - ②是否非空 已限定非空（not null） 那就不额外占1字节
  - ③字符串长度 str_len占2字节√
- c_id int,（占5字节）
  - ①是否非空 未限定非空 那将额外占1字节 √
  - ②int 占4字节√
- phone char(11),（占36字节）
  - ①11字长（utf8字符集，一个字长3字节 ）11*3=33√

int(xx) xx无论是多少 int永远4字节 xx只是填充占位符（一种标识 一般配合zerofill使用的）

组合索引满足最左前缀原则就会生效，我们看到三次Explain 的查询中stu_class_phone索引都生效了，第一次采用name构建索引树，第二次采用name+c_id构建索引树，第三次采用name+c_id+phone构建索引树。第一次：key_len就是name的存储字节数，767；第二次：key_len就是name+c_id的存储字节数，767+5=772；第三次：255 * 3 + 2 + 5 + 11 * 3 + 1 = 806
我们再看一条执行计划：

```sql
mysql> explain select * from student where name='Joe' and phone ='13500000000';
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-----------------------+
| id | select_type | table   | partitions | type | possible_keys   | key             | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | student | NULL       | ref  | stu_class_phone | stu_class_phone | 767     | const |    1 |    50.00 | Using index condition |
+----+-------------+---------+------------+------+-----------------+-----------------+---------+-------+------+----------+-----------------------+
```

为什么不是255 * 3 + 11 * 3 + 1 +2=801；却是767？我们看下ref为const，说明只有一个索引键生效，明显就是name，因为 不符合最左前缀原则，phone列被忽视了；也可能是mysql做了优化，发现通过name和phone构建的索引树对查询列 （* 表示全部列）并没有加快了查询速率，自行优化，减少键长。



**拓展：优秀的索引是什么样的？**

- 键长 短
- 精度 高

比如，在保证查询精度的情况下，两个索引的key_len分别为10字节和100字节，数据行的量也一样（大数据量效果更佳），100字节索引检索的时间会比10字节的要多；再者，一个磁盘页能存储的10字节的索引记录的量是100字节的10倍。

##### 1.3.7.4 最左前缀原则

在MySQL建立联合索引时会遵守最左前缀匹配原则，即最左优先（查询条件精确匹配索引的左边连续一列或几列，则构建对应列的组合索引树），在检索数据时也从联合索引的最左边开始匹配。

为了方便讲解，我写了点个人理解的概念声明，如下图：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle.png)

```sql
mysql> create table t(
    -> a int not null,
    -> b char(10) not null,
    -> c int not null,
    -> d varchar(20) not null,
    -> index abc(a,b,c)
    -> )engine=innodb charset=utf8;
    
mysql> insert into t values(1,'hello',1,'world');
mysql> insert into t values(2,'hello',2,'mysql');
```

以下均为筛选条件不包含主键索引情况下：（主键索引优先级最高）

只要筛选条件中含有组合索引最左边的列但不含有主键搜索键的时候，至少会构建包含组合索引最左列的索引树。（如：index(a)）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle-condition1.png)

如果筛选条件全是组合索引最左连续列作为搜索键，将构建连续列组合索引树。（比如：index(a,b)却不能index(a,c)）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle-condition4.png)

MySQL查询优化器会优化and连接，将组合索引列规则排号。（比如：b and a 等同于 a and b）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle-condition5.png)

##### 

查询列都是组合索引列且筛选条件全是组合索引列时，会构建满列组合索引树（index(a,b,c) ）【覆盖索引】

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle-condition2.png)

筛选条件包含普通搜索键但没包含组合索引列最左键，不会构建组合索引树

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/the-left-prefix-principle-condition3.png)

##### 1.3.7.5 前缀索引

有时候需要索引很长的字符列，这会让索引变得大且慢。通常可以以某列开始的部分字符作为索引，这样可以大大节约索引空间，从而提高索引效率。但这样也会降低索引的选择性。索引的选择性是指不重复的索引值和数据表的记录总数的比值，索引的选择性越高则查询效率越高。

以下是一个百万级数据表的简化呈现

![prefix-index](/Users/liuxiangren/mysql-learning/senior-img/prefix-index-example.png)

图一 area 字段没有设置为索引，图二 area 字段设置为前4字符作为索引，图三 area 字段设置前5字符作为索引，当数据是百万当量时候，毫无疑问，图三的索引速度将大大优越于前两个图场景。

```sql
CREATE TABLE `x_test` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `x_name` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
) ENGINE=InnoDB  DEFAULT CHARSET=utf8
```

通过存储过程插入10万数据（不建议使用此方法，太慢了）

```sql
DROP PROCEDURE IF EXISTS proc_initData;
DELIMITER $
CREATE PROCEDURE proc_initData()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i<=100000 DO
        INSERT INTO x_test VALUES(null,RAND()*100000);
        SET i = i+1;
    END WHILE;
END $
CALL proc_initData();
```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/ten-milion-records-example.png)

不使用索引查询某条记录

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/ten-milion-records-example-no-key.png)

使用索引查询某条记录

```sql
alter table x_test add index(x_name(4));
```

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/ten-milion-records-example-with-key.png)

前缀字符并非越多越好，需要在索引的选择性和索引IO读取量中做出衡量。

##### 1.3.7.5 覆盖索引与回表

上文我们介绍过索引可以划分为聚簇索引和辅助索引。在InnoDB中的主键索引就是聚簇索引，主键索引的查询效率也是非常高的，除此之外，还有非聚簇索引，其查询效率稍逊。覆盖索引其形式就是，搜索的索引键中的字段恰好是查询的字段（或是组合索引键中的其它字段）。覆盖索引的查询效率极高，原因在与其不用做回表查询。
student表中存在组合索引 stu_class_phone(name,c_id,phone)，student表结构如下：

```sql
mysql> desc student;
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int(11)      | NO   | PRI | NULL    | auto_increment |
| name     | varchar(255) | NO   | MUL | NULL    |                |
| c_id     | int(11)      | YES  |     | NULL    |                |
| phone    | char(11)     | YES  |     | NULL    |                |
| guardian | varchar(50)  | NO   |     | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
```

最直观的呈现：（通过explain执行分析SQL可观测到Extra字段值包含Using index）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/cover-and-back-table-index-a.png)

当然对于组合索引你还可以查询组合索引键中的其他字段：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/cover-and-back-table-index-c.png)

但是不能包含杂质搜索键（不属于所搜索索引中的列）

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/cover-and-back-table-index-other.png)

典型使用场景： 全表count查询，根据某关键字建立索引，直接count（关键字）即可，如果是count(\*) 则需要回表搜索。（此项做保留，近期发现count(\*) 也是使用了using index，有可能是新版本mysql内部做了优化处理

```sql
alter table student add key(name);

mysql> explain select count(name) from student;
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | index | NULL          | name | 767     | NULL |    1 |   100.00 | Using index |
+----+-------------+---------+------------+-------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.7.6 回表

查询的列数据作为索引树的键值，直接在索引树中得到反馈（存在于索引节点），不用遍历如InnoDB中的叶子节点（存放数据表各行数据）就可得到查询的数据（不用回表）。

下面以InnoDB表中的辅助索引作图示说明：

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/back-table-example.png)

##### 1.3.7.7 全文索引 FULLTEXT

通过数值比较、范围过滤等就可以完成绝大多数我们需要的查询，但是，如果希望通过关键字的匹配来进行查询过滤，那么就需要基于相似度的查询，而不是原来的精确数值比较。全文索引，就是为这种场景设计的，通过建立倒排索引,可以极大的提升检索效率,解决判断字段是否包含的问题。

```markdown
FULLTEXT VS LIKE+%
```

使用LIKE+%确实可以实现模糊匹配，适用于文本比较少的时候。对于大量的文本检索，LIKE+%与全文索引的检索速度相比就不是一个数量级的。

例如: 有title字段，需要查询所有包含 "政府"的记录.，使用 like "%政府%“方式查询，查询速度慢（全表查询）。且当查询包含"政府” OR "中国"的字段时，使用like就难以简单满足，而全文索引就可以实现这个功能。

```sql
FULLTEXT的支持情况
```

- MySQL 5.6 以前的版本，只有 MyISAM 存储引擎支持全文索引；
- MySQL 5.6 及以后的版本，MyISAM 和 InnoDB 存储引擎均支持全文索引;
- 只有字段的数据类型为 char、varchar、text 及其系列才可以建全文索引

InnoDB内部并不支持中文、日文等，因为这些语言没有分隔符。可以使用插件辅助实现中文、日文等的全文索引。

###### 1.3.7.7.1 创建全文索引

```sql
//建表的时候
FULLTEXT KEY keyname(colume1,colume2)  // 创建联合全文索引列

//在已存在的表上创建
create fulltext index keyname on xxtable(colume1,colume2);

alter table xxtable add fulltext index keyname (colume1,colume2);
```

###### 1.3.7.7.2 使用全文索引

全文索引有独特的语法格式，需要配合match 和 against 关键字使用

- match()函数中指定的列必须是设置为全文索引的列
- against()函数标识需要模糊查找的关键字

```sql
create table fulltext_test(
     id int auto_increment primary key,
     words varchar(2000) not null,a
     artical text not null,
     fulltext index words_artical(words,artical)
)engine=innodb default charset=utf8;

insert into fulltext_test values(null,'a','a');
insert into fulltext_test values(null,'aa','aa');
insert into fulltext_test values(null,'aaa','aaa');
insert into fulltext_test values(null,'aaaa','aaaa');
```

好，我们使用全文搜索查看一下

```sql
select * from fulltext_test where match(words,artical) against('a');
select * from fulltext_test where match(words,artical) against('aa');
select * from fulltext_test where match(words,artical) against('aaa');
select * from fulltext_test where match(words,artical) against('aaaa');
```

发现只有aaa和aaaa才能查到记录，为什么会这样呢？

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/usage-of-full-text-index.png)

###### 1.3.7.7.3 全文索引关键词长度阈值

这其实跟全文搜索的关键词的长度阈值有关，可以通过`show variables like '%ft%';`查看。可见InnoDB的全文索引的关键词 最小索引长度 为3。上文使用的是InnoDB引擎建表，同时也解释为什么只有3a以上才有搜索结果。

![在这里插入图片描述](/Users/liuxiangren/mysql-learning/senior-img/full-text-index-keyword-len-threshold.png)

设置关键词长度阈值，可以有效的避免过短的关键词，得到的结果过多也没有意义。

也可以手动配置关键词长度阈值，修改MySQL配置文件，在[mysqld]的下面追加以下内容，设置关键词最小长度为5。

```sql
[mysqld]
innodb_ft_min_token_size = 5
ft_min_word_len = 5
```

然后重启MySQL服务器，还要修复索引，不然参数不会生效

```sql
repair table 表名 quick;
```

为什么上文搜索关键字为aaa的时候，有且仅有一条aaa的记录，为什么没有aaaa的记录呢？

###### 1.3.7.7.4 全文索引模式

**自然语言的全文索引 IN NATURAL LANGUAGE MODE**

默认情况下，或者使用 IN NATURAL LANGUAGE MODE 修饰符时，match() 函数对文本集合执行自然语言搜索，上面的例子都是自然语言的全文索引。

自然语言搜索引擎将计算每一个文档对象和查询的相关度。这里，相关度是基于匹配的关键词的个数，以及关键词在文档中出现的次数。在整个索引中出现次数越少的词语，匹配时的相关度就越高。

MySQL在全文查询中会对每个合适的词都会先计算它们的权重，如果一个词出现在多个记录中，那它只有较低的权重；相反，如果词是较少出现在这个集的文档中，它将得到一个较高的权重。

MySQL默认的阀值是50%。如果一个词语的在超过 50% 的记录中都出现了，那么自然语言的搜索将不会搜索这类词语。

上文关键词长度阈值是3，所以相当于只有两条记录：aaa 和 aaaa
aaa 权重 2/2=100% 自然语言的搜索将不会搜索这类词语aaa了 而是进行精确查找 aaaa不会出现在aaa的结果集中。

这个机制也比较好理解，比如一个数据表存储的是一篇篇的文章，文章中的常见词、语气词等等，出现的肯定比较多，搜索这些词语就没什么意义了，需要搜索的是那些文章中有特殊意义的词，这样才能把文章区分开。
**布尔全文索引 IN BOOLEAN MODE**

在布尔搜索中，我们可以在查询中自定义某个被搜索的词语的相关性，这个模式和lucene中的BooleanQuery很像,可以通过一些操作符,来指定搜索词在结果中的包含情况。

建立如下表：

```sql
CREATE TABLE articles (
    id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    FULLTEXT (title,body)
) ENGINE=InnoDB
```

- `+` （AND）全文索引列必须包含该词且全文索引列（之一）有且仅有该词
- `-` （NOT）表示必须不包含,默认为误操作符。如果只有一个关键词且使用了`-` ，会将这个当成错误操作，相当于没有查询关键词；如果有多个关键词，关键词集合中不全是负能符（`~ -`），那么`-`则强调不能出现。

```sql
-- 查找title,body列中有且仅有apple（是apple不是applexx 也不是 xxapple）但是不含有banana的记录
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple -banana' IN BOOLEAN MODE);
```

- `>` 提高该词的相关性，查询的结果靠前
- `<` 降低该词的相关性，查询的结果靠后

```sql
-- 返回同时包含apple（是apple不是applexx 也不是 xxapple）和banana或者同时包含apple和orange的记录。但是同时包含apple和banana的记录的权重高于同时包含apple和orange的记录。
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple +(>banana <orange)' IN BOOLEAN MODE);
```

- `~` 异或，如果包含则降低关键词整体的相关性

```sql
-- 返回的记录必须包含apple（且不能是applexx 或 xxapple），但是如果同时也包含banana会降低权重（只出现apple的记录会出现在结果集前面）。但是它没有 +apple -banana 严格，因为后者如果包含banana压根就不返回。
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('+apple ~banana' IN BOOLEAN MODE);
```

- `*` 通配符，表示关键词后面可以跟任意字符

```sql
-- 返回的记录可以为applexx
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('apple*' IN BOOLEAN MODE);
```

- 空格 表示OR

```sql
-- 查找title,body列中包含apple（是apple不是applexx 也不是 xxapple）或者banana的记录，至少包含一个
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('apple banana' IN BOOLEAN MODE); 
```

- `""` 双引号，效果类似`like '%some words%'`

```sql
-- 模糊匹配 “apple banana goog”会被匹配到，而“apple good banana”就不会被匹配
SELECT * FROM articles WHERE MATCH (title,body) AGAINST ('"apple banana"' IN BOOLEAN MODE);
```

###### 1.3.7.7.5 InnoDB中FULLTEXT中文支持

InnoDB内部并不支持中文、日文等，因为这些语言没有分隔符。可以使用插件辅助实现中文、日文等的全文索引。

MySQL内置ngram插件便可解决该问题。

```sql
FULLTEXT (title, body) WITH PARSER ngram
ALTER TABLE articles ADD FULLTEXT INDEX ft_index (title,body) WITH PARSER ngram;
CREATE FULLTEXT INDEX ft_index ON articles (title,body) WITH PARSER ngram;
```

#### 1.3.8 索引失效

数据库表中添加索引后确实会让查询速度起飞，但前提必须是正确的使用索引来查询，如果以错误的方式使用，则即使建立索引也会不奏效。即使建立索引，索引也不会生效：

```sql
 create table tb1(
     nid int auto_increment primary key,
     name varchar(100) not null,
     email varchar(100) not null,
     num int,
     no_index char(10),
     index(name),
     index(email),
     index(num)
)engine=innodb;
```

以下说明都 排除覆盖索引 情况下：

##### 1.3.8.1 \> < 范围查询

mysql 会一直向右匹配直到遇到索引搜索键使用`>、<`就停止匹配。一旦权重最高的索引搜索键使用`>、<`范围查询，那么其它`>、<`搜索键都无法用作索引。即索引最多使用一个`>、<`的范围列，因此如果查询条件中有两个`>、<`范围列则无法全用到索引。

##### 1.3.8.2 like %xx

如搜索键值以通配符`%开头`（如：`like '%abc'`），则索引失效，直接全表扫描；若只是以%结尾，则不影响索引构建。

```sql
mysql> explain select * from tb1 where name like '%oe';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.8.3 对索引列进行运算

如果查询条件中含有函数或表达式，将导致索引失效而进行全表扫描。 `select * from user where YEAR(birthday) < 1990`

```sql
mysql> explain select * from tb1 where length(name)>2;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.8.4 or 条件索引问题

`or` 的条件列除了同时是主键的时候，索引才会生效。其他情况下的，无论条件列是什么，索引都失效。

```sql
mysql> explain select * from tb1 where nid=1 or nid=2;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    2 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from tb1 where name='Joe' or name='Tom';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | name          | NULL | NULL    | NULL |    3 |    66.67 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.8.5 数据类型不一致（隐式类型转换导致的索引失效）

如果列是字符串类型，传入条件是必须用引号引起来，不然报错或索引失效。

```sql
mysql> explain select * from tb1 where name=12;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | name          | NULL | NULL    | NULL |    3 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.8.6 != 问题

普通索引使用 `!=`索引失效，主键索引没影响。
where语句中索引列使用了负向查询，可能会导致索引失效。
负向查询包括：NOT、!=、<>、NOT IN、NOT LIKE等。

```sql
mysql> explain select * from tb1 where name!='Joe';
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | name          | NULL | NULL    | NULL |    3 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

##### 1.3.8.7 联合索引违背最左匹配原则

联合索引中，where中索引列违背最左匹配原则，一定会导致索引失效（上文有说）

##### 1.3.8.8 order by问题

order by 对主键索引排序会用到索引，其他的索引失效

```sql
mysql> explain select * from tb1 order by name;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
|  1 | SIMPLE      | tb1   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from tb1 order by nid;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------+
|  1 | SIMPLE      | tb1   | NULL       | index | NULL          | PRIMARY | 4       | NULL |    3 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------+
```

#### 1.3.9 【主键妙用】LIMIT分页

若需求是每页显示10条数据，如何建立分页？

我们可以先使用LIMIT尝试：

```sql
--第一页
SELECT * FROM table_name LIMIT 0,10;
--第二页
SELECT * FROM table_name LIMIT 10,10;
--第三页
SELECT * FROM table_name LIMIT 20,10;
```

但是这样做有如下弊端：
每一条select语句都会从1遍历至当前位置，若跳转到第100页，则会遍历1000条记录

改善：
若已知每页的max_id和min_id，则可以通过主键索引来快速定位:

```sql
--下一页
SELECT * FROM table_name WHERE id in (SELECT id FROM table_name WHERE id > max_id LIMIT 10);
--上一页
SELECT * FROM table_name WHERE id in (SELECT id FROM table_name WHERE id < min_id ORDER BY id DESC LIMIT 10);
--当前页之后的某一页
SELECT * FROM table_name WHERE id in (SELECT id FROM (SELECT id FROM (SELECT id FROM table_name WHERE id < min_id ORDER BY id desc LIMIT (页数差*10)) AS N ORDER BY N.id ASC LIMIT 10) AS P ORDER BY P.id ASC);
--当前页之前的某一页
SELECT * FROM table_name WHERE id in (SELECT id FROM (SELECT id FROM (SELECT id FROM table_name WHERE id > max_id LIMIT (页数差*10)) AS N ORDER BY N.id DESC LIMIT 10) AS P) ORDER BY id ASC;
```

可能版本会不支持：`This version of MySQL doesn't yet support 'LIMIT` 只要加多一个子查询即可

```sql
SELECT * FROM x_test WHERE id in (SELECT id FROM ( SELECT id FROM x_test WHERE id>max_id LIMIT 10) t);
```





### 1.10 索引学习参考

MySQL索引总结：https://blog.csdn.net/wangfeijiu/article/details/113409719



