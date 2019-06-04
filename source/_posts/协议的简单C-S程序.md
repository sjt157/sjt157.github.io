---
title: 协议的简单C/S程序
date: 2018-04-15 20:10:45
tags: Linux
categories: Linux
---

* C/S
---
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <string.h>
 
#define MAX_LINE    100
 
int main(int argc,char **argv)
{
    struct sockaddr_in sin;
    char buf[MAX_LINE];
    int s_fd;
    int port = 8000;
    char *str = "test string";
    char *serverIP = "127.0.0.1";
    int n;
    if(argc > 1)
    {
        str = argv[1];
    }
     //服务器端填充 sockaddr结构
    bzero(&sin , sizeof(sin));
     
    sin.sin_family = AF_INET;
    inet_pton(AF_INET,serverIP,(void *)&sin.sin_addr);
    sin.sin_port = htons(port);
	
	
	int bReuseaddr=1;
	struct timeval nNetTimeout={0,1000};//1绉?
     
    if((s_fd = socket(AF_INET,SOCK_STREAM,0)) == -1)
    {
        perror("fail to create socket");
        exit(1);
    }
	
	/*if(setsockopt(s_fd,SOL_SOCKET ,SO_REUSEADDR,&bReuseaddr,sizeof(int)) == -1)
    {
        perror("fail to set opt reuseaddr");
        exit(1);
    }
      */
	if(setsockopt(s_fd,SOL_SOCKET ,SO_RCVTIMEO,&nNetTimeout,sizeof(nNetTimeout)) == -1) 
    {
        perror("fail to set opt rcvtimeout");
        exit(1);
    }
	
    if(connect(s_fd,(struct sockaddr *)&sin,sizeof(sin)) == -1)
    {
        perror("fail to connect server");
        exit(1);
    }
     
    n = send(s_fd, str , strlen(str) + 1, 0);
    if(n == -1)
    {
        perror("fail to send");
        exit(1);
    }
     
    n = recv(s_fd ,buf , MAX_LINE, 0);
    if(n == -1)
    {
        perror("fail to recv");
        exit(1);
    }
    printf("the length of str = %s\n" , buf);
    if(close(s_fd) == -1)
    {
        perror("fail to close");
        exit(1);
    }
    return 0;
}
```
* server
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <ctype.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
 
#define INET_ADDR_STR_LEN   1024
#define MAX_LINE 100
 
int main(int argc,char **argv)
{
    struct sockaddr_in sin;
    struct sockaddr_in cin;
    int l_fd;
    int c_fd;
    socklen_t len;
    char buf[MAX_LINE];
    char addr_p[INET_ADDR_STR_LEN];
    int port = 8000;
    int n;
    bzero(&sin , sizeof(sin));
     
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = INADDR_ANY;
    sin.sin_port = htons(port);
    
	//参数设置
	int bReuseaddr=1;
	struct timeval nNetTimeout={0,10000};//10秒
	
    if((l_fd = socket(AF_INET,SOCK_STREAM,0)) == -1)
    {
        perror("fail to create socket");
        exit(1);
    }
	
	if(setsockopt(l_fd,SOL_SOCKET ,SO_REUSEADDR,&bReuseaddr,sizeof(int)) == -1)
    {
        perror("fail to set opt reuseaddr");
        exit(1);
    }
	if(setsockopt(l_fd,SOL_SOCKET ,SO_RCVTIMEO,&nNetTimeout,sizeof(nNetTimeout)) == -1) 
    {
        perror("fail to set opt rcvtimeout");
        exit(1);
    }
	
    if(bind(l_fd,(struct sockaddr *)&sin ,sizeof(sin) ) == -1) 
    {
        perror("fail to bind");
        exit(1);
    }
    if(listen(l_fd,10) == -1)
    {
        perror("fail to listen");
        exit(1);
    }
    printf("waiting.....\n");
    while(1)
    {
        //if((c_fd = accept(l_fd,(struct sockaddr *)&cin, &len)) == -1)
		if((c_fd = accept(l_fd,NULL, 0)) == -1)
        {
     //      perror("fail to accept");
     //       exit(1);
	   continue;
        }
         
        n = recv(c_fd , buf, MAX_LINE, 0);
        if(n == -1)
        {
            perror("fail to recv");
            exit(1);
        }
        else if(n == 0)
        {
            printf("the connect has been closed\n");
            close(c_fd);
            continue;
        }
        //inet_ntop(AF_INET,&cin.sin_addr,addr_p,sizeof(addr_p));
        printf("content is : %s\n",buf);
         
        n = strlen(buf);
        sprintf(buf,"%d",n);
         
        n = send(c_fd , buf, sizeof(buf) + 1 , 0);
        if( n == -1)
        {
            perror("fail to send");
            exit(1);
        }
        if(close(c_fd) == -1)
        {
            perror("fail to close");
            exit(1);
        }
    }
    if(close(l_fd) == -1)
    {
        perror("fail to close");
        exit(1);
    }
    return 0;
}

```
---
* linux下基于简单socket编程实现C/S
```c
#include <stdio.h>
#include <stdlib.h>
#include <netinet/in.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
 
#define MAX_LINE 1024
#define INET_ADDR_STR 16 
 
 
void my_fun(char *p)
{
    if(p == NULL)
    {
        return;
    }
    for( ; *p != '\0' ; p++)
    {
        if((*p >= 'a') && (*p <= 'z'))
        {
            *p = *p - 32;
        }
    }
    return;
}
int main(int argc,char **argv)
{
    struct sockaddr_in sin;     //服务器通信地址结构
    struct sockaddr_in cin;     //保存客户端通信地址结构
    int l_fd;
    int c_fd;
    socklen_t len;
    char buf[MAX_LINE];     //存储传送内容的缓冲区
    char addr_p[INET_ADDR_STR]; //存储客户端地址的缓冲区
    int port = 8000;
    int n;
    bzero((void *)&sin,sizeof(sin));
    sin.sin_family = AF_INET;   //使用IPV4通信域
    sin.sin_addr.s_addr = INADDR_ANY;   //服务器可以接受任意地址
    sin.sin_port = htons(port); //端口转换为网络字节序
     
    l_fd = socket(AF_INET,SOCK_STREAM,0);   //创建套接子,使用TCP协议
    bind(l_fd,(struct sockaddr *)&sin,sizeof(sin));
     
    listen(l_fd,10);    //开始监听连接
     
    printf("waiting ....\n");
    while(1)
    {
        c_fd = accept(l_fd,(struct sockaddr *)&cin,&len);
         
        n = read(c_fd,buf,MAX_LINE);    //读取客户端发送来的信息
        inet_ntop(AF_INET,&cin.sin_addr,addr_p,INET_ADDR_STR);      //将客户端传来地址转化为字符串
        printf("client IP is %s,port is %d\n",addr_p,ntohs(cin.sin_port));
        printf("content is : %s\n", buf);   //打印客户端发送过来的数据
        my_fun(buf);
        write(c_fd,buf,n);          //转换后发给客户端
 
        close(c_fd);
    }
    printf("buf = %s\n",buf);
    if((close(l_fd)) == -1)
    {
        perror("fail to close\n");
        exit(1);
    }
    return 0;
}
```

```c

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <netinet/in.h>
 
 
#define MAX_LINE 1024
 
int main(int argc,char **argv)
{
    struct sockaddr_in sin;     //服务器的地址
    char buf[MAX_LINE];
    int sfd;
    int port = 8000;
    char *str = "test string";
    char *serverIP = "127.0.0.1";
    if(argc > 1)
    {
        str = argv[1];  //读取用户输入的字符串
    }
    bzero((void *)&sin,sizeof(sin));
    sin.sin_family = AF_INET;   //使用IPV4地址族
     
    inet_pton(AF_INET,serverIP,(void *)&(sin.sin_addr));
    sin.sin_port = htons(port);
    /*理论上建立socket时是指定协议，应该用PF_xxxx，设置地址时应该用AF_xxxx。当然AF_INET和PF_INET的值是相同的，混用也不会有太大的问题。也就是说你socket时候用PF_xxxx，设置的时候用AF_xxxx也是没关系的，这点随便找个TCPIP例子就可以验证出来了。如下，不论是AF_INET还是PF_INET都是可行的，只不过这样子的话，有点不符合规范。*/  
    sfd = socket(AF_INET,SOCK_STREAM,0);
   
     
    connect(sfd,(struct sockaddr *)&(sin),sizeof(sin));
 
    printf("str = %s\n" , str);
    write(sfd , str , strlen(str) + 1);
    read(sfd , buf , MAX_LINE);
    printf("recive from server: %s\n" , buf);
 
    close(sfd);
     
    return 0;
}
```

