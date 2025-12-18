# 代码注释规范

本文档定义 Python 代码注释的撰写规范，采用 **Google 风格** docstring。

---

## Docstring 规范

### 采用风格

本项目统一使用 **Google 风格** docstring，理由：
- 可读性最佳，结构清晰
- IDE 支持良好（VS Code、PyCharm）
- 团队学习成本低

### 模块级 Docstring

```python
"""数据处理模块。

本模块提供数据清洗、转换、验证等功能，支持批量处理和流式处理。

Typical usage:
    from data_processor import DataProcessor

    processor = DataProcessor(config)
    result = processor.process(data)
"""
```

### 类 Docstring

```python
class DataProcessor:
    """数据处理器，支持多种数据源的统一处理。

    负责数据的读取、清洗、转换和输出，支持 MySQL、MongoDB 等数据源。

    Attributes:
        config: 处理器配置对象
        batch_size: 批处理大小
        logger: 日志记录器

    Example:
        processor = DataProcessor(config)
        processor.load_data(source)
        result = processor.transform()
    """

    def __init__(self, config: ProcessorConfig) -> None:
        """初始化数据处理器。

        Args:
            config: 处理器配置，包含数据源、批次大小等设置
        """
```

### 函数/方法 Docstring

```python
def process_data(
    data: list[dict],
    batch_size: int = 100,
    validate: bool = True
) -> dict:
    """处理数据并返回统计结果。

    对输入数据进行清洗和转换，支持批量处理以优化内存使用。

    Args:
        data: 待处理的数据列表，每个元素为字典格式
        batch_size: 批处理大小，默认 100。设置过大可能导致内存溢出
        validate: 是否进行数据校验，默认 True

    Returns:
        包含处理统计信息的字典，格式如：
        {
            "total": 1000,
            "success": 950,
            "failed": 50,
            "errors": [...]
        }

    Raises:
        ValueError: 当 data 为空时抛出
        TypeError: 当 data 元素不是字典时抛出
        ProcessingError: 当处理过程中发生不可恢复错误时抛出

    Example:
        >>> result = process_data([{"id": 1}, {"id": 2}])
        >>> print(result["total"])
        2
    """
```

### 简单函数

对于简单、自解释的函数，单行 docstring 即可：

```python
def get_user_count() -> int:
    """返回系统中的用户总数。"""
    return User.objects.count()
```

### 生成器函数

```python
def read_large_file(
    file_path: str,
    chunk_size: int = 1024
) -> Generator[str, None, None]:
    """逐块读取大文件内容。

    Args:
        file_path: 文件路径
        chunk_size: 每次读取的字节数，默认 1024

    Yields:
        str: 文件内容块

    Raises:
        FileNotFoundError: 当文件不存在时抛出
        PermissionError: 当没有读取权限时抛出

    Example:
        >>> for chunk in read_large_file("large.txt"):
        ...     process(chunk)
    """
```

### 异步函数

```python
async def fetch_user_data(
    user_id: int,
    timeout: float = 30.0
) -> dict:
    """异步获取用户数据。

    从远程 API 获取用户信息，支持超时控制。

    Args:
        user_id: 用户 ID
        timeout: 请求超时时间（秒），默认 30.0

    Returns:
        用户数据字典，包含 id、name、email 等字段

    Raises:
        asyncio.TimeoutError: 请求超时
        aiohttp.ClientError: 网络请求失败

    Note:
        此函数需要在异步上下文中调用。

    Example:
        >>> async def main():
        ...     user = await fetch_user_data(123)
        ...     print(user["name"])
    """
```

### 上下文管理器

```python
@contextmanager
def database_transaction(
    connection: Connection,
    isolation_level: str = "READ COMMITTED"
) -> Generator[Cursor, None, None]:
    """数据库事务上下文管理器。

    自动处理事务的提交和回滚。

    Args:
        connection: 数据库连接对象
        isolation_level: 事务隔离级别，默认 "READ COMMITTED"

    Yields:
        Cursor: 数据库游标对象

    Raises:
        DatabaseError: 数据库操作失败时抛出（会自动回滚）

    Warning:
        不要在事务内执行长时间操作，可能导致锁等待超时。

    Example:
        >>> with database_transaction(conn) as cursor:
        ...     cursor.execute("UPDATE users SET status = 'active'")
    """
```

### Note/Warning/See Also 章节

用于补充重要信息：

```python
def calculate_risk_score(data: dict) -> float:
    """计算风险评分。

    Args:
        data: 包含用户行为数据的字典

    Returns:
        风险评分，范围 0.0-1.0

    Note:
        评分算法基于 XGBoost 模型，模型版本 v2.1。
        模型每月更新一次。

    Warning:
        输入数据必须经过预处理，原始数据可能导致评分偏差。
        生产环境请使用 preprocess_data() 进行预处理。

    See Also:
        preprocess_data: 数据预处理函数
        RiskModel: 风险模型类
    """
```

---

## 何时需要 Docstring

### 必须有 Docstring

| 场景 | 说明 |
|------|------|
| 公开 API | 所有 public 函数、类、方法 |
| 模块文件 | 每个 .py 文件顶部 |
| 复杂逻辑 | 算法、业务规则、数据转换 |
| 异常处理 | 会抛出异常的函数 |

### 可以省略 Docstring

| 场景 | 说明 |
|------|------|
| 私有方法 | `_internal_method()` 且逻辑简单 |
| 魔术方法 | `__str__`、`__repr__` 等标准实现 |
| 简单属性 | `@property` 返回简单值 |
| 测试函数 | 函数名已足够说明测试内容 |

---

## 行内注释规范

### 原则

```
注释解释"为什么"，而不是"做什么"
代码本身应该说明"做什么"
```

### 正确示例

```python
# 使用二分查找优化性能，数据量超过 10000 时效果明显
index = binary_search(sorted_data, target)

# 临时方案：MySQL 8.0 以下不支持窗口函数，需要子查询
# TODO(zhangsan): 升级数据库后改用 ROW_NUMBER()
result = execute_legacy_query(sql)

# 保留原始数据用于审计，不要删除
original_data = data.copy()
```

### 错误示例

```python
# ❌ 错误：注释说明显而易见的事情
i = i + 1  # i 加 1

# ❌ 错误：注释与代码不一致（更新代码后忘记更新注释）
# 获取用户列表
orders = get_order_list()  # 实际获取的是订单

# ❌ 错误：注释掉的代码（应该删除，Git 会保留历史）
# user = get_user(id)
# if user:
#     process(user)
```

---

## TODO/FIXME 标记

### 格式规范

```python
# TODO(author): 描述待办事项
# FIXME(author): 描述需要修复的问题
# HACK(author): 临时方案，需要后续优化
# NOTE: 重要说明，提醒阅读者注意
# XXX: 存在问题但暂时可用
```

### 使用示例

```python
# TODO(zhangsan): 添加缓存支持，预计提升 50% 性能
def get_user_stats(user_id: int) -> dict:
    pass

# FIXME(lisi): 并发情况下可能出现竞态条件
# 相关 issue: #123
def update_counter(key: str) -> None:
    pass

# HACK(wangwu): 临时绕过第三方库 bug，等待 v2.0 修复
# 参考: https://github.com/xxx/issues/456
response = requests.get(url, timeout=30)

# NOTE: 此函数会修改传入的 data 参数
def process_in_place(data: list) -> None:
    pass
```

### Issue 关联格式

```python
# TODO(zhangsan): 添加缓存支持 (#123)
# TODO(zhangsan): 重构数据层 (PROJ-456)
# FIXME(lisi): 修复竞态条件 (https://github.com/org/repo/issues/789)
```

支持的关联格式：
- `(#123)` - GitHub/GitLab issue 编号
- `(PROJ-456)` - Jira 等项目管理工具编号
- `(URL)` - 完整链接

### 管理要求

- 定期清理过期 TODO（建议每个迭代检查）
- FIXME 应关联 issue 追踪
- HACK 必须说明原因和预期解决时间
- 使用 `git grep "TODO\|FIXME"` 定期检查待办项

---

## 类型注解 vs 注释

### 优先使用类型注解

```python
# ✅ 推荐：类型注解
def process(data: list[dict], config: Config) -> Result:
    pass

# ❌ 不推荐：注释说明类型
def process(data, config):
    # type: (list, Config) -> Result
    pass
```

### 复杂类型处理

```python
from typing import TypeAlias, Callable

# 使用类型别名提高可读性
UserDict: TypeAlias = dict[str, str | int | None]
ProcessFunc: TypeAlias = Callable[[list[UserDict]], dict]

def batch_process(
    users: list[UserDict],
    processor: ProcessFunc
) -> dict:
    """批量处理用户数据。"""
    pass
```

---

## IDE 配置

### VS Code 设置

```json
{
    "autoDocstring.docstringFormat": "google",
    "autoDocstring.startOnNewLine": true,
    "autoDocstring.includeExtendedSummary": true
}
```

### PyCharm 设置

```
Settings → Tools → Python Integrated Tools
→ Docstrings → Docstring format: Google
```

---

## 检查清单

### 编写时检查

- [ ] 公开 API 有完整 docstring
- [ ] Args/Returns/Raises 与实际一致
- [ ] 复杂逻辑有行内注释说明"为什么"
- [ ] TODO/FIXME 标记包含作者和描述
- [ ] 没有注释掉的代码

### 审查时检查

- [ ] docstring 与代码行为一致
- [ ] 没有显而易见的无用注释
- [ ] 类型注解完整，无需注释说明类型
- [ ] FIXME 已关联 issue
