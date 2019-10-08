title: HashMap源码分析
author: Loux
cover: /img/post/14.jpg
tags:
  - 源码分析
categories:
  - 后端知识
img: \medias\featureimages\20.jpg
date: 2019-07-21 09:36:00
---
Map是Java集合中重要的分支之一。在工作中，我们经常会用到HashMap,今天就来分析以下HashMap的底层实现原理。首先，HashMap是采用的数据结构是数组+链表的形式，用一个图可以表示为：


![HashMap数据结构](/images/pasted-10.png)
在HashMap中，每个数组元素后都可能跟着一个链表，这样的一列链表我们称之为桶(bucket)。
#### 关键成员变量
在HashMap中，关键的成员变量如下：
* <b>table</b>：Node<K,V>类型的数组，用来存储键值对元素，是HashMap的核心
* <b>size</b>：用来记录HashMap的元素个数，每当元素增加或减少时size会进行相应的变化
* <b>loadFactor</b>：影响因子，即用来计算数组中可容纳最多桶的个数,为常量数值0.75。例如数组长度为16，那么最多容纳桶的个数为16x0.75=12个
* <b>threshold</b>：即最大容纳桶的个数，为数组长度与影响因子的乘积
* <b>entrySet</b>：Set<Map.Entry<K,V>>类型的集合，其中一个Entry<K,V>就是一个HashMap的key-value元素，entrySet主要被用作遍历HashMap集合
* <b>keySet</b>：Set<K>类型的集合，存储了HashMap中所有的Key
* <b>modCount</b>：记录HashMap修改的次数，例如增加了5个元素，再删除了3个元素，那么modCount的值为5+3=8 
  
#### 方法追踪
##### HashMap()方法
HashMap为创建map集合时所使用的构造方法，它其中的类容很简单，只是简单的初始化了影响因子loadFactor为0.75,并没有初始化table的值
```bash
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
##### put()方法
我们都知道HashMap拥有无序、key不能重复、允许空值特点。那么它为何会有这些特点呢？这就要讲put()方法了。在添加元素时调用put方法。我们先来看看put方法的源码
```bash
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
我们发现put方法中其实是调用了putVal方法，而putVal方法中又调用了hash(key)方法。
##### hash()方法
hash()方法其实是用来计算要添加元素的key的hash值，而hash值则关系到添加的元素在数组中存储的位置
```bash
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
##### putVal()方法
先贴出putVal()的源码
```bash
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
怎么解释这段代码呢？  
1. 首先进入方法后，判断table的值是否为null，为null则说明为第一次添加，首先调用resize()方法进行扩容，进入到resize()方法后我们发现首次扩容的长度为16，之后的每次扩容都是增加一倍的容量
2. 接下来判断tab[i = (n - 1) & hash]处是否有值，若没有值，则直接用hash,key,value,next=null的值新建一个Node<K,V>并赋值到tab[i]处，那么HashMap无序的原因就找到了，因为在数组中存储的位置并不是依次的，而是通过与运算得到的；若tab[i]处有值，则说明两者的hash值相同，继续判断equals和==，若两者中任意一个比较为true，则说明key已经重复，直接用新元素覆盖掉tab[i]处的元素，这也就是为什么HashMap中key不允许重复
3. 若两者的key不相同，则判断是否属于树的结点，若为树的结点则将该结点添加到树后(这点后面再介绍)。若不为树结点则继续往后走。接下来循环tab[i]处的桶，判断要添加的元素的key与桶中的元素是否重复，若重复则覆盖，若不重复则添加到桶中最后一个元素后面
4. 判断桶中的元素是否大于等于8个，若大于8个时调用treeifyBin(tab, hash)方法。需要注意的是该方法中并不是直接将链表变为红黑树，而是先进行判断table.length是否小于64，若小于64则先扩容。只有当桶中个数>=8与table.length均成立的时候，链表才会转换为红黑树
5. 最后再次判断是否需要扩容

##### resize()方法
resize()方法即扩容方法，第一次当table为空时，给table赋值为16的初始长度。以后每次进入resize()方法时table的长度均翻倍，并根据hash值重新计算每个元素的存放位置(可能会打乱原来的顺序)
#### hash冲突的解决
既然是HashMap,肯定利用到了hash表。在HashMap中主要是通过键值对的键的hash值来计算元素应该存储的位置的。那么元素多了，肯定会存在计算出的hash值重复的情况，这就是我们平常所说的hash冲突。通过查阅相关资料可以得到hash冲突的解决有以下几种方法：
##### 开放定址法
开放定址法是解决hash冲突的一种简单方法，其核心在于从表中发生冲突的那个元素起，按照一定的规律寻找下一个空闲的位置，若不为空则按照规律继续寻找直到遇到空的位置存放冲突的元素。开放地址法中常用的有：
* <b>线性探测法：</b>向后面依次判断,直到遇到空闲的位置，若到达表尾没有找到则继续从表头开始判断
* <b>平方探测法：</b>h(key)的值依次加上1的平方，2的平方，3的平方直到找到空闲位置...若找不到则说明该hash表基本满了，需要进行扩容
* <b>双散列函数探测法：</b>这种方法使用两个散列函数hl和h2。其中hl和前面的h一样，以关键字为自变量，产生一个0至m—l之间的数作为散列地址；h2也以关键字为自变量，产生一个l至m—1之间的、并和m互素的数(即m不能被该数整除)作为探查序列的地址增量(即步长)，探查序列的步长值是固定值l；对于平方探查法，探查序列的步长值是探查次数i的两倍减l；对于双散列函数探查法，其探查序列的步长值是同一关键字的另一散列函数的值

##### 链地址法
链接地址法的思路是将哈希值相同的元素构成一个同义词的单链表，并将单链表的头指针存放在哈希表的第i个单元中，查找、插入和删除主要在同义词链表中进行。链地址法适用于经常进行插入和删除的情况。  
根据HashMap独特的数据结构，我们可以想到它正是采用了这种方式解决hash值冲突的问题。
在HashMap中，解决冲突最重要的两个方法则是hashCode()与equals()。key.hashCode()可以获得key的hash值，hash值与表的长度-1进行与运算后得出存放的位置，若该位置已有元素则利用equals()判断已有元素的key与要添加元素的key是否为true,为true则说明两者完全一样，直接进行覆盖操作；若为false则说明两者hash值相同可能只是凑巧而已，两者实际并不相同，将新元素以单链表的形式加在已有元素的后面。
<b>特别要注意的是可以通过重写key对象所在类的hashCode()与equals()来实现满足自己实际业务逻辑的需要</b>

##### 再哈希法
就是同时构造多个不同的哈希函数：
Hi = RHi(key) i= 1,2,3 ... k;
当H1 = RH1(key) 发生冲突时，再用H2 = RH2(key) 进行计算，直到冲突不再产生，这种方法不易产生聚集，但是增加了计算时间。

##### 建立公共溢出区
将哈希表分为公共表和溢出表，当溢出发生时，将所有溢出数据统一放到溢出区。

#### jdk1.7与1.8中的区别
##### jdk1.7
1. 底层数据结构为数组+链表
2. 创建HashMap对象时初始化了初始容量为16
3. 数组中元素为Entry类型  

##### jdk1.8
1. 底层数组结构为数组+链表+红黑树
2. 创建HashMap对象时仅仅初始化了影响因子为0.75，只有当第一次添加时才会初始化长度为16
3. 数组中元素为Node类型