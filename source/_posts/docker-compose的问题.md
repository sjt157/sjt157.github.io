---
title: docker-compose的问题
date: 2019-02-22 17:44:45
tags: Docker
categories: Docker
---
今天我启动docker-compose的时候，出现了如下问题
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/WhiteList/1.png)

经过查询得知<https://blog.csdn.net/tianshuhao521/article/details/84782309>，
原因是关闭防火墙之后docker需要重启，执行以下命令重启docker即可：
service docker restart

但是我重启不行，journalctl -xe查看日志，发现
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/WhiteList/2.png)

原来有访问控制了，可能是因为我们实验室对该服务器的内核进行了白名单。lsmod命令一输，结果如下，发现确实有一个操作系统安全module。
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/WhiteList/3.png)

通过rmmod卸载掉之后就可以 了。

通过dmesg查看信息发现，
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/WhiteList/4.png)

### 进一步思考：
他的这个原理是什么？有待补充
