---
title: 使用travis-ci自动部署github上的项目
date: 2019-02-03 16:22:32
tags: TravisCI
categories: TravisCI
---


1. travis-ci账户
用github账户登录，找到对应项目进行勾选，如下图。

2. 生成token，如下图。
3. 配置token，如下图。
（配置私密的环境变量时一定要加密，因为会显示在日志中且能够被他人看到）

4. 还需要添加一些环境变量使起更方便(地址别填错了)，如下图。

###有个问题：
hexo g 之后并没有在travis中看到相关信息，难道非要用git推送吗？还有.travis.yml应该放在哪，我现在放在了blogs目录下。

答案：项目应该有两个分支，一个源码分支，一个发布之后的分支，.travis.yml应该放在源码分支的根目录下。

### 参考
<https://www.cnblogs.com/morang/p/7228488.html>
<https://blog.csdn.net/woblog/article/details/51319364>
<http://lujiahao.tk/2018/06/27/Travis%20CI%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90/>


### 为博客添加一个新分支hexo-source，并设置为默认分支，这样就是用两个分支进行管理了。
执行git add .、git commit -m ""、git push origin hexo来提交hexo网站源文件
依次执行hexo g和hexo d生成静态网页部署至Github上

### hexo备份
参考:

<https://www.jianshu.com/p/57b5a384f234>

<https://www.jianshu.com/p/a27e9761ecf3>
