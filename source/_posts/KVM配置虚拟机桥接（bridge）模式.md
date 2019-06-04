---
title: KVM配置虚拟机桥接（bridge）模式
date: 2018-03-14 20:18:56
tags: KVM
categories: KVM
---
主机：Ubuntu14.04 64bit 
虚拟机：Ubuntu14.04 64bit 
VMM:KVM

基础知识：默认创建的网桥virbr0用于NAT模式，如果想用brigde模式，需要自己创建一个br0。Bridge方式即虚拟网桥的网络连接方式，是客户机和子网里面的机器能够互相通信。可以使虚拟机成为网络中具有独立IP的主机。 桥接网络（也叫物理设备共享）被用作把一个物理设备复制到一台虚拟机。网桥多用作高级设置，特别是主机多个网络接口的情况。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-bridge/1.png)
  
1. 修改网络配置文件 /etc/network/interfaces ，此处是DHCP方式，也可以static方式。（Ubuntu系统，centos在etc/sysconfig里）输入 brctl addbr br0    建立一个网桥br0，此时在Linux内核里创建虚拟网卡br0，enp11s0 是host主机的网卡。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-bridge/2.png)  

2. 重新启动网络服务便可
重启计算机或者sudo systemctl restart networking / sudo systemctl restart network-manager，此时输入ifconfig，显示如图。（据说br0与enp11s0只能有一个有ip地址，此处不是很明白，但是目前并不影响使用。）
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-bridge/3.png)  
 
3. 输入该命令，可以看到此时已有br0的网桥。
  
 
4. 创建虚拟机时，指定—bridge=br0。（--bridge与—network只能二选一，不能同时指定）
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-bridge/4.png)  
```
	virt-install --virt-type kvm --name test2 --ram 1024 --vcpus 1 --bridge=br0 \ --cdrom=/home/lib206/vmimage/ubuntu-16.04.1-server-amd64.iso \--disk path=/home/lib206/vmimage/Test.img,size=20,format=raw --graphics vnc,port=5901  --os-type=linux  --boot cdrom

```
5. 创建成功后，如果启动虚拟机时出现不能引导Booting from No bootable device，按下图所示设置，可能是引导顺序的原因。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/KVM-bridge/5.png)  
 
6. 此时虚拟机即处于桥接模式，ip由路由器DHCP随即分配。通过以上步骤的设置KVM的桥接问题解决了，此时可以查看一下虚拟机网络IP地址，应该跟主机的IP地址处于同一个网段；但是还是有问题的， 无线网卡桥接是不成功的，默认的是有线网卡！

7. 如果想指定该虚拟机的ip地址，按另一篇文章设置。
