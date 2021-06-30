# 6 NIO和BIO、AIO

## BIO

Block-IO：InputStream、OutputStream，Reader和Writer（Tomcat7.0是BIO）

* 阻塞式
* 面向流
* 线程多
* 性能差

![image-20201129013017809](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/Java/images/image-20201129013017809.png)

## NIO

NonBlock-IO：构建**多路复用**的、同步非阻塞的IO操作。例如：Input Output 文件系统、网络服务、内存（Tomcat8.0是NIO）

* 非阻塞式
* 面向管道\(双向\)
* 线程少
* 性能好

![image.png](../../.gitbook/assets/nio.png)

### NIO核心

* Channel：类似流
  * FileChannel
  * DatagramChannel
  * SocketChannel
  * ServerSocketChannel
* Buffer
  * ByteBuffer
  * CharBuffer
  * DoubleBuffer
  * FloatBuffer
  * IntBuffer
  * LongBuffer
  * ShortBuffer
  * MappedByteBuffer
* Selector：允许单线程处理多个Channel

### IO多路复用

调用系统级别的select  poll  epoll

![image-20201130000929554](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/Java/images/image-20201130000929554.png)

![image-20201130001008646](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/Java/images/image-20201130001008646.png)

![image-20201130001036160](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/Java/images/image-20201130001036160.png)

## AIO

Asynchronous IO：基于事件和回调机制

![image-20201130001948616](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🍰编程/Java/images/image-20201130001948616.png)

