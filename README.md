# MySQL

## 1 Mysql基础小知识

mysql安装好了，如何进行登录呢？

```markdown
mysql -uroot -p；回车输入密码
```

### 1.1 MySQL密码修改

方法1：用Set Password命令

```markdown
首先登录MySQL
格式：mysql> set password for 用户名@localhost = password('新密码');
例子：mysql> set password for root@localhost=passowrd('123456')
```

方法2：用mysqladmin

```markdown
格式：mysqladmin -u用户名 -p旧密码 password 新密码
例子：mysqladmin -uroot -p123456 password 654321
```

方法3：用UPDATE直接编辑user表

```markdown
首先登陆MySQL
mysql> user mysql;
mysql> update user set password=passowrd('123456') where user='root' and host = 'localhost';
mysql> flush privileges;
```

方法4：在忘记root密码时候，可以这样

```markdown
windows:
1.打开正在运行的Mysql服务。
2.打开DOS窗口，转到mysql\bin目录。
3.输入mysqld --skip-grant-tables 回车。 --skip-grant-tables的意思是启动Mysql服务的时候跳过权限表的验证
4.再开一个DOS窗口(因为刚才那个窗口已经不能动了)，转到mysql\bin目录
5.输入mysql回车，如果成功将出现Mysql提示符 > 。
6.连接权限数据库： use mysql；。
7.修改密码： update user set password=password('123456') where ueser='root';（不要忘记加分号）
8.刷新权限(必须步骤)： flush privileges; 。
9.退出 quit。
10.注销系统，在进入使用用户名root和刚才设置的新密码进行登录
```

### 1.2 Mysql卸载

```markdown
1.双击安装包，点击下一步，然后点击remove，卸载
2.手动删除Program Files的Mysql目录
3.手动删除ProgramData目录(这个目录是隐藏的)中的MySQL
```

### 1.3 相关术语

sql、DB、DBMS分别是什么，他们之间的关系是什么？

```markdown
DB:DataBase(数据库，数据库实际上在硬盘上以文件的形式存在的)

DBMS：DataBase Management Sytstem(数据库管理系统，常见的有：Mysql Oracle DB3 Sybase SqlServer)

SQL：结构化的查询语言，是一门标准通用的语言，标准的sql使用于所有的数据库产品，SQL属于高级语言，在其执行的时候，实际上内部也会先进行编译，然后再执行sql(sql语句的编译由DBMS完成)，DBMS负责执行sql语句， 通过执行sql语句来操作DB当中的数据。
```

### 1.4 表

```markdown
table是数据库的基本组成单元，所有的数据都以表格的形式组织，目的是可读性强
一张表包括行和列
行：被称为数据(data)/记录(record)
列：被称为字段(column)
```

### 1.5 SQL概览

学习Mysql主要还是学习通用的SQL语句，那么SQL语句包括增删改查，SQL语句怎么分类呢？

```markdown
DQL(数据查询语言): 查询语句，凡是select语句都是DQL；
DML(数据操作语言): insert delete update，对表当中的数据进行增删改。
DDL(数据定义语言): create drop alter，对表结构的增删改
TCL(事务控制语言): commit提交视图，rollback回滚事务
DCL(数据控制语言): grant授权、revoke撤销权限等
```

## 2 MySQL

### 2.1 创建数据库

```sql
CRETE DATABASE [IF NOT EXISTS] db_name
[create_sepecification[,create_specification]...]

create_specification:
	[DEFAULT] CHARACTER SET charset name
	[DEFAULT] COLLATE collation_name
```



```markdown
1.CHARACTER SET ：指定数据库采用的字符集，如果不指定默认utf-8
2.COLLATE ： 指定数据库字符集的校对规则(常用的是utf8_bin[区分大小写]、uft8-general_ci[不区分到小写]注意默认是utf8_general_ci)[举例说明database.sql文件]
```

### 2.2 库的相关操作

#### 2.2.1 查看有哪些数据库

```sql
show databases; #查看有哪些数据库，这个属于MySql的命令
```

#### 2.2.2 创建自己数据库

```sql
#可以参考上面的
create databse mc; #创建mc的database，这个属于MySql的命令
```

#### 2.2.3 使用自己创建的数据库

```sql
use mc; #进入自己创建的出具库，这个不是SQL语句，是MySQL的命令
```

#### 2.2.4 查看当前使用的数据库有哪些表

```sql
show tables; #这个不是sql语句，属于MySql的命令
```

#### 2.2.5 导入数据的命令

```sql
source d:\myfile\resource\mv.sql
```

sql脚本中的数据量太大时，无法打开，请使用source命令完成初始化

#### 2.2.6 删除数据库

```sql
drop database mc;
```

#### 2.2.7 查看当前使用的数据库

```sql
select database();
```



### 2.3 表的相关操作

#### 2.3.1 查看表结构

```sql
desc 表名
```

```sql
mysql> desc DEPT;
+--------+-------------+------+-----+---------+-------+
| Field  | Type        | Null | Key | Default | Extra |
+--------+-------------+------+-----+---------+-------+
| DEPTNO | int         | NO   | PRI | NULL    |       |部门编号
| DNAME  | varchar(14) | YES  |     | NULL    |       |部门名称
| LOC    | varchar(13) | YES  |     | NULL    |       |部门位置
+--------+-------------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```

```sql
mysql> desc EMP;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| EMPNO    | int         | NO   | PRI | NULL    |       |员工编号
| ENAME    | varchar(10) | YES  |     | NULL    |       |员工姓名
| JOB      | varchar(9)  | YES  |     | NULL    |       |工作岗位
| MGR      | int         | YES  |     | NULL    |       |上级领导编号
| HIREDATE | date        | YES  |     | NULL    |       |入职时间
| SAL      | double(7,2) | YES  |     | NULL    |       |月薪
| COMM     | double(7,2) | YES  |     | NULL    |       |补助/津贴
| DEPTNO   | int         | YES  |     | NULL    |       |部门编号
+----------+-------------+------+-----+---------+-------+
8 rows in set (0.01 sec)
```

```sql
mysql> desc salgrade;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| GRADE | int  | YES  |     | NULL    |       |等级
| LOSAL | int  | YES  |     | NULL    |       |最低薪资
| HISAL | int  | YES  |     | NULL    |       |最高薪资
+-------+------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```

#### 2.3.2 查看表内容

```sql
select * from DEPT;
mysql> select * from DEPT;
+--------+------------+----------+
| DEPTNO | DNAME      | LOC      |
+--------+------------+----------+
|     10 | ACCOUNTING | NEW YORK |
|     20 | RESEARCH   | DALLAS   |
|     30 | SALES      | CHICAGO  |
|     40 | OPERATIONS | BOSTON   |
+--------+------------+----------+
4 rows in set (0.01 sec)
```

```sql
select * from EMP;
mysql> select * from EMP;
+-------+--------+-----------+------+------------+---------+---------+--------+
| EMPNO | ENAME  | JOB       | MGR  | HIREDATE   | SAL     | COMM    | DEPTNO |
+-------+--------+-----------+------+------------+---------+---------+--------+
|  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800.00 |    NULL |     20 |
|  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600.00 |  300.00 |     30 |
|  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250.00 |  500.00 |     30 |
|  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975.00 |    NULL |     20 |
|  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250.00 | 1400.00 |     30 |
|  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850.00 |    NULL |     30 |
|  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450.00 |    NULL |     10 |
|  7788 | SCOTT  | ANALYST   | 7566 | 1987-04-19 | 3000.00 |    NULL |     20 |
|  7839 | KING   | PRESIDENT | NULL | 1981-11-17 | 5000.00 |    NULL |     10 |
|  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500.00 |    0.00 |     30 |
|  7876 | ADAMS  | CLERK     | 7788 | 1987-05-23 | 1100.00 |    NULL |     20 |
|  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950.00 |    NULL |     30 |
|  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000.00 |    NULL |     20 |
|  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300.00 |    NULL |     10 |
+-------+--------+-----------+------+------------+---------+---------+--------+
14 rows in set (0.00 sec)
```

```sql
select * from salgrade;
mysql> select * from salgrade;
+-------+-------+-------+
| GRADE | LOSAL | HISAL |
+-------+-------+-------+
|     1 |   700 |  1200 |
|     2 |  1201 |  1400 |
|     3 |  1401 |  2000 |
|     4 |  2001 |  3000 |
|     5 |  3001 |  9999 |
+-------+-------+-------+
5 rows in set (0.00 sec)
```

#### 2.3.3 查看指定库中的表

```sql
show tables from mysql;
```

#### 2.3.4 查看表的创建语句

```sql
show create table EMP（表名）;
```

#### 2.3.5 简单表查询

```sql
select FiledName1,FiledName2... from TABLE_NAME;
```

#### 2.3.6 给列赋别名

```sql
select ename,sal*12 as '年薪' from emp;
```

#### 2.3.7 简单查询建议

不建议使用select * 效率低下

#### 2.3.8 查询执行顺序

先from，然后where，最后select

#### 2.3.9 不等于查询

```sql
select ename,sal from emp where sal <> 3000;
```

#### 2.3.10 在什么区间

```sql
select ename,sal from emp where sal>=1000 and sal<= 3000;

select ename,sql from emp where sal between 1000 and 3000; #between...and...是闭区间[1000~3000]
```

between...and...还可以用在字符上面

```sql
select ename from emp where ename between 'A' and 'C'; #between ... and ...字符左闭右开
```

#### 2.3.11 Null

在数据库中，NULL不是一个值，代表什么也没有，为空，所以不能用等号衡量，必须使用 is null或者is not null

```sql
select ename,sal,comm from emp where comm is not null;

select ename,sal,comm from emp where comm is null or comm = 0;
```



#### 2.3.12 and和or的优先级

and 和 or同时出现时，and优先级较高

#### 2.3.13 in

```sql
select ename,sal from emp where sal in (1100,3000);

select ename,sal from emp where sal not in (1100,3000);
```

#### 2.3.14 模糊查询

```sql
select ename from emp where ename like '%O%';

#第二个字符是A的
select ename from emp where ename like '%_A%';

#找出名字中有下划线的
select ename from emp where ename like '%\_%';
```

#### 2.3.15 order by

默认是升序的

```sql
select ename,sal from emp order by sal desc; #降序排序
select ename,sal from emp order by sal asc;  #升序排列
```

按照工资的降序排列，当工资相同的时候再按照名字的升序排列

```sql
select ename,sal from emp order by sal desc,ename asc; #先按照薪资排，再按照名字排
#注意：越靠前的字段越能起到主导作用，只有当前面的字段无法完成排序时，才会启用后面的字段
```

Order by 后面跟着数字，表示按照第几列进行排序

```sql
select ename,sal from emp order by 2; #就是按照sal进行排序的，默认升序

select * from emp order by 6;#其实还是按照薪资来，但不建议使用数字，建议列名写死
```

执行的顺序

```sql
select * 		 				第3步
from tablename 			第1步
where ... 					第2步
order by ...				第4步 这个order by 是最后执行的
```

#### 2.3.16 分组函数

count 计数、sum 求和、avg 平均值、max 最大值、min 最小值

```sql
#工资总和
select sum(sal) from emp;
#最高工资
select max(sal) from emp;
#最低工资
select min(sal) from emp;
#平均工资
select avg(sql) from emp;
#总人数
select count(*) from emp;
```

注意：所有的分组函数都是对"某一组"数据进行操作的。

分组函数一共五个，分组函数还有另一个名字：多行处理函数。特点就是输入多行，最终输出的结果是1行

##### 2.3.16.1 分组函数自动忽略NULL

```sql
select count(comm) from emp; #这会导致少统计了，因为他把为null的都忽略了
```

##### 2.3.16.2 单行处理函数

什么是单行处理函数，可以简单理解为输入一行，输出一行

```sql
#计算年薪
select ename,(sal+comm)*12 as yearsal from emp;

#计算每个员工的年薪
select ename,(800+NULL)*12 as yearsal from emp; #这样做是不对的
***重点:所有的数据库都是这样规定的，只要有Null参与的运算，结果一定是null***

ifnull()空处理函数,属于单行处理函数
select ename,ifnull(comm,0) from emp;

#正确的计算每个员工的年薪
select ename,(sal+ifnull(comm,0))*12 from emp;
```

##### 2.3.16.3 sum注意点

```sql
select sum(com) from emp;

mysql> select sum(comm) from emp;
+-----------+
| sum(comm) |
+-----------+
|   2200.00 |
+-----------+
1 row in set (0.00 sec)
执行的结果不为null，上面2.2.16.2中我们说过，只要有null参与的运算，结果一定是null，这个结果不为null，说明sum函数会自动忽略null

也就是说，不需要这样写：
select sum(comm) from emp where comm is not null;
```

##### 2.3.16.4 avg注意点

```sql
#查询出工资比平均薪资大的
select ename,sql from emp where sal>avg(sal); #很遗憾这行会报错的 Invalid use of group function

#这时因为，sql语句中有一个语法规则，分组函数不可直接出现在where子句中的，why???
```

##### 2.3.16.5 count(*)和count(FiledName)区别

```sql
select count(FieldName) from emp; #count(FieldName)不为空的，count(*)就是所有
```

##### 2.3.16.6 分组函数的组合使用

```sql
select count(*),sum(sal),avg(sal),max(sal),min(sal) from emp; 
```



#### 2.3.17 group by 和 having

group by  :  按照某个字段或者某些字段进行分组。

having ： 是对分组之后的数据进行再次过滤

```sql
#找出每个工作岗位的最高薪资
select max(sal) from emp group by job;

#注意：分组函数一般都会配合group by联合使用，这也是他们为什么被称为分组函数的原因，并且任何一个分组函数(count sum avg max min)都是在group by语句执行结束之后才会执行的。
#当一条sql语句没有group by的话，整张表的数据会自成一组。

#现在可以回答了：
#这时因为，sql语句中有一个语法规则，分组函数不可直接出现在where子句中的，why???
****因为，group by 是在 where 执行之后，才会执行的，分组函数，只能在分完组之后才能使用*****
```

执行顺序回顾

```sql
select ...		5
from ...			1
where ...			2
group by ...	3
having ...		4
order by ...	6
```

```sql
select avg(sal) from emp [没有的话，会默认有个 group by ，这个by的东西就是这张表]
```

分组函数联合这个group by使用，只有group by之后，才去执行这个分组函数，恰好，这个group by执行的顺序是在where后面执行，你直接where sal > avg(sal)这样写，那还没进行group by，是没法使用avg(sal)的分组函数的，所以where后面不能接分组函数的！

#### 2.3.18 找出高于平均工资

```sql
select ename,sal from emp where sal > (select avg(sal) from emp);
```

根据工作岗位找出每个岗位中的最高薪资

```sql
select max(sal),job from emp group by job;
```

再加一个ename的字段查询呢？

```sql
select ename,max(sal),job from emp group by job; # this is incompatible with sql_mode=only_full_group_by

注意：当一条语句中又group by的话，select 后面只能跟分组函数和参与分组的字段
```

每个工作岗位的平均薪资

```sql
select job,avg(sal) from emp group by job;
```

多个字段能不能联合起来一起分组呢？

```sql
#找出每个部门不同工作岗位的最高薪资？
select job,max(sal) from emp group by job; #按工作岗位进行最高工资的查找

select deptno,job,max(sal) from emp group by deptno,job;
mysql> select deptno,job,max(sal) from emp group by deptno,job;
+--------+-----------+----------+
| deptno | job       | max(sal) |
+--------+-----------+----------+
|     20 | CLERK     |  1100.00 |
|     30 | SALESMAN  |  1600.00 |
|     20 | MANAGER   |  2975.00 |
|     30 | MANAGER   |  2850.00 |
|     10 | MANAGER   |  2450.00 |
|     20 | ANALYST   |  3000.00 |
|     10 | PRESIDENT |  5000.00 |
|     30 | CLERK     |   950.00 |
|     10 | CLERK     |  1300.00 |
+--------+-----------+----------+
9 rows in set (0.01 sec)
```

#### 2.3.19 having

找出每个部门的最高薪资，要求显示薪资大于2900的数据

```sql
#找出每个部门最高的薪资
select deptno,max(sal) from emp group by deptno;
mysql> select deptno,max(sal) from emp group by deptno;
+--------+----------+
| deptno | max(sal) |
+--------+----------+
|     20 |  3000.00 |
|     30 |  2850.00 |
|     10 |  5000.00 |
+--------+----------+
3 rows in set (0.00 sec)
#再找出薪资大于2900的
select deptno,max(sal) from emp group by deptno having max(sal)>2900;

这种效率比较低，都查出来最大的值了，然后再不要，直接用where给排除掉，能用where尽量用where
select deptno,max(sal) from emp where sal>2900 group by deptno;


***还有where 搞不定的情况***
例如：找出每个部门的平均薪资，要求显示薪资大于2000的数据；
select deptno,avg(sal) from emp group by deptno;
where是没法用的，因为他不能在group by之前使用分组函数的
解决方案：
select deptno,avg(sal) from emp group by deptno having avg(sal)>2000;
mysql> select deptno,avg(sal) from emp group by deptno having  avg(sal)>2000;
+--------+-------------+
| deptno | avg(sal)    |
+--------+-------------+
|     20 | 2175.000000 |
|     10 | 2916.666667 |
+--------+-------------+
2 rows in set (0.00 sec)
```

再回顾下执行语句

```sql
select ...		5
from ...			1
where ...			2
group by ...	3
having ...		4(为了分组过后的数据而存在IDE-不可单独出现)
order by ...	6
```

#### 2.3.20 去重查询

查询出岗位

```sql
select job from emp;

select distinct(job) from emp;
```

错误示例

```sql
select ename,distinct(job) from emp;
注意：distinct只能出现在所有字段的最前面
```

distinct出现在最前面

```sql
select deptno,job from emp order by deptno;
mysql> select deptno,job from emp order by deptno;
+--------+-----------+
| deptno | job       |
+--------+-----------+
|     10 | MANAGER   |
|     10 | PRESIDENT |
|     10 | CLERK     |
|     20 | CLERK     |
|     20 | MANAGER   |
|     20 | ANALYST   |
|     20 | CLERK     |
|     20 | ANALYST   |
|     30 | SALESMAN  |
|     30 | SALESMAN  |
|     30 | SALESMAN  |
|     30 | MANAGER   |
|     30 | SALESMAN  |
|     30 | CLERK     |
+--------+-----------+
14 rows in set (0.00 sec)

select distinct deptno,job from emp;
mysql> select distinct deptno,job from emp;
+--------+-----------+
| deptno | job       |
+--------+-----------+
|     20 | CLERK     |
|     30 | SALESMAN  |
|     20 | MANAGER   |
|     30 | MANAGER   |
|     10 | MANAGER   |
|     20 | ANALYST   |
|     10 | PRESIDENT |
|     30 | CLERK     |
|     10 | CLERK     |
+--------+-----------+
9 rows in set (0.01 sec)
distinct出现在最前面，表示后面的所有字段联合去重
```

统计岗位的数量

```sql
select count(distinct(job)) from emp;
mysql> select count(distinct(job)) from emp;
+----------------------+
| count(distinct(job)) |
+----------------------+
|                    5 |
+----------------------+
1 row in set (0.01 sec)
```

### 2.4 连接查询

在实际开发中，大部分的情况下都不是从单表中查询出数据的，一般都是从多张表联合查询出最终的结果的。一般一个业务都会对应多张表，比如：学生和班级，起码两张表。

在表的连接查询方面有一种现象被称为：笛卡尔积现象。(笛卡尔乘积现象)

案例：找出每一个员工的部门名称，要求显示员工名和部门名。

```sql
select * from emp;
mysql> select * from emp;
+-------+--------+-----------+------+------------+---------+---------+--------+
| EMPNO | ENAME  | JOB       | MGR  | HIREDATE   | SAL     | COMM    | DEPTNO |
+-------+--------+-----------+------+------------+---------+---------+--------+
|  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800.00 |    NULL |     20 |
|  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600.00 |  300.00 |     30 |
|  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250.00 |  500.00 |     30 |
|  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975.00 |    NULL |     20 |
|  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250.00 | 1400.00 |     30 |
|  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850.00 |    NULL |     30 |
|  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450.00 |    NULL |     10 |
|  7788 | SCOTT  | ANALYST   | 7566 | 1987-04-19 | 3000.00 |    NULL |     20 |
|  7839 | KING   | PRESIDENT | NULL | 1981-11-17 | 5000.00 |    NULL |     10 |
|  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500.00 |    0.00 |     30 |
|  7876 | ADAMS  | CLERK     | 7788 | 1987-05-23 | 1100.00 |    NULL |     20 |
|  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950.00 |    NULL |     30 |
|  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000.00 |    NULL |     20 |
|  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300.00 |    NULL |     10 |
+-------+--------+-----------+------+------------+---------+---------+--------+
14 rows in set (0.00 sec)


select * from dept;
mysql> select * from dept;
+--------+------------+----------+
| DEPTNO | DNAME      | LOC      |
+--------+------------+----------+
|     10 | ACCOUNTING | NEW YORK |
|     20 | RESEARCH   | DALLAS   |
|     30 | SALES      | CHICAGO  |
|     40 | OPERATIONS | BOSTON   |
+--------+------------+----------+
4 rows in set (0.00 sec)

select ename,dname from emp,dept; 查出来了14*4=56条了。
```







#### 2.4.1 连接查询的分类

根据年代来划分的话

SQL(92)

SQL(99)

根据表的连接方式来划分，包括

#### 2.4.2 内连接

##### 2.4.2.1 等值连接



##### 2.4.2.2 非等值连接



##### 2.4.2.3 自连接



#### 2.4.3 外连接

##### 2.4.3.1 左外连接(左连接)



##### 2.4.3.2 右外连接(右连接)



#### 2.4.4 全连接



## 10 常用命令

### 10.1 查看数据库版本

```sql
select version();
```

### 10.2 退出Mysql

```sql
quit 或者 exit
```

#### 10.3 退出一条语句

```sql
#写着写着不想写了
\c
```

## 11 建议

1. 字符串使用单引号括起来