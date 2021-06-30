# 0 基本概念

系统学习多线程知识

深入浅出Java多线程 [http://concurrent.redspider.group/](http://concurrent.redspider.group/)

廖雪峰老师Java教程 [https://www.liaoxuefeng.com/wiki/1252599548343744/1304521607217185](https://www.liaoxuefeng.com/wiki/1252599548343744/1304521607217185)

马士兵JUC教程 [https://github.com/bjmashibing/JUC](https://github.com/bjmashibing/JUC)

## JUC

俗称的Java并发编程，java.util.concurrent包的简称。

## 进程与线程的基本概念

### 进程

进程就是**应用程序在内存中分配的空间，也就是正在运行的程序**，各个进程之间互不干扰。同时进程保存着程序每一个时刻运行的状态。

CPU采用**时间片轮转**的方式运行进程：CPU为每个进程分配一个时间段，称作它的时间片。如果在时间片结束时进程还在运行，则暂停这个进程的运行，并且CPU分配给另一个进程（这个过程叫做**上下文切换**）。如果进程在时间片结束前阻塞或结束，则CPU立即进行切换，不用等待时间片用完。

使用进程+CPU时间片轮转方式的操作系统，在宏观上看起来同一时间段执行多个任务，换句话说，**进程让操作系统的并发成为了可能**。虽然并发从宏观上看有多个任务在执行，但在事实上，对于**单核CPU**来说，**任意具体时刻都只有一个任务在占用CPU资源。**

### 线程

那么能不能让这些子任务同时执行呢？于是人们又提出了线程的概念，**让一个线程执行一个子任务，这样一个进程就包含了多个线程，每个线程负责一个单独的子任务。**

### 进程和线程的区别

进程让操作系统的并发性成为了可能，而线程让进程的内部并发成为了可能。

进程是一个独立的运行环境，而线程是在进程中执行的一个任务。他们两个本质的区别是**是否单独占有内存地址空间及其它系统资源（比如I/O）**：

* 进程单独占有一定的内存地址空间，所以进程间存在**内存隔离**，数据是分开的，数据共享复杂但是同步简单，各个进程之间互不干扰；而**线程共享**所属进程占有的内存地址空间和资源，数据共享简单，但是同步复杂。
* 进程单独占有一定的内存地址空间，一个进程出现问题不会影响其他进程，不影响主程序的稳定性，可靠性高；一个线程崩溃可能影响整个程序的稳定性，可靠性较低。
* 进程单独占有一定的内存地址空间，进程的创建和销毁不仅需要保存寄存器和栈信息，还需要资源的分配回收以及页调度，开销较大；线程只需要保存寄存器和栈信息，开销较小。

另外一个重要区别是，**进程是操作系统进行资源分配的基本单位，而线程是操作系统进行调度的基本单位**，即CPU分配时间的单位 。

**多进程的方式也可以实现并发，为什么我们要使用多线程？**

多进程方式确实可以实现并发，但使用多线程，有以下几个好处：

* **进程间的通信比较复杂**，而线程间的通信比较简单，通常情况下，我们需要使用共享资源，这些资源在线程间的通信比较容易。
* 进程是**重量级**的，而线程是轻量级的，故多线程方式的**系统开销更小**。

### 上下文切换

上下文切换（有时也称做进程切换或任务切换）是指 CPU 从一个进程（或线程）切换到另一个进程（或线程）。上下文是指**某一时间点 CPU 寄存器和程序计数器的内容。**

寄存器是cpu内部的少量的速度很快的闪存，通常存储和访问计算过程的中间值提高计算机程序的运行速度。

程序计数器是一个专用的寄存器，用于表明指令序列中 CPU 正在执行的位置，存的值为正在执行的指令的位置或者下一个将要被执行的指令的位置，具体实现依赖于特定的系统。

举例说明 线程A - B

1.先挂起线程A，将其在cpu中的状态保存在内存中。

2.在内存中检索下一个线程B的上下文并将其在 CPU 的寄存器中恢复,执行B线程。

3.当B执行完，根据程序计数器中指向的位置恢复线程A。

CPU通过为每个线程分配CPU时间片来实现多线程机制。CPU通过时间片分配算法来循环执行任务，当前任务执行一个时间片后会切换到下一个任务。

但是，在切换前会保存上一个任务的状态，以便下次切换回这个任务时，可以再加载这个任务的状态。所以任务从保存到再加载的过程就是一次上下文切换。

上下文切换通常是计算密集型的，意味着此操作会**消耗大量的 CPU 时间，故线程也不是越多越好**。如何减少系统中上下文切换次数，是提升多线程性能的一个重点课题。

## 线程的创建

### 线程创建的三种方式

1. 继承Thread
2. 实现Runnable \(支持lambda表达式\)
3. 线程池，实现Callable\(带返回值 和 尝试取消\)

继承Thread

```java
class ThreadA extends Thread{
    @Override
    public void run() {

        System.out.println("Thread A");
    }
}

public static void main(String[] args) {
    Thread a = new ThreadA();
    a.start();
}
```

实现Runnable \(支持lambda表达式\)

```java
/**
 * Runnable 示例
 */
class ThreadB implements Runnable {

    @Override
    public void run() {
        System.out.println("Thread B");
    }
}

public class RunnableDemo  {

    public static void main(String[] args) {
        /**
         * public
         * class Thread implements Runnable {
         * }
         * 注：Runnable需要借助new Thread的构造方法实现创建线程
         */
        Thread b = new Thread(new ThreadB(), "Thread B");
        b.start();

        // Java8
        new Thread(() -> {
            System.out.println("Java8 Thread B");
        }).start();
    }
}
```

实现Callable

```java
/**
 * public interface Callable<V> {}
 * 可指定call()返回类型
 */
class ThreadC implements Callable {

    @Override
    public Object call() throws Exception {
        System.out.println("Thead C");
        return Thread.currentThread().getName();
    }
}

public static void main(String[] args) throws ExecutionException, InterruptedException {
    // 使用Executors创建线程池，但是阿里巴巴Java开发手册不建议使用
    // 原因是为了规避资源耗尽风险，也让开发者更了解线程池规则
    ExecutorService executorService = Executors.newCachedThreadPool();

    ThreadC c = new ThreadC();
    // 带返回结果的

    Future result = executorService.submit(c);
    // 注意调用get方法会阻塞当前线程，直到得到结果
    // 用途：阻塞，一直到线程结束，常用于线程池
    result.get();
}
```

### Thread类与Runnable接口的优劣比较

* Java是”单继承，多实现“，因此Runnable更轻量，更灵活，更符合面向对象编程。同时降低了线程对象和线程任务的耦合性
* 推荐使用Runnable

### Future

Future：未来的意思

我们可以使用Runnable和Thread来创建一个新的线程，但是问题是没有返回值。

因为在jDK1.5提供了Callable，那这个值怎么拿？就是通过Future去获取，而FutureTask是其实现类，FutureTask既是一个Future，也是一个Task

## 线程组和线程优先级

### 线程组

Java中用ThreadGroup来表示线程组，使用线程组可以对线程进行批量控制。如果在new Thread时没有显式指定，**那么默认将父线程（当前执行new Thread的线程）**线程组设置为自己的线程组。

### 线程优先级

范围是1~10，默认是5，10最大。

**Java程序中对线程所设置的优先级只是给操作系统一个建议，操作系统不一定会采纳。而真正的调用顺序，是由操作系统的线程调度算法决定的**。

### 守护线程

如果某线程是守护线程，那如果所有的非守护线程结束，这个守护线程也会自动结束。

## 线程中的各种方法解释

* sleep\(\)
* yield\(\)
* join

### sleep\(\)

睡眠，回到等待队列，**就绪状态**。

sleep\(long\)⽅法仅释放CPU使⽤权，锁仍然占⽤；线程被放⼊超时等待队列，与yield相⽐，它会使线程较⻓时间得不到运⾏。

### yield\(\)

放弃，回到等待队列，**就绪状态**。

yield\(\)⽅法仅释放CPU执⾏权，**锁仍然占⽤**，线程会被放⼊就绪队列，会在短时间内再次执⾏。

注：使用场景很少，基本不使用

### join\(\)

**等待join的这个线程执行完成后，再继续执行当前线程。**

通常用来等待另外一个线程执行，例如：保证线程顺序执行。

示例：

下面程序中，ThreadA线程进行了sleep，如果不调用`thread.join();`那么就会先执行main程序。

通过join方法，main程序就会等待TheadA线程执行完毕后再继续执行。

```java
/**
 * join() 示例
 *
 * 底层调用的还是wait()
 */
public class JoinDemo {

    public static void main(String[] args) throws InterruptedException{
        Thread thread = new Thread(new ThreadA());
        thread.start();
        // 如果无，会先打印----等待线程 thread执行完毕，再执行当前线程 这句话
        thread.join();
        System.out.println("等待线程 thread执行完毕，再执行当前线程 这句话");
    }

    static class ThreadA implements Runnable{
        @Override
        public void run() {
            try{
                System.out.println("我是子线程，我先睡一秒");
                Thread.sleep(1000L);
                System.out.println("我是子线程，我睡完了一秒");
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
    }
}
```

### interrupt\(\)

**线程中断**。目前在Java里还没有安全直接的方法来停止线程，但是Java提供了线程中断机制来处理需要中断线程的情况。

使用的场景很少，很少用来控制业务逻辑。

如何正常关闭线程？

不要去用stop\(\)，让线程正常结束，也就是run\(\)方法正常结束即可。使用线程中断可能会导致状态不一致。

### 线程中的各种方法使用示例

```java
/**
 * Thread一些方法 示例
 */
public class ThreadMethodDemo {

    public static void main(String[] args) throws InterruptedException{
        // 返回对当前正在执行的线程对象的引用；native方法
        Thread currentThread =
                Thread.currentThread();

                // 启动，加了synchronized
                currentThread.start();
                // 放弃，让出一段时间CPU
                currentThread.yield();
                // 睡眠，使当前线程睡眠一段时间，会释放CPU资源，但不释放锁
                Thread.sleep(1);
                // 等待加入，当前线程等待另一个线程执行完毕之后再继续执行
                currentThread.join();

                // 设置优先级
                currentThread.setPriority(2);
                // 设置为守护线程
                currentThread.setDaemon(true);
                // 是否中断
                currentThread.isInterrupted();
                // 尝试中断
                currentThread.interrupt();

    }
}
```

Object.wait / notify / notifyAll

wait：native方法，会释放CPU资源，释放锁。必须在synchronized块中才能调用wait\(\)方法

notify：建议是调用：notifyAll\(\)而不是notify\(\)，因为notify\(\)只会唤醒其中一个，结果随机。

```java
/**
 * 两个线程交叉打印，一个打印数字，一个打印大写字母，结果为A1B2...Y25Z26
 */
public class WaitAndNotifyDemo {

    private static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException{
        new Thread(new ThreadA()).start();
        Thread.sleep(100);
        new Thread(new ThreadB()).start();
    }

    static class ThreadA implements Runnable{
        @Override
        public void run() {
            synchronized (lock){
                for (int i = 1; i <= 26; i++) {
                    try{
                        System.out.print((char)(i+64));
                        // notifyAll()方法叫醒另一个正在等待的线程
                        lock.notifyAll();
                        // 使用wait()方法陷入等待并释放lock锁
                        lock.wait();
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
                lock.notifyAll();
            }
        }
    }

    static class ThreadB implements Runnable{
        @Override
        public void run() {
            synchronized (lock){
                for (int i = 1; i <= 26; i++) {
                    try{
                        System.out.print(i);
                        lock.notifyAll();
                        lock.wait();
                    }catch (InterruptedException e){
                        e.printStackTrace();
                    }
                }
                lock.notifyAll();
            }
        }
    }
}
```

**获取当前方法的名称**

```text
Thread.currentThread().getStackTrace()[1].getMethodName()
```

### sleep\(\) 和wait\(\)对比

* wait可以指定时间，也可以不指定；而sleep必须指定时间。
* wait释放cpu资源，**同时释放锁**；sleep释放cpu资源，但是不释放锁，所以易死锁。
* wait必须放在synchronized中，而sleep可以在任意位置

## 线程中的各种状态解释

* NEW： 新建状态，还未调用start\(\)
* RUNNABLE：正在运行状态，正在运行run\(\)，RUNNABLE包括了传统操作系统线程的ready（**就绪状态**）和running（运行状态）两个状态的
* BLOCKED：阻塞状态，运行中的线程，因为某些操作被阻塞而挂起。例如：加了synchronized，没得到锁的时候会进行阻塞。
* WAITING：等待状态，运行中的线程，因为某些操作在等待中
* TIMED\_WAITING：超时等待状态，运行中的线程，因为执行sleep\(\)方法正在计时等待，与WAITING相比多了时长设置。
* TERMINATED：线程已终止，因为run\(\)方法执行完毕，完了就结束了

> 例如：今天你下班后准备去食堂吃饭。你来到食堂**仅有的一个窗口**，发现前面已经有个人在窗口前了，此时你必须得等前面的人从窗口离开才行。
>
> 假设你是线程t2，你前面的那个人是线程t1。此时t1**占有了锁**（食堂唯一的窗口），t2正在等待锁的释放，所以此时t2就处于BLOCKED状态。

注：当没获取到锁，等待锁时就处于BLOCKED，阻塞状态。

> 你等了好几分钟现在终于轮到你了，突然你们有一个“不懂事”的经理突然来了。你看到他你就有一种不祥的预感，果然，他是来找你的。
>
> 他把你拉到一旁叫你待会儿再吃饭，说他下午要去作报告，赶紧来找你了解一下项目的情况。你心里虽然有一万个不愿意但是你还是从食堂窗口走开了。
>
> 此时，假设你还是线程t2，你的经理是线程t1。**虽然你此时都占有锁**（窗口）了，“不速之客”来了你还是得**释放掉锁**。此时你t2的状态就是WAITING。
>
> 然后经理t1获得锁，进入RUNNABLE状态。
>
> 要是经理t1不主动唤醒你t2（notify、notifyAll..），可以说你t2只能一直等待了。

注：当调用`Object.wait()、Thread.join()、LockSupport.park()`时，就必须得释放锁，进行等待。此时**需要主动唤醒**，不然会一直等待。

> 到了第二天中午，又到了饭点，你还是到了窗口前。
>
> 突然间想起你的同事叫你等他一起，他说让你等他十分钟他改个bug。
>
> 好吧，你说那你就等等吧，你就离开了窗口。
>
> 很**快十分钟过去了，你见他还没来，你想都等了这么久了还不来，那你还是先去吃饭好了。**
>
> 这时你还是线程t1，你改bug的同事是线程t2。t2让t1等待了指定时间，此时t1等待期间就属于TIMED\_WATING状态。
>
> t1等待10分钟后，就自动唤醒，拥有了去争夺锁的资格。

注：与WAITING相比，多了等待时长设置，到了一定时间会**自动唤醒**。也可以手动唤醒。

### 线程状态转换图

![image.png](../../.gitbook/assets/java5.png)

![image.png](../../.gitbook/assets/java6.png)

