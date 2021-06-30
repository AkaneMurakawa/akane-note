# 7 AQS（CLH\)

## JUC同步锁

下面列举的**同步锁**的实现，**原理其实都是基于AQS实现**

* ReentrantLock
* CountDownLatch
* CyclicBarrier
* Phaser
* ReadWriteLock - StampedLock
* Semaphore
* Exchanger

## AQS

AQS：AbstractQueuedSynchronizer，即抽象队列同步器

从字面意思上理解:

* 抽象：抽象类，只实现一些主要逻辑，有些方法由子类实现；（模板方法设计模式）
* 队列：使用先进先出（FIFO）队列存储数据
* 同步：实现了同步的功能。

## AQS作用

那AQS有什么用呢？AQS是一个用来构建**锁和同步器**的框架。

比如ReentrantLock，Semaphore，ReentrantReadWriteLock，SynchronousQueue，FutureTask等等都是基于AQS的。

当然，我们自己也能利用AQS非常轻松容易地构造出符合我们自己需求的同步器，只要实现它的几个protected方法就可以了。

## AQS实现原理

AQS核⼼思想：

如果被请求的共享资源**空闲**，则将当前请求资源的线程设置为有效的⼯作线程。并且将共享资源设置为锁定状态。

如果被请求的共享资源被**占⽤**，那么就需要⼀套**线程阻塞等待 以及 被唤醒时锁分配**的机制，AQS使用的CLH队列。

### CLH队列

CLH：Craig，Landin and Hagersten 三人的简称

⼀个虚拟的双向队列，即将暂时获取不到锁的线程加⼊到队列中。

### AQS的数据结构-CLH队列

AQS内部使用了一个先进先出（FIFO）的双端队列（双向链表），并使用了两个指针head和tail用于标识队列的头部和尾部。

为什么需要双向链表而不是单向？答：因为需要获取前一个的状态

资源：state ，获取到锁

![image.png](../../.gitbook/assets/java36.png)

### 资源共享模式

资源有两种共享模式，或者说两种同步方式：

* 独占模式（Exclusive）：排他锁。资源是独占的，一次只能一个线程获取。如ReentrantLock。
* 共享模式（Share）：共享锁。同时可以被多个线程获取，具体的资源个数可以通过参数指定。如Semaphore/CountDownLatch。

一般情况下，子类只需要根据需求实现其中一种模式，当然也有同时实现两种模式的同步类，如ReadWriteLock。

## AQS源码分析

采用的是模板方法设计模式

![image.png](../../.gitbook/assets/java37.png)

### AQS-Lock分析

[https://www.cnblogs.com/waterystone/p/4920797.html](https://www.cnblogs.com/waterystone/p/4920797.html)

从ReentrantLock开始，我们可以看到lock上面的注释：

1. 去获取锁，如果没有其他线程占用锁则直接返回，获取到锁则将当前线程设置为锁持有。
2. 如果当前线程已经持有锁，就将计数（state）+1返回，可重入锁。
3. 如果获取不到锁则进如等待直到获取到锁，这时锁的计数保持为1。

```java
private final Lock lock = new ReentrantLock();
lock.lock();

    /**
     * Acquires the lock.
     *
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.lock();
    }
```

#### Sync

接着我们看一下这个sync是什么

注释说明：Sync是基于AQS（AbstractQueuedSynchronizer）实现的同步锁，子类实现了公平锁和非公平锁版本，使用state\(这里翻译为：资源\)表示持有锁的数量。

```java
/**
 * Base of synchronization control for this lock. Subclassed
 * into fair and nonfair versions below. Uses AQS state to
 * represent the number of holds on the lock.
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
```

接下来，我们就来看看公平锁版本

acquire：这里是排他锁模式（exclusive mode），至少要进行一次tryAcquire尝试获取锁。如果获取失败，则线程进行排队，可能会重复阻塞和非阻塞直到获取锁成功。

arg：acquires获取资源的个数，这个值源码里定义为：未解释（uninterpreted）。意味着你喜欢怎么定义就怎么定义

```java
/**
 * Sync object for fair locks
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

// ---------------------------------------------------------------------------------------------------------    
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
        // &&短路与，尝试获取锁，获取不到则进入等待
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

#### tryAcquire

尝试获取锁

```java
    /**
     * The synchronization state.
     * volatile保证线程内存可见性，禁止指令重排
     */
    private volatile int state;

/**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        // 获取state状态，0表示未被占用
        int c = getState();
        if (c == 0) {
            // hasQueuedPredecessors是否存在其他同时在竞争，用于公平锁情况处理
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) { // cas
                setExclusiveOwnerThread(current); // 设置独占模式下的锁占用者为当前线程
                return true;
            }
        }
        // 如果当前锁持有者就是当前线程，state + acquires,实现可重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

#### addWaiter

为当前线程创建等待队列，或入队。

实现原理：使用CAS自旋锁将当前线程添加到tail

mode：EXCLUSIVE排他独占模式，SHARED共享模式

```java
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        // node的前节点指向tail
        node.prev = pred;
        // 设置tail为node成功，cas
        if (compareAndSetTail(pred, node)) {
            // tail的下一个节点指向node
            pred.next = node;
            return node;
        }
    }
    // 尾节点tail为null，说明还未创建等待队列，进行创建
    // 和上面的插入方式相同，只不过多了初始化队列
    enq(node);
    return node;
}

//  ----------------------------------------------------------------------------------------------------------------
    /**
     * Inserts node into queue, initializing if necessary. See picture above.
     * @param node the node to insert
     * @return node's predecessor
     */

    private Node enq(final Node node) {
        // 死循环
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 初始化队列，将head设置Node，CAS操作
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // node的前节点指向tail
                node.prev = t;
                // 设置tail为node成功
                if (compareAndSetTail(t, node)) {
                    // tail的下一个节点指向node
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

这里需要注意的是：如果队列已经创建，则会入队，但是我们发现addWaiter入队时并没有用for \(;;\) ，那这里会有个问题，如果并发入队失败怎么办？

如果`compareAndSetTail(pred, node)`这里失败，那当然就走到enq\(\)方法了，这里就使用了for \(;;\) 保证最后一定会入队成功。因此可以看到enq\(\)里初始化队列和入队分了两个分支进行，当t=null则初始化队列，否则入队操作。

#### 初始化队列

```java
/** 
 * 步骤一：head = new Node()
 * 步骤二：tail = head;
 * /
```

```java
Node t = tail;
if (t == null) { // Must initialize
    // 初始化队列，将head设置Node，CAS操作
    if (compareAndSetHead(new Node()))
    tail = head;
}
```

#### 入队（先断后连）

```java
// 步骤一：pred临时节点，存储tail
// 步骤二：设置node节点为tail（断）
// 步骤三：pred(old tail)的next指向node（连）
```

```java
Node pred = tail;
    if (pred != null) {
        // node的前节点指向tail
        node.prev = pred;
        // 设置tail为node成功，cas
        if (compareAndSetTail(pred, node)) {
            // tail的下一个节点指向node
            pred.next = node;
            return node;
        }
    }
```

#### acquireQueued

当进入等待队列后，便需要进行等待（park）

```java
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
        for (;;) {
            // 获取前一个节点
            final Node p = node.predecessor();
            // 如果前一个节点是head节点，再次尝试获取锁，因为可能前一个节点已经释放锁
            if (p == head && tryAcquire(arg)) {
                // 获取锁成功，则将head节点设置为当前节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取锁失败后是否需要park(阻塞等待)
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // parkAndCheckInterrupt：进行park线程
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### shouldParkAfterFailedAcquire

```java
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block. This is the main signal
 * control in all acquire loops.  Requires that pred == node.prev.
 *
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前一个节点的等待状态
    int ws = pred.waitStatus;
    // 前一个是SIGNAL状态，说明当前node可以park掉，然后由前一个节点负责唤醒unpark
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;

    // 如果前一个节点取消了排队，则将pred的前一个节点作为node的前一个节点
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } 
    // 如果都不是，只能是0或-3、-2了，此时需要等待一个【车队】，则会将前一个节点设置为SIGNAL
    else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

#### waitStatus

表示线程等待的状态

CANCELLED：1 已取消

SIGNAL：-1 唤醒，需要唤醒下一个节点（unpark）

CONDITION：-2 等待一个条件

PROPAGATE：-3 传播，共享模式

0：以上都不是，默认值

```java
/** waitStatus value to indicate thread has cancelled */
static final int CANCELLED =  1;
/** waitStatus value to indicate successor's thread needs unparking */
static final int SIGNAL    = -1;
/** waitStatus value to indicate thread is waiting on condition */
static final int CONDITION = -2;
/**
 * waitStatus value to indicate the next acquireShared should
 * unconditionally propagate
 */
static final int PROPAGATE = -3;

/**
 * Status field, taking on only the values:
 *   SIGNAL:     The successor of this node is (or will soon be)
 *               blocked (via park), so the current node must
 *               unpark its successor when it releases or
 *               cancels. To avoid races, acquire methods must
 *               first indicate they need a signal,
 *               then retry the atomic acquire, and then,
 *               on failure, block.
 *   CANCELLED:  This node is cancelled due to timeout or interrupt.
 *               Nodes never leave this state. In particular,
 *               a thread with cancelled node never again blocks.
 *   CONDITION:  This node is currently on a condition queue.
 *               It will not be used as a sync queue node
 *               until transferred, at which time the status
 *               will be set to 0. (Use of this value here has
 *               nothing to do with the other uses of the
 *               field, but simplifies mechanics.)
 *   PROPAGATE:  A releaseShared should be propagated to other
 *               nodes. This is set (for head node only) in
 *               doReleaseShared to ensure propagation
 *               continues, even if other operations have
 *               since intervened.
 *   0:          None of the above
 *
 * The values are arranged numerically to simplify use.
 * Non-negative values mean that a node doesn't need to
 * signal. So, most code doesn't need to check for particular
 * values, just for sign.
 *
 * The field is initialized to 0 for normal sync nodes, and
 * CONDITION for condition nodes.  It is modified using CAS
 * (or when possible, unconditional volatile writes).
 */
volatile int waitStatus;
```

### AQS - unlock分析

```java
/**
 * Attempts to release this lock.
 *
 * <p>If the current thread is the holder of this lock then the hold
 * count is decremented.  If the hold count is now zero then the lock
 * is released.  If the current thread is not the holder of this
 * lock then {@link IllegalMonitorStateException} is thrown.
 *
 * @throws IllegalMonitorStateException if the current thread does not
 *         hold this lock
 */
public void unlock() {
    sync.release(1);
}
```

#### release

```java
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
    // 尝试释放
    if (tryRelease(arg)) {
        Node h = head;
        // 如果head不为空且head的等待状态非0，说明需要唤醒下一个节点，waitStatus解释见前面
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

#### tryRelease

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        // 释放锁
        setExclusiveOwnerThread(null);
    }
    // 更新state
    setState(c);
    return free;
}
```

#### unparkSuccessor

waitStatus解释见前面

```java
/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    // 如果状态是负数，尝试把它设置为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    // 下个一个节点为空或已取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 等待队列中所有还有用的结点，都向前移动
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒下一个节点
        LockSupport.unpark(s.thread);
}
```

## 总结

AQS是一个抽象队列同步器，当不存在锁竞争的时候，直接获取锁。

如果存在锁竞争，则会初始化一个等待队列，一种双向链表队列，简称为CLH队列。在tail入队时，使用的是无锁操作，即CAS自旋锁。这样只要锁住tail即可，而不用锁住整个链表。

当入队成功后，需判断前置节点（这也是为什么使用双向链表的原理）的状态（waitStatus），如果前一个是头节点，会再次尝试获取锁，因为可能前一个执行完了。如果未获取到锁，然后设置当前线程是否应该阻塞（shouldParkAfterFailedAcquire）。

当释放锁时，需要唤醒下一个节点。

