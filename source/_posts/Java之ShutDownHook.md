---
title: Java之ShutDownHook
date: 2019-01-18 17:05:13
tags: Java
categories: Java
---



### 线上JVM挂掉怎么办，不要怕，有优雅停机

在线上Java程序中经常遇到进程程挂掉，一些状态没有正确的保存下来，这时候就需要在JVM关掉的时候执行一些清理现场的代码。Java中得ShutdownHook提供了比较好的方案。

### 什么时候可以用钩子

1）程序正常退出
2）使用System.exit()
3）终端使用Ctrl+C触发的中断
4）系统关闭
5）使用Kill pid命令干掉进程（kill -9 pid不会调用钩子）
6) OOM宕机


### 如何添加钩子

`Runtime.addShutdownHook(Thread hook)`

```Java
在JDK中方法的声明：
public void addShutdownHook(Thread hook)
参数
hook -- 一个初始化但尚未启动的线程对象，注册到JVM钩子的运行代码。
异常
IllegalArgumentException -- 如果指定的钩已被注册，或如果它可以判定钩已经运行或已被运行
IllegalStateException -- 如果虚拟机已经是在关闭的过程中
SecurityException -- 如果存在安全管理器并且它拒绝的RuntimePermission（“shutdownHooks”）

```

```Java

/**
	 * JVM的关闭钩子--JVM正常关闭才会执行
	 */
	public static void addHook(){
		Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
			
			public void run() {
				//清空缓存信息
				System.out.println("网关正常关闭前执行  清空所有缓存信息...............................");
				ClientChannelCache.clearAll();
				CacheQueue.clearIpCountRelationCache();
				CacheQueue.clearMasterChannelCache();
			}
		}));
	}

```

### 注意的地方

同一个JVM最好只使用一个关闭钩子，而不是每个服务都使用一个不同的关闭钩子，使用多个关闭钩子可能会出现当前这个钩子所要依赖的服务可能已经被另外一个关闭钩子关闭了。为了避免这种情况，建议关闭操作在单个线程中串行执行，从而避免了再关闭操作之间出现竞态条件或者死锁等问题。

### 参考
<https://www.cnblogs.com/shuo1208/p/5871224.html>
<https://www.cnblogs.com/langtianya/p/4300282.html>



