# 9 CyclicBarrierDemo

前面提到CountDownLatch是“门闩”或“屏障”的意思，但缺点是一旦计数值count被降为0后，就不能再重新设置了，它**只能起一次“屏障”的作用**。

而CyclicBarrier拥有CountDownLatch的所有功能，还可以使用reset\(\)方法重置屏障，**称为“循环的屏障”。** Barrier：屏障

![image-20201129232948405](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/多线程/images/image-20201129232948405.png)

用途：限流，入口特别多，但是出口限制住（但是一般用Guava RateLimiter）

* CyclicBarrier
* cyclicBarrier.await\(\);阻塞
* cyclicBarrier.reset\(\); 重置屏障

```java
import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;


public class CyclicBarrierDemo {
    static class PreTaskThread implements Runnable {

        private String task;
        private CyclicBarrier cyclicBarrier;

        public PreTaskThread(String task, CyclicBarrier cyclicBarrier) {
            this.task = task;
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            // 假设总共三个关卡
            for (int i = 1; i < 4; i++) {
                try {
                    Random random = new Random();
                    Thread.sleep(random.nextInt(1000));
                    System.out.println(String.format("关卡%d的任务%s完成", i, task));
                    // 阻塞
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
                cyclicBarrier.reset(); // 重置屏障
            }
        }
    }

    public static void main(String[] args) {
        // 3个线程满开始执行
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, () -> {
            System.out.println("本关卡所有前置任务完成，开始游戏...");
        });

        new Thread(new PreTaskThread("加载地图数据", cyclicBarrier)).start();
        new Thread(new PreTaskThread("加载人物模型", cyclicBarrier)).start();
        new Thread(new PreTaskThread("加载背景音乐", cyclicBarrier)).start();
    }
}
```

结果：

```text
关卡1的任务加载背景音乐完成
关卡1的任务加载人物模型完成
关卡1的任务加载地图数据完成
本关卡所有前置任务完成，开始游戏...
关卡2的任务加载地图数据完成
关卡2的任务加载背景音乐完成
关卡2的任务加载人物模型完成
本关卡所有前置任务完成，开始游戏...
关卡3的任务加载背景音乐完成
关卡3的任务加载人物模型完成
关卡3的任务加载地图数据完成
本关卡所有前置任务完成，开始游戏...
```

