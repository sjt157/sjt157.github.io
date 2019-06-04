---
title: BloomFilter
date: 2018-09-14 20:57:40
tags: DataStructure
categories: DataStructure
---
## 简介
BloomFilter（布隆过滤器）是一种可以高效地判断元素是否在某个集合中的算法。

在很多日常场景中，都大量存在着布隆过滤器的应用。例如：检查单词是否拼写正确、网络爬虫的URL去重、黑名单检验，微博中昵称不能重复的检测。在工业界中，Google著名的分布式数据库BigTable也用了布隆过滤器来查找不存在的行或列，以减少磁盘查找的IO次数；Google Chrome浏览器使用BloomFilter来判断一个网站是否为恶意网站。

对于以上场景，可能很多人会说，用HashSet甚至简单的链表、数组做存储，然后判断是否存在不就可以了吗？

当然，对于少量数据来说，HashSet是很好的选择。但是对于海量数据来说，BloomFilter相比于其他数据结构在空间效率和时间效率方面都有着明显的优势。

但是，布隆过滤器具有一定的误判率，有可能会将本不存在的元素判定为存在。因此，对于那些需要“零错误”的应用场景，布隆过滤器将不太适用。具体的原因将会在第二部分中介绍。

在本文的第二部分，本文将会介绍BloomFilter的基本算法思想；第三部分将会基于Google开源库Guava来讲解BloomFilter的具体实现；在第四部分中，将会介绍一些开源的BloomFilter的扩展，以解决目前BloomFilter的不足。

## 算法讲述
布隆过滤器是基于Hash来实现的，在学习BloomFilter之前，也需要对Hash的原理有基本的了解。个人认为，BloomFilter的总体思想实际上和bitmap很像，但是比bitmap更节省空间，误判率也更低。

BloomFilter的整体思想并不复杂，主要是使用k个Hash函数将元素映射到位向量的k个位置上面，并将这k个位置全部置为1。当查找某元素是否存在时，查找该元素所对应的k位是否全部为1即可说明该元素是否存在。

## 缺点
BloomFilter 由于并不存储元素，而是用位的01来表示元素是否存在，并且很有可能一个位时被多个元素同时使用。所以无法通过将某元素对应的位置为0来删除元素。

幸运的是，目前学术界和工业界都有很多方法扩展已解决以上问题。

```c
#include "bloomfilter.h"
#include "hashs.h"
//#include "md5.h" 
#include "crc32.h"
//#include "sha1.h"
#include <stdlib.h>
#include <pthread.h>

#define HASH_FUNC_NUM 8
#define BLOOM_SIZE 1000000
#define BITSIZE_PER_BLOOM  32
#define LIMIT   (BLOOM_SIZE * BITSIZE_PER_BLOOM)

/* 
 * m=10n, k=8 when e=0.01 (m is bitsize, n is inputnum, k is hash_func num, e is error rate)
 * here m = BLOOM_SIZE*BITSIZE_PER_BLOOM = 32,000,000 (bits)
 * so n = m/10 = 3,200,000 (urls)
 * enough for crawling a website
 */
static int bloom_table[BLOOM_SIZE] = {0};
pthread_mutex_t bt_lock = PTHREAD_MUTEX_INITIALIZER;

//static MD5_CTX md5;
//static SHA1_CONTEXT sha;  

static unsigned int encrypt(char *key, unsigned int id)
{
    unsigned int val = 0;

    switch(id){
        case 0:
            val = times33(key); break;
        case 1:
            val = timesnum(key,31); break;
        case 2:
            val = aphash(key); break;
        case 3:
            val = hash16777619(key); break;
        case 4:
            val = mysqlhash(key); break;
        case 5:
            //basically multithreads supported
            val = crc32((unsigned char *)key, strlen(key));
            break;
        case 6:
            val = timesnum(key,131); break;
            /*
               int i;
               unsigned char decrypt[16];
               MD5Init(&md5);
               MD5Update(&md5, (unsigned char *)key, strlen(key));
               MD5Final(&md5, decrypt);
               for(i = 0; i < 16; i++)
               val = (val << 5) + val + decrypt[i];
               break;
               */
        case 7:
            val = timesnum(key,1313); break;
            /*
               sha1_init(&sha);  
               sha1_write(&sha, (unsigned char *)key, strlen(key));
               sha1_final(&sha);
               for (i=0; i < 20; i++)  
               val = (val << 5) + val + sha.buf[i];
               break;
               */
        default:
            // should not be here
            abort();
    }
    return val;
}

int search(char *url)
{
    unsigned int h, i, index, pos;
    int res = 0;

    pthread_mutex_lock(&bt_lock);

    for (i = 0; i < HASH_FUNC_NUM; i++) {
        h = encrypt(url, i);
        h %= LIMIT;
        index = h / BITSIZE_PER_BLOOM;
        pos = h % BITSIZE_PER_BLOOM;
        if (bloom_table[index] & (0x80000000 >> pos))
            res++;
        else
            bloom_table[index] |= (0x80000000 >> pos);
    }

    pthread_mutex_unlock(&bt_lock);

    return (res == HASH_FUNC_NUM);
}
             
```
