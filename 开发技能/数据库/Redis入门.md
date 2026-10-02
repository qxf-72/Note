# 基础概念


Redis 可以看成是**运行在另一个进程中远程数据结构容器**。

```text
skynet
  │
  │ TCP
  ▼
127.0.0.1:6379
  │
  ▼
redis-server
```

Redis Service 本质上就是监听端口，客户端建立 TCP 连接之后，不断发送请求，然后等待 Service 的回应。

Skynet 的 `skynet.db.redis` 本质上就是进行这样的操作。

Redis 中的数据结构可以和 C++STL 的数据结构进行对应：

```text
C++ / Lua                    Redis

std::string                  String

unordered_map<string, ...>   Hash

unordered_set                Set

按 score 排序的集合           Sorted Set

deque / queue                List
```

但是存在根本区别：

```text
普通容器：

unordered_map
   ↓
在当前进程内存里


Redis：

Hash
   ↓
在 redis-server 进程里
   ↓
通过网络访问
```

所以 Redis 操作不像是普通内存访问那么便宜。


`redis-cli` 的两种使用方式：
- repl 模式：输入多行命令。
- 单命令模式

不同编程语言对应的客户端库：

```text
Python        redis-py
JavaScript    node-redis
Java          Jedis / Lettuce
Go            go-redis
C             hiredis
C++           Boost.Redis
...
```


**Redis 连接通常应该复用**，而不是每次业务清酒都重新 connect。因为频繁建立/关闭连接有额外成本。

Redis command 的结构：

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

<br/>


<br/>

# 下载使用


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


<br/>


<br/>

# 数据类型

Redis 不是一个只能 `key->value` 的缓存，而是一个**通过网络操作多种数据结构的服务器**。可以把 Redis 想象成远程版的 STL。

Redis 的关键问题在于**某个业务问题适合用 Redis 的哪一种数据结构表达**。

## `String`

Redis String 是一个**字节序列**，存储的是一段二进制数据，所以不单单是可以存储字符串。

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

---

## `Hash`

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

---

## `Set`

Redis Set 是无序的唯一**字符串**集合。对 Set 增加、删除、检查成员，都是 O（1）。

```redis
SADD guild:1001:members 10001
SADD guild:1001:members 10001
```

不会得到两个 10001。

Set 最重要的用于是存储**集合关系**。比如判断某个玩家有哪些好友，以及某个玩家是不是指定玩家的好友。

```redis
SISMEMBER friends:10001 10002
```

---

## `Sorted Set`

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

**Redis rank、从 0 开始**。

---


## `List`

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

---

## TTL

TTL 是 Key 的生命周期属性。


```redis
EXPIRE session:abc 1800
```

表示 key 为 session：abc 的数据 1800 秒之后删除。

---


<br/>


<br/>

# 数据类型的选择


不要从“Redis 有什么命令”出发，而要从“业务数据有什么结构和操作需求”出发。

## String

一个 key -> 一个值。

适合：
- token -> uid
- 在线状态
- counter
- 整体序列化对象

如果整个数据总是一起读写，String 简单且紧凑。

---

## Hash

一个 key 下有多个 field -> value。

适合：
- 简单、扁平的小对象
- 需要频繁单独访问/修改字段

例如：
player:10001
  level -> 20
  gold  -> 1000

---

## Set

唯一成员集合，无顺序。

适合：
- 公会成员
- 好友
- 黑名单
- 已领取奖励

重点能力：
membership / union / intersection / difference

---

## Sorted Set

唯一成员 + score，并按 score 排序。

适合：
- 排行榜
- 匹配优先级
- 有序集合

如果只是唯一成员，用 Set；
如果还需要排序，用 Sorted Set。

---

## List

按照插入位置维护顺序。

适合：
- FIFO
- LIFO
- 简单队列

顺序来自插入顺序。

---

## Stream
事件/消息序列，支持 consumer group。

以后需要：
- 多消费者
- 消息确认
- 事件流

时再学习。

---

## 最重要的选择

简单值
→ String

简单对象 + 单字段访问
→ Hash

唯一集合
→ Set

唯一集合 + score / 排序
→ Sorted Set

简单 FIFO/LIFO
→ List

复杂事件流
→ Stream


---


# Key 与 TTL

## `key -> value`

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

TTL = **Time To Live**。过期机制独立于 value 类型；到期之后 Redis 会自动删除这个 key。

给已经存在的 key 设置寿命：

```redis
SET online:10001 gate1
EXPIRE online:10001 60
```

对已经带过期时间的 key 再调用 `EXPIRE`，会把 TTL 更新成新的时间。可以利用此机制实现心跳刷新在线 TTL。


查询剩余寿命：

```redis
TTL session:abc123
```

返回的单位也是秒。

```text
TTL key

>= 0
    有 TTL，返回剩余秒数

-1
    key 存在，但没有过期时间

-2
    key 不存在
```

还有

```redis
PEXPIRE
PTTL
```

单位是**毫秒**。


取消超时机制：

```redis
PERSIST a                        
```

Redis 持久化的是**实际过期时间**，因此 Redis Server 停止运行时，时间仍然会继续流逝。

---


## `SCAN`


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

<br/>


<br/>



# Pipeline





---