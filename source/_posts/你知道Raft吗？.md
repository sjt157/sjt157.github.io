---
title: 你知道Raft吗？
date: 2019-05-25 18:09:38
tags: 分布式
categories: 分布式
---

### 什么是 paxos 算法以及raft算法
paxos：多个proposer发请提议（每个提议有id+value），acceptor接受最新id的提议并把之前保留的提议返回。当超过半数的accetor返回某个提议时，此时要求value修改为propeser历史上最大值，propeser认为可以接受该提议，于是广播给每个acceptor，acceptor发现该提议和自己保存的一致，于是接受该提议并且learner同步该提议。

raft：raft要求每个节点有一个选主的时间间隔，每过一个时间间隔向master发送心跳包，当心跳失败，该节点重新发起选主，当过半节点响应时则该节点当选主机，广播状态，然后以后继续下一轮选主

### raft原理的动画演示：
<http://thesecretlivesofdata.com/raft/>

### 寻找一种易于理解的一致性算法（扩展版）

<https://learnblockchain.cn/2019/03/22/easy_raft/#more>

### Java 版本的 Raft(CP) KV 分布式存储
<https://github.com/stateIs0/lu-raft-kv>
