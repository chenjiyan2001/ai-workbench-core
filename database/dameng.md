# 达梦数据库使用规范

## 适用范围

本规范适用于使用达梦数据库 (DM8) 的场景，重点关注与 MySQL/Oracle 的兼容性差异。

## 概述

达梦数据库是国产关系型数据库，语法接近 Oracle，但也兼容部分 MySQL 语法。

## 连接配置

### JDBC 连接 (推荐)

```python
# 使用 JayDeBeApi (Python JDBC)
import jaydebeapi

conn = jaydebeapi.connect(
    "dm.jdbc.driver.DmDriver",
    "jdbc:dm://localhost:5236/WORKBENCH",
    ["SYSDBA", "password"],
    "/path/to/DmJdbcDriver18.jar",
)
```

### ODBC 连接

```python
import pyodbc

conn = pyodbc.connect(
    "DRIVER={DM8 ODBC DRIVER};"
    "SERVER=localhost;"
    "PORT=5236;"
    "DATABASE=WORKBENCH;"
    "UID=SYSDBA;"
    "PWD=password"
)
```

### SQLAlchemy 连接

```python
from sqlalchemy import create_engine

# 使用 dmPython 驱动
engine = create_engine(
    "dm+dmPython://SYSDBA:password@localhost:5236/WORKBENCH",
    pool_size=10,
    max_overflow=20,
)
```

## 命名规范

### 与 MySQL 的差异

```sql
-- 达梦默认区分大小写，建议统一使用大写

-- ✅ 正确 (大写)
CREATE TABLE USER_INFO (
    ID BIGINT PRIMARY KEY,
    USER_NAME VARCHAR(100),
    CREATED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 或者使用双引号保持小写 (不推荐)
CREATE TABLE "user_info" (
    "id" BIGINT PRIMARY KEY
);
```

### 命名建议

- 表名/字段名: 大写，下划线分隔
- 与 MySQL 规范保持一致，但使用大写

## 数据类型映射

### MySQL → 达梦

| MySQL | 达梦 | 说明 |
|-------|------|------|
| BIGINT | BIGINT | 一致 |
| INT | INT | 一致 |
| VARCHAR(n) | VARCHAR(n) | 一致 |
| TEXT | TEXT / CLOB | TEXT 有长度限制，长文本用 CLOB |
| DATETIME | TIMESTAMP | 推荐使用 TIMESTAMP |
| TINYINT(1) | BIT / INT | 布尔值 |
| JSON | CLOB | 达梦无原生 JSON，用 CLOB 存储 |
| AUTO_INCREMENT | IDENTITY | 自增语法不同 |

### 自增主键

```sql
-- MySQL
CREATE TABLE user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY
);

-- 达梦
CREATE TABLE USER (
    ID BIGINT IDENTITY(1, 1) PRIMARY KEY
);

-- 或使用序列
CREATE SEQUENCE SEQ_USER_ID START WITH 1 INCREMENT BY 1;
CREATE TABLE USER (
    ID BIGINT PRIMARY KEY
);
-- 插入时: INSERT INTO USER (ID, ...) VALUES (SEQ_USER_ID.NEXTVAL, ...)
```

## SQL 兼容性

### 分页查询

```sql
-- MySQL
SELECT * FROM user LIMIT 10 OFFSET 20;

-- 达梦 (支持 LIMIT，但推荐使用标准写法)
SELECT * FROM USER LIMIT 10 OFFSET 20;

-- 或使用 ROWNUM (Oracle 风格)
SELECT * FROM (
    SELECT U.*, ROWNUM RN FROM USER U WHERE ROWNUM <= 30
) WHERE RN > 20;

-- 达梦 8 也支持 FETCH (SQL 标准)
SELECT * FROM USER
ORDER BY ID
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### 字符串函数

```sql
-- MySQL: CONCAT 可多参数
SELECT CONCAT('a', 'b', 'c');

-- 达梦: CONCAT 只支持两参数，多个需嵌套
SELECT CONCAT(CONCAT('a', 'b'), 'c');

-- 或使用 || 运算符 (推荐)
SELECT 'a' || 'b' || 'c';
```

### 日期函数

```sql
-- MySQL
SELECT NOW();
SELECT DATE_FORMAT(created_at, '%Y-%m-%d');

-- 达梦
SELECT SYSDATE;  -- 或 CURRENT_TIMESTAMP
SELECT TO_CHAR(CREATED_AT, 'YYYY-MM-DD');
```

### UPSERT 操作

```sql
-- MySQL: INSERT ... ON DUPLICATE KEY UPDATE
INSERT INTO user (id, name, email)
VALUES (1, '张三', 'test@example.com')
ON DUPLICATE KEY UPDATE name = VALUES(name);

-- 达梦: MERGE INTO
MERGE INTO USER T
USING (SELECT 1 AS ID, '张三' AS NAME FROM DUAL) S
ON (T.ID = S.ID)
WHEN MATCHED THEN
    UPDATE SET T.NAME = S.NAME
WHEN NOT MATCHED THEN
    INSERT (ID, NAME) VALUES (S.ID, S.NAME);
```

### 批量插入

```sql
-- MySQL
INSERT INTO user (name, email) VALUES
('张三', 'a@test.com'),
('李四', 'b@test.com');

-- 达梦 (同样支持)
INSERT INTO USER (NAME, EMAIL) VALUES
('张三', 'a@test.com'),
('李四', 'b@test.com');

-- 或使用 INSERT ALL
INSERT ALL
    INTO USER (NAME, EMAIL) VALUES ('张三', 'a@test.com')
    INTO USER (NAME, EMAIL) VALUES ('李四', 'b@test.com')
SELECT 1 FROM DUAL;
```

## Python 代码适配

### 统一数据库接口

```python
from abc import ABC, abstractmethod
from typing import Any

class DatabaseConnector(ABC):
    """数据库连接器抽象基类"""

    @abstractmethod
    async def execute(self, query: str, params: dict[str, Any]) -> None:
        pass

    @abstractmethod
    async def fetch_all(self, query: str, params: dict[str, Any]) -> list[dict]:
        pass


class DamengConnector(DatabaseConnector):
    """达梦数据库连接器"""

    def __init__(self, connection_string: str) -> None:
        self.conn = self._create_connection(connection_string)

    async def execute(self, query: str, params: dict[str, Any]) -> None:
        # 转换参数占位符: :name → ?
        query = self._convert_placeholders(query)
        cursor = self.conn.cursor()
        cursor.execute(query, list(params.values()))
        self.conn.commit()

    def _convert_placeholders(self, query: str) -> str:
        """转换参数占位符格式"""
        # 达梦 JDBC 使用 ? 占位符
        import re
        return re.sub(r':(\w+)', '?', query)
```

### SQL 方言适配

```python
class SQLDialect:
    """SQL 方言适配器"""

    @staticmethod
    def get_dialect(db_type: str) -> "SQLDialect":
        dialects = {
            "mysql": MySQLDialect(),
            "dameng": DamengDialect(),
        }
        return dialects.get(db_type, MySQLDialect())

    def paginate(self, query: str, limit: int, offset: int) -> str:
        raise NotImplementedError


class MySQLDialect(SQLDialect):
    def paginate(self, query: str, limit: int, offset: int) -> str:
        return f"{query} LIMIT {limit} OFFSET {offset}"

    def upsert(self, table: str, data: dict, key_fields: list[str]) -> str:
        # MySQL ON DUPLICATE KEY UPDATE
        ...


class DamengDialect(SQLDialect):
    def paginate(self, query: str, limit: int, offset: int) -> str:
        return f"{query} LIMIT {limit} OFFSET {offset}"

    def upsert(self, table: str, data: dict, key_fields: list[str]) -> str:
        # 达梦 MERGE INTO
        ...
```

## 事务处理

```python
# 达梦事务处理与 MySQL 类似
async def transfer_with_transaction(from_id: int, to_id: int, amount: float) -> None:
    conn = get_dameng_connection()
    try:
        conn.autocommit = False

        cursor = conn.cursor()
        cursor.execute(
            "UPDATE ACCOUNT SET BALANCE = BALANCE - ? WHERE ID = ?",
            [amount, from_id],
        )
        cursor.execute(
            "UPDATE ACCOUNT SET BALANCE = BALANCE + ? WHERE ID = ?",
            [amount, to_id],
        )

        conn.commit()
    except Exception as e:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

## 性能优化

### 索引创建

```sql
-- 与 MySQL 基本一致
CREATE INDEX IDX_USER_EMAIL ON USER(EMAIL);
CREATE UNIQUE INDEX UK_USER_EMAIL ON USER(EMAIL);

-- 组合索引
CREATE INDEX IDX_TASK_STATUS_CREATED ON TASK(STATUS, CREATED_AT);
```

### 执行计划

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM USER WHERE EMAIL = 'test@example.com';

-- 或
SET AUTOTRACE ON;
SELECT * FROM USER WHERE EMAIL = 'test@example.com';
SET AUTOTRACE OFF;
```

## 常见问题

### 1. 大小写问题

```sql
-- 问题: 表名/字段名大小写敏感
-- 解决: 统一使用大写，或用双引号

-- 创建时用小写
CREATE TABLE "user" ("id" INT);
-- 查询时必须用双引号
SELECT * FROM "user";
```

### 2. 字符集问题

```sql
-- 创建数据库时指定字符集
CREATE DATABASE WORKBENCH
    CHARSET 'UTF-8'
    COLLATE 'UTF-8';
```

### 3. 驱动兼容问题

```python
# 优先级: JDBC > ODBC > dmPython
# JDBC 驱动兼容性最好

# 如果使用 dmPython，注意版本匹配
pip install dmPython==2.4.5  # 与 DM8 版本对应
```

## 禁止事项

- ❌ 禁止混用大小写命名
- ❌ 禁止使用 MySQL 专有语法而不做适配
- ❌ 禁止忽略驱动版本兼容性
- ❌ 禁止不测试就上线 (与 MySQL 有细微差异)

## 检查清单

- [ ] 统一使用大写命名
- [ ] 验证 SQL 语法兼容性
- [ ] 测试分页、日期函数等常用操作
- [ ] 确认 JDBC 驱动版本匹配
- [ ] 自增主键使用 IDENTITY 或序列
- [ ] 有 MySQL→达梦 的回归测试
