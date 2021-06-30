# 16 线程池

## 为什么要使用线程池

使用线程池主要有以下三个原因：

1. 创建/销毁线程需要消耗系统资源，线程池可以**复用已创建的线程**。
2. **控制并发的数量**。并发数量过多，可能会导致资源消耗过多，从而造成服务器崩溃。（主要原因）
3. **可以对线程做统一管理**。

## 线程池分类

* ThreadPoolExecutor
* ForkJoinPool

## Executor架构体系

Java中的线程池顶层接口是Executor接口，Executor提供了线程执行的方法execute\(\)，ExecutorService提供了线程中断方法，并且还提供了一个新的线程异步执行的方法submit\(\)。ThreadPoolExecutor是其基本实现

ExecutorService

​ AbstractExecutorService \(java.util.concurrent\) ​ ThreadPoolExecutor \(java.util.concurrent\) ​ ForkJoinPool \(java.util.concurrent\)

### execute\(\)和submit\(\)区别

Executor提供基本的执行方法是execute，但是缺点是改方法没有返回值。因此在ExecutorService中又提供了带返回值的线程执行方法submit

ExecutorService：具备管理执行器和任务生命周期的方法，提交任务机制更完善。

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


// -----------------------------------------------------------------------------------------
public interface ExecutorService extends Executor {
    Future<?> submit(Runnable task);
```

**区别：**

* execute\(\) ：提交无返回值的任务，所以无法判断任务在线程池中是否执行成功。
* submit\(\) ：异步提交，线程池会返回⼀个 Future 类型的对象。**调用Future的get\(\)方法，会阻塞当前线程直到任务完成**。也可以调用 get（long timeout， TimeUnit unit） 设置阻塞的时长。

因此，使用线程池的时候，需要注意对象引用不要用线程池顶级接口Executor，否则无法调用submit\(\)方法

```java
public static void main(String[] args) {
    ThreadPoolExecutor pool1 = new ThreadPoolExecutor(
            5,
            10,
            10L,
            TimeUnit.MINUTES,
            new LinkedBlockingDeque<>(50),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy()
    );

    // pool1.submit();
    // pool1.execute();

    Executor pool2 = new ThreadPoolExecutor(
            5,
            10,
            10L,
            TimeUnit.MINUTES,
            new LinkedBlockingDeque<>(50),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy()
    );
    // pool2无submit，因为对象引用是线程池顶级接口Executor
    // pool1.execute();
}
```

### Future

Future：未来的意思

我们可以使用Runnable和Thread来创建一个新的线程，但是问题是没有返回值。

因为在jDK1.5提供了Callable，那这个值怎么拿？就是通过Future去获取，而FutureTask是其实现类，FutureTask既是一个Future，也是一个Task。

注：CompletableFuture是更高级的Future，提供了更多特性。

**关键字：Future、FutureTask、CompletableFuture**

## ThreadPoolExecutor

接下来，看看线程池的基本实现类，ThreadPoolExecutor，一共提供了7个参数。

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
        5,
        10,
        10L,
        TimeUnit.MINUTES,
        new LinkedBlockingDeque<>(50),
        Executors.defaultThreadFactory(),
        new ThreadPoolExecutor.AbortPolicy()
);
```

## 线程池参数

重要的参数！重要的参数！重要的参数！

### corePoolSize

corePoolSize：核心线程数

**什么是核心线程数解释**

线程池中有两类线程，核心线程和非核心线程。

* 核心线程默认情况下会**一直存在**于线程池中，即使这个核心线程什么都不干（铁饭碗）
* 而非核心线程如果长时间的闲置，就会被销毁（临时工）。

### maximumPoolSize

maximumPoolSize：最大线程数

最大线程数 = 核心线程数 + 非核心线程数

注：当任务队列满时，会创建非核心线程数去执行。当线程池满时，会采用拒绝策略。

### keepAliveTime

keepAliveTime：非核心线程存活时长

非核心线程如果处于闲置状态超过该值，就会被销毁。如果设置allowCoreThreadTimeOut\(true\)，则也会作用于核心线程。

keepAliveTime = 0 则非核心线程执行完毕后直接销毁结束，不存活。

### TimeUnit

unit：keepAliveTime的单位

例如：TimeUnit.MINUTES 分

### BlockingQueue

BlockingQueue：任务队列

当**线程**数满了，没有空余的线程执行**任务**时，任务就进入了等待队列（阻塞队列）。BlockingQueue一般用于生产者-消费者模式

注：这里有个线程和任务的概念。阻塞队列的概念见【多线程 -并发容器】一章

四种阻塞队列：

1. LinkedBlockingQueue：链式阻塞队列，底层数据结构是**链表**，默认大小是**Integer.MAX\_VALUE**，也可以指定大小。
2. ArrayBlockingQueue：数组阻塞队列，底层数据结构是**数组**，需要指定队列的大小。
3. SynchronousQueue：同步队列，内部容量**为0**，put需等待take，take需等待put。
4. DelayQueue：延迟队列，按照延迟时间入队，**延迟时间到了才能从队列中获取到该元素**。

### ThreadFactory

ThreadFactory：线程工程

创建线程的工厂 ，ThreadFactory提供了一个接口newThread。意味着，可以在创建线程时统一**设置一些参数，例如：线程名称**。

注：默认的线程工厂是在Executors类中，Executors.defaultThreadFactory\(\)。前面我们提到了Executor，两者命名有些相近，后面会讨论。

```java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}

// ---------------------------------------------------------------------------------------------------------------------------
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }


    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            // 设置为非守护线程
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            // 设置优先级为默认
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

### RejectedExecutionHandler

RejectedExecutionHandler：拒绝策略

当线程数量 &gt; maximumPoolSize时，线程池存不下了，就需要一种拒绝策略处理。

四种拒绝策略 ：

1. ThreadPoolExecutor.AbortPolicy：丢弃任务，并抛出RejectedExecutionException异常。**默认拒绝处理策略**，
2. ThreadPoolExecutor.DiscardPolicy：丢弃新来的任务，但是不抛出异常
3. ThreadPoolExecutor.DiscardOldestPolicy：丢弃最早的任务，然后重新尝试执行程序（如果再次失败，重复此过程）
4. ThreadPoolExecutor.CallerRunsPolicy：由调用的线程处理该任务，即调用`execute`方法的线程来执行

注：ThreadPoolExecutor.CallerRunsPolicy存在一个风险：会创建大量的线程，导致线上OOM。

当然如果业务量没那么高还是可以使用，能够保证消息不丢失。

如果不了解线程业务量，建议使用默认策略，即AbortPolicy丢失并抛异常。

拒绝策略不止四种，上面只是提供的拒绝策略。当然我们也可以自定义拒绝策略，只需实现RejectedExecutionHandler即可。

例如：将消息保存下来。

```java
public class SavePolicy implements RejectedExecutionHandler {

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // send to mq
    }
}
```

### 小结

线程池中核心线程数、最大线程数、非核心线程数存活时间及其单位、阻塞队列，这五个参数是必须自定义的，是非常重要的参数。

对于创建线程工厂与拒绝策略不是必须自定义的，我们可以使用默认的设置。

线程池的流程如下图：

![&#x7EBF;&#x7A0B;&#x6C60;.png](../../.gitbook/assets/线程池.png)

## Executor和Executors

在JDK源码中，有一个命名习惯，如数组的工具类叫Arrays，容器的工具类叫Collections

Executor是线程池的顶级接口，线程池的工具类就叫做：Executors，也叫线程池的工厂

## 四种常见线程池

### newSingleThreadExecutor

singleThread：单线程的线程池（为什么需要单线程的线程池？因为线程池有任务队列，并且可以管理线程的生命周期）

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

有且仅有一个核心线程，能够保证任务按照顺序执行。如果唯一的线程不空闲，则进入任务队列。

用途：**保证任务按照顺序执行**

### newCachedThreadPool

Cached：缓存的意思，重在复用。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

没有核心线程0。当需要执行很多**短时间**的任务时，CacheThreadPool的线程复用率比较高，而且线程60s后会回收，意味着即使没有任务进来，也不会占用很多资源。

用途：执行很多短时间的任务

### newFixedThreadPool

fixed：固定的

```java
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

核心线程=最大线程数，因此无非核心线程。任务队列用的是LinkedBlockingQueue，默认大小是Integer.MAX\_VALUE

用途：执行任务量比较平缓

#### newFixedThreadPool与CachedThreadPool对比

* 任务量比较平缓用FixedThreadPool
* 任务量忽高忽低，用CacheThreadPool

注：一般我们都不用，而是根据需要自定义

### newScheduledThreadPool

Scheduled：定时

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

用途：定时任务线程池

示例：

```java
public class SchedulePoolDemo {

    public static void main(String[] args) {
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(5);
        // initialDelay: 初始延迟时间
        // period: 周期
        // 2秒执行一次
        executor.scheduleAtFixedRate(() ->{
            System.out.println("***2秒执行一次****");
        },0,2, TimeUnit.SECONDS);
    }
}
```

面试题：假如提供一个闹钟服务，订阅的人很多，10亿人，怎么优化？

## 线程池大小的选择

如果线程池中的线程数量过多，最终会竞争稀缺的CPU和内存，浪费大量的时间在上下文切换。

如果线程池中的线程数量过少，无法充分利用。

* CPU密集型：N\(thread\) = N\(cpu\) +1
* I/O密集型：N\(thread\) = N\(cpu\)  _U\(cpu\)_  \(1+W/C\)

Brian Goetz建议，可以通过以下公式估算：

> N\(thread\) = N\(cpu\)  _U\(cpu\)_  \(1+W/C\)
>
> N\(cpu\)：机器多少CPU
>
> U\(cpu\)：期望的CPU利用率
>
> W/C：等待时间（wait）与计算时间（computation）的比率

最终还是以压力测试为准

查看机器线程数，当然我们一般可以用top命令查看，top接着按1

```java
public static void main(String[] args) {
    System.out.println(Runtime.getRuntime().availableProcessors());
}
```

8核机器示例设置：

```java
private static final ThreadPoolExecutor EXECUTOR = new ThreadPoolExecutor(5, 10, 10L, TimeUnit.MINUTES,
            new LinkedBlockingQueue<>(50), new MonitorThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
```

## 线程池状态

线程有自己的状态，同样线程池本身也有自己的状态，线程池本身有一个调度线程，用于管理线程池的各种任务和事件，例如：创建/销毁线程，任务队列管理，线程队列管理等。

RUNNING、SHUTDOWN、STOP、TIDYING 、TERMINATED。

RUNNING：创建线程池后

SHUTDOWN：调用了shutdown\(\)方法后。此时，线程池不能接受新的任务，清除一些空闲worker，会等待阻塞队列的任务完成。

STOP：调用了shutdownNow\(\)方法后。此时，线程池不能接受新的任务，中断所有线程，阻塞队列中没有被执行的任务全部丢弃。

TIDYING（整理的意思）：当所有任务终止，ctl记录的”任务数量”为0，此时为TIDYING状态。

TERMINATED：调用terminated\(\)方法后

## ThreadPoolExecutor源码分析

### 常用变量解释

```java
// `ctl`，可以看做一个int类型的数字，高3位表示线程池状态，低29位表示worker数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// `COUNT_BITS`，`Integer.SIZE`为32，所以`COUNT_BITS`为29
private static final int COUNT_BITS = Integer.SIZE - 3;
// `CAPACITY`，线程池允许的最大线程数。1左移29位，然后减1，即为 2^29 - 1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 线程池有5种状态，按大小排序如下：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// `runStateOf()`，获取线程池状态，通过按位与操作，低29位将全部变成0
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// `workerCountOf()`，获取线程池worker数量，通过按位与操作，高3位将全部变成0
private static int workerCountOf(int c)  { return c & CAPACITY; }
// `ctlOf()`，根据线程池状态和线程池worker数量，生成ctl值
// rs: runState 线程池状态
// wc:workerCount worker数量
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */
// `runStateLessThan()`，线程池状态小于xx
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
// `runStateAtLeast()`，线程池状态大于等于xx
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
```

### execute

从线程池的execute开始，execute的处理流程如下：

1. 新增一个任务，当核心线程能够处理，直接处理
2. 核心线程无空闲，则进入等待队列
3. 核心线程无空闲，等待队列已满，创建非核心线程处理
4. 线程池已满，等待队列已满，执行拒绝策略

注意：这里区分线程池数量，任务队列数量满的情况。

![image.png](../../.gitbook/assets/java38.png)

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
        // 获取线程池状态
        int c = ctl.get();
        // workerCountOf：获取线程池worker的线程数量
        // 当前线程数小于corePoolSize, 则调用addWorker创建核心线程执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            // 再次获取线程池状态
            c = ctl.get();
        }
        // 如果大于corePoolSize，则将任务添加到workQueue队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果isRunning返回false(状态检查)，则remove这个任务，然后执行拒绝策略
            // isRunning返回false，说明线程池执行过shutdown()
            if (!isRunning(recheck) && remove(command))
                reject(command);
            // 线程池处于running状态，但是没有线程，则创建线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 如果放入workQueue失败，则创建非核心线程执行任务
        // 如果这时创建非核心线程失败(当前线程总数不小于maximumPoolSize时)，就会执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

### 为什么会二次检查线程池状态？

```java
 // 如果超过corePoolSize，则将任务添加到workQueue队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果isRunning返回false(状态检查)，则remove这个任务，然后执行拒绝策略
            // isRunning返回false，说明线程池执行过shutdown()
            if (!isRunning(recheck) && remove(command))
                reject(command);
            // 线程池处于running状态，但是没有线程，则创建线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
```

因为在多线程的情况下，线程池的状态是时刻变化的，因此需要再次检查。如果isRunning返回false，说明线程池执行过shutdown\(\)。

你应该也发现了，每次都会再重新获取线程池状态，也是同样的道理。

### 线程池的线程如何实现复用？

我们知道，当我们创建线程的时候，调用了start\(\)方法结束后，就会自动销毁线程，那线程池是如何实现线程的复用呢？

答案便在于addWorker\(\)

### addWorker

创建线程执行任务

注：addWorker主要分为两步，第一步使用双层循环将worker数量+1。第二步创建worker，并启动线程执行任务。

简单来说：count++ ——&gt;addworkder ——&gt; start

```java
// core：用于判断创建的是否核心线程
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层自旋
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 1. 线程池状态大于SHUTDOWN时，直接返回false
        // 2. 线程池状态等于SHUTDOWN，且firstTask不为null，直接返回false
        // 3. 线程池状态等于SHUTDOWN，且队列为空，直接返回false
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        // 内层自旋
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                // 1.如果core是true，证明需要创建的线程为核心线程，则先判断当前线程是否大于核心线程
                // 2.如果core是false，证明需要创建的是非核心线程，则先判断当前线程数是否大于总线程数
                // 如果不小于，则返回false
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 使用CAS的方式增加worker数量。
            // 若增加成功，则直接跳出外层循环进入到第二部分
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 线程池状态发生变化，对外层循环进行自旋
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 1.创建一个worker对象
        w = new Worker(firstTask);
        // 2.实例化一个Thread对象，注意这里调用的thread
        final Thread t = w.thread;
        if (t != null) {
            // 3.线程池全局锁
            final ReentrantLock mainLock = this.mainLock;
            // worker的添加必须是串行的，因此需要加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 检查线程状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                     // worker已经调用过了start()方法，则不再创建worker
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // worker创建并添加到workers成功
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        // 更新`largestPoolSize`变量
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 启动worker线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            // worker线程启动失败，说明线程池状态发生了变化（关闭操作被执行），需要进行shutdown相关操作
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

### Worker

要想搞清楚线程池如何实现复用，还得继续研究Worker类，从上面我们也可以猜测到，Worker是对于线程的一层包装。

阅读Worker，从类的介绍开始。

Worker实现了Runnable，同时又继承了AQS，主要用于维护正在运行的任务。 Worker继承了AQS，用于简化每个任务获取和释放锁。Worker并没有使用ReentrantLock，而是实现了简单的非重入锁，因为不希望调用setCorePoolSize之类方法时能够获取到该锁。另外，为了防止线程真正开始运行任务之前中断，lock锁状态（AQS中的state，作用见【多线程-AQS】）的初始化设置为负值\(-1\)，在启动时会将其清除。

```java
/**
 * Class Worker mainly maintains interrupt control state for
 * threads running tasks, along with other minor bookkeeping.
 * This class opportunistically extends AbstractQueuedSynchronizer
 * to simplify acquiring and releasing a lock surrounding each
 * task execution.  This protects against interrupts that are
 * intended to wake up a worker thread waiting for a task from
 * instead interrupting a task being run.  We implement a simple
 * non-reentrant mutual exclusion lock rather than use
 * ReentrantLock because we do not want worker tasks to be able to
 * reacquire the lock when they invoke pool control methods like
 * setCorePoolSize.  Additionally, to suppress interrupts until
 * the thread actually starts running tasks, we initialize lock
 * state to a negative value, and clear it upon start (in
 * runWorker).
 */
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
```

接着看构造函数

thread：运行worker的线程，null意味着失败

firstTask：要执行的任务，可能为null

```java
/** Thread this worker is running in.  Null if factory fails. */
final Thread thread;
/** Initial task to run.  Possibly null. */
Runnable firstTask;
Worker(Runnable firstTask) {
    // lock的初始值
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    // 使用线程工厂创建一个线程
    this.thread = getThreadFactory().newThread(this);
}
```

因为Worker实现了Runnable，接下来我们可以看看其run方法，最后调用的是runWorker

```java
/** Delegates main run loop to outer runWorker  */
public void run() {
    runWorker(this);
}
```

runWorker分析

```java
 final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        // 线程启动之后，通过unlock方法释放锁，为了可以让外部进行中断
        w.unlock(); // allow interrupts
        // 是否进入过自旋
        boolean completedAbruptly = true;
        try {
            // 核心部分：进行自旋，worker会不断的获取任务，当队列为空时，当前线程会被阻塞等待
            // 只要getTask不为null，就不会退出。注意getTask会进行阻塞，下面会进行单独分析
            // 1. 如果firstTask不为null，则执行firstTask
            // 2. 如果firstTask为null，则调用getTask()从队列获取任务
            while (task != null || (task = getTask()) != null) {
                // 进行加锁操作，保证thread不被其他线程中断(除非线程池被中断)
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 如果线程池处于中断状态，则将当前线程将中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 注：beforeExecute和afterExecute用于扩展
                    // 执行beforeExecute
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务，注意这里是调用run，从而达到复用线程的目的
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 执行afterExecute方法
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null; // help gc
                    // 已完成的任务+1
                    w.completedTasks++;
                    // 解锁操作
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 自旋操作被退出，说明线程池正在结束
            processWorkerExit(w, completedAbruptly);
        }
    }
```

getTask分析

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // allowCoreThreadTimeOut：false，默认。核心线程即使空闲也不会被销毁
        // allowCoreThreadTimeOut：true。核心线程在keepAliveTime内仍空闲则会被销毁
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 1.如果运行线程数超过了最大线程数，但是任务队列空了，递减worker数量
        //
        // 2.如果设置 允许线程超时 或者线程数量超过了核心线程数量，并且线程在规定时间未poll(获取数据并移除)任务，
        // 且队列为空，递减worker数量
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 递减worker数量
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 1.timed为true，则会调用workQueue的poll方法获取任务
            // 2.如果超过keepAliveTime时长， poll返回null
            // 如果timed为false，则调用workQueue.take()方法（workQueue为BlockingQueue，take会阻塞等待任务加入）
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take(); // 会进行阻塞，挂起
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

至此，线程复用的谜题解开。实际上，Worker对任务线程（thread）进行了封装。而Worker本身也实现了Runnable，当创建Worker的时候，其实是调用的worker线程的start\(\)。

实现线程的复用便在于Worker里重写的run\(\)方法，通过AQS实现阻塞，不断的获取任务队列里的任务，任务队列是一个BlockingQueue，take\(\)会进行阻塞等待任务加入，达到不占用资源的目的。**然后调用其run\(\)方法。获取队列为null则进行阻塞**。

### 线程池小结

线程池主要利用了BlockingQueue阻塞队列，里面有几个重要的概念：核心线程、最大线程数、非核心线程存活时长、线程工厂、任务队列、拒绝策略。

线程池的复用主要利用了自旋锁，以及BlockingQueue的阻塞特性。

## ForkJoinPool

前面提到线程池其实有两类，一类是ThreadPoolExecutor，另一类就是ForkJoinPool

### Fork/Join介绍

fork：分叉

join：汇总

Fork/Join其实是一种分而治之的思想**。fork将一个大任务分解成若干个小任务，而join将各个小任务汇总起来，得到大任务的结果。**

![image.png](../../.gitbook/assets/java40.png)

### 工作窃取算法

ForkJoinPool是Fork/Join框架的一种实现，它能够将任务递归分解成最小的任务，最后汇总得到结果。

Fork/Join是一种分而治之的思想，其实现利用了**工作窃取算法**

工作窃取算法：

将任务分成一小块，一小块。当多个线程执行时，如果某个线程已经执行完自己队列的任务，会从其他线程里**窃取**任务过来执行。![image.png](../../.gitbook/assets/java41.png)

注意：当一个线程窃取另一个线程的时候，为了减少两个任务线程之间的竞争，通常使用**双端队列**来存储任务。被窃取的任务线程都从双端队列的**头部**拿任务执行，而窃取其他任务的线程从双端队列的**尾部**执行任务。

另外，当一个线程在窃取任务时要是没有其他可用的任务了，这个线程会进入**阻塞状态**以等待再次“工作”。

![image.png](../../.gitbook/assets/java42.png)

### ForkJoinTask

Fork/Join其实是对任务的分叉和汇总。因此我们要实现Fork/Join，就必须定义我们的两个东西：

1. 任务如何进行分叉？
2. 任务如何进行汇总？

在JUC包下，提供了一个基础的ForkJoin任务类，就是ForkJoinTask。只需要重写对应的方法即可实现任务的分叉和汇总。

但是，我们通常不适用，而是使用其更高级的实现类：RecursiveTask或RecursiveAction（Recursive：递归的意思）

RecursiveTask和RecursiveAction区分：

* RecursiveTask有返回值
* RecursiveAction无返回值

### 示例

* RecursiveTask.fork\(\)
* RecursiveTask.join\(\)

示例一：斐波那契计算

```java
public class ForkJoinDemo {

    public static void main(String[] args) throws InterruptedException, ExecutionException  {
        ForkJoinDemo demo = new ForkJoinDemo();
        demo.test();
    }

    public void test() throws InterruptedException, ExecutionException {
        // 使用ForkJoinPool
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        System.out.println("CPU核数: " + Runtime.getRuntime().availableProcessors());
        long start = System.currentTimeMillis();
        Fibonacci fibonacci = new Fibonacci(40);
        ForkJoinTask<Integer> future = forkJoinPool.submit(fibonacci);
        System.out.println("result: " + future.get());
        long end = System.currentTimeMillis();
        System.out.println(String.format("耗时：%d millis", end - start));
    }

    /**
     * extends RecursiveTask<Integer>
     */
    class Fibonacci extends RecursiveTask<Integer>{

        int n;

        public Fibonacci(int n) {
            this.n = n;
        }

        @Override
        protected Integer compute() {
            // 这里先假设 n >= 0
            if (n <= 1){
                return n;
            }else {
                // f(n-1)
                Fibonacci f1 = new Fibonacci(n-1);
                // f(n-2)
                Fibonacci f2 = new Fibonacci(n-2);
                f1.fork(); // 分叉
                f2.fork(); // 分叉

                // f(n) = f(n-1) + f(n-2)
                return f1.join() + f2.join(); // 汇总
            }
        }
    }
}
```

示例二：数组求和

```java
public class ForkJoinDemo2 {

    static final int MAX_NUM = 1000;
    static int[] numbers = new int[10000];
    static Random r = new Random();
    static {
        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = r.nextInt(100);
        }
        System.out.println("stream api sum:" + Arrays.stream(numbers).sum()); //stream api
    }
    class SumTask extends RecursiveTask<Long>{
        int start, end;
        public SumTask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {
            // 如果数量 < 最大数量 直接计算
            if (end-start <= MAX_NUM){
                long sum = 0L;
                for (int i = start; i < end; i++) {
                    sum += numbers[i];
                }
                return sum;
            }else {
                int mid = start + (end-start) / 2;
                SumTask sumTask1 = new SumTask(start, mid);// 分叉
                SumTask sumTask2 = new SumTask(mid, end);// 分叉
                sumTask1.fork();
                sumTask2.fork();
                return sumTask1.join() + sumTask2.join(); // 汇总
            }
        }
    }


    public static void main(String[] args) {
        ForkJoinDemo2 demo = new ForkJoinDemo2();
        demo.test();
    }

    public void test(){
        // 使用ForkJoinPool
        System.out.println("CPU核数: " + Runtime.getRuntime().availableProcessors());
        long start = System.currentTimeMillis();

        ForkJoinPool forkJoinPool = new ForkJoinPool();
        SumTask task = new SumTask(0, numbers.length);
        forkJoinPool.execute(task);
        System.out.println("result: " + task.join());
        long end = System.currentTimeMillis();
        System.out.println(String.format("耗时：%d millis", end - start));
    }

}
```

### newWorkStealingPool

在前面我们介绍了四种常见的线程池，其实在Executors中还有特殊的一种：newWorkStealingPool，叫做：工作窃取线程池

newWorkStealingPool是对ForkJoinPool的封装，简化了操作

```text
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```

示例：

```java
public class WorkStealingPoolDemo {
    public static void main(String[] args) throws IOException {
        ExecutorService service = Executors.newWorkStealingPool();
        System.out.println("CPU核数: " + Runtime.getRuntime().availableProcessors());

        service.execute(new R(1000));
        service.execute(new R(2000));
        service.execute(new R(2000));
        service.execute(new R(2000)); //daemon
        service.execute(new R(2000));

        //由于产生的是精灵线程（守护线程、后台线程），主线程不阻塞的话，看不到输出
        System.in.read();
    }

    static class R implements Runnable {
        int time;
        R(int t) {
            this.time = t;
        }

        @Override
        public void run() {
            try {
                TimeUnit.MILLISECONDS.sleep(time);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(time  + " " + Thread.currentThread().getName());
        }
    }
}
```

结果：

线程2执行一次后，窃取了未执行的任务，执行了两次

```text
CPU核数: 4
1000 ForkJoinPool-1-worker-2
2000 ForkJoinPool-1-worker-1
2000 ForkJoinPool-1-worker-3
2000 ForkJoinPool-1-worker-0
2000 ForkJoinPool-1-worker-2
```

## Java8 Stream并行计算原理

Java8提供了并行式的流式计算，使我们可以用简单的代码提高计算效率。（可能大部分人对此并不感兴趣，但面试中偶尔也会问到，今天我们就来研究一下）

ParalleStreamAPI示例：

```java
public class StreamParallelDemo {
    public static void main(String[] args) {
        Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .parallel()
                .reduce((a, b) -> {
                    System.out.println(String.format("%s: %d + %d = %d",
                            Thread.currentThread().getName(), a, b, a + b));
                    return a + b;
                })
                .ifPresent(System.out::println);
    }
}
```

首先，生成一个Stream，接着看parallel\(\)方法，只是把一个标识`sourceStage.parallel`设置为`true`。然后返回实例本身

```java
public final S parallel() {
    sourceStage.parallel = true;
    return (S) this;
}
```

继续看reduce\(\)方法

```java
@Override
public final Optional<P_OUT> reduce(BinaryOperator<P_OUT> accumulator) {
    return evaluate(ReduceOps.makeRef(accumulator));
}

// -------------------------------------------------------------------------------------------------------------------
    /**
     * Evaluate the pipeline with a terminal operation to produce a result.
     *
     * @param <R> the type of result
     * @param terminalOp the terminal operation to be applied to the pipeline.
     * @return the result
     */
    final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
        assert getOutputShape() == terminalOp.inputShape();
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        linkedOrConsumed = true;

        // 是否使用并行模式
        return isParallel()
               ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
               : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
    }


// -------------------------------------------------------------------------------------------------------------------
// 是否使用并行模式，其实就是一开始parallel()方法设置的属性
    @Override
    public final boolean isParallel() {
        return sourceStage.parallel;
    }
```

因此，重点在terminalOp.evaluateParallel，继续看看`TerminalOp`接口的`evaluateParallel`方法（遇到多态时，可以通过debug的形式）

`TerminalOp`接口的实现类有这样几个内部类：

* java.util.stream.FindOps.FindOp
* java.util.stream.ForEachOps.ForEachOp
* java.util.stream.MatchOps.MatchOp
* java.util.stream.ReduceOps.ReduceOp

可以看到，对应的是Stream的几种主要的计算操作。这里的示例代码使用的是reduce计算，就来看ReduceOp类

```java
@Override
public <P_IN> R evaluateParallel(PipelineHelper<T> helper,
                                 Spliterator<P_IN> spliterator) {
    return new ReduceTask<>(this, helper, spliterator).invoke().get();
}
```

evaluateParallel又创建了ReduceTask实例，学习过多线程后，想必对于这样的命名很敏感。

顾名思义，ReduceTask是一个task任务，因此很有可能就是利用forkjoin的思想，查看其继承架构体系，答案果然如此。

> ReduceTask -&gt; AbstractTask -&gt; CountedCompleter -&gt; ForkJoinTask

### 小结

Java8并行计算的原理，其实就是利用了ForkJoinPool，当调用parallel时就将其设置为并行式计算，当调用reduce则创建ForkJoinTask进行处理。

