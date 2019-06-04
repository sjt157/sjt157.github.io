---
title: 谷歌之Percolator模型
date: 2018-12-15 16:10:29
tags: TiDB
categories: TiDB
---
## 论文
今天读了一篇论文<https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf>
## 思考
1. 在谷歌的索引系统中。通过MapReduce将爬取下来的页面弄到Percolator中，该模型提供用户随机访问一个超大的数据库，避免了MapReduce的全局scan。该模型中因为有很多线程并发，所以提供了ACID。
2. Percolator由一系列observers组成。每个observer可以完成一个任务而且可以为下游的observers指派更多的任务。
3. Percolator使用于增量处理。
4. 该模型如下图，A Percolator system由这三部分组成。还需要依赖两个服务the timestamp oracle and the lightweight lock service。该模型的事务：Percolator provides cross-row, cross-table transactions with ACID snapshot-isolation semantics.

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/TiDB/1.png)
5. 提出了snapshot isolation。意思如图

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/TiDB/2.png)
6. 这里边还讲了个很有意思的比喻。扫描线程一多可能会导致parallelism的减少，拿公交车来作比喻，公交车一慢，会导致等的乘客过多，从而导致车更慢。该模型中为了解决这个问题，采取了如下做法：当一个线程发现别的线程慢的时候，它选择一个随机的位置进行扫描。

## TiDB

通过查询知道TiKV 的事务采用的就是是 Percolator 模型，并且做了大量的优化。事务的细节这里不详述，大家可以参考论文。这里只提一点，TiKV 的事务采用乐观锁，事务的执行过程中，不会检测写写冲突，只有在提交过程中，才会做冲突检测，冲突的双方中比较早完成提交的会写入成功，另一方会尝试重新执行整个事务。当业务的写入冲突不严重的情况下，这种模型性能会很好，比如随机更新表中某一行的数据，并且表很大。但是如果业务的写入冲突严重，性能就会很差，举一个极端的例子，就是计数器，多个客户端同时修改少量行，导致冲突严重的，造成大量的无效重试。

* Percolator原理比较简单，总体来说就是一个经过优化的二阶段提交的实现，进行了一个二级锁的优化。TiDB 的事务模型沿用了 Percolator 的事务模型。 总体的流程如下：
```
读写事务
1) 事务提交前，在客户端 buffer 所有的 update/delete 操作。 2) Prewrite 阶段:
首先在所有行的写操作中选出一个作为 primary，其他的为 secondaries。
PrewritePrimary: 对 primaryRow 写入 L 列(上锁)，L 列中记录本次事务的开始时间戳。写入 L 列前会检查:
1.	是否已经有别的客户端已经上锁 (Locking)。
2.	是否在本次事务开始时间之后，检查 W 列，是否有更新 [startTs, +Inf) 的写操作已经提交 (Conflict)。
在这两种种情况下会返回事务冲突。否则，就成功上锁。将行的内容写入 row 中，时间戳设置为 startTs。
将 primaryRow 的锁上好了以后，进行 secondaries 的 prewrite 流程:
1.	类似 primaryRow 的上锁流程，只不过锁的内容为事务开始时间及 primaryRow 的 Lock 的信息。
2.	检查的事项同 primaryRow 的一致。
当锁成功写入后，写入 row，时间戳设置为 startTs。
3) 以上 Prewrite 流程任何一步发生错误，都会进行回滚：删除 Lock，删除版本为 startTs 的数据。
4) 当 Prewrite 完成以后，进入 Commit 阶段，当前时间戳为 commitTs，且 commitTs> startTs :
1.	commit primary：写入 W 列新数据，时间戳为 commitTs，内容为 startTs，表明数据的最新版本是 startTs 对应的数据。
2.	删除L列。
如果 primary row 提交失败的话，全事务回滚，回滚逻辑同 prewrite。如果 commit primary 成功，则可以异步的 commit secondaries, 流程和 commit primary 一致， 失败了也无所谓。

事务中的读操作
1.	检查该行是否有 L 列，时间戳为 [0, startTs]，如果有，表示目前有其他事务正占用此行，如果这个锁已经超时则尝试清除，否则等待超时或者其他事务主动解锁。注意此时不能直接返回老版本的数据，否则会发生幻读的问题。
2.	读取至 startTs 时该行最新的数据，方法是：读取 W 列，时间戳为 [0, startTs], 获取这一列的值，转化成时间戳 t, 然后读取此列于 t 版本的数据内容。
由于锁是分两级的，primary 和 seconary，只要 primary 的行锁去掉，就表示该事务已经成功 提交，这样的好处是 secondary 的 commit 是可以异步进行的，只是在异步提交进行的过程中 ，如果此时有读请求，可能会需要做一下锁的清理工作。

```



## 初体验TiDB

* 通过docker-compose来部署，目前为止，所以组件已经准备完毕，如图
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/TiDB/3.png)

```
访问集群: mysql -h 127.0.0.1 -P 4000 -u root
访问集群 Grafana 监控页面: http://localhost:3000 默认用户名和密码均为 admin。
集群数据可视化： http://localhost:8010

```
效果如图：
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/TiDB/4.png)
