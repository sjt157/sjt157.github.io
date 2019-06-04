---
title: 一个诡异的shell脚本问题
date: 2019-03-29 20:52:04
tags: Shell
categories: Shell
---



### 起因
在论坛上看见有人问了一个问题，<https://javaweb.io/post/394/reply/1246>,其实这个shell脚本很简单，就是根据进程名字得到进程号，但是很诡异的出来了异常结果。

我也自己尝试了一下，直接执行命令确实是返回一个pid，但是在shell脚本里，却是3个pid号。这是为什么呢？

### 峰回路转
看见了这个<https://akaedu.github.io/book/ch31s02.html>
大概知道了什么原因：

用户在命令行输入命令后，一般情况下Shell会fork并exec该命令，但是Shell的内建命令例外，执行内建命令相当于调用Shell进程中的一个函数，并不创建新的进程。以前学过的cd、alias、umask、exit等命令即是内建命令，凡是用which命令查不到程序文件所在位置的命令都是内建命令，内建命令没有单独的man手册，要在man手册中查看内建命令，应该用man bash-builtins

一句话总结就是：**多出来的那两个是shell子进程的pid号**

### 过程

#### 原来的错误脚本是这样的
```shell
#!/bin/bash
set -x
process=$1
pid=$(ps -ef | grep $process  | grep -v grep | awk '{print $2}')

echo $pid
sleep 5
while true
do
   echo aaaaaaaa
done
```
所以说 在终端输入sh a.sh rcu_sched（我拿这个进程举例）的时候，其实起了三个进程，其中有两个是sh a.sh rcu_sched,这两个是Shell的临时进程。解决办法就是加上grep -v sh（grep的-v参数是取反，也就是说grep -v grep是过滤掉那些带grep关键字进程，）


### 收获
虽然浪费了好长时间才弄懂，但是收获了别的东西

#### 调试shell脚本

开启调试功能
* 通过sh -x 脚本名  #显示脚本执行过程
* 脚本里set -x选项,轻松跟踪调试shell脚本


首先使用“-n”选项检查语法错误，然后使用“-x”选项跟踪脚本的执行，使用“-x”选项之前，别忘了先定制PS4变量的值来增强“-x”选项的输出信息，至少应该令其输出行号信息(先执行export PS4='+[$LINENO]'，更一劳永逸的办法是将这条语句加到您用户主目录的.bash_profile文件中去)，这将使你的调试之旅更轻松。也可以利用trap,调试钩子等手段输出关键调试信息，快速缩小排查错误的范围，并在脚本中使用“set -x”及“set +x”对某些代码块进行重点跟踪。这样多种手段齐下，相信您已经可以比较轻松地抓出您的shell脚本中的臭虫了。如果您的脚本足够复杂，还需要更强的调试能力，可以使用shell调试器bashdb，这是一个类似于GDB的调试工具，可以完成对shell脚本的断点设置，单步执行，变量观察等许多功能，使用bashdb对阅读和理解复杂的shell脚本也会大有裨益。关于bashdb的安装和使用，不属于本文范围，您可参阅http://bashdb.sourceforge.net/上的文档并下载试用。

#### 一个脚本检测网站

该网站能帮你检测你的脚本有啥问题
<https://www.shellcheck.net/>

#### Linux Shell脚本实现根据进程名杀死进程
Shell脚本源码如下：
```shell
#!/bin/sh
#根据进程名杀死进程
if [ $# -lt 1 ]
then
  echo "缺少参数：procedure_name"
  exit 1
fi
 
PROCESS=`ps -ef|grep $1|grep -v grep|grep -v PPID|awk '{ print $2}'`
for i in $PROCESS
do
  echo "Kill the $1 process [ $i ]"
  kill -9 $i
done
```

#### 交互式 Bash Shell 获取进程 pid
在已知进程名(name)的前提下，交互式 Shell 获取进程 pid 有很多种方法，典型的通过 grep 获取 pid 的方法为（这里添加 -v grep是为了避免匹配到 grep 进程）：

ps -ef | grep "name" | grep -v grep | awk '{print $2}'

或者不使用 grep（这里名称首字母加[]的目的是为了避免匹配到 awk 自身的进程）：

ps -ef | awk '/[n]ame/{print $2}'

如果只使用 x 参数的话则 pid 应该位于第一位：

ps x | awk '/[n]ame/{print $1}'

最简单的方法是使用 pgrep：

pgrep -f name

如果需要查找到 pid 之后 kill 掉该进程，还可以使用 pkill：

pkill -f name

如果是可执行程序的话，可以直接使用 pidof

pidof name


#### 获取 Shell 脚本自身进程 pid
```shell
这里涉及两个指令： 
1. $$ ：当前 Shell 进程的 pid 
2. $! ：上一个后台进程的 pid 可以使用这两个指令来获取相应的进程 pid。例如，如果需要获取某个正在执行的进程的 pid（并写入指定的文件）：
myCommand && pid=$!
myCommand & echo $! >/path/to/pid.file
注意，在脚本中执行 $! 只会显示子 Shell 的后台进程 pid，如果子 Shell 先前没有启动后台进程，则没有输出。
```
#### 查看指定进程是否存在
在获取到 pid 之后，还可以根据 pid 查看对应的进程是否存在（运行），这个方法也可以用于 kill 指定的进程。
```shell
if ps -p $PID > /dev/null
then
   echo "$PID is running"
   # Do something knowing the pid exists, i.e. the process with $PID is running
fi
```
#### 使用 Shell 对进程资源进行监控

<https://www.ibm.com/developerworks/cn/linux/l-cn-shell-monitoring/index.html>

#### 父子进程(pid相差1)
<https://www.cnblogs.com/jian-99/p/7719085.html>

### 参考
<http://weyo.me/pages/techs/linux-get-pid/>



