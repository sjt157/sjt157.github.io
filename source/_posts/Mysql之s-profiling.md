---
title: Mysql之s_profiling
date: 2019-01-18 11:41:57
tags: Mysql
categories: Mysql
---


今天在听Cetus课程的时候，看到了老师说了set profiling=1； 并不知道这是干什么的，来记录一下。
 

Query Profiler 来定位一条 Query 的性能瓶颈，这里我们再详细介绍一下 Profiling 的用途及使用方法。

要想优化一条 Query，我们就需要清楚的知道这条 Query 的性能瓶颈到底在哪里，是消耗的 CPU计算太多，还是需要的的 IO 操作太多？要想能够清楚的了解这些信息，在 MySQL 5.0 和 MySQL 5.1正式版中已经可以非常容易做到了，那就是通过 Query Profiler 功能。

MySQL 的 Query Profiler 是一个使用非常方便的 Query 诊断分析工具，通过该工具可以获取一条Query 在整个执行过程中多种资源的消耗情况，如 CPU，IO，IPC，SWAP 等，以及发生的 PAGE FAULTS，CONTEXT SWITCHE 等等，同时还能得到该 Query 执行过程中 MySQL 所调用的各个函数在源文件中的位置。

下面我们看看 Query Profiler 的具体用法。

1. 开启 profiling 参数
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Mysql_Set_profile/1.png)

2. 执行Query
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Mysql_Set_profile/2.png)

 在开启 Query Profiler 功能之后，MySQL 就会自动记录所有执行的 Query 的 profile 信息了。
3. 取系统中保存的所有 Query 的 profile 概要信息
 ![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Mysql_Set_profile/3.png)
通过执行 “SHOW PROFILE” 命令获取当前系统中保存的多个 Query 的 profile 的概要信息。
4. 针对单个 Query 获取详细的 profile 信息
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Mysql_Set_profile/4.png)
上面的例子中是获取 CPU 和 Block IO 的消耗，非常清晰，对于定位性能瓶颈非常适用。希望得到取其他的信息，都可以通过执行 “SHOW PROFILE *** FOR QUERY n” 来获取，各位读者朋友可以自行测试熟悉。

### 参考
<https://www.cnblogs.com/ggjucheng/archive/2012/11/15/2772058.html>
