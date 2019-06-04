---
title: Linux之iptables
date: 2019-03-23 11:20:53
tags: Linux
categories: Linux
---
昨天读了一篇文章说，服务器不能把防火墙关闭。那么我在unbuntu上通过ufw enable开启了防火墙，然后我执行了以下命令，懵逼了，这都是啥啊。

### Filter表

主要和主机自身有关，真正负责主机防火墙功能（过滤流入，流出，主机的数据包）。filter是主机默认使用的表。这表定义了三个链。生产场景单台，服务器的防火墙功能全靠这张表。

* INPUT：负责过滤进入主机的数据包

* FORWARD：负责转发流进主机的数据包，起转发的作用，和NAT关系很大。 net.ipv4.ip_forward=0 这个参数很重要，有的场景需要开启。echo 1  > /proc/sys/net/ipv4/ip_forward暂时开始路由转发。

* OUTPUT ： 就是处理从主机发出去的数据包

```shell
root@ubuntu16-desktop:~# iptables -nL //列表形式显示默认表(filter)的所有信息 L list 列表 -n number 数字
Chain INPUT (policy DROP)
target     prot opt source               destination         
ufw-before-logging-input  all  --  0.0.0.0/0            0.0.0.0/0 （表示anywhere）          
ufw-before-input  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-input  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-logging-input  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-reject-input  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-track-input  all  --  0.0.0.0/0            0.0.0.0/0           

//默认不能进行转发，就是从容器里边 到主机，默认是不行的。如果想要实现不同主机上的容器进行通信，需要对此进行修改。但是我做了下实验，开不开起net.ipv4.ip_forward，容器和主机都可以相互ping通啊。
Chain FORWARD (policy DROP)
target     prot opt source               destination         
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0      
     
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ufw-before-logging-forward  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-before-forward  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-forward  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-logging-forward  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-reject-forward  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-track-forward  all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
ufw-before-logging-output  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-before-output  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-output  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-after-logging-output  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-reject-output  all  --  0.0.0.0/0            0.0.0.0/0           
ufw-track-output  all  --  0.0.0.0/0            0.0.0.0/0           

Chain DOCKER (4 references)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.217        tcp dpt:9092
ACCEPT     tcp  --  0.0.0.0/0            172.17.0.2           tcp dpt:9000
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.217        tcp dpt:3000
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.216        tcp dpt:9092
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.215        tcp dpt:9092
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.213        tcp dpt:2181
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.218        tcp dpt:9300
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.218        tcp dpt:9200
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.214        tcp dpt:2181
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.212        tcp dpt:2181
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.1          tcp dpt:8086
ACCEPT     tcp  --  0.0.0.0/0            172.18.20.3          tcp dpt:3000
ACCEPT     tcp  --  0.0.0.0/0            172.17.0.6           tcp dpt:3306

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination         
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

//默认drop
Chain DOCKER-ISOLATION-STAGE-2 (4 references)
target     prot opt source               destination         
DROP       all  --  0.0.0.0/0            0.0.0.0/0           
DROP       all  --  0.0.0.0/0            0.0.0.0/0           
DROP       all  --  0.0.0.0/0            0.0.0.0/0           
DROP       all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-after-forward (1 references)
target     prot opt source               destination         

Chain ufw-after-input (1 references)
target     prot opt source               destination         
ufw-skip-to-policy-input  udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:137
ufw-skip-to-policy-input  udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:138
ufw-skip-to-policy-input  tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:139
ufw-skip-to-policy-input  tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:445
ufw-skip-to-policy-input  udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:67
ufw-skip-to-policy-input  udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:68
ufw-skip-to-policy-input  all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type BROADCAST

Chain ufw-after-logging-forward (1 references)
target     prot opt source               destination         
LOG        all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 10 LOG flags 0 level 4 prefix "[UFW BLOCK] "

Chain ufw-after-logging-input (1 references)
target     prot opt source               destination         
LOG        all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 10 LOG flags 0 level 4 prefix "[UFW BLOCK] "

Chain ufw-after-logging-output (1 references)
target     prot opt source               destination         

Chain ufw-after-output (1 references)
target     prot opt source               destination         

Chain ufw-before-forward (1 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 3
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 4
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 11
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 12
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ufw-user-forward  all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-before-input (1 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ufw-logging-deny  all  --  0.0.0.0/0            0.0.0.0/0            ctstate INVALID
DROP       all  --  0.0.0.0/0            0.0.0.0/0            ctstate INVALID
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 3
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 4
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 11
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 12
ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:67 dpt:68
ufw-not-local  all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     udp  --  0.0.0.0/0            224.0.0.251          udp dpt:5353
ACCEPT     udp  --  0.0.0.0/0            239.255.255.250      udp dpt:1900
ufw-user-input  all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-before-logging-forward (1 references)
target     prot opt source               destination         

Chain ufw-before-logging-input (1 references)
target     prot opt source               destination         

Chain ufw-before-logging-output (1 references)
target     prot opt source               destination         

Chain ufw-before-output (1 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ufw-user-output  all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-logging-allow (0 references)
target     prot opt source               destination         
LOG        all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 10 LOG flags 0 level 4 prefix "[UFW ALLOW] "

Chain ufw-logging-deny (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0            ctstate INVALID limit: avg 3/min burst 10
LOG        all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 10 LOG flags 0 level 4 prefix "[UFW BLOCK] "

Chain ufw-not-local (1 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
RETURN     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type MULTICAST
RETURN     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type BROADCAST
ufw-logging-deny  all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 10
DROP       all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-reject-forward (1 references)
target     prot opt source               destination         

Chain ufw-reject-input (1 references)
target     prot opt source               destination         

Chain ufw-reject-output (1 references)
target     prot opt source               destination         

Chain ufw-skip-to-policy-forward (0 references)
target     prot opt source               destination         
DROP       all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-skip-to-policy-input (7 references)
target     prot opt source               destination         
DROP       all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-skip-to-policy-output (0 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-track-forward (1 references)
target     prot opt source               destination         

Chain ufw-track-input (1 references)
target     prot opt source               destination         

Chain ufw-track-output (1 references)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW
ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW

Chain ufw-user-forward (1 references)
target     prot opt source               destination         

Chain ufw-user-input (1 references)
target     prot opt source               destination         

Chain ufw-user-limit (0 references)
target     prot opt source               destination         
LOG        all  --  0.0.0.0/0            0.0.0.0/0            limit: avg 3/min burst 5 LOG flags 0 level 4 prefix "[UFW LIMIT BLOCK] "
REJECT     all  --  0.0.0.0/0            0.0.0.0/0            reject-with icmp-port-unreachable

Chain ufw-user-limit-accept (0 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain ufw-user-logging-forward (0 references)
target     prot opt source               destination         

Chain ufw-user-logging-input (0 references)
target     prot opt source               destination         

Chain ufw-user-logging-output (0 references)
target     prot opt source               destination         

Chain ufw-user-output (1 references)
target     prot opt source               destination  

```
### NAT表
```shell
root@ubuntu16-desktop:~# iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

//MASQUERADE就是针对这种场景而设计的，他的作用是，从服务器的网卡上，自动获取当前ip地址来做NAT
比如下边的命令：
iptables -t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j MASQUERADE
如此配置的话，不用指定SNAT的目标ip了
不管现在eth0的出口获得了怎样的动态ip，MASQUERADE会自动读取eth0现在的ip地址然后做SNAT出去
这样就实现了很好的动态SNAT地址转换

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0    //MASQUERADE为地址伪装，在iptables中有着和SNAT相近的效果       
MASQUERADE  all  --  172.28.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.18.0.0/16        0.0.0.0/0           
MASQUERADE  all  --  172.29.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.18.20.217        172.18.20.217        tcp dpt:9092
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:9000
MASQUERADE  tcp  --  172.18.20.217        172.18.20.217        tcp dpt:3000
MASQUERADE  tcp  --  172.18.20.216        172.18.20.216        tcp dpt:9092
MASQUERADE  tcp  --  172.18.20.215        172.18.20.215        tcp dpt:9092
MASQUERADE  tcp  --  172.18.20.213        172.18.20.213        tcp dpt:2181
MASQUERADE  tcp  --  172.18.20.218        172.18.20.218        tcp dpt:9300
MASQUERADE  tcp  --  172.18.20.218        172.18.20.218        tcp dpt:9200
MASQUERADE  tcp  --  172.18.20.214        172.18.20.214        tcp dpt:2181
MASQUERADE  tcp  --  172.18.20.212        172.18.20.212        tcp dpt:2181
MASQUERADE  tcp  --  172.18.20.1          172.18.20.1          tcp dpt:8086
MASQUERADE  tcp  --  172.18.20.3          172.18.20.3          tcp dpt:3000
MASQUERADE  tcp  --  172.17.0.6           172.17.0.6           tcp dpt:3306

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9094 to:172.18.20.217:9092
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9000 to:172.17.0.2:9000
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3000 to:172.18.20.217:3000
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9093 to:172.18.20.216:9092
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9092 to:172.18.20.215:9092
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2185 to:172.18.20.213:2181
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9300 to:172.18.20.218:9300
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:9200 to:172.18.20.218:9200
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2186 to:172.18.20.214:2181
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2184 to:172.18.20.212:2181
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8086 to:172.18.20.1:8086
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:5000 to:172.18.20.3:3000
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3306 to:172.17.0.6:3306

```


### ubuntu中查看规则
iptables-save

```shell
# Generated by iptables-save v1.6.0 on Fri Mar 22 03:55:53 2019
*nat
:PREROUTING ACCEPT [4895:635241]
:INPUT ACCEPT [3858:547896]
:OUTPUT ACCEPT [2016:126826]
:POSTROUTING ACCEPT [2018:126930]
:DOCKER - [0:0]
//这条规则把目标地址类型属于主机系统的本地网络地址的数据包，在数据包进入NAT表PREROUTING链时，都让它们直接jump到一个名为DOCKER的链。
//iptables提供了众多的扩展模块，以支持更多的功能。addrtype就是这样的一个扩展模块，提供的是Address type match的功能。引用的方式就是 -m 模块名。比如还有conntrack模块。iptables -m addrtype --help查看帮助信息。
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER 

-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER

//这条规则是处理docker的SNAT地址伪装用的。具体含义是将容器网络网段发送到外部的数据包(!-o docker0)伪装成宿主机的ip，就是讲数据包的原来的容器ip换成了宿主机ip，做了一次snat。
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

-A POSTROUTING -s 172.28.0.0/16 ! -o br-c333d38dcc9f -j MASQUERADE
//我自己建的网桥
-A POSTROUTING -s 172.18.0.0/16 ! -o br-fd7621436686 -j MASQUERADE
-A POSTROUTING -s 172.29.0.0/16 ! -o br-f8eb065b6c3a -j MASQUERADE
-A POSTROUTING -s 172.18.20.217/32 -d 172.18.20.217/32 -p tcp -m tcp --dport 9092 -j MASQUERADE
-A POSTROUTING -s 172.17.0.2/32 -d 172.17.0.2/32 -p tcp -m tcp --dport 9000 -j MASQUERADE
-A POSTROUTING -s 172.18.20.217/32 -d 172.18.20.217/32 -p tcp -m tcp --dport 3000 -j MASQUERADE
-A POSTROUTING -s 172.18.20.216/32 -d 172.18.20.216/32 -p tcp -m tcp --dport 9092 -j MASQUERADE
-A POSTROUTING -s 172.18.20.215/32 -d 172.18.20.215/32 -p tcp -m tcp --dport 9092 -j MASQUERADE
-A POSTROUTING -s 172.18.20.213/32 -d 172.18.20.213/32 -p tcp -m tcp --dport 2181 -j MASQUERADE
-A POSTROUTING -s 172.18.20.218/32 -d 172.18.20.218/32 -p tcp -m tcp --dport 9300 -j MASQUERADE
-A POSTROUTING -s 172.18.20.218/32 -d 172.18.20.218/32 -p tcp -m tcp --dport 9200 -j MASQUERADE
-A POSTROUTING -s 172.18.20.214/32 -d 172.18.20.214/32 -p tcp -m tcp --dport 2181 -j MASQUERADE
-A POSTROUTING -s 172.18.20.212/32 -d 172.18.20.212/32 -p tcp -m tcp --dport 2181 -j MASQUERADE
-A POSTROUTING -s 172.18.20.1/32 -d 172.18.20.1/32 -p tcp -m tcp --dport 8086 -j MASQUERADE
-A POSTROUTING -s 172.18.20.3/32 -d 172.18.20.3/32 -p tcp -m tcp --dport 3000 -j MASQUERADE
-A POSTROUTING -s 172.17.0.6/32 -d 172.17.0.6/32 -p tcp -m tcp --dport 3306 -j MASQUERADE

//docker启动，分别在filter和nat建立了名为DOCKER的chain，在forward转发链增加了一些ACCEPT规则，在nat增加了postrouting和prerouting以及output的规则。docker创建了一个名为DOKCER的自定义的链条Chain,iptables自定义链条的好处就是可以让防火墙的策略更加的层次化… …
-A DOCKER -i docker0 -j RETURN
-A DOCKER -i br-c333d38dcc9f -j RETURN
-A DOCKER -i br-fd7621436686 -j RETURN
-A DOCKER -i br-f8eb065b6c3a -j RETURN

//如下命令表示把所有10.8.0.0网段的数据包SNAT成192.168.5.3的ip然后发出去(这里没有这个规则)
//iptables -t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j SNAT –to-source 192.168.5.3

-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 9094 -j DNAT --to-destination 172.18.20.217:9092
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 9000 -j DNAT --to-destination 172.17.0.2:9000
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 3000 -j DNAT --to-destination 172.18.20.217:3000
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 9093 -j DNAT --to-destination 172.18.20.216:9092
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 9092 -j DNAT --to-destination 172.18.20.215:9092
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 2185 -j DNAT --to-destination 172.18.20.213:2181
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 9300 -j DNAT --to-destination 172.18.20.218:9300
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 9200 -j DNAT --to-destination 172.18.20.218:9200
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 2186 -j DNAT --to-destination 172.18.20.214:2181
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 2184 -j DNAT --to-destination 172.18.20.212:2181
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 8086 -j DNAT --to-destination 172.18.20.1:8086
-A DOCKER ! -i br-fd7621436686 -p tcp -m tcp --dport 5000 -j DNAT --to-destination 172.18.20.3:3000
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 3306 -j DNAT --to-destination 172.17.0.6:3306
COMMIT
# Completed on Fri Mar 22 03:55:53 2019
# Generated by iptables-save v1.6.0 on Fri Mar 22 03:55:53 2019


*filter
:INPUT DROP [31:868]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:DOCKER - [0:0]
:DOCKER-ISOLATION-STAGE-1 - [0:0]
:DOCKER-ISOLATION-STAGE-2 - [0:0]
:DOCKER-USER - [0:0]
:ufw-after-forward - [0:0]
:ufw-after-input - [0:0]
:ufw-after-logging-forward - [0:0]
:ufw-after-logging-input - [0:0]
:ufw-after-logging-output - [0:0]
:ufw-after-output - [0:0]
:ufw-before-forward - [0:0]
:ufw-before-input - [0:0]
:ufw-before-logging-forward - [0:0]
:ufw-before-logging-input - [0:0]
:ufw-before-logging-output - [0:0]
:ufw-before-output - [0:0]
:ufw-logging-allow - [0:0]
:ufw-logging-deny - [0:0]
:ufw-not-local - [0:0]
:ufw-reject-forward - [0:0]
:ufw-reject-input - [0:0]
:ufw-reject-output - [0:0]
:ufw-skip-to-policy-forward - [0:0]
:ufw-skip-to-policy-input - [0:0]
:ufw-skip-to-policy-output - [0:0]
:ufw-track-forward - [0:0]
:ufw-track-input - [0:0]
:ufw-track-output - [0:0]
:ufw-user-forward - [0:0]
:ufw-user-input - [0:0]
:ufw-user-limit - [0:0]
:ufw-user-limit-accept - [0:0]
:ufw-user-logging-forward - [0:0]
:ufw-user-logging-input - [0:0]
:ufw-user-logging-output - [0:0]
:ufw-user-output - [0:0]
-A INPUT -j ufw-before-logging-input
-A INPUT -j ufw-before-input
-A INPUT -j ufw-after-input
-A INPUT -j ufw-after-logging-input
-A INPUT -j ufw-reject-input
-A INPUT -j ufw-track-input
//filter表中的forward主要是对容器和宿主机以及本机容器之间数据包的放行。
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -o br-c333d38dcc9f -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o br-c333d38dcc9f -j DOCKER
-A FORWARD -i br-c333d38dcc9f ! -o br-c333d38dcc9f -j ACCEPT
-A FORWARD -i br-c333d38dcc9f -o br-c333d38dcc9f -j ACCEPT
-A FORWARD -o br-fd7621436686 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o br-fd7621436686 -j DOCKER
-A FORWARD -i br-fd7621436686 ! -o br-fd7621436686 -j ACCEPT
-A FORWARD -i br-fd7621436686 -o br-fd7621436686 -j ACCEPT
-A FORWARD -o br-f8eb065b6c3a -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o br-f8eb065b6c3a -j DOCKER
-A FORWARD -i br-f8eb065b6c3a ! -o br-f8eb065b6c3a -j ACCEPT
-A FORWARD -i br-f8eb065b6c3a -o br-f8eb065b6c3a -j ACCEPT
-A FORWARD -j ufw-before-logging-forward
-A FORWARD -j ufw-before-forward
-A FORWARD -j ufw-after-forward
-A FORWARD -j ufw-after-logging-forward
-A FORWARD -j ufw-reject-forward
-A FORWARD -j ufw-track-forward
-A OUTPUT -j ufw-before-logging-output
-A OUTPUT -j ufw-before-output
-A OUTPUT -j ufw-after-output
-A OUTPUT -j ufw-after-logging-output
-A OUTPUT -j ufw-reject-output
-A OUTPUT -j ufw-track-output
-A DOCKER -d 172.18.20.217/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 9092 -j ACCEPT
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 9000 -j ACCEPT
-A DOCKER -d 172.18.20.217/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 3000 -j ACCEPT
-A DOCKER -d 172.18.20.216/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 9092 -j ACCEPT
-A DOCKER -d 172.18.20.215/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 9092 -j ACCEPT
-A DOCKER -d 172.18.20.213/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 2181 -j ACCEPT
-A DOCKER -d 172.18.20.218/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 9300 -j ACCEPT
-A DOCKER -d 172.18.20.218/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 9200 -j ACCEPT
-A DOCKER -d 172.18.20.214/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 2181 -j ACCEPT
-A DOCKER -d 172.18.20.212/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 2181 -j ACCEPT
-A DOCKER -d 172.18.20.1/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 8086 -j ACCEPT
-A DOCKER -d 172.18.20.3/32 ! -i br-fd7621436686 -o br-fd7621436686 -p tcp -m tcp --dport 3000 -j ACCEPT
-A DOCKER -d 172.17.0.6/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 3306 -j ACCEPT
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -i br-c333d38dcc9f ! -o br-c333d38dcc9f -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -i br-fd7621436686 ! -o br-fd7621436686 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -i br-f8eb065b6c3a ! -o br-f8eb065b6c3a -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -o br-c333d38dcc9f -j DROP
-A DOCKER-ISOLATION-STAGE-2 -o br-fd7621436686 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -o br-f8eb065b6c3a -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
-A DOCKER-USER -j RETURN
-A ufw-after-input -p udp -m udp --dport 137 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp -m udp --dport 138 -j ufw-skip-to-policy-input
-A ufw-after-input -p tcp -m tcp --dport 139 -j ufw-skip-to-policy-input
-A ufw-after-input -p tcp -m tcp --dport 445 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp -m udp --dport 67 -j ufw-skip-to-policy-input
-A ufw-after-input -p udp -m udp --dport 68 -j ufw-skip-to-policy-input
-A ufw-after-input -m addrtype --dst-type BROADCAST -j ufw-skip-to-policy-input
-A ufw-after-logging-forward -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW BLOCK] "
-A ufw-after-logging-input -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW BLOCK] "
-A ufw-before-forward -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-forward -p icmp -m icmp --icmp-type 3 -j ACCEPT
-A ufw-before-forward -p icmp -m icmp --icmp-type 4 -j ACCEPT
-A ufw-before-forward -p icmp -m icmp --icmp-type 11 -j ACCEPT
-A ufw-before-forward -p icmp -m icmp --icmp-type 12 -j ACCEPT
-A ufw-before-forward -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A ufw-before-forward -j ufw-user-forward
-A ufw-before-input -i lo -j ACCEPT
-A ufw-before-input -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-input -m conntrack --ctstate INVALID -j ufw-logging-deny
-A ufw-before-input -m conntrack --ctstate INVALID -j DROP
-A ufw-before-input -p icmp -m icmp --icmp-type 3 -j ACCEPT
-A ufw-before-input -p icmp -m icmp --icmp-type 4 -j ACCEPT
-A ufw-before-input -p icmp -m icmp --icmp-type 11 -j ACCEPT
-A ufw-before-input -p icmp -m icmp --icmp-type 12 -j ACCEPT
-A ufw-before-input -p icmp -m icmp --icmp-type 8 -j ACCEPT
-A ufw-before-input -p udp -m udp --sport 67 --dport 68 -j ACCEPT
-A ufw-before-input -j ufw-not-local
-A ufw-before-input -d 224.0.0.251/32 -p udp -m udp --dport 5353 -j ACCEPT
-A ufw-before-input -d 239.255.255.250/32 -p udp -m udp --dport 1900 -j ACCEPT
-A ufw-before-input -j ufw-user-input
-A ufw-before-output -o lo -j ACCEPT
-A ufw-before-output -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ufw-before-output -j ufw-user-output
-A ufw-logging-allow -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW ALLOW] "
-A ufw-logging-deny -m conntrack --ctstate INVALID -m limit --limit 3/min --limit-burst 10 -j RETURN
-A ufw-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW BLOCK] "
-A ufw-not-local -m addrtype --dst-type LOCAL -j RETURN
-A ufw-not-local -m addrtype --dst-type MULTICAST -j RETURN
-A ufw-not-local -m addrtype --dst-type BROADCAST -j RETURN
-A ufw-not-local -m limit --limit 3/min --limit-burst 10 -j ufw-logging-deny
-A ufw-not-local -j DROP
-A ufw-skip-to-policy-forward -j DROP
-A ufw-skip-to-policy-input -j DROP
-A ufw-skip-to-policy-output -j ACCEPT
-A ufw-track-output -p tcp -m conntrack --ctstate NEW -j ACCEPT
-A ufw-track-output -p udp -m conntrack --ctstate NEW -j ACCEPT
-A ufw-user-limit -m limit --limit 3/min -j LOG --log-prefix "[UFW LIMIT BLOCK] "
-A ufw-user-limit -j REJECT --reject-with icmp-port-unreachable
-A ufw-user-limit-accept -j ACCEPT
COMMIT
# Completed on Fri Mar 22 03:55:53 2019


```
### SNAT与DNAT
SNAT，DNAT，MASQUERADE都是NAT，MASQUERADE是SNAT的一个特例。
SNAT是指在数据包从网卡发送出去的时候，把数据包中的源地址部分替换为指定的IP，这样，接收方就认为数据包的来源是被替换的那个IP的主机。

MASQUERADE是用发送数据的网卡上的IP来替换源IP，因此，对于那些IP不固定的场合，比如拨号网络或者通过dhcp分配IP的情况下，就得用MASQUERADE。

DNAT，就是指数据包从网卡发送出去的时候，修改数据包中的目的IP，表现为如果你想访问A，可是因为网关做了DNAT，把所有访问A的数据包的目的IP全部修改为B，那么，你实际上访问的是B。

### centos中的防火墙

CentOS 6使用iptables作为防火墙，CentOS 7使用firewalld作为防火墙。
* CentOS 7关闭防火墙运行：
systemctl stop firewalld.service
* CentOS 7禁止防火墙开机自动运行：
systemctl disable firewalld.service
* 临时允许443/TCP端口,立即生效
firewall-cmd --add-port=443/tcp
* 永久允许443/TCP端口,重启防火墙后生效
firewall-cmd --permanent --add-port=443/tcp
* 永久打开端口需要reload一下，如果用了reload临时打开的端口就失效了
firewall-cmd --reload
* 查看防火墙所有区域的设置，包括添加的端口和服务
 firewall-cmd --list-all

**firewalld在rule上配置非常不友好，k8s不支持firewalld，还是需要iptables来处理**


### 在服务端禁止掉,客户端ssh的访问

```shell
iptables -t filter -A INPUT -p tcp --dport 22 -j DORP  也可以不指定表 默认是filter 表

iptables -t 指定表

Iptables -A 添加规则到指定链的结尾，最后一条  append 添加

iptables -I   是添加规则到指定链的开头，第一条 insert 插入 插到第一行 首先执行

iptables -p   指定协议  默认 不写 指的所有端口 

iptables -J 指定动作  ACCEPT 接受 DROP 丢弃 REJECT 拒绝（拒绝会透漏信息一般不用） 

iptables --line-numbers  显示序号

禁掉端口号 之后，跑到机房 VMworkstations关闭防火墙  -F清除规则 或者重启 即可

iptables -t filter -A INPUT -p tcp --deport 52113 -j DROP
```

### 允许80端口
iptables -A INPUT -p tcp --dport 80 -j ACCEPT

### 允许其他人能ping通
iptables -A INPUT -p icmp -m icmp --icmp-type any -j ACCEPT

### 允许关联的包通过
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT


### 如何永久保存
我们在命令行中的操作，都是储存在内存当中，没有在磁盘，重启服务器就会丢失

我们要将其存到配置文件，做到永久存储

* 方法一：/etc/init.d/iptables save
* 方法二:重定向iptables-save>/etc/sysconfig/iptables.bak


### 如何查看 所有端口

nmap host的IP  -p 1-65535

### ubuntu相关命令
* 允许ssh
ufw allow 22
* 删除已经添加过的规则：
sudo ufw delete allow 22
只打开使用tcp/ip协议的22端口：
* udo ufw allow 22/tcp
打开来自192.168.0.1的tcp请求的80端口：
* sudo ufw allow proto tcp from 192.168.0.1 to any port 22
允许某特定 IP
* sudo ufw allow from 192.168.254.254


