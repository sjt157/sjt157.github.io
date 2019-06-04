---
title: 网络之TIME_WAIT与C_WAIT
date: 2019-03-17 10:37:13
tags: TCP/IP
categories: TCP/IP
---


### 如何查看TIME_WAIT与CLOSE_WAIT情况
通过命令查看
`netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`

发现这台服务器上

```
[root@data9 ~]# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
CLOSE_WAIT 22    CLOSE_WAIT 表示被动关闭，出现这个说明在对方关闭连接之后，没有进行相应处理。
ESTABLISHED 55   ESTABLISHED 表示正在通信
TIME_WAIT 3  TIME_WAIT 表示主动关闭
```

### TIME_WAIT(主动关闭)
为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？
这是因为：虽然双方都同意关闭连接了，而且握手的4个报文也都协调和发送完毕，按理可以直接回到CLOSED状态（就好比从SYN_SEND状态到ESTABLISH状态那样）；但是因为我们必须要假想网络是不可靠的，你无法保证你最后发送的ACK报文会一定被对方收到，因此对方处于LAST_ACK状态下的SOCKET可能会因为超时未收到ACK报文，而重发FIN报文，所以这个TIME_WAIT状态的作用就是用来重发可能丢失的ACK报文，并保证于此。

#### 为什么会有Time_wait状态

1. 可靠的终止TCP连接，若处于time_wait的客户端发送给服务器确认报文段丢失的话，服务器将在此重新发送FIN报文段，那么客户端必须处于一个可接收的状态就是time_wait而不是close状态。 
2. 保证迟来的TCP报文段有足够的时间被识别并丢弃，linux 中一个TCP端口不能打开两次或两次以上，当客户端处于time_wait状态时我们将无法使用此端口建立新连接，如果不存在time_wait状态，新连接可能会收到旧连接的数据。Time_wait持续的时间是2MSL，保证旧的数据可以丢弃，因为网络中的数据最大存在MSL( aximum segment lifetime)

#### 如何解决
解决思路很简单，就是让服务器能够快速回收和重用那些TIME_WAIT的资源。

/etc/sysctl.conf文件的修改
```shell

#对于一个新建连接，内核要发送多少个 SYN 连接请求才决定放弃,不应该大于255，默认值是5，对应于180秒左右时间 
net.ipv4.tcp_syn_retries=2
#net.ipv4.tcp_synack_retries=2
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为300秒
net.ipv4.tcp_keepalive_time=1200
net.ipv4.tcp_orphan_retries=3
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间
net.ipv4.tcp_fin_timeout=30  
#表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。
net.ipv4.tcp_max_syn_backlog = 4096
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭
net.ipv4.tcp_syncookies = 1

#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭
net.ipv4.tcp_tw_reuse = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭
net.ipv4.tcp_tw_recycle = 1

##减少超时前的探测次数 
net.ipv4.tcp_keepalive_probes=5 
##优化网络设备接收队列 
net.core.netdev_max_backlog=3000 

修改完之后执行/sbin/sysctl -p让参数生效。

这里头主要注意到的是net.ipv4.tcp_tw_reuse
net.ipv4.tcp_tw_recycle 
net.ipv4.tcp_fin_timeout 
net.ipv4.tcp_keepalive_*
这几个参数。

net.ipv4.tcp_tw_reuse和net.ipv4.tcp_tw_recycle的开启都是为了回收处于TIME_WAIT状态的资源。
net.ipv4.tcp_fin_timeout这个时间可以减少在异常情况下服务器从FIN-WAIT-2转到TIME_WAIT的时间。
net.ipv4.tcp_keepalive_*一系列参数，是用来设置服务器检测连接存活的相关配置。

```

### Close_Wait(被动关闭)
简单来说CLOSE_WAIT数目过大是由于被动关闭连接处理不当导致的。
我说一个场景，服务器A会去请求服务器B上面的apache获取文件资源，正常情况下，如果请求成功，那么在抓取完资源后服务器A会主动发出关闭连接的请求，这个时候就是主动关闭连接，连接状态我们可以看到是TIME_WAIT。如果一旦发生异常呢？假设请求的资源服务器B上并不存在，那么这个时候就会由服务器B发出关闭连接的请求，服务器A就是被动的关闭了连接，如果服务器A被动关闭连接之后自己并没有释放连接，那就会造成CLOSE_WAIT的状态了。

#### 如何解决？
待补充

#### 最后来回答两个问题：
1. 为啥一台机器区区几百个close_wait就导致不可继续访问?不合理啊，一台机器不是号称最大可以打开65535个端口吗？
回答：由于原因#4和#3所以导致整个IoLoop慢了，进而因为#2导致很多请求堆积，也就是说很多请求在被真正处理前已经在backlog里等了一会了。导致了SLB这端的链接批量的超时，同时又由于close_wait状态不会自动消失，导式最终无法再这个端口上创建新的链接引起了停止服务。
2. 为啥明明有多个服务器承载，却几乎同时出了close_wait？又为什么同时不能再服务？那要SLB还有啥用呢？
回答：有了上一个答案，结合SLB的特性，这个也就很好解释。这就是所谓的洪水蔓延，当SLB发现下面的一个节点不可用会把请求routing到其他可用节点上，导致其他节点压力增大。也犹豫相同原因，加速了其他节点出现close_wait.


### 参考
<https://blog.csdn.net/shootyou/article/details/6622226>
