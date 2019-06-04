---
title: Docker容器可视化监控中心搭建
date: 2019-03-23 17:22:47
tags: Docker
categories: Docker
---



### 背景
我的服务器上跑了大概15个容器，比如kafka，zookeeper，ES等，我想监控他们怎么办？

### 准备镜像
* adviser：负责收集容器的随时间变化的数据
* influxdb：负责存储时序数据
* grafana：负责分析和展示时序数据


### 从下镜像到放弃（最终还是没成功）
 
#### 解决 Docker pull 出现的net/http: TLS handshake timeout 的一个办法
使用国内的Docker仓库daocloud：(没尝试)
```
$  "DOCKER_OPTechoS=\"\$DOCKER_OPTS --registry-mirror=http://f2d6cb40.m.daocloud.io\"" | sudo tee -a /etc/default/docker
$ sudo service docker restart
```

#### 遇到一个问题
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: x509: certificate is valid for goldopen.org, www.goldopen.org, not registr
y-1.docker.io.

经查资料，说
```
I suspect it's a man-in-the-middle proxy that modifies the SSL request. There are some instructions for configuring a proxy to be used by the docker daemon here; 
```
<https://github.com/moby/moby/issues/33154>

#### 我试了一下这玩意 

`openssl s_client -connect registry-1.docker.io:443`

返回了 以下内容

```shell
CONNECTED(00000003)
depth=4 C = US, O = "Starfield Technologies, Inc.", OU = Starfield Class 2 Certification Authority
verify return:1
depth=3 C = US, ST = Arizona, L = Scottsdale, O = "Starfield Technologies, Inc.", CN = Starfield Services Root Certificate Authority - G2
verify return:1
depth=2 C = US, O = Amazon, CN = Amazon Root CA 1
verify return:1
depth=1 C = US, O = Amazon, OU = Server CA 1B, CN = Amazon
verify return:1
depth=0 CN = goldopen.org
verify return:1
---
Certificate chain
 0 s:/CN=goldopen.org
   i:/C=US/O=Amazon/OU=Server CA 1B/CN=Amazon
 1 s:/C=US/O=Amazon/OU=Server CA 1B/CN=Amazon
   i:/C=US/O=Amazon/CN=Amazon Root CA 1
 2 s:/C=US/O=Amazon/CN=Amazon Root CA 1
   i:/C=US/ST=Arizona/L=Scottsdale/O=Starfield Technologies, Inc./CN=Starfield Services Root Certificate Authority - G2
 3 s:/C=US/ST=Arizona/L=Scottsdale/O=Starfield Technologies, Inc./CN=Starfield Services Root Certificate Authority - G2
   i:/C=US/O=Starfield Technologies, Inc./OU=Starfield Class 2 Certification Authority
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIFdDCCBFygAwIBAgIQAhgLcb8KjmLQqtrwjZIfSTANBgkqhkiG9w0BAQsFADBG
MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRUwEwYDVQQLEwxTZXJ2ZXIg
Q0EgMUIxDzANBgNVBAMTBkFtYXpvbjAeFw0xODA4MzEwMDAwMDBaFw0xOTA5MzAx
MjAwMDBaMBcxFTATBgNVBAMTDGdvbGRvcGVuLm9yZzCCASIwDQYJKoZIhvcNAQEB
BQADggEPADCCAQoCggEBAJD0no7+tthdbiSHNU76RGaGvIJ1OiwnHdwP37x86XlO
fUkpd3eiCvCcZ02xHhK7EDrI2pW6qKLQ7lMheK6yjlqEjT8tbQt2KUOebabRGIoR
KEYREbH917UUuS1zI5r64LR0kOfytNuKVkC5yCk8RXnh1xx2ZtqC4eBIFlNQzCEe
J9rS1hkPfF36+pEJkxVCluz3/RDDHoEBl8r/fxmcDzjUEI3fPIyEM3XGGKv52WPB
GsoG7l/Lmdm7qSEiXaLjRvi3EWfq48lDPcYSEFTeFq47k+s6Ua1K4xbdTVPVuvyL
DwqGyF0HAefLDDRQEJ/dZx6NOapD3RjppeMBNHoteVMCAwEAAaOCAoswggKHMB8G
A1UdIwQYMBaAFFmkZgZSoHuVkjyjlAcnlnRb+T3QMB0GA1UdDgQWBBReRu9vDv2Z
hjPTej4AAZBjGRGT+jApBgNVHREEIjAgggxnb2xkb3Blbi5vcmeCEHd3dy5nb2xk
b3Blbi5vcmcwDgYDVR0PAQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggr
BgEFBQcDAjA7BgNVHR8ENDAyMDCgLqAshipodHRwOi8vY3JsLnNjYTFiLmFtYXpv
bnRydXN0LmNvbS9zY2ExYi5jcmwwIAYDVR0gBBkwFzALBglghkgBhv1sAQIwCAYG
Z4EMAQIBMHUGCCsGAQUFBwEBBGkwZzAtBggrBgEFBQcwAYYhaHR0cDovL29jc3Au
c2NhMWIuYW1hem9udHJ1c3QuY29tMDYGCCsGAQUFBzAChipodHRwOi8vY3J0LnNj
YTFiLmFtYXpvbnRydXN0LmNvbS9zY2ExYi5jcnQwDAYDVR0TAQH/BAIwADCCAQUG
CisGAQQB1nkCBAIEgfYEgfMA8QB2AKS5CZC0GFgUh7sTosxncAo8NZgE+RvfuON3
zQ7IDdwQAAABZZGDMPgAAAQDAEcwRQIhAJMT8qC4MfkvcZVFy5TJkAwVEeIa6WVh
Xx7kIbczhKY/AiAuMRHhM8Fyb9dY9TdIb6BOGRryTQU+3VX8FKq92X7LXgB3AId1
v+dZfPiMQ5lfvfNu/1aNR1Y2/0q1YMG06v9eoIMPAAABZZGDMb0AAAQDAEgwRgIh
AIiRYC3UzfG92mYkaK+nuSZ801XfDTJceBHbKZNNAv7HAiEAts4qSJowajeMAG24
5yTh1/Y2yTkLxiR7oGkb9Bx8DBowDQYJKoZIhvcNAQELBQADggEBALarCZePQIy/
J821dkTEOp1p2QurBUpU6AqADnoAzwHhF4NNBx0jXUwwgbHFVs6XFizrYfsUKulE
U9xPS7xlGAIJNoo7r/7DV0nZtiqW1Re726PAjoWAUJOdXowFH9ddXNBtT7rRPK4h
XauQcfU7KvIBVUV0NGWqyAhbcUFMVYV7Sj10FqvrUbcE7wviv/be1KVHkc3bJhYC
3ACNy99lzfMRDOyQrG7sgYvwIKYC7W+f1kOyRZJ2FjaXn0woLT5yRe7QrRijZOrf
1NIGaoPAYnC6AO+WZtbKf468BNhzu60fhjaR7giQwKoSG1lb9wzIDZQBUMYhODh8
fC5MbIHlTUI=
-----END CERTIFICATE-----
subject=/CN=goldopen.org
issuer=/C=US/O=Amazon/OU=Server CA 1B/CN=Amazon
---
No client certificate CA names sent
Peer signing digest: SHA512
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 5336 bytes and written 431 bytes
---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: 42F89657AF05B89E836E24D639F5F84F23A3622E2D6661AD8C6C05E49CE30EC0
    Session-ID-ctx: 
    Master-Key: 2BCBB105DA476E2BF44FC37499AEE16BE237DA4BD72FBEB58BA636A7CF14B6442F424D8067C5122F1B0D6409583E7EA0
    Key-Arg   : None
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    Start Time: 1552801551
    Timeout   : 300 (sec)
    Verify return code: 0 (ok)
---

HTTP/1.1 400 BAD_REQUEST
Content-Length: 0
Connection: Close

closed

```

#### 我又试了一下这个

`curl https://registry-1.docker.io/v2/ && echo Works || echo Problem`
输出
```shell
curl: (51) SSL: certificate subject name (goldopen.org) does not match target host name 'registry-1.docker.io'
Problem

```
这是因为 curl 访问 https 服务器时，会验证服务器证书的有效性和证书域名与访问域名一致性

解决办法：
* 修改 curl 选项，使其不验证服务器证书
curl_setopt($curl, CURLOPT_SSL_VERIFYHOST, FALSE);
curl_setopt($process, CURLOPT_SSL_VERIFYPEER, FALSE);
* 针对 curl 命令，-k 选项，也可以使其不验证证书
* 保证证书域名与访问的域名一致，因为访问的是一个 IP，在 hosts 文件添加IP域名关系映射，然后使用服务器证书的域名进行访问

### 重启之路
#### 重启docekr引擎，而不用关闭容器
```
修改docker 配置文件
sudo   vim  /etc/docker/daemon.json
添加"live-restore": true选项，比如：
{

    "live-restore": true,

}
```
`sudo   service     docker   restart ` 

#### 重启中出错
```
Job for docker.service failed. See "systemctl status docker.service" and "journalctl -xe" for details.
```
可能跟/etc/docker/daemon.json 文件中错误的配置引起的 有关系 ,我把这个文件删掉了

#### 在运行docker容器时可以加如下参数来保证每次docker服务重启后容器也自动重启：
`docker run  --restart=always`

如果已经启动了则可以使用如下命令：
`docker update --restart=always <CONTAINER ID> `


### 以上都不行，那么还是换中国镜像吧！！！

Docker 中国官方镜像加速可通过 registry.docker-cn.com 访问。目前该镜像库只包含流行的公有镜像，而私有镜像仍需要从美国镜像库中拉取。

#### 方式一:使用以下命令直接从该镜像加速地址进行拉取。
`docker pull registry.docker-cn.com/myname/myrepo:mytag1`
比如

`docker pull registry.docker-cn.com/grafana/grafana`

注:除非您修改了Docker守护进程的–registry-mirror参数,否则您将需要完整地指定官方镜像的名称。例如，library/ubuntu、library/redis、library/nginx。

#### 方式二：通过配置文件的方式
给Docker守护进程配置加速器

通过配置文件启动Docker,修改 /etc/docker/daemon.json 文件并添加上 registry-mirrors 键值。
```shell
sudo vim /etc/docker/daemon.json1

{
 "registry-mirrors": ["https://registry.docker-cn.com"]

也可选用网易的镜像地址：http://hub-mirror.c.163.com

{ 
“registry-mirrors”: [“http://hub-mirror.c.163.com“] 
 }
```
修改保存后，重启 Docker 以使配置生效。

`sudo service docker restart`


### 排除万难，下面开始正式部署该监控项目

##### 部署Influxdb服务

 `docker run -d -p 8086:8086 -v ~/influxdb:/var/lib/influxdb --net "mybridge" --name influxdb registry.docker-cn.com/tutum/influxdb`


进入influxdb容器内部，并执行influx命令：
`docker exec -it influxdb influx`

然后创建数据库test和root用户用于本次试验测试

`CREATE DATABASE "test"`
`CREATE USER "root" WITH PASSWORD 'root' WITH ALL PRIVILEGES`


#### 部署cAdvisor服务

谷歌的cadvisor可以用于收集Docker容器的时序信息，包括容器运行过程中的资源使用情况和性能数据。
运行cadvisor服务

`docker run -d -v /:/rootfs -v /var/run:/var/run -v /sys:/sys -v /var/lib/docker:/var/lib/docker --link=influxdb:influxdb --net "mybridge" --name cadvisor google/cadvisor:v0.27.3  --storage_driver=influxdb --storage_driver_host=influxdb:8086 --storage_driver_db=test --storage_driver_user=root --storage_driver_password=root`

**特别注意项**：

在运行上述docker时，这里有可能两个其他配置项需要添加（CentOS, RHEL需要）：


* --privileged=true


设置为true之后，容器内的root才拥有真正的root权限，可以看到host上的设备，并且可以执行mount；否者容器内的root只是外部的一个普通用户权限。由于cadvisor需要通过socket访问docker守护进程，在CentOs和RHEL系统中需要这个这个选项。


* --volume=/cgroup:/cgroup:ro

对于CentOS和RHEL系统的某些版本（比如CentOS6），cgroup的层级挂在/cgroup目录，所以运行cadvisor时需要额外添加–volume=/cgroup:/cgroup:ro选项。

#### 部署Grafana服务

`docker run -d -p 5000:3000 -v ~/grafana:/var/lib/grafana --link=influxdb:influxdb --net "mybridge" --name grafana grafana/grafana`

用3000:3000报错

```
docker: Error response from daemon: driver failed programming external connectivity on endpoint grafana (4939b60d4efcd97a12f101237a2844ffd435296d988
f2305cdc46a5cccf521c9): Bind for 0.0.0.0:3000 failed: port is already allocated.
```
换成5000:3000，然后又出现如下错误
```
GF_PATHS_DATA='/var/lib/grafana' is not writable.
You may have issues with file permissions, more information here: http://docs.grafana.org/installation/docker/#migration-from-a-previous-version-of-
the-docker-container-to-5-1-or-latermkdir: cannot create directory '/var/lib/grafana/plugins': Permission denied
```
解决办法：给~/grafana目录赋予777权限


### 如何监控

1. 首先登陆grafana界面，如图

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/1.png)

用户名 密码均为admin

2. 在Grafana上有好几个步骤需要做，这里Install Grafana已经完成了，接下来我们需要：

Add data source
Create dashboard
…...
Add Data Source

3. 点击Add data source进入，如图
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/2.png)

我们需要根据实际情况来填写各项内容：
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/3.png)

4. 需要添加仪表盘（Dashboard）

Add Dashboard

点击Add dashboard进入

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/4.png)
这里有很多类型的仪表盘供选择，我们选用最常用的Graph就好
进入之后，点击Panel Title下拉列表，再选择Edit进行编辑即可

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/5.png)

5. 在Edit里面主要的就是需要添加查询的条件，继续看下文

Add Query Editor

查询条件中我们可以选择要监控的指标：

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/6.png)
这里选一个memory usage好了，然后要监控的容器选择grafana自身好了。
当然这里不止可以监控一个指标，也不止可以监控一个容器，更多组合我们只需要在下面并列着一个一个添加query条目就好！

最后我添加了三个监控条件，分别用于监控grafana、influxdb和cadvisor三个容器的memory usage指标，并将其同时显示于图中，怎么样是不是很直观！

![image](https://github.com/sjt157/MarkDownPhotos/raw/master/Docker_grafana/7.png)

### 参考

<https://blog.roncoo.com/article/132750>
