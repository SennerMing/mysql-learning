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

```markdown
BTree又叫多路平衡搜索树，一棵M叉的BTree特性如下：
 - 树中每个节点最多包含m个孩子
 - 除根节点与叶子节点外，每个节点至少有[ceil(m/2)个孩子]
 - 若根节点不是叶子节点，则至少有两个孩子
 - 所有的叶子节点都在同一层
 - 每个非叶子节点由n个key与n+1个指针组成，其中[ceil(m/2)-1] <= n <= m-1
** ceil 向上取整 **

以5叉BTree为例，key的数量：公式推导[ceil(m/2)-1] <= n <= m-1。所以2<=n<=4。当n>4时，中间节点分裂到父节点，两边节点分裂。
```

BTree小栗子：

插入C N G A H E K Q M F W L T Z D P R X Y S数据为例。

1. 插入前4个字母，下面五个小格子代表，每个给叶子节点由n个key和n+1个指针组成，这边就是4个key和5个指针

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-first.png" alt="image-20210701224101267" style="zoom:50%;" />

2. 插入H，那么第一层已经没有位置了，也就是说n>4时，中间的元素G字母向上分裂到新的节点(G为什么是中间节点，你想想26个英文字母顺序：A...C...G H..N，这样一排列这5个字母，G可不就是在中间啦)

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-second.png" alt="image-20210701215459300" style="zoom:50%;" />

3. 插入E，K，Q不需要分裂

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-third.png" alt="image-20210701220044621" style="zoom:50%;" />

4. 插入M，中间元素M字母向上分裂到父节点G；思路是，M比G大走右边，但是右边子树已经有4个节点了，所以进行排序找出中间的元素(..H..K..M..N..Q)，那中间的字母就是要插入的M，按照前面的BTree规则，只要超过了4个节点，中间的就要向上分裂，然后就想第2步那样分裂成[HK] [NQ]、那么就变成

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-forth.png" alt="image-20210701225149796" style="zoom:50%;" />

5. 插入F，W，L，T不需要分裂

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-fifth.png" alt="image-20210701225833755" style="zoom:50%;" />

6. 插入Z，中间元素T向上分裂到父节点中，那Z的位置应该是在第二层最右侧也就是(..N..Q..T..W..Z)，那么中间元素显而易见的就是T了

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-six.png" alt="image-20210701230416381" style="zoom:50%;" />

7. 然后插入D，中间元素D向上分裂到父节点中，然后插入P,R,X,Y不需要分裂

   <img src="/Users/liuxiangren/mysql-learning/senior-img/btree-character-seven.png" alt="image-20210701230859489" style="zoom:50%;" />

8. 最后插入S，NPQR节点已经为4个了，中间的Q向上分裂，但分裂后父节点DGMT也是4个节点了，中间节点M向上分裂

到此，该BTREE树就已经构建完成了，BTREE树和二叉树相比，查询数据的效率更高，因为对于相同的数量来说，BTREE的层级结构比二叉树小，因此搜索速度快。







#### 1.3.2 B+TREE

B+树是B-树的变体，也是一种多路搜索树

![image-20210701114751867](/Users/liuxiangren/mysql-learning/senior-img/B+TREE.png)



B+树的特征：

- 所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
- 不可能在非叶子结点命中；
- 非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
- 每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。
- 更适合文件索引系统；

```markdown
B+Tree结构：
B+Tree为BTree的变种，B+Tree与BTree的区别为：
 - n叉B+Tree最多含有n个key，而BTree最多含有n-1个key
 - B+Tree的叶子节点保存所有的key信息，依key大小顺序排列
 - 所有的非叶子节点都可以看做是key的索引部分

由于B+Tree只有叶子节点保存key信息，查询任何key都要从root走到叶子。所以B+Tree的查询效率更加稳定
```

##### 1.3.2.1 MySQL中的B+TREE

MySQL索引数据结构对经典的B+TREE进行了优化。在原B+TREE的基础上，增加了一个指向相邻叶子节点的链表指针，就形成了带有顺序指针的B+TREE，提高了区间访问的性能。

MySQL中的B+TREE索引结构示意图：

![image-20210702160449953](/Users/liuxiangren/mysql-learning/senior-img/mysql-b+tree-index-struct-desc.png)





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

##### 1.3.4.1 逻辑分类

有多种逻辑划分的方式，比如按功能划分，按组成索引的列数划分等

###### 1.3.4.1.1 **按功能划分**

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

###### 1.3.4.1.2 按列数划分

- 单例索引：一个索引只包含一个列，一个表可以有多个单例索引。

- 组合索引：一个组合索引包含两个或两个以上的列。查询的时候遵循 mysql 组合索引的 “最左前缀”原则，即使用 where 时条件要按照建立索引的时候字段的排列方式放置索引才会生效。



##### 1.3.4.2 物理分类

分为聚簇索引和非聚簇索引（有时也称辅助索引或二级索引）

###### 1.3.4.2.1 聚簇索引和非聚簇索引

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

##### 1.3.4.3 索引分类（最强版）

1. 单值索引：即一个索引只包含单个列，一个表可以有多个单列索引
2. 唯一索引：索引列的值必须唯一，但允许有空值
3. 复合索引：即一个索引包含多个列



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

## 2 索引(真枪实弹版)

### 2.1 索引语法

索引，在创建表的时候，可以同时创建，也可以随时增加新的索引。

准备环境：

```sql
create database demo_01 default charset=utf8mb4;

use demo_01;

create table `city`(
`city_id` int(11) NOT NULL AUTO_INCREMENT,
`city_name` varchar(50) NOT NULL,
`country_id` int(11) NOT NULL,
PRIMARY KEY (`city_id`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;

create table country(
country_id int(11) NOT NULL AUTO_INCREMENT,
country_name varchar(100) NOT NULL,
PRIMARY KEY (country_id)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;



insert into `city`(`city_id`,`city_name`,`country_id`) values(1,'西安',1);
insert into `city`(`city_id`,`city_name`,`country_id`) values(2,'NewYork',2);
insert into `city`(`city_id`,`city_name`,`country_id`) values(3,'北京',1);
insert into `city`(`city_id`,`city_name`,`country_id`) values(4,'上海',1);


insert into `country`(`country_id`,`country_name`) values(1,'CHINA');
insert into `country`(`country_id`,`country_name`) values(2,'AMERICA');
insert into `country`(`country_id`,`country_name`) values(3,'JAPAN');
insert into `country`(`country_id`,`country_name`) values(4,'UK');


#查看表结构
desc city;
#查看建表语句
show create table city;
```

#### 2.1.1 创建索引

语法：

```sql
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX index_name
[USING index_type]
ON tbl_name(index_col_name,...)

index_col_name : column_name[(length)][ASC | DESC]
```

示例：为city表中的city_name字段创建索引；

```sql
create index idx_city_name on city(city_name);
```



#### 2.1.2 查看索引

语法：

```sql
show index from table_name;
```

示例：查看city表中的索引信息

```sql
show index from city \G;
*************************** 1. row ***************************
        Table: city
   Non_unique: 0
     Key_name: PRIMARY
 Seq_in_index: 1
  Column_name: city_id
    Collation: A
  Cardinality: 4
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE 【默认都是BTREE索引】
      Comment: 
Index_comment: 
      Visible: YES
   Expression: NULL
*************************** 2. row ***************************
        Table: city
   Non_unique: 1
     Key_name: idx_city_name
 Seq_in_index: 1
  Column_name: city_name
    Collation: A
  Cardinality: 4
     Sub_part: NULL
       Packed: NULL
         Null: 
   Index_type: BTREE 【默认都是BTREE索引】
      Comment: 
Index_comment: 
      Visible: YES
   Expression: NULL
2 rows in set (0.01 sec)
```

#### 2.1.3 删除索引

语法：

```sql
DROP INDEX index_name ON table_name;
```

示例：想要删除city表上的索引 idx_city_name，可以如下操作：

```sql
drop index idx_city_name on city;
```

#### 2.1.4 ALTER命令

语法：

```sql
1). alter table tb_name add primary key(column_list);
	该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL
	
2). alter table tb_name add unique index_name(column_list);
	该语句创建索引的值必须是唯一的(除了NULL外，NULL可能会出现多次)

3). alter table tb_name add index index_name(column_list);
	添加普通索引，索引值可以出现多次

4). alter table tb_name add fulltext index_name(column_list);
	该语句指定了索引为FULLTEXT，用于全文索引
```

### 2.2 索引设计原则

索引的设计可以遵循一些已有的原则，创建索引的时候请尽量考虑符合这些原则，便于提升索引的使用效率，更高效的使用索引。

- 对查询频次较高，且数据量比较大的表建立索引
- 索引字段的选择，最佳候选列应当从where子句的条件中提取，如果where子句中的组合比较多，那么应当挑选最常用、过滤效果最好的列的组合
- 使用唯一索引，区分度越高，使用索引的效率越高
- 索引可以有效的提升查询数据的效率，但索引数量不是多多益善，索引越多，维护索引的代价自然也就水涨船高。对于插入、更新、删除等DML操作比较频繁的表来说，索引过多，会引入相当高的维护代价，降低DML操作的效率，增加相应操作的时间消耗。另外索引过多的话，MySQL也会犯选择困难症，虽然最终仍然会找到一个可用的索引，但无疑提高了选择的代价
- 使用短索引，索引创建之后也是使用硬盘来存储的，因此提升索引访问的I/O效率，也可以提升总体的访问效率。假如构成索引的字段总长度比较短，那么在给定大小的存储块内可以存储更多的索引值，相应的可以有效提升MySQL访问索引的I/O效率
- 利用最左前缀，N个列组合而成的组合索引，那么相当于是创建了N个索引，如果查询时where子句中使用了组成该索引的前几个字段，那么这条查询SQL可以利用组合索引来提升查询效率。

```sql
创建复合索引：
create index idx_name_email_status on tb_seller(NAME,EMAIL,STATUS);

就相当于
	对name创建了索引；
	对name，email创建了索引；
	对name，email，status创建了索引；
```

## 3 视图

### 3.1 视图概述

视图(View)是一种虚拟存在的表。视图并不在数据库中实际存在，行和列数据来自定义视图的查询中使用的表，并且是在使用视图时动态生成的。通俗地讲，视图就是一条SELECT语句执行后返回的结果集。所以我们在创建视图的时候，主要的工作就落在创建这条SQL查询语句上。

视图相对于普通的表的优势主要包括以下几项。

- 简单：使用视图的用户完全不需要关心后面对应的表的结构、关联条件和筛选条件，对用户来说已经是过滤好的符合条件的结果集
- 安全：使用视图的用户完全不需要关心后面对应的结构、关联条件和筛选条件，对用户来说已经是过滤好的符合条件的结果集。
- 数据独立：一旦视图的结构确定了，可以屏蔽表结构变化对用户的影响，源表增加列对视图没有影响；源表修改列名，则可以通过修改视图来解决，不会造成对访问者的影响

在实际应用当中，为mysql用户隐藏某些数据库字段

### 3.2 创建或者修改视图

创建视图的语法：

```sql
CREATE [OR REPLACE] [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]

VIEW view_name [(column_list)]

AS select_statement

[WITH [CASCADED | LOCAL] CHECK OPTION]
```

修改视图的语法为：

```sql
ALTER [ALGORITHM = {UNDEFINED | MERGE | TEMPLATE}]

VIEW view_name [(column_list)]

AS select_statement

[WITH [CASCADED | LOCAL] CHECK OPTION]
```

```markdown
选项：
	WITH [CASCADED | LOCAL] CHECK OPTION 决定了是否允许更新数据使记录不再满足视图的条件
	
	LOCAL : 只要满足本视图的条件就可以更新
	CASCADED : 必须满足所有针对该视图的所有视图的条件才可以更新
```

实例：

```sql
需求：创建一个视图，显示国家和城市的信息

#查询城市名和对应的国家名
 select c.* ,t.country_name from city c,country t where c.country_id = t.country_id;
+---------+-----------+------------+--------------+
| city_id | city_name | country_id | country_name |
+---------+-----------+------------+--------------+
|       1 | 西安      |          1 | CHINA        |
|       2 | NewYork   |          2 | AMERICA      |
|       3 | 北京      |          1 | CHINA        |
|       4 | 上海      |          1 | CHINA        |
+---------+-----------+------------+--------------+
4 rows in set (0.00 sec)

#那就是创建一个视图，内容就是上面查询出来的结果
create view view_city_country as select c.* ,t.country_name from city c,country t where c.country_id = t.country_id;

#那怎么去操作这个视图呢，前面提到过，这个视图其实就是一张虚拟的表， 那我们怎么操作表就怎么去操作视图
select * from view_city_country;
+---------+-----------+------------+--------------+
| city_id | city_name | country_id | country_name |
+---------+-----------+------------+--------------+
|       1 | 西安      |          1 | CHINA        |
|       2 | NewYork   |          2 | AMERICA      |
|       3 | 北京      |          1 | CHINA        |
|       4 | 上海      |          1 | CHINA        |
+---------+-----------+------------+--------------+
4 rows in set (0.00 sec)

#视图可不可以更新呢？答案是可以的
update view_city_country set city_name='纽约' where city_id = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
#查询出视图的结果
select * from view_city_country;
+---------+-----------+------------+--------------+
| city_id | city_name | country_id | country_name |
+---------+-----------+------------+--------------+
|       1 | 西安      |          1 | CHINA        |
|       2 | 纽约      |          2 | AMERICA      |
|       3 | 北京      |          1 | CHINA        |
|       4 | 上海      |          1 | CHINA        |
+---------+-----------+------------+--------------+
4 rows in set (0.00 sec)


#那么视图的修改结果会不会影响源表数据呢？答案是有影响的，其实更新语句操作的就是源表的数据
select * from city;
+---------+-----------+------------+
| city_id | city_name | country_id |
+---------+-----------+------------+
|       1 | 西安      |          1 |
|       2 | 纽约      |          2 |
|       3 | 北京      |          1 |
|       4 | 上海      |          1 |
+---------+-----------+------------+
4 rows in set (0.00 sec)

#其实视图这个视图到底能不能更新，还取决于上面提到的[WITH [CASCADED | LOCAL] CHECK OPTION]
如果是 WITH LOCAL CHECK OPTION的话，那么只要满足本视图的条件就可以进行更新
如果是 WITH CASCADED CHECK OPTION的话，那么必须满足针对该视图的所有视图的条件才可以更新，默认值。

当使用WITH CHECK OPTION子句创建视图时，MySQL会通过视图检查正在更改的每个行，例如插入，更新，删除，以使其符合视图的定义。因为mysql允许基于另一个视图创建视图，它还会检查依赖视图中的规则以保持一致性。为了确定检查的范围，mysql提供了两个选项：LOCAL和CASCADED。如果我们没有在WITH CHECK OPTION子句中显式指定关键字，则mysql默认使用CASCADED。
关于这两个检查级别可以参考：https://blog.csdn.net/luyaran/article/details/81018763
```

视图不建议进行更新操作，因为其的出现在生产中主要是为了简化查询而来。

### 3.3 查看视图

从MySQL5.1版本开始，使用SHOW TABLES命令的时候不仅显示表的名字，同时也会显示视图的名字，二不存在的单独显示视图的SHOW VIEWS命令

```sql
show tables;
+-------------------+
| Tables_in_demo_01 |
+-------------------+
| city              |
| country           |
| view_city_country |
+-------------------+
3 rows in set (0.01 sec)
```

同样，在使用SHOW TABLES STATUS命令的时候，不但可以显示表的信息，同时也可以显示视图的信息

```sql
show table status like 'view_city_country' \G;
*************************** 1. row ***************************
           Name: view_city_country
         Engine: NULL
        Version: NULL
     Row_format: NULL
           Rows: NULL
 Avg_row_length: NULL
    Data_length: NULL
Max_data_length: NULL
   Index_length: NULL
      Data_free: NULL
 Auto_increment: NULL
    Create_time: 2021-07-02 19:21:48
    Update_time: NULL
     Check_time: NULL
      Collation: NULL
       Checksum: NULL
 Create_options: NULL
        Comment: VIEW
1 row in set (0.00 sec)
```

#### 3.3.1 查看创建视图语句

```sql
show create view view_city_country;
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| View              | Create View                                                                                                                                                                                                                                                                                                                    | character_set_client | collation_connection |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
| view_city_country | CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `view_city_country` AS select `c`.`city_id` AS `city_id`,`c`.`city_name` AS `city_name`,`c`.`country_id` AS `country_id`,`t`.`country_name` AS `country_name` from (`city` `c` join `country` `t`) where (`c`.`country_id` = `t`.`country_id`) | utf8mb4              | utf8mb4_0900_ai_ci   |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------+----------------------+
1 row in set (0.01 sec)
```

### 3.4 删除视图

语法：

```sql
DROP VIEW [IF EXISTS] view_name [,view_name] ...[RESTRICT | CASCADE]
```

实例：

```sql
DROP VIEW view_city_country;

drop view if exists view_city_country;
Query OK, 0 rows affected (0.01 sec)
```

## 4 存储过程和函数

### 4.1 存储过程和函数概述

存储过程和函数是事先经过编译并存储在数据库中的一段SQL语句的集合，调用存储过程和函数可以简化应用开发人员的很多工作，减少数据在数据库和应用服务器之间的传输，对于提高数据处理的效率是有好处的。

存储过程和函数的区别在于函数必须有返回值，而存储过程没有。

函数：是一个有返回值的过程；

过程：是一个没有返回值的函数；

### 4.2 创建存储过程

```sql
CREATE PROCEDURE procedure_name ([proc_parameter[...]])
begin
		-- SQL 语句
end;
```

实例：

```sql
#先指定分隔符
delimiter $
#再进行存储过程的创建
create procedure proc_test1() begin select 'Hello Mysql'; end$
#再恢复分隔符
delimiter ;
```

**知识小贴士**

```markdown
DELIMITER
该关键字用来声明SQL语句的分隔符，告诉MySQL解释器，该段命令是否已经结束了，MySQL是否可以执行了。默认情况下，delimiter是分号";"，在命令行客户端中，如果有一行命令以分号结束，那么回车后，MySQL将会执行该命令。
```

### 4.3 调用存储过程

```sql
call procedure_name();

call proc_test1();
+-------------+
| Hello Mysql |
+-------------+
| Hello Mysql |
+-------------+
1 row in set (0.01 sec)

Query OK, 0 rows affected (0.01 sec)
```

### 4.4 查看存储过程

```sql
#查询db_name数据库中的所有的存储过程
select name from mysql.proc where db='db_name';
#MySQL8
SELECT ROUTINE_TYPE, ROUTINE_NAME FROM INFORMATION_SCHEMA.ROUTINES WHERE ROUTINE_SCHEMA='dbname';

SELECT ROUTINE_TYPE, ROUTINE_NAME FROM INFORMATION_SCHEMA.ROUTINES WHERE ROUTINE_SCHEMA='demo_01';
+--------------+--------------+
| ROUTINE_TYPE | ROUTINE_NAME |
+--------------+--------------+
| PROCEDURE    | proc_test1   |
+--------------+--------------+
1 row in set (0.00 sec)


#查询存储过程的状态信息
show procedure status \G;

#查询某个存储过程的定义
show create procedure test.proc_test1 \G;

show create procedure demo_01.proc_test1 \G;
```

### 4.5 删除存储过程

```sql
DROP PROCEDURE [IF EXISTS] sp_name;
```

示例

```sql
drop procedure if exists proc_test1;
```

### 4.6 语法

存储过程是可以编程的，意味着可以使用变量，表达式，控制结构，来完成比较复杂的功能。

#### 4.6.1 变量

##### 4.6.1.1 DECLARE

通过DECLARE可以定义一个局部变量，该变量的作用范围只能在BEGIN...END块中。

```sql
DECLARE var_name[,...] type [DEFAULT value]
```

示例：

```sql
delimiter $

#可以一次声明多个变量，中间用,号分隔就行了
create procedure pro_test2()
begin
	declare num int default 5;
	select num+10;
	#select concat('num的值为:',num);
end$

delimiter ;

call pro_test2();
+--------+
| num+10 |
+--------+
|     15 |
+--------+
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

##### 4.6.1.2 SET

直接赋值使用SET，可以赋常量或者表达式，具体语法如下：

```sql
SET var_name= expr [, var_name = expr] ...
```

示例：

```sql
DELIMITER $

CREATE PROCEDURE pro_test3()
BEGIN
DECLARE NAME VARCHAR(20);
SET NAME = 'MYSQL';
SELECT NAME;
END$

DELIMITER ;

call pro_test3();
+-------+
| name  |
+-------+
| mysql |
+-------+
1 row in set (0.00 sec)
```

```sql
delimiter $

create procedure pro_test4()
begin
declare num int default 0;
set num = num+10;
select num;
end$

delimiter ;

call pro_test4();
+------+
| num  |
+------+
|   10 |
+------+
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```



##### 4.6.1.3 INTO

通过select...into进行赋值操作

```sql
delimiter $

create procedure test_proc5()
begin
declare num int default 0;
select count(*) into num from city;
select concat('city表中的记录数为：',num);
end$

delimiter ;


call test_proc5();
+------------------------------------------+
| concat('city表中的记录数为:',num)        |
+------------------------------------------+
| city表中的记录数为:4                     |
+------------------------------------------+
1 row in set (0.01 sec)

#如果此时在city表中插入一条数据，那么再次调用的使用那么就返回为5了
```

#### 4.6.2 IF

语法：

```sql
if search_condition then statement_list
	[else if search_condition then statement_list] ...
	[else statement_list]
end if;
```

需求：

```sql
根据定义的身高变量，判定当前身高的所属的身材类型
	180及以上	-----> 身材高挑
	170-180 	-----> 标准身材
	170已下		 -----> 一般身材
```

实例：

```sql
delimiter $

create procedure pro_test6()
begin
declare height int default 175;
declare description varchar(50) default '';
if height >= 180 then set description='身材高挑';
elseif height >= 170 then set description='标准身材';
else set description='一般身材';
end if;
select concat('身高',height,'对应的身材描述:',description);
end$

delimiter ;

call pro_test6();
+--------------------------------------------------------------+
| concat('身高',height,'对应的身材描述:',description)          |
+--------------------------------------------------------------+
| 身高175对应的身材描述:标准身材                               |
+--------------------------------------------------------------+
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

#### 4.6.3 传递参数

语法格式：

```sql
create procedure procedure_name([in/out/inout] 参数名 参数类型)
...

IN			: 该参数可以作为参数，也就是需要调用方传入值，默认
OUT			:	该参数作为输出，也就是该参数可以作为返回值
INOUT	: 既可以作为输入参数，也可以作为输出参数
```

##### 4.6.3.1 IN-输入

需求

```markdown
根据定义的身高变量，判定当前身高的所属的身材类型
```

实例

```sql
delimiter $

create procedure pro_test7(in height int)
begin
declare description varchar(50) default '';
if height >= 180 then set description='身材高挑';
elseif height >= 170 then set description='标准身材';
else set description='一般身材';
end if;
select concat('身高',height,'对应的身材描述:',description);
end$

delimiter ;

#调用一下看看
call pro_test7(180);
+--------------------------------------------------------------+
| concat('身高',height,'对应的身材描述:',description)          |
+--------------------------------------------------------------+
| 身高180对应的身材描述:身材高挑                               |
+--------------------------------------------------------------+
1 row in set (0.00 sec)
```

##### 4.6.3.2 OUT-输出

需求

```markdown
根据传入的身高变量，获取当前身高的所属的身材类型
```

实例

```sql
delimiter $

create procedure pro_test8(in height int,out description varchar(100))
begin
if height >= 180 then set description='身材高挑';
elseif height >= 170 and height < 180 then set description='标准身材';
else set description='一般身材';
end if;
end$

delimiter ;

#调用,@代表的就是用户的会话变量
call pro_test8(188,@description);
select @description;
+--------------+
| @description |
+--------------+
| 身材高挑     |
+--------------+
1 row in set (0.00 sec)
```

**小知识**

@description:这种变量要在变量名称前面加上"@"符号，叫做用户会话变量，代表整个会话过程，都是他的作用域，这个类似于全局变量一样

@@global.sort_buffer_size:这种在变量前加上"@@"符号的，叫做系统变量

```sql
set name = 'sennerming';
ERROR 1193 (HY000) : Unknown system variable 'name'

set @name = 'sennerming'; #一点问题也没有
select @name; #就能查的到了
```



#### 4.6.4 case结构

语法结构：

```sql
方式一：
case case_value
when when_value then statement_list
[when when_value then statement_list]...
[else statement_list]
end case;

方式二：
case
when search_condition then statement_list
[when search_condition then statement_list]...
[else statement_list]
end case;
```

需求

```markdown
给定一个月份，然后计算出所在的季度
```

```sql
delimiter $

create procedure pro_test9(in mon int,out result varchar(10))
begin
case
	when mon >=1 and mon <=2 then set result='第一季度';
	when mon >=4 and mon <=6 then set result='第二季度';
	when mon >=7 and mon <=9 then set result='第三季度';
	else set result='第四季度';
end case;
end$

delimiter ;

#调用
call pro_test9(2,@result);
Query OK, 0 rows affected (0.00 sec)

mysql> select @result;
+--------------+
| @result      |
+--------------+
| 第一季度     |
+--------------+
1 row in set (0.00 sec)
```

#### 4.6.5 while循环

语法结构

```sql
while search_condtion do
 statemtn_list
end while;
```

需求

```sql
计算从1加到n的值
```

示例

```sql
delimiter $

create procedure pro_test10(in num int,out result int)
begin
declare count int default 0;
while count <= 10 do
set num=num+1;
set count=count+1;
end while;
set result = num;
end$

delimiter ;

call pro_test10(100,@result);
Query OK, 0 rows affected (0.01 sec)

mysql> select @result;
+---------+
| @result |
+---------+
|     111 |
+---------+
1 row in set (0.00 sec)

drop procedure if exists pro_test10;
```



#### 4.6.6 repeat结构

有条件的循环控制语句，当满足条件的时候退出循环。while是满足条件财智星，repeat是满足条件就退出循环。

语法结构

```sql
REPEAT
	statement_list
	
	until search_condtion #这里不需要加分号！！！！

END REPEAT;
```

需求

```markdown
计算从1加到n的值
```

示例

```sql
delimiter $

create procedure pro_test11(in n int)
begin
declare total int default 0;
repeat
set total=total+n;
set n=n-1;
until n=0
end repeat;
select total;
end$

delimiter ;

#调用下看看
call pro_test11(8);
+-------+
| total |
+-------+
|    36 |
+-------+
1 row in set (0.00 sec)
```



#### 4.6.7 loop语句

LOOP实现简单的循环，退出循环的条件需要使用其他的语句定义，通常可以使用leave语句实现，具体语法如下

```sql
[begin_label:] LOOP
statement_list
END LOOP [end_label]
```

如果不在statement_list中增加退出循环的语句，那么LOOP语句可以用来实现简单的死循环。

#### 4.6.8 leave语句

用来从标注的流程构造中退出，通常和BEGIN...END或者循环一起使用，下面是一个使用LOOP和LEAVE的简单的例子，退出循环：

```sql
delimiter $

create procedure pro_test12(in n int)
begin
declare total int default 0;
ins: LOOP
if n<=0 then leave ins;
else set n=n-1;set total=total+1;
end if;
end LOOP ins;
select concat('总值是:',total);
end$

delimiter ;

#调用下看看
call pro_test12(8);
+----------------------------+
| concat('总值是:',total)    |
+----------------------------+
| 总值是:8                   |
+----------------------------+
1 row in set (0.00 sec)

```



#### 4.6.9 游标/光标

游标是用来存储查询结果集的数据类型，在存储过程和函数中可以使用光标对结果集进行循环的处理。光标的使用包括光标的声明、OPEN、FETCH和CLOSE，其语法如下：

##### 4.6.9.1 声明光标

```sql
DECLARE cursor_name CURSOR FOR select_statement;
```

##### 4.6.9.2 OPEN光标

```sql
OPEN cursor_name;
```

##### 4.6.9.3 FETCH光标

```sql
FETCH cursor_name INTO var_name [,var_name]...

相当于一个指针
+---------+-----------+------------+
| city_id | city_name | country_id |
+---------+-----------+------------+
| 1       |	西安市     | 1          |  <=== FETCH一次
+---------+-----------+------------+
| 2       |	北京市     | 1          |  <=== FETCH二次
+---------+-----------+------------+
```

##### 4.6.9.4 CLOSE光标

```sql
CLOSE cursor_name;
```

实例：

```sql
#初始化脚本
create table emp(
id int(11) not null auto_increment,
name varchar(50) not null comment '姓名',
age int(11) comment '年龄',
salary int(11) comment '薪水',
primary key (id)
)engine=innodb default charset=utf8;

insert into emp(id,name,age,salary) values(null,'金毛狮王',18,10000),(null,'白眉鹰王',19,8000),(null,'吉吉国王',8,28000);
```

```sql
select * from emp;
+----+--------------+------+--------+
| id | name         | age  | salary |
+----+--------------+------+--------+
|  1 | 金毛狮王     |   18 |  10000 |
|  2 | 白眉鹰王     |   19 |   8000 |
|  3 | 吉吉国王     |    8 |  28000 |
+----+--------------+------+--------+
3 rows in set (0.00 sec)
```

需求

```markdown
逐行展示这张emp表的数据
```

```sql
delimiter $

create procedure peo_test13()
begin
	declare e_id int(11);
	declare e_name varchar(50);
	declare e_age int(11);
	declare e_salary int(11);
	declare emp_result cursor for select * from emp;
	open emp_result;
	fetch emp_result into e_id,e_name,e_age,e_salary;
	select concat('id=',e_id,',name=',e_name,',age=',e_age,',salary=',e_salary);
	fetch emp_result into e_id,e_name,e_age,e_salary;
	select concat('id=',e_id,',name=',e_name,',age=',e_age,',salary=',e_salary);
	close emp_result;
end$

delimiter ;

#调用下看看
call peo_test13();
+----------------------------------------------------------------------+
| concat('id=',e_id,',name=',e_name,',age=',e_age,',salary=',e_salary) |
+----------------------------------------------------------------------+
| id=1,name=金毛狮王,age=18,salary=10000                               |
+----------------------------------------------------------------------+
1 row in set (0.00 sec)

+----------------------------------------------------------------------+
| concat('id=',e_id,',name=',e_name,',age=',e_age,',salary=',e_salary) |
+----------------------------------------------------------------------+
| id=2,name=白眉鹰王,age=19,salary=8000                                |
+----------------------------------------------------------------------+
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)


那我多FETCH几次怎么办呢？也就是说总共有3条数据，我FETCH了四次，会出现什么情况呢？
那第四条FETCH会报错：
ERROR 1329 (02000) : No data - zero rows fetched, selected,or processed
```

##### 4.6.9.5 FETCH报错解决

```sql
通过循环结构，来解决这个问题

思路一：先count(*) ---> num；num减到0就退出
思路二：FETCH不到数据的时候，就会触发一个事件，定义一个边界变量，通过判断这个边界变量来进行循环的退出
```

```sql
delimiter $

create procedure peo_test15()
begin
	declare e_id int(11);
	declare e_name varchar(50);
	declare e_age int(11);
	declare e_salary int(11);
	declare has_data defalut 1;
	declare emp_result cursor for select * from emp;
	DECLARE EXIT HANDLER FOR NOT FOUND SET has_data=0; #必须写在游标之后
	
	open emp_result;
	repeat
	fetch emp_result into e_id,e_name,e_age,e_salary;
	select concat('id=',e_id,',name=',e_name,',age=',e_age,',salary=',e_salary);
	until has_data=0
	end repeat;
	close emp_result;
end$

delimiter ;
```



### 4.7 存储函数

语法结构

```sql
create function function_name([param type...])
returns type
begin
	...
end;
```

案例

```sql
#定义一个存储过程，获取满足条件(city表中的查询条件)的总记录数
select * from city;
+---------+-----------+------------+
| city_id | city_name | country_id |
+---------+-----------+------------+
|       1 | 西安      |          1 |
|       2 | 纽约      |          2 |
|       3 | 北京      |          1 |
|       4 | 上海      |          1 |
+---------+-----------+------------+
4 rows in set (0.00 sec)


delimiter $

create function count_city(countryId int)
returns int
begin
declare cnum int;
select count(*) into cnum from city where country_id=countryId;
return cnum;
end$

delimiter ;

ERROR 1418 (HY000): This function has none of...
mysql的设置默认是不允许创建函数

1、更改全局配置
      SET GLOBAL log_bin_trust_function_creators = 1;  
      有主从复制的时候 , 从机必须要设置  不然会导致主从同步失败

2、更改配置文件my.cnf  
      log-bin-trust-function-creators=1   重启服务生效

#如何调用
select count_city(1);

#如何删除
drop function count_city;
```

## 5 触发器

### 5.1 介绍

触发器是与表有关的数据库对象，指在insert/update/delete之前或之后，触发并执行触发器中定义的SQL语句集合。触发器的这种特性可以协助应用在数据库端确保数据的完整性，日志记录，数据校验等操作。

使用别名OLD和NEW来引用触发器中发生变化的记录内容，这与其他的数据库是相似的。现在触发器还只支持行级触发，不支持语句级触发。

| 触发器类型     | NEW和OLD的使用                                       |
| -------------- | ---------------------------------------------------- |
| INSERT型触发器 | NEW表示将要或者已经新增的数据                        |
| UPDATE型触发器 | OLD表示修改之前的数据，NEW表示将要或已经修改后的数据 |
| DELETE型触发器 | OLD表示将要或者已经删除的数据                        |

### 5.2 创建触发器

语法结构

```sql
create trigger trigger_name

before/after insert/update/delete

on tbl_name

[ for each row ] -- mysql只有行级触发器，Oracle中既有行级触发器，又有语句级的触发器

begin
 trigger_statemrnt;
end;
```

实例

需求

```sql
通过触发器记录emp表的数据变更日志，包含增加、修改、删除；
```

首先创建一张日志表

```sql
create table emp_logs(
  id int(11) not null auto_increment,
  operation varchar(20) not null comment '操作类型,insert/update/deleete',
  operate_time datetime not null comment '操作时间',
  operate_id int(11) not null comment '操作的记录的ID',
  operate_params varchar(500) comment '操作参数',
  primary key(`id`)
)engine=innodb default charset=utf8;
```

创建insert型触发器，完成emp插入数据时的日志记录：

```sql
DELIMITER $

create trigger emp_insert_trigger
after insert
on emp
for each row
begin
insert into emp_logs(id,operation,operate_time,operate_id,operate_params) values(null,'insert',now(),new.id,concat('插入后(id:',new.id,',name:',new.name,',age:',new.age,',salary:',new.salary,')'));
end$

delimiter ;

#测试这个触发器
insert into emp(id,name,age,salary) values(null,'Canada King Of Pao',8,28000);

select * from emp_logs;
+----+-----------+---------------------+------------+------------------------------------------------------------+
| id | operation | operate_time        | operate_id | operate_params                                             |
+----+-----------+---------------------+------------+------------------------------------------------------------+
|  1 | insert    | 2021-07-03 08:51:29 |          4 | 插入后(id:4,name:Canada King Of Pao,age:8,salary:28000)    |
+----+-----------+---------------------+------------+------------------------------------------------------------+
1 row in set (0.00 sec)
```



创建update后的触发器，完成emp数据修改时的日志记录

```sql
delimiter $

create trigger emp_update_trigger
after update
on emp
for each row
begin
insert into emp_logs(id,operation,operate_time,operate_id,operate_params)
values(null,'update',now(),new.id,concat('修改前(id:',old.id,',name:',old.name,',age:',old.age,',salary:',old.salary,';修改后:id',new.id,',name:',new.name,',age:',new.age,',salary:',new.salary));
end$

delimiter ;

#测试一下
insert into emp(id,name,age,salary) values(null,'隔壁老王',35,58000);

select * from emp;
+----+--------------------+------+--------+
| id | name               | age  | salary |
+----+--------------------+------+--------+
|  1 | 金毛狮王           |   18 |  10000 |
|  2 | 白眉鹰王           |   19 |   8000 |
|  3 | 吉吉国王           |    8 |  28000 |
|  4 | Canada King Of Pao |    8 |  28000 |
|  5 | 隔壁老王           |   35 |  58000 |
+----+--------------------+------+--------+
5 rows in set (0.00 sec)


update emp set salary=5800 where id=5;

select * from emp_logs;
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
| id | operation | operate_time        | operate_id | operate_params                                                                                          |
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
|  1 | insert    | 2021-07-03 08:51:29 |          4 | 插入后(id:4,name:Canada King Of Pao,age:8,salary:28000)                                                 |
|  2 | insert    | 2021-07-03 09:21:01 |          5 | 插入后(id:5,name:隔壁老王,age:35,salary:58000)                                                          |
|  3 | update    | 2021-07-03 09:22:13 |          5 | 修改前(id:5,name:隔壁老王,age:35,salary:58000;修改后:id5,name:隔壁老王,age:35,salary:5800               |
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
3 rows in set (0.00 sec)
```

创建delete后触发器

```sql
delimiter $

create trigger emp_delete_trigger
after delete
on emp
for each row
begin
insert into emp_logs(id,operation,operate_time,operate_id,operate_params)
values(null,'delete',now(),old.id,concat('删除前(id:',old.id,',name:',old.name,',age:',old.age,',salary:',old.salary));
end$

delimiter ;

select * from emp;
+----+--------------------+------+--------+
| id | name               | age  | salary |
+----+--------------------+------+--------+
|  1 | 金毛狮王           |   18 |  10000 |
|  2 | 白眉鹰王           |   19 |   8000 |
|  3 | 吉吉国王           |    8 |  28000 |
|  4 | Canada King Of Pao |    8 |  28000 |
|  5 | 隔壁老王           |   35 |   5800 |
+----+--------------------+------+--------+
5 rows in set (0.01 sec)

delete from emp where id=4;

select * from emp_logs;
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
| id | operation | operate_time        | operate_id | operate_params                                                                                          |
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
|  1 | insert    | 2021-07-03 08:51:29 |          4 | 插入后(id:4,name:Canada King Of Pao,age:8,salary:28000)                                                 |
|  2 | insert    | 2021-07-03 09:21:01 |          5 | 插入后(id:5,name:隔壁老王,age:35,salary:58000)                                                          |
|  3 | update    | 2021-07-03 09:22:13 |          5 | 修改前(id:5,name:隔壁老王,age:35,salary:58000;修改后:id5,name:隔壁老王,age:35,salary:5800               |
|  4 | delete    | 2021-07-03 09:28:04 |          4 | 删除前(id:4,name:Canada King Of Pao,age:8,salary:28000                                                  |
+----+-----------+---------------------+------------+---------------------------------------------------------------------------------------------------------+
4 rows in set (0.00 sec)
```



### 5.3 删除触发器

语法

```sql
drop trigger [schema_name.]trigger_name
```

如果没有指定schema_name，默认为当前数据库。

### 5.4 查看触发器

可以通过执行SHOW TRIGGERS命令查看触发器的状态、语法等信息

语法

```sql
show triggers;
```











































