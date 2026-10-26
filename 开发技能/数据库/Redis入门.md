# Overview

Redis 的全称是 **RE**mote **DI**ctionary **S**erver（远程字典服务器）。其采用单线程事件循环模型，避免多线程上下文切换和锁竞争，redis 6.0 之后引入多线程 IO，网络读写可并行。
- 单线程仅指网络请求处理模块采用单线程模型，持久化（RDB/AOF）会在后台开启独立线程/进程处理，不会阻塞主线程。


Redis 主要由两个程序组成：
- Redis 客户端 `redis-cli` — 命令行工具，用于连接服务器并执行命令
- Redis 服务器 `redis-server` — 服务端程序，提供 Redis 服务

Redis 可以看成是**运行在另一个进程中的远程数据结构容器**。Redis Service 本质上就是监听端口（默认是 6379），客户端建立 TCP 连接之后，不断发送请求，然后等待 Service 的回应。**Redis 连接通常应该复用**，而不是每次业务请求都重新 connect。因为频繁建立/关闭连接有额外成本。

在远程 redis 服务商执行命令，使用的也是 redis-cli 命令：
```bash
-h host -p port -a password
```


一个普通的 Redis Server 实例，可以包含多个逻辑数据库（Logical Database），每个数据库都有自己独立的 Key 空间，但是它们共享同一个 Redis 实例的内存、CPU 和持久化等资源（所以一个数据库使用了大量内存，依然可能影响其他数据库）。

在普通的 Redis Open Source 单机部署中，默认提供 16 个逻辑数据库，编号从 `0` 到 `15`。这个数量可以通过配置修改。

但是在 Redis Cluster 模式中只支持 DB 0，Cluster 中不同节点共享 Key 空间。


---

Redis 中的数据结构可以和 C++STL 的数据结构进行概念上的对应：

```text
C++ / Lua                    Redis

std::string                  String

unordered_map<string, ...>   Hash

unordered_set                Set

按 score 排序的集合           Sorted Set

deque / queue                List
```

但是 Redis 是通过网络访问数据的，所以 Redis 操作不像是普通内存访问那么便宜。

---

Redis command 的结构（所有命令都是原子执行）：

```text
COMMAND key arg1 arg2 ...
```

COMMAND 的命名非常规律，同一种数据结构的命令通常具有统一前缀：

String：

```text
SET
GET
INCR
```

Hash：

```text
HSET
HGET
HGETALL
HDEL
HINCRBY
```

Set：

```text
SADD
SREM
SISMEMBER
SMEMBERS
```

Sorted Set：

```text
ZADD
ZRANK
ZREVRANK
ZRANGE
ZSCORE
```

List：

```text
LPUSH
RPUSH
LPOP
RPOP
LRANGE
```


---

# 安装使用


在 ubuntu 中使用 apt 安装 redis：

```bash
sudo apt update
sudo apt install redis-server
```

管理命令：

```bash
sudo systemctl start redis-server  
sudo systemctl stop redis-server
sudo systemctl restart redis-server
sudo systemctl status redis-server
```


---

# 基础数据类型

Redis 的关键问题在于**某个业务问题适合用 Redis 的哪一种数据结构表达**。

## String

Redis String 是一个**字节序列**，存储的是一段二进制数据，所以仅仅是可以存储字符串，另外，字符串是二进制安全的（具有已知长度，不受任何特殊终止字符的影响）。

用途一：普通值

```redis
SET player:10001:name Felix
GET player:10001:name
```

用途二：计数器

可以配合数字操作：

```redis
SET player:10001:gold 100
INCR player:10001:gold
```

或者：

```redis
INCRBY player:10001:gold 1
```

用途三：临时状态+TTL

```redis
SET session:abc123 10001 EX 1800
```

表示该 String 数据 1800 秒之后过期。


|等级|命令|掌握要求|
|---|---|---|
|A|`SET`、`GET`|基本读写，必须掌握 `NX`、`XX`、`EX`、`PX` 选项|
|A|`INCR`、`INCRBY`、`DECR`、`DECRBY`|原子计数与增减|
|A|`MSET`、`MGET`|批量读写，理解返回顺序|
|B|`GETDEL`、`GETEX`、`APPEND`、`STRLEN`|了解用途，需要时能查文档|
|C|`SETRANGE`、`GETRANGE`、`INCRBYFLOAT` 等|知道可操作部分内容或浮点数，无需记忆|


SET 命令的语法：

```bash
SET key value [NX | XX] [GET] [EX seconds | PX milliseconds | EXAT unix-time-seconds | PXAT unix-time-milliseconds | KEEPTTL]
```


---

## Hash

一组 field-value pair，例如一个玩家的状态：

```text
player:10001

name  = Felix
level = 20
gold  = 1000
exp   = 5200
```

可以使用 Hash 表示成：

```redis
HSET player:10001 name Felix
HSET player:10001 level 20
HSET player:10001 gold 1000
HSET player:10001 exp 5200
```

或者：

```redis
HSET player:10001 name Felix level 20 gold 1000 exp 5200
```


读取：

```redis
HGET player:10001 level
```

全部读取：

```redis
HGETALL player:10001
```

**简单、字段化、需要单独修改某些字段的数据**，使用 Hash 来存储很合适。


|等级|命令|掌握要求|
|---|---|---|
|A|`HSET`、`HGET`|字段读写|
|A|`HMGET`、`HGETALL`|获取多个或全部字段|
|A|`HDEL`、`HEXISTS`|删除与存在性检查|
|A|`HINCRBY`|字段原子计数|
|B|`HLEN`、`HKEYS`、`HVALS`、`HSCAN`|数量查询、遍历|
|C|`HSTRLEN`、`HRANDFIELD` 等|特殊操作，按需查询|


---

## Set

Redis Set 是**无序的唯一字符串集合**。对 Set 增加、删除、检查成员，都是 O（1）。

```redis
SADD guild:1001:members 10001
SADD guild:1001:members 10001
```

不会得到两个 10001。

Set 最重要的用于是存储**集合关系**。比如判断某个玩家有哪些好友，以及某个玩家是不是指定玩家的好友。

```redis
SISMEMBER friends:10001 10002
```

|等级|命令|掌握要求|
|---|---|---|
|A|`SADD`、`SREM`|添加、删除集合成员|
|A|`SISMEMBER`|判断是否属于集合|
|A|`SCARD`|统计成员数量|
|A|`SINTER`、`SUNION`、`SDIFF`|交集、并集、差集|
|B|`SMEMBERS`、`SSCAN`|获取或遍历成员，注意大集合|
|B|`SPOP`、`SRANDMEMBER`|随机移除或随机获取成员|
|C|`SMOVE`、`SINTERSTORE`、`SUNIONSTORE` 等|特殊集合操作|

---

## Sorted Set

Sorted Set 是**唯一字符串集合**，每个成员关联一个 `score`，并且根据 score 报错顺序。

```redis
ZADD key score value
```

适合用于游戏排行榜：

```redis
ZADD rank:season:1 3500 10001
ZADD rank:season:1 4200 10002
ZADD rank:season:1 2900 10003
```

输出排行榜：
```redis
ZREVRANGE rank:season:1 0 -1 WITHSCORES
```

查看玩家积分：

```redis
ZSCORE rank:season:1 10001
```

查看玩家排名：

```redis
ZREVRANK rank:season:1 10001
```

**Redis rank、从 0 开始**。支持负数索引：-1 值最后一个元素。


|等级|命令|掌握要求|
|---|---|---|
|A|`ZADD`、`ZREM`|添加、修改、删除成员|
|A|`ZSCORE`、`ZINCRBY`|查询和增加积分|
|A|`ZRANGE`|正序、逆序和按分数范围查询|
|A|`ZRANK`、`ZREVRANK`|查询玩家排名|
|A|`ZCARD`|查询排行榜成员数|
|B|`ZCOUNT`、`ZPOPMIN`、`ZPOPMAX`|范围统计、弹出最高或最低成员|
|B|`ZSCAN`、`ZMSCORE`|遍历、批量读取积分|
|C|`ZINTER`、`ZUNION`、`ZRANDMEMBER` 等|特殊运算，按需查询|

---

## List

Redis List 是**按插入顺序排列的字符串列表**，类似于 queue。

```redis
RPUSH queue 10001
RPUSH queue 10002
RPUSH queue 10003
```

然后

```redis
LPOP queue
```

得到 10001。

|等级|命令|掌握要求|
|---|---|---|
|A|`LPUSH`、`RPUSH`|从两端插入|
|A|`LPOP`、`RPOP`|从两端弹出|
|A|`LRANGE`、`LLEN`|区间查询和长度|
|A|`BLPOP`|阻塞式获取任务|
|B|`LTRIM`、`LINDEX`、`LSET`|裁剪、索引访问、修改元素|
|B|`LMOVE`、`BLMOVE`|在两个 List 之间原子移动元素|
|C|`LPOS`、`LINSERT`、`LREM` 等|特殊元素定位与修改|

---

## Bitmap


Bitmap 通过只能存放 0 或者 1，可以用来统计状态。

```redis
SETBIT key offset value
```

value 只能是 0 或者 1 。

|等级|命令|掌握要求|
|---|---|---|
|实操重点|`SETBIT`、`GETBIT`|设置、查询某一位|
|实操重点|`BITCOUNT`|统计置为 1 的位数|
|B|`BITOP`|位与、或、异或等运算|
|C|`BITPOS`、`BITFIELD`、`BITFIELD_RO`|了解用途即可|

---

## TTL

TTL 不是一类数据结构，而是 Key 的生命周期属性。

```redis
EXPIRE session:abc 1800
```

表示 key 为 `session:abc` 的数据 1800 秒之后删除。

---

# Key 与 TTL

## key->value

Redis 里每个数据对象都有一个**唯一 key**，value 才是 String、Hash、Set、Sorted Set、List 等具体数据结构。key 本身是字符串，而且 Redis 并没有真正的“命名空间”机制，因此命名冲突需要应用自己避免。官方推荐通过 `:` 把 key 分段组织起来。


一个好的 key 命名应该表达业务语义，在游戏服务器中，建议使用这种风格：

```text
实体:ID:用途
```

例如：

```text
player:10001
player:10001:friends

guild:20001:members

session:abcdef

online:10001

rank:arena:season:7
```

---

## Key 操作

查询一个 key 是否存在：

```redis
EXISTS online:10001
```

删除一个 key：

```redis
DEL session:abc123
```

整个 key->value 都会被删除。

查询 key 对应 value 的数据结构类型：

```redis
TYPE player:10001
```


---

## TTL

TTL 也即是 Time To Live。过期机制独立于 value 类型；到期之后 Redis 会自动删除这个 key。**也就是 TTL 的对象是 key**。

给已经存在的 key 设置寿命：

```redis
SET online:10001 gate1
EXPIRE online:10001 60
```

对已经带过期时间的 key 再调用 `EXPIRE`，会把 TTL **更新成新的时间**。可以利用此机制实现心跳刷新在线 TTL。

查询剩余寿命：

```redis
TTL session:abc123
```

- >=0：剩余存活秒数。
- -1：key 存在，但是没有过期时间。
- -2：key 不存在。

```redis
PEXPIRE
PTTL
```

进行类似的操作，单位是毫秒。


取消超时机制：

```redis
PERSIST a                        
```

Redis 持久化的是**实际过期时间**，因此 Redis Server 停止运行时，时间仍然会继续流逝。

---


## KEYS 和 SCAN

查找 key：

```redis
KEYS online:*
```

会返回所有以 online: 开头的 key。

KEYS 并不适合业务代码，因为 KEYS 会一次遍历并返回所有匹配的 key，当 key 数量很多时，扫描一次 keyspace 会阻塞 Redis Server。


使用 SCAN 进行增量遍历，每次处理有限数量：

```redis
SCAN 0 MATCH online:* COUNT 100
```

```text
从 cursor=0 开始

每次大约处理一小批

只看 online:* 类型的 key
```

Redis 返回：

```
next_cursor
+
这一批 keys
```

---


# Pipeline





---