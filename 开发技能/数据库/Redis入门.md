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




---