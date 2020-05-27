##  Java 并发

### Thread 启动的两种方式和区别

采用继承Thread类方式：
（1）优点：编写简单，如果需要访问当前线程，无需使用Thread.currentThread()方法，直接使用this，即可获得当前线程。
（2）缺点：因为线程类已经继承了Thread类，所以不能再继承其他的父类。
采用实现Runnable接口方式：
（1）优点：
线程类只是实现了Runable接口，还可以继承其他的类。
可以多个线程共享同一个目标对象，所以非常适合多个相同线程来处理同一份资源的情况。
（2）缺点：编程稍微复杂，如果需要访问当前线程，必须使用Thread.currentThread()方法。

Runnable可以实现资源共享但Thread不能原因：

因为一个线程只能启动一次，通过Thread实现线程时，线程和线程所要执行的任务是捆绑在一起的。也就使得一个任务只能启动一个线程，不同的线程执行的任务是不相同的，所以没有必要，也不能让两个线程共享彼此任务中的资源。

一个任务可以启动多个线程，通过Runnable方式实现的线程，实际是开辟一个线程，将任务传递进去，由此线程执行。**可以实例化多个 Thread对象，将同一任务传递进去，也就是一个任务可以启动多个线程来执行它。这些线程执行的是同一个任务，所以他们的资源是共享。**

#### sleep 和 wait 的区别

1、sleep是线程中的方法，但是wait是Object中的方法。

2、sleep方法不会释放lock，但是wait会释放，而且会加入到等待队列中。

3、sleep方法不依赖于同步器synchronized，但是wait需要依赖synchronized关键字。

4、sleep不需要被唤醒（休眠之后推出阻塞），但是wait需要（不指定时间需要被别人中断）。

#### start 和 run 的区别

new 一个 Thread，线程进入了新建状态;调用 start() 方法，会启动一个线程并使线程进入了就绪状态，当分配到时间片后就可以开始运行了。 start() 会执行线程的相应准备工作，然后自动执行 run() 方法的内容，这是真正的多线程工作。 而直接执行 run() 方法，会把 run 方法当成一个 main 线程下的普通方法去执行，并不会在某个线程中执行它，所以这并不是多线程工作。

### synchronized 关键字

* **修饰实例方法**

* **修饰代码块**

* **修饰静态方法**：相当于给当前类加锁，会作用于类的所有实例。线程 A 调用实例对象的非静态 synchronized 方法，线程 B 调用这个实例对象所属类的静态 synchronized 方法。不会发生冲突。**因为访问静态synchronized 方法占用的是当前类的锁，访问 非静态 synchronized 方法是占用当前实例对象的锁**

* **修饰类**

  

#### 单例模式

```java
public class Singleton{
    
    private volatile static Singleton instance;
    
    private Singleton(){
        
    }
    
    public static Singleton getInstance(){
        // 先判断对象有没有实例化，没有实例化才进入加锁代码
        if (instance == null){
            // 对类加锁
            synchronized (Singleton.class){
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

 

##### 为何使用 volatile 关键字修饰变量？

`instance = new Singleton();` 这段代码分三步执行：

1. 为 instance 分配内存空间
2. 初始化 instance
3. 将 instance 指向分配的内存地址

JVM 由于具有指令重排的特征，执行顺序可能变成 1  -> 3 -> 2 。指令重排在单线程不会出现问题。但是在多线程环境下会导致一个线程获得还未初始化的实例。 如 T1 执行了 1和 3 ，T2 调用 `getInstance()` 执行，先判断 instance 不为空，返回 instance，但是此时 instance 还未被初始化

因此 **volatile 可以防止 JVM 指令重排**



#### 底层实现原理

* ##### 同步语句块

  Synchronized 经过 javac 编译后，会形成  `monitorenter` 和 `monitorexit` 指令，分别指向同步代码块开始和结束的位置

  * 执行 `monitorenter`  指令，线程试图获取锁，也就是获取 monitor （其存在于每个Java对象的对象头中），锁的计数器 +1。
  * 在执行 `monitorexit` 指令，线程释放锁，锁的计数器 - 1
  * 获取锁失败，那么当前线程就要阻塞等待，直到所被另外一个线程释放为止 （锁的计数器为0）

  

* ##### 同步方法

  * 使用 `ACC_SYNCHRONIZED` 表示来判断是否是一个同步方法。
  * 原理是通过方法调用指令检查该方法在常量池中是否包含 ACC_SYNCHRONIZED 标记符，如果有，JVM 要求线程在调用之前请求锁
  
  

#### Synchronized  和 ReentrantLock 的区别

* 都是可重入锁

  > 可重入锁：自己可以再次获得自己的内部锁。一个线程获得了某个对象的锁，此时这对象锁还没有被释放。当他想要再次获得这个锁是可以的，因为不可重入的话，会造成死锁。同一个线程每次获得锁，计数器增1，要等到计数器为0的时候才能释放锁
  >
  > **重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数**

* Synchronized 依赖于 JVM ，ReentrantLock 依赖于 API

* ReentrantLock 增加了一些高级功能

  * **等待可中断**：等待中的线程也以放弃等待 `lock.lockInterruptibly()`

  * **可以实现公平锁**：先等待的线程可以先获得锁，可以在构造方法中指定是否为公平锁 （等待时长）

  * **可实现选择性通知**：ReentanctLock 可以绑定多个 Condition 实例。线程对象可以注册在指定的Condition实例中，从而有选择性的通知。

    Synchronized 与 wait()/notify(), notifyAll() 方法实现等待/ 通知机制。 notifyAll()  会通知所有处于等待线程，而 Condition 的 signalAll() 方法只会唤醒注册在该Condition 实例中的所有等待线程

> Synchronized 底层是监听器，ReentrantLoack 底层是 AQS，AQS主要是管理同步状态，阻塞队列和支持此线程中断



####  Synchronized  和 volatile 的区别

* **volatile 是线程同步的轻量级实现（只修饰变量），性能更好** 

* **多线程访问 volatile 不会发生阻塞**

* **volatile 关键字可以实现数据的可见性。不能保证原子性。synchronized 两者都能实现** 

  可见性：只要某个线程修改了某个变量，其他线程都能知道这个变量被修改过了。因为修改变量后会先同步到主内存，再对主内存的变量刷新得到的。通过添加 volatile 关键字来提示JVM这个变量是不稳定的，要从主内存中读取。 （还有一个作用是通过添加内存屏障的方式**防止指令重排**。见前面单例模式）

* volatile 主要用于变量在多线程之间的可见性，synchronized关键字解决的是多线程访问资源的同步性



#### CAS与synchronized的使用情景

> **简单的来说CAS适用于写比较少的情况下（多读场景，冲突一般较少），synchronized适用于写比较多的情况下（多写场景，冲突一般较多）**

1. 对于资源竞争较少（线程冲突较轻）的情况，使用synchronized同步锁进行线程阻塞和唤醒切换以及用户态内核态间的切换操作额外浪费消耗cpu资源；而CAS基于硬件实现，不需要进入内核，不需要切换线程，操作自旋几率较少，因此可以获得更高的性能。
2. 对于资源竞争严重（线程冲突严重）的情况，CAS自旋的概率会比较大，从而浪费更多的CPU资源，效率低于synchronized。

补充： Java并发编程这个领域中synchronized关键字一直都是元老级的角色，很久之前很多人都会称它为 **“重量级锁”** 。但是，在JavaSE 1.6之后进行了主要包括为了减少获得锁和释放锁带来的性能消耗而引入的 **偏向锁** 和 **轻量级锁** 以及其它**各种优化**之后变得在某些情况下并不是那么重了。synchronized的底层实现主要依靠 **Lock-Free** 的队列，基本思路是 **自旋后阻塞**，**竞争切换后继续竞争锁**，**稍微牺牲了公平性，但获得了高吞吐量**。在线程冲突较少的情况下，可以获得和CAS类似的性能；而线程冲突严重的情况下，性能远高于CAS。

### ThreadLocal

使得线程有自己的专属变量，ThreadLocal 类让每个线程绑定自己的值。如果创建 ThreadLocal 变量，每个访问这个变量的线程都会有这个变量的本地副本，可以使用get（） 和 set（） 方法来获取默认值或将其值改成当前线程所存在副本的值。

#### 实现原理

从 `Thread`类源代码入手。

```java
public class Thread implements Runnable {
     ......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;

    //与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
     ......
}
```

从上面`Thread`类 源代码可以看出`Thread` 类中有一个 `threadLocals` 和 一个 `inheritableThreadLocals` 变量，它们都是 `ThreadLocalMap` 类型的变量,我们可以把 `ThreadLocalMap` 理解为`ThreadLocal` 类实现的定制化的 `HashMap`。默认情况下这两个变量都是null，只有当前线程调用 `ThreadLocal` 类的 `set`或`get`方法时才创建它们，实际上调用这两个方法的时候，我们调用的是`ThreadLocalMap`类对应的 `get()`、`set()`方法。

`ThreadLocal`类的`set()`方法

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

通过上面这些内容，得出结论：

* **最终的变量是放在了当前线程的 ThreadLocalMap 中，并不是存在 ThreadLocal 上，ThreadLocal 可以理解为只是ThreadLocalMap的封装，传递了变量值。** `ThrealLocal` 类中可以通过`Thread.currentThread()`获取到当前线程对象后，直接通过`getMap(Thread t)`可以访问到该线程的`ThreadLocalMap`对象。

* **每个Thread中都具备一个ThreadLocalMap，而ThreadLocalMap可以存储多个以ThreadLocal为key的键值对。通过`get()` 方法 和 `set()` 方法来获取和设置值** 

  ThreadLocalMap 是 ThreadLocal 里面的静态内部类

  ```java
  // key value 都被看成是 Entry 类
  static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;
  
      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }
  
  // 去存放这个类一个table 就和 hashMap 这个类table 存放 Node 类似的
  private Entry[] table;
  ```

  比如我们在同一个线程中声明了两个 `ThreadLocal` 对象的话，会使用这个 `Thread`内部的`ThreadLocalMap` 存放数据的，`ThreadLocalMap`的 key 就是 `ThreadLocal`对象，value 就是 `ThreadLocal` 对象调用`set`方法设置的值。`ThreadLocal` 是 map结构是为了让每个线程可以关联多个 `ThreadLocal`变量。这也就解释了 ThreadLocal 声明的变量为什么在每一个线程都有自己的专属本地变量。

  

  

#### ThreadLocal 中的内存泄露问题

`ThreadLocalMap`  中的 key 是弱引用（可有可无），value 是强引用。在 ThreadLocal 没有被外部强引用的状态下，垃圾回收时会先回收 key，value 不会清理。不做任何措施的话，value 无法被 GC 回收，这个时候可能产生内存泄露



### 线程池

> 池化技术如 线程池、数据库连接池、Http 连接池等等。主要是为了减少每次获取资源的消耗，提高对资源的利用率

线程池提供了一种限制和管理资源（包括执行一个任务）。每个线程池维护一些基本统计信息，例如已经完成的任务数量。

* 降低资源消耗。重复利用已经创建的线程
* 提高响应速度。任务到达时，任务可以不需要等到线程创建就能执行
* 提高线程可管理性。使用线程池对线程进行统一分配 调优和监控



#### Runnable 和 Callable 的区别

Runnable 接口**不会返回结果或者抛出检查异常**， Callable 接口可以



#### excute() 方法 和 submit() 方法的区别

* `excute()` 提交不需要返回值的任务，无法判断任务是否被线程池成功执行与否
* `submit()` 方法用于提交需要返回值的任务，会返回一个 `Future` 类型的对象。通过这个对象哦安短任务是否成功。通过get方法获取返回的值。get 方法会阻塞当前线程直到任务完成



#### 如何创建线程池

允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

> Executors 返回线程池对象的弊端如下：
>
> - **FixedThreadPool 和 SingleThreadExecutor** ： 允许请求的队列长度为 Integer.MAX_VALUE ，可能堆积大量的请求，从而导致OOM。
> - **CachedThreadPool 和 ScheduledThreadPool** ： 允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致OOM。



**方式一：通过构造方法实现** ![ThreadPoolExecutor构造方法](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/ThreadPoolExecutor%E6%9E%84%E9%80%A0%E6%96%B9%E6%B3%95.png) 



**方式二：通过Executor 框架的工具类Executors来实现** 我们可以创建三种类型的ThreadPoolExecutor：

- **FixedThreadPool** ： 该方法返回一个固定线程数量的线程池。该线程池中的线程数量始终不变。当有一个新的任务提交时，线程池中若有空闲线程，则立即执行。若没有，则新的任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务。
- **SingleThreadExecutor：** 方法返回一个只有一个线程的线程池。若多余一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出的顺序执行队列中的任务。
- **CachedThreadPool：** 该方法返回一个可根据实际情况调整线程数量的线程池。线程池的线程数量不确定，但若有空闲线程可以复用，则会优先使用可复用的线程。**若所有线程均在工作，又有新的任务提交，则会创建新的线程处理任务**。所有线程在当前任务执行完毕后，将返回线程池进行复用。

对应Executors工具类中的方法如图所示： ![Executor框架的工具类](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/Executor%E6%A1%86%E6%9E%B6%E7%9A%84%E5%B7%A5%E5%85%B7%E7%B1%BB.png)



### ThreadPoolExecutor 类

```java
/**
     * 用给定的初始参数创建一个新的ThreadPoolExecutor。
     */
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
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

#### 常见构造参数

* `int corePoolSize`  ：核心线程参数，定义了最小可以同时运行的线程数量
* `int maximumPoolSize` : 当前对列存放的任务数量达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数
* `BlockingQueue<Runnable> workQueue`：当新任务来的时候会先判断当前线程数是否到达核心线程数，如果到达到则会被存放在队列中
* `long keepAliveTime `：当线程池中线程的数量大于核心线程数的时候，如果没有新的任务提交，核心线程外的线程不会立即被销毁，而是等到时间超过了 `keepAliveTime` 才会被销毁
* `TimeUnit unit` ：keepAliveTime 时间单位
* `ThreadFactory threadFactory` 创建新线程时会用到
* `RejectedExecutionHandler handler`：饱和策略



#### 饱和策略

若当前同时运行的线程数量达到最大线程数量并且队列已经满了时，采取的一些策略

- **ThreadPoolExecutor.AbortPolicy**：抛出 `RejectedExecutionException`来拒绝新任务的处理。
- **ThreadPoolExecutor.CallerRunsPolicy**：调用执行自己的线程运行任务。您不会丢弃任何任务请求。但是这种策略会降低对于新任务提交速度，影响程序的整体性能。另外，这个策略会增加队列容量，当最大池被填满时，这个策略为我们提供伸缩队列
- **ThreadPoolExecutor.DiscardPolicy：** 不处理新任务，直接丢弃掉。
- **ThreadPoolExecutor.DiscardOldestPolicy：** 此策略将丢弃最早的未处理的任务请求。



#### 线程池提交任务运行过程

![https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-7/%E5%9B%BE%E8%A7%A3%E7%BA%BF%E7%A8%8B%E6%B1%A0%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.png](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-7/图解线程池实现原理.png)



### Atomic 原子类型

原子类说简单点就是具有原子/原子操作特征的类，一个操作是不可中断的。即使是在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程干扰。存放在 `java.util.concurrent` 包中

**基本类型**

使用原子的方式更新基本类型

- AtomicInteger：整形原子类
- AtomicLong：长整型原子类
- AtomicBoolean：布尔型原子类

**数组类型**

使用原子的方式更新数组里的某个元素

- AtomicIntegerArray：整形数组原子类
- AtomicLongArray：长整形数组原子类
- AtomicReferenceArray：引用类型数组原子类

**引用类型**

- AtomicReference：引用类型原子类
- AtomicStampedReference：原子更新引用类型里的字段原子类
- AtomicMarkableReference ：原子更新带有标记位的引用类型

**对象的属性修改类型**

- AtomicIntegerFieldUpdater：原子更新整形字段的更新器
- AtomicLongFieldUpdater：原子更新长整形字段的更新器
- AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

#### AtomicInteger 原理

**AtomicInteger 类常用方法**

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后
```

```java
 // setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
static {
	try {
		valueOffset = unsafe.objectFieldOffset
				(AtomicInteger.class.getDeclaredField("value"));
	} catch (Exception ex) { throw new Error(ex); }
}
private volatile int value;
```

AtomicInteger 类主要利用 CAS (compare and swap) + volatile 和 native 方法来保证原子操作，从而避免 synchronized 的高开销，执行效率大为提升。

CAS的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。UnSafe 类的 objectFieldOffset() 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址，返回值是 valueOffset。另外 value 是一个volatile变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值。



### AQS

AQS的全称为（AbstractQueuedSynchronizer），这个类在java.util.concurrent.locks包下面。

#### Lock的实现

实现Lock接口的类有很多，以下为几个常见的锁实现

- ReentrantLock：表示重入锁，它是唯一一个实现了Lock接口的类。重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数
- ReentrantReadWriteLock：重入读写锁，它实现了ReadWriteLock接口，在这个类中维护了两个锁，一个是ReadLock，一个是WriteLock，他们都分别实现了Lock接口。读写锁是一种适合**读多写少**的场景下解决线程安全问题的工具，基本原则是：**读和读不互斥、读和写互斥、写和写互斥**。也就是说涉及到影响数据变化的操作都会存在互斥。
- StampedLock： stampedLock是JDK8引入的新的锁机制，可以简单认为是读写锁的一个改进版本，读写锁虽然通过分离读和写的功能使得读和读之间可以完全并发，但是读和写是有冲突的，**如果大量的读线程存在，可能会引起写线程的饥饿。stampedLock是一种乐观的读策略，使得乐观锁完全不会阻塞写线程**



#### AQS对资源的的两种状态

从使用层面来说，AQS的功能分为两种：独占和共享

- 独占锁，每次只能有一个线程持有锁，ReentrantLock就是以独占方式实现的互斥锁
  - 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
  - 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的
- 共享锁，允许多个线程同时获取锁，并发访问共享资源，比如ReentrantReadWriteLock 实现共享读



#### 实现原理

* 当资源被占用时，AQS提供了一个线程阻塞等待以及被唤醒的机制
* 由CLH队列实现的，他是一个虚拟的双向队列，不存在队列实例，仅存在结点与结点之间的关联关系
* AQS 将每个请求共享资源的线程封装成一个CLH队列的一个结点（Node）来实现锁的分配

![https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/AQS%E5%8E%9F%E7%90%86%E5%9B%BE.png](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-6/AQS原理图.png)

#### state资源技术

AQS使用一个int成员变量 state 来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。使用volatile修饰保证线程可见性

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

状态信息通过protected类型的getState，setState，compareAndSetState进行操作

```java
//返回同步状态的当前值
protected final int getState() {  
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
        state = newState;
}
//原子地（CAS操作）将同步状态值设置为给定值update如果当前同步状态的值等于expect（期望值）
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

- 以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。
- 再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS(Compare and Swap)减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

不同的实现状态 state 的含义是不同的

#### AQS 组件

- **Semaphore(信号量)-允许多个线程同时访问：**Semaphore(信号量)可以指定多个线程同时访问某个资源。
- **CountDownLatch （倒计时器）：** CountDownLatch是一个同步工具类，用来协调多个线程之间的同步。这个工具通常用来控制线程等待，它可以让某一个线程等待直到倒计时结束，再开始执行。使一个线程等待其他线程各自执行完毕后再执行。是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。
- **CyclicBarrier(循环栅栏)：** CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。**它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活**。CyclicBarrier默认的构造方法是 CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，每个线程调用await()方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。



#### AQS 使用了模板设计方法

同步器的设计是基于模板方法模式的，如果需要自定义同步器一般的方式是这样（模板方法模式很经典的一个应用）：

1. 使用者继承AbstractQueuedSynchronizer并重写指定的方法。（这些重写方法很简单，无非是对于共享资源state的获取和释放）
2. 将AQS组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

这和我们以往通过实现接口的方式有很大区别，这是模板方法模式很经典的一个运用。

**AQS使用了模板方法模式，自定义同步器时需要重写下面几个AQS提供的模板方法：**

```java
isHeldExclusively()//该线程是否正在独占资源。只有用到condition才需要去实现它。
tryAcquire(int)//独占方式。尝试获取资源，成功则返回true，失败则返回false。
tryRelease(int)//独占方式。尝试释放资源，成功则返回true，失败则返回false。
tryAcquireShared(int)//共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
tryReleaseShared(int)//共享方式。尝试释放资源，成功则返回true，失败则返回false。
```

默认情况下，每个方法都抛出 `UnsupportedOperationException`。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS类中的其他方法都是final ，所以无法被其他类使用，只有这几个方法可以被其他类使用。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。



## Java多线程常见容器

### CurrentHashMap

### CopyOnWriteArrayList

* `ReentrantReadWriteLock` 读写锁的思想非常类似，也就是**读读共享、写写互斥、读写互斥、写读互斥**，但此思想更近一步，提高了性能
* JDK 中提供了 `CopyOnWriteArrayList` 相比于读写锁思想更近一步
* 读取是完全不用加锁的
* 写入也不会阻塞读取操作。只有写入和写入之间需要进行同步等待

#### 基本原理

* 所有可变操作都是通过创建底层数据的新副本操作的，修改的是副本内容，写完之后再替换原来的数据，这样写的时候不影响读操作

  > 在计算机，如果你想要对一块内存进行修改时，我们不在原有内存块中进行写操作，而是将内存拷贝一份，在新的内存中进行写操作，写完之后呢，就将指向原来内存指针指向新的内存，原来的内存就可以被回收掉了

* 读操作无须加锁，因为不会发生修改，只会发生替换
* 写操作需要加锁，保证同步，避免多个线程写出多个副本来

### ConcurrentLinkedQueue

* Java线程安全的队列可以分为**阻塞队列**（BlockingQueue）和 **非阻塞队列**（ConcurrentLinkedQueue）

* **阻塞队列可以通过加锁来实现，非阻塞队列可以通过 CAS 操作实现。**



### BlockingQueue

阻塞队列（BlockingQueue）被广泛使用在“生产者-消费者”问题中

* BlockingQueue 提供了可阻塞的插入和移除的方法。当队列容器已满，生产者线程会被阻塞，直到队列未满；当队列容器为空时，消费者线程会被阻塞，直至队列非空时为止
* BlockingQueue 是一个接口，继承自 Queue 。以下是其三种主要的实现类

#### ArrayBlockingQueue

* **ArrayBlockingQueue** 是 BlockingQueue 接口的有界队列实现类，底层采用**数组**来实现。ArrayBlockingQueue 一旦创建，容量不能改变。
* 其并发控制采用可重入锁来控制，不管是插入操作还是读取操作，都需要获取到锁才能进行操作。当队列容量满时，尝试将元素放入队列将导致操作阻塞;尝试从一个空队列中取一个元素也会同样阻塞
* ArrayBlockingQueue 默认情况下不能保证线程访问队列的公平性，可以在构造函数中指定公平性

#### LinkedBlockingQueue

* **LinkedBlockingQueue** 底层基于**单向链表**实现的阻塞队列
* 可以当做无界队列也可以当做有界队列来使用，同样满足 FIFO 的特性，
* 与 ArrayBlockingQueue 相比起来具有更高的吞吐量，为了防止 LinkedBlockingQueue 容量迅速增，损耗大量内存。通常在创建 LinkedBlockingQueue 对象时，会指定其大小，如果未指定，容量等于 Integer.MAX_VALUE

#### PriorityBlockingQueue

* **PriorityBlockingQueue** 是一个支持优先级的无界阻塞队列。默认情况下元素采用自然顺序进行排序，也可以通过自定义类实现 `compareTo()` 方法来指定元素排序规则，或者初始化时通过构造器参数 `Comparator` 来指定排序规则。
* PriorityBlockingQueue 并发控制采用的是 **ReentrantLock**，队列为无界队列（ArrayBlockingQueue 是有界队列，LinkedBlockingQueue 也可以通过在构造函数中传入 capacity 指定队列最大的容量，但是 PriorityBlockingQueue 只能指定初始的队列大小，后面插入元素的时候，**如果空间不够的话会自动扩容**）
* 不可以插入 null 值，同时，插入队列的对象必须是可比较大小的（comparable），否则报 ClassCastException 异常
* 它的插入操作 put 方法不会 block，因为它是无界队列（队列理论上来说永远不会满），take 方法在队列为空的时候会阻塞

### **ConcurrentSkipListMap**

#### 什么是跳表？跳表与平衡树

* 单链表查询数据即使链表是有序的，查询数据需要从头遍历链表

* 跳表是一种可以用来快速查找的数据结构，有点类似于平衡树。它们都可以对元素进行快速的查找。但一个重要的区别是：对平衡树的插入和删除往往很可能导致平衡树进行一次全局的调整。而对跳表的插入和删除只需要对整个数据结构的局部进行操作即可。
* 在高并发的情况下，你会需要一个全局锁来保证整个平衡树的线程安全。而对于跳表，你只需要部分锁即可。这样，在高并发环境下，你就可以拥有更好的性能。而就查询的性能而言，跳表的时间复杂度也是 **O(logn)** 所以在并发数据结构中，JDK 使用跳表来实现一个 Map。

跳表的本质是同时维护了多个链表，并且链表是分层的

![2级索引跳表](E:\研究生学习\Typora图片\93666217.jpg)

查找时，可以从顶级链表开始找。一旦发现被查找的元素大于当前链表中的取值，就会转入下一层链表继续找

