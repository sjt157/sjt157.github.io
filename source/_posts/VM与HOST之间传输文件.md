---
title: VM与HOST之间传输文件
date: 2018-12-12 19:41:33
tags: KVM
categories: KVM
---


今天老师有一个需求，是从host向qemu创建的虚拟机中传输文件，调研了几个方法，起初是通过网络传输，但是在新的这个平台上没有搭建网桥等设备，而且想还没有别的其他的方式。经过查询，主要有以下方式：
1. 共享文件夹的方式
   Docker中也有此方法
  <https://blog.csdn.net/zhongbeida_xue/article/details/80747212?utm_source=blogxgwz9>

2. 把数据存到usb设备（可真可假）中
   <https://blog.csdn.net/kingtj/article/details/82952783>

## 总结：
qemu-kvm虚拟机与宿主机之间实现文件传输，大概两类方法：
* 虚拟机与宿主机之间，使用网络来进行文件传输。这个需要先在宿主机上配置网络桥架，在qemu-kvm启动配置网卡就可以实现文件传输。
* 使用9psetup协议实现虚拟机与宿主机之间文件传输。该方法先要宿主机需要在内核中配置了9p选项，即：
```
     CONFIG_NET_9P=y
     CONFIG_net_9P_VIRTIO=y
     CONFIG_NET_9P_DEBUG=y (可选项)
     CONFIG_9P_FS=y
     CONFIG_9P_FS_POSIX_ACL=y
```
另外，qemu在编译时需要支持ATTR/XATTR。

综上，两类方法配置起来都比较麻烦。其实有一个比较简单的方法
```
在虚拟机环境下，我们可能会遇到在宿主机和客户机之间传输文件的需求，目前有几种方法可以实现这个例如通过9p协议，或者为客户机和宿主机之间搭建一个网络等。这些都太不容易实现，下面我介绍一种简单的方法。

1. 使用dd创建一个文件，作为虚拟机和宿主机之间传输桥梁

dd if=/dev/zero of=/var/lib/libvirt/images/share.img bs=1M count=350
2. 格式化share.img文件
mkfs.ext4/var/lib/libvirt/images/share.img
3. 在宿主机上创建一个文件夹，
mkdir /tmp/share
mount -o loop/var/lib/libvirt/images/share.img /tmp/share
这样，在宿主机上把需要传输给虚拟机的文件放到/tmp/share 下即可。
4. 启动qemu-kvm虚拟机，可以额外为客户机添加上一块硬盘。

-drive file=/var/lib/libvirt/images/share.img,if=virtio

5. 在虚拟机中 mount上添加的一块硬盘。即可以获得宿主机上放在/tmp/share文件夹下的文件，具体做法是：通过dmesg的输出找到新挂在的硬盘是什么，然后将硬盘直接mount上来。

mount -t ext4 /dev/vdb /mnt/   
```

当然，该方法虽然简单，但它也有缺点, 宿主机和虚拟机文件传输不能实时传输。如果需要传输新文件，需要重启虚拟机。

### 有没有其他的方式呢还？待补充。。。

