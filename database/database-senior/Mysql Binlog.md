# 0-1Learning

![alt text](../../static/common/svg/luoxiaosheng.svg "公众号")
![alt text](../../static/common/svg/luoxiaosheng_learning.svg "学习")
![alt text](../../static/common/svg/luoxiaosheng_wechat.svg "微信")
![alt text](../../static/common/svg/luoxiaosheng_gitee.svg "码云")

## Mysql Binlog

MySQL 的二进制日志 binlog 可以说是 MySQL 最重要的日志，它记录了所有的 DDL 和 DML 语句（除了数据查询语句select、show等），以事件形式记录，还包含语句所执行的消耗的时间。
MySQL的二进制日志是事务安全型的。
binlog 的主要目的是复制和恢复。

### MySQL日志介绍
MySQL中一般有以下几种日志：

|日志类型|	写入日志的信息|
|---|---|
|错误日志	    |记录在启动，运行或停止mysqld时遇到的问题|
|通用查询日志	|记录建立的客户端连接和执行的语句|
|二进制日志	|记录更改数据的语句|
|中继日志	    |从复制主服务器接收的数据更改|
|慢查询日志	|记录所有执行时间超过 long_query_time 秒的所有查询或不使用索引的查询|
|DDL日志（元数据日志）	|元数据操作由DDL语句执行|

### Binlog日志的两个最重要的使用场景
* MySQL主从复制：MySQL Replication在Master端开启binlog，Master把它的二进制日志传递给slaves来达到master-slave数据一致的目的
* 数据恢复：通过使用 mysqlbinlog工具来使恢复数据

### 启用 Binlog
一般来说开启binlog日志大概会有1%的性能损耗。

启用binlog，通过配置 /etc/my.cnf 或 /etc/mysql/mysql.conf.d/mysqld.cnf 配置文件的 log-bin 选项：

在配置文件中加入 log-bin 配置，表示启用binlog，如果没有给定值，写成 log-bin=，则默认名称为主机名。（注：名称若带有小数点，则只取第一个小数点前的部分作为名称）
```
[mysqld]
log-bin=my-binlog-name
```
也可以通过 SET SQL_LOG_BIN=1 命令来启用 binlog，通过 SET SQL_LOG_BIN=0 命令停用 binlog。启用 binlog 之后须重启MySQL才能生效。

### 常用的Binlog操作命令
```
# 是否启用binlog日志
show variables like 'log_bin';

# 查看详细的日志配置信息
show global variables like '%log%';

# mysql数据存储目录
show variables like '%dir%';

# 查看binlog的目录
show global variables like "%log_bin%";

# 查看当前服务器使用的biglog文件及大小
show binary logs;

# 查看主服务器使用的biglog文件及大小

# 查看最新一个binlog日志文件名称和Position
show master status;


# 事件查询命令
# IN 'log_name' ：指定要查询的binlog文件名(不指定就是第一个binlog文件)
# FROM pos ：指定从哪个pos起始点开始查起(不指定就是从整个文件首个pos点开始算)
# LIMIT [offset,] ：偏移量(不指定就是0)
# row_count ：查询总条数(不指定就是所有行)
show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];

# 查看 binlog 内容
show binlog events;

# 查看具体一个binlog文件的内容 （in 后面为binlog的文件名）
show binlog events in 'master.000003';

# 设置binlog文件保存事件，过期删除，单位天
set global expire_log_days=3; 

# 删除当前的binlog文件
reset master; 

# 删除slave的中继日志
reset slave;

# 删除指定日期前的日志索引中binlog日志文件
purge master logs before '2019-03-09 14:00:00';

# 删除指定日志文件
purge master logs to 'master.000003';
```

### 写 Binlog 的时机

对支持事务的引擎如InnoDB而言，必须要提交了事务才会记录binlog。binlog 什么时候刷新到磁盘跟参数 sync_binlog 相关。

* 如果设置为0，则表示MySQL不控制binlog的刷新，由文件系统去控制它缓存的刷新；
* 如果设置为不为0的值，则表示每 sync_binlog 次事务，MySQL调用文件系统的刷新操作刷新binlog到磁盘中。
* 设为1是最安全的，在系统故障时最多丢失一个事务的更新，但是会对性能有所影响。

如果 sync_binlog=0 或 sync_binlog大于1，当发生电源故障或操作系统崩溃时，可能有一部分已提交但其binlog未被同步到磁盘的事务会被丢失，恢复程序将无法恢复这部分事务。

在MySQL 5.7.7之前，默认值 sync_binlog 是0，MySQL 5.7.7和更高版本使用默认值1，这是最安全的选择。一般情况下会设置为100或者0，牺牲一定的一致性来获取更好的性能。

### Binlog 文件以及扩展
binlog日志包括两类文件:
* 二进制日志索引文件（文件名后缀为.index）用于记录所有有效的的二进制文件
* 二进制日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML语句事件

binlog是一个二进制文件集合，每个binlog文件以一个4字节的魔数开头，接着是一组Events:
* 魔数：0xfe62696e对应的是0xfebin；
* Event：每个Event包含header和data两个部分；header提供了Event的创建时间，哪个服务器等信息，data部分提供的是针对该Event的具体信息，如具体数据的修改；
* 第一个Event用于描述binlog文件的格式版本，这个格式就是event写入binlog文件的格式；
* 其余的Event按照第一个Event的格式版本写入；
* 最后一个Event用于说明下一个binlog文件；
* binlog的索引文件是一个文本文件，其中内容为当前的binlog文件列表

当遇到以下3种情况时，MySQL会重新生成一个新的日志文件，文件序号递增：
* MySQL服务器停止或重启时
* 使用 flush logs 命令；
* 当 binlog 文件大小超过 max_binlog_size 变量的值时；

>max_binlog_size 的最小值是4096字节，最大值和默认值是 1GB (1073741824字节)。事务被写入到binlog的一个块中，所以它不会在几个二进制日志之间被拆分。因此，如果你有很大的事务，为了保证事务的完整性，不可能做切换日志的动作，只能将该事务的日志都记录到当前日志文件中，直到事务结束，你可能会看到binlog文件大于 max_binlog_size 的情况。

### Binlog 的日志格式
记录在二进制日志中的事件的格式取决于二进制记录格式。支持三种格式类型：

* STATEMENT：基于SQL语句的复制（statement-based replication, SBR）
* ROW：基于行的复制（row-based replication, RBR）
* MIXED：混合模式复制（mixed-based replication, MBR）

在 MySQL 5.7.7 之前，默认的格式是 STATEMENT，在 MySQL 5.7.7 及更高版本中，默认值是 ROW。
日志格式通过 binlog-format 指定，如 binlog-format=STATEMENT、binlog-format=ROW、binlog-format=MIXED。

```
Statement
每一条会修改数据的sql都会记录在binlog中
优点：不需要记录每一行的变化，减少了binlog日志量，节约了IO, 提高了性能。
缺点：由于记录的只是执行语句，为了这些语句能在slave上正确运行，因此还必须记录每条语句在执行的时候的一些相关信息，以保证所有语句能在slave得到和在master端执行的时候相同的结果。另外mysql的复制，像一些特定函数的功能，slave与master要保持一致会有很多相关问题。

Row
5.1.5版本的MySQL才开始支持 row level 的复制,它不记录sql语句上下文相关信息，仅保存哪条记录被修改。
优点： binlog中可以不记录执行的sql语句的上下文相关的信息，仅需要记录那一条记录被修改成什么了。所以row的日志内容会非常清楚的记录下每一行数据修改的细节。而且不会出现某些特定情况下的存储过程，或function，以及trigger的调用和触发无法被正确复制的问题.
缺点:所有的执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，这样可能会产生大量的日志内容。

注：将二进制日志格式设置为ROW时，有些更改仍然使用基于语句的格式，包括所有DDL语句，例如CREATE TABLE， ALTER TABLE，或 DROP TABLE。

Mixed
从5.1.8版本开始，MySQL提供了Mixed格式，实际上就是Statement与Row的结合。
在Mixed模式下，一般的语句修改使用statment格式保存binlog，如一些函数，statement无法完成主从复制的操作，则采用row格式保存binlog，MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选择一种。
```

### mysqlbinlog 命令的使用
服务器以二进制格式将binlog日志写入binlog文件，如何要以文本格式显示其内容，可以使用 mysqlbinlog 命令。
```
# mysqlbinlog 的执行格式
mysqlbinlog [options] log_file ...

# 查看bin-log二进制文件（shell方式）
mysqlbinlog -v --base64-output=decode-rows /var/lib/mysql/master.000003

# 查看bin-log二进制文件（带查询条件）
mysqlbinlog -v --base64-output=decode-rows /var/lib/mysql/master.000003 \
    --start-datetime="2019-03-01 00:00:00"  \
    --stop-datetime="2019-03-10 00:00:00"   \
    --start-position="5000"    \
    --stop-position="20000"
```

截取一段进行分析：
```
# at 21019
#190308 10:10:09 server id 1  end_log_pos 21094 CRC32 0x7a405abc 	Query	thread_id=113	exec_time=0	error_code=0
SET TIMESTAMP=1552011009/*!*/;
BEGIN
/*!*/;
```
上面输出包括信息：
* position: 位于文件中的位置，即第一行的（# at 21019）,说明该事件记录从文件第21019个字节开始
* timestamp: 事件发生的时间戳，即第二行的（#190308 10:10:09）
* server id: 服务器标识（1）
* end_log_pos 表示下一个事件开始的位置（即当前事件的结束位置+1）
* thread_id: 执行该事件的线程id （thread_id=113）
* exec_time: 事件执行的花费时间
* error_code: 错误码，0意味着没有发生错误
* type:事件类型Query


### Binlog 事件类型
binlog 事件的结构主要有3个版本：

v1: 在 MySQL 3.23 中使用
v3: 在 MySQL 4.0.2 到 4.1 中使用
v4: 在 MySQL 5.0 及以上版本中使用
现在一般不会使用MySQL5.0以下版本，所以下面仅介绍v4版本的binlog事件类型。binlog 的事件类型较多，本文在此做一些简单的汇总

|事件类型	|说明|
|---|---|

UNKNOWN_EVENT	此事件从不会被触发，也不会被写入binlog中；发生在当读取binlog时，不能被识别其他任何事件，那被视为UNKNOWN_EVENT
START_EVENT_V3	每个binlog文件开始的时候写入的事件，此事件被用在MySQL3.23 – 4.1，MYSQL5.0以后已经被 FORMAT_DESCRIPTION_EVENT 取代
QUERY_EVENT	执行更新语句时会生成此事件，包括：create，insert，update，delete；
STOP_EVENT	当mysqld停止时生成此事件
ROTATE_EVENT	当mysqld切换到新的binlog文件生成此事件，切换到新的binlog文件可以通过执行flush logs命令或者binlog文件大于 max_binlog_size 参数配置的大小；
INTVAR_EVENT	当sql语句中使用了AUTO_INCREMENT的字段或者LAST_INSERT_ID()函数；此事件没有被用在binlog_format为ROW模式的情况下
LOAD_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL 3.23版本中使用
SLAVE_EVENT	未使用
CREATE_FILE_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL4.0和4.1版本中使用
APPEND_BLOCK_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL4.0版本中使用
EXEC_LOAD_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL4.0和4.1版本中使用
DELETE_FILE_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL4.0版本中使用
NEW_LOAD_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL4.0和4.1版本中使用
RAND_EVENT	执行包含RAND()函数的语句产生此事件，此事件没有被用在binlog_format为ROW模式的情况下
USER_VAR_EVENT	执行包含了用户变量的语句产生此事件，此事件没有被用在binlog_format为ROW模式的情况下
FORMAT_DESCRIPTION_EVENT	描述事件，被写在每个binlog文件的开始位置，用在MySQL5.0以后的版本中，代替了START_EVENT_V3
XID_EVENT	支持XA的存储引擎才有，本地测试的数据库存储引擎是innodb，所有上面出现了XID_EVENT；innodb事务提交产生了QUERY_EVENT的BEGIN声明，QUERY_EVENT以及COMMIT声明，如果是myIsam存储引擎也会有BEGIN和COMMIT声明，只是COMMIT类型不是XID_EVENT
BEGIN_LOAD_QUERY_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL5.0版本中使用
EXECUTE_LOAD_QUERY_EVENT	执行LOAD DATA INFILE 语句时产生此事件，在MySQL5.0版本中使用
TABLE_MAP_EVENT	用在binlog_format为ROW模式下，将表的定义映射到一个数字，在行操作事件之前记录（包括：WRITE_ROWS_EVENT，UPDATE_ROWS_EVENT，DELETE_ROWS_EVENT）
PRE_GA_WRITE_ROWS_EVENT	已过期，被 WRITE_ROWS_EVENT 代替
PRE_GA_UPDATE_ROWS_EVENT	已过期，被 UPDATE_ROWS_EVENT 代替
PRE_GA_DELETE_ROWS_EVENT	已过期，被 DELETE_ROWS_EVENT 代替
WRITE_ROWS_EVENT	用在binlog_format为ROW模式下，对应 insert 操作
UPDATE_ROWS_EVENT	用在binlog_format为ROW模式下，对应 update 操作
DELETE_ROWS_EVENT	用在binlog_format为ROW模式下，对应 delete 操作
INCIDENT_EVENT	主服务器发生了不正常的事件，通知从服务器并告知可能会导致数据处于不一致的状态
HEARTBEAT_LOG_EVENT	主服务器告诉从服务器，主服务器还活着，不写入到日志文件中


### Binlog 事件的结构
一个事件对象分为事件头和事件体，事件的结构如下：
```
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    |
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
|        +----------------------------+
|        | extra_headers    19 : x-19 |
+=====================================+
| event  | fixed part        x : y    |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```
如果事件头的长度是 x 字节，那么事件体的长度为 (event_length - x) 字节；设事件体中 fixed part 的长度为 y 字节，那么 variable part 的长度为 (event_length - (x + y)) 字节






