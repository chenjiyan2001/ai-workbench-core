# CLAUDE.md - AI Workbench 开发规范指令

本文件为 Claude 在协作开发时的行为指令，Claude 必须遵循本仓库中的所有规范文档。

## 项目概述

- **项目类型**: 数据处理平台
- **主要语言**: Python 3.11+
- **涉及数据库**: MySQL、MongoDB、Redis、Milvus、达梦
- **集成服务**: LLM (打标/摘要/抽取)

## 规范文档位置

Claude 在编写代码前，必须参考以下规范：

| 场景 | 规范文档 |
|------|---------|
| Git 操作 | [development/git.md](development/git.md) |
| Python 代码 | [development/python.md](development/python.md) |
| 日志记录 | [development/logging.md](development/logging.md) |
| 配置管理 | [development/config.md](development/config.md) |
| 错误处理 | [development/error-handling.md](development/error-handling.md) |
| MySQL | [database/mysql.md](database/mysql.md) |
| MongoDB | [database/mongodb.md](database/mongodb.md) |
| Redis | [database/redis.md](database/redis.md) |
| Milvus | [database/milvus.md](database/milvus.md) |
| 达梦 | [database/dameng.md](database/dameng.md) |
| LLM 调用 | [integration/llm.md](integration/llm.md) |
| API 设计 | [integration/api.md](integration/api.md) |

## 核心指令

### 代码编写

1. **先读规范**: 编写代码前，先阅读相关规范文档
2. **遵循命名**: 严格遵循各规范中的命名约定
3. **类型注解**: Python 代码必须有完整的类型注解
4. **日志规范**: 使用 structlog，遵循日志级别定义
5. **错误处理**: 使用自定义异常类，包含错误码

### 数据库操作

1. **SQL 注入防护**: 必须使用参数化查询，禁止字符串拼接
2. **事务边界**: 明确事务范围，避免长事务
3. **索引意识**: 查询必须考虑索引使用情况
4. **连接管理**: 使用连接池，及时释放连接

### LLM 调用

1. **成本意识**: 记录每次调用的 token 消耗
2. **结构化输出**: 强制使用 JSON Schema 约束输出
3. **可追溯**: 记录 model_id、prompt_version、input_hash
4. **错误降级**: 实现重试和降级策略

### Git 操作

1. **提交信息**: 遵循 Conventional Commits 格式
2. **分支命名**: 遵循 `type/description` 格式
3. **原子提交**: 每次提交只做一件事

## 语言要求

- 代码注释: 中文
- 文档: 中文
- Git 提交信息: 中文
- 变量/函数命名: 英文 (遵循 Python 命名规范)

## 禁止事项

- 禁止在代码中硬编码密钥、密码、连接字符串
- 禁止在日志中打印敏感信息 (密码、token、身份证号等)
- 禁止使用 `SELECT *`，必须明确字段列表
- 禁止忽略异常 (空的 except 块)
- 禁止在生产代码中使用 print()，必须使用日志
