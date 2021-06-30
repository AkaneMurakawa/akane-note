# 8 CountDownLatch

CountDown代表计数递减，Latch是“门闩”（mén shuān ）的意思。也有人把它称为“屏障”。

顾名思义，一个门闩卡住，然后不停的递减，当减少到0的时候才继续执行。理解为：**倒计时器**

![image-20201129232924574](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/多线程/images/image-20201129232924574.png)

用途：等待一些线程执行结束后，才开始执行本线程的任务。例如：压测的时候，写单元测试可以用来模拟并发，等多少线程才开始执行。

* CountDownLatch countDownLatch = new CountDownLatch\(3\);初始化总数 
* countDownLatch.countDown\(\); 每调用一次减1
* countDownLatch.await\(\);进行阻塞，当减到0才开始运行后面的代码

例如：某个线程在执行任务之前，需要等待其它线程完成一些前置任务，必须等所有的前置任务都完成，才开始执行本线程的任务。

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;


public class CountDownLatchDemo {
    // 定义前置任务线程
    static class PreTaskThread implements Runnable {

        private String task;
        private CountDownLatch countDownLatch;

        public PreTaskThread(String task, CountDownLatch countDownLatch) {
            this.task = task;
            this.countDownLatch = countDownLatch;
        }

        @Override
        public void run() {
            try {
                Random random = new Random();
                Thread.sleep(random.nextInt(1000));
                System.out.println(task + " - 任务完成");
                // countDown减1
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        // 假设有三个模块需要加载
        CountDownLatch countDownLatch = new CountDownLatch(3);

        // 主任务
        new Thread(() -> {
            try {
                System.out.println("等待数据加载...");
                System.out.println(String.format("还有%d个前置任务", countDownLatch.getCount()));
                // 进行阻塞
                countDownLatch.await();
                System.out.println("数据加载完成，正式开始游戏！");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // 前置任务
        new Thread(new PreTaskThread("加载地图数据", countDownLatch)).start();
        new Thread(new PreTaskThread("加载人物模型", countDownLatch)).start();
        new Thread(new PreTaskThread("加载背景音乐", countDownLatch)).start();
    }
}
```

结果：

```text
等待数据加载...
还有3个前置任务
加载人物模型 - 任务完成
加载背景音乐 - 任务完成
加载地图数据 - 任务完成
数据加载完成，正式开始游戏！
```

