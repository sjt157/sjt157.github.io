---
title: python多线程
date: 2018-03-29 18:35:48
tags: Python
categories: Python
---

### python多线程（假多线程）
1. 如果你的代码是IO密集型，多线程可以明显提高效率。例如制作爬虫（我就不明白为什么Python总和爬虫联系在一起…不过也只想起来这个例子…），绝大多数时间爬虫是在等待socket返回数据。这个时候C代码里是有release GIL的，最终结果是某个线程等待IO的时候其他线程可以继续执行。如果你的代码是CPU密集型，多个线程的代码很有可能是线性执行的。所以这种情况下多线程是鸡肋，效率可能还不如单线程因为有context switch
2. 一般网络传输类应用都是IO密集型，读写硬盘操作多也算是IO密集型
3. 我们都知道，比方我有一个4核的CPU，那么这样一来，在单位时间内每个核只能跑一个线程，然后时间片轮转切换。但是Python不一样，它不管你有几个核，单位时间多个核只能跑一个线程，然后时间片轮转。看起来很不可思议？但是这就是GIL搞的鬼。任何Python线程执行前，必须先获得GIL锁，然后，每执行100条字节码，解释器就自动释放GIL锁，让别的线程有机会执行。这个GIL全局锁实际上把所有线程的执行代码都给上了锁，所以，多线程在Python中只能交替执行，即使100个线程跑在100核CPU上，也只能用到1个核。通常我们用的解释器是官方实现的CPython，要真正利用多核，除非重写一个不带GIL的解释器。（Java中的多线程是可以利用多核的，这是真正的多线程！）
4. 在Python中，可以使用多线程，但不要指望能有效利用多核。如果一定要通过多线程利用多核，那只能通过C扩展来实现，不过这样就失去了Python简单易用的特点。不过，也不用过于担心，Python虽然不能利用多线程实现多核任务，但可以通过多进程实现多核任务。多个Python进程有各自独立的GIL锁，互不影响。

```python
 # 直接调用函数的多线程
from threading import Thread
import time


def loop(name, seconds):
    print('start loop', name, 'at:', time.ctime())
    time.sleep(1)
    print('end loop', name, 'at:', time.ctime())

if __name__ == '__main__':
    loops = [2, 4]
    nloops = range(len(loops))
    threads = []
    print('start at:', time.ctime())
    for i in nloops:
        t = Thread(target=loop, args=(i, loops[i],))
        threads.append(t)
    for i in nloops:
        threads[i].start()
    for i in nloops:
        threads[i].join()
    print('all done at:', time.ctime())



 # 使用可调用的类对象
from threading import Thread
import time


def loop(name, seconds):
    print('start loop', name, 'at:', time.ctime())
    time.sleep(1)
    print('end loop', name, 'at:', time.ctime())

class ThreadFunc(object):
    def __init__(self, func, args, name=''):
        self.name = name
        self.func = func
        self.args = args
    def __call__(self):  # 需要定义一个特殊的方法__call__
        self.func(*self.args)
if __name__ == '__main__':
    loops = [2, 4]
    nloops = range(len(loops))
    threads = []
    print('start at:', time.ctime())
    for i in nloops:
        t = Thread(target=ThreadFunc(loop, (i, loops[i]), loop.__name__))
        threads.append(t)
    for i in nloops:
        threads[i].start()
    for i in nloops:
        threads[i].join()
    print('all done at:', time.ctime())


# 继承Thread类，也就是子类化
from threading import Thread
import time


def loop(name, seconds):
    print('start loop', name, 'at:', time.ctime())
    time.sleep(1)
    print('end loop', name, 'at:', time.ctime())

class ThreadFunc(Thread):
    def __init__(self, func, args, name=''):
        super(ThreadFunc, self).__init__()
        self.name = name
        self.func = func
        self.args = args
    # 将上种方式中可调用的方法改为run方法，其实就是对Thread类中run方法的重写
    def run(self):
        self.func(*self.args)
if __name__ == '__main__':
    loops = [2, 4]
    nloops = range(len(loops))
    threads = []
    print('start at:', time.ctime())
    for i in nloops:
        t = ThreadFunc(loop, (i, loops[i]), loop.__name__)
        threads.append(t)
    for i in nloops:
        threads[i].start()
    for i in nloops:
        threads[i].join()
    print('all done at:', time.ctime())

 # 在一般推荐的方法中，我们用最后一种方式
```
* 线程同步
如果多个线程共同对某个数据修改，则可能出现不可预料的结果，为了保证数据的正确性，需要对多个线程进行同步。
使用Thread对象的Lock和Rlock可以实现简单的线程同步，这两个对象都有acquire方法和release方法，对于那些需要每次只允许一个线程操作的数据，可以将其操作放到acquire和release方法之间。如下：多线程的优势在于可以同时运行多个任务（至少感觉起来是这样）。但是当线程需要共享数据时，可能存在数据不同步的问题。
考虑这样一种情况：一个列表里所有元素都是0，线程"set"从后向前把所有元素改成1，而线程"print"负责从前往后读取列表并打印。那么，可能线程"set"开始改的时候，线程"print"便来打印列表了，输出就成了一半0一半1，这就是数据的不同步。为了避免这种情况，引入了锁的概念。锁有两种状态——锁定和未锁定。每当一个线程比如"set"要访问共享数据时，必须先获得锁定；如果已经有别的线程比如"print"获得锁定了，那么就让线程"set"暂停，也就是同步阻塞；等到线程"print"访问完毕，释放锁以后，再让线程"set"继续。经过这样的处理，打印列表时要么全部输出0，要么全部输出1，不会再出现一半0一半1的尴尬场面。

```python
import threading
import time


class myThread(threading.Thread):
    def __init__(self, threadID, name, counter):
        threading.Thread.__init__(self)
        self.threadID = threadID
        self.name = name
        self.counter = counter

    def run(self):
        print("Starting " + self.name)
        # 获得锁，成功获得锁定后返回True
        # 可选的timeout参数不填时将一直阻塞直到获得锁定
        # 否则超时后将返回False
        threadLock.acquire()
        print_time(self.name, self.counter, 3)
        # 释放锁
        threadLock.release()


def print_time(threadName, delay, counter):
    while counter:
        time.sleep(delay)
        print("%s: %s" % (threadName, time.ctime(time.time())))
        counter -= 1


threadLock = threading.Lock()
threads = []

# 创建新线程
thread1 = myThread(1, "Thread-1", 1)
thread2 = myThread(2, "Thread-2", 2)

# 开启新线程
thread1.start()
thread2.start()

# 添加线程到线程列表
threads.append(thread1)
threads.append(thread2)

# 等待所有线程完成
for t in threads:
    t.join()
print("Exiting Main Thread")

```


