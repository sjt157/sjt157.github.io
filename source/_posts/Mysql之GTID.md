---
title: Mysql之GTID
date: 2019-03-23 15:09:24
tags: Mysql
categories: Mysql
---

### 万恶之源
今天看一个博客的时候,发现了一个新东西GTID

这篇博客说道:
CDC模块可以从mysql那里拿binlog
CDC 模块解析 binlog，产生特定格式的变更消息，也就完成了一次变更抓取。但这还不够，CDC 模块本身也可能挂掉，那么恢复之后如何保证不丢数据又是一个问题。这个问题的解决方案也是要针对不同数据源进行设计的，就 MySQL 而言，通常会持久化已经消费的 binlog 位点或 Gtid(MySQL 5.6之后引入)来标记上次消费位置。其中更好的选择是 Gtid，因为该位点对于一套 MySQL 体系（主从或多主）是全局的，而 binlog 位点是单机的，无法支持主备或多主架构。

<https://aleiwu.com/post/vimur/>

我开始不懂了。。。。。。。。


### 什么是GTID

MySQL5.6在5.5的基础上增加了GTID，    GTID即全局事务ID（global transaction identifier），GTID实际上是由UUID+TID组成的。其中UUID是一个MySQL实例的唯一标识。TID代表了该实例上已经提交的事务数量，并且随着事务提交单调递增，所以GTID能够保证每个MySQL实例事务的执行（不会重复执行同一个事务，并且会补全没有执行的事务）。下面是一个GTID的具体形式：

4e659069-3cd8-11e5-9a49-001c4270714e:1-77

GTID用于在binlog中唯一标识一个事务。当事务提交时，MySQL Server在写binlog的时候，会先写一个特殊的Binlog Event，类型为GTID_Event，指定下一个事务的GTID，然后再写事务的Binlog。主从同步时GTID_Event和事务的Binlog都会传递到从库，从库在执行的时候也是用同样的GTID写binlog，这样主从同步以后，就可通过GTID确定从库同步到的位置了。也就是说，无论是级联情况，还是一主多从情况，都可以通过GTID自动找点儿，而无需像之前那样通过File_name和File_position找点儿了。

### GTID有啥用？

* 因为清楚了GTID的格式，所以通过UUID可以知道这个事务在哪个实例上提交的。

* 通过GTID可以极方便的进行复制结构上的故障转移，新主设置。很好的解决了下面这个图（图来自高性能MySQL第10章）的问题。


上面图的意思是：Server1(Master)崩溃，根据从上show slave status获得Master_log_File/Read_Master_Log_Pos的值，Server2(Slave)已经跟上了主，Server3(Slave)没有跟上主。这时要是把Server2提升为主，Server3变成Server2的从。这时在Server3上执行change的时候需要做一些计算，这里就不做说明了，具体的说明见高性能MySQL第10章，相对来说是比较麻烦的。

这个问题在5.6的GTID出现后，就显得非常的简单。由于同一事务的GTID在所有节点上的值一致，那么根据Server3当前停止点的GTID就能定位到Server2上的GTID。甚至由于MASTER_AUTO_POSITION功能的出现，我们都不需要知道GTID的具体值，直接使用CHANGE MASTER TO MASTER_HOST='xxx'， MASTER_AUTO_POSITION命令就可以直接完成failover的工作。

  
原理：
从服务器连接到主服务器之后，把自己执行过的GTID(Executed_Gtid_Set)<SQL线程> 、获取到的GTID(Retrieved_Gtid_Set）<IO线程>发给主服务器，主服务器把从服务器缺少的GTID及对应的transactions发过去补全即可。当主服务器挂掉的时候，找出同步最成功的那台从服务器，直接把它提升为主即可。如果硬要指定某一台不是最新的从服务器提升为主， 先change到同步最成功的那台从服务器， 等把GTID全部补全了，就可以把它提升为主了。

### 使用GTID注意事项
 
* 开启GTID以后，无法使用sql_slave_skip_counter跳过事务。前面介绍过了，使用GTID找点儿时，主库会把从库缺失的GTID，发送给从库，所以skip是没有用的。为了提前发现问题，MySQL在gtid模式下，直接禁止使用set global sql_slave_skip_counter ＝ x。正确的做法是，通过set grid_next= 'zzzz'（'zzzz'为待跳过的事务），然后执行BIGIN;COMMIT产生一个空事务，占据这个GTID，再START SLAVE，会发现下一条事务的GTID已经执行过，就会跳过这个事务了

* 如果一个GTID已经执行过，再遇到重复的GTID，从库会直接跳过，可看作GTID执行的幂等性。

* 使用限制：<https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-restrictions.html>


### 业界经验：

1. <https://code.facebook.com/posts/1542273532669494/lessons-from-deploying-mysql-gtid-at-scale/>

2. <https://www.percona.com/blog/2015/02/10/online-gtid-rollout-now-available-percona-server-5-6/>

 

### 参考
<https://www.cnblogs.com/zhoujinyi/p/4717951.html>
<https://www.cnblogs.com/zejin2008/p/7705473.html>
