# 6 ReadWriteLock与StampedLock

## ReadWriteLock

ReadWriteLock其实是一个**共享锁和排他锁的概念，读锁共享，写锁独占。**

ReadWriteLock可以解决多线程同时读，但只有一个线程能写的问题。

用途：读多写少

读：多个线程可以同时获取，不互斥

写：排他锁，只能有一个线程写

例如，一个论坛的帖子，回复可以看做写入操作，它是不频繁的，但是，浏览可以看做读取操作，是非常频繁的，这种情况就可以使用ReadWriteLock。

```java
public class ReentrantReadWriteLockDemo {

    private static final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private static final Lock readLock = rwLock.readLock();
    private static final Lock writeLock = rwLock.writeLock();
    private static int value ;

    /**
     * 写
     */
    public static void read(Lock lock){
        try {
            lock.lock();
            Thread.sleep(1000);
            System.out.println("read over!" + value);
        }catch (InterruptedException e){
        } finally {
            lock.unlock();
        }

    }

    /**
     * 读
     */
    public static void write(Lock lock, int v){
        try{
            lock.lock();
            Thread.sleep(1000);
            value = v;
            System.out.println("write over!");
        }catch (InterruptedException e){
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        // 共享锁
        Runnable read = () -> read(readLock);
        // 排他锁
        Runnable write = () -> write(writeLock, new Random().nextInt(100));

        for (int i = 0; i < 10; i++) {
            new Thread(read).start();
        }

        for (int i = 0; i < 2; i++) {
            new Thread(write).start();
        }
    }
}
```

结果：随机

从结果可以看出，读不堵塞，共享锁。但是写排斥，排他锁

```text
read over!0
read over!0
read over!0
read over!0
read over!0
read over!0
read over!0
write over!
read over!85
read over!85
read over!85
write over!
```

## StampedLock

[https://www.cnblogs.com/konck/p/9691538.html](https://www.cnblogs.com/konck/p/9691538.html)

ReadWriteLock可以解决多线程同时读，但只有一个线程能写的问题。

但是有一个问题：如果有线程正在读，写线程**需要等待读**线程释放锁后才能获取写锁，即读的过程中不允许写，这是一种悲观的读锁。

于是java8提供了一种新的读写锁：StampedLock，号称读写锁中的性能之王。

Stamped时间戳的意思，利用的就是乐观锁。在CAS中提到的ABA问题，JDK1.5提供了类**AtomicStampedReference**（stamped：时间戳）来解决 ABA问题。

`StampedLock`提供了乐观读锁，可取代`ReadWriteLock`以进一步提升并发性能；

改进之处在于：读的过程中也允许获取写锁后写入！这样一来，读的数据就可能不一致，所以，需要额外的判断读的过程中是否有写入，这种读锁是一种乐观锁。

StampedLock把读锁细分为乐观读和悲观读，读的时候先尝试乐观读，如果失败，则升级为悲观读。

注意：StampedLock是不可重入锁

用途： 更高效同时可以避免写饥饿的读写锁，但是代码变得更加复杂。

* stampedLock.writeLock\(\);
* stampedLock.unlockWrite\(stamp\);
* stampedLock.tryOptimisticRead\(\);
* stampedLock.readLock\(\);
* stampedLock.unlockWrite\(stamp\);

```java
public class StampedLockDemo {

    private final StampedLock stampedLock = new StampedLock();

    private double x;
    private double y;

    public void move(double deltaX, double deltaY){
        // 获取写锁
        long stamp = stampedLock.writeLock();
        try{
            x += deltaX;
            y += deltaY;
        }finally {
            // 释放写锁
            stampedLock.unlockWrite(stamp);
        }
    }


    public double distanceFromOrigin(){
        // 获取一个乐观读锁
        long stamp = stampedLock.tryOptimisticRead();

        // 这两行代码不具有原子性
        double currentX = x;
        double currentY = y;

        // 检查乐观读锁后是否有其他写锁发生
        if (!stampedLock.validate(stamp)){
            // 获取一个悲观读锁
            stamp = stampedLock.readLock();
            try{
                currentX = x;
                currentY = y;
            }finally {
                // 释放悲观读锁
                stampedLock.unlockWrite(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

## ReadWriteLock与StampedLock对比

* `StampedLock`和`ReadWriteLock`相比，**写入**的加锁是完全一样的，不同的是读取。
* `StampedLock`通过`tryOptimisticRead()`获取一个乐观读锁，并返回版本号。接着进行读取，读取完成后，通过`validate()`去验证版本号，如果在读取过程中没有写入，版本号不变，验证成功，就可以放心地继续后续操作。如果在读取过程中有写入，版本号会发生变化，验证将失败。在失败的时候，再通过获取悲观读锁再次读取。
* 由于写入的概率不高，程序在绝大部分情况下可以通过乐观读锁获取数据，极少数情况下使用悲观读锁获取数据。

