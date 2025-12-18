# MongoDB 使用规范

## 适用范围

本规范适用于所有使用 MongoDB 数据库的场景。

## 命名规范

### 数据库命名

```
格式: {项目}_{模块}
小写，下划线分隔

✅ 正确
workbench_docs
workbench_logs
workbench_metadata

❌ 错误
WorkbenchDocs     -- 不要驼峰
workbench-docs    -- 不要横线
```

### 集合命名

```
格式: {模块}_{实体}
小写，下划线分隔，使用复数形式 (与 MySQL 不同)

✅ 正确
users
task_logs
data_source_configs
llm_call_records

❌ 错误
User              -- 不要首字母大写
user              -- 应使用复数
tbl_users         -- 不要前缀
```

### 字段命名

```
使用 camelCase (MongoDB 惯例)

✅ 正确
_id               -- 主键 (MongoDB 默认)
userId            -- 引用 ID
userName          -- 用户名
createdAt         -- 创建时间
updatedAt         -- 更新时间
isDeleted         -- 布尔值

❌ 错误
user_id           -- MongoDB 惯用 camelCase
created_at        -- 应为 createdAt
```

## 文档设计

### 基础结构

```javascript
// 每个文档应包含的基础字段
{
  "_id": ObjectId("..."),
  // 业务字段...
  "createdAt": ISODate("2024-01-15T10:30:00Z"),
  "updatedAt": ISODate("2024-01-15T10:30:00Z"),
  "version": 1  // 可选: 乐观锁
}
```

### 嵌入 vs 引用

```javascript
// ✅ 嵌入: 数据经常一起查询，且不会无限增长
{
  "_id": ObjectId("..."),
  "name": "张三",
  "addresses": [  // 地址数量有限，嵌入合适
    {"city": "北京", "street": "..."},
    {"city": "上海", "street": "..."}
  ]
}

// ✅ 引用: 数据独立变化，或可能无限增长
{
  "_id": ObjectId("..."),
  "name": "张三",
  "orderIds": [ObjectId("..."), ...]  // 订单可能很多，使用引用
}

// ❌ 错误: 无限增长的嵌入数组
{
  "name": "产品A",
  "comments": [  // 评论可能无限增长
    {"user": "...", "content": "..."},
    // ... 可能有成千上万条
  ]
}
```

### 数组大小限制

```javascript
// 嵌入数组应有明确上限
// MongoDB 文档大小限制: 16MB

// ✅ 正确: 限制数组大小
{
  "recentTags": {
    "$slice": -100  // 只保留最近 100 个
  }
}

// 或在应用层控制
if (doc.tags.length >= 100) {
  // 转为引用或分页
}
```

## 索引规范

### 索引原则

1. `_id` 自动创建唯一索引
2. 查询条件字段必须有索引
3. 联合索引遵循 ESR 原则 (Equality, Sort, Range)
4. 单集合索引不超过 10 个

### 索引类型

```javascript
// 单字段索引
db.users.createIndex({ "email": 1 })

// 联合索引 (遵循 ESR 原则)
db.tasks.createIndex({ "status": 1, "createdAt": -1 })

// 唯一索引
db.users.createIndex({ "email": 1 }, { unique: true })

// TTL 索引 (自动过期)
db.logs.createIndex({ "createdAt": 1 }, { expireAfterSeconds: 2592000 })  // 30天

// 文本索引
db.articles.createIndex({ "title": "text", "content": "text" })

// 部分索引 (只索引满足条件的文档)
db.orders.createIndex(
  { "status": 1 },
  { partialFilterExpression: { "status": { "$eq": "pending" } } }
)
```

### ESR 原则

```javascript
// ESR: Equality, Sort, Range
// 索引字段顺序: 等值查询字段 → 排序字段 → 范围查询字段

// 查询
db.tasks.find({
  "status": "pending",        // Equality
  "priority": { "$gte": 3 }   // Range
}).sort({ "createdAt": -1 })  // Sort

// ✅ 正确的索引顺序
db.tasks.createIndex({ "status": 1, "createdAt": -1, "priority": 1 })

// ❌ 错误的索引顺序
db.tasks.createIndex({ "createdAt": -1, "status": 1, "priority": 1 })
```

## 查询规范

### Python 驱动使用

```python
from motor.motor_asyncio import AsyncIOMotorClient
from pymongo import ASCENDING, DESCENDING

# 连接配置
client = AsyncIOMotorClient(
    "mongodb://localhost:27017",
    maxPoolSize=50,
    minPoolSize=10,
    maxIdleTimeMS=30000,
)
db = client.workbench_docs


# ✅ 正确: 明确投影字段
async def get_user(user_id: str) -> dict | None:
    return await db.users.find_one(
        {"_id": ObjectId(user_id)},
        projection={"name": 1, "email": 1, "createdAt": 1},
    )


# ✅ 正确: 使用索引提示
async def get_pending_tasks(limit: int = 100) -> list[dict]:
    cursor = db.tasks.find(
        {"status": "pending"},
        projection={"title": 1, "priority": 1, "createdAt": 1},
    ).sort("createdAt", DESCENDING).limit(limit).hint("status_1_createdAt_-1")
    return await cursor.to_list(length=limit)


# ❌ 错误: 无投影，返回所有字段
async def get_user(user_id: str) -> dict | None:
    return await db.users.find_one({"_id": ObjectId(user_id)})
```

### 批量操作

```python
from pymongo import UpdateOne, InsertOne

# ✅ 正确: 使用 bulk_write 批量操作
async def batch_update_status(task_ids: list[str], status: str) -> int:
    operations = [
        UpdateOne(
            {"_id": ObjectId(tid)},
            {"$set": {"status": status, "updatedAt": datetime.utcnow()}},
        )
        for tid in task_ids
    ]
    result = await db.tasks.bulk_write(operations, ordered=False)
    return result.modified_count


# ❌ 错误: 循环单条更新
async def batch_update_status(task_ids: list[str], status: str) -> None:
    for tid in task_ids:
        await db.tasks.update_one({"_id": ObjectId(tid)}, {"$set": {"status": status}})
```

### 聚合管道

```python
# ✅ 正确: 使用聚合管道
async def get_task_stats_by_status() -> list[dict]:
    pipeline = [
        {"$match": {"createdAt": {"$gte": start_date}}},  # 先过滤
        {"$group": {
            "_id": "$status",
            "count": {"$sum": 1},
            "avgPriority": {"$avg": "$priority"},
        }},
        {"$sort": {"count": -1}},
    ]
    cursor = db.tasks.aggregate(pipeline)
    return await cursor.to_list(length=None)
```

## 更新规范

### 原子更新操作

```python
# ✅ 正确: 使用原子操作符
await db.counters.update_one(
    {"_id": "page_views"},
    {
        "$inc": {"count": 1},              # 原子递增
        "$set": {"updatedAt": datetime.utcnow()},
        "$setOnInsert": {"createdAt": datetime.utcnow()},
    },
    upsert=True,
)

# ✅ 正确: 数组操作
await db.users.update_one(
    {"_id": user_id},
    {
        "$push": {"tags": {"$each": new_tags, "$slice": -100}},  # 限制数组大小
        "$set": {"updatedAt": datetime.utcnow()},
    },
)

# ❌ 错误: 先读后写 (非原子)
doc = await db.counters.find_one({"_id": "page_views"})
doc["count"] += 1
await db.counters.replace_one({"_id": "page_views"}, doc)
```

### 乐观锁

```python
async def update_with_optimistic_lock(
    doc_id: str,
    updates: dict,
    expected_version: int,
) -> bool:
    """乐观锁更新"""
    result = await db.documents.update_one(
        {"_id": ObjectId(doc_id), "version": expected_version},
        {
            "$set": {**updates, "updatedAt": datetime.utcnow()},
            "$inc": {"version": 1},
        },
    )
    return result.modified_count == 1
```

## 事务使用

```python
# MongoDB 4.0+ 支持多文档事务
async def transfer_funds(from_id: str, to_id: str, amount: float) -> None:
    async with await client.start_session() as session:
        async with session.start_transaction():
            # 扣款
            await db.accounts.update_one(
                {"_id": from_id, "balance": {"$gte": amount}},
                {"$inc": {"balance": -amount}},
                session=session,
            )
            # 入款
            await db.accounts.update_one(
                {"_id": to_id},
                {"$inc": {"balance": amount}},
                session=session,
            )
```

## Schema 验证

```javascript
// 使用 JSON Schema 验证文档结构
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "createdAt"],
      properties: {
        name: {
          bsonType: "string",
          minLength: 1,
          maxLength: 100
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+$"
        },
        status: {
          enum: ["active", "inactive", "deleted"]
        },
        createdAt: {
          bsonType: "date"
        }
      }
    }
  },
  validationLevel: "moderate",  // strict / moderate
  validationAction: "error"     // error / warn
})
```

## 禁止事项

- ❌ 禁止无投影的查询 (返回所有字段)
- ❌ 禁止无索引的查询
- ❌ 禁止无限增长的嵌入数组
- ❌ 禁止使用 `$where` 操作符 (性能差)
- ❌ 禁止在应用层做 join (应重新设计数据模型)
- ❌ 禁止忽略写入结果检查

## 检查清单

- [ ] 集合/字段命名符合规范
- [ ] 文档包含 createdAt/updatedAt
- [ ] 查询有明确的投影 (projection)
- [ ] 关键查询有索引支持
- [ ] 嵌入数组有大小限制
- [ ] 批量操作使用 bulk_write
- [ ] 高并发场景使用原子操作
