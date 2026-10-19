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