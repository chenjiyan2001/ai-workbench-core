# 变更日志 (Git Upgrades)

本文件记录项目的所有版本变更历史，由 Claude 在每次提交时自动更新。

---

## 版本记录

### [v0.1.0] - 2024-12-18

#### 📋 变更摘要
初始版本，建立项目规范文档体系，为数据处理平台开发提供统一规范。

#### ✨ 新增 (Added)
- **开发规范** (development/)
  - Git 使用规范：分支命名、提交规范、操作权限矩阵、Claude 行为指南
  - Python 代码规范：代码风格、类型注解、项目结构
  - 日志规范：日志级别、格式、敏感信息脱敏
  - 配置管理规范：环境变量、密钥管理、Pydantic Settings
  - 错误处理规范：异常体系、错误码、重试策略

- **数据库规范** (database/)
  - MySQL 规范：命名、SQL 编写、索引、事务
  - MongoDB 规范：集合设计、索引、聚合管道
  - Redis 规范：Key 命名、过期策略、缓存模式
  - Milvus 规范：向量集合、索引配置、搜索优化
  - 达梦规范：兼容性处理、SQL 适配

- **集成规范** (integration/)
  - LLM 调用规范：Prompt 管理、成本控制、调用追踪
  - API 设计规范：RESTful、响应格式、认证授权

- **架构文档** (architecture/)
  - 整体架构概述
  - ADR-0001: 采用模块化单体架构决策

- **项目配置**
  - CLAUDE.md：Claude 行为指令文件
  - .gitignore：Git 忽略规则
  - 变更日志模板

#### 🐛 修复 (Fixed)
- 无

#### 🔧 变更 (Changed)
- 无

#### 🗑️ 移除 (Removed)
- 无

#### 📁 涉及文件
```
.gitignore
CLAUDE.md
README.md
development/git.md
development/python.md
development/logging.md
development/config.md
development/error-handling.md
database/mysql.md
database/mongodb.md
database/redis.md
database/milvus.md
database/dameng.md
integration/llm.md
integration/api.md
architecture/overview.md
architecture/adr/0001-modular-monolith.md
docs/upgrades/git-upgrades.md
```

#### 💬 备注
项目初始化版本，建立完整的规范文档体系，供团队成员和 Claude 在后续开发中参考遵循。

---

## 变更日志模板

> Claude 在记录新版本时，复制以下模板并填写

```markdown
### [vX.Y.Z] - YYYY-MM-DD

#### 📋 变更摘要
一句话描述本次变更的核心内容。

#### ✨ 新增 (Added)
- 新增功能或文件

#### 🐛 修复 (Fixed)
- 修复的 Bug

#### 🔧 变更 (Changed)
- 修改的现有功能

#### 🗑️ 移除 (Removed)
- 删除的功能或文件

#### 📁 涉及文件
```
path/to/file1.py
path/to/file2.py
```

#### 💬 备注
补充说明（可选）。
```

---

## 版本号规则

| 版本位 | 变更类型 | 示例 |
|--------|---------|------|
| major (X) | 重大变更，不兼容 | v1.0.0 → v2.0.0 |
| minor (Y) | 新功能，向下兼容 | v1.0.0 → v1.1.0 |
| patch (Z) | Bug 修复 | v1.0.0 → v1.0.1 |

---

## 标签说明

| 标签 | 含义 |
|------|------|
| ✨ Added | 新增功能 |
| 🐛 Fixed | Bug 修复 |
| 🔧 Changed | 功能变更 |
| 🗑️ Removed | 移除功能 |
| 📁 Files | 涉及文件 |
| 💬 Notes | 备注说明 |
