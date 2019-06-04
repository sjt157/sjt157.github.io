---
title: 网络之raw_socket
date: 2019-02-22 16:53:56
tags: TCP/IP
categories: TCP/IP
---



## 什么是Raw_Socket?
raw socket，即原始套接字，可以接收本机网卡上的数据帧或者数据包,对与监听网络的流量和分析是很有作用的.一共可以有3种方式创建这种socket

1.socket(AF_INET, SOCK_RAW, IPPROTO_TCP|IPPROTO_UDP|IPPROTO_ICMP)发送接收ip数据包

2.socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP|ETH_P_ARP|ETH_P_ALL))发送接收以太网数据帧

3.socket(AF_INET, SOCK_PACKET, htons(ETH_P_IP|ETH_P_ARP|ETH_P_ALL))用于抓包程序，过时了,不要用啊

(raw socket只有在Linux平台上才能发挥它本该有的功能。因为windows进行了限制，比如只能发送UDP包，只能发送正确的UDP包，不能冒充源地址，即源地址只能填本机地址)

## 通过一个例子来理解Raw_scoket的原理

比如网卡收到了一个 14+20+8+100+4 的udp的以太网数据帧.

1. 首先,网卡对该数据帧进行硬过滤(根据网卡的模式不同会有不同的动作,如果设置了promisc混杂模式的话,则不做任何过滤直接交给下一层输入例程,否则非本机mac或者广播mac会被直接丢弃).按照上面的例子,如果成功的话,会进入ip输入例程.但是在进入ip输入例程之前,系统会检查系统中是否有通过socket(AF_PACKET, SOCK_RAW, ..)创建的套接字.如果有的话并且协议相符,在这个例子中就是需要ETH_P_IP或者ETH_P_ALL类型.系统就给每个这样的socket接收缓冲区发送一个数据帧拷贝.然后进入下一步.

2. 其次,进入了ip输入例程(ip层会对该数据包进行软过滤,就是检查校验或者丢弃非本机ip或者广播ip的数据包等,具体要参考源代码),例子中就是如果成功的话会进入udp输入例程.但是在交给udp输入例程之前,系统会检查系统中是否有通过socket(AF_INET, SOCK_RAW, ..)创建的套接字.如果有的话并且协议相符,在这个例子中就是需要IPPROTO_UDP类型.系统就给每个这样的socket接收缓冲区发送一个数据帧拷贝.然后进入下一步。

3. 最后,进入udp输入例程 ...

ps:如果校验和出错的话,内核会直接丢弃该数据包的.而不会拷贝给sock_raw的套接字,因为校验和都出错了,数据肯定有问题的包括所有信息都没有意义了.

## socket的分类
根据网路的7层模型，socket可以分为3种。
* 传输层socket-最常用的socket，非raw socket
* 网络层socket - 网络层 raw socket
* MAC层socket -收发数据链路层数据

## raw socket的用途
1. 通过raw socket来接收发向本机的ICMP,IGMP协议包,或者用来发送这些协议包.
2. 接收发向本机但TCP/IP栈不能够处理的IP包：现在许多操作系统在实现网络部分的时候,通常只实现了常用的几种协议,如tcp,udp,icmp等,但象其它的如ospf,ggp等协议,操作系统往往没有实现,如果自己有必要编写位于其上的应用,就必须借助raw socket来实现,这是因为操作系统遇到自己不能够处理的数据包(ip头中的protocol所指定的上层协议不能处理)就将这个包交给协议对应的raw socket.
3. 用来发送一些自己制定源地址等特殊作用的IP包(自己写IP头,TCP头等等)，因为内核不能识别的协议、格式等将传给原始套接字，因此，可以使用原始套接字定义用户自己的协议格式

## 内核接收网络数据后在rawsocket上处理原则
 (1):对于UDP/TCP产生的IP数据包,内核不将它传递给任何网络层的原始套接字,而只是将这些数据直接交给对应的传输层UDP/TCP数据socket处理句柄。所以,只能通过MAC层原始套接字将第3个参数指定为htons(ETH_P_IP)来访问TCP/UDP数据。

 (2):对于ICMP和EGP等使用IP数据包承载数据但又在传输层之下的协议类型的IP数据包,内核不管是否已经有注册了句柄来处理这些数据,都会将这些IP数据包复制一份传递给协议类型匹配的原始套接字.（网络层套接字能截获除TCP/UDP以外传输层协议号protocol相同的ip数据） 

 (3):对于不能识别协议类型的数据包,内核进行必要的校验,然后会查看是否有类型匹配的原始套接字负责处理这些数据,如果有的话,就会将这些IP数据包复制一份传递给匹配的原始套接字,否则,内核将会丢弃这个IP数据包,并返回一个ICMP主机不可达的消息给源主机.

 (4):如果原始套接字bind绑定了一个地址,核心只将目的地址为本机IP地址的数包传递给原始套接字,如果某个原始套接字没有bind地址,核心就会把收到的所有IP数据包发给这个原始套接字. 

 (5):如果原始套接字调用了connect函数,则核心只将源地址为connect连接的IP地址的IP数据包传递给这个原始套接字. 

 (6):如果原始套接字没有调用bind和connect函数,则核心会将所有协议匹配的IP数据包传递给这个原始套接字.

## 啥是Packet套接字？？
在linux环境中要从链路层（MAC）直接收发数据帧，可以通过libpcap与libnet两个动态库来分别完成收与发的工作。虽然它已被广泛使用，但在要求进行跨平台移植的软件中使用仍然有很多弊端。

这里介绍一种更为直接地、无须安装其它库的从MAC层收发数据帧的方式，即通过定义链路层的套接字来完成。

Packet套接字用于在MAC层上收发原始数据帧，这样就允许用户在用户空间完成MAC之上各个层次的实现。给无论是进行开发还是测试的人们带来了极大的便利性。

Packet套接字的定义方式与传送层的套接字定义类似，如下：

packet_socket=socket(PF_PACKET,int socket_type,int protocol);
（这个套接字的打开需要用户有root权限）

## RAW套接字和Packet套接字的比较
* Packet套接字
需要关联到一个特定的网卡直接发送，无需经过路由查找和地址解析。这是显然的，路由查找的目的无非也就是定位到一个网卡，现在网卡已经有了，直接发送即可，至于发到了哪里，能不能到达目的地，听天由命了。
* RAW套接字
这种RAW套接字发送的报文是需要经过路由查找的，只是说IP头以及IP上层的协议以及数据可以自己构造。

## 代码：用rawSocket来进行自定义IP报文的源地址。
```c
#include <unistd.h>
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <netinet/udp.h>
#include<memory.h>
#include<stdlib.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h> // sockaddr_ll
#include<arpa/inet.h>
#include<netinet/if_ether.h>
#include<iomanip>
#include<iostream>
 
// The packet length
#define PCKT_LEN 100
 
//UDP的伪头部
struct UDP_PSD_Header
{
	u_int32_t src;
	u_int32_t des;
	u_int8_t  mbz;
	u_int8_t ptcl;
	u_int16_t len;
};
//计算校验和
unsigned short csum(unsigned short *buf, int nwords)
{ 
	unsigned long sum;
	for (sum = 0; nwords > 0; nwords--)
	{
		sum += *buf++;
	}
	sum = (sum >> 16) + (sum & 0xffff);
	sum += (sum >> 16);
	return (unsigned short)(~sum);
}
 
 
// Source IP, source port, target IP, target port from the command line arguments
int main(int argc, char *argv[])
{
	int sd;
	char buffer[PCKT_LEN] ;
	//查询www.chongfer.cn的DNS报文
	unsigned char DNS[] = { 0xd8, 0xcb , 0x01, 0x00, 0x00, 0x01, 0x00 ,0x00,
		0x00, 0x00, 0x00, 0x00, 0x03, 0x77, 0x77, 0x77,
		0x08, 0x63, 0x68, 0x6f, 0x6e, 0x67, 0x66, 0x65,
		0x72, 0x02, 0x63, 0x6e, 0x00, 0x00, 0x01, 0x00,
		0x01 };
	struct iphdr *ip = (struct iphdr *) buffer;
	struct udphdr *udp = (struct udphdr *) (buffer + sizeof(struct iphdr));
	// Source and destination addresses: IP and port
	struct sockaddr_in sin, din;
	int  one = 1;
	const int *val = &one;
	//缓存清零
	memset(buffer, 0, PCKT_LEN);
 
	if (argc != 5)
	{
		printf("- Invalid parameters!!!\n");
		printf("- Usage %s <source hostname/IP> <source port> <target hostname/IP> <target port>\n", argv[0]);
		exit(-1);
	}
 
	// Create a raw socket with UDP protocol
	sd = socket(AF_INET, SOCK_RAW, IPPROTO_UDP);
	if (sd < 0)
	{
		perror("socket() error");
		// If something wrong just exit
		exit(-1);
	}
	else
		printf("socket() - Using SOCK_RAW socket and UDP protocol is OK.\n");
	//IPPROTO_TP说明用户自己填写IP报文
	//IP_HDRINCL表示由内核来计算IP报文的头部校验和，和填充那个IP的id 
	if (setsockopt(sd, IPPROTO_IP, IP_HDRINCL, val, sizeof(int)))
	{
		perror("setsockopt() error");
		exit(-1);
	}
	else
		printf("setsockopt() is OK.\n");
 
	// The source is redundant, may be used later if needed
	// The address family
	sin.sin_family = AF_INET;
	din.sin_family = AF_INET;
	// Port numbers
	sin.sin_port = htons(atoi(argv[2]));
	din.sin_port = htons(atoi(argv[4]));
	// IP addresses
	sin.sin_addr.s_addr = inet_addr(argv[1]);
	din.sin_addr.s_addr = inet_addr(argv[3]);
 
	// Fabricate the IP header or we can use the
	// standard header structures but assign our own values.
	ip->ihl = 5;
	ip->version = 4;//报头长度，4*32=128bit=16B
	ip->tos = 0; // 服务类型
	ip->tot_len = ((sizeof(struct iphdr) + sizeof(struct udphdr)+sizeof(DNS)));
	//ip->id = htons(54321);//可以不写
	ip->ttl = 64; // hops生存周期
	ip->protocol = 17; // UDP
	ip->check = 0;
	// Source IP address, can use spoofed address here!!!
	ip->saddr = inet_addr(argv[1]);
	// The destination IP address
	ip->daddr = inet_addr(argv[3]);
 
	// Fabricate the UDP header. Source port number, redundant
	udp->source = htons(atoi(argv[2]));//源端口
	// Destination port number
	udp->dest = htons(atoi(argv[4]));//目的端口
	udp->len = htons(sizeof(struct udphdr)+sizeof(DNS));//长度
	//forUDPCheckSum用来计算UDP报文的校验和用
	//UDP校验和需要计算 伪头部、UDP头部和数据部分
	char * forUDPCheckSum = new char[sizeof(UDP_PSD_Header) + sizeof(udphdr)+sizeof(DNS)+1];
	memset(forUDPCheckSum, 0, sizeof(UDP_PSD_Header) + sizeof(udphdr) + sizeof(DNS) + 1);
	UDP_PSD_Header * udp_psd_Header = (UDP_PSD_Header *)forUDPCheckSum;
	udp_psd_Header->src = inet_addr(argv[1]);
	udp_psd_Header->des = inet_addr(argv[3]);
	udp_psd_Header->mbz = 0;
	udp_psd_Header->ptcl = 17;
	udp_psd_Header->len = htons(sizeof(udphdr)+sizeof(DNS));
	memcpy(forUDPCheckSum + sizeof(UDP_PSD_Header), udp, sizeof(udphdr));
	memcpy(forUDPCheckSum + sizeof(UDP_PSD_Header) + sizeof(udphdr), DNS, sizeof(DNS));
 
	//ip->check = csum((unsigned short *)ip, sizeof(iphdr)/2);//可以不用算
	//计算UDP的校验和，因为报文长度可能为单数，所以计算的时候要补0
	udp->check = csum((unsigned short *)forUDPCheckSum,(sizeof(udphdr)+sizeof(UDP_PSD_Header)+sizeof(DNS)+1)/2);
 
	setuid(getpid());//如果不是root用户，需要获取权限	
 
	// Send loop, send for every 2 second for 2000000 count
	printf("Trying...\n");
	printf("Using raw socket and UDP protocol\n");
	printf("Using Source IP: %s port: %u, Target IP: %s port: %u.\n", argv[1], atoi(argv[2]), argv[3], atoi(argv[4]));
	std::cout << "Ip length:" << ip->tot_len << std::endl;
	int count;
	//将DNS报文拷贝进缓存区
	memcpy(buffer + sizeof(iphdr) + sizeof(udphdr), DNS, sizeof(DNS));
	
	for (count = 1; count <= 2000000; count++)
	{
		
		if (sendto(sd, buffer, ip->tot_len, 0, (struct sockaddr *)&din, sizeof(din)) < 0)
			// Verify
		{
			perror("sendto() error");
			exit(-1);
		}
		else
		{
			printf("Count #%u - sendto() is OK.\n", count);
			sleep(2);
		}
	}
	close(sd);
	return 0;
}

```

结果：在IP为192.168.1.50的机器上冒充那个IP为192.168.1.71的机器向阿里的DNS服务器114.114.114.114上发送DNS查询报文
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Raw_socker/1.png)

在IP为192.168.1.71的机器上收到阿里DNS服务器传回来的DNS报文
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Raw_socker/2.png)

为啥是 unreachable - admin prohibited, length 413，经过查询是因为防火墙的原因。

> 关闭centos防火墙，service iptables stop
关闭ubuntu防火墙，ufw stop。一般用户，只需如下设置：
sudo apt-get install ufw
sudo ufw enable
sudo ufw default deny
以上三条命令已经足够安全了，如果你需要开放某些服务，再使用sudo ufw allow开启。

关闭防火墙之后，再次查看，如下结果：
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Raw_socker/3.png)

## 参考：
<https://blog.csdn.net/firefoxbug/article/details/7561159>
<https://blog.51cto.com/a1liujin/1697465>
<https://blog.csdn.net/yong61/article/details/8549654>
<https://blog.csdn.net/dog250/article/details/83830872>
<https://www.cnblogs.com/sammyliu/p/4981194.html>
<https://blog.csdn.net/luchengtao11/article/details/73878760>
