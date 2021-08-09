# 2 索引

## 前言

开发过程中免不了会使用到索引，今天就来和大家探讨一下数据库中的索引，便于大家能够更加了解索引、合理的设计和使用索引。

## 为什么会有索引

我们查阅东西的时候，总是会通过关键字去查询，想想如果不能通过关键字，而是要你一页一页去查询会怎么样，那还不得累死。

在我们看书的时候，书一般都会有一个目录，通过目录我们可以快速找到我们想要的章节。有了目录，我们就能够快速的查询。

## 索引是越多越好吗

**这个目录就相当于索引，这就是索引的目的，加快查询。但是有了目录我们打印书籍的时候，就得多打印几页，这就是索引额外的磁盘开销。**

* 数据量小的不需要建立索引，建立会增加磁盘开销
* 更多的索引意味着更多的空间

## 索引的常见模型

在了解索引的好处后，我们来看看索引的模型。我们知道，数据最终是要存储的，那么索引又是如何存储的呢？了解索引的存储模型，有利于我们合理设计索引。

我们常见的基本数据类型，例如：int、long等都是存储简单的数据的。而要存储庞大的数据，就需要一种特殊的数据结构，树或Hash。

对于数据结构感兴趣的，可以看这个可视化的数据结构网站：[https://www.cs.usfca.edu/~galles/visualization/Algorithms.html](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)

比较常见的索引数据类型：

* Hash
* B+Tree
* BitMap（Oracle支持，MySQL支持）

**索引数据结构选择**

说明，以下图是通过[https://www.cs.usfca.edu/~galles/visualization/Algorithms.html制作的。](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html制作的。)

**索引必须是有序**，现在我们希望有一种数据结构，能够减少磁盘读取，而且又能快速查询，支持范围查询。

### 1、Hash

Hash是利用了数组快速查询的特点，但缺点是**不能范围查询，不能排序**， 并且数据重复会产生链表。

![image.png](../../.gitbook/assets/hash.png)!\[

### 2、二叉树排序树

缺点：由于左值&lt;=根，右值&gt;=根，如果是单一的数据，树可能会太高，增加读取磁盘的次数。

![image.png](../../.gitbook/assets/二叉树.png)

### 3、红黑树

红黑树比起二叉树，改进的一点就是会自动平衡，提高查询性能。**红黑树变色目的，减少自旋次数，自旋影响插入效率。**\( 每一条路径，**黑节点的个数**是一样的\)

但是由于数据量上来时，还是会导致树太高。

![image.png](../../.gitbook/assets/红黑树.png)

![image.png](../../.gitbook/assets/红黑树2.png)

### 4、B Trees

与前面不同的是，B Trees是一个N叉树，这样就有效减少了我们磁盘的读取次数。

缺点是：

1. 每个节点都有key，同时也有data。**而每个页存储空间是有限的**，如果data比较大的话会导致每个节点存储的key数量变小
2. （磁盘扫描的时候，都是以4kb为单位进行扫描，我们在查看文件属性的时候，可以看到是一个**大小**，还有一个是**占用空间**）
3. 当存储的数据量很大的时候，同样会导致深度较大，增加查询时磁盘IO次数，影响性能。

![image.png](../../.gitbook/assets/数据结构选择.png)

### 5、B+ Trees

B+ Trees是对B Trees的改进，**非叶子节点只存索引，不存数据，增加存储空间，降低树的高度**。例如：查找10，可以即使找到了10，但是是非叶子节点，因此还需要继续查找。必须查到叶子节点才会有数据

可以看到，每个叶子节点都指向相邻的叶子节点的地址（顺序查询性能更高），这样有利于做范围查询。

![image.png](../../.gitbook/assets/B+%20tree.png)

B+Trees是BTrees的一种变种。B+Trees和BTrees的不同主要在于：

* B+Tree中的非叶子结点不存储数据，**只存储键值（索引）**；
* B+Tree的叶子结点没有指针，所有键值都会出现在叶子结点上，且key存储的键值对应data数据的物理地址；
* B+Tree的每个非叶子节点由n个键值key和n个指针point组成；
* 所有叶子节点均有一个链指针**指向下一个**叶子节点

## MySQL索引知识点介绍

### InnoDB

* 若一个主键被定义，该主键则作为密集索引
* 若没有主键被定义，该表的第一个唯一非空索引则作为密集索引
* 若不满足以上条件，InnoDB内部会生成一个隐藏主键（密集索引）
* 非主键索引存储相关键位和其对应的主键值，包含两次查找

### 索引分类

主键索引、唯一索引、普通索引、全文索引、联合索引

### 数据页

InnoDB 的数据是按**数据页**为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。在 InnoDB 中，每个数据页的大小默认是 16KB。

通俗来说：MySQL将记录不是写成一个很大的文件，并不是像我们介绍B+Trees结构一样，而是以数据页为单位，每个数据页默认大小是16KB。写完了就新增一个新的数据页。

### 索引维护——索引合并、索引列分裂和页合并

![l1.png](../../.gitbook/assets/l1.png)

B+ 树为了维护索引有序性，在**插入新值**的时候需要做必要的维护。以上面这个图为例，如果插入新的行 ID 值为 700，则只需要在 R5 的记录后面插入一个新记录。如果新插入的 ID 值为 400，就相对麻烦了，需要逻辑上挪动后面的数据，空出位置。

* **页分裂**：而更糟的情况是，如果 R5 所在的数据页已经满了，根据 B+ 树的算法，这时候**需要申请一个新的数据页**，然后挪动部分数据过去。这个过程称为页分裂。在这种情况下，性能自然会受影响。

  除了性能外，页分裂操作还**影响数据页的利用率**。原本放在一个页的数据，现在分到两个页中，整体空间利用率降低大约 50%。

* **页合并**：当然有分裂就有合并。当相邻两个页由于**删除了数据，利用率很低之后，会将数据页做合并**。合并的过程，可以认为是分裂过程的逆过程。

### 聚簇索引和非聚簇索引、 唯一索引和普通索引、二级索引

* 聚簇索引：不是单独的索引类型，而是指数据存储方式。叶子节点**存的是整行数据**。在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。 
* 非聚簇索引：叶子节点**内容是主键的值**，不直接存储数据。在 InnoDB 里，非主键索引也被称为二级索引（secondary index）。

![image-20201128175838166](../../.gitbook/assets/image-20201128175838166.png)

InnoDB 采用聚集\(聚簇\)索引，MyISAM 采用非聚集索引。我们也可以通过下图MySQL的数据存储划分结果得知。

说明：.frm是表的结构定义，在MyISAM中，MYD是数据的存储，MYI是索引的存储。在InnoDB中，.ibd包含了数据和索引的存储。

其实，只要看数据和索引是否存在一起，便能判断。

![image.png](../../.gitbook/assets/file1.png)

### 回表

我们知道，在InnoDB中采用的是聚簇索引方式，主键索引中，叶子节点存放的是所有数据。而非主键索引（二级索引）叶子节点存放的是主键的值。

假如有以下查询：

* 如果语句是 select \* from T where ID=500，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；
* 如果语句是 select  _from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。\*这个过程称为回表_。

如果查询的值已经在索引上了，则不需要再回表查询。

### 联合索引 与 覆盖索引

如果所有要**查询的值**已经在索引上了，不需要再**回表**查询，则称为**覆盖索引**。

由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。

```text
CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_id_card_name` (`id_card`,`name`)
) ENGINE=InnoDB

-- 
 select id_card,name from tuser where id_card='xxxx'
```

**当extra有using index时，表示用到了覆盖索引**

```text
mysql> explain select store_id,film_id from inventory\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: inventory
   partitions: NULL
         type: index
possible_keys: NULL
          key: idx_store_id_film_id
      key_len: 3
          ref: NULL
         rows: 4581
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.01 sec)
```

### 前缀索引

可以匹配某一列的开头部分

有时候需要索引很长的字符串，这会让索引变的大且慢，通常情况下可以使用某个列开始的部分字符串，这样大大的节约索引空间，从而提高索引效率，**但这会降低索引的选择性**，索引的选择性是指不重复的索引值和数据表记录总数的比值，范围从1/\#T到1之间。索引的选择性越高则查询效率越高，因为选择性更高的索引可以让mysql在查找的时候过滤掉更多的行。

一般情况下某个列前缀的选择性也是足够高的，足以满足查询的性能，但是对应BLOB,TEXT,VARCHAR类型的列，必须要使用前缀索引，因为mysql不允许索引这些列的完整长度，使用该方法的诀窍在于要选择足够长的前缀以保证较高的选择性，通过又不能太长。

**选择合适的前缀索引**

可以通过字段 占用 总数的比例来选前缀索引的长度。我们可以先选择一个初始值，然后一个字符一个字符的增加，判断结果是否有明显变化。

如：下面5、6时差距挺大的，而选择7时候变化已经不大了，因此可以选择前缀索引长度为6

```text
select count(DISTINCT left(code, 5)) / count(*) from translation; -- 0.7
select count(DISTINCT left(code, 6)) / count(*) from translation; -- 0.865
select count(DISTINCT left(code, 7)) / count(*) from translation; -- 0.875
```

前面是选择合适的前缀索引。当然，反之一个索引建立了，你可以用以下命令查看索引的区分度

```text
show index from t;
```

一个索引上不同的值的个数，我们称之为“**基数”（cardinality**）。也就是说，**这个基数越大，索引的区分度越好。**

**创建前缀索引**

使用前缀索引后，可能会导致查询语句读数据的次数变多。使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。

```text
-- 指定索引长度
mysql> alter table translation add index idx_code(email(6));
```

### 索引下推

MySQL 5.6 引入的索引下推优化（index condition pushdown\)， 可以在索引遍历过程中，**对索引中包含的字段先做判断，注意是索引中包含有这个字段才行**，直接过滤掉不满足条件的记录，减少回表次数。

```text
mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
```

对于user\_table表，我们现在有（name,age）联合索引

![image.png](../../.gitbook/assets/file2.png)

InnoDB 在 \(name,age\) 索引内部就判断了 age 是否等于 10，对于不等于 10 的记录，直接判断并跳过。在我们的这个例子中，只需要对 ID4、ID5 这两条记录回表取数据判断，就只需要回表 2 次。

### 最左前缀原则

最左优先，使用函数不会走索引。

比如有\(a, b\)联合索引，会对a先排序，再对b排序存储

### 强制使用索引force index

有时候MySQL会选错索引，可以使用force index强制指定索引。但是通常不建议使用force index，因为我们无法预知业务的变化，有可能随着业务的变化不适用了，我们应该相信MySQL的优化能力。

```text
explain select * from itdragon_order_list force index(idx_order_levelDate) order by order_level,input_date;
```

### 为什么会选错索引？

1. 有时候某个执行计划虽然需要读取更多的页面，但是他的成本却更小，因为如果这些页面都是顺序读或者这些页面都已经在内存中的话，那么它的访问成本将很小，mysql层面并不知道哪些页面在内存中，哪些在磁盘，所以查询之际执行过程中到底需要多少次IO是无法得知的
2. mysql的优化是基于成本模型的优化，但是有可能不是最快的优化
3. mysql不考虑其他并发执行的查询
4. mysql不会考虑不受其控制的操作成本

### 删除索引 - 重建索引

**为什么表数据删掉一些，表文件大小不变？删除了索引，表文件大小还是不变？**

在InnoDB 引擎里

1. 虽然删除了表的部分记录，但是它的索引还在, 并未释放。
2. 不论是删除主键还是创建主键，都会将整个表重建。而重新建表才能重建索引。

**你可以使用以下命令重建索引**

```text
alter table T engine=InnoDB
```

假设我有一张 test\_index 表，并建立 idx\_a\_b\_c 联合索引

```text
CREATE TABLE `test_index` (
  `id` int(11) NOT NULL,
  `a` int(32) DEFAULT NULL,
  `b` int(32) DEFAULT NULL,
  `c` int(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_a_b_c` (`a`,`b`,`c`)
) ENGINE=InnoDB;

insert into test_index values(1,1,1,1);
insert into test_index values(2,2,2,2);
insert into test_index values(3,3,3,3);
insert into test_index values(4,4,4,4);
insert into test_index values(5,5,5,5);
insert into test_index values(6,6,6,6);
insert into test_index values(7,7,7,7);
insert into test_index values(8,8,8,8);
insert into test_index values(9,9,9,9);
insert into test_index values(10,10,10,10);
```

现在我们查看一下当前 test\_index 表里面的数据大小

![image.png](../../.gitbook/assets/file3.png)

接下里，我们试着去删除数据 和 索引，发现文件大小并没有什么变化

```text
DELETE from test_index;

ALTER TABLE `test_index`
DROP INDEX `idx_a_b_c`;
```

![image.png](../../.gitbook/assets/file4.png)

当我们执行以下语句时，文件终于变小了

```text
alter table T engine=InnoDB
```

![image.png](../../.gitbook/assets/file5.png)

注：当DBA帮我们重建索引的时候，你会看到，有时候DBA会重新建立一张表，先关闭索引，然后把数据复制过去，现在你知道是什么原因了吧。

### 代理主键和自然主键

代理主键：与业务关务

自然主键：与业务有关

### 索引失效

使用函数不会走索引

* 隐式类型转换。例如：索引是int id，结果写成了id ='1'。MySQL会进行隐式类型转换。
* 隐式字符编码转换，例如一张表用utf8，另一张表用utf8mb4，join时候就会索引失效，因为MySQL会隐式字符编码转换
* 其它使用了函数转换
* 不符合最左匹配原则
* 使用了不等号，范围查询

## 优化

1. **更新频繁**（删除或更新可能造成页分裂和页合并），区分度不高的，不宜建索引
2. 使用函数不会走索引
3. 区分度高的放最前面，最左前缀原则
4. 如果明确知道只有一条结果返回，limit能够提高效率（减少判断）
5. 推荐使用代理主键，自然主键与业务耦合，不适合维护
6. 可以使用hash、或者字符串转数字（如使用ip转数字函数）来存储索引字段，占用空间更少

## 测试

现在我们有一张 test\_index 表，为了使用范围查询，这里我将类型设置为int。

```text
CREATE TABLE `test_index` (
  `id` int(11) NOT NULL,
  `a` int(32) DEFAULT NULL,
  `b` int(32) DEFAULT NULL,
  `c` int(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_a_b_c` (`a`,`b`,`c`)
) ENGINE=InnoDB;

insert into test_index values(1,1,1,1);
insert into test_index values(2,2,2,2);
insert into test_index values(3,3,3,3);
insert into test_index values(4,4,4,4);
insert into test_index values(5,5,5,5);
```

**范围查询 type： range**

[range](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_range) can be used when a key column is compared to a constant using any of the [=](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal), [&lt;&gt;](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_not-equal), [&gt;](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than), [&gt;=](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_greater-than-or-equal), [&lt;](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than), [&lt;=](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_less-than-or-equal), [IS NULL](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_is-null), [&lt;=&gt;](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to), [BETWEEN](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between), [LIKE](https://dev.mysql.com/doc/refman/5.7/en/string-comparison-functions.html#operator_like), or [IN\(\)](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_in) operators:

```text
mysql> explain select * from test_index where a > 1 and b > 2;
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_index | NULL       | range | idx_a_b_c     | idx_a_b_c | 5       | NULL |    4 |    33.33 | Using where; Using index |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
1 row in set (0.04 sec)
```

**最左优先 type：index**

```text
mysql> explain select * from test_index where b = 2;
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_index | NULL       | index | NULL          | idx_a_b_c | 15      | NULL |    5 |    20.00 | Using where; Using index |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
1 row in set (0.04 sec)
```

**idx\_a\_b\_c索引中b出现范围查询**

```text
mysql> explain select * from test_index where a = 2 and b > 1 and c = 2;
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
| id | select_type | table      | partitions | type  | possible_keys | key       | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | test_index | NULL       | range | idx_a_b_c     | idx_a_b_c | 10      | NULL |    1 |    20.00 | Using where; Using index |
+----+-------------+------------+------------+-------+---------------+-----------+---------+------+------+----------+--------------------------+
1 row in set (0.05 sec)
```

**and时，前后顺序无关，MySQL优化器会进行优化**

**唯一索引对应的type是const**，通过索引一次就可以找到结果，你可以理解为常量。

普通索引对应的type是ref，表示非唯一性索引扫描，找到值还要进行扫描，直到将索引文件扫描完为止，显而易见，const的性能要高于ref

```text
mysql> explain select * from test_index where a = 2 and b = 2;
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | test_index | NULL       | ref  | idx_a_b_c     | idx_a_b_c | 10      | const,const |    1 |   100.00 | Using index |
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
1 row in set (0.04 sec)

mysql> explain select * from test_index where b = 2 and a = 2;
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
| id | select_type | table      | partitions | type | possible_keys | key       | key_len | ref         | rows | filtered | Extra       |
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | test_index | NULL       | ref  | idx_a_b_c     | idx_a_b_c | 10      | const,const |    1 |   100.00 | Using index |
+----+-------------+------------+------------+------+---------------+-----------+---------+-------------+------+----------+-------------+
1 row in set (0.05 sec)
```

## 选读 - EXPLAIN Output Format（Query Execution Plan）

参考官方文档：[https://dev.mysql.com/doc/refman/5.7/en/explain-output.html\#explain\_rows](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain_rows)

示例说明：[explain执行计划.md](https://github.com/AkaneMurakawa/akane-note/tree/83689663b4c2a2a0c5a5afb1d9dea3c90e87b904/🌎数据库/MySQL/explain执行计划.md)

> The EXPLAIN statement provides information about how MySQL executes statements:
>
> EXPLAIN works with SELECT, DELETE, INSERT, REPLACE, and UPDATE statements.

一般来说，得保证查询至少达到range级别，最好能达到ref。**一张表一次查询最多用到一个索引。**

一般我们只需要记住几个即可：sytem &gt; conf &gt; ref &gt; range &gt; index &gt; all

优化：

* 如果type为index或者all就可能需要优化了
* 如果出现了using filesort 或using temporary也可能需要优化了

| id | SELECT的查询序列号 |
| :--- | :--- |
| select\_type | SELECT类型  SIMPLE: 简单SELECT\(不使用UNION或子查询\)  PRIMARY: **最外面的SELECT**  UNION: UNION中的第二个或后面的SELECT语句  DEPENDENT UNION: UNION中的第二个或后面的SELECT语句，取决于外面的查询  UNION RESULT: UNION 的结果  SUBQUERY: 子查询中的第一个SELECT  DEPENDENT SUBQUERY: 子查询中的第一个SELECT,取决于外面的查询  DERIVED: 派生表\(FROM子句的子查询，derived: 派生\)  MATERIALIZED: 被物化的子查询  UNCACHEABLE SUBQUERY: 一个子查询，其结果不能被缓存，必须为外部查询的每一行重新计算  UNCACHEABLE UNION: UNION操作中，内层的不可被物化的子查询\(类似于UNCACHEABLE SUBQUERY\) |
| table | 表名 |
| partitions | 分区 |
| type | 联表类型\(join tyep\)下面按照从 最佳类型 到 最坏类型 进行排序:  system: 表仅有一行\(=系统表\)。这是const联接类型的一个特例  const: **表最多有一个匹配行**，该行在查询开始时读取。因为只有一行，所以该行中列的值可以被优化器的**其余部分视为常量**。const表非常快，因为它们只读取一次   eq\_ref: 对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型  ref: 对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取 fulltext: 执行FULLTEXT索引  ref\_or\_null: 该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行  index\_merge: 该联接类型表示使用了索引合并优化方法  unique\_subquery: 该类型替换了下面形式的IN子查询的 eq\_ref: value IN \(SELECT primary\_key FROM single\_table WHERE some\_expr\) unique\_subquery是一个索引查找函数，可以完全替换子查询，效率更高  index\_subquery: 该联接类型类似于unique\_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引: value IN \(SELECT key\_column FROM single\_table WHERE some\_expr\)  range: 只检索给定范围内的行，使用一个索引来选择行  index: 该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小  ALL: 对于每个来自于先前的表的行组合，进行全表扫描 |
| possible\_keys | MySQL可以使用哪个索引，但并不一定会使用 \(查看表中包含的索引：SHOW INDEX FROM **tbl\_name**.\) |
| key | 显示MySQL实际决定使用的键\(索引\)。如果没有选择索引，键是NULL |
| key\_len | 显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL |
| ref | 显示使用哪个列或常数与key一起从表中选择行 |
| rows | 显示MySQL认为它执行查询时必须检查的行数。多行之间的数据相乘可以估算要处理的行数。 |
| filtered | 显示了通过条件过滤出的行数的百分比估计值 |
| Extra | MySQL解决查询的详细信息  Distinct: MySQL发现第1个匹配行后，停止为当前的行组合搜索更多的行  Not exists: MySQL能够对查询进行LEFT JOIN优化，发现1个匹配LEFT JOIN标准的行后，不再为前面的的行组合在该表内检查更多的行 range checked for each record \(index map: \#\): MySQL没有发现好的可以使用的索引，但发现如果来自前面的表的列值已知，可能部分索引可以使用  **Using filesort: MySQL需要额外的一次传递，以找出如何按排序顺序检索行 。MySQL无法利用索引完成的排序称为“文件排序”** Using index: 从只使用索引树中的信息而不需要进一步搜索读取实际的行来检索表中的列信息  **Using temporary: 为了解决查询，MySQL需要创建一个临时表来容纳结果。常见于排序order by和分组group by**  Using where: WHERE 子句用于限制哪一个行匹配下一个表或发送到客户  Using sort\_union\(...\), Using union\(...\), Using intersect\(...\): 这些函数说明如何为index\_merge联接类型合并索引扫描  Using index for group-by: 类似于访问表的Using index方式，Using index for group-by表示MySQL发现了一个索引，可以用来查询GROUP BY 或 DISTINCT查询的所有列，而不要额外搜索硬盘访问实际的表 |

