---
title: JVM致命错误日志hs_err_pid.log分析
date: 2019-03-11 17:59:38
tags: JVM
categories: JVM
---


### 实验出问题了

在服务器上执行
`nohup java -jar server-v1.0.jar -d /data/wave/wzm/out3/2018-03/ -s 172.18.20.215 -t wave201803 > nohup5.out 2>&1 &`之后的nohup.out文件，如下
```java
nohup: ignoring input
ERROR StatusLogger No log4j2 configuration file found. Using default configuration: logging only errors to the console.
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
完成 resource 初始化！
22:14:44.694 [nioEventLoopGroup-2-1] ERROR com.globigdata.server.LineParseHandler - java.io.IOException: Invalid argument
5, tid=0x00007f799d0fb700
#
# JRE version: Java(TM) SE Runtime Environment (8.0_171-b11) (build 1.8.0_171-b11)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.171-b11 mixed mode linux-amd64 compressed oops)
# Problematic frame:
# J 1748 C2 sun.nio.ch.IOUtil.write(Ljava/io/FileDescriptor;[Ljava/nio/ByteBuffer;IILsun/nio/ch/NativeDispatcher;)J (509 bytes) @ 0x00007f79a5
2ee461 [0x00007f79a52edce0+0x781]#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /data/hs_err_pid18285.log
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
#
```

### 什么时候会产生该文件？
当JVM发生致命错误导致崩溃时，会生成一个hs_err_pid_xxx.log这样的文件，该文件包含了导致 JVM crash 的重要信息，我们可以通过分析该文件定位到导致 JVM Crash 的原因，从而修复保证系统稳定。

### he_err_pid.log包含的内容
#### 日志头文件
```java
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x00007f79a52ee461, pid=18285, tid=0x00007f799d0fb700
#
# JRE version: Java(TM) SE Runtime Environment (8.0_171-b11) (build 1.8.0_171-b11)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.171-b11 mixed mode linux-amd64 compressed oops)
# Problematic frame:
# J 1748 C2 sun.nio.ch.IOUtil.write(Ljava/io/FileDescriptor;[Ljava/nio/ByteBuffer;IILsun/nio/ch/NativeDispatcher;)J (509 bytes) @ 0x00007f79a5
2ee461 [0x00007f79a52edce0+0x781]#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp

```
这段内容主要简述了导致 JVM Crash 的原因。常见的原因有 JVM 自身的 bug，应用程序错误，JVM 参数，服务器资源不足，JNI 调用错误等。当然还有一些版本和配置信息，
```java
 SIGSEGV (0xb) at pc=0x00007f79a52ee461, pid=18285, tid=0x00007f799d0fb700
```
非预期的错误被 JRE 检测到了，其中

* SIGSEGV ：信号量
* 0xb ：信号码
* pc=0x00007f79a52ee461 ：程序计数器的值
* pid=18285 ：进程号
* tid=0x00007f799d0fb700：线程号

SIGSEGV(0xb) 表示 JVM Crash 时正在执行 JNI 代码，常见的描述还有EXCEPTION_ACCESS_VIOLATION，该描述表示 JVM Crash 时正在执行 JVM 自身的代码，这往往是因为 JVM 的 Bug 导致的 Crash；另一种常见的描述是EXCEPTION_STACK_OVERFLOW，该描述表示这是个栈溢出导致的错误，这往往是应用程序中存在深层递归导致的。
```java
# Problematic frame:
# J 1748 C2 sun.nio.ch.IOUtil.write(Ljava/io/FileDescriptor;[Ljava/nio/ByteBuffer;IILsun/nio/ch/NativeDispatcher;)J (509 bytes) @ 0x00007f79a5
2ee461 [0x00007f79a52edce0+0x781]#
```
这个信息比较重要，问题帧信息：

C : 表示帧类型为本地帧
j ：解释的Java帧
V : 虚拟机帧
v ：虚拟机生成的存根栈帧
J ：其他帧类型，包括编译后的Java帧

#### 导致 crash 的线程信息

```java
---------------  T H R E A D  ---------------

Current thread (0x00007f79b462c000):  JavaThread "nioEventLoopGroup-2-1" [_thread_in_Java, id=18354, stack(0x00007f799cffb000,0x00007f799d0fc0
00)]
//表示导致虚拟机终止的非预期的信号信息
siginfo: si_signo: 11 (SIGSEGV), si_code: 1 (SEGV_MAPERR), si_addr: 0x00007f789bc0e010

Registers:
RAX=0x000000075390acc8, RBX=0x0000000789d35f30, RCX=0x00007f790d624180, RDX=0x0000000000000000
RSP=0x00007f799d0fa5a0, RBP=0x0000000000000008, RSI=0x00000000eaf66bf5, RDI=0x0000000003a9c856
R8 =0x000000074d36a0f8, R9 =0x0000000000000067, R10=0x00007f789bc0e010, R11=0x00000000f8008092
R12=0x0000000000000000, R13=0x00007f79b651b000, R14=0x0000000001000000, R15=0x00007f79b462c000
RIP=0x00007f79a52ee461, EFLAGS=0x0000000000010202, CSGSFS=0x0000000000000033, ERR=0x0000000000000006
  TRAPNO=0x000000000000000e

Top of Stack: (sp=0x00007f799d0fa5a0)
0x00007f799d0fa5a0:   000000074d36a0f8 0000000000000000
0x00007f799d0fa5b0:   d9755167d974151e 00007f799d0fa6e8
0x00007f799d0fa5c0:   000000075390acc8 00007f79a536192c
0x00007f799d0fa5d0:   00000000d97580ab 00007f799d0fa620
0x00007f799d0fa5e0:   00007f799d0fa630 00000006cbab8a10
0x00007f799d0fa5f0:   00000006cbc133f0 00000002000733c0
0x00007f799d0fa600:   00000006cba00668 000733bf00000001
0x00007f799d0fa610:   00000006cba0a8e0 00007f79a5423da4
0x00007f799d0fa620:   00000006cba0a860 000000074d36a0f8
0x00007f799d0fa630:   000733c07e2166e0 00000006cba0a870
0x00007f799d0fa640:   00000006cba0a8e0 0000000001801f04
0x00007f799d0fa650:   00000006000733c0 d974151c7e216868
0x00007f799d0fa660:   00000006cbac04b0 0000000700000000
0x00007f799d0fa670:   0000000000000001 d97580c700000000
0x00007f799d0fa680:   0000000000000000 00000006cba85ef8
0x00007f799d0fa690:   0000000000000031 0000000000000031
0x00007f799d0fa6a0:   0000000000000000 00000006cba0aa60
0x00007f799d0fa6b0:   00000006cba0a8f0 00007f79a552c3d4
0x00007f799d0fa6c0:   00000006cba0a958 0000000001801f04
0x00007f799d0fa6d0:   000000074d36a0f8 d974150e000733c0
0x00007f799d0fa6e0:   0000000000000000 000000000000000f
0x00007f799d0fa6f0:   00000006cba0a870 0000000025c5eeec
0x00007f799d0fa700:   00000006cba0aa60 00007f79a537d62c
0x00007f799d0fa710:   00000006cba0a930 00007f79a516830c
0x00007f799d0fa720:   00000006cba0a930 00000000d974151e
0x00007f799d0fa730:   00000006cba0a8f0 00007f79b462c000
0x00007f799d0fa740:   00007f799d0fa770 00007f79bc56196d
0x00007f799d0fa750:   0000000000000004 00007f79bbd76eff
0x00007f799d0fa760:   0000000000000031 0000000025c6a2c9
0x00007f799d0fa770:   00000000d974152b 00007f79a55495e4
0x00007f799d0fa780:   0000000000000400 d9750bed00000032
0x00007f799d0fa790:   0000608676b9d6c9 0000000000000032

Instructions: (pc=0x00007f79a52ee461)
0x00007f79a52ee441:   f8 0f 85 58 05 00 00 48 63 c9 48 03 4b 10 83 fd
0x00007f79a52ee451:   04 0f 84 34 05 00 00 44 8b 50 18 4f 8b 54 d4 18
0x00007f79a52ee461:   49 89 0a 4d 63 d9 4d 89 5a 08 41 ba 01 00 00 00
0x00007f79a52ee471:   8b 7c 24 58 e9 fd f8 ff ff 33 c0 e9 51 fe ff ff

Register to memory mapping:

RAX=0x000000075390acc8 is an oop
sun.nio.ch.IOVecWrapper
 - klass: 'sun/nio/ch/IOVecWrapper'
RBX=0x0000000789d35f30 is an oop
RCX=0x00007f790d624180 is an unknown value
RDI=0x0000000003a9c856 is an unknown value
R8 =0x000000074d36a0f8 is an oop
[Ljava.nio.ByteBuffer;
 - klass: 'java/nio/ByteBuffer'[]
 - length: 16777216
R9 =0x0000000000000067 is an unknown value
R10=0x00007f789bc0e010 is an unknown value
R11=0x00000000f8008092 is an unknown value
R12=0x0000000000000000 is an unknown value
R13=0x00007f79b651b000 is an unknown value
R14=0x0000000001000000 is an unknown value
R15=0x00007f79b462c000 is a thread

//栈顶程序计数器旁的操作码，它们可以被反汇编成系统崩溃前执行的指令。
Stack: [0x00007f799cffb000,0x00007f799d0fc000],  sp=0x00007f799d0fa5a0,  free space=1021k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)

//线程栈信息。包含了地址、栈顶、栈计数器和线程尚未使用的栈信息。到这里就基本上已经确定了问题所在原因了。
Stack: [0x00007f799cffb000,0x00007f799d0fc000],  sp=0x00007f799d0fa5a0,  free space=1021k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
J 2132 C2 sun.nio.ch.SocketChannelImpl.write([Ljava/nio/ByteBuffer;II)J (436 bytes) @ 0x00007f79a5423da4 [0x00007f79a5423a20+0x384]
J 2159 C2 io.netty.channel.socket.nio.NioSocketChannel.doWrite(Lio/netty/channel/ChannelOutboundBuffer;)V (259 bytes) @ 0x00007f79a552c3d4 [0x
00007f79a552bf00+0x4d4]J 2300 C2 io.netty.channel.nio.NioEventLoop.processSelectedKey(Ljava/nio/channels/SelectionKey;Lio/netty/channel/nio/AbstractNioChannel;)V (14
2 bytes) @ 0x00007f79a516830c [0x00007f79a5168180+0x18c]J 2134% C2 io.netty.channel.nio.NioEventLoop.run()V (221 bytes) @ 0x00007f79a55495e4 [0x00007f79a5548f40+0x6a4]
j  io.netty.util.concurrent.SingleThreadEventExecutor$5.run()V+44
j  io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run()V+4
j  java.lang.Thread.run()V+11
v  ~StubRoutines::call_stub
V  [libjvm.so+0x695b96]  JavaCalls::call_helper(JavaValue*, methodHandle*, JavaCallArguments*, Thread*)+0x1056
V  [libjvm.so+0x6960a1]  JavaCalls::call_virtual(JavaValue*, KlassHandle, Symbol*, Symbol*, JavaCallArguments*, Thread*)+0x321
V  [libjvm.so+0x696537]  JavaCalls::call_virtual(JavaValue*, Handle, KlassHandle, Symbol*, Symbol*, Thread*)+0x47
V  [libjvm.so+0x71596e]  thread_entry(JavaThread*, Thread*)+0x7e
V  [libjvm.so+0xa7f243]  JavaThread::thread_main_inner()+0x103
V  [libjvm.so+0xa7f38c]  JavaThread::run()+0x11c
V  [libjvm.so+0x92e0f8]  java_start(Thread*)+0x108
C  [libpthread.so.0+0x7e25]  start_thread+0xc5
```
这部分内容包含触发 JVM 致命错误的线程详细信息和线程栈
```java
Current thread (0x00007f79b462c000):  JavaThread "nioEventLoopGroup-2-1" [_thread_in_Java, id=18354, stack(0x00007f799cffb000,0x00007f799d0fc0
00)]
```
* 0x00007f79b462c000：出错的线程指针
* JavaThread：线程类型，可能的类型包括
JavaThread：Java线程
VMThread : JVM 的内部线程
CompilerThread：用来调用JITing，实时编译装卸class 。 通常，jvm会启动多个线程来处理这部分工作，线程名称后面的数字也会累加，例如：CompilerThread1
GCTaskThread：执行gc的线程
WatcherThread：JVM 周期性任务调度的线程，是一个单例对象
ConcurrentMarkSweepThread：jvm在进行CMS GC的时候，会创建一个该线程去进行GC，该线程被创建的同时会创建一个SurrogateLockerThread（简称SLT）线程并且启动它，SLT启动之后，处于等待阶段。CMST开始GC时，会发一个消息给SLT让它去获取Java层Reference对象的全局锁：Lock

* nioEventLoopGroup-2-1：线程名称
* _thread_in_native：当前线程状态。该描述还包含有：

_thread_in_native：线程当前状态，状态枚举包括：
_thread_uninitialized：线程还没有创建，它只在内存原因崩溃的时候才出现
_thread_new：线程已经被创建，但是还没有启动
_thread_in_native：线程正在执行本地代码，一般这种情况很可能是本地代码有问题
_thread_in_vm：线程正在执行虚拟机代码
_thread_in_Java：线程正在执行解释或者编译后的Java代码
_thread_blocked：线程处于阻塞状态
…_trans：以_trans结尾，线程正处于要切换到其它状态的中间状态

* id=18354：线程ID
* stack(0x00007f799cffb000,0x00007f799d0fc000)：栈区间



#### 所有线程信息

```java
Java Threads: ( => current thread )
  0x00007f79280ae000 JavaThread "Thread-1" [_thread_blocked, id=18363, stack(0x00007f799caf8000,0x00007f799cbf9000)]
  0x00007f7928076800 JavaThread "kafka-producer-network-thread | producer-1" daemon [_thread_in_native, id=18361, stack(0x00007f799cbf9000,0x0
0007f799ccfa000)]  0x00007f7928011800 JavaThread "threadDeathWatcher-3-1" daemon [_thread_blocked, id=18355, stack(0x00007f799ccfa000,0x00007f799cdfb000)]
=>0x00007f79b462c000 JavaThread "nioEventLoopGroup-2-1" [_thread_in_Java, id=18354, stack(0x00007f799cffb000,0x00007f799d0fc000)]
  0x00007f79b4268800 JavaThread "Service Thread" daemon [_thread_blocked, id=18341, stack(0x00007f799dedf000,0x00007f799dfe0000)]
  0x00007f79b4253000 JavaThread "C1 CompilerThread3" daemon [_thread_blocked, id=18338, stack(0x00007f799dfe1000,0x00007f799e0e1000)]
  0x00007f79b4251000 JavaThread "C2 CompilerThread2" daemon [_thread_blocked, id=18336, stack(0x00007f799e0e2000,0x00007f799e1e2000)]
  0x00007f79b424f000 JavaThread "C2 CompilerThread1" daemon [_thread_blocked, id=18335, stack(0x00007f799e1e3000,0x00007f799e2e3000)]
  0x00007f79b424c000 JavaThread "C2 CompilerThread0" daemon [_thread_blocked, id=18333, stack(0x00007f799e2e4000,0x00007f799e3e4000)]
  0x00007f79b424a800 JavaThread "Signal Dispatcher" daemon [_thread_blocked, id=18331, stack(0x00007f799e3e4000,0x00007f799e4e5000)]
  0x00007f79b4217800 JavaThread "Finalizer" daemon [_thread_blocked, id=18328, stack(0x00007f799e4e5000,0x00007f799e5e6000)]
  0x00007f79b4213000 JavaThread "Reference Handler" daemon [_thread_blocked, id=18325, stack(0x00007f799e5e6000,0x00007f799e6e7000)]
  0x00007f79b4009000 JavaThread "main" [_thread_blocked, id=18288, stack(0x00007f79bcf58000,0x00007f79bd058000)]
```
_thread_blocked表示阻塞
#### 安全点和锁信息

```java
VM state:not at safepoint (normal execution)
```
虚拟机状态。not at safepoint 表示正常运行。其余状态：

at safepoint：所有线程都因为虚拟机等待状态而阻塞，等待一个虚拟机操作完成；
synchronizing：一个特殊的虚拟机操作，要求虚拟机内的其它线程保持等待状态。
```java
VM Mutex/Monitor currently owned by a thread: None
```
虚拟机的 Mutex 和 Monitor目前没有被线程持有。Mutex 是虚拟机内部的锁，而 Monitor 则关联到了 Java 对象。

#### 堆信息
```java
Heap:
 PSYoungGen      total 889856K, used 459726K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 64% used [0x000000076e900000,0x000000077ffbb808,0x0000000789c00000)
  from space 444416K, 39% used [0x0000000789c00000,0x0000000794638000,0x00000007a4e00000)
  to   space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
 ParOldGen       total 2669568K, used 2452299K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 91% used [0x00000006cba00000,0x00000007614d2d60,0x000000076e900000)
 Metaspace       used 15147K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Card table byte_map: [0x00007f79b9b78000,0x00007f79ba31c000] byte_map_base: 0x00007f79b651b000
```
新生代、老年代、元空间一目了然。

Card table表示一种卡表，是 jvm 维护的一种数据结构，用于记录更改对象时的引用，以便 gc 时遍历更少的 table 和 root。

#### 本地代码缓存
```java
CodeCache: size=245760Kb used=5251Kb max_used=5700Kb free=240509Kb
 bounds [0x00007f79a5000000, 0x00007f79a55b0000, 0x00007f79b4000000]
 total_blobs=2174 nmethods=1737 adapters=350
 compilation: enabled
```
一块用于编译和保存本地代码的内存。
#### 编译事件
```java
Compilation events (10 events):
Event: 6618.434 Thread 0x00007f79b424f000 2366       4       io.netty.util.internal.PromiseNotificationUtil::tryFailure (66 bytes)
Event: 6618.439 Thread 0x00007f79b424f000 nmethod 2366 0x00007f79a53d9110 code [0x00007f79a53d9260, 0x00007f79a53d9328]
Event: 6618.486 Thread 0x00007f79b424c000 nmethod 2364 0x00007f79a52d9a50 code [0x00007f79a52d9d60, 0x00007f79a52db770]
Event: 6618.722 Thread 0x00007f79b4251000 nmethod 2361% 0x00007f79a54b0550 code [0x00007f79a54b0860, 0x00007f79a54b21f0]
Event: 6618.723 Thread 0x00007f79b424f000 2367   !   4       io.netty.channel.ChannelOutboundBuffer::failFlushed (42 bytes)
Event: 6618.726 Thread 0x00007f79b424f000 nmethod 2367 0x00007f79a540c810 code [0x00007f79a540c980, 0x00007f79a540ca78]
Event: 6651.937 Thread 0x00007f79b4253000 2368       3       sun.misc.Unsafe::getAndAddLong (27 bytes)
Event: 6651.939 Thread 0x00007f79b4253000 nmethod 2368 0x00007f79a5529c10 code [0x00007f79a5529d80, 0x00007f79a5529f90]
Event: 6652.498 Thread 0x00007f79b424f000 2369       4       io.netty.util.internal.PlatformDependent0::directBufferAddress (8 bytes)
Event: 6652.519 Thread 0x00007f79b424f000 nmethod 2369 0x00007f79a5147050 code [0x00007f79a5147180, 0x00007f79a51471b8]
```
记录10次编译事件。这里的信息也印证了上面的结论。
#### gc 相关记录
```java
GC Heap History (10 events):
Event: 6483.329 GC heap before
{Heap before GC invocations=1775 (full 41):
 PSYoungGen      total 889856K, used 433403K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 97% used [0x000000076e900000,0x000000078903ecd8,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669359K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8cbc50,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Event: 6534.282 GC heap after
Heap after GC invocations=1775 (full 41):
 PSYoungGen      total 889856K, used 419683K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 94% used [0x000000076e900000,0x00000007882d8de0,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669358K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8cbbe0,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
}
Event: 6534.282 GC heap before
{Heap before GC invocations=1776 (full 42):
 PSYoungGen      total 889856K, used 419683K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 94% used [0x000000076e900000,0x00000007882d8de0,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669358K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8cbbe0,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Event: 6579.155 GC heap after
Heap after GC invocations=1776 (full 42):
 PSYoungGen      total 889856K, used 419683K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 94% used [0x000000076e900000,0x00000007882d8de0,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669358K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8cbb28,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
}
Event: 6579.728 GC heap before
{Heap before GC invocations=1777 (full 43):
 PSYoungGen      total 889856K, used 445440K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 100% used [0x000000076e900000,0x0000000789c00000,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669362K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8cc8c8,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Event: 6618.338 GC heap after
Heap after GC invocations=1777 (full 43):
 PSYoungGen      total 889856K, used 318952K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 71% used [0x000000076e900000,0x000000078207a230,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669283K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8b8cb0,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
}
Event: 6620.112 GC heap before
{Heap before GC invocations=1778 (full 44):
 PSYoungGen      total 889856K, used 445440K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 100% used [0x000000076e900000,0x0000000789c00000,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2669283K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 99% used [0x00000006cba00000,0x000000076e8b8cb0,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Event: 6649.833 GC heap after
Heap after GC invocations=1778 (full 44):
 PSYoungGen      total 889856K, used 0K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 0% used [0x000000076e900000,0x000000076e900000,0x0000000789c00000)
  from space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
  to   space 444416K, 0% used [0x0000000789c00000,0x0000000789c00000,0x00000007a4e00000)
 ParOldGen       total 2669568K, used 2452299K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 91% used [0x00000006cba00000,0x00000007614d2d60,0x000000076e900000)
 Metaspace       used 15146K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
}
}

Deoptimization events (10 events):
 ParOldGen       total 2669568K, used 2452299K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 91% used [0x00000006cba00000,0x00000007614d2d60,0x000000076e900000)
 Metaspace       used 15147K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
Event: 6657.512 GC heap after
Heap after GC invocations=1779 (full 44):
 PSYoungGen      total 889856K, used 174304K [0x000000076e900000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 445440K, 0% used [0x000000076e900000,0x000000076e900000,0x0000000789c00000)
  from space 444416K, 39% used [0x0000000789c00000,0x0000000794638000,0x00000007a4e00000)
  to   space 444416K, 0% used [0x00000007a4e00000,0x00000007a4e00000,0x00000007c0000000)
 ParOldGen       total 2669568K, used 2452299K [0x00000006cba00000, 0x000000076e900000, 0x000000076e900000)
  object space 2669568K, 91% used [0x00000006cba00000,0x00000007614d2d60,0x000000076e900000)
 Metaspace       used 15147K, capacity 15314K, committed 15616K, reserved 1062912K
  class space    used 1858K, capacity 1927K, committed 2048K, reserved 1048576K
}

```

#### jvm 内存映射
```java
Dynamic libraries:
00400000-00401000 r-xp 00000000 fd:00 2149917265                         /usr/java/jdk1.8.0_171/bin/java
00600000-00601000 rw-p 00000000 fd:00 2149917265                         /usr/java/jdk1.8.0_171/bin/java
01b84000-01ba5000 rw-p 00000000 00:00 0                                  [heap]
6cba00000-76e900000 rw-p 00000000 00:00 0
76e900000-7c0000000 rw-p 00000000 00:00 0
7c0000000-7c0200000 rw-p 00000000 00:00 0
7c0200000-800000000 ---p 00000000 00:00 0
7f78a4000000-7f78a7001000 rw-p 00000000 00:00 0
7f78a7001000-7f78a8000000 ---p 00000000 00:00 0
7f78a8000000-7f78ab001000 rw-p 00000000 00:00 0
7f78ab001000-7f78ac000000 ---p 00000000 00:00 0
7f78ac000000-7f78af001000 rw-p 00000000 00:00 0
7f78af001000-7f78b0000000 ---p 00000000 00:00 0
7f78b0000000-7f78b3001000 rw-p 00000000 00:00 0
.........
```
这些信息是虚拟机崩溃时的虚拟内存列表区域。它可以告诉你崩溃原因时哪些类库正在被使用，位置在哪里，还有堆栈和守护页信息。

* 00400000-00401000：内存区域
* r-xp：权限，r/w/x/p/s分别表示读/写/执行/私有/共享
* 00000000：文件内的偏移量


#### jvm 启动参数
```java
VM Arguments:
java_command: server-v1.0.jar -d /data/wave/wzm/out3/2018-02/ -s 172.18.20.215 -t wave201802
java_class_path (initial): server-v1.0.jar
Launcher Type: SUN_STANDARD

Environment Variables:
JAVA_HOME=/usr/java/jdk1.8.0_171
JRE_HOME=/usr/java/jdk1.8.0_171/jre
CLASSPATH=.:/usr/java/jdk1.8.0_171/lib/dt.jar:/usr/java/jdk1.8.0_171/lib/tools.jar:/usr/java/jdk1.8.0_171/jre/lib
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/java/jdk1.8.0_171/bin:/usr/java/jdk1.8.0_171/jre/bin:/root/bin
SHELL=/bin/bash

```
jvm 虚拟机参数和环境变量

#### 服务器信息
```java
---------------  S Y S T E M  ---------------

OS:CentOS Linux release 7.4.1708 (Core)

uname:Linux 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64
libc:glibc 2.17 NPTL 2.17
rlimit: STACK 8192k, CORE 0k, NPROC 62424, NOFILE 102400, AS infinity
load average:19.77 19.23 14.25
```

### 参考
<https://www.jianshu.com/p/7652f931cafd>


