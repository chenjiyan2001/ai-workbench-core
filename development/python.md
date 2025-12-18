# Python 代码规范

## 适用范围

本规范适用于所有 Python 3.11+ 项目代码。

## 代码风格

### 基础规范

- 遵循 PEP 8
- 使用 4 空格缩进 (禁止 Tab)
- 行宽限制: 88 字符 (Black 默认)
- 文件编码: UTF-8

### 格式化工具

```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ['py311']

[tool.isort]
profile = "black"
line_length = 88

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W", "UP"]
```

## 命名约定

| 类型 | 规则 | 示例 |
|------|------|------|
| 模块 | snake_case | `mysql_connector.py` |
| 包 | snake_case | `data_processing/` |
| 类 | PascalCase | `MySQLConnector` |
| 函数/方法 | snake_case | `get_user_by_id()` |
| 变量 | snake_case | `user_count` |
| 常量 | UPPER_SNAKE | `MAX_RETRY_COUNT` |
| 私有属性 | _prefix | `_internal_cache` |
| 类型变量 | PascalCase | `T`, `UserType` |

### 命名示例

```python
# ✅ 正确
class DatabaseConnector:
    MAX_POOL_SIZE = 10

    def __init__(self, connection_string: str) -> None:
        self._pool: ConnectionPool | None = None
        self.connection_string = connection_string

    def get_connection(self) -> Connection:
        ...

    def _create_pool(self) -> None:
        ...

# ❌ 错误
class database_connector:  # 类名应 PascalCase
    maxPoolSize = 10       # 常量应 UPPER_SNAKE

    def GetConnection(self):  # 方法应 snake_case
        ...
```

## 类型注解

### 必须添加类型注解的位置

1. 函数参数
2. 函数返回值
3. 类属性
4. 复杂的局部变量

### 类型注解示例

```python
from typing import Any
from collections.abc import Iterable, Mapping

# ✅ 正确
def process_records(
    records: list[dict[str, Any]],
    batch_size: int = 100,
    *,
    validate: bool = True,
) -> tuple[int, list[str]]:
    """处理记录并返回 (成功数, 错误列表)"""
    processed: int = 0
    errors: list[str] = []
    ...
    return processed, errors

# 类型别名
type UserId = int
type UserRecord = dict[str, Any]
type QueryResult = list[UserRecord]

# 泛型
from typing import TypeVar, Generic

T = TypeVar("T")

class Repository(Generic[T]):
    def get(self, id: int) -> T | None:
        ...

    def list(self) -> list[T]:
        ...
```

### 类型注解规范

```python
# Python 3.10+ 风格 (推荐)
def func(items: list[str] | None = None) -> dict[str, int]:
    ...

# 避免使用旧风格
from typing import List, Dict, Optional  # ❌ 不推荐
def func(items: Optional[List[str]] = None) -> Dict[str, int]:
    ...
```

## 导入规范

### 导入顺序

```python
# 1. 标准库
import os
import sys
from collections.abc import Mapping
from typing import Any

# 2. 第三方库
import structlog
from fastapi import FastAPI
from pydantic import BaseModel

# 3. 本地模块
from .config import Settings
from .models import User
```

### 导入规则

```python
# ✅ 正确
from datetime import datetime, timedelta
from mypackage.module import ClassName

# ❌ 错误
from datetime import *        # 禁止通配符导入
import datetime as dt         # 避免不必要的别名
from mypackage.module import *
```

## 函数设计

### 函数长度

- 单个函数不超过 50 行
- 超过时应拆分为多个小函数

### 参数设计

```python
# ✅ 正确: 使用 keyword-only 参数
def query_users(
    *,  # 强制后续参数必须使用关键字
    status: str,
    limit: int = 100,
    offset: int = 0,
) -> list[User]:
    ...

# 调用时
users = query_users(status="active", limit=50)

# ❌ 错误: 过多位置参数难以阅读
def query_users(status, limit, offset, include_deleted, sort_by):
    ...
```

### 返回值设计

```python
# ✅ 正确: 使用数据类或 NamedTuple
from dataclasses import dataclass

@dataclass
class QueryResult:
    data: list[User]
    total: int
    has_more: bool

def query_users() -> QueryResult:
    ...

# ❌ 错误: 返回裸元组难以理解
def query_users() -> tuple[list, int, bool]:  # 什么是第二个、第三个？
    ...
```

## 类设计

### 使用 dataclass

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    id: int
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    tags: list[str] = field(default_factory=list)

    def __post_init__(self) -> None:
        self.email = self.email.lower()
```

### 使用 Pydantic (数据校验场景)

```python
from pydantic import BaseModel, Field, field_validator

class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: str
    age: int = Field(..., ge=0, le=150)

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("无效的邮箱格式")
        return v.lower()
```

## 文档字符串

### 格式: Google Style

```python
def connect_database(
    host: str,
    port: int,
    *,
    timeout: float = 30.0,
) -> Connection:
    """建立数据库连接

    Args:
        host: 数据库主机地址
        port: 端口号
        timeout: 连接超时时间 (秒)

    Returns:
        数据库连接对象

    Raises:
        ConnectionError: 连接失败时抛出
        TimeoutError: 连接超时时抛出

    Example:
        >>> conn = connect_database("localhost", 3306)
        >>> conn.ping()
        True
    """
    ...
```

### 何时需要文档字符串

- ✅ 公开的类和函数
- ✅ 复杂的内部函数
- ❌ 简单的私有方法 (如 getter/setter)
- ❌ 测试函数

## 项目结构

```
project/
├── src/
│   └── package_name/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── user_service.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_user_service.py
├── pyproject.toml
└── README.md
```

## 检查清单

- [ ] 代码通过 black 格式化
- [ ] 代码通过 ruff 检查
- [ ] 所有公开函数有类型注解
- [ ] 复杂逻辑有文档字符串
- [ ] 无循环导入
- [ ] 无未使用的导入和变量
