---
title: python中的eval函数
date: 2018-03-29 21:57:32
tags: Python
categories: Python
---

## python中的eval()

* **这个函数意义在哪？**
在编译语言里要动态地产生代码，基本上是不可能的，但动态语言是可以，意味着软件已经部署到服务器上了，但只要作很少的更改，只好直接修改这部分的代码，就可立即实现变化，不用整个软件重新加载。

先举个栗子：
```
   a = 1
   g = {'a': 20}
   print(eval("a+1", g))  #返回21
```

再举个栗子
```
# test eval() and locals()
x = 1
y = 1
num1 = eval("x+y")
print(num1)


def g():
x = 2
y = 2
num3 = eval("x+y")
print(num3)
num2 = eval("x+y", globals())  # 搜全局
num4 = eval("x+y",globals(),locals())  # 先搜索局部，搜索到停止
print(num2)
print(num4)
g()   # 值依次为2 4 2 4
```
* **eval在字符串对象和list、dictinoary、tuple对象之间互相转换**
```
def evala():
    l = '[1,2,3,4,[5,6,7,8,9]]'
    d = "{'a':123,'b':456,'c':789}"
    t = '([1,3,5],[5,6,7,8,9],[123,456,789])'

    print(type(l), type(eval(l)))
    print(type(d), type(eval(d)))
    print(type(t), type(eval(t)))
evala()
```
结果为：
```
	<class 'str'> <class 'list'>
    <class 'str'> <class 'dict'>
    <class 'str'> <class 'tuple'>

```
* **locals()对象的值不能修改，globals()对象的值可以修改**
```
#test globals() and locals()

z=0
def f():    
    z = 1    
    print (locals())        
    locals()["z"] = 2    
    print (locals())      
f() 
globals()["z"] = 2
print (z)  # 结果为{'z': 1}  {'z': 1}   2

```
* **eval有安全性问题,比如用户恶意输入就会获得当前目录文件**
`eval("__import__('os').system('dir')")`

* **怎么避免安全问题？**
1. 自行写检查函数；
2. 使用ast.literal_eval
===

**tips**
`if __name__ == "__main__":` 第一开始不是很理解，今天经过学习知道了这句代码的作用。是编写私有化部分 ，这句代码以上的部分，可以被其它的调用，以下的部分只有这个文件自己可以看见，如果文件被调用了，其他人是无法看见私有化部分的。比如进行单元测试的时候会用到！他的原理是每个py文件都有__name__属性，如果当前属性和__main__一样，则值为真，否则为假，就不执行！
