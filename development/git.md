# Git 使用规范

## 适用范围

本规范适用于所有使用 Git 进行版本控制的项目，同时作为 Claude 执行 Git 操作的行为指南。

---

## Claude 操作权限矩阵

### 危险等级说明

| 等级 | 标识 | 含义 | Claude 行为 |
|------|------|------|-------------|
| 🟢 安全 | SAFE | 只读或可撤销 | 直接执行，无需确认 |
| 🟡 谨慎 | CAUTION | 有副作用但可恢复 | 执行前简要说明 |
| 🔴 危险 | DANGER | 不可逆或影响他人 | **必须用户确认后执行** |
| ⛔ 禁止 | FORBIDDEN | 高风险操作 | **拒绝执行，解释原因** |

### 操作权限速查表

| 操作 | 危险等级 | 触发时机 | Claude 行为 |
|------|---------|---------|-------------|
| `git status` | 🟢 安全 | 任何时候了解仓库状态 | 直接执行 |
| `git diff` | 🟢 安全 | 查看改动内容 | 直接执行 |
| `git log` | 🟢 安全 | 查看提交历史 | 直接执行 |
| `git branch` | 🟢 安全 | 查看分支列表 | 直接执行 |
| `git fetch` | 🟢 安全 | 获取远程更新 | 直接执行 |
| `git stash` | 🟡 谨慎 | 临时保存工作进度 | 说明后执行 |
| `git add` | 🟡 谨慎 | 准备提交改动 | 说明后执行 |
| `git checkout -b` | 🟡 谨慎 | 开始新功能/修复 | 说明后执行 |
| `git commit` | 🟡 谨慎 | 完成一个逻辑单元 | **用户要求时执行** |
| `git pull` | 🟡 谨慎 | 同步远程更新 | 说明后执行 |
| `git merge` | 🔴 危险 | 合并分支 | **必须用户确认** |
| `git push` | 🔴 危险 | 推送到远程 | **必须用户确认** |
| `git reset --soft` | 🔴 危险 | 撤销提交保留改动 | **必须用户确认** |
| `git reset --hard` | ⛔ 禁止 | 丢弃所有改动 | **拒绝执行** |
| `git push --force` | ⛔ 禁止 | 强制推送 | **拒绝执行** |
| `git rebase` | ⛔ 禁止 | 变基操作 | **拒绝执行** |

---

## 操作触发时机详解

### 何时创建分支

```
触发条件 (满足任一即创建分支):
├── 用户明确要求开发新功能
├── 用户要求修复 Bug
├── 用户要求进行代码重构
├── 预计改动超过 3 个文件
└── 改动可能影响现有功能

不创建分支的情况:
├── 仅修改文档/注释
├── 修改单个配置文件
├── 用户明确说"直接改"
└── 当前已在功能分支上
```

**Claude 行为**:
```
IF 用户要求开发功能/修复问题:
    IF 当前在 main 分支 AND 改动较大:
        询问: "建议创建分支 feature/xxx 进行开发，是否同意？"
    ELSE IF 当前已在功能分支:
        直接在当前分支开发
```

### 何时提交 (Commit)

```
触发条件 (满足任一即可提交):
├── 完成一个独立的逻辑单元
│   ├── 一个函数/类实现完成
│   ├── 一个 Bug 修复完成
│   ├── 一个配置变更完成
│   └── 一组相关文件修改完成
├── 用户明确要求提交
├── 即将切换到其他任务
└── 工作告一段落需要保存

不立即提交的情况:
├── 代码未完成，存在语法错误
├── 测试未通过
├── 改动不完整，会破坏功能
└── 用户说"先不提交"
```

**Claude 行为**:
```
默认: 完成代码编写后，不主动提交

IF 用户明确要求提交 OR 用户说"提交一下":
    执行 git add + git commit
    提交信息遵循 Conventional Commits 规范

IF 用户确认变更效果符合要求 (识别以下信号):
    - "代码能跑通了"
    - "测试通过了"
    - "数据符合要求了"
    - "功能正常了"
    - "Bug 修复了"
    - "可以了"/"没问题了"
THEN:
    Claude 应主动建议: "变更已验证通过，建议提交并记录本次变更，是否进行？"
```

### 何时推送 (Push)

```
触发条件:
├── 用户明确要求推送
├── 用户要求创建 PR
├── 用户说"同步到远程"
└── 用户同意 Claude 的提交建议后

Claude 永远不会主动推送，必须用户确认
```

---

## 提交与变更日志流程

### 完整提交流程

当用户确认变更效果符合要求并同意提交时，Claude 应按以下流程执行：

```
1. git add .                              # 暂存改动
2. git commit -m "<type>(<scope>): <msg>" # 提交
3. git tag -a v<version> -m "<msg>"       # 打 tag
4. 更新 docs/upgrades/git-upgrades.md    # 记录变更日志
5. git add docs/upgrades/git-upgrades.md # 暂存变更日志
6. git commit -m "docs: 更新变更日志"     # 提交变更日志
7. (用户确认后) git push && git push --tags # 推送
```

### Tag 版本规范

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

### 变更日志位置

```
项目根目录/docs/upgrades/git-upgrades.md
```

### Claude 提交建议话术

```
当用户确认变更有效时，Claude 应说:

"变更已验证通过，建议进行以下操作：
1. 提交本次改动：<type>(<scope>): <描述>
2. 打 tag：v<version>
3. 更新变更日志

是否执行？"
```

### 何时拉取 (Pull)

```
触发条件:
├── 开始工作前同步最新代码
├── 用户要求更新代码
├── 准备合并分支前
└── 提示远程有更新时

Claude 行为: 说明后执行，如有冲突则报告用户
```

---

## 分支规范

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
fix/null-pointer-exception
hotfix/data-loss-prevention
refactor/llm-prompt-engine
docs/api-specification

# ❌ 错误
new-feature              # 缺少类型前缀
Feature/UserAuth         # 不要用大写
fix_bug                  # 用 / 不用 _
feature/add_user         # 描述部分用 - 不用 _
```

### 分支生命周期

```
main (保护分支，始终可部署)
  │
  ├── feature/user-login ──────┬── 开发完成 ──→ PR ──→ 合并 ──→ 删除分支
  │                            │
  ├── fix/mysql-timeout ───────┤
  │                            │
  └── hotfix/critical-bug ─────┘
```

---

## 提交规范

### Conventional Commits 格式

```
<type>(<scope>): <subject>

[body]

[footer]
```

### Type 类型选择指南

| Type | 使用场景 | 示例 |
|------|---------|------|
| `feat` | 新增功能、新增文件 | feat(api): 添加用户登录接口 |
| `fix` | 修复 Bug | fix(mysql): 修复连接泄漏问题 |
| `docs` | 仅文档变更 | docs: 更新 README |
| `style` | 格式调整(不影响逻辑) | style: 格式化代码 |
| `refactor` | 重构(不改变功能) | refactor(llm): 重构调用逻辑 |
| `perf` | 性能优化 | perf(query): 优化查询性能 |
| `test` | 测试相关 | test: 添加单元测试 |
| `chore` | 构建/依赖/配置 | chore: 升级依赖版本 |
| `revert` | 回滚提交 | revert: 回滚 feat(xxx) |

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

### 提交示例

```bash
# ✅ 正确示例
feat(mysql): 添加批量插入功能
fix(redis): 修复 Key 过期回调失效问题
docs: 补充 MongoDB 索引设计说明
refactor(llm): 统一不同模型的调用接口
perf(query): 优化大数据量分页查询
chore: 升级 FastAPI 到 0.110 版本

# ❌ 错误示例
添加了新功能               # 缺少 type
feat: Add new feature    # 应使用中文
feat(mysql): 添加了xxx。  # 不要用句号和"了"
FEAT(MYSQL): XXX         # 不要全大写
update                   # 太笼统
fix bug                  # 不具体
```

### Body 正文 (较大改动时使用)

```bash
fix(mysql): 修复连接池泄漏问题

问题原因：异常情况下连接未正确归还到连接池
解决方案：在 finally 块中确保连接释放
影响范围：所有使用 MySQL 连接的模块
```

---

## Claude Git 操作行为规范

### 提交前检查清单

Claude 在执行 `git commit` 前必须确认:

```
□ 不包含敏感信息 (密钥/密码/token)
□ 不包含调试代码 (print/console.log)
□ 不包含 IDE 配置文件
□ 提交信息符合 Conventional Commits 格式
□ 改动与提交信息描述一致
```

### 冲突处理

```
IF 执行 git pull 出现冲突:
    1. 停止操作
    2. 报告用户: "拉取时发现冲突，涉及以下文件: [文件列表]"
    3. 询问: "是否需要我协助解决冲突？"
    4. 等待用户指示
```

### 敏感文件检测

```
IF 暂存区包含以下文件:
    - .env / .env.*
    - *credentials*
    - *secret*
    - *.pem / *.key
    - config/**/prod*

THEN:
    警告用户: "检测到可能包含敏感信息的文件，确认要提交吗？"
    列出具体文件名
    等待用户确认
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
```

---

## 常用命令参考

### 🟢 安全操作 (直接执行)

```bash
git status                    # 查看状态
git diff                      # 查看改动
git diff --staged             # 查看已暂存改动
git log --oneline -10         # 查看最近10条提交
git branch                    # 查看本地分支
git branch -a                 # 查看所有分支
git fetch                     # 获取远程更新
```

### 🟡 谨慎操作 (说明后执行)

```bash
git add <file>                # 暂存指定文件
git add .                     # 暂存所有改动
git checkout -b <branch>      # 创建并切换分支
git switch -c <branch>        # 创建并切换分支 (新语法)
git stash                     # 暂存当前工作
git stash pop                 # 恢复暂存的工作
git pull                      # 拉取并合并
```

### 🔴 危险操作 (必须用户确认)

```bash
git commit -m "message"       # 提交 (用户要求时)
git push                      # 推送到远程
git push -u origin <branch>   # 推送新分支
git merge <branch>            # 合并分支
git branch -d <branch>        # 删除已合并分支
git reset --soft HEAD~1       # 撤销最近提交(保留改动)
```

### ⛔ 禁止操作 (拒绝执行)

```bash
git push --force              # 强制推送
git push -f                   # 强制推送
git reset --hard              # 硬重置(丢失改动)
git clean -fd                 # 删除未跟踪文件
git rebase                    # 变基操作
git commit --amend            # 修改已推送的提交
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

### 推送操作
- [ ] 推送前确认分支正确
- [ ] 推送前确认无敏感信息
- [ ] 新分支使用 `-u` 设置上游
