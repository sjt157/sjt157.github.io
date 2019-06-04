---
title: Java之字节码知识
date: 2019-06-04 14:00:27
tags: Java
categories: Java
---

第一次买了一本掘金小册<JVM 字节码从入门到精通>，花了19.9元，记录下收获。

----------

* 用xxd 命令以 16 进制的方式查看这个 class 文件。

* javap，用来方便的窥探 class 文件内部的细节。javap 有比较多的参数选项，其中-c -v -l -p -s是最常用的。

* javap的-l的作用：-l 输出行及局部变量表。

* 虚拟机常见的实现方式有两种：Stack based 的和 Register(寄存器) based。比如基于 Stack 的虚拟机有Hotspot JVM、.net CLR，这种基于 Stack 实现虚拟机是一种广泛的实现方法。而基于 Register 的虚拟机有 Lua 语言虚拟机 LuaVM 和 Google 开发的安卓虚拟机 DalvikVM。

* 整个 JVM 指令执行的过程就是局部变量表与操作数栈之间不断 load、store 的过程

* 一个对象创建的套路是这样的：new、dup、invokespecial，下次遇到同样的指令要形成条件反射。

* HSDB 全称是：Hotspot Debugger，是内置的 JVM 工具，可以用来深入分析 JVM 运行时的内部状态。HSDB 位于 JDK 安装目录下的 lib/sa-jdi.jar 中， 启动 HSDB

* 随着 JDK7 的发布，字节码指令集新增了一个重量级指令 invokedynamic。这个指令为为多语言在 JVM 上百花齐放提供了坚实的技术支撑。

* 鸭子类型（Duck Typing）：当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子

在鸭子类型中，关注点在于对象的行为，能做什么；而不是关注对象所属的类型，不关注对象的继承关系

* MethodHandle 又被称为方法句柄或方法指针， 是java.lang.invoke 包中的 一个类，它的出现使得 Java 可以像其它语言一样把函数当做参数进行传递。MethodHandle 类似于反射中的 Method 类，但它比 Method 类要更加灵活和轻量级。

```java
public class Foo {
    public void print(String s) {
        System.out.println("hello, " + s);
    }
    public static void main(String[] args) throws Throwable {
        Foo foo = new Foo();

        MethodType methodType = MethodType.methodType(void.class, String.class);
        MethodHandle methodHandle = MethodHandles.lookup().findVirtual(Foo.class, "print", methodType);
        methodHandle.invokeExact(foo, "world");
    }
}

运行输出
hello, world
```

* Groovy 采用 invokedynamic 指令有哪些好处?

标准化。使用 Bootstrap Method、CallSite、MethodHandle 机制使得动态调用的方式得到统一
保持了字节码层的统一和向后兼容。把动态方法的分派逻辑下放到语言实现层，未来版本可以很方便的进行优化、修改
高性能。接近原生 Java 调用的性能，也可以享受到 JIT 优化等

* 关于为什么用 invokedynamic 来实现 Lambda，Oracle 的开发者专门写了一篇文章 Translation of Lambda Expressions，介绍了 Java 8 Lambda 设计时的考虑以及实现方法。 文中提到 Lambda 表达式可以通过内部类、method handle、dynamic proxies 等方式实现，但是这些方法各有优劣。实现 Lambda 表达式需要达成两个目标：

为未来的优化提供最大的灵活性
保持类文件字节码格式的稳定
使用 invokedynamic 可以很好的兼顾这两个目标。
<http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html>

* lambda 表达式与普通的匿名内部类的实现方式不一样，在编译阶段只是新增了一个 invokedynamic 指令，并没有在编译期间生成匿名内部类，lambda 表达式的内容会被编译成一个静态方法。在运行时 LambdaMetafactory.metafactory 这个工厂方法来动态生成一个内部类 class，该内部类会调用前面生成的静态方法。 lambda 表达式最终还是会生成一个内部类，只不过不是在编译期间而是在运行时，未来的 JDK 会怎么实现 Lambda 表达式可能还会有变化。

* JDK7 引入的 String 的 switch 实现流程分为下面几步：

计算字符串 hashCode
使用 lookupswitch 对整型 hashCode 进行分支
对相同 hashCode 值的字符串进行最后的字符串匹配
执行 case 块代码

* finally为什么一定会被执行？
因为底层实现它复制了三份。。。
```java
public void foo() {
    try {
        tryItOut1();
        handleFinally();
    } catch (MyException1 e) {
        handleException(e);
        handleFinally();
    } catch (Throwable e) {
        handleFinally();
        throw e;
    }
}
```

* finally中有return。如果 finally 中有 return，因为它先于其它的执行，会覆盖其它的返回（包括异常）

* 第一，try-with-resource 语法并不是简单的在 finally 里中加入了closable.close()方法，因为 finally 中的 close 方法如果抛出了异常会淹没真正的异常；
第二，引入了 suppressed 异常的概念，能抛出真正的异常，且会调用 addSuppressed 附带上 suppressed 的异常。


* 因为编译器必须保证，无论同步代码块中的代码以何种方式结束（正常 return 或者异常退出），代码中每次调用 monitorenter 必须执行对应的 monitorexit 指令。为了保证这一点，编译器会自动生成一个异常处理器，这个异常处理器的目的就是为了同步代码块抛出异常时能执行 monitorexit。这也是字节码中，只有一个 monitorenter 却有两个 monitorexit 的原因

* 方法级的 synchronized实现原理
方法级的同步与上述有所不同，它是由常量池中方法的 ACC_SYNCHRONIZED 标志来隐式实现的。
```java
synchronized public void testMe() {
}

对应字节码

public synchronized void testMe();
descriptor: ()V
flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```
JVM 不会使用特殊的字节码来调用同步方法，当 JVM 解析方法的符号引用时，它会判断方法是不是同步的（检查方法 ACC_SYNCHRONIZED 是否被设置）。如果是，执行线程会先尝试获取锁。如果是实例方法，JVM 会尝试获取实例对象的锁，如果是类方法，JVM 会尝试获取类锁。在同步方法完成以后，不管是正常返回还是异常返回，都会释放锁

* 泛型真的被完全擦除了吗
LocalVariableTypeTable 和 Signature 是针对泛型引入的新的属性，用来解决泛型的参数类型识别问题，Signature 最为重要，它的作用是存储一个方法在字节码层面的特征签名，这个属性保存的不是原生类型，而是包括了参数化类型的信息。我们依然可以通过反射的方式拿到参数的类型。所谓的擦除，只是把方法 code 属性的字节码进行擦除。

* 因为很多情况下，反射只会调用一次，因此 JVM 想了一招，设置了 15 这个 sun.reflect.inflationThreshold 阈值，反射方法调用超过 15 次时（从 0 开始），采用 ASM 生成新的类，保证后面的调用比 native 要快。如果小于 15 次的情况下，还不如生成直接 native 来的简单直接，还不造成额外类的生成、校验、加载。这种方式被称为 「inflation 机制」。inflation 这个单词也比较有意思，它的字面意思是「膨胀；通货膨胀」。

* 基于 javaagent 来实现的，比如热部署 JRebel、性能调试工具 XRebel、听云、newrelic 等。它能实现的基本功能包括

可以在加载 class 文件之前做拦截，对字节码做修改，可以实现 AOP、调试跟踪、日志记录、性能分析
可以在运行期对已加载类的字节码做变更，可以实现热部署等功能。

* javaagent 有两个重要的入口类：Premain-Class 和 Agent-Class，分别对应入口函数 premain 和 agentmain，其中 agentmain 可以采用远程 attach API 的方式远程挂载另一个 JVM 进程。

* ASM 库是设计模式中访问者模式的典型应用，三大核心类 ClassReader(读取字节码、分析)、ClassVisitor（修改各个节点的字节码）、ClassWriter （ClassWriter 的 toByteArray 方法则把最终修改的字节码以 byte 数组的形式返回）

* 字节码反编译查看工具 jdgui，luyten
字节码浏览工具 jclasslib
ASM
vim、hex editor

* vim 十六进制查看文件
vim -b ./com/jclarity/censum/CensumStartupChecks.class 使用 16 进制模式打开:%!xxd

回到普通模式:%!xxd -r保存退出

* 如果你去做一个商业版本的软件，有哪些手段可以防止别人用类似的手段破解呢？”
1. 使用自定义的 classloader，原来的 class 文件经过加密处理，直接解压拿到的的 class 文件没法直接查看，在 classloader 里面进行解密然后加载
2. 把核心的逻辑写到 jni 层

* 页面加载时间、首屏时间、页面渲染时间 我们在 chrome console 里输入 window.performance.timing 就可以拿到详细的各阶段时间

* 怎么样做嵌码？
Java 服务端：使用我们之前介绍过的 javaagent 字节码 instrument 技术进行字节码改写
Node.js 阿里有开源 pandora.js 可以参考借鉴
安卓：用 gradle 插件在编译期进行 hook
iOS：Hook（Method Swizzling）

* 因为 APM 会产生调用次数放大，一次调用可能产生几十次上百次的的链路调用 trace。因此数据一定要合并上报，减少网络的开销。 这个场景就是 合并上报，指定时间还没达到批量的阈值，有多少条报多少条，针对此场景我写了一个非常简单的工具类，用 BlockingQueue 实现了带超时的批量取

```java
private static final int SIZE = 5000;
private static final int BATCH_FETCH_ITEM_COUNT = 50;
private static final int MAX_WAIT_TIMEOUT = 30;
private BlockingQueue<String> queue = new LinkedBlockingQueue<>(SIZE);
    
public boolean add(final String logItem) {
  
    // 如果队列已满，需要超时等待一段时间，使用此方法
queue.offer(logItem, 10, TimeUnit.MILLISECONDS)
// 如果队列已满，直接需要返回add失败，使用此方法
  return queue.offer(logItem);
}
public List<String> batchGet() {
    List<String> bulkData = new ArrayList<>();
    batchGet(queue, bulkData, BATCH_FETCH_ITEM_COUNT, MAX_WAIT_TIMEOUT, TimeUnit.SECONDS);
    return bulkData;
}
public static <E> int batchGet(BlockingQueue<E> q,Collection<? super E> buffer, int numElements, long timeout, TimeUnit unit) throws InterruptedException {
    long deadline = System.nanoTime() + unit.toNanos(timeout);
    int added = 0;
    while (added < numElements) {
       // drainTo非常高效，我们先尝试批量取，能取多少是多少，不够的poll来凑
        added += q.drainTo(buffer, numElements - added);
        if (added < numElements) {
            E e = q.poll(deadline - System.nanoTime(), TimeUnit.NANOSECONDS);
            if (e == null) {
                break;
            }
            buffer.add(e);
            added++;
        }
    }
    return added;
}
```

* 实时告警 实时数据处理可以用时序数据库，也可以用 Redis + Lua 的方式，有告警的情况下可以达到分钟级别的微信、邮件通知，也可以在业务上做告警收敛，避免告警风暴

* Disruptor在APM系统中可以用在日志模块，异步写日志，也可以用在agent模块，收集trace日志，在collector端根据traceID聚合日志

* 跨进程链路调用的实现
跨进程调用是采用在调用方注入当前 traceId 和 spanId 来实现的。以调用 Dubbo 为例，Dubbo 真正的远程调用是在 com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke()函数
```java
public Result invoke(Invocation invocation) throws RpcException {
    return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
}
```
需要把上述代码改写为
```java
public Result invoke(Invocation invocation) throws RpcException {
    invocation.setAttachment("X-APM-TraceId", "traceId-1001");
    invocation.setAttachment("X-APM-SpanId", "spanId-A001");
    return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
}
```
* 实现一个 APM 系统最核心的部分：字节码改写、ThreadLocal 实现调用栈、跨进程调用。
<https://github.com/arthur-zhang/geek01/tree/master/javaagent-demo>

* 把 Nginx 加入到 APM 链路中来
OpenResty 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。 使用 OpenResty 可以比较灵活的实现添加 header，获取耗时、状态码等信息，用少量的代码就可以把 Nginx 加入到 APM 链路中来
```lua
- 定义Headers
local X_APM_TRACE_ID = "X-APM-TraceId"
local X_APM_SPAN_ID = 'X-APM-SpanId'
local X_APM_SPAN_NAME = 'Nginx'

-- 生成Nginx Span Id
local ngx_span_id = string.gsub(uuid(), '-', '')

-- 从Header中，获取父Span信息
local ngx_span_parent = nil
if req_headers ~= nil then
    ngx_span_parent = req_headers[X_APM_SPAN_ID]
end

-- 向Header中，写入Nginx Span相关信息
local trace_id = req_headers[X_APM_TRACE_ID]
if trace_id == nil then
    trace_id = string.gsub(uuid(), '-', '')
    ngx.req.set_header(X_APM_TRACE_ID, trace_id)
end

ngx.req.set_header(X_APM_SPAN_ID, ngx_span_id)
ngx.req.set_header(X_APM_SPAN_NAME, X_APM_SPAN_NAME)
ngx.req.set_header(X_APM_SAMPLED, X_APM_SAMPLED)
```

* 好的书籍
Java虚拟机规范(Java SE 8版)

这本书我看了很多遍，每次看都有新的收获，强烈推荐

自己动手写Java虚拟机

这本书主要讲的是用 Go 语言来实现 Java 虚拟机，需要你有一点点的 Go 语言基础，我对着这本书敲完了，然后用 Kotlin 重写了一遍，真正了解了 class 文件解析、字节码指令运行的详细细节，也推荐大家看一看

深入理解Java虚拟机:JVM高级特性与最佳实践

这本神书不用多说，很经典，笔试面试必备

JRockit权威指南 深入理解JVM

这本书也是了解 JVM 非常不多的书籍，里面提到了很多调优相关的东西

* ASM guide
<https://asm.ow2.io/asm4-guide.pdf>

### 掘金小册地址
<https://juejin.im/book/5c25811a6fb9a049ec6b23ee>
