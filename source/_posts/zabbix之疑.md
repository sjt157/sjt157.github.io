---
title: zabbix之疑
date: 2019-05-25 19:13:50
tags: zabbix
categories: zabbix
---



我们的云平台CC节点上zabbix_server进程非常多啊，多达129个，我咋觉得有点不正常呢。
```shell
[root@cc /]# ps -ef | grep zabbix_server | wc -l
129
```
这么多进程需要占用多少内存啊得，
```shell
for pid in `ps -C zabbix_server -o pid --no-heading`; do grep Pss /proc/$pid/smaps;done|awk '{sum+=$2};END{print sum/1024/1024"G"}'
```
一看，还行啊，才用了0.103609G。也就是103m。这么多进程才用了这么点内存，emmmm。。。。。


### 统计内存要用Pss，不要用Rss

用ps命令统计所有zabbix_server进程的rss值进行累加得出占用总内存，此种方法不对，会导致结果偏大。很多人通过累加 “ps  aux” 命令显示的 RSS 列来统计全部进程总共占用的物理内存大小，这是不对的。RSS(resident set size)表示常驻内存的大小，但是由于不同的进程之间会共享内存，所以把所有进程RSS进行累加的方法会重复计算共享内存，得到的结果是偏大的。

正确的方法是累加 /proc/[1-9]*/smaps 中的 Pss 。/proc/<pid>/smaps 包含了进程的每一个内存映射的统计值，详见proc(5)的手册页。Pss(Proportional Set Size)把共享内存的Rss进行了平均分摊，比如某一块100MB的内存被10个进程共享，那么每个进程就摊到10MB。这样，累加Pss就不会导致共享内存被重复计算了。

### Zabbix-Server 的处理进程:
* watchdog：负责监视所有的进程的状态的，一旦某个进程关闭，watchdog将会通知zabbix重新启动该进程
* housekeeper：负责清理过期的数据
* aleter：报警使用
* poller：拉取数据
* httppoller：监控web页面的专业拉取数据工具
* discoverer: 发现资源
* pinger：使用Fping/Ping等工具探测主机是否在线
* db_config_syncer：数据库配置同步
* db_date_syncer：数据库数据同步
* nodewatcher：监控各个节点
* timer：计时器
* escalator：报警升级

### zabbix_get的使用

使用zabbix_get获取cpu负载，还可以获取其他信息。。。
在CC主机上，执行
`./zabbix_get -s 172.21.4.101 -p 10050 -k "system.cpu.load[all,avg15]"`


### 参考
<https://www.jianshu.com/p/b67cbd507ac3>

