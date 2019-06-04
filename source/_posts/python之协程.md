---
title: python之协程
date: 2018-04-07 10:36:22
tags: Python
categories: Python
---

* Generator 其中一个特性就是不是一次性生成数据，而是生成一个可迭代的对象，在迭代时，根据我们所写的逻辑来控制其启动时机。
* Generator 另一个很大作用可以说就是当做协程使用。协程就是你可以暂停执行的函数。简而言之，协程是比线程更为轻量的一种模型，我们可以自行控制启动与停止的时机。知乎上说的，最易懂的解释：*你行你就上，不行旁边等着让别人上，啥时候行了你再上。*在 Python 中其实没有专门针对协程的这个概念，社区一般而言直接将 Generator 作为一种特殊的协程看待，想想，我们可以用 next 或 __next__() 方法或者是 send() 方法唤醒我们的 Generator ，在运行完我们所规定的代码后， Generator 返回并将其所有状态冻结。这是不是很让我们 Excited 呢！！
* 总而言之，协程比线程更节省资源，效率更高，并且更安全。如果使用线程做过重要的编程，你就知道写出程序有多么困难，因为调度程序任何时候都能中断线程。必须记住保留锁，去保护程序中的重要部分，防止多步操作在执行的过程中中断，防止数据处于无效状态。而协程默认会做好全方位保护，以防止中断。我们必须显式产出才能让程序的余下部分运行。对协程来说，无需保留锁，在多个线程之间同步操作，协程自身就会同步，因为在任意时刻只有一个协程运行。想交出控制权时，可以使用 yield 或 yield from 把控制权交还调度程序。这就是能够安全地取消协程的原因：按照定义，协程只能在暂停的 yield处取消，因此可以处理 CancelledError 异常，执行清理操作。
* 这是一个异步编程的例子，将代码与事件循环及其相关的函数一一对应起来。这个例子里包含的几个协程，代表着火箭发射的倒计时，并且看起来是同时开始的。这是通过并发实现的异步编程；3个不同的协程将分别独立运行，并且都在同一个线程内完成。
```python
import datetime
import heapq
import types
import time


class Task:
    """Represent how long a coroutine should before starting again.

    Comparison operators are implemented for use by heapq. Two-item
    tuples unfortunately don't work because when the datetime.datetime
    instances are equal, comparison falls to the coroutine and they don't
    implement comparison methods, triggering an exception.

    Think of this as being like asyncio.Task/curio.Task.
    """

    def __init__(self, wait_until, coro):
        self.coro = coro
        self.waiting_until = wait_until

    def __eq__(self, other):
        return self.waiting_until == other.waiting_until

    def __lt__(self, other):
        return self.waiting_until < other.waiting_until


class SleepingLoop:
    """An event loop focused on delaying execution of coroutines.

    Think of this as being like asyncio.BaseEventLoop/curio.Kernel.
    """

    def __init__(self, *coros):
        self._new = coros
        self._waiting = []

    def run_until_complete(self):
        # Start all the coroutines.
        for coro in self._new:
            wait_for = coro.send(None)
            heapq.heappush(self._waiting, Task(wait_for, coro))
        # Keep running until there is no more work to do.
        while self._waiting:
            now = datetime.datetime.now()
            # Get the coroutine with the soonest resumption time.
            task = heapq.heappop(self._waiting)
            if now < task.waiting_until:
                # We're ahead of schedule; wait until it's time to resume.
                delta = task.waiting_until - now
                time.sleep(delta.total_seconds())
                now = datetime.datetime.now()
            try:
                # It's time to resume the coroutine.
                wait_until = task.coro.send(now)
                heapq.heappush(self._waiting, Task(wait_until, task.coro))
            except StopIteration:
                # The coroutine is done.
                pass


@types.coroutine
def sleep(seconds):
    """Pause a coroutine for the specified number of seconds.

    Think of this as being like asyncio.sleep()/curio.sleep().
    """
    now = datetime.datetime.now()
    wait_until = now + datetime.timedelta(seconds=seconds)
    # Make all coroutines on the call stack pause; the need to use `yield`
    # necessitates this be generator-based and not an async-based coroutine.
    actual = yield wait_until
    # Resume the execution stack, sending back how long we actually waited.
    return actual - now


async def countdown(label, length, *, delay=0):
    """Countdown a launch for `length` seconds, waiting `delay` seconds.

    This is what a user would typically write.
    """
    print(label, 'waiting', delay, 'seconds before starting countdown')
    delta = await sleep(delay)
    print(label, 'starting after waiting', delta)
    while length:
        print(label, 'T-minus', length)
        waited = await sleep(1)
        length -= 1
    print(label, 'lift-off!')


def main():
    """Start the event loop, counting down 3 separate launches.

    This is what a user would typically write.
    """
    loop = SleepingLoop(countdown('A', 5), countdown('B', 3, delay=2),
                        countdown('C', 4, delay=1))
    start = datetime.datetime.now()
    loop.run_until_complete()
    print('Total elapsed time is', datetime.datetime.now() - start)


if __name__ == '__main__':
    main()
# A waiting 0 seconds before starting countdown
#B waiting 2 seconds before starting countdown
#C waiting 1 seconds before starting countdown
#A starting after waiting 0:00:00.001000
#A T-minus 5
#C starting after waiting 0:00:01.000058
#C T-minus 4
#A T-minus 4
#B starting after waiting 0:00:02.000115
#B T-minus 3
#C T-minus 3
#A T-minus 3
#B T-minus 2
#C T-minus 2
#A T-minus 2
#B T-minus 1
#C T-minus 1
#A T-minus 1
#B lift-off!
#C lift-off!
#A lift-off!
#Total elapsed time is 0:00:05.001286
```
* 但是基于async的协程和基于生成器的协程会在对应的暂停表达式上面有所不同？主要原因是出于最优化Python性能的考虑，确保你不会将刚好有同样API的不同对象混为一谈。由于生成器默认实现协程的API，因此很有可能在你希望用协程的时候错用了一个生成器。而由于并不是所有的生成器都可以用在基于协程的控制流中，你需要避免错误地使用生成器。但是由于 Python 并不是静态编译的，它最好也只能在用基于生成器定义的协程时提供运行时检查。这意味着当用types.coroutine时，Python 的编译器将无法判断这个生成器是用作协程还是仅仅是普通的生成器（记住，仅仅因为types.coroutine这一语法的字面意思，并不意味着在此之前没有人做过types = spam的操作），因此编译器只能基于当前的情况生成有着不同限制的操作码。关于基于生成器的协程和async定义的协程之间的差异，我想说明的关键点是只有基于生成器的协程可以真正的暂停执行并强制性返回给事件循环。你可能不了解这些重要的细节，因为通常你调用的像是asyncio.sleep() function 这种事件循环相关的函数，由于事件循环实现他们自己的API，而这些函数会处理这些小的细节。对于我们绝大多数人来说，我们只会跟事件循环打交道，而不需要处理这些细节，因此可以只用async定义的协程。但是如果你和我一样好奇为什么不能在async定义的协程中使用asyncio.sleep()，那么这里的解释应该可以让你顿悟。
* Generator迭代的就是通过内建的next（）或__next__()方法调用内建的send（）方法。
* 与其它特性一起，PEP 342 为生成器引入了 send() 方法。这让我们不仅可以暂停生成器，而且能够传递值到生成器暂停的地方。还是以我们的 range() 为例，你可以让序列向前或向后跳过几个值。
```python
def jumping_range(up_to):
    """Generator for the sequence of integers from 0 to up_to, exclusive.

    Sending a value into the generator will shift the sequence by that amount.
    """
    index = 0
    while index < up_to:
        jump = yield index
        if jump is None:
            jump = 1
        index += jump


if __name__ == '__main__':
    iterator = jumping_range(5)
    print(next(iterator))  # 0
    print(iterator.send(2))  # 2
    print(next(iterator))  # 3
    print(iterator.send(-1))  # 2
    for x in iterator:
        print(x)  # 3, 4
```

* openstack中就是用了协程模型，利用Python库Eventlet可以产生很多协程，这些协程之间只有在调用到了某些特殊的Eventlet库函数的时候（比如睡眠sleep、IO调用等）才会发生切换。协程的实现主要是在协程休息时把当前的寄存器保存起来，然后重新工作时将其恢复，可以简单的理解为，在单个线程内部有多个栈去保存切换时的线程上下文，因此，协程可以理解为一个线程内的伪并发方式。但是由于Eventlet本身的一些局限性，目前openstack考虑用AsynclIO来代替他。

