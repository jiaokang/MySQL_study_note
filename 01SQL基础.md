# SQL分类

SQL语句主要分为下属三个类别

## DDL 

DDL（Data Definition Languages）语句：数据定义语言，这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter等.

## DML

DML（Data Manipulation Language）语句 ：数据操纵语句，用于添加、删除、更新和查
询数据库记录，并检查数据完整性，常用的语句关键字主要包括 insert、delete、udpate 和
select 等。

## DCL

DCL（Data Control Language）语句：数据控制语句，用于控制不同数据段直接的许可和
访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。主要的
语句关键字包括 grant、revoke 等。



## 创建数据库

### 链接

启动MySQL服务以后,输入以下命令可以用服务自带的客户端访问数据库

`[mysql@db3 ~]$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor. Commands end with ; or \g.
Your MySQL connection id is 7344941 to server version: 5.1.9-beta-log
Type 'help;' or '\h' for help. Type '\c' to clear the buffer.
mysql>`

<font color = red>在以上命令行中，mysql 代表客户端命令，-u 后面跟连接的数据库用户，-p 表示需要输入密
码</font>

### 创建库

`create database test1`

### 查看系统中存在哪些库

`show databases;`

**information_schema**:主要存储了系统中的一些数据库对象信息。比如用户表信息、列信息、权限信息、字符集信息、分区信息等

**mysql**:存储了系统的用户权限信息

### 切换到要操作的数据库

`use daname;`

### 查看当前数据库中的所有数据表

`show tables;`

### 删除数据库

`drop database dbname;`

### 创建表

`create table emp(ename varchar(10), sal decimal(10,2));`

列名 数据类型

### 查看表

`desc tablename`

虽然 desc 命令可以查看表定义，但是其输出的信息还是不够全面，为了查看更全面的表定义信息，有时就需要通过查看创建表的 SQL 语句来得到，可以使用如下命令实现：

`show create table tablename \G`

在MySQL中，`\G`是一种特殊的用法，用于替代分号`;`来结束SQL语句并以更易读的方式显示查询结果。

**当使用`\G`替代分号时，在执行`SHOW CREATE TABLE`语句时，会以垂直格式显示结果。这样可以使得输出更加清晰易读，并且每个字段的信息都单独占据一行，方便查看和分析。所以，`SHOW CREATE TABLE emp \G;`语句将以垂直格式展示`emp`表的创建语句和相关信息。**

### 删除表

`drop table tablename`

### 修改表

在大多数情况下，表结构的更改一般都使用 alter table 语句，以下是一些常用的命令

**修改表类型**

`ALTER TABLE tablename MODIFY [COLUMN] column_definition [FIRST | AFTER col_name]`



修改表 emp 的 ename 字段定义，将 varchar(10)改为 varchar(20)：

`alter table emp modify ename varchar(20);`



**增加表字段**

`ALTER TABLE tablename ADD [COLUMN] column_definition [FIRST | AFTER col_name]`

表 emp 上新增加字段 age，类型为 int(3)：

`alter table emp add column age int(3);`

**删除表字段**

`ALTER TABLE tablename DROP [COLUMN] col_name`

将字段 age 删除掉

` alter table emp drop column age;`

**字段改名**

`ALTER TABLE tablename CHANGE [COLUMN] old_col_name column_definition
[FIRST|AFTER col_name]`

将 age 改名为 age1，同时修改字段类型为 int(4)：

`alter table emp change age age1 int(4) ;`

<font color=red>注意：change 和 modify都可以修改表的定义，不同的是 change 后面需要写两次列名，不方便。
但是 change 的优点是可以修改列名称，modify则不能。</font>

**修改字段排列顺序**

前面介绍的的字段增加和修改语法（ADD/CNAHGE/MODIFY）中，都有一个可选项 first|after
column_name，这个选项可以用来修改字段在表中的位置，默认 ADD 增加的新字段是加在
表的最后位置，而 CHANGE/MODIFY 默认都不会改变字段的位置。
例如，将新增的字段 birth date 加在 ename 之后：

`alter table emp add birth date after ename;`

修改字段 age，将它放在最前面：

`alter table emp modify age int(3) first;`

<font color=red>注意：CHANGE/FIRST|AFTER COLUMN 这些关键字都属于 MySQL 在标准 SQL 上的扩展，在
其他数据库上不一定适用。</font>

**修改表名**

`ALTER TABLE tablename RENAME [TO] new_tablename`

将表 emp 改名为 emp1

`alter table emp rename emp1;`

## DML语句

DML 操作是指对数据库中表记录的操作，主要包括表记录的插入（insert）、更新（update）、删除（delete）和查询（select），是开发人员日常使用最频繁的操作。下面将依次对它们进行介绍。

### 插入记录

