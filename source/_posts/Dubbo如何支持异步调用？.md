---
title: Dubbo如何支持异步调用？
date: 2019-03-11 19:12:49
tags: Dubbo
categories: Dubbo
---


### 为什么异步？
异步化的好处也是比较明显的，可以加快后台的处理效率，做到代码直接的解耦，Dubbo就是一个支持异步调用的RPC框架。

### 在Dubbo中使用异步
* 异步调用的场景

假设系统A，远程调用B系统的某个方法，这个方法与数据库的交互很多，逻辑相对复杂，
正常的代码执行的时间是3秒，A系统调用完B系统之后，还需要做一些其他的逻辑操作，
这个代码耗时可能需要4秒，等这个3秒的逻辑做完之后，根据B系统返回的结果再做一些其他的操作，
那么同步调用的时间是3秒+4秒 = 7秒，那么一次操作的时间就是7秒

* 同步调用的实现：

接口实现（provider和consumer端都需要）
```java

    package org.bazinga.service;  
      
    public interface AsyncInvokeService {  
          
        public Integer getResult();  
      
    }  
```
接口实现，我们默认线程sleep三秒，3秒代表代码复杂的逻辑操作和数据库的交互的时间
```java

    package org.bazinga.service.impl;  
      
    import org.bazinga.service.AsyncInvokeService;  
      
    public class AsyncInvokeServiceImpl implements AsyncInvokeService {  
      
        public Integer getResult() {  
              
            try {  
                Thread.sleep(3000l); //模拟复杂的逻辑操作时间和数据库交互的时间消耗  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
            return 1;  
        }  
      
      
    }  
```
在不开启异步调用的配置的时候，spring的配置文件和普通配置是一样的spring-dubbo-provider-async.xml，
需要注意的是我们线程故意睡了3秒，这边我们配置timeout的时间为4秒，否则就会调用超时：
```xml

    <?xml version="1.1" encoding="UTF-8"?>  
    <beans xmlns="http://www.springframework.org/schema/beans"  
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"  
        xsi:schemaLocation="http://www.springframework.org/schema/beans    
           http://www.springframework.org/schema/beans/spring-beans.xsd    
           http://code.alibabatech.com/schema/dubbo    
           http://code.alibabatech.com/schema/dubbo/dubbo.xsd">  
             
        <dubbo:application owner="lyncc" name="bazinga-app" />  
        <!--zookeeper注册中心 -->  
        <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>   
          
        <dubbo:protocol name ="dubbo" port="20880" />  
        <!-- 发布这个服务  调用超时是4秒，支持异步调用-->  
        <dubbo:service  protocol="dubbo"  timeout="4000"  interface ="org.bazinga.service.AsyncInvokeService" ref="asyncInvokeService"/>         
        <!-- 和本地bean一样实现服务 -->  
        <bean id="asyncInvokeService" class="org.bazinga.service.impl.AsyncInvokeServiceImpl" />  
          
    </beans>  
```
消费端的spring配置文件spring-dubbo-consumer-async.xml：
```xml

    <?xml version="1.1" encoding="UTF-8"?>  
    <beans xmlns="http://www.springframework.org/schema/beans"  
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"  
        xsi:schemaLocation="http://www.springframework.org/schema/beans    
           http://www.springframework.org/schema/beans/spring-beans.xsd    
           http://code.alibabatech.com/schema/dubbo    
           http://code.alibabatech.com/schema/dubbo/dubbo.xsd">  
             
        <dubbo:application owner="lyncc" name="bazinga-consumer" />  
        <!--zookeeper注册中心 -->  
        <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>   
          
        <dubbo:reference id="asyncInvokeService" interface="org.bazinga.service.AsyncInvokeService"/>   
          
    </beans>  
```
服务提供者端的测试类：
```java

    package org.bazinga.service.test;  
      
    import org.springframework.context.support.ClassPathXmlApplicationContext;  
      
    public class DubboxProviderAsyncService {  
      
        public static void main(String[] args) throws InterruptedException {  
            ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(  
                    "spring-dubbo-provider-async.xml");  
            context.start();  
            Thread.sleep(2000000l);  
        }  
      
    }  
```
服务消费者端的测试类：
```java

    package org.bazinga.service.test;  
      
    import org.bazinga.service.AsyncInvokeService;  
    import org.springframework.context.support.ClassPathXmlApplicationContext;  
      
    public class DubboConsumerAsyncService {  
      
        public static void main(String[] args) throws InterruptedException {  
            ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(  
                    "spring-dubbo-consumer-async.xml");  
            context.start();  
      
            long beginTime = System.currentTimeMillis();  
      
            for (int count = 0; count < 10; count++) { // 调用10次  
                 AsyncInvokeService asyncInvokeService =  
                 (AsyncInvokeService)context.getBean("asyncInvokeService");  
                 Integer result = asyncInvokeService.getResult(); //wait 返回结果 等待3秒  
                  
                 Thread.sleep(4000l); //模拟本地复杂的逻辑操作，耗时4秒  
                  
                 Integer localcalcResult = 2;//本地经过4秒处理得到的计算数据是2  
                  
                 System.out.println(result + localcalcResult);//根据远程调用返回的结果和本地操作的值，得到结果集  
      
            }  
            System.out.println("call 10 times,cost time is "  
                    + (System.currentTimeMillis() - beginTime));  
      
            Thread.sleep(2000000l);  
        }  
    }  

```
先启动DubboxProviderAsyncService，然后再启动DubboConsumerAsyncService的main函数：

消费端的控制台打印信息是：
运行没有问题，但是调用10次，一共耗时71秒，假如改成异步调用，我们不需要等待调用返回的结果，而是在用的时候，再去获取值的话，这样会大大的提高执行的速度

* 异步调用实现

异步调用的实现，很简单，现在我们修改一下配置文件，使其支持异步调用，其实配置相对比较简单，只需要在调用端的spring的配置文件中加上async=”true”,注意一定是在调用端配置该关键字，异步调用，顾名思义，就是需要告之调用者，调用之后不需要等待（Note：修改的是调用的spring配置文件spring-dubbo-consumer-async.xml）
```xml

    <?xml version="1.1" encoding="UTF-8"?>  
    <beans xmlns="http://www.springframework.org/schema/beans"  
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"  
        xsi:schemaLocation="http://www.springframework.org/schema/beans    
           http://www.springframework.org/schema/beans/spring-beans.xsd    
           http://code.alibabatech.com/schema/dubbo    
           http://code.alibabatech.com/schema/dubbo/dubbo.xsd">  
             
        <dubbo:application owner="lyncc" name="bazinga-consumer" />  
        <!--zookeeper注册中心 -->  
        <dubbo:registry protocol="zookeeper" address="127.0.0.1:2181"/>   
          
        <dubbo:reference id="asyncInvokeService" interface="org.bazinga.service.AsyncInvokeService" async="true"/>   
          
    </beans>  
```
其实就是加了一个标签的支持，async="true"，这样就支持了异步的调用了

修改消费者的测试代码，配置文件的修改只是告诉Dubbo，调用者会进行异步调用，但如何异步调用，还是需要调用者自己去实现的，实现依赖于RpcContext：
```java

    package org.bazinga.service.test;  
      
    import java.util.concurrent.Future;  
      
    import org.bazinga.service.AsyncInvokeService;  
    import org.springframework.context.support.ClassPathXmlApplicationContext;  
      
    import com.alibaba.dubbo.rpc.RpcContext;  
      
    public class DubboConsumerAsyncService {  
      
        public static void main(String[] args) throws InterruptedException {  
            ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(  
                    "spring-dubbo-consumer-async.xml");  
            context.start();  
      
            long beginTime = System.currentTimeMillis();  
      
            for (int count = 0; count < 10; count++) { // 调用10次  
    //           AsyncInvokeService asyncInvokeService =  
    //           (AsyncInvokeService)context.getBean("asyncInvokeService");  
    //           Integer result = asyncInvokeService.getResult(); //wait 返回结果 等待3秒  
    //            
    //           Thread.sleep(4000l); //模拟本地复杂的逻辑操作，耗时4秒  
    //            
    //           Integer localcalcResult = 2;//本地经过4秒处理得到的计算数据是2  
    //            
    //           System.out.println(result + localcalcResult);//根据远程调用返回的结果和本地操作的值，得到结果集  
                   
                AsyncInvokeService asyncInvokeService = (AsyncInvokeService) context  
                .getBean("asyncInvokeService");  
                Integer remotingResult = asyncInvokeService.getResult(); // 不等待  
                  
                Thread.sleep(4000l); // 模拟本地复杂的逻辑操作，耗时4秒  
                  
                Future<Integer> future = RpcContext.getContext().getFuture();  
                try {  
                remotingResult = future.get();  
                } catch (java.util.concurrent.ExecutionException e) {  
                e.printStackTrace();  
                  
                }  
                Integer localcalcResult = 2;// 本地经过4秒处理得到的计算数据是2  
                  
                System.out.println(remotingResult + localcalcResult);// 根据远程调用返回的结果和本地操作的值，得到结果集  
      
            }  
            System.out.println("call 10 times,cost time is "  
                    + (System.currentTimeMillis() - beginTime));  
      
            Thread.sleep(2000000l);  
        }  
    }  
```
关键代码就是：

    `Integer remotingResult = asyncInvokeService.getResult(); // 不等待`  

这行代码不会阻塞至服务提供者端把数据返回，而是直接返回，然后睡了4秒，这个4秒，模仿的是本地的逻辑操作，这是其实远程的逻辑也在执行，这样就可以并行操作了，最后调用：

    `Future<Integer> future = RpcContext.getContext().getFuture(); ` 

获取到远程调用异步返回的结果，完成最后的操作，我们可以看啊看控制台打印的结果：
可以看出结果没有变，仍旧是3，但是调用时间变成了41秒，异步调用的好处就可以显示出来了

