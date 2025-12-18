# 错误处理规范

## 适用范围

本规范适用于所有需要进行错误处理的 Python 代码。

## 异常体系设计

### 基础异常类

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class ErrorDetail:
    """错误详情"""
    code: str
    message: str
    details: dict[str, Any] | None = None


class WorkbenchError(Exception):
    """基础异常类"""

    def __init__(
        self,
        code: str,
        message: str,
        details: dict[str, Any] | None = None,
    ) -> None:
        self.code = code
        self.message = message
        self.details = details or {}
        super().__init__(message)

    def to_dict(self) -> dict[str, Any]:
        return {
            "code": self.code,
            "message": self.message,
            "details": self.details,
        }
```

### 分类异常

```python
# 配置错误 (启动时)
class ConfigurationError(WorkbenchError):
    """配置错误"""
    pass


# 数据库错误
class DatabaseError(WorkbenchError):
    """数据库错误基类"""
    pass

class ConnectionError(DatabaseError):
    """连接错误"""
    pass

class QueryError(DatabaseError):
    """查询错误"""
    pass

class TransactionError(DatabaseError):
    """事务错误"""
    pass


# 外部服务错误
class ExternalServiceError(WorkbenchError):
    """外部服务错误基类"""
    pass

class LLMError(ExternalServiceError):
    """LLM 调用错误"""
    pass

class APIError(ExternalServiceError):
    """API 调用错误"""
    pass


# 业务逻辑错误
class BusinessError(WorkbenchError):
    """业务逻辑错误"""
    pass

class ValidationError(BusinessError):
    """数据校验错误"""
    pass

class NotFoundError(BusinessError):
    """资源不存在"""
    pass

class PermissionError(BusinessError):
    """权限不足"""
    pass
```

## 错误码规范

### 错误码格式

```
格式: {模块}.{类别}.{具体错误}

模块:
- SYS: 系统级
- DB: 数据库
- LLM: LLM服务
- BIZ: 业务逻辑
- API: API相关

类别:
- CONN: 连接
- QUERY: 查询
- AUTH: 认证授权
- VALID: 校验
- TIMEOUT: 超时
```

### 错误码定义

```python
from enum import StrEnum


class ErrorCode(StrEnum):
    # 系统级错误 SYS.*
    SYS_INTERNAL = "SYS.INTERNAL.UNKNOWN"
    SYS_CONFIG_INVALID = "SYS.CONFIG.INVALID"
    SYS_CONFIG_MISSING = "SYS.CONFIG.MISSING"

    # 数据库错误 DB.*
    DB_CONN_FAILED = "DB.CONN.FAILED"
    DB_CONN_TIMEOUT = "DB.CONN.TIMEOUT"
    DB_QUERY_FAILED = "DB.QUERY.FAILED"
    DB_QUERY_TIMEOUT = "DB.QUERY.TIMEOUT"
    DB_TRANSACTION_FAILED = "DB.TRANSACTION.FAILED"

    # LLM 错误 LLM.*
    LLM_CONN_FAILED = "LLM.CONN.FAILED"
    LLM_RATE_LIMITED = "LLM.RATE.LIMITED"
    LLM_RESPONSE_INVALID = "LLM.RESPONSE.INVALID"
    LLM_TOKEN_EXCEEDED = "LLM.TOKEN.EXCEEDED"

    # 业务错误 BIZ.*
    BIZ_VALID_FAILED = "BIZ.VALID.FAILED"
    BIZ_NOT_FOUND = "BIZ.NOT_FOUND"
    BIZ_PERMISSION_DENIED = "BIZ.PERMISSION.DENIED"
    BIZ_DUPLICATE = "BIZ.DUPLICATE"
```

### 使用示例

```python
# 抛出异常
raise DatabaseError(
    code=ErrorCode.DB_CONN_TIMEOUT,
    message="数据库连接超时",
    details={"host": host, "timeout_seconds": 30},
)

raise ValidationError(
    code=ErrorCode.BIZ_VALID_FAILED,
    message="数据校验失败",
    details={"field": "email", "reason": "格式不正确"},
)
```

## 异常处理模式

### 基本模式

```python
import structlog

logger = structlog.get_logger()

# ✅ 正确: 明确捕获特定异常
try:
    result = await query_database(sql)
except ConnectionError as e:
    logger.error("数据库连接失败", error=e.to_dict())
    raise
except QueryError as e:
    logger.error("查询执行失败", error=e.to_dict())
    raise

# ❌ 错误: 捕获所有异常并吞掉
try:
    result = await query_database(sql)
except Exception:
    pass  # 绝对禁止
```

### 异常转换

```python
async def fetch_user(user_id: int) -> User:
    """获取用户，将底层异常转换为业务异常"""
    try:
        result = await db.fetch_one(
            "SELECT * FROM users WHERE id = :id",
            {"id": user_id},
        )
    except Exception as e:
        logger.exception("查询用户失败", user_id=user_id)
        raise DatabaseError(
            code=ErrorCode.DB_QUERY_FAILED,
            message="查询用户失败",
            details={"user_id": user_id},
        ) from e

    if result is None:
        raise NotFoundError(
            code=ErrorCode.BIZ_NOT_FOUND,
            message="用户不存在",
            details={"user_id": user_id},
        )

    return User(**result)
```

### 上下文管理器

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator


@asynccontextmanager
async def db_transaction() -> AsyncGenerator[Connection, None]:
    """数据库事务上下文管理"""
    conn = await get_connection()
    try:
        await conn.begin()
        yield conn
        await conn.commit()
    except Exception as e:
        await conn.rollback()
        logger.error("事务回滚", error=str(e))
        raise TransactionError(
            code=ErrorCode.DB_TRANSACTION_FAILED,
            message="事务执行失败",
        ) from e
    finally:
        await conn.close()


# 使用
async with db_transaction() as conn:
    await conn.execute(...)
    await conn.execute(...)
```

## 重试策略

### 重试装饰器

```python
import asyncio
from functools import wraps
from typing import Type


def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple[Type[Exception], ...] = (Exception,),
):
    """重试装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            current_delay = delay

            for attempt in range(1, max_attempts + 1):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt < max_attempts:
                        logger.warning(
                            "操作失败，准备重试",
                            function=func.__name__,
                            attempt=attempt,
                            max_attempts=max_attempts,
                            delay=current_delay,
                            error=str(e),
                        )
                        await asyncio.sleep(current_delay)
                        current_delay *= backoff

            logger.error(
                "重试次数耗尽",
                function=func.__name__,
                max_attempts=max_attempts,
            )
            raise last_exception

        return wrapper
    return decorator


# 使用
@retry(max_attempts=3, delay=1.0, exceptions=(ConnectionError, TimeoutError))
async def connect_database():
    ...
```

### 可重试异常

```python
# 可重试的异常类型
RETRYABLE_EXCEPTIONS = (
    ConnectionError,
    TimeoutError,
    # 数据库临时错误
    # LLM 限流错误
)

# 不可重试的异常类型 (业务错误)
NON_RETRYABLE_EXCEPTIONS = (
    ValidationError,
    NotFoundError,
    PermissionError,
)
```

## API 错误响应

### 响应格式

```python
from pydantic import BaseModel


class ErrorResponse(BaseModel):
    """API 错误响应"""
    success: bool = False
    error: ErrorDetail


# 响应示例
{
    "success": false,
    "error": {
        "code": "BIZ.NOT_FOUND",
        "message": "用户不存在",
        "details": {
            "user_id": 123
        }
    }
}
```

### FastAPI 异常处理

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()


@app.exception_handler(WorkbenchError)
async def workbench_error_handler(
    request: Request,
    exc: WorkbenchError,
) -> JSONResponse:
    """处理业务异常"""
    status_code = _get_status_code(exc)
    return JSONResponse(
        status_code=status_code,
        content={"success": False, "error": exc.to_dict()},
    )


def _get_status_code(exc: WorkbenchError) -> int:
    """根据异常类型返回 HTTP 状态码"""
    if isinstance(exc, NotFoundError):
        return 404
    if isinstance(exc, ValidationError):
        return 400
    if isinstance(exc, PermissionError):
        return 403
    if isinstance(exc, (DatabaseError, ExternalServiceError)):
        return 503
    return 500
```

## 禁止事项

```python
# ❌ 禁止空的异常捕获
try:
    ...
except Exception:
    pass

# ❌ 禁止捕获后只打印
try:
    ...
except Exception as e:
    print(e)  # 应该使用 logger 并重新抛出

# ❌ 禁止使用裸 raise
try:
    ...
except Exception:
    raise Exception("出错了")  # 丢失了原始异常信息

# ✅ 正确: 使用 from 保留异常链
try:
    ...
except Exception as e:
    raise WorkbenchError(...) from e

# ❌ 禁止在异常中包含敏感信息
raise DatabaseError(
    code=ErrorCode.DB_CONN_FAILED,
    message="连接失败",
    details={"password": password},  # 禁止
)
```

## 检查清单

- [ ] 使用自定义异常类而非内置异常
- [ ] 异常包含错误码和上下文信息
- [ ] 捕获特定异常而非 Exception
- [ ] 异常有完整的日志记录
- [ ] 使用 `from e` 保留异常链
- [ ] 可重试操作有重试机制
- [ ] API 有统一的错误响应格式
