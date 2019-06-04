---
title: Netty之高水位、低水位
date: 2019-01-19 14:55:00
tags: Netty
categories: Netty
---



我们一听高低水位，肯定首先想到的肯定就是“大爸”（三峡大坝），我们都知道，三峡的水位曾经到达过172.85米，最高限制水位175米，其实这就是三峡的高水位，如果再进水，那么恐怕啥都不好用了。
同理，Netty中有缓冲区，就相当于大坝起存储缓冲作用。当缓冲区达到一定大小时则不能写入，避免被撑爆。
Netty中提供 了writeBufferLowWaterMark和writeBufferHighWaterMark选项用来控制高低水位。可以通过监控当前写缓冲区的水位状况，来避免占用大量的内存，因为ChannelOutboundBuffer本身是无界的，所以用的时候要注意。
（感觉跟Netty提供的Traffic Shaping流量整形功能有点像）。
我们来看一下源码：

```
/**
 * WriteBufferWaterMark is used to set low water mark and high water mark for the write buffer.
 * <p>
 * If the number of bytes queued in the write buffer exceeds the
 * {@linkplain #high high water mark}, {@link Channel#isWritable()}
 * will start to return {@code false}.
 * <p>
 * If the number of bytes queued in the write buffer exceeds the
 * {@linkplain #high high water mark} and then
 * dropped down below the {@linkplain #low low water mark},
 * {@link Channel#isWritable()} will start to return
 * {@code true} again.
 */
public final class WriteBufferWaterMark {

    private static final int DEFAULT_LOW_WATER_MARK = 32 * 1024;
    private static final int DEFAULT_HIGH_WATER_MARK = 64 * 1024;

    public static final WriteBufferWaterMark DEFAULT =
            new WriteBufferWaterMark(DEFAULT_LOW_WATER_MARK, DEFAULT_HIGH_WATER_MARK, false);

    private final int low;
    private final int high;

    /**
     * Create a new instance.
     *
     * @param low low water mark for write buffer.
     * @param high high water mark for write buffer
     */
    public WriteBufferWaterMark(int low, int high) {
        this(low, high, true);
    }

    /**
     * This constructor is needed to keep backward-compatibility.
     */
    WriteBufferWaterMark(int low, int high, boolean validate) {
        if (validate) {
            if (low < 0) {
                throw new IllegalArgumentException("write buffer's low water mark must be >= 0");
            }
            if (high < low) {
                throw new IllegalArgumentException(
                        "write buffer's high water mark cannot be less than " +
                                " low water mark (" + low + "): " +
                                high);
            }
        }
        this.low = low;
        this.high = high;
    }

    /**
     * Returns the low water mark for the write buffer.
     */
    public int low() {
        return low;
    }

    /**
     * Returns the high water mark for the write buffer.
     */
    public int high() {
        return high;
    }

    @Override
    public String toString() {
        StringBuilder builder = new StringBuilder(55)
            .append("WriteBufferWaterMark(low: ")
            .append(low)
            .append(", high: ")
            .append(high)
            .append(")");
        return builder.toString();
    }

}


```
从注释里头可以看到控制的是写缓冲，高低水位这两个参数控制的是Channel.isWritable()方法，当超过高水位时返回False，降到低水位之下后，又重新可写。

```

private static final AtomicIntegerFieldUpdater<ChannelOutboundBuffer> UNWRITABLE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "unwritable");

    private volatile int unwritable;

    /**
     * Returns {@code true} if and only if {@linkplain #totalPendingWriteBytes() the total number of pending bytes} did
     * not exceed the write watermark of the {@link Channel} and
     * no {@linkplain #setUserDefinedWritability(int, boolean) user-defined writability flag} has been set to
     * {@code false}.
     */
    public boolean isWritable() {
        return unwritable == 0;
    }

    /**
     * Get how many bytes must be drained from the underlying buffer until {@link #isWritable()} returns {@code true}.
     * This quantity will always be non-negative. If {@link #isWritable()} is {@code true} then 0.
     */
    public long bytesBeforeWritable() {
        long bytes = totalPendingSize - channel.config().getWriteBufferLowWaterMark();
        // If bytes is negative we know we are writable, but if bytes is non-negative we have to check writability.
        // Note that totalPendingSize and isWritable() use different volatile variables that are not synchronized
        // together. totalPendingSize will be updated before isWritable().
        if (bytes > 0) {
            return isWritable() ? 0 : bytes;
        }
        return 0;
    }

    /**
     * Decrement the pending bytes which will be written at some point.
     * This method is thread-safe!
     */
    void decrementPendingOutboundBytes(long size) {
        decrementPendingOutboundBytes(size, true, true);
    }

    private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
        if (size == 0) {
            return;
        }

        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
        if (notifyWritability && newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()) {
            setWritable(invokeLater);
        }
    }

    private void setWritable(boolean invokeLater) {
        for (;;) {
            final int oldValue = unwritable;
            final int newValue = oldValue & ~1;
            if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
                if (oldValue != 0 && newValue == 0) {
                    fireChannelWritabilityChanged(invokeLater);
                }
                break;
            }
        }
    }

    private void fireChannelWritabilityChanged(boolean invokeLater) {
        final ChannelPipeline pipeline = channel.pipeline();
        if (invokeLater) {
            Runnable task = fireChannelWritabilityChangedTask;
            if (task == null) {
                fireChannelWritabilityChangedTask = task = new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireChannelWritabilityChanged();
                    }
                };
            }
            channel.eventLoop().execute(task);
        } else {
            pipeline.fireChannelWritabilityChanged();
        }
    }

```
bytesBeforeWritable方法先判断totalPendingSize是否大于lowWatermark，如果不大于则返回0，如果大于且isWritable返回true则返回0，否则返回差值
decrementPendingOutboundBytes方法会判断，如果notifyWritability为true且newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()，则调用setWritablesetWritable(invokeLater)
setWritable会更新unwritable，如果是从非0变为0，还会触发fireChannelWritabilityChanged进行通知
```
ChannelOutboundBuffer.setUnwritable
netty-all-4.1.25.Final-sources.jar!/io/netty/channel/ChannelOutboundBuffer.java
    private static final AtomicIntegerFieldUpdater<ChannelOutboundBuffer> UNWRITABLE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "unwritable");

    private volatile int unwritable;
    
    /**
     * Returns {@code true} if and only if {@linkplain #totalPendingWriteBytes() the total number of pending bytes} did
     * not exceed the write watermark of the {@link Channel} and
     * no {@linkplain #setUserDefinedWritability(int, boolean) user-defined writability flag} has been set to
     * {@code false}.
     */
    public boolean isWritable() {
        return unwritable == 0;
    }

    /**
     * Get how many bytes can be written until {@link #isWritable()} returns {@code false}.
     * This quantity will always be non-negative. If {@link #isWritable()} is {@code false} then 0.
     */
    public long bytesBeforeUnwritable() {
        long bytes = channel.config().getWriteBufferHighWaterMark() - totalPendingSize;
        // If bytes is negative we know we are not writable, but if bytes is non-negative we have to check writability.
        // Note that totalPendingSize and isWritable() use different volatile variables that are not synchronized
        // together. totalPendingSize will be updated before isWritable().
        if (bytes > 0) {
            return isWritable() ? bytes : 0;
        }
        return 0;
    }

    /**
     * Increment the pending bytes which will be written at some point.
     * This method is thread-safe!
     */
    void incrementPendingOutboundBytes(long size) {
        incrementPendingOutboundBytes(size, true);
    }

    private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
        if (size == 0) {
            return;
        }

        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
        if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) {
            setUnwritable(invokeLater);
        }
    }

    private void setUnwritable(boolean invokeLater) {
        for (;;) {
            final int oldValue = unwritable;
            final int newValue = oldValue | 1;
            if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
                if (oldValue == 0 && newValue != 0) {
                    fireChannelWritabilityChanged(invokeLater);
                }
                break;
            }
        }
    }

    private void fireChannelWritabilityChanged(boolean invokeLater) {
        final ChannelPipeline pipeline = channel.pipeline();
        if (invokeLater) {
            Runnable task = fireChannelWritabilityChangedTask;
            if (task == null) {
                fireChannelWritabilityChangedTask = task = new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireChannelWritabilityChanged();
                    }
                };
            }
            channel.eventLoop().execute(task);
        } else {
            pipeline.fireChannelWritabilityChanged();
        }
    }

```
bytesBeforeUnwritable方法先判断highWatermark与totalPendingSize的差值，totalPendingSize大于等于highWatermark，则返回0；如果小于highWatermark，且isWritable为true，则返回差值，否则返回0
incrementPendingOutboundBytes方法判断如果newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()，则调用setUnwritable(invokeLater)
setUnwritable会更新unwritable，如果是从0变为非0，还会触发fireChannelWritabilityChanged进行通知

综上，lowWatermark及highWatermark分别在decrementPendingOutboundBytes及incrementPendingOutboundBytes方法里头用到(目前应该是这两个方法起作用)，当小于lowWatermark或者大于highWatermark的时候，分别触发setWritable及setUnwritable，更改ChannelOutboundBuffer的unwritable字段，进而影响isWritable方法；在isWritable为true的时候会立马执行写请求，当返回false的时候，写请求会被放入队列等待isWritable为true时才能执行这些堆积的写请求。



### 实践出真知 
在Netty的物联网网关中，就可以通过new WriteBufferWaterMark(32 * 1024 * 1024, 64 * 1024 * 1024)来设置水位线，防止服务器处理能力极其低下但连接正常时，造成channel中缓存大量数据影响网关性能。具体设置多大我们可以根据我们的应用需要支持多少连接数和系统资源进行合理规划。高水位线和低水位线是字节数，默认高水位是64K，低水位是
32K。

### 参考

<https://www.jianshu.com/p/a1166c34ae46>
<https://gitee.com/willbeahero/IOTGate>
<https://www.jianshu.com/p/890525ff73cb>


