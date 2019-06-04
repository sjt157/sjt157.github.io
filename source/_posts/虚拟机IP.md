---
title: 虚拟机IP
date: 2018-03-15 10:54:12
tags: KVM
categories: KVM
---
### 如何不进去虚拟机看到其IP
1. 在linux上玩过kvm的朋友基本都晓得，在宿主机上运行了虚拟主机以后，我们无法直接看到某一个虚拟主机IP地址。比如
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-IP/1.png)

如果我们想知道test这个虚拟机的IP地址，那么是无法直接看到的。但是我们可以通过一个小技巧间接得到其IP，也就是利用ARP。
2. 编辑虚拟主机配置文件。
查看虚拟机的配置文件test.xml，（或者通过`virsh dumpxml test`）找到MAC标签，可以得到该虚拟机的MAC地址，记录下mac后退出，然后通过arp -a判定虚拟机IP地址，由此可以发现此虚拟机IP为192.168.122.118，由此实现了不登陆虚拟机即可得到其IP。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-IP/2.png)

(注意这里一定要加上-i 忽略大小写。不然因为大小写问题有可能查不到)
说明：这里只根据通信缓存记录的mac 、IP地址手段做排查。也有可能找不到。最好的办法是自己写一个脚本跟网段内的所有服务器都ping一次，记录下mac、ip地址以后再查找就没问题,因为这个方法就根据ARP缓存得到的。


### 虚拟机指定固定IP (方法1)
1. 由于bridge模式默认创建虚拟机的时候用的DHCP方式，分配给虚拟机的IP可能会发生改变，有时候很麻烦。那么如何指定其IP呢？
进去虚拟机，编辑/etc/network/interface文件，由
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-IP/3.png)
变成
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-IP/4.png)
然后输入 systemctl restart networking或者实在不行重启，即可更改为192.168.1.111的地址。

### 虚拟机指定固定IP (方法2)
* 把MAC与IP之间进行绑定,配置DHCP的配置文件
```
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#
 
# Use this to enble / disable dynamic dns updates globally.
ddns-update-style none;
 
ignore client-updates;
 
# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
#authoritative;
 
# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;
 
# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.
# This is a very basic subnet declaration.
 
# A slightly different configuration for an internal subnet.
subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.30 192.168.0.39;
  option domain-name-servers 192.168.0.31;
  option domain-name "wan.hust.china";
  option routers 192.168.0.1;
  option broadcast-address 192.168.0.255;
  default-lease-time 21600;
  max-lease-time 43200;
 
  host pc001 {
	  hardware ethernet 66:66:66:66:66:0b;
	  fixed-address 192.168.0.88;
  }
}
```
* 如今就能够通过在创建虚拟机时指定MAC地址来间接指定IP地址了：
```python
 /usr/local/qemu-kemari-v0.2.14/bin/qemu-system-x86_64 -m 1024 /images/test2.img -net nic,mac=66:66:66:66:66:0b -net tap,ifname=tap1,script=/etc/qemu-ifup,downscript=no -vnc :6 -enable-kvm

```

### 注意的问题
这里要注意：当两台虚拟机都指定同一个与特定IP绑定的MAC地址时。DHCP并不报错。而是给两台虚拟机都分配这个特定的IP。
以下是我在网上找到的一些帖子。来解释同一个网段内能够有两台电脑拥有同样的IP和MAC地址：

事实上是能够的，你全然能够把两台电脑的IP 和MAC改成一样。不但能够上网并且还没IP冲突。
这样的方法不但能够突破路由封锁用在ADSL共享上网。并且还能够用在IEEE802.1X认证上网的环境中。可是前提必需要用同样的帐号来拨号上网（前提认证server没设验证帐号的反复性）。我的机子是通过学校校园网接入internet的，client採用802.1x认证client软件“STAR Supplicant拨号软件”来拨号上网，在我们学校里能够将两台机子的IP和MAC改成一样然后用同样的一个帐号来达到共享上网的目的，只是在我们学校仅仅能够在同一个宿舍的两台机子才干够共享上网，由于我们学校的server不单止验证帐号,ip,MAC并且还验证接入serverIP（NAS
IP），和接入serverport（NAS port)，不同的宿舍接在学校交换机不同的port，所以仅仅限于同一个宿舍用这样的法共享上网。

　　至于为什么不会引起IP冲突并且还能上网,这是由于ARP工作的缺陷，系统之所以会发现网上有相的IP的而提示“IP冲突”，是由于系统在启动时,TCP/IP中的ARP会广播一个免费ARP(free arp)请求包到网段上，这个ARP(free arp)包包括自己的IP和MAC。假设网段上有机子回应了这个包，这台发广播的机子就会觉得局域网有别的机子使用和自己同样的IP。

　　比如：PC A和PC B的IP和MAC全然一样，PC A的系统启动时会广播一个包括自己IP和MAC的免费ARP(free arp)请求包到网段上，假设PC B回应了这个请求。PC A会觉得自己的IP和网络上的IP有冲突并发出提示（这就是为什么IP冲突一般发生系统刚启动完毕时），问题是PC B根本不会回应这个请求包。这是由于这个请求包的IP和MAC和PC B自己的全然一样，而PC B会觉得是自己发的包。所以不会回应，既然不会回应自然不会发生IP冲突了。

　　好了。让我来解释下一个问题。就是两台机子的IP和MAC一样究竟会不会导致不能上网：

既然能够。那么网络上的硬件设备是如何区份这些数据究竟是哪台机的呢？？大家都知道局域网内是用硬件地址来通迅的，局域网的二层设备(如交换机）维护着一张地址表，地址表记录着本设备每一个port所相应的MAC（注：不是port的MAC，而是port所连设备的MAC）。设备要经过地址学习状态才干知道这些port所相应的MAC，当一个帧经过设备的某介port时，设备会检查该帧源地址和目的地址,然后再对比自己的地址表，看地址表中是否存在该源地址的相应项。若不存在则port会变为“地址学习状态”，将该地址保存在地址表中组成一个新的表项。假设PCA和PCB都连在同一个交换机上。则交换机经过“地址学习状态”后，地址表中存在两个同样的地址项，只是它们所相应的port是不同的。当交换机在外部接收到一个目的地址为该地址（PCA和PCB同样的MAC地址）的帧时，则会检查地址表。检查地址表后会发现存在两个同样地址的表项。于是交换机会将该帧转发到这两个表项所相应的port，(至于交换机是用组播的方式还是说用一个帧发两遍的方式转发给这两个port我就不太清楚了)。

　　路由器也一样。不同是的路由器的地址表是路由表。存放的是IP而不是硬件地址。

　　连接这两个port的PCA和PCB都会收到相同的帧，既然会收到相同的帧，那么计算机如何才知道哪些帧才是自己想要的呢？这取决于工作在TCP/ip上层协议。尽管网卡是接收了这个帧，可是上层的协议进行进一步的分用，也能够说成是过滤，当TCP/IP的网络接口层(也叫链路层)收到一个帧，会检查帧头中的帧类型，假设是ARP类型的就交给ARP协议去处理，假设是RARP类型就会交给RARP协议处理。假设是IP类型会去掉帧头并把这个帧传给上一层（即网络层来处理）。网络层会依据包头（去掉帧头就叫IP包了）中的协议类型来分用。如是TCMP类型就交给ICMP协议处理。假设是IGMP类型就交给IGMP协议处理。假设是TCP或UDP就把包头去掉并交给上一层（即传输层）来片理
，去掉IP包头后就叫做报文分段了（传输层的单位），相同传输层也会对报文分段的头部进行检查从而进行进一步的分用，假设是TCP类型的交给TCP协议处理，假设是UDP类型就交给UDP协议处理，TCP或UDP会依据报文分段的头部中的“目的port号”来交给应用层（交给应用层前会把报文分段的头部去掉），然后应用层的用户进程会依据该“port号”来决是否接收这个数据，比如QQ某个进程打开了UDP 1324这个port，传输层的UDP协议会把全部接收到的且“目的port号”为1324的报文分段交给QQ的这个进程。 这样就完毕接收数据的整个过程。尽管两台电脑都会接收到不是属于自己的数据帧。可是在把帧交给上层协议片理时有可能会被丢充。就如应用层的QQ进程不会接到除“目的port号”为1324以外的其他数据包，由于这些数据在应用层前已经被丢弃。

在同一个局域网内理论上是同意同样MAC地址存在的。为什么使用同样的MAC地址的PC都能在同一网段内执行呢？首先。我们要明白的是，局域网内的通信是以帧为基础的，也就是我们通常说的MAC地址。而不是IP地址。其次，路由器（特指平时我们家庭用的那种路由，如tplink等）或交换机（如cisco）在局域网内是依据mac-table进行数据的交换。
并且这些表都有特定的生存期。不是静态的。

如今如果，有两台PC（PC1和PC2）的MAC地址同样。分别接在路由器（或交换机）的两个端口（port1，port2）上，pc1首先发起连接魔兽游戏的server的请求，那么在路由器（或交换机）上就会在mac-table加入PC1的mac地址到port1上。当魔兽游戏的server反应请求后，路由器也会把信息转发到port1上给PC1.同理。当PC2也要登陆魔兽游戏server时。过程也一样。

可是。路由器的mac-table是动态的，当pc1请求连接并且被路由器记录这个mac地址相应的端口为port1时，pc2突然发起连接魔兽server的请求，那么路由器的mac-table就会更改次MAC地址的相应端口，把这个mac地址的端口改为port2。那么pc1的响应消息就会直接发给PC2,造成pc1不能上网。

当然。发生这样的情况比較少。由于，请求响应都是在几秒甚至几十毫秒内完毕的。

所以这也解释了，为什么当中一台PC接收大量数据时，还有一台会断网的原因啦。
