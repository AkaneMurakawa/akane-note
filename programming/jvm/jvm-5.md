# 5 垃圾回收器

## 单词说明

`Parallel：并行` `Concurrent ：并发` `Serial：串行`

`Heap：堆` `Stack：栈` `Native Stack：本地方法栈` `Survivor：幸存者` `Metaspace：元空间` `Young：年轻` `Old：年老` `New:年轻代` `Tenured：终身制` `Permanent：永久`

`Garbage Collector: 垃圾收集器，简称GC` `Memory Fragmentation：内存碎片` `Floating Garbage：浮动垃圾`

`Mark-Sweep：标记清除` `copying：复制` `Mark-Compact：标记压缩`

`Ratio: 比，比率` `Region: 区域`

* 并发concurrent：任务的提交，有处理多个任务的能力，不一定要同时。
* 并行parallel：任务的执行，有同时处理多个任务的能力。

## 垃圾回收器分类

**注：红线表示能组合，红色箭头表示常用组合。Young和Old的组合，例如：PS+PO，表示Young用PS，Old用PO。在下图中Old表示的是老年代版本**。

垃圾回收组合：

* Serial + Serial Old
* **PS+PO（默认）**
* ParNew + CMS

![image-20201122131438954](../../.gitbook/assets/image-20201122131438954.png)

### 1、Serial GC\(-XX:+UseSerialGC 复制算法\)

> a stop-the-world, copying collector which uses a single GC thread.

**串行单线程GC**，会STW。标记复制算法

* 单CPU效率最高
* jvm参数指定-client时默认的垃圾收集器

![image-20201121174902931](../../.gitbook/assets/image-20201121174902931.png)

JVM运行模式

* Server
* Client

```text
C:\Users\AkaneMurakawa>java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode) # Server模式
```

### 2、Serial Old GC（老奶奶）\(-XX:+UseSerialOldGC 标记-压缩算法\)

> a stop-the-world, mark-sweep-compact collector that uses a single GC thread.

同上，**Old表示内存区中的老年代版本**，只不过用的是mark-sweep-compact算法

### 3、Parallel Scavenge GC（PS）\(-XX:+UseParallelGC 复制算法\)

> a stop-the-world, copying collector which uses a multiple GC thread.

术语简称PS，默认。

**并行多线程GC**

* 它关心的是**吞吐量**。吞吐量 = 用户代码time/\(用户time+ GC time\)；

![image-20201121175136813](../../.gitbook/assets/image-20201121175136813.png)

### 4、Parallel Old GC（PO）\(-XX:+UseParallelOldGC 标记-压缩算法\)

> a compacting collector that uses multiple GC threads.

术语简称PO

同上，**Old表示内存区中的老年代版本**，只不过用的是整理算法

### 5、ParNew GC （Parallel New）\(-XX:+UseParNewGC 复制算法\)

> a stop-the-world, copying collector which uses multiple GC thread.
>
> It differs from "Parallel Scavenge" in that it has enhancements that make it usable with CMS
>
> For example, "ParNew" does the synchronization needed so that it can run during the concurrent phases of CMS.

**并行多线程GC**，会STW

和Parallel Scavenge GC（PS）差不多，是PS的变种。只不过做了增强，为了更好的和CMS结合使用。

[https://docs.oracle.com/en/java/javase/13/gctuning/ergonomics.html\#GUID-3D0BB91E-9BFF-4EBB-B523-14493A860E73](https://docs.oracle.com/en/java/javase/13/gctuning/ergonomics.html)

### 6、CMS（Concurrent Mark Sweep）\(-XX:+UseConcMarkSweepGC 标记清除算法\)

注：为了配合CMS，诞生了PN，CMS是**1.4**版本后期引入，CMS开启了并发回收的过程，但是CMS问题较多，因此目前没有任何一个JDK版本默认为CMS。

**CMS并发垃圾回收是因为无法忍受STW**

注意：这里用的是标记清除算法

**并发标记扫描GC**

* 占用CPU，**空间碎片（Concurrent Mark Sweep采用标记清除算法\(Mark-Sweep\)）**，并发异常
* CMS收集器关注点是**响应时间（STW越短，响应时间越好），低延迟**，**关注的是用户体验**；

![image-20201122012746375](../../.gitbook/assets/image-20201122012746375.png)

#### CMS四个阶段

阶段一：初始标记（CMS initial mark），**会STW**（注：图中全黄色的表示会STW）

阶段二：并发标记（CMS concurrent mark）

阶段三：重新标记（CMS remark），**会STW**

阶段四：并发清除（CMS concurrent sweep）

![image-20201122015018539](../../.gitbook/assets/image-20201122015018539.png)

**初始标记**

暂停所有的其他线程，并记录下直接与root相连的对象，**速度很快** ；

**并发标记**

**同时开启GC和⽤户线程**，⽤⼀个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。

因为⽤户线程可能会不断的更新引⽤域，所以GC线程⽆法保证可达性分析的实时性。所以这个算法⾥会跟踪记录这些发⽣引⽤更新的地⽅。

**重新标记**

重新标记阶段就是为了修正并发标记期间因为⽤户程序继续运⾏⽽导致标记产⽣变动的那⼀部分对象的标记记录，这个阶段的停顿时间⼀般会⽐初始标记阶段的时间稍⻓，远远⽐并发标记阶段时间短

**并发清除**

开启⽤户线程，同时GC线程开始对为标记的区域做清扫。（浮动垃圾）

### CMS两大问题

1. Memory Fragmentation（内存碎片）

   > -XX:+UseCMSCompactAtFullCollection
   >
   > -XX:CMSFullGCsBeforeCompaction 默认为0 指的是经过多少次FGC才进行压缩

2. Floating Garbage（浮动垃圾）

   > Concurrent Mode Failure启动Serial Old
   >
   > 产生：if the concurrent collector is unable to finish reclaiming the unreachable objects before the tenured generation fills up, or if an allocation cannot be satisfiedwith the available free space blocks in the tenured generation, then theapplication is paused and the collection is completed with all the applicationthreads stopped
   >
   > 解决方案：降低触发CMS的阈值
   >
   > PromotionFailed
   >
   > 解决方案类似，保持老年代有足够的空间
   >
   > –XX:CMSInitiatingOccupancyFraction 92% 可以降低这个值，让CMS保持老年代足够的空间

ConcurrentMarkSweep 老年代 并发的， 垃圾回收和应用程序同时运行，降低STW的时间\(200ms\) CMS问题比较多，所以现在没有一个版本默认是CMS，只能手工指定 CMS既然是MarkSweep，就一定会有碎片化的问题。

碎片到达一定程度，CMS的老年代分配对象分配不下的时候，使用**SerialOld** 进行老年代回收。

想象一下： PS + PO -&gt; 加内存 换垃圾回收器 -&gt; PN + CMS + SerialOld（几个小时 - 几天的STW） 几十个G的内存，单线程回收 -&gt; G1 + FGC 几十个G -&gt; 上T内存的服务器

### 并发标记阶段的算法

CMS在并发标记阶段是如何标记的？

答：算法：三色标记 + Incremental Update

[https://www.jianshu.com/p/12544c0ad5c1](https://www.jianshu.com/p/12544c0ad5c1)

[https://zhuanlan.zhihu.com/p/105495961/](https://zhuanlan.zhihu.com/p/105495961/)

三色标记法步骤：

* 首先将对象用三种颜色表示，分别是白色、灰色和黑色。
* 最开始所有对象都是白色的，然后把其中全局变量和函数栈里的对象置为灰色。
* 第二步把灰色的对象全部置为黑色，然后把原先灰色对象指向的变量都置为灰色，以此类推。
* 等发现没有对象可以被置为灰色时，所有的白色变量就一定是需要被清理的垃圾了。

相比传统的标记清扫算法，三色标记最大的好处是可以**异步执行**，从而可以以中断时间极少的代价或者完全没有中断来进行整个 GC。

三色标记法因为多了一个白色的状态来存放不确定的对象，所以可以异步地执行。当然异步执行的代价是可能会造成一些遗漏，因为那些早先被标记为黑色的对象可能目前已经是不可达的了。所以三色标记法是一个 false negative（假阴性）的算法。

### 7、G1（Garbage-First）\(-XX:+UseG1Gc 复制+标记压缩算法\)

算法：三色标记 + SATB

G1收集器特点：

* 并发和并行
* 分代收集
* 空间整理
* 可预测停顿

**G1为jdk1.7实验性gc，为java9默认的gc**

* 将整个Java堆内存划分成多个大小相等的Region
* 年轻代和老年代不再物理隔离

![image-20201129154455566](../../.gitbook/assets/image-20201129154455566.png)

### 8、ZGC\(jdk11\)

ZGC \(1ms\) PK C++ 算法：ColoredPointers + LoadBarrier

### 9、Shenandoah

算法：ColoredPointers + WriteBarrier

### 10、Epsilon

## 垃圾收集器跟内存大小的关系

1. Serial 几十兆
2. PS 上百兆 - 几个G
3. CMS - 20G
4. G1 - 上百G
5. ZGC - 4T - 16T（JDK13）

## 查询JDK版本使用的默认回收器

`java -XX:+PrintCommandLineFlags -version` 或 jmap -heap pid，可以看到

```text
using thread-local object allocation.

Parallel GC with 8 thread(s)
```

## 查询内存分配信息

`java -XX:+PrintGCDetails -version`

## 如何查看设置的参数

`java -XX:+PrintFlagsFinal -version | grep NewSize`

`java -XX:+PrintFlagsInitial -version`

注：通过grep过滤，PrintFlagsInitial可以查看默认值

