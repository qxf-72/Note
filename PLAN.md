
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

# 逐日计划

## MongoDB：10 月 19—23 日、11 月 2 日

|日期|覆盖内容|实验与验收|
|---|---|---|
|**10/19**|Document、BSON、Database、Collection、`_id`；基本 CRUD；写结果|建立 8 条玩家数据，完成插入、查询、更新、删除。能解释 acknowledged、matched、modified、upserted。额外认识 View、Capped、Time Series、GridFS 是什么。|
|**10/20**|完整 Query Filter、数组查询、Projection、sort/skip/limit、Cursor；完整常用 Update Operators、arrayFilters、upsert、bulkWrite|至少完成 10 个查询、8 个更新；验证 $elemMatch、null/缺失、`$ []/$[id] `。能解释 ordered 与 unordered bulk。 |
|**10/21**|数据建模、嵌入/引用、Schema Validation、无界数组、重复数据；条件更新、幂等设计|为玩家、背包、邮件、挑战历史分别设计模型；给 players 增加校验；为发奖设计 request_id 幂等方案。|
|**10/22**|索引完整地图：单字段、复合、唯一、多键、TTL、部分、稀疏、隐藏、文本、地理、哈希、通配符；`explain()` |对至少 1 万条挑战记录比较建索引前后执行计划。A 类索引需要会用，其余索引做到 B/C。|
|**10/23**|Aggregation、单用途统计方法；原子性与事务|写一个至少包含 `$match/$unwind/$group/$sort/$limit` 的 Pipeline，再增加一次 $lookup 或 $facet；完成一个最小事务。|
|**11/2**|read/write concern、read preference、Session、retryable writes；Replica Set、Change Streams、Sharding；WiredTiger、备份、安全、监控|这一天以知识图和拓扑图为主。画副本集、分片集群和事务生命周期；用五句话分别解释至少 20 个 B/C 术语。|

## Redis：10 月 26—30 日、11 月 3 日

|日期|覆盖内容|实验与验收|
|---|---|---|
|**10/26**|String、Hash、List、Set、ZSet；Bitmap、Bitfield、Geo、HyperLogLog 概览|做五种游戏场景选型表；实作排行榜、计数器、好友集合和队列。随机抽 10 种结构，说明用途和坑。|
|**10/27**|Key、TTL、SCAN 系列；过期、淘汰；DEL 与 UNLINK；内存与 big/hot key|验证覆盖写对 TTL 的影响；完整 SCAN 到 cursor=0 并去重；检查当前 `maxmemory-policy`。|
|**10/28**|Pipeline、MULTI/EXEC/WATCH、Lua、Functions 概览；RESP、客户端连接和连接池|比较普通写入和 Pipeline；复现 GET/SET 丢更新，再分别用原子命令、WATCH、Lua 解决。|
|**10/29**|Pub/Sub、阻塞 List、Stream、Consumer Group、PEL、ACK、claim、trim；Keyspace Notification|建立一组 Stream 消费者，模拟未 ACK、消费者退出和消息重新领取；说明 Pub/Sub 为什么不适合可靠任务队列。|
|**10/30**|RDB、AOF、恢复；复制、Sentinel、Cluster；slot、hash tag、MOVED/ASK|画四张故障时间线：主库宕机、复制延迟、Sentinel 切换、Cluster 节点故障。不要求搭完整集群。|
|**11/3**|ACL、TLS、protected mode；INFO、SLOWLOG、LATENCY、MEMORY；客户端超时/重连/重试；备份迁移、模块和 Redis Stack 概览|对一个独立 Redis 实例做基础诊断；随机抽 20 个 B/C 术语，至少 16 个能在 90 秒内解释。|


---

# 11 月 4—10 日验收调整

- **11 月 4 日：数据库覆盖审计**
    
    - MongoDB 随机 20 项。
    - Redis 随机 20 项。
    - 每项 60～90 秒回答“定义、用途、入口、一个坑”。
    - 至少 **32/40 合格**，否则当天只补缺失的一级主题。
- **11 月 5 日：第一次综合模拟**
    
    - Skynet 12 题、MongoDB 4 题、Redis 4 题。
    - Mongo/Redis 题会继续追加两个变体条件。
- **11 月 6 日：代码与故障模拟**
    
    - Skynet 驱动返回值。
    - MongoDB 条件更新、事务失败。
    - Redis Pipeline、事务、主从延迟。
    - Mongo 成功但 Redis 失败的恢复方案。
- **11 月 9 日：第二次综合模拟**
    
    - 总体至少 85%。
    - 数据库 A 类题正确率至少 90%。
    - B/C 类随机术语至少 80% 能正确解释。
- **11 月 10 日：最终知识图**
    
    - MongoDB 一张二级知识树。
    - Redis 一张二级知识树。
    - 每张图必须覆盖上面列出的全部模块。
    - 晚上不再新增内容。

明确暂不纳入当前考核范围：

- MongoDB、Redis 源码和内部算法证明；
- 全部命令、全部参数、全部聚合表达式的背诵；
- 所有语言驱动；
- 完整生产集群部署；
- MongoDB Atlas/Enterprise/Ops Manager/Kubernetes 的详细配置；
- Redis Cloud/Enterprise 和各种模块的内部实现；
- 每一个版本的 Release Notes。

---