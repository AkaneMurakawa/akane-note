# 13 LockSupport

线程挂起和恢复

用途：线程的挂起和恢复，例如：交叉打印

* 叫醒指定的线程
* LockSupport.unpark\(t1\)可以在LockSupport.park\(\);前调用

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.LockSupport;

public class LockSupportDemo {

    public static void main(String[] args) {


        Thread t1 = new Thread(() ->{

            for (int i = 0; i < 10; i++) {
                System.out.println(i);
                if (i == 5){
                    LockSupport.park();
                }
                try {
                    Thread.sleep(1000);
                }catch (InterruptedException e){}
            }
        });

        t1.start();

        try {
            TimeUnit.SECONDS.sleep(8);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("after 8 secondes");
        LockSupport.unpark(t1);
    }
}
```

结果：

```text
0
1
2
3
4
5
after 8 secondes
6
7
8
9
```

