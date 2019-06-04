---
title: Docker化秒杀疑云
date: 2019-01-25 17:58:55
tags: Docker
categories: Docker
---


0. 为什么用docker-compose就有一堆错误，而一个一个起容器就不会有错误呢？所有的操作都一样啊。。莫非跟网关有关系吗？？一个用br-xxxxx，一个用docker0。docker-compose用的网关是br-10e925ab0d49这种形式的。而且每次br还不一样。没有docker0作为网关

下面是报的一些错误：

1. 
```java
Caused by: com.rabbitmq.client.AuthenticationFailureException: ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN. For details 
see the broker logfile.	at com.rabbitmq.client.impl.AMQConnection.start(AMQConnection.java:342) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:909) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:859) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:799) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at org.springframework.amqp.rabbit.connection.AbstractConnectionFactory.createBareConnection(AbstractConnectionFactory.java:352) ~[spring-ra
bbit-1.7.4.RELEASE.jar!/:na]

```
原因：默认安装好rabbitmq就有服务（windows服务），但没有web监控（localhost:15672）;
默认的端口就是5672，用户guest密码guest，但这个用户名只能在本机访问，如果要网络访问，需要做用户管理；
参考：<https://blog.csdn.net/lwkcn/article/details/25086467>
得知，就是写错用户名和密码了。

2. 将rabbitmq的172.21.6.57改为localhost之后 报错信息变成了
```java
2019-01-21 02:32:22.362 ERROR 1 --- [cTaskExecutor-2] o.s.a.r.l.SimpleMessageListenerContainer : Failed to check/redeclare auto-delete queue(s).

org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused (Connection refused)
	at org.springframework.amqp.rabbit.support.RabbitExceptionTranslator.convertRabbitAccessException(RabbitExceptionTranslator.java:62) ~[sprin
g-rabbit-1.7.4.RELEASE.jar!/:na]	at org.springframework.amqp.rabbit.connection.AbstractConnectionFactory.createBareConnection(AbstractConnectionFactory.java:368) ~[spring-ra
bbit-1.7.4.RELEASE.jar!/:na]	at org.springframework.amqp.rabbit.connection.CachingConnectionFactory.createConnection(CachingConnectionFactory.java:573) ~[spring-rabbit-1
.7.4.RELEASE.jar!/:na]	at org.springframework.amqp.rabbit.core.RabbitTemplate.doExecute(RabbitTemplate.java:1430) ~[spring-rabbit-1.7.4.RELEASE.jar!/:na]
	at org.springframework.amqp.rabbit.core.RabbitTemplate.execute(RabbitTemplate.java:1411) ~[spring-rabbit-1.7.4.RELEASE.jar!/:na]
	at org.springframework.amqp.rabbit.core.RabbitTemplate.execute(RabbitTemplate.java:1387) ~[spring-rabbit-1.7.4.RELEASE.jar!/:na]
	at org.springframework.amqp.rabbit.core.RabbitAdmin.getQueueProperties(RabbitAdmin.java:336) ~[spring-rabbit-1.7.4.RELEASE.jar!/:na]
	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.redeclareElementsIfNecessary(SimpleMessageListenerContainer.java:
1171) ~[spring-rabbit-1.7.4.RELEASE.jar!/:na]	at org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer$AsyncMessageProcessingConsumer.run(SimpleMessageListenerContainer
.java:1422) [spring-rabbit-1.7.4.RELEASE.jar!/:na]	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_111]
Caused by: java.net.ConnectException: Connection refused (Connection refused)
	at java.net.PlainSocketImpl.socketConnect(Native Method) ~[na:1.8.0_111]
	at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350) ~[na:1.8.0_111]
	at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206) ~[na:1.8.0_111]
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188) ~[na:1.8.0_111]
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_111]
	at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_111]
	at com.rabbitmq.client.impl.SocketFrameHandlerFactory.create(SocketFrameHandlerFactory.java:50) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:907) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:859) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:799) ~[amqp-client-4.0.3.jar!/:4.0.3]
	at org.springframework.amqp.rabbit.connection.AbstractConnectionFactory.createBareConnection(AbstractConnectionFactory.java:352) ~[spring-ra
bbit-1.7.4.RELEASE.jar!/:na]	... 8 common frames omitted

```

3. 应用间网络还不完善，应该使用网络隔离，app在front，其余都在back。
   `docker network create front`命令是创建网络的。

4. 默认的rabbitmq:3镜像没有management，应该用rabbitmq:management，我还怕是不是因为版本不一致导致的问题，专门用的rabbitmq:3.7.5-management，还是不行，还是报错。

5. 
```java
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	at sun.reflect.GeneratedConstructorAccessor31.newInstance(Unknown Source) ~[na:na]
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45) ~[na:1.8.0_111]
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423) ~[na:1.8.0_111]
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:425) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:989) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.MysqlIO.<init>(MysqlIO.java:341) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2189) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2222) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2017) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:779) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:47) ~[mysql-connector-java-5.1.44.jar!/:5.1.44]
	at sun.reflect.GeneratedConstructorAccessor28.newInstance(Unknown Source) ~[na:na]

```
莫非是因为 在容器里边，他不认 localhost吗？？？必须写真实的IP才能找到。。。。

6. 为什么没写到mysql中去呢 ，秒杀成功后吗，应该库存减少啊 但是并没有减少。。。。
可能是因为发送了 秒杀请求，但是没有进行消息确认，所以 库存的数据量并不会减少。。。下一次又会读取到7 这个数量。。。。

7. 现在的业务逻辑有个错误啊 ，我第一次秒杀成功后，在点击秒杀，确实说 非法请求了 
但是redis里还是减少了一个 商品 ，但是mysql里没有减少，说明没有发送成功秒杀的消息给mysql   所以并没有减少。


8. `update s_goods set start_date='2019-01-22 13:55:50' where id=1;`    注意有八个小时的时差 数据库里是05点，页面上显示13点。
9. 
```java
com.sjt.miaosha.rabbitmq.MQSender        : send message:{"goodsId":1,"user":{"id":15812341234,
"lastLoginDate":1525086640000,"loginCount":1,"nickname":"jack","password":"b631975e482e55b7692106f55a5b0a82","registerDate":1525086636000,"salt":"1a2b3c"}}2019-01-22 05:52:35.118  INFO 1 --- [io-65510-exec-8] o.s.a.r.c.CachingConnectionFactory       : Created new connection: rabbitConnectionFactory#1ca
81352:40/SimpleConnection@3f516387 [delegate=amqp://admin@172.21.6.57:5672/, localPort= 57984]2019-01-22 05:53:02.388  INFO 1 --- [io-65510-exec-3] c.s.miaosha.controller.LoginController   : LoginVo [mobile=15812341234, password=ae6712ae454da
07f5a86e2b5b21f4ea2]2019-01-22 05:53:10.551  INFO 1 --- [io-65510-exec-1] com.sjt.miaosha.rabbitmq.MQSender        : send message:{"goodsId":1,"user":{"id":15812341234,
"lastLoginDate":1525086640000,"loginCount":1,"nickname":"jack","password":"b631975e482e55b7692106f55a5b0a82","registerDate":1525086636000,"salt":"1a2b3c"}}
```


