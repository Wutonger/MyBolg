title: ReentrantLock源码分析
author: Loux

cover: /img/post/25.jpg

tags:

  - 并发编程
  - 源码分析
categories:
  - 并发编程
date: 2019-11-14 15:04:00

---

ReentrantLock是Lock接口的一个重要实现类，中文翻译过来为“可重入锁”，它提供了比内置锁Synchronized更加强大的功能，例如能实现公平锁、可以tryLock，一定时间内线程获取不到锁就放弃等等。在Java 6以后性能并不是Synchronized与ReentrantLock的主要差距，所以在我们平时的使用中，要根据具体情况来选择具体使用哪种锁。

# ReentrantLock类整体结构

我们先来看一下ReentrantLock的主要属性以及构造函数：

```java
   //Sync继承自AQS类 
   private final Sync sync;
    
    //无参构造函数创建的为非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    //false : 非公平锁   true : 公平锁 
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

构造函数很简单，并且我们可以看到ReentrantLock类在初始化的时候其实是创建了Sync对象。FairSync与NonfairSync都继承自Sync，它们之间的关系如图所示：

![Sync类与其父子类](/images/image-20191114155322309.png)

AQS依赖FIFO队列实现了阻塞锁和相关的同步器，它内部有一个`private volatile int state`变量。实际上多线程对于锁的竞争其实就是对state变量写入的竞争。一旦state从0变为1，代表该锁已经被某一线程拥有，其它线程则进入一个FIFO的队列进行等待。同一线程若已经持有锁，再次获取锁时将会把state的值加1，释放则会减1，直到state的值变为0，该线程才彻底的将锁释放。

# ReentrantLock源码分析

## lock()方法

```java
    public void lock() {
        sync.lock();
    }
```

lock方法特别简单，其实就是调用了sync对象的lock方法。这里就要分为创建的是公平锁还是非公平锁了。

<b>第一种情况—公平锁</b>

FairSync中的lock方法也十分简单

```java
final void lock() {
    acquire(1);
}
```

在它的lock()方法中又调用了acquire(）方法，acquire()是AQS中的方法

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

先执行tryAcquire()方法获取锁，若失败则调用addWaiter()方法将当前线程线程包装为一个Node节点加入FIFO队列中等待尝试获取锁，若成功并导致自己被中断则acquireQueued()方法返回true，导致selfInterrupt()触发，这个方法只是简单的调用了当前线程的interrupt()方法。若成功而自己未被中断，则什么也不发生。

此时该方法中又调用了tryAcquire()方法，该方法在FairSync中

```java
protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //获取AQS中state的值
            int c = getState();
            // c==0,没有线程获取锁
            if (c == 0) {
                //先检查该线程前面是否还有排队的线程，若没有
                //则执行CAS操作将state的值加1
                //若执行CAS操作成功，则当前线程成功获取到锁
                //调用setExclusiveOwnerThread()设置其为锁的拥有者
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //c!=0并且当前线程为锁的拥有者，说明该线程重入获取锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                //将state的值简单的更新就ok
                setState(nextc);
                return true;
            }
            return false;
        }
```

按照我的注释来看，其实这段代码的逻辑也比较简单。而ReentrantLock之所以为重入锁的原因，则关键在于`else if`里的代码。

```java
/**
*此方法将当前线程加入FIFO队列，尝试获取锁
*/
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            //不断的自旋，直到获取锁成功
            for (;;) {
                //获取到当前线程节点的前一个节点
                final Node p = node.predecessor();
                //若前一个节点为头节点，则尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    //获取锁成功，将自己设为队列的head节点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //若没有轮到自己，则检查是否应该被中断
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

整合起来，我们可以总结整个获取锁的过程

1. 先尝试tryAcquire()方法获取锁，成功后直接返回
2. 若没有获取成功，则将自己加入FIFO队列
3. 不断自旋查看自己的排队情况
4. 若排队轮到自己，则尝试tryAcquire()获取锁
5. 若没有轮到自己，则重复第三步

<b>第二种情况—非公平锁

非公平锁的tryAcquire()方法也很简单

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }

final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            //c == 0时减少了对前面是否还有等待线程的判断
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

即我们可以得到非公平锁与公平锁的不同，

* 公平锁：获取锁时先检查前面是否有线程在等待，若有则将自己加入到等待队列中，直到轮到自己，才能获取锁
* 非公平锁：先执行获取锁的操作，获取不到才将自己加入到等待队列中，轮到自己时，才获取锁

## unlock()方法源码分析

对于unlock()方法的分析，在公平锁与非公平锁中，均为一种实现

```java
    public void unlock() {
        sync.release(1);
    }
```

可以看到，unlock()方法中，调用了release()方法，release()方法是AQS中的方法

```java
    public final boolean release(int arg) {
        //调用tryRelease()方法尝试释放锁
        if (tryRelease(arg)) {
            //释放锁成功后唤醒满足条件的线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

这个方法中会先执行tryRelease()释放锁，若释放成功，则获取队列的head线程判断其waitStatus是否不为0，若不为0则唤醒队列中第一个waitStatus<0的线程。接下来我们看看tryRelese()方法的源码

```java
protected final boolean tryRelease(int releases) {
            //首先获取释放锁后state的值
            int c = getState() - releases;
            //当前线程不为拥有锁的线程（那还释放什么，直接抛出异常）
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            //若释放后，state==0说明释放锁之后锁空闲
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            //设置state的值
            setState(c);
            return free;
        }
```

# 总结

整体来看，ReentrantLock实现的方式其实就是通过FIFO队列来保存等待锁的线程。通过AQS类中的state保存锁的持有数量，从而实现可重入性。同时我们也要分清公平锁与非公平锁的区别。

其实不止是ReentrantLock，JUC包下很多类都使用到了AQS类中的东西。AQS类可以说是并发编程的重中之重，我们一定要好好掌握。

***



