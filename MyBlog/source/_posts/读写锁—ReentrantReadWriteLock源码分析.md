title: 读写锁—ReentrantReadWriteLock 源码分析
author: Loux

cover: /img/post/26.jpg

tags:

  - 源码分析
categories:
  - 并发编程
date: 2019-11-18 14:42:00

---

前面我们分析了ReentrantLock类的实现，可以发现它的上锁策略跟Synchronized一样，都是十分保守的，不管要做什么操作，先把锁加上再说。但是我们知道，线程安全问题主要是由于对线程共享资源的写入操作导致的。倘若某一个程序，只有少量的写入操作，有大量的读取操作，这个时候使用ReentrantLock加锁，不管是读还是写，都会互斥，但是我们需要的仅仅是读/写，写/写互斥，而读/读操作并不需要互斥，这样在这个程序中由于使用了ReentrantLock会导致性能不必要的大大降低。那么有没有一种锁可以实现对读写两种不同的操作采用不同的互斥策略呢？当然有了，它就是ReentrantReadWriteLock。

# ReentrantReadWriteLock简介

ReentrantReadWriteLock为读与写操作提供了不同的互斥策略，写锁的互斥性更高而读锁的互斥性更低，这样的设计在一些适当的条件下可以大大的提高程序的性能，例如上述所举得有大量读操作与少量写操作的程序。但是相对而言，ReentrantReadWriteLock的内部实现也更加的复杂，锁的复杂度要高于ReentrantLock，所以在读操作不多时，效率会低于ReentrantLock，所以对于这两种锁的选用，我们一定要在不同的情况进行具体的分析。

同时我们还需要了解锁升级与锁降级这两种概念

<b>锁升级</b>

如果当前线程持有读锁后，可以不释放读锁就获得写锁，就是锁升级。显而易见，ReentrantReadWriteLock是不支持锁升级的， 原因在于：必须确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么读操作读取到的资源可能就不是一个最新的状态，这就违反了happen-before原则。

<b>锁降级</b>

如果当前线程持有写锁后，可以不释放写锁而获取读锁，就是锁降级，ReentrantReadWriteLock是支持锁降级的，因为这并不会影响线程安全。

关于ReentrantReadWriteLock的使用这里就不在这里细说了。本文主要是分析它的两种不同的加锁、释放锁的策略。

# ReentrantReadWriteLock结构

首先我们来看一下ReentrantReadWriteLock类大体的结构

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    
   private final ReentrantReadWriteLock.ReadLock readerLock;
    
   private final ReentrantReadWriteLock.WriteLock writerLock;
    
   final Sync sync;
    
    //无参构造-默认非公平锁
    public ReentrantReadWriteLock() {
        this(false);
    }
    
    //有参构造
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
    
    //ReadLock内部类
    public static class ReadLock implements Lock, java.io.Serializable {
        
        private final Sync sync;
        
        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
        
        //读锁的加锁使用了共享加锁模式
        public void lock() {
            sync.acquireShared(1);
        }
        
        public void unlock() {
            sync.releaseShared(1);
        }
        ......
    }
    
    //WriteLock内部类
    public static class WriteLock implements Lock, java.io.Serializable {

        private final Sync sync;
        
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
        
        //写锁加锁使用了独占模式
        public void lock() {
            sync.acquire(1);
        }
        
        public void unlock() {
            sync.release(1);
        }
        ......
    }
    
}
```

我们可以发现，ReentrantReadWriteLock类中有两个内部类ReadLock与WriteLock，而两个内部内中Sync实例实际上使用的还是外部传进来的sync实例。也就是说两个锁实际上操作的其实是同一个Sync对象。

而Sync类、FairSync与NonFairSync与AQS的关系就不细说了，跟上次分析的ReentrantLock一样。

而ReadLock中的lock方法，调用了AQS类中的`acquireShared()`方法，WriteLock中的lock方法，调用了AQS中的`acquire()`方法，一个是共享，一个是独占，实际上还是回归到了AQS类中。

同时我们已经知道了AQS中的内部属性state是加锁与释放锁的关键。

对于独占模式来说，当state!=0且持有锁的线程不为当前线程时，当前线程不能获取到锁，而对于共享模式来说，不管state是否为0，其它线程均可以获取到锁。也就是说两种模式对于state变量的操作不一样。一个变量是怎么进行两种不同的判断操作方式呢？用一般的方式肯定是不可行的，Sync类中将state的值分为了高16位与低16位(int类型的变量一共32位)，其中高16位的值用于ReadLock(共享模式)，低16位的值用于WriteLock(独占模式)。

![state变量实现独占、共享双操作](/images/image-20191118164501020.png)

