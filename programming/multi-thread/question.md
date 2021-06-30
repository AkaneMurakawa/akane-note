# 多线程 - 面试题

## 问题：实现一个同步容器

写一个固定容量同步容器，拥有put和get方法，以及getCount方法，能够支持2个生产者线程以及10个消费者线程阻塞调用

### 解决一

```java
public class ContainerDemo<T> {

    final private LinkedList<T> lists = new LinkedList<>();
    final private int MAX = 10;
    private int count = 0;

    private Lock lock = new ReentrantLock();
    private Condition producer = lock.newCondition();
    private Condition consumer = lock.newCondition();

    public void put(T t){
        try {
            lock.lock();
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
        try {
            lock.lock();
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

## 问题：实现一个容器

实现一个容器，提供两个方法，add，size。

写两个线程，线程1添加10个元素到容器中，线程2实现监控元素的个数，当个数到5个时，线程2给出提示并结束。

### 解决一

```java
 /**
 * wait / notify
 * @param <V>
 */
public class Collection1Demo<V> {

    volatile List<V> list = new ArrayList();

    public void add(V v){
        list.add(v);
    }

    public int size(){
        return list.size();
    }

    public static void main(String[] args) {
        Collection1Demo<Integer> collection = new Collection1Demo<>();
        final Object lock = new Object();

        Thread t1 = new Thread(() -> {
            System.out.println("t1开始");
            synchronized (lock){
                for (int i = 0; i < 10; i++) {
                    collection.add(i+1);
                    System.out.println("add " + (i+1));
                    if (collection.size() == 5){
                        lock.notify(); // notify不会释放锁
                        try {
                            lock.wait(); // wait会释放锁
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
//                    try {
//                        TimeUnit.SECONDS.sleep(1);
//                    } catch (InterruptedException e) {
//                        e.printStackTrace();
//                    }
                }
            }

        });


        Thread t2 = new Thread(() -> {
            System.out.println("t2开始");
            synchronized (lock){
                if(collection.size() != 5){
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("t2结束");
                lock.notify();
            }
        });

        t1.start();
        t2.start();

    }
}
```

### 解决二

```java
/**
 * CountDownLatch
 * @param <V>
 */
public class Collection2Demo<V> {

    volatile List<V> list = new ArrayList();

    public void add(V v){
        list.add(v);
    }

    public int size(){
        return list.size();
    }

    public static void main(String[] args) {
        Collection2Demo<Integer> collection = new Collection2Demo<>();

        CountDownLatch countDownLatch1 = new CountDownLatch(1);
        CountDownLatch countDownLatch2 = new CountDownLatch(1);

        Thread t1 = new Thread(() -> {
            System.out.println("t1开始");
            for (int i = 0; i < 10; i++) {
                collection.add(i+1);
                System.out.println("add " + (i+1));
                if (collection.size() == 5) {
                    countDownLatch2.countDown();
                    try {
                        countDownLatch1.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }

        });


        Thread t2 = new Thread(() -> {
            System.out.println("t2开始");
            if (collection.size() != 5) {
                try {
                    countDownLatch2.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            countDownLatch1.countDown();

            System.out.println("t2结束");
        });

        t1.start();
        t2.start();

    }
}
```

### 解决三

```java
/**
 * LockSupport
 * @param <V>
 */
public class Collection3Demo<V> {

    volatile List<V> list = new ArrayList();

    public void add(V v){
        list.add(v);
    }

    public int size(){
        return list.size();
    }

    static Thread t1 , t2 = null;

    public static void main(String[] args) {
        Collection3Demo<Integer> collection = new Collection3Demo<>();

        t1 = new Thread(() -> {
            System.out.println("t1开始");
            for (int i = 0; i < 10; i++) {
                collection.add(i+1);
                System.out.println("add " + (i+1));
                if (collection.size() == 5) {
                    LockSupport.unpark(t2);
                    LockSupport.park();
                }
            }

        });


        t2 = new Thread(() -> {
            System.out.println("t2开始");
            if (collection.size() != 5) {
                LockSupport.park();
            }
            System.out.println("t2结束");
            LockSupport.unpark(t1);

        });

        t1.start();
        t2.start();

    }
}
```

## 问题：交叉打印

两个线程交叉打印，一个打印数字，一个打印大写字母，结果为A1B2...Y25Z26

### 解决一

```java
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

### 其他解决

* lcok Condition
* cas
* LockSupport
* BlockingQueue
* PipedStream
* Exchanger
* TransferQueue

## 问题：死锁示例

### 解决一

```java
/**
 * 死锁示例
 */
public class DeadLockDemo {

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            method1();
            method2();
        });

        Thread t2 = new Thread(() -> {
            method1();
            method2();
        });



        // 如果 method1() 和 method2() 都由两个或多个线程调⽤,则存在死锁的可能性
        t1.start();
        t2.start();

        // 解决方式：method1和method2使用相同的锁顺序，例如method1和method3不会死锁
        //Thread t3 = new Thread(() -> {
        //    method1();
        //    method3();
        //});
        //t3.start();
    }


    public static void method1(){
        synchronized (Integer.class){
            System.out.println("lock on Integer class");
            synchronized (String.class){
                System.out.println("lock on String class");
            }
        }
    }

    public static void method2(){
        synchronized (String.class){
            System.out.println("lock on String class");
            synchronized (Integer.class){
                System.out.println("lock on Integer class");

            }
        }
    }

    public static void method3(){
        synchronized (Integer.class){
            System.out.println("lock on Integer class");
            synchronized (String.class){
                System.out.println("lock on String class");
            }
        }
    }
}
```

## Thread.sleep\(0\)的作用是什么

由于Java采用抢占式的线程调度算法，因此可能会出现某条线程常常获取到CPU控制权的情况，为了让某些优先级比较低的线程也能获取到CPU控制权，可以使用Thread.sleep\(0\)手动触发一次操作系统分配时间片的操作，这也是平衡CPU控制权的一种操作。

## 分布式并发、Redis竞争Key问题

解决：使用分布式锁

