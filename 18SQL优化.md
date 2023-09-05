# 第十八章 SQL 优化

## 18.1.1通过show status命令了解各种SQL的执行频率

MySQL 客户端连接成功后，通过 show [session|global]status 命令可以提供服务器状态信息，也可以在操作系统上使用 mysqladmin extended-status 命令获得这些消息。show[session|global] status 可以根据需要加上参数“session”或者“global”来显示 session 级（当前连接）的统计结果和global 级（自数据库上次启动至今）的统计结果。如果不写，默认使用参数是“session”。

`show status like 'Com_%';`

## 18.1.2 定位执行效率较低的SQL语句

- 通过慢查询日志定位那些执行效率较低的 SQL 语句，用--log-slow-queries[=file_name]选项启动时，mysqld 写一个包含所有执行时间超过 long_query_time 秒的 SQL 语句的日志文件
- 慢查询日志在查询结束以后才纪录，所以在应用反映执行效率出现问题的时候查询慢查询日志并不能定位问题，可以使用show processlist命令查看当前MySQL在进行的线程，包括线程的状态、是否锁表等，可以实时地查看 SQL 的执行情况，同时对一些锁表操作进行优化。

要开启MySQL慢查询日志，您可以按照以下步骤进行操作：

1. 打开MySQL的配置文件 `my.cnf` 或者 `my.ini`，这个文件一般位于MySQL安装目录下的 `etc` 子目录中。

2. 在配置文件中找到 `[mysqld]` 标签，如果没有则自行添加该标签。在该标签下添加以下内容：

   ```ini
   slow_query_log = 1       # 开启慢查询日志
   slow_query_log_file = /path/to/slow-query.log  # 指定慢查询日志文件路径（替换为实际路径）
   long_query_time = 1      # 定义慢查询的阈值，单位为秒（根据实际情况调整）
   ```

   注意：`/path/to/slow-query.log` 可替换为您希望存储慢查询日志的文件路径和名称。

3. 保存并关闭配置文件。

4. 重启MySQL服务，以使配置生效。

之后，MySQL将开始记录执行时间超过定义的 `long_query_time` 阈值的查询到指定的慢查询日志文件中。您可以根据需要自行调整 `long_query_time` 的值，以更精确地控制慢查询的定义和记录。

## 18.1.3 通过EXPLAIN分析低效SQL的执行计划

通过以上步骤查询到效率低的 SQL 语句后，可以通过 EXPLAIN或者 DESC命令获取 MySQL如何执行 SELECT 语句的信息，包括在 SELECT 语句执行过程中表如何连接和连接的顺序，比如想计算 2006 年所有公司的销售额，需要关联 sales 表和 company 表，并且对 moneys 字段做求和（sum）操作，相应 SQL 的执行计划如下：

```sql
mysql> explain select sum(moneys) from sales a,company b where a.company_id = b.id and a.year
= 2006\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: a
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
*************************** 2. row ***************************
id: 1
select_type: SIMPLE
table: b
type: ref
possible_keys: ind_company_id
key: ind_company_id
key_len: 5
ref: sakila.a.company_id
rows: 1
Extra: Using where; Using index
2 rows in set (0.00 sec)
```



- select_type：表示 SELECT 的类型，常见的取值有 SIMPLE（简单表，即不使用表连接或者子查询）、PRIMARY（主查询，即外层的查询）、UNION（UNION 中的第二个或者后面的查询语句）、SUBQUERY（子查询中的第一个 SELECT）等。
- table：输出结果集的表
- type：表示表的连接类型，性能由好到差的连接类型为 system（表中仅有一行，即常量表）、const（单表中最多有一个匹配行，例如 primary key 或者 unique index）、eq_ref（对于前面的每一行，在此表中只查询一条记录，简单来说，就是多表连接中使用primary key或者unique index）、ref （与eq_ref类似，区别在于不是使用primary
  key 或者 unique index，而是使用普通的索引）、ref_or_null（与 ref 类似，区别在于条件中包含对 NULL 的查询）、index_merge(索引合并优化)、unique_subquery（in的后面是一个查询主键字段的子查询）、index_subquery （与 unique_subquery 类似，区别在于 in 的后面是查询非唯一索引字段的子查询）、range（单表中的范围查询）、index（对于前面的每一行，都通过查询索引来得到数据）、all （对于前面的每一行，都通过全表扫描来得到数据）。
- possible_keys：表示查询时，可能使用的索引。
- key：表示实际使用的索引。
- key_len：索引字段的长度。
- rows：扫描行的数量。
- Extra：执行情况的说明和描述。

## 18.1.4 确定问题并采取响应的优化措施

创建索引



## 18.2.1 索引的存储分类

InnoDB存储引擎支持以下几种类型的索引：

1. B+ 树索引：B+ 树是 InnoDB 默认的索引类型，用于支持常规的等值查询、范围查询和排序操作。

2. 主键索引：每个表只能有一个主键，主键索引在 InnoDB 中是唯一的索引方式。如果没有显式定义主键，则会自动生成一个隐藏的主键索引。

3. 唯一索引：唯一索引要求索引列的值都是唯一的，可以用来保证数据的唯一性。一个表可以有多个唯一索引。

4. 外键索引：外键索引用于建立表与表之间的关联关系，保证数据的完整性。它是基于外键约束来实现的，对应的索引可以加速关联查询。

5. 全文索引：全文索引用于全文搜索，可以加速针对文本内容的模糊匹配。InnoDB 支持全文索引，但需要使用特定的全文索引类型，如 `FULLTEXT`。

6. 组合索引：组合索引是将多个列组合在一起创建的索引，可以提供多列的查询条件。通过合理设计组合索引，可以提高查询效率。

注意，InnoDB 存储引擎的默认索引类型是 B+ 树索引，而不支持 Hash 索引。选择合适的索引类型要根据具体的数据特点和查询需求进行评估和选择。

## 18.2.2 使用索引

在 MySQL 中，下列几种情况下有可能使用到索引,对于创建的多列索引，只要查询的条件中用到了最左边的列，索引一般就会被使用，举例说明如下。

首先按 company_id，moneys 的顺序创建一个复合索引，具体如下：

```sql
create index ind_sales2_companyid_moneys on sales2(company_id,moneys);
```

然后按 company_id 进行表查询，具体如下：

```sql
explain select * from sales2 where company_id = 2006\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: sales2
type: ref
possible_keys: ind_sales2_companyid_moneys
key: ind_sales2_companyid_moneys
key_len: 5
ref: const
rows: 1
Extra: Using where
1 row in set (0.00 sec)
```

可以发现即便 where 条件中不是用的 company_id 与 moneys 的组合条件，索引仍然能用到，这就是索引的前缀特性。但是如果只按 moneys 条件查询表，那么索引就不会被用到，具体如下：

```sql
explain select * from sales2 where moneys = 1\G;
```

对于使用 like 的查询，后面如果是常量并且只有％号不在第一个字符，索引才可能会被使用

```sql
explain select * from company2 where name like '%3'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: ALL
possible_keys: NULL
key: NULL
key_len: NULL
ref: NULL
rows: 1000
Extra: Using where
1 row in set (0.00 sec)
```

```sql
explain select * from company2 where name like '3%'\G;
*************************** 1. row ***************************
id: 1
select_type: SIMPLE
table: company2
type: range
possible_keys: ind_company2_name
key: ind_company2_name
key_len: 11
ref: NULL
rows: 103
Extra: Using where
1 row in set (0.00 sec)
```

可以发现第一个例子没有使用索引，而第二例子就能够使用索引，区别就在于“%”的位置不同，前者把“%”放到第一位就不能用到索引，而后者没有放到第一位就使用了索引。另外，如果如果 like 后面跟的是一个列的名字，那么索引也不会被使用

### 存在索引但不使用索引

在下列情况下，虽然存在索引，但是 MySQL 并不会使用相应的索引。

- 如果 MySQL 估计使用索引比全表扫描更慢，则不使用索引。例如如果列key_part1 均匀分布在 1 和 100 之间，下列查询中使用索引就不是很好

​		```SELECT * FROM table_name where key_part1 > 1 and key_part1 < 90```;

- 如果使用 MEMORY/HEAP 表并且 where 条件中不使用“=”进行索引列，那么不会用到索引。heap 表只有在“=”的条件下才会使用索引
- 用 or分割开的条件，如果 or前的条件中的列有索引，而后面的列中没有索引，那么涉及到的索引都不会被用到
- 如果列类型是字符串，那么一定记得在 where 条件中把字符常量值用引号引起来，否则的话即便这个列上有索引，MySQL 也不会用到的，因为，MySQL 默认把输入的常量值进行转换以后才进行检索。如下面的例子中company2表中的name字段是字符型的，但是 SQL 语句中的条件值 294 是一个数值型值，因此即便在 name 上有索引，MySQL 也不能正确地用上索引，而是继续进行全表扫描

## 18.2.3 查看索引使用情况

如果索引正在工作，Handler_read_key 的值将很高，这个值代表了一个行被索引值读的次数，很低的值表明增加索引得到的性能改善不高，因为索引并不经常使用。Handler_read_rnd_next 的值高则意味着查询运行低效，并且应该建立索引补救。这个值的含义是在数据文件中读下一行的请求数。如果正进行大量的表扫描，Handler_read_rnd_next 的值较高，则通常说明表索引不正确或写入的查询没有利用索引，具体如下

```sql
show status like 'Handler_read%';
+-----------------------+-------+
| Variable_name | Value |
+-----------------------+-------+
| Handler_read_first | 0 |
| Handler_read_key | 5 |
| Handler_read_next | 0 |
| Handler_read_prev | 0 |
| Handler_read_rnd | 0 |
| Handler_read_rnd_next | 2055 |
+-----------------------+-------+
6 rows in set (0.00 sec)
```

## 18.3 两个简单实用的优化方法

### 18.3.1 定期分析表和检查表

分析表的语法如下:

```sql
ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ..
```

### 18.3.2 定期优化表

优化表的语法如下：

```sql
OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
```

## 18.4 常用的SQL优化

### 18.4.1 大批量插入数据

当用 load 命令导入数据的时候，适当的设置可以提高导入的速度。
对于 MyISAM 存储引擎的表，可以通过以下方式快速的导入大量的数据

```sql
ALTER TABLE tbl_name DISABLE KEYS;
loading the data
ALTER TABLE tbl_name ENABLE KEYS;
```

DISABLE KEYS 和 ENABLE KEYS 用来打开或者关闭 MyISAM 表非唯一索引的更新。在导入大量的数据到一个非空的 MyISAM 表时，通过设置这两个命令，可以提高导入的效率。对于导入大量数据到一个空的 MyISAM 表，默认就是先导入数据然后才创建索引的，所以不用进行设置。
下面例子中，用 LOAD 语句导入数据耗时 115.12 秒：

```sql
mysql> load data infile '/home/mysql/film_test.txt' into table film_test2;
Query OK, 529056 rows affected (1 min 55.12 sec)
Records: 529056 Deleted: 0 Skipped: 0 Warnings: 0
```

而用 alter table tbl_name disable keys 方式总耗时 6.34 + 12.25 = 18.59 秒，提高了 6 倍多。

```sql
mysql> alter table film_test2 disable keys;
Query OK, 0 rows affected (0.00 sec)
216
mysql> load data infile '/home/mysql/film_test.txt' into table film_test2;
Query OK, 529056 rows affected (6.34 sec)
Records: 529056 Deleted: 0 Skipped: 0 Warnings: 0
mysql> alter table film_test2 enable keys;
Query OK, 0 rows affected (12.25 sec)
```

上述优化只能用于MyISAM引擎



因为 InnoDB 类型的表是按照主键的顺序保存的，所以将导入的数据按照主键的顺序排列，可以有效地提高导入数据的效率。例如，下面文本film_test3.txt是按表film_test4的主键存储的，那么导入的时候共耗时
27.92秒。

```sql
mysql> load data infile '/home/mysql/film_test3.txt' into table film_test4;
Query OK, 1587168 rows affected (22.92 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
```

而下面的 film_test4.txt 是没有任何顺序的文本，那么导入的时候共耗时 31.16 秒

```sql
mysql> load data infile '/home/mysql/film_test4.txt' into table film_test4;
Query OK, 1587168 rows affected (31.16 sec)
Records: 1587168 Deleted: 0 Skipped: 0 Warnings: 0
```

- 当被导入的文件按表主键顺序存储的时候比不按主键顺序存储的时候
  快 1.12 倍

- 在导入数据前关闭唯一性校验,导入后恢复唯一性校验可以提高导入的效率

  ```sql
  SET UNIQUE_CHECKS=0 --关闭唯一性校验
  SET UNIQUE_CHECKS=1 --开启唯一性校验
  ```

- 如果应用使用自动提交的方式，建议在导入前执行 SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行 SET AUTOCOMMIT=1，打开自动提交，也可以提高导入的效率

### 18.4.2 优化insert语句

- 如果同时从同一客户插入很多行，尽量使用多个值表的 INSERT 语句，这种方式将大大缩减客户端与数据库之间的连接、关闭等消耗，使得效率比分开执行的单个 INSERT 语句快(在一些情况中几倍)

​		```insert into test values(1,2),(1,3),(1,4)…``

- 如果从不同客户插入很多行，能通过使用 INSERT DELAYED 语句得到更高的速度。DELAYED 的含义是让 INSERT 语句马上执行，其实数据都被放在内存的队列中，并没有真正写入磁盘，这比每条语句分别插入要快的多；LOW_PRIORITY 刚好相反，在所有其他用户对表的读写完后才进行插入
- 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）；
- 如果进行批量插入，可以增加 bulk_insert_buffer_size 变量值的方法来提高速度，但是，这只能对 MyISAM 表使用
- 当从一个文本文件装载一个表时，使用 LOAD DATA INFILE。这通常比使用很多 INSERT 语句快 20 倍