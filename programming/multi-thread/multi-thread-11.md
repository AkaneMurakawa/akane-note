# 11 Semaphore

Semaphore：信号的意思。信号量。允许多个线程同时访问（对比：ReentrantLock只允许一个）

![image-20201129233238100](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/多线程/images/image-20201129233238100.png)

用途：限流，控制并发数量

* semaphore.acquire\(\); 获取permit，获得锁，获取不到则阻塞
* semaphore.release\(\); // 释放permit

```java
/**
 * 
 * 而这个“信号”是一个int类型的数据，也可以看成是一种“资源”。
 */
public class SemaphoreDemo {

    static class MyThread implements Runnable {

        private int value;
        private Semaphore semaphore;

        public MyThread(int value, Semaphore semaphore) {
            this.value = value;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            try {
                semaphore.acquire(); // 获取permit，获得锁，获取不到则阻塞
                System.out.println(String.format("当前线程是%d, 还剩%d个资源，还有%d个线程在等待",
                        value, semaphore.availablePermits(), semaphore.getQueueLength()));
                // 睡眠随机时间，打乱释放顺序
                Random random =new Random();
                Thread.sleep(random.nextInt(1000));
                semaphore.release(); // 释放permit
                System.out.println(String.format("线程%d释放了资源", value));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        // permits允许多少个资源
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 10; i++) {
            new Thread(new MyThread(i, semaphore)).start();
        }
    }
}
```

结果：

```text
当前线程是1, 还剩1个资源，还有0个线程在等待
当前线程是0, 还剩2个资源，还有0个线程在等待
当前线程是2, 还剩0个资源，还有0个线程在等待
线程1释放了资源
当前线程是3, 还剩0个资源，还有6个线程在等待
当前线程是4, 还剩0个资源，还有5个线程在等待
线程2释放了资源
线程0释放了资源
当前线程是5, 还剩0个资源，还有4个线程在等待
当前线程是7, 还剩0个资源，还有2个线程在等待
当前线程是6, 还剩0个资源，还有2个线程在等待
线程4释放了资源
线程5释放了资源
线程7释放了资源
当前线程是8, 还剩0个资源，还有1个线程在等待
线程3释放了资源
当前线程是9, 还剩0个资源，还有0个线程在等待
线程6释放了资源
线程8释放了资源
线程9释放了资源
```

