title: 一篇文章理解Java垃圾回收机制
author: Loux

cover: /img/post/22.jpg

tags:

  - JVM
categories:
  - JVM
date: 2019-10-23 16:07:00

---

垃圾回收（Garbage Collection），简称为GC,也就是说清理”垃圾“占用的空间，使得内存得到充分的利用，同时也为了防止内存泄漏。以前大学的时候学C++语言的时候，每次创建对象要用构造函数，对象使用完毕后又要使用析构函数释放对象占用的内存，太麻烦了。那么为什么在Java中，我们将对象使用完成后没有手动释放内存呢？那是因为在Java中GC是自动的，在程序运行过程中，GC线程为守护线程，会自动的清理JVM内存中可被回收的内存区域。  

那么GC是Java语言的专有吗？并不是的，早在1960年，基于 MIT 的 Lisp 首先提出了垃圾回收的概念，而这时 Java 还没有出世呢！所以实际上 GC 并不是Java的专利，GC 的历史远远大于 Java 的历史。

##  怎样定义垃圾？

要进行垃圾回收的第一步就是要找到垃圾，也就是说要知道哪些对象应该被回收。目前主流的两种方法有两种：引用计数法、可达性分析法。

### 引用计数法

 引用计数算法（Reachability Counting）是通过在对象头中分配一个空间来保存该对象被引用的次数（Reference Count）。如果该对象被其它对象引用，则它的引用计数加1，如果删除对该对象的引用，那么它的引用计数就减1，当该对象的引用计数为0时，那么该对象就会被回收。 

> Student s = new Student();

此时，一个新创建的Student类的对象有了一个引用"s",该对象的引用加1

> s = null

这时，该Student类对象的引用减1，变为0。下次GC到来时，此对象将会被回收。

这个方法看起来既简单，又高效。但是目前的HotSpot虚拟机并未采用这种方式，因为这种方式有一个严重的缺点。

```code
public class ReferenceCountingGC {
    public Object instance;
    public ReferenceCountingGC(String name){}
}
public static void testGC(){
    ReferenceCountingGC a = new ReferenceCountingGC("objA");
    ReferenceCountingGC b = new ReferenceCountingGC("objB");
    a.instance = b;
    b.instance = a;
    a = null;
    b = null;
}
```

在程序的最后，虽然a与b已经指向空了，但是由于这两个对象互相引用着对方，导致这两个对象的引用计数永远不会减到0，也就是说这两个对象永远不会被回收，这样既白白的占了空间，又造成了内存泄漏。

![循环引用问题](/images/1571822709027.png)

### 可达性分析算法

可达性分析算法（Reachability Analysis）的基本思路是，通过一些被称为引用链（GC Roots）的对象作为起点，从这些节点开始向下搜索，搜索走过的路径被称为（Reference Chain)，当一个对象到 GC Roots 没有任何引用链相连时（即从 GC Roots 节点到该节点不可达），则证明该对象是不可用的。

![可达性分析法](/images/1571822059095.png)

可达性分析算法，成功解决了循环引用得问题，只要对象无法与GC Roots建立直接或间接的联系，系统就会判定该对象为可回收对象。例如上图中，object5、object6、object7与GC Roots的链接断了，所以它们是可回收对象。

在JVM中，可作为GC Roots的对象有以下四种：

* 虚拟机栈（栈帧中的局部变量表）中的引用对象
* 方法区中的静态属性引用的变量
* 方法区中常量引用的对象
* 本地方法栈中Native方法引用的对象

**栈帧中局部变量表中引用的对象**

```code
public class StackLocalParameter {
    public StackLocalParameter(String name){}
}

public static void testGC(){
    StackLocalParameter s = new StackLocalParameter("localParameter");
    s = null;
}
```

此时s为GC Roots,当s置为空时，也断了与"localParameter"对象的联系，此对象将被回收。

**方法区中静态属性引用的对象**

```code
public class MethodAreaStaicProperties {
    public static MethodAreaStaicProperties m;
    public MethodAreaStaicProperties(String name){}
}

public static void testGC(){
    MethodAreaStaicProperties s = new MethodAreaStaicProperties("properties");
    s.m = new MethodAreaStaicProperties("parameter");
    s = null;
}
```

此时s作为GC Roots,s置为null时，"properties"对象由于无法对其建立链接，而被回收。但是m由与作为类的静态属性，也属于GC Roots,而m与"parameter"对象的链接并未被置空，所以"parameter"对象并不会被回收。

**方法区中常量引用的对象**

与上述例子同样的道理......省略

**本地方法栈中Native方法引用的对象**

 任何 Native 接口都会使用某种本地方法栈，实现的本地方法接口是使用 C 连接模型的话，那么它的本地方法栈就是 C 栈。当线程调用 Java 方法时，虚拟机会创建一个新的栈帧并压入 Java 栈。然而当它调用的是本地方法时，虚拟机会保持 Java 栈不变，不再在线程的 Java 栈中压入新的帧，虚拟机只是简单地动态连接并直接调用指定的本地方法。

![java方法调用native方法](/images/image-20191024091952505.png)

## 垃圾回收算法

说完了JVM怎么识别哪些对象是“垃圾”，那么知道了什么是垃圾后，又该用什么样的方式对这些“垃圾”进行回收呢？由于Java虚拟机规范并没有规定如何进行垃圾回收，所以不同的虚拟机可能会采用不同的垃圾回收算法，但是有一些常用的算法被经常使用。

### 标记-清理算法

![标记-清理算法](/images/image-20191024094941601.png)

标记-清理算法理解起来也很简单，它分为两个步骤，第一步先将可回收的内存区域进行标记，第二步将这些被标记的区域内存清空，等待被再次使用。但是这个方法有一个很大的问题就是会产生内存碎片。

我们知道java在为对象分配内存时，会开辟连续的空间。图中所示回收后的空闲空间有些是被夹在两个已被使用空间之间的，这样会导致如果我要开辟一块大一点的空间时，这些被夹住的小内存空间不可用。这样就会导致浪费很多的内存。

### 复制算法

![复制算法](/images/image-20191024100526114.png)

复制算法将内存分为两块相同大小的区域，当一块区域被使用满后，就将这块内存上存活的对象移动到另外一块区域上去，然后再将这块区域全部清空。

但是这样也有一个弊端，明明我有100M可用内存，现在只能当作50M内存空间来使用，感觉太亏了。

### 标记-整理算法

![标记-整理算法](/images/image-20191024102908683.png)

标记-整理算法是在标记-清除算法上改进得来的， 但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，再清理掉端边界以外的内存区域。 

 标记整理算法一方面在标记-清除算法上做了升级，解决了内存碎片的问题，也规避了复制算法只能利用一半内存区域的弊端。看起来很美好，但从上图可以看到，它对内存变动更频繁，需要整理所有存活对象的引用地址，在效率上比复制算法要差很多。 

## JVM分代收集策略

 按对象存活周期的不同将内存划分为几块。一般是把 Java 堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用标记-清理或者标记 -整理算法来进行回收。

 Java 堆（Java Heap）是JVM所管理的内存中最大的一块，堆又是垃圾收集器管理的主要区域，这里我们主要分析一下 Java 堆的结构。  Java 堆主要分为2个区域-年轻代与老年代，其中年轻代又分 Eden 区和 Survivor 区，其中 Survivor 区又分 From 和 To 2个区。那么为什么需要 Survivor 区，为什么Survivor 还要分为From区和To区呢？

### Eden区

IBM 公司的专业研究表明，有将近98%的对象是朝生夕死，所以针对这一现状，大多数情况下，对象会在新生代 Eden 区中进行分配，当 Eden 区没有足够空间进行分配时，虚拟机会发起一次 Minor GC，Minor GC 相比 Major GC 更频繁，回收速度也更快。

通过 Minor GC 之后，Eden 会被清空，Eden 区中绝大部分对象会被回收，而那些无需回收的存活对象，将会进到 Survivor 的 From 区（若 From 区不够，则直接进入 Old 区）。

### Survivor区

 Survivor 区相当于是 Eden 区和 Old 区的一个缓冲，类似于我们交通灯中的黄灯。Survivor 又分为2个区，一个是 From 区，一个是 To 区。每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区（如果 To 区不够，则直接进入 Old 区）。

### 为什么需要Survivor区

那为什么不直接弄个Eden->Old，而中间还要加个Survivor缓冲区呢？试想一下，如果没有Survivor区，每次Minor GC,存活的对象就会被送到老年代，老年代的空间很快就会被填满，而老年代中进行的Major GC会消耗大量的时间。其实有些对象虽然第一次Minor GC没有被回收，但是有可能第二次、第三次就会被回收。

所以，Survivor 的存在意义就是减少被送到老年代的对象，进而减少 Major GC 的发生。Survivor 的预筛选保证，只有经历16次 Minor GC 还能在新生代中存活的对象，才会被送到老年代。 

### 为什么Survivor区需要两个

还记得我们刚刚所说的复制算法么，复制算法正是需要两块内存。说到这里应该就能联想到，两块Survivor区的存在正是为了解决内存碎片化的问题。

Survivor 如果只有一个区域会怎样？Minor GC 执行后，Eden 区被清空了，存活的对象放到了 Survivor 区，而之前 Survivor 区中的对象，可能也有一些是需要被清除的。问题来了，这时候我们怎么清除它们？在这种场景下，我们只能标记清除，而我们知道标记清除最大的问题就是内存碎片，在新生代这种经常会消亡的区域，采用标记清除必然会让内存产生严重的碎片化。而如果采用标记整理，由于新生代的对象非常容易被回收，所以会产生大量的碎片空间，依次整理十分的降低回收效率。**因为 Survivor 有2个区域，所以每次 Minor GC，会将之前 Eden 区和 From 区中的存活对象复制到 To 区域。第二次 Minor GC 时，From 与 To 职责兑换，这时候会将 Eden 区和 To 区中的存活对象再复制到 From 区域，以此反复**。

这种机制最大的好处就是，整个过程中，永远有一个 Survivor space 是空的，另一个非空的 Survivor space 是无碎片的。那么，Survivor 为什么不分更多块呢？比方说分成三个、四个、五个?显然，如果 Survivor 区再细分下去，每一块的空间就会比较小，容易导致 Survivor 区满，两块 Survivor 区可能是经过权衡之后的最佳方案。

### Old区

老年代占据着2/3的堆空间，只有在Major GC的时候才会进行清理，而每次Major GC都会触发“stop-the-world”，内存空间越大，STW的时间就会越长，所以内存也不是越大越好。由于复制算法在对象存活率较高的老年代会进行很多次的复制操作，效率很低，所以老年代这里采用的是标记 - 整理算法。

在内存担保机制下，若Survivor区中空间不足，对象会直接进入老年代，另外还有几种对象会进入老年代中。

* 大对象— 大对象指需要大量连续内存空间的对象，这部分对象不管是不是“朝生夕死”，都会直接进到老年代。这样做主要是为了避免在 Eden 区及2个 Survivor 区之间发生大量的内存复制。
* 长期存活对象— 虚拟机给每个对象定义了一个对象年龄（Age）计数器。正常情况下对象会不断的在 Survivor 的 From 区与 To 区之间移动，对象在 Survivor 区中每经历一次 Minor GC，年龄就增加1岁。当年龄增加到15岁时，这时候就会被转移到老年代。当然，这里的15，JVM 也支持进行特殊设置。 
* 动态年龄对象—虚拟机并不重视要求对象年龄必须到15岁，才会放入老年区，如果 Survivor 空间中相同年龄所有对象大小的总合大于 Survivor 空间的一半，年龄大于等于该年龄的对象就可以直接进去老年区，无需等你“成年”。 

---

本文参考文章- https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653199017&idx=1&sn=fb5e9ad1d18ffe750f5418f6d98a2551&chksm=8c99e873bbee616508a27ca537031e2a42dcba8e2bb1f17d799e5c9379a61901f97b60ef8f6f&mpshare=1&scene=23&srcid=&sharer_sharetime=1571802790883&sharer_shareid=f0d05f5f73ce574ebbbbfefac445b28c#rd 