# MySQL日志

## 错误日志

错误日志会记录如下信息

- mysql执行过程中的错误信息
- mysql执行过程中的告警信息
- event scheduler运行时所产生信息
- mysq启动和停止过程中产生的信息
- 主从复制结构中，重从服务器IO复杂线程的启动信息

查看错误日志的位置，和告警信息

```SQL
show variables where 
variable_name="log_error" or 
variable_name="log_warnings"

+---------------+------------------------------------------------------+
| Variable_name | Value                                                |
+---------------+------------------------------------------------------+
| log_error     | /var/log/mysqld.log                                  |
| log_warnings  | 2                                                    |
+---------------+------------------------------------------------------+
2 rows in set (0.001 sec)
```

`log_error`是错误日志的位置，`log_warnings`表示是否记录告警信息到错误日志，0表示不记录告警信息，1表示记录告警信息，大于1表示各类告警信息

## 一般查询日志

查询日志分为一般查询日志和慢查询日志，它们是通过查询是否超出变量 `long_query_time` 指定时间的值来判定的。在超时时间内完成的查询是一般查询，可以将其记录到一般查询日志中；超出时间的查询是慢查询，可以将其记录到慢查询日志中。

一般查询日志记录了数据库执行的命令，不管这些语法是否正确，都会被记录。由于数据库操作命令非常多而且比较频繁，所以开启了查询日志以后，数据库可能需要不停的写入查询，这样会增大服务器的IO压力，增加很多的系统开销，所以默认情况下，mysql的一般查询日志是没有开启的。

```SQL
show variables where 
variable_name = "long_query_time" or
variable_name like "%general_log%" or 
variable_name = "log_output";

+------------------+---------------------------------------------------+
| Variable_name    | Value                                             |
+------------------+---------------------------------------------------+
| general_log      | OFF                                               |
| general_log_file | /usr/local/mysql/data/iZbp1ja865ota0u0aap1jmZ.log |
| log_output       | FILE                                              |
| long_query_time  | 10.000000                                         |
+------------------+---------------------------------------------------+　　
```

- `general_log`表示查询日志是否开启，`ON`表示开启，`OFF`表示未开启，默认`OFF`。使用命令开启一般查询日志：

  ```SQL
  set global general_log=on;
  ```

- `log_output`表示当查询日志开启以后，以哪种方式存放。`FILE`表示存放于文件中，`TABLE`表示存放于表mysql.general_log中，`FILE,TABLE`表示同时存放于文件和表中, `NONE`表示不记录日志。`log_output`不仅控制查询日志，还控制慢查询日志。

- `general_log_file`表示查询日志存放于文件的路径

- `long_query_time`为判定慢查询的时间

## 慢查询日志

查询超出变量`long_query_time`指定时间的为慢查询，默认为10秒。但是查询获取锁，包括锁等待的时间不计入查询时间内。慢查询日志是在查询执行完毕且已经完全释放锁之后才记录的，因此慢查询日志记录的顺序和执行的SQL查询语句顺序可能会不一致。例如，语句1先执行，查询速度慢；语句2后执行，但查询速度快，则语句2先记录。

```SQL
show variables where 
variable_name like "slow_query%" or 
variable_name = "log_queries_not_using_indexes";

+-------------------------------+------------------------------+
| Variable_name                 | Value                        |
+-------------------------------+------------------------------+
| log_queries_not_using_indexes | OFF                          |
| slow_query_log                | ON                           |
| slow_query_log_file           | /usr/local/mysql/data/iZbp1j |
|                               | a865ota0u0aap1jmZ-slow.log   |
+-------------------------------+------------------------------+
```

- `slow_query_log`表示查询日志是否开启，`ON`表示开启，`OFF`表示未开启，默认`OFF`.  使用命令开启慢查询日志：

  ```sql 
   set global slow_query_log=1;
  ```

- `slow_query_log_file`表示查询日志存放于文件的路径

- `log_queries_not_using_indexes`表示如果运行的sql语句没有使用到索引，也被记录到慢查询日志，`OFF`表示不记录，`ON`表示记录，默认`OFF`

## 二进制日志

二进制日志是一个二进制文件，记录了对MySQL数据库执行更改的所有操作，语句以事件的形式记录，所以包括了发生时间、执行时长、操作数据等信息。但是他不记录SELECT、SHOW等那些不改变数据库的SQL语句。二进制日志主要用于数据库恢复和主从复制，以及审计操作。所以出于安全和功能考虑，极不建议将二进制日志和数据目录放在同一磁盘上。

对于事务表的操作，二进制日志只在事务提交的时候一次性写入，提交前的每个二进制日志记录都先cache，提交时写入。所以对于事务表来说，一个事务中可能包含多条二进制日志事件，它们会在提交时一次性写入。而对于非事务表的操作，每次执行完语句就直接写入。

### 相关变量

```SQL
show variables where
variable_name="log_bin" or
variable_name="log_bin_basename" or
variable_name="max_binlog_size" or
variable_name="log_bin_index" or
variable_name="binlog_format" or
variable_name="sql_log_bin" or
variable_name="sync_binlog";

+------------------+------------+
| Variable_name    | Value      |
+------------------+------------+
| binlog_format    | ROW        |
| log_bin          | OFF        |
| log_bin_basename |            |
| log_bin_index    |            |
| max_binlog_size  | 1073741824 |
| sql_log_bin      | ON         |
| sync_binlog      | 1          |
+------------------+------------+
```

- `log_bin`表示二进制日志是否开启，`ON`表示开启，`OFF`表示未开启，默认`OFF`
- `log_bin_basename`: 二进制日志文件前缀名，二进制日志就记录在该文件中
- `max_binlog_size`: 二进制日志文件的最大大小，超过此大小，二进制文件会自动滚动
- `log_bin_index`: 二进制日志文件索引文件名，用于记录所有的二进制文件
- `binlog_format`：决定了二进制日志的记录方式，`STATEMENT`以语句的形式记录，`ROW`以数据修改的形式记录，`MIXED`以语句和数据修改混合形式记录
- `sql_log_bin`：决定是否对二进制日志进行日志记录，`ON`表示执行记录，`OFF`表示不执行记录，默认`OFF`，这个是会话级别变量可以通`SET sql_log_bin = {0|1}`来改变该变量值
- `sync_binlog`：决定二进制日志写入磁盘时机，如果sync_binlog为0，操作系统来决定什么时候写入磁盘，如果sync_binlog为N（N=1,2,3..），则每N次事务提交后，都立即将内存中的二进制日志写入磁盘，如何选择取决于安全性与性能的权衡

### 记录方式

二进制日志的记录方式有3种：

- STATEMENT：记录对数据库做出修改的语句。比如，update A set test='test'，如果使用statement模式，那么这条update语句将被记录到二进制日志中。使用statement模式，优点是binlog日志量少，IO压力小，性能高，缺点是为了尽可能一致的还原操作，除了记录语句本身外，可能还需要记录一些相关信息，而且，在使用一些特定函数时，并不能保证恢复操作与记录完全一致
- ROW：记录对数据库做出的修改的语句所影响到的数据行以及这些行的修改。比如，update A set test='test'，如果使用row模式，那么这条update所影响到的每一行每一列的值都会记录在binlog中。使用row模式时，优点是能完还原和复制被日志记录时的操作，缺点是日志量较大，IO压力比较大，性能消耗比较大
- MIXED：混合上述两种模式，一般使用statement方式进行记录，如果遇到一些特殊函数使用row方式进行记录，这种记录方式称为mixed

### 查看

使用show：

```SQL
SHOW BINARY LOGS                                   # 查看使用了哪些日志文件
SHOW BINLOG EVENTS in 'mysql-bin.000005' from 961; # 查看日志中从位置961开始的日志
SHOW MASTER STATUS                                 # 显式主服务器中的二进制日志信息
```

### 恢复数据库

可以使用二进制日志恢复数据库。只需指定二进制日志的起始位置（可指定终止位置）并将其保存到sql文件中，由mysql命令来载入恢复即可。

```sh
mysqlbinlog --start-position=373 --stop-position=1871 bin_log_file.000001 > recover_file.sql
```

```SQL
#临时关闭binlog
set sql_log_bin=0;
#执行sql文件
source recover_file.sql
#开启binlog
set sql_log_bin=1;
```

