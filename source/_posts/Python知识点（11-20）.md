---
title: Python知识点（11-20）
date: 2018-03-31 13:54:38
tags: Python
categories: Python
---

11. __new__和__init__的区别

 * __new__是一个静态方法,而__init__是一个实例方法.
 * __new__方法会返回一个创建的实例,而__init__什么都不返回.
 * 只有在__new__返回一个cls的实例时后面的__init__才能被调用.
 * 当创建一个新实例时调用__new__,初始化一个实例时用__init__.
 * __metaclass__是创建类时起作用.所以我们可以分别使用__metaclass__,__new__和__init__来分别在类创建,实例创建和实例初始化的时候做一些小手脚.

12. 单例模式的四种方式：
```python
# 使用__new__方法
class Singleton(object):
    @staticmethod
    def __new__(cls, *args, **kwargs):
        if not hasattr(cls, '_instance'):    # 若还没有实例
            orig = super(Singleton, cls)
            cls._instance = orig.__new__(cls, *args, **kwargs)   # 创建一个实例
        return cls._instance     # 返回一个实例

class MyClass(Singleton):
    a = 1

# 共享属性 创建实例时把所有实例的__dict_
 #_指向同一个字典,这样它们具有相同的属性和方法.

class Borg(object):
    _state = {}
    def __new__(cls, *args, **kwargs):
        ob = super(Borg, cls).__new__(cls, *args, **kwargs)
        ob.__dict__ = cls._state
        return ob
class MyClass2(Borg):
    a = 1

# 装饰器版本
def singleton(cls, *args, **kwargs):
    instance = {}
    def getinstance():
        if cls not in instance:
            instance[cls] = cls(*args, **kwargs)
        return instance[cls]
    return getinstance
@singleton()
class MyClass:
    pass

# import方法
# mysingleton.py
class My_Singleton(object):
    def foo(self):
        pass
my_singleton = My_Singleton()
# to use
from mysingleton import my_singleton
my_singleton.foo()

```

13. Python中的作用域: 当 Python 遇到一个变量的话他会按照这样的顺序进行搜索：本地作用域（Local）→当前作用域被嵌入的本地作用域（Enclosing locals）→全局/模块作用域（Global）→内置作用域（Built-in）

14. Lambda 表达式:你在某处就真的只需要一个能做一件事情的函数而已，连它叫什么名字都无关紧要。Lambda 表达式就可以用来做这件事。
```python
a=map(lambda x: x*x, [y for y in range(10)])
print(list(a))


# 这样写不如Lambda，因为有污染环境的函数sq
def sq(x):
    return x*x
a=map(sq, [y for y in range(10)])
print(list(a))

```

15. 我们习以为常的复制就是深复制，即将被复制对象完全再复制一遍作为独立的新个体单独存在。而浅复制并不会产生一个独立的对象单独存在。
```python
# 对于普通对象深浅复制一样，而对于下面这样的复杂对象就不同了
# 就是列表里嵌套列表
import copy
a = [1, 2, 3, 4, ['a', 'b']]
b = a # 赋值，传对象的引用
c = copy.copy(a)  # 浅拷贝 对于['a','b']只是引用
#在浅拷贝中对于子对象，python会把它当作一个公共镜像存储起来，所有对他的复制都被当成一个引用
d = copy.deepcopy(a)  # 深拷贝
print(a is b)   # a和b是同一个object
print(a is c)  # 也不是同一个object
print(a is d) # 可以发现a和d并不是一个object
a.append(5) # 修改对象a
a[4].append('c')
print(b)
print(c)
print(d)

True
False
False
[1, 2, 3, 4, ['a', 'b', 'c'], 5]
[1, 2, 3, 4, ['a', 'b', 'c']]
[1, 2, 3, 4, ['a', 'b']]

```

16. Python垃圾回收机制
 Python GC主要使用引用计数（reference counting）来跟踪和回收垃圾。在引用计数的基础上，通过“标记-清除”（mark and sweep）解决容器对象可能产生的循环引用问题，通过“分代回收”（generation collection）以空间换时间的方法提高垃圾回收效率。
 1 引用计数
 PyObject是每个对象必有的内容，其中ob_refcnt就是做为引用计数。当一个对象有新的引用时，它的ob_refcnt就会增加，当引用它的对象被删除，它的ob_refcnt就会减少.引用计数为0时，该对象生命就结束了。
 优点:简单 实时性
 缺点:维护引用计数消耗资源  循环引用
 2 标记-清除机制
 基本思路是先按需分配，等到没有空闲内存的时候从寄存器和程序栈上的引用出发，遍历以对象为节点、以引用为边构成的图，把所有可以访问到的对象打上标记，然后清扫一遍内存空间，把所有没标记的对象释放。
 3 分代技术
 分代回收的整体思想是：将系统中的所有内存块根据其存活时间划分为不同的集合，每个集合就成为一个“代”，垃圾收集频率随着“代”的存活时间的增大而减小，存活时间通常利用经过几次垃圾回收来度量。
 Python默认定义了三代对象集合，索引数越大，对象存活时间越长。
 举例：
 当某些内存块M经过了3次垃圾收集的清洗之后还存活时，我们就将内存块M划到一个集合A中去，而新分配的内存都划分到集合B中去。当垃圾收集开始工作时，大多数情况都只对集合B进行垃圾回收，而对集合A进行垃圾回收要隔相当长一段时间后才进行，这就使得垃圾收集机制需要处理的内存少了，效率自然就提高了。在这个过程中，集合B中的某些内存块由于存活时间长而会被转移到集合A中，当然，集合A中实际上也存在一些垃圾，这些垃圾的回收会因为这种分代的机制而被延迟。

17. read,readline和readlines
 * read 读取整个文件
 * readline 读取下一行,使用生成器方法
 * readlines 读取整个文件到一个迭代器以供我们遍历

18. 生成器
```python
# 所有你可以用在for...in...语句中的都是可迭代的:比如lists,strings,files...
# 因为这些可迭代的对象你可以随意的读取所以非常方便易用,
# 但是你必须把它们的值放到内存里,当它们有很多值时就会消耗太多的内存.
mylist = [x*x for x in range(3)]
for i in mylist:
    print(i)

# 生成器也是迭代器的一种,但是你只能迭代它们一次.原因很简单,因为它们不是
# 全部存在内存里,它们只在要调用的时候在内存里生成:
# 生成器和迭代器的区别就是用()代替[],还有你不能用for i in mygenerator第二次
# 调用生成器:首先计算0,然后会在内存里丢掉0去计算1,直到计算完4.
mygenerator = (x*x for x in range(3) )
for i in mygenerator:
    print(i)
# 当你的函数要返回一个非常大的集合并且你希望只读一次的话,那么用下面的这个就非常的方便了.
def createGenerator():
    mlist = range(3)
    for i in mlist:
        yield i*i   # 函数运行并没有碰到yeild语句就认为生成器已经为空了.
                   # 原因有可能是循环结束或者没有满足if/else之类的.
mgenerator = createGenerator()
for i in mgenerator:
    print(i)

```

19. Python中调用外部命令
```python
# 这是一个简单的对于给定目录下的目录tar的脚本，有坑！注意输入空格这样的sehll字符！！！！
#!/usr/bin/env python
# a python script to auto backup a directory file by SJT
import os
Directory=raw_input("Please enter directory you want to backup:")
dirs=os.listdir(Directory)
for filename in dirs:
    fulldirfile=os.path.join(Directory,filename)
    if os.path.isdir(fulldirfile):
       os.system("tar -czvf  "+fulldirfile+".tar.gz "+ fulldirfile)

```

20. 对字典进行排序是不可能的,只有把字典转换成另一种方式才能排序.字典本身是无序的,但是像列表元组等其他类型是有序的.所以你需要用一个元组列表来表示排序的字典.
```python
import operator
x = {1:2, 3:4, 4:3, 2:1, 0:0}
sorted_x_by_value = sorted(x.items(), key=operator.itemgetter(1))
print(sorted_x_by_value)
# 结果为[(0, 0), (2, 1), (1, 2), (4, 3), (3, 4)]
sorted_x_by_key = sorted(x.items(), key=operator.itemgetter(0))
print(sorted_x_by_key)
# 结果为[(0, 0), (1, 2), (2, 1), (3, 4), (4, 3)]

```
