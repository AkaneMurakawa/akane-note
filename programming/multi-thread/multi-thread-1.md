# 1 线程通信

线程通信：线程之间的消息通信，数据传输。

线程之间的几种通信方式

* 锁（synchronized）
* 等待/通知机制（wait、notify\)
* **信号量**（Semaphore）
* **管道**
* join、sleep、ThreadLocal

### 锁与同步

锁：保证了线程之间按照**一定的顺序**执行

### 等待/通知机制

Java多线程的等待/通知机制是基于Object类的wait\(\)方法和notify\(\), notifyAll\(\)方法来实现的。

```java
/**
 * 等待和唤起 示例
 *
 * 需要注意的是等待/通知机制使用的是使用同一个对象锁，如果你两个线程使用的是不同的对象锁，那它们之间是不能用等待/通知机制通信的。
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
                for (int i = 0; i < 5; i++) {
                    try{
                        System.out.println("Thread A:" + i);
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
                for (int i = 0; i < 5; i++) {
                    try{
                        System.out.println("Thread B:" + i);
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

### 信号量

JDK提供了一个类似于“信号量”功能的类Semaphore。我们也可以使用volatile关键字实现的信号量通信。

> 比如我现在有一个需求，我想让线程A输出0，然后线程B输出1，再然后线程A输出2…以此类推。我应该怎样实现呢？

```java
/**
 * 信号量 示例
 *
 */
public class SignalDemo {

    private static volatile int signal = 0;

    public static void main(String[] args) {
        new Thread(new ThreadA()).start();
        new Thread(new ThreadB()).start();
    }

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            while (signal < 5) {
                if (signal % 2 == 0) {
                    System.out.println("threadA: " + signal);
                    // 不保证原子性
                    signal++;
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            while (signal < 5) {
                if (signal % 2 == 1) {
                    System.out.println("threadB: " + signal);
                    signal = signal + 1;
                }
            }
        }
    }
}
```

### 管道

管道是基于“管道流”的通信方式。

JDK提供了PipedWriter、 PipedReader、 PipedOutputStream、 PipedInputStream。其中，前面两个是基于字符的，后面两个是基于字节流的。

```java
import java.io.IOException;
import java.io.PipedReader;
import java.io.PipedWriter;

/**
 * 管道 示例
 * <p>
 * 管道是基于“管道流”的通信方式。
 * JDK提供了PipedWriter、 PipedReader、 PipedOutputStream、 PipedInputStream。
 * 其中，前面两个是基于字符的，后面两个是基于字节流的。
 * </p>
 */
public class PipeDemo {

    public static void main(String[] args) throws IOException, InterruptedException{
        PipedWriter pipedWriter = new PipedWriter();
        PipedReader pipedReader = new PipedReader();
        // 连接，通信
        pipedWriter.connect(pipedReader);
        new Thread(new ReaderThread(pipedReader)).start();
        Thread.sleep(1000);
        new Thread(new WriterThread(pipedWriter)).start();

    }

    static class ReaderThread implements Runnable{
        private PipedReader pipedReader;

        public ReaderThread(PipedReader pipedReader) {
            this.pipedReader = pipedReader;
        }

        @Override
        public void run() {
            System.out.println("this is reader");
            int receive = 0;
            try {
                while ((receive = pipedReader.read()) != -1){
                    System.out.println((char)receive);
                }
            }catch (IOException e){
                e.printStackTrace();
            }
        }
    }

    static class WriterThread implements Runnable{
        private PipedWriter pipedWriter;

        public WriterThread(PipedWriter pipedWriter) {
            this.pipedWriter = pipedWriter;
        }

        @Override
        public void run() {
            System.out.println("this is writer");
            int receive = 0;
            try {
                pipedWriter.write("test");
            }catch (IOException e){
                e.printStackTrace();
            }finally {
                try{
                    pipedWriter.close();
                }catch (IOException e){
                    e.printStackTrace();
                }
            }
        }
    }
}
```

### 其他方式

join\(\)、sleep\(\)、ThreadLocal类

## 进程的通信方式

管道、内存共享、信号量、消息队列

