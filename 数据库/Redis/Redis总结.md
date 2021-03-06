> 转载： [redis 总结——重构版](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247484858&idx=1&sn=8e222ea6115e0b69cac91af14d2caf36&chksm=cea24a71f9d5c367148dccec3d5ddecf5ecd8ea096b5c5ec32f22080e66ac3c343e99151c9e0&token=1082669959&lang=zh_CN&scene=21#wechat_redirect)

## 1. redis简介

简单来说 redis 就是一个数据库，不过与传统数据库不同的是 redis 的数据是存在**内存**中的，所以存写速度非常快，因此 redis 被广泛应用于缓存方向。另外，redis 也经常用来做**分布式锁**。redis 提供了多种数据类型来支持不同的业务场景。除此之外，redis 支持事务 、持久化、LUA脚本、LRU驱动事件、多种集群方案。

## 2. 为什么要用 redis /为什么要用缓存

主要从“高性能”和“高并发”这两点来看待这个问题。

### 2.1 高性能

假如用户第一次访问数据库中的某些数据。这个过程会比较慢，因为是从硬盘上读取的。将该用户访问的数据存在数缓存中，这样下一次再访问这些数据的时候就可以直接从缓存中获取了。操作缓存就是直接操作内存，所以速度相当快。如果数据库中的对应数据改变的之后，同步改变缓存中相应的数据即可！

![](https://image.ldbmcs.com/2019-04-26-013556.jpg)

### 2.2 高并发

直接操作缓存能够承受的请求是远远大于直接访问数据库的，所以我们可以考虑把数据库中的部分数据转移到缓存中去，这样用户的一部分请求会直接到缓存这里而不用经过数据库。

![](https://image.ldbmcs.com/2019-04-26-013624.jpg)

## 3. 为什么要用 redis 而不用 map/guava 做缓存?

> 下面的内容来自 segmentfault 一位网友的提问，地址：https://segmentfault.com/q/1010000009106416

缓存分为本地缓存和分布式缓存。以 Java 为例，使用自带的 map 或者 guava 实现的是本地缓存，最主要的特点是轻量以及快速，生命周期随着 jvm 的销毁而结束，并且在多实例的情况下，每个实例都需要各自保存一份缓存，缓存不具有一致性。

使用 redis 或 memcached 之类的称为分布式缓存，在多实例的情况下，各实例共用一份缓存数据，缓存具有一致性。缺点是需要保持 redis 或 memcached服务的高可用，整个程序架构上较为复杂。

## 4. redis 和 memcached 的区别

对于 redis 和 memcached 我总结了下面四点。现在公司一般都是用 redis 来实现缓存，而且 redis 自身也越来越强大了！

1. **redis支持更丰富的数据类型（支持更复杂的应用场景）**：Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等数据结构的存储。memcache支持简单的数据类型，String。

2. **Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而Memecache把数据全部存在内存之中。**

3. **集群模式**：memcached没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是 redis 目前是原生支持 cluster 模式的.

4. **Memcached是多线程，非阻塞IO复用的网络模型；Redis使用单线程的多路 IO 复用模型。**

> 来自网络上的一张图，这里分享给大家！

![](https://image.ldbmcs.com/2019-04-26-013923.jpg)

## 5. redis 常见数据结构以及使用场景分析

### 5.1 String

**常用命令**: set,get,decr,incr,mget 等。

String数据结构是简单的key-value类型，value其实不仅可以是String，也可以是数字。 
常规key-value缓存应用； 

常规计数：微博数，粉丝数等。

### 5.2 Hash

**常用命令**： hget,hset,hgetall 等。

Hash 是一个 String 类型的 field 和 value 的映射表，**hash 特别适合用于存储对象**，后续操作的时候，你可以直接仅仅修改这个对象中的某个字段的值。 比如我们可以Hash数据结构来存储用户信息，商品信息等等。比如下面我就用 hash 类型存放了我本人的一些信息：

```
key=JavaUser293847
value={
  “id”:1,
  “name”:“SnailClimb”,
  “age”: 22,  
  “location”: “Wuhan,Hubei”
}
```

### 5.3 List

**常用命令**: lpush,rpush,lpop,rpop,lrange等

list 就是链表，Redis list 的应用场景非常多，也是Redis最重要的数据结构之一，比如微博的关注列表，粉丝列表，消息列表等功能都可以用Redis的 list 结构来实现。

Redis list 的实现为一个**双向链表**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

另外可以通过 lrange 命令，就是从某个元素开始读取多少个元素，可以基于 list 实现分页查询，这个很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西（一页一页的往下走），性能高。

### 5.4 Set

**常用命令**：sadd,spop,smembers,sunion 等

set 对外提供的功能与list类似是一个列表的功能，**特殊之处在于 set 是可以自动排重的**。

当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。可以基于 set 轻易实现交集、并集、差集的操作。

比如：在微博应用中，可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。Redis可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程，具体命令如下：

```
sinterstore key1 key2 key3     将交集存在key1内
```
### 5.5 Sorted Set

**常用命令**： zadd,zrange,zrem,zcard等

和set相比，sorted set增加了一个权重参数score，使得**集合中的元素能够按score进行有序排列**。

举例： 在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息，适合使用 Redis 中的 SortedSet 结构进行存储。

## 6. redis 设置过期时间

Redis中有个设置时间过期的功能，即对存储在 redis 数据库中的值可以设置一个过期时间。作为一个缓存数据库，这是非常实用的。如我们一般项目中的 token 或者一些登录信息，尤其是短信验证码都是有时间限制的，按照传统的数据库处理方式，一般都是自己判断过期，这样无疑会严重影响项目性能。

我们 set key 的时候，都可以给一个 `expire time`，就是过期时间，通过过期时间我们可以指定这个 key 可以存活的时间。

如果假设你设置了一批 key 只能存活1个小时，那么接下来1小时后，redis是怎么对这批key进行删除的？

**定期删除+惰性删除。**

通过名字大概就能猜出这两个删除方式的意思了。

1. **定期删除**：redis默认是每隔 100ms 就**随机抽取**一些设置了过期时间的key，检查其是否过期，如果过期就删除。注意这里是随机抽取的。为什么要随机呢？你想一想假如 redis 存了几十万个 key ，每隔100ms就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很大的负载！

2. **惰性删除** ：定期删除可能会导致很多过期 key 到了时间并没有被删除掉。所以就有了惰性删除。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存里，**除非你的系统去查一下那个 key，才会被redis给删除掉**。这就是所谓的惰性删除，也是够懒的哈！

但是仅仅通过设置过期时间还是有问题的。我们想一下：**如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致redis内存块耗尽了。怎么解决这个问题呢？**所以Redis提供了**内存淘汰机制**来解决这个问题。

> 为什么不使用定时删除？所谓定时删除，指的是用一个定时器来负责监视key，当这个key过期就自动删除，虽然内存及时释放，但是十分消耗CPU资源，因此一般不推荐采用这一策略。

## 7. redis 内存淘汰机制

Redis在使用内存达到某个阈值（通过`maxmemory`配置)的时候，就会触发内存淘汰机制，选取一些key来删除。内存淘汰有许多策略，下面分别介绍这几种不同的策略。

redis 配置文件 redis.conf 中有相关注释，我这里就不贴了，大家可以自行查阅或者通过这个网址查看： http://download.redis.io/redis-stable/redis.conf

```
# maxmemory <bytes> 配置内存阈值
# maxmemory-policy noeviction 
```

- **noeviction**：当内存不足以容纳新写入数据时，新写入操作会报错。**默认策略**

- **allkeys-lru**：当内存不足以容纳新写入数据时，在键空间中，**移除最近最少使用的key**。

- **allkeys-random**：当内存不足以容纳新写入数据时，在键空间中，**随机移除某个key**。

- **volatile-lru**：当内存不足以容纳新写入数据时，**在设置了过期时间的键空间中，移除最近最少使用的key**。

- **volatile-random**：当内存不足以容纳新写入数据时，**在设置了过期时间的键空间中，随机移除某个key**。

- **volatile-ttl**：当内存不足以容纳新写入数据时，**在设置了过期时间的键空间中，有更早过期时间的key优先移除**。

如何选取合适的策略？**比较推荐的是两种lru策略**。根据自己的业务需求。如果你使用Redis只是作为缓存，不作为DB持久化，那推荐选择`allkeys-lru`；如果你使用Redis同时用于缓存和数据持久化，那推荐选择`volatile-lru`。

> LRU是Least Recently Used的缩写，即最近最少使用。LRU源于操作系统的一种页面置换算法，选择最近最久未使用的页面予以淘汰。在Redis里，就是选择最近最久未使用的key进行删除。

## 8. redis 持久化机制（怎么保证 redis 挂掉之后再重启数据可以进行恢复）

Redis作为一个**内存**数据库，数据是以内存为载体存储的，那么一旦Redis服务器进程退出，服务器中的数据也会消失。为了解决这个问题，Redis提供了持久化机制，也就是把内存中的数据保存到磁盘当中，避免数据意外丢失

Redis提供了两种持久化方案：**RDB持久化**和**AOF持久化**，一个是快照的方式，一个是类似日志追加的方式

### 8.1 RDB

RDB持久化是通过**快照**的方式，即在指定的时间间隔内将内存中的数据集快照写入磁盘。在创建快照之后，用户可以备份该快照，可以将快照复制到其他服务器以创建相同数据的服务器副本，或者在重启服务器后恢复数据。**RDB是Redis默认的持久化方式**。

#### 8.1.1 快照持久化

RDB持久化会生成RDB文件，该文件是一个**压缩**过的**二进制文件**，可以通过该文件还原快照时的数据库状态，即生成该RDB文件时的服务器数据。RDB文件默认为当前工作目录下的`dump.rdb`，可以根据配置文件中的`dbfilename`和`dir`设置RDB的文件名和文件位置。

```bash
# 设置 dump 的文件名
dbfilename dump.rdb

# 工作目录
# 例如上面的 dbfilename 只指定了文件名，
# 但是它会写入到这个目录下。这个配置项一定是个目录，而不能是文件名。
dir ./
```

**触发快照的时机**

- 执行`save`和`bgsave`命令。
- 配置文件设置`save <seconds> <changes>`规则，自动间隔性执行`bgsave`命令。
- 主从复制时，从库全量复制同步主库数据，主库会执行`bgsave`。
- 执行`flushall`命令清空服务器数据。
- 执行`shutdown`命令关闭Redis时，会执行`save`命令。

#### 8.1.2 save和bgsave命令

执行`save`和`bgsave`命令，可以手动触发快照，生成RDB文件，两者的区别如下：

使用`save`命令会阻塞Redis服务器进程，服务器进程在RDB文件创建完成之前是不能处理任何的命令请求。

```bash
127.0.0.1:6379> save
OK
```

而使用`bgsave`命令不同的是，`basave`命令会`fork`一个子进程，然后该子进程会负责创建RDB文件，而服务器进程会继续处理命令请求。

```bash
127.0.0.1:6379> bgsave
Background saving started
```

> `fork()`是由操作系统提供的函数，作用是创建当前进程的一个副本作为子进程。

![2020-07-04-FRnFq8](https://image.ldbmcs.com/2020-07-04-FRnFq8.jpg)

> `fork`一个子进程，子进程会把数据集先写入临时文件，写入成功之后，再替换之前的RDB文件，用二进制压缩存储，这样可以保证RDB文件始终存储的是完整的持久化内容。

#### 8.1.3 自动间隔触发

在配置文件中设置`save <seconds> <changes>`规则，可以自动间隔性执行`bgsave`命令。

```bash
################################ SNAPSHOTTING  ################################
#
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behaviour will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#
#   save ""

save 900 1
save 300 10
save 60 10000
```

`save <seconds> <changes>`表示在seconds秒内，至少有changes次变化，就会自动触发`gbsave`命令。

- `save 900 1`  当时间到900秒时，如果至少有1个key发生变化，就会自动触发`bgsave`命令创建快照。
- `save 300 10`  当时间到300秒时，如果至少有10个key发生变化，就会自动触发`bgsave`命令创建快照。
- `save 60 10000`    当时间到60秒时，如果至少有10000个key发生变化，就会自动触发`bgsave`命令创建快照。

### 8.2 AOF持久化

除了RDB持久化，Redis还提供了AOF（Append Only File）持久化功能，AOF持久化会把被执行的写命令写到AOF文件的末尾，记录数据的变化。默认情况下，Redis是没有开启AOF持久化的，**开启后，每执行一条更改Redis数据的命令，都会把该命令追加到AOF文件中**，这是会降低Redis的性能，但大部分情况下这个影响是能够接受的，另外使用较快的硬盘可以提高AOF的性能。

可以通过配置`redis.conf`文件开启AOF持久化，关于AOF的配置如下：

```bash
# appendonly参数开启AOF持久化
appendonly no

# AOF持久化的文件名，默认是appendonly.aof
appendfilename "appendonly.aof"

# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的
dir ./

# 同步策略
# appendfsync always
appendfsync everysec
# appendfsync no

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 加载aof出错如何处理
aof-load-truncated yes

# 文件重写策略
aof-rewrite-incremental-fsync yes
```

#### 8.2.1 AOF的实现

AOF需要记录Redis的每个写命令，步骤为：命令追加（append）、文件写入（write）和文件同步（sync）。

**命令追加(append)**

开启AOF持久化功能后，服务器每执行一个写命令，都会把该命令以协议格式先追加到`aof_buf`缓存区的末尾，而不是直接写入文件，避免每次有命令都直接写入硬盘，减少硬盘IO次数。

**文件写入(write)和文件同步(sync)**

对于何时把`aof_buf`缓冲区的内容写入保存在AOF文件中，Redis提供了多种策略：

- `appendfsync always`：将`aof_buf`缓冲区的所有内容写入并同步到AOF文件，每个写命令同步写入磁盘。
- `appendfsync everysec`：将`aof_buf`缓存区的内容写入AOF文件，每秒同步一次，该操作由一个线程专门负责。
- `appendfsync no`：将`aof_buf`缓存区的内容写入AOF文件，什么时候同步由操作系统来决定。

`appendfsync`选项的默认配置为`everysec`，即每秒执行一次同步。

关于AOF的同步策略是涉及到操作系统的`write`函数和`fsync`函数的，在《Redis设计与实现》中是这样说明的。

> 为了提高文件写入效率，在现代操作系统中，当用户调用`write`函数，将一些数据写入文件时，操作系统通常会将数据暂存到一个内存缓冲区里，当缓冲区的空间被填满或超过了指定时限后，才真正将缓冲区的数据写入到磁盘里。
>
> 这样的操作虽然提高了效率，但也为数据写入带来了安全问题：如果计算机停机，内存缓冲区中的数据会丢失。为此，系统提供了`fsync`、`fdatasync`同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保写入数据的安全性。

从上面的介绍我们知道，我们写入的数据，操作系统并不一定会马上同步到磁盘，所以Redis才提供了`appendfsync`的选项配置。当该选项时为`always`时，数据安全性是最高的，但是会对磁盘进行大量的写入，Redis处理命令的速度会受到磁盘性能的限制；`appendfsync everysec`选项则兼顾了数据安全和写入性能，以每秒一次的频率同步AOF文件，即便出现系统崩溃，最多只会丢失一秒内产生的数据；如果是`appendfsync no`选项，Redis不会对AOF文件执行同步操作，而是有操作系统决定何时同步，不会对Redis的性能带来影响，但假如系统崩溃，可能会丢失不定数量的数据。

#### 8.2.2 AOF重写(rewrite)

在了解AOF重写之前，我们先来看看AOF文件中存储的内容是啥，先执行两个写操作。

```bash
127.0.0.1:6379> set s1 hello
OK
127.0.0.1:6379> set s2 world
OK
```

然后我们打开`appendonly.aof`文件，可以看到如下内容：

```bash
*3
$3
set
$2
s1
$5
hello
*3
$3
set
$2
s2
$5
world
```

> 该命令格式为Redis的序列化协议（RESP）。`*3`代表这个命令有三个参数，`$3`表示该参数长度为3。

看了上面的AOP文件的内容，我们应该能想象，随着时间的推移，Redis执行的写命令会越来越多，AOF文件也会越来越大，过大的AOF文件可能会对Redis服务器造成影响，如果使用AOF文件来进行数据还原所需时间也会越长。

时间长了，AOF文件中通常会有一些冗余命令，比如：过期数据的命令、无效的命令（重复设置、删除）、多个命令可合并为一个命令（批处理命令）。所以AOF文件是有精简压缩的空间的。

AOF重写的目的就是减小AOF文件的体积，不过值得注意的是：**AOF文件重写并不需要对现有的AOF文件进行任何读取、分享和写入操作，而是通过读取服务器当前的数据库状态来实现的**。

文件重写可分为手动触发和自动触发，手动触发执行`bgrewriteaof`命令，该命令的执行跟`bgsave`触发快照时类似的，都是先`fork`一个子进程做具体的工作。

```bash
127.0.0.1:6379> bgrewriteaof
Background append only file rewriting started
```

自动触发会根据`auto-aof-rewrite-percentage`和`auto-aof-rewrite-min-size 64mb`配置来自动执行`bgrewriteaof`命令。

```bash
# 表示当AOF文件的体积大于64MB，且AOF文件的体积比上一次重写后的体积大了一倍（100%）时，会执行`bgrewriteaof`命令
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

下面看一下执行`bgrewriteaof`命令，重写的流程：

![2020-07-04-QyCh0O](https://image.ldbmcs.com/2020-07-04-QyCh0O.jpg)

- 重写会有大量的写入操作，所以服务器进程会`fork`一个子进程来创建一个新的AOF文件。

- 在重写期间，服务器进程继续处理命令请求，如果有写入的命令，追加到`aof_buf`的同时，还会追加到`aof_rewrite_buf`AOF重写缓冲区。

- 当子进程完成重写之后，会给父进程一个信号，然后父进程会把AOF重写缓冲区的内容写进新的AOF临时文件中，再对新的AOF文件改名完成替换，这样可以保证新的AOF文件与当前数据库数据的一致性。

### 8.3 数据恢复

Redis4.0开始支持RDB和AOF的混合持久化（可以通过配置项 `aof-use-rdb-preamble` 开启）

- 如果是redis进程挂掉，那么重启redis进程即可，直接基于AOF日志文件恢复数据。
- 如果是redis进程所在机器挂掉，那么重启机器后，尝试重启redis进程，尝试直接基于AOF日志文件进行数据恢复，如果AOF文件破损，那么用`redis-check-aof fix`命令修复。
- 如果没有AOF文件，会去加载RDB文件。
- 如果redis当前最新的AOF和RDB文件出现了丢失/损坏，那么可以尝试基于该机器上当前的某个最新的RDB数据副本进行数据恢复。

### 8.4 RDB vs AOF

上面介绍了RDB持久化和AOF持久化，那么来看一下他们各自的优缺点以及该如何选择持久化方案。

#### 8.4.1 RDB和AOF优缺点

关于RDB和AOF的优缺点，官网上面也给了比较详细的说明[redis.io/topics/pers…](https://redis.io/topics/persistence)

**RDB**

优点：

- RDB快照是一个压缩过的非常紧凑的文件，保存着某个时间点的数据集，适合做数据的备份，灾难恢复。
- 可以最大化Redis的性能，在保存RDB文件，服务器进程只需fork一个子进程来完成RDB文件的创建，父进程不需要做IO操作。
- 与AOF相比，恢复大数据集的时候会更快。

缺点：

- RDB的数据安全性是不如AOF的，保存整个数据集的过程是比繁重的，根据配置可能要几分钟才快照一次，如果服务器宕机，那么就可能丢失几分钟的数据。
- Redis数据集较大时，fork的子进程要完成快照会比较耗CPU、耗时。

**AOF**

优点：

- 数据更完整，安全性更高，秒级数据丢失（取决fsync策略，如果是everysec，最多丢失1秒的数据）。
- AOF文件是一个只进行追加的日志文件，且写入操作是以Redis协议的格式保存的，内容是可读的，适合误删紧急恢复。

缺点：

- 对于相同的数据集，AOF文件的体积要大于RDB文件，数据恢复也会比较慢。
- 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB。 不过在一般情况下， 每秒 fsync 的性能依然非常高。

#### 8.4.2 如何选择RDB和AOF

- 如果是数据不那么敏感，且可以从其他地方重新生成补回的，那么可以关闭持久化。

- 如果是数据比较重要，不想再从其他地方获取，且可以承受数分钟的数据丢失，比如缓存等，那么可以只使用RDB。

- 如果是用做内存数据库，要使用Redis的持久化，建议是RDB和AOF都开启，或者定期执行`bgsave`做快照备份，RDB方式更适合做数据的备份，AOF可以保证数据的不丢失。

## 9. redis 事务

**Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务(transaction)功能**。事务提供了一种将多个命令请求打包，然后一次性、按顺序地执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。

在传统的关系式数据库中，常常用 ACID 性质来检验事务功能的可靠性和安全性。在 Redis 中，事务总是具有原子性（Atomicity)、一致性(Consistency)和隔离性（Isolation），并且当 Redis 运行在某种特定的持久化模式下时，事务也具有持久性（Durability）。

## 10. 缓存雪崩，缓存击穿，缓存穿透

### 10.1 缓存雪崩

简介：缓存同一时间大面积的失效，所以，后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉。

解决办法：

- 事前：尽量保证整个 redis 集群的高可用性，发现机器宕机尽快补上。选择合适的内存淘汰策略。

- 事中：本地ehcache缓存 + hystrix限流&降级，避免MySQL崩掉

- 事后：利用 redis 持久化机制保存的数据尽快恢复缓存

![](https://image.ldbmcs.com/2019-04-26-021840.jpg)

### 10.2 缓存穿透

简介：一般是黑客故意去请求缓存中不存在的数据，导致所有的请求都落到数据库上，造成数据库短时间内承受大量请求而崩掉。

解决办法： 有很多种方法可以有效地解决缓存穿透问题，最常见的则是采用**布隆过滤器**，将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。另外也有一个更为简单粗暴的方法（我们采用的就是这种），如果一个查询返回的数据为空（不管是数 据不存在，还是系统故障），我们仍然把这个空结果进行缓存，但它的过期时间会很短，最长不超过五分钟。

参考：

https://blog.csdn.net/zeb_perfect/article/details/54135506enter link description here

### 10.3 缓存击穿

#### 10.3.1 什么是缓存击穿

在高并发的情况下，大量的请求同时查询同一个key时，此时这个key正好失效了，就会导致同一时间，这些请求都会去查询数据库，这样的现象我们称为**缓存击穿**。

#### 10.3.2 击穿带来的问题

会造成某一时刻数据库请求量过大。

#### 10.3.3 解决办法

采用**分布式锁**，只有拿到锁的第一个线程去请求数据库，然后插入缓存，当然每次拿到锁的时候都要去查询一下缓存有没有。

## 11. 如何解决 Redis 的并发竞争 Key 问题

所谓 Redis 的并发竞争 Key 的问题也就是多个系统同时对一个 key 进行操作，但是最后执行的顺序和我们期望的顺序不同，这样也就导致了结果的不同！

推荐一种方案：分布式锁（zookeeper 和 redis 都可以实现分布式锁）。（如果不存在 Redis 的并发竞争 Key 问题，不要使用分布式锁，这样会影响性能）

基于zookeeper临时有序节点可以实现的分布式锁。大致思想为：每个客户端对某个方法加锁时，在zookeeper上的与该方法对应的指定节点的目录下，生成一个唯一的瞬时有序节点。 判断是否获取锁的方式很简单，只需要判断有序节点中序号最小的一个。 当释放锁的时候，只需将这个瞬时节点删除即可。同时，其可以避免服务宕机导致的锁无法释放，而产生的死锁问题。完成业务流程后，删除对应的子节点释放锁。

在实践中，当然是从以可靠性为主。所以首推Zookeeper。

参考：https://www.jianshu.com/p/8bddd381de06

## 12. 如何保证缓存与数据库双写时的数据一致性？

你只要用缓存，就可能会涉及到缓存与数据库双存储双写，你只要是双写，就一定会有数据一致性的问题，那么你如何解决一致性问题？

一般来说，就是如果你的系统不是严格要求缓存+数据库必须一致性的话，缓存可以稍微的跟数据库偶尔有不一致的情况，最好不要做这个方案，读请求和写请求串行化，串到一个内存队列里去，这样就可以保证一定不会出现不一致的情况

串行化之后，就会导致系统的吞吐量会大幅度的降低，用比正常情况下多几倍的机器去支撑线上的一个请求。