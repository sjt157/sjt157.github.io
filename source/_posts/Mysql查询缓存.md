---
title: Mysql查询缓存
date: 2019-02-22 17:03:31
tags: Mysql
categries: Mysql
---



### 开启查询缓存后，查询语句的解析过程：

> 在解析一个查询语句之前，如果查询缓存是打开的，那么MySQL会优先检查这个查询是否命中查询缓存中的数据。如果当前的查询恰好命中了查询缓存，那么在返回查询结果之前MySQL会检查一次用户权限。若权限没有问题，MySQL会跳过所有其他阶段（解析、优化、执行等），直接从缓存中拿到结果并返回给客户端。这种情况下，查询不会被解析，不用生成执行计划，不会被执行。

### 开启查询缓存
#### 设置使用查询缓存的方式
使用 query_cache_type 变量来开启查询缓存，开启方式有三种：

* ON : 正常缓存。表示在使用 SELECT 语句查询时，若没指定 SQL_NO_CACHE 或其他非确定性函数，则一般都会将查询结果缓存下来。
* DEMAND ：指定SQL_CACHE才缓存。表示在使用 SELECT 语句查询时，必须在该 SELECT 语句中指定 SQL_CACHE 才会将该SELECT语句的查询结果缓存下来。

> 例如：select SQL_CACHE name from user where id = 15;    #只有明确指定  SQL_CACHE 的SELECT语句，才会将查询结果缓存。

* OFF： 关闭查询缓存。

#### 立刻生效，重启服务失效

* `mysql> set global query_cache_type=1;`
* 当my.cnf 中，query_cache_type = OFF ，启动mysql服务后，在mysql命令行中使用上面语句开启查询缓存，会报错：ERROR 1651 (HY000): Query cache is disabled; restart the server with query_cache_type=1 to enable it。遇到这种情况，是无法在mysql命令行中开启查询缓存的，必须修改my.cnf的query_cache_type = ON，然后重启mysql服务。

### 设置查询缓存的大小
> query_cache_size ：查询缓存的总体可用空间。
* **注意**：如果 query_cache_size=0 ，那即便你设置了 query_cache_type = ON，查询缓存仍然是无法工作的。

### 查询缓存相关参数
```
uery_cache_limit :  MySQL能够缓存的最大查询结果；如果某查询的结果大小大于此值，则不会被缓存；
query_cache_min_res_unit : 查询缓存中分配内存的最小单位；(注意：此值通常是需要调整的，此值被调整为接近所有查询结果的平均值是最好的)
                           计算单个查询的平均缓存大小：（query_cache_size-Qcache_free_memory）/Qcache_queries_in_cache
query_cache_size : 查询缓存的总体可用空间，单位为字节；其必须为1024的倍数；
query_cache_type: 查询缓存类型；是否开启缓存功能，开启方式有三种{ON|OFF|DEMAND}；
query_cache_wlock_invalidate : 当其它会话锁定此次查询用到的资源时，是否不能再从缓存中返回数据；（OFF表示可以从缓存中返回数据）
```

### 查询缓存状态
```
mysql> SHOW  GLOBAL STATUS  LIKE  'Qcache%';
+-------------------------+----------+
| Variable_name            | Value   |
+-------------------------+----------+
| Qcache_free_blocks       | 1       | #查询缓存中的空闲块
| Qcache_free_memory       | 16759656| #查询缓存中尚未使用的空闲内存空间
| Qcache_hits              | 16      | #缓存命中次数
| Qcache_inserts           | 71      | #向查询缓存中添加缓存记录的条数
| Qcache_lowmem_prunes     | 0       | #表示因缓存满了而不得不清理部分缓存以存储新的缓存，这样操作的次数。若此数值过大，则表示缓存空间太小了。
| Qcache_not_cached        | 57      | #没能被缓存的次数
| Qcache_queries_in_cache  | 0       | #此时仍留在查询缓存的缓存个数
| Qcache_total_blocks      | 1       | #共分配出去的块数
+-------------------------+----------+
```

### 衡量缓存是否有效
#### 方式一：
```
mysql> SHOW GLOBAL STATUS WHERE Variable_name='Qcache_hits' OR Variable_name='Com_select';
+---------------+-----------+
| Variable_name | Value |
+---------------+-----------+
| Com_select    | 279292490 | #非缓存查询次数
| Qcache_hits   | 307366973 | # 缓存命中次数
+---------------+-----------

```
> 缓存命中率：Qcache_hits/(Qcache_hits+Com_select)

#### 方式二：“命中和写入”的比率
这是另外一种衡量缓存是否有效的指标。

```
mysql> SHOW GLOBAL STATUS WHERE Variable_name='Qcache_hits' OR Variable_name='Qcache_inserts';
+----------------+-----------+
| Variable_name  | Value     |
+----------------+-----------+
| Qcache_hits | 307416113    | #缓存命中次数
| Qcache_inserts | 108873957 | #向查询缓存中添加缓存记录的条数
```
> “命中和写入”的比率: Qcache_hits/Qcache_inserts # 如果此比值大于3:1, 说明缓存也是有效的；如果高于10:1，相当理想；

### 缓存失效问题
当数据表改动时，基于该数据表的任何缓存都会被删除。（表层面的管理，不是记录层面的管理，因此失效率较高）

**注意事项**：

* 应用程序，不应该关心query cache的使用情况。可以尝试使用，但不能由query cache决定业务逻辑，因为query cache由DBA来管理。
* 缓存是以SQL语句为key存储的，因此即使SQL语句功能相同，但如果多了一个空格或者大小写有差异都会导致匹配不到缓存。


## 参考：
<https://www.jianshu.com/p/5c3ddf9a454c>
<https://juejin.im/post/5c6b9c09f265da2d8a55a855>
