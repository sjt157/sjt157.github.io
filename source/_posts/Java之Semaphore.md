---
title: Java之Semaphore
date: 2019-03-17 10:24:24
tags: Java
categories: Java
---

### N个线程循环打印

```java
public class 老司机1 implements Runnable {

    private static final Object LOCK = new Object();
    /**
     * 当前即将打印的数字
     */
    private static int current = 0;
    /**
     * 当前线程编号，从0开始
     */
    private int threadNo;
    /**
     * 线程数量
     */
    private int threadCount;
    /**
     * 打印的最大数值
     */
    private int maxInt;

    public 老司机1(int threadNo, int threadCount, int maxInt) {
        this.threadNo = threadNo;
        this.threadCount = threadCount;
        this.maxInt = maxInt;
    }

    @Override
    public void run() {
        while (true) {
            synchronized (LOCK) {
                // 判断是否轮到当前线程执行
                while (current % threadCount != threadNo) {
                    if (current > maxInt) {
                        break;
                    }
                    try {
                        // 如果不是，则当前线程进入wait
                        LOCK.wait();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
                // 最大值跳出循环
                if (current > maxInt) {
                    break;
                }
                System.out.println("thread" + threadNo + " : " + current);
                current++;
                // 唤醒其他wait线程
                LOCK.notifyAll();
            }
        }
    }

    public static void main(String[] args) {
        int threadCount = 3;
        int max = 100;
        for (int i = 0; i < threadCount; i++) {
            new Thread(new 老司机1(i, threadCount, max)).start();
        }
    }
}



```

### 上述方法存在的问题
核心方法在run里面，可以看见和我们交替打印奇偶数原理差不多，这里将我们的notify改成了notifyAll，这里要注意一下很多人会将notifyAll理解成其他wait的线程全部都会执行，其实是错误的。这里只会将wait的线程解除当前wait状态，也叫作唤醒，由于我们这里用同步锁synchronized块包裹住，那么唤醒的线程会做会抢夺同步锁。
这个老司机的代码的确能跑通，但是有一个问题是什么呢？当我们线程数很大的时候，由于我们不确定唤醒的线程到底是否是下一个要执行的就有可能会出现抢到了锁但不该自己执行，然后又进入wait的情况，比如现在有100个线程，现在是第一个线程在执行，他执行完之后需要第二个线程执行，但是第100个线程抢到了，发现不是自己然后又进入wait，然后第99个线程抢到了，发现不是自己然后又进入wait，然后第98,97...直到第3个线程都抢到了，最后才到第二个线程抢到同步锁，这里就会白白的多执行很多过程，虽然最后能完成目标。

### 通过Semaphore进行改进

#### Mutex与Semaphore的比较

* Mutex是一把钥匙，一个人拿了就可进入一个房间，出来的时候把钥匙交给队列的第一个。一般的用法是用于串行化对critical section代码的访问，保证这段代码不会被并行的运行。

* Semaphore是一件可以容纳N人的房间，如果人不满就可以进去，如果人满了，就要等待有人出来。对于N=1的情况，称为binary semaphore。一般的用法是，用于限制对于某一资源的同时访问。

* Binary semaphore与Mutex的差异：

在 有的系统中Binary semaphore与Mutex是没有差异的。在有的系统上，主要的差异是mutex一定要由获得锁的进程来释放。而semaphore可以由其它进程释 放（这时的semaphore实际就是个原子的变量，大家可以加或减），因此semaphore可以用于进程间同步。Semaphore的同步功能是所有 系统都支持的，而Mutex能否由其他进程释放则未定，因此建议mutex只用于保护critical section。而semaphore则用于保护某变量，或者同步。

#### semaphore
信号量(Semaphore)，有时被称为信号灯，是在多线程环境下使用的一种设施, 它负责协调各个线程, 以保证它们能够正确、合理的使用公共资源。 

Semaphore分为单值和多值两种，前者只能被一个线程获得，后者可以被若干个线程获得。

#### 举个栗子
以一个停车场是运作为例。为了简单起见，假设停车场只有三个车位，一开始三个车位都是空的。这是如果同时来了五辆车，看门人允许其中三辆不受阻碍的进入，然后放下车拦，剩下的车则必须在入口等待，此后来的车也都不得不在入口处等待。这时，有一辆车离开停车场，看门人得知后，打开车拦，放入一辆，如果又离开两辆，则又可以放入两辆，如此往复。
在这个停车场系统中，车位是公共资源，每辆车好比一个线程，看门人起的就是信号量的作用。
更进一步，信号量的特性如下：信号量是一个非负整数（车位数），所有通过它的线程（车辆）都会将该整数减一（通过它当然是为了使用资源），当该整数值为零时，所有试图通过它的线程都将处于等待状态。在信号量上我们定义两种操作： Wait（等待） 和 Release（释放）。 当一个线程调用Wait等待）操作时，它要么通过然后将信号量减一，要么一自等下去，直到信号量大于一或超时。Release（释放）实际上是在信号量上执行加操作，对应于车辆离开停车场，该操作之所以叫做“释放”是应为加操作实际上是释放了由信号量守护的资源。

#### 如何改进
我们上一个线程持有下一个线程的信号量，通过一个信号量数组将全部关联起来,代码如下: 通过这种方式，我们就不会有白白唤醒的线程，每一个线程都按照我们所约定的顺序去执行，让每个线程的执行都能再你手中得到控制
```java
static int result = 0;
    public static void main(String[] args) throws InterruptedException {
        int N = 3;
        Thread[] threads = new Thread[N];
        final Semaphore[] syncObjects = new Semaphore[N];
        for (int i = 0; i < N; i++) {
            syncObjects[i] = new Semaphore(1);
            if (i != N-1){
                syncObjects[i].acquire();
            }
        }
        for (int i = 0; i < N; i++) {
            final Semaphore lastSemphore = i == 0 ? syncObjects[N - 1] : syncObjects[i - 1];
            final Semaphore curSemphore = syncObjects[i];
            final int index = i;
            threads[i] = new Thread(new Runnable() {

                public void run() {
                    try {
                        while (true) {
                            lastSemphore.acquire();
                            System.out.println("thread" + index + ": " + result++);
                            if (result > 100){
                                System.exit(0);
                            }
                            curSemphore.release();
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                }
            });
            threads[i].start();
        }
    }
```
#### 另外一个方法

```java
package chapter_1_stackandqueue;

import java.util.concurrent.Semaphore;

/**
 * 
 * 整体思路，每个线程持有一把锁，初始化，只有第一个线程有锁
 * 执行完成之后把锁传递下去
 *
 */
public class a {

    /**
     * 最大线程数
     */
    public static final int THREAD_NUMBER = 10;

    /**
     * 最大数
     */
    public static final int MAX_COUNT = 100;

    /**
     * 输出值
     */
    private static int count = 0;

    public static void main(String[] args) {
        Semaphore[] semaphores = new Semaphore[THREAD_NUMBER];
        for (int i = 0; i < THREAD_NUMBER; i++) {
            if (i == 0) {
                semaphores[i] = new Semaphore(1);
            } else {
                semaphores[i] = new Semaphore(0);
            }
        }

        for (int i = 0; i < THREAD_NUMBER; i++) {
            new Thread(new TestThread(semaphores, i)).start();
        }
    }


    public static class TestThread implements Runnable {

        private Semaphore[] semaphores;

        private int number = 0;

        public TestThread(Semaphore[] semaphores, int number) {
            this.number = number;
            this.semaphores = semaphores;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    semaphores[number].acquire();
                    if (count >= MAX_COUNT) {
                        break;
                    }
                    System.out.println(Thread.currentThread().getName() + "：" + count);
                    count++;
                    int current = number + 1;
                    if (current >= THREAD_NUMBER) {
                        current = 0;
                    }
                    semaphores[current].release();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}

```

### 参考
<https://juejin.im/post/5c89b9515188257e5b2befdd>
