# MySQL 使用规范

## 适用范围

本规范适用于所有使用 MySQL 数据库的场景。

## 命名规范

### 数据库命名

```sql
-- 格式: {项目}_{模块}
-- 小写，下划线分隔

-- ✅ 正确
workbench_core
workbench_metadata
workbench_audit

-- ❌ 错误
WorkbenchCore     -- 不要驼峰
workbench-core    -- 不要横线
wb_core           -- 不要缩写
```

### 表命名

```sql
-- 格式: {模块}_{实体}
-- 小写，下划线分隔，使用单数形式

-- ✅ 正确
user                  -- 用户表
user_role             -- 用户角色关联表
task_execution_log    -- 任务执行日志表
data_source_config    -- 数据源配置表

-- ❌ 错误
Users                 -- 不要复数
tbl_user              -- 不要前缀
userRole              -- 不要驼峰
```

### 字段命名

```sql
-- 格式: {描述}_{类型后缀}
-- 小写，下划线分隔

-- ✅ 正确
id                    -- 主键
user_id               -- 外键
name                  -- 名称
email                 -- 邮箱
created_at            -- 创建时间 (_at 后缀)
updated_at            -- 更新时间
is_deleted            -- 布尔值 (is_ 前缀)
is_active
status_code           -- 状态码 (_code 后缀)
retry_count           -- 计数 (_count 后缀)
description           -- 描述

-- ❌ 错误
ID                    -- 不要大写
userId                -- 不要驼峰
create_time           -- 应为 created_at
isDeleted             -- 应为 is_deleted
```

### 索引命名

```sql
-- 格式: idx_{表名}_{字段名}
-- 唯一索引: uk_{表名}_{字段名}

-- ✅ 正确
idx_user_email
idx_task_status_created_at
uk_user_email

-- ❌ 错误
index_1
user_email_index
```

## 字段设计

### 必备字段

每个业务表必须包含以下字段：

```sql
CREATE TABLE example (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    -- 业务字段...
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='示例表';
```

### 字段类型选择

| 场景 | 推荐类型 | 说明 |
|------|---------|------|
| 主键 | BIGINT UNSIGNED | 自增主键 |
| 外键 | BIGINT UNSIGNED | 与主键类型一致 |
| 短文本 (<255) | VARCHAR(n) | 指定合适长度 |
| 长文本 | TEXT | 不需要索引的长文本 |
| 布尔值 | TINYINT(1) | 0=false, 1=true |
| 金额 | DECIMAL(18,2) | 精确数值 |
| 时间戳 | DATETIME | 不使用 TIMESTAMP |
| 枚举 | VARCHAR(32) | 不使用 ENUM 类型 |
| JSON | JSON | MySQL 5.7+ |

### 字段约束

```sql
-- ✅ 正确: 明确 NOT NULL 和默认值
CREATE TABLE user (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL COMMENT '用户名',
    email VARCHAR(255) NOT NULL COMMENT '邮箱',
    status VARCHAR(32) NOT NULL DEFAULT 'active' COMMENT '状态',
    login_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '登录次数',
    last_login_at DATETIME NULL COMMENT '最后登录时间',
    PRIMARY KEY (id)
);

-- ❌ 错误: 缺少约束
CREATE TABLE user (
    id INT,                  -- 应该 BIGINT UNSIGNED NOT NULL AUTO_INCREMENT
    name VARCHAR(100),       -- 应该 NOT NULL
    status VARCHAR(32),      -- 应该有默认值
);
```

### 表注释规范

```sql
-- 表和字段必须有 COMMENT
CREATE TABLE task_execution_log (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
    task_id BIGINT UNSIGNED NOT NULL COMMENT '任务ID',
    status VARCHAR(32) NOT NULL DEFAULT 'pending' COMMENT '状态: pending/running/success/failed',
    error_message TEXT NULL COMMENT '错误信息',
    started_at DATETIME NULL COMMENT '开始时间',
    finished_at DATETIME NULL COMMENT '结束时间',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    INDEX idx_task_status (task_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='任务执行日志表';
```

**注释要求：**
- 表注释说明表用途
- 枚举类字段注释列出所有可选值
- 外键字段注释说明关联表

### 分区表设计

适用于大数据量表（>1000 万行），常见场景：日志表、历史数据表。

```sql
-- 按时间范围分区（推荐）
CREATE TABLE operation_log (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    action VARCHAR(64) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id, created_at),  -- 分区键必须包含在主键中
    INDEX idx_user_action (user_id, action)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p202501 VALUES LESS THAN (TO_DAYS('2025-02-01')),
    PARTITION p202502 VALUES LESS THAN (TO_DAYS('2025-03-01')),
    PARTITION p202503 VALUES LESS THAN (TO_DAYS('2025-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 添加新分区
ALTER TABLE operation_log ADD PARTITION (
    PARTITION p202504 VALUES LESS THAN (TO_DAYS('2025-05-01'))
);

-- 删除旧分区（快速清理历史数据）
ALTER TABLE operation_log DROP PARTITION p202501;
```

**分区注意事项：**
- 分区键必须包含在主键/唯一索引中
- 查询条件应包含分区键，否则会扫描所有分区
- 定期维护分区（添加新分区、清理旧分区）

## SQL 编写规范

### 基本格式

```sql
-- ✅ 正确: 关键字大写，字段小写，适当换行
SELECT
    u.id,
    u.name,
    u.email,
    r.role_name
FROM user u
INNER JOIN user_role ur ON u.id = ur.user_id
INNER JOIN role r ON ur.role_id = r.id
WHERE u.is_deleted = 0
    AND u.status = 'active'
ORDER BY u.created_at DESC
LIMIT 100;

-- ❌ 错误: 全部小写，无格式
select * from user u inner join user_role ur on u.id=ur.user_id where u.is_deleted=0 and u.status='active'
```

### 禁止 SELECT *

```sql
-- ✅ 正确: 明确字段列表
SELECT id, name, email, status
FROM user
WHERE id = 123;

-- ❌ 错误: 使用 SELECT *
SELECT * FROM user WHERE id = 123;
```

### 参数化查询

```python
# ✅ 正确: 使用参数化查询
await db.execute(
    "SELECT id, name FROM user WHERE email = :email AND status = :status",
    {"email": email, "status": "active"},
)

# ✅ 正确: 使用 ORM
user = await User.filter(email=email, status="active").first()

# ❌ 错误: 字符串拼接 (SQL 注入风险)
await db.execute(f"SELECT * FROM user WHERE email = '{email}'")
```

### 批量操作

```python
# ✅ 正确: 批量插入
await db.execute_many(
    "INSERT INTO user (name, email) VALUES (:name, :email)",
    [{"name": "a", "email": "a@test.com"}, {"name": "b", "email": "b@test.com"}],
)

# ❌ 错误: 循环单条插入
for user in users:
    await db.execute("INSERT INTO user (name) VALUES (:name)", {"name": user.name})
```

### JOIN 规范

```sql
-- ✅ 推荐：明确 JOIN 类型，使用表别名
SELECT
    u.id,
    u.name,
    o.order_no,
    o.amount
FROM user u
INNER JOIN order o ON u.id = o.user_id
WHERE u.status = 'active'
    AND o.created_at >= '2025-01-01';

-- ❌ 避免：隐式 JOIN（逗号连接）
SELECT u.id, o.order_no
FROM user u, order o
WHERE u.id = o.user_id;
```

**JOIN 原则：**

| 原则 | 说明 |
|------|------|
| 明确类型 | 使用 INNER/LEFT/RIGHT JOIN，不用隐式连接 |
| 小表驱动 | 小结果集的表放在前面 |
| 避免多表 | 单次查询不超过 3 表 JOIN |
| ON vs WHERE | 连接条件用 ON，过滤条件用 WHERE |

```sql
-- LEFT JOIN 时注意：ON 和 WHERE 的区别
-- ON 条件不满足时保留左表数据，右表为 NULL
-- WHERE 条件会过滤掉不满足的行

-- ✅ 正确：查询所有用户及其订单（无订单的用户也显示）
SELECT u.name, o.order_no
FROM user u
LEFT JOIN order o ON u.id = o.user_id AND o.status = 'paid';

-- ❌ 错误：这样会过滤掉无订单的用户
SELECT u.name, o.order_no
FROM user u
LEFT JOIN order o ON u.id = o.user_id
WHERE o.status = 'paid';
```

### 子查询 vs JOIN

```sql
-- 场景：查询有订单的用户

-- ✅ 推荐：使用 EXISTS（通常性能更好）
SELECT u.id, u.name
FROM user u
WHERE EXISTS (
    SELECT 1 FROM order o WHERE o.user_id = u.id
);

-- ✅ 可用：使用 JOIN + DISTINCT
SELECT DISTINCT u.id, u.name
FROM user u
INNER JOIN order o ON u.id = o.user_id;

-- ⚠️ 慎用：IN 子查询（数据量大时性能差）
SELECT id, name
FROM user
WHERE id IN (SELECT user_id FROM order);
```

**选择原则：**

| 场景 | 推荐方式 |
|------|---------|
| 判断存在性 | EXISTS |
| 需要关联表字段 | JOIN |
| IN 列表小且固定 | IN (值列表) |
| IN 子查询结果大 | 改用 EXISTS 或 JOIN |

## 索引规范

### 索引原则

1. 主键自动创建聚簇索引
2. 外键字段必须创建索引
3. WHERE/ORDER BY/GROUP BY 常用字段创建索引
4. 联合索引遵循最左前缀原则
5. 单表索引不超过 5 个

### 索引类型选择

| 类型 | 适用场景 | 示例 |
|------|---------|------|
| B+Tree（默认） | 范围查询、排序、精确匹配 | 大多数场景 |
| 唯一索引 | 业务唯一约束 | email、手机号 |
| 前缀索引 | 长字符串字段 | VARCHAR(500) 取前 20 字符 |
| 覆盖索引 | 避免回表查询 | 查询字段都在索引中 |

```sql
-- 前缀索引（长文本）
CREATE INDEX idx_user_email ON user(email(20));

-- 覆盖索引设计
CREATE INDEX idx_user_status_name ON user(status, name);
-- 此查询无需回表：
SELECT name FROM user WHERE status = 'active';
```

### 联合索引设计

```sql
-- 遵循最左前缀原则
CREATE INDEX idx_order_user_status_time ON order(user_id, status, created_at);

-- ✅ 能使用索引
SELECT * FROM order WHERE user_id = 1;
SELECT * FROM order WHERE user_id = 1 AND status = 'paid';
SELECT * FROM order WHERE user_id = 1 AND status = 'paid' AND created_at > '2025-01-01';

-- ❌ 无法使用索引（跳过了 user_id）
SELECT * FROM order WHERE status = 'paid';
SELECT * FROM order WHERE created_at > '2025-01-01';
```

**联合索引字段顺序原则：**
1. 等值查询字段在前
2. 范围查询字段在后
3. 选择性高的字段在前

### 索引失效场景

```sql
-- ❌ 对索引列使用函数
SELECT * FROM user WHERE DATE(created_at) = '2025-01-01';
-- ✅ 改为范围查询
SELECT * FROM user WHERE created_at >= '2025-01-01' AND created_at < '2025-01-02';

-- ❌ 隐式类型转换
SELECT * FROM user WHERE phone = 13800138000;  -- phone 是 VARCHAR
-- ✅ 使用正确类型
SELECT * FROM user WHERE phone = '13800138000';

-- ❌ LIKE 前缀通配符
SELECT * FROM user WHERE name LIKE '%张三';
-- ✅ 后缀通配符可用索引
SELECT * FROM user WHERE name LIKE '张三%';

-- ❌ OR 连接非索引字段
SELECT * FROM user WHERE email = 'a@test.com' OR age = 20;  -- age 无索引
-- ✅ 改用 UNION
SELECT * FROM user WHERE email = 'a@test.com'
UNION
SELECT * FROM user WHERE age = 20;

-- ❌ NOT IN / NOT EXISTS / !=
SELECT * FROM user WHERE status != 'deleted';
-- ✅ 使用 IN 列出需要的值
SELECT * FROM user WHERE status IN ('active', 'pending');

-- ❌ IS NULL（部分场景）
-- 建议业务上避免 NULL，使用默认值代替
```

### 索引维护

```sql
-- 查看索引使用情况
SELECT
    table_name,
    index_name,
    stat_value AS pages,
    stat_description
FROM mysql.innodb_index_stats
WHERE database_name = 'your_db' AND table_name = 'your_table';

-- 分析表（更新统计信息）
ANALYZE TABLE user;

-- 重建索引（碎片整理）
ALTER TABLE user ENGINE=InnoDB;

-- 或使用 OPTIMIZE
OPTIMIZE TABLE user;
```

**维护建议：**
- 定期执行 ANALYZE TABLE（建议每周）
- 大量删除后考虑重建索引
- 监控索引大小和碎片率

## 事务规范

### 事务边界

```python
# ✅ 正确: 明确事务边界，保持事务短小
async with db.transaction():
    user = await create_user(data)
    await create_user_role(user.id, role_id)

# ❌ 错误: 事务中包含外部调用
async with db.transaction():
    user = await create_user(data)
    await send_email(user.email)  # 禁止！外部调用应在事务外
    await create_user_role(user.id, role_id)
```

### 事务隔离级别

```sql
-- 默认使用 REPEATABLE READ
-- 特殊场景可调整为 READ COMMITTED

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 避免长事务

```python
# ❌ 错误: 长事务
async with db.transaction():
    users = await get_all_users()  # 可能返回大量数据
    for user in users:
        await process_user(user)  # 长时间处理

# ✅ 正确: 分批处理
users = await get_all_users()
for batch in chunk(users, 100):
    async with db.transaction():
        for user in batch:
            await process_user(user)
```

### 死锁处理

**死锁产生条件：**
- 多个事务互相等待对方持有的锁
- 常见于并发更新同一批数据

**预防措施：**

```python
# ✅ 按固定顺序访问资源
async def transfer(from_id: int, to_id: int, amount: Decimal):
    # 按 ID 顺序加锁，避免死锁
    ids = sorted([from_id, to_id])
    async with db.transaction():
        for id in ids:
            await db.execute("SELECT * FROM account WHERE id = :id FOR UPDATE", {"id": id})
        # 执行转账逻辑

# ❌ 错误：不固定的加锁顺序
async def transfer_wrong(from_id: int, to_id: int, amount: Decimal):
    async with db.transaction():
        await db.execute("SELECT * FROM account WHERE id = :id FOR UPDATE", {"id": from_id})
        await db.execute("SELECT * FROM account WHERE id = :id FOR UPDATE", {"id": to_id})
```

**死锁处理：**

```python
from mysql.connector.errors import DatabaseError

MAX_RETRIES = 3

async def execute_with_retry(func):
    for attempt in range(MAX_RETRIES):
        try:
            return await func()
        except DatabaseError as e:
            if "Deadlock" in str(e) and attempt < MAX_RETRIES - 1:
                await asyncio.sleep(0.1 * (attempt + 1))  # 退避重试
                continue
            raise
```

### 乐观锁 vs 悲观锁

| 类型 | 实现方式 | 适用场景 |
|------|---------|---------|
| 悲观锁 | SELECT ... FOR UPDATE | 冲突频繁、数据一致性要求高 |
| 乐观锁 | 版本号/时间戳 | 冲突少、读多写少 |

**悲观锁：**

```sql
-- 加行锁
SELECT * FROM inventory WHERE product_id = 123 FOR UPDATE;

-- 执行更新
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123;

COMMIT;
```

**乐观锁：**

```sql
-- 表结构增加版本号
ALTER TABLE inventory ADD COLUMN version INT NOT NULL DEFAULT 0;

-- 更新时检查版本号
UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE product_id = 123 AND version = 5;

-- 检查影响行数，为 0 表示冲突，需重试
```

```python
# Python 实现
async def update_with_optimistic_lock(product_id: int, delta: int) -> bool:
    result = await db.execute(
        """
        UPDATE inventory
        SET quantity = quantity + :delta, version = version + 1
        WHERE product_id = :product_id AND version = :version
        """,
        {"product_id": product_id, "delta": delta, "version": current_version}
    )
    return result.rowcount > 0  # False 表示需要重试
```

## 连接管理

### 连接池配置

```python
# 推荐配置
DATABASE_CONFIG = {
    "pool_size": 10,           # 连接池大小
    "max_overflow": 20,        # 最大溢出连接
    "pool_timeout": 30,        # 获取连接超时 (秒)
    "pool_recycle": 3600,      # 连接回收时间 (秒)
    "pool_pre_ping": True,     # 使用前检测连接有效性
}
```

**配置说明：**

| 参数 | 建议值 | 说明 |
|------|-------|------|
| pool_size | CPU 核数 * 2 | 常驻连接数 |
| max_overflow | pool_size * 2 | 峰值额外连接 |
| pool_timeout | 30 | 等待连接超时 |
| pool_recycle | 3600 | 避免 MySQL 8 小时断开 |
| pool_pre_ping | True | 防止使用失效连接 |

### ORM 选型

| ORM | 特点 | 推荐场景 |
|-----|------|---------|
| SQLAlchemy | 功能全面、生态成熟 | 复杂业务、需要灵活控制 |
| Tortoise-ORM | 异步原生、Django 风格 | 异步项目、FastAPI |
| Peewee | 轻量简洁 | 小型项目、快速开发 |

**本项目推荐：SQLAlchemy 2.0 + asyncio**

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "mysql+aiomysql://user:pass@localhost/db",
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
    pool_pre_ping=True,
)

AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

# 使用
async with AsyncSessionLocal() as session:
    result = await session.execute(select(User).where(User.id == 1))
    user = result.scalar_one_or_none()
```

### 连接监控

```sql
-- 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';

-- 查看最大连接数
SHOW VARIABLES LIKE 'max_connections';

-- 查看连接详情
SHOW PROCESSLIST;

-- 查看等待锁的连接
SELECT * FROM information_schema.INNODB_TRX;
```

**监控指标：**

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| Threads_connected | > max_connections * 80% | 连接数过高 |
| Threads_running | > CPU 核数 * 2 | 活跃线程过多 |
| Slow_queries | 持续增长 | 慢查询增多 |

### 故障处理与重连

```python
from sqlalchemy.exc import OperationalError, DisconnectionError
import asyncio

class DatabaseManager:
    def __init__(self):
        self.engine = None

    async def get_connection(self, max_retries: int = 3):
        """获取连接，支持重试。"""
        for attempt in range(max_retries):
            try:
                async with self.engine.connect() as conn:
                    yield conn
                return
            except (OperationalError, DisconnectionError) as e:
                if attempt < max_retries - 1:
                    await asyncio.sleep(1 * (attempt + 1))
                    continue
                raise

    async def health_check(self) -> bool:
        """健康检查。"""
        try:
            async with self.engine.connect() as conn:
                await conn.execute(text("SELECT 1"))
            return True
        except Exception:
            return False
```

**常见故障处理：**

| 错误 | 原因 | 处理方式 |
|------|------|---------|
| Connection refused | MySQL 未启动/网络问题 | 检查服务状态、网络连通性 |
| Too many connections | 连接数超限 | 增大 max_connections 或优化连接池 |
| Lock wait timeout | 锁等待超时 | 检查长事务、优化锁粒度 |
| MySQL server has gone away | 连接被 MySQL 关闭 | 使用 pool_recycle、pool_pre_ping |

## 查询优化

### 分页查询

```sql
-- ✅ 正确: 使用游标分页 (大数据量)
SELECT id, name
FROM user
WHERE id > :last_id
ORDER BY id
LIMIT 100;

-- 可接受: LIMIT OFFSET (小数据量)
SELECT id, name
FROM user
ORDER BY id
LIMIT 100 OFFSET 0;

-- ❌ 错误: 大 OFFSET
SELECT id, name
FROM user
ORDER BY id
LIMIT 100 OFFSET 100000;  -- 性能差
```

### 避免的写法

```sql
-- ❌ 函数作用于索引字段
SELECT * FROM user WHERE DATE(created_at) = '2024-01-01';
-- ✅ 改为
SELECT * FROM user WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02';

-- ❌ 隐式类型转换
SELECT * FROM user WHERE phone = 13800138000;  -- phone 是 VARCHAR
-- ✅ 改为
SELECT * FROM user WHERE phone = '13800138000';

-- ❌ LIKE 前缀通配
SELECT * FROM user WHERE name LIKE '%张';
-- ✅ 如需全文搜索，使用 Elasticsearch
```

### EXPLAIN 执行计划解读

```sql
EXPLAIN SELECT u.id, u.name FROM user u WHERE u.email = 'test@example.com';
```

**关键字段解读：**

| 字段 | 说明 | 优化目标 |
|------|------|---------|
| type | 访问类型 | const > eq_ref > ref > range > index > ALL |
| key | 实际使用的索引 | 确保使用预期索引 |
| rows | 预估扫描行数 | 越小越好 |
| Extra | 额外信息 | 避免 Using filesort、Using temporary |

**type 类型说明：**

| type | 含义 | 性能 |
|------|------|------|
| const | 主键/唯一索引等值查询 | 最优 |
| eq_ref | JOIN 时主键/唯一索引 | 优秀 |
| ref | 非唯一索引等值查询 | 良好 |
| range | 索引范围扫描 | 可接受 |
| index | 全索引扫描 | 较差 |
| ALL | 全表扫描 | 最差，需优化 |

**Extra 常见值：**

| Extra | 含义 | 是否需优化 |
|-------|------|-----------|
| Using index | 覆盖索引 | ✅ 好 |
| Using where | 使用 WHERE 过滤 | 正常 |
| Using filesort | 额外排序 | ⚠️ 考虑优化 |
| Using temporary | 使用临时表 | ⚠️ 考虑优化 |
| Using join buffer | JOIN 缓冲 | ⚠️ 检查索引 |

### 慢查询分析

**开启慢查询日志：**

```sql
-- 查看当前配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询（动态）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过 1 秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
```

**分析慢查询：**

```bash
# 使用 mysqldumpslow 分析
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 参数说明:
# -s t: 按查询时间排序
# -t 10: 显示前 10 条
```

**慢查询优化步骤：**

1. 使用 EXPLAIN 分析执行计划
2. 检查是否使用索引
3. 检查 type 是否为 ALL/index
4. 检查 rows 是否过大
5. 根据分析结果添加/优化索引

## 数据安全

### SQL 注入防护

```python
# ✅ 参数化查询（已在 SQL 编写规范中说明）
await db.execute(
    "SELECT * FROM user WHERE email = :email",
    {"email": user_input}
)

# ❌ 永远不要拼接用户输入
await db.execute(f"SELECT * FROM user WHERE email = '{user_input}'")
```

### 敏感数据处理

| 数据类型 | 存储方式 | 说明 |
|---------|---------|------|
| 密码 | bcrypt/argon2 哈希 | 禁止明文、禁止 MD5/SHA1 |
| 手机号/身份证 | 加密存储或脱敏 | 根据业务需求选择 |
| API Key | 加密存储 | 使用 AES-256 |

```python
# 密码存储示例
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

hashed = pwd_context.hash(password)  # 存储
verified = pwd_context.verify(password, hashed)  # 验证
```

### 权限管理

```sql
-- 创建只读用户（应用查询）
CREATE USER 'app_readonly'@'%' IDENTIFIED BY 'password';
GRANT SELECT ON database.* TO 'app_readonly'@'%';

-- 创建读写用户（应用操作）
CREATE USER 'app_readwrite'@'%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE, DELETE ON database.* TO 'app_readwrite'@'%';

-- 禁止 DROP/ALTER 等危险权限给应用账号
```

**权限原则：**
- 应用账号只授予必要权限
- 生产环境禁止使用 root 账号
- 定期审计账号权限

### 备份策略

| 备份类型 | 频率 | 保留时间 | 工具 |
|---------|------|---------|------|
| 全量备份 | 每日 | 7 天 | mysqldump / xtrabackup |
| 增量备份 | 每小时 | 24 小时 | binlog |
| 跨区备份 | 每日 | 30 天 | 异地存储 |

```bash
# mysqldump 全量备份
mysqldump -u root -p --single-transaction --routines --triggers \
    database_name > backup_$(date +%Y%m%d).sql

# 恢复
mysql -u root -p database_name < backup_20250101.sql
```

## 运维相关

### 慢日志配置

```ini
# my.cnf 配置
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON
```

### 监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| QPS | 每秒查询数 | 根据基线设置 |
| TPS | 每秒事务数 | 根据基线设置 |
| Threads_connected | 当前连接数 | > 80% max_connections |
| Threads_running | 活跃线程数 | > CPU 核数 * 2 |
| Slow_queries | 慢查询累计 | 持续增长 |
| Innodb_buffer_pool_hit_rate | 缓冲池命中率 | < 95% |

```sql
-- 查看关键指标
SHOW GLOBAL STATUS LIKE 'Questions';  -- 累计查询数
SHOW GLOBAL STATUS LIKE 'Com_select';  -- SELECT 次数
SHOW GLOBAL STATUS LIKE 'Com_insert';  -- INSERT 次数
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';  -- 缓冲池请求
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';  -- 磁盘读取
```

### 容量规划

**表大小查询：**

```sql
SELECT
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY data_length DESC;
```

**容量规划建议：**

| 表行数 | 建议措施 |
|--------|---------|
| < 100 万 | 正常使用 |
| 100 万 - 1000 万 | 考虑添加索引优化 |
| 1000 万 - 1 亿 | 考虑分区表 |
| > 1 亿 | 考虑分库分表 |

## 禁止事项

- ❌ 禁止使用 `SELECT *`
- ❌ 禁止在代码中拼接 SQL
- ❌ 禁止在事务中进行外部调用 (HTTP/消息队列等)
- ❌ 禁止使用 ENUM 类型
- ❌ 禁止无 WHERE 条件的 UPDATE/DELETE
- ❌ 禁止在索引字段上使用函数
- ❌ 禁止大事务 (超过 1000 行修改)

## 检查清单

### 建表
- [ ] 表/字段命名符合规范
- [ ] 必备字段齐全 (id, created_at, updated_at)
- [ ] 字段类型选择正确
- [ ] 表和字段都有 COMMENT

### 索引
- [ ] 外键字段有索引
- [ ] 常用查询字段有索引支持
- [ ] 联合索引遵循最左前缀原则
- [ ] 使用 EXPLAIN 验证索引生效

### SQL
- [ ] 使用参数化查询
- [ ] 无 SELECT * 使用
- [ ] JOIN 不超过 3 表
- [ ] 大数据量使用游标分页

### 事务
- [ ] 事务边界清晰
- [ ] 无长事务
- [ ] 事务中无外部调用

### 安全
- [ ] 敏感数据加密存储
- [ ] 应用账号权限最小化
- [ ] 有备份策略
