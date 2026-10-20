# Overview

MongoDB 可以粗略理解为一个以 **Document（文档）** 为核心的数据库系统。文档采用类似 JSON 的结构表示，底层实际使用 **BSON（Binary JSON）** 存储。

## MongoDB 与 MySQL 的概念对应

作为一种非关系型数据库（NoSQL），MongoDB 与 MySQL 有一些可以帮助理解的概念对应关系：

|MySQL|MongoDB|
|---|---|
|Database|Database|
|Table|Collection|
|Row|Document|
|Column|Field|
|SQL|MongoDB Query API|

需要注意，这些只是帮助理解的类比，并不是完全一一对应。

MongoDB 的分布式能力还包括：

- **Replica Set（副本集）**：主要用于数据复制、高可用和自动故障转移。
- **Sharding（分片）**：将数据分布到多个节点，实现水平扩展。

不能简单地认为“集群 = Replica Set”。

---

## 数据建模思想

MySQL 通常更强调数据之间的关系、规范化、减少冗余以及维护数据一致性。

MongoDB 通常更强调应用程序的**访问模式**，即考虑应用程序一次需要哪些数据，以及如何把经常一起访问、生命周期相关的数据组织在一起。

因此，MongoDB 中经常会根据实际业务需求，将相关数据**嵌入（Embed）**到同一个 Document 中。

---

## Document

MongoDB 的文档与 JSON 非常相似，例如：

```JavaScript
{
    _id: ObjectId("507f1f77bcf86cd799439011"),
    name: "Alice",
    birthdate: ISODate("1990-01-01T00:00:00Z"),

    address: {
        street: "123 Main St",
        city: "Springfield",
        state: "IL"
    },

    hobbies: [
        "reading",
        "hiking",
        "coding"
    ]
}
```

可以看到，一个 Document 可以包含：

- 字符串
- 数字（如 Int 32、Int 64、Double、Decimal 128）
- 日期（BSON Date，Shell 中常显示为 `ISODate(...)`）
- ObjectId
- 数组
- 嵌套 Document
- Document 数组

因此 MongoDB 的 Document 比标准 JSON 支持的数据类型更加丰富。

MongoDB 可以把相关数据嵌入同一个 Document 中，例如：

```JavaScript
{
    _id: 10001,
    name: "Felix",
    gold: 1000,

    equipment: {
        weapon: {
            id: 10001,
            level: 10
        }
    },

    inventory: [
        {
            item_id: 20001,
            count: 10
        }
    ]
}
```

将经常一起访问的数据组织在同一个 Document 中，可以减少跨集合的数据关联操作；同时，由于 MongoDB 对单个 Document 的操作具有原子性，也方便对相关数据进行原子更新。

但并不是所有相关数据都应该嵌入。数据建模需要结合实际访问模式、数据规模、生命周期以及是否会无限增长等因素，在**嵌入（Embed）和引用（Reference）**之间进行选择。

---

## 原子性与事务

MongoDB 对**单个 Document 的操作具有原子性**。

例如：

```JavaScript
{
    $inc: {
        gold: -100
    }
}
```

可以原子地完成金币扣减。

对于需要同时修改多个 Document 的业务，MongoDB 也支持**多文档事务**，可以将多个读写操作组织成一个具有 ACID 特性的整体，使这些操作要么全部成功，要么全部失败。

---

## Replica Set

MongoDB 支持 **Replica Set（副本集）**，用于数据复制、高可用和自动故障转移。

一个简单的 Replica Set 可以理解为：

```
Primary
   │
   ├── 数据复制 ──→ Secondary
   │
   └── 数据复制 ──→ Secondary
```

Primary 负责主要的写操作，Secondary 保存数据副本。

当 Primary 发生故障时，副本集可以进行选举，让其他成员中的合适节点成为新的 Primary，从而实现自动故障转移。Replica Set 是 MongoDB 分布式系统能力的一部分。

---

## Sharding

MongoDB 支持 **Sharding（分片）**，即将数据分布到多个节点，以支持更大的数据规模和更高的吞吐量，实现水平扩展。

MongoDB 使用 **Shard Key（分片键）** 来决定数据如何分布到不同的 Shard。

可以简单理解为：

```
              MongoDB
                 │
             Sharding
                 │
       ┌─────────┼─────────┐
       ↓         ↓         ↓
    Shard A   Shard B   Shard C
     一部分     一部分     一部分
      数据       数据       数据
```

与 Replica Set 的区别：

```
Replica Set
    ↓
多个节点保存数据副本
    ↓
高可用、故障转移

Sharding
    ↓
多个节点保存不同的数据
    ↓
水平扩展
```

---

# Document

Document 是 MongoDB 中数据的基本单位，由一组 `field -> value` 的键值对组成。

## 点表示法

MongoDB 使用**点表示法（Dot Notation）**访问嵌套 Document 中的字段和数组中的元素。
- 访问嵌套 Document 中的字段，使用 `"a.b.c"` 这样的字段路径。
- 访问数组中的元素，使用 `"array.index"`，索引从 `0` 开始。

例如：

```JavaScript
{
    name: {
        first: "Felix",
        last: "Qiu"
    },

    hobbies: [
        "coding",
        "gaming"
    ]
}
```

可以使用：

```JavaScript
"name.first"    → "Felix"
"hobbies.0"     → "coding"
```

---

## 字段名称

- 字段名称必须是字符串。
- 同一个 Document 中的字段名称必须唯一。
- 嵌套 Document 可以使用与上一级 Document 相同的字段名称，例如：
```JavaScript
{
    name: "Felix",
    player: {
        name: "Qiu"
    }
}
```

这里两个 `name` 属于不同层级，因此没有冲突。

- 字段名称不能包含 `null` 字符。
- MongoDB 5.0+ 对字段名中的 `$` 和 `.` 提供了更好的支持，但仍存在一些限制，因此在实际业务中**尽量不要使用** `**$**` **和** `**.**` **作为字段名**。
    
---

## Document 大小限制

单个BSON Document 的最大大小为 **16 MiB**。

因此，对于可能无限增长的数据，不应该一直嵌入同一个 Document 中。

例如玩家的历史战绩可能不断增加：

```
players
    └── match_results
            └── 不断增长
```

这种情况下应该考虑将历史数据拆分到独立的 Collection 中。

---

## GridFS

由于单个 BSON Document 的最大大小为 16 MiB，MongoDB 提供了 GridFS 这一文件存储规范，用于存储和检索大型文件。

GridFS 将一个文件拆分为多个数据块（Chunk），每个数据块保存为一个 Document，并将文件的元数据单独保存到一个 Document 中。

### 存储结构

默认情况下，GridFS 使用两个 Collection：

- `fs.files`：每个文件对应一条元数据 Document，记录文件 ID、文件名、文件大小、数据块大小、上传时间及自定义元数据等。
- `fs.chunks`：每个数据块对应一条 Document，包含以下字段：
    - `_id`：数据块 Document 的唯一标识。
    - `files_id`：引用 `fs.files` 中对应文件的 `_id`。
    - `n`：数据块的序号，从 0 开始。
    - `data`：保存当前数据块的二进制数据。

默认情况下，每个数据块的大小为 255 KiB，最后一个数据块可以更小。

### 核心能力

- **分块读写**：可以逐块上传和下载文件，无需将整个文件一次性加载到内存中。
- **范围读取**：可以读取文件的特定部分，例如跳转到视频或音频文件的中间位置。
- **文件关联**：通过 `files_id` 将文件元数据与多个数据块关联，通过 `n` 确定数据块的顺序。

### 限制

- 不支持多 Document 事务。
- 不支持对整个文件内容进行原子更新。如果需要替换文件，可以采用多版本方案：先上传新版本，再原子更新指向当前版本的元数据，最后根据需要删除旧版本。
    

### 适用场景

- 存储超过 16 MiB 的文件。
- 需要分块读取或范围访问的文件。
- 需要将文件及其元数据统一纳入 MongoDB 管理的场景。

对于不超过 16 MiB、且不需要分块读取的文件，可以考虑直接使用 BSON Binary 字段存储。

---

## 字段顺序

BSON Document 中的字段是有序的。

在比较两个 Document 时，**字段顺序可能影响是否相等**：

```
{a: 1, b: 1}
```

与：

```
{b: 1, a: 1}
```

在 Document 比较中并不等价。

但是，查询处理过程中 MongoDB 可能重新排列字段，因此业务代码**不应该依赖查询结果中的字段顺序**。

---

## `_id`

标准 Collection 中，每个 Document 都需要一个唯一的 `_id` 字段作为主键。

- `_id` 必须唯一。
- `_id` 不可变。
- MongoDB 默认会在 `_id` 上建立唯一索引。
- 如果插入 Document 时没有提供 `_id`，MongoDB Driver 通常会自动生成一个 `ObjectId`；如果客户端没有生成，服务器也可以补充生成。
- `_id` 不一定必须是 `ObjectId`，也可以使用业务上已有的唯一标识，例如游戏中的 `uid`。


---

# Collection

Collection 是 MongoDB 中存储 Document 的集合，可以类比 MySQL 中的 Table。

## 懒创建

MongoDB 支持 Database 和 Collection 的隐式创建。

切换到一个不存在的 Database：

```JavaScript
use mini_game
```

不会立即创建 Database。只有在该 Database 中**首次写入数据**时，Database 和对应的 Collection 才会被实际创建。

例如：

```JavaScript
use mini_game

db.players.insertOne({
    _id: 10001,
    gold: 1000
})
```

执行后，`mini_game` Database 和 `players` Collection 都会被创建。

---

## Schema

Collection 默认不会强制其中所有 Document 使用完全相同的 Schema，即不同 Document 可以具有不同的字段和数据类型。

如果业务需要对 Document 的结构和字段类型施加约束，可以使用 **Schema Validation**，在插入和更新数据时进行校验。

MongoDB 的 Schema 灵活并不意味着不需要设计 Schema，实际开发中仍然需要根据业务需求设计合理的数据结构。

---

## 显式创建 Collection

虽然 MongoDB 支持 Collection 的隐式创建，但如果需要提前指定一些 **Collection 级别的选项**，可以使用：

```JavaScript
db.createCollection()
```

显式创建 Collection。

---

## View

MongoDB View 是一种**只读的可查询对象**，其内容由底层 Collection 或其他 View 上的 Aggregation Pipeline（聚合管道）定义。

标准 View 不独立存储查询结果，每次查询时都会根据底层数据按需计算。

### View 与 Projection 的区别

|对比|Projection|View|
|---|---|---|
|本质|一次查询的字段选择规则|数据库中保存的查询定义|
|作用范围|当前查询|可以被多次查询、多个客户端复用|
|功能|主要控制查询结果返回哪些字段|可通过聚合管道进行筛选、转换、分组、关联等|
|是否独立存储结果|否|否，标准 View 不存储查询结果|
|是否可以单独查询|不能独立查询|可以通过 View 名称查询|

### 使用示例

创建一个 View，筛选等级不低于 20 的玩家，并仅返回指定字段：

```JavaScript
db.createView(
    "high_level_players",
    "players",
    [
        {
            $match: {
                level: { $gte: 20 }
            }
        },
        {
            $project: {
                name: 1,
                gold: 1,
                level: 1
            }
        }
    ]
)
```

查询 View：

```JavaScript
db.high_level_players.find()

db.high_level_players.find({
    gold: { $gte: 1000 }
})
```

View 的核心不是一种新的数据存储结构，而是将 Aggregation Pipeline 封装为**可重复使用的查询对象**。查询 View 时，MongoDB 会将外部查询与 View 的管道组合并进行查询优化。

### 物化视图

MongoDB 还支持按需物化视图（On-Demand Materialized View）。它**将聚合结果保存到磁盘中的 Collection**，适合重复执行的复杂统计查询。

|对比|标准 View|按需物化视图|
|---|---|---|
|是否保存计算结果|否|是|
|查询时|根据底层数据按需计算|读取已保存的结果|
|数据更新|查询时根据当前底层数据计算|需要重新执行聚合管道刷新结果|
|是否能直接创建普通索引|否，使用底层 Collection 的索引|可以，在保存结果的 Collection 上创建索引|
|主要适用场景|复用查询逻辑|重复进行的复杂统计查询|

物化视图通常通过 `$merge` 或 `$out` 将聚合结果写入 Collection。刷新时机由应用或相应任务安排，不会因为底层数据变化而自动刷新。

### 在 Skynet 中使用 View

如果当前使用的 Skynet MongoDB 驱动没有专门的 `createView()` 方法，可以先通过 `mongosh` 或数据库初始化脚本创建 View。

MongoDB 也支持使用 `create` 数据库命令创建 View；如果驱动支持相应的 `runCommand` 调用，也可以考虑通过该方式创建。

创建之后，可以像访问 Collection 一样获取 View，并使用常见的查询接口：

```lua
local skynet = require "skynet"
local mongo = require "skynet.db.mongo"

skynet.start(function()
    local client = mongo.client({
        host = "127.0.0.1",
        port = 27017,
    })

    local db = client["mini_game"]
    local high_level_players = db.high_level_players

    local call_ok, player = pcall(function()
        return high_level_players:findOne({
            _id = 10001,
        })
    end)

    if not call_ok then
        skynet.error("Query view failed: ", player)
        return
    end

    if player == nil then
        skynet.error("Player not found")
        return
    end

    skynet.error("Player name: ", player.name)
end)
```


View 只支持读取，不支持直接插入、更新和删除数据；如果需要修改玩家信息，仍然应操作底层 Collection。

### View 与权限控制

在启用 MongoDB 身份验证的情况下，可以授予用户对某个 View 的 `find` 权限，而不授予其对底层 Collection 的 `find` 权限。

例如，可以为后台只读账号授权访问只包含玩家公开信息的 View，而不允许其直接读取完整的玩家数据。

但需要注意：

- 创建 View 时，创建者需要具备相应的数据库权限，以及访问源 Collection 和管道中引用对象所需的权限。
    
- View 本身不会自动撤销其他已有权限。如果某个账号已经拥有底层 Collection 的读取权限，它仍然可以直接读取底层数据。
    
- View 的定义可能被 `getCollectionInfos()` 或 `explain` 等操作查看，因此不要在定义中直接写入敏感字段值或秘密信息。
    

View 可以辅助实现数据访问隔离，但需要与 MongoDB 的身份验证和权限管理配合使用。



---


## Capped Collection

Capped Collection（固定大小集合）是 MongoDB 中一种预先指定存储空间上限的特殊集合，类似于循环缓冲区（Circular Buffer）。

创建时必须指定存储空间上限 `size`，还可以通过 `max` 指定最大 Document 数量。当达到空间上限或 Document 数量上限时，MongoDB 会自动淘汰旧 Document，为新数据腾出空间。

### 创建方式

```JavaScript
use mini_game

db.createCollection("server_logs", {
    capped: true,
    size: 10 * 1024 * 1024,
    max: 1000
})
```

- `capped: true`：创建固定大小集合。
- `size`：最大存储空间，单位为字节，必填。
- `max`：最大 Document 数量，可选。
- `size` 限制优先；如果空间先达到上限，即使 Document 数量未达到 `max`，也会淘汰旧数据。
    

### 查询数据

基本查询与普通集合类似。可以使用 `$natural` 排序，高效获取最近插入的 Document：

```JavaScript
db.server_logs.find()
    .sort({ $natural: -1 })
    .limit(10)
```

|排序方式|行为|
|---|---|
| `{ $natural: 1 }` |按自然顺序读取|
| `{ $natural: -1 }` |按相反的自然顺序读取|

注意：存在多个并发写入者时，MongoDB 不保证查询结果严格按照插入顺序返回。如果需要按照业务事件发生时间排序，应使用时间字段。

### Tailable Cursor（可追加游标）

Capped Collection 支持 Tailable Cursor。

普通游标耗尽当前结果后会结束；可追加游标在耗尽当前结果后仍然保持打开状态，可以继续读取之后插入的新 Document，类似 Linux 的 `tail -f`。

需要显式启用 Tailable Cursor，普通 `find()` 不会自动启用该行为。

### 限制与注意事项

- 不支持分片。
    
- 不能在事务中写入。
    
- 不建议更新已有 Document，因为更新可能扩大其大小，导致意外行为。
    
- 聚合管道的 `$out` 不能写入 Capped Collection。
    
- 并发写入时不能依赖自然顺序保证严格的业务事件顺序。
    

### 适用场景

适合持续追加、可以淘汰旧数据、只需要保留近期记录的场景，例如近期诊断日志。

如果需求是按照时间自动过期，例如只保留最近 7 天的日志，通常应该优先考虑普通 Collection + TTL 索引。MongoDB 官方指出，TTL 索引通常具有更好的性能和灵活性。


---

## Time Series Collection

**时间序列数据（Time Series Data）**：随着时间不断产生的一系列测量数据，可以通过分析数据随时间的变化获取信息。例如：
- 每秒的在线人数。
- 每隔 10 秒记录一次 CPU 使用率。
- 每次请求的响应延迟。
    

时间序列数据通常具有三个组成部分：
- **Time**：数据采集或事件发生的时间。
- **Metadata**：标识数据属于哪个序列的相对稳定的信息，例如服务器 ID。
- **Metrics**：随时间变化的测量指标，例如 CPU 使用率、在线人数。
    

时间序列数据通常具有以下特点：
- **数据持续追加**：长期积累可能产生大量记录，采集间隔不一定固定。
- **相邻数据具有相似性**：例如同一台服务器的 Metadata 相同，相邻时刻的指标值可能接近。
- **查询围绕时间范围展开**：通常需要查询某个数据来源在特定时间范围内的记录。

### 存储原理：Bucket

Time Series Collection（时间序列集合）是一种针对时间序列数据优化的特殊 Collection，其底层采用分桶存储和优化的压缩格式。

MongoDB 会将具有**相同 Metadata、时间接近**的多条 Measurement（测量文档）组织到一个 Bucket 中。

例如，`server_metrics` 时间序列集合对应的底层系统集合通常是：

`system.buckets.server_metrics`

该系统集合由 MongoDB 自动创建和管理，用于保存实际的 Bucket 数据。开发者仍然通过逻辑集合 `server_metrics` 进行查询与插入，无需直接访问底层 Bucket。

通过分桶与压缩，MongoDB 可以减少冗余信息、降低存储开销，并优化适合时间序列数据的查询。

### 创建方式

```JavaScript
use mini_game

db.createCollection("server_metrics", {
    timeseries: {
        timeField: "timestamp",
        metaField: "metadata",
        granularity: "seconds"
    },
    expireAfterSeconds: 7 * 24 * 60 * 60
})
```

主要参数：

- `timeseries.timeField`：必填，指定测量时间所在的字段，字段值必须为有效的 BSON Date。
    
- `timeseries.metaField`：可选，指定标识数据序列的元数据字段。其值应尽量稳定，通常具有以下特点：
    
    - 很少发生变化。
        
    - 经常用于查询过滤。
        
    - 能够标识数据的来源或所属序列。
        
- `timeseries.granularity`：可选，帮助 MongoDB 根据测量数据的时间间隔优化分桶；默认是 `"seconds"`。
    

| `granularity` |单个 Bucket 的最大时间跨度|
|---|---|
| `"seconds"` |1 小时|
| `"minutes"` |24 小时|
| `"hours"` |30 天|

注意：`granularity` 不代表采集频率，也不意味着每个 Bucket 一定覆盖完整的最大时间跨度。

- `expireAfterSeconds`：可选，指定测量数据的保留时间，单位为秒。
    
    - 过期时间根据 `timeField` 的值计算。
        
    - 通常需要等待一个 Bucket 中的全部记录过期后，后台任务才会移除这个 Bucket。
        
    - 数据过期后并不保证立即删除。
        

### CRUD 操作与限制

时间序列集合支持常见的插入、查询和聚合操作，语法与普通 Collection 基本相同。

但 MongoDB 8.0 对更新操作有严格限制：

- 查询条件只能匹配 `metaField`。
    
- 只能修改 `metaField` 的值。
    
- 更新内容必须使用更新操作符表达式。
    
- 必须使用多文档更新，例如 `updateMany()` 或 `multi: true`。
    
- 不支持 `upsert: true`。
    

此外：

- 不支持在多文档事务中写入。
    
- 不支持唯一索引。
    

这些限制体现了时间序列集合的设计目标：**适合持续追加、较少修改的历史测量数据，而不是频繁更新的业务状态。**

### 适用场景

适合保存：

- 游戏服务器的 CPU、内存使用率。
    
- 历史在线人数。
    
- 网络请求延迟、吞吐量等监控指标。
    

不适合直接保存需要频繁修改、唯一性约束或事务写入的核心业务数据，例如玩家当前金币、等级和需要唯一比赛 ID 约束的战绩。



---

# BSON

BSON（Binary JSON）是一种**二进制序列化格式**，用于 MongoDB 中存储 Document，以及进行RPC。

与 JSON 相比，BSON 提供了更加丰富、明确的数据类型。

BSON 的常用数据类型：

```
BSON
│
├── 基本数据
│   ├── String
│   ├── Boolean
│   ├── Int32
│   ├── Int64
│   ├── Double
│   ├── Decimal128
│   └── Null
│
├── 结构数据
│   ├── Document（Object）
│   └── Array
│
├── 特殊类型
│   ├── ObjectId
│   ├── Date
│   ├── Binary
│   └── Regex
│
└── MongoDB 内部/特殊用途
    ├── Timestamp
    ├── MinKey
    ├── MaxKey
    └── 一些已弃用类型
```

其中：

- **Decimal 128** 是一种 128 位**十进制浮点数**类型，用于需要较高十进制精度的场景，例如金融数据。
    
- **String** 使用 UTF-8 编码。
    
- **Null** 表示字段存在，但其值明确为 `null`；**字段值为** `**null**` **与字段不存在是两个不同的概念**。
    
- **ObjectId** 长度为 12 字节，由以下部分组成：
    - 4 字节：Unix 时间戳，单位为秒。
    - 5 字节：随机值。
    - 3 字节：递增计数器。
    - ObjectId 中包含创建时的时间戳，因此可以通过 `ObjectId.getTimestamp()` 获取其对应的时间。ObjectId **只是大致按照创建时间排序，并不是严格递增的全局 ID**。
    
- **Date** 是一个 64 位有符号整数，表示自 Unix Epoch（1970-01-01 00:00:00 UTC）以来的毫秒数。
    
    - `ISODate()` 是 `mongosh` 提供的日期辅助函数，用于创建/表示 ISO 8601 格式的 Date。
        
- **Binary** 可以理解为一段原始二进制数据，即字节序列。
    
- **Timestamp** 主要用于 MongoDB 内部机制；对于应用程序中的业务时间，通常应该使用 **Date**。

---

# CRUD

主要使用 `mongod` 和 `mongosh` 实践 MongoDB 的 CRUD 操作。

首先运行 MongoDB 服务端：

```bash
mongod
```

然后使用 MongoDB Shell 客户端连接：

```bash
mongosh
```

MongoDB 的 CRUD 主要围绕 **Document 和 Query Filter** 展开。

对于查询、更新和删除操作，可以将其理解为：

```
Filter + Operation
```

例如：

```
find(filter, projection)

updateOne(filter, update, options)

updateMany(filter, update, options)

replaceOne(filter, replacement, options)

deleteOne(filter, options)

deleteMany(filter, options)
```

其中：

- `filter`：决定操作哪些 Document，可以类比 SQL 的 `WHERE`。
- `update`：决定如何修改 Document，可以类比 SQL 的 `SET`。
- `projection`：决定查询结果返回哪些字段，可以类比 SQL 中 `SELECT` 后指定的字段。
- 查询、更新、删除使用相同的 Query Filter 语法。

---

## Create

主要使用：

```
insertOne()
insertMany()
```

插入一个 Document：

```JavaScript
db.players.insertOne({
    _id: 10001,
    name: "Felix",
    gold: 1000
})
```

插入多个 Document：

```JavaScript
db.players.insertMany([
    {
        _id: 10001,
        name: "Felix",
        gold: 1000
    },
    {
        _id: 10002,
        name: "Alice",
        gold: 2000
    }
])
```

---

## Read

主要使用：

```
find()
```

查询所有 Document：

```JavaScript
db.players.find()
```

不指定 Filter 时，会匹配 Collection 中的所有 Document。

`find()` 实际返回的是一个 **Cursor（游标）**，而不是一次性返回所有 Document。`mongosh` 会自动迭代 Cursor 并显示结果。

> 游标记录的是**遍历查询结果的所需要的状态**。可能是磁盘位置、索引下标。默认返回索引，分批次返回结果。如果需要一次返回返回所有结果（**`toArray()` 会遍历游标，将其中剩余的查询结果全部收集到客户端内存中的数组里，然后返回这个数组**。）：
> ```JavaScript
> const docs = db.players.find({
>     gold: { $gte: 100 }
> }).toArray()
> ```


条件查询：

```JavaScript
db.players.find({
    gold: { $gte: 1000 }
})
```

Query Filter 可以进一步使用：

```
Query Filter
    │
    ├── 字段匹配
    ├── 比较操作符
    ├── 逻辑操作符
    ├── 数组条件
    ├── 嵌套 Document
    └── ...
```

Projection 投影：

`find()` 还可以指定 `projection`，控制查询结果中返回哪些字段：

```JavaScript
db.players.find(
    { gold: { $gte: 1000 } },
    { name: 1, gold: 1 }
)
```

这里：

```
Filter
    ↓
决定返回哪些 Document

Projection
    ↓
决定这些 Document 返回哪些字段
```

---

## Update

主要有 3 个更新 API：

```
updateOne()
updateMany()
replaceOne()
```

updateOne()：

```JavaScript
db.players.updateOne(
    { _id: 10001 },
    { $inc: { gold: 100 } }
)
```

`filter` 决定目标 Document，`update` 决定如何修改。

`updateOne()` **最多更新一个**符合条件的 Document；**即使有多个 Document 满足 Filter，也只更新其中一个。**

常见的 `update` 形式是使用 Update Operators，例如：

```JavaScript
{
    $set: { gold: 2000 },
    $inc: { level: 1 }
}
```

updateMany()：

```JavaScript
db.players.updateMany(
    { level: 10 },
    { $inc: { gold: 100 } }
)
```

需要注意，`updateMany()` 虽然可以修改多个 Document，但**整个操作并不是跨多个 Document 的原子操作**；MongoDB 只保证每个单独 Document 的写操作具有原子性。

replaceOne()：

```JavaScript
db.players.replaceOne(
    { _id: 10001 },
    {
        _id: 10001,
        name: "Felix",
        gold: 2000
    }
)
```

`updateOne()` / `updateMany()` 通常使用更新操作符修改 Document 中的部分字段，而 `replaceOne()` 则使用新的 Document **整体替换**原来的 Document。

`replaceOne()` 可以使替换后的 Document 具有不同的字段结构，但不应该简单理解为“修改了 Collection 的 Schema”。

---

## Delete

主要使用：

```
deleteOne()
deleteMany()
```

删除一个 Document：

```
db.players.deleteOne({
    _id: 10001
})
```

`deleteOne()` **最多删除一个符合条件**的 Document。

删除多个 Document：

```JavaScript
db.players.deleteMany({
    level: { $lt: 10 }
})
```

如果：

```JavaScript
db.players.deleteMany({})
```

则会匹配并删除 Collection 中的所有 Document，因此生产环境中需要特别谨慎。

---

## 批量写入

**Bulk Write（批量写入）**允许在一次批量操作中执行多个写操作，并且可以混合不同类型的操作：

```
一次 Bulk Write
    │
    ├── insert
    ├── update
    ├── update
    ├── delete
    └── insert
```

主要使用：

```JavaScript
db.players.bulkWrite()
```

支持：

```
insertOne
updateOne
updateMany
replaceOne
deleteOne
deleteMany
```

适合需要批量执行多个插入、更新、删除操作的场景。

---

## 操作结果

MongoDB 的更新操作会返回操作结果，其中几个重要字段是：

```
matchedCount
modifiedCount
upsertedId
```

例如：

```JavaScript
db.players.updateOne(
    { _id: 10001 },
    { $inc: { gold: 100 } }
)
```

可能返回：

```JavaScript
{
    acknowledged: true,
    matchedCount: 1,
    modifiedCount: 1,
    upsertedCount: 0,
    upsertedId: null
}
```

其中：

- `acknowledged`：MongoDB 是否确认了这次写操作。
- `matchedCount`：Filter 匹配到多少个 Document。
- `modifiedCount`：实际发生修改的 Document 数量。
- `upsertedId`：如果使用 `upsert: true` 并且最终插入了新 Document，则记录新 Document 的 `_id`。
- `upsertedCount`：通过 upset 新插入了多少个文档。

同理，delete 操作也会返回结果：

|返回字段|`deleteOne()`|`deleteMany()`|
|---|---|---|
|`acknowledged`|是否确认写操作|是否确认写操作|
|`deletedCount`|`0` 或 `1`|`0` 或更多|


---

## Query Filter

常用 Query Filter：

|类型|内容|
|---|---|
|等值| `{ level: 10 }` |
|比较| `$gt`、`$gte`、`$lt`、`$lte`、`$ne` |
|集合| `$in`、`$nin` |
|逻辑|多字段隐式 AND、`$and`、`$or`、`$not`、`$nor` |
|嵌套字段| `"profile.region"` 点号写法|
|字段判断| `$exists`、`$type` |

其中多个字段直接写在同一个 Filter 中时，默认是逻辑 AND。

---

## Update Operators

常用 Update Operators：

|操作符|用途|
|---|---|
| `$set` |设置或新增字段|
| `$unset` |删除字段|
| `$inc` |将数值字段加上指定值；正数增加，负数减少|
| `$push` |向数组追加元素，允许重复|
| `$addToSet` |仅当元素不存在时加入数组|
| `$pull` |删除数组中所有匹配条件的元素|


---

## Options

|API|常见 Options|作用|
|---|---|---|
|`find()`|`sort`、`skip`、`limit`|排序、跳过结果、限制返回数量|
||`hint`、`maxTimeMS`|指定索引、限制执行时间|
||`collation`、`comment`|指定字符串比较规则、添加操作备注|
|`insertOne()`|`writeConcern`|指定写入确认要求|
|`insertMany()`|`ordered`、`writeConcern`|控制批量插入遇到错误时的行为、写入确认要求|
|`updateOne()`|`upsert`、`arrayFilters`|是否更新插入、指定要更新的数组元素|
||`sort`（MongoDB 8.0+）|多个文档匹配时，决定更新哪一个|
||`hint`、`collation`、`maxTimeMS`|指定索引、字符串比较规则、执行时间限制|
|`updateMany()`|`upsert`、`arrayFilters`|是否更新插入、指定要更新的数组元素|
||`hint`、`collation`、`maxTimeMS`|指定索引、字符串比较规则、执行时间限制|
|`replaceOne()`|`upsert`、`sort`（8.0+）|是否更新插入、多匹配时决定替换哪个文档|
||`hint`、`collation`|指定索引、字符串比较规则|
|`deleteOne()` / `deleteMany()`|`hint`、`collation`、`maxTimeMS`|指定索引、字符串比较规则、执行时间限制|
|`bulkWrite()`|`ordered`、`writeConcern`|控制批量操作的执行顺序与写入确认|


`sort`：排序

`1` 表示升序，`-1` 表示降序。

```JavaScript
{ sort: { level: -1 } }
```

例如按玩家等级从高到低排序。

`limit`：限制结果数量

```JavaScript
{ limit: 20 }
```

最多返回 20 个文档。

`skip`：跳过结果

```JavaScript
{ skip: 100 }
```

跳过前 100 个结果，常用于分页。

`upsert`：更新插入

```JavaScript
{ upsert: true }
```

有匹配文档就更新；没有匹配文档就尝试插入新文档。默认是 `false`。

对游戏服务端尤其要注意：修改玩家金币时，通常不应随意开启 `upsert`，否则可能意外创建玩家文档。

`ordered`：控制批量操作的执行方式

```JavaScript
{ ordered: false }
```

主要用于 `insertMany()` 和 `bulkWrite()`。设为 `false` 后，某个操作失败时，MongoDB 通常会继续处理其他操作；但不能依赖它们按输入顺序执行。

`arrayFilters`：指定要更新的数组元素

配合 `$[item]` 等数组更新语法使用，适用于 `updateOne()`、`updateMany()` 等支持该选项的更新 API。

> 假设玩家的 Document 是：
> 
> ```JavaScript
> {
>     _id: 10001,
>     name: "Felix",
>     inventory: [
>         { item_id: 100, count: 5 },
>         { item_id: 200, count: 10 },
>         { item_id: 300, count: 3 }
>     ]
> }
> ```
> 
> 现在需要把 `item_id = 200` 的物品数量增加 5。MongoDB 的更新语句是：
> 
> ```JavaScript
> db.players.updateOne(
>     { _id: 10001 },
>     {
>         $inc: {
>             "inventory.$[item].count": 5
>         }
>     },
>     {
>         arrayFilters: [
>             { "item.item_id": 200 }
>         ]
>     }
> )
> ```
> 
> ① Update 中的数组占位符： `inventory.$[item].count`
> 
> 意思是：修改 `inventory` 数组中符合条件的元素的 `count` 字段。`item` 是自定义的标识符，不是数组下标，也不是实际字段名。
> 
> ② Options 中的 arrayFilters：`{ "item.item_id": 200 }`
> 
> 意思是：只有 `item_id` 等于 200 的数组元素，才参与这次更新。



`hint`：指定使用哪个索引

```JavaScript
{ hint: { level: 1 } }
```

主要用于查询或写操作的索引选择与性能调优。指定的索引必须存在，不建议在不了解查询计划时随意使用。

`writeConcern`：指定写入确认要求

```JavaScript
{ writeConcern: { w: "majority" } }
```

MongoDB 需要以什么程度确认这次写入已经被接受/持久化到副本集？`w: "majority"` 大致表示等待多数副本确认这次写操作。


---


# 数据建模

[Data Modeling in MongoDB](https://www.mongodb.com/docs/v8.0/data-modeling/)
[Best Practices for Data Modeling in MongoDB](https://www.mongodb.com/docs/v8.0/data-modeling/best-practices/)
- [Embedded Data in Your MongoDB Schema](https://www.mongodb.com/docs/v8.0/data-modeling/embedding/)
- [Reference Data in Your MongoDB Schema](https://www.mongodb.com/docs/v8.0/data-modeling/referencing/)
- [Handle Duplicate Data](https://www.mongodb.com/docs/v8.0/data-modeling/handle-duplicate-data/)
- [Data Consistency](https://www.mongodb.com/docs/v8.0/data-modeling/data-consistency/)
    - [Enforce Data Consistency with Transactions](https://www.mongodb.com/docs/v8.0/data-modeling/enforce-consistency/transactions/)
    - [Enforce Data Consistency with Embedding](https://www.mongodb.com/docs/v8.0/data-modeling/enforce-consistency/embed-data/)
- [Schema Validation](https://www.mongodb.com/docs/v8.0/core/schema-validation/)
    - [Specify JSON Schema Validation](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-json-schema/)
        - [Specify Allowed Field Values](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-json-schema/specify-allowed-field-values/)
        - [Tips for JSON Schema Validation](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-json-schema/json-schema-tips/)
    - [Specify Validation With Query Operators](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-query-expression-rules/)
    - [Specify Validation Level for Existing Documents](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-validation-level/)
    - [Choose How to Handle Invalid Documents](https://www.mongodb.com/docs/v8.0/core/schema-validation/handle-invalid-documents/)
    - [Bypass Schema Validation](https://www.mongodb.com/docs/v8.0/core/schema-validation/bypass-document-validation/)
    - [View Existing Validation Rules](https://www.mongodb.com/docs/v8.0/core/schema-validation/view-existing-validation-rules/)
    - [Modify Schema Validation](https://www.mongodb.com/docs/v8.0/core/schema-validation/update-schema-validation/)
    - [Query for and Modify Valid or Invalid Documents](https://www.mongodb.com/docs/v8.0/core/schema-validation/use-json-schema-query-conditions/)
    - [Specify Validation for Polymorphic Collections](https://www.mongodb.com/docs/v8.0/core/schema-validation/specify-validation-polymorphic-collections/)
- [Designing Your Schema](https://www.mongodb.com/docs/v8.0/data-modeling/schema-design-process/)
    - [Identify Application Workload](https://www.mongodb.com/docs/v8.0/data-modeling/schema-design-process/identify-workload/)
    - [Map Schema Relationships](https://www.mongodb.com/docs/v8.0/data-modeling/schema-design-process/map-relationships/)
    - [Apply Design Patterns](https://www.mongodb.com/docs/v8.0/data-modeling/schema-design-process/apply-patterns/)
    - [Create Indexes to Support Your Queries](https://www.mongodb.com/docs/v8.0/data-modeling/schema-design-process/create-indexes/)
- [Schema Design Patterns](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/)
    - [Handle Computed Values](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/handle-computed-values/)
    - [Group Data](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/group-data/)
    - [Polymorphic Data](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/polymorphic-data/)
    - [Document and Schema Versioning](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/data-versioning/)
    - [Archive Pattern](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/archive/)
    - [Single Collection Pattern](https://www.mongodb.com/docs/v8.0/data-modeling/design-patterns/single-collection/)
- [Schema Design Anti-Patterns](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/)
    - [Avoid Unbounded Arrays](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/unbounded-arrays/)
    - [Reduce the Number of Collections](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/reduce-collections/)
    - [Remove Unnecessary Indexes](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/unnecessary-indexes/)
    - [Bloated Documents](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/bloated-documents/)
    - [Reduce `$lookup` Operations](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/reduce-lookup-operations/)
- [Document Relationships](https://www.mongodb.com/docs/v8.0/applications/data-models-relationships/)
    - [Model One-to-One Relationships with Embedded Documents](https://www.mongodb.com/docs/v8.0/tutorial/model-embedded-one-to-one-relationships-between-documents/)
    - [Model One-to-Many Relationships with Embedded Documents](https://www.mongodb.com/docs/v8.0/tutorial/model-embedded-one-to-many-relationships-between-documents/)
    - [Model One-to-Many Relationships with Document References](https://www.mongodb.com/docs/v8.0/tutorial/model-referenced-one-to-many-relationships-between-documents/)
    - [Model Many-to-Many Relationships with Embedded Documents](https://www.mongodb.com/docs/v8.0/tutorial/model-embedded-many-to-many-relationships-between-documents/)
- [Model Tree Structures](https://www.mongodb.com/docs/v8.0/applications/data-models-tree-structures/)
    - [Model Tree Structures with Parent References](https://www.mongodb.com/docs/v8.0/tutorial/model-tree-structures-with-parent-references/)
    - [Model Tree Structures with Child References](https://www.mongodb.com/docs/v8.0/tutorial/model-tree-structures-with-child-references/)
    - [Model Tree Structures with an Array of Ancestors](https://www.mongodb.com/docs/v8.0/tutorial/model-tree-structures-with-ancestors-array/)
    - [Model Tree Structures with Materialized Paths](https://www.mongodb.com/docs/v8.0/tutorial/model-tree-structures-with-materialized-paths/)
    - [Model Tree Structures with Nested Sets](https://www.mongodb.com/docs/v8.0/tutorial/model-tree-structures-with-nested-sets/)
- [Model Specific Application Contexts](https://www.mongodb.com/docs/v8.0/applications/data-models-applications/)
    - [Model Data for Atomic Operations](https://www.mongodb.com/docs/v8.0/tutorial/model-data-for-atomic-operations/)
    - [Model Data to Support Keyword Search](https://www.mongodb.com/docs/v8.0/tutorial/model-data-for-keyword-search/)
    - [Model Monetary Data](https://www.mongodb.com/docs/v8.0/tutorial/model-monetary-data/)
    - [Model IoT Data](https://www.mongodb.com/docs/v8.0/tutorial/model-iot-data/)



## 最佳实践


对于 MongoDB 的数据建模，应该先分析工作负载，而不是先设计试题，虽然 MongoDB 的模式可以迭代，但是还是尽早规划为佳。

嵌入和引用的总判断规则：

|问题|更倾向嵌入|更倾向引用|
|---|---|---|
|是否总是一起读取？|是|否|
|是否总是一起更新？|是|否|
|子对象能否脱离父对象独立存在？|不能|能|
|子对象数量是否有明确上限？|有|无上限或很大|
|是否经常单独查询子对象？|很少|经常|
|是否为复杂多对多关系？|通常不适合|适合|
|嵌入后是否需要大量复制？|少量或可接受|复制成本过高|
|子对象更新频率是否很高？|低或与父对象一起更新|高|
|是否要求一次写入原子地修改全部相关数据？|很适合|可能需要事务|

### 嵌入数据

其实就是把相关数据作为子文档或者数组，存放在同一个 BSON 文档中。

很适合使用嵌入的情况：
- 包含或者所有权关系
- 数量有明确上限
- 总是和父对象一起读取或者更新
- 生命周期和父对象一致：例如删除订单时，订单的明细也应该一起删除。

使用嵌入的风险或注意事项：
- document 大小有上限，所以在最开始需要考虑数据增长之后的最坏情况。
- 无界数组
- 只需要少量子数据，却反复读取整个 document
- 高并发更新集中到同一个 document：会形成性能瓶颈。

### 引用数据

通常是在一个 document 中保持另一个 document 的 `_id`。

适合引用的情况：
- 复杂的多对多关系
- 子对象数量很大或没有上限
- 关联对象经常单独查询
- 关联对象有独立生命周期
- 关联对象更新频繁
- 嵌入会造成大量副本
- 父对象与子对象分别写入
- 子对象体积较大

MongoDB 不自动保证引用完整性，需要应用自行处理。例如在删除了主 document 之后，其引用的记录并不会自动删除，这类失去有效关联对象的记录，就是**孤儿记录**。

### 数据冗余

有意识的数据冗余不是单纯的空间换性能，而是**用额外存储和写入维护成本，换取更高效的读取**；是否值得，取决于数据变化频率，以及你对数据陈旧的容忍程度。

常见的冗余数据：

|数据类型|例子|副本应该怎样处理？|维护成本|
|---|---|---|---|
|不变数据|账号创建时间|创建后基本不需要更新|低|
|历史快照|下单时的商品价格|保留当时的值，不随源数据改变|低|
|对陈旧敏感|商品当前库存、玩家当前余额|如果复制，需要及时同步|高|
|可容忍陈旧|推荐列表、分析统计结果|可以通过后台任务延迟刷新|较低，但有刷新成本|

数据冗余并不是免费的：

|对比|不复制数据|复制数据|
|---|---|---|
|存储空间|较少|较多|
|查询过程|可能需要额外查询或 `$lookup`|可以直接读取|
|写入复杂度|通常较低|可能需要同步多个副本|
|数据一致性|维护相对简单|需要明确副本的更新规则|
|适用情况|数据频繁变化，或关联查询成本可接受|查询频繁，且复制数据的维护成本可接受|


数据冗余并不是无脑的复制数据，而是只复制需要频繁查询的字段。

### 数据一致性

研究的是同一事实被保持在多个位置是，如何保持一致。常见方法：

|方案|核心思路|适用条件|主要代价|
|---|---|---|---|
|Transactions|多个文档一起提交或回滚|多份数据必须原子更新|事务管理和额外性能成本|
|Embedding|调整 Schema，将相关数据合并存储|数据关系适合嵌入|Document 大小、访问模式限制|
|Atlas Database Triggers|数据修改后自动触发其他更新|允许短暂不一致|传播延迟、失败处理和运维成本|

还有一种常见的方式：不复制，值保持引用，每次都读取权威数据。

在处理数据一致性时需要量化“允许多久”—— 也就是可以容忍延迟多久。
- 如果容忍度高，可以考虑异步更新。
- 容忍度低，则考虑使用事务进行同步更新。


---

## Schema Validation

MongoDB 虽然允许同一个 collection 中的 document 具有不同结构，灵活模式并不是没有模式，可以通过模式验证来统一把关建立数据约束。

模式验证的相关概念：

|配置|解决的问题|
|---|---|
|`validator`|什么样的文档才算有效|
|`validationLevel`|哪些写操作需要执行校验|
|`validationAction`|校验失败后拒绝还是只记录警告|
|`bypassDocumentValidation`|是否允许某一次操作临时绕过校验|

### JSON Schema 验证

一个例子：

```JavaScript
db.createCollection("students", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      title: "Student document",
      required: ["name", "year"],
      properties: {
        name: {
          bsonType: "string",
          minLength: 1,
          description: "name 必须是非空字符串"
        },
        year: {
          bsonType: "int",
          minimum: 2017,
          maximum: 3017,
          description: "year 必须是指定范围内的 Int32"
        },
        gpa: {
          bsonType: ["double", "null"],
          minimum: 0,
          maximum: 4,
          description: "gpa 必须是 0～4 的 double，也可以是 null"
        },
        status: {
          enum: ["active", "graduated", "suspended"]
        }
      }
    }
  },
  
  validationLevel: "strict",
  
  validationAction: "error"
})
```


| 规则                      | 含义                     | 例子                          |
| ------------------------- | ------------------------ | ----------------------------- |
| `bsonType`                | 限定 BSON 类型           | `bsonType: "string"`          |
| `required`                | 指定必须存在的字段       | `required: ["name", "gold"]`  |
| `minimum` / `maximum`     | 限定数值范围             | `minimum: 0`                  |
| `minLength` / `maxLength` | 限定字符串长度           | `minLength: 1`                |
| `enum`                    | 限定允许的值             | `enum: ["online", "offline"]` |
| `pattern`                 | 使用正则表达式约束字符串 | `pattern: "^[A-Z]+$"`         |
| `items`                   | 限定数组元素的规则       | 数组元素必须是整数            |
| `properties`              | 描述字段及其约束         | `gold: { bsonType: "int" }`   |
| `additionalProperties`  |    是否允许出现properties中没有声明的字段  |   `false`表示不允许   |

另外， 文档校验失败时，MongoDB 会把相关的 `title` 和 `description` 放入错误信息。


需要注意：
- 当 `additionalProperties: false` 时，一定要把 `_id` 也列入 `properties`。
- 字段不存在和字段存在但是值为 null 是两种情况。

某些写操作可以显式使用 `bypassDocumentValidation: true` **绕过验证**，但启用了访问控制时，执行者必须具有 `bypassDocumentValidation` 权限。例如：

```JavaScript
db.runCommand({
  insert: "students",
  documents: [
    {
      name: "Alice",
      year: Int32(2016)
    }
  ],
  bypassDocumentValidation: true
})
```

查看某个 collection 当前验证规则：

```JavaScript
db.getCollectionInfos(
    { name: "players" }
)[0].options
```

更新校验规则：

```JavaScript
db.runCommand({
    collMod: "players",
    validator: {
        $jsonSchema: {
            required: ["name", "gold"],
            properties: {
                name: { bsonType: "string" },
                gold: {
                    bsonType: ["int", "long"],
                    minimum: 0
                }
            }
        }
    }
})
```

实践中应把传入的 `validator` 当成完整的新校验器，不要认为 MongoDB 会把它与旧规则递归合并。


### 查询运算符表达跨字段规则

JSON Schema 很适合检查字段类型、是否存在以及数值范围，但有些规则设计多个字段之间的关系。

例如，一个订单有：

```JavaScript
{
    price: 100,
    quantity: 2,
    total: 200
}
```

其中业务规定： $total = price \times quantity$ ，则需要：

```JavaScript
db.createCollection("orders", {
    validator: {
        $expr: {
            $eq: [
                "$total",
                { $multiply: ["$price", "$quantity"] }
            ]
        }
    }
})
```


### Validation Level

一个集合已经存储了很多数据，后来才添加验证规则，`validationLevel` 决定 MongoDB 如何对待**插入和更新**操作：


| `validationLevel` | 行为                                                                                                 | 适用场景                         |
| ----------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------- |
| `strict`（默认）  | 对所有插入和更新执行验证                                                                             | 数据结构已明确，希望严格执行约束 |
| `moderate`        | 对新插入的文档，以及原本符合规则的已有文档执行验证；原本不符合规则的旧文档更新时，不强制要求通过验证 | 逐步迁移旧数据                   |
|     `off`        |      不执行文档校验          |                                  |

`strict` 不会在你添加验证规则时自动扫描并修复所有历史数据。

### Validation Action

`validationAction` 决定验证失败后，MongoDB 怎么处理。

|配置|行为|
|---|---|
|`error`（默认）|拒绝写入，并返回验证错误|
|`warn`|允许写入，但将违反规则的情况记录到 MongoDB 日志|

常见组合如下：

|组合|适用情形|
|---|---|
|`strict + error`|稳定生产环境，全面强制约束|
|`strict + warn`|上线新规则前观察所有违规写入|
|`moderate + error`|阻止新坏数据，但允许维护历史脏数据|
|`off`|临时完全关闭校验，不适合长期使用|

### 处理有效或无效文档

把 `$jsonSchema` 用作查询条件；查找不符合条件的文档时，可以使用 `$nor` 对其取反。


先定义规则：

```JavaScript
const rules = {
  $jsonSchema: {
    required: ["item", "qty", "instock"],
    properties: {
      item: {
        bsonType: "string"
      },
      qty: {
        bsonType: "int"
      },
      instock: {
        bsonType: "bool"
      }
    }
  }
}
```

然后把规则当成查询条件，例如：

```JavaScript
db.inventory.find({
  $nor: [rules]
})
```


### 多态集合的验证

如果同一个 Collection 本来就需要存储不同结构的 Document，可以使用多个 Schema，并通过 `oneOf` 等规则限制每种文档类型。

例如：

```JavaScript
db.runCommand({
  collMod: "accounts",
  validator: {
    $jsonSchema: {
      oneOf: [
        customerSchema,
        accountSchema
      ]
    }
  }
})
```

其中 customerSchema、accountSchema 是提前定义好的规则，`oneof` 的含义是**恰好满足其中一种** Schema。

---

## Schema 设计反模式


反模式（Anti-pattern） 指的是：一种看起来能够解决问题、甚至在短期内很方便，但在特定场景下会反复导致性能、维护成本或可靠性问题的设计方式。

它不是说这种写法绝对错误，而是提醒：当系统具备某些特征时，这种设计通常不是一个好的选择。

### 避免无界数组

无界数组：数组元素可能持续增长，而 Schema 设计没有可靠的限制其规模。

**子集模式**

把最常用的一部分数据（限制数量，因此是有界的）嵌入主 document 中，另外使用一个 collection 保持完整数据。

优点：
- 控制了主 document 的大小。
- 高频访问数据一次查询就能得到结果。不必执行额外查询。

缺点：
- 数据冗余。
- 可能需要同时更新两个地方，考虑数据一致性问题。

**引用模式**

如果数据量很大、持续增长，或者需要频繁独立查询和更新，此时使用子集模式并不能解决问题，反而增添数据冗余问题，考虑使用引用模式。

使用引用模式时，不要在主 document 中嵌入数组保存引用 document 的 `_id`，这样依然会产生无界数组问题。恰当的处理方法是：让子 document 保存父对象的 `_id`。

---