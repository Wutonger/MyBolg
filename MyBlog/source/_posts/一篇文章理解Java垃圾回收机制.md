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

![循环引用问题](/images/1571822709027.png)

在程序的最后，虽然a与b已经指向空了，但是由于这两个对象互相引用着对方，导致这两个对象的引用计数永远不会减到0，也就是说这两个对象永远不会被回收，这样既白白的占了空间，又造成了内存泄漏。

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

此时s为GC Root,当s置为空时，也断了与"localParameter"对象的联系，此对象将被回收。

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

此时s作为GC Root,s置为null时，"properties"对象由于无法对其建立链接，而被回收。但是m由与作为类的静态属性，也属于GC Roots,而m与"parameter"对象的链接并未被置空，所以"parameter"对象并不会被回收。

**方法区中常量引用的对象**

与上述例子同样的道理......省略

**本地方法栈中Native方法引用的对象**

 任何 Native 接口都会使用某种本地方法栈，实现的本地方法接口是使用 C 连接模型的话，那么它的本地方法栈就是 C 栈。当线程调用 Java 方法时，虚拟机会创建一个新的栈帧并压入 Java 栈。然而当它调用的是本地方法时，虚拟机会保持 Java 栈不变，不再在线程的 Java 栈中压入新的帧，虚拟机只是简单地动态连接并直接调用指定的本地方法。