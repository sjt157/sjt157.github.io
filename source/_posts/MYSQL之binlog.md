---
title: MYSQL之binlog
date: 2019-01-18 20:58:21
tags: Mysql
categories: Mysql
---


### 什么是binlog
binlog日志用于记录所有更新了数据或者已经潜在更新了数据（例如，没有匹配任何行的一个DELETE）的所有语句。语句以“事件”的形式保存，它描述数据更改。

### binlog作用
因为有了数据更新的binlog，所以可以用于实时备份，与master/slave主从复制结合。

### binlog的形式
二进制日志包括两类文件： 
二进制日志索引文件（文件名后缀为.index）用于记录所有的二进制文件； 
二进制日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML(除了数据查询语句)语句事件。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/MYSQL_BinLog/1.png)

### 命令
show variables like '%log_bin%'；
show variables like 'log_bin';
show variables like '%general_log%';
show variables like '%log_%'; 

### 如何开启binlog
在my.cnf中加入以下内容（加入的位置不对的话会报错，说不认识这些选项）：
应该在[mysqld]选项下加入
```
server-id=1 #server-id表示单个结点的id，这里由于只有一个结点，所以可以把id随机指定为一个数，这里将id设置成1。若集群中有多个结点，则id不能相同
binlog_format           = MIXED                         #binlog日志格式，mysql默认采用statement，建议使用mixed
log-bin                 = /var/lib/mysql/mysql-bin   #binlog日志文件
expire_logs_days        = 7                           #binlog过期清理时间
max_binlog_size         = 100m                       #binlog每个日志文件大小
binlog_cache_size       = 4m                        #binlog缓存大小
max_binlog_cache_size   = 512m                     #最大binlog缓存大小
```

重新启动用 `service mariadb  restart`就好了


### Tips（艰难的mysql重启过程）
（1）重启mysql遇到的问题
```
 （1）通过rpm包安装的MySQL通过下面这样的方式重启

 service mysqld restart
/etc/inint.d/mysqld start

（2）从源码包安装的MySQL（我的服务器上是/usr/bin）
// Linux关闭MySQL的命令
$mysql_dir/bin/mysqladmin -uroot -p shutdown

// linux启动MySQL的命令
$mysql_dir/bin/mysqld_safe &
用这个命令启动的时候一直卡在
[1] 11513
[root@rabbitmq bin]# 190118 19:30:00 mysqld_safe Logging to '/var/log/mariadb/mariadb.log'.
190118 19:30:00 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
为什么呢？？？？
查看了一下日志，说没有找到mysql-bin.index文件
190118 19:23:41 [Note] /usr/libexec/mysqld (mysqld 5.5.56-MariaDB) starting as process 11421 ...
190118 19:23:41 mysqld_safe mysqld from pid file /var/run/mariadb/mariadb.pid ended
190118 19:30:00 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
190118 19:30:00 [ERROR] mysqld: File '/data/mysql/mysql-bin.index' not found (Errcode: 2)
190118 19:30:00 [ERROR] Aborting

在/data/mysql/创建了一个mysql-bin.index文件再次启动，还是报错
190118 19:30:00 [Note] /usr/libexec/mysqld (mysqld 5.5.56-MariaDB) starting as process 11740 ...
190118 19:30:00 mysqld_safe mysqld from pid file /var/run/mariadb/mariadb.pid ended
190118 19:39:04 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
190118 19:39:04 [ERROR] mysqld: File '/data/mysql/mysql-bin.index' not found (Errcode: 13)
190118 19:39:04 [ERROR] Aborting

190118 19:39:04 [Note] /usr/libexec/mysqld: Shutdown complete

errcode13，一般就是权限问题，通过命令 chown -R mysql:mysql /data/mysql/mysql-bin.index
将原来为  -rw-r--r--. 1 root root 0 1月  18 19:38 mysql-bin.index
改为-rw-r--r--. 1 mysql mysql 0 1月  18 19:38 mysql-bin.index
再次启动，还是报错
190118 19:45:05 [Note] /usr/libexec/mysqld (mysqld 5.5.56-MariaDB) starting as process 12690 ...
190118 19:45:05 mysqld_safe mysqld from pid file /var/run/mariadb/mariadb.pid ended
190118 19:45:38 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
190118 19:45:38 [ERROR] mysqld: File '/data/mysql/mysql-bin.~rec~' not found (Errcode: 13)
190118 19:45:38 [ERROR] MYSQL_BIN_LOG::open_purge_index_file failed to open register  file.
190118 19:45:38 [ERROR] MYSQL_BIN_LOG::open_index_file failed to sync the index file.
190118 19:45:38 [ERROR] Aborting

190118 19:45:38 [Note] /usr/libexec/mysqld: Shutdown complete

换一种思路，更改log日志存在地点
log-bin                 = /var/lib/mysql/mysql-bin 
这次启动时 抱如下信息
190118 19:52:58 [Note] /usr/libexec/mysqld (mysqld 5.5.56-MariaDB) starting as process 13554 ...
190118 19:52:58 mysqld_safe mysqld from pid file /var/run/mariadb/mariadb.pid ended
190118 19:56:48 mysqld_safe Starting mysqld daemon with databases from /var/lib/mysql
190118 19:56:48 [Note] /usr/libexec/mysqld (mysqld 5.5.56-MariaDB) starting as process 13855 ...
190118 19:56:48 InnoDB: The InnoDB memory heap is disabled
190118 19:56:48 InnoDB: Mutexes and rw_locks use GCC atomic builtins
190118 19:56:48 InnoDB: Compressed tables use zlib 1.2.7
190118 19:56:48 InnoDB: Using Linux native AIO
190118 19:56:48 InnoDB: Initializing buffer pool, size = 128.0M
190118 19:56:48 InnoDB: Completed initialization of buffer pool
190118 19:56:48 InnoDB: highest supported file format is Barracuda.
190118 19:56:48  InnoDB: Waiting for the background threads to start
190118 19:56:49 Percona XtraDB (http://www.percona.com) 5.5.52-MariaDB-38.3 started; log sequence number 1597945
190118 19:56:50 [Note] Plugin 'FEEDBACK' is disabled.
190118 19:56:50 [Note] Server socket created on IP: '0.0.0.0'.
190118 19:56:50 [Note] /usr/libexec/mysqld: ready for connections.
Version: '5.5.56-MariaDB'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MariaDB Server

还是报错，是不是mysqld_safe这个命令有毒啊。。。。换 service mariadb  restart这个命令。。。


```
 （2）每次服务器（数据库）重启，服务器会调用flush logs;，会新创建一个binlog日志

 （3）mysqld_safe脚本执行的基本流程:
```
1、查找basedir和ledir。
2、查找datadir和my.cnf。
3、对my.cnf做一些检查，具体检查哪些选项请看附件中的注释。
4、解析my.cnf中的组[mysqld]和[mysqld_safe]并和终端里输入的命令合并。
5、调用parse_arguments函数解析用户传递的所有参数($@)。
6、对系统日志和错误日志的判断和相应处理具体可以参考附件中的注释，及选项--err-log参数的赋值。
7、对选项--user，--pid-file，--socket及--port进行处理及赋值，保证启动时如果不给出这些参数它也会有值。
8、启动mysqld.
a)启动时会判断一个进程号是否存在，如果存在那么就在错误日志中记录"A mysqld process already exists"并且退出。
b)如不存在就删除进程文件，如果删除不了，那么就在错误日志中记录"Fatal error: Can't remove the pid file"并退出。
9、启动时对表进行检查。如果启动的时候检查表的话设置key_buffer and sort_buffer会提高速度并且减少磁盘空间的使用。也可以使用myisam-recover选项恢复出错的myisam表。
10、如果启动时你什么参数都没有给，那么它会选用一些特定的参数启动，具体哪些参数请看附件注释。
11、如果服务器异常关闭，那么会restart。

最后用三步来总结
检查环境
检查配置选项
启动及启动后的处理

总结：选用mysqld_safe启动的好处。
1、mysqld_safe增加了一些安全特性，例如当出现错误时重启服务器并向错误日志文件写入运行时间信息。
2、如果有的选项是mysqld_safe 启动时特有的，那么可以终端指定，如果在配置文件中指定需要放在[mysqld_safe]组里面，放在其他组不能被正确解析。
3、mysqld_safe启动能够指定内核文件大小 ulimit -c $core_file_size以及打开的文件的数量ulimit -n $size。
4、MySQL程序首先检查环境变量，然后检查配置文件，最后检查终端的选项，说明终端指定选项优先级最高
```
（4）命令systemctl enable mysqld.service一直报错
Failed to execute operation: No such file or directory
在CentOS7中已经不在支持mysql，就算你已经安装了，CentOS7还是表示很嫌弃
使用 systemctl status mariadb.service 
CentOS7不支持 mysqld，无法启动了，所以才需要装mariadb


到现在可以发现终于把binlog给启动了
show variables like '%log_bin%'; 
 ![image](https://github.com/sjt157/MarkDownPhotos/raw/master/MYSQL_BinLog/2.png)

----------

### 常用binlog操作命令
```
show master logs;查看所有binlog日志列表
show master status;查看master状态，即最后(最新)一个binlog日志的编号名称，及其最后一个操作事件pos结束点(Position)值
flush logs; 刷新log日志，自此刻开始产生一个新编号的binlog日志文件
reset master;重置(清空)所有binlog日志
show binlog events in 'mysql-bin.000002';查看binlog日志内容（以表格形式）

```

### binlog的三种工作模式 
（1）Row level （我的数据库上默认的是ROW）
　　ROW是基于行级别的,他会记录每一行记录的变化,就是将每一行的修改都记录到binlog里面,记录的非常详细，但sql语句并没有在binlog里。 
　　日志中会记录每一行数据被修改的情况，然后在slave端对相同的数据进行修改。在replication里面也不会因为存储过程触发器等造成Master-Slave数据不一致的问题,但是有个致命的缺点日志量比较大.由于要记录每一行的数据变化,当执行update语句后面不加where条件的时候或alter table的时候,产生的日志量是相当的大。 
　　 
　　 
（2）Statement level（默认） 
　　每一条被修改数据的sql都会记录到master的bin-log中，slave在复制的时候sql进程会解析成和原来master端执行过的相同的sql再次执行 
　　优点：解决了 Row level下的缺点，不需要记录每一行的数据变化，减少bin-log日志量，节约磁盘IO，提高新能
   缺点：在statement模式下，由于他是记录的执行语句，所以，为了让这些语句在slave端也能正确执行，那么他还必须记录每条语句在执行的时候的一些相关信息，也就是上下文信息，以保证所有语句在slave端被执行的时候能够得到和在master端执行时候相同的结果。另外就是，由于mysql现在发展比较快，很多的新功能不断的加入，使mysql的复制遇到了不小的挑战，自然复制的时候涉及到越复杂的内容，bug也就越容易出现。在statement中，目前已经发现不少情况会造成Mysql的复制出现问题，主要是修改数据的时候使用了某些特定的函数或者功能的时候会出现，比如：sleep()函数在有些版本中就不能被正确复制，在存储过程中使用了last_insert_id()函数，可能会使slave和master上得到不一致的id等等。由于row是基于每一行来记录的变化，所以不会出现，类似的问题。

（3）Mixed（混合模式） 
　　结合了Row level和Statement level的优点。 
　　在默认情况下是statement,但是在某些情况下会切换到row状态，如当一个DML更新一个ndb引擎表，或者是与时间用户相关的函数等。在主从的情况下，在主机上如果是STATEMENT模式，那么binlog就是直接写now()，然而如果这样的话，那么从机进行操作的时间，也执行now()，但明显这两个时间不会是一样的，所以对于这种情况就必须把STATEMENT模式更改为ROW模式，因为ROW模式会直接写值而不是写语句（该案例是错误的，即使是STATEMENT模式也可以使用now()函数，具体原因以后再分析）。同样ROW模式还可以减少从机的相关计算，如在主机中存在统计写入等操作时，从机就可以免掉该计算把值直接写入从机。

### MySQL的日志（主要是Binlog）对系统性能的影响
（1）日志产生的性能影响
由于日志的记录带来的直接性能损耗就是数据库系统中最为昂贵的IO资源。

MySQL的日志主要包括错误日志（ErrorLog），更新日志（UpdateLog），二进制日志（Binlog），查询日志（QueryLog），慢查询日志（SlowQueryLog）等。
特别注意：更新日志是老版本的MySQL才有的，目前已经被二进制日志替代。

在默认情况下，系统仅仅打开错误日志，关闭了其他所有日志，以达到尽可能减少IO损耗提高系统性能的目的。
但是在一般稍微重要一点的实际应用场景中，都至少需要打开二进制日志，因为这是MySQL很多存储引擎进行增量备份的基础，也是MySQL实现复制的基本条件。
有时候为了进一步的mysql性能优化，定位执行较慢的SQL语句，很多系统也会打开慢查询日志来记录执行时间超过特定数值（由我们自行设置）的SQL语句。

一般情况下，在生产系统中很少有系统会打开查询日志。因为查询日志打开之后会将MySQL中执行的每一条Query都记录到日志中，会该系统带来比较大的IO负担，而带来的实际效益却并不是非常大。一般只有在开发测试环境中，为了定位某些功能具体使用了哪些SQL语句的时候，才会在短时间段内打开该日志来做相应的分析。
所以，在MySQL系统中，会对性能产生影响的MySQL日志（不包括各存储引擎自己的日志）主要就是Binlog了。


### 一般企业binlog模式的选择： 

互联网公司使用MySQL的功能较少（不用存储过程、触发器、函数），选择默认的Statement level； 
用到MySQL的特殊功能（存储过程、触发器、函数）则选择Mixed模式； 
用到MySQL的特殊功能（存储过程、触发器、函数），又希望数据最大化一直则选择Row模式；

mysql对于日志格式的选定原则:如果是采用 INSERT，UPDATE，DELETE 等直接操作表的情况，则日志格式根据 binlog_format 的设定而记录,如果是采用 GRANT，REVOKE，SET PASSWORD 等管理语句来做的话，那么无论如何 都采用 SBR 模式记录。

### 参考

https://blog.csdn.net/z1988316/article/details/7883147?utm_source=blogxgwz2
https://blog.csdn.net/intelrain/article/details/80451120
https://blog.csdn.net/weixin_38187469/article/details/79273962
https://blog.csdn.net/keda8997110/article/details/50895171
