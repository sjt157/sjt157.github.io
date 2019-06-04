---
title: 读《JAVA并发编程实践》（译）笔记
date: 2019-06-02 11:11:58
tags: Java
categories: Java
---

* 多数Servlet都可以实现为无状态的，降低了线程安全的负担，只有当servlet要为不同的请求记录一些信息时，才需要线程安全。

* 耗时的计算或操作，执行这些操作期间不要占用锁，比如IO

* 不要让this引用在构造期间逸出

* SynchronousQueue这类队列只有在消费者充足的时候比较合适，他们总能为下一个任务做好准备。

* 可以使用Semaphore把任何容器转化为有界的阻塞容器

* Exchanger是非常有用的，比如一个线程向缓冲写入一个数据，另一个线程消费这个数据，就可以使用Exchanger进行会面。

* 中段通常是实现取消最明智的选择

* Java中的中断

* 使用finally块和显式close方法的结合来管理资源，会比使用finalizer起到更好地作用，尽量不要使用finalizer。

* 扩展ThreadPoolExecutor
这个的设计是可扩展的，有函数钩子，beforeExecute、afterExecute和terminate。

* 如何解决活锁
加入随机性。以太协议在重复发生冲突时，同样包含一个指数的撤退协议，减少了拥塞和多个冲突的机站重复失败的风险。

* 自旋等待适合短期的等待，而挂起适合长时间等待。

* 减少锁的竞争（三种方式）
减少锁持有的时间、减少频率、用协调机制替代独占锁

* 垃圾回收
在技术上是不可能强制垃圾回收执行的，System.gc仅仅是建议JVM在一个合适的时候进行垃圾回收。

* 静态分析工具
FindBugs <http://findbugs.sourceforege.net>




