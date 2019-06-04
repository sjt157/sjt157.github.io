---
title: Shell知识点
date: 2018-03-18 20:56:48
tags: Shell
---

### 使用read读取行

* read可以一次读取所有的值到多个变量里，使用$IFS分割，比如应用于/etc/passwd文件。

```
	#!/bin/bash
	while IFS=: read user pass uid gid fullname homedir Shell
	do
  		echo $user
	done < /etc/passwd

```
当到文件末尾时，read会以非0退出，终止while循环。可能会觉得把/etc/passwd的重定向放到末尾有些奇怪，不过这是必须的。
如果这样写，就不会终止！因为每次循环，Shell都会打开/etc/passwd一次，且read只读取文件的第一行。

```
	#!/bin/bash
	while IFS=: read user pass uid gid < /etc/passwd fullname homedir Shell
	do
  		echo $user
	done 

```

* `make 1> results 2> ERRS`  该命令将标准错误输出到ERRS，将标准输出传给results。

* 命令替换：就是指Shell执行命令并将命令替换为执行该命令后的结果。可以用两个反引号，不过这种方式容易混淆，所以还有一种方式就是使用`$(...)` ,现在多用此。

```
$# 传递到脚本的参数个数
$*	以一个单字符串显示所有向脚本传递的参数
$$	脚本运行的当前进程ID号
$!	后台运行的最后一个进程的ID号
$@	与$#相同，但是使用时加引号，并在引号中返回每个参数。
$-	显示Shell使用的当前选项，与set命令功能相同。
$?	显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误
```
