# API 设计规范

## 适用范围

本规范适用于所有 RESTful API 的设计与实现。

## URL 设计

### 基本规则

```
格式: /{version}/{resource}/{id}/{sub-resource}
小写，连字符分隔，使用复数名词

✅ 正确
GET  /v1/users
GET  /v1/users/123
GET  /v1/users/123/roles
POST /v1/tasks
GET  /v1/data-sources

❌ 错误
GET  /v1/getUsers          -- 不要用动词
GET  /v1/user/123          -- 应使用复数
GET  /v1/users/123/getRole -- 不要用动词
GET  /v1/dataSources       -- 应使用连字符
```

### 资源命名

```
# 资源类型
/v1/users                  # 用户
/v1/tasks                  # 任务
/v1/data-sources           # 数据源
/v1/llm-jobs               # LLM 任务
/v1/quality-rules          # 质量规则

# 子资源
/v1/users/123/roles        # 用户的角色
/v1/tasks/456/logs         # 任务的日志
/v1/data-sources/789/schema # 数据源的 Schema
```

### 版本控制

```
# URL 路径版本 (推荐)
/v1/users
/v2/users

# 主版本变更时机:
# - 删除端点
# - 删除或重命名字段
# - 改变字段类型
# - 改变业务语义
```

## HTTP 方法

| 方法 | 用途 | 幂等 | 示例 |
|------|------|------|------|
| GET | 查询资源 | 是 | GET /v1/users/123 |
| POST | 创建资源 | 否 | POST /v1/users |
| PUT | 全量更新 | 是 | PUT /v1/users/123 |
| PATCH | 部分更新 | 是 | PATCH /v1/users/123 |
| DELETE | 删除资源 | 是 | DELETE /v1/users/123 |

### 方法使用示例

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel

app = FastAPI()


class UserCreate(BaseModel):
    name: str
    email: str


class UserUpdate(BaseModel):
    name: str | None = None
    email: str | None = None


# GET - 查询列表
@app.get("/v1/users")
async def list_users(
    skip: int = 0,
    limit: int = 20,
    status: str | None = None,
) -> dict:
    ...


# GET - 查询单个
@app.get("/v1/users/{user_id}")
async def get_user(user_id: int) -> dict:
    ...


# POST - 创建
@app.post("/v1/users", status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate) -> dict:
    ...


# PUT - 全量更新
@app.put("/v1/users/{user_id}")
async def replace_user(user_id: int, user: UserCreate) -> dict:
    ...


# PATCH - 部分更新
@app.patch("/v1/users/{user_id}")
async def update_user(user_id: int, user: UserUpdate) -> dict:
    ...


# DELETE - 删除
@app.delete("/v1/users/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(user_id: int) -> None:
    ...
```

## 请求格式

### 查询参数

```python
# 分页
GET /v1/users?page=1&page_size=20
GET /v1/users?offset=0&limit=20  # 也可以用 offset/limit

# 排序
GET /v1/users?sort_by=created_at&order=desc

# 过滤
GET /v1/users?status=active&role=admin

# 搜索
GET /v1/users?q=张三

# 字段选择
GET /v1/users?fields=id,name,email

# 组合示例
GET /v1/users?status=active&sort_by=created_at&order=desc&page=1&page_size=20
```

### 请求体

```json
// POST /v1/users
{
  "name": "张三",
  "email": "zhangsan@example.com",
  "role_ids": [1, 2, 3],
  "metadata": {
    "department": "技术部"
  }
}
```

## 响应格式

### 成功响应

```python
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")


class Response(BaseModel, Generic[T]):
    """统一响应格式"""
    success: bool = True
    data: T
    message: str | None = None


class PaginatedResponse(BaseModel, Generic[T]):
    """分页响应格式"""
    success: bool = True
    data: list[T]
    pagination: "Pagination"


class Pagination(BaseModel):
    """分页信息"""
    page: int
    page_size: int
    total: int
    total_pages: int
    has_next: bool
    has_prev: bool
```

### 响应示例

```json
// GET /v1/users/123
{
  "success": true,
  "data": {
    "id": 123,
    "name": "张三",
    "email": "zhangsan@example.com",
    "created_at": "2024-01-15T10:30:00Z"
  }
}

// GET /v1/users?page=1&page_size=20
{
  "success": true,
  "data": [
    {"id": 1, "name": "张三", ...},
    {"id": 2, "name": "李四", ...}
  ],
  "pagination": {
    "page": 1,
    "page_size": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
  }
}

// POST /v1/users (201 Created)
{
  "success": true,
  "data": {
    "id": 124,
    "name": "新用户",
    ...
  },
  "message": "用户创建成功"
}
```

## 错误响应

### 错误格式

```python
class ErrorDetail(BaseModel):
    """错误详情"""
    code: str           # 错误码
    message: str        # 错误信息
    field: str | None   # 出错字段 (校验错误时)
    details: dict | None  # 额外信息


class ErrorResponse(BaseModel):
    """错误响应"""
    success: bool = False
    error: ErrorDetail
```

### 错误示例

```json
// 400 Bad Request - 参数错误
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求参数校验失败",
    "details": {
      "errors": [
        {"field": "email", "message": "邮箱格式不正确"},
        {"field": "name", "message": "名称不能为空"}
      ]
    }
  }
}

// 401 Unauthorized - 未认证
{
  "success": false,
  "error": {
    "code": "UNAUTHORIZED",
    "message": "请先登录"
  }
}

// 403 Forbidden - 无权限
{
  "success": false,
  "error": {
    "code": "FORBIDDEN",
    "message": "无权访问该资源"
  }
}

// 404 Not Found - 资源不存在
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "用户不存在",
    "details": {"user_id": 999}
  }
}

// 500 Internal Server Error - 服务器错误
{
  "success": false,
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "服务器内部错误，请稍后重试"
  }
}
```

### HTTP 状态码

| 状态码 | 含义 | 使用场景 |
|-------|------|---------|
| 200 | OK | GET/PUT/PATCH 成功 |
| 201 | Created | POST 创建成功 |
| 204 | No Content | DELETE 成功 |
| 400 | Bad Request | 参数错误 |
| 401 | Unauthorized | 未认证 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 资源冲突 |
| 422 | Unprocessable Entity | 业务逻辑错误 |
| 429 | Too Many Requests | 限流 |
| 500 | Internal Server Error | 服务器错误 |
| 503 | Service Unavailable | 服务不可用 |

## 认证与授权

### Bearer Token

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> User:
    """验证 Token 并获取当前用户"""
    token = credentials.credentials
    try:
        payload = decode_token(token)
        user = await get_user(payload["user_id"])
        if not user:
            raise HTTPException(status_code=401, detail="用户不存在")
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Token 无效")


# 使用
@app.get("/v1/users/me")
async def get_me(current_user: User = Depends(get_current_user)) -> dict:
    return {"data": current_user}
```

### 权限检查

```python
from functools import wraps


def require_permission(permission: str):
    """权限检查装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user: User = Depends(get_current_user), **kwargs):
            if permission not in current_user.permissions:
                raise HTTPException(status_code=403, detail="无权限")
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator


@app.delete("/v1/users/{user_id}")
@require_permission("user:delete")
async def delete_user(user_id: int, current_user: User = Depends(get_current_user)) -> None:
    ...
```

## 限流

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)


@app.get("/v1/users")
@limiter.limit("100/minute")
async def list_users(request: Request) -> dict:
    ...


# 响应头
# X-RateLimit-Limit: 100
# X-RateLimit-Remaining: 95
# X-RateLimit-Reset: 1704067200
```

## 文档

### OpenAPI 配置

```python
from fastapi import FastAPI

app = FastAPI(
    title="Workbench API",
    description="数据处理平台 API",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
)
```

### 端点文档

```python
@app.get(
    "/v1/users/{user_id}",
    summary="获取用户详情",
    description="根据用户 ID 获取用户详细信息",
    response_description="用户详情",
    responses={
        200: {"description": "成功"},
        404: {"description": "用户不存在"},
    },
    tags=["用户管理"],
)
async def get_user(
    user_id: int = Path(..., description="用户 ID", example=123),
) -> Response[User]:
    """
    获取用户详情

    - **user_id**: 用户唯一标识
    """
    ...
```

## 最佳实践

### 幂等性设计

```python
# 使用幂等键防止重复创建
@app.post("/v1/orders")
async def create_order(
    order: OrderCreate,
    idempotency_key: str = Header(..., alias="X-Idempotency-Key"),
) -> dict:
    # 检查幂等键是否已处理
    existing = await cache.get(f"idempotency:{idempotency_key}")
    if existing:
        return json.loads(existing)

    # 创建订单
    result = await do_create_order(order)

    # 缓存结果
    await cache.set(f"idempotency:{idempotency_key}", json.dumps(result), ex=86400)

    return result
```

### 异步任务

```python
# 长时间任务返回任务 ID
@app.post("/v1/tasks", status_code=status.HTTP_202_ACCEPTED)
async def create_task(task: TaskCreate) -> dict:
    task_id = await enqueue_task(task)
    return {
        "success": true,
        "data": {
            "task_id": task_id,
            "status": "pending",
            "status_url": f"/v1/tasks/{task_id}",
        },
        "message": "任务已提交",
    }


# 轮询任务状态
@app.get("/v1/tasks/{task_id}")
async def get_task_status(task_id: str) -> dict:
    task = await get_task(task_id)
    return {
        "success": True,
        "data": {
            "task_id": task_id,
            "status": task.status,  # pending / running / completed / failed
            "progress": task.progress,
            "result": task.result if task.status == "completed" else None,
        },
    }
```

## 禁止事项

- ❌ 禁止在 URL 中使用动词
- ❌ 禁止使用 GET 进行数据修改
- ❌ 禁止在 URL 中暴露敏感信息
- ❌ 禁止返回不统一的响应格式
- ❌ 禁止忽略分页 (可能返回大量数据)
- ❌ 禁止在错误响应中暴露堆栈信息

## 检查清单

- [ ] URL 使用复数名词、连字符
- [ ] HTTP 方法使用正确
- [ ] 响应格式统一
- [ ] 错误响应包含错误码
- [ ] 有认证和权限控制
- [ ] 有限流保护
- [ ] API 文档完整
