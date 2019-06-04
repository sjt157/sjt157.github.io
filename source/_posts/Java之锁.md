---
title: Java之锁
date: 2019-03-23 13:33:26
tags: Java
categories: Java
---
### synchronized

#### 基本原理
Java中每个对象都有一个内置锁(监视器,也可以理解成锁标记)，而synchronized就是使用对象的内置锁(监视器)来将代码块(方法)锁定的！
* 同步代码块：

monitorenter和monitorexit指令实现的

* 同步方法（在这看不出来需要看JVM底层实现）

方法修饰符上的ACC_SYNCHRONIZED实现。

synchronized底层是是通过monitor对象，对象有自己的对象头，存储了很多信息，其中一个信息标示是被哪个线程持有。

具体可参考：

<https://blog.csdn.net/chenssy/article/details/54883355>

<https://blog.csdn.net/u012465296/article/details/53022317>


#### 为什么说Synchronized是重量级锁？
这时因为当获取monitor的线程发现此monitor已被其他线程持有时会陷入BLOCKED阻塞状态，而这一操作是通过LWP来完成的，会引起用户态到内核态的切换，这甚至会导致切换到内核态调用OS将线程阻塞的时间比同步代码块执行所需的时间还要长。

#### 简述锁的等级 （方法锁、对象锁、类锁）

* 无论是修饰方法还是修饰代码块都是 对象锁,当一个线程访问一个带synchronized方法时，由于对象锁的存在，所有加synchronized的方法都不能被访问（前提是在多个线程调用的是同一个对象实例中的方法）
* 无论是修饰静态方法还是锁定某个对象,都是 类锁.一个class其中的静态方法和静态变量在内存中只会加载和初始化一份，所以，一旦一个静态的方法被申明为synchronized，此类的所有的实例化对象在调用该方法时，共用同一把锁，称之为类锁。

* synchronized修饰静态方法获取的是类锁(类的字节码文件对象)，synchronized修饰普通方法或代码块获取的是对象锁。

它俩是不冲突的，也就是说：获取了类锁的线程和获取了对象锁的线程是不冲突的！
```java
public class SynchoronizedDemo {

    //synchronized修饰非静态方法
    public synchronized void function() throws InterruptedException {
        for (int i = 0; i <3; i++) {
            Thread.sleep(1000);
            System.out.println("function running...");
        }
    }
    //synchronized修饰静态方法
    public static synchronized void staticFunction()
            throws InterruptedException {
        for (int i = 0; i < 3; i++) {
            Thread.sleep(1000);
            System.out.println("Static function running...");
        }
    }

    public static void main(String[] args) {
        final SynchoronizedDemo demo = new SynchoronizedDemo();

        // 创建线程执行静态方法
        Thread t1 = new Thread(() -> {
            try {
                staticFunction();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        // 创建线程执行实例方法
        Thread t2 = new Thread(() -> {
            try {
                demo.function();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        // 启动
        t1.start();
        t2.start();
    }
}
```
Tips:
1. synchronized关键字不能继承。也就是说子类重写了父类中用synchronized修饰的方法，子类的方法仍然不是同步的。
2. 定义接口方法时，不能使用synchronized关键字。
3. 构造方法不能使用synchronized关键字，但是可以使用synchronized代码块。



#### synchronized锁的膨胀过程是怎样的？
您指的是偏向锁、轻量级锁和重量级锁吗？就是如果临界区只有一个线程访问，这时会将锁对象的Mark Word中的偏向线程ID指向该线程，此后该线程进入临界区将省去锁获取-释放，毕竟锁获取-释放是有开销的。但是如果访问临界区的线程变多了，这时撤销偏向和重置偏向会有一定的开销，因而膨胀成轻量级锁，思路是会在线程的栈中存一份锁对象的Mark Word副本，称之为Displaced Mark Word，并该内存的指针存入锁对象的Mark Word。每个线程在获取锁的时候都需要CAS替换该指针，使其指向自己栈中的Displaced Mark Word。CAS替换失败则说明并发程度较高，膨胀成重量级锁，具有排他性。


#### 在spring事务中使用synchronized应该注意的问题

@Transcational注解和synchronized一起使用了，加锁的范围没有包括到整个事务。所以我们可以这样做：

新建一个名叫SynchronizedService类，让其去调用addEmployee()方法，整个代码如下：

```java
@RestController
public class EmployeeController {

    @Autowired
    private SynchronizedService synchronizedService ;

    @RequestMapping("/add")
    public void addEmployee() {
        for (int i = 0; i < 1000; i++) {
            new Thread(() -> synchronizedService.synchronizedAddEmployee()).start();
        }
    }
}

// 新建的Service类
@Service
public class SynchronizedService {

    @Autowired
    private EmployeeService employeeService ;
	
    // 同步
    public synchronized void synchronizedAddEmployee() {
        employeeService.addEmployee();

    }
}

@Service
public class EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    
    @Transactional
    public void addEmployee() {

        // 查出ID为8的记录，然后每次将年龄增加一
        Employee employee = employeeRepository.getOne(8);
        System.out.println(Thread.currentThread().getName() + employee);
        Integer age = employee.getAge();
        employee.setAge(age + 1);

        employeeRepository.save(employee);

    }
}

```

### ReentrantLock
#### 什么情况下使用ReenTrantLock：
需要实现ReenTrantLock的三个独有功能时。

#### ReenTrantLock独有的能力：
1. ReenTrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。
2. ReenTrantLock提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程。
3. ReenTrantLock提供了一种能够中断等待锁的线程的机制，通过lock.lockInterruptibly()来实现这个机制。

#### ReenTrantLock实现的原理：
在网上看到相关的源码分析，本来这块应该是本文的核心，但是感觉比较复杂就不一一详解了，简单来说，ReenTrantLock的实现是一种自旋锁，通过循环调用CAS操作来实现加锁。它的性能比较好也是因为避免了使线程进入内核态的阻塞状态。想尽办法避免线程进入内核的阻塞状态是我们去分析和理解锁设计的关键钥匙。

#### synchronized 和 ReentrantLock 有什么不同
* 能否可重入
他们两个都是可重入的，（可重入指的是同一个线程每进入一次，锁的计数器都自增1，所以要等到锁的计数器下降为0时才能释放锁。）平常开发中大多数还是用的synchronized,比较高级的才用后者，
* 能否造成死锁
Synchronized不会造成死锁，因为jvm会自动释放，别的锁有可能会造成死锁。
* 锁的实现
Synchronized是依赖于JVM实现的，而ReenTrantLock是JDK实现的，有什么区别，说白了就类似于操作系统来控制实现和用户自己敲代码实现的区别。前者的实现是比较难见到的，后者有直接的源码可供阅读。
* 性能的区别
在Synchronized优化以前，synchronized的性能是比ReenTrantLock差很多的，但是自从Synchronized引入了偏向锁，轻量级锁（自旋锁）后，两者的性能就差不多了，在两种方法都可用的情况下，官方甚至建议使用synchronized，其实synchronized的优化我感觉就借鉴了ReenTrantLock中的CAS技术。都是试图在用户态就把加锁问题解决，避免进入内核态的线程阻塞。
* 功能区别
便利性：很明显Synchronized的使用比较方便简洁，并且由编译器去保证锁的加锁和释放，而ReenTrantLock需要手工声明来加锁和释放锁，为了避免忘记手工释放锁造成死锁，所以最好在finally中声明释放锁。
锁的细粒度和灵活度：很明显ReenTrantLock优于Synchronized

### 读写锁

读写锁。分为读锁和写锁，多个读锁不互斥，读锁与写锁互斥，由Java虚拟机控制。如果代码允许很多线程同时读，但不能同时写，就上读锁；如果代码不允许同时读，并且只能有一个线程在写，就上写锁。

读写锁的接口是ReadWriteLock，具体实现类是 ReentrantReadWriteLock。synchronized属于互斥锁，任何时候只允许一个线程的读写操作，其他线程必须等待；而ReadWriteLock允许多个线程获得读锁，但只允许一个线程获得写锁，效率相对较高一些。

```java
使用枚举创建一个读写锁的单例
public enum Locker {

	INSTANCE;

	private static final ReadWriteLock lock = new ReentrantReadWriteLock();

	public Lock writeLock() {
		return lock.writeLock();
	}

}
再在addCount()方法中对count++;上锁。示例如下。
public static void addCount() {
	// 上锁
	Lock writeLock = Locker.INSTANCE.writeLock();
	writeLock.lock();
	count++;
	// 释放锁
	writeLock.unlock();
}

```


### 自旋锁，

### 偏向锁，轻量级锁，
偏向锁和轻量级锁就是为了避免阻塞，避免操作系统的介入
偏向锁：通常只有一个线程在临界区执行
轻量级锁：可以用多个线程交替进入临界区，在竞争不激烈的时候，稍微自旋等待一会就能获得锁
重量级锁：出现了激烈的竞争，只好阻塞


### 可重入锁，
#### 什么时候应该使用可重入锁

### 公平锁，非公平锁，

（先申请先得到）多个线程在等待同一个锁时，必须按照申请锁的时间顺序 来依次获取，而非公平锁则不保证这一点。Synchronized 不是公平锁，reetrantlock 默认下也 是非公平的，但是在构造函数中，可以设置为公平的。

### 乐观锁，悲观锁

#### 什么是乐观锁悲观锁

悲观锁：就是很悲观，每次去拿数据的时候都认为别人会修改， 所以每次在拿数据的时候都会上锁。这样别人想拿这个数据就会 block 直到它拿到锁。传统 的关系型数据库就用到了很多这种机制，比如行锁，写锁等，都是在操作之前上锁。2）乐 观锁：就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新 的时候会判断一下在此期间别人有没有去更新这个数据。适用于多读，比如 write_condition. 两种锁各有优缺点，不能认为一种比一种好。

#### CAS如何解决ABA问题
CAS操作需要输入两个数值，一个旧值（原值）和一个新值，在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。
如何解决ABA问题，简单来说就是CAS只是比较值是否一样，但有可能值一样，但已经不是原来的东西了，所以通过引入版本号来解决该ABA问题，引入版本号之后，变量的修改操作就变成了1A---2B---3A。


### 死锁与活锁的区别，死锁与饥饿的区别
虽然trylock（）方案避免了无尽的死锁，但是不能避免死锁，这样可能发生活锁现象，活锁就是如果所有死锁线程同时超时，他们极有可能再次陷入死锁，虽然死锁没有永远持续下去，但是对资源的争夺状况却没有改善
有一种方法可以减少活锁的几率，比如为每个线程设置不同的超时时间

### 参考
<https://www.cnblogs.com/wpf-7/p/9639671.html>
<http://www.zhenganwen.top/posts/5c6e8cdf/>
<http://zhenganwen.top/posts/ca0f0d75/>
<https://blog.csdn.net/wufaliang003/article/details/78797203>
