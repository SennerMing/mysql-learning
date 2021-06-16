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

连接查询建议，给表取别名

```markdown
第一：执行效率高一些，字段前面加表的别名，不用再所有的表都查找一遍
第二：可读性好一些
```

怎么避免笛卡尔积现象？避免了笛卡尔积现象，会减少记录的匹配次数吗？

```markdown
加过滤条件，不会减少匹配次数，知不是过显示的是有效的记录
```

所以找出每一个员工的部门名称，要求显示员工名和部门名。

```sql
select e.ename,d.dname from emp e,dept d where e.deptno = d.deptno; //SQL92,以后不要用了
```



#### 2.4.1 连接查询的分类

根据年代来划分的话

SQL(92)

SQL(99)

根据表的连接方式来划分，包括

#### 2.4.2 内连接

##### 2.4.2.1 等值连接

案例：查询每个员工的部门名称，要求显示员工名和部门名

```sql
select e.ename,d.dname 
from emp e,dept d 
where e.deptno = d.deptno; #SQL 92不能用

select e.ename,d.dname 
from emp e 
[inner可以省略的] join dept d 
on e.deptno = d.deptno; #SQL 99 结构清晰
SQL99语法结构更清晰一些：表的连接条件和后来的where条件分离了
```

##### 2.4.2.2 非等值连接

案例找出每个员工的工资等级，要求显示员工名、工资、工资等级。

```sql
select ename,sal from emp;
mysql> select ename,sal from emp;
+--------+---------+
| ename  | sal     |
+--------+---------+
| SMITH  |  800.00 |
| ALLEN  | 1600.00 |
| WARD   | 1250.00 |
| JONES  | 2975.00 |
| MARTIN | 1250.00 |
| BLAKE  | 2850.00 |
| CLARK  | 2450.00 |
| SCOTT  | 3000.00 |
| KING   | 5000.00 |
| TURNER | 1500.00 |
| ADAMS  | 1100.00 |
| JAMES  |  950.00 |
| FORD   | 3000.00 |
| MILLER | 1300.00 |
+--------+---------+
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



```sql
select 
e.ename,e.sal,s.grade
from emp e
join salgrade s
on e.sal between s.losal and s.hisal;
+--------+---------+-------+
| ename  | sal     | grade |
+--------+---------+-------+
| SMITH  |  800.00 |     1 |
| ALLEN  | 1600.00 |     3 |
| WARD   | 1250.00 |     2 |
| JONES  | 2975.00 |     4 |
| MARTIN | 1250.00 |     2 |
| BLAKE  | 2850.00 |     4 |
| CLARK  | 2450.00 |     4 |
| SCOTT  | 3000.00 |     4 |
| KING   | 5000.00 |     5 |
| TURNER | 1500.00 |     3 |
| ADAMS  | 1100.00 |     1 |
| JAMES  |  950.00 |     1 |
| FORD   | 3000.00 |     4 |
| MILLER | 1300.00 |     2 |
+--------+---------+-------+
14 rows in set (0.00 sec)
```



##### 2.4.2.3 自连接

自连接，最大的特点是，一张表看做两张表

案例：找出每个员工的上级领导，要求显示员工名和对应的领导名

```sql
select empno,ename,mgr from emp;
mysql> select empno,ename,mgr from emp;
+-------+--------+------+
| empno | ename  | mgr  |
+-------+--------+------+
|  7369 | SMITH  | 7902 |
|  7499 | ALLEN  | 7698 |
|  7521 | WARD   | 7698 |
|  7566 | JONES  | 7839 |
|  7654 | MARTIN | 7698 |
|  7698 | BLAKE  | 7839 |
|  7782 | CLARK  | 7839 |
|  7788 | SCOTT  | 7566 |
|  7839 | KING   | NULL |
|  7844 | TURNER | 7698 |
|  7876 | ADAMS  | 7788 |
|  7900 | JAMES  | 7698 |
|  7902 | FORD   | 7566 |
|  7934 | MILLER | 7782 |
+-------+--------+------+
14 rows in set (0.00 sec)

mgr表示的是该员工的领导编号

上面的查询结果是一张员工表，我们将其定义为员工表↑

对应这上面的表我们手动筛选出被当做领导的员工，我们将其定义为领导表↓
+-------+--------+------+
| empno | ename  | mgr  |
+-------+--------+------+
|  7566 | JONES  | 7839 |
|  7698 | BLAKE  | 7839 |
|  7782 | CLARK  | 7839 |
|  7788 | SCOTT  | 7566 |
|  7839 | KING   | NULL |
|  7902 | FORD   | 7566 |
+-------+--------+------+

那么员工的领导的员工编号 = 领导的员工编号：那么就是 员工表的mgr = 领导表empno
select empl.ename,boss.ename
from emp empl
inner join emp boss
on empl.mgr = boss.empno;

+--------+-------+
| ename  | ename |
+--------+-------+
| SMITH  | FORD  |
| ALLEN  | BLAKE |
| WARD   | BLAKE |
| JONES  | KING  |
| MARTIN | BLAKE |
| BLAKE  | KING  |
| CLARK  | KING  |
| SCOTT  | JONES |
| TURNER | BLAKE |
| ADAMS  | SCOTT |
| JAMES  | BLAKE |
| FORD   | JONES |
| MILLER | CLARK |
+--------+-------+
13 rows in set (0.00 sec)

KING 是大哥大，上面没有领导了，这个还是不完美的，并没有找到所有的员工，因为KING没有了
```



#### 2.4.3 外连接

什么是外连接，和内连接有什么区别？

内连接：假设A和B表进行连接查询，使用内连接的话，凡是A表和B表能够匹配上的记录都查询出来。A和B两张表没有主副之分，两张表是平等的



外连接：假设A和B表进行连接，使用外连接的话，AB两张表中有一个是主表，有一张表示副表，主要查询主表中的数据，捎带着查询副表，当副表中的数据没有和主表中的数据匹配的上，副表自动模拟出NULL值与之匹配。



外连接最终要的特点就是：主表的数据无条件的全部查询出来

##### 2.4.3.1 左外连接(左连接)

左外连接(左连接)：表示左边这张表是主表。

案例：找出每个员工的上级领导(每个员工必须全部显示)

```sql
#这个是内连接的，改改吧
select empl.ename,boss.ename
from emp empl
inner join emp boss
on empl.mgr = boss.empno;

#改成外连接，员工表必须是主表喔
select empl.ename,boss.ename
from emp empl
left [outer] join emp boss		#这个outer是可以省略的
on empl.mgr = boss.empno;

+--------+-------+
| ename  | ename |
+--------+-------+
| SMITH  | FORD  |
| ALLEN  | BLAKE |
| WARD   | BLAKE |
| JONES  | KING  |
| MARTIN | BLAKE |
| BLAKE  | KING  |
| CLARK  | KING  |
| SCOTT  | JONES |
| KING   | NULL  |
| TURNER | BLAKE |
| ADAMS  | SCOTT |
| JAMES  | BLAKE |
| FORD   | JONES |
| MILLER | CLARK |
+--------+-------+
14 rows in set (0.00 sec) #这就查全乎了，14条记录
```



##### 2.4.3.2 右外连接(右连接)

右外连接(右连接)：表示右边的这张表是主表。

案例：找出哪个部门没有员工？

```sql
所有员工
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
```

```sql
所有部门
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
```

分析，部门表应该是主表

```sql
select dept.dname,emp.ename
from dept
left join emp
on dept.deptno = emp.deptno;

+------------+--------+
| dname      | ename  |
+------------+--------+
| ACCOUNTING | MILLER |
| ACCOUNTING | KING   |
| ACCOUNTING | CLARK  |
| RESEARCH   | FORD   |
| RESEARCH   | ADAMS  |
| RESEARCH   | SCOTT  |
| RESEARCH   | JONES  |
| RESEARCH   | SMITH  |
| SALES      | JAMES  |
| SALES      | TURNER |
| SALES      | BLAKE  |
| SALES      | MARTIN |
| SALES      | WARD   |
| SALES      | ALLEN  |
| OPERATIONS | NULL   |
+------------+--------+
15 rows in set (0.00 sec)

没有员工所以还要将ename不为NULL的都去掉
select dept.dname
from dept
left join emp
on dept.deptno = emp.deptno
where emp.ename is null;

+------------+
| dname      |
+------------+
| OPERATIONS |
+------------+
1 row in set (0.00 sec)
```



#### 2.4.4 全连接

三张表怎么连接查询呢？兄弟

案例：找出每一个员工的部门名称以及工资等级

```sql
#找出所有的员工
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
```

```sql
#找出所有的部门
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
```

```sql
#找出所有的工资等级
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

分析分析

```sql
...		A
join	B
join 	C
on ...
A先和B连接再和C连接
```

```sql
select emp.ename,dept.dname,salgrade.grade
from emp
join dept
on emp.deptno = dept.deptno
join salgrade
on emp.sal between salgrade.losal and salgrade.hisal;

+--------+------------+-------+
| ename  | dname      | grade |
+--------+------------+-------+
| SMITH  | RESEARCH   |     1 |
| ALLEN  | SALES      |     3 |
| WARD   | SALES      |     2 |
| JONES  | RESEARCH   |     4 |
| MARTIN | SALES      |     2 |
| BLAKE  | SALES      |     4 |
| CLARK  | ACCOUNTING |     4 |
| SCOTT  | RESEARCH   |     4 |
| KING   | ACCOUNTING |     5 |
| TURNER | SALES      |     3 |
| ADAMS  | RESEARCH   |     1 |
| JAMES  | SALES      |     1 |
| FORD   | RESEARCH   |     4 |
| MILLER | ACCOUNTING |     2 |
+--------+------------+-------+
14 rows in set (0.00 sec)

```

难度升级

案例：找出每个员工的部门名称、工资等级以及上级领导

```sql
select emp.ename,dept.dname,salgrade.grade,boss.ename
from emp
left join dept
on emp.deptno = dept.deptno
left join salgrade
on emp.sal between salgrade.losal and salgrade.hisal
left join emp as boss
on emp.mgr = boss.empno;

+--------+------------+-------+-------+
| ename  | dname      | grade | ename |
+--------+------------+-------+-------+
| SMITH  | RESEARCH   |     1 | FORD  |
| ALLEN  | SALES      |     3 | BLAKE |
| WARD   | SALES      |     2 | BLAKE |
| JONES  | RESEARCH   |     4 | KING  |
| MARTIN | SALES      |     2 | BLAKE |
| BLAKE  | SALES      |     4 | KING  |
| CLARK  | ACCOUNTING |     4 | KING  |
| SCOTT  | RESEARCH   |     4 | JONES |
| KING   | ACCOUNTING |     5 | NULL  |
| TURNER | SALES      |     3 | BLAKE |
| ADAMS  | RESEARCH   |     1 | SCOTT |
| JAMES  | SALES      |     1 | BLAKE |
| FORD   | RESEARCH   |     4 | JONES |
| MILLER | ACCOUNTING |     2 | CLARK |
+--------+------------+-------+-------+
14 rows in set (0.00 sec)
```



#### 2.4.5 联合查询

案例：找出工作岗位是SALESMAN和MANAGER的员工

```sql
#第一种方式
select ename,job from emp where job = 'MANAGER' or job = 'SALESMAN';

#第二种
select ename,job from emp where job in ('MANAGER','SALESMAN');
+--------+----------+
| ename  | job      |
+--------+----------+
| ALLEN  | SALESMAN |
| WARD   | SALESMAN |
| JONES  | MANAGER  |
| MARTIN | SALESMAN |
| BLAKE  | MANAGER  |
| CLARK  | MANAGER  |
| TURNER | SALESMAN |
+--------+----------+
7 rows in set (0.00 sec)

#第三种union
select ename,job from emp where job = 'MANAGER'
union
select ename,job from emp where job = 'SALESMAN';
```

两张不相干的数据拼接在一块显示



### 2.5 嵌套子查询

什么是子查询？子查询都可以出现在哪里？ 

```sql
select ...(sub select)
from ...(sub select)
where ...(sub select)
```

#### 2.5.1 where 语句中的子查询

案例：找出高于平均薪资的员工信息。

```sql
select * from emp where sal > avg(sal); #错误！where后面不能直接使用分组函数

#先找出平均薪资
select avg(sal) from emp;

#再where过滤
select * from emp where sal > (select avg(sal) from emp);
mysql> select * from emp where sal > (select avg(sal) from emp);
+-------+-------+-----------+------+------------+---------+------+--------+
| EMPNO | ENAME | JOB       | MGR  | HIREDATE   | SAL     | COMM | DEPTNO |
+-------+-------+-----------+------+------------+---------+------+--------+
|  7566 | JONES | MANAGER   | 7839 | 1981-04-02 | 2975.00 | NULL |     20 |
|  7698 | BLAKE | MANAGER   | 7839 | 1981-05-01 | 2850.00 | NULL |     30 |
|  7782 | CLARK | MANAGER   | 7839 | 1981-06-09 | 2450.00 | NULL |     10 |
|  7788 | SCOTT | ANALYST   | 7566 | 1987-04-19 | 3000.00 | NULL |     20 |
|  7839 | KING  | PRESIDENT | NULL | 1981-11-17 | 5000.00 | NULL |     10 |
|  7902 | FORD  | ANALYST   | 7566 | 1981-12-03 | 3000.00 | NULL |     20 |
+-------+-------+-----------+------+------------+---------+------+--------+
6 rows in set (0.00 sec)
```

#### 2.5.2 from 语句中的子查询

案例：找出每个部门平均薪水的薪资等级

```sql
select dept.dname,avg(emp.sal) as avgsal
from dept
left join emp
on dept.deptno = emp.deptno
group by dept.dname;

+------------+-------------+
| dname      | avgsal      |
+------------+-------------+
| ACCOUNTING | 2916.666667 |
| RESEARCH   | 2175.000000 |
| SALES      | 1566.666667 |
| OPERATIONS |        NULL |
+------------+-------------+
4 rows in set (0.00 sec)

薪水等级怎么办？
将上述的表再和salgrade表进行连接查询，条件是 上表的avgsal between salgrade.losal and hisal;

select t.*,s.grade
from
(select dept.dname,avg(emp.sal) as avgsal from dept left join emp on dept.deptno = emp.deptno group by dept.dname) t
join salgrade s
on t.avgsal between s.losal and s.hisal;

+------------+-------------+-------+
| dname      | avgsal      | grade |
+------------+-------------+-------+
| ACCOUNTING | 2916.666667 |     4 |
| RESEARCH   | 2175.000000 |     4 |
| SALES      | 1566.666667 |     3 |
+------------+-------------+-------+
3 rows in set (0.00 sec)
```

案例：找出每个部门平均薪水的等级

```sql
select dept.dname,avg(emp.sal)
from dept
left join emp
on dept.deptno = emp.deptno
group by dept.dname;
```

#### 2.5.3 select 语句中的子查询

案例：找出每个员工所在的部门名称，要求显示员工名和部门名

```sql
select emp.ename,dept.dname
from emp
left join dept
on emp.deptno = dept.deptno;

#还可以
select emp.ename,
(select dept.dname from dept where emp.deptno = dept.deptno) as dname 
from emp;

+--------+------------+
| ename  | dname      |
+--------+------------+
| SMITH  | RESEARCH   |
| ALLEN  | SALES      |
| WARD   | SALES      |
| JONES  | RESEARCH   |
| MARTIN | SALES      |
| BLAKE  | SALES      |
| CLARK  | ACCOUNTING |
| SCOTT  | RESEARCH   |
| KING   | ACCOUNTING |
| TURNER | SALES      |
| ADAMS  | RESEARCH   |
| JAMES  | SALES      |
| FORD   | RESEARCH   |
| MILLER | ACCOUNTING |
+--------+------------+
14 rows in set (0.00 sec)
```



### 2.6 limit

limit(重点中的重点，分页查询就靠他)

limit是mysql特有的，其他数据库没有，不通用。(oracle中有一个相同的机制，叫做rownum)

limit取结果集中的部分数据，这是他的作用

```sql
语法机制：
limit startIndex,length
startIndex：表示起始位置
length：表示取几个
```

案例：去除工资前5名的员工(思路：降序取前五个)

```sql
select ename,sal from emp order by sal desc;
select ename,sal from emp order by sal desc limit 0,5;
select ename,sal from emp order by sal desc limit 5; #默认的前面是0
```

**limit是SQL语句最后执行的环节**

```sql
select ...		5
from ... 			1
where ...			2
group by ...	3
having ...		4
order by ...	6
limit ...			7
```

案例：找出工资排名在第4到第9名的员工

```sql
select ename,sal from emp order by sal desc limit 3,6;
+--------+---------+
| ename  | sal     |
+--------+---------+
| JONES  | 2975.00 |
| BLAKE  | 2850.00 |
| CLARK  | 2450.00 |
| ALLEN  | 1600.00 |
| TURNER | 1500.00 |
| MILLER | 1300.00 |
+--------+---------+
6 rows in set (0.01 sec)
```

案例

通用页码计算： limit (pageNo-1)*pageSize,pageSize

## 3.表的创建

语法规则

```sql
create table TABLE_NAME(
  FIELD_NAME_1 数据类型,
  FIELD_NAME_2 数据类型,
  FIELD_NAME_3 数据类型,
  FIELD_NAME_4 数据类型,
  FIELD_NAME_5 数据类型
);
```

数据类型，常见

```sql
INT 		整数型(java中的int)
BIGINT	长整型(java中的long)
FLOAT		浮点型(java中的float double)
CHAR		定长字符串(java中的string)
VARCHAR	可变长字符串(StringBuffer/StringBuilder)
DATE		日期类型(java中的java.sql.Date类型)
BLOB		二进制大对象(存储图片、视频等流媒体信息) Binary Large Object(java中的Object)
CLOB		字符大对象(存储大文本，比如，可以存储4G的字符串。) Character Large Object (java中的Object)
```

char 和 varchar

```markdown
在实际开发中，当某个字段中的数据长度不发生改变的时候，是定长的，例如：性别、生日等都是采用char
当一个字段的数据长度不确定，例如：简介、姓名等都是采用varchar的
```

BLOB和CLOB类型

```sql
电影表： t_movie
id(int) 		name(varchar) 		playtime(data/char) 		haibao(BLOB)		history(CLOB)
-----------------------------------------------------------------------------------
1							猪猪侠
2
3
insert ... 直接把影片数据存到数据比较少
```

表明一般建议以：”t\_“或者”tbl\_“开头

```sql
#创建学生表
#学生信息包括：学号(bigint)、姓名(varchar)、性别(char)、班级编号(int)、生日(char)

create table t_student(
	id bigint,
  name varchar(255),
  gender char(1),
  class_no varchar(255),
  birth char(10)
);
```

### 3.1 带默认值

```sql
create table t_student(
	id bigint,
  name varchar(255),
  gender char(1) default 1,
  class_no varchar(255),
  birth char(10)
);
```



### 3.2 insert 插入数据

语法规则：

```sql
insert into TABLE_NAME(FIELD_NAME_1,FIELD_NAME_2,...) values(值1,值2,值3...)
```

插入数据：

```sql
insert into t_student(id,name,gender,class_no,birth) 
values(1,'zhangsan','1','class_1','1950-12-01');
```

当一条insert语句执行成功后，表格当中必然会多一行记录。

一次插入多行数据

```sql
insert into t_student(id,name,gender,class_no,birth)
values(2,"lisi","0","class_2","1951-11-02"),(3,"wangwu","1","class_2","1951-08-02");
```



### 3.3 删除表

```sql
drop table t_student; #通用

drop table if exists t_student;	//当这个表存在的话删除 oracle不支持
```



### 3.4 表的复制及批量插入

表的复制

```sql
create table emp1 as select * from emp;
Query OK, 14 rows affected, 2 warnings (0.02 sec)
Records: 14  Duplicates: 0  Warnings: 2

create table emp2 as select empno,ename from emp;
Query OK, 14 rows affected (0.01 sec)
Records: 14  Duplicates: 0  Warnings: 0
```

批量插入

```sql
create table dept1 as select * from dept;

insert into dept1 select * from dept;
```

 

## 4 数据修改

语法规则

```sql
update TABLE_NAME set FIELD_NAME_1 = FIELD_VALUE_1 where 条件;
```

案例：将部门10的LOC修改为SHANGHAI，将部门名称修改为RENSHIBU

```sql
update dept1 set loc = 'SHANGHAI',dname = 'RENSHIBU' where deptno = 10;
```

更新所有的记录

```sql
update dept1 set loc = 'x',dname='y' #所有数据都修改了
```

## 5 数据删除

语法规则

```sql
delete from TABLE_NAME where 条件;
#没有条件，全部删除

#删除10部门数据
delete from dept1 where deptno = 10;

#删除所有记录
delete from dept1;
#注意：如果这个表的数据量特别大的话，删除一张表要好几个小时，delete没有释放数据的空间，还可以回滚的
```

怎么删除大表数据？

```sql
#反复确认两次后，采用另一种方式，直接截断，只留表头，其他的全部扔掉
truncate table dept1; #表被截断，不可回滚，永久丢失
```

## 6 表的结构修改

先建表

```sql
CREATE TABLE `test_a` (
`id` char(32) NOT NULL COMMENT '主键ID',
`work_id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增列',
`creator_id` char(32) DEFAULT NULL COMMENT '创建人',
`create_time` datetime DEFAULT NULL COMMENT '创建时间',
`modifier_id` char(32) DEFAULT NULL COMMENT '修改人',
`modify_time` datetime DEFAULT NULL COMMENT '修改时间',
`status` tinyint(1) NOT NULL DEFAULT '1' COMMENT '状态:1 启动 0 停用',
`deleted` tinyint(1) NOT NULL DEFAULT '0' COMMENT '删除标记：0、未删除 1、已删除',
`handle_mission` tinyint(1) unsigned zerofill DEFAULT NULL COMMENT '手动完成，0，正常上刊任务；1，手动完成任务',
`signed_customer` varchar(64) DEFAULT NULL COMMENT '签约客户',
# 主键索引
PRIMARY KEY (`id`) USING BTREE,
# 唯一索引
UNIQUE KEY `index_id` (`id`) USING BTREE,
# 索引
KEY `index_signed_customer` (`signed_customer`) USING BTREE,
) ENGINE=InnoDB AUTO_INCREMENT=100 DEFAULT CHARSET=utf8mb4
```



修改表信息

```sql
1.修改表名 
alter table test_a rename to test_b;

2.修改表注释 
alter table test_a comment 'test_a字段注释';
```

修改字段信息

```sql
1.修改字段类型
alter table test_a modify column user_name text;

2.修改字段类型和注释
alter table test_a modify column user_name varchar(20) COMMENT '用户名字';

3.设置字段允许为空
alter table test_a modify column user_name varchar(255) default null COMMENT '用户名字';

4.修改为自增主键
alter table test_a modify column a_id int(5) auto_increment ;

5.修改字段名字(要重新指定该字段的类型)
alter table test_a change user_name user_name2 varchar(255) not null comment '其他的备注信息';
```

表增加字段

```sql
1.增加一个字段，设好数据类型，且不为空，添加注释
alter table test_a add user_name varchar(255) not null comment '用户名字';

2.给字段增加主键
alter table test_a add primary key (a_id);

3.增加一个字段（a_id）并设置为主键 
alter table test_a add a_id int(5) not null , add primary key (a_id);

4.增加字段并设置为自增主键
alter table test_a add a_id int(5) not null auto_increment , add primary key (a_id);
```

删除表字段

```sql
1.删除字段
alter table test_a drop user_name;
```

其他

```sql
1.在某个字段(a_id)后增加字段
alter table test_a add column user_age int(4) not null default 0 AFTER a_id；

2.调整字段顺序 #(注意gateway_id出现了2次)
alter table test_a change user_age user_age int not null AFTER a_id ;
```

## 7 约束

什么是约束？常见的约束有哪些？

```markdown
在创建表的时候，可以给表的字段添加相应的约束，添加约束的目的是为了保证表中的数据的合法性、有效性、完整性。
```

常见的约束有哪些？

```markdown
非空约束(not null) 			约束的字段不能为null
唯一约束(unique) 				约束的字段不能重复
主键约束(primary key)		约束的字段不能为null也不能重复
外键约束(foreign key)		(简称FK)
检查约束(check)
#注意，Oracle数据库中check约束，但是mysql没有，目前mysql不支持该约束
```

### 7.1 非空约束

```sql
drop table if exists t_user;
create table t_user(
	id int,
  username varchar(255) not null,
  password varchar(255)
);
insert into t_user(id,password) values(1,'123');
Error 1364(HY000): Field 'username' doesn't have a default value
```

注意：not null约束只有列级约束，没有表级约束

### 7.2 唯一性约束

唯一约束修饰的字段具有唯一性，不能重复。但是可以为NULL

给某一列添加uniques，示例：

````sql
drop table if exists t_user;
create  table t_user(
	id int,
  username varchar(255) unique,
);

insert into t_user values(1,'zhangsan');
insert into t_user values(2,'zahngsan');
Error 1062 (2300):Duplicate entry 'zhangsan' for key 'username';

#那这样行不行呢？
insert into t_user(id) values(2);
insert into t_user(id) values(3);
insert into t_user(id) values(4);

#所以这个unique，可以为NULL，但是不能重复
````

能不能给多个字段加unique，示例：

```sql
#让username和usercode加起来唯一
drop table if exists t_user;
create  table t_user(
	id int,
  username varchar(255),
  usercode varchar(255),
  unique(username,usercode)	#多个字段联合起来添加1个约束unique，这个称为表级约束
);

#不重复
insert into t_user values(1,'111','zs');
insert into t_user values(2,'222','ls');
insert into t_user values(3,'333','ww');

#重复
insert into t_user values(1,'111','zs');


和下面这个是有本质区别的
create  table t_user(
	id int,
  username varchar(255) unique, #这是列级约束
  usercode varchar(255) unique,
);

insert into t_user values(1,'111','zs');
insert into t_user values(2,'222','ls');
```

### 7.3 主键约束

怎么给一张表添加主键约束

```sql
drop table if exists t_user;
create table t_user(
	id int primary key,
  username varchar(255),
  email varchar(255)
);

insert into t_user values(1,'zs','zs@qq.com');
insert into t_user values(2,'ls','ls@qq.com');
insert into t_user values(3,'ww','ww@qq.com');
insert into t_user values(4,'zl','zl@qq.com');

insert into t_user(username,email) values('xx','xx@qq.com');
Error 1364(HY000) : Field 'id' dosen't have a default value;

insert into t_user(id,username,email) values('5','x1','x1@qq.com');
Error 1062 (23000) : Duplicate entry '1' for key 'PRIMARY';
```

根据以上的测试得出：id是主键，因为添加了主键的约束，主键字段中的数据不能为NULL，也不能重复

主要的特点：不能为NULL，也不能重复。

注意：主键是列级约束

**主键相关的术语**

```markdown
主键约束
主键字段
主键值
```

**主键有什么作用？**

```sql
- 表的设计三范式中又去要求，第一范式就是要求任何一张表都应该有主键
- 主键的作用：主键值是这行记录在这张表当中的唯一标识。(就像一个人的身份证号码一样。)
```

**主键的分类？**

根据主键字段的字段数量来划分：

单一主键：推荐的，常用的

复合主键：多个字段联合起来添加一个主键约束； 复合主键不建议使用，因为复合主键违背三范式。



根据主键性质来划分：

自然主键：主键值最好就是一个和业务没有任何关系的自然数。

业务主键：主键值和系统的业务挂钩，例如：拿着银行卡的卡号做主键，拿着身份证号码作为主键(不推荐使用)

最好不要拿着和业务挂钩的字段作为主键。因为以后的业务一旦发生改变的时候，主键值可能也需要随之发生变化；但是有时候没有办法变化，因为变化可能会导致主键值重复。

实际业务中这样写

```sql
id(PK)			id_card(unique)
-----------------------------
自然主键 		|									|
-----------------------------
			  	 |								 |
```

**注意：一张表的主键约束只能有一个(必须记住)**

使用标记约束方式定义主键

```sql
drop table if exists t_user;
create table t_user(
	id int,
  username varchat(255),
  primary key(id)
);

insert into t_user(id,username) values(1,'zs');
insert into t_user(id,username) values(2,'ls');
insert into t_user(id,username) values(3,'ws');
insert into t_user(id,username) values(4,'cs');
```

**复合主键**

```sql
drop table if exists t_user;
create table t_user(
	id int,
  username varchar(255),
  password varchar(255),
  primary key(id,username)
);
```

### 7.4 主键自增

```sql
drop table if exists t_user;
create table t_user(
	id int primary key auto_increment,
  username varchar(255),
  password varchar(255)
);

insert into t_user(username) values('zs');
insert into t_user(username) values('ls');
insert into t_user(username) values('ws');
insert into t_user(username) values('es');
insert into t_user(username) values('rs');

select * from t_user;
```

### 7.5 外键约束

关于外键约束的相关术语

```markdown
外键约束：foreign key
外键字段：添加有外键约束的字段
外键值：外键字段中的每一个值
```

业务背景：

请设置及数库表，用来维护学生和班级的信息

```sql
#第一种方案：一张表存储所有数据
no(PK)			name				class_no					class_name
----------------------------------------------------------
1							zs					101							北京市云岗区云岗北里
2							ls					101							北京市云岗区云岗北里
3							ws					101							北京市云岗区云岗北里
4							es					102							北京市云岗区云岗北里
5							rs					102							北京市云岗区云岗北里
6							ts					102							北京市云岗区云岗北里
```

缺点：冗余[不推荐]

```sql
#第二种方案：两张表(班级表和学生表)
t_class 班级表
cno(pk)					cname
------------------
101					北京市云岗区云岗北里
102					北京市云岗区云岗北里

t_student 学生表
sno(pk)					sname				cno(该字段添加外键约束FK)
--------------------------------------------------
1								 zs1				101
2								 zs2				101
3								 zs3				102
4								 zs4				102

#将以上表的建表语句写出来
t_studet中的classno字段引用t_calss表中的cno字段，此时t_student表叫做子表。t_class表叫做父表。
删除数据的时候，先删除子表，再删除父表。
添加数据的时候，先添加父表，再添加子表。
创建表的时候，先创建父表，再创建子表。
删除表的时候，先删除子表，再删除父表，父表不能随便删掉的。

drop table if exists t_student;
drop table if exists t_class;

create table t_class(
	cno int,
  cname varchar(255),
  primary key(cno)
);

create table t_student(
	sno int,
  sname vatchar(255),
  classno int,
  foreign key(classno) references t_class(cno)
);

insert into t_class values(101,'xxxxx1');
insert into t_class values(102,'xxxxx2');

insert into t_student values(1,'zs1',101);
insert into t_student values(2,'zs2',101);
insert into t_student values(3,'zs3',102);
insert into t_student values(4,'zs4',102);
insert into t_student values(5,'zs5',102);

insert into t_student values(6,'zs6',103);
Error 1452 (23000) : Cannot add or update a child row : a foreign key constraint fails;
```

**外键是否能为NULL？**

```sql
insert into t_student(sno,sname) values(7,'zs7');
#执行成功的，外键可以为NULL的
```

**外键字段引用其他表的某个字段的时候，被引用的字段必须是主键吗？**

外键的值，至少具有唯一unique性，但不一定是主键

## 8 存储引擎

### 8.1 存储引擎的使用

数据库中的各个表(在创建表时)指定的存储引擎来处理。

服务器可用的引擎依赖于以下因素：

```markdown
Mysql版本
服务器开发时如何被设置
启动的选项
```

查看当前服务器有哪些存储引擎可用，可以使用SHOW ENGINES语句：

```sql
SHOW ENGINES
```

查看表的创建语句

```sql
show create table emp;

ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

### 8.2 存储引擎

完整建表语句

```sql
create table t_x(
	id int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

建表的时候可以指定存储引擎，也可以指定字符集。

Mysql默认使用的存储引擎是InnoDB方式，默认采用的字符集是UTF8

**什么是存储引擎？**

存储引擎这个名字中只有在MySQL中存在。(Oracle中有对应的机制，但是不叫做存储引擎，ORACLE中没有特殊的名字，就是"表的存储方式")

Mysql支持很多存储引擎，每一个存储引擎都对应了一种不同的存储方式。

每一个存储引擎都有自己的优缺点，需要在合适的时机选择合适的引擎。



**MySQL支持的引擎**

#### 8.2.1 MyISAM存储引擎

```markdown
Engine	:	MyISAM
		Support				:		YES
		Comment				:		MyISAM storage engine
		Transactions	:		NO											(不支持事务)
		XA						:		NO
		Savepoints		: 	NO
```

MyISAM这种存储引擎不支持事务，最常用的存储引擎，但是不是默认的

他管理的表具有以下特征：

```markdown
一、使用三个文件表示每个表：
		格式文件	-	存储表结构的定义(mytable.frm)
		数据文件	-	存储表行的内容(mytabls.MYD)
		索引文件	-	存储表上索引(mytable.MYI)
二、灵活的AUTO_INCREMENT字段处理
三、可被转换为压缩、只读表来节省空间
```

优点：可被压缩，节省存储空间，并且可以转换为只读表，提高检索效率。

缺点：不支持事务。



#### 8.2.2 InnoDB存储引擎

```markdown
Engine	:	InnoDB
		Support				:		DEFAULT
		Comment				:		Supports transactions,row-level locking,and foreign keys
		Transactions	:		YES
		XA						:		YES
		Savepoints		: 	YES
```

InnoDB存储引擎是MySQL的缺省引擎

他管理的表具有下列主要特征：

````sql
一、每个InnoDB表在数据库目录中以.frm格式文件表示
二、InnoDB表空间tablespace被用于存储表的内容，不能被压缩，不节省空间
三、提供一组用来记录事务性活动的日志文件
四、用COMMIT(提交)、SAVEPOINT及ROLLBACK(回滚)支持事务处理
五、提供全ACID兼容
六、在MySQL服务器崩溃后提供自动恢复
七、多版本(MVVC)和行及锁定
八、支持外键及引用的完整性，包括级联删除和更新
````

优点：支持事务，最安全，InnoDB是事务型数据库的首选引擎，支持事务安全表（ACID）

表的结构存储在xxx.frm文件中，数据存储在tablespace这样的空间中（逻辑概念），不支持压缩，不节省空间，无法被转换成只读。

这种InnoDB存储引擎在MySQL数据库崩溃之后提供自动恢复机制。

InnoDB支持级联删除和级联更新。

#### 8.2.3 MEMORY存储引擎

使用MEMORY存储引擎的表，其数据存储在内存中，且行的长度固定，这两个特点是的MEMORY存储引擎非常快。

```markdown
Engine	: MEMORY
		Support				:		YES
		Comment				:		Hash based,stored in memory,useful for temporary tables
		Transactions	:		NO
		XA						:		NO
		Savepoints		: 	NO
```

缺点：不支持事务，数据容易丢失，所有数据和索引都是存储在内存当中的。

优点：查询速度最快。

以前叫做HEAP引擎

```markdown
MEMORY存储引擎管理的表具有下列特征：
一、在数据库目录中，每个表均以.frm格式的文件表示
二、表数据及索引被存储在内存中
三、表级锁机制
四、不能包含TEXT和BLOB字段
```

### 8.3 选择合适的引擎

```markdown
一、MyISAM表最合适于大量的数据读，少量数据更新的混合操作。MyISAM表的另一种适用情形是使用压缩的只读表
二、如果查询中包含较多的数据更新操作，应使用InnoDB。其行级锁机制和多半的支持为数据读取和更新的混合操作提供了良好的并发机制
三、可使用MEMORY存储引擎来存储非永久需要的数据，或者是能够从基于磁盘的表中重新生成数据
```



## 9 事务

### 9.1 什么是事务

```markdown
一个事务是一个完整的业务逻辑单元，不可再分。
比如：银行账户转账，从A账户向B账户转账10000，需要执行两条update语句
update t_act set balance = balance-10000 where actno = 'act-001';
update t_act set balance = balance+10000 where actno = 'act-002';
以上两条DML语句必须同时成功或者同时失败，不允许出现一条成功，一条失败。
```

要想保证以上两条DML语句同时成功或者同时失败，那么就需要使用数据库的"事务机制"

只有DML才和事务相关，(insert delete update)，因为这三个语句都是和数据库当中的"数据相关的。

事务的存在是为了保证数据的完整性，安全性。



### 9.2 事务理解

```markdown
开启事务机制(开始)

执行insert语句--->insert...(执行成功后，把这个执行记录到数据库的操作历史当中，并不会向文件中保存一条数据，不会真正的修改硬盘上的数据)

执行update语句---->update...(执行成功后，把这个执行记录到数据库的操作历史当中，并不会真正修改硬盘上的数据)

执行delete语句---->delete...(一样的记录...)

提交或者回滚事务(结束)
```

### 9.3 事务的特性-ACID

A：原子性 事务是最小的工作单元，不可再分的

C：一致性 事务必须保证多条DML语句同时成功或者同时失败

I：隔离性  事务A与事务B之间具有隔离性

D：持久性 最终数据必须持久化到硬盘中，事务才算成功的结束



### 9.4 事务的隔离级别

事务隔离性存在隔离级别，理论上隔离级别包括四个：

```markdown
第一级别：读未提交(READ_UNCOMMIT),当前事务可以读取到对方未提交的数据；读未提交存在脏读(Dirty Read)现象-表示读到了脏的数据。
第二级别：读已提交(READ_COMMITTED),对方事务提交之后的数据我方才可以读取到。解决了脏读现象，读已提交存在的问题是-不可重复读。行锁

第三级别：可重复读(REPETABLE_READ),解决了不可重复读的问题，存在的问题是幻读。
在READ_COMMITTED的隔离级别下出现的问题-不可重复读：两个事务，一个事务把一张表的数据清空，另外一个事务还是能读到数据...。表锁

第四级别：串行(SERIALIZABLE)
```

### 9.5 演示事务

mysql事务默认情况下是自动提交的(什么是自动提交？只要执行任意一条DML语句则提交一次)

如何关闭自动提交？

```sql
start transaction;
```

演示

```sql
#准备表：
drop table if exists t_user;
create table t_user(
	id int(11) primary key auto_increment,
  username varchar(255)
);

start transaction;
insert into t_user(username) values('ws');
rollback; #事务结束
```



使用两个事务演示以上的隔离级别

打开两个命令行窗口登录mysql

```sql
1.mysql -uroot -p
2.show variables like 'transaction%';
3.set global transaction isolation level read uncommitted;
4.select @@transaction_isolation;
```



#### 9.5.1 修改事务隔离级别

数据库事务的隔离级别有4种，由低到高分别为read uncommitted（读未提交） 、read committed（读提交） 、repeatable read（重复读） 、Serializable（序列化） 。

语法

```sql
SET [GLOBAL | SESSION] TRANSACTION
    transaction_characteristic [, transaction_characteristic] ...

transaction_characteristic: {
    ISOLATION LEVEL level
  | access_mode
}

level: {
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
}

access_mode: {
     READ WRITE
   | READ ONLY
}
```

##### 使用例子

设置本次会话的事务隔离级别，只在本会话有效，不会影响到其它会话

```sql
-- 设置本次会话的事务隔离级别，只在本会话有效，不会影响到其它会话
mysql> set session transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

-- 再次查看发现已改成了read committed
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| READ-COMMITTED          |
+-------------------------+
-- 再登录其它窗口，再查看，发现还是repeatable read
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
```

设置全局的事务隔离级别，该设置不会影响当前已经连接的会话，设置完毕后，新打开的会话，将使用新设置的事务隔离级别

```sql
-- 设置全局的事务隔离级别
mysql> set global transaction isolation level read committed;
Query OK, 0 rows affected (0.00 sec)

-- 不影响本次已连接会话的事务，所以查到的还是修改前的
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
-- 再打开一个新会话，则变成了修改后的read committed
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| READ-COMMITTED          |
+-------------------------+

```

设置下一次事务操作的隔离级别，该设置会随着下一次事务的提交而失效

```sql
-- 查看自动提交是否开启
mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
-- 关闭自动提交
mysql> set autocommit = 0;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+

-- 设置本次事务
set transaction isolation level read uncommitted;

-- 提交事务
commit;
```

#### 9.5.3 通过配置文件my.ini也可以修改事务

```ini
[mysqld]
transaction-isolation = REPEATABLE-READ
transaction-read-only = OFF
```

## 10 索引

**什么是索引？有什么用？**

```markdown
索引就相当于一本书的目录，通过目录可以快速的找到对应的资源。
在数据库方面，查询一张表的时候有两种检索方式：
第一种方式：全表扫描
第二种方式：根据索引检索(效率很高)

问：索引为什么提高了检索的效率？
答：其缩小了扫描的范围
```

索引虽然可以提高检索的效率，但不能随意的添加索引，因为索引也是数据库当中的对象，也需要数据库不断的维护，他是有维护成本的。比如，表中的数据经常被修改这样就不适合添加索引，因为数据一旦修改，索引需要重新进行排序，进行维护。

```markdown
添加索引是给某一个字段，或者说某些字段添加索引。

select ename,sal from emp where ename='SMITH';
当ename字段上没有添加索引的时候，以上sql语句会进行全表扫描，扫描ename字段中的所有值。
当ename字段上添加索引的时候，以上sql语句会根据索引扫描，快速定位。
```



**怎么创建索引对象？怎么删除索引对象？**



**执行计划**

查看SQL语句的执行计划，可以使用explain

```sql
mysql> explain select ename from emp where sal = 5000;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   14 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

#给sal字段添加索引
create index emp_sal_index on emp(sal);

mysql> explain select ename from emp where sal = 5000;
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key           | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | emp   | NULL       | ref  | emp_sal_index | emp_sal_index | 9       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+---------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

#删除索引对象
drop index 索引名称 on 表名
```



**什么时候考虑给字段添加索引？**（满足什么条件）

````markdown
数据量庞大(根据需求与线上的环境)
该字段很少的DML操作(因为字段进行修改，索引需要进行索引的重排)
该字段经常作为查询字段
````



**索引底层采用的数据结构是 B+Tree**



[索引类型](http://c.biancheng.net/view/7897.html)



### 10.1 **mysql联合索引**

命名规则：表名_字段名
1、需要加索引的字段，要在where条件中
2、数据量少的字段不需要加索引
3、如果where条件中是**OR**关系，加索引不起作用
4、符合最**左**原则

联合索引又叫复合索引。对于复合索引:Mysql从左到右的使用索引中的字段，一个查询可以只使用索引中的一部份，但只能是最左侧部分。例如索引是key index (a,b,c). 可以支持**a** | **a,b**| **a,b,c** 3种组合进行查找，但不支持 b,c进行查找 .当最左侧字段是常量引用时，索引就十分有效。



两个或更多个列上的索引被称作复合索引。
利用索引中的附加列，您可以缩小搜索的范围，但使用一个具有两列的索引 不同于使用两个单独的索引。
复合索引的结构与电话簿类似，人名由姓和名构成，电话簿首先**按姓氏对进行排序**，然后按名字对有相同姓氏的人进行排序。
如果您知 道姓，电话簿将非常有用；
如果您知道姓和名，电话簿则更为有用，
但如果您**只知道名不姓，电话簿将没有用处**。

所以说创建复合索引时，应该仔细考虑列的顺序。对索引中的所有列执行搜索或仅对前几列执行搜索时，复合索引非常有用；仅对后面的任意列执行搜索时，复合索引则没有用处。



当一个表有多条索引可走时, Mysql 根据查询语句的成本来选择走哪条索引, 联合索引的话, 它往往计算的是第一个字段(最左边那个), 这样往往会走错索引. 如: 
索引Index_1(Create_Time, Category_ID), Index_2(Category_ID) 

如果每天的数据都特别多, 而且有很多category, 但具体每个category的记录不会很多.

当查询SQL条件为select …where create_time ….and category_id=..时, 很可能不走索引Index_1, 而走索引Index_2, 导致查询比较慢.

解决办法是将索引字段的顺序调换一下.



### 10.2 **创建索引**

在执行CREATE TABLE语句时可以创建索引，也可以单独用CREATE INDEX或ALTER TABLE来为表增加索引。

1．ALTER TABLE

ALTER TABLE用来创建普通索引、UNIQUE索引或PRIMARY KEY索引。

ALTER TABLE table_name ADD INDEX index_name (column_list)

ALTER TABLE table_name ADD UNIQUE (column_list)

ALTER TABLE table_name ADD PRIMARY KEY (column_list)

其中table_name是要增加索引的表名，column_list指出对哪些列进行索引，多列时各列之间用逗号分隔。索引名index_name可选，缺省时，MySQL将根据第一个索引列赋一个名称。另外，ALTER TABLE允许在单个语句中更改多个表，因此可以在同时创建多个索引。



2．CREATE INDEX

CREATE INDEX可对表增加普通索引或UNIQUE索引。

CREATE INDEX index_name ON table_name (column_list)

CREATE UNIQUE INDEX index_name ON table_name (column_list)

table_name、index_name和column_list具有与ALTER TABLE语句中相同的含义，索引名不可选。另外，不能用CREATE INDEX语句创建PRIMARY KEY索引。



3．索引类型

在创建索引时，可以规定索引能否包含重复值。如果不包含，则索引应该创建为PRIMARY KEY或UNIQUE索引。对于单列惟一性索引，这保证单列不包含重复的值。对于多列唯一性索引，保证多个值的组合不重复。

PRIMARY KEY索引和UNIQUE索引非常类似。
事实上，PRIMARY KEY索引仅是一个具有**名称PRIMARY**的UNIQUE索引。这表示一个表只能包含一个PRIMARY KEY，因为一个表中不可能具有两个同名的索引。

下面的SQL语句对students表在sid上添加PRIMARY KEY索引。

ALTER TABLE students ADD PRIMARY KEY (sid)

 

4．删除索引

可利用ALTER TABLE或DROP INDEX语句来删除索引。类似于CREATE INDEX语句，DROP INDEX可以在ALTER TABLE内部作为一条语句处理，语法如下。

DROP INDEX index_name ON talbe_name

ALTER TABLE table_name DROP INDEX index_name

ALTER TABLE table_name DROP PRIMARY KEY

其中，前两条语句是等价的，删除掉table_name中的索引index_name。



第3条语句只在删除PRIMARY KEY索引时使用，因为一个表只可能有一个PRIMARY KEY索引，因此不需要指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。

如果从表中删除了某列，则索引会受到影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。



### 10.3 什么情况下使用索引

**表的主关键字**

自动建立唯一索引

如zl_yhjbqk（用户基本情况）中的hbs_bh（户标识编号）

**表的字段唯一约束**

ORACLE利用索引来保证数据的完整性

如lc_hj（流程环节）中的lc_bh+hj_sx（流程编号+环节顺序）

**直接条件查询的字段**

在SQL中用于条件约束的字段

如zl_yhjbqk（用户基本情况）中的qc_bh（区册编号）

select * from zl_yhjbqk where qc_bh=’<????甼曀???>7001’

**查询中与其它表关联的字段**

字段常常建立了外键关系

如zl_ydcf（用电成份）中的jldb_bh（计量点表编号）

select * from zl_ydcf a,zl_yhdb b where a.jldb_bh=b.jldb_bh and b.jldb_bh=’540100214511’

**查询中排序的字段**

排序的字段如果通过索引去访问那将大大提高排序速度

select * from zl_yhjbqk order by qc_bh（建立qc_bh索引）

select * from zl_yhjbqk where qc_bh=’7001’ order by cb_sx（建立qc_bh+cb_sx索引，注：只是一个索引，其中包括qc_bh和cb_sx字段）

**查询中统计或分组统计的字段**

select max(hbs_bh) from zl_yhjbqk

select qc_bh,count(*) from zl_yhjbqk group by qc_bh

### 10.4 **什么情况下应不建或少建索引**

**表记录太少**

如果一个表只有5条记录，采用索引去访问记录的话，那首先需访问索引表，再通过索引表访问数据表，一般索引表与数据表不在同一个数据块，这种情况下ORACLE至少要往返读取数据块两次。而不用索引的情况下ORACLE会将所有的数据一次读出，处理速度显然会比用索引快。

如表zl_sybm（使用部门）一般只有几条记录，除了主关键字外对任何一个字段建索引都不会产生性能优化，实际上如果对这个表进行了统计分析后ORACLE也不会用你建的索引，而是自动执行全表访问。如：

select * from zl_sybm where sydw_bh=’5401’（对sydw_bh建立索引不会产生性能优化）

**经常插入、删除、修改的表**

对一些经常处理的业务表应在查询允许的情况下尽量减少索引，如zl_yhbm，gc_dfss，gc_dfys，gc_fpdy等业务表。

**数据重复且分布平均的表字段**

假如一个表有10万行记录，有一个字段A只有T和F两种值，且每个值的分布概率大约为50%，那么对这种表A字段建索引一般不会提高数据库的查询速度。

**经常和主字段一块查询但主字段索引值比较多的表字段**

如gc_dfss（电费实收）表经常按收费序号、户标识编号、抄表日期、电费发生年月、操作 标志来具体查询某一笔收款的情况，如果将所有的字段都建在一个索引里那将会增加数据的修改、插入、删除时间，从实际上分析一笔收款如果按收费序号索引就已 经将记录减少到只有几条，如果再按后面的几个字段索引查询将对性能不产生太大的影响。

对千万级MySQL数据库建立索引的事项及提高性能的手段

一、注意事项：

首先，应当考虑表空间和磁盘空间是否足够。我们知道索引也是一种数据，在建立索引的时候势必也会占用大量表空间。因此在对一大表建立索引的时候首先应当考虑的是空间容量问题。

其次，在对建立索引的时候要对表进行加锁，因此应当注意操作在业务空闲的时候进行。

二、性能调整方面：

首当其冲的考虑因素便是磁盘I/O。物理上，应当尽量把索引与数据分散到不同的磁盘上（不考虑阵列的情况）。逻辑上，数据表空间与索引表空间分开。这是在建索引时应当遵守的基本准则。

其次，我们知道，在建立索引的时候要对表进行全表的扫描工作，因此，应当考虑调大初始化参数db_file_multiblock_read_count的值。一般设置为32或更大。

再次，建立索引除了要进行全表扫描外同时还要对数据进行大量的排序操作，因此，应当调整排序区的大小。

  9i之前，可以在session级别上加大sort_area_size的大小，比如设置为100m或者更大。

  9i以后，如果初始化参数workarea_size_policy的值为TRUE，则排序区从pga_aggregate_target里自动分配获得。

最后，建立索引的时候，可以加上nologging选项。以减少在建立索引过程中产生的大量redo，从而提高执行的速度。

### 10.5 MySql在建立索引优化时需要注意的问题

 

设计好MySql的索引可以让你的数据库飞起来，大大的提高数据库效率。设计MySql索引的时候有一下几点注意：

1，创建索引

对于查询占主要的应用来说，索引显得尤为重要。很多时候性能问题很简单的就是因为我们忘了添加索引而造成的，或者说没有添加更为有效的索引导致。如果不加

索引的话，那么查找任何哪怕只是一条特定的数据都会进行一次全表扫描，如果一张表的数据量很大而符合条件的结果又很少，那么不加索引会引起致命的性能下降。
但是也不是什么情况都非得建索引不可，比如性别可能就**只有两个值**，建索引不仅没什么优势，还会影响到更新速度，这被称为**过度**索引。

2，复合索引

比如有一条语句是这样的：select * from users where area=’beijing’ and age=22;

如果我们是在area和age上**分别创建**单个索引的话，由于mysql查询每次只能使用**一个索引**，所以虽然这样已经相对不做索引时全表扫描提高了很多效率，但是如果在area、age两列上创建复合索引的话将带来更高的效率。如果我们创建了(area, age,salary)的复合索引，那么其实相当于创建了(area,age,salary)、(area,age)、(area)三个索引，这被称为最佳左前缀特性。
因此我们在创建复合索引时应该将**最常用**作限制条件的列放在**最左边**，依次递减。

3，索引不会包含有NULL值的列

只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有**一列**含有**NULL值**，那么这一列对于此复合索引就是**无效**的。所以我们在数据库设计时不要让字段的默认值为NULL。

4，使用短索引

对串列进行索引，如果可能应该指定一个**前缀长度**。例如，如果有一个CHAR(255)的 列，如果在前10 个或20 个字符内，多数值是惟一的，那么就不要对整个列进行索引。**短索引**不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

5，排序的索引问题

mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含**多个列的排序**，如果需要最好给这些列创建**复合索引**。

6，like语句操作

一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而**like “aaa%”**可以使用索引。

7，不要在列上进行运算

select * from users where

YEAR(adddate)

8，不使用NOT IN

NOT IN都不会使用索引将进行全表扫描。NOT IN可以**NOT EXISTS代替**



**注意**

主键和具有unique约束的字自动会添加索引，根据主键查询效率较高，尽量根据主键进行检索

## 11 视图

### 11.1 **什么是视图？**

站在不同的角度去看到的数据(同一张表的数据，通过不同的角度去看待)。

视图是一个**虚拟表**，是sql的查询结果，其内容由查询定义。同真实的表一样，视图包含一系列带有名称的列和行数据，在使用视图时动态生成。视图的数据变化会影响到基表，基表的数据变化也会影响到视图[insert update delete ] ; 创建视图需要create view 权限，并且对于查询涉及的列有select权限；使用create or replace 或者 alter修改视图，那么还需要改视图的drop权限。

### 11.2 **怎样创建视图？**

```sql
create view myview as select empno,ename from emp;
drop view myview;
```

注意：只有DQL语句才能以视图对象的方式创建出来

对视图进行增删改查，会影响到原表的数据。（通过视图影响原表数据的，不是直接操作的原表）

### 11.3 应用场景

1、做表的权限限制可以应用。工作中存在很常见的一个工作场景时，开放数据库中的部分表字段给第三方，但是某些表中又有一些敏感的字段不能开放，那么能应用的解决方案有
一、开发接口，二、建立中间表，第一种无疑是增加了很多的工作量，第二种又增加了资源的消耗。而能够利用视图则很好的解决了这个场景带来的问题。

2、简化复杂的查询操作。对于一些复杂的查询操作，如果用一条语句来执行，语句的可读性可能会大大减低，如果能引入视图的概念，则能较好的把复杂的语句化整为零。

```markdown
1.权限控制时可以用
比如某几个列，允许用户查询，其他列不允许，可以通过视图，开放其中一列或激烈，知道权限控制的作用

2.简化复杂的查询
查询每个栏目下商品的平均价格，并按平均价格排序，查出平均价前3高的栏目

3.视图能不能更新、删除、添加？
如果视图的每一行是与物理表一一对应的，则可以，view的行是由物理表多行经过计算得到的结果，view不可以更新的
```

**大数据分表时可以用到**

比如,表的行数超过200万行时,就会变慢,

可以把一张的表的数据拆成4张表来存放.

News表

Newsid, 1,2,3,4

News1,news2,news3,news4表



把一张表的数据分散到4张表里,分散的方法很多,

最常用可以用id取模来计算.

Id%4+1 = [1,2,3,4]

比如 $_GET['id'] = 17,

17%4 + 1 = 2, $tableName = 'news'.'2'

Select * from news2 where id = 17;



还可以用视图,把4张表形成一张视图

Create view news as select from n1 union select from n2 union.........



**面向视图操作**

```sql
select * from myview;
```



### 11.4 视图的好处

```markdown
1.安全
一些数据表有着重要的信息，有些字段是保密的，不让用户直接看到，这时就可以创建一个视图，在这张视图汇总只保留一部分字段。这样，用户就可以查询到自己需要的字段，不能查看保密的字段
2.性能
关系型数据库的数据尝尝会分表存储，使用外键建立这些表之间的关系。这时，数据库查询通常会用到连接(JOIN)。这样做不但麻烦，效率相对也比较低。如果建立一个视图，将相关的表和字段组合在一起，就可以避免使用JOIN查询数据。
3.灵活
如果系统中有一张旧的表，这张表由于涉及的问题，即将被废弃。然而很多应用都是基于这张表，不易修改。这时就可以使用视图，视图中的数据直接映射到新建的表。这样就可以少做很多改动，也达到了升级数据表的目的。
```





## 12 设计三范式

### 12.1 什么是设计范式？

设计表的依据。按照这三个范式设计的表不会出现数据冗余

```markdown
一、第一范式
确保每列保持原子性
任何一张表都应该有主键，并且每一个字段是原子性的，不可再分
第一范式是最基本的范式。如果数据库表中的所有字段值都是不可分解的原子值，就说明该数据库表满足了第一范式。

第一范式的合理遵循需要根据系统的实际需求来定。比如某些数据库系统中需要用到“地址”这个属性，本来直接将“地址”属性设计成一个数据库表的字段就行。但是如果系统经常会访问“地址”属性中的“城市”部分，那么就非要将“地址”这个属性重新拆分为省份、城市、详细地址等多个部分进行存储，这样在对地址中某一部分操作的时候将非常方便。这样设计才算满足了数据库的第一范式。
tbl_user
id			name				gender				age				phone				province				city				address
--------------------------------------------------------------------------------------------
1				zs					男							18				12454				HeBei						TongZhou		asda
2				ls					女							25				32445				GuangDong				GuangDong		asdf

二、第二范式
维护关系表 多对多，三张表，关系表两个外键
建立在第一范式基础之上，所有非关键字字段完全依赖主键，不能产生部分依赖

第二范式在第一范式的基础之上更进一层。第二范式需要确保数据库表中的每一列都和主键相关，而不能只与主键的某一部分相关（主要针对联合主键而言）。也就是说在一个数据库表中，一个表中只能保存一种数据，不可以把多种数据保存在同一张数据库表中。比如要设计一个订单信息表，因为订单中可能会有多种商品，所以要将订单编号和商品编号作为数据库表的联合主键。
tbl_order_before
id				oid				name				num				unit				price				custumer			phone
--------------------------------------------------------------------------------------------
1					001				挖掘机				1					台						120000			 张三					12457
2					002				铲车				 2				 台					 80000				李四				 87852	

这样就产生一个问题：这个表中是以订单编号和商品编号作为联合主键。这样在该表中商品名称、单位、商品价格等信息不与该表的主键相关，而仅仅是与商品编号相关。所以在这里违反了第二范式的设计原则。

而如果把这个订单信息表进行拆分，把商品信息分离到另一个表中，把订单项目表也分离到另一个表中，就非常完美了。如下所示。
tbl_student
id					name				phone
--------------------------------
1						张三						12457
2						李四						87852


tbl_teacher
id					name					phone
--------------------------------
1						王老师					17745
2						李老师					25588

tbl_stu_teah_relation
id				s_id(fk)				t_id(fk)		
---------------------------------
1						1									1			
2						2	 				  			1
3						1									2

这样设计，在很大程度上减小了数据库的冗余。如果要获取订单的商品信息，使用商品编号到商品信息表中查询即可。


三、第三范式
建立在第二范式基础之上，所有非主键字段直接依赖主键，不能产生传递依赖。
一对多,两张表，多的表加外键
tbl_class
id		name
-----------
1			高一2班
2			高二3班
3			高一5班

tbl_student
id					name					phone					class_id(fk)
---------------------------------------------------
1						张三						12457					1
2						李四						87852					2

```



注意：有时候会使用冗余换时间



## 15 一些题目

1.取得每个部门最高薪水的人员名称

```sql
select emp.ename,t.max_sal
from (
select a.deptno,max(a.sal) as max_sal
from emp a
group by a.deptno) t
join emp
on t.deptno = emp.deptno and t.max_sal = emp.sal;
```

2、哪些人的薪水在部门平均水平之上

```sql
select emp.deptno,emp.ename,emp.sal
from emp
left join
(select a.deptno,avg(a.sal) as avg_sal
from emp a
group by a.deptno) t
on t.deptno = emp.deptno
where emp.sal > t.avg_sal;
```

3、取得部门中(所有人的)平均的薪水等级

```sql
select salgrade.grade,t.* 
from salgrade
join (
select emp.deptno,avg(sal) as avg_sal
from emp
group by emp.deptno) t
on t.avg_sal between salgrade.losal and salgrade.hisal;
```

4、不准用分组函数(MAX)取得最高薪水(给出两种方案)

```sql
select emp.sal from emp order by emp.sal desc limit 1;

select emp3.sal from emp emp3 where emp3.sal not in(
select emp1.sal from emp emp1
join emp emp2
on emp1.sal < emp2.sal);
```

5、取得平均薪水最高的部门的部门编号(给出两种方案)

```sql
select emp.deptno,avg(sal) as avg_sal
from emp
group by emp.deptno
order by avg_sal desc limit 1;


select emp.deptno,avg(sal) as avg_sal
from emp
group by emp.deptno
having avg_sal =(
select max(t.avg_sal) from
(select emp.deptno,avg(sal) as avg_sal
from emp
group by emp.deptno) t);
```

6、取得平均薪资最高部门的部门名称

```sql
select dept.dname
from dept
join (
select emp.deptno,avg(emp.sal) as avg_sal
from emp
group by emp.deptno
order by avg_sal desc limit 1) t
on dept.deptno = t.deptno;
```

7、求平均薪水的等级最低的部门的部门名称

```sql
select dept.dname,x.grade from dept 
join(
select t.deptno,salgrade.grade from(
select emp.deptno,avg(emp.sal) as avg_sal
from emp
group by emp.deptno) t
join salgrade
on t.avg_sal between salgrade.losal and salgrade.hisal
order by salgrade.grade desc) x
on dept.deptno = x.deptno
+------------+-------+
| dname      | grade |
+------------+-------+
| SALES      |     3 |
| RESEARCH   |     4 |
| ACCOUNTING |     4 |
+------------+-------+
3 rows in set (0.01 sec)
```

8、取得比普通员工(员工代码没有再mgr字段上出现的)的最高薪水还要高的领导人姓名

```sql
#这些都是普通员工
select distinct(mgr) from emp;

#比普通员工的最高工资还高，那就不是普通员工了
select emp.ename from emp
where emp.sal>(
select max(sal) from emp
where emp.empno not in
(select distinct(mgr) from emp where mgr is not null));
```



## 20 常用命令

### 20.1 查看数据库版本

```sql
select version();
```

### 20.2 退出Mysql

```sql
quit 或者 exit
```

### 20.3 退出一条语句

```sql
#SQL写着写着不想写了
\c
```

### 20.4 远程登录

```sql
mysql -h192.168.10.52 -uroot -p123456
```

### 20.5 数据导出

```sql
在windows：
mysqldump musicianclub>D:\xxx\musician.sql -uroot -p123456
```



## 21 建议

1. 字符串使用单引号括起来