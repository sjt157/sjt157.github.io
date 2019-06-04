---
title: Docker之seccomp
date: 2019-03-11 12:56:57
tags: Docker
categories: Docker
---
### 什么是seccomp
seccomp（全称securecomputing mode）是linux kernel从2.6.23版本开始所支持的一种安全机制。

在Linux系统里，大量的系统调用（systemcall）直接暴露给用户态程序。但是，并不是所有的系统调用都被需要，而且不安全的代码滥用系统调用会对系统造成安全威胁。通过seccomp，我们限制程序使用某些系统调用，这样可以减少系统的暴露面，同时是程序进入一种“安全”的状态。

### 如何使用seccomp
seccomp可以通过系统调用ptrctl(2)或者通过系统调用seccomp(2)开启，前提是内核配置中开启了CONFIG_SECCOMP和CONFIG_SECCOMP_FILTER。

seccomp支持两种模式：SECCOMP_MODE_STRICT和SECCOMP_MODE_FILTER。
* 在SECCOMP_MODE_STRICT模式下，进程不能使用read(2)，write(2)，_exit(2)和sigreturn(2)以外的其他系统调用。
* 在SECCOMP_MODE_FILTER模式下，可以利用BerkeleyPacket Filter配置哪些系统调用及它们的参数可以被进程使用。

### 如何查看是否使用了seccomp
通常有两种方法：

* 利用prctl(2)的PR_GET_SECCOMP的参数获取当前进程的seccomp状态。返回值0表示没有使用seccomp;返回值2表示使用了seccomp并处于SECCOMP_MODE_FILTER模式；其他情况进程会被SIGKILL信号杀死。

* 从Linux3.8开始，可以利用/proc/[pid]/status中的Seccomp字段查看。如果没有seccomp字段，说明内核不支持seccomp。
* 
下面这个是docker守护进程的status
```c
Name:	dockerd
State:	S (sleeping)
Tgid:	1345
Ngid:	0
Pid:	1345
PPid:	1
TracerPid:	0
Uid:	0	0	0	0
Gid:	0	0	0	0
FDSize:	256
Groups:	
NStgid:	1345
NSpid:	1345
NSpgid:	1345
NSsid:	1345
VmPeak:	 1150756 kB
VmSize:	 1150628 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	  102020 kB
VmRSS:	   87212 kB
VmData:	 1049204 kB
VmStk:	     132 kB
VmExe:	   46768 kB
VmLib:	    4612 kB
VmPTE:	     484 kB
VmPMD:	      24 kB
VmSwap:	       0 kB
HugetlbPages:	       0 kB
Threads:	36
SigQ:	0/62554
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	fffffffe3bfa2800
SigIgn:	0000000000000000
SigCgt:	ffffffffffc1feff
CapInh:	0000000000000000
CapPrm:	0000003fffffffff
CapEff:	0000003fffffffff
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
Seccomp:	0    //这是0，表示没有使用seccomp
Cpus_allowed:	ff
Cpus_allowed_list:	0-7
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	2334
nonvoluntary_ctxt_switches:	4
```

### 在docker中使用

只有在使用 seccomp 构建 Docker 并且内核配置了 CONFIG_SECCOMP 的情况下，此功能才可用。要检查你的内核是否支持 seccomp：
```
$ cat /boot/config-`uname -r` | grep CONFIG_SECCOMP=
CONFIG_SECCOMP=y
```
### 为容器传递配置文件
默认的 seccomp 配置文件为使用 seccomp 运行容器提供了一个合理的设置，并禁用了大约 44 个超过 300+ 的系统调用。它具有适度的保护性，同时提供广泛的应用兼容性。默认的 Docker 配置文件可以在 这里 找到。
<https://github.com/moby/moby/blob/master/profiles/seccomp/default.json>

实际上，该配置文件是白名单，默认情况下阻止访问所有的系统调用，然后将特定的系统调用列入白名单。该配置文件工作时需要定义 SCMP_ACT_ERRNO 的 defaultAction 并仅针对特定的系统调用覆盖该 action。SCMP_ACT_ERRNO 的影响是触发 Permission Denied 错误。接下来，配置文件中通过将 action 被覆盖为 SCMP_ACT_ALLOW，定义一个完全允许的系统调用的特定列表。最后，一些特定规则适用于个别的系统调用，如 personality，socket，socketcall 等，以允许具有特定参数的那些系统调用的变体（to allow variants of those system calls with specific arguments）。

seccomp 有助于以最小权限运行 Docker 容器。不建议更改默认的 seccomp 配置文件。

运行容器时，如果没有通过 --security-opt 选项覆盖容器，则会使用默认配置。例如，以下显式指定了一个策略：
```shell
docker run --rm \
             -it \
             --security-opt seccomp=/path/to/seccomp/profile.json \
             hello-world
```


### 不使用默认的 seccomp 配置文件
可以传递 unconfined 以运行没有默认 seccomp 配置文件的容器。

```shell
docker run --rm -it --security-opt seccomp=unconfined debian:jessie \
    unshare --map-root-user --user sh -c whoami
```
### 参考
<https://blog.csdn.net/mashimiao/article/details/73607485>
<https://blog.csdn.net/kikajack/article/details/79596843>
