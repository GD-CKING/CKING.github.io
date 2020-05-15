---
title: MySQL 的慢查询优化方式
date: 2020-05-14 17:42:14
tags: MySQL
categories: MySQL
---

## 慢查询日志概念

MySQL 的慢查询日志是 MySQL 提供的一种日志记录，它用来记录在 MySQL 中相应时间超过阈值的语句，具体指运行时间超过 `long_query_time` 值的 SQL。则会被记录到慢查询日志中。`long_query_time` 的默认值是 10，意思是运行 10s 以上的语句。



默认情况下，MySQL 数据库并不启动慢查询日志，需要我们手动来设置这个参数。当然，如果不是调优需要的话，一般不需要启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。慢查询日志支持将日志记录写入文件，也支持将日志记录写入数据库表。



## 慢查询日志相关参数

MySQL 慢查询的相关参数解释：

- **slow_query_log**：是否开启慢查询日志，`1` 表示开启，`0` 表示关闭

- **log-slow-queries**：旧版（5.6 一下版本）MySQL 数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件 `host_name-slow.log`

- **slow-query-log-file**：新版（5.6 及以上版本）MySQL 数据库慢查询日志存储路径。可以不设置该参数，系统则会默认给一个缺省的文件 `host_name-slow.log`

- **long_query_time**：慢查询阈值，当查询时间多于设定的阈值时，记录日志

- **log_queries_not_using_indexes**：未使用索引的查询也被记录到慢查询日志中（可选项）

- **log_output**：日志存储方式。`log_output = 'FILE'` 表示将日志存入文件，默认值是 `FILE`。`log_output = 'TABLE'` 表示将日志存入数据库，这样日志信息就会被写入到 `mysql.slow_log` 表中。MySQL 数据库支持两种同时日志存储方式，配置的时候以逗号隔开即可。如：`log_output = 'FILE, TABLE'`。日志记录到系统的专用日志表中，要比记录到文件耗费更多的系统资源，因此对于需要启用慢查询日志，又需要能够获得更高的系统性能，那么建议优先记录到文件。



## MySQL 慢查询

### 开启 MySQL 慢查询

- 方式一：在 `my.ini` 增加几行：主要是慢查询的定义时间，以及慢查询 log 日志记录（slow_query_log）

- 方式二：通过 MySQL 数据库开启慢查询



### 分析慢查询日志

直接分析 MySQL 慢查询日志，利用 `explain` 关键字可以模拟优化器执行 SQL 查询语句，来分析 SQL 慢查询语句。例如：

```mysql
EXPLAIN SELECT * FROM res_user ORDER BY modifiedtime LIMIT 0, 1000
```



得到如下结果：

table | type | possible_keys | key | key_len | ref | rows | Extra



显示结果分析：

- **table**：显示这一行的数据是关于那张表的

- **type**：这是重要的列，显示连接使用了何种类型。从最好到最差的连接类型为：const、eq_reg、ref、range、index 和 all

- **rows**：显示需要扫描的行数

- **key**：使用的索引



### 常见的慢查询优化

#### 索引没起作用的情况

##### 使用 LIKE 关键字的查询语句

在使用 LIKE 关键字进行查询的查询语句中，如果匹配字符串的第一个字符为 "%"，索引不会起作用。只有 "%" 不在第一个位置索引才会起作用



##### 使用多列索引的查询语句

MySQL 可以为多个字段创建索引。一个索引最多可以包括 16 个字段。对于多列索引，需要符合最左前缀匹配原则



#### 优化数据库结构

合理的数据库结构不仅可以使数据库占用更小的磁盘空间，而且能够使查询速度更快。数据库结构的设计，需要考虑数据冗余、查询和更新的速度、字段的数据类型是否合理等多方面的内容。



##### 将字段很多的表分解成多个表

对于字段比较多的表，如果有些字段的使用频率很低，可以将这些字段分离出来形成新表。因为当一个表的数据量很大时，会由于使用频率低的字段的存在而变慢。



##### 增加中间表

对于需要经常联合查询的表，可以建立中间表以提高查询效率。通过建立中间表，把需要经常联合查询的数据插入到中间表中，然后将原来的联合查询改为对中间表的查询，一次来提高查询效率



#### 优化 LIMIT 分页

在系统中需要分页的操作通常会使用 `limit` 加上偏移量的方法实现，同时加上合适的 order by 子句。如果有对应的索引，通常效率会不错，否则 MySQL 需要做大量的文件排序操作。



一个头疼的问题就是当偏移量很大的时候，例如可能是 `limit 10000, 20` 这样的查询，这时 MySQL 需要查询 10020 条然后只返回最后 20 条，前面的 10000 条记录都将被舍弃，这样的代价很高。



优化此类查询的一个最简单的方法时尽可能地使用索引覆盖扫描，而不是扫描所有的列。然后根据需要做一次关联操作再返回所需的列，对于偏移量很大的时候这样做的效率会得到很大提升。



对于下面的查询：

```mysql
SELECT id, title FROM collect LIMIT 90000, 10 
```



该语句存在的最大问题在于 limit M, N 中偏移量 M 太大（我们暂不考虑筛选字段上要不要添加索引的影响），导致每次查询都要先从整个表中找到满足条件的前 M 条记录，之后舍弃这 M 条记录并从第 M + 1 条记录开始再依次找到 N 条满足条件的记录。



如果表非常大，且筛选字段没有合适的索引，且 M 特别大，那么这样的代价是非常高的。如果我们下一次的查询能从前一次查询结束后标记的位置开始查找，找到满足条件的 100 条记录，并记下下一次查询应该开始的位置，以便于下一次查询能直接从改位置开始，这样就不必每次查询都先从整个表中先找到满足条件的前 M 条记录，舍弃，在从 M + 1 开始再找到 100 条满足条件的记录了



方法一：先查询出主键 id 值

```mysql
SELECT id, title FROM collect WHERE id >= (SELECT id FROM collect ORDER BY id LIMIT 90000, 1) LIMIT 10;
```



原理：先查询出 90000 条数据对应的主键 id 的值，然后直接通过改 id 的值直接查询该 id 后面的数据。



方法二：“关延迟联”

如果这个表非常大，那么这个查询可以改写成如下的方式：

```mysql
SELECT news.id, news.description FROM news INNER JOIN (SELECT id FROM news ORDER BY title LIMIT 50000, 5) as myNew using(id);
```



这里的 “关延迟联” 将大大提升查询的效率，它让 MySQL 扫描尽可能少的页面，获取需要的记录后再根据关联列回原表查询需要的所有列。这个技术也可以用在优化关联查询中的 LIMIT。



方法三：建立复合索引

下面的 SQL 语句

```mysql
SELECT * FROM acct_trans_log WHERE acct_id = 3095 ORDER BY create_time DESC LIMIT 0, 10
```



可以通过建立复合索引 acct_id 和 create_time 进行优化



## 日志分析工具 mysqldumpslow

在生产环境中，如果要手工分析日志，查找、分析 SQL，显然是个体力活，MySQL 提供了日志分析工具 mysqldumpslow



查看 mysqldumpslow 的帮助信息：

```verilog
Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]
Parse and summarize the MySQL slow query log. Options are
--verbose   verbose
--help       write this text to standard output
-v           verbose
-s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
              ar: average rows sent
                c: count
                r: rows sent
-r           reverse the sort order (largest last instead of first)
-a           don't abstract all numbers to N and strings to 'S'
-g PATTERN   grep: only consider stmts that include this string
              default is '*', i.e. match all
-l           don't subtract lock time from total time
```



-s，是表示按照何种方式排序，

> c：访问计数
>
> l：锁定时间
>
> r：返回记录
>
> t：查询时间
>
> al：平均锁定时间
>
> ar：平均返回记录数
>
> at：平均查询时间



-t，是 top n 的意思，即为返回前面多少条的数据；



-g，后边可以写一个正则匹配模式，大小写不敏感的；



比如，得到返回记录集最多的 10 个 SQL

```sh
mysqldumpslow -s r -t 10 /database/mysql/mysql06_slow.log
```



得到访问次数最多的 10 个 SQL：

```sh
mysqldumpslow -s c -t 10 /database/mysql/mysql06_slow.log
```



得到按照时间排序的前 10 条里面含有左连接的查询语句：

```sh
mysqldumpslow -s t -t 10 -g "left join" /database/mysql/mysql06_slow.log
```



另外建议在使用这些命令时结合 | 和 more 使用，否则有可能出现刷屏的情况

```sh
mysqldumpslow -s r -t 20 /mysqldata/mysql/mysql06_slow.log | more
```



## 参考资料

[常见 MySQL 的慢查询优化方式](https://mp.weixin.qq.com/s/1r6lFQE4pxeo0zsV3yVkXg)

