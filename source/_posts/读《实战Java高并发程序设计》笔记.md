---
title: 读《实战Java高并发程序设计》笔记
date: 2019-06-02 09:51:30
tags: Java
categories: Java
---


* 一句说解释活锁

两个线程都主动释放资源给对方，就会导致这个资源在两个线程间来回跳动，谁也拿不到资源执行，就类似于两个人互相谦让。

* 并发级别
阻塞（synchroized、lock）、无饥饿（公平锁与非公平锁）、无障碍（直接去操作，发生冲突回滚，类似，类似乐观锁）、无锁（CAS）、无等待（RCU Read Copy Update 读不限制，修改副本）

* 指令重排

指令重排是为了减少中断，CPU是流水线模型。
哪些指令不能重排：Happens before规则

* 线程状态
线程的等待在等什么？wait的等待线程在等notify，join的等待线程在等目标线程的终止。

* 启动线程
要用start(),start创建一个新的线程并运行run方法。如果只用run（），代表在当前线程执行，只是一个普通的方法。

* 线程等待
Object.wait和Thread.sleep(),都可以让线程等待。wait可以被唤醒，wait释放锁。sleep不释放锁。

* 守护线程
设置守护线程，必须在start()方法前设置，否则线程永远不会停下来。

* HashMap的死循环问题在jdk8不会出现

* 可重入锁
重入锁可以提供中断处理的能力。比如一个等待的线程，他在等待锁，他可以收到一个通知，被告知无需等待了。也就是在等待锁的过程中，可以响应等中断。而对于synchroized来讲，就是要么有锁执行，要么无锁等待。
重入锁可以是公平的。

* Condition
Condition与重入锁配合，Synchronized与wait和notify配合
```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
* @author 宋建涛 E-mail:1872963677@qq.com
* @version 创建时间：2019年6月1日 下午7:47:12
* 类说明
*/
public class ReenterLockCondition implements Runnable {
	public static ReentrantLock lock = new ReentrantLock();
	public static Condition condition = lock.newCondition();
	@Override
	public void run() {
		try {
			lock.lock();
			condition.await();//在condition上等待
			System.out.println("Thread is going on");
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
	
	public static void  main(String[] args) throws InterruptedException {
		ReenterLockCondition t1 = new ReenterLockCondition();
		Thread t= new Thread(t1);
		t.start();
		Thread.sleep(2000);
		//通知线程t继续执行
		lock.lock(); //先获取相关的锁
		condition.signal(); //告诉等待在condotion上的线程可以继续执行了
		lock.unlock();
	}
}

```

* 信号量

锁一次只能要求一个线程操控共享资源，而信号量可以允许多个。比如你要让5个线程5个线程的执行，可以设置一个信号量为5.

* 线程阻塞工具类
LockSupport，可以在线程内任意位置让线程阻塞。与Object.wait（）相比，他不需要先获得某个对象的锁。使用park函数时，可以为当前线程设置一个阻塞对象，这个阻塞对象会出现在线程Dump中，更加容易分析问题。

* 线程池的队列
如果使用无界队列来存储任务，要注意如果任务创建和处理的速度相差很大，可能会造成OOM。


* ForkJoin
```java
import java.util.ArrayList;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;

/**
* @author 宋建涛 E-mail:1872963677@qq.com
* @version 创建时间：2019年6月1日 下午8:32:32
* 类说明
*/
//继承RecursiveTask，可以携带返回值
public class CountTask extends RecursiveTask<Long> {
	private static final int THRESHOLD = 10000;
	private long start;
	private long end;
	
	public CountTask(long start, long end) {
		this.start = start;
		this.end = end;
	}
	public Long compute() {
		long sum = 0;
		boolean canCompute = (end-start)<THRESHOLD;
		if(canCompute) {
			for(long i=start;i<=end;i++) {
				sum +=i;
			}
		} else {
			long step = (start+end)/100;
			ArrayList<CountTask> subTasks =  new ArrayList<CountTask>();
			long pos = start;
			//划分成100个小任务
			for(int i=0;i<100;i++) {
				long lastOne = pos+step;
				if(lastOne>end) lastOne=end;
				CountTask subTask = new CountTask(pos, lastOne);
				pos+=step+1;
				subTasks.add(subTask);
				//提交任务
				subTask.fork();
			}
			for(CountTask t :subTasks) {
				sum+=t.join();
			}
		}
		return sum;
	}
	
	 public static void main(String[] args) {
		 ForkJoinPool forkJoinPool = new ForkJoinPool();
		 CountTask task = new CountTask(0, 200000L);
		 ForkJoinTask<Long> result = forkJoinPool.submit(task);
		 try {
			 long res = result.get();
			 System.out.println(res);
		} catch (Exception e) {
			// TODO: handle exception
		}
		 
	 }
}

```

* 逃逸分析
锁消除的一项关键技术就是逃逸分析，就是观察一个变量是否会逃出每一个作用域。逃逸分析必须在server模式下进行，可以使用-XX：+DoEscapeAnalysis参数进行逃逸分析。使用-XX：+EliminateLocks参数打开锁消除。

* 避免死锁
使用无锁或可重入锁，通过可重入锁的中断或限时等待。

* 创建单例不要使用双重检验方式，太垃圾了

* 不变模式通过回避问题而不是解决问题的态度解决多线程并发访问控制，不变对象是不需要进行同步的。不变模式可以提高系统的并发性能和并发量。

* ConcurrentLinkedQueue队列性能很高，因为使用了CAS。但是使用CAS编程很难，但是我们可以利用Disruptor框架。

* CPU cache的优化：解决伪共享问题。CPU有个高速缓存，高速缓存读写数据最小的单位是缓存行。如果两个变量放在一个缓存行中，在多线程访问中，可能会让缓存行失效导致效率低下，所以可以通过填充padding的方式来不让这两个变量在同一个缓存行。

* 冒泡排序、希尔排序都可以并行化

* 虽然NIO在网络操作中提供了非阻塞的方法，但是IO行为还是同步的，在IO操作准备好时就通知你，但是AIO是IO完成后才通知你。通过回调函数。

* 读写锁的改进：StampedLock。读写锁用的悲观锁策略，StampedLock用的乐观锁策略。

* 如何调试并行程序。

* Jetty核心代码示例






