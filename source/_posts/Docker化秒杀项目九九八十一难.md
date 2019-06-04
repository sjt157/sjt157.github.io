---
title: Docker化秒杀项目九九八十一难
date: 2019-01-20 16:18:36
tags: Docker
categories: Docker
---


## Docker化Mysql

在我的Docker秒杀项目中，第一开始我都是把执行脚本和创建数据库文件都先弄到一个镜像中去，然后基于这个新镜像在起docker容器，但是后来一想，这种方法很麻烦，有没有一种更好的方法呢？通过一篇文章，有了灵感。

### Volume持久化
首先，我们可以在创建数据库容器的时候指定容器内数据库创建
`docker run --name mysqldock -e MYSQL_ROOT_PASSWORD=admin -e MYSQL_DATABASE=inst1 -d -p 3066:3066 mysql`
通过`-e MYSQL_DATABASE`这个配置就可以在创建容器时创建inst1这个数据库。

官方文档对`docker commit`的说明是：
The commit operation will not include any data contained in volumes mounted inside the container.

意思是commit操作并不会包含容器内挂载数据卷中的数据变化。难道是因为mysql容器的挂载数据卷引起的？
也就是说数据库容器/var/lib/mysql路径作为volume挂载在host主机上/var/lib/docker/volumes/该容器ID。

但是通过-e参数创建的数据库就还在，继续看`docker commit`官方文档
It can be useful to commit a container’s file changes or settings into a new image.
对于文件变更(在容器内新创建一个文件)和设置（-e MYSQL_DATABASE=inst1）有用。

到此，原来我的疑问也就明朗了。


----------


### 那么我们如何来对数据库容器进行持久化呢？

我们可以通过docker提供的数据挂载来实现。
docker的数据挂载分为三种，volume, bind mount和tmpfs，关于三种的具体说明，强烈推荐大家看一下官网的文档。这边简单说明一下：

* volume是由docker默认及推荐的挂载方式，volume由docker直接管理，同一个volume可以共享给多个容器使用，volume和容器的生命周期完全独立，容器删除时volume仍然存在，除非使用docker volume相应命令删除volume；缺点是volume在宿主机上比较难定位，在宿主机上直接操作volume比较困难。

* bind mount是直接将宿主机文件系统上的文件路径映射到容器中，两边双向同步，显而易见，有缺点也有优点，优点是可以直接访问，也可以被别的程序使用，比如我们打包一个本地应用到本地/target路径，我们就可以把这个路径使用bind mount的方式挂在到依赖他的应用的docker容器中，这样本地应用打包后，docker里的数据卷也会同时更新；缺点也是显而易见的，因为你可以把任何文件路径使用bind mount的方式绑定到容器中，这样有可能一些安全问题，比如把宿主机的系统文件绑定到容器中。

* tmpfs这种方式是使用宿主机的内存作为存储，不会写到宿主机的文件系统中，和前两种区别较大。


### 什么是Volume呢？
为了能够保存（持久化）数据以及共享容器间的数据，Docker提出了Volume的概念。简单来说，Volume就是目录或者文件，它可以绕过默认的联合文件系统，而以正常的文件或者目录的形式存在于宿主机上。

**注意：**-v 的时候不能使用同一个源地址，否则创建起来的容器会闪退（再创建一个data目录来区分）

### 下面是各种磨难

1. 为什么我自己新创建的数据库镜像mysql默认是关闭的？必须的通过`/etc/init.d/mysql start` 才能启动呢？？？

在脚本中还不行，必须得手动进入容器内进行执行该命令，百思不得姐啊 。。。。。
通过`docker logs`发现
MySQL Community Server 5.7.24 is started.
mysql: [Warning] Using a password on the command line interface can be insecure.
/mysql/createDatabase.sh: 21: /mysql/createDatabase.sh: cannot open miaosha.sql: No such file
发现数据库已经起来了，但是说不能打开sql文件，难道是因为权限问题，？？？？必须在容器内才能拥有权限？？？（经证实。不是权限问题，是sql文件路径问题）

2. 这次我手动在容器内执行 脚本 就能成功，（第一次说找不到sql文件，第二次就成功了，到底是什么鬼？？？？？？）
[info] A MySQL Server is already started.
mysql: [Warning] Using a password on the command line interface can be insecure.
mysql: [Warning] Using a password on the command line interface can be insecure.

最后，终于成功了，是因为路径的原因。
原来为`mysql -u${USERNAME} -p${PASSWORD}  miaosha < miaosha.sql`
改为
  ` mysql -u${USERNAME} -p${PASSWORD}  miaosha < /mysql/miaosha.sql`

3. docker容器启动后马上退出
经查阅资料：
Docker容器同时只能管理一个进程，如果这个进程退出那么容器也就退出了，但这不表示容器只能运行一个进程(其他进程可在后台运行)，但是要使容器不退出必须有一个前台执行的进程。
我这里通过，`tail -f /dev/null`这个命令 不让它退出。

4. 挂载某一文件的问题

挂载之前需要改变文件权限为777，要不会引起修改宿主机上的文件 会引起内容不同步的问题
参考<https://blog.csdn.net/qq_21816375/article/details/78032521>


### 成功，数据库启动正常
`docker run  -d -p 3306:3306 --name miaoshaMysql  -e MYSQL_ROOT_PASSWORD=123 -v /
home/ubuntu16/miaosha-docker-compose/config/miaoshaMysql:/mysql -v /home/ubuntu16/miaosha-docker-compose/data/miaoshaMysql:/var/lib/mysql sjt157/miaoshamysql:v4`

## Docker化Rabbitmq

遇到这个错误
[error] Error when reading /var/lib/rabbitmq/.erlang.cookie: eacces

用这个命令解决`chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie`

最终成功启动命令
`docker run -d --name miaosharabbit -p 5672:5672 -p 15672:15672 -v /home/ubuntu16/miaosha-dock
er-compose/config/miaoshaRabbit:/rabbitmq sjt157/miaosharabbitmq:v4`


## Docker化redis持久化
参考mysql的持久化，我们也需要对redis进行持久化，这样我们的容器退出后才不会所有东西全部丢失。参考<https://blog.csdn.net/haoxiaoyong1014/article/details/80241677>

最终命令
`docker run -d -p 6379:6379 --name miaosharedis sjt157/miaosharedis:v2`

## docker-compose
我们每次都docker run很麻烦，所以用docker-compose。
```Shell
version: '2.1'

services:
  mysql:
    image: sjt157/miaoshamysql:v4
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=123
     # - MYSQL_DATABASE=root
    volumes:
      - /home/ubuntu16/miaosha-docker-compose/config/miaoshaMysql:/mysql
      - /home/ubuntu16/miaosha-docker-compose/data/miaoshaMysql:/var/lib/mysql
      - /home/ubuntu16/miaosha-docker-compose/logs/miaoshaMysql:/logs
    restart: on-failure
  redis:
    image: sjt157/miaosharedis:v2
    ports:
      - "6379:6379"
    volumes:
      - /home/ubuntu16/miaosha-docker-compose/data/miaoshaRedis:/data
      - /home/ubuntu16/miaosha-docker-compose/logs/miaoshaRedis:/logs
    restart: on-failure
  rabbitmq:
    image: sjt157/miaosharabbitmq:v4
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - /home/ubuntu16/miaosha-docker-compose/config/miaoshaRabbit:/rabbitmq
      - /home/ubuntu16/miaosha-docker-compose/logs/miaoshaRabbit:/logs
    restart: on-failure
  miaosha:
    image: sjt157/miaoshaapp:v2
    ports:
      - "65534:65510"
    depends_on:
      - mysql
      - redis
      - rabbitmq
    restart: on-failure

```


通过docker-compose up -d启动成功
```
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                              
                                               NAMES7152d7c2f012        sjt157/miaoshaapp:v2        "java -jar -Dloader.…"   4 minutes ago       Up 4 minutes        0.0.0.0:65534->65510/tcp          
                                                miaosha-docker-compose_miaosha_1_6a294f3118b7f9fada42b2b9        sjt157/miaosharabbitmq:v4   "docker-entrypoint.s…"   4 minutes ago       Up 4 minutes        4369/tcp, 0.0.0.0:5672->5672/tcp, 
5671/tcp, 25672/tcp, 0.0.0.0:15672->15672/tcp   miaosha-docker-compose_rabbitmq_1_b1a645775dbfda0590907e62        sjt157/miaosharedis:v2      "docker-entrypoint.s…"   4 minutes ago       Up 4 minutes        0.0.0.0:6379->6379/tcp            
                                                miaosha-docker-compose_redis_1_6d55cb3068980974f7a50423        sjt157/miaoshamysql:v4      "docker-entrypoint.s…"   4 minutes ago       Up 4 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp 
                                                miaosha-docker-compose_mysql_1_321cab31d805
```
通过`docker-compose down` 停止。



### 参考
<https://www.jianshu.com/p/530d00f97cbf>
<https://blog.csdn.net/haoxiaoyong1014/article/details/80241677>
















