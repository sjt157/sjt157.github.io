---
title: python之闭包
date: 2018-04-17 17:23:05
tags: Python
categories: Python
---


* 无意间，看到这么一道Python面试题：以下代码将输出什么？
```python
def Fun():
    temp = [lambda x:i*x for i in range(4)]
    return temp
for everyLambda in Fun():
    print(everyLambda(2))
```
不是0,2,4,6，而是6,6,6,6

* Python 的闭包的后期绑定导致的 late binding，这意味着在闭包中的变量是在内部函数被调用的时候被查找。因为在 for 里面 i 的值是不断改写的，但是 lambda 里面只是储存了 i 的符号，调用的时候再查找。所以结果是，当任何 testFun() 返回的函数被调用，在那时，i 的值是在它被调用时的周围作用域中查找，到那时，无论哪个返回的函数被调用，for 循环都已经完成了，i 最后的值是 3，因此，每个返回的函数 testFun 的值都是 3。因此一个等于 2 的值被传递进以上代码，它们将返回一个值 6 （比如： 3 x 2）。

* 那如何才能是0,2,4,6呢？
1. 创建一个闭包，通过使用默认参数立即绑定它的参数。为什么你加了默认参数就成功了呢？因为在创建函数的时候就要获取默认参数的值，放到 lambda 的环境中，所以这里相当于存在一个赋值，从而 lambda 函数环境中有了一个独立的 i。
```python
  def Fun():
    temp = [lambda x,i=i:i*x for i in range(4)]
    return temp
for everyLambda in Fun():
    print(everyLambda(2))
```
2. 使用functools.partial 函数，把函数的某些参数（不管有没有默认值）给固定住（也就是相当于设置默认值）
```python
  from functools import partial
from operator import mul
def Fun():
    return [partial(mul,i) for i in range(4)]
for everyLambda in Fun():
    print(everyLambda(2))
```
3. 用生成器
```python
  def Fun():
    return (lambda x,i=i:i*x for i in range(4))
for everyLambda in Fun():
    print(everyLambda(2))
```
4. 利用yield的惰性求值的思想
```python
  def Fun():
    for i in range(4):
        yield lambda x: i*x
for everyLambda in Fun():
    print(everyLambda(2))
```
