---
title: Docker之为何不能停止容器
date: 2019-03-22 22:50:20
tags: Docker
categories: Docker
---


### 背景
我做了个实验，在ubuntu服务器上加载了os_sec模块后，用docker run hello-world后一直卡住，关闭xshell后，docker ps后发现，此容器还在。
那么当我想用docker stop命令停止此容器时，会给我正常显示，但是就是发现stop 不掉。

最后用docker rm -f 容器 可以强制删掉。但是通过查看进程它还在，无语了。是不是因为hello-world容器没有做退出容器的信号处理，所以你用stop也关不掉。

我又试图起redis容器，发现不行，抱以下错误，原因是因为我用了os_sec模块。或者一直处于created状态。
```
/usr/local/bin/docker-entrypoint.sh: 12: /usr/local/bin/docker-entrypoint.sh: find: Operation not permitted
```
### Docker主要执行流程
#### Docker Stop主要流程
1. Docker 通过containerd向容器主进程发送SIGTERM信号后等待一段时间后，如果从containerd收到了容器退出消息那么容器退出成功。
2. 在上一步中，如果等待超时，那么Docker将使用Docker kill 方式试图终止容器
#### Docker Kill主要流程
1. Docker引擎通过containerd使用SIGKILL发向容器主进程，等待一段时间后，如果从containerd收到容器退出消息，那么容器Kill成功
2. 在上一步中如果等待超时，Docker引擎将跳过Containerd自己亲自动手通过kill系统调用向容器主进程发送SIGKILL信号。如果此时kill系统调用返回主进程不存在，那么Docker kill成功。否则引擎将一直死等到containerd通过引擎，容器退出。
#### Docker stop中存在的问题
在上文中我们看到Docker stop首先间接向容器主进程发送sigterm信号试图通知容器主进程优雅退出。但是容器主进程如果没有显示处理sigterm信号的话，那么容器主进程对此过程会不会有任何反应，此信号被忽略了。这里和常规认识不同，在常规想法中任何进程的默认sigterm处理应该是退出。但是namespace中pid==1的进程，sigterm默认动作是忽略。也即是容器首进程如果不处理sigterm，那么此信号默认会被忽略，这就是很多时候Docker Stop不能立即优雅关闭容器的原因——因为容器主进程根本没有处理SIGTERM。
特别指出linux上全局范围内pid=1的进程，不能被sigterm、sigkill、sigint终止
进程组首进程退出后，子进程收到sighub

### Docker kill为何会阻塞
#### 容器主/子进程处于D状态
进程D状态表示进程处于不可中断睡眠状态，一般都是在等待IO资源。当然有些时候如果系统IO出现问题，那么将有大量的进程处于D状态。在这种状态，信号是无法将进程唤醒；只有等待进程自己从D状态中返回。而且在常规内核中，如果某个进程一直处于D状态，那么理论上除了重启系统那么没有什么方法或手段将它从D中接回。
从上面解释Docker kill第二步中可以看到一旦容器中主进程或者子进程处于D状态，那么Docker将等待，一直等到所有容器主进程和其子进程都退出后才返回，那么此时Docker kill就卡住了。
#### 问题解释
当出现问题时刻，宿主机上发现大量的stress进程（实际是容器的进程）处于D状态，而系统响应变慢。问题可以这样解释：
1. Docker kill通过containerd间接向容器主进程发送SIGKill信号以后，由于系统响应慢，容器内部子进程（stress）处于D状态，那么在超时时间内containerd没有上报容器退出。Docker kill走到了直接发送Sigkill阶段
2. 在此阶段前，容器内部主进程退出了，所以系统调用kill 发送SIGKILL很快就返回进程不存在了。引擎认为自己把容器杀死了，Docker kill成功返回了。
3. 在一定时间后容器子进程从D状态中恢复，它们退出了，containerd上报容器退出，引擎清理资源，此时Docker ps看到容器才是退出状态

### 怎么在docker容器中捕获信号
比如我们可以向容器的应用发送一个重新加载信号，容器中的应用程序在收到信号后执行相应的处理程序完成重新加载配置文件的任务。
**注意，只有容器中的1号进程能够收到信号**

比如在dockerfile中写了ENTRYPOINT ["java","-jar",]这种写法会让应用程序在容器中以1号进程的身份运行，这样执行的应用程序在容器中是1号进程，给容器发送SIGTERM信号可以关闭容器。比如通过docker container kill --signal="SIGTERM" XXXX 这个命令。

如果应用程序不是容器中的1号进程，比如ENTRYPOINT["./XXX.sh"],这样通过给容器发送SIGTERM信号，已经无法退出程序了。因为此场景，应用程序是由bash脚本启动，bash作为容器中的1号进程收到了SIGTERM信号，但是它没有做出任何的相应动作。可以通过docker stop来停止（发送的是SIGKILL信号）。

#### 在脚本中捕获信号
创建另外一个启动应用程序的脚本文件 XXXX.sh，内容如下：
```shell
#!/usr/bin/env bash
set -x
pid=0
 
# SIGUSR1-handler
my_handler() {
 echo "my_handler"
}
 
# SIGTERM-handler
term_handler() {
 if [ $pid -ne 0 ]; then
 kill -SIGTERM "$pid"
 wait "$pid"
 fi
 exit 143; # 128 + 15 -- SIGTERM
}
# setup handlers
# on callback, kill the last background process, which is `tail -f /dev/null` and execute the specified handler
trap 'kill ${!}; my_handler' SIGUSR1
trap 'kill ${!}; term_handler' SIGTERM
 
# run application
node app &
pid="$!"
 
# wait forever
while true
do
 tail -f /dev/null & wait ${!}
done
```
这个脚本文件在启动应用程序的同时可以捕获发送给它的 SIGTERM 和 SIGUSR1 信号，并为它们添加了处理程序。其中 SIGTERM 信号的处理程序就是向我们的 node 应用程序发送 SIGTERM 信号。

容器中的 1 号进程是非常重要的，如果它不能正确的处理相关的信号，那么应用程序退出的方式几乎总是被强制杀死而不是优雅的退出。究竟谁是 1 号进程则主要由 EntryPoint, CMD, RUN 等指令的写法决定，所以这些指令的使用是很有讲究的。

### 总结

容器主进程最好需要自己处理SIGTERM信号，因为这是你优雅退出的机会。如果你不处理，那么在Docker stop里你会收到Kill，你未保存的数据就会直接丢失掉。
Docker stop和Docker kill返回并不意味着容器真正退出成功了，必须通过docker ps查看。
对于通过restful与docker 引擎链接的客户端，需要在docker stop和kill restful请求链接上加上超时。对于docker cli用户，需要有另外的机制监控Docker stop或Docker kill命令超时卡死
处于D状态一致卡死的进程，内核无法杀死，docker系统也救不了它。只有重启系统才能清除。

### 参考
<https://www.jianshu.com/p/813d8362d497>
