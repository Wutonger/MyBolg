title: Java线程池的原理与使用—面试必备
author: Loux

cover: /img/post/30.jpg

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
            //获取到线程池状态
            int c = ctl.get();
            int rs = runStateOf(c);

            //若线程池状态大于等于Shutdown(stop,ditying,terminated)
            //或firstTask != null 或 workQueue.isEmpty()
            //则不创建新的线程，直接返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //获取到线程数量
                int wc = workerCountOf(c);
                //线程数量是否大于最大数量 或 是否大于界限
                //若条件满足一个，则直接返回false
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //否则就可以创建线程准备执行任务了
                //若cas成功，则直接跳出整个循环
                //若cas失败说明有其它线程也在尝试在线程池中创建线程
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                //由于并发原因，可能ctl的值已经被改变，重新获取一下
                c = ctl.get();  // Re-read ctl
                //若线程池的状态改变，再进行一次外层retry循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        
        //此时，我们已经可以创建线程了
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //新建一个worker(线程)，并赋予它task
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                //先加锁，保证线程安全
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    //获取线程池的状态
                    int rs = runStateOf(ctl.get());
                    //判断线程池的状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //若新建线程为启动状态，直接抛出异常（我才创建，都没有启动）
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        //将新worker添加进HashSet
                        workers.add(w);
                        int s = workers.size();
                        //这里是为了记录线程池中线程的最大值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        //workerAdded=true，说明已经添加成功了
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //添加成功后启动线程，并设置workerStarted=true
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //若启动线程失败，则执行addWorkerFailed(w)方法
            //其实里面的逻辑就是做清理工作，因为线程添加成功了但是启动失败了，肯定要清除添加的线程
            if (! workerStarted)
                addWorkerFailed(w);
        }
        //最后返回线程是否启动的boolean值
        return workerStarted;
    }
```

我们来总结一下addWorker方法中的逻辑：

第一步：首先查看线程池状态是否异常，若状态异常则直接返回false

第二步：获取线程数量，判断线程数量是否超过限制，若超过则直接返回false

第三步：以CAS的方式更新线程数量，然后新建worker对象加入HashSet并启动线程，启动失败执行清除操作，最后的返回值是用来判断线程是否启动的boolean值

现在我们已经知道了线程池是怎样创建线程，怎样添加任务的了。那么workQueue队列中的任务又是怎么被执行的呢？还记得我们说了么，JDK将线程池中的线程封装为了Worker对象，要想知道如何执行，还需要看Worker的源码。

```java
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable
    {
       
        final Thread thread;
        
        Runnable firstTask;
        
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
    }
```

这里列出了一些主要的属性和构造方法。我们可以看到Worker中的构造方法接收了一个task，并且用`getThreadFactory().newThread(this)`方法创建的thread，也就是说是通过工厂创建的，还将Worker对象本身作为Runnable的实现传进了newThread(Runnable command)方法中，因为Worker是实现了Runnable接口的。也就是说在addWorker()方法中启动线程执行run()方法，其实就是执行Worker对象的run方法。而Worker类中run()方法如下：

```java
        public void run() {
            runWorker(this);
        }
```

接下来我们需要继续看ThreadPoolExecutor类中的runWorker(Runnable command)方法

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        //获取Worker对象的firstTask,可以为null
        Runnable task = w.firstTask;
        //获取成功后将Worker对象的firstTask设置为null，防止任务重复执行
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //若没有firstTask则从workQueue中取出task
            while (task != null || (task = getTask()) != null) {
                //加锁
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //执行task的run方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    //执行完成后清空task，进行下一次while循环getTask()
                    task = null;
                    //该worker对象执行完成的任务次数加1
                    w.completedTasks++;
                    //释放锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //当没有workQueue中没有task可以执行 || 发生异常，调用processWorkerExit()方法
            processWorkerExit(w, completedAbruptly);
        }
    }
```

这段代码其实就是循环从workQueue中取task然后执行，大意很好理解。但是里面的一些细节我还是不太懂，希望以后能够解决。

# 总结

到这里为止，线程池中创建线程、添加任务、执行任务的源码差不多已经分析完毕了。可能有一些地方表述不正确或不准确，还请指正。

接下来对文中的一些比较重要的、面试中常常问到的进行一些总结。

一、线程池中创建线程的时机有哪些？

> * 如果当前线程数< corePoolSize时 ,在每次添加任务的时候会新建线程，此时界限为corePooSize
> * 当线程数>corePoolSize时，会将任务放入队列中，此时会重新获取线程的数量，若为0，则会创建线程，此时界限为maximumPoolSize
> * 若任务队列已满，会尝试创建线程，此时界限为maximumPoolSize

二、什么时候会对任务采取拒绝策略？

> * 在添加任务时线程数量达到了corePoolSize,会将任务入队列，但此时线程池不为running状态了，那么会将队列中的任务移除并执行拒绝策略
> * 在添加任务时，线程数大于corePoolSize并且任务队列已经满了，此时会尝试创建线程，若线程数量已经达到了maximumPoolSize，则创建线程会失败，并对任务进行拒绝策略

三、线程池中有哪些重要属性

>* corePoolSize — 核心线程数
>
>* maximumPoolSize — 最大线程数，即线程池所允许创建的最大线程的数量
>
>* workQueue — 一个阻塞队列，用来保存线程池要执行的任务
>
>* keepAliveTime — 空闲线程的存活时间，当然如果关闭该线程后线程池中线程数量少于了corePoolSize，则空闲线程不会被关闭
>
>* threadFactory — 用于生成线程，一般我们使用默认的`Executors.defaultThreadFactory()`

最后本文参考了文章<a href="https://javadoop.com/post/java-thread-pool">深度解读 java 线程池设计思想及源码实现</a>