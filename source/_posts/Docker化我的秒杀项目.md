---
title: Docker化我的秒杀项目
date: 2018-12-09 17:58:10
tags: Docker
categories: Docker
---


## Docker化秒杀项目
* 首先构建jar包，因为该秒杀基于maven的，第一开始想把配置文件啥的都放一起，可以用一些eclipse插件啥的，但是后来一想这样不好，因为如果我想改配置的话需要重新构建，所以在网上找了个方法，构建好maven install 之后是这样子的，参考的这篇文章<http://https://blog.csdn.net/qq_22857293/article/details/79416165>，最后构建之后的结果如图，可以看出第三方依赖，配置文件，资源分离开来：
* （当把jar包通过Xshell上传文件的时候出现了不能上传的问题，这是因为目标目录没有相应写权限造成的，通过chmod o+w 赋予相应权限即可。） *
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/DockerMiaoSha/1.png)

* 启动jar包的命令是java -jar -Dloader.path=.,config,resources,3rd-lib miaosha.jar
然后我写了Dockerfile，内容如下：
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/DockerMiaoSha/2.png)
构建镜像 docker build -t sjt157/miaoshaapp:v2 .（注意后边有个.）
运行docker容器：docker run -d -p 65534:65510 sjt157/miaoshaapp:v2

## Docker化Rabbit 
```python

首先pull官方镜像，Docker pull rabbitmq:3
启动docker run -d --name miaosharabbit -p 5672:5672 -p 15672:15672 rabbitmq:3
当时启动rabbit容器的时候出现了异常 An unexpected connection driver error occurred的话，执行这个
```
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/DockerMiaoSha/3.png)

## Docker化Mysql
```python

首先拉取镜像，docker pull mysql:5.7
使用镜像创建容器
docker run -p 3306:3306 --name miaoshaMysql -e MYSQL_ROOT_PASSWORD=123 -d mysql:5.7
进入容器
docker exec -it miaoshaMysql bash
进入mysql -uroot -p
输入 GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123' WITH GRANT OPTION;
FLUSH PRIVILEGES;
复制sql文件到容器
docker inspect -f '{{.ID}}' miaoshaMysql
docker cp 本地路径 容器长ID:容器路径
我把文件考到了根目录下
docker cp /home/ubuntu16/docker/miaoshaMysql/seckill.sql 561e41a0347d:/
首先进入mysql 创建数据库
create database miaosha;
在执行mysql -uroot -p  miaosha < miaosha.sql命令导入文件
我要把原镜像 重新弄一个我的数据库镜像
docker commit -m "added miaosha.sql" -a "sjt157" 561e41a0347d sjt157/miaoshamysql:v2
运行我自己的镜像
docker run -p 3306:3306 --name miaoshamysql -e MYSQL_ROOT_PASSWORD=123 -d sjt157/miaoshamysql:v2
```    

* 此处有个疑问？？？
 为什么在官方mysql镜像上 我做了相应操作，存了文件，执行了命令，只有文件被保存了，但是 我做的其他命令 却没有保存上？？？比如创建数据库表等命令 这是为什么？那如果是用Dockerfile创建的自己的镜像会不会也是这样呢，相关命令不保存？？？还有一个，为什么容器已经是退出状态了，却也不能重名呢？？必须要rm掉吗。因为已经是退出状态了，所以不能kill。

## Docker化Redis   
不用 redis.password=123456这个加密码的配置了，因为默认的官方镜像是没有的。
docker pull  redis:4（有个疑问。不能制定详细的版本信息吗？？应该可以把）
启动redis容器docker run -p 6379:6379 --name miaosharedis -d redis
最后可以看到4个容器均运行成功。

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/DockerMiaoSha/4.png)

## 下一步工作
这样太麻烦了，每次都要手动执行，能不能自动编排呢？利用compose来操作。
