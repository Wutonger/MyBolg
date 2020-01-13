title: Java线程池的原理与使用—面试必备
author: Loux

cover: /img/post/24.jpg

tags:

  - 并发编程
  - 源码分析
categories:
  - 并发编程
date: 2020-01-13 09:57:00

---

我们为什么要使用多线程处理程序？答案当然是为了提高程序的性能。也就是说我们为每一个程序都创建一个线程，让多个线程来执行多个任务。

但是频繁的创建线程是有很大的弊端的。例如

* 线程的创建需要时间和资源，如果一个线程运行的任务非常简单，可能会导致创建线程的时间比执行任务的时间还要长，这当然是我们不愿意看到的。
* 若创建的线程数量过多，会导致大量线程竞争CPU资源，频繁的切换线程，造成系统的额外开销。若采用非公平锁，还有可能某些线程永远都无法竞争到CPU。
* 系统能容纳的线程数量是有上限的，超过这个上限，应用就会崩溃。

回想一下，以前我们是不是也遇到过类似的问题。例如数据库连接问题，频繁的连接、关闭数据库也会导致性能下降，当时我们使用了数据库连接池，在池子中预先创建了一些数据库连接，当程序使用时就拿出一个连接，使用完毕后就放回，若池子中没有连接了，就进行等待。

是不是有一种类似的方法能处理频繁创建线程的问题呢？当然有了，JDK为我们提供了线程池来解决这一问题。

# 什么是线程池

线程池的作用是维护一定数量的线程，并能接收任意数量的任务，这些任务会被池子中的线程并发执行，若任务数量大于池中线程数量，线程会执行多次任务，直到任务全部被执行完毕为止。

Java中为我们提供了一整套Executor框架来使用线程池。这里给出一个简单的使用案例

```java
 public class JDKExecutorClient {
    public static ExecutorService executor = Executors.newFixedThreadPool(5);
    
    public static void main(String[] args) {
        Stream.iterate(1, item -> item + 1).limit(10).forEach(item -> {
                    executor.execute(() -> {
                       System.out.println(Thread.currentThread().getName() + "执行任务结束!");
                    });
                }
        );
    }
 }
```

这里我们创建一个有5个线程的线程池，并且循环添加了10个任务，来看最后的执行结果

![线程池执行结果](/images/image-20200113102820048.png)

# Executor结构

关于线程池相关类的继承结构如下：

![](/images/image-20200113120839438.png)

**Executor**接口是最顶层接口，只有一个添加任务的方法`execute(Runnable runnable)`

**ExecutorService**接口为Executor的下层接口，顾名思义，它在Executor的基础上新增了很多的方法

**AbstractExecutorService**提供了ExecutorService的部分实现，供子类调用

**ThreadPoolExector**就是我们的线程池了，它提供了关于线程池所需要的许多功能

刚刚我们实例中使用的`Executors.newFixedThreadPool(int args)`方法属于Executors类，加"s"也就是工具类，里面提供了很多静态方法供我们调用。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

我们可以看到，工具类的方法在创建线程池的时候其实还是调用了ThreadPoolExecutor类的构造方法。

# 源码分析

## Executor接口

```java
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```

可以看到，Executor接口就只有一个`void execute(Runnable command)`方法，根据上面的注释可以知道这个方法添加了一个任务，这个任务会被执行。

## ExecutorService接口

```java
public interface ExecutorService extends Executor {
    /**
    *关闭线程池，不接收新添加的任务，已添加的任务会执行完成
    **/
    void shutdown();

    /**
    *强制关闭线程池，并尝试关闭正在执行的所有任务，返回从未被执行任务的列表
    **/
    List<Runnable> shutdownNow();

    /**
    *线程池是否关闭
    **/
    boolean isShutdown();
    
    /**
    *提交一个task任务，返回的Future对象持有任务执行结果
    **/
    <T> Future<T> submit(Callable<T> task);

    /**
    *提交一个runnable任务，第二个参数作为返回值放入Future中
    **/
    <T> Future<T> submit(Runnable task, T result);

    /**
    *执行所有任务，返回Future类型的一个list
    **/
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    
    /**
    *和上面一个方法不同的是设置了超时时间
    **/
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
    *执行任务，任意一个任务有返回时返回该任务的执行结果，未执行的则被取消
    **/
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

   /**
   *和上面一个方法不同的是设置了超时时间
   **/
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

我们可以看到ExecutorService定义了很多功能，提交、执行任务，还能获取任务执行结果扽等等。所以说一般我们在定义线程池时最好使用ExecutorService接口，这样可以有多种操作线程池的方式，若使用Executor则只有一个execute()方法用于添加任务。

## AbstractExecutorService类

这个抽象类实现了ExecutorService部分的方法，供子类进行调用。

```java
public abstract class AbstractExecutorService implements ExecutorService {
    
    /**
    *将runnable包装为FutureTask，使得任务执行完成有返回值
    **/
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    
    /**
    *提交任务
    **/
    public Future<?> submit(Runnable task) {
        //提交的任务为空，则直接抛出NullPointerException
        if (task == null) throw new NullPointerException();
        //将任务包装为FutureTask
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        //提交、执行任务
        execute(ftask);
        return ftask;
    }
    
    /**
    *同样是提交任务，只是这个任务具有返回值
    **/
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        //包装为FutureTask
        RunnableFuture<T> ftask = newTaskFor(task, result);
        //提交、执行任务
        execute(ftask);
        return ftask;
    }    
    

}
```

这里我们只了解常用的方法就OK了。我们发现submit方法中并没有真正的开启一个线程来执行任务，只是在内部调用了`execute()`方法，也就是说execute()这个核心方法的实现还没有出现。根据上面的Executor结构可以看出，实现它的正是ThreadPoolExecutor类

## ThreadPoolExecutor类

ThreadPoolExecutor类是核心，同时也是线程池具体的实现。它实现了一个线程池所需要的各种方法，例如任务提交、线程管理等。

首先我们要清楚一点，execute(Runnable command)方法中的Runnable对象并不是用于new Thread(runnable...).start()，这样就违背了线程池设计的初衷。还记得我们说过吗，维护一定数量的线程，这些线程会获取到runnable的run()方法并执行。这是刚接触线程池的同学特别容易混淆也是很重要的一个概念，它是线程池设计的精髓。

这里我画了一个图，根据图中的描述我们能知道线程池大概的工作原理

![](/images/image-20200113154524322.png)

我们再来看一下ThreadPoolExecutor的构造方法

```java
/**
*构造方法只是设置了参数，并没有启动任何线程
*这是很好的设计，当没有添加任务就启动线程是在浪费系统资源
**/
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

对于构造方法我们可以根据方法上的注释来了解参数的含义

corePoolSize — 核心线程数

maximumPoolSize — 最大线程数，即线程池所允许创建的最大线程的数量

workQueue — 一个阻塞队列，用来保存线程池要执行的任务

keepAliveTime — 空闲线程的存活时间，当然如果关闭该线程后线程池中线程数量少于了corePoolSize，则空闲线程不会被关闭

threadFactory — 用于生成线程，一般我们使用默认的`Executors.defaultThreadFactory()`

handler — 当线程池中任务队列已经装满，又有新的任务提交，采取的策略由这个参数决定，例如抛出异常、直接拒绝等等。

我们还需要知道ThreadPoolExecutor中一条重要的属

`private final HashSet<Worker> workers = new HashSet<Worker>();`

这就是线程集合，其中Worker是对Thread的封装。

接下来我们来分析核心方法— execute(Runnable command)方法

```java
    public void execute(Runnable command) {
        //若任务为空，则直接抛出NullPointerException
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        //ctl:高3位存放线程状态，低29位存放线程数量
        int c = ctl.get();
        //若此时运行的线程数量少于corepoolSize，则尝试创建新的线程
        //并将command作为它的第一个task来执行
        if (workerCountOf(c) < corePoolSize) {
            //添加成功后直接返回了
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //要么当前线程数大于等于coolpoolSize,要么添加线程失败了
        //若线程处于Running状态，则将这个任务添加到任务队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            //加入任务队列成功后再次判断是否为running状态，若不为running状态了
            //则移除该任务，并且执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //若线程数量已经为0，则创建新的线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //若任务队列满了，添加不了任务，则尝试创建线程
        //若创建线程失败，执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

我们发现，execute方法中多次调用了addWorker()方法来创建线程，下面我们来看一下这个方法的源码

```java
    //参数1：添加给这个新建线程执行的任务，可以为null
    //参数2：选用创建线程的界限 true代表corePoolSize false代表maximumPoolSize
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

