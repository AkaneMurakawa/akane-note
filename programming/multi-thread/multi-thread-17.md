# 17 JMH

## JMH介绍

Java Microbenchmark Harness，java基准测试工具

用途：测试并发量，QPS等，测试开发职位常使用的工具。当然有时候也有用Jmeter

官网： [http://openjdk.java.net/projects/code-tools/jmh/](http://openjdk.java.net/projects/code-tools/jmh/)

官方样例： [http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)

## JMH使用

### 1. 添加maven依赖

```markup
         <!-- https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-core -->
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.21</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-generator-annprocess -->
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.21</version>
            <scope>test</scope>
        </dependency>
```

### 2. 设置

* IDEA安装JMH插件 JMH plugin
* 由于用到了注解，打开运行程序注解配置

> compiler -&gt; Annotation Processors -&gt; Enable Annotation Processing

### 3. 编写所需测试类

```java
public class PS {

    static List<Integer> nums = new ArrayList<>();
    static {
        Random r = new Random();
        for (int i = 0; i < 10000; i++) nums.add(1000000 + r.nextInt(1000000));
    }

    public static void foreach() {
        nums.forEach(v->isPrime(v));
    }

    public static void parallel() {
        nums.parallelStream().forEach(PS::isPrime);
    }

    static boolean isPrime(int num) {
        for(int i=2; i<=num/2; i++) {
            if(num % i == 0) return false;
        }
        return true;
    }
}
```

### 4.编写单元测试

注：一定要在test package下面

```java
import com.example.thread.other.PS;
import org.openjdk.jmh.annotations.*;

public class PSTest {
    // 所需测试的代码上标记
    @Benchmark
    // 预热，由于JVM中对于特定代码会存在优化（本地化），预热对于测试结果很重要
    // iterations: 迭代次数
    @Warmup(iterations = 1, time = 3)
    // 线程数
    @Fork(5)
    // 基准测试的模式，这里为QPS，ops/time
    @BenchmarkMode(Mode.Throughput)
    // 总共测试几次
    @Measurement(iterations = 1, time = 3)
    public void testForEach() {
        PS.foreach();
    }
}
```

结果：

```text
# JMH version: 1.21
# VM version: JDK 1.8.0_171, Java HotSpot(TM) 64-Bit Server VM, 25.171-b11
# VM invoker: C:\Program Files\Java\jdk1.8.0_171\jre\bin\java.exe
# VM options: -Dfile.encoding=UTF-8
# Warmup: 1 iterations, 3 s each
# Measurement: 1 iterations, 3 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Benchmark: com.example.PSTest.testForEach

# Run progress: 0.00% complete, ETA 00:00:30
# Fork: 1 of 5
# Warmup Iteration   1: 0.283 ops/s
Iteration   1: 0.440 ops/s

# Run progress: 20.00% complete, ETA 00:00:37
# Fork: 2 of 5
# Warmup Iteration   1: 0.415 ops/s
Iteration   1: 0.469 ops/s

# Run progress: 40.00% complete, ETA 00:00:28
# Fork: 3 of 5
# Warmup Iteration   1: 0.467 ops/s
Iteration   1: 0.411 ops/s

# Run progress: 60.00% complete, ETA 00:00:19
# Fork: 4 of 5
# Warmup Iteration   1: 0.470 ops/s
Iteration   1: 0.460 ops/s

# Run progress: 80.00% complete, ETA 00:00:09
# Fork: 5 of 5
# Warmup Iteration   1: 0.462 ops/s
Iteration   1: 0.474 ops/s


Result "com.example.PSTest.testForEach":
  0.451 ±(99.9%) 0.099 ops/s [Average]
  (min, avg, max) = (0.411, 0.451, 0.474), stdev = 0.026
  CI (99.9%): [0.352, 0.550] (assumes normal distribution)


# Run complete. Total time: 00:00:47

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark            Mode  Cnt  Score   Error  Units
PSTest.testForEach  thrpt    5  0.451 ± 0.099  ops/s
```

最后为测试报告

```text
Benchmark            Mode  Cnt  Score   Error  Units
PSTest.testForEach  thrpt    5  0.451 ± 0.099  ops/s
```

### 问题

运行测试类，如果遇到下面的错误：

```java
ERROR: org.openjdk.jmh.runner.RunnerException: ERROR: Exception while trying to acquire the JMH lock (C:\WINDOWS\/jmh.lock): C:\WINDOWS\jmh.lock (拒绝访问。), exiting. Use -Djmh.ignoreLock=true to forcefully continue.
    at org.openjdk.jmh.runner.Runner.run(Runner.java:216)
    at org.openjdk.jmh.Main.main(Main.java:71)
```

这个错误是因为JMH运行需要访问系统的TMP目录，解决办法是：

打开RunConfiguration -&gt; Environment Variables -&gt; include system environment viables

