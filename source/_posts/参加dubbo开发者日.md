---
title: 参加dubbo开发者日
date: 2019-05-26 17:35:13
tags: Dubbo
categories: Dubbo
---

今天参加了一个dubbo开发者日，听说有小马哥就去了。上午讲了三个分享。
第一个介绍了springcloud alibaba生态；第二个介绍了seata分布式事务，为什么引入seata呢？因为xa协议本来设计就不是适用于分布式的，xa加了很多很多的锁，但是又因为大部分情况是不需要加那么多锁的 ，所以seate的处理方式就是尝试把锁打碎，这样性能就能上去了。另外，seata也可以兼容at模式，tcc模式，saga模式甚至xa模式。其中有人提了个问题就是，如何保障tc（seata包含tm事务管理者、tc事务协调者和rm资源管理者）的高可用，老师提示可以利用分布式kv存储，这样即使某个存储挂了，也能工作。还有一个问题是，undo日志执行失败了怎么办，老师提示可以利用全局锁，会申请一个锁来保证全局事务，如果全局锁不释放，别的是操控不了的。seata的设计理念好像是基于xid全局事务id，tm确定事务边界，也就是说事务什么时候开始，什么时候结束。通过与tc的协调合作，保障每个分支事务正确执行，如果哪个分支事务执行失败可以进行回滚。

#### 自己对分布式事务的理解
* 我们可以通过消息实现分布式事务方案，比如这样的一个场景，某笔订单成功后，为用户加一定的积分。但是需要注意的是如何保证消息的可靠性。
* 还可以通过补偿来实现，比基于消息的要难。阿里GTS/fescar本质上也是这补偿的编程模型，只不过补偿代码自动生成，无需业务干预，同时接管应用数据源，禁止业务修改处于全局事务状态中的记录。因此，其关于读场景的适用性，可参考补偿。但其在写的适用场景由于引入了全局事务时的写锁，其写适用性介于 TCC以及补偿之间 。
* 还可以基于TCC。TCC实际上是最为复杂的一种情况，其能处理所有的业务场景，但无论出于性能上的考虑，还是开发复杂度上的考虑，都应该尽量避免该类事务。
* 还可以基于SAGA，SAGA可以看做一个异步的、利用队列实现的补偿事务。
* 不同业务场景应按需引入不同的事务形态。

下午有4个分享，其中有一个是sentinel 1.6网关限流新特性介绍。

还有小马哥的分享是Apache Dubbo服务自省设计与实现。小马哥的博客在这里
<https://mercyblitz.github.io/about/>。其中主要提到了dubbo2.7 把配置分成了三个部分，服务信息，元数据信息等，这样分散开始为了减轻zk的压力，dubbo3.0还会结合k8s。还提到了一个优化点是而且也简化了url的参数，在2.7.3版本会提供一种轻量的服务注册机制，减小粒度。提到了zk的缺点，为什么性能上不去呢，因为zk一下子起那么多路径，肯定上不去。而且zk是个CP系统，本身就不是很适合做注册中心。最后一个老师的分享是为什么把注册中心从zk迁移到nacos。

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/1.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/2.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/3.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/4.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/5.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/6.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/7.jpg)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Dubbo/8.jpg)
