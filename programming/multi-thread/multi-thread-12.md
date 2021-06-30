# 12 Exchanger

Exchanger：交换机

![image-20201129233325419](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/多线程/images/image-20201129233325419.png)

用途：

用于两个线程交换数据，注意只能是两个线程。

支持泛型，也就是说可以在两个线程之间传送任何数据。

例如：两个线程之间想要传送字符串

* exchanger.exchange

```java
public class ExchangerDemo {

    public static void main(String[] args) throws InterruptedException {
        // Exchanger<V> 这里设置为String类型
        Exchanger<String> exchanger = new Exchanger<>();

        new Thread(() -> {
            try {
                System.out.println("这是线程A，得到了另一个线程的数据："
                        + exchanger.exchange("aaaaa")); // exchanger.exchange()交换数据，会进行阻塞
                // public V exchange(V x) throws InterruptedException {
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        System.out.println("这个时候线程A是阻塞的，在等待线程B的数据");
        Thread.sleep(1000);

        new Thread(() -> {
            try {
                System.out.println("这是线程B，得到了另一个线程的数据："
                        + exchanger.exchange("bbbbb"));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

结果：

```text
这个时候线程A是阻塞的，在等待线程B的数据
这是线程B，得到了另一个线程的数据：aaaaa
这是线程A，得到了另一个线程的数据：bbbbb
```

