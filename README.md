# AI Workbench Core - 规范文档库

本仓库存放数据处理平台的开发规范与架构文档，供团队成员和 Claude 参考遵循。

## 文档索引

### 开发规范 (development/)

| 文档 | 说明 |
|------|------|
| [git.md](development/git.md) | Git 分支、提交、Tag 基础规范 |
| [git-operations.md](development/git-operations.md) | Git 操作场景指南 |
| [git-claude.md](development/git-claude.md) | Claude Git 行为规范 |
| [python.md](development/python.md) | Python 代码风格、类型注解、项目结构 |
| [logging.md](development/logging.md) | 日志级别、格式、敏感信息处理 |
| [config.md](development/config.md) | 配置管理、环境变量、密钥处理 |
| [error-handling.md](development/error-handling.md) | 异常定义、错误码、重试策略 |
| [scripts.md](development/scripts.md) | 脚本编写规范（参数、多线程、断点续传） |

### 数据库规范 (database/)

| 文档 | 说明 |
|------|------|
| [mysql.md](database/mysql.md) | MySQL 命名、SQL 编写、索引、事务 |
| [mongodb.md](database/mongodb.md) | MongoDB 集合设计、索引、查询模式 |
| [redis.md](database/redis.md) | Redis Key 命名、过期策略、数据结构 |
| [milvus.md](database/milvus.md) | Milvus 向量集合设计、索引配置 |
| [dameng.md](database/dameng.md) | 达梦数据库兼容性、特殊处理 |

### 集成规范 (integration/)

| 文档 | 说明 |
|------|------|
| [llm.md](integration/llm.md) | LLM 调用、Prompt 管理、成本控制 |
| [api.md](integration/api.md) | API 设计、版本控制、错误响应 |

### 架构文档 (architecture/)

| 文档 | 说明 |
|------|------|
| [overview.md](architecture/overview.md) | 整体架构设计 |
| [adr/](architecture/adr/) | 架构决策记录 (ADR) |

### 文档撰写规范 (docs-standard/)

| 文档 | 说明 |
|------|------|
| [readme.md](docs-standard/readme.md) | 项目文档规范（README、使用指南） |
| [api-docs.md](docs-standard/api-docs.md) | API 文档规范 |
| [code-comments.md](docs-standard/code-comments.md) | 代码注释规范 |
| [architecture.md](docs-standard/architecture.md) | 架构文档规范 |
| [changelog.md](docs-standard/changelog.md) | 变更日志规范 |

### 变更记录 (docs/upgrades/)

| 文档 | 说明 |
|------|------|
| [git-upgrades.md](docs/upgrades/git-upgrades.md) | Git 版本变更日志 |

## 规范等级说明

- **MUST / 必须**: 强制要求，违反会导致代码审查不通过
- **SHOULD / 应该**: 推荐做法，特殊情况可例外但需说明理由
- **MAY / 可以**: 可选建议，视具体场景决定

## 如何使用

1. **开发前**: 阅读相关规范文档
2. **开发中**: 遵循规范编写代码
3. **代码审查**: 以规范为依据进行 Review
4. **Claude 协作**: Claude 会自动读取 CLAUDE.md 并遵循这些规范
