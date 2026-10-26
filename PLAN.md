
## 三层掌握标准

|等级|验收要求|
|---|---|
|**A：能实操**|能写命令或 Lua table、预测结果、解释返回值、指出边界和坑|
|**B：能解释**|能说明用途、核心机制、主要取舍和故障边界；不要求搭生产集群|
|**C：认识概念**|60～90 秒内说明它是什么、什么时候可能用、与相邻技术有什么区别|

知识卡固定五行：

```
名称：
是什么：
解决什么问题：
主要入口/命令：
一个限制或坑：
```

---

# MongoDB 覆盖范围

统一从 [MongoDB 8.0 Manual](https://www.mongodb.com/docs/v8.0/) 和 [Manual Contents](https://www.mongodb.com/docs/v8.0/contents/) 进入。

## A 类：必须会操作

1. **文档基础**
    
    - Database、Collection、Document
    - BSON 常用类型、`_id`、ObjectId
    - 数组、嵌套文档、dot notation
    - null 与字段不存在
    - 16 MiB 限制
2. **CRUD 与 MQL**
    
    - `insertOne/insertMany`
    - `find`、Projection、sort、skip、limit、Cursor
    - `updateOne/updateMany/replaceOne`
    - `deleteOne/deleteMany`
    - `findOneAndUpdate/findAndModify`
    - `bulkWrite`、ordered/unordered
    - upsert、返回结果和异常
3. **查询条件**
    
    - 比较、逻辑、元素、数组查询
    - `$in/$nin/$exists/$type/$elemMatch`
    - 正则和嵌套字段
    - 数字、字符串、日期类型差异
4. **更新操作**
    
    - `$set/$unset/$inc/$min/$max/$rename`
    - `$push/$addToSet/$pull`
    - `$each/$slice/$position`
    - `$`、`$[]`、$identifier
    - `arrayFilters`
    - $setOnInsert
5. **建模与校验**
    
    - 访问模式优先
    - 嵌入与引用
    - 一对一、一对多、多对多
    - 无界数组、重复数据
    - Schema Validation
6. **索引与性能**
    
    - 单字段、复合、唯一、多键、TTL、部分索引
    - 复合索引前缀、ESR
    - 排序、覆盖查询、写放大
    - `explain()`、`COLLSCAN/IXSCAN/FETCH`
    - `totalKeysExamined/totalDocsExamined`
7. **聚合**
    
    - Pipeline 和阶段间数据流
    - `$match/$project/$set/$unwind`
    - `$group/$sort/$limit/$count`
    - `$lookup/$facet`
    - `$out/$merge`
    - 表达式和累加器
8. **数据正确性**
    
    - 单文档原子性
    - 条件更新
    - 原子性与幂等性的区别
    - 最小多文档事务

## B 类：必须能解释

- read concern、write concern、read preference
- Session、因果一致性
- Retryable Reads/Writes
- 事务的 start、commit、abort、重试和成本
- Replica Set、Primary、Secondary、oplog、选举、failover、rollback、复制延迟
- Change Streams、resume token
- Sharding、shard、mongos、config server
- Range/Hashed Shard Key
- chunk、balancer、targeted query、scatter-gather
- SCRAM、用户、角色、最小权限、TLS
- 日志、Profiler、`serverStatus/dbStats/collStats/currentOp`
- `mongodump/mongorestore`、文件快照、RPO/RTO

## C 类：至少知道是什么

- WiredTiger、cache、checkpoint、journal、compression
- text、geospatial、hashed、wildcard、hidden、sparse index
- View、Capped Collection、Time Series Collection、GridFS
- Collation
- Stable API、Feature Compatibility Version
- Encryption at Rest、CSFLE、Queryable Encryption
- Atlas、Atlas Search、Vector Search，只需认识用途

---

# Redis 覆盖范围

统一从 [Develop with Redis](https://redis.io/docs/latest/develop/) 和 [Redis Open Source](https://redis.io/docs/latest/operate/oss_and_stack/) 进入。

先执行：

```
redis-server --version
redis-cli INFO server
```

记录公司实际 Redis 版本。较新的 Functions、Sharded Pub/Sub 等特性，如果当前版本不支持，只做到 C 类。

## A 类：必须会操作

1. **核心数据类型**
    
    - String、Hash、List、Set、Sorted Set
    - 常用命令、返回值和复杂度
    - 每种类型对应的游戏业务场景
2. **Key 与生命周期**
    
    - Key 命名
    - `TYPE/EXISTS/DEL/UNLINK`
    - `EXPIRE/PEXPIRE/TTL/PTTL/PERSIST`
    - 覆盖写对 TTL 的影响
    - 过期与淘汰的区别
3. **遍历**
    
    - `SCAN/HSCAN/SSCAN/ZSCAN`
    - Cursor、MATCH、COUNT
    - 空批、重复结果、游标归零
    - 为什么业务代码避免 `KEYS`
4. **排行榜**
    
    - `ZADD/ZINCRBY/ZSCORE`
    - `ZRANGE/ZREVRANGE`
    - `ZRANK/ZREVRANK`
    - 同分、member 唯一性、删除和更新
5. **批量与并发**
    
    - Pipeline
    - 单命令原子性
    - `MULTI/EXEC/WATCH`
    - Lua 脚本
    - GET→计算→SET 丢更新
6. **消息**
    
    - Pub/Sub 基础
    - Stream：`XADD/XREAD/XRANGE`
    - Consumer Group、PEL、ACK、pending、claim、trim
7. **排障**
    
    - `INFO`
    - `CLIENT`
    - `SLOWLOG`
    - `MEMORY`
    - 连接池、超时、重连和重试边界

## B 类：必须能解释

- Bitmap、Bitfield、Geo、HyperLogLog
- 阻塞 List
- Pub/Sub 离线丢消息和无确认机制
- Stream 消费组和消息重投
- RDB、AOF、fsync、AOF rewrite、恢复窗口
- fork 对内存和延迟的影响
- `maxmemory`、淘汰策略
- big key、hot key、内存碎片
- 异步复制、全量同步、部分同步、复制延迟
- Sentinel 的监控、quorum、故障转移、客户端发现
- Cluster slot、hash tag、`MOVED/ASK`、多 Key 限制
- ACL、protected mode、网络隔离、TLS
- Slow Log、Latency Monitor、命令统计

## C 类：至少知道是什么

- RESP 协议
- Keyspace Notification
- Client-side Caching、Tracking
- Redis Functions
- Sharded Pub/Sub
- `redis-benchmark`
- 备份、迁移、RDB 导入
- Redis Modules
- RedisJSON、Search、TimeSeries、Bloom、Vector 等扩展能力
- Redis Cloud、Redis Enterprise，只需知道与 Redis OSS 的关系

---