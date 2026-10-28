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

Redis String 是一个**字节序列**，存储的是一段二进制数据，所以不仅仅是可以存储字符串，另外，字符串是二进制安全的（具有已知长度，不受任何特殊终止字符的影响）。

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


一个好的 key 命名应该表达业务语义，在游戏服务器中，建议使用这种风格：**实体:ID:用途**

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
EXPIRE online:10001 60
```

对已经带过期时间的 key 再调用 `EXPIRE`，会把 TTL **更新成新的时间**。可以利用此机制实现心跳刷新在线 TTL。

查询剩余寿命：

```redis
TTL session:abc123
```

- `>=0`：剩余存活秒数。
- `-1`：key 存在，但是没有过期时间。
- `-2`：key 不存在。

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


Redis 客户端和服务端是通过 tcp 进行通信的，所以一个操作的耗时约为：服务端执行命令操作耗时+RTT。当执行命令耗时长时，RTT 可以不纳入考虑，但是需要执行一系列操作耗时较短的命令时，RTT 就成为了性能瓶颈，可以引入管道来提高查询速度。

需要注意：管道不保证原子性，多条命令分别执行。


## 底层原理

Redis 使用 RESP（Redis Serialization Protocol）作为客户端和服务端之间的通信协议。

当客户端想同时发送多条命令时，底层把这些命令的 RESP 编码连续放入发送缓冲区，其实并不需要专门的 PIPELINE 命令，也不需要创建 Pipeline 对象。
- 不要求管道中的多条命令位于同一个 TCP 数据包中。
- 服务器依然是单线程逐条处理命令。
- 命令的回复和请求顺序对应。

---

## 实战使用


### redis-cli

redis-cli 的交互模式没有 PILELINE 命令，普通的交互模式都是输入一条命令就立即发送等待回复。

要使用管道有点麻烦，通常是在 Linux Shell 中调用 `redis-cli --pipe`。例如：

假如要发送：

```redis
SET pipe:key 100
INCR pipe:key
GET pipe:key
```

需要使用标准 RESP 格式进行编码：

```bash
printf '*3\r\n$3\r\nSET\r\n$8\r\npipe:key\r\n$3\r\n100\r\n*2\r\n$4\r\nINCR\r\n$8\r\npipe:key\r\n*2\r\n$3\r\nGET\r\n$8\r\npipe:key\r\n' | redis-cli --pipe
```

这里的 RESP 内容实际上表示三条命令：
```text
*3                  # 命令包含 3 个参数
$3  SET             # SET
$8  pipe:key        # key
$3  100             # value

*2                  # 命令包含 2 个参数
$4  INCR
$8  pipe:key

*2
$3  GET
$8  pipe:key
```


其中 `*3` 表示数组有三个元素，`$8` 表示接下来是一个长度为 8 字节的字符串。上面只是便于理解的表示，真实 RESP 使用 `\r\n` 分隔。

发送完成后，运行：

```bash
redis-cli GET pipe:key
```

可以看到，在 redis-cli 中使用管道比较麻烦，一般都将命令写入文件，然后编写一个脚本来处理。


### skynet

在 skynet 中使用管道很方便，例如：

```lua
local skynet = require "skynet"
local redis = require "skynet.db.redis"

skynet.start(function()
    local db = redis.connect {
        host = "127.0.0.1",
        port = 6379,
        db = 0
    }

    local ops = {
        {"SET", "demo:player:1001:hp", "100"},
        {"INCRBY", "demo:player:1001:hp", "20"},
        {"GET", "demo:player:1001:hp"},
    }

    local result = db:pipeline(ops, {})   -- 一般默认传入{}作为第二个参数表明收集全部命令执行结果，否则只会返回最后一条命令执行的结果

    for i, r in ipairs(result) do
        if r.ok then
            skynet.error("command", i, "result", tostring(r.out))
        else
            skynet.error("command", i, "failed", tostring(r.out))
        end
    end

    db:disconnect()
    skynet.exit()
end)
```

- Pipeline 中出现命令错误，不意味着整个批次失败。
- Redis 通常会继续处理后面的命令。
- 前面成功执行的写入不会因为后面失败而自动撤销。



Redis Cluster 中 Pipeline 的使用方式有些不同，例如：


```lua
local slot = db:keyslot("{foo}")
local conn = db:get_connection_by_slot(slot)   -- 取得对应节点的连接

local ret = conn:pipeline({
    {"HINCRBY", "{foo}hello", 1, 1},
    {"DEL", "{foo}hello"},
    {"HSET", "{foo}hello", 1, 1},
    {"HGETALL", "{foo}hello"},
}, {})
```


---


# 事务


redis 的事务和 mysql、mongodb 的事务类似，但有一些操作没有提供：执行过程中发生错误无法回滚。


## 实战使用

### redis-cli

例如：

```bash
SET score 100

MULTI
INCR score
INCR score
GET score
EXEC
```

如果想在**提交之前**不想继续事务，使用 `DISCARD` 丢弃尚未执行的命令。


两类错误：
- 执行之前：也就是入队阶段，此时发生错误，例如命令参数数量错误等，redis 会在入队时就发现错误，后续执行 exec，队列中所有指令都不会执行。
- 执行中：执行过程中某条指令发生错误，不会影响其他指令的执行，也不会回滚。

### 乐观锁机制：WATCH

举个例子：

```bash
WATCH gold
GET gold

MULTI
SET gold 20
EXEC
```

被 WATCH 的键会被监控以检测其变化。如果在**调用 EXEC之前**，至少有一个被监控的键被修改，则整个事务会中止（事务不会被执行）——我们只需重复操作，希望这次不会有新的竞态条件。这种锁机制称为 乐观锁。在许多用例中，多个客户端访问不同键，因此冲突的可能性较低——通常无需重复操作。

WATCH 实际上是一个条件命令：我们要求 Redis 仅在被监控的键未被修改时才执行事务。这包括：
- 客户端的修改（如写命令）。
- Redis 本身的修改（如过期或驱逐）。在 Redis 6.0.9 之前，过期键不会导致事务中止。

当 EXEC 被调用时，所有键都会被取消监控，无论事务是否被中止。此外，当客户端连接关闭时，所有监控的键也会被取消监控。还可以使用不带参数的 UNWATCH 命令来清除所有监控的键。


### skynet 使用陷阱

提供了一下接口，和 redis-cli 的使用一样：

```lua
db:multi()
db:exec()
db:discard()
db:watch("gold")
db:unwatch()
```

**Redis 事务状态绑定的是客户端连接，而不是 Skynet 协程**

同一个服务中不同写出共享一个 redis 连接，他们的命令可能会穿插交错，会导致一些致命问题：

```text
协程A                       协程B
MULTI
                        SET B:gold 200
SET A:gold 100
EXEC
```

虽然在协程 B 中并没有调用事务，但是其发送到指令莫名其妙的进入了 QUEUE，更关键的是，如果 EXEC 之后 协程 B 的这条命令执行失败了，但是协程 B 并不知道，因为其在发出指令后被纳入事务中，并且得到成功标志。

解决方案：
- 每笔事务独占一个 Redis 连接。
- 使用 skynet.queue 串行化同一连接上的操作。
- 将完整的 MULTI 和 EXEC 封装如依次 Pipeline 调用中。


**自动重连可能破坏事务上下文**

Skynet 的 `socketchannel` 支持连接异常后的自动重连。但是 Redis 的 `WATCH` 和 `MULTI` 状态属于具体 TCP 连接。

如果某个连接在 `WATCH` 之后、`EXEC` 之前断开并重新建立，那么旧连接上的监视状态就丢失了。

因此不能简单认为：

```lua
db:watch(...)
-- 中途断线并自动重连
db:multi()
...
db:exec()
```

仍然具有最初预期的 WATCH 并发保护。

**Redis Cluster 限制**

对于 Redis Cluster，涉及多个 Key 的事务要求**相关 Key 属于同一个 Hash Slot**。

例如：

```
player:{1001}:gold
player:{1001}:diamond
```

具有相同的 Hash Tag，因此会被分配到相同的 Hash Slot。

而：

```
player:1001:gold
player:1002:gold
```

通常不属于相同 Slot，不能直接在一个跨槽事务中进行操作。


> Hosh Slot 使用哈希函数获得，在 Redis Cluster 中一个节点负责一个范围内 Hash Slot，当 Key 中包含有效的 `{...}` 结构时，Redis 只对花括号中的内容进行哈希计算。


一个完整的例子：

```lua
local skynet = require "skynet"
local redis = require "skynet.db.redis"

local conf = {
    host = "127.0.0.1",
    port = 6379,
    db = 0
}

-- 使用独占连接，防止其他协程污染事务状态
local function exchange(uid, cost, reward)
    assert(cost > 0 and cost % 1 == 0)
    assert(reward > 0 and reward % 1 == 0)

    local gold_key = "player:{" .. uid .. "}:gold"
    local diamond_key = "player:{" .. uid .. "}:diamond"

    for attempt = 1, 5 do
        local db

        local ok, success, result = pcall(function()
            db = redis.connect(conf)

            -- 同时监控金币和钻石
            assert(db:watch(gold_key, diamond_key) == "OK")

            local gold_str = db:get(gold_key) or "0"
            local diamond_str = db:get(diamond_key) or "0"

            -- 演示中要求货币以整数存储
            assert(gold_str:match("^%-?%d+$"))
            assert(diamond_str:match("^%-?%d+$"))

            local gold = tonumber(gold_str)
            local diamond = tonumber(diamond_str)

            assert(gold and diamond, "invalid currency")

            -- 余额不足，不需要提交事务
            if gold < cost then
                db:unwatch()
                return false, "NOT_ENOUGH_GOLD"
            end

            -- 检查钻石数据可以正常递增
            assert(diamond >= 0, "invalid diamond")
            
            assert(db:multi() == "OK")
            assert(db:decrby(gold_key, cost) == "QUEUED")
            assert(db:incrby(diamond_key, reward) == "QUEUED")

            local replies = db:exec()

            -- WATCH 冲突：事务没有执行
            if replies == nil then
                return false, "CONFLICT"
            end

            return true, replies
        end)

        -- 结束本次尝试使用的独占连接
        if db then
            db:disconnect()
        end

        if not ok then
            skynet.error("Redis transaction error:", tostring(success))

            -- EXEC 阶段发生网络错误时，
            -- 无法总是确认事务是否已提交
            return false, "UNKNOWN_COMMIT_STATE"
        end

        if success then
            return true, result
        end

        if result ~= "CONFLICT" then
            return false, result
        end

        -- 只对确定未执行的 WATCH 冲突重试
        skynet.sleep(attempt * 2)
    end

    return false, "TOO_MANY_CONFLICTS"
end

skynet.start(function()
    local db = redis.connect(conf)

    db:set("player:{1001}:gold", 100)
    db:set("player:{1001}:diamond", 0)

    db:disconnect()

    -- 两个并发请求，都想消耗 80 金币
    for i = 1, 2 do
        skynet.fork(function()
            local ok, result = exchange("1001", 80, 1)
            skynet.error(
                "exchange", i,
                "success:", ok,
                "result:", result
            )
        end)
    end
end)
```


---

# EVAL 命令

Redis 会将 Lua 脚本作为一个不可以被其他客户端命令穿插的整体。可以：
- 保证完整业务判断与修改过程的原子性。
- 减少 RTT。

## Lua 脚本的使用


语法：

```bash
EVAL script keysnum [key ...]  [arg ...]
```

其中
- `script`：是一段以字符串表示 Lua 代码。
- `keynum`：后续有多少个参数是 key。
- `key`：脚本需要访问的 key，脚本中通过 `KEYS[1]` ，`KEYS[2]` 访问。下标从 1 开始。
- `arg`：普通参数，脚本中通过 `ARGV[1]`，`ARGV[2]` 访问。


Redis 中内嵌的是 Lua 5.1 解释器，有专门的 API 调用 Redis 命令，例如：

```lua
local value = redis.call("GET", KEYS[1])
redis.call("SET", KEYS[1], ARGV[1])
return value
```

```lua
local result = redis.pcall("INCR", KEYS[1])

if result.err then
    return redis.error_reply(result.err)
end

return result
```

|| `redis.call()` | `redis.pcall()` |
|---|---|---|
|执行 Redis 命令|支持|支持|
|成功后的行为|返回执行结果|返回执行结果|
|命令出错|抛出 Lua 异常，通常终止脚本|返回包含 `err` 的 Lua table|
|适合场景|常规业务命令|需要在脚本内部处理错误|

另外，在脚本执行过程中，存在两次数据类型转换：

```text
redis命令结果 --> Lua运行时数据类型 -->  脚本内部处理 --> 编码成RESP返回给客户端 --> skynet驱动解码为lua值
```

默认 RESP2 下有几个值得注意的转换规则：

|脚本返回值|客户端收到的 RESP2 类型|
|---|---|
|Lua 整数 `100`|Integer|
|Lua 字符串 `"100"`|Bulk String|
|Lua 浮点数 `3.14`|Integer `3`|
|Lua `true`|Integer `1`|
|Lua `false` / `nil`|Null|
|Lua 数组 `{1, 2, 3}`|Array|


如果需要返回浮点数，先把浮点数转化为字符串之后再返回。对于 lua table 可以使用 redis 内置的 cjson 编码得到 json 字符串，客户端再解码。

```lua
return cjson.encode({
    gold = 100,
    diamond = 50
})
```

从上面的数据类型转换可以窥探到 skynet 想 redis 发送 eval 后经历的过程。

Redis 的脚本沙箱不允许任意加载 Lua 模块。

官方文档指出，Redis 默认的**脚本繁忙判断阈值为 5 秒**。


## 横向对比

|对比维度|Pipeline|MULTI / EXEC|Lua 脚本|
|---|---|---|---|
|核心目的|减少网络往返|连续执行一组命令|在服务端执行带逻辑的原子操作|
|多命令原子性|不保证|保证不被其他命令穿插|保证不被其他命令穿插|
|根据中间结果判断|客户端需要后续处理|不直接支持|支持|
|失败自动回滚|不支持|不支持|不支持|
|一次提交请求|通常可以|可以流水发送，需 EXEC 提交|可以|
|配合 WATCH 乐观锁|非其核心功能|支持|通常不需要|

---

## 脚本缓存

Redis 提供了脚本缓存机制。

其基本思路是：
1. 通过 `SCRIPT LOAD` 将 Lua 脚本加载到 Redis。
2. Redis 计算脚本的 SHA 1 摘要，并缓存脚本。
3. 客户端保存 SHA 1。
4. 后续通过 `EVALSHA` + 参数执行脚本，无需重复传输源码。

> SHA-1（Secure Hash Algorithm 1）是一种密码学哈希算法（散列算法），它能够将任意长度的数据转换成一个固定长度的 160 位（20 字节）哈希值，通常使用 40 个十六进制字符表示。


例如：

```bash
> SCRIPT LOAD "return redis.call('GET', KEYS[1])" 
 "a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0" 
> EVALSHA a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0 1 mykey "value"
```

脚本缓存并不是永久可靠的存储。

缓存可能因为 Redis 重启、主从切换或者执行 `SCRIPT FLUSH` 而丢失。Redis 7.4 起，使用 `EVAL` / `EVAL_RO` 加载的脚本还可能受到 LRU 缓存淘汰机制影响。


如果 Redis 找不到脚本，返回：

```text
NOSCRIPT No matching script
```

处理思路：

```text
EVALSHA
   |
   +-- 执行成功 → 返回结果
   |
   +-- NOSCRIPT → 重新加载脚本 → 再执行
   |
   +-- 其他错误 → 上报或按相应错误策略处理
```


---

## skynet 使用 redis 脚本


可以直接使用：

```lua
db:eval(script, 1, key, arg)
db:script("load", script)
db:evalsha(sha, 1, key, arg)
```

并不需要为这三个命令单独实现驱动。

---


## 拓展

Redis 7.0 及以后版本使用脚本**效果复制**（Effects Replication）。

大致过程：

```text
Redis 主节点执行 Lua
        |
        +-- SET A 100
        +-- INCR B
        +-- DEL C
        |
        v
收集脚本产生的写命令
        |
        v
以 MULTI / EXEC 形式传播给副本和 AOF
```

也就是说，不是要求副本重新执行整段 Lua 逻辑，而是传播脚本实际产生的**写操作**。

**单个 Redis 脚本不能被当作跨多个 Redis 节点的分布式事务来使用**

在 Cluster 中，涉及多个 Key 的脚本通常需要保证这些 Key 能够由同一个节点处理。

---

# 持久化

```text
Redis
├── 1. 持久化机制
│   ├── RDB
│   ├── AOF
│   └── 对比与混合持久化
├── 2. 内存管理
│   ├── 内存限制
│   ├── Key 过期机制
│   └── 内存淘汰策略
└── 3. 常见问题与性能优化
    ├── Big Key / Hot Key
    ├── 缓存穿透 / 击穿 / 雪崩
    └── 监控与问题排查
```

## Overview


Redis 是以内存为主要数据存储介质的数据库。但是存在持久化机制。

持久化（persistence）不等于高可用（High availablity），也不等于备份（backup）。

---


## RDB

RDB（Redis Database）采用的是快照机制：
- 在某个时间点，将 redis 内存中的整个数据集序列化，生成一个 RDB 文件，下次启动 Redis 时，通过这个文件恢复数据。

RDB 的数据丢失风险需要取决于最近一次成功的快照与故障之间的时间间隔。

RDB 核心命令：

|命令|功能|是否阻塞正常请求|
|---|---|---|
|`SAVE`|同步生成 RDB 文件|是|
|`BGSAVE`|后台生成 RDB 文件|正常运行阶段基本不阻塞，但 `fork` 有短暂开销|
|`LASTSAVE`|查询上一次成功保存的时间|否|

- save：在主进程中完成数据集序列化和磁盘写入，当数据量大时，会造成明显的服务停顿。
- bgsave：创建子进程，由子进程负责生成快照，父进程继续处理客户端请求。
	- 子进程保存的是 fork 那一刻的逻辑数据状态。因为 COW 机制，主进程修改了数据，子进程进行操作系统内存页层面的复制。
	- bgsave 虽然不会阻塞主进程，但是不是没有代价：`fork()` 需要处理页表等内核数据结构，可能使主线程短暂暂停。

RDB 自动触发机制

配置：

```text
save 60 1000
```

表示上次成功保存达到 60 秒，**并且**期间至少发生 1000 次数据修改时，redis 可以触发一次快照。

多组规则可以同时存在，之间是**或**关系，只要一组达到条件就会触发。



优势和局限：
- 文件紧凑易于备份，也具有较快的恢复速度，**适合定期备份以及允许一定数据丢失的场景**。
- 快照之间的修改可能全部丢失。



---


## AOF

AOF（Append Only File）采用的是操作日志方式。AOF 通过记录修改命令来重建该状态。在磁盘中使用的是 Redis 协议格式 RESP，而不是简单保存在终端输入的原始文本。

Redis 重启时，就可以根据日志重建数据集。不过，**“记录修改命令”不代表日志会永久保留所有历史命令**。

AOF 流程：

```text
    Skynet
       |
  Redis 写命令
       |
       v
 Redis 执行命令
       |
更新内存数据集
       |
生成 AOF 日志
       |
 AOF 缓冲区
       |
    write()
       |
 OS Page Cache
       |
    fsync()
       |
    磁盘
```

其中：
- `write()`：将数据写入文件描述符。对于普通文件，成功返回通常意味着数据已经进入操作系统管理的文件缓存，并不代表已经完成持久化落盘。
- `fsync()`：要求操作系统将文件相关的缓存数据同步到持久存储设备。


appendfsync 的三种策略：

|策略|工作方式|可靠性|性能|
|---|---|---|---|
| `always` |每批次 AOF 写入执行同步落盘|较高|较低|
| `everysec` |通常每秒执行一次 fsync|较高，但存在短暂丢失窗口|较高|
| `no` |Redis 不主动执行 fsync，交由操作系统安排|较低|较高|

官方推荐的默认 AOF 同步策略是 `everysec`。

```bash
appendonly yes
appendfsync always
```

在这种模式下，Redis 会在确认相应 AOF 数据已经同步落盘后，再向客户端返回写入成功的响应。

但它也并非严格的“每执行一条命令就调用一次 fsync”。Redis 可以对一批客户端命令进行合并写入和同步。


```bash
appendonly yes
appendfsync everysec
```

通常将它描述为“可能丢失约 1 秒的数据”。

但是要注意，1 秒不是任何故障条件下都绝对成立的硬性上限。磁盘繁忙、同步延迟等情况可能扩大这一窗口。

```bash
appendonly yes
appendfsync no
```

Redis 仍然会生成 AOF 日志并写入文件，但不主动调用 fsync 来保证定期落盘，而是让操作系统自行安排。

它并不意味着“不写 AOF”，也不等于“关闭持久化”。

**AOF Rewrite（日志重写）机制**

Rewrite 的目标不是恢复历史，而是根据当前内存数据集，生成能够重建当前状态的精简持久化文件。

这里有一个特别容易误解的地方：

AOF Rewrite 并不是读取旧 AOF 文件，删除其中重复的命令。

它主要根据 Redis 当前内存中的数据生成新的基础持久化文件。

这与数据库日志的“重新组织和压缩”有一定相似性，但不能直接等同于 MongoDB 的事务日志或 WiredTiger Journal。

通过下面的命令可以主动触发：

```bash
redis-cli BGREWRITEAOF
```

Redis 还支持自动触发。例如常见配置：

```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

大意是当 AOF 文件相对于基准大小增长达到 100%，且满足最低 64 MB 的大小要求时，可以触发自动重写。

这里的基准通常是上一次成功重写后的大小；如果尚未发生过重写，则使用启动时的相关基准大小。

**Redis 7.0+ 的多文件 AOF**

一个典型的目录可能是：

```
appendonlydir/
├── appendonly.aof.1.base.rdb
├── appendonly.aof.1.incr.aof
└── appendonly.aof.manifest
```

各文件的作用如下：

|文件|功能|
|---|---|
|`base.rdb`|保存重写时的基础数据状态|
|`incr.aof`|保存基础状态之后的增量修改|
|`manifest`|记录各 AOF 文件及加载顺序|


---


## 混合持久化

**同时开启 RDB 和 AOF**

```
save 3600 1
appendonly yes
appendfsync everysec
```

Redis 可以生成独立的 RDB 快照，同时维护 AOF 持久化文件。

重启时，如果两种持久化数据都存在并且可用，Redis 优先通过 AOF 恢复数据，因为它通常包含更新的状态。

**AOF 的 RDB preamble**

```
aof-use-rdb-preamble yes
```

它表示 AOF 重写生成的基础文件可以采用 RDB 格式。

于是，一组 AOF 文件可能同时包含 RDB 格式的基础数据和 AOF 格式的增量操作。

这个选项并不意味着 Redis 自动启用了独立的定期 RDB 快照。

两者不是一回事。


---

## 工程选择

从实际工程角度，我建议你主要考察三个指标：数据丢失容忍度、恢复速度和性能代价。

|对比项|RDB|AOF everysec|两者同时启用|
|---|---|---|---|
|持久化方式|定期快照|持续操作日志|两套机制|
|典型数据丢失窗口|快照间隔内|约 1 秒，异常情况下可能更长|主要取决于 AOF|
|恢复速度|通常较快|取决于 Base 和增量日志|正常优先加载 AOF|
|磁盘开销|相对较低|有持续写入开销|综合开销更大|
|主要资源风险|fork、COW、磁盘 I/O|fsync、Rewrite、磁盘 I/O|两者都需要考虑|
|适合场景|定期备份、允许数据丢失|需要较小丢失窗口|重视数据安全与备份|

官方倾向于在重要数据场景下结合使用两种机制，但这不是一条无条件适用于所有业务的规则。

在游戏开发中，由于使用 Mongodb 保存玩家的权威持久化状态，redis 负责缓存、会话和可重建的临时业务数据，所以不一定需要开启 redis 持久化。


**高可用和持久化区分**

这里先建立一个概念：
- 持久化处理数据在磁盘上的保存与恢复。
- 复制（Redis Replication）处理数据在 Redis 节点之间的同步。
- Sentinel、Cluster 等机制可以参与故障检测和切换。

即使开启 AOF，单台 Redis 所在的服务器宕机后仍可能无法提供服务。

而 Redis 主从复制默认是异步的。即使增加副本，也不能保证所有已经确认成功的写入在故障切换后都绝不会丢失。

---

# 内存管理

Redis 主要使用内存存储数据，那么当数据不断增加、内存逐渐不足时，Redis 如何管理和释放内存？

## 内存限制

限制 Redis 用于数据管理的内存规模。Redis 提供一个配置项：

```
maxmemory 512mb
```

也可以在运行时动态配置：

```
CONFIG SET maxmemory 512mb
```

使用下面的命令查看：

```
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

其中：

- `maxmemory`：Redis 的内存限制。
- `maxmemory-policy`：Redis 达到内存限制后的处理策略。

`maxmemory 0` 通常意味着不主动设置这个数据内存上限，而不是禁止 Redis 使用内存。

Redis 内存使用量不断增加，实际内存使用可能短暂超过 maxmemory，Redis 不保证进程内存绝不越界。当 Redis 的受限内存使用量超过 `maxmemory` 时，根据策略，可能出现两种主要结果：
- 拒绝增加内存的写操作，向客户端返回 OOM 错误。
- 主动删除部分 Key，释放内存，再继续处理请求。

由 `maxmemory-policy` 决定。

例如：

```
maxmemory-policy noeviction
```

表示内存不足时不主动淘汰 Key。

```
maxmemory-policy allkeys-lru
```

表示可以从所有 Key 中寻找近期较少访问的 Key 进行淘汰。


---

## Key 过期机制

传统的 Redis Key 过期实现主要使用两个字典：

```
Redis 数据库
    |
    +-- 主字典（Keyspace）
    |     |
    |     +-- player:1001  -> 玩家数据
    |     +-- session:1001 -> 会话信息
    |
    +-- 过期字典（Expires）
          |
          +-- session:1001 -> 过期时间戳（绝对过期时间）
```

主字典保存数据，过期字典维护有 TTL 的 Key 所对应的过期时间。

这样不需要为所有永不过期的 Key 都增加一份时间戳开销。

**惰性过期**

当客户端访问 Key 时，Redis 检查该 Key 是否已经到期。

如果已经到期，就将其视为不存在，并在适当情况下删除。

**定期主动过期**

为了处理那些已经过期但没有被访问的 Key，Redis 会主动抽样检查带 TTL 的 Key。

---

## 内存淘汰策略：Eviction

传统 Redis Open Source 版本中，主要有八种常用策略。

|策略|候选 Key 范围|选择依据|
|---|---|---|
|`noeviction`|不淘汰|内存不足时拒绝相关写入|
|`allkeys-lru`|所有 Key|近期最少使用|
|`allkeys-lfu`|所有 Key|使用频率最低|
|`allkeys-random`|所有 Key|随机|
|`volatile-lru`|带 TTL 的 Key|近期最少使用|
|`volatile-lfu`|带 TTL 的 Key|使用频率最低|
|`volatile-random`|带 TTL 的 Key|随机|
|`volatile-ttl`|带 TTL 的 Key|剩余 TTL 最短|

---

# 常见问题和性能优化

## Big Key

单个 key 对应的 value 占用大量内存。

Redis 没有统一规定：超过多少 KB 或多少个元素就必须算 Big Key。可以参考：

|数据类型|值得重点排查的情况|
|---|---|
|String|Value 达到数百 KB 至 1MB 以上|
|Hash|包含上万甚至更多字段|
|List|包含数万到数十万元素|
|Set|包含大量成员|
|ZSet|包含大量成员，或范围查询返回大量结果|
|Stream|消息持续积累而没有合理裁剪|

**判断 Big Key 不能只看内存大小，还要结合访问频率和操作复杂度**：，一个 200 KB 的 Key，如果每秒被读取几万次，也可能成为严重的性能问题。一个 2 MB 的 Key，如果几乎不访问，而且删除和恢复方式合理，影响可能没有想象中大。


---


