---
title: python之intern机制
date: 2018-04-08 18:21:26
tags: Python
categories: Python
---

## python之intern机制(字符串驻留机制)
* 在主流面向对象的编程语言中intern 机制对于处理字符串已经成为一种标配，通过 intern 机制可以提高字符串的处理效率，当然，解释器内部很对 intern 机制的使用策略是有考究的，有些场景会自动使用 intern ，有些地方需要通过手动方式才能启动。python中如果创建的对象一样，那么就会自动启动intern机制，所有对象会指向同一个地址。先看个栗子：
```python
a = "helloworld"
b = "helloworld"
c = "helloworld"
print(id(a))
print(id(b))
print(id(c))  # 三个对象的内存地址一样
```
其实这就是python的intern机制，这样可以节省内存。看一下源码
```c
static PyObject *interned;
void PyString_InternInPlace(PyObject **p)
{
    register PyStringObject *s = (PyStringObject *)(*p);
    PyObject *t;
    if (s == NULL || !PyString_Check(s))
        Py_FatalError("PyString_InternInPlace: strings only please!");
    /* If it's a string subclass, we don't really know what putting
       it in the interned dict might do. */
    if (!PyString_CheckExact(s))
        return;
    if (PyString_CHECK_INTERNED(s))
        return;
    if (interned == NULL) {
        interned = PyDict_New();
        if (interned == NULL) {
            PyErr_Clear(); /* Don't leave an exception */
            return;
        }
    }
    t = PyDict_GetItem(interned, (PyObject *)s);
    if (t) {
        Py_INCREF(t);
        Py_DECREF(*p);
        *p = t;
        return;
    }

    if (PyDict_SetItem(interned, (PyObject *)s, (PyObject *)s) < 0) {
        PyErr_Clear();
        return;
    }
    /* The two references in interned are not counted by refcnt.
       The string deallocator will take care of this */
    Py_REFCNT(s) -= 2;
    PyString_CHECK_INTERNED(s) = SSTATE_INTERNED_MORTAL;
}
```
* 可以看到interned的定义是一个PyObject,但从下面的代码可以看出，在interned=nul的时候，interned = PyDict_New();所以它实际上是一个PyDictObject，我们可以暂时理解为c++里面的map对象。对一个PyStringObject对象进行intern机制处理的时候，会通过PyDict_GetItem去从Interned对象中查找有没有一样的已经创建的对象，有的话就直接拿来用，没有的话就说明这种对象是第一次创建，用PyDict_SetItem函数把相应的信息存到interned里面，下次再创建一样的就能从中找到了。
* Python解释器会缓冲256个字符串, 第257个字符串多次赋值不同的变量名, id()查看的结果就不同了。
* intern机制的优点是。须要值同样的字符串的时候（比方标识符）。直接从池里拿来用。避免频繁的创建和销毁。提升效率，节约内存。缺点是，拼接字符串、对字符串改动之类的影响性能。

由于是不可变的。所以对字符串改动不是inplace操作。要新建对象。

这也是为什么拼接多字符串的时候不建议用+而用join()。join()是先计算出全部字符串的长度，然后一一拷贝，仅仅new一次对象。

须要小心的坑。并非全部的字符串都会採用intern机制。仅仅包括下划线、数字、字母的字符串才会被intern。





