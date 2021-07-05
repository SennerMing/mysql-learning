## MySQL基础

## 1.数据类型

### 1.1 整数类型

| 类型      | 占用空间 | 最小值               | 最大值               |
| --------- | -------- | -------------------- | -------------------- |
|           | (字节)   | (Signed/Unsigned)    | (Signed/Unsigned)    |
| TINYINT   | 1        | -128                 | 127                  |
|           |          | 0                    | 255                  |
| SAMLLINT  | 2        | -32768               | 32767                |
|           |          | 0                    | 65535                |
| MEDIUMINT | 3        | -8388608             | 8388607              |
|           |          | 0                    | 16777215             |
| INT       | 4        | -2147483648          | 2147483647           |
|           |          | 0                    | 4294967295           |
| BIGINT    | 8        | -9223372036854775808 | 9223372036854775807  |
|           |          | 0                    | 18446744073709551615 |

#### 1.1.1 如何创建无符号的整型数据呢？

```sql
#切换数据库
use test;

#创建无符号的int类型
create table z (a int unsigned);
drop table z;

#创建有符号的数据类型字段
create table z (a int unsigned,b tinyint signed);
insert into z(1,-1);
```

#### 1.1.2 整型重要属性

```markdown
1.UNSIGNED/SIGNED
	是否有符号

2.ZEROFILL
	显示属性
	值不做任何修改

3.AUTO_INCREMENT
	自增
	每张表一个
	必须是索引的一部分
```

##### 1.1.2.1 zerofill

```sql
#ZEROFILL解释
show create table z\G
*************************** 1. row ***************************
       Table: z
Create Table: CREATE TABLE `z` (
  `a` int(10) unsigned DEFAULT NULL,
  `b` tinyint(4) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)

#当我们调用查看建表语句的时候啊，我们会看到int(10)还有tinyint(4)，你看tinyint是一个字节的长度，int是4个字节长度，很显然这个10和4不是表示字节数的。

#怎么让这个zerofill出现呢？执行下面的语句
alter table z change column a a int unsigned zerofill;

show create table z\G
*************************** 1. row ***************************
       Table: z
Create Table: CREATE TABLE `z` (
  `a` int(10) unsigned zerofill DEFAULT NULL,
  `b` tinyint DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
#这个时候zerofill就出现了

#查看数据，那a里面的值就变成了0000000001
select * from z;
+------------+------+
| a          | b    |
+------------+------+
| 0000000001 |   -1 |
+------------+------+
1 row in set (0.00 sec)

#那问一下，我们修改a字段让a int(10) ---> a int(4)，那么我们再插入一个values(10000,10)会发生什么事情呢？
alter table z change column a a int(4) unsigned zerofill;
show create table z\G
*************************** 1. row ***************************
       Table: z
Create Table: CREATE TABLE `z` (
  `a` int(4) unsigned zerofill DEFAULT NULL,
  `b` tinyint DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)

insert into z values(10000,0);
Query OK, 1 row affected (0.00 sec)

mysql> select * from z;
+-------+------+
| a     | b    |
+-------+------+
|  0001 |   -1 |
| 10000 |    0 |
+-------+------+
2 rows in set (0.00 sec)

所以int(4)我们插入10000也是可以的
```

**1.zerofill这个值就是帮我们填充0的，而int(10)、tinyint(4)括号里面的数字，是限制填充0的个数的，对插入的值的长度没有影响！且zero不能加在有符号的整数字段上面！**

**2.是int(10)、tinyint(4)也好，括号里面的数字是显示用，并不会对我们插入整数的长度进行限制！如果整型字段为无符号(unsigned)则可以加上zerofill属性；如果为有符号(signed)则不可以加上zerofill属性！**

##### 1.1.2.2 auto_increment

```sql
truncate table z;

alter table z change column a a int auto_increment primary key;

show create table z \G;
*************************** 1. row ***************************
       Table: z
Create Table: CREATE TABLE `z` (
  `a` int NOT NULL AUTO_INCREMENT,
  `b` tinyint DEFAULT NULL,
  PRIMARY KEY (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)

#即使我们输入的值为null，也照样插的进去，而且是自增长的！
insert into z values(NULL,10);
insert into z values(NULL,20);

mysql> select * from z;
+---+------+
| a | b    |
+---+------+
| 1 |   10 |
| 2 |   20 |
+---+------+
2 rows in set (0.00 sec)

#当前会话下，上一次自增的值是多少，通常应用程序用
select last_insert_id(); 
+------------------+
| last_insert_id() |
+------------------+
|                2 |
+------------------+
1 row in set (0.01 sec)

#删除一条数据再插入一条数据，看看自增主键会不会回溯
insert into z values(NULL,30);
delete from z where a=3;
insert into z values(NULL,40);
select * from z;
+---+------+
| a | b    |
+---+------+
| 1 |   10 |
| 2 |   20 |
| 4 |   40 |
+---+------+
3 rows in set (0.00 sec)
```

**注意，如果使用整型做自增主键的话，尽量使用bigint而不用int，因为int可表示的数据太小了，有符号21个亿，无符号42个亿，数据量不大的话，可以使用int**

**AUTO_INCREMENT：自增主键不会回溯，即使插入为NULL，也会进行自增长**

###### 1.1.2.2.1**自增值回溯问题**

```sql
#什么时候自增值会回溯呢？
delete from z where a=4;
#这个时候我们进行数据库的重启操作
exit
#重启数据库
/etc/init.d/mysql.server restart
#再插入一条数据
insert into z values(NULL,50);
#查询数据
select * from z;
+---+------+
| a | b    |
+---+------+
| 1 |   10 |
| 2 |   20 |
| 4 |   50 |
+---+------+
3 rows in set (0.00 sec)

#那么为4的自增主键会被再次启用
```

**引发的问题**

```markdown
AOTU_INCREMENT的原理就是在mysql server重新启动后，进行自增的id就会被重置成最大数量+1
select max(auto_increment column)+1 from z;

问题就是，我命名把id为最大值的数据删掉了，然后数据库重启了之后，(别人)插入一条数据，而这条原本删除的，数据又出现了。
Mysql8.0解决了这个问题，自增的字段被定义为persist，也是就是说auto_increment字段原本是几(比如是4，删掉4后)，再次重启数据库之后，自增字段就是几(再新增就是5)
```

#### 1.1.3 总结

- 推荐不要使用UNSIGNED
- 范围本质上没有大的改变
- UNSIGNED可能会有溢出现象发生
- 自增INT类型主键建议使用BIGINT

## 2 数字类型

单精度类型：FLOAT

双精度类型：DOUBLE

高精度类型：DECIMAL

**FLOCT和DOUBLE，M*G/G不一定等于M的**；**财务、账务系统必须使用decimal类型**

| 类型    | 占用空间 | 精度   | 精确性        |
| ------- | -------- | ------ | ------------- |
| FLOAT   | 4        | 单精度 | 低            |
| DOUBLE  | 8        | 双精度 | 低、比FLOAT高 |
| DECIMAL | 变长     | 高精度 | 非常高        |

FLOAT(M,D)/DOUBLE(M,D)/DECIMAL(M,D)表示显示M位整数，其中D位位于小数点后面。

```sql
#操作double类型的数据，sum啊什么的，会出问题的！
select sum(double_type_price) from account;
```

### 2.1 函数

#### 2.1.1 floor向下取整 函数

```sql
select floor(-1.9);
+-------------+
| floor(-1.9) |
+-------------+
|          -2 |
+-------------+
1 row in set (0.00 sec)

select floor(1.9);
+------------+
| floor(1.9) |
+-------------+
|          1 |
+------------+
```

#### 2.1.2 round四舍五入 函数

```sql
mysql> select round(1.4),round(1.5);
+------------+------------+
| round(1.4) | round(1.5) |
+------------+------------+
|          1 |          2 |
+------------+------------+
1 row in set (0.00 sec)
```

#### 2.1.3 rand随机数 函数

```sql
#rand()
select rand();
+--------------------+
| rand()             |
+--------------------+
| 0.3879876838885133 |
+--------------------+
1 row in set (0.00 sec)
```

应用场景

```sql
一、范围内的数字
#取一个范围内的数
select floor(i+rand()*(j-i));
#0-100之内的一个数
select floor(1+rand()*99);

二、为可变长的字段填充假数据，比如varchar(128)
select repeat('a',3);

select repeat('a',floor(1+rand()*127));
那么插入的a就是可变长度的了
```

## 3 字符串类型

|              | 说明           | N的含义 | 是否有字符集 | 最大长度 |
| ------------ | -------------- | ------- | ------------ | -------- |
| CHAR(N)      | 定长字符       | 字符    | 是           | 255      |
| VARCHAR(N)   | 变长字符       | 字符    | 是           | 16384    |
| BINARY(N)    | 变长二进制字节 | 字节    | 否           | 255      |
| VARBINARY(N) | 变长二进制字节 | 字节    | 否           | 16384    |
| TINYBLOB     | 二进制大对象   | 字节    | 否           | 256      |
| BLOB         | 二进制大对象   | 字节    | 否           | 16K      |
| MEDIUMBLOB   | 二进制大对象   | 字节    | 否           | 16M      |
| LONGBLOB     | 二进制大对象   | 字节    | 否           | 4G       |
| TINYTEXT     | 大对象         | 字节    | 是           | 256      |
| TEXT         | 大对象         | 字节    | 是           | 16K      |
| MEDIUMTEXT   | 大对象         | 字节    | 是           | 16M      |
| LONGTEXT     | 大对象         | 字节    | 是           | 4G       |

BLOB ==> VARBINARY

TEXT ===> VARCHAR

### 3.1 charset 字符集

查看当前数据库字符集

```sql
show variables like '%character%';
```



#### 3.1.1 修改表的字符集

```sql
alter table z convert to character set utf8mb4;
```

**character set**

a set of symbols and encodings

**常见的字符集:utf8、uft8mb4、gbk、gb18030**

```sql
#查看字符集
show character set;

show variables like '%character%';
```

生产中只推荐uft8mb4，像是手机的表情符号utf8就存不下

#### 3.1.2 修改库字符集

```shell
#打开mysql.cnf
vim /etc/my.cnf
#在[mysqld]后面添加
character_set_server=utf8mb4
#然后重启数据库
/etc/init.d/mysql.server restart
```

注意

```sql
alter table z charset=utf8mb4;
show create table z\G
#虽然这张表的字符集改为了utf8mb4，但是之前创建的列还是原来的字符集
#再新增列，才是utf8mb4的字符集
alter table z add column b varchar(10)

```

#### 3.1.3 HEX-字符串转16进制

```sql
select hex('我');
0xE68891;

select 0xE68891;
我

可以用insert语句直接将十六进制的数据直接插入字符串类型的数据表中的
```

#### 3.1.4 CAST-转字符串

```sql
select cast(123 as char(10));

#换字符集转换
select hex(cast('a' as char(1) charset gbk));
```

### 3.2 collation 排序规则

```sql
collation
 - set of rules for comparing characters in a character set;
```

```sql
select 'a'='a';
select 'a'='A'; 

select 'a'='a ';
+----------+
| 'a'='a ' |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec)
select 'a'='A  ';
+----------+
| 'a'='A ' |
+----------+
|        0 |
+----------+
```

查看所有字符集及其对应的排序规则

```sql
show charset;
show character set;
```

### 3.3 md5 hash 函数

```sql
create table y(password varchar(35));

insert int y values(md5('bb'));
```

