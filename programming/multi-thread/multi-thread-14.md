# 14 ThreadLocal

## ThreadLocal

作用：进行线程隔离，为每个线程提供一个私有变量。

问：那为什么不直接声明一个私有变量？

答：如果将类的某个**静态变量**（user ID或者transaction ID）与线程状态关联，声明变量的形式就不行了，因此有了ThreaLocal

```java
private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<>();
```

### ThreadLocal实现原理

ThreadLocal是一种以空间换时间的做法，把数据进行隔离，数据不共享。

在**每个Thread里面维护了一个ThreadLocal.ThreadLocalMap** ，然后将数据放到这个map中。

ThreadLocalMap是⼀个类似 Map的数据结构，key 为当前Thread对象里维护的ThreadLocal对象，值为 Object 对象。

```java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护
ThreadLocal.ThreadLocalMap threadLocals = null;
//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}

// ------------------------------------------------------------------------------------------------------------------------
public class ThreadLocal<T> {
    public void set(T value) {
         Thread t = Thread.currentThread();
         ThreadLocalMap map = getMap(t);
         if (map êX null)
            map.set(this, value); //thisThreadLocal
         else
            createMap(t, value);
         }
         ThreadLocalMap getMap(Thread t) {
         return t.threadLocals;
     }
```

### ThreadLocal 内存泄露问题

ThreadLocalMap类似于WeakHashMap， 使⽤的 key 为 ThreadLocal 的**弱引⽤**，⽽ value 是强引⽤。

```java
static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

key：弱引用。如果 ThreadLocal 没有被外部强引⽤，在垃圾回收的时候，key可能被清理掉，但value不会。

这样ThreadLocalMap 中就会出现**key为null的Entry**。value 永远⽆法被GC 回收，产生内存泄漏。

### ThreadLocalMap泄漏解决办法

ThreadLocalMap中有一个类似于WeakHashMap中的expungeStaleEntries\(\)方法，叫做：expungeStaleEntry\(\)

expungeStaleEntry\(\)会自动清理value，原理同WeakHashMap，使用ReferenceQueue：GC后，就会将数据放入到ReferenceQueue队列里面，**进行通知**，便可以利用这个通知进行处理key为null的value了。

解决方法：**使⽤完 ThreadLocal 后 ⼿动调⽤ remove\(\) ⽅法**

```java
        /**
         * Remove the entry for key.
         */
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    // 清理value
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

