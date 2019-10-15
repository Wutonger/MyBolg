title: JVM内存自定义分配参数
author: Loux
cover: /img/post/21.jpg
tags:
  - JVM
categories:
  - JVM
date: 2019-10-15 15:03:00
---

前面说了JVM的内存模型，在jdk8中分为堆、虚拟机栈、本地方法栈、程序计数器、元空间。其中堆空间主要是用来存放对象的，因此它占的空间相对于其它区域来说，要大很多。在实际的项目中，我们往往要根据项目的特点与服务器内存大小对JVM内存空间大小进行调整，也就是俗称的“JVM调优”。

## 堆空间默认大小

在JVM中，堆被划分为了新生代(Young)、老年代(Old),其中新生代又被分为了Eden Space、From Survivor Space、To Survivor Space。而这样划分的目的是为了JVM能更好的管理堆内存中的对象，包括内存分配以及GC。

在默认情况下：

* 堆内存初始大小为操作系统可用内存的1/64
* 堆内存可被分配的最大内存为操作系统可用内存的1/4
* 新生代与老年代占堆内存的比例分别为1/3、2/3
* Eden Space : From Survivor Space : To Survivor Spce = 8 : 1 : 1

## 调整内存的参数

 **-Xmx**  java虚拟机堆区内存可被分配的最大上限 

 **-Xms**  java虚拟机堆区内存初始内存分配的大小 

  **-XX:NewSize**   新生代初始内存的大小，应该小于-Xms的值 

 **-XX:NewRatio**  Yong 和 Old的比例，比如值为2，则Old是Yong的2倍，即Yong Generation占据内存的1/3 

 **-XX:Maxnewsize**   Yong的最大值大小 

 **-Xmn**  对 -XX:newSize、-XX:MaxnewSize两个参数的同时配置，也就是说如果通过-Xmn来配置新生代的内存大小，那么-XX:newSize = -XX:MaxnewSize　=　-Xmn 

 **XX:Surviorratio**   Eden和一个Suivior的比例，比如值为5，即Eden是To(S2)的比例是5，（From和To是一样大的），此时Eden占据Yong Generation的5/7 

