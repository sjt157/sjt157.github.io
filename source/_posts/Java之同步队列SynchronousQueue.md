---
title: Java之同步队列SynchronousQueue
date: 2019-06-01 16:43:19
tags: Java
categories: Java
---



### 怎么理解这个“特殊”的队列
一般印象中，阻塞队列都有缓冲，但是这个队列是没有缓冲的，就是说这里边不能存东西。数据是在配对的生产者和消费者线程之间直接传递的，并不会将数据缓冲数据到队列中。可以这样来理解：生产者和消费者互相等待对方，握手（传数据，一个put，一个take），然后一起离开。

### 什么地方需要这个队列

* 非常适合于传递性设计，在这种设计中，在一个线程中运行的对象要将某些信息、事件或任务传递给在另一个线程中运行的对象，它就必须与该对象同步。 
* 在线程池里的一个典型应用是Executors.newCachedThreadPool()就使用了SynchronousQueue，这个线程池根据需要（新任务到来时）创建新的线程，如果有空闲线程则会重复使用，线程空闲了60秒后会被回收。
* 全宇宙的JAVA IT人士应该都知道ThreadPoolExecutor的执行流程：

core线程还能应付的,则不断的创建新的线程;
core线程无法应付,则将任务扔到队列里面;
队列满了(意味着插入任务失败),则开始创建MAX线程,线程数达到MAX后,队列还一直是满的,则抛出RejectedExecutionException.
这个执行流程有个小问题,就是当core线程无法应付请求的时候,会立刻将任务添加到队列中,如果队列非常长,而任务又非常多,那么将会有频繁的任务入队列和任务出队列的操作。

根据实际的压测发现,这种操作也是有一定消耗的。其实JAVA提供的`SynchronousQueue`队列是一个零长度的队列,任务都是直接由生产者递交给消费者,中间没有入队列的过程,可见JAVA API的设计者也是有考虑过入队列这种操作的开销。

#### 扩展ThreadPoolExecutor的一种办法(参考的tomcat的ThreadPoolExecutor源码)
只有当达到MAx数量时，才放入队列。。。。而不是超过core数量时
<https://blog.csdn.net/linsongbin1/article/details/78275283>

### 自己实现SynchronousQueue

#### 使用wait和notify实现
阻塞算法实现通常在内部采用一个锁来保证多个线程中的put()和take()方法是串行执行的。采用锁的开销是比较大的，还会存在一种情况是线程A持有线程B需要的锁，B必须一直等待A释放锁，即使A可能一段时间内因为B的优先级比较高而得不到时间片运行。所以在高性能的应用中我们常常希望规避锁的使用。


```java
/**
* @author 宋建涛 E-mail:1872963677@qq.com
* @version 创建时间：2019年6月1日 下午3:23:59
* 类说明
*/
class NativeSynchronousQueue<E> {
	boolean putting = false;
    E item = null;

    public synchronized E take() throws InterruptedException {
        while (item == null)
            wait();
        E e = item;
        item = null;
        notifyAll();
        return e;
    }

    public synchronized void put(E e) throws InterruptedException {
        if (e == null)
            return;
        while (putting)
            wait();
        putting = true;
        item = e;
        notifyAll();
        while (item != null)
            wait();
        putting = false;
        notifyAll();
    }
}
public class NativeSynchronousQueueTest {
	public static void main(String[] args) throws InterruptedException {
        final NativeSynchronousQueue<String> queue = new NativeSynchronousQueue<String>();
        Thread putThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("put thread start");
                try {
                    queue.put("1");
                } catch (InterruptedException e) {
                }
                System.out.println("put thread end");
            }
        });

        Thread takeThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("take thread start");
                try {
                    System.out.println("take from putThread: " + queue.take());
                } catch (InterruptedException e) {
                }
                System.out.println("take thread end");
            }
        });

        putThread.start();
        Thread.sleep(1000);
        takeThread.start();
    }
}

```
#### 信号量实现
```java
import java.util.concurrent.Semaphore;

/**
* @author 宋建涛 E-mail:1872963677@qq.com
* @version 创建时间：2019年6月1日 下午3:46:28
* 类说明
*/
class SemaphoreSynchronousQueue<E> {
    E item = null;
    Semaphore sync = new Semaphore(0);
    Semaphore send = new Semaphore(1);
    Semaphore recv = new Semaphore(0);
 
    public E take() throws InterruptedException {
        recv.acquire();
        E x = item;
        sync.release();
        send.release();
        return x;
    }
 
    public void put (E x) throws InterruptedException{
        send.acquire();
        item = x;
        recv.release();
        sync.acquire();
    }
}
public class SemaphoreSynchronousQueueTest {
	public static void main(String[] args) throws InterruptedException {
        final SemaphoreSynchronousQueue<String> queue = new SemaphoreSynchronousQueue<String>();
        Thread putThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("put thread start");
                try {
                    queue.put("1");
                } catch (InterruptedException e) {
                }
                System.out.println("put thread end");
            }
        });

        Thread takeThread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("take thread start");
                try {
                    System.out.println("take from putThread: " + queue.take());
                } catch (InterruptedException e) {
                }
                System.out.println("take thread end");
            }
        });

        putThread.start();
        Thread.sleep(1000);
        takeThread.start();
    }
}

```

#### Java5实现
Java 5的实现相对来说做了一些优化，只使用了一个锁，使用队列代替信号量也可以允许发布者直接发布数据，而不是要首先从阻塞在信号量处被唤醒。
```java
import java.util.Queue;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.ReentrantLock;

public class Java5SynchronousQueue<E> {
    ReentrantLock qlock = new ReentrantLock();
    Queue waitingProducers = new Queue();
    Queue waitingConsumers = new Queue();

    static class Node extends AbstractQueuedSynchronizer {
        E item;
        Node next;

        Node(Object x) { item = x; }
        void waitForTake() { /* (uses AQS) */ }
           E waitForPut() { /* (uses AQS) */ }
    }

    public E take() {
        Node node;
        boolean mustWait;
        qlock.lock();
        node = waitingProducers.pop();
        if(mustWait = (node == null))
           node = waitingConsumers.push(null);
         qlock.unlock();

        if (mustWait)
           return node.waitForPut();
        else
            return node.item;
    }

    public void put(E e) {
         Node node;
         boolean mustWait;
         qlock.lock();
         node = waitingConsumers.pop();
         if (mustWait = (node == null))
             node = waitingProducers.push(e);
         qlock.unlock();

         if (mustWait)
             node.waitForTake();
         else
            node.item = e;
    }
}
```

#### java6实现

Java 6的SynchronousQueue的实现采用了一种性能更好的无锁算法 — 扩展的“Dual stack and Dual queue”算法。性能比Java5的实现有较大提升。竞争机制支持公平和非公平两种：非公平竞争模式使用的数据结构是后进先出栈(Lifo Stack)；公平竞争模式则使用先进先出队列（Fifo Queue），性能上两者是相当的，一般情况下，Fifo通常可以支持更大的吞吐量，但Lifo可以更大程度的保持线程的本地化。

代码实现里的Dual Queue或Stack内部是用链表(LinkedList)来实现的，其节点状态为以下三种情况：

持有数据 – put()方法的元素
持有请求 – take()方法
空

这个算法的特点就是任何操作都可以根据节点的状态判断执行，而不需要用到锁。

其核心接口是Transfer，生产者的put或消费者的take都使用这个接口，根据第一个参数来区别是入列（栈）还是出列（栈）。

```java
/**
     * Shared internal API for dual stacks and queues.
     */
    static abstract class Transferer {
        /**
         * Performs a put or take.
         *
         * @param e if non-null, the item to be handed to a consumer;
         *          if null, requests that transfer return an item
         *          offered by producer.
         * @param timed if this operation should timeout
         * @param nanos the timeout, in nanoseconds
         * @return if non-null, the item provided or received; if null,
         *         the operation failed due to timeout or interrupt --
         *         the caller can distinguish which of these occurred
         *         by checking Thread.interrupted.
         */
        abstract Object transfer(Object e, boolean timed, long nanos);
    }

TransferQueue实现如下(摘自Java 6源代码)，入列和出列都基于Spin和CAS方法：

/**
         * Puts or takes an item.
         */
        Object transfer(Object e, boolean timed, long nanos) {
            /* Basic algorithm is to loop trying to take either of
             * two actions:
             *
             * 1. If queue apparently empty or holding same-mode nodes,
             *    try to add node to queue of waiters, wait to be
             *    fulfilled (or cancelled) and return matching item.
             *
             * 2. If queue apparently contains waiting items, and this
             *    call is of complementary mode, try to fulfill by CAS'ing
             *    item field of waiting node and dequeuing it, and then
             *    returning matching item.
             *
             * In each case, along the way, check for and try to help
             * advance head and tail on behalf of other stalled/slow
             * threads.
             *
             * The loop starts off with a null check guarding against
             * seeing uninitialized head or tail values. This never
             * happens in current SynchronousQueue, but could if
             * callers held non-volatile/final ref to the
             * transferer. The check is here anyway because it places
             * null checks at top of loop, which is usually faster
             * than having them implicitly interspersed.
             */

            QNode s = null; // constructed/reused as needed
            boolean isData = (e != null);

            for (;;) {
                QNode t = tail;
                QNode h = head;
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin

                if (h == t || t.isData == isData) { // empty or same-mode
                    QNode tn = t.next;
                    if (t != tail)                  // inconsistent read
                        continue;
                    if (tn != null) {               // lagging tail
                        advanceTail(t, tn);
                        continue;
                    }
                    if (timed &amp;&amp; nanos &lt;= 0)        // can't wait
                        return null;
                    if (s == null)
                        s = new QNode(e, isData);
                    if (!t.casNext(null, s))        // failed to link in
                        continue;

                    advanceTail(t, s);              // swing tail and wait
                    Object x = awaitFulfill(s, e, timed, nanos);
                    if (x == s) {                   // wait was cancelled
                        clean(t, s);
                        return null;
                    }

                    if (!s.isOffList()) {           // not already unlinked
                        advanceHead(t, s);          // unlink if head
                        if (x != null)              // and forget fields
                            s.item = s;
                        s.waiter = null;
                    }
                    return (x != null)? x : e;

                } else {                            // complementary-mode
                    QNode m = h.next;               // node to fulfill
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read

                    Object x = m.item;
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }

                    advanceHead(h, m);              // successfully fulfilled
                    LockSupport.unpark(m.waiter);
                    return (x != null)? x : e;
                }
            }
        }
```

#### java8实现
待补充。。。


### 参考
<http://ifeve.com/java-synchronousqueue/>
