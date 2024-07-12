---
layout: post
title: 关于 Java 内存泄漏介绍
categories: [Java]
description: 内存泄漏的定义：对象不再被应用程序使用，但垃圾回收器却不能移除它们，因为它们还在被引用。
keywords: Java, Memory Leaks
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

​

 本文翻译自：[The Introduction of Java Memory Leak](http://www.programcreek.com/2013/10/the-introduction-of-memory-leak-what-why-and-how/ "The Introduction of Java Memory Leak")[s](http://www.programcreek.com/2013/10/the-introduction-of-memory-leak-what-why-and-how/ "s")  



Java中一个最重要的优点是它的内存管理，你只需要创建对象，Java垃圾回收器就会负责分配和释放内存。然而，情况并不是所谓的那么简单，因为在Java应用程序中经常会发生内存泄漏。

本教程阐释了什么是内存泄漏，为什么它会发生，以及如何阻止它们。

## **1. 什么是内存泄漏？**

内存泄漏的定义：**对象不再被应用程序使用，但垃圾回收器却不能移除它们，因为它们还在被引用。**

为了理解这个定义，我们需要理解对象在内存中的状态，下面的图阐释了什么是不再使用和什么是不再引用。

![](/images/posts/2016-05-25-Java-Memory-Leaks-Intro/p1.jpeg)![](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw== "点击并拖拽以移动")

在上面的图中，有未引用的对象和在引用的对象。未引用的对象将会被垃圾回收器回收，然而在引用的对象不会被垃圾回收器回收。未引用的对象肯定是无用的，因为没有其它的对象引用它了。然而，不再使用的对象不都是不在被引用的。它们中的有一些也正在被引用，这就是内存泄漏的来源！

## **2. 为什么会发生内存泄漏？**

让我们看一看下面的例子和了解为什么会发生内存泄漏。在下面的这个例子中，对象A引用对象B。A的生命周期（t1 - t4） 比B的生命周期（t2 - t3）长。当在应用程序中B不再被使用，A仍然持有它的引用。在这种情况下，垃圾回收器不能够把B从内存中移除。这就可能造成内存溢出问题。因为如果A像引用B一样引用了更多的对象，这里就可能存在大量的对象不能被回收而且它们还消耗了内存空间。

相同的，B持有大量的其它引用对象也可能发生内存溢出问题。那些被B引用的对象也不会被垃圾回收器回收，所有不再被使用的对象将消耗宝贵的内存空间。

![](/images/posts/2016-05-25-Java-Memory-Leaks-Intro/p2.jpeg)![](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw== "点击并拖拽以移动")​

## **3. 怎样阻止内存泄漏？**

以下是防止内存泄漏的快速操作技巧。

1. 注意集合类，如HashMap,ArrayList等。它们在相同的地方可以找到内存泄漏。当它们被声明为static的时候，它们的生命周期和应用程序的生命周期一样。

2. 注意事件的监听和回调。如果注册了监听器，但在类不再被使用的时候没有取消注册的监听器，就有可能发生内存泄漏。

3. “如果一个类管理它自己的内存，程序员应该警惕内存泄漏。” 通常一个对象的成员变量指向其它对象需要被置为null。

## **4. 一个小提问：为什么JDK6中的subString()方法会造成内存泄漏？**

要回答这个问题，你可以阅读  [The substring() Method in JDK6 and JDK7](http://www.programcreek.com/2013/09/the-substring-method-in-jdk-6-and-jdk-7/ "The substring() Method in JDK6 and JDK7")

## 参考文章：

1. Bloch, Joshua. Effective java. Addison-Wesley Professional, 2008.  

2. IBM Developer Work. [http://www.ibm.com/developerworks/library/j-leaks/](http://www.ibm.com/developerworks/library/j-leaks/)

​
