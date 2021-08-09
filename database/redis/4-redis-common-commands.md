# 4 Redis常用命令

[https://www.runoob.com/redis/redis-keys.html](https://www.runoob.com/redis/redis-keys.html)

## 下载安装

```text
$ sudo apt update $ sudo apt install redis-server
```

## 自定义安装

```text
下载tar.gz文件，解压
make
make install
```

## 启动关闭服务

```text
查看服务运行状态：
$ sudo systemctl status redis
$ ps -ef | grep redis

启动 redis 服务：
$ sudo service redis start

关闭 redis 服务：
$ sudo service redis stop

重启 redis 服务：
$ sudo service redis restart

查看服务，也可以知道服务启动位置
akane@ubuntu:/usr/bin$ ps -ef | grep redis
redis       920      1  0 06:21 ?        00:00:00 /usr/bin/redis-server 127.0.0.1:6379
akane      2409   2335  0 06:24 pts/0    00:00:00 redis-cli
akane      2657   2642  0 06:30 pts/1    00:00:00 grep --color=auto redis

查看配置文件配置
akane@ubuntu:/usr/bin$ whereis redis.conf
redis: /etc/redis

默认RDB文件所在目录
# The filename where to dump the DB
dbfilename dump.rdb

# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /var/lib/redis
```

## 启动和连接、安全

```text
启动服务：
$ sudo redis-server /etc/redis/redis.conf
(注: 和上面不同的是，可以指定配置文件，配置文件中也有说明这种启动方式)

客户端连接：
$ redis-cli -h host -p port -a password
例如：
$ redis-cli -h 127.0.0.1 -p 6379

测试连接:
127.0.0.1:6379> ping
PONG

关闭：
127.0.0.1:6379>shutdown

安全
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> config set requirepass "123456"
OK
127.0.0.1:6379> ping
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth 123456
OK
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> config set requirepass ""
OK
127.0.0.1:6379> ping
PONG
127.0.0.1:6379>
```

## 性能测试

```text
akane@ubuntu:/usr/bin$ redis-benchmark 
====== PING_INLINE ======
  100000 requests completed in 1.31 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1

99.02% <= 1 milliseconds
99.89% <= 2 milliseconds
99.95% <= 4 milliseconds
99.96% <= 5 milliseconds
99.99% <= 6 milliseconds
99.99% <= 7 milliseconds
100.00% <= 8 milliseconds
100.00% <= 9 milliseconds
76277.65 requests per second
```

## 数据库

```text
返回当前数据库的 key 的数量：
127.0.0.1:6379>DBSIZE

数据库切换(0 ~ databases-1)：
SELECT <dbid>
例如：
127.0.0.1:6379>SELECT 0 
将key移动到数据库
127.0.0.1:6379>move key db

清除当前数据库
127.0.0.1:6379>flushdb
清除所有数据库
127.0.0.1:6379>flushall
```

## 获取配置信息

```text
127.0.0.1:6379> config get /loglevel
127.0.0.1:6379> config get maxmemory
```

## 基本数据类型

```text
查找key：
127.0.0.1:6379>keys  *

127.0.0.1:6379>set name akane
127.0.0.1:6379>get name

是否存在key：
127.0.0.1:6379>exists name
设置key过期时间：
127.0.0.1:6379>expire key seconds
以时间戳的形式设置过期时间：
127.0.0.1:6379>expireat key timestamp
查看多久过期(ttl: time to live)：
127.0.0.1:6379>ttl key
移除过期：
127.0.0.1:6379>persist name
删除key：
127.0.0.1:6379>del key
获取key类型
127.0.0.1:6379>type key

127.0.0.1:6379> set dump1 v1
OK
dump key 序列化key，并返回被序列化的值：
127.0.0.1:6379> dump dump1 
"\x00\x02v1\a\x00\xa0\xd7e\xad\xc3\x9a\xacA"

127.0.0.1:6379>rename key newKey
127.0.0.1:6379>renamenx key newKey
```

### String

使用场景：计数器、统计多单位的数量、粉丝数、对象缓存存储、分布式锁\(setnx\)

```text
127.0.0.1:6379> set name akane
OK
127.0.0.1:6379> get name
"akane"
127.0.0.1:6379> strlen name
(integer) 5
127.0.0.1:6379> append name murakawa
(integer) 13
127.0.0.1:6379> get name
"akanemurakawa"
127.0.0.1:6379> setnx  nx nx
(integer) 1
127.0.0.1:6379> get nx
"nx"
127.0.0.1:6379> setnx  nx nx2
(integer) 0
127.0.0.1:6379> get nx
"nx"

127.0.0.1:6379> set views 0
OK
127.0.0.1:6379> incr views
(integer) 1
127.0.0.1:6379> incr views
(integer) 2
127.0.0.1:6379> decr views
(integer) 1
127.0.0.1:6379> get views
"1"
127.0.0.1:6379> incrby views 5
(integer) 6
127.0.0.1:6379> get views
"6"

127.0.0.1:6379> getrange name 0 -1
"akanemurakawa"
127.0.0.1:6379> getrange name 0 5
"akanem"
127.0.0.1:6379> get name
"akanemurakawa"
127.0.0.1:6379> setrange name 0 A
(integer) 13
127.0.0.1:6379> get name
"Akanemurakawa"

127.0.0.1:6379> setex name2 100 name2
OK
127.0.0.1:6379> ttl name2
(integer) 97
127.0.0.1:6379> ttl name2
(integer) 90

Redis中巧妙的设计
127.0.0.1:6379> mset user:1:name v1 user:2:age v2
127.0.0.1:6379> mset k1 v1 k2 v2
OK
127.0.0.1:6379> mget k1 k2
1) "v1"
2) "v2"
127.0.0.1:6379> msetnx是一个原子性操作，要么成功，要么失败

127.0.0.1:6379> getset name3 name3
(nil)
127.0.0.1:6379> get name3
"name3"
```

![image-20201129013744378](../../.gitbook/assets/image-20201129013744378.png)

### Hash

使用场景：和String差不多，但更适合存**对象**

```text
127.0.0.1:6379> hset myhash k1 v1
(integer) 1
127.0.0.1:6379> hset myhash k2 v2
(integer) 1
127.0.0.1:6379> hget myhash k1
"v1"
127.0.0.1:6379> hmset myhash k3 v3 k4 v4
OK
127.0.0.1:6379> hgetall myhash
1) "k1"
2) "v1"
3) "k2"
4) "v2"
5) "k3"
6) "v3"
7) "k4"
8) "v4"
127.0.0.1:6379> hdel myhash k4
(integer) 1
127.0.0.1:6379> hgetall myhash
1) "k1"
2) "v1"
3) "k2"
4) "v2"
5) "k3"
6) "v3"
127.0.0.1:6379> hlen myhash
(integer) 3
127.0.0.1:6379> hexists myhash k1
(integer) 1
127.0.0.1:6379> hkeys myhash
1) "k1"
2) "k2"
3) "k3"
127.0.0.1:6379> hvals myhash
1) "v1"
2) "v2"
3) "v3"
```

### List

使用场景：默认是**先进后出**的方式。可用作堆、队列、阻塞队列。最新消息排行榜

```text
127.0.0.1:6379> lpush list one
(integer) 1
127.0.0.1:6379> lpush list two
(integer) 2
127.0.0.1:6379> lrange list 0 1
1) "two"
2) "one"
127.0.0.1:6379> llen list
(integer) 2
127.0.0.1:6379> lpop list
"two"
127.0.0.1:6379> lpush list three
(integer) 2
127.0.0.1:6379> lpush list four
(integer) 3
127.0.0.1:6379> lrange list 0 -1
1) "four"
2) "three"
3) "one"
127.0.0.1:6379> lindex list 1
"three"
127.0.0.1:6379> lrem list 1 one
(integer) 1
127.0.0.1:6379> lrange list 0 -1
1) "four"
2) "three"
127.0.0.1:6379> ltrim list 0 1
OK
127.0.0.1:6379> lrange list 0 -1
1) "four"
2) "three"
```

### Set

无序不重复集合，**可以方便的做集合的操作\(并集、交集、差集\)**

使用场景：微博共同关注、共同爱好、推荐好友

```text
127.0.0.1:6379> sadd myset one
(integer) 1
127.0.0.1:6379> sadd myset two
(integer) 1
127.0.0.1:6379> smembers myset
1) "two"
2) "one"
127.0.0.1:6379> sismember myset one
(integer) 1
127.0.0.1:6379> scard myset
(integer) 2
127.0.0.1:6379> srem myset one
(integer) 1
127.0.0.1:6379> smembers myset
1) "two"
127.0.0.1:6379> srandmember myset 1
1) "two"
127.0.0.1:6379> srandmember myset 2
1) "two"
127.0.0.1:6379> srandmember myset 4
1) "two"
127.0.0.1:6379> smembers myset
1) "two"
127.0.0.1:6379> sadd myset one
(integer) 1
127.0.0.1:6379> smembers myset
1) "two"
2) "one"
127.0.0.1:6379> srandmember myset 4
1) "two"
2) "one"
127.0.0.1:6379> srandmember myset 1
1) "two"
127.0.0.1:6379> srandmember myset 3
1) "two"
2) "one"
```

### Zset

有序不重复集合

使用场景：排行榜应用实现，取Top N

普通消息 1.重要消息 2.带权重进行判断

```text
(integer) 1
127.0.0.1:6379> zadd myzset 2 two
(integer) 1
127.0.0.1:6379> zrange myset 0 -1
(error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379> zrange myzset 0 -1
1) "one"
2) "two"
127.0.0.1:6379> zrangebyscore myzset -inf +inf
1) "one"
2) "two"
```

### Geospatial

底层是**Zset**，可以使用zset命令操作

使用场景：地理位置计算

```text
127.0.0.1:6379> geoadd china:city 11 11
(error) ERR wrong number of arguments for 'geoadd' command
127.0.0.1:6379> geoadd china:city 11 11 beijing
(integer) 1
127.0.0.1:6379> geoadd china:city 12 12 shenzhen
(integer) 1
127.0.0.1:6379> geoadd china:city 13 13 shanghai
(integer) 1
127.0.0.1:6379> geopos china:city beijing
1) 1) "10.99999934434890747"
   2) "10.9999991200151257"
127.0.0.1:6379> geodist china:city beijing shenzhen
"155725.9779"
127.0.0.1:6379> geodist china:city beijing shenzhen km
"155.7260"
127.0.0.1:6379> georadius china:key 110 30 100 km
(empty list or set)

127.0.0.1:6379> zrange china:city 0 -1
1) "beijing"
2) "shenzhen"
3) "shanghai"
```

### Hyperloglog

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素

本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

缺点：0.81%错误率

**什么是基数?**

比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数\(不重复元素\)为5。 基数估计就是在误差可接受的范围内，快速计算基数。

使用场景：网页的UV（一个人一天访问一个网站多次，但只算一个人）

```text
127.0.0.1:6379> pfadd mykey a b c d
(integer) 1
127.0.0.1:6379> pfcount mykey
(integer) 4
127.0.0.1:6379> pfadd mykey2 a e
(integer) 1
127.0.0.1:6379> pfmerge mykey3 mykey mykey2
OK
127.0.0.1:6379> pfcount mykey3
(integer) 5
```

### Bitmaps

使用场景：位存储，统计用户信息：活跃不活跃，登录未登录，打卡（两个状态）

```text
127.0.0.1:6379> setbit sign 0 1
(integer) 0
127.0.0.1:6379> setbit sign 1 1
(integer) 0
127.0.0.1:6379> setbit sign 2 1
(integer) 0
127.0.0.1:6379> setbit sign 3 0
(integer) 0
127.0.0.1:6379> setbit sign 4 0
(integer) 0
127.0.0.1:6379> getbit sign 3
(integer) 0
127.0.0.1:6379> getbit sign 2
(integer) 1
127.0.0.1:6379> bitcount sign
(integer) 3
```

### Redis数据类型实现原理

底层数据类型基础

* 简单动态字符串
* 链表
* 字典
* 跳跃表
* 整数集合
* 压缩列表
* 对象

## Redis 事务

* Redis单条命令具有原子性，但是事务不保证原子性。如果中间出现了失败，已经执行的不会回滚，后面的依然被执行
* Redis事务无隔离级别

Redis 事务可以一次执行多个命令，遵从以下原则：

* 批量操作在发送 EXEC 命令前被放入队列缓存。
* 收到 EXEC 命令后进入事务执行，事务中**任意命令执行失败，其余的命令依然被执行**。
* 在事务**执行过程**，**其他客户端**提交的**命令请求不会插入到事务**执行命令序列中。

关键字：事务开始multi、执行所有事务块exec，放弃事务discard

```text
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> keys *
QUEUED
127.0.0.1:6379> exec
1) OK
2) OK
3) 1) "k2"
   2) "k1"
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379>
```

### Redis乐观锁

Redis Watch 命令用于监视一个\(或多个\) key ，**如果在事务执行之前这个\(或这些\) key 被其他命令所改动，那么事务将被打断**

```text
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> set money 100
OK
127.0.0.1:6379> unwatch
OK
127.0.0.1:6379> watch money
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incrby money 10
QUEUED
执行失败：
127.0.0.1:6379> exec
(nil)

另外一个线程
127.0.0.1:6379> decrby money 50
(integer) 50
```

## Redis订阅

一种消息通信模式，关键字：public发布者、subscribe订阅者

缺点：消息的发布是无状态的，无法保证可达

解决：使用Kafka、rabbitmq

使用场景：

1. 实时消息系统
2. 频道当聊天室

```text
# 创建频道,用于接收
subscribe channel [channel ...]
# 发送频道
publish channel message
# 注：由于是不同的用户，需要打开两个客户端进行测试

订阅者
127.0.0.1:6379> subscribe wechat
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "wechat"
3) (integer) 1
1) "message"
2) "wechat"
3) "product"

发布者
127.0.0.1:6379> publish wechat "product"
(integer) 1
127.0.0.1:6379>
```

## Redis脚本

Redis 脚本使用 Lua 解释器来执行脚本

Lua脚本是类似Redis事务，有一定的原子性，不会被其他命令插队，可以完成一些Redis事务性的操作。

关键字：eval

```text
eval script numkeys key [key ...] arg [arg ...]
例如：
EVAL "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
```

## 批量执行命令

```text
$ cat cmds.txt
set foo1 bar1
set foo2 bar2
set foo3 bar3
......
$ cat cmds.txt | redis-cli
OK
OK
OK
...

相当于
$ redis-cli < cmds.txt
OK
OK
OK
...
```

## 重复执行

```text
// 间隔1s，执行5次，观察qps的变化
$ redis-cli -r 5 -i 1 info | grep ops
instantaneous_ops_per_sec:43469
instantaneous_ops_per_sec:47460
instantaneous_ops_per_sec:47699
instantaneous_ops_per_sec:46434
instantaneous_ops_per_sec:47216
```

## 监控服务器状态

```text
$ redis-cli --stat
------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections
2          6.66M    100     0       11591628 (+0)       335
2          6.66M    100     0       11653169 (+61541)   335
2          6.66M    100     0       11706550 (+53381)   335
2          6.54M    100     0       11758831 (+52281)   335
2          6.66M    100     0       11803132 (+44301)   335
2          6.66M    100     0       11854183 (+51051)   335
```

## 其它

```text
keys 阻塞 scan 无阻塞
```

