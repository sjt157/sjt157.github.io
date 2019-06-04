---
title: Linux下的一些小工具
date: 2019-03-17 10:48:26
tags: Linux
categories: Linux
---
 

### 扫描端口占用情况
```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-
import socket, time, thread
socket.setdefaulttimeout(3) #设置默认超时时间

def socket_port(ip, port):
    """
    输入IP和端口号，扫描判断端口是否占用
    """
    try:
        if port >=65535:
            print u'端口扫描结束'
        s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        result=s.connect_ex((ip, port))
        if result==0:
            lock.acquire()
            print ip,u':',port,u'端口已占用'
            lock.release()
    except:
        print u'端口扫描异常'

def ip_scan(ip):
    """
    输入IP，扫描IP的0-65534端口情况
    """
    try:
        print u'开始扫描 %s' % ip
        start_time=time.time()
        for i in range(0,65534):
            thread.start_new_thread(socket_port,(ip, int(i)))
        print u'扫描端口完成，总共用时：%.2f' %(time.time()-start_time)
#       raw_input("Press Enter to Exit")
    except:
        print u'扫描ip出错'

if __name__=='__main__':
    url=raw_input('Input the ip you want to scan: ')
    lock=thread.allocate_lock()
    ip_scan(url)

```

### 判断哪些进程占用的资源多
```shell
#!/bin/bash
for proc in $(find /proc -maxdepth 1 -regex '/proc/[0-9]+'); do
    printf "%2d %5d %s\n"   "$(cat $proc/oom_score)"   "$(basename $proc)"  "$(cat $proc/cmdline | tr '\0' ' ' | head -c 50)"
done 2>/dev/null | sort -nr | head -n 40

```
输出结果如下
```
root@ubuntu16-desktop:~# sh process_order.sh 
53 20647 /opt/jdk-10.0.2/bin/java -Xms1g -Xmx1g -XX:+UseCon
46 20593 /opt/jdk-10.0.2/bin/java -Xms1g -Xmx1g -XX:+UseCon
46 20455 /opt/jdk-10.0.2/bin/java -Xms1g -Xmx1g -XX:+UseCon
37 10601 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Xmx1G 
37 10582 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Xmx1G 
37 10497 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Xmx1G 
32 13841 java -jar server-v1.0.jar 
31  6022 /usr/java/jdk1.8.0_171/bin/java -Xms1000m -Xmx1000
19 19807 java -jar -Dloader.path=.,config,resources,3rd-lib
 9  1362 nm-applet 
 8  3022 java -Duser.dir=/kafka-manager-1.3.1.8 -Dconfig.fi
 6   746 /usr/sbin/NetworkManager --no-daemon 
 4  4511 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Dzooke
 4  3024 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Dzooke
 3  3381 /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Dzooke
 2 21999 /opt/java/bin/java -Xmx20m -Dflume.root.logger=INF
 2 21553 /opt/java/bin/java -Xmx20m -Dflume.root.logger=INF
 1   946 /usr/bin/containerd 
 1 23736 grunt                            
 1  1310 /usr/sbin/unity-greeter
```

### 隐藏命令的操作
通过`export HISTCONTROL=ignorespace`  
这样只要先输入空格 后边的命令就不会被记录 

禁用当前会话的所有历史记录
如果你想禁用某个会话所有历史，你可以在开始命令行工作前简单地清除环境变量 HISTSIZE 的值即可。执行下面的命令来清除其值
export HISTSIZE=0
HISTSIZE 表示对于 bash 会话其历史列表中可以保存命令的个数（行数）。默认情况，它设置了一个非零值，例如在我的电脑上，它的值为 1000。
所以上面所提到的命令将其值设置为 0，结果就是直到你关闭终端，没有东西会存储在历史记录中。记住同样你也不能通过按向上的箭头按键或运行 history 命令来看到之前执行的命令。
<https://www.cnblogs.com/rusking/p/5715715.html>


### Linux性能分析的网站
<http://www.brendangregg.com/linuxperf.html>

### Linux中如何检测IP地址冲突问题

arping命令是用于发送arp请求到一个相邻主机的工具，arping使用arp数据包，通过ping命令检查设备上的硬件地址。能够测试一个ip地址是否是在网络上已经被使用，并能够获取更多设备信息。功能类似于ping。

arping命令是用于发送ARP请求到一个相邻主机的工具，通过ARP响应报文检查设备上的硬件地址。它能够测试一个IP地址是否是在网络上已经使用，并能够获取更多设备信息。该功能类似于ping命令。
arping命令选项：

```shell
-b：用于发送以太网广播帧（FFFFFFFFFFFF）。arping一开始使用广播地址，在收到响应后就使用unicast地址。
-q：quiet output不显示任何信息；
-f：表示在收到第一个响应报文后就退出；
-timeout：设定一个超时时间，单位是秒。如果到了指定时间，arping还没到完全收到响应则退出；
-c count：表示发送指定数量的ARP请求数据包后就停止。如果指定了deadline选项，则arping会等待相同数量的arp响应包，直到超时为止；
-s source：设定arping发送的arp数据包中的SPA字段的值。如果为空，则按下面处理，如果是DAD模式（冲突地址探测），则设置为0.0.0.0，如果是Unsolicited ARP模式（Gratutious ARP）则设置为目标地址，否则从路由表得出；
-I interface：设置ping使用的网络接口。
```
IP地址冲突检测
在出问题的主机上，可以使用"arping -I ethN x.x.x.x"命令（其中x.x.x.x为本接口的IP地址）检测地址冲突，如果没有任何输出，则表示本IP地址无冲突。如果有冲突的话，该命令会显示冲突的IP地址使用的MAC地址。

<http://man.linuxde.net/arping>

### 内存分页监控（sar -B）
例如，每10秒采样一次，连续采样3次，监控内存分页：
`sar -B 10 3`

输出项说明：
```shell
pgpgin/s：表示每秒从磁盘或SWAP置换到内存的字节数(KB)
pgpgout/s：表示每秒从内存置换到磁盘或SWAP的字节数(KB)
fault/s：每秒钟系统产生的缺页数,即主缺页与次缺页之和(major + minor)
majflt/s：每秒钟产生的主缺页数.
pgfree/s：每秒被放入空闲队列中的页个数
pgscank/s：每秒被kswapd扫描的页个数
pgscand/s：每秒直接被扫描的页个数
pgsteal/s：每秒钟从cache中被清除来满足内存需要的页个数
%vmeff：每秒清除的页(pgsteal)占总扫描页(pgscank+pgscand)的百分比
```

### SS命令
ss 是 socket statistics 的缩写。顾名思义，ss 命令可以用来获取socket 统计信息，它可以显示和netstat 类似的内容。但 ss 的优势在于它能够显示更多更详细的有关TCP和连接状态的信息，而且比netstat更快速更高效。
　　
当服务器的socket连接数量变得非常大时，无论是使用netstat命令还是 cat  /proc/net/tcp，执行速度都会很慢。可能你不会有切身的感受，但请相信我，当服务器维持的连接达到上万个的时候，使用 netstat 等于浪费生命，而用 ss才是 节省时间。

天下武功唯快不破。ss快的秘诀在于，他利用了TCP协议栈中 tcp_diag.   tcp_diag 是一个用于分析统计的模块，可以获得Linux 内核中第一手的信息，这就确保了ss的快捷高效。当然，如果你的系统中没有 tcp_diag,ss也可以正常运行，只是效率会变得稍慢。（但仍然比  netstat 要快。）

### 查找某一个时间点以后创建或者修改的文件

首先创建一个对比的时间点的文件
`touch -t 201407201710.00 abc  //创建一个文件，他的mtime是2014-07-20-17:10:00 ` 
要查找2014-07-20-17:10:00 这个时间点以后创建或者修改的文件，使用命令
`#find -newer abc           //跟刚才创建的文件对比`

### rz和sz 传输文件
rz是上传（从本地上传到服务器），可以使用rz -y实现覆盖上传
sz是下载。 实现下载可以直接使用szfilename，其中filename就是你想要下载的文件的名字，如果是目录需要打包成单个文件在实现下载。
