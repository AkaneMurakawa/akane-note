# 15 并发容器

所谓的容器，就是用于存储数据的一个东西，重点是存储。例如：我们的ArrayList和HashMap

但这些都不线程安全的，那当遇到并发的时候，我们就应该用并发容器。今天就来探究一下。请注意区分同步锁和并发容器，一个是锁，一个是容器。

## 容器分类

在开始之前，首先对容器进行一下分类。从物理上，只有数组和链表两种数据结构区分。

从用途上可以分为集合和Map，集合又可以分为List和Set、Queue

从实现的角度上，又可以分为数组实现和队列\(链表\)实现，树实现。

从线程安全角度，又可以分为线程安全和非安全。

* Collecttion
  * List
    * ArrayList
    * LinkedList
    * Vector\(Stack\)
    * CopyOnWriteList
  * Set（不允许重复、且无序）
    * HashSet \(LinkedHashSet\)
    * SortedSet\(TreeSet：红黑树结构，树保证有序\)
    * EnumSet
    * CopyOnWriteArryaSet
    * ConcurrentSkipListSet
  * Queue（队列）
    * Deque
    * BlockingQueue
    * PriorityQueue
    * ConcurrentLinkedQueue
    * DelayQueue
* Map
  * HashMap
  * TreeMap
  * WeakHashMap

注：TreeMap、TreeSet以及JDK1.8的HashMap底层都⽤到了红⿊树。

## 早期的容器

早期的容器，如ArrayList和HashMap，这些都不是线程安全的，于是在此基础上加synchronized，实现了线程安全的Vector、Hashtable。

但是现在2020年已经基本不使用，原因是：Vector、Hashtable自带锁，本质上是synchronized，在线程多的情况下效率不高。

## 并发List

* ArrayList的并发版本——&gt;CopyOnWriteArrayList
* LinkedList的并发版本——&gt;ConcurrentLinkedQueue、ConcurrentLinkedDeque

## 并发Set

* HashSet的并发版本——&gt;CopyOnWriteArraySet
* LinkedHashSet
* TreeSet的并发版本——&gt; ConcurrentSkipListSet（有序）

## 并发Queue

* Queue的并发版本——&gt;ConcurrentLinkedQueue
* Deque的并发版本——&gt;ConcurrentLinkedDeque

## 并发Map

* HashMap的并发版本——&gt;ConcurrentHashMap
* LinkedHashMap
* TreeMap的并发版本——&gt;ConcurrentSkipListMap

## 小结

各容器的并发实现，总共分为三种命名方法：

一种是Concurrent\*

一种是ConcurrentSkipList\*

一种是CopyOnWrite\*

## 并发Map解释

### ConcurrentHashMap

ConcurrentHashMap是HashMap的线程安全版本，在synchronized版本的基础上做了改进。

### ConcurrentHashMap是如何解决并发的

答：1.7用的Segment分段锁，1.8用的是synchronized和CAS。[https://cloud.tencent.com/developer/article/1443877](https://cloud.tencent.com/developer/article/1443877)

JDK1.7使用的是分段锁

分段锁就是**将数据分段，对每一段数据分配一把锁**。当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

有些方法需要跨段，比如size\(\)、isEmpty\(\)、containsValue\(\)，需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。

注：有点类似表锁和Next-key Lock

![image.png](../../.gitbook/assets/java39.png)

JDK1.8使用synchronized和CAS极大的提高了HashMap的并发能力，只要做了以下两点优化：

* synchronized：锁首个节点，这样**只要hash不冲突，就不会产生并发**
* addCount：主要是为了扩容，利用的是CAS。注意addCount不在synchronized代码块里

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//对hashCode进行再散列，算法为(h ^ (h >>> 16)) & HASH_BITS
    int binCount = 0;
 //这边加了一个循环，就是不断的尝试，因为在table的初始化和casTabAt用到了compareAndSwapInt、compareAndSwapObject
    //因为如果其他线程正在修改tab，那么尝试就会失败，所以这边要加一个for循环，不断的尝试
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果table为空，初始化；否则，根据hash值计算得到数组索引i，如果tab[i]为空，直接新建节点Node即可。注：tab[i]实质为链表或者红黑树的首节点。
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果tab[i]不为空并且hash值为MOVED(-1)，说明该链表正在进行transfer操作，返回扩容完成后的table。
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 针对首个节点进行加锁操作，而不是segment，进一步减少线程冲突
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果在链表中找到值为key的节点e，直接设置e.val = value即可。
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 如果没有找到值为key的节点，直接新建Node并加入链表即可。
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 如果首节点为TreeBin类型，说明为红黑树结构，执行putTreeVal操作。
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 如果节点数>＝8，那么转换链表结构为红黑树结构。
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 计数增加1，有可能触发transfer操作(扩容)。
    addCount(1L, binCount);
    return null;
}
```

### ConcurrentSkipListMap

HashMap的并发容器是CocurrentHashMap，但是TreeMap却比较特殊。

由于TreeMap使用的是红黑树的数据结构，如果使用CAS实现起来太复杂。

因此换了一种方法，那就是跳表，实现了ConcurrentSkipListMap。TreeSet同，ConcurrentSkipListSet。

底层使用的是跳表数据结构，高并发并且排序。[https://www.cnblogs.com/crazymakercircle/p/13928386.html](https://www.cnblogs.com/crazymakercircle/p/13928386.html)

SkipList：跳表的意思

#### 跳表

是一种”空间换时间“的数据结构

[https://www.jianshu.com/p/fef9817cc943](https://www.jianshu.com/p/fef9817cc943)

## CopyOnWrite容器

写时复制，适合读多写少的的场景。[http://concurrent.redspider.group/article/03/16.html](http://concurrent.redspider.group/article/03/16.html)

CopyOnWrite容器：即**写时复制的容器**，当我们往一个容器中添加元素的时候，不直接往容器中添加，而是将当前容器进行copy，复制出来一个新的容器，然后向新容器中添加我们需要的元素，最后将原容器的引用指向新容器。

* CopyOnWriteArrayList
* CopyOnWriteArraySet

思想：

* 读写分离
* 最终一致性
* 牺牲空间解决并发冲突

### 优缺点

**优点**：适合读多写少的的场景

**缺点**：每次执行写操作都会将原容器进行拷贝一份，数据量大的时候，内存会存在较大的压力，可能会引起频繁Full GC。

​ 在写操作执行过程中，读不会阻塞，但读取到的却是老容器的数据，**无法满足实时性**。

### 自定义CopyOnWriteMap

参考CopyOnWriteArrayList实现的CopyOnWriteMap

```java
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {

    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap(int initialCapacity) {
        this.internalMap = new HashMap<K, V>(initialCapacity);
    }

    @Override
    public V get(Object key) {
        return internalMap.get(key);
    }

    @Override
    public V put(K key, V value) {
        synchronized (this){
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            return val;
        }
    }

    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }

    @Override
    public int size() {
        return 0;
    }

    @Override
    public boolean isEmpty() {
        return false;
    }

    @Override
    public boolean containsKey(Object key) {
        return false;
    }

    @Override
    public boolean containsValue(Object value) {
        return false;
    }

    @Override
    public V remove(Object key) {
        return null;
    }

    @Override
    public void clear() {

    }

    @Override
    public Set<K> keySet() {
        return null;
    }

    @Override
    public Collection<V> values() {
        return null;
    }

    @Override
    public Set<Entry<K, V>> entrySet() {
        return null;
    }
}
```

### 使用示例

**场景：**假如我们有一个搜索的网站需要屏蔽一些“关键字”，“黑名单”每晚定时更新，每当用户搜索的时候，“黑名单”中的关键字不会出现在搜索结果当中，并且提示用户敏感字。

```java
/**
 * CopyOnWrite 应用示例
 *
 * 需要各位小伙伴特别特别注意一个问题，此处的场景是每晚凌晨 "黑名单"定 时更新，
 * 原因是CopyOnWrite容器有数据一致性的问题，它只能保证最终数据一致性。
 *
 * 所以如果我们希望写入的数据马上能准确地读取，请不要使用CopyOnWrite容器。
 */
public class BlackListServiceImpl {

    // 减少扩容开销。根据实际需要，初始化CopyOnWriteMap的大小，避免写时CopyOnWriteMap扩容的开销
    private static CopyOnWriteMap<String, Boolean> blackListMap =
            new CopyOnWriteMap<String, Boolean>(1000);

    public static boolean isBlackList(String key) {
        return blackListMap.get(key) == null ? false : true;
    }

    public static void addBlackList(String id) {
        blackListMap.put(id, Boolean.TRUE);
    }

    /**
     * 批量添加黑名单
     * (使用批量添加。因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。
     * 如使用上面代码里的addBlackList方法)
     * @param keys
     */
    public static void addBlackList(Map<String,Boolean> keys) {
        blackListMap.putAll(keys);
    }


}
```

## Queue

List有时候并不能满足我们的需求，用队列可以**在线程间传递数据**，例如：Queue可实现生产者、消费者模式

Queue提供了几个基本的方法：

* offer：添加数据，返回true和false 
* add：添加数据，失败则抛异常
* peek：获取数据但不移除 
* poll：获取数据并移除

### BlockingQueue

blocking：阻塞的意思

BlockingQueue是对Queue进一步的改进，用于并发编程的容器。Queue提供的基本方法都不会阻塞，BlockingQueue提供了两个**阻塞**的方法put\(\)和take\(\)

put\(\)和take\(\)的原理：其实利用的就是LockSupport的park和unPark方法，当队列里无任务时，take就会被park阻塞住，当有任务来说，unPark唤醒阻塞的线程。

![image-20201129233428789](../../.gitbook/assets/image-20201129233428789.png)

### ArrayBlockingQueue

* 由数组结构组成的有界阻塞队列
* 可以初始化队列大小， 且一旦初始化不能改变
* 构造方法中的fair表示控制对象的内部锁是否采用公平锁，默认是非公平锁

### LinkedBlockingQueue

* 由链表结构组成的阻塞队列
* 默认队列的大小是Integer.MAX\_VALUE，也可以指定大小。按照先进先出的原则对元素进行排序。

### DelayQueue

delay：延迟的意思

* 按照延迟时间入队，**延迟时间到了才能从队列中获取到该元素**。
* 因为是按照延迟时间入队，因此是有序的。
* DelayQueue是一个**没有大小限制**的队列，因此插入数据（生产者）永远不会被阻塞，而获取数据消费者）会被阻塞。（因此注意生产速度不能大于消费速\) 

用途：按时间进行任务调度，本质是一个优先队列PriorityQueue。

```java
static BlockingQueue<MyTask> tasks = new DelayQueue<>();

// 添加的任务必须实现java.util.concurrent.Delayed 接口
static class MyTask implements Delayed {
```

### PriorityBlockingQueue

* 基于优先级的无界阻塞队列，默认采用的是非公平锁
* 优先级通过PriorityBlockingQueue构造函数传入Compator对象

```java
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
```

示例：

```java
public class PriorityBlockingQueueDemo {
    static class User{
        int age;
        String name;

        public User(int age, String name) {
            this.age = age;
            this.name = name;
        }
    }


    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<User> queue = new PriorityBlockingQueue<>(10,
                (o1, o2) -> o1.age - o2.age
        );
        queue.put(new User(18, "a"));
        queue.put(new User(30, "b"));
        queue.put(new User(20, "c"));
        queue.put(new User(25, "d"));

        for (int i = 0; i < 4; i++) {
            System.out.println(queue.take().age);
        }
    }
}
```

结果：

```java
18
20
25
30
```

### SynchronousQueue

Synchronous：同步的意思，synchronize的形容词adj

* **容量为0**
* put\(\)必须等待take\(\)，插入数据会**阻塞等待**消费者消费结束

用途：等待消费完成再进行下面任务。类似于exchanger，两个线程交换数据

```java
public class SynchronusQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<String> strs = new SynchronousQueue<>(); // 容量为0

        new Thread(()->{
            try {
                System.out.println("take:" + strs.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        strs.put("aaa"); // 阻塞等待消费者消费
        System.out.println("size:" + strs.size()); // size:0
        strs.put("bbb"); // 无消费者，会一直阻塞
        System.out.println("size:" + strs.size());
    }
}
```

### TransferQueue

Transfer：传递

* TransferQueue是`LinkedBolckingQueue` 和 `SynchronousQueue`的改进
* LinkedTransferQueue采用一种**预占模式**。
* take\(\)时，如果发现队列为空，则会生成一个null节点，然后park住等待生产者。
* 如果生产者将元素填充到该节点上，则会唤醒该节点的消费者线程。

总结：与SynchronousQueue相比，多了队列容量，容量不是0。相当于更灵活的SynchronousQueue

用途：等消费完成再进行下面的任务

示例：

* put：不进行阻塞
* transfer：阻塞
* take：阻塞

```java
public class TransferQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        LinkedTransferQueue<String> strs = new LinkedTransferQueue<>();

        new Thread(() -> {
            try {
                System.out.println("take：" + strs.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        strs.transfer("aaa");
        System.out.println("transfer");
        strs.put("bbbb"); // aync，不进行阻塞
        new Thread(() -> {
            try {
                System.out.println("take：" + strs.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        strs.transfer("dddd"); // 无消费者，会一直阻塞
        try {
            System.out.println("take：" + strs.take());// 前面dddd一行已经阻塞，因此不会调用这一行，但是前面注释掉同样会进行阻塞
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

结果：

```text
take：aaa
transfer
take：bbbb
```

