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

而FairSync与NonFairSync与AQS的关系就不细说了，跟上次分析的ReentrantLock基本一样。

而ReadLock中的lock方法，调用了AQS类中的`acquireShared()`方法，WriteLock中的lock方法，调用了AQS中的`acquire()`方法，一个是共享，一个是独占，实际上还是回归到了AQS类中。

同时我们已经知道了AQS中的内部属性state是加锁与释放锁的关键。

对于独占模式来说，当state!=0且持有锁的线程不为当前线程时，当前线程不能获取到锁，而对于共享模式来说，不管state是否为0，其它线程均可以获取到锁。也就是说两种模式对于state变量的操作不一样。一个变量是怎么进行两种不同的判断操作方式呢？用一般的方式肯定是不可行的，Sync类中将state的值分为了高16位与低16位(int类型的变量一共32位)，其中高16位的值用于ReadLock(共享模式)，低16位的值用于WriteLock(独占模式)。

![state变量实现独占、共享双操作](/images/image-20191118164501020.png)

# 源码分析

## Sync类主要内容分析

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 以下的内容就是将state设置为高16位与低16位两部分
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    // 取读锁的获取次数，包括重入（高16位）
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    //取写锁的重入次数，0代表写锁为可获取状态（低16位）
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    // 这个类用来保存当前线程读锁的重入次数
    static final class HoldCounter {
        // 持有读锁数量（重入次数）
        int count = 0;
        // 线程 id
        final long tid = getThreadId(Thread.currentThread());
    }

    static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
        //获取一个初始化的HoldCounter对象
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    
    //readHolds用来记录当前线程持有的读锁数量
    private transient ThreadLocalHoldCounter readHolds;

    // 记录最后一个读锁线程的数量，用于提高性能
    private transient HoldCounter cachedHoldCounter;

    // 第一个获取读锁的线程
    private transient Thread firstReader = null;
    //第一个获取读锁线程持有的读锁数量
    private transient int firstReaderHoldCount;

    Sync() {
        // 初始化 readHolds 这个 ThreadLocal 属性
        readHolds = new ThreadLocalHoldCounter();
        // 为了保证 readHolds 的内存可见性
        setState(getState()); // ensures visibility of readHolds
    }
    ...
}
```

为什么对于`读锁`来说，每个线程都需要维护一个`ThreaLocalHoldCounter`对象呢？因为我们知道了读锁是共享锁，可以被多个线程获取，并且每个线程也能重入，所以为了判断线程获取读锁是不是重入，就必须记录每个线程拥有读锁的个数。

## 读锁lock()方法

ReadLock的`lock()`方法实现如下：

```java
        public void lock() {
            sync.acquireShared(1);
        }
```

在lock()中调用了AQS类中的`acquireShared()`方法

```java
    public final void acquireShared(int arg) {
        //若获取读锁失败，执行doAcquireShared()方法
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

这里判断了tryAcquireShared(arg) 执行的结果是否小于0，若小于0则执行doAcquireShared(arg)方法，那么我们先来看看Sync类中tryAcquireShared()的源码：

```java
        protected final int tryAcquireShared(int unused) {
            //获取当前线程
            Thread current = Thread.currentThread();
            //获取state的值便于后续获取读锁与写锁获取的次数
            int c = getState();
            //有线程持有写锁且该线程不为当前线程，获取读锁失败
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            //获取读锁的获取次数
            int r = sharedCount(c);
            /**
            *是否应该被阻塞，若不被阻塞则继续判断
            *读锁获取的次数是否会溢出(2^16-1),若不会溢出则执行CAS操作
            *compareAndSetState(c, c + SHARED_UNIT)是将state的高16位加1
            *若执行成功说明获取读锁成功了
            **/
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                //r==0,说明当前线程是第一个获取读锁的线程
                //设置firstReader与firstReaderHoldCount
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                //若r!=0,且当前线程为firstReader,将其拥有锁的次数加1
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    //当前线程不为第一个获取锁的线程，将其线程id与获取锁的次数进行缓存
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                //获取读锁成功
                return 1;
            }
            //若当前线程需要block || 读锁的值快溢出 || CAS操作执行失败
            return fullTryAcquireShared(current);
        }
```

这段代码也很好理解，关键在于最后执行`fullTryAcquireShared(current)`的条件

有三种情况：

第一种：当前线程需要被block住。在公平锁中，若前面还有线程在队列中等待，则需要被block；在非公平锁中，若head节点(已获取到锁的节点)的后面一个节点是否来获取写锁的，如果是，让它先获取，在这里我们可以看出写锁的优先权是要比读锁高的。

第二种情况：读锁获取的次数已经要溢出了，这种情况发生的很少。

第三种情况：CAS操作执行失败，CAS失败的原因可能是读锁进行竞争，也有可能是和写锁进行竞争。

fullTryAcquireShared()方法中，会继续尝试获取读锁，源码如下：

```java
final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            //开始自旋尝试获取锁，若CAS失败，会一直自旋
            for (;;) {
                int c = getState();
                //若其它线程已持有了写锁,直接返回-1
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                } else if (readerShouldBlock()) {
                    /**
                    *exclusiveCount(c)==0，写锁空闲
                    *readerShouldBlock()为true，说明阻塞队列中有其它线程在等待
                    *此时向下继续执行的原因是为了处理重入锁
                    *若该线程已经有读锁存在，即使readerShouldBlock()为true
                    *依然会尝试CAS获取锁
                    **/
                    if (firstReader == current) {
                        
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            /**cachedHoldCounter中缓存的不是当前线程
                            *那么获取当前线程的holdCounter
                            *若当前线程获取读锁的次数为0，说明不为重入操作
                            *既然不为重入操作，那么当然会返回-1了
                            **/
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                //读锁持有已到上限了，抛出异常
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                //尝试CAS获取读锁,若成功进行下面的逻辑
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    //若CAS执行成功前读锁次数为0，则将当前线程设置为firstReader
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        //当前线程已经是firstReader,将持有锁的次数加1
                        firstReaderHoldCount++;
                    } else {
                        //不为当前线程，则将其缓存进cachedHoldCounter中
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    //获取锁成功
                    return 1;
                }
            }
        }
```

上述代码注释的很详细了，其实也很好理解，就是获取读锁失败后尝试自旋再次获取读锁的过程。

那么我们知道了tryAcquireShared(arg)<0，会执行`doAcquireShared(arg)`方法，那么这个方法又是用来干什么的呢？其实就是将当前线程包装为节点加入FIFO等待队列，这里就不细展开了，关于加入等待队列的代码在ReentrantLock中有类似的。

## 读锁unlock()方法

ReadLock调用unlock()方法整个流程源码如下：

```java
        public void unlock() {
           sync.releaseShared(1);
        }
    
        //AQS类中的方法
        public final boolean releaseShared(int arg) {
            if (tryReleaseShared(arg)) {
                //若释放读锁成功后state==0，则唤醒后继节点获取写锁的线程
                doReleaseShared();
                return true;
            }
            return false;
        }

        protected final boolean tryReleaseShared(int unused) {
            //获取当前线程
            Thread current = Thread.currentThread();
            /**
            *若当前线程为firstReader,且持有锁的次数为1，则将firstReader置为null
            *若持有锁的次数不为1，则将其减1
            **/
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                /**
                *当前线程不为firstReader,则判断是否为最后一个获取读锁的线程
                *若不是，则到ThreadLocal中取出holdCounter对象
                **/
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                //若当前线程获取读锁的数量小于等于1，则将ThreadLocal删除掉
                //因为这次释放锁后当前线程就没有持有读锁了
                if (count <= 1) {
                    readHolds.remove();
                    //这种情况是由于lock一次，unlock多次出现的
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                //将count减1
                --rh.count;
            }
            //这里通过自旋尝试将state的高16位减一，若不成功，则一直CAS
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    /**若CAS成功后，state==0了，说明没有线程持有读锁写锁了
                    *此时return true有助于执行后续的doReleaseShared()方法
                    *唤醒后面节点获取写锁的线程，这里再次说明，写锁获取有优先权
                    **/
                    return nextc == 0;
            }
        }
```

需要注意的是，在释放读锁的过程中，若当前线程释放这一次后读锁为0了，要将ThreadLocal中存储的删除；并且当CAS执行成功后state的值为0，要唤醒后面节点中获取写锁的线程。

<b>写锁由于和ReentrantLock一样是独占锁，lock与unlock的方式与其类似。这里就不分析了，有兴趣可以自己再分析一下</b>

# 总结

本次分析源码的过程中，也在网上看了不少优质文章对于ReentrantReadWrite的分析思路，给了我很大的帮助，在这里由衷的感谢。另外由于本人水平有限，文中的分析有错误的地方还请多批评指正。

