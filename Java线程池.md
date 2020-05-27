# Java 线程池

## 线程池体系结构

## 1.简介

经历了 Java 内存模型、J.U.C 基础之 AQS 、CAS 、Lock 、并发工具类、并发容器、阻塞队列、Atomic 类后，我们开始 J.U.C 的最后一部分：线程池。在这个部分你将了解到下面四个部分：

* 线程池的基础架构
* 线程池的原理分析
* 线程池核心类的源码分析
* 线程池调优

## 2.Executor 体系

我们先看线程池的基础架构图:

![image](E:\研究生学习\Work\技术笔记\Java线程池.assets\2019-03-20-150237.png)

### 2.1.Executor

java.util.concurrent.Executor ，任务的执行者接口，线程池框架中几乎所有类都直接或者间接实现 Executor 接口，它是线程池框架的基础。

Executor 提供了一种将“任务提交”与“任务执行”分离开来的机制，它仅提供了一个 #execute(Runnable command) 方法，用来执行已经提交的 Runnable 任务。代码如下：

```java
public interface Executor {

    void execute(Runnable command);
    
}
```

### 2.2.ExcutorService

java.util.concurrent.ExcutorService ，继承 Executor 接口，**它是“执行者服务”接口，它是为”执行者接口 Executor “服务而存在的。准确的地说，ExecutorService 提供了”将任务提交给执行者的接口( submit 方法)**”，”让执行者执行任务( invokeAll , invokeAny 方法)”的接口等等。代码如下：

```java
public interface ExecutorService extends Executor {

    /**
     * 启动一次顺序关闭，执行以前提交的任务，但不接受新任务
     */
    void shutdown();

    /**
     * 试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表
     */
    List<Runnable> shutdownNow();

    /**
     * 如果此执行程序已关闭，则返回 true。
     */
    boolean isShutdown();

    /**
     * 如果关闭后所有任务都已完成，则返回 true
     */
    boolean isTerminated();

    /**
     * 请求关闭、发生超时或者当前线程中断，无论哪一个首先发生之后，都将导致阻塞，直到所有任务完成执行
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    // ========== 提交任务 ==========

    /**
     * 提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    Future<?> submit(Runnable task);

    /**
     * 执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### 2.3.AbstractExecutorService

java.util.concurrent.AbstractExecutorService ，抽象类，实现 ExecutorService 接口，为其提供默认实现。

AbstractExecutorService 除了实现 ExecutorService 接口外，还提供了 #newTaskFor(...) 方法，返回一个 RunnableFuture 对象，在运行的时候，它将调用底层可调用任务，作为 Future 任务，它将生成可调用的结果作为其结果，并为底层任务提供取消操作。

### 2.4.ScheduledExecutorService

java.util.concurrent.ScheduledExecutorService ，继承 ExecutorService ，为一个“延迟”和“定期执行”的 ExecutorService 。他提供了如下几个方法，安排任务在给定的延时执行或者周期性执行。代码如下：

```java
// 创建并执行在给定延迟后启用的 ScheduledFuture。
<V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit)

// 创建并执行在给定延迟后启用的一次性操作。
ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)

// 创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；
//也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)

// 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)

```

### 2.5.ThreadPoolExecutor

大名鼎鼎的“线程池”，后续做详细介绍。

### 2.6.ScheduledThreadPoolExecutor

java.util.concurrent.ScheduledThreadPoolExecutor ，继承 ThreadPoolExecutor ，并且实现 ScheduledExecutorService 接口，是两者的集大成者，相当于提供了“延迟”和“周期执行”功能的 ThreadPoolExecutor 。

### 2.7.Executors

静态工厂类，提供了 Executor、ExecutorService 、ScheduledExecutorService、ThreadFactory 、Callable 等类的静态工厂方法，通过这些工厂方法我们可以得到相对应的对象。

* 创建并返回设置有常用配置字符串的 ExecutorService 的方法。
* 创建并返回设置有常用配置字符串的 ScheduledExecutorService 的方法。
* 创建并返回“包装的” ExecutorService 方法，它通过使特定于实现的方法不可访问来禁用重新配置。
* 创建并返回 ThreadFactory 的方法，它可将新创建的线程设置为已知的状态。
* 创建并返回非闭包形式的 Callable 的方法，这样可将其用于需要 Callable 的执行方法中。

## 3.Future 体系

Future 接口和实现 Future 接口的 FutureTask 代表了线程池的异步计算结果。

AbstractExecutorService 提供了 #newTaskFor(...) 方法，返回一个RunnableFuture 对象。除此之外，当我们把一个 Runnable 或者 Callable 提交给（ submit 方法）ThreadPoolExecutor 或者 ScheduledThreadPoolExecutor 时，他们则会向我们返回一个FutureTask对象。如下：

```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}

<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<> submit(Runnable task);
```

### 3.1.Future

java.util.concurrent.Future ，作为异步计算的顶层接口，Future 对具体的 Runnable 或者 Callable 任务提供了三种操作：

* 执行任务的取消
* 查询任务是否完成
* 获取任务的执行结果

其接口定义如下：

```java
public interface Future<V> {

    /**
     * 试图取消对此任务的执行
     * 如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。
     * 当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。
     * 如果任务已经启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * 如果在任务正常完成前将其取消，则返回 true
     */
    boolean isCancelled();

    /**
     * 如果任务已完成，则返回 true
     */
    boolean isDone();

    /**
     *   如有必要，等待计算完成，然后获取其结果
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * 如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### 3.2.RunnableFuture

java.util.concurrent.RunnableFuture ，继承 Future、Runnable 两个接口，为两者的合体，即所谓的 Runnable 的 Future 。提供了一个 #run() 方法，可以完成 Future 并允许访问其结果。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    //在未被取消的情况下，将此 Future 设置为计算的结果
    void run();
}
```

### 3.3.FutureTask

java.util.concurrent.FutureTask ，实现 RunnableFuture 接口，既可以作为 Runnable 被执行，也可以作为 Future 得到 Callable 的返回值。

## ThreadPoolExecutor

　　如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。

　　那么有没有一种办法使得线程可以复用，就是执行完一个任务，并不被销毁，而是可以继续执行其他的任务？

## 介绍

ThreadPoolExecutor 主要解决了两个问题：

* 当有大量的异步任务执行的时候，提供了一个比较好的表现（线程复用，减少每个任务的调用开销）
* 它们提供了一种限制和管理资源的方法，包括执行任务集合时消耗的线程



![image-20200222215417018](E:\研究生学习\Work\技术笔记\Untitled.assets\image-20200222215417018.png)

Executor是一个顶层接口，在它里面只声明了一个方法execute(Runnable)，返回值为void，参数为Runnable类型，从字面意思可以理解，就是用来执行传进去的任务的；

然后ExecutorService接口继承了Executor接口，并声明了一些方法：submit、invokeAll、invokeAny以及shutDown等；

抽象类AbstractExecutorService实现了ExecutorService接口，基本实现了ExecutorService中声明的所有方法；

然后ThreadPoolExecutor继承了类AbstractExecutorService。

在ThreadPoolExecutor类中有几个非常重要的方法：`execute()` `submit()`  `shutdown()` `shutdownNow()`

**execute() 和 submit() 方法的区别**

 　execute()方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

　 submit()方法是在ExecutorService中声明的方法，在AbstractExecutorService就已经有了具体的实现，在ThreadPoolExecutor中并没有对其进行重写，这个方法也是用来向线程池提交任务的，但是它和execute()方法不同，它能够返回任务执行的结果，去看submit()方法的实现，会发现它实际上还是调用的execute()方法，只不过它利用了Future来获取任务执行结果

### 构造函数

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

* `int corePoolSize`  ：核心线程参数，定义了最小可以同时运行的线程数量。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建corePoolSize个线程或者一个线程。当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中；
* `int maximumPoolSize` : 当前对列存放的任务数量达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数
* `BlockingQueue<Runnable> workQueue`：当新任务来的时候会先判断当前线程数是否到达核心线程数，如果到达到则会被存放在队列中
  * ArrayBlockingQueue：基于数组结构的有界阻塞队列，FIFO。
  * LinkedBlockingQueue：基于链表结构的有界阻塞队列，FIFO。
  * SynchronousQueue：不存储元素的阻塞队列，每个插入操作都必须等待一个移出操作，反之亦然。
  * PriorityBlockingQueue：具有优先界别的阻塞队列。
* `long keepAliveTime `：当线程池中线程的数量大于核心线程数的时候，如果没有新的任务提交，核心线程外的线程不会立即被销毁，而是等到时间超过了 `keepAliveTime` 才会被销毁
* `TimeUnit unit` ：keepAliveTime 时间单位
* `ThreadFactory threadFactory` 创建新线程时会用到
* `RejectedExecutionHandler handler`：饱和策略 当数达到最大线程数阻塞队列满的时候，可以执行饱和策略
  * 若当前同时运行的线程数量达到最大线程数量并且队列已经满了时，采取的一些策略

    - **ThreadPoolExecutor.AbortPolicy**：抛出 `RejectedExecutionException`来拒绝新任务的处理。
    - **ThreadPoolExecutor.CallerRunsPolicy**：调用执行自己的线程运行任务。您不会丢弃任何任务请求。但是这种策略会降低对于新任务提交速度，影响程序的整体性能。另外，这个策略会增加队列容量，当最大池被填满时，这个策略为我们提供伸缩队列
    - **ThreadPoolExecutor.DiscardPolicy：** 不处理新任务，直接丢弃掉。
    - **ThreadPoolExecutor.DiscardOldestPolicy：** 此策略将丢弃最早的未处理的任务请求。


## 实现原理

### 一、变量

#### 线程的状态

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     *
     * In order to pack them into one int, we limit workerCount to
     * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
     * billion) otherwise representable. If this is ever an issue in
     * the future, the variable can be changed to be an AtomicLong,
     * and the shift/mask constants below adjusted. But until the need
     * arises, this code is a bit faster and simpler using an int.
     *
     * The workerCount is the number of workers that have been
     * permitted to start and not permitted to stop.  The value may be
     * transiently different from the actual number of live threads,
     * for example when a ThreadFactory fails to create a thread when
     * asked, and when exiting threads are still performing
     * bookkeeping before terminating. The user-visible pool size is
     * reported as the current size of the workers set.
     *
     * The runState provides the main lifecycle control, taking on values:
     *
     *   RUNNING:  Accept new tasks and process queued tasks
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
     *   TERMINATED: terminated() has completed
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
     *
     * Detecting the transition from SHUTDOWN to TIDYING is less
     * straightforward than you'd like because the queue may become
     * empty after non-empty and vice versa during SHUTDOWN state, but
     * we can only terminate if, after seeing that it is empty, we see
     * that workerCount is 0 (which sometimes entails a recheck -- see
     * below).
     */
    /**
     * ctl 为原子类型的变量, 有两个概念
     * workerCount, 表示有效的线程数
     * runState, 表示线程状态, 是否正在运行, 关闭等
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 容量 2^29 - 1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // 线程池五种状态
    // 高三位为 111，表示接收新任务 处理排队业务
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 高三为 000 不接受任务 处理排队任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 高三位 001 不接受新任务 处理排队任务 并中断正在进行的任务
    private static final int STOP       =  1 << COUNT_BITS;
    // 001 所有任务都已终止, 工作线程为0, 线程转换到状态TIDYING, 将运行terminate()钩子方法
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 11, 标识terminate（）已经完成
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl 用来计算线程的方法
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
}
```

 ctl 是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它包含两部分的信息：线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，这里可以看到，使用了Integer类型来保存，高3位保存runState，低29位保存workerCount。COUNT_BITS 就是29，CAPACITY就是1左移29位减1（29个1），这个常量表示workerCount的上限值，大约是5亿。

下面再介绍下线程池的运行状态，线程池一共有五种状态，分别是：

| 状态       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| RUNNING    | **能接受新提交的任务，并且也能处理阻塞队列中的任务**         |
| SHUTDOWN   | 关闭状态，不再接受新提交的任务，**但却可以继续处理阻塞队列中已保存的任务**。在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方法进入该状态） |
| STOP       | 不能接受新任务，也不处理队列中的任务，**会中断正在处理任务的线程**。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态 |
| TIDYING    | 如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态 |
| TERMINATED | 在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做 |

进入TERMINATED的条件如下：

- 线程池不是RUNNING状态；
- 线程池状态不是TIDYING状态或TERMINATED状态；
- 如果线程池状态是SHUTDOWN并且workerQueue为空；
- workerCount为0；
- 设置TIDYING状态成功。

下图为线程池的状态转换过程：

![img](E:\研究生学习\Work\技术笔记\Untitled.assets\963903-20190422210922434-953703435.png)

计算线程的几个方法：

| 方法          | 描述                         |
| ------------- | ---------------------------- |
| runStateOf    | 获取运行状态                 |
| workerCountOf | 获取活动线程数               |
| ctlOf         | 获取运行状态和活动线程数的值 |

 #### 其他重要参数

```java
private final BlockingQueue<Runnable> workQueue;              //任务缓存队列，用来存放等待执行的任务
private final ReentrantLock mainLock = new ReentrantLock();   //线程池的主要状态锁，对线程池状态（比如线程池大小
                                                              //、runState等）的改变都要使用这个锁
private final HashSet<Worker> workers = new HashSet<Worker>();  //用来存放工作集
 
private volatile long  keepAliveTime;    //线程存货时间   
private volatile boolean allowCoreThreadTimeOut;   //是否允许为核心线程设置存活时间
private volatile int   corePoolSize;     //核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
private volatile int   maximumPoolSize;   //线程池最大能容忍的线程数
 
private volatile int   poolSize;       //线程池中当前的线程数
 
private volatile RejectedExecutionHandler handler; //任务拒绝策略
 
private volatile ThreadFactory threadFactory;   //线程工厂，用来创建线程
 
private int largestPoolSize;   //用来记录线程池中曾经出现过的最大线程数
 
private long completedTaskCount;   //用来记录已经执行完毕的任务个数
```

　每个变量的作用都已经标明出来了，这里要重点解释一下corePoolSize、maximumPoolSize、largestPoolSize三个变量。

> 　　corePoolSize在很多地方被翻译成核心池大小，其实我的理解这个就是线程池的大小。举个简单的例子：
>
> 　　假如有一个工厂，工厂里面有10个工人，每个工人同时只能做一件任务。
>
> 　　因此只要当10个工人中有工人是空闲的，来了任务就分配给空闲的工人做；
>
> 　　当10个工人都有任务在做时，如果还来了任务，就把任务进行排队等待；
>
> 　　如果说新任务数目增长的速度远远大于工人做任务的速度，那么此时工厂主管可能会想补救措施，比如重新招4个临时工人进来；
>
> 　　然后就将任务也分配给这4个临时工人做；
>
> 　　如果说着14个工人做任务的速度还是不够，此时工厂主管可能就要考虑不再接收新的任务或者抛弃前面的一些任务了。
>
> 　　当这14个工人当中有人空闲时，而新任务增长的速度又比较缓慢，工厂主管可能就考虑辞掉4个临时工了，只保持原来的10个工人，毕竟请额外的工人是要花钱的。

 

　　这个例子中的corePoolSize就是10，而maximumPoolSize就是14（10+4）。

　　也就是说corePoolSize就是线程池大小，maximumPoolSize在我看来是线程池的一种补救措施，即任务量突然过大时的一种补救措施。

　　不过为了方便理解，在本文后面还是将corePoolSize翻译成核心池大小。

　　largestPoolSize只是一个用来起记录作用的变量，用来记录线程池中曾经有过的最大线程数目，跟线程池的容量没有任何关系。

### 二、execute() 方法

```java
  /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
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
        // 获取当前线程池的状态，获得当前工作线程数和线程的状态
        int c = ctl.get();
        // 计算工作线程数 并判断是否小于核心线程数 
        // workCountOf 取出低 29 位的值，表示当前活动的线程数，
        // 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中
        
        if (workerCountOf(c) < corePoolSize) {
            // 并把任务添加到该线程中
            /*
             * addWorker中的第二个参数表示限制添加线程的数量的判断
             * 如果为true，根据corePoolSize来判断；
             * 如果为false，则根据maximumPoolSize来判断
         	*/
            if (addWorker(command, true))
                return;
            // 提交失败再次获取当前状态
            c = ctl.get();
        }
        // 判断线程状态 如果是当前运行的状态 并且任务添加到队列成功
        if (isRunning(c) && workQueue.offer(command)) {
            // 再次获取状态
            int recheck = ctl.get();
             /*
             * 再次判断线程池的运行状态，如果不是运行状态，
             * 由于之前已经把command添加到workQueue中了，
             * 这时需要移除该command
             * 执行过后通过handler使用拒绝策略对该任务进行处理，整个方法返回
             */
            if (! isRunning(recheck) && remove(command))
                // 调用拒绝策略
                reject(command);
            // 如果工作线程为0 则把他加入到运行队列
             /*
              * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
              * 这里传入的参数表示：
              * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
              * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
              * 如果判断workerCount大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。    
              */
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 提交任务失败 走拒绝策略
        /*
         * 如果执行到这里，有两种情况：
         * 1. 线程池已经不是RUNNING状态；
         * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
         * 这时，再次调用addWorker方法，但第二个参数传入为false，将线程池的有限线程数量的上限设置为maximumPoolSize；
         * 如果失败则拒绝该任务
         */
        else if (!addWorker(command, false))
            reject(command);
    }
```

![img](E:\研究生学习\Work\技术笔记\Java线程池.assets\677054-20170408210905472-1864459025.png)

简单来说，在执行 execute() 方法时如果状态一直是RUNNING时，的执行过程如下：

1. 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务；
2. 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中；
3. 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务；
4. 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

这里要注意一下 `addWorker(null, false)` ，也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workQueue中获取任务。所以，在 workerCountOf(recheck) == 0 时执行 addWorker(null, false) 也是为了保证线程池在RUNNING状态下必须要有一个线程来执行任务。

execute方法执行流程如下：

![img](E:\研究生学习\Work\技术笔记\Untitled.assets\963903-20190422213224964-266272939.png)

#### 为啥还要有第二个 if

源码里解释如下：

> If a task can be successfully queued, then we still need to double-check whether we should have added a thread (because existing ones died since last checking) or that the pool shut down since entry into this method. So we recheck state and if necessary roll back the enqueuing if stopped, or start a new thread if there are none.

如果线程添加至队列成功，我们还需要再次判断一下我们是否需要再添加一个线程，有以下两个原因：

1. 如果线程在上一次检查之后死去了，如果不是 running 状态，这时我们需要“回滚” 之前的加入状态，即把这个线程从工作队列取出来，调用拒绝策略。

2. 线程池进入这个方法之后处于关闭了，不再处理新的线程了。那如果没有线程的工作数为0，则创建一个线程，但是不去启动，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断。 ` addWorker(null, false);` 是为了保证线程池在允许时有线程在执行任务

   


### 三、addWorker 方法

这个方法主要是在线程池中创建一个新的线程并执行，参数如下

* **`Runable firstTack`** 代表加入后执行的第一个任务。当前活动线程少于给定的值时或者当队列满的时候，会直接创建这个线程；
* **`boolean core`**   True 表示在新增线程时会判断当前活动线程数是否少于corePoolSize ；False表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize

return false ：线程创建失败、线程池停止或者 shut down 

return true：创建成功

**addWorker方法有4种传参的方式：**

  1、addWorker(command, true)

  2、addWorker(command, false)

  3、addWorker(null, false)

  4、addWorker(null, true)

在execute方法中就使用了前3种，结合这个核心方法进行以下分析
  第一个：线程数小于corePoolSize时，放一个需要处理的task进Workers Set。如果Workers Set长度超过corePoolSize，就返回false
  第二个：当队列被放满时，就尝试将这个新来的task直接放入Workers Set，而此时Workers Set的长度限制是maximumPoolSize。如果线程池也满了的话就返回false
  第三个：放入一个空的task进workers Set，长度限制是maximumPoolSize。这样一个task为空的worker在线程执行的时候会去任务队列里拿任务，这样就相当于创建了一个新的线程，只是没有马上分配任务
  第四个：这个方法就是放一个null的task进Workers Set，而且是在小于corePoolSize时，如果此时Set中的数量已经达到corePoolSize那就返回false，什么也不干。实际使用中是在prestartAllCoreThreads()方法，这个方法用来为线程池预先启动corePoolSize个worker等待从workQueue中获取任务执行

```java
/**
 * Checks if a new worker can be added with respect to current
 * pool state and the given bound (either core or maximum). If so,
 * the worker count is adjusted accordingly, and, if possible, a
 * new worker is created and started, running firstTask as its
 * first task. This method returns false if the pool is stopped or
 * eligible to shut down. It also returns false if the thread
 * factory fails to create a thread when asked.  If the thread
 * creation fails, either due to the thread factory returning
 * null, or due to an exception (typically OutOfMemoryError in
 * Thread.start()), we roll back cleanly.
 *
 * @param firstTask the task the new thread should run first (or
 * null if none). Workers are created with an initial first task
 * (in method execute()) to bypass queuing when there are fewer
 * than corePoolSize threads (in which case we always start one),
 * or when the queue is full (in which case we must bypass queue).
 * Initially idle threads are usually created via
 * prestartCoreThread or to replace other dying workers.
 *
 * @param core if true use corePoolSize as bound, else
 * maximumPoolSize. (A boolean indicator is used here rather than a
 * value to ensure reads of fresh values after checking other pool
 * state).
 * @return true if successful
 */
/**
 * 检查任务是否可以提交
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层循环
    for (;;) {
        // 获取运行状态
        int c = ctl.get();
        int rs = runStateOf(c);

        /*
         * 这个if判断
         * 如果rs >= SHUTDOWN，则表示此时不再接收新任务；
         * 接着判断以下3个条件，只要有1个不满足，则返回false：
         * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
         * 2. firsTask为空
         * 3. 阻塞队列不为空
         * 
         * 首先考虑rs == SHUTDOWN的情况
         * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false；
         * 然后，如果firstTask为空，并且workQueue也为空，则返回false，
         * 因为队列中已经没有任务了，不需要再添加线程了
         */
        // Check if queue empty only if necessary. 检查线程池是否关闭
        if (rs >= SHUTDOWN &&
            !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;
        // 内层循环
        for (;;) {
            // 获取线程数
            int wc = workerCountOf(c);
            // 工作线程大于容量 或者大于 核心或最大线程数
            /*
             * 如果wc超过CAPACITY，也就是ctl的低29位的最大值（二进制是29个1），返回false；
             * 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，
             * 如果为false则根据maximumPoolSize来比较。
             */
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // CAS 线程数增加, 成功则调到外层循环
            /*
             * 尝试增加workerCount，如果成功，则跳出第一个for循环
             */
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 如果增加workerCount失败，则重新获取ctl的值
            c = ctl.get();  // Re-read ctl
            // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    /**
     * 创建新worker 开始新线程
     */
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 根据firstTask来创建Worker对象
        w = new Worker(firstTask);
        // 每一个Worker对象都会创建一个线程
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                /*
                 * rs < SHUTDOWN表示是RUNNING状态；
                 * 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                 * 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                 */
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 判断线程是否存活, 已存活抛出非法异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //  设置包含池中的所有工作线程。仅在持有mainLock时访问 workers是 HashSet 集合
                    workers.add(w);
                    int s = workers.size();
                    // 设置池最大大小, 并将 workerAdded设置为 true
                    // largestPoolSize记录着线程池中出现过的最大线程数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                // 解锁
                mainLock.unlock();
            }
            // 添加成功 开始启动线程 并将 workerStarted 设置为 true
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 启动线程失败
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

![img](E:\研究生学习\Work\技术笔记\Java线程池.assets\677054-20170408211358816-1277836615.png)

**具体过程**

**循环校验**

1. 外层循环判断检查线程池是否关闭

   ```java
   if (rs >= SHUTDOWN &&
               !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
               return false;
   ```

   这个if判断 `s >= SHUTDOWN`，则表示此时不再接收新任务；接着判断以下3个条件，只要有1个不满足，则返回false：

   * `rs == SHUTDOWN`，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务，所以在firstTask不为空的时候会返回false；

      * `firsTask == null` 当阻塞队列为空的时候，因为队列中已经没有任务了，不需要再添加线程了
      * 阻塞队列不为空

2. 内层循环检查线程的个数,

   * 检查 工作线程大于容量 或者大于核心或最大线程数 是的话return false

   * **CAS** 尝试增加workCount，如果成功，则跳出第一个for循环
   * 如果失败则重新获取线程池的状态，看线程池的状态是否改变，如果改变则返回最外层的for 循环继续检验状态；没改变则继续重试 CAS 增加 workCount

   ```java 
   // 内层循环
   for (;;) {
       // 获取线程数
       int wc = workerCountOf(c);
       // 工作线程大于容量 或者大于 核心或最大线程数
       /*
        * 如果wc超过CAPACITY，也就是ctl的低29位的最大值（二进制是29个1），返回false；
        * 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，
        * 如果为false则根据maximumPoolSize来比较。
        */
       if (wc >= CAPACITY ||
           wc >= (core ? corePoolSize : maximumPoolSize))
           return false;
       // CAS 线程数增加, 成功则调到外层循环
       /*
        * 尝试增加workerCount，如果成功，则跳出第一个for循环
        */
       if (compareAndIncrementWorkerCount(c))
           break retry;
       // 如果增加workerCount失败，则重新获取ctl的值
       c = ctl.get();  // Re-read ctl
       // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
       if (runStateOf(c) != rs)
           continue retry;
       // else CAS failed due to workerCount change; retry inner loop
   }
   ```

**开启线程工作**

1. 根据 firstTask 来创建一个 Worker 对象。每个 Worker 对象都会创建一个线程。

2. 加锁 (可重入锁) 添加线程。
3. 加锁期间重新判断线程池的状态，如果ThreadFactory启动失败或者线程池 shutdown 则返回失败
4. 在 线城池处于RUNNING 或者 SHOTDOWN 且 firstTask == null 时：
5. 线程如果存活的话 抛出非法异常 （此时线程应该还没有被启动）
6. 设置包含池中的所有工作线程。仅在持有mainLock时访问 workers是 HashSet 集合，将线程 work 加入到 words中， 他记录了的线程池里的所有工作线程
7. 更新largestPoolSize，largestPoolSize记录着线程池中出现过的最大线程数量  workerAdded = true
8. 解锁
9. 添加成功 开始启动线程 并将 workerStarted 设置为 true 
10. 否则返回失败

```java
 /**
     * 创建新worker 开始新线程
     */
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 根据firstTask来创建Worker对象
        w = new Worker(firstTask);
        // 每一个Worker对象都会创建一个线程
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                /*
                 * rs < SHUTDOWN表示是RUNNING状态；
                 * 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程。
                 * 因为在SHUTDOWN时不会在添加新的任务，但还是会执行workQueue中的任务
                 */
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 判断线程是否存活, 已存活抛出非法异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //  设置包含池中的所有工作线程。仅在持有mainLock时访问 workers是 HashSet 集合
                    workers.add(w);
                    int s = workers.size();
                    // 设置池最大大小, 并将 workerAdded设置为 true
                    // largestPoolSize记录着线程池中出现过的最大线程数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                // 解锁
                mainLock.unlock();
            }
            // 添加成功 开始启动线程 并将 workerStarted 设置为 true
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 启动线程失败
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

**为啥 ReentranLock** 

> Lock held on access to workers set and related bookkeeping.While we could use a concurrent set of some sort, it turns out to be generally preferable to use a lock. Among the reasons is that this serializes interruptIdleWorkers, which avoids  unnecessary interrupt storms, especially during shutdown.Otherwise exiting threads would concurrently interrupt those that have not yet interrupted. It also simplifies some of the associated statistics bookkeeping of largestPoolSize etc. We also hold mainLock on shutdown and shutdownNow, for the sake of ensuring workers set is stable while separately checking  permission to interrupt and actually interrupting.

使用锁来实现加入线程池的线程集合，相关的最大线程数量的记录。尽管我们可以用一些并发的集合，但是事实证明我们使用锁更好。它序列化了interruptIdleWorkers（中断的空闲任务？？），从而避免了不必要的中断风暴。**特别是在线程池shut down期间，退出的线程将同时中断那些尚未中断的线程。这会引起一个中断风暴。**它还简化了一些相关的统计数据簿记的最大池大小等。我们现在也在关机和关机时保持这个锁，以确保工人的设置是稳定的，同时分别检查中断和实际中断的权限。

### 四、Worker 类



```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

