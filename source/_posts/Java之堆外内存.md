---
title: Java之堆外内存
date: 2019-03-23 13:51:21
tags: Java
categories: Java
---
### 堆外内存

Netty的ByteBuffer采用DIRECT BUFFERS，使用堆外直接内存进行Socket读写，不需要进行字节缓冲区的二次拷贝。如果大家自己动手写过NIO和AIO的程序，就会知道我们接触的网络传输数据是直接和ByteBuffer打交道的。在JAVA的API中，一个ByteBuffer读完数据后要flip一下，将当前操作位置设置为0.<Netty In Action>里详细介绍了Netty的ByteBuf怎样将这个缓冲区分成一段一段的，还可以压缩，将读的数据滑向一侧。而堆外内存的零拷贝，如果有JVM基础也很好理解。Java内存模型里提到每个线程都有自己的高速工作内存空间，而不是直接访问主内存。想不用工作内存，直接主内存可见，就要用volatile关键字修饰。所以堆内内存，走JVM就存在这个拷贝开销。
<https://www.cnblogs.com/xiexj/p/6874654.html>


有个小问题：堆内内存包括永久代码？

和堆内内存相对应，堆外内存就是把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机），这样做的结果就是能够在一定程度上减少垃圾回收对应用程序造成的影响。

作为JAVA开发者我们经常用java.nio.DirectByteBuffer对象进行堆外内存的管理和使用，它会在对象创建的时候就分配堆外内存。

DirectByteBuffer类是在Java Heap外分配内存，对堆外内存的申请主要是通过成员变量unsafe来操作，下面介绍构造方法

```java
DirectByteBuffer(int cap) {                 
 
        super(-1, 0, cap, cap);
        //内存是否按页分配对齐
        boolean pa = VM.isDirectMemoryPageAligned();
        //获取每页内存大小
        int ps = Bits.pageSize();
        //分配内存的大小，如果是按页对齐方式，需要再加一页内存的容量
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        //用Bits类保存总分配内存(按页分配)的大小和实际内存的大小
        Bits.reserveMemory(size, cap);
 
        long base = 0;
        try {
           //在堆外内存的基地址，指定内存大小
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        //计算堆外内存的基地址
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }

```
在Cleaner 内部中通过一个列表，维护了一个针对每一个 directBuffer 的一个回收堆外内存的 线程对象(Runnable)，回收操作是发生在 Cleaner 的 clean() 方法中。

```java
private static class Deallocator implements Runnable  {
    private static Unsafe unsafe = Unsafe.getUnsafe();
    private long address;
    private long size;
    private int capacity;
    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }
 
    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}

```
### 使用堆外内存的优点

1. 减少了垃圾回收
因为垃圾回收会暂停其他的工作。

2. 加快了复制的速度
堆内在flush到远程时，会先复制到直接内存（非堆内存），然后在发送；而堆外内存相当于省略掉了这个工作。

同样任何一个事物使用起来有优点就会有缺点，堆外内存的缺点就是内存难以控制，使用了堆外内存就间接失去了JVM管理内存的可行性，改由自己来管理，当发生内存溢出时排查起来非常困难。

### 使用DirectByteBuffer的注意事项

* java.nio.DirectByteBuffer对象在创建过程中会先通过Unsafe接口直接通过os::malloc来分配内存，然后将内存的起始地址和大小存到java.nio.DirectByteBuffer对象里，这样就可以直接操作这些内存。这些内存只有在DirectByteBuffer回收掉之后才有机会被回收，因此如果这些对象大部分都移到了old，但是一直没有触发CMS GC或者Full GC，那么悲剧将会发生，因为你的物理内存被他们耗尽了，因此为了避免这种悲剧的发生，通过-XX:MaxDirectMemorySize来指定最大的堆外内存大小，当使用达到了阈值的时候将调用System.gc来做一次full gc，以此来回收掉没有被使用的堆外内存。

* 使用 HeapByteBuffer 还需要经过一次 DirectByteBuffer 的拷贝，在追求极致性能的场景下是可以通过直接复用堆外内存来避免的。

* 多线程下使用 HeapByteBuffer 进行文件读写，要注意 ThreadLocal<Util.BufferCache> bufferCache 导致的堆外内存膨胀的问题。

### 监控堆外内存
在Java VisualVM里安装插件，Buffer Pools 插件可以监控堆外内存（包含 DirectByteBuffer 和 MappedByteBuffer）

### 堆外内存如何回收？
* 通过System.gc
```
public class WriteByDirectByteBufferTest {
    public static void main(String[] args) throws IOException, InterruptedException {
        ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);
        System.in.read();
        buffer = null;
        System.gc(); //GC 时会触发堆外空闲内存的回收。
        new CountDownLatch(1).await();
    }
}

```
* 手动回收可以立刻释放堆外内存，不需要等待到 GC 的发生。
```java
public class WriteByDirectByteBufferTest {
    public static void main(String[] args) throws IOException, InterruptedException {
        ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);
        System.in.read();
        ((DirectBuffer) buffer).cleaner().clean();
        new CountDownLatch(1).await();
    }
}

```

### 开源堆外缓存框架

关于堆外缓存的开源实现。查询了一些资料后了解到的主要有：

* Ehcache 3.0：3.0基于其商业公司一个非开源的堆外组件的实现。
* Chronical Map：OpenHFT包括很多类库，使用这些类库很少产生垃圾，并且应用程序使用这些类库后也很少发生Minor GC。类库主要包括：Chronicle Map，Chronicle Queue等等。
* OHC：来源于Cassandra 3.0， Apache v2。
* Ignite: 一个规模宏大的内存计算框架，属于Apache项目。


### MappedBytebuffer

MappedByteBuffer 映射出一片文件内容之后，不会全部加载到内存中，而是会进行一部分的预读（体现在占用的那 100M 上），MappedByteBuffer 不是文件读写的银弹，它仍然依赖于 PageCache 异步刷盘的机制。通过 Java VisualVM 可以监控到 mmap 总映射的大小，但并不是实际占用的内存量。

### 如何回收MappedBytebuffer
```java
public class MmapUtil {
    public static void clean(MappedByteBuffer mappedByteBuffer) {
        ByteBuffer buffer = mappedByteBuffer;
        if (buffer == null || !buffer.isDirect() || buffer.capacity() == 0)
            return;
        invoke(invoke(viewed(buffer), "cleaner"), "clean");
    }

    private static Object invoke(final Object target, final String methodName, final Class<?>... args) {
        return AccessController.doPrivileged(new PrivilegedAction<Object>() {
            public Object run() {
                try {
                    Method method = method(target, methodName, args);
                    method.setAccessible(true);
                    return method.invoke(target);
                } catch (Exception e) {
                    throw new IllegalStateException(e);
                }
            }
        });
    }

    private static Method method(Object target, String methodName, Class<?>[] args)
            throws NoSuchMethodException {
        try {
            return target.getClass().getMethod(methodName, args);
        } catch (NoSuchMethodException e) {
            return target.getClass().getDeclaredMethod(methodName, args);
        }
    }

    private static ByteBuffer viewed(ByteBuffer buffer) {
        String methodName = "viewedBuffer";
        Method[] methods = buffer.getClass().getMethods();
        for (int i = 0; i < methods.length; i++) {
            if (methods[i].getName().equals("attachment")) {
                methodName = "attachment";
                break;
            }
        }
        ByteBuffer viewedBuffer = (ByteBuffer) invoke(buffer, methodName);
        if (viewedBuffer == null)
            return buffer;
        else
            return viewed(viewedBuffer);
    }
}

```
测试类:通过一顿复杂的反射操作，成功地手动回收了 Mmap 的内存映射。
```java
public class WriteByMappedByteBufferTest {
    public static void main(String[] args) throws IOException, InterruptedException {
        File data = new File("/tmp/data.txt");
        data.createNewFile();
        FileChannel fileChannel = new RandomAccessFile(data, "rw").getChannel();
        MappedByteBuffer map = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, 1024L * 1024 * 1024);
        System.in.read();
        MmapUtil.clean(map);
        new CountDownLatch(1).await();
    }
}

```
### 参考
<https://blog.csdn.net/ZYC88888/article/details/80228531>
<https://juejin.im/post/5c8de9f5e51d453651442c6a>

