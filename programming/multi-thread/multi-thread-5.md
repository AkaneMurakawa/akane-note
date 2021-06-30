# 5 ReentrantLock

## ReentrantLock可重入锁

ReentrantLock：重入锁的意思

ReentrantLock和synchronized一样，是可重入锁，但是却提供了更多特性，用于代替synchronized。

注：有些喜欢将ReentrantLock简称为lock，比如问你lock和synchronized的区别

## 示例

### 示例1：手动释放锁

说明一点，ReentranLock必须使用try-finally手动释放锁，而synchronized不需要，synchronized遇到异常也会释放锁。

```java
/**
 * ReentrantLock 示例
 */
public class ReentrantLockDemo {

    private final Lock lock = new ReentrantLock();

    public String getTask() throws InterruptedException{

        try{
            lock.lock();
        }finally {
            lock.unlock();
        }
    }
}
```

### 示例2：尝试获取锁

ReentranLock还可以尝试获取锁，你可以选择wait还是不wait。另外可以设置等待的时长。

```text
public boolean tryLock(long timeout, TimeUnit unit)
```

```java
/**
 * ReentrantLock 示例
 */
public class ReentrantLockDemo {

    private final Lock lock = new ReentrantLock();

    public String getTask() throws InterruptedException{
        boolean locked = false;
        try{
            locked = lock.tryLock();
            if (locked){

            }
        }finally {
            if (locked){
                lock.unlock();
            }
        }
    }
}
```

### 示例3：提供中断等待锁的线程机制

ReentrantLock可以通过lock.lockInterruptibly\(\)，实现中断等待锁的线程机制。

注：如果是用lock\(\)获取不到锁则会一直阻塞，也不能中断线程。而如果是lockInterruptibly，则可以通过Thread.interrupt\(\)中断等待锁的线程。

```java
    public static void main(String[] args) throws InterruptedException{
        final Lock lock = new ReentrantLock();
        // 获取锁
        lock.lock();
        Thread.sleep(2000);
        Thread t1 = new Thread( () -> {
//            lock.lock();

            try{
               lock.lockInterruptibly();
               System.out.println("获取锁");
            }catch (InterruptedException e){
               System.out.println("interrupted");
            }

        });
        t1.start();
        Thread.sleep(1000);
        t1.interrupt();
    }
```

结果：interrupted

### 示例4：可指定公平锁和非公平锁

ReentrantLock可以指定为公平锁和非公平锁，⽽synchronized只能是⾮公平锁。ReentrantLock默认情况是⾮公平锁。

公平锁：先等待的线程先获得锁，本质上是有一个等待队列

```java
private final Lock lock = new ReentrantLock(true);


    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     * 默认非公平锁
     */ 
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) { 
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### 示例5：等待通知机制

* ReentrantLock同样提供了等待/通知机制（锁可以绑定**多个条件，多个Condition**），更灵活。
* 而synchronized只能有**一个**对象（synchronized一个对象，wait和notify）。

```java
/**
 * 写一个固定容量同步容器，拥有put和get方法，以及getCount方法，能够支持2个生产者线程以及10个消费者线程阻塞调用
 * @param <T>
 */
public class ContainerDemo<T> {

    final private LinkedList<T> lists = new LinkedList<>();
    final private int MAX = 10;
    private int count = 0;

    private Lock lock = new ReentrantLock();
    private Condition producer = lock.newCondition();
    private Condition consumer = lock.newCondition();

    public void put(T t){
        try {lock.lock();
            while (lists.size() == MAX){
                producer.await(); // 阻塞生产者
            }
            lists.add(t);
            ++count;
            consumer.signalAll(); // 通知消费者进行消费
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public T get(){
        T t = null;
        try {lock.lock();
            while (lists.size() == 0){
                consumer.await(); // 阻塞消费者者
            }
            t = lists.removeFirst();
            count--;
            producer.signalAll(); // 通知生产者者进行生产
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
        return t;
    }

    public static void main(String[] args) {
        ContainerDemo<String> c = new ContainerDemo<>();
        // consumer
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                for (int j = 0; j < 5; j++) {
                    System.out.println(c.get());
                }
            }).start();
        }

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                for (int j = 0; j < 25; j++) {
                    c.put(Thread.currentThread().getName() +" " + j);
                }
            }, "p"+i).start();
        }
    }
}
```

## ReentrantLock实现原理

本质是CAS，底层调用的是Unsafe的park方法加锁

```java
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 获得锁
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
// --------------------------------------------------------------------------------------------------------------------
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
// --------------------------------------------------------------------------------------------------------------------
// 等待队列
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 死循环
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 失败后是否需要park
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
// --------------------------------------------------------------------------------------------------------------------
    /**
     * Convenience method to park and then check if interrupted
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
// --------------------------------------------------------------------------------------------------------------------
//  LockSupport.park(this);最终调用Unsafe的park()
//  在CAS中也有介绍过Unsafe，而park与unpark用于线程挂起和恢复
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
```

总的来说，ReentrantLock是基于CAS的，不同的是多了线程的挂起和恢复，而这个操作是通过Unsafe的park和unpark实现的。

## ReentrantLock和synchronized区别

1. 和synchronized一样，ReentrantLock是支持可重入锁。**本质上ReentrantLock是CAS自旋锁，synchronized是锁对象头，锁升级的概念。**
2. 和synchronized不同的是，ReentrantLock可以尝试获取锁，并且可以设置获取锁等待的时长。ReentrantLock需手动释放锁。
3. ReentrantLock提供了一种**中断等待锁的线程**机制。
4. ReentrantLock可以指定为公平锁和非公平锁，⽽synchronized只能是⾮公平锁。
5. ReentrantLock同样提供了等待/通知机制（锁可以绑定**多个条件，多个Condition**），更灵活。而synchronized只能有**一个**对象（synchronized一个对象\)。

