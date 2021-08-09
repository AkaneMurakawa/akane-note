# 4 重要的日志

详细内容见\[MySQL架构.pdf\] 和MySQL实战45讲 -林晓斌 网名丁奇，前阿里资深技术专家

在 MySQL 里，如果每一次的更新操作都需要写进磁盘，然后磁盘也要找到对应的那条记录，然后再更新，整个过程 IO 成本、查找成本都很高。为了解决这个问题，MySQL 的设计者就用了类似**酒店掌柜粉板**的思路来提升更新效率。

> 一个有趣而通俗的例子，你是一个酒店老板，来的客户点餐需要记账。一开始你只记在账本里，然后发现人多的时候就忙不过来了。
>
> 这时候，你加了一块黑板，先记在黑板上，等有空的时候再记到账本里。

## WAL \(Write-Ahead-Logging\)

MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging，**它的关键点就是先写日志，再写磁盘**，也就是先写粉板，等不忙的时候再写账本。

## 重要日志

* Redo Log（重做日志）
* Bin Log （归档日志）
* Undo Log

### Redo Log（重做日志）

就像上面例子中的黑板，黑板的大小是固定，与此类似Redo Log是**固定大小**的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，**是循环写的过程.**

![redolog.png](../../.gitbook/assets/redolog.png)

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

### Bin Log（归档日志）

就像上面例子中的账本，记录的比较详细，是server层的日志。

binlog 有两种格式，**一种是 statement，一种是 row**。可能你在其他资料上还会看到有第三种格式，叫作 mixed，其实它就是前两种格式的混合。

* statement 格式，binlog 里面记录的就是 SQL 语句的原文，甚至连注释都一起记录
* row格式会记录行的内容，记两条，更新前和更新后都有

**delete加limit问题**

* 当binlog\_format 是statement格式时，delete 带 limit，很可能会出现**主备数据不一致**的情况
* 当 binlog\_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题。

为什么会有 mixed 格式的 binlog？

* 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
* 但是row格式占用空间大。
* **mixed 格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致**，如果有可能，就用 row 格式，否则就用 statement 格式。

### Bin Log与Redo Log区别

1. redo是**innodb**独有的，binlog是所有引擎都可以使用的
2. redo是物理日志，记录的是在某个数据页上**做了什么修改**，binlog是逻辑日志，记录的是这个语句的**原始逻辑**
3. redo是**循环**写的，空间会用完，binlog是可以**追加**写的，不会覆盖之前的日志信息

### 两阶段提交

![&#x4E24;&#x9636;&#x6BB5;&#x63D0;&#x4EA4;.png](../../.gitbook/assets/两阶段提交.png)

### Undo Log

Undo Log是为了实现事务的原子性，在MySQL数据库InnoDB存储引擎中，还用Undo Log来实现多版本并发控制\(简称：MVCC\)

注意：undo log是逻辑日志，可以理解为：

– 当delete一条记录时，undo log中会记录一条对应的insert记录

– 当insert一条记录时，undo log中会记录一条对应的delete记录

– 当update一条记录时，它记录一条对应相反的update记录

数据行里的DB\_TRX\_ID、DB\_ROLL\_PTR、DB\_ROW\_ID

![image-20201129003107999](../../.gitbook/assets/image-20201129003107999.png)

### 数据页

InnoDB 的数据是按**数据页**为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。在 InnoDB 中，每个数据页的大小默认是 16KB。

## change buffer

当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InnoDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

虽然名字叫作 change buffer，实际上它是可以持久化的数据。也就是说，change buffer 在内存中有拷贝，也会被写入到磁盘上。

**那么，什么条件下可以使用 change buffer 呢？**

对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。比如，要插入 \(4,400\) 这个记录，就要先判断现在表中是否已经存在 k=4 的记录，而这必须要将数据页读入内存才能判断。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 change buffer 了。

**因此，唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。**

change buffer 用的是 buffer pool 里的内存，因此不能无限增大。change buffer 的大小，可以通过参数 **innodb\_change\_buffer\_max\_size** 来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

**内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。**

回到文章开头的问题，你不难想象，平时执行很快的更新操作，其实就是在写内存和日志，而 MySQL 偶尔“抖”一下的那个瞬间，**可能就是在刷脏页（flush**）。

* 第一种场景是，粉板满了，记不下了。这个场景，对应的就是 InnoDB 的 **redo log 写满了**。这时候系统会停止所有更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写。
* 第二种场景是，这一天生意太好，要记住的事情太多，掌柜发现自己快记不住了，赶紧找出账本把孔乙己这笔账先加进去。**对应的就是系统内存不足。**
* 第三种场景是，生意不忙的时候，或者打烊之后。这时候柜台没事，掌柜闲着也是闲着，不如更新账本。对应的就是 **MySQL 认为系统“空闲”**的时候。
* 第四种场景是，年底了咸亨酒店要关门几天，需要把账结清一下。对应的就是 **MySQL 正常关闭**的情况。

## binlog 的写入机制

其实，binlog 的写入逻辑比较简单：事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。

事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。

![binlog.png](../../.gitbook/assets/binlog.png)

可以看到，每个线程有自己 binlog cache，但是共用同一份 binlog 文件。

* 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。
* 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘的 IOPS。

write 和 fsync 的时机，是由参数 sync\_binlog 控制的：

* sync\_binlog=0 的时候，表示每次提交事务都只 write，不 fsync；
* sync\_binlog=1 的时候，表示每次提交事务都会执行 fsync；
* sync\_binlog=N\(N&gt;1\) 的时候，表示每次提交事务都 write，但累积 N 个事务后才 fsync。

因此，在出现 IO 瓶颈的场景里，将 sync\_binlog 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值。

但是，将 sync\_binlog 设置为 N，对应的风险是：如果主机发生异常重启，**会丢失最近 N 个事务的 binlog 日志**。

## redo log 的写入机制

事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 的。

为了控制 redo log 的写入策略，InnoDB 提供了 **innodb\_flush\_log\_at\_trx\_commit** 参数，它有三种可能取值：

* 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中 ;
* 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘；
* 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache。

## 双“1”配置

通常我们说 MySQL 的“双 1”配置，指的就是 sync\_binlog 和 innodb\_flush\_log\_at\_trx\_commit 都设置成 1。

也就是说，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog。

你可以用以下命令查看，默认是双1。

```text
 show variables like 'innodb_flush_log_at_trx_commit';
 show variables like 'sync_binlog';
```

但在这里我查看到生产上数据库，innodb\_flush\_log\_at\_trx\_commit设置为2，sync\_binlog会设置为1000。这是为什么呢？在【MySQL实战45讲 -林晓斌】23章中便解释了这个。

什么时候线上生产库设置成""非双1"

1. 业务高峰期，一般设置为非双1，是设置成innodb\_flush\_log\_at\_trx\_commit = 2，sync\_binlog=1000
2. 备库延迟
3. 批量导入数据时候

