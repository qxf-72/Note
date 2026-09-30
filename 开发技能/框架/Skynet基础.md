# [Skynet 设计综述](https://blog.codingnow.com/2012/09/the_design_of_skynet.html)

Skynet 可以理解为一个基于 Actor 思想的服务器框架：

**用一个较小的多线程消息调度核心，把业务拆成许多通过消息通信的 Service；框架保证同一个 Service 的消息处理串行化，而不同 Service 可以被多个 Worker 并行调度，再利用 Lua coroutine 把异步消息流程包装成容易编写的顺序业务代码。**

> Actor 模型是一种**并发编程模型**。核心思想是把系统拆分成许多相对独立的 Actor，每个 Actor 管理自己的状态，通过**消息传递**进行通信，而不是让多个执行单元直接操作同一份共享状态。
> 
> 一个 Actor 通常负责接收消息、处理消息、修改自己的内部状态，以及向其他 Actor 发送消息。
> 
> 这种模型可以把大量共享状态和锁竞争转化为消息传递问题，因此比较适合高并发和分布式系统。

**Actor 模型强调逻辑上的状态隔离，并不意味着 Skynet 中的 Service 具有 OS 进程级别的内存隔离。** Skynet 中的 Service 默认运行在同一个进程、同一个地址空间中。

---

## Skynet 的核心思想

可以把 Skynet 最核心的工作抽象成：

```
Service A
   │
   │ send(message)
   ▼
Skynet
   │
   ▼
Service B
   │
   └── callback(message)
```

每个 Service：
- 有一个唯一的 `handle`
- 有自己的消息队列
- 注册一个 callback 用于处理消息
- **没有消息时处于挂起状态，不会主动占用 CPU**

如果 Service 需要定时执行逻辑，可以通过系统产生的 timeout 消息来驱动。

可以粗略理解成：

```cpp
class Service {
    Queue<Message> queue;
    void callback(Message msg);
};
```

Skynet 最核心的工作就是：

```cpp
send(A, B, message);
```

把消息发送给目标 Service，再调度它执行相应的 callback。

Skynet 核心刻意保持简单。

它主要负责：

```
Service
   ↓
消息
   ↓
Service
```

而 RPC、数据编码、业务协议、跨机器通信等能力，可以建立在这个消息机制之上。


> RPC 全称 **Remote Procedure Call，远程过程调用**。RPC 是一种“如何向远程服务发请求并获得结果”的通信抽象。RPC 框架会把通信细节包装起来，使其看起来像是函数调用。

---

## 多线程 + Service

Skynet 最初可以考虑使用：

```
一个业务模块
    ↓
一个 OS 进程
```

但如果业务本身使用 Lua，就可以利用不同的 Lua State 隔离 Lua 的运行环境，因此没有必要为每个业务模块创建一个独立进程。

整体可以粗略理解为：

```
                 Skynet 进程

┌────────────────────────────────────┐
│                                    │
│ Service A   Service B   Service C  │
│    │           │           │       │
│ Lua State   Lua State   C Service  │
│                                    │
│          Worker Thread Pool        │
│                                    │
└────────────────────────────────────┘
```

这里要注意：

**Service 和 Lua State 不是同一个概念。**

Skynet Service 可以使用 C 编写，也可以使用 Lua 编写。

对于 Lua Service，可以粗略理解为：

```
Lua Service
    ↓
Lua State
```

### Lua State

`Lua State` 可以理解为一套相对独立的 Lua 虚拟机运行环境。它维护 Lua 程序运行所需要的大量状态，例如：
- Lua 栈
- 全局变量
- registry
- 已加载模块
- Lua 对象
- GC 状态
- Lua VM 的运行状态
    

因此可以粗略理解成：

```
Lua State ≈ 一个 Lua 虚拟机实例
```

不过这种隔离是 **Lua 运行环境层面的隔离**，不是 OS 进程提供的内存保护。

这些 Service 仍然处在：

```
同一个 OS 进程
        ↓
同一个虚拟地址空间
```

这也意味着 Service 之间交换数据的成本可以非常低。

---

## 消息传递

因为不同 Service 位于同一个进程中，所以理论上可以直接传：

```cpp
void* data
size_t size
```

接收方可以直接访问这块内存，从而实现零拷贝。

但是这样需要非常小心地管理：

```text
数据生命周期
所有权
什么时候释放
谁负责释放
```

因此 Skynet 默认采用更安全的方案。普通情况下可以粗略理解为：

```text
发送方数据
    ↓
skynet_send
    ↓
Skynet malloc 一块连续内存
    ↓
复制消息数据
    ↓
接收方 callback
    ↓
Skynet free
```

Skynet 也提供 `PTYPE_TAG_DONTCOPY`：

```text
PTYPE_TAG_DONTCOPY
        ↓
不复制数据
        ↓
直接转移这块内存的所有权
```

这样可以在需要时减少一次数据复制。

---

## 消息结构

Skynet 的一条消息在内部大致为：

```cpp
struct skynet_message {
    uint32_t source;
    int session;
    void* data;
    size_t sz;
};
```

这里主要包含：

```text
source
  │
  └── 谁发送的

session
  │
  └── 请求/响应匹配

data + sz
  │
  └── 消息内容
```

此外还有一个非常重要的概念：

```
type
```

在当时的实现中，`type` 并没有作为独立字段直接放进这个结构，而是编码进了 `size` 的高位。

---

## session

Skynet 核心提供的是：

```
单向消息
```

也就是：

```
A ─────message────→ B
```

但是业务中经常需要：

```
请求 → 响应
```

例如：

```text
Service A                     Service B

request
session = 100
   ──────────────────────────>

                              处理

response
session = 100
   <──────────────────────────
```

A 发出请求时获得一个 session：

```
session = 100
```

然后记录：

```
session 100
    ↓
对应哪个请求
```

B 处理完成以后，把相同的 session 带回来。

于是 A 收到：

```
session = 100
```

就知道这个响应属于之前的哪个请求。

因此： **session 本质上解决的是异步消息环境中的请求—响应匹配问题。**

---

## type

`type` 更准确地说表示：

> **当前消息使用哪一组通信协议。**

而不仅仅是普通意义上的“消息类别”。不同消息可能使用：

```text
TEXT
RESPONSE
CLIENT
HARBOR
MULTICAST
...
```

不同协议可以有不同的数据编码和处理方式。

这样就没有必要规定：

```
整个 Skynet
      ↓
只能使用一种统一消息编码格式
```

而是可以：

```
Service / Protocol A
        ↓
某种编码方式

Service / Protocol B
        ↓
另一种编码方式
```

底层利用 type 区分使用哪种协议进行处理。

这体现了 Skynet 的一个重要设计思想：**底层提供机制，但尽量不强制上层采用统一策略。**

---

## 消息调度

Skynet 使用了**两级消息队列**。

### 第一级：Service 私有消息队列

```
Service A → Queue A

Service B → Queue B

Service C → Queue C
```

每个 Service 都拥有自己的消息队列。

发送消息：

```
send → Service B
```

实际上就是把消息加入：

```
Queue B
```

### 第二级：Global Queue

Skynet 还有一个全局消息队列：

```
Global Queue

┌─────────┐
│ Queue A │
│ Queue C │
│ Queue F │
└─────────┘
```

里面存放的是：

**当前有消息需要处理的 Service Queue。**

Worker Thread：

```
          Global Queue
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
   Worker1  Worker2  Worker3
```

Worker 从 Global Queue 中取出一个 Service Queue，然后：

```
取一个 Service Queue
        ↓
取出一条消息
        ↓
调用对应 Service 的 callback
        ↓
如果还有消息
        ↓
重新放回 Global Queue
```

如果 Service 的消息队列已经为空：

```
Queue Empty
    ↓
不再放回 Global Queue
```

因此没有工作的 Service 不会产生 CPU 空转。

### 最关键的线程安全保证

同一个 Service 的消息队列被一个 Worker 处理时：

```
Worker 1
   ↓
Service A
```

其他 Worker 不会同时获得 Service A 的消息队列。

因此不会出现：

```
Worker 1 ──→ Service A callback
Worker 2 ──→ Service A callback
```

同时执行的情况。

所以：

```
同一个 Service
      ↓
消息串行执行
```

而：

```
Service A ← Worker 1

Service B ← Worker 2

Service C ← Worker 3
```

又可以并行执行。

因此 Skynet 实现的是：

```
Service 内部串行
+
Service 之间并行
```

这也是 Actor 模型在 Skynet 中非常重要的体现。

在文章描述的设计中，每次从 Service Queue 中只处理一条消息，然后就重新参与调度。

这样虽然局部效率稍低，但能够提高公平性，避免某个消息特别多的 Service 长时间霸占 Worker。

---

## Lua coroutine 的作用

Lua coroutine 可以把底层：

```
异步消息
```

包装成上层看起来类似：

```
同步函数调用
```

例如：

```lua
local result = skynet.call(service, "lua", "query")
print(result)
```

代码看起来像：

```
调用
 ↓
等待
 ↓
得到结果
```

但底层实际类似：

```
Coroutine A

发送 request
session = 123
      ↓
记录：

123 → Coroutine A
      ↓
yield
      ↓
callback 返回
```

此时 Worker 并没有被阻塞。

后来：

```
response
session = 123
      ↓
查表
      ↓
找到 Coroutine A
      ↓
resume
```

于是 Lua 代码就可以继续：

```
print(result)
```

因此可以总结成：

```
异步消息系统
      +
Lua Coroutine
      =
类似同步代码的编程体验
```

这个思想非常重要。

coroutine 并没有把异步 IO 变成真正的同步阻塞 IO，而只是让**异步状态机更容易用顺序代码表达**。

---

## Gate

Skynet 内部使用的是：

```
Message
```

但外部客户端使用的是：

```
TCP byte stream
```

因此需要一个组件把：

```
TCP
 ↓
Skynet Message
```

连接起来。

这个组件就是：

```
Gate
```

大致为：

```
Game Client
     │
     │ TCP
     ▼
    Gate
     │
     │ message
     ▼
   Agent
     │
     ▼
业务 Service
```

Gate 主要负责：

```
监听 TCP 端口
        ↓
accept 连接
        ↓
读取 TCP 数据
        ↓
进行必要的分包
        ↓
转成 Skynet Message
        ↓
交给其他 Service
```

Gate 自己尽量不处理具体业务逻辑。

业务逻辑可以交给 Lua Service。

文章当时还设计了：

```
watchdog
agent
broker
```

等不同的数据转发方式。

需要注意的是，在文章描述的设计中：

**Gate 主要负责读取外部连接，并不负责向客户端回写数据。**

发送数据由其他专门的 Service 完成。

---

## Connection

Gate 解决的是：

```
外部客户端
    ↓
Skynet
```

而 Connection 主要解决：

```
Skynet
    ↓
外部服务
```

例如：

```
Skynet
   │
   ├── Redis
   ├── MySQL
   └── 其他 TCP 服务
```

Connection 使用：

```
epoll
```

监听外部 fd 的可读事件。

收到数据以后：

```
socket readable
       ↓
Connection
       ↓
Skynet message
       ↓
目标 Service
```

这样 Service 就不需要：

```
阻塞 recv()
```

等待外部服务返回数据。

这是非常重要的，因为：

```
Worker Thread
      ↓
阻塞 IO
      ↓
整个 Worker 无法继续调度其他 Service
```

会浪费 Skynet 的 Worker。

因此 Skynet 内部的 Service 不推荐执行阻塞式外部 IO。

---

## 跨节点通信：Harbor

虽然 Skynet 的核心设计围绕：

```
单进程
+
多线程
+
Service
```

但它也考虑了多个 Skynet 节点之间的通信。

Service 地址是 32 bit：

```
┌──────────┬──────────────────────┐
│ 高 8 bit │      低 24 bit      │
│ Harbor ID│   Service Handle     │
└──────────┴──────────────────────┘
```

因此可以根据 Service 地址判断：

```
目标 Service
    │
    ├── 当前节点 → 本地消息
    │
    └── 其他节点 → 远程消息
```

如果：

```
Service A → Service B
```

而 B 位于其他 Skynet 节点：

```
A
 ↓
Local Harbor
 ↓
TCP
 ↓
Remote Harbor
 ↓
B
```

Harbor 负责不同 Skynet 节点之间的数据转发。

这样业务 Service 使用消息接口时，不需要过度关心：

```
目标 Service
到底在本机
还是远程机器
```

---

## 名字服务和 Master

数字 handle：

```
:01000042
```

并不适合人直接使用。

因此 Skynet 支持给 Service 起名字，例如：

```
login
database
chat
```

跨 Skynet 节点时，还需要解决：

```
service name
      ↓
在哪个节点？
      ↓
具体 handle 是什么？
```

于是文章中引入了：

```
Master
```

Master 用于同步：

```
全局 Service 名字
+
Skynet 节点地址
```

可以粗略理解成一个全局的：

```
name → address
```

管理服务。

---

## 组播

游戏服务器中经常存在：

```
一条消息
   ↓
发送给很多 Service
```

例如：

```
Player A
   │
   ├── Player B
   ├── Player C
   ├── Player D
   └── Player E
```

如果每次都复制完整消息：

```
copy
copy
copy
copy
```

会造成额外开销。

因此组播消息采用：

```
Message
   +
reference count
```

让多个接收方共享同一个消息数据。

但是 Skynet 没有让普通单播消息也全部使用引用计数。

原因是：

```
不用组播的人
      ↓
不应该承担引用计数的额外成本
```

这体现了 Skynet 很重要的设计原则：

**只有真正使用某个机制时，才为这个机制支付额外成本。**

---

## 整体架构

```
                       Client
                         │
                         │ TCP
                         ▼
                      ┌──────┐
                      │ Gate │
                      └───┬──┘
                          │
                       Message
                          │
                          ▼

┌──────────────────────────────────────┐
│             Skynet Node              │
│                                      │
│ Service A   Service B    Service C   │
│    │           │            │        │
│ Queue A     Queue B      Queue C     │
│      \         │          /          │
│       \        │         /           │
│          Global Queue                │
│               │                      │
│       Worker Thread Pool             │
│                                      │
│ Lua Service 可以拥有独立 Lua State  │
└───────────────┬──────────────────────┘
                │
       ┌────────┴──────────┐
       │                   │
  Connection             Harbor
       │                   │
       ▼                   ▼
 Redis / MySQL         Skynet Node
```

---


<br/>


<br/>

# [Quickstart ](https://github.com/cloudwu/skynet/wiki/GettingStarted)

## Skynet 是什么？

Skynet 不是一个开箱即用的游戏服务器引擎，而更像是**一套构建服务器的工具和运行框架**。

Skynet 最初主要面向游戏服务器场景，因为游戏服务器通常需要同时处理大量相对独立、但又需要长期保存状态的业务，例如：
- 玩家
- 战斗
- 副本
- NPC
- 场景

Skynet 的一个核心目标，就是**利用现代多核 CPU，让大量 Service 高效并发运行**。

可以把 Skynet 粗略看成一个简单的“操作系统”，它负责调度大量 Service：

```
              Skynet
         类似一个小型 OS
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
Lua VM A    Lua VM B    Lua VM C
Service A   Service B   Service C
    │          │          │
    └────── 消息通信 ──────┘
```

Skynet 负责的事情主要包括：

```
Service 调度
消息传递
网络事件
定时器
```

其中很多事件最终都会被转换为消息，再交给对应的 Service 处理。

---

## Service = 逻辑 + 状态

对于游戏服务器，Skynet 不太鼓励把业务设计成传统 Web 服务那种完全无状态的模式：

```
收到请求
  ↓
数据库读取状态
  ↓
处理业务
  ↓
数据库写回
```

游戏中的玩家、场景、副本等通常存在大量需要长期维护的上下文状态。

如果每次业务操作都依赖数据库读取和写入，数据库很容易成为性能瓶颈。

因此 Skynet 更倾向于：

```
Service
   =
业务逻辑
   +
内存中的业务状态
```

例如玩家上线时：

```
数据库
  ↓
加载玩家数据
  ↓
Player Service
  ↓
在线期间主要操作内存状态
  ↓
适当时机持久化
  ↓
数据库
```

此时数据库更偏向于承担：

```
持久化
备份
故障恢复
```

而不是每个业务操作都同步访问数据库。

不过 Skynet **并不规定具体业务架构**。

例如玩家系统既可以设计成：

```
Player 1 → Service 1
Player 2 → Service 2
Player 3 → Service 3
```

也可以设计成：

```
              Player Service
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Player 1   Player 2   Player 3
```

也就是说：

**Skynet 提供 Service 和消息机制，但 Service 应该如何拆分，需要由具体业务决定。**

---

## 网络 IO

Skynet 中可能同时存在大量 Service，但真正执行 Service 代码的 Worker Thread 数量是有限的，通常与 CPU 核心数量相关。

可以粗略理解为：

```
大量 Service
    ↓
少量 Worker Thread
    ↓
多核 CPU
```


如果某个 Service 直接执行阻塞式网络 IO：

```
recv(fd, buf, size, 0);
```

那么：

```
Service A
   ↓
recv()
   ↓
阻塞
   ↓
Worker Thread 也被阻塞
```

此时受影响的不只是 Service A，因为这个 Worker Thread 暂时无法继续调度其他 Service。

因此：

**不要在 Skynet Service 中直接使用可能阻塞的系统网络 API，而应该使用 Skynet 提供的网络层。**

正确的思路更接近：

```
socket
  ↓
Skynet 网络层
  ↓
等待 IO 时挂起业务流程
  ↓
Worker 可以执行其他 Service
  ↓
IO ready
  ↓
转换为消息
  ↓
继续处理
```

---

## 单个 Service 内可以有多个 coroutine

一个 Service 内部可以同时存在多个 Lua coroutine：

```
同一个 Service
   │
   ├── coroutine A
   ├── coroutine B
   ├── coroutine C
   └── coroutine D
```

例如：

```lua
local result = skynet.call(service, "lua", "query")
```

执行到 `skynet.call` 后，如果需要等待远程 Service 返回结果：

```
Coroutine A
    ↓
发送 request
    ↓
yield
```

此时只是：

```
Coroutine A 被挂起
```

并不是：

```
整个 Service 被阻塞
```

更不是：

```
Worker Thread 被阻塞
```

在等待期间，该 Service 仍然可以处理其他消息：

```
Coroutine A
    ↓
yield

Coroutine B
    ↓
处理另一条消息

Coroutine C
    ↓
处理其他任务
```

等到 A 等待的响应到达后：

```
response
   ↓
根据 session 找到 Coroutine A
   ↓
resume
```

A 再继续执行。

因此：

**Skynet Service 在一条业务流程挂起等待响应时，仍然可以处理其他消息。**

---

## Service 内的 coroutine 是并发，不是并行

这一点非常重要。

同一个 Service 内虽然可以存在多个 coroutine：

```
Coroutine A
Coroutine B
Coroutine C
```

但它们不会同时在多个 CPU 核心上运行。

而是：

```
Coroutine A
   ↓ yield
Coroutine B
   ↓ yield
Coroutine C
   ↓ yield
Coroutine A
```

所以同一个 Service 内属于：

```
协作式并发
```

而不是：

```
多线程并行
```

真正的多核并行发生在不同 Service 之间：

```
Worker 1 → Service A

Worker 2 → Service B

Worker 3 → Service C
```

因此可以总结为：

```
不同 Service 之间
       ↓
多线程并行

同一个 Service 内
       ↓
coroutine 协作式并发
```

---

## coroutine 的状态陷阱

例如：

```lua
money = 100

function foo()
    local old = money

    local x = skynet.call(other, "lua", "xxx")

    money = old + x
end
```

执行过程可能是：

```
Coroutine A

money = 100
old = 100
    ↓
skynet.call()
    ↓
yield
```

在 A 挂起期间，另一个 coroutine 可能执行：

```
Coroutine B

money = 200
```

之后 A 恢复：

```
Coroutine A

old == 100
money == 200
```

但 A 仍然执行：

```
money = old + x
```

于是可能覆盖 Coroutine B 已经做出的修改。

所以必须记住：

**一条 coroutine 挂起以后，Service 内部状态可能已经被其他 coroutine 修改。**

因此在调用可能 `yield` 的 API 之后，例如：

```
skynet.call(...)
skynet.sleep(...)
socket.read(...)
```

恢复执行时，不能默认之前观察到的状态仍然成立。

可以把一次 `yield` 粗略理解成：

```
这里允许其他业务逻辑插进来执行
```

因此恢复后往往需要重新检查共享状态。

---

## `skynet.fork` 不是真正的 OS 线程

Skynet 中有：

```lua
skynet.fork(function()
    ...
end)
```

这里虽然有时会使用“线程”这个词，但它实际上并不是：

```
pthread
OS Thread
```

而是：

```
Lua coroutine
```

因此：

```
skynet.fork
≠
pthread_create
```

同一个 Service 内的多个 `skynet.fork`：

```
共享同一个 Lua State
+
协作式调度
+
不会真正并行执行
```

---

## Cluster

Skynet 早期设计曾经追求一种很强的**位置透明性**：

```
Service A
   ↓
send
   ↓
Service B
```

业务层不需要关心：

```
Service B 在本机
还是在另一台机器
```

也就是说，希望把：

```
本地 Service 调用
```

和：

```
远程 Service 调用
```

抽象成几乎完全相同的形式。

后来作者认为：

**把本地消息和跨机器消息完全抽象成一样，并不是一个好的设计。**

原因是两者本质上存在很大的可靠性差异。

本地 Service：

```
同一个进程
   ↓
通信成本低
   ↓
只要进程还在，通常比较可靠
```

而跨机器通信需要面对：

```
网络断开
机器宕机
进程崩溃
超时
网络分区
部分失败
```

这些问题无法通过“位置透明”真正消除。

因此现代 Skynet 更推荐显式使用 Cluster：

```
Skynet Instance A
       │
       │ cluster
       ▼
Skynet Instance B
```

把另一台机器上的 Skynet 实例视为：

```
外部服务
```

并显式处理网络通信可能出现的失败。

因此：

**虽然 Skynet 仍然支持多节点模式，但已经不推荐早期那种把本地调用和远程调用完全透明化的设计。**

---

## 核心理解

这篇文章最重要的几个知识点可以总结成：

```
Skynet
  │
  ├── Service = 逻辑 + 状态
  │
  ├── Service 之间通过消息通信
  │
  ├── 不同 Service 可以多线程并行
  │
  ├── 同一 Service 内通过 coroutine 并发
  │
  ├── coroutine 并不真正并行
  │
  ├── yield 后内部状态可能已经变化
  │
  ├── 不应该阻塞 Worker Thread
  │
  └── 跨节点通信应显式面对网络的不可靠性
```

可以进一步概括为：

**Skynet 用多线程调度大量 Service 来利用多核 CPU；每个 Service 内部再使用 Lua coroutine 组织异步业务流程。不同 Service 可以真正并行，而同一 Service 内的 coroutine 只是协作式并发。网络 IO、定时器和 Service 间调用都尽量通过异步消息机制完成，从而避免阻塞有限的 Worker Thread。**


---


<br/>


<br/>


# [Bootstrap](https://github.com/cloudwu/skynet/wiki/Bootstrap)

**Bootstrap** 主要讲的是一件事：**一个 Skynet 节点从执行 `skynet config` 开始，到最终启动用户自己的 `main.lua`，中间到底经历了什么。**


## Skynet 如何启动

启动一个 Skynet 节点时，需要执行：

```
./skynet config
```

其中 `config` 是 Skynet 的配置文件。

整个启动流程可以先粗略理解为：

```
./skynet config
       ↓
读取配置文件
       ↓
初始化 Skynet C Core
       ↓
启动基础 Service
       ↓
启动用户 Service
```

Skynet 的一个运行进程可以称为一个 **Skynet Node**。

---

## 第一个 Service：logger

Skynet 启动后，第一个创建的 Service 是：

```
logger
```

`logger` 用于处理 Skynet 中其他 Service 的日志输出。

例如：

```
Service
   │
   │ skynet_error(...)
   ▼
 logger
   │
   ├── stdout
   └── log file
```

配置文件中的：

```
logger = ...
```

可以指定日志文件。

如果没有设置，则日志默认输出到标准输出。

---

## 第二个 Service：bootstrap

第二个启动的 Service 由配置文件中的：

```
bootstrap = ...
```

决定。

默认类似：

```
bootstrap = "snlua bootstrap"
```

这里可以拆成两个部分：

```
snlua
  │
  └── 要启动的底层 Service

bootstrap
  │
  └── 传递给 snlua 的参数
```

因此：

```
bootstrap = "snlua bootstrap"
```

可以理解为：

> 创建一个 `snlua` Service，并让它加载并执行 `bootstrap.lua`。

整体过程：

```
启动 snlua
    ↓
创建 Lua State
    ↓
参数 = bootstrap
    ↓
查找 bootstrap.lua
    ↓
执行 service/bootstrap.lua
```

---

## snlua 是什么

`snlua` 可以粗略理解为：

> **承载 Lua Service 的底层 C Service。**

Lua Service 本身最终运行在 Lua VM / Lua State 中，而 **`snlua` 负责创建这个 Lua 环境并加载对应的 Lua 文件**。

可以粗略理解为：

```
Lua Service
    ↓
snlua
    ↓
Lua State
    ↓
xxx.lua
```

例如：

```
skynet.newservice("foo")
```

底层可以粗略理解为：

```
创建 snlua
    ↓
创建 Lua State
    ↓
加载 foo.lua
    ↓
形成新的 Lua Service
```

因此：

```
snlua bootstrap
```

本质上就是：

```
创建 Lua Service
      ↓
执行 bootstrap.lua
```

---

## bootstrap.lua 的作用

`bootstrap.lua` 并不是用户业务代码，而是：

> **Skynet 自身的系统启动 Service。**

它负责把 Skynet 所需要的一系列基础 Service 启动起来。

整个关系不是：

```
Skynet
  ↓
main.lua
```

而是：

```
Skynet C Core
      ↓
logger
      ↓
snlua bootstrap
      ↓
bootstrap.lua
      ↓
系统基础 Service
      ↓
用户 main.lua
```

---

##  launcher

`bootstrap.lua` 会启动：

```
skynet.launch("snlua", "launcher")
```

然后注册名字：

```
.launcher
```

`launcher` 可以粗略理解为：

> **负责创建和管理其他 Service 的基础 Service。**

以后使用：

```
skynet.newservice(...)
```

创建 Service 时，底层流程和 `launcher` 有密切关系。

因此它需要在系统启动的早期阶段建立起来。

---

## 单节点与多节点

bootstrap 启动过程中还会根据配置判断当前 Skynet 是：

```
单节点
```

还是：

```
多节点
```

### 单节点模式

如果：

```
harbor = 0
```

表示当前节点不需要使用旧的 Harbor 多节点通信机制。

此时会使用：

```
cdummy
```

处理一些和全局名字相关的逻辑。

可以简单记成：

```
单节点
  ↓
cdummy
```


### 旧版多节点模式

Skynet 早期多节点体系中使用：

```
cmaster
cslave
```

结构大致为：

```
Master Node
   │
   ├── cmaster
   └── cslave


Slave Node
   │
   └── cslave
```

它们负责节点之间：

```
通信
消息转发
全局名字同步
```

不过现在 Skynet 已经不推荐这种强调“位置透明”的 Harbor 多节点模式。

现代 Skynet 更推荐：

```
Skynet Instance A
       │
     cluster
       │
       ▼
Skynet Instance B
```

显式处理不同 Skynet 实例之间的网络通信。

因此学习 Bootstrap 时，对：

```
cmaster
cslave
cdummy
harbor
```

理解其历史启动作用即可。

---

##  DataCenter

在某些运行模式下，bootstrap 还会启动：

```
datacenterd
```

并注册：

```
DATACENTER
```

DataCenter 可以粗略理解为：

> Skynet 中用于保存和共享一些全局数据的 Service。

当前阶段知道：

```
bootstrap
    ↓
datacenterd
    ↓
DATACENTER
```

即可。

---

## service_mgr

随后 bootstrap 会启动：

```
skynet.newservice("service_mgr")
```

`service_mgr` 主要负责管理：

```
UniqueService
```

所谓 UniqueService，就是：

> 确保某一种 Service 只被创建一次。

例如：

```
第一次：

uniqueservice("database")
       ↓
创建 database Service


第二次：

uniqueservice("database")
       ↓
发现已经存在
       ↓
直接返回已有 Service
```

因此可以理解为：

```
service_mgr
     ↓
管理唯一 Service
```

---

## bootstrap 和 start 的区别

这是 Bootstrap 启动流程中最容易混淆的一点。

配置中的：

```
bootstrap = "snlua bootstrap"
```

指定的是：

> **Skynet 框架自己的启动入口。**

也就是：

```
bootstrap
    ↓
service/bootstrap.lua
```

而真正决定：

> **用户业务从哪里开始运行**

的是：

```
start
```

例如：

```
start = "main"
```

表示：

```
bootstrap.lua
     ↓
读取 start
     ↓
main
     ↓
创建 main Service
     ↓
执行 main.lua
```

如果没有显式设置：

```
start
```

通常默认使用：

```
main
```

因此要牢记：

```
bootstrap
   │
   └── Skynet 自己的启动入口


start
   │
   └── 用户业务的启动入口
```

---

## 为什么推荐使用不同 config

如果有多个测试程序或不同应用，不应该反复修改：

```
bootstrap.lua
```

而应该准备不同的配置文件。

例如：

```
config.game
config.test
config.db
```

分别设置：

```
-- config.game
start = "game"

-- config.test
start = "test"

-- config.db
start = "database_test"
```

然后使用：

```
./skynet config.game
```

或者：

```
./skynet config.test
```

这样同一套 Skynet 可以启动不同的应用程序。

---

## bootstrap 为什么最后会退出

`bootstrap.lua` 的任务只是：

> **把整个 Skynet 系统启动起来。**

例如：

```
bootstrap
   ↓
启动 launcher
   ↓
启动节点相关 Service
   ↓
启动 DataCenter
   ↓
启动 service_mgr
   ↓
启动 main
```

完成这些工作后：

```
bootstrap
```

本身已经没有继续存在的必要。

因此最后会调用：

```
skynet.exit()
```

即：

```
bootstrap
   ↓
完成系统初始化
   ↓
启动用户 Service
   ↓
退出
```

---

## 完整启动流程

整个 Skynet 启动过程可以概括为：

```
             ./skynet config
                    │
                    ▼
              读取配置文件
                    │
                    ▼
             Skynet C Core
                    │
                    ▼
                logger
                    │
                    ▼
            snlua bootstrap
                    │
                    ▼
             bootstrap.lua
                    │
        ┌───────────┼────────────┐
        │           │            │
        ▼           ▼            ▼
    launcher    节点服务     DataCenter
                    │
             ┌──────┴──────┐
             ▼             ▼
          单节点          多节点
             │             │
          cdummy      cmaster/cslave

                    │
                    ▼
               service_mgr
                    │
                    ▼
             读取 config.start
                    │
                    ▼
                 main
                    │
                    ▼
                main.lua
                    │
                    ▼
              用户业务 Service

同时：

bootstrap 完成初始化
        ↓
   skynet.exit()
```

---


<br/>


<br/>

# [Config](https://github.com/cloudwu/skynet/wiki/Config)

这篇 **Config** 主要讲的是：**Skynet 的配置文件怎么写，以及各个配置项分别控制什么。**


## Config 本质上是一段 Lua 代码


启动 skynet 的命令：

```bash
./skynet config
```

这里的 config 并不是简单的配置文件，而是一段 Lua 代码，所以使用 Lua 的变量和字符串拼接，而不只是简单的静态键值对。


---

## Config 最重要的机制：env


Skynet 启动时会读取它自己需要的配置项。暂时没有直接使用的配置项，会以**字符串形式**保存在 Skynet 内部的：env。

Lua Service 可以通过：

```lua
skynet.getenv("xxx")
```

读取。

例如配置：

```
mysql_host = "127.0.0.1"
mysql_port = 3306
```

业务 Service 可以：

```
local skynet = require "skynet"

local host = skynet.getenv("mysql_host")
```

---

<br/>


<br/>




# [LuaAPI](https://github.com/cloudwu/skynet/wiki/LuaAPI)

这部分讲述一个 Lua Service 如何启动、如何找到其他 Service、如何收发消息、如何 RPC、如何使用 coroutine/定时器，以及如何退出和调试。

## Lua Service 到底是什么？

Skynet 本身使用 C 编写，理论上 Service 可以直接使用 C 实现。Skynet 提供了一个 C Service——snlua：

```text
snlua
  ↓
创建 Lua VM / Lua State
  ↓
加载某个 .lua 文件
  ↓
形成 Lua Service
```

Lua Service 中通常第一行就是：

```lua
local skynet = require "skynet"
```

这里的 `skynet` 模块**不能脱离 Skynet 单独用普通 Lua 解释器运行**，因为每个 Skynet Service 都依赖底层的 `skynet_context`。


每个 skynet 服务，最重要的职责就是处理别的服务发送过来的消息，以及向别的服务发送消息。每条 skynet 消息由五个元素构成：
- session：一个非负整数，使用 0 这个特殊号码代表一条消息不需要回应。
- source：每个服务由一个 32 位整数标识，新的启动的服务不会使用已用过的地址。
- type：消息类别。最常用的消息类别“lua”，广泛用于 skynet 服务间通讯。
- message：消息都 C 指针，在 Lua 层看来是一个 lightuserdata。
- size：消息长度。

---

## 服务地址

每个服务都有一个 32 bit 的数字地址，这个地址的高 8 bit 表明了它所属的节点。

- `skynet.self()` 用于获得服务自己的地址。
- `skynet.harbor()` 用于获得服务所属的节点。
- `skynet.address(address)` 用于把一个地址数字转换为一个可用于阅读的字符串。
- `skynet.register(name)` 可以为自己注册一个别名。（别名必须在 16 个字符以内）
- `skynet.name(name, address)` 为一个地址命名。`skynet.name(name, skynet.self())` 和 `skynet.register(name)` 功能等价。


**本地名字**：以 `.` 开头，只在当前 Skynet Node 中有效。
- `skynet.localname()`：查询地址。

**全局名字**：以普通字母开头。


---

## 消息分发和回应

注册消息处理函数：

```lua
local CMD = {}

function CMD.hello(name)
    print("hello", name)
end

skynet.dispatch("lua", function(session, source, cmd, ...)
    local f = assert(CMD[cmd])
    f(...)
end)
```

当其他 Service 可以发送：

```lua
skynet.send(addr, "lua", "hello", "xiaofeng")
```

**每收到一条消息，Skynet Lua 层会用一个独立 coroutine 来处理**。



---


## 消息的序列化


---


## 消息推送和远程调用


---

## 服务的启动和退出



---