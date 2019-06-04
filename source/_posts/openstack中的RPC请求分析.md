---
title: openstack中的RPC请求分析
date: 2018-04-08 18:23:00
tags: openstack
categories: openstack
---

## openstack中的RPC请求分析

1. **首先，什么是RPC请求？**
  我们都知道，rpc就是远程过程调用，是Openstack中一种用来实现跨进程(或者跨机器)的通信机制。Openstack中同项目内(如nova, neutron, cinder...)各服务(service)及通过RPC实现彼此间通信。Openstack中还有另外两种跨进程的通信方式：数据库和Rest API。*一般情况下，openstack各个项目之间通过RestAPI接口进行相互访问，而项目内部服务之间则通过RPC请求的方式进行通信。*
2. **其他参考**
  官网详细介绍了Openstack RPC中的基本概念及API设计<http://https://wiki.openstack.org/wiki/Oslo/Messaging>，其实它的rpc的设计参考了Sun RPC的设计，Sun RPC的介绍可以参看<http://http://en.wikipedia.org/wiki/Open_Network_Computing_Remote_Procedure_Call>这篇文章。
3. **RPC的使用场景**
* 随机调用某server上的一个方法：
Invoke Method on One of Multiple Servers
这个应该是Openstack中最常用的一种RPC调用，每个方法都会有多个server来提供，client调用时由底层机制选择一个server来处理这个调用请求。像nova-scheduler, nova-conductor都可以以这种多部署方式提供服务。这种场景通过AMQP的topic exchange实现。
所有server在binding中为binding key指定一个相同的topic， client在调用时使用这个topic既可实现。
* 调用某特定server上的一个方法：
Invoke Method on a Specific Server 一般Openstack中的各种scheduler会以这种方式调用。通常scheduler都会先选定一个节点，然后调用该节点上的服务。这种场景通过AMQP的topic exchange实现。每个server在binding中为其binding key指定一个自己都有的topic， client在调用时使用这个topic既可实现。
* 调用所有server上的一个方法：
Invoke Method on all of Multiple Servers 这种其实就是一个广播系统。就像开会议，台上的人讲话，台下的人都能听到。Openstack中有些rpcapi.py的某些方法带有fanout=True参数，这些都是让所有server处理某个请求的情况。例子： neutron中所有plugin都会有一个AgentNotifierApi，这个rpc是用来调用安装在compute上的L2 agent。因为存在多个L2 agent(每个compute上都会有)，所以要用广播模式。这种场景通过AMQP的fanout exchange实现。每个server在binding中将其队列绑定到一个fanout exchange， client在调用时指定exchange类型为fanout即可。server和client使用同一个exchange。

4. **rpc.call和rpc.cast的区别：**
* RPC.call：发送请求到消息队列，等待返回最终结果。
* RPC.cast：发送请求到消息队列，不需要等待最终返回的结果。
其实还有一种rpc调用，也就是RPC.Notifier:发送各类操作消息到队列，不需要等待最终的返回结果。RPC.call、RPC.cast一般用于同一个项目下的服务之间进行的“内部“请求；RPC.Notifier发送的操作消息，目前被ceilometer notification服务所接收。
* **举个栗子**：比如虚拟机创建过程，创建虚拟机等TaskAPI任务，已经由nova-conductor承担，因此nova-api监听到创建虚拟机的HTTP请求后，会通过RPC调用（是cast）nova.conductor.manager.ComputeTaskManager中的build_instance()方法。nova-conductor会在build_instances()中生成request_spec字典，其中包括了详细的虚拟机信息，nova-conductor通过rpc.call方法向nova-scheduler发出请求，nova-scheduler依据这些信息为虚拟机选择一个最佳的主机，然后返回给nova-conductor。（为什么有返回，因为是call）然后nova-conductor再通过nova-compute创建虚拟机。nova-compute首先会使用Resource Tracker的Claim机制检测一下主机的可用资源是否能够满足新建虚拟机的需要，然后通过具体的virt Driver创建虚拟机。
