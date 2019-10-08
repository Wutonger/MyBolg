title: ThreadLocal类解析
author: Loux
cover: /img/post/23.jpg
tags:
  - 多线程
  - 并发编程
categories:
  - 并发编程
date: 2019-09-05 19:38:00
---
在java并发编程中，ThreadLocal类的使用在某些场合中有着非常精妙的作用，它本身的原理其实并不复杂。那么什么是ThreadLocal呢？ThreadLocal其实是一个本地线程副本变量工具类，使用它可以方便的存储、取出该线程的变量，而不影响其它的线程。在高并发的场景下，能实现无状态调用，比较适合多个线程依赖不同的变量进行操作的场景。
这里在网上找了一张ThreadLocal的结构图，如下：

![ThreadLocal结构图](/images/pasted-20.png)
从上图我们可以得知

