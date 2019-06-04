---
title: Epoll惊群问题
date: 2019-05-25 18:19:48
tags: Linux
categories: Linux
---
### strace命令
strace是个功能强大的Linux调试分析诊断工具，可用于跟踪程序执行时进程系统调用(system call)和所接收的信号，尤其是针对源码不可读或源码无法再编译的程序。

在Linux系统中，用户程序运行在一个沙箱(sandbox)里，用户进程不能直接访问计算机硬件设备。当进程需要访问硬件设备(如读取磁盘文件或接收网络数据等)时，必须由用户态模式切换至内核态模式，通过系统调用访问硬件设备。strace可跟踪进程产生的系统调用，包括参数、返回值和执行所消耗的时间。若strace没有任何输出，并不代表此时进程发生阻塞；也可能程序进程正在自己的沙箱里执行某些不需要与系统其它部分发生通信的事情。strace从内核接收信息，且无需以任何特殊方式来构建内核。


* strace命令格式如下：

`strace [-dffhiqrtttTvVxx] [-a column] [-e expr] [-o file] [-p pid] [-s strsize] [-u username] [-E var=val] [command [arg ...]]` 或

`strace -c [-e expr] [-O overhead] [-S sortby] [-E var=val] [command [arg ...]]`

通过不同的选项开关，strace提供非常丰富的跟踪功能。最简单的应用是，跟踪可执行程序运行时的整个生命周期，将所调用的系统调用的名称、参数和返回值输出到标准错误输出stderr(即屏幕)或-o选项所指定的文件。注意，命令(command)必须位于选项列表之后。

#### 啥也没有
如果我们用 strace 跟踪一个进程，输出结果很少，是不是说明进程很空闲？其实试试 ltrace，可能会发现别有洞天。记住有内核态和用户态之分。

我第一开始执行`strace -fF  -o b.txt -e trace=accpet  ./spider`啥也没有，以为是上边的原因，结果其实是因为你的代码里没有accept相关的调用，所以肯定没有了啊，那我不写的话就相当于记录所有的东西。

#### 命令说明
<https://www.cnblogs.com/clover-toeic/p/3738156.html>

### 惊群问题
<https://blog.csdn.net/dog250/article/details/80837278>

```
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netdb.h>
#include <fcntl.h>
#include <sys/wait.h>
#include <time.h>
#include <signal.h>

#define COUNT 1

int mode = 0;
int slp = 0;

int pid[COUNT] = {0};
int count = 0;

void server(int epfd) 
{
    struct epoll_event *events;
    int num, i;
        struct timespec ts;

    events = calloc(64, sizeof(struct epoll_event));

    while (1) {
        int sd, csd;
        struct sockaddr in_addr;

        num = epoll_wait(epfd, events, 64, -1);
        if (num <= 0) {
            continue;
        }
        /*
        ts.tv_sec = 0;
        ts.tv_nsec = 1;
        if(nanosleep(&ts, NULL) != 0) {
            perror("nanosleep");
            exit(1);
        }
        */
        // 用于测试ET模式下丢事件的情况
        if (slp) {
            sleep(slp);
        }

        sd = events[0].data.fd;
        socklen_t in_len = sizeof(in_addr);

        csd = accept(sd, &in_addr, &in_len);
        if (csd == -1) {
            // 打印这个说明中了epoll LT惊群的招了。
            printf("shit xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:%d\n", getpid()); 
            continue;
        }
        // 本进程一共成功处理了多少个请求。
        count ++;
        printf("get client:%d\n", getpid()); 
        close(csd);
    }
}

static void siguser_handler(int sig)
{
    // 在主进程被Ctrl-C退出的时候，每一个子进程均要打印自己处理了多少个请求。
    printf("pid:%d  count:%d\n", getpid(), count);
    exit(0);
}

static void sigint_handler(int sig)
{
    int i = 0;
    // 给每一个子进程发信号，要求其打印自己处理了多少个请求。
    for (i = 0; i < COUNT; i++) {
        kill(pid[i], SIGUSR1);
    }
}

int main (int argc, char *argv[])
{
    int ret = 0;
        int listener;
    int c = 0;
    struct sockaddr_in saddr;
    int port;
    int status;
        int flags;
    int epfd;
    struct epoll_event event;


    if (argc < 4) {
        exit(1);
    }

    // 0为LT模式，1为ET模式
    mode = atoi(argv[1]);
    port = atoi(argv[2]);
    // 是否在处理accept之前耽搁一会儿，这个参数更容易重现问题
    slp = atoi(argv[3]);

    signal(SIGINT, sigint_handler);

    listener = socket(PF_INET, SOCK_STREAM, 0);

    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(port);
    saddr.sin_addr.s_addr = INADDR_ANY;

    bind(listener, (struct sockaddr*)&saddr, sizeof(saddr));
    listen(listener, SOMAXCONN);

    flags = fcntl (listener, F_GETFL, 0);
    flags |= O_NONBLOCK;
    fcntl (listener, F_SETFL, flags);


    epfd = epoll_create(64);
    if (epfd == -1) {
        perror("epoll_create");
        abort();
    }

    event.data.fd = listener;
    event.events = EPOLLIN;
    if (mode == 1) {
        event.events |= EPOLLET;
    } else if (mode == 2) {
        event.events |= EPOLLONESHOT;
    } 

    ret = epoll_ctl(epfd, EPOLL_CTL_ADD, listener, &event);
    if (ret == -1) {
        perror("epoll_ctl");
        abort();
    }


    for(c = 0; c < COUNT; c++) {
        int child;
            child = fork();
            if(child == 0) {
                    // 安装打印count值的信号处理函数
                    signal(SIGUSR1, siguser_handler);
                    server(epfd);
            }
        pid[c] = child;
        printf("server:%d  pid:%d\n", c+1, child);
        }
    wait(&status);
    sleep(1000000);
    close (listener);
}


```

### 参考
<https://www.cnblogs.com/clover-toeic/p/3738156.html>
<https://huoding.com/2015/10/16/474>
