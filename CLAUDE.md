# CLAUDE.md - AI Workbench 开发规范指令

本文件为 Claude 在协作开发时的行为指令，Claude 必须遵循本仓库中的所有规范文档。

## 项目概述

- **项目类型**: 数据处理平台
- **主要语言**: Python 3.11+
- **涉及数据库**: MySQL、MongoDB、Redis、Milvus、达梦
- **集成服务**: LLM (打标/摘要/抽取)

## 规范文档位置

| 场景 | 规范文档 |
|------|---------|
| Git 基础规范 | [development/git.md](development/git.md) |
| Git 操作指南 | [development/git-operations.md](development/git-operations.md) |
| Git Claude 行为 | [development/git-claude.md](development/git-claude.md) |
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
| 项目文档规范 | [docs-standard/readme.md](docs-standard/readme.md) |
| API 文档规范 | [docs-standard/api-docs.md](docs-standard/api-docs.md) |
| 代码注释规范 | [docs-standard/code-comments.md](docs-standard/code-comments.md) |
| 架构文档规范 | [docs-standard/architecture.md](docs-standard/architecture.md) |
| 变更日志规范 | [docs-standard/changelog.md](docs-standard/changelog.md) |

### 文档查阅原则

**只在必要时查阅文档，避免不必要的 token 消耗。**

| 需要查阅 | 不需要查阅 |
|---------|-----------|
| 首次接触某模块/功能 | 同一对话中已读过该文档 |
| 不确定规范细节时 | 常规简单操作 |
| 涉及危险操作需确认权限 | 用户给出明确具体指令 |
| 用户明确要求按规范执行 | 日常讨论/问答 |

## Claude/Codex 协作分工

Claude 与 Codex 在本项目中有明确的职责分工：

### Claude 职责

| 职责 | 说明 |
|------|------|
| 代码编写与修改 | 核心工作，专注于实现用户需求 |
| 用户交互 | 理解需求、解答问题、确认意图 |
| Git 操作决策与执行 | 识别时机、建议操作、执行命令 |
| 提交信息编写 | 简短的 Conventional Commits 格式 |
| 需求分析与方案设计 | 与用户沟通确定技术方案 |

### Codex 职责

| 职责 | 说明 |
|------|------|
| 变更日志编写 | 更新 `docs/upgrades/git-upgrades.md` |
| 大量文本输出 | 文档生成、报告编写等 |
| 代码审查与建议 | Review 代码改动 |
| 复杂分析报告 | 技术分析、方案对比等 |

### 协作原则

1. **Claude 主导交互**: 用户沟通由 Claude 负责
2. **输出任务委托**: 大量文本输出委托给 Codex
3. **审查闭环**: 代码修改后由 Codex 进行 review
4. **聚焦原则**: Claude 聚焦代码，Codex 聚焦文档

---

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
4. **识别用户意图**: 用户不会直接说"创建分支"，需从自然语言中识别
   - "帮我开发一个功能" → 建议创建 feature 分支
   - "这里有 bug，帮我修" → 建议创建 fix 分支
   - "帮我重构这个模块" → 建议创建 refactor 分支
5. **适时建议**: 当用户确认功能完成时，建议提交和推送
6. **变更日志**: 提交时委托 Codex 编写变更日志

## 语言要求

- 代码注释: 中文
- 文档: 中文
- Git 提交信息: 中文
- 变量/函数命名: 英文 (遵循 Python 命名规范)

## 命令执行环境

### 用户系统环境

| 项目 | 值 |
|------|-----|
| 操作系统 | Windows |
| 终端类型 | PowerShell / CMD |
| 工作目录 | `E:\ai-workbench-core` |

### Claude 内部执行环境

| 项目 | 实际情况 |
|------|---------|
| Bash 执行器 | Unix shell (`/usr/bin/bash`) |
| 路径风格 | Unix (`/`) |
| 环境变量 | `$VAR` |

### 命令记录规范

**文档中的终端命令必须记录两种版本：**

```markdown
### 安装依赖

**Unix (Linux/macOS):**
```bash
export DATABASE_URL="mysql://user:pass@localhost/db"
pip install -r requirements.txt
```

**Windows (PowerShell):**
```powershell
$env:DATABASE_URL="mysql://user:pass@localhost/db"
pip install -r requirements.txt
```
```

### 执行规范

| 场景 | 规范 |
|------|------|
| Claude 内部执行 | **始终使用 Unix 语法**（Bash 工具） |
| 提供给用户执行 | **根据用户系统选择**（参考上方「用户系统环境」） |
| 文档撰写 | **记录双版本**（Unix + Windows） |

### 常用命令对照表

| 操作 | Unix (Bash) | Windows (PowerShell) |
|------|-------------|---------------------|
| 设置环境变量 | `export VAR=value` | `$env:VAR="value"` |
| 读取环境变量 | `echo $VAR` | `echo $env:VAR` |
| 路径分隔符 | `/` | `\` 或 `/`（PowerShell 兼容） |
| 多命令串联 | `cmd1 && cmd2` | `cmd1; cmd2` 或 `cmd1 -and cmd2` |
| 删除文件 | `rm file` | `Remove-Item file` |
| 删除目录 | `rm -rf dir` | `Remove-Item -Recurse -Force dir` |
| 查看文件 | `cat file` | `Get-Content file` |
| 查找文件 | `find . -name "*.py"` | `Get-ChildItem -Recurse -Filter "*.py"` |

### 示例

**Claude 内部执行（始终 Unix）：**
```bash
# ✅ 正确
git status
python script.py
export MY_VAR=value && python app.py

# ❌ 错误：Windows 语法
$env:MY_VAR="value"
```

**提供给用户（本项目为 Windows）：**
```powershell
# 设置环境变量并运行
$env:DATABASE_URL="mysql://localhost/db"
python app.py
```

## 禁止事项

- 禁止在代码中硬编码密钥、密码、连接字符串
- 禁止在日志中打印敏感信息 (密码、token、身份证号等)
- 禁止使用 `SELECT *`，必须明确字段列表
- 禁止忽略异常 (空的 except 块)
- 禁止在生产代码中使用 print()，必须使用日志
