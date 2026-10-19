# [Skynet 设计综述](https://blog.codingnow.com/2012/09/the_design_of_skynet.html)

Skynet 可以理解为一个基于 Actor 思想的服务器框架：

**用一个较小的多线程消息调度核心，把业务拆成许多通过消息通信的 Service；框架保证同一个 Service 的消息处理串行化，而不同 Service 可以被多个 Worker 并行调度，再利用 Lua coroutine 把异步消息流程包装成容易编写的顺序业务代码。**

> Actor 模型是一种**并发编程模型**。核心思想是把系统拆分成许多相对独立的 Actor，每个 Actor 管理自己的状态，通过**消息传递**进行通信。
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

Skynet Service 可以使用 C 编写，也可以使用 Lua 编写。对于 Lua Service，可以粗略理解为：

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

接收方可以直接访问这块内存，从而实现零拷贝。但是这样需要非常小心地管理：

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
source：谁发送的

session： 请求/响应匹配

data + sz：消息内容
```

此外还有一个非常重要的概念：

```text
type
```

在当时的实现中，`type` 并没有作为独立字段直接放进这个结构，而是编码进了 `size` 的高位。

---

## session

Skynet 核心提供的是：单向消息。但是业务中经常需要对发来的消息进行响应，例如：

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

A 发出请求时获得一个 session，然后记录 session 对应哪个请求。B 处理完成以后，把相同的 session 带回来。于是 A 收到 session 后，就知道这个响应属于之前的哪个请求。

因此： **session 本质上解决的是异步消息环境中的请求—响应匹配问题。**

---

## type

`type` 更准确地说表示： **当前消息使用哪一组通信协议。**

而不仅仅是普通意义上的“消息类别”。不同消息可能使用：

```text
TEXT
RESPONSE
CLIENT
HARBOR
MULTICAST
...
```

**不同协议可以有不同的数据编码和处理方式**。于是可以：

```text
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

```text
Service A → Queue A

Service B → Queue B

Service C → Queue C
```

每个 Service 都拥有自己的消息队列。

发送消息：

```text
send → Service B
```

实际上就是把消息加入 Queue B 。


### 第二级：Global Queue

Skynet 还有一个全局消息队列：

```text
Global Queue

┌─────────┐
│ Queue A │
│ Queue C │
│ Queue F │
└─────────┘
```

里面存放的是：**当前有消息需要处理的 Service 的 私有Queue。**

Worker Thread：

```text
          Global Queue
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
   Worker1  Worker2  Worker3
```

Worker 从 Global Queue 中取出一个 Service Queue，然后：

```text
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

```text
Queue Empty
    ↓
不再放回 Global Queue
```

因此没有工作的 Service 不会产生 CPU 空转。

### 最关键的线程安全保证

同一个 Service 的消息队列被一个 Worker 处理时，其他 Worker 不会同时获得 Service A 的消息队列。因此不会出现同一个 service 被多个线程同时执行的情况。

Skynet 实现的是：

```text
Service 内部串行
+
Service 之间并行
```

这也是 Actor 模型在 Skynet 中非常重要的体现。

在文章描述的设计中，每次从 Service Queue 中只处理一条消息，然后就重新参与调度。

这样虽然局部效率稍低，但能够提高公平性，避免某个消息特别多的 Service 长时间霸占 Worker。

---

## Lua coroutine 的作用

Lua coroutine 可以把底层 异步消息，包装成上层看起来类似 同步函数调用。例如：

```lua
local result = skynet.call(service, "lua", "query")
print(result)
```

代码看起来像：

```text
调用
 ↓
等待
 ↓
得到结果
```

但底层实际类似：

```text
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

```text
response
session = 123
      ↓
查表
      ↓
找到 Coroutine A
      ↓
resume
```

于是 Lua 代码就可以继续执行。

coroutine 并没有把异步 IO 变成真正的同步阻塞 IO，而只是让**异步状态机更容易用顺序代码表达**。

---

## Gate

Skynet 内部使用的是：

```text
Message
```

但外部客户端使用的是：

```text
TCP byte stream
```

因此需要一个组件把：TCP 和 Skynet Message 连接起来。

这个组件就是：Gate。大致为：

```text
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

```text
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

Gate 自己尽量不处理具体业务逻辑。业务逻辑可以交给 Lua Service。

文章当时还设计了：

```text
watchdog
agent
broker
```

等不同的数据转发方式。需要注意的是，在文章描述的设计中：**Gate 主要负责读取外部连接，并不负责向客户端回写数据。** 发送数据由其他专门的 Service 完成。

---

## Connection

Gate 解决的是：

```text
外部客户端
    ↓
Skynet
```

而 Connection 主要解决：

```text
Skynet
    ↓
外部服务
```

例如：

```text
Skynet
   │
   ├── Redis
   ├── MySQL
   └── 其他 TCP 服务
```

Connection 使用 epoll 监听外部 fd 的可读事件。收到数据以后：

```text
socket readable
       ↓
Connection
       ↓
Skynet message
       ↓
目标 Service
```

这样 Service 就不需要阻塞 recv() 等待外部服务返回数据。

这是非常重要的，因为：

```text
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

```text
单进程
+
多线程
+
Service
```

但它也考虑了多个 Skynet 节点之间的通信。

Service 地址是 32 bit：

```text
┌──────────┬──────────────────────┐
│ 高 8 bit │      低 24 bit      │
│ Harbor ID│   Service Handle     │
└──────────┴──────────────────────┘
```

因此可以根据 Service 地址判断：

```text
目标 Service
    │
    ├── 当前节点 → 本地消息
    │
    └── 其他节点 → 远程消息
```

如果：

```text
Service A → Service B
```

而 B 位于其他 Skynet 节点：

```text
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

Harbor 负责不同 Skynet 节点之间的数据转发。这样业务 Service 使用消息接口时，不需要过度关心：目标 Service 到底在本机，还是远程机器


---

## 名字服务和 Master

数字 handle：

```text
:01000042
```

并不适合人直接使用。

因此 Skynet 支持给 Service 起名字，例如：

```text
login
database
chat
```

跨 Skynet 节点时，还需要解决：

```text
service name
      ↓
在哪个节点？
      ↓
具体 handle 是什么？
```

于是文章中引入了：

```text
Master
```

Master 用于同步：

```text
全局 Service 名字
+
Skynet 节点地址
```

可以粗略理解成一个全局的 name → address 管理服务。

---

## 组播

游戏服务器中经常存在一条消息发送给很多 Service 的情况，如果每次都复制完整消息。会造成额外开销。因此组播消息采用：

```text
Message
   +
reference count
```

让多个接收方共享同一个消息数据。但是 Skynet 没有让普通单播消息也全部使用引用计数。

原因是：

```text
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

游戏中的玩家、场景、副本等通常存在大量需要长期维护的上下文状态。如果每次业务操作都依赖数据库读取和写入，数据库很容易成为性能瓶颈。

因此 Skynet 更倾向于：Service = 业务逻辑+ 内存中的业务状态。例如玩家上线时：

```text
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

```text
持久化
备份
故障恢复
```

而不是每个业务操作都同步访问数据库。


**Skynet 提供 Service 和消息机制，但 Service 应该如何拆分，需要由具体业务决定。**

---

## 网络 IO

Skynet 中可能同时存在大量 Service，但真正执行 Service 代码的 Worker Thread 数量是有限的，通常与 CPU 核心数量相关。

可以粗略理解为：

```text
大量 Service
    ↓
少量 Worker Thread
    ↓
多核 CPU
```


如果某个 Service 直接执行阻塞式网络 IO，那么：

```text
Service A
   ↓
recv()
   ↓
阻塞
   ↓
Worker Thread 也被阻塞
```

此时受影响的不只是 Service A，因为这个 Worker Thread 暂时无法继续调度其他 Service。

因此：**不要在 Skynet Service 中直接使用可能阻塞的系统网络 API，而应该使用 Skynet 提供的网络层。**

正确的思路更接近：

```text
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

一个 Service 内部可以同时存在多个 Lua coroutine，例如：

```lua
local result = skynet.call(service, "lua", "query")
```

执行到 `skynet.call` 后，如果需要等待远程 Service 返回结果：

```text
Coroutine A
    ↓
发送 request
    ↓
yield
```

此时只是，Coroutine A 被挂起，并不是整个 Service 被阻塞，更不是Worker Thread 被阻塞。在等待期间，该 Service 仍然可以处理其他消息：

```text
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

```text
response
   ↓
根据 session 找到 Coroutine A
   ↓
resume
```

A 再继续执行。

因此：**Skynet Service 在一条业务流程挂起等待响应时，仍然可以处理其他消息。**

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

```text
Coroutine A

money = 100
old = 100
    ↓
skynet.call()
    ↓
yield
```

在 A 挂起期间，另一个 coroutine 可能执行：

```text
Coroutine B

money = 200
```

之后 A 恢复：

```text
Coroutine A

old == 100
money == 200
```

但 A 仍然执行：

```text
money = old + x
```

于是可能覆盖 Coroutine B 已经做出的修改。

所以必须记住：**一条 coroutine 挂起以后，Service 内部状态可能已经被其他 coroutine 修改。**

因此在调用可能 `yield` 的 API 之后，例如：

```lua
skynet.call(...)
skynet.sleep(...)
socket.read(...)
```

恢复执行时，不能默认之前观察到的状态仍然成立。恢复后往往需要重新检查共享状态。

---

## Cluster

Skynet 早期设计曾经追求一种很强的**位置透明性**：

```text
Service A
   ↓
send
   ↓
Service B
```

业务层不需要关心：Service B 在本机，还是在另一台机器？


也就是说，希望把本地 Service 调用和远程 Service 调用，抽象成几乎完全相同的形式。

后来作者认为：**把本地消息和跨机器消息完全抽象成一样，并不是一个好的设计。**

原因是两者本质上存在很大的可靠性差异。

本地 Service：

```text
同一个进程
   ↓
通信成本低
   ↓
只要进程还在，通常比较可靠
```

而跨机器通信需要面对：

```text
网络断开
机器宕机
进程崩溃
超时
网络分区
部分失败
```

这些问题无法通过“位置透明”真正消除。因此现代 Skynet 更推荐显式使用 Cluster 集群：

```text
Skynet Instance A
       │
       │ cluster
       ▼
Skynet Instance B
```

把另一台机器上的 Skynet 实例视为：外部服务，并显式处理网络通信可能出现的失败。

因此：**虽然 Skynet 仍然支持多节点模式，但已经不推荐早期那种把本地调用和远程调用完全透明化的设计。**

> 在 Skynet 中，`cluster` 通常指集群通信机制，用于连接多个独立运行的 Skynet 节点，让它们能够相互调用 Service。


---

## 核心理解

概括为：

Skynet 用多线程调度大量 Service 来利用多核 CPU；每个 Service 内部再使用 Lua coroutine 组织异步业务流程。不同 Service 可以真正并行，而同一 Service 内的 coroutine 只是协作式并发。网络 IO、定时器和 Service 间调用都尽量通过异步消息机制完成，从而避免阻塞有限的 Worker Thread。


---

# [Bootstrap](https://github.com/cloudwu/skynet/wiki/Bootstrap)

**Bootstrap** 主要讲的是一件事：**一个 Skynet 节点从执行 `skynet config` 开始，到最终启动用户自己的 `main.lua`，中间到底经历了什么。**


## Skynet 如何启动

启动一个 Skynet 节点时，需要执行：

```bash
./skynet config
```

其中 `config` 是 Skynet 的配置文件。

整个启动流程可以先粗略理解为：

```text
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

Skynet 启动后，第一个创建的 Service 是：logger，用于处理 Skynet 中其他 Service 的日志输出。

配置文件中的：

```lua
logger = ...
```

可以指定日志文件。如果没有设置，则日志默认输出到标准输出。

---

## 第二个 Service：bootstrap

第二个启动的 Service 由配置文件中的：

```lua
bootstrap = ...
```

决定。默认类似：

```lua
bootstrap = "snlua bootstrap"
```

这里可以拆成两个部分：

```text
snlua： 要启动的底层 Service

bootstrap：传递给 snlua 的参数
```

因此可以理解为：创建一个 `snlua` Service，并让它加载并执行 `bootstrap.lua`。

整体过程：

```text
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

`snlua` 可以粗略理解为： **承载 Lua Service 的底层 C Service。**

Lua Service 本身最终运行在 Lua VM / Lua State 中，而 **`snlua` 负责创建这个 Lua 环境并加载对应的 Lua 文件**。

例如：

```lua
skynet.newservice("foo")
```

底层可以粗略理解为：

```text
创建 snlua
    ↓
创建 Lua State
    ↓
加载 foo.lua
    ↓
形成新的 Lua Service
```

每个 Lua Service 本身就是一个由 `snlua` 实现的 Service 实例。`snlua` 是其底层实现类型。不过，`logger`、`harbor` 等服务可以由 C 层模块实现，不需要通过 `snlua` 来运行 Lua 脚本。

---

## bootstrap.lua 的作用

`bootstrap.lua` 并不是用户业务代码，而是： **Skynet 自身的系统启动 Service。**

它负责把 Skynet 所需要的一系列基础 Service 启动起来。

```text
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

```lua
skynet.launch("snlua", "launcher")
```

然后注册名字：`.launcher`，`launcher` 可以粗略理解为： **负责创建和管理其他 Service 的基础 Service。**

以后使用：

```lua
skynet.newservice(...)
```

创建 Service 时，底层流程和 `launcher` 有密切关系。因此它需要在系统启动的早期阶段建立起来。

---

## 单节点与多节点

bootstrap 启动过程中还会根据配置判断当前 Skynet 是：单节点，还是多节点？

### 单节点模式

如果：

```lua
harbor = 0
```

表示当前节点不需要使用旧的 Harbor 多节点通信机制。此时会使用cdummy处理一些和全局名字相关的逻辑。

### 旧版多节点模式

Skynet 早期多节点体系中使用：

```text
cmaster
cslave
```

结构大致为：

```text
Master Node
   │
   ├── cmaster
   └── cslave


Slave Node
   │
   └── cslave
```

它们负责节点之间：

```text
通信
消息转发
全局名字同步
```

不过现在 Skynet 已经不推荐这种强调“位置透明”的 Harbor 多节点模式。现代 Skynet 更推荐：

```text
Skynet Instance A
       │
     cluster
       │
       ▼
Skynet Instance B
```

显式处理不同 Skynet 实例之间的网络通信。

---

##  DataCenter

在某些运行模式下，bootstrap 还会启动：datacenterd

并注册：

```
DATACENTER
```

DataCenter 可以粗略理解为： Skynet 中用于保存和共享一些全局数据的 Service。

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

```lua
skynet.newservice("service_mgr")
```

`service_mgr` 主要负责管理UniqueService。

---

## bootstrap 和 start 的区别

这是 Bootstrap 启动流程中最容易混淆的一点。

配置中的：

```lua
bootstrap = "snlua bootstrap"
```

指定的是： **Skynet 框架自己的启动入口。**

而真正决定： **用户业务从哪里开始运行** 的是：

```lua
start
```

如果没有显式设置 start，通常默认使用 main。


---

## 完整启动流程

整个 Skynet 启动过程可以概括为：

```text
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


Skynet 启动时会读取它自己需要的配置项。暂时没有直接使用的配置项，会以**字符串形式**保存在 Skynet 内部的 env。

Lua Service 可以通过：

```lua
skynet.getenv("xxx")
```

读取。

例如配置：

```lua
mysql_host = "127.0.0.1"
mysql_port = 3306
```

业务 Service 可以：

```lua
local skynet = require "skynet"

local host = skynet.getenv("mysql_host")
```

---


# [LuaAPI](https://github.com/cloudwu/skynet/wiki/LuaAPI)

这一部分主要讲述**一个 Lua Service 如何启动、如何定位其他 Service、如何收发消息和进行 RPC、如何使用 coroutine 和定时器，以及如何退出和调试。**


## Lua Service 到底是什么？

Skynet 框架本身主要使用 C 编写，Service 也可以直接使用 C 实现。Skynet 提供了一个 C Service —— `snlua`：

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

这里的 `skynet` 模块**不能脱离 Skynet，直接使用普通 Lua 解释器运行**，因为每个 Skynet Service 都依赖底层的 `skynet_context` C 对象，该对象由 `snlua` 注入 Lua VM。

一条 Skynet 消息主要由五个元素构成：

- `session`：请求—响应匹配标识，由请求方生成并由 Skynet 管理。按惯例，**`session = 0` 表示这条消息不需要回应**。
- `source`：消息来源 Service 的地址。每个 Service 使用一个 32 bit 整数标识。Service 退出后，新 Service 通常不会立即复用这个地址，除非地址空间发生回绕。
- `type`：消息类别。一个 Service 最多可以接收 256 种消息类别，不同类别可以使用不同的编码方式。最常用的是 `"lua"`。
- `message`：底层消息数据的 C 指针，在 Lua 层表现为 `lightuserdata`。
- `size`：消息数据长度。

通常业务代码不需要直接操作 `message + size`，因为协议的 `unpack` 会把底层数据转换成 Lua 对象。

---

## Service 地址

每个 Service 都有一个 32 bit 数字地址。在 LuaAPI 描述的旧式 Harbor 多节点模型中：

```text
┌──────────┬──────────────────────┐
│ 高 8 bit │      低 24 bit      │
│ 节点编号 │   Service Handle     │
└──────────┴──────────────────────┘
```

常用 API：

```lua
skynet.self()
```

获得当前 Service 自己的地址。

```lua
skynet.harbor()
```

获得当前 Service 所属的节点编号。

```lua
skynet.address(address)
```

把数字地址转换成方便阅读的字符串形式。

### Service 名字

以下 API 属于 `skynet.manager` 扩展接口，使用前需要：

```lua
require "skynet.manager"
```

给当前 Service 注册名字：

```lua
skynet.register(name)
```

名字不能超过 16 个字符。

给指定 Service 地址命名：

```lua
skynet.name(name, address)
```


名字分为两种:

**本地名字**以 `.` 开头：

```text
.database
.launcher
```

只在当前 Skynet Node 内有效。

可以通过：

```lua
skynet.localname(".database")
```

查询对应地址。

`skynet.localname(name)` 是一个**非阻塞 API**。


**全局名字**以普通字母开头，在旧 Harbor 多节点模型中可以用于跨节点通信。不过不建议过度依赖全局名字。通常更推荐在业务层交换并保存 Service 的数字地址。


> *可以给指定服务注册一个本地名字，之后就可以通过注册的名字来和这个服务进行通信交互，避免了传地址的麻烦。*
> 另外全局名字只在 harbor 不为 0 的情况才有意义，如果是 harbor 为 0，则使用 cluster.register 注册名字，而且这个名字而不是全局名字，需要指定 node 。


---

## 消息协议与分发

### `skynet.dispatch`

注册某种消息类型的处理函数：

```lua
skynet.dispatch(type, function(session, source, ...)
    ...
end)
```

最常见的是 `"lua"`：

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

其他 Service 可以发送：

```lua
skynet.send(addr, "lua", "hello", "xiaofeng")
```

然后接收方执行：

```text
"hello"
   ↓
CMD["hello"]
   ↓
CMD.hello("xiaofeng")
```

这种 `CMD[cmd]` 写法只是 Skynet 中非常常见的惯例，并不是框架强制规定。

### 消息协议

Skynet 还允许通过：

```lua
skynet.register_protocol {
    ...
}
```

注册新的消息协议。 每种协议最重要的是提供：

```text
pack
  ↓
Lua 数据 → 消息数据

unpack
  ↓
消息数据 → Lua 数据
```

通常业务开发直接使用 Skynet 已经注册好的 `"lua"` 协议即可。

---

## 每条消息由独立 coroutine 处理

每收到一条消息，Skynet Lua 层都会使用一个独立 coroutine 处理：

```text
Message A → Coroutine A

Message B → Coroutine B

Message C → Coroutine C
```

同一个 Lua State 在任意时刻只有一条 coroutine 真正在执行。不过当 Coroutine A 调用了可能阻塞的 Skynet API：

```text
Coroutine A
    ↓
skynet.call()
    ↓
yield

Coroutine B
    ↓
开始处理另一条消息
```

Service 就可能发生**业务逻辑重入**。

因此，Lua Service 通常不需要考虑传统意义上的线程数据竞争，但必须注意 coroutine yield 后发生的**逻辑并发问题**。

---

## 消息的回应

如果其他 Service 使用：

```lua
skynet.call(...)
```

请求当前 Service，当前 Service 通常需要发送回应。可以使用：

```lua
skynet.ret(skynet.pack(result))
```

常见写法：

```lua
skynet.dispatch("lua", function(session, source, cmd, ...)
    local f = assert(CMD[cmd])

    skynet.ret(
        skynet.pack(f(...))
    )
end)
```

需要注意：**对于同一个消息处理 coroutine，** `skynet.ret` **只能调用一次。**

如果暂时不能立即回应，可以使用：

```lua
local response = skynet.response()
```

保存一个回应闭包。以后：

```lua
response(true, result)
```

发送正常响应。也可以：

```lua
response(false)
```

通知调用方该请求失败。


`skynet.ret` 和 `skynet.response` 都是**非阻塞 API**。

如果收到 `session ~= 0` 的请求，但明确不准备回应，可以使用：

```lua
skynet.ignoreret()
```

告诉框架忽略这次响应，否则 Skynet 可能记录未响应请求的日志。

---

## 消息的序列化

Skynet 的 `"lua"` 协议默认使用：

```lua
skynet.pack(...)
skynet.unpack(...)
```

序列化 Lua 数据。例如：

```lua
local msg, size = skynet.pack({
    name = "Felix",
    level = 10,
})
```

返回：

```text
lightuserdata + size
```

也就是一块连续的底层内存及其长度。

反序列化：

```lua
local data = skynet.unpack(msg, size)
```

如果不是直接把数据交给 Skynet 消息框架，而只是希望序列化成 Lua 字符串，可以使用：

```lua
local data = skynet.packstring(...)
```

默认序列化支持：

```text
string
boolean
number
lightuserdata
table
```

但对带复杂 metatable / metamethod 的 Lua 对象支持有限。

需要注意：**并不是所有 Service 间通信都必须使用** `skynet.pack` **。**  Skynet 的消息协议可以自定义，只是 `"lua"` 协议通常使用 `pack/unpack`。

---

## 消息推送与 RPC

### `skynet.send`

```lua
skynet.send(address, "lua", ...)
```

用于发送一条消息，但**不等待回应**。


因此它是：异步、单向、非阻塞。而且 `send` 本身不会产生 yield，所以当前 coroutine 在 `send` 调用期间不会因为这个 API 发生重入。

### `skynet.call`

需要等待对方返回结果时：

```lua
local result = skynet.call(address, "lua", ...)
```

过程类似：

```text
Service A

生成 session
     ↓
发送 request
     ↓
当前 coroutine yield
     ↓

Service B
     ↓
处理请求
     ↓
response

Service A
     ↓
根据 session 找回 coroutine
     ↓
resume
```

因此从业务代码来看非常像RPC

但要特别注意：`skynet.call` **阻塞的是当前 coroutine，而不是整个 Service，更不是 Worker Thread。**

等待响应时，当前 Service 仍然可以处理其他消息。所以还是需要考虑逻辑并发问题。

还有一个重要限制：`skynet.call` **本身没有内建超时机制。** 如果对方一直不回应，当前 coroutine 就会一直等待，需要业务层自己实现超时控制。

### 常用消息 API

|API|是否等待结果|当前 coroutine|
|---|---|---|
| `skynet.send` |否|继续运行|
| `skynet.call` |是|yield|
| `skynet.ret` |返回结果|不阻塞|
| `skynet.response` |延迟返回|不阻塞|

---

## Service 的启动和退出

### `skynet.start`

每个 Lua Service 都必须注册启动函数：

```lua
skynet.start(function()
    ...
end)
```

Lua 文件顶层代码仍然会先执行：

```lua
local skynet = require "skynet"

-- 顶层代码先执行

skynet.start(function()
    -- Service 正式初始化
end)
```

但是：**不要在** `skynet.start` **外部调用 Skynet 的阻塞 API。** 因为此时框架还无法正确唤醒被挂起的 coroutine。


### `skynet.init`

如果某个 Lua 库需要在 `start` 之前执行初始化逻辑，可以：

```lua
skynet.init(function()
    ...
end)
```

它尤其适合库代码注册初始化任务。


### `skynet.newservice`

这是一个**阻塞 API**，所以不要在新 Service 的 `start` 中写：

```lua
while true do
    ...
end
```

否则 `newservice` 永远不会返回。

如果 Service 初始化失败，`newservice` 会抛出异常。

另外需要注意：

```lua
skynet.newservice("foo", arg1, arg2)
```

这些启动参数底层实际上是通过**字符串拼接和拆分**传递的，因此不适合传递复杂 Lua 对象。

更推荐：

```lua
local foo = skynet.newservice("foo")

skynet.call(foo, "lua", "start", complex_config)
```

---

## UniqueService

同一个 Lua 文件可以启动多次：

```text
foo.lua
  ↓
Service A

foo.lua
  ↓
Service B
```

两个 Service 地址不同。

如果希望同名 Service 在一个 Skynet Node 中只存在一个实例，可以使用：

```lua
local addr = skynet.uniqueservice("foobar")
```

和 `newservice` 不同： 同一个名字的 Service 在同一个 Skynet Node 中只会启动一次。

如果该 Service：

```text
已经启动
或
正在启动
```

后续调用：

```lua
skynet.uniqueservice("foobar")
```

都会得到第一次创建的那个 Service 地址。


如果明确知道某个 UniqueService 应该已经存在，可以通过：

```lua
skynet.queryservice("foobar")
```

查询它。如果还没有启动，`queryservice` 会**等待**它启动。

---

## Service 退出

当前 Service 主动退出：

```lua
skynet.exit()
```

调用以后：

```text
当前 Service
   ↓
停止运行
```

而且当前 Service 中仍处于等待状态的 coroutine 也会被中断。

因此调用 `skynet.exit()` 时要注意尚未完成的 RPC。

还可以通过：

```lua
skynet.kill(address)
```

强制关闭其他 Service。但**不推荐这样做**。

更合理的方式是：

```text
Service A
   │
   │ 发送 exit 消息
   ▼
Service B
   │
   ├── 保存状态
   ├── 清理资源
   └── skynet.exit()
```

让 Service 自己完成有序退出。

---

## 时间与 coroutine

Skynet 内部时钟精度为：

```text
1 / 100 秒
=
10 ms
```

### 时间 API

`skynet.now()`

返回 Skynet 节点内部时钟值，单位是 `1/100 s`。

这个数值本身不表示真实 UTC 时间，主要用于：

```
两次 now() 的差值
        ↓
计算经过时间
```


`skynet.starttime()`

返回 Skynet Node 进程启动时的 UTC 时间，单位为秒。


`skynet.time()`

返回当前 UTC 时间，单位为秒，精度约为小数点后两位。

近似：

```
skynet.now() / 100 + skynet.starttime()
```


`skynet.hpc()`

提供高精度计时，适合性能分析，返回纳秒级的 64 位计数值。


## `skynet.sleep`

```lua
skynet.sleep(ti)
```

将当前 coroutine 挂起 `ti` 个时间单位。`sleep` 可能被：

```lua
skynet.wakeup(...)
```

**提前唤醒**。这种情况下它会返回：

```text
"BREAK"
```

---

## `skynet.timeout`

如果希望： 一段时间后执行某个函数，但当前 coroutine 不等待。

可以：

```lua
skynet.timeout(100, function()
    print("1 second later")
end)
```

过程：

```text
当前 coroutine
     │
timeout()
     │
     └──────────────→ 继续执行


1 秒以后
     ↓
新的 coroutine (需要注意：协程对象在调用timeout时就创建了)
     ↓
执行 callback
```



---

## `skynet.fork`

创建一个新的 Lua coroutine：

```lua
skynet.fork(function()
    ...
end)
```

从功能上类似：

```lua
skynet.timeout(0, function()
    ...
end)
```

但 `fork` 不需要向框架注册 timer，因此更加高效。

---

## `skynet.wait / skynet.wakeup`

挂起当前 coroutine：

```lua
skynet.wait(token)
```

另一个 coroutine 可以：

```lua
skynet.wakeup(token)
```

将它唤醒。

`token` 必须能够唯一标识等待者：可以使用协程对象作为 token `coroutine.running()`，其实不传参数时，默认也是使用这个作为 token。

`wakeup` 也可以唤醒通过 `skynet.sleep` 挂起的 coroutine。（正如前面所说的）

---

## `skynet.yield`

如果需要主动交出执行权：

```lua
skynet.yield()
```

相当于：

```lua
skynet.sleep(0)
```

适合：

```text
长时间计算
    ↓
中间没有阻塞 API
    ↓
主动 yield
    ↓
让其他任务获得执行机会
```

---

## 日志与消息跟踪

业务 Service 一般使用：

```lua
skynet.error("hello skynet")
```

写入 Skynet 日志。


### `skynet.trace`

```lua
skynet.trace()
```

可以开启当前消息处理流程的调用链跟踪。

例如：

```
Service A
    ↓ call
Service B
    ↓ call
Service C
```

这些消息可以通过同一个 trace tag 联系起来，对于排查复杂 RPC 调用链很有帮助。

---

## `skynet.manager`

有一些偏底层、普通业务 Service 很少使用的 API，被放在：

```lua
require "skynet.manager"
```

中。

包括：

```text
skynet.launch
skynet.kill
skynet.abort
skynet.register
skynet.name
skynet.forward_type
skynet.filter
skynet.monitor
```

这些 API 更多用于：

```text
Skynet 基础设施
Service 管理
消息转发
系统监控
```

---

# [Coroutine](https://github.com/cloudwu/skynet/wiki/Coroutine)

这部分主要讲述在 Skynet Service 里，为什么不能随便使用 Lua 原生 `coroutine.create/resume/yield`，以及如果真的需要“自己再套一层 coroutine”，应该怎么做。

## 使用 C++理解 Service

可以把一个 Lua Service 想成：

```text
一个 Service
   │
   ├── 一个 Lua State
   │
   └── 很多个 coroutine
           │
           ├── 处理消息 A
           ├── 处理消息 B
           └── skynet.fork(...)
```

这些 coroutine 类似于用户态任务，而不是 std::thread/pthread，真正的线程调度在 Skynet 底层。

---

## 为何不能直接使用 Lua 原生 coroutine？

```lua
coroutine.create()
coroutine.resume()
coroutine.yield()
```

**不要和 skynet 的阻塞 API 混用**。因为 skynet 本身就是靠 `coroutine.yield` 来实现阻塞 API 的。当前 `skynet.lua` 的实现正是维护 `session_id_coroutine` 映射，在 `call` 时挂起，Response 到来后再恢复对应 coroutine。这时候如果套一个原生 coroutine，如下：

```lua
local co = coroutine.create(function()
    local r = skynet.call(db, "lua", "query")
    print(r)
end)

coroutine.resume(co)
```

实际调用栈为：

```text
Skynet 管理的 coroutine A
        │
        │ coroutine.resume
        ▼
   你创建的 coroutine B
        │
        │ skynet.call
        ▼
coroutine.yield("SUSPEND")
```

这会把 skynet 调度器内部的控制消息截胡到自己创建的 coroutine 里面，所以会得到不可预期的返回值，并且打算 skynet 自己的处理流程。

---

## skynet.coroutine

有时候需要实现一个库，确实需要用到 coroutine，可以使用 skynet 封装过后的 coroutine 接口：

```lua
local coroutine = require("skynet.coroutine")
```

`skynet.coroutine` 的 API 基本与 Lua 原生 coroutine 一致。但内部多做了一层：**它知道什么 yield 是你的，什么 yield 是 Skynet 框架自己的。**

**`skynet.coroutine` 最核心的作用：转发 Skynet 的 yield**。

skynet.coroutine 相较于原生 Lua coroutine 新增的 API：

```lua
coroutine.thread(co)
```

用于查询这个嵌套 coroutine 最终属于哪个 skynet coroutine。

例如：

```text
Skynet Message
      ↓
Coroutine A      ← Skynet 管理的根 coroutine
      │
      ▼
Coroutine B      ← skynet.coroutine.create
      │
      ▼
Coroutine C      ← skynet.coroutine.create
```

虽然执行到了 C：

```text
C
 ↓
B
 ↓
A
```

但是对于 Skynet 来说，它真正管理的是：

```text
A
```

该 API 还有第二个返回值：

```lua
local thread, main = coroutine.thread(co)
```

如果 co 是通过 skynet.coroutine 创建并间接驱动的，main 返回 false，否则返回 true。

---


# [CriticalSection](https://github.com/cloudwu/skynet/wiki/CriticalSection)

该部分主要讲解：同一个 Service 虽然不会被多个线程同时执行，但不同 coroutine 会在 `skynet.call / sleep / wait` 这些 yield 点交错执行，所以仍然可能出现“逻辑竞态”。`skynet.queue` 就是用来保护这种临界区的。

## skynet.queue

为了避免 coroutine 因为 yield 而交错执行造成的逻辑竞态，skynet 提供了：

```lua
local queue = require "skynet.queue"
local cs = queue()
```

假设：

```lua
function CMD.foobar()
    cs(func1)
end

function CMD.foo()
    cs(func2)
end
```

queue 可以保证 func 1 和 func 2 不会相互插入执行，即使他们中途可能因为 yield 被挂起。


---


## 锁住的是代码区域

例如：

```lua
function CMD.foobar()
    print("step1")

    cs(function()
        print("step2")
        skynet.call(...)
        print("step3")
    end)

    print("step4")
end
```

只有被 cs 括起来的部分属于临界区。所以 queue 锁住的不是整个 service ，而是代码区域。


---

## 源码实现


```lua
local skynet = require "skynet"
local coroutine = coroutine
local xpcall = xpcall
local traceback = debug.traceback
local table = table

function skynet.queue()
	local current_thread
	local ref = 0
	local thread_queue = {}

	local function xpcall_ret(ok, ...)
		ref = ref - 1
		if ref == 0 then
			current_thread = table.remove(thread_queue,1)
			if current_thread then
				skynet.wakeup(current_thread)
			end
		end
		assert(ok, (...))
		return ...
	end

	return function(f, ...)
		local thread = coroutine.running()
		if current_thread and current_thread ~= thread then
			table.insert(thread_queue, thread)
			skynet.wait()
			assert(ref == 0)	-- current_thread == thread
		end
		current_thread = thread

		ref = ref + 1
		return xpcall_ret(xpcall(f, traceback, ...))
	end
end

return skynet.queue

```




---


# [Socket](https://github.com/cloudwu/skynet/wiki/Socket)


该部分主要内容：Skynet 底层继续使用异步、非阻塞的网络模型，但在 Lua 业务层利用 coroutine，把异步 IO 包装成看起来像阻塞式 `read()` 的顺序代码。

## Skynet Socket 整体结构

底层 socket 线程收到网络事件之后，不会直接调用 Lua 函数，而是把结果包装成 `PTYPE_SOCKET` 消息，然后发送给对应的 Service。

```text
       Linux Socket
            │
            ▼
    Skynet Socket Thread
（底层异步网络事件处理）
            │
            │ PTYPE_SOCKET
            ▼
       Lua Service
            │
      skynet.socket
            │
    ┌───────┴────────┐
    │                │
coroutine A      coroutine B
socket.read      处理其他业务
```

当前 `socket.lua` 的实现就是在收到 `SKYNET_SOCKET_TYPE_DATA` 后把数据压入缓冲区，并检查等待的读取条件；满足后唤醒等待 coroutine。**只会阻塞 coroutine，不会阻塞线程**。


和 POSIX 的区别是，skynet 可以利用 coroutine 来保持缓冲区等状态：

```text
以前：

EPOLLIN
 ↓
自己维护状态机


Skynet：

PTYPE_SOCKET
 ↓
框架维护 buffer + coroutine
 ↓
恢复原来的执行位置
```

---

## 客户端


```lua
local socket = require "skynet.socket"

local id = socket.open("127.0.0.1", 8888)
```

对应 POSIX 接口中的：

```cpp
socket();
connect();
```

`socket.open(address, port)` 会阻塞**当前 coroutine** 直到连接过程完成。

 `id` 是 **Skynet socket id**，不是当 Linux 原始 fd。它是 Skynet 网络层用来标识 socket 的句柄，而且**这个 id 在整个 Skynet Node 中是可见的**。


---

## 服务器端

```lua
local listen_id = socket.listen("0.0.0.0", 8888)

socket.start(listen_id, function(id, addr)
    -- 新连接
end)
```

accept 回调获得的新 socket **不会马上开始把数据发给当前 Service**。必须再调用 `socket.start(new_id)`。因为 Skynet 的结构是：

```text
Gate Service
      ↓
accept
      ↓
拿到 socket id
      ↓
把 id 发给 Agent Service
      ↓
Agent socket.start(id)
```

也就是转移所有权，谁调用 `socket.start(id)` 就拥有这个 listen_id 的所有权。**哪个 Service 调用了 `socket.start(id)`，Skynet 就把相应的网络消息转发到哪个 Service**。

如果一个 service 需要放弃所有权，清楚这个 service 中关于这个 socket 的状态：

```lua
socket.abandon(id)
```

---

读取消息的 api：

```lua
local data = socket.read(id, sz)
```

表示恰好读取 sz 个字节。如果缓冲区没有 sz 个字节数据会挂起当前 coroutine，直到有足够数据之后再唤醒 coroutine。


如果不传长度参数：返回当前可以获得的尽可能多的数据，如果一个字节也没有，依然会将 coroutine 挂起等待。


Skynet 的 API 本身就是按“一个等待读取者”设计的。所以**不要多个 coroutine 同时读一个 socket**。

---


```lua
local line = socket.readline(id)
```

默认分隔符为 `\n`，也可以指定分隔符，其返回 buffer 中分隔符前面的部分，**不包含分隔符**。


---

```lua
socket.readall(id)
```

**一直读，直到连接断开，然后把所有数据返回**。如果是只一个长连接，可以会长时间阻塞不返回。

---

```lua
socket.block(id)
```

挂起 coroutine，直到 socket 有数据可读，或者连接关闭。功能类似于 POSIX 中的epoll_wait()。

---


```lua
socket.write(id, data)
```

这个 api 不会阻塞协程**：`write` 是把字符串加入正常写队列，由框架在 socket 可写时发送**。

---


```lua
socket.lwrite(id, data)
```

这是 low priority 的写操作，skynet 内部有：

```text
normal write queue
low priority write queue
```

只有在排空 normal write queue 之后才会发送 low priority write queue 的数据。

---

跟 read 操作不同的是，多个 service 可以向同一个 socket 写，skynet 保证**单次 `write` 的字符串不会被拆开后夹入另一个 write 的数据**。



---

```lua
socket.warning(id, function(id, size)
    ...
end)
```

监控积压量，当**写缓冲区**数据积压量超过 size KiB 是会产生 warning。

---

```lua
socket.close(id)
```

推荐的清理方式，可能 yield，因为如果有其他 coroutine 在读，会等操作结束之后再 close。

```text
close
 ↓
优雅地协调当前读取流程
 ↓
清理 socket
```

---

## UDP

skynet udp 使用 callback 来处理数据，收到 udp 数据包时直接调用 callback：

```lua
local id = socket.udp(function(data,from)
	...
end,"0.0.0.0",8000)
```

使用

```lua
socket.udp_address(from)
```

将来源地址转换成 IP+port。

---


## 域名查询

```lua
socket.open("example.com,80)
```

底层直接调用：

```cpp
getaddrinfo()
```

**可能阻塞整个 socket 线程**。所以推荐的 DNS 查询方式：

```lua
local dns = require ("skynet.dns")
dns.server()
local ip = dns.resolve("example.com")
```

---


# [GateServer](https://github.com/cloudwu/skynet/wiki/GateServer)

**Gate 是游戏服务器的接入层，负责管理客户端连接、把 TCP 字节流切成完整数据包，再转发给真正处理业务的 Service。** Skynet 提供通用模板 `lualib/snax/gateserver.lua`，而 `service/gate.lua` 是基于这个模板写好的一个完整 Gate。


## `snax.gateserver`


`snax.gateserver` 对应 `lualib/snax/gateserver.lua`，是一个**框架模板**。

`service/gate.lua` 是使用 `gateserver.lua` 写出来的一个具体 Gate Service。

```text
gateserver.lua
    │
    │ 通用框架
    ▼
 gate.lua
    │
    │ 一个实际 Gate 实现
    ▼
游戏服务器
```

gateserver 的最简单使用：

```lua
local gateserver = require "snax.gateserver"

local handler = {}

-- 定义各种 callback

gateserver.start(handler)
```

`gateserver.start` 默认自己调用 `skynet.start()`，所以不需要写：

```lua
skynet.start(function()
	...
end)
```


GateServer 框架负责发现事件，自己利用框架写的 gateservice 需要定义事件发生后怎么处理。


```text
PTYPE_SOCKET
     ↓
gateserver.lua
     ↓
netpack/filter
     ↓
识别事件
     │
     ├── connect    → handler.connect
     ├── packet     → handler.message
     ├── close      → handler.disconnect
     ├── error      → handler.error
     └── warning    → handler.warning
```

---

## `handler.connect`

客户端建立 TCP 连接后，Gate Server 接受连接，并触发：

```lua
handler.connect(fd, addr)
```

此时，TCP 连接已经建立，但 Gate 尚未启动对该客户端 Socket 的读取。

在默认的连接处理流程中，Gate 会通知 Watchdog 创建 Agent。待 Agent 创建完成后，Watchdog 通知 Gate 建立 `fd` 与 Agent 的对应关系，并调用：

```lua
gateserver.openclient(fd)
```

该函数内部通过 `socketdriver.start(fd)` 启动 Socket 读取。

需要注意，客户端在此期间仍然可以发送数据。这些数据通常会暂存在服务端操作系统的 TCP 接收缓冲区中，不会仅仅因为 Gate 尚未开始读取就立即丢失。待 Gate 启动读取后，暂存的数据便可以继续被处理；但内核缓冲区容量有限，不能无限期地不读取数据。

**这样设计的主要目的是保证消息路由关系已经建立后，再开始正常处理客户端消息。** 在默认流程中，Gate 先获得 Agent 的地址，再启动读取，可以避免客户端消息进入正常转发流程时，Gate 尚未准备好对应的 Agent。

补充：`gateserver.openclient(fd)` 主要负责启动 Socket 读取，并不是建立 TCP 连接。若业务需要在 Agent 创建之前处理登录或认证消息，也可以设计为先启动读取，并将消息转交给 Watchdog 处理。

---

## `handler.message`


```lua
handler.message(...)
```

收到的不是一次 TCP `recv` 的结果，而是 GateServer 使用 netpack 完成粘包、半包处理之后得到的**一条完整的应用层 packet**。

当完整数据包被切出来后才调用 `handler.message(fd, msg, sz)`。`msg` 是 C 内存指针，`sz` 是长度。


GateServer 使用的分包协议：

```text
┌────────────────┬──────────────────────┐
│ 2 bytes length │       payload        │
└────────────────┴──────────────────────┘
```

为了性能，`handler.message` 收到的是 `msg+sz`，而不是 `string`，如果立即转化成 Lua string 会做一次内存复制：

```text
C buffer
   ↓ copy
Lua string
```

如果需要 Lua 自己处理：

```lua
local str = netpack.tostring(msg,sz)
```

```text
C buffer
   ↓
拷贝到 Lua string
   ↓
释放原来的 C buffer
```

但是官方仓库 gate.lua 使用的是另一个函数：

```lua
skynet.send(watchdog, "lua", "socket", "data", fd, skynet.tostring(msg, sz))
skynet.trash(msg,sz)
```



当前 `gate.lua` 注册：

```lua
skynet.register_protocol {
    name = "client",
    id = skynet.PTYPE_CLIENT,
}
```

而 Agent 也注册：

```lua
skynet.register_protocol {
    name = "client",
    id = skynet.PTYPE_CLIENT,

    unpack = ...,
    dispatch = ...
}
````

所以：

```text
Gate
 ↓
redirect(..., "client", ...)
 ↓
Agent
 ↓
PTYPE_CLIENT dispatcher
````

```text
外部客户端 TCP packet
       ↓
Skynet 内部 PTYPE_CLIENT 消息
       ↓
Agent
```

---

## `handler.command`

**Lua 协议消息会进入 `handler.command`**，而它的返回值会经过 `skynet.pack/ret` 返回调用方。


所以 Gate 实际上有两套输入：

```text
                  Gate

客户端世界             Skynet 世界
    │                     │
 TCP packet           Lua message
    │                     │
    ▼                     ▼
message()             command()
```

---

## `nodelay`

```lua
local addr,port = skynet.call(watchdog, "lua", "start", {
		port = 8888,
		maxclient = max_client,
		nodelay = true,
	})
```

配置

```lua
nodelay = true
```

会给客户端 socket 设置：

```text
TCP_NODELAY
```

也就是**关闭 Nagle 算法，因为游戏服务器经常有大量的 `小包+低延迟要求` ** 。


> Nagle 算法是 TCP 中用来**减少大量“小数据包”发送**的一种算法。
> 
> 它解决的问题是：如果应用程序频繁调用 `send()`，每次只发送几个字节，TCP 可能产生很多“小包”。
> 
> Nagle 算法的核心思想可以概括成：
> 
> **如果之前发送的数据还没有收到 ACK，就先把后续的小数据攒起来，不要立即发送。**

---


## `GateServer` 和 `skynet.socket`


`skynet.socket` 是 coroutine 风格的 socket API。而 `GateServer` ：

```text
直接接管 PTYPE_SOCKET
+
socketdriver
+
netpack
```

**`snax.gateserver` 模板不能在同一个 Service 中和 `skynet.socket` 一起使用**，因为它已经接管了 socket 类消息。

因为一个 Server 需要一个明确的 dispatcher，如果混用会导致两套机制同时抢同一个 Service 的 Socket 消息。

---


# [DebugConsole](https://github.com/cloudwu/skynet/wiki/DebugConsole)


`DebugConsole` 可以理解为 skynet 自带的运行时运维/诊断终端。

DebugConsole ≈ ps + top + pstack + netstat + 一部分 gdb + Skynet 自己的 Service/消息诊断工具。


list：列出当前所有 Service。

```text
Service 地址
+
Service 类型
+
启动参数
```


stat：统计所有 service 的运行情况。

```text
消息队列长度
挂起请求数量
已经处理的消息数量
CPU 时间（profile=true 时）
```


task：显示指定 service 所有当前被挂起的请求以及它们的调用栈。


ping：测试 service 能否正常相应。


signal：如果 service 陷入死循环，卡住了整个线程，以至于无法回复task、ping 命令。signal 可以打断正在执行的 Lua 字节码，并抛出错误显示调用栈，专门用于排查 endless loop。

mem：让所有 service 报告自己的 Lua VM 内存占用。

gc：执行一次完整的 GC 之后再报告内存。


netstat：展示 skynet 管理的网络连接。

info：查看某个 service 的信息，适合在stat 全局扫描之后使用 info 查看指定 service 的具体信息。


logon/logoff：需要 config 中配置 logpath，捕获 skynet message 到指定文件，直到 logoff 停止。

---


# MongoDB

这部分主要是关于 skynet 框架封装的 MongoDB 的接口使用。至于 mongosh 的 CRUD ，见《MongoDB 入门》对应章节。

|MongoDB Shell|Skynet Lua 驱动常见接口|
|---|---|
|`insertOne()`|`safe_insert()`|
|`insertMany()`|`safe_batch_insert()`|
|`find()`|`find()`|
|查询单个文档|`findOne()`|
|`updateOne()`|`safe_update(..., false, false)`|
|`updateMany()`|`safe_update(..., false, true)`|
|`replaceOne()`|`safe_update()` 传入替换文档|
|`deleteOne()`|`safe_delete(filter, true)`|
|`deleteMany()`|`safe_delete(filter, false)`|
|`findAndModify()`|`findAndModify()`|

## 建立连接


```lua
local skynet = require "skynet"
local mongo = require "skynet.db.mongo"

skynet.start(function()
    local client = mongo.client({
        host = "127.0.0.1",
        port = 27017,
    })

    local db = client["mini_game"]
    local players = db.players

    -- 后续通过 players 操作玩家数据
end)
```

其实还以指定登录的用户、身份认证机制，以后再补充。

---

## Create

Lua table 是传递数据的载体，驱动会把 table 编码成 BSON，然后发给 MongoDB，所以对于 update operator 这些以 `$` 开头的不合法的 Lua 标识符，需要使用 `["  "]` 这样的写法。


插入单条数据：

```lua
local ok, err, result = players:safe_insert({
    _id = 10001,
    name = "Felix",
    gold = 1000,
})
```

批量插入：

```lua
local ok, err, result = players:safe_batch_insert({
    {
        _id = 10001,
        name = "Felix",
        gold = 1000,
    },
    {
        _id = 10002,
        name = "Alice",
        gold = 2000,
    },
})
```

`safe_batch_insert` 并不是事务。

返回值：
- ok：操作是否成功。
- err：错误信息，成功时为 nil。
- result：MongoDB 返回的原始命令结果。
	- `result.ok` MongoDB 命令是否成功执行，通常 `1` 表示成功，即便写操作出错，但是命令成功执行了，也会返回 1 。
	- `result.n` 此次插入命令插入的文档数量，成功插入一条时通常为 `1`


---


## Read

查询当个 document：

```lua
local player = players:findOne(
    {
        _id = 10001,
    },
    {
        name = 1,
        gold = 1,
        _id = 0,
    }
)

if player == nil then
    return false, "PLAYER_NOT_FOUND"
end
```

返回的是一个 Lua table。如果找不到文档，则返回 nil，如果命令错误，则会抛出 Lua 错误。

查询多个 document：

```lua
local cursor = players:find({
    gold = {
        ["$gte"] = 1000,
    },
})

while cursor:hasNext() do
    local player = cursor:next()

    skynet.error(
        player.name,
        player.gold
    )
end

cursor:close()
```

返回的是游标。与 mongosh 不同，没有提供 toArray 方法。

排序、分页、限制数量操作：

```lua
local cursor = players:find(
    {
        level = 10,
    },
    {
        name = 1,
        level = 1,
    }
):sort({
    level = -1,
}):skip(10):limit(20)
```

使用的游标方法链。更多游标对象的操作：
- `cursor:sort(...)`
- `cursor:skip(...)`
- `cursor:limit(...)`
- `cursor:hint(...)`
- `cursor:maxTimeMS(...)`
- `cursor:batchSize(...)`

---

## Update

```lua
players:safe_update(query, update, upsert, multi)
```
- `query`：Filter，匹配哪些 Document。
- `update`：更新操作符或替换文档。
- `upsert` 没匹配到时是否插入。
- `multi`：是否更新所有匹配文档。

由于驱动没有直接名为 `replaceOne()` 的方法，所以替换操作：

```lua
local ok, err, result = players:safe_update(
    {
        _id = 10001,
    },
    {
        _id = 10001,
        name = "Felix",
        gold = 2000,
    },
    false,
    false
)
```

以上几个函数的返回值中的 result：
- `ok`MongoDB 命令是否成功执行，通常 `1` 表示成功
- `n`匹配的文档数量；在 upsert 等场景下还需要结合其他字段判断
- `nModified`实际发生修改的文档数量


另外，还有将读取和修改操作合并为原子操作的 ** `findAndModify`，该函数参数接口风格和其他的 CRUD 接口的都不一样**  。返回值也不一样：

```lua
local result = players:findAndModify({
    query = {
        _id = 10001,
        gold = {
            ["$gte"] = 100,
        },
    },
    update = {
        ["$inc"] = {
            gold = -100,
        },
    },
    new = true,
})

if result.ok ~= 1 then
    -- 命令执行异常
end

if result.value == mongo.null then
    -- 没有匹配文档
end

```

由于接口时传递一个 table，所以使用更灵活：
- `query`：匹配目标文档。
- `update`：修改文档。
- `remove`：是否删除匹配的文档。
- `sort`：多个文档匹配时，选择顺序。
- `new`：是否返回更新后的文档。
- `fields`：控制返回的字段。
- `upsert`：没有匹配文档时是否插入。


此外，还有直接构造原始更新命令参数的方法：

```lua
local ok, err, result = players:raw_safe_update({
    q = {
        _id = 10001,
    },
    u = {
        ["$inc"] = {
            ["inventory.$[item].count"] = 5,
        },
    },
    upsert = false,
    multi = false,
    arrayFilters = {
        {
            ["item.item_id"] = 200,
        },
    },
})
```


---

## Detele

```lua
players:safe_delete(query, single)
```
---

## 批量操作


该接口的设计和 CRUD 有点不一样：是由 client 来调用的：

```lua
local ok, err, result, failed_ops = client:bulkWrite({
    -- 1. 插入玩家
    {
        op = "insert",
        collection = "mini_game.players",
        insert = {
            _id = 10003,
            name = "Bob",
            gold = 500,
            level = 1,
        },
    },

    -- 2. 给一个玩家增加金币
    {
        op = "update",
        collection = "mini_game.players",
        filter = {
            _id = 10001,
        },
        update = {
            ["$inc"] = {
                gold = 100,
            },
        },
        upsert = false,
        multi = false,
    },

    -- 3. 给所有 10 级玩家增加金币
    {
        op = "update",
        collection = "mini_game.players",
        filter = {
            level = 10,
        },
        update = {
            ["$inc"] = {
                gold = 50,
            },
        },
        upsert = false,
        multi = true,
    },

    -- 4. 删除一个玩家
    {
        op = "delete",
        collection = "mini_game.players",
        filter = {
            _id = 10002,
        },
        multi = false,
    },
}, nil, true)

if not ok then
    skynet.error("bulkWrite failed: ", err)
    -- 注意：部分操作可能已经成功，不能直接假定全部失败。
    return false, "DB_ERROR"
end

return true, result
```


---


## CRUD 使用


再实际使用时，通常不会直接调用接口，而是套在 `pcall` 里面，这样有利于区分两类问题：
- 数据库返回写操作错误：例如唯一索引冲突。`safe_` 方法通常会返回 `false, err, result`。
- Lua 调用过程中抛出异常：例如网络请求、驱动内部错误。这类情况需要通过 `pcall()` 捕获。

---