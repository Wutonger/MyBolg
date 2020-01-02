title: 经典并发容器—ConcurrentHashMap源码分析
author: Loux
cover: /img/post/27.jpg
tags:
  - 并发编程
  - 源码分析
categories:
  - 并发编程
date: 2019-12-18 11:15:00
---
HashMap对于我们来说已经很了解了，在单线程环境操作Map时我们总会选择它。但是在多线程下对HashMap的操作可能会导致一些不能预估问题的发生，这主要是因为HashMap中的get()、put()等方法都不是线程安全的。我们可以看到HashMap源码中没有对多线程操作做处理。

![HashMap内部方法](/images/image-20191218113218923.png)

那么如果我们要保证Map在多线程环境下依然能正常使用，我们可以采取哪些方式呢？这里我知道的方式有以下三种

`第一种:使用Collections.synchronizedMap(Map map)方法来创建线程安全的Map集合`

`第二种:使用HashTable`

`第三种:使用ConcurrentHashMap`

那么问题来了，解决并发问题的Map有三种，我们为什么非要使用ConcurrentHashMap呢？

# 为什么要用ConcurrentHashMap

先简单的说一下前两种实现方式的原理。

## Collections.synchronizedMap()

我们可以看一下SynchronizedMap类的源码

![](/images/image-20191218115508074.png)

在SynchronizedMap内部维护了一个普通的Map对象，还有一个排它锁mutex。在调用 Collections.synchronizedMap(Map map) 方法的时候，会调用SynchronizedMap类的构造方法。并执行`mutex = this;`，即该synchronizedMap对象就是这把锁，后面对synchronizedMap的操作都要通过mutex这把锁来进行同步，这里列出一部分源码



![](/images/image-20191219152528077.png)

即Collections.synchronizedMap(Map map)方法的本质是创建了一个线程安全的Map,而线程安全是通过synchronized同步机制保证的，非常的简单粗暴。

## HashTable

HashTable其实也与第一种方式有着相同的本质，看一下源码就能理解

![](/images/image-20191219153307477.png)

我们可以看到，HashTable的每个影响线程安全的方法都是由synchronized修饰的。

也就是说从效率上来讲，这两种方式基本是差不多的。而且采用的方法都很粗暴。你不是线程不安全吗？我给你全部加上synchronized就完事了哦。

这时候你可能就会想到了，有没有可能是ConcurrentMap采用了更巧妙的方式，使得其效率比前两个高所以在实际使用中使用它更多呢？答案当然是肯定的。

# ConcurrentHashMap简介

ConcurrentHashMap的作者是大名鼎鼎的Doug Lea,一位慈祥的老爷子。你要问我他是谁，我只能用百科来表示他有多牛b。



![](/images/image-20191219154433902.png)

整个JUC包都是他贡献的，给跪了。

从数据结构上来看，ConcurrentHashMap与HashMap一样，都是数组+链表的形式（详情可以看我以前一篇分析HashMap源码的文章）。这里我们只说说它与HashMap有什么不同的地方，或者说它是怎样实现线程安全的。

从JDK1.7到JDK1.8，线程安全的实现方式有了很大的改变。

![JDK1.7中ConcurrentHashMap结构](/images/image-20191219164314975.png)

从图上可以看出，在JDK1.7中，ConcurrentHashMap中的数组被分为了很多段，每一段都被称为一个"segment",也就是ConcurrentHashMap其实是由segment类型组成的数组，而SegMent类又继承于ReentrantLock,即每个segment只能同时被一个线程访问，线程A访问segment1的时候，不影响线程B访问segment2。这样通过这种“分段锁机制”就实现了多线程下的线程安全。其并发能容纳最大的线程数量其实也就是Map中segment的数量。

我们可以看下在JDK1.7中Segment的源码

```java
static final class Segment<K, V> extends ReentrantLock implements Serializable  {
	    private static final long serialVersionUID  =  2249069246763182397L;
        //Segment中用于存放数据的数组
	    transient volatile HashEntry<K, V>[] table;    
    
	    transient int count;             
    
	    transient int modCount;               
        //负载因子
	    final float loadFactor;
}
```

具体的就不仔细说了。我们主要看看它在JDK1.8中是怎样实现的。

# ConcurrentHashMap在JAVA8中的实现

在JAVA8中，ConcurrentHashMap的实现有了比较大的变动。

首先，它的基本结构还是和JAVA8中的HashMap一样，链表长度大于8且容量达到了64就会将链表变为红黑树，其目的是为了增加查找时的效率。

而在线程安全方面，在Java8中主要是采用CAS操作与synchronized关键字来实现，相比于Java7，并发性能有了很大的提高。

## ConcurrentHashMap的主要组成部分

`transient volatile Node<K,V>[] table`

Node类型的数组，这个就是用来真实储存数据的，volatile修饰说明当某一线程对其进行修改后，对于其它线程都是立即可见的。

`Node`

存储单个key-value元素的节点，并且还存储了链表中下一个节点的引用

`private static final int DEFAULT_CAPACITY=16`

默认初始化时Node数组的大小

`static final int TREEIFY_THRESHOLD=8`

转换为红黑树时链表长度的临界点

`static final int MOVED=-1`

标识作用，用于识别扩容时正在转移数据

`private transient volatile Node<K,V>[] nextTable`

数据转移时新的Node数组

`ForwardingNode`

进行扩容的时候，会将链表转移到新的数组中去，在这时会将数组中头节点替换为ForwardingNode,它不保存key-value,只保存了扩容后nextTable的引用，此时进行查找操作时，需要到nextTable中去找。

## 初始化方法

ConcurrentHashMap有多个构造函数，一个是空的，什么都不做；另外两个是带参数的，我们来看看带参数的构造函数

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

这里我们可以看到，如果在实例化对象的时候指定了容量，则初始化sizeCtl。而sizeCtl得值总是接近于  (1.5 * initialCapacity + 1) 最近的2^n的值，例如initialCapacity为10，那么sizeCtl的值为16，如果initialCapacity为13，那么sizeCtl的值为32。

而sizeCtl正式用来控制数组的初始化与扩容的。

## put过程分析

整个Node数组其实是在第一次put的时候才被初始化的。在put的过程中调用了一系列的方法。接下来我们就对put的过程进行分析。

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key || value为空时直接抛出异常，这里后面会进行解释
    if (key == null || value == null) throw new NullPointerException();
    //计算出key的hash值
    int hash = spread(key.hashCode());
    //记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
      //如果tab未被初始化，则调用initTable()初始化tab
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
      //tab不为空，则计算出hash值对应在数组中的下标的值，并判断是否被占用
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //若没有被占用，则直接用CAS操作将key-value封装为node节点放入tab
            //这时put操作就已经结束了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   
        }
      //f.hash==MOVED，表明tab正在做扩容，此时当前线程参与帮助数据转移
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
      //下面分支处理key的hash值映射在tab中的位置已经存在值的情况
        else {
            V oldVal = null;
            //采用同步代码块，锁住该值
            synchronized (f) {
              	//再次确认该位置的值是否已经发生了变化,因为在这这钱可能tab[i]处的值已经被其它线程改变了
                if (tabAt(tab, i) == f) {
                  //fh大于0，表示该位置存储的是链表
                    if (fh >= 0) {
                        //用于累加记录链表的长度
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                          //如果存在一样hash值的node，那么根据onlyIfAbsent的值选择覆盖value或者不覆盖，若覆盖，put操作也结束了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                          //若找到该链表最后也没有进行覆盖，则将其包装为Node节点加在链表的最后一个节点之后
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                  //下面的逻辑处理链表已经转为红黑树时的key/value保存
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
          //node保存完成后，判断链表长度是否已经超出阈值，则进行哈希表扩容或者将链表转化为红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
  	//计数，并且判断哈希表中使用的桶位是否超出阈值，超出的话进行扩容
    addCount(1L, binCount);
    return null;
}
```

整个put流程大概我们已经清楚了，但是对于初始化tab、扩容，我们都是很模糊的，只知道它大概做了什么，而不知道具体的实现。

## 初始化tab数组：initTable()方法

首先我们需要知道的是，初始化方法是会存在并发问题的，我们必须防止多个线程多次初始化table,也就是说当一个线程初始化table后，我们必须保证其它线程已经知道该table已被初始化了，这样其它线程才不会做初始化操作。那么在initTable()中是怎样保证的呢？我们还需要根据源码来看

```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //若sizeCtl<0说明其它线程正在执行初始化操作，当前线程让出cpu执行权直到table初始化完成
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            //若没有其它线程正在初始化table,则当前线程将sizeCtl的值用CAS设置为-1，设置成功说明抢到了锁，table正在被当前线程初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    //再次判断table是否为空，若不为空开始初始化操作
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //将sizeCtl设置为sc
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

这里我们可以看到，保证table只被多个线程初始化一次的秘诀就是通过对sizeCtl的CAS操作。

## 扩容源码分析

在put()方法执行中，当我们把元素加入到哈希表中去之后会调用treeifyBin()方法来决定是把链表转为红黑树还是扩容数组。

```java
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            //如果table的长度小于64，这时选择扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            //将数组中以index开头的链表转变为红黑树
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                //加同步锁，防止多个线程一起操作
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        //遍历链表，将node转换为treenode
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        //转换完成，将红黑树设置在数组的index位置中
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

这里我们可以看到，在选择扩容时，调用了`tryPresize(n << 1)`,其中n<<1操作就是求n*2,也就是扩容后的大小。也就是说这个方法的参数是扩容后数组的容量。

```java
    private final void tryPresize(int size) {
        //若size大于最大容量了，这时c为最大值，否则通过tableSizeFor()计算出要满足的数组容量
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            //这里处理table未初始化的逻辑，和上面初始化方法相似。主要是由于putAll方法导致的
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            //已到最大值，无法扩容
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                //若sc,即sizeCtl<0,说明其它线程正在扩容，当前线程协助扩容
                if (sc < 0) {
                    Node<K,V>[] nt;
                    //判断扩容线程是否到达上限，若到达上限则退出
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    //未到达上限，则更新sizeCtl的值
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //将旧数组中的数据移入新数组中
                        transfer(tab, nt);
                }
                //当前线程第一个开始扩容的，则将sizeCtl的值设置为负数，代表有线程在扩容了
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    //此时null代表需要新建扩容后的数组
                    transfer(tab, null);
            }
        }
    }
```

这时我们会好奇，这个扩容的流程只是链表过长引起的呀。那么当数组容纳的元素个数超过了负载因子*容量的值，此时又是从哪里走的扩容流程呢？

我们可以看上面的put()方法代码中最后一句代码

`addCount(1L,binCount)`

这里其实就是对数组中保存的元素进行计数，同时根据目前数组中容纳桶的个数来判断是否需要扩容。扩容的逻辑和tryPresize()方法基本一致，这里就不细展开了。

## get流程分析

相比与put流程来说，get方法的流程就简单多了，因为它不涉及扩容、转变红黑树等操作。它的基本流程如下：

* 计算key的hash值
* 根据hash值找到数组坐标 (n-1)&hash
* 根据该节点性质进行遍历链表or红黑树查找（若该节点value为null，则直接返回）

知道了基本的流程后，再来看源码就很容易了

```java
   public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            //如果数组中的节点就是我们需要的则直接返回
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            //如果该节点hash值<0,说明正在扩容或者是红黑树
            //hash==-1,说明正在扩容，该节点为ForwardingNode,通过find方法在nextTable中查找
            //hash==-2,	说明该节点为TreeBin,同样通过treebin的find方法查找
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //以上都不满足，说明当前节点为链表的头节点，遍历链表查找即可
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

# null值问题

我们知道在HashMap中key or value值都是可以为null的，但在ConcurrentHashMap中两者是均不能为null的，这是因为什么原因呢？

有人就要说了，这是因为put()方法中对null进行了判断，若key和value为null会直接抛出空指针异常。但是为什么要这样设计呢？

我们知道ConcurrentHashMap是为多线程环境设置的。那么对于value来说，如果允许为空会导致程序的二义性。

第一种情况：本来该key值对应的value就为null

第二种情况：key值不存在，所以直接返回null

这样就会导致在调用get()方法返回null之后出现困惑，到底是第一种情况还是第二种情况呢？

又有人会说了，那么为什么HashMap中就能允许value为null呢？

这是因为HashMap被用在单线程环境，可以通过contains(key)方法来判断key值是否存在。但是由于ConcurrentHashMap是多线程环境，调用contains(key)得到的结果也是不准确的，例如线程A调用了map.contains(key)后线程B将该key值删除了，这时map.get(key)返回的就为null，此时程序员会得出结论，该key值映射的value就为null，而实际上该key值是根本不存在的。而禁止value为null可以防止二义性的出现

说以我们得出结论，在ConcurrentHashMap中，调用map.get(key)为null，一定说明了该key值在map中不存在。

那么key为什么也不能为null呢？我想大概是因为key为null本身就是无意义的。所以在ConcurrentHashMap中这样设置。

# 总结

对于我们程序员来说，在工作中使用到某个东西时，我们应当学会阅读其源码，理解其原理。源码中有很多优秀的设计思想，可以改变我们自己写代码的习惯。就比如我自己来说，我感觉现在自己写代码就已经比以前精简多了，有了一定的进步。

同时由于本人水平有限，文中的理解可能有不正确的地方，还请多多指教。