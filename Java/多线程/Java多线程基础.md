## Thread的API

### isAlive()

判断当前线程是否处于活动状态。

### sleep()

在指定的毫秒数内让当前正在执行的线程休眠，即暂停执行。这个正在执行的线程是指`this.currentThread`返回的线程。

### 停止线程

`Thread.interrupt()`方法去终止正在运行的线程。但调用`interrupt()`方法仅仅是在当前线程中打了一个停止的标记，没有立即停止线程。

```java
@Override
public void run(){
  if(this.interrupted()){
    throw new InterruptedException();
  }
  ...
}

@Override
public void run(){
  if(this.interrupted()){
    return;
  }
  ...
}
```



#### 判断是否线程是否是停止状态

1. Thread.interrupted()：测试当前线程(即调用此方法的线程)是否已经是中断状态，执行后具有将状态标志置为false的功能。
2. this.isInterrupted()：测试线程Thread对象是否已经是中断状态，但不清楚状态标志。

#### Stop()暴力停止

方法stop()已经被作废，因为如果强制让线程停止则可能使一些清理性的工作得不到完成，另外一种情况就是对锁定的对象进行了解锁，导致数据得不到同步的处理，出现数据不一致的问题。



### 暂停线程

#### suspend与resume方法

在Java线程中，可以使用suspend方法暂停线程，使用resume方法恢复线程的执行。

缺点：

1. 对象锁不释放，造成公共的同步对象的独占，使得其他线程无法访问公共同步对象。
2. 因为线程的暂停而导致数据不同步的情况。

#### yield方法

让线程放弃当前的CPU资源，将它让给其他的任务去占用CPU执行时间。也有可能刚刚放弃了，马上又获得CPU时间片。

### 线程的优先级

Java线程的优先级分为1~10这10个等级，超过这个范围会抛异常。可以通过`setPriority()`方法来设置线程优先级。线程的优先级具有继承性。

### 守护线程

Java线程中有两种线程，一种是用户线程，另一种是守护线程。

守护线程有保姆的特性，当进程中不存在非守护线程了，则守护线程自动销毁。典型的守护线程就是垃圾回收线程。

调用`thread.setDaemon(true)`来设置一个线程为守护线程。



## 并发访问

只有共享和可变的变量才有必要进行线程安全的处理。

### synchronized同步方法

关键字synchronized取得的锁都是对象锁(当前类的实例对象)，而不是一段代码或方法当作锁，哪个线程先执行带synchronized关键字的方法，哪个线程就持有该方法所属对象的锁Lock。

1. A线程先持有Object对象的Lock锁，B线程可以以**异步**的方式调用object对象中的**非synchronized**类型方法。
2. A线程先持有object对象的Lock锁，B线程如果在这时调用object对象中的synchronized类型的方法则需要等待。
3. 关键字synchronized拥有**锁重入**的功能。也就是当一个线程得到一个对象锁后，再次请求此对象锁时是可以再次得到该对象的锁。
4. 出现异常后，锁会自动释放。
5. 同步方法不具有继承性。


### synchronized同步代码块

```java
synchronized (obj){
  // 同步代码块
}
```

当线程能获得obj对象锁才能进入同步代码块，obj对象作为对象监听器来实现同步的功能。

锁非this对象具有一定的优点：如果在一个类中有很多个synchronized方法，这时虽然能实现同步，但会堵塞；但如果使用同步代码块锁非this对象，则synchronized代码块中的程序与同步方法是异步的，不与其他锁this同步方法争抢this锁，则大大提高运行效率。

由于String常量池的存在，如果两个String对象的值相等，那么它们是同一个对象。因此当它们作为对象监视器使需要注意它们是否是同一个锁。

### 静态同步synchronized方法与synchronized(class)代码块

synchronized关键字加到static静态方法上是给Class类上锁，而synchronized关键字加到非static静态方法上是给对象上锁。Class锁可以对类的所有对象实例起作用。

### 锁对象的改变

线程竞争的锁是线程一开始遇到同步方法或同步代码块的对象或基本数据类型值，只要锁对象不改变，尽管属性改变了，始终也是同一个锁。

**参考：**[How the Java virtual machine performs thread synchronization](https://www.javaworld.com/article/2076971/java-concurrency/how-the-java-virtual-machine-performs-thread-synchronization.html)

### volatile关键字

volatile的主要作用是使变量在多个线程间可见，适用于多线程并发读，同步写的场景，因为volatile并不能保证操作的原子性。

若JVM设置为-server模式时，线程私有堆栈中的值和公共堆栈中的值可能会不同步，而使用volatile关键字可以消除这种不同步的现象。

- 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存。
- 当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起进行**指令重排序**。

volatile与synchronized之间的区别：

1. volatile时线程同步的轻量级实现。volatile只能修饰于变量，而synchronized可以修饰方法以及代码块。
2. 多线程访问volatile不会发生阻塞，synchronized会出现阻塞。
3. volatile能保证数据的可见性，但不能保证原子性；synchronized可以保证原子性，会将私有内存和公有内存中的数据做同步。
4. volatile解决的是变量在多个线程之间的可见性，synchronized解决的是多线程资源访问的同步性。


## Lock

### ReentrantLock

```java
public class Service {
    private Lock lock = new ReentrantLock();

    public void visit() {
        try {
            lock.lock();
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

当`lock.lock()`被调用后线程就持有了锁，其他线程只有等待锁被释放时才能再次争抢锁，这一点与synchronized关键字一样。

使用ReentrantLock结合Condition类可以实现线程的选择性通知和多路通知，但必须先调用`lock.lock()`，让线程先获得锁。Object类的wait方法相当于Condition的await方法，Object类的notify(notifyAll)方法相当于Condition的signal(signalAll)方法。这实际上是将锁和信号分离开了，这样一个锁里面可以有多个条件队列。

#### 公平锁与非公平锁

公平锁表示线程获取锁的次序是按照线程加锁的顺序来分配的，非公平锁是利用随机抢占机制来获取锁。

```java
lock = new ReentrantLock(isFair);
```

#### 关于Lock对象的一些方法

- getHoldCount的作用是查询当前线程调用`lock()`的次数。
- getQueueLength作用是返回正等待获得此锁定的线程估计数。
- getWaitQueueLength(Condition condition) 的作用是返回等待与此锁定相关的给定条Condition的线程估计数。

### ReentrantReadWriteLock

读写锁表示也有两个锁，一个是读操作相关的锁，也称为共享锁；另一个是写操作相关的锁，也称为排他锁。也就是多个读锁之间不互斥，读锁与写锁互斥，写锁与写锁互斥。

```java
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    public void read() {
        try {
            lock.readLock().lock();
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.readLock().unlock();
        }
    }
    public void write() {
        try {
            lock.writeLock().lock();
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock();
        }
    }
```


### Lock与synchronized的区别

1. Lock的加锁和解锁都是由java代码配合native方法（调用操作系统的相关方法）实现的，而synchronize的加锁和解锁的过程是由JVM管理的。
2. 当一个线程使用synchronize获取锁时，若锁被其他线程占用着，那么当前只能被阻塞，直到成功获取锁。而Lock则提供超时锁和可中断等更加灵活的方式，在未能获取锁的条件下提供一种退出的机制（等待可中断）。
3. 一个锁内部可以有多个Condition实例，即有多路条件队列，而synchronize只有一路条件队列；同样Condition也提供灵活的阻塞方式，在未获得通知之前可以通过中断线程以及设置等待时限等方式退出条件队列。
4. synchronize对线程的同步仅提供独占模式，而Lock即可以提供独占模式，也可以提供共享模式。
5. synchronized的可读性比Lock高，因为Lock的加锁解锁需要借助try-finally语句。
6. Lock可以实现公平锁，synchronized只能实现非公平锁。


## 线程间通信

### 等待与通知机制

`wait()`的作用是使当前执行代码的线程进行等待，并置入预执行队列。在调用`wait()`之前，线程必须获得该对象的对象级别锁，只能在同步方法或同步代码块内调用`wait()`方法。在执行`wait()`后，当前线程释放锁，在`wait()`返回前，其他线程竞争锁。

`notify()`也要在同步方法或同步代码块内调用，该方法通知那些可能等待该对象的对象锁的其他线程，如果有多个线程在等待，则由线程规划器随机挑选其中一个呈wait状态的线程，对其发出通知notify，使它获得对象锁。但在执行notify方法后，当前线程不会马上释放该对象锁，呈wait状态的线程也不能马上获得该对象锁，要等线程退出synchronized代码块后，当前线程才会释放锁。

`notifyAll()`方法可以使所有正在等待队列中等待同一共享资源的全部线程从等待状态退出，进入可运行状态。

每个锁对象都有两个队列，一个是就绪队列，一个是阻塞队列。就绪队列存储了将要获得锁的线程，阻塞队列存储了被阻塞的线程，一个线程被唤醒后，才会进入就绪队列，等待CPU的调度。反之，一个线程调用`wait()`后，会进入阻塞队列。

当线程处于wait状态时，调用线程对象的`interrupt()`方法会出现InterruptedException异常。

#### 生产者与消费者

```
    public synchronized void p() throws InterruptedException {
        while (val <= 0)
            wait();
        --val;
    }

    public synchronized void v() throws InterruptedException {
        while (val >= maxVal)
            wait();
        ++val;
        notifyAll();
    }
}
```

### 管道

1. 字节流：PipedInputStream和PipedOutputStream
2. 字符流：PipedReader和PipedWriter

### 实战：交叉备份

使用volatile声明一个标记，使用wait()/notifyAll()来协调各线程交叉访问数据库。

```java
volatile private boolean isToBackupA = false;
synchronized public void backupA() {
  try {
    while (isToBackupA == true)
    	wait();
    // 备份
    isToBackupA = false;
    notifyAll();
  } catch(InterruptedException e){
    Thread.currentThread.interrupt();
  }
}
```

### join()

如果父线程想等待子线程执行完成之后再继续往下运行，可以调用`join()`方法来阻塞父线程。join与synchronized都能实现线程同步，它们的区别是：join内部使用`wait()`将线程放入等待队列，而synchronized关键字使用的时对象监视器。

`join()`和 `wait()`执行后会释放锁， `sleep ` 执行后不会释放锁。

### ThreadLocal

ThreadLocal主要解决的是每个线程绑定自己的值，可以将ThreadLocal类比成全局存放数据的盒子，盒子中可以存放每个线程的私有数据。

覆盖initialValue()方法具有初始值。

```java
    public static class ThreadLocalExt extends ThreadLocal {
        @Override
        protected Object initialValue() {
            return "默认值";
        }
    }
```

使用InheritableThreadLocal可以在子线程中取得父线程继承下来的值。

解决SimpleDateFormat类的线程同步问题可以将SimpleDateFormat通过ThreadLocal绑定到各个线程。

```java
private static ThreadLocal<SimpleDateFormat> t = new ThreadLocal<SimpleDateFormat>();
public static SimpleDateFormat getSimpleDateFormat(String dataPat){
  SimpleDateFormat sdf = null;
  sdf = t.get();
  if (sdf == null){
    sdf = new SimpleDateFormat(dataPat);
    t.set(sdf);
  }
  return sdf;
}
```



## 单例模式

### 使用enum实现单例

```java
public enum Singleton {
  INSTANCE;
  private Connection conn;
  private Singleton(){
    conn = ...;
  }
  public static Connection getConnection(){
    return Singleton.INSTANCE.conn;
  }
}
```

