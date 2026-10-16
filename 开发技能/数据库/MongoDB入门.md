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

因此不能简单地认为“集群 = Replica Set”。

---

## MongoDB 的数据建模思想

MySQL 通常更强调数据之间的关系、规范化、减少冗余以及维护数据一致性。

MongoDB 通常更强调应用程序的**访问模式**，即考虑应用程序一次需要哪些数据，以及如何把经常一起访问、生命周期相关的数据组织在一起。

因此，MongoDB 中经常会根据实际业务需求，将相关数据**嵌入（Embed）**到同一个 Document 中。

---

## Document

MongoDB 的文档与 JSON 非常相似，例如：

```
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

```
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

```
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

```
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

```
"name.first"    → "Felix"
"hobbies.0"     → "coding"
```

---

## 字段名称

- 字段名称必须是字符串。
- 同一个 Document 中的字段名称必须唯一。
- 嵌套 Document 可以使用与上一级 Document 相同的字段名称，例如：
```
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
    
--

## Document 大小限制

BSON Document 的最大大小为 **16 MiB**。

因此，对于可能无限增长的数据，不应该一直嵌入同一个 Document 中。

例如玩家的历史战绩可能不断增加：

```
players
    └── match_results
            └── 不断增长
```

这种情况下应该考虑将历史数据拆分到独立的 Collection 中。

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

## Database 与 Collection 的懒创建

MongoDB 支持 Database 和 Collection 的隐式创建。

切换到一个不存在的 Database：

```
use mini_game
```

不会立即创建 Database。只有在该 Database 中**首次写入数据**时，Database 和对应的 Collection 才会被实际创建。

例如：

```
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

```
db.createCollection()
```

显式创建 Collection。


---


# BSON

BSON（Binary JSON）是一种**二进制序列化格式**，用于 MongoDB 中存储 Document，以及进行远程过程调用（RPC）。

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

- **Decimal 128** 是一种 128 位十进制浮点数类型，用于需要较高十进制精度的场景，例如金融数据。
    
- **String** 使用 UTF-8 编码。
    
- **Null** 表示字段存在，但其值明确为 `null`；**字段值为** `**null**` **与字段不存在是两个不同的概念**。
    
- **ObjectId** 长度为 12 字节，由以下部分组成：
    
    - 4 字节：Unix 时间戳，单位为秒。
        
    - 5 字节：随机值。
        
    - 3 字节：递增计数器。
        
    
    ObjectId 中包含创建时的时间戳，因此可以通过 `ObjectId.getTimestamp()` 获取其对应的时间。
    
    需要注意，ObjectId **只是大致按照创建时间排序，并不是严格递增的全局 ID**。
    
- **Date** 是一个 64 位有符号整数，表示自 Unix Epoch（1970-01-01 00:00:00 UTC）以来的毫秒数。
    
    - `ISODate()` 是 `mongosh` 提供的日期辅助函数，用于创建/表示 ISO 8601 格式的 Date。
        
- **Binary** 可以理解为一段原始二进制数据，即字节序列。
    
- **Timestamp** 主要用于 MongoDB 内部机制；对于应用程序中的业务时间，通常应该使用 **Date**。

---

# CRUD




---