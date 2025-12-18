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

## 索引规范

### 索引原则

1. 主键自动创建聚簇索引
2. 外键字段必须创建索引
3. WHERE/ORDER BY/GROUP BY 常用字段创建索引
4. 联合索引遵循最左前缀原则
5. 单表索引不超过 5 个

### 索引示例

```sql
-- 单字段索引
CREATE INDEX idx_user_email ON user(email);

-- 联合索引 (遵循最左前缀)
CREATE INDEX idx_task_status_created ON task(status, created_at);

-- 唯一索引
CREATE UNIQUE INDEX uk_user_email ON user(email);

-- 覆盖索引 (避免回表)
CREATE INDEX idx_user_status_name ON user(status, name);
-- 查询时可以只用索引: SELECT name FROM user WHERE status = 'active'
```

### 索引使用检查

```sql
-- 使用 EXPLAIN 检查索引使用情况
EXPLAIN SELECT id, name FROM user WHERE email = 'test@example.com';

-- 检查 type 列:
-- const/eq_ref/ref: 好
-- range: 可接受
-- index/ALL: 需要优化
```

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

## 连接池配置

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

## 禁止事项

- ❌ 禁止使用 `SELECT *`
- ❌ 禁止在代码中拼接 SQL
- ❌ 禁止在事务中进行外部调用 (HTTP/消息队列等)
- ❌ 禁止使用 ENUM 类型
- ❌ 禁止无 WHERE 条件的 UPDATE/DELETE
- ❌ 禁止在索引字段上使用函数
- ❌ 禁止大事务 (超过 1000 行修改)

## 检查清单

- [ ] 表/字段命名符合规范
- [ ] 必备字段齐全 (id, created_at, updated_at)
- [ ] 字段类型选择正确
- [ ] 使用参数化查询
- [ ] 关键查询有索引支持
- [ ] 事务边界清晰，无长事务
- [ ] 无 SELECT * 使用
