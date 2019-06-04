---
title: qemu-kvm-io线程
date: 2018-03-11 14:06:36
tags: KVM
categories: KVM 
---
qemu-kvm一般会有4个线程，主线程是io thread，还有一个vcpu thread，一个signal thread。
对于io thread，可以从kvm_main_loop(qemu-kvm.c)开始看：
两个pipe是比较重要的一点：
1. 它调用qemu_eventfd创建一个event pipe，pipe read fd用qemu_set_fd _hadnler2注册一个read fd handler，也就是说如果fd read ready，io thread将通过某种机制调用这个handler，即io_thread_wakeup。换句话说，这个event pipe读的操作应该是在io_thread_wakeup()函数中，那么是由谁执行写操作的呢？这个还不是很清楚，只知道write fd记录到了全局变量io_thread_fd中，也就是说在qemu-kvm任何地方都可以引用io_thread_fd向pipe写东西。
2. 它还调用qemu_signalfd创建一个signal thread，不过现在还不清楚为什么要单独创建一个这样的线程。qemu_signalfd中还会创建一个signal pipe，read fd同样用qemu_set_fd _hadnler2注册一个read fd handler——sigfd_handler，而write fd作为参数传递给signal thread执行函数，也就是说写操作应该是由signal thread负责的。
 为什么要创建一个signal thread和signal pipe呢？
       上面提到用qemu_set_fd_handler2注册的handler在fd ready时，io thread会通过某种机制调用相应地handler。这个机制是main_loop_wait函数实现的。main_loop_wait函数中将所有注册的r/w fd都加入到对应地r/w fd_set中，然后调用select系统调用，这个系统调用将检查相应fd_set中的fd是否有已ready的，如果没有一个ready，将使得调用线程阻塞直到超时，否则将返回，并将原来fd_set修改，将没有ready的fd从原fd_set中删除。返回后，通过遍历handler队列，检测其fd是否在fd_set中，并调用相应的handler。

        我记得中说过，发送给这个qemu process的信号可以由process内任意的thread来处理， 但是每个线程都可以设置mask来选择是否处理某信号。那么为什么要单独设置一个signal thread来接受信号呢？实际上，用signal thread这种机制是将原本对指定信号的处理由异步方式改成同步方式。
         默认异步方式时，信号发送给本qemu process后，内核将检测是否有pending signal，如果有内核将调用相应的handler，而这个执行实体可以是process中任意的thread。这样，thread有可能因为信号的到来，而被临时打断，去执行handler。
         同步方式是这样的，首先sigprocmask系统调用屏蔽掉指定的signal，这样process将不再接受这个signal(当然，每个thread都有屏蔽码的，但是应该是调用pthread_sigmask，调用sigprocmask将使整个process不接受)。然后，创建一个signal thread，其执行代码调用sigwaitinfo函数，等待指定的信号，如果无信号则signal thread阻塞在这个系统调用中，否则返回，将siginfo_t写入signal pipe中。最后，io thread通过select系统调用检测到signal pipe read_fd  ready，然后调用相应的read_fd handler，fd handler再调用相应的signal handler。
        可以看到，同步方式虽然代价比异步方式大得多，需要多创建一个signal pipe和signal thread，但是它不会突然打断thread执行，对于程序响应性有利。这个跟interrupt和polling之间的比较类似，当外部事情来时，执行内核的中断服务例程，interrupt handler有可能发送一个信号给要处理事件的process。

之前发现一个很奇怪的现象：在我的电脑上跑的qemu-kvm有4个线程，而其他同学电脑上跑只有2个线程。我原本觉得奇怪，按照前面的论述，qemu-kvm至少得有3个线程才是，只有2个线程的话，那么signal thread去哪里了呢？带着这个问题，对两种情况的代码做了一下trace，结果发现是在qemu_signalfd (compatfd.c)函数中出现了差异—如果系统有定义CONFIG_SIGNALFD宏，那么就不会创建signal thread，否则就创建。不太理解的是，我的装的是32位CentOS，结果没有SIGNALFD，而其他装的是64位CentOS，是有SIGNALFD的。
那么signalfd是什么呢？简单的说，就是一个线程可以创建一个signalfd，指定哪些signal可以由它来接受，当信号来时，就不是像原来那样去调用signal handler，而是将siginfo_t写入signalfd，线程通过read或select或poll就可以知道signalfd是否接受到了新的signal。
这样，在由signalfd情况下，就没有必要创建signal thread了。这样，就减少了一点开销。原来是signal thread通过sigwaitinfo检测到signal，然后将siginfo_t写入signal pipe，io thread通过select检测到signal pipe read fd ready，然后读出siginfo_t，再调用handler；现在是，内核直接将siginfo_t写入signalfd，io thread通过select检测到signalfd ready，然后读出读出siginfo_t，再调用handler。很显然，省去了额外signal thread的开销。
总之，只要先用sigprocmask将可能接受的signal block掉，再用select+signalfd或者是sigwaitinfo都可以讲异步signal转变为同步signal。
