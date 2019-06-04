---
title: Python知识点（1-10）
date: 2018-03-30 14:32:59
tags: Python
categories: Python
---

### python知识点
1. 在python中，strings, tuples, 和numbers是不可更改的对象，而list,dict等则是可以修改的对象。当一个引用传递给函数的时候,函数自动复制一份引用,这个函数里的引用和外边的引用没有半毛关系了.所以第一个例子里函数把引用指向了一个不可变对象,当函数返回的时候,外面的引用没半毛感觉.而第二个例子就不一样了,函数内的引用指向的是可变对象,对它的操作就和定位了指针地址一样,在内存里进行修改.
```python
a = 1
b = []
c = []
 #If you pass an immutable object to a method, you still can't rebind 
 #the outer reference, and you can't even mutate the object.
def fun(a):
    a = 2
 #If you pass a mutable object into a method,
 #the method gets a reference to that same object
 #and you can mutate it
def fun2(b):
    b.append(1)
 #if you rebind the reference in the method, the outer scope
 #will know nothing about it, and after you're done,
 #the outer reference will still point at the original object.
def fun3(c):
    c = [1]
fun(a)
fun2(b)
fun3(c)
print(a)  #1
print(b)   #[1]
print(c)   #[]

```

2. 元类（metaclass），经常用在ORM这种复杂的结构！
  参考[深刻理解Python中的元类(metaclass)](http://python.jobbole.com/21351/)

3. 在类里每次定义方法的时候都需要绑定这个实例,就是foo(self, x),为什么要这么做呢?因为实例方法的调用离不开实例,我们需要把实例自己传给函数,调用的时候是这样的a.foo(x)(其实是foo(a, x)).类方法一样,只不过它传递的是类而不是实例,A.class_foo(x).注意这里的self和cls可以替换别的参数,但是python的约定是这俩,还是不要改的好.
```python
x = 1
def foo(x):
    print("executing foo(%s)" % (x))
class A(object):
    # self和cls是对类或者实例的绑定
    def foo(self, x):
        print("executing foo(%s,%s)" % (self, x))
    @classmethod
    def class_foo(cls, x):
        print("executing class_foo(%s,%s)" % (cls, x))
    @staticmethod
    def static_foo(x):
        print("executing static_foo(%s)" % x)
a = A()
# 普通方法的调用
foo(x)
# 实例方法的调用
a.foo(x)
# 类方法的调用
a.class_foo(x)
A.class_foo(x)
# 静态方法的调用
a.static_foo(x)  
A.static_foo(x)

 # 程序结果为：
executing foo(1)
executing foo(<__main__.A object at 0x0000000002A5A518>,1)
executing class_foo(<class '__main__.A'>,1)
executing class_foo(<class '__main__.A'>,1)
executing static_foo(1)
executing static_foo(1)
```
4. 类变量和实例变量:类变量就是供类使用的变量,实例变量就是供实例使用的.这里p1.name="bbb"是实例调用了类变量,这其实和上面第一个问题一样,就是函数传参的问题,p1.name一开始是指向的类变量name="aaa",但是在实例的作用域里把类变量的引用改变了,就变成了一个实例变量,self.name不再引用Person的类变量name了.
```python

	class Person:
    name="aaa"
 
p1=Person()
p2=Person()
p1.name="bbb"
print p1.name  # bbb
print p2.name  # aaa
print Person.name  # aaa

```
 类变量就是供类使用的变量,实例变量就是供实例使用的. 这里p1.name="bbb"是实例调用了类变量,这其实和上面第一个问题一样,就是函数传参的问题,p1.name一开始是指向的类变量name="aaa",但是在实例的作用域里把类变量的引用改变了,就变成了一个实例变量,self.name不再引用Person的类变量name了. 可以看看下面的例子:
```python
    class Person:
    name=[]
 
p1=Person()
p2=Person()
p1.name.append(1)
print p1.name  # [1]
print p2.name  # [1]
print Person.name  # [1]
```

5. Python自省
   自省就是面向对象的语言所写的程序在运行时,所能知道对象的类型.简单一句就是运行时能够获得对象的类型.比如type(),dir(),getattr(),hasattr(),isinstance().
6. python共有三种推导式，那什么是推导式呢？推导式是可以从一个数据序列构建另一个新的数据序列的结构体。
```python
 #  列表推导式
multiples = [i for i in range(30) if i % 3 is 0]
print(multiples)
# Output: [0, 3, 6, 9, 12, 15, 18, 21, 24, 27]


def squared(x):
    return x*x
multiples = [squared(i) for i in range(30) if i % 3 is 0]
print multiples
#  Output: [0, 9, 36, 81, 144, 225, 324, 441, 576, 729]

#  将俩表推导式的[]改成()即可得到生成器。

multiples = (i for i in range(30) if i % 3 is 0)
print(type(multiples))
#  Output: <type 'generator'>

 # 字典推导式
 # 字典推导和列表推导的使用方法是类似的，只不中括号该改成大括号。直接举例说明：
  # 通过把key大小写合并 合并值
mcase = {'a': 10, 'b': 34, 'A': 7, 'Z': 3}
mcase_frequency = {
    k.lower(): mcase.get(k.lower(), 0) + mcase.get(k.upper(), 0)
    for k in mcase.keys()
    if k.lower() in ['a','b']
}
print(mcase_frequency)
#  Output: {'a': 17, 'b': 34}
 
# 快速更换key和value
mcase = {'a': 10, 'b': 34, 'c': 30}
mcase_frequency = {v: k for k, v in mcase.items()}
print(mcase_frequency)
#  Output: {10: 'a', 34: 'b'}

 #  集合推导式
squared = {x**2 for x in [1, 1, 2]}
print(squared)
# Output: set([1, 4])

```
7. 单下划线和双下划线
 首先是单下划线开头，这个被常用于模块中，在一个模块中以单下划线开头的变量和函数被默认当作内部函数，如果使用 from a_module import * 导入时，这部分变量和函数不会被导入。不过值得注意的是，如果使用 import a_module 这样导入模块，仍然可以用 a_module._some_var 这样的形式访问到这样的对象。在 Python 的官方推荐的代码样式中，还有一种单下划线结尾的样式，这在解析时并没有特别的含义，但通常用于和 Python 关键词区分开来，比如如果我们需要一个变量叫做 class，但 class 是 Python 的关键词，就可以以单下划线结尾写作 class_。双下划线开头的命名形式在 Python 的类成员中使用表示名字改编 (Name Mangling)，即如果有一 Test 类里有一成员 __x，那么 dir(Test) 时会看到 _Test__x 而非 __x。这是为了避免该成员的名称与子类中的名称冲突。但要注意这要求该名称末尾没有下划线。双下划线开头双下划线结尾的是一些 Python 的“魔术”对象，如类成员的 __init__、__del__、__add__、__getitem__ 等，以及全局的 __file__、__name__ 等。 Python 官方推荐永远不要将这样的命名方式应用于自己的变量或函数，而是按照文档说明来使用。另外单下划线开头还有一种一般不会用到的情况在于使用一个 C 编写的扩展库有时会用下划线开头命名，然后使用一个去掉下划线的 Python 模块进行包装。如 struct 这个模块实际上是 C 模块 _struct 的一个 Python 包装。
```python
class MyClass():
    def __init__(self):
        self.__superprivate = "Hello"
        self._semiprivate = ",World！"
class OtherClass(MyClass):
    def __init__(self):
        self.__superprivate = "hhaaa"
        self.b = "ssss"
mc = MyClass()
oc = OtherClass()
print(mc._semiprivate) # 只有mc能访问该私有变量
print(mc.__dict__)
print(oc.__dict__)
# print(oc.__superprivate)  # 访问不了，提示没有该属性
print(oc._OtherClass__superprivate)  # 这样就可以访问
# ,World！
#{'_MyClass__superprivate': 'Hello', #'_semiprivate': ',World！'}
#{'b': 'ssss', '_OtherClass__superprivate': #'hhaaa'}
#hhaaa
```
8. 当你不确定你的函数里将要传递多少参数时你可以用*args   **kwargs允许你使用没有事先定义的参数名
9. 面向切面编程AOP和装饰器
  装饰器是一个很著名的设计模式，经常被用于有切面需求的场景，较为经典的有插入日志、性能测试、事务处理等。装饰器是解决这类问题的绝佳设计，有了装饰器，我们就可以抽离出大量函数中与函数功能本身无关的雷同代码并继续重用。概括的讲，装饰器的作用就是为已经存在的对象添加额外的功能。
```python
# 字体变粗装饰器
def makebold(fn):
    # 装饰器将返回新的函数
    def warpper():
        return "<b>"+ fn() + "</b>"
    return warpper
@makebold
def say():
    return "hello"
print(say())   # <b>hello</b>
```
10. 函数重载主要是为了解决两个问题。可变参数类型。可变参数个数。一个基本的设计原则是，仅仅当两个函数除了参数类型和参数个数不同以外，其功能是完全相同的，此时才使用函数重载，如果两个函数的功能其实不同，那么不应当使用重载，而应当使用一个名字不同的函数。但是因为python可以接受任何类型的参数、和任何数量的参数，所以不需要重载。



