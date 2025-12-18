# 日志规范

## 适用范围

本规范适用于所有需要记录日志的 Python 代码。

## 日志框架

使用 **structlog** 作为统一日志框架，输出结构化 JSON 日志。

```python
import structlog

logger = structlog.get_logger()
```

## 日志级别

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| **DEBUG** | 调试信息，生产环境不输出 | 变量值、执行路径 |
| **INFO** | 正常业务流程的关键节点 | 任务开始/完成、用户登录 |
| **WARNING** | 异常但可恢复的情况 | 重试、降级、配置缺失使用默认值 |
| **ERROR** | 错误但系统仍可运行 | 单条记录处理失败、外部服务超时 |
| **CRITICAL** | 严重错误，系统无法继续 | 数据库连接失败、配置错误 |

### 级别选择指南

```python
# ✅ DEBUG: 调试信息
logger.debug("查询参数", query=query, params=params)

# ✅ INFO: 业务关键节点
logger.info("任务开始", task_id=task_id, total_records=1000)
logger.info("任务完成", task_id=task_id, processed=950, failed=50)

# ✅ WARNING: 可恢复的异常
logger.warning("连接超时，正在重试", attempt=2, max_attempts=3)
logger.warning("配置缺失，使用默认值", key="timeout", default=30)

# ✅ ERROR: 单条失败但不影响整体
logger.error("记录处理失败", record_id=123, error=str(e))

# ✅ CRITICAL: 系统级故障
logger.critical("数据库连接失败，服务无法启动", host=host)
```

## 日志格式

### 结构化日志 (生产环境)

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "info",
  "logger": "task_runner",
  "message": "任务完成",
  "task_id": "abc-123",
  "processed": 950,
  "failed": 50,
  "duration_ms": 3420
}
```

### 配置示例

```python
import structlog

def configure_logging(json_format: bool = True) -> None:
    """配置日志"""
    processors = [
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]

    if json_format:
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
    )
```

## 日志内容规范

### 必须包含的上下文

```python
# ✅ 正确: 包含足够的上下文
logger.info(
    "用户登录成功",
    user_id=user.id,
    ip_address=request.client.host,
    user_agent=request.headers.get("user-agent"),
)

logger.error(
    "数据库查询失败",
    table="users",
    query_type="select",
    error_code=e.code,
    error_message=str(e),
)

# ❌ 错误: 信息不足
logger.info("登录成功")  # 谁登录的？
logger.error("查询失败")  # 什么查询？为什么失败？
```

### 任务执行日志模式

```python
async def process_task(task_id: str, records: list[dict]) -> None:
    logger = structlog.get_logger().bind(task_id=task_id)

    # 任务开始
    logger.info("任务开始", total_records=len(records))

    processed = 0
    failed = 0
    start_time = time.monotonic()

    for record in records:
        try:
            await process_record(record)
            processed += 1
        except Exception as e:
            failed += 1
            logger.error(
                "记录处理失败",
                record_id=record.get("id"),
                error=str(e),
            )

    # 任务完成
    duration_ms = (time.monotonic() - start_time) * 1000
    logger.info(
        "任务完成",
        processed=processed,
        failed=failed,
        duration_ms=round(duration_ms, 2),
    )
```

## 敏感信息处理

### 禁止记录的信息

- ❌ 密码、密钥、Token
- ❌ 身份证号、银行卡号
- ❌ 完整手机号、邮箱
- ❌ 数据库连接字符串中的密码
- ❌ 请求/响应中的敏感字段

### 脱敏处理

```python
def mask_sensitive(value: str, visible_chars: int = 4) -> str:
    """敏感信息脱敏"""
    if len(value) <= visible_chars:
        return "*" * len(value)
    return value[:visible_chars] + "*" * (len(value) - visible_chars)

# 使用
logger.info(
    "用户信息",
    phone=mask_sensitive("13812345678", 3),  # 138********
    id_card=mask_sensitive("110101199001011234", 4),  # 1101**************
)
```

### 自动脱敏处理器

```python
SENSITIVE_KEYS = {"password", "token", "secret", "api_key", "credential"}

def mask_sensitive_processor(
    logger: Any,
    method_name: str,
    event_dict: dict,
) -> dict:
    """自动脱敏敏感字段"""
    for key in event_dict:
        if any(s in key.lower() for s in SENSITIVE_KEYS):
            event_dict[key] = "***MASKED***"
    return event_dict
```

## 性能日志

### 接口耗时

```python
import time
from functools import wraps

def log_duration(func):
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.monotonic()
        try:
            result = await func(*args, **kwargs)
            return result
        finally:
            duration_ms = (time.monotonic() - start) * 1000
            logger.info(
                "函数执行完成",
                function=func.__name__,
                duration_ms=round(duration_ms, 2),
            )
    return wrapper
```

### 慢查询告警

```python
SLOW_QUERY_THRESHOLD_MS = 1000

async def execute_query(query: str) -> list:
    start = time.monotonic()
    result = await db.fetch_all(query)
    duration_ms = (time.monotonic() - start) * 1000

    if duration_ms > SLOW_QUERY_THRESHOLD_MS:
        logger.warning(
            "慢查询检测",
            query=query[:200],  # 截断过长查询
            duration_ms=round(duration_ms, 2),
            threshold_ms=SLOW_QUERY_THRESHOLD_MS,
        )

    return result
```

## 错误日志

### 异常记录

```python
try:
    result = await risky_operation()
except SpecificError as e:
    # 已知异常: 记录错误信息
    logger.error(
        "操作失败",
        error_type=type(e).__name__,
        error_code=e.code,
        error_message=str(e),
    )
    raise
except Exception as e:
    # 未知异常: 记录完整堆栈
    logger.exception(
        "未预期的错误",
        error_type=type(e).__name__,
    )
    raise
```

## 禁止事项

```python
# ❌ 禁止使用 print
print("debug info")  # 应使用 logger.debug()

# ❌ 禁止记录敏感信息
logger.info("用户登录", password=password)

# ❌ 禁止空的异常捕获
try:
    ...
except Exception:
    pass  # 必须记录日志

# ❌ 禁止在循环中频繁记录
for item in large_list:
    logger.info("处理", item=item)  # 应该聚合后记录

# ❌ 禁止日志消息拼接
logger.info(f"用户 {user_id} 登录")  # 应使用结构化参数
```

## 检查清单

- [ ] 使用 structlog 而非 print 或 logging
- [ ] 日志级别选择正确
- [ ] 包含足够的上下文信息
- [ ] 无敏感信息泄露
- [ ] 异常有完整的错误日志
- [ ] 关键操作有耗时记录
