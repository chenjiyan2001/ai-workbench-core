# Redis 使用规范

## 适用范围

本规范适用于所有使用 Redis 的场景，包括缓存、会话、队列、分布式锁等。

## Key 命名规范

### 命名格式

```
格式: {业务}:{模块}:{标识}
使用冒号分隔，小写字母

✅ 正确
user:profile:12345           -- 用户资料
cache:query:md5hash          -- 查询缓存
session:token:abc123         -- 会话 Token
lock:task:processing:456     -- 分布式锁
queue:email:pending          -- 消息队列
counter:api:rate:user:789    -- 限流计数器

❌ 错误
user_profile_12345           -- 应使用冒号
UserProfile:12345            -- 应全小写
user:profile:                -- 不要空标识
12345                        -- 无业务前缀
```

### Key 长度限制

- 建议 Key 长度不超过 128 字符
- 避免过长的 Key 名称，影响内存和网络传输

### 常用前缀约定

| 前缀 | 用途 | 示例 |
|------|------|------|
| `cache:` | 缓存数据 | `cache:user:123` |
| `session:` | 会话数据 | `session:token:xxx` |
| `lock:` | 分布式锁 | `lock:order:processing:456` |
| `queue:` | 消息队列 | `queue:task:pending` |
| `counter:` | 计数器 | `counter:daily:visits` |
| `rate:` | 限流 | `rate:api:user:789` |
| `tmp:` | 临时数据 | `tmp:import:batch:xxx` |

## 数据类型选择

### 类型对照表

| 场景 | 推荐类型 | 示例 |
|------|---------|------|
| 简单值缓存 | String | 用户信息 JSON |
| 对象属性 | Hash | 用户多个字段 |
| 有序列表 | List | 最近访问记录 |
| 去重集合 | Set | 用户标签 |
| 排行榜 | Sorted Set (ZSet) | 积分排名 |
| 位图统计 | Bitmap | 用户签到 |
| 基数统计 | HyperLogLog | UV 统计 |

### String

```python
import redis.asyncio as redis

# 简单缓存
await r.set("cache:user:123", json.dumps(user_data), ex=3600)
data = await r.get("cache:user:123")

# 计数器 (原子操作)
await r.incr("counter:api:calls")
await r.incrby("counter:page:views", 10)

# 分布式锁
await r.set("lock:task:123", "owner_id", nx=True, ex=30)
```

### Hash

```python
# ✅ 适合: 对象的多个字段，部分读写
await r.hset("user:123", mapping={
    "name": "张三",
    "email": "zhangsan@example.com",
    "status": "active",
})

# 获取单个字段
name = await r.hget("user:123", "name")

# 获取多个字段
data = await r.hmget("user:123", ["name", "email"])

# 获取所有字段
user = await r.hgetall("user:123")

# 原子递增
await r.hincrby("user:123", "login_count", 1)
```

### List

```python
# ✅ 适合: 有序列表，队列
# 最近访问记录 (保留最近 100 条)
await r.lpush("recent:user:123", item_id)
await r.ltrim("recent:user:123", 0, 99)

# 简单队列
await r.rpush("queue:tasks", task_data)  # 生产者
task = await r.lpop("queue:tasks")       # 消费者

# 阻塞队列
task = await r.blpop("queue:tasks", timeout=30)
```

### Sorted Set

```python
# ✅ 适合: 排行榜，延时队列
# 积分排行榜
await r.zadd("leaderboard:daily", {"user:123": 1000, "user:456": 800})

# 获取排名 (从高到低)
top10 = await r.zrevrange("leaderboard:daily", 0, 9, withscores=True)

# 获取用户排名
rank = await r.zrevrank("leaderboard:daily", "user:123")

# 延时队列 (score 为执行时间戳)
await r.zadd("delayed:tasks", {task_id: execute_at_timestamp})
# 获取到期任务
tasks = await r.zrangebyscore("delayed:tasks", 0, time.time())
```

## 过期策略

### 必须设置过期时间

```python
# ✅ 正确: 设置过期时间
await r.set("cache:user:123", data, ex=3600)  # 1小时
await r.setex("session:token:xxx", 86400, session_data)  # 1天

# ❌ 错误: 不设置过期时间 (可能导致内存溢出)
await r.set("cache:user:123", data)
```

### 过期时间建议

| 场景 | 过期时间 | 说明 |
|------|---------|------|
| 热点数据缓存 | 5-30 分钟 | 频繁访问 |
| 普通数据缓存 | 1-24 小时 | 一般场景 |
| 会话 Token | 1-7 天 | 根据安全要求 |
| 分布式锁 | 10-60 秒 | 防止死锁 |
| 临时数据 | 5-30 分钟 | 处理完即删 |

### 过期时间随机化

```python
import random

# ✅ 防止缓存雪崩: 过期时间加随机偏移
base_ttl = 3600
jitter = random.randint(0, 300)  # 0-5 分钟随机
await r.set(key, value, ex=base_ttl + jitter)
```

## 缓存模式

### Cache-Aside (旁路缓存)

```python
async def get_user(user_id: int) -> dict:
    """Cache-Aside 模式"""
    cache_key = f"cache:user:{user_id}"

    # 1. 先查缓存
    cached = await r.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. 缓存未命中，查数据库
    user = await db.fetch_user(user_id)
    if user is None:
        return None

    # 3. 写入缓存
    await r.set(cache_key, json.dumps(user), ex=3600)
    return user


async def update_user(user_id: int, data: dict) -> None:
    """更新时删除缓存"""
    await db.update_user(user_id, data)
    await r.delete(f"cache:user:{user_id}")  # 删除而非更新
```

### 缓存穿透防护

```python
async def get_user(user_id: int) -> dict | None:
    """防止缓存穿透: 缓存空值"""
    cache_key = f"cache:user:{user_id}"

    cached = await r.get(cache_key)
    if cached == "NULL":  # 空值标记
        return None
    if cached:
        return json.loads(cached)

    user = await db.fetch_user(user_id)
    if user is None:
        # 缓存空值，短过期时间
        await r.set(cache_key, "NULL", ex=300)
        return None

    await r.set(cache_key, json.dumps(user), ex=3600)
    return user
```

## 分布式锁

### 标准实现

```python
import uuid

async def acquire_lock(
    key: str,
    timeout: int = 10,
    blocking: bool = True,
    block_timeout: int = 5,
) -> str | None:
    """获取分布式锁"""
    lock_key = f"lock:{key}"
    token = str(uuid.uuid4())

    if blocking:
        end_time = time.time() + block_timeout
        while time.time() < end_time:
            if await r.set(lock_key, token, nx=True, ex=timeout):
                return token
            await asyncio.sleep(0.1)
        return None
    else:
        if await r.set(lock_key, token, nx=True, ex=timeout):
            return token
        return None


async def release_lock(key: str, token: str) -> bool:
    """释放分布式锁 (Lua 脚本保证原子性)"""
    lock_key = f"lock:{key}"
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    result = await r.eval(script, 1, lock_key, token)
    return result == 1


# 使用
async def process_task(task_id: str) -> None:
    token = await acquire_lock(f"task:{task_id}")
    if not token:
        raise Exception("获取锁失败")
    try:
        await do_work()
    finally:
        await release_lock(f"task:{task_id}", token)
```

### 使用 redis-py 内置锁

```python
from redis.asyncio.lock import Lock

async def process_with_lock(task_id: str) -> None:
    lock = Lock(r, f"lock:task:{task_id}", timeout=30)
    async with lock:
        await do_work()
```

## 连接池配置

```python
import redis.asyncio as redis

# 推荐配置
pool = redis.ConnectionPool(
    host="localhost",
    port=6379,
    db=0,
    max_connections=50,           # 最大连接数
    socket_timeout=5,             # 读写超时
    socket_connect_timeout=5,     # 连接超时
    retry_on_timeout=True,        # 超时重试
    decode_responses=True,        # 自动解码
)

r = redis.Redis(connection_pool=pool)
```

## 禁止事项

- ❌ 禁止使用 `KEYS *` 命令 (生产环境)
- ❌ 禁止存储大 Value (超过 1MB)
- ❌ 禁止不设置过期时间
- ❌ 禁止在事务中使用阻塞命令
- ❌ 禁止使用 `FLUSHALL` / `FLUSHDB` (生产环境)
- ❌ 禁止存储敏感信息明文

## 检查清单

- [ ] Key 命名符合规范
- [ ] 所有 Key 设置了过期时间
- [ ] 数据类型选择正确
- [ ] 大 Value 已拆分或压缩
- [ ] 分布式锁有超时和释放机制
- [ ] 缓存有穿透/雪崩防护
- [ ] 连接池配置合理
