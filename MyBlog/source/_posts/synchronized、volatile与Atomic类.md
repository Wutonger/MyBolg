title: volatile关键字与Atomic类
author: Loux
cover: /img/post/4.jpg
tags:
  - 多线程
  - 并发编程
categories:
  - 并发编程
date: 2019-07-27 20:40:00
---
#### 多线程三大特性
说这三个东西之前，先了解一下多线程的三大特性：原子性、可见性、有序性
* 原子性：即一个操作是不可中断的，一旦开始执行直到该操作执行结束其它线程都不能剥夺其执行权  
* 可见性：即一个线程中对于共享变量的修改对于其它线程是可见的，其它线程可以马上获取到被修改后的值，可见性问题仅仅发生在多线程并发执行的情况下，对于串行化执行来说不存在该问题
* 有序性：在并发时，程序的执行可能会出现乱序，给人的直观感觉就是写在前面的代码，会在后面执行。有序性问题的原因是因为程序在执行时，可能会进行指令重排，重排后的指令与原指令的顺序未必一致  

#### volatile关键字
在讲volatile之前，先讲一下java中线程是怎么修改共享变量的值的。
1. 将主内存中的变量拷贝一份副本到线程自己的工作空间(cpu缓存)
2. 线程工作空间中拿到该变量的副本值进行修改等操作，修改完毕后又将变量新的值赋给工作空间的副本
3. 线程执行完毕后会将线程修改的值刷回主内存
简单的画个图可以表示为：

![线程修改共享变量](/images/pasted-11.png)
也就是说多线程环境下，每个线程中修改的其实是线程工作空间拷贝的变量副本，并且彼此并不知道其它线程将该值修改到了多少，这就造成了一个很尴尬的局面—最后变量的值可能和我们所预期的不一样。我们来看一个简单的例子
```bash
public class VolatileDemo implements Runnable {
    private static boolean isRunning = true;
    public void run() {
       while (isRunning){
         //不执行任何代码
       }
    }
    public static void main(String[] args) {
        new Thread(new VolatileDemo()).start();
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        isRunning = false;
        System.out.println("已将isRunning的值设置为false");
        System.out.println(isRunning);
    }
}
```
执行结果如下（等待10s后）：
![运行结果](/images/pasted-12.png)

我们会发现即使主线程在休眠3s后已经将isRunning的值修改为了false,但是程序依然没有停止运行，这正是由于线程间对于变量的修改不可修改导致的。

那么我们就像有没有一种方式能让每个线程对变量的修改对于其它线程都是可见的呢？答案是当然有啦，我们能想到的Java早就已经想到了。使用<b>volatile关键字</b>修饰的变量能保证在线程中修改该变量的值后立马将其刷回主存，若其它线程中也存在该变量的副本，会通知其它线程“你们工作空间中的副本已经过时了，无效了”，其它线程发现自己工作空间中的副本值无效后，会从主内存中获取该变量的最新值，这样也就保证了可见性。同样是上一个例子，我们修改isRunning
> private volatile static boolean isRunning = true;  

再次运行程序后发现当isRunning被修改为false后程序正确停止了

![运行结果](/images/pasted-13.png)
这也就说明当主线程修改了isRunning的值为false后,另外一个线程对于这个值是可见的。但是我们要注意的是volatile只能保证变量的可见性，而<b>不能保证原子性</b>,即表示线程执行到一半或者刚获取到副本的值时cpu资源可以被其它线程剥夺。举个简单的例子，我们都知道count++不属于原子操作，它通常分为三个步骤，第一步拿到count的值,第二步执行count+1，第三步将第二步运算的结果赋值给count。那么用volatile修饰count，并在线程中执行count++会是怎样的呢？这里给出了一个JavaDemo的示例
```bash
public class VolatileCountDemo implements Runnable{
    private volatile static int count=0;
    public void run() {
        for (int i = 0; i <1000 ; i++) {
            count++;
        }
    }
    public static void main(String[] args) throws InterruptedException {
        for (int i= 0;i<10;i++){
            new Thread(new VolatileCountDemo(),"线程"+i).start();
        }
        while(Thread.activeCount()>2) { //确保10个线程都执行完毕
            Thread.sleep(100);
        }
        System.out.println("count的值为"+count);
    }
}

```
运行结果如下：
![运行结果](/images/pasted-14.png)
按照预期来说最后输出count的值应该为10000，但是运行了几次后发现结果都比10000小，要么是8000多，要么是9000多。这是为什么呢？这正是由于volatile不能保证原子性导致的。例如某个时刻count的值为1000,线程A正在执行将count++操作，当线程A的执行引擎刚获取到count的值后，cpu的执行权立马给了线程B,线程B正常的执行完count++后将修改的count值刷回主内存，这时count的值为1001，线程A也确实更新到了最新的count值了，但是由于在B执行前线程A的执行引擎已经获取到count的值了，所以不会重新取一次，而会继续计算count++操作的第二、第三步，这会造成线程A也将count修改为1001并刷新回主存覆盖掉线程B修改的1001，这样本来count的值应该为1002的，而实际结果只有1001。这样也就解释了为什么执行多次的结果后count的值均比10000小。此时需要保证count++操作的原子性，我们只能用加锁的方法了。将run()方法修改为：
>   static Object  object = new Object();  //该锁对象被定义的类创建的线程锁共享
    public  void run() {
        for (int i = 0; i <1000 ; i++) {
            synchronized (object) {
                count++;
                }
                ......

再次运行程序后，我们会发现无论运行多少次，最终打印的结果都是10000。但其实对于这种情况，除了加锁外，还可以用另外一种方式来保证其操作的原子性，那就是从jdk1.5开始在JUC包下提供的Atomic类系列

#### Atomic类
在java 1.5的java.util.concurrent.atomic包下提供了一些原子操作类，即对基本数据类型的 自增（加1操作），自减（减1操作）、以及加法操作（加一个数），减法操作（减一个数）进行了封装，保证这些操作是原子性操作。atomic是利用CAS来实现原子性操作的（Compare And Swap）。现在同样是上面那个例子，如果将count改为用AtomicInteger类来操作会有什么结果呢？Java代码示例如下
```bash
public class AtomicDemo implements Runnable{
    private  static AtomicInteger count= new AtomicInteger(0);
    static Object  object = new Object();
    public  void run() {
        for (int i = 0; i <1000 ; i++) {
            count.incrementAndGet();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        for (int i= 0;i<10;i++){
            new Thread(new AtomicDemo(),"线程"+i).start();
        }
        while(Thread.activeCount()>2) {
            Thread.sleep(100);
        }
        System.out.println("count的值为"+count);
    }
}
```
程序运行结果如下：

![AtomicDemo运行结果](/images/pasted-15.png)
运行5次程序后，发现每一次打印的结果都为10000，和我们预期的结果一样。保证了线程操作的安全性。但是值得注意的是Atomic包下的类只能保证自身方法是原子的，即也就是说多个方法的调用并不能保证其原子性。
<b>什么是CAS（Compare And Swap）?</b>
Compare And Swap中文翻译过来就是比较并变换。CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可”。Java并发包(JUC)中大量使用了CAS操作,涉及到并发的地方都调用了sun.misc.Unsafe类方法进行CAS操作。
下面我们就来看看当执行count.incrementAndGet()方法时发生了什么......
```bash
     /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
    
    //以下为unsafe类中的方法
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
    
     /**
     * 获取obj对象中offset偏移地址对应的整型field的值。
     * @param obj 包含需要去读取的field的对象
     * @param obj中整型field的偏移量
     */
     public native int getIntVolatile(Object obj, long offset);
     
     /**
     * 比较obj的offset处内存位置中的值和期望的值，如果相同则更新。此更新是不可中断的。
     * @param obj 需要更新的对象
     * @param offset obj中整型field的偏移量
     * @param expect 希望field中存在的值
     * @param update 如果期望值expect与field的当前值相同，设置filed的值为这个新值
     * @return 如果field的值被更改返回true
     */
     public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```
第一次count 为0时线程A调用incrementAndGet时，传参为 var1=AtomicInteger(0)，var2为var1 里面 0 的偏移量，比如为9000，var4为需要加的数值1,var5为线程工作内存值，do里面会先执行一次，通过getIntVolatile 获取obj对象中offset偏移地址对应的整型field的值，此时var5=0，while里面compareAndSwapInt比较obj的9000处内存位置中的值和期望的值var5，如果相同则更新obj的值为(var5+var4=1)，此时更新成功，返回true,则 while结束循环，return var5。  

当count为0时，线程B和线程A同时读取到count的值，线程B也是取到的var5=0,当线程B执行到compareAndSwapInt时，线程A已经执行完compareAndSwapInt，已经将内存地址为9000处的值修改为1，此时线程B执行compareAndSwapInt返回false,则继续循环执行do里面的语句，再次取内存地址偏移量为9000处的值为1，再去执行compareAndSwapInt，更新obj的值为(var5+var4=2)，返回为true,结束循环，return var5。  

<b>最后值得注意的是</b>CAS只是关注了比较的前后值是否改变，而无法清楚比较过程中变量的变更明细，假若一个变量初次读取是A，在compare阶段依然是A，但其实可能在此过程中，它先被改为B，再被改回A，而CAS是无法意识到的。这就是我们所说的ABA问题。解决的办法也很简单，再加一个版本号变量，每次修改变量值的时候版本号的变量都会自增1，并且在compare阶段的时候也会加上对版本号的比较。例如AtomicStampReference类就是采用的这种方案，它的核心方法是compareAndSet()方法，而方法中多了一个对stamp的比较，就是对变量版本号的比较，stamp的值也是在每次变量进行更新时进行维护。
#### 总结
volatile关键字保证了多线程的可见性、有序性，但不保证其原子性
Atomic类保证了多线程的原子性