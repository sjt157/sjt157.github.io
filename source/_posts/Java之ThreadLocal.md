---
title: Java之ThreadLocal
date: 2019-03-11 14:58:14
tags: Java
categories: Java
---


### 什么是ThreadLocal
ThreadLocal是线程执行时的上下文,用于存放线程局部变量。ThreadLocal 类为每一个线程都维
护了自己独有的变量拷贝。每个线程都拥有自己独立的变量，其作用在于数据独立。

* ThreadLocal 采用 hash 表的方式来为每个线程提供一个变量的副本
* ThreadLocal是本地变量，无法跨jvm传递
* ThreadLocalMap 是存放局部变量的容器
* Thread中通过变量ThreadLocal.ThreadLocalMap threadLocals来持有ThreadLocalMap的实例
* ThreadLocal则是ThreadLocalMap的manager，控制着ThreadLocalMap的创建、存取、删除等工作。

#### ThreadLocalMap

ThreadLocalMap和Map接口没有关系，它是使用数组来存储变量的：private Entry[] table,table的初始容量是16，当table的实际长度大于容量时进行成倍扩容，所以table的容量始终是2的幂。

#### Entry

Entry使用ThreadLocal对象作为键，注意不是使用线程(Thread)对象作为键。

WeakReference表示一个对象的弱引用，java将对象的引用按照强弱等级分为四种：

* 强引用："Person p = new Person()"代表一个强引用，只要p存在，GC不会回收Person对象。
* 软引用：SoftReference代表一个软引用，在内存不足将要发生内存溢出时，GC会回收软引用对象。
* 弱引用：WeakReference代表一个弱引用，其生命周期在下次垃圾回收之前，不管内存足不足，都会被GC回收。
* 虚引用：PhantomReference代表一个虚引用，无法通过虚引用获取引用对象的值，也被称为"幽灵引用"，它的意义就是检查对象是否被回收。

关于弱引用的一个小栗子:
```java
import java.lang.ref.WeakReference;
public class WeakReferenceTest {
 public static void main(String[] args) {
 Object o = new Object();
 WeakReference<Object> wof = new WeakReference<Object>(new Object());
 System.out.println(wof.get()==null);//false
 System.out.println(o==null);//false
 System.gc();// 通知系统GC
 System.out.println(wof.get()==null);//true
 System.out.println(o==null);//false
 }
}
```
Entry定义成弱引用的目的是确保没有了ThreadLocal对象的强引用时，能释放ThreadLocalMap中的变量内存。因为ThreadLocalMap是隐藏在内部的，程序员不可见，所以必须要有一个机制能释放ThreadLocalMap对象中的变量内存。
```java
// 定义ThreadLocal
public static final ThreadLocal<Session> sessions = new ThreadLocal<Session>();
在某个时刻：
sessions = null;
说明已不使用sessions了，应该释放ThreadLocalMap中的变量内存。

```

#### 存入
逻辑：

s1:从当前线程中拿出ThreadLocalMap,如果为空进行创建create，不为空进行值存入转入s2
s2:使用ThreadLocal实例作为key和value调用ThreadLocalMap的set方法
s3:使用ThreadLocal实例的threadLocalHashCode与table的容量取模，计算值要放入table中的坐标
s4:使用这个坐标线性向后探测table，如果发现相同的key则更新值，返回。如果发现实现的key(ThreadLocal实例被GC回收,因为它是WeakReference，所以key为空)，转入s5,未发现相同的key和实现的key，转入s6
s5:调用replaceStaleEntry方法清理talbe中key失效的entry，在清理过程中发现相同的key进行值更新，否则新建Entry插入table(坐标为staleSlot)，返回。
s6:新建一个Entry插入talbe(坐标为i)，使用i和自增后的长度sz调用cleanSomeSlots做table的连续段清理，转入s7
s7:清理之后发现table长度大于等于扩容阈值threshold，进行table扩容

源码

```java
public void set(T value) {
 Thread t = Thread.currentThread();
 // getMap(t):t.threadLocals,s1 从当前线程中拿出ThreadLocalMap
 ThreadLocalMap map = getMap(t);
 if (map != null)// map 不为空set
 map.set(this, value); // s2 ThreadLocal的实例(this)作为key
 else // map为空create
 createMap(t, value);
}
private void set(ThreadLocal key, Object value) {
 Entry[] tab = table;
 int len = tab.length;
 int i = key.threadLocalHashCode & (len-1);// s3 hash计算出table坐标
 // 线性探测
 for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
 ThreadLocal k = e.get();
 // 找到对应的entry
 if (k == key) {
 e.value = value;// 更新值
 return;
 }
 // 替换失效的entry 
 if (k == null) {
 replaceStaleEntry(key, value, i);//清理
 return;
 }
 }
 tab[i] = new Entry(key, value);// 插入新值
 int sz = ++size; // 长度增加
 // table连续清理，并判断是否扩容
 if (!cleanSomeSlots(i, sz) && sz >= threshold)
 rehash();// 扩容table，并重新hash
}
```

#### 扩容

逻辑：

s1:进行一次全量的清理(清理对应ThreadLocal已经被GC回收的entry)
s2:因为进行了一次清理，所以talbe的长度会变小，改变扩容的阈值，由原来的2/3改为1/2，如果table长度大于等于阈值，扩容转入s3
s3:新建一个数组，容量是table容量的2倍，数组拷贝,首先使用hash算法生成entry在新数组中的坐标，如果发生碰撞，使用线性探测重新确定坐标

代码：
```java
private void rehash() {
 expungeStaleEntries(); // s1 做一次全量清理
 // s2 size很可能会变小调低阈值来判断是否需要扩容
 if (size >= threshold - threshold / 4)
 resize();// 扩容
}
private void resize() { // s3
 Entry[] oldTab = table;
 int oldLen = oldTab.length;// 原来的容量
 int newLen = oldLen * 2; // 扩容两倍
 Entry[] newTab = new Entry[newLen];// 新数组
 int count = 0;
 for (int j = 0; j < oldLen; ++j) {// 拷贝
 Entry e = oldTab[j];
 if (e != null) {
 ThreadLocal k = e.get();
 if (k == null) { // key失效，被回收
 e.value = null; // 帮助GC
 } else {
 // Hash获取元素的坐标
 int h = k.threadLocalHashCode & (newLen - 1);
 while (newTab[h] != null)
 h = nextIndex(h, newLen); // 线性探测 获取坐标
 newTab[h] = e;
 count++;
 }
 }
 }
 setThreshold(newLen);// 设置新的扩容阈值
 size = count;
 table = newTab;
}
```
#### 魔数

HASH_INCREMENT = 0x61c88647这个数字和斐波那契散列有关(数学问题感兴趣可以深入研究)，通过这个数字可以得到均匀的散列码。

一个小栗子:
```java
public class Hash {
 private static AtomicInteger nextHashCode = new AtomicInteger();
 private static final int HASH_INCREMENT = 0x61c88647;
 private static int nextHashCode() {
 return nextHashCode.getAndAdd(HASH_INCREMENT);
 }
 public static void main(String[] args) {
 int length = 32;
 for(int i=0;i<n;i++) {
 System.out.println(nextHashCode()&(length-1));
 }
 }
}
```
会发现生成的散列码非常均匀，如果把length改为31就会发现得到的散列码不那么均匀了。

length-1的二进制表示就是低位连续的N个1,nextHashCode()&(length-1)的值就是nextHashCode()的低N位, 这样就能均匀的产生均匀的分布,这是为什么ThreadLocalMap中talbe的容量必须为2的幂的原因。

#### 取值

逻辑：

s1:从当前线程中拿出ThreadLocalMap,如果为空进行初始化设置setInitialValue，不为空，使用ThreadLocal实例作为key从ThreadLocalMap中取值，转入s2
s2:使用hash算法生成key对应entry在table中的坐标i,如果table[i]对应的entry不为空且key未失效，说明命中直接返回，否则转入s3
s3:线性探测table,如果发现相同的key，返回。如果发现失效的key，调用expungeStaleEntry清理talbe,探测完毕返回null
源码：
```java
public T get() {
 Thread t = Thread.currentThread();// 当前线程
 ThreadLocalMap map = getMap(t);// 拿出ThreadLocalMap
 if (map != null) { // s1 ThreadLocalMap不为空
 ThreadLocalMap.Entry e = map.getEntry(this);
 if (e != null)
 return (T)e.value;
 }
 return setInitialValue();// ThreadLocalMap为空
}

private Entry getEntry(ThreadLocal key) {
 int i = key.threadLocalHashCode & (table.length - 1); // hash坐标
 Entry e = table[i];
 if (e != null && e.get() == key) // s2 key有效，命中返回
 return e;
 else
 return getEntryAfterMiss(key, i, e); // 线性探测，继续查找
}


private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) { // s3
 Entry[] tab = table;
 int len = tab.length;
 while (e != null) {
 ThreadLocal k = e.get();
 if (k == key) // 找到目标
 return e;
 if (k == null) // entry对应的ThreadLocal已经被回收，清理无效entry
 expungeStaleEntry(i);
 else
 i = nextIndex(i, len); // 往后走
 e = tab[i];
 }
 return null;
}
```
#### 删除

源码：
```java
public void remove() {
 ThreadLocalMap m = getMap(Thread.currentThread());// 从当前线程拿出ThreadLocalMap
 if (m != null)
 m.remove(this);// 删除，key为ThreadLocal实例
}
private void remove(ThreadLocal key) {
 Entry[] tab = table;
 int len = tab.length;
 int i = key.threadLocalHashCode & (len-1);// hash定位
 for (Entry e = tab[i];
 e != null;
 e = tab[i = nextIndex(i, len)]) {
 if (e.get() == key) {
 e.clear();// 断开弱引用
 expungeStaleEntry(i);// 从i开始，进行段清理
 return;
 }
 }
}
```
#### 怎么防止内存泄露

通过上文可以看到ThreadLocal为应对内存泄露做的工作：

* 将Entry定义成弱引用，如果ThreadLocal实例不存在强引用了那么Entry的key就会失效
* get()、set()方法都进行失效key的清理
即便是这样也不能保证万无一失：

* 通常情况下为了使线程可以共用ThreadLocal，会这样定义：static final ThreadLocal threadLocal,这样static变量的生命周期是随class一起的，所以它永远不会被GC回收，这个强引用在key就不会失效。
* 不使用static定义threadLocal，由于有一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value的存在，在长生命周期的线程中(比如线程池)也有内存泄露的风险。短生命周期的线程则无所谓，因为随着线程生命周期的结束，一切都烟消云散。
当然这并不可怕，只要在使用完threadLocal后调用下remove()方法，清除数据，就可以了。

### 秒杀中ThreadLocal的应用
```java
package com.sjt.miaosha.access;

import com.sjt.miaosha.entity.User;

public class UserContext {
    //每一个线程都对应自己的threadlocal
	private static ThreadLocal<User> userHolder = new ThreadLocal<User>();
	
	public static void setUser(User user) {
		userHolder.set(user);
	}
	
	public static User getUser() {
		return userHolder.get();
	}
}

```
实例中用thradlocal保存每个用户上下文。
```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		
		if(handler instanceof HandlerMethod) {
			User user = getuser(request, response);  //得到用户信息
			UserContext.setUser(user);   //放入Usercontext中，里边是Threadlocal
			HandlerMethod hm = (HandlerMethod) handler;
			AccessLimit accessLimit = hm.getMethodAnnotation(AccessLimit.class);
			if(accessLimit == null) {
				return true;
			}
			int seconds = accessLimit.seconds();
			int maxCount = accessLimit.maxCount();
			boolean needLogin = accessLimit.needLogin();
			String key = request.getRequestURI();
			if(needLogin) {
				if(user == null) {
					render(response, CodeMsg.SESSION_ERROR);
					return false;
				}
				key += "_" + user.getId();
			}else {
				// do nothing
			}
			AccessKey ak = AccessKey.withExpire(seconds);
			Integer count = redisService.get(ak, key, Integer.class);
			if(count == null) {
				redisService.set(ak, key, 1);
			}else if(count < maxCount){
				redisService.incr(ak, key);
			}else {
				render(response, CodeMsg.ACCESS_LIMIT_REACHED);
				return false;
			}
		}
		
		return true;
	}

```

### Hibernate中的OpenSessionInView
ThreadLocal的出现可以减少通过参数来传递（使代码更加简洁，降低耦合性），Hibernate中的OpenSessionInView，就始终保证当前线程只有一个在使用中的Connection（或Hibernate Session），代码如下：
```java
public class ConnectionManager {  
  
    /** 线程内共享Connection，ThreadLocal通常是全局的，支持泛型 */  
    private static ThreadLocal<Connection> threadLocal = new ThreadLocal<Connection>();  
      
    public static Connection getCurrConnection() {  
        // 获取当前线程内共享的Connection  
        Connection conn = threadLocal.get();  
        try {  
            // 判断连接是否可用  
            if(conn == null || conn.isClosed()) {  
                // 创建新的Connection赋值给conn(略)  
                // 保存Connection  
                threadLocal.set(conn);  
            }  
        } catch (SQLException e) {  
            // 异常处理  
        }  
        return conn;  
    }  
      
    /** 
     * 关闭当前数据库连接 
     */  
    public static void close() {  
        // 获取当前线程内共享的Connection  
        Connection conn = threadLocal.get();  
        try {  
            // 判断是否已经关闭  
            if(conn != null && !conn.isClosed()) {  
                // 关闭资源  
                conn.close();  
                // 移除Connection  
                threadLocal.remove();  
                conn = null;  
            }  
        } catch (SQLException e) {  
            // 异常处理  
        }  
    }  
}
```
### ThreadLocal和同步机制synchonzied相比

1. synchonzied同步机制是为了实现同步多线程对相同资源的并发访问控制。同步的主要目的是保证多线程间的数据共享。同步会带来巨大的性能开销，所以同步操作应该是细粒度的（对象中的不同元素使用不同的锁，而不是整个对象一个锁）。如果同步使用得当，带来的性能开销是微不足道的。使用同步真正的风险是复杂性和可能破坏资源安全,而不是性能。 

2. ThreadLocal以空间换取时间，提供了一种非常简便的多线程实现方式。因为多个线程并发访问无需进行等待，所以使用ThreadLocal会获得更大的性能。

3. ThreadLocal中的对象，通常都是比较小的对象。另外使用ThreadLocal不能使用原子类型，只能使用Object类型。ThreadLocal的使用比synchronized要简单得多。 

4. synchronized是利用锁的机制，使变量或代码块在某一时该只能被一个线程访问。而ThreadLocal为每一个线程都提供了变量的副本，使得每个线程在某一时间访问到的并不是同一个对象，这样就隔离了多个线程对数据的数据共享。而Synchronized却正好相反，它用于在多个线程间通信时能够获得数据共享。 

5. Synchronized用于线程间的数据共享，而ThreadLocal则用于线程间的数据隔离。 


### 在线程池中使用ThreadLocal需要注意
* 线程池由于未创建新的线程，导致线程变量也是之前的内容。

解决办法:
一、直接在进入线程时remove操作。

二、使用后清空。
<https://blog.csdn.net/qq_33522040/article/details/85322094>

* ThreadLocal可以为当前线程保存局部变量，而InheritableThreadLocal则可以在创建子线程的时候将父线程的局部变量传递到子线程中。

如果使用了线程池(如Executor)，那么即使即使父线程已经结束，子线程依然存在并被池化。这样，线程池中的线程在下一次请求被执行的时候，ThreadLocal对象的get()方法返回的将不是当前线程中设定的变量，因为池中的“子线程”根本不是当前线程创建的，当前线程设定的ThreadLocal变量也就无法传递给线程池中的线程。
因此，必须将外部线程中的ThreadLocal变量显式地传递给线程池中的线程。
<https://blog.csdn.net/comliu/article/details/3186778?utm_source=blogxgwz2>

### Netty的FastThreadLocal
参考我的博客

### 并发包中ThreadLocalRandom类
Random在多线程下存在竞争种子原子变量更新操作失败后自旋等待的缺点，从而引出ThreadLocalRandom类，ThreadLocalRandom使用ThreadLocal的原理，让每个线程内持有一个本地的种子变量，该种子变量只有在使用随机数时候才会被初始化，多线程下计算新种子时候是根据自己线程内维护的种子变量进行更新，从而避免了竞争。
<https://blog.csdn.net/wangyunpeng0319/article/details/78903541>

### 小结

1. ThreadLocal是线程执行时的上下文,用于存放线程局部变量。它不能解决并发情况下数据共享的问题

2. ThreadLocal是以ThreadLocal对象本身作为key的，不是线程(Thread)对象

3. ThreadLocal存在内存泄露的风险，要养成用完即删的习惯

4. ThreadLocal使用散列定位数据存储坐标，如果发生碰撞，使用线性探测重新定位，这在高并发场景下会影响一点性能。改善方法如netty的FastThreadLocal，使用固定坐标，以空间换时间，后面会分析FastThreadLocal实现。


### 参考
<https://www.cnblogs.com/handsomeye/p/5390720.html>

