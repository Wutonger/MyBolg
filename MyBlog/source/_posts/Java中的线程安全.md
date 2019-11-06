title: Java中的线程安全
author: Loux
cover: /img/post/11.jpg
tags:
  - 多线程
  - 并发编程
categories:
  - 并发编程
date: 2019-07-09 19:38:00
---
为了提高程序运行的效率，Java为我们提供了多线程编程。而在多线程编程中，访问一个对象的多个线程由于存在对共享数据的操作，会产生线程不安全的问题，而这个时候就需要人为的对其进行控制。
#### 线程安全
正如上述所说，如果多线程编程中不能保证线程安全，那么程序每一次的运行结果都将是不确定的，这样当然不符合我们的设计结果。那么什么是线程安全呢？官方给出的定义如下：  
> 多个线程访问同一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他操作，调用这个对象的行为都可以获得正确的结果，那么这个对象就是线程安全的。

通俗来解释说，也就是多个线程操作的结果与单个线程操作结果相同，这样的对象就是线程安全的。  
举个例子，ArrayList集合在进行添加元素的时候，可能会分为两个步骤。第一步在item[size]的位置存储该元素的值，第二步则是增大size的值。  
在单线程的情况下，如果 Size = 0，添加一个元素后，此元素在位置 0，而且 Size=1。这样执行的结果是正确的，并且是符合我们预期的。  
而在多线程的情况下，例如有线程A和线程B，线程 A 先将元素存放在位置 0。但是此时 CPU 调度线程A暂停，线程 B 得到运行的机会。线程B也向此 ArrayList 添加元素，因为线程A还没来得及执行size加1的操作，所以此时 Size 仍然等于 0 ，所以线程B也将元素存放在位置0。然后线程A和线程B都继续运行，都增加 Size 的值。而结果是元素实际上只有一个，存放在位置 0，而 Size 却等于 2。这就是“线程不安全”了。  
通过上面的小例子，我们可以总结到线程不安全需要满足的条件如下：
* 多个线程在操作共享的数据
* 对共享数据的操作不为读操作  
而其实对于上面的实例，我们可以进行人为的控制，例如将存储元素操作与增大size值的操作绑定到一起，形成原子操作。这样每个线程都必须完成这个原子操作后才会让出CPU的执行权，这样就达到了线程安全的目的。那么解决线程安全问题可以有哪些方法呢？  

##### synchronized关键字
使用Synchronized关键字可以保证线程安全,它其实是给要执行的代码或者方法加上了同步锁，只有获得该锁的线程才能执行，而没有锁的线程只能进行等待，直到执行的线程释放锁。而synchronized同步锁在实际应用中可以分为以下两种
* 同步代码块
* 同步方法  

<b>同步代码块</b>：即对某一个代码块加同步锁，只有获得锁的线程才能执行该代码块的内容（不过对于代码块外的代码，不受锁的限制）。对于一个售票的demo，利用同步代码块保证线程安全的Java代码示例如下：
```java
class SaleTicket implements Runnable {
    Object object = new Object();
    //共享数据
    private  int ticketNum = 100;
    public void run() {
            while (true) {
                synchronized (object) {
                    if (ticketNum > 0) {
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
    //ticketNum--操作其实分为3个步骤，所以为线程不安全的，必须手动保证                   
                        System.out.println("线程" + Thread.currentThread().getName() + "销售电影票" + ticketNum--);
                    }
                }
            }
        }
}
public class TestSaleTicket{
    public static void main(String[] args) {
        SaleTicket saleTicket = new SaleTicket();
        Thread thread1 = new Thread(saleTicket,"window1");
        Thread thread2 = new Thread(saleTicket,"window2");
        thread1.start();
        thread2.start();
    }
}
```
<b>同步方法</b>：对一个方法加同步锁，使得只有获得锁的线程才能执行该方法，需要注意的是对于非静态方法来说，锁对象为调用该同步方法的对象。对于静态方法来说，锁对象为同步方法所在类的class。对于上面的Demo,利用同步方法保证线程安全的Java代码示例如下：  
```java
class SaleTicket implements Runnable {
    Object object = new Object();
    //共享数据
    private  int ticketNum = 100;
    public void run() {
            while (true) {
                   sale();
            }
        }
    public synchronized void sale(){
        if (ticketNum > 0) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程" + Thread.currentThread().getName() + "销售电影票" + ticketNum--);
        }
    }
}
public class TestSaleTicket{
    public static void main(String[] args) {
        SaleTicket saleTicket = new SaleTicket();
        Thread thread1 = new Thread(saleTicket,"window1");
        Thread thread2 = new Thread(saleTicket,"window2");
        thread1.start();
        thread2.start();
    }
}
```
##### lock锁
<b>同步锁</b>:同步锁也是保证线程安全的手段之一。java.util.concurrent.locks.Lock提供了比synchronized代码块和synchronized方法更广泛的锁操作。同时它提供了很多种类型的锁的实现类。Lock同步锁中常用的方法为：
> public void lock() //加同步锁  
 public void unlock() //释放同步锁  
 
同时lock()方法加锁后，必须手动的调用unlock()锁，否则会照成同步锁一直不被释放，其它线程始终获取不了执行权，形成死锁。所以lock()方法的内容最好使用try--catch包裹起来，在finally中调用unlock()方法。这样即使出现异常，也会执行unlock()方法，不会造成死锁。对于上面的Demo,利用同步方法保证线程安全的Java代码示例如下：  
```java
class SaleTicket implements Runnable {
    /**
     * 参数true:公平锁，多个线程都公平的拥有执行权
     * 参数false:独占锁，执行完成才释放
     * */
    private Lock lock = new ReentrantLock(true);
    //共享数据
    private  int ticketNum = 100;
    public void run() {
            while (true) {
                lock.lock();
                try {
                    if (ticketNum > 0) {
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        System.out.println("线程" + Thread.currentThread().getName() + "销售电影票" + ticketNum--);
                    }
                }finally {
                    lock.unlock();
                }
            }
        }
}
public class TestSaleTicket{
    public static void main(String[] args) {
        SaleTicket saleTicket = new SaleTicket();
        Thread thread1 = new Thread(saleTicket,"window1");
        Thread thread2 = new Thread(saleTicket,"window2");
        thread1.start();
        thread2.start();

    }
}
```
<b>synchronized与Lock的区别</b>：
* synchronized是java中内置的关键字；而Lock则是java中的一个接口，它提供了多种实现了Lock接口的锁
* synchronized无法判断到线程是否获取到锁；而Lock则可以进行判断
* synchronized会自动释放锁（线程执行完成或者执行过程中发生了异常）；而Lock必须手动释放锁，否则容易照成死锁
* 当线程A执行synchronized修饰的代码时候，B会进行等待，此时如果线程A进入阻塞状态，B会一直等待下去;而Lock可能不会等待下去，当B尝试获取不到锁时，可能会直接结束了