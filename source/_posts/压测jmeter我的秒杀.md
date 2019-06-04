---
title: 压测jmeter我的秒杀
date: 2019-05-31 21:07:16
tags: SpringBoot
categories: SpringBoot
---



### jmeter步骤
1. 启动bin目录下的jmeter.bat
2. 添加一个thread group，设置参数比如1000,0,1（循环一次）
3. 添加一个config element里的http default request
4. 添加sample 里的http request
5. 添加一个聚合报告，run

### 压测结果
throughput才38.1/sec,这也太低了吧。为啥。也就是说一秒钟只能接受38个请求。。
改变了下线程池的参数，变为执行10次，即10000次，throughout达到了273.59。也太低。。
每次跑，结果都不一样啊，最高1000。


### 查看dump
看了下进程日志，发现了很多信息。

#### dump文件中描述的线程状态
runnable：运行中状态，在虚拟机内部执行，可能已经获取到了锁，可以观察是否有locked字样。 
blocked：被阻塞并等待锁的释放。 
wating：处于等待状态，等待特定的操作被唤醒，一般停留在park(), wait(), sleep(),join() 等语句里。 
time_wating：有时限的等待另一个线程的特定操作。 
terminated：线程已经退出。

#### 进程区域的划分

进入区====拥有区====等待区

进入区（Entry Set）：等待获取对象锁，一旦对象锁释放，立即参与竞争。 
拥有区（The Owner）：已经获取到锁。 
等待区（Wait Set）：表示线程通过wait方法释放了对象锁，并在等待区等待被唤醒。

#### 方法调用修饰
locked: 成功获取锁 
waiting to lock：还未获取到锁，在进入去等待； 
waiting on：获取到锁之后，又释放锁，在等待区等待； 
parking to wait for：等待许可证； （参考LockSupport.park和unpark操作）

举个栗子：
```java
"Finalizer" #3 daemon prio=8 os_prio=0 tid=0x00007f94c81d2800 nid=0x13 in Object.wait() [0x00007f94b4cd7000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000006c7226010> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x00000006c7226010> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
```

### 意外发现的小问题
* 部署在docker里，返回的数据有乱码。我看了一下编码方式是UTF-8啊，没毛病啊。
解决办法：在dockerfile里，添加`ENV LANG C.UTF-8`.重新制作docker镜像，docker run -ti [镜像] 进入容器后执行locale发现编码格式已经被修改为C.UTF-8，之前出现的中文文件名乱码问题也没有了。

* 在Request Header里的Cookie: Hm_lvt_1fc1803bb58c30947181c3b6b65e7d9d=1552486046; token=22ad4279f56a4fa6a39b6e7ea3b23bdf，其中Hm_lvt是啥，其实这是百度统计的东西，不用管。


### 参考
<https://blog.csdn.net/sheldon178/article/details/79543671>
