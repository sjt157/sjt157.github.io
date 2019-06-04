---
title: Java之checkForComodification
date: 2019-03-17 10:09:01
tags: Java
categories: Java
---
### 什么时候会发生checkForComodification？
使用增强for循环的话会发生什么：
```java
List<String> userNames = new ArrayList<String>() {{
    add("Hollis");
    add("hollis");
    add("HollisChuang");
    add("H");
}};

for (String userName : userNames) {
    if (userName.equals("Hollis")) {
        userNames.remove(userName);
    }
}

System.out.println(userNames);
```
以上代码，使用增强for循环遍历元素，并尝试删除其中的Hollis字符串元素。运行以上代码，会抛出以下异常：
```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(Unknown Source)
	at java.util.ArrayList$Itr.next(Unknown Source)
	at chapter_1_stackandqueue.a.main(a.java:22)

```

同样的，在增强for循环中使用add方法添加元素，结果也会同样抛出该异常。
之所以会出现这个异常，是因为触发了一个Java集合的错误检测机制——fail-fast 。


### 那么什么是fail-fast？

fail-fast，即快速失败，它是Java集合的一种错误检测机制。当多个线程对集合（非fail-safe的集合类）进行结构上的改变的操作时，有可能会产生fail-fast机制，这个时候就会抛出ConcurrentModificationException（当方法检测到对象的并发修改，但不允许这种修改时就抛出该异常）。
同时需要注意的是，即使不是多线程环境，如果单线程违反了规则，同样也有可能会抛出改异常。
那么，在增强for循环进行元素删除，是如何违反了规则的呢？
要分析这个问题，我们先将增强for循环这个语法糖进行解糖，得到以下代码：

```java
public static void main(String[] args) {
    // 使用ImmutableList初始化一个List
    List<String> userNames = new ArrayList<String>() {{
        add("Hollis");
        add("hollis");
        add("HollisChuang");
        add("H");
    }};

    Iterator iterator = userNames.iterator();
    do
    {
        if(!iterator.hasNext())
            break;
        String userName = (String)iterator.next();
        if(userName.equals("Hollis"))
            userNames.remove(userName);
    } while(true);
    System.out.println(userNames);
}

```
运行以上代码，同样会抛出异常。那么我们就看下checkForComodification方法的代码，看下抛出异常的原因。

### 查看Iterator中的next()方法

因为报错的原因是执行了：String string = (String)iterator.next();这句，那查看Iterator类的代码
```java
//把集合修改的次数（modCount）赋值给expectedModCount记录下来
        int expectedModCount = modCount;    

        public E next() {
            //主要用来判断集合的修改次数是否合法
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }


        //主要用来判断集合的修改次数是否合法
        final void checkForComodification() {
            //modCount代表集合修改的次数（例如：每list.add()一次就加1，list.remove()一次也加1）
            //expectedModCount的值是等于开始遍历集合时的修改的次数
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }

```
checkForComodification（）方法中的modCount与expectedModCount的值不相等就会抛出异常。那么他们两个值分别在哪里被修改了呢？其实modCount的值是在list集合执行add，remove等操作的时候赋值，而expectedModCount是在new Iterator()时，把modCount的值赋给了expectedModCount记录。

### remove/add 做了什么

首先，我们要搞清楚的是，到底modCount和expectedModCount这两个变量都是个什么东西。
通过翻源码，我们可以发现：

* modCount是ArrayList中的一个成员变量。它表示该集合实际被修改的次数。
* expectedModCount 是 ArrayList中的一个内部类——Itr中的成员变量。expectedModCount表示这个迭代器期望该集合被修改的次数。其值是在ArrayList.iterator方法被调用的时候初始化的。只有通过迭代器对集合进行操作，该值才会改变。
* Itr是一个Iterator的实现，使用ArrayList.iterator方法可以获取到的迭代器就是Itr类的实例。

他们之间的关系如下：
```java
class ArrayList{
    private int modCount;
    public void add();
    public void remove();
    private class Itr implements Iterator<E> {
        int expectedModCount = modCount;
    }
    public Iterator<E> iterator() {
        return new Itr();
    }
}

```

再继续看remove源码

```java
  /*
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
可以看到，它只修改了modCount，并没有对expectedModCount做任何操作。

### 简单总结一下
简单总结一下，之所以会抛出ConcurrentModificationException异常，是因为我们的代码中使用了增强for循环，而在增强for循环中，集合遍历是通过iterator进行的，但是元素的add/remove却是直接使用的集合类自己的方法。这就导致iterator在遍历的时候，会发现有一个元素在自己不知不觉的情况下就被删除/添加了，就会抛出一个异常，用来提示用户，可能发生了并发修改！

### 如何正确使用
* 直接使用Iterator提供的remove方法。
```java
    List<String> userNames = new ArrayList<String>() {{
        add("Hollis");
        add("hollis");
        add("HollisChuang");
        add("H");
    }};

    Iterator iterator = userNames.iterator();

    while (iterator.hasNext()) {
        if (iterator.next().equals("Hollis")) {
            iterator.remove();
        }
    }
    System.out.println(userNames);

```
如果直接使用Iterator提供的remove方法，那么就可以修改到expectedModCount的值。那么就不会再抛出异常了

* 使用Java 8中提供的filter过滤

Java 8中可以把集合转换成流，对于流有一种filter操作， 可以对原始 Stream 进行某项测试，通过测试的元素被留下来生成一个新 Stream。
```java
    List<String> userNames = new ArrayList<String>() {{
        add("Hollis");
        add("hollis");
        add("HollisChuang");
        add("H");
    }};

    userNames = userNames.stream().filter(userName -> !userName.equals("Hollis")).collect(Collectors.toList());
    System.out.println(userNames);

```
* 使用增强for循环其实也可以
如果，我们非常确定在一个集合中，某个即将删除的元素只包含一个的话， 比如对Set进行操作，那么其实也是可以使用增强for循环的，只要在删除之后，立刻结束循环体，不要再继续进行遍历就可以了，也就是说不让代码执行到下一次的next方法。
```java
    List<String> userNames = new ArrayList<String>() {{
        add("Hollis");
        add("hollis");
        add("HollisChuang");
        add("H");
    }};

    for (String userName : userNames) {
        if (userName.equals("Hollis")) {
            userNames.remove(userName);
            break;
        }
    }
    System.out.println(userNames);

```
* 直接使用fail-safe的集合类
在Java中，除了一些普通的集合类以外，还有一些采用了fail-safe机制的集合类。这样的集合容器在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。
由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发ConcurrentModificationException。
```java
ConcurrentLinkedDeque<String> userNames = new ConcurrentLinkedDeque<String>() {{
    add("Hollis");
    add("hollis");
    add("HollisChuang");
    add("H");
}};

for (String userName : userNames) {
    if (userName.equals("Hollis")) {
        userNames.remove();
    }
}

```
基于拷贝内容的优点是避免了ConcurrentModificationException，但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。
java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

### 参考
<https://blog.csdn.net/u012987546/article/details/52190851>
<https://juejin.im/post/5c8717ad5188257dda56c381>



