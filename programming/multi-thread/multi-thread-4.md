# 4 CAS

## 自旋锁

虽然用锁或 synchronized 关键字可以实现原子操作，但是加锁或使用 synchronized 关键字带来的性能损耗较大。

因此又出现了一种锁，叫做自旋锁，号称无锁优化 。

自旋锁：在循环里不断的尝试获取锁。

### 自旋锁和系统锁区别

* 系统锁不占用CPU，进入等待队列。所以执行时间长、线程数比较多的用系统锁
* 自旋锁占用CPU，一直在循环里尝试获取锁。所以执行时间短、线程数比较少用自旋锁

## CAS

自旋锁是一种概念，而CAS 是实现自旋锁的基础，CAS 利用 CPU 指令保证了操作的原子性，以达到锁的效果。CAS可以实现乐观锁，它实际上是直接利用了 CPU 层面的指令，所以性能很高。

注：CAS有些喜欢叫做自旋锁，也有人喜欢叫做无锁，这些都是概念的东西。

CAS的全称是：比较并交换（Compare And Swap）。在CAS中，有这样三个值：

* V：要更新的变量\(var\)
* E：预期值\(expected\) 
* N：新值\(new\)

比较并交换的过程如下：

> cas\(Value, Expected, NewValue\)
>
> if V == E
>
> ​ V= New
>
> otherwise try again or fail

判断V是否等于E，如果等于，将V的值设置为New；

如果不等，说明已经有其它线程更新了V。

这里的**预期值E本质上指的是“旧值”**。

**那有没有可能在判断了`V == E`之后，正准备更新时，被其它线程更改了V的值呢？**

答：不会的。因为CAS是一种原子操作，系统级别的。它是一种系统原语，是一条CPU的原子指令，从CPU层面保证它的原子性。

### CAS实现原理

CAS是一种原子操作，Java中利用的是一个sun.misc包下Unsafe类。里面的方法大部分是native方法，也就是说底层调用的是C或C++。

简单来说，CAS的实现原理：便是利用了Unsafe类，而这个类底层调用C或C++，里面的方法提供了原子操作。

Unsafe类下提供了几个原子操作的native方法：

注：这是JDK1.8版本

```java
boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);
boolean compareAndSwapInt(Object o, long offset,int expected,int x);
boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```

Unsafe类使得Java有了指针的能力

```java
// 直接操作内存
public native long allocateMemory(long var1);
public native void freeMemory(long var1);
public native void putInt(long var1, int var3);
public native int getInt(long var1);
public native int pageSize();
public native long reallocateMemory(long var1, long var3);

// 可以直接生成类的实例
public native Object allocateInstance(Class<?> var1) throws InstantiationException;
// 可以直接操作类或实例变量
public native long objectFieldOffset(Field var1);
```

线程挂起和恢复

```java
public native void unpark(Object var1);

public native void park(boolean var1, long var2);
```

## CAS三大问题

* ABA问题
* 循环时间长开销大
* 只能保证一个共享变量的原子操作

### ABA问题

所谓ABA问题，就是一个值原来是A，变成了B，又变回了A。当一个线程想要去操作时，发现是A于是操作成功了。但实际上却被更新了两次。

这个时候使用CAS是检查不出变化的，但可能存在潜藏的问题。例如：如果是基本数据类型，没什么问题，但是**对象引用**就可能出现问题

### ABA问题解决

ABA问题的解决思路是在变量前面追加上**版本号或者时间戳**。（在MySQL中MVCC也有类似的思想）

> 例如：
>
> A 1.0
>
> B 2.0
>
> A 3.0

JDK1.5开始，提供了类**AtomicStampedReference**（stamped：时间戳）来解决 ABA问题。

这个类的compareAndSet方法的作用是首先检查**当前引用是否等于预期引用**，并且检查当前标志是否等于预期标志，如果二者都相等，才使用CAS设置为新的值和标志。

### 循环时间长开销大

CAS经常和自旋一起使用。而自旋是占用CPU的，循环里不断的重试。如果自旋CAS长时间不成功，会占用大量的CPU资源。

### 循环时间长开销大解决思路

让JVM支持处理器提供的**pause指令**。

pause指令能让自旋失败时cpu睡眠一小段时间再继续自旋，从而**使得读操作的频率低**很多。

### 只能保证一个共享变量的原子操作

JDK 1.5之前，只能保证一个共享变量进行原子操作，那如果我有多个变量怎么办？

有两种解决方案：

1. JDK 1.5开始，可以使用AtomicReference类，然后把多个变量放到一个对象里面进行CAS操作；面向对象编程。
2. 加锁。锁内的临界区代码可以保证只有当前线程能操作。

## 原子操作-AtomicInteger类源码分析

AtomicXXX是原子操作的，实现的原理便是：**CAS \(compare and swap\) + volatile + native**

CAS操作：其实底层调用的是Unsafe类的native方法，底层由C或C++实现。

volatile ：禁止内存指令重排，保证内存可见性。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    // 保证可见性
    private volatile int value;

        /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

示例：

```java
/**
 * Atomic 示例
 */
public class AtomicIntergerDemo {

    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(0);
        int i = atomicInteger.incrementAndGet();
        i = atomicInteger.incrementAndGet();
        System.out.println(i);
    }
}
```

## 乐观锁和悲观锁

锁按照不同的角度，又分为乐观锁和悲观锁。乐观锁的实现用的就是CAS思想。

### 悲观锁

顾名思义很悲观，认为每次访问共享资源都会发生冲突。因此每次都给数据操作加锁。

### 乐观锁

顾名思义很乐观，认为每次访问共享资源不会发生冲突。因此无需加锁。乐观锁又称为“无锁”。

**但一旦出现多个线程冲突时，就使用CAS自旋锁来保证执行的安全性。注：自旋会消耗一定的性能。**

### 对比

乐观锁：多用于“读多写少“的环境，避免频繁加锁影响性能；

悲观锁：多用于”写多读少“的环境，避免频繁失败和**重试**影响性能。

