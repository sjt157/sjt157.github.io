---
title: Python知识点（21-30）
date: 2018-04-01 14:44:29
tags: Python
categories: Python
---

21. super函数
```python
class T(object):
    a = 0
class A(T):
    pass
class B(T):
    a = 2
class D(T):
    a = 3
class C(A, D, B):
    pass

c = C()
print(super(C, c).a)  # c继承的哪个值是根据MRO顺序来的，按照广度优先，根据ADB的顺序找到第一个定义a的类
# 然后就是它，所以super(C,c).a 是3
print(c.a)
```
22. 在循环中获取索引
```python
# 在循环中获取索引(数组下标),用enumerate
ints = [8, 23, 45 ,12, 78]
for idx, val in enumerate(ints):
    print(idx, val)
```

23. 如何移除换行符?`'test string\n'.rstrip()`

24. 合并列表中的列表,一共有三种方法，用列表推导式最快 原因：当有L个子串的时候用+(即sum)的时间复杂度是O(L**2)--每次迭代的时候作为中间结果的列表的长度就会越来越长,而且前一个中间结果的所有项都会再拷贝一遍给下一个中间结果.所以当你的列表l含有L个字串:l列表的第一项需要拷贝L-1次,而第二项要拷贝L-2次,以此类推;所以总数为I * (L**2)/2.列表推导式(list comprehension)只是生成一个列表,每次运行只拷贝一次(从开始的地方拷贝到最终结果).
```python
$ python -mtimeit -s'l=[[1,2,3],[4,5,6], [7], [8,9]]*99' '[item for sublist in l for item in sublist]'
10000 loops, best of 3: 143 usec per loop
$ python -mtimeit -s'l=[[1,2,3],[4,5,6], [7], [8,9]]*99' 'sum(l, [])'
1000 loops, best of 3: 969 usec per loop
$ python -mtimeit -s'l=[[1,2,3],[4,5,6], [7], [8,9]]*99' 'reduce(lambda x,y: x+y,l)'
1000 loops, best of 3: 1.1 msec per loop
```

25. 反转字符串 ` 'hello world'[::-1]`
26. 通常每一个实例x都会有一个__dict__属性，用来记录实例中所有的属性和方法，也是通过这个字典，
可以让实例绑定任意的属性。而__slots__属性作用就是，当类C有比较少的变量，而且拥有__slots__属性时，
类C的实例 就没有__dict__属性，而是把变量的值存在一个固定的地方。如果试图访问一个__slots__中没有
的属性，实例就会报错。这样操作有什么好处呢？__slots__属性虽然令实例失去了绑定任意属性的便利，
但是因为每一个实例没有__dict__属性，却能有效节省每一个实例的内存消耗，有利于生成小而精
干的实例。 为什么需要这样的设计呢？
在一个实际的企业级应用中，当一个类生成上百万个实例时，即使一个实例节省几十个字节都可以节省
一大笔内存，这种情况就值得使用__slots__属性。
