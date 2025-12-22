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
| **工作流程** | [development/workflow.md](development/workflow.md) |
| **CI/CD 配置** | [development/ci.md](development/ci.md) |
| Git 基础规范 | [development/git.md](development/git.md) |
| Git 操作指南 | [development/git-operations.md](development/git-operations.md) |
| Git Claude 行为 | [development/git-claude.md](development/git-claude.md) |
| Python 代码 | [development/python.md](development/python.md) |
| 日志记录 | [development/logging.md](development/logging.md) |
| 日志格式 | [development/logging-format.md](development/logging-format.md) |
| 日志 ETL | [development/logging-etl.md](development/logging-etl.md) |
| 配置管理 | [development/config.md](development/config.md) |
| 错误处理 | [development/error-handling.md](development/error-handling.md) |
| 脚本编写 | [development/scripts.md](development/scripts.md) |
| 并发编程 | [development/python-concurrency.md](development/python-concurrency.md) |
| 性能优化 | [development/python-performance.md](development/python-performance.md) |
| 通用组件 | [development/python-components.md](development/python-components.md) |
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

## 工作流程（强制执行）

> **详细说明**：本章节为核心流程的简化版，完整内容请参考 [development/workflow.md](development/workflow.md)

### 核心理念

**测试驱动 + 代码审查的 AI 开发流程**，避免 AI 生成代码演化为不可维护的"石山代码"。

```
核心公式：Prompt → Test → Review → Implementation → Review → CI/CD
```

**四大原则**：
1. **测试是可执行的需求**，不是验证工具
2. **Prompt 是第一等工程资产**，必须存档
3. **Review 是强制质量关卡**，Codex + 人工双重审查
4. **流程强制顺序执行**，禁止跳过阶段

---

### 五阶段模型

#### 阶段 0：Prompt（人类主导）

**目的**：明确需求、边界、假设

**必须包含**：
- 业务目标（Why）
- 行为定义（What）
- 边界条件 / 排除项
- 示例（正例 / 反例）
- 暂不关心的内容（Out of Scope）

**产出物**：
- 文本 Prompt（存档到项目文档或 Git）

**禁止**：
- ❌ 不写代码
- ❌ 不写测试

---

#### 阶段 1：Test（AI 主导，强约束）

**目的**：将 Prompt 转化为可执行规范

**测试地位**：
> **Tests are the source of truth.**

**必须覆盖**：
1. 正常路径（Happy Path）
2. 边界条件
3. 反例（明确禁止的情况）
4. 不变量（长期必须成立的规则）

**禁止**：
- ❌ 不写实现代码
- ❌ 不假设实现细节
- ✅ 允许写失败的测试

---

#### 阶段 2：Test Review（Codex + 人工，强制）

**目的**：确保测试正确、完整、符合需求

**Review 流程**：

1. **Codex Review**（自动）
   - Claude 通过 **codex-ccb** 调用 Codex 进行审查
   - Codex 检查：
     - 测试覆盖是否完整
     - 边界条件是否遗漏
     - 测试逻辑是否正确
     - 是否有冗余测试

2. **人工 Review**（必须）
   - 用户审查 Codex 的反馈
   - 确认测试符合业务需求
   - 批准进入下一阶段

**产出物**：
- Codex review 报告
- 人工批准记录

**禁止**：
- ❌ 跳过 Codex review
- ❌ 跳过人工确认
- ❌ 在 review 未通过时进入 Implementation

---

#### 阶段 3：Implementation（AI 主导，最小化）

**目的**：让所有测试通过，不引入额外行为

**实现原则**：
- Tests 决定"对错"
- 实现只负责"如何做到"
- **不做顺手优化**

**禁止**：
- ❌ 修改测试（除非测试本身被否定）
- ❌ 扩展需求
- ❌ 提前抽象

---

#### 阶段 4：Implementation Review（Codex + 人工，强制）

**目的**：确保实现质量、代码规范、无安全隐患

**Review 流程**：

1. **Codex Review**（自动）
   - Claude 通过 **codex-ccb** 调用 Codex 进行审查
   - Codex 检查：
     - 代码质量（可读性、可维护性）
     - 是否遵循规范（命名、注释、类型注解）
     - 性能问题
     - 安全隐患
     - 是否引入额外功能（未测试覆盖）

2. **人工 Review**（必须）
   - 用户审查 Codex 的反馈
   - 确认代码符合团队标准
   - 批准合并到主干

**产出物**：
- Codex review 报告
- 人工批准记录

**禁止**：
- ❌ 跳过 Codex review
- ❌ 跳过人工确认
- ❌ 在 review 未通过时合并代码

---

### Claude 执行规范

#### 识别流程适用场景

**必须使用完整流程**：
- 规则复杂（数据治理 / 金融 / 合规）
- 需求可能反复调整
- 长期维护项目
- 用户明确要求"测试优先"

**可简化（但仍建议 Prompt → 最小测试 → 实现）**：
- 一次性脚本
- 明确、短生命周期任务
- 简单 Bug 修复

#### 强制执行规则

**阶段 0（Prompt）时**：
1. 与用户确认需求
2. 明确边界与排除项
3. **不主动写代码或测试**

**阶段 1（Test）时**：
1. 根据 Prompt 编写测试
2. 覆盖所有边界条件
3. **不写实现代码**
4. 测试允许失败（红灯）

**阶段 2（Test Review）时**：
1. **必须通过 codex-ccb** 调用 Codex 进行测试审查
2. 向用户展示 Codex 的反馈
3. **等待用户明确批准**后才能进入下一阶段
4. 如果 Codex 或用户发现问题，返回阶段 1 修复

**阶段 3（Implementation）时**：
1. 让测试通过（绿灯）
2. **不修改已通过的测试**
3. **不添加测试未覆盖的功能**

**阶段 4（Implementation Review）时**：
1. **必须通过 codex-ccb** 调用 Codex 进行代码审查
2. 向用户展示 Codex 的反馈
3. **等待用户明确批准**后才能合并代码
4. 如果 Codex 或用户发现问题，返回阶段 3 修复

#### 阶段切换确认

**进入 Test 阶段前**，询问用户：
> "Prompt 已明确，是否开始编写测试？"

**进入 Test Review 阶段前**：
1. Claude 通过 codex-ccb 调用 Codex，提交测试代码审查请求
2. 向用户展示：
   > "测试已完成，正在请求 Codex review..."
   >
   > [Codex 反馈内容]
   >
   > "是否批准进入 Implementation 阶段？"

**进入 Implementation 阶段前**，等待用户确认：
> "Test Review 通过，是否开始实现？"

**进入 Implementation Review 阶段前**：
1. Claude 通过 codex-ccb 调用 Codex，提交实现代码审查请求
2. 向用户展示：
   > "实现已完成，正在请求 Codex review..."
   >
   > [Codex 反馈内容]
   >
   > "是否批准合并代码？"

**进入 CI/CD 阶段前**，等待用户确认：
> "Implementation Review 通过，是否推送到远程仓库触发 CI？"

#### Git Worktree 使用（可选）

对于复杂项目，建议使用 Worktree 物理隔离：

| 阶段 | Worktree 路径 | 允许修改 |
|------|-------------|---------|
| Prompt | `prompt/*` | 文档 |
| Test | `test/*` | `tests/` |
| Implementation | `impl/*` | `src/` |

**使用时机**：
- 多人协作
- 并行开发多个功能
- 需要物理隔离风险

---

### 心智模型

```
Prompt 决定方向
Test 冻结决策
Review 守护质量（Codex + 人工）
Implementation 可随时推翻
CI/CD 自动验证
```

你不是在管理代码，而是在管理：
- **决策**（Prompt + Test）
- **质量**（Review 双重关卡）
- **风险**（测试覆盖的边界）
- **不确定性**（测试未覆盖的部分）

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

### CI/CD 操作

1. **推送前检查**: 在推送到远程仓库前，本地运行 CI 检查
   ```powershell
   pytest tests/ -v
   ruff check src/
   mypy src/
   ```
2. **CI 失败处理**: CI 失败时，修复问题而非绕过检查
3. **禁止绕过**: 禁止添加 `# type: ignore`、`# noqa` 等注释绕过检查
4. **新项目初始化**: 主动询问用户是否启用 CI，并配置 `.gitlab-ci.yml`
5. **CI 配置参考**: 详见 [development/ci.md](development/ci.md)

## 语言要求

- 代码注释: 中文
- 文档: 中文
- Git 提交信息: 中文
- 变量/函数命名: 英文 (遵循 Python 命名规范)

## 字符集与时区

### 字符集

**全局编码规范：UTF-8（无 BOM）**

所有场景必须使用 UTF-8 编码，包括但不限于：

| 场景 | 要求 |
|------|------|
| **源代码文件** | UTF-8（Python、JavaScript、配置文件等） |
| **脚本文件** | UTF-8（Shell、PowerShell、批处理等） |
| **文档文件** | UTF-8（Markdown、文本文件等） |
| **Git 仓库** | UTF-8（所有文本文件） |
| **终端输出** | UTF-8（确保终端配置支持 UTF-8） |
| **数据库** | `utf8mb4`（MySQL）、`UTF-8`（其他） |
| **API 通信** | UTF-8（JSON、XML、HTTP Header） |

**注意事项**：
- 所有 UTF-8 文件**不使用 BOM**（Byte Order Mark）
- IDE/编辑器默认编码设置为 UTF-8
- 终端环境变量设置：`$env:PYTHONIOENCODING="utf-8"`（Windows）

### 时区

| 场景 | 处理方式 |
|------|---------|
| 默认时区 | 东八区 (UTC+8, Asia/Shanghai) |
| 代码中 | **一般不显式声明**，依赖系统/环境配置 |
| 数据库存储 | 根据业务需求选择（本地时间或 UTC） |
| 问题排查 | 发现时间不一致时，才强制指定东八区 |

### 示例

```python
# ✅ 正常情况：不显式声明时区
from datetime import datetime
now = datetime.now()

# ✅ 时间不一致时：强制指定东八区
from datetime import datetime, timezone, timedelta
tz_shanghai = timezone(timedelta(hours=8))
now = datetime.now(tz_shanghai)

# 或使用 pytz/zoneinfo
from zoneinfo import ZoneInfo
now = datetime.now(ZoneInfo("Asia/Shanghai"))
```

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
