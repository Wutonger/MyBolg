title: 并发的根源—Java内存模型
author: Loux

cover: /img/post/19.jpg

tags:

  - 并发编程
categories:
  - 并发编程
date: 2019-10-25 09:52:00

---

Java内存模型即JMM（Java Memory Model）,它与JVM运行时数据区域是不同的两个概念；前者实际上是一种规范，它规定了Java程序的运行行为，包括多线程对共享内存读取时，所能读到的值应该遵守的规则；而后者则是JVM中对于内存的逻辑划分。

## 为什么需要JMM

随着这些年CPU的不断发展，CPU的性能已经越来越强大，但受迫与频率提升的困难，目前的CPU多数已经朝着多核发展。而软件开发人员为了充分利用到CPU的性能，程序中的对于多线程的使用也越来越多。CPU在进行运算时，会对指令做一些优化使得速度更快，这些优化在单线程应用中没有问题，但是对于多线程程序来说可能会出现一些意料不到的问题。

目前CPU的运算速度已经是非常快了，性能瓶颈主要出现在对内存的访问上，所以为了提升效率，CPU使用了缓存技术，用了L1,L2,L3一共三级缓存。CPU会先将主存中的数据复制到缓存，这样在运算时就可以从缓存中读取数据了，在计算完成后再把数据从缓存中更新回主存。这种操作方式在运算期间不用访问主存，运算速度大大提高。

![多线程对于共享变量的运算](/images/image-20191025103132732.png)

而这样就会导致一个问题，多个线程对某一个共享变量操作时，每个线程对变量的操作其实都不是主存中真正的值，而是缓存中存储的值，不知道存储了多久了。这样再运算结束后每个线程都将运算结果刷新回主存后我们就会发现“为什么计算的结果不是我们预期的值？”。

并且JMM允许编译器和缓存保持对数据操作顺序优化的自由度。除非程序使用Synchronized或者volatile显式的告诉处理器需要确保可见性。这意味着如果你没有进行同步，那么多线程程序对于数据的操作，将会呈现不同的顺序。也就是前面一节讲的有序性，我们基于代码顺序对数据赋值顺序的推论，在多线程程序中可能会不成立。 

## Hapens-Before原则

 Happens-Before在多线程领域具有重大意义，它可以指导你如何开发多线程的程序，而不至于陷入混乱之中。你所开发的多线程程序，如果想对共享变量的操作符合你设想的顺序，那么需要依照Happens-Before原则来开发。happens-Before并不是指操作A先于操作B发生，而是指操作A的结果在什么情况下可以被后面操作B所获取。Hapens-Before 原则包括以下内容：

* 程序顺序规则：如果程序中A操作在B操作之前，那么线程中A操作将在B操作之前执行
* 上锁原则：不同线程对同一个锁的lock操作一定在unlock之前
* volatile变量原则：对于volatile的写操作会早于对其的读操作
* 线程启动原则：调用ThreadA.start()方法，那么ThreadA.start()一定比该线程的其它方法早执行
* 传递规则：如果A早于B执行，B早于C执行，那么A一定早于C执行
* 线程中断规则： 对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生 
* 线程终结规则： 如果线程A终结了，并且导致另外一个线程B中的ThreadA.join()方法取得返回，那么线程A中所有的操作都早于线程B在ThreadA.join()之后的动作发生。 
* 对象终结规则：一个对象的初始化操作肯定先于它的finalize()方法



我们只有充分理解了happens-before原则，才能在编写多线程程序的时候，尽量避免数据的不一致性，让多线程程序在必要的时候按照我们设计的次序执行。 

---





