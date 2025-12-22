# Git 基础规范

本文档定义 Git 分支、提交、Tag 等基础规范。操作指南见 [git-operations.md](git-operations.md)，Claude 行为规范见 [git-claude.md](git-claude.md)。

---

## 项目命名规范

### 仓库/项目名称

Git 仓库和项目名称必须遵循以下规范：

```
格式: 全小写，使用短横线 (-) 分隔单词 (kebab-case)
编码: UTF-8
```

**示例**:
```bash
# ✅ 正确
ai-workbench-core
data-processor
user-service
llm-integration
mysql-connector

# ❌ 错误
AIWorkbench          # 不要用大写
ai_workbench         # 用 - 不用 _
aiworkbench          # 多个单词需要分隔符
AI-Workbench-Core    # 不要用大写
```

---

## 分支规范

### 分支模型

本项目采用 **简化主干开发模式**：

```
main (主干，保护分支，始终可部署)
  │
  ├── feature/*  ─── 新功能开发 ─── PR ─── 合并 ─── 删除
  ├── fix/*      ─── Bug 修复
  ├── hotfix/*   ─── 生产紧急修复 (从 tag 切出)
  └── refactor/* ─── 代码重构
```

**分支保护规则**：
- `main` 分支：禁止直推，必须通过 PR 合并
- 功能分支：合并后删除

### 分支命名

```
格式: <type>/<short-description>
全小写，使用连字符分隔单词

type 类型:
├── feature/    新功能开发
├── fix/        Bug 修复
├── hotfix/     生产环境紧急修复
├── refactor/   代码重构
├── docs/       文档更新
├── test/       测试相关
└── chore/      构建/依赖/工具
```

**示例**:
```bash
# ✅ 正确
feature/user-authentication
feature/add-mysql-connector
fix/redis-connection-timeout
hotfix/data-loss-prevention
refactor/llm-prompt-engine

# ❌ 错误
new-feature              # 缺少类型前缀
Feature/UserAuth         # 不要用大写
fix_bug                  # 用 / 不用 _
feature/add_user         # 描述部分用 - 不用 _
```

### 分支生命周期

```
1. 从 main 创建分支
   git checkout -b feature/xxx main

2. 开发完成后推送
   git push -u origin feature/xxx

3. 创建 PR，代码审查

4. 合并到 main (squash 或 merge commit)

5. 删除功能分支
   git branch -d feature/xxx
   git push origin --delete feature/xxx
```

---

## 提交规范

### Conventional Commits 格式

```
<type>(<scope>): <subject>

[body]

[footer]
```

### Type 类型

| Type | 使用场景 | 示例 |
|------|---------|------|
| `feat` | 新增功能 | feat(api): 添加用户登录接口 |
| `fix` | 修复 Bug | fix(mysql): 修复连接泄漏问题 |
| `docs` | 仅文档变更 | docs: 更新 README |
| `style` | 格式调整(不影响逻辑) | style: 格式化代码 |
| `refactor` | 重构(不改变功能) | refactor(llm): 重构调用逻辑 |
| `perf` | 性能优化 | perf(query): 优化查询性能 |
| `test` | 测试相关 | test: 添加单元测试 |
| `chore` | 构建/依赖/配置 | chore: 升级依赖版本 |
| `revert` | 回滚提交 | revert: 回滚 feat(xxx) |

### 提交类型与版本递增映射

将 Conventional Commits 类型映射到 SemVer 版本递增（基于项目实际变更影响）：

| 变更类型 | 定义 | 版本递增 | Commit Type | 示例 |
|---------|------|---------|------------|------|
| **MAJOR** | 存在行为变化 / 破坏性调整 | v1.0.0 → v2.0.0 | 任何 type + `!` 或 `BREAKING CHANGE` | `feat!:` / `refactor!:` |
| **MINOR** | 新增能力，不破坏既有行为 | v1.0.0 → v1.1.0 | `feat` / `perf` | `feat(api): 添加新接口` |
| **PATCH** | 文档 / 内部调整 | v1.0.0 → v1.0.1 | `fix` / `docs` / `style` / `test` / `chore` | `fix: 修复 bug` |

**判断标准**：

```
MAJOR (破坏性变更)：
├── 不兼容的 API 修改
├── 删除已有功能
├── 修改默认行为
├── 工作流程重大调整
└── 需要用户修改代码才能适配

MINOR (新功能)：
├── 新增功能、接口、配置项
├── 性能优化（不改变行为）
├── 向下兼容的功能增强
└── 用户无需修改代码即可使用

PATCH (修复/文档)：
├── Bug 修复
├── 文档更新
├── 代码格式调整
├── 测试补充
└── 构建工具更新
```

**提交时必须标注变更类型**（在提交信息中注明）：

```bash
# MAJOR 变更（必须使用 ! 或 BREAKING CHANGE）
feat!: 重构工作流程规范

变更类型：MAJOR（存在行为变化 / 破坏性调整）

BREAKING CHANGE: 工作流程从三阶段升级为五阶段模型
...

# MINOR 变更
feat(api): 添加数据导出功能

变更类型：MINOR（新增能力，不破坏既有行为）
...

# PATCH 变更
docs: 更新 MySQL 规范

变更类型：PATCH（文档调整）
...
```

### Scope 范围 (可选)

```
按模块划分:
├── connector   # 连接器相关
├── mysql       # MySQL 相关
├── mongodb     # MongoDB 相关
├── redis       # Redis 相关
├── milvus      # Milvus 相关
├── dameng      # 达梦相关
├── llm         # LLM 相关
├── api         # API 相关
├── config      # 配置相关
└── core        # 核心模块
```

### Subject 主题规范

- ✅ 使用中文
- ✅ 不超过 50 个字符
- ✅ 使用祈使语气 (添加/修复/更新/删除/重构)
- ❌ 不以句号结尾
- ❌ 不使用"了" (添加 ✓ / 添加了 ✗)

### Body 正文 (较大改动时使用)

```bash
fix(mysql): 修复连接池泄漏问题

问题原因：异常情况下连接未正确归还到连接池
解决方案：在 finally 块中确保连接释放
影响范围：所有使用 MySQL 连接的模块
```

### 破坏性变更 (BREAKING CHANGE)

当提交包含不兼容的 API 变更时：

**方式一：在 type 后加 `!`**
```bash
feat(api)!: 修改用户接口返回格式

BREAKING CHANGE: 用户列表接口返回结构从数组改为分页对象
迁移方法：将 response 改为 response.data
```

**方式二：在 footer 中说明**
```bash
refactor(connector): 重构数据库连接器接口

BREAKING CHANGE: ConnectorBase 类方法签名变更
- connect() 改为 async connect()
- 移除 sync_connect() 方法
```

### 关联 Issue/任务

在 footer 中使用以下格式：

```bash
feat(api): 添加数据导出功能

实现批量数据导出为 CSV/Excel 格式

Refs: #123              # 关联 issue
Closes: #456            # 关闭 issue
```

### 提交示例

```bash
# ✅ 正确示例
feat(mysql): 添加批量插入功能
fix(redis): 修复 Key 过期回调失效问题
docs: 补充 MongoDB 索引设计说明

# ❌ 错误示例
添加了新功能               # 缺少 type
feat: Add new feature    # 应使用中文
feat(mysql): 添加了xxx。  # 不要用句号和"了"
update                   # 太笼统
```

---

## Tag 版本规范

### 版本号格式 (SemVer)

```
格式: v<major>.<minor>.<patch>

版本号递增规则:
├── major: 重大变更，不兼容的 API 修改
├── minor: 新功能，向下兼容
└── patch: Bug 修复，向下兼容

示例:
├── v1.0.0 → v1.0.1  # Bug 修复
├── v1.0.1 → v1.1.0  # 新功能
└── v1.1.0 → v2.0.0  # 重大变更
```

### 预发布标签

```
v1.0.0-alpha.1   - 内测版
v1.0.0-beta.1    - 公测版
v1.0.0-rc.1      - 发布候选
v1.0.0           - 正式版
```

### Tag 管理策略

#### 个人/中小型项目

```
├── 打 tag 时机: 功能完成可用时，不必每次提交都打
├── 版本粒度: 粗粒度，v1.0 → v1.1 → v2.0
├── tag 类型: 轻量 tag 即可 (git tag v1.0)
├── 清理策略: 可选，一般不用管
└── 变更日志: 可选，README 记录即可
```

#### 公司/大型项目

```
├── 打 tag 时机: 仅在正式发布时，与 Release 流程绑定
├── 版本粒度: 严格 SemVer (v1.2.3)
├── tag 类型: Annotated tag + 签名 (git tag -s)
├── 清理策略: **永不删除**，审计需要
└── 变更日志: **必须**，CHANGELOG.md 或 GitHub Releases
```

### 打 Tag 时机判断

```
何时应该打 tag:
├── 功能开发完成，可交付使用
├── Bug 修复后，版本稳定
├── 重要里程碑节点
└── 准备发布/部署时

何时不需要打 tag:
├── 日常小修小补 (typo、注释、格式)
├── 开发中间状态
├── 实验性改动
└── 仅文档更新 (除非是重要版本文档)
```

### 本项目 Tag 规则

```
本规范文档库采用: 个人项目策略

├── 版本格式: vX.Y.Z
├── 打 tag 时机: 新增/重大修改规范文档时
├── 日常小修订: 只提交不打 tag
├── 变更日志: docs/upgrades/git-upgrades.md
└── 清理策略: 不清理
```

---

## PR 规范

### PR 标题

与 Commit Message 格式一致

### PR 描述模板

```markdown
## 变更说明
简述本次 PR 的目的和改动内容

## 变更类型
- [ ] 新功能 (feat)
- [ ] Bug 修复 (fix)
- [ ] 重构 (refactor)
- [ ] 文档 (docs)
- [ ] 其他

## 测试情况
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 手动测试通过

## 检查清单
- [ ] 代码符合项目规范
- [ ] 已更新相关文档
- [ ] 无敏感信息泄露

## 关联 Issue
Closes #xxx
```

---

## .gitignore 必须包含

```gitignore
# Python
__pycache__/
*.py[cod]
.venv/
.env
.env.*

# IDE
.idea/
.vscode/
*.swp
*.swo

# 敏感文件
*.pem
*.key
credentials.json
secrets.yaml
**/secrets/

# 日志和临时文件
*.log
logs/
tmp/
.cache/

# 系统文件
.DS_Store
Thumbs.db
```

---

## 检查清单

### 分支操作
- [ ] 功能开发前从 main 创建分支
- [ ] 分支命名符合 `<type>/<description>` 格式
- [ ] 分支描述使用小写和连字符

### 提交操作
- [ ] 提交信息符合 Conventional Commits 格式
- [ ] 每次提交是一个完整的逻辑单元
- [ ] 不包含敏感信息
- [ ] 不包含调试代码
- [ ] 破坏性变更使用 `!` 或 `BREAKING CHANGE`

### 推送操作
- [ ] 推送前确认分支正确
- [ ] 推送前确认无敏感信息
- [ ] 新分支使用 `-u` 设置上游
