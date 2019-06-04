---
title: 国密算法之SM4
date: 2018-12-12 19:24:55
tags: TrustComputing
categories: TrustComputing
---

## 为什么用SM4加密后文件会略长于明文文件？

* 首先了解什么是SM4算法
* 参考：
1. <https://blog.csdn.net/Soul_Programmer_Swh/article/details/80263822>
2. <https://www.cnblogs.com/TaiYangXiManYouZhe/p/4317519.html>
3. <https://blog.csdn.net/cg129054036/article/details/83016958>

 原因就在于sm4 算法加解密时，采用的是 TCM_ES_SM4_CBC 模式，该模式会填 充加密数据以保证其长度为 16 的整数倍，因此加密后文件会略长于明文文件，解 密后文件长度将恢复。

* 那么什么是又CBC？
在密码学中，分组加密（英语：Block cipher），又称分块加密或块密码，是一种对称密钥算法。它将明文分成多个等长的模块（block），使用确定的算法和对称密钥对每组分别加密解密。分组加密是极其重要的加密协议组成，其中典型的如DES和AES作为美国政府核定的标准加密算法，应用领域从电子邮件加密到银行交易转帐，非常广泛。
现代分组加密创建在迭代的思想上产生密文。其思想由克劳德·香农在他1949年的重要论文《保密系统的通信理论》（Communication Theory of Secrecy Systems）中提出，作为一种通过简单操作如替代和排列等以有效改善保密性的方法。[1] 迭代产生的密文在每一轮加密中使用不同的子密钥，而子密钥生成自原始密钥。
DES加密在1977年由前美国国家标准局（今“美国国家标准与技术研究所”）颁布，是现代分组加密设计的基础思想。它同样也影响了密码分析的学术进展。

<https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F#%E5%AF%86%E7%A0%81%E5%9D%97%E9%93%BE%E6%8E%A5%EF%BC%88CBC%EF%BC%89>


## SM9算法
* SM9标识密码算法是由国密局发布的一种IBE(Identity-Based Encryption)算法。IBE算法以用户的身份标识作为公钥，不依赖于数字证书。

## IBC(基于标示的密码技术)
* 基于证书的公钥体制在应用中面临诸多问题，特别是证书使用过程的复杂性使得不具备相关知识的普通用户难以驾驭。为了降低公钥系统中密钥管理和使用的复杂性，Shamir在1984[S84]年提出了基于标识的密码技术(Identity-Based Cryptography - IBC)：即用户的标识就可以用做用户的公钥（更加准确地说是用户的公钥可以从用户的标识和系统指定的一个方法计算得出）。
* 在这种情况下，用户不需要申请和交换证书，从而极大地简化了密码系统管理的复杂性。用户的私钥由系统中的一受信任的第三方（密钥生成中心）使用标识私钥生成算法计算生成。这样的系统具有天然的密码委托功能，适合于有监管的应用环境。

##  NTLS(下一代安全传输协议)
![image](https://github.com/sjt157/MarkDownPhotos/raw/master/TrustComputing/10.png)
