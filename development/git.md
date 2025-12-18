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

## Claude 与 Codex 分工

```
┌─────────────────────────────────────────────────────────────┐
│                        Claude 职责                          │
├─────────────────────────────────────────────────────────────┤
│ ✅ 代码编写与修改                                           │
│ ✅ 与用户交互、理解需求                                      │
│ ✅ Git 操作决策与执行                                        │
│ ✅ 识别操作时机并建议用户                                    │
│ ✅ 提交信息编写 (简短)                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        Codex 职责                           │
├─────────────────────────────────────────────────────────────┤
│ ✅ 变更日志编写 (docs/upgrades/git-upgrades.md)             │
│ ✅ 大量文本输出任务                                          │
│ ✅ 代码审查与建议                                            │
│ ✅ 复杂分析报告生成                                          │
└─────────────────────────────────────────────────────────────┘
```

**提交流程分工**:
```
用户确认变更有效 → Claude 建议提交 → 用户同意 →
├── Claude: git add + git commit + git tag
├── Codex: 编写变更日志到 docs/upgrades/git-upgrades.md
├── Claude: git add 变更日志 + git commit
└── Claude: (用户确认后) git push
```

---

## 操作触发时机详解

### 何时创建分支

**用户不会直接说"创建分支"，Claude 需要从以下信号识别**:

```
用户可能说的话 → Claude 识别 → 建议操作

"帮我开发一个 xxx 功能"        → 新功能开发    → feature/xxx
"这个地方有 bug，帮我修一下"   → Bug 修复     → fix/xxx
"帮我重构一下这个模块"         → 代码重构     → refactor/xxx
"性能太差了，优化一下"         → 性能优化     → perf/xxx
"添加一个 xxx 的接口"          → 新功能开发    → feature/xxx
"这个逻辑不对，改一下"         → Bug 修复     → fix/xxx
"数据处理流程需要调整"         → 功能变更     → feature/xxx
```

**Claude 决策逻辑**:
```
IF 用户描述的任务涉及:
    - 新增功能/接口/模块
    - 修复问题/Bug
    - 重构/优化
    - 预计改动 >= 3 个文件
    - 可能影响现有功能
THEN:
    1. 检查当前分支: git branch --show-current
    2. IF 当前在 main 分支:
        建议: "这个任务建议在新分支上进行，我来创建 feature/xxx 分支？"
    3. IF 当前已在功能分支:
        继续在当前分支开发

不建议创建分支的情况:
├── 仅修改文档/注释/README
├── 修改单个配置项
├── 用户说"直接改"/"小改动"
├── 当前已在对应功能分支上
└── 用户明确拒绝分支建议
```

**Claude 建议话术**:
```
场景1 - 新功能:
"这个功能开发建议在新分支进行，我来创建 feature/user-export 分支？"

场景2 - Bug修复:
"修复这个问题建议在单独分支进行，创建 fix/data-sync-error 分支？"

场景3 - 用户拒绝后:
用户: "不用，直接改"
Claude: "好的，直接在 main 分支修改。" (继续执行)
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

### Tag 管理策略

#### 个人/中小型项目

```
特点: 迭代快、发布灵活、维护成本敏感

├── 打 tag 时机: 功能完成可用时，不必每次提交都打
├── 版本粒度: 粗粒度，v1.0 → v1.1 → v2.0
├── tag 类型: 轻量 tag 即可 (git tag v1.0)
├── 清理策略: 可选，一般不用管
└── 变更日志: 可选，README 记录即可

版本号建议:
├── v0.x.x  - 开发阶段，随意迭代
├── v1.0    - 首个稳定版
├── v1.1    - 新功能
└── v2.0    - 大改动/不兼容
```

#### 公司/大型项目

```
特点: 多人协作、需要审计追溯、与 CI/CD 集成

├── 打 tag 时机: 仅在正式发布时，与 Release 流程绑定
├── 版本粒度: 严格 SemVer (v1.2.3)，每个 patch 都记录
├── tag 类型: Annotated tag + 签名 (git tag -s)
├── 清理策略: **永不删除**，审计需要
└── 变更日志: **必须**，CHANGELOG.md 或 GitHub Releases

预发布标签:
├── v1.0.0-alpha.1   - 内测版
├── v1.0.0-beta.1    - 公测版
├── v1.0.0-rc.1      - 发布候选
└── v1.0.0           - 正式版

CI/CD 集成:
push tag → 触发构建 → 自动发布 → 生成 Release Notes
```

#### 打 Tag 时机判断

```
何时应该打 tag:
├── 功能开发完成，可交付使用
├── Bug 修复后，版本稳定
├── 重要里程碑节点
├── 准备发布/部署时
└── 需要标记可回溯的状态

何时不需要打 tag:
├── 日常小修小补 (typo、注释、格式)
├── 开发中间状态
├── 实验性改动
└── 仅文档更新 (除非是重要版本文档)
```

#### Tag 清理策略

```
⚠️ Tag 通常不清理，原因:
├── 代表版本发布的历史快照
├── 支持回溯特定版本代码
├── 追踪问题引入时间点
└── 审计和合规需要

过多 tag 的影响 (实际很小):
├── 技术层面:
│   ├── 存储空间: 几乎无影响 (每个约 41 字节)
│   ├── 网络传输: git fetch --tags 稍慢
│   └── 查询性能: 几万个 tag 时可能变慢
│
└── 管理层面 (主要问题):
    ├── 查找困难: 难以快速定位目标版本
    ├── 语义稀释: tag 失去"重要版本"标记意义
    └── 认知负担: 团队难以理解每个 tag

如需清理 (谨慎操作):
├── 仅清理未推送的本地 tag
├── 保留所有 major/minor 版本
├── 与团队沟通后再删除远程 tag
└── 删除前记录到文档备案
```

#### 本项目 Tag 规则

```
本规范文档库采用: 个人项目策略

├── 版本格式: vX.Y.Z
├── 打 tag 时机: 新增/重大修改规范文档时
├── 日常小修订: 只提交不打 tag
├── 变更日志: docs/upgrades/git-upgrades.md
└── 清理策略: 不清理
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

## 进阶场景操作指南

### 场景1: 撤销与回退

**典型触发条件**:
```
用户可能说:
├── "刚刚提交错了"
├── "commit message 写错了"
├── "把敏感信息提交进去了"
├── "需要回到某个稳定版本"
├── "这个提交不该进来，想撤销"
└── "我想丢掉最近几次提交"
```

**操作决策树**:
```
是否已推送到远程?
├── 未推送:
│   ├── 只改提交信息 → git commit --amend (🟡 谨慎)
│   ├── 撤销提交但保留改动 → git reset --soft HEAD~1 (🟡 谨慎)
│   └── 完全丢弃提交和改动 → git reset --hard HEAD~1 (🔴 危险)
│
└── 已推送:
    ├── 共享分支 (main/develop) → git revert <sha> (🟡 谨慎)
    └── 个人分支 → git reset + git push --force-with-lease (🔴 危险)
```

**Claude 行为**:
| 操作 | 危险等级 | 行为 |
|------|---------|------|
| `git revert` | 🟡 谨慎 | 确认 SHA 后执行 |
| `git commit --amend` (未推送) | 🟡 谨慎 | 确认未推送后执行 |
| `git reset --soft/mixed` | 🔴 危险 | **必须用户确认** |
| `git reset --hard` | ⛔ 禁止 | **拒绝执行，建议替代方案** |
| 敏感信息清除 | ⛔ 禁止 | **拒绝自动处理，提示安全流程** |

**⚠️ 敏感信息泄露处理**:
```
IF 用户说"提交了密码/密钥/token":
    1. 立即停止，不要继续推送
    2. 告知用户: "git revert 只能撤销内容，无法从历史中抹除"
    3. 建议: "请立即轮换泄露的密钥，并联系安全团队处理历史清除"
    4. Claude 不执行历史重写操作
```

---

### 场景2: 分支合并

**典型触发条件**:
```
用户可能说:
├── "功能开发好了要合并"
├── "PR 冲突了，帮我解决"
├── "把主干最新改动合到我的分支"
└── "需要保持线性历史"
```

**操作决策树**:
```
合并方向?
├── 主干 → 功能分支 (同步更新):
│   ├── 保守方式 → git fetch && git merge origin/main (🟡 谨慎)
│   └── 线性方式 → git fetch && git rebase origin/main (🔴 危险)
│
└── 功能分支 → 主干 (完成合并):
    ├── 推荐: 通过 PR/MR 合并 (🟢 安全)
    └── 本地合并 (不推荐): git merge --no-ff (🔴 危险)
```

**冲突处理流程**:
```bash
# 1. 查看冲突文件
git status

# 2. 编辑解决冲突 (手动或使用工具)
# 冲突标记:
# <<<<<<< HEAD
# 当前分支代码
# =======
# 合并分支代码
# >>>>>>> branch-name

# 3. 标记已解决
git add <冲突文件>

# 4. 继续合并/变基
git merge --continue
# 或
git rebase --continue
```

**Claude 行为**:
| 操作 | 危险等级 | 行为 |
|------|---------|------|
| `git fetch` | 🟢 安全 | 直接执行 |
| `git merge origin/main` (到功能分支) | 🟡 谨慎 | 说明后执行 |
| `git merge` (到主干) | 🔴 危险 | **必须用户确认，建议使用 PR** |
| `git rebase` | ⛔ 禁止 | **拒绝执行** |
| 冲突解决 | 🟡 谨慎 | 报告冲突文件，询问是否协助 |

---

### 场景3: 暂存与恢复 (Stash)

**典型触发条件**:
```
用户可能说:
├── "我得切到别的分支修 bug，但现在改了一半"
├── "先把当前工作保存起来"
├── "拉取前提示有本地修改冲突"
└── "恢复之前保存的工作"
```

**操作流程**:
```bash
# 保存当前工作 (推荐带 -u 包含未跟踪文件)
git stash push -u -m "wip: 功能描述"

# 查看暂存列表
git stash list

# 查看暂存内容
git stash show -p stash@{0}

# 恢复工作 (推荐先 apply 再 drop)
git stash apply stash@{0}   # 应用但保留记录
git stash drop stash@{0}    # 确认无误后删除

# 或一步完成 (有风险)
git stash pop               # 应用并删除
```

**Claude 行为**:
| 操作 | 危险等级 | 行为 |
|------|---------|------|
| `git stash push -u -m "..."` | 🟡 谨慎 | 说明后执行 |
| `git stash list/show` | 🟢 安全 | 直接执行 |
| `git stash apply` | 🟡 谨慎 | 说明后执行 |
| `git stash pop` | 🟡 谨慎 | **询问确认** (冲突时记录会丢失) |
| `git stash drop/clear` | 🔴 危险 | **必须用户确认** |

**⚠️ 注意事项**:
- `stash pop` 冲突后记录已删除，建议先 `apply` 确认无误再 `drop`
- 忘记 `-u` 会导致未跟踪文件丢失
- 重要进度推荐 WIP 提交而非 stash

---

### 场景4: 同步与更新

**典型触发条件**:
```
用户可能说:
├── "我本地落后了/需要同步最新代码"
├── "pull 的时候提示 diverged"
├── "push 被拒绝了"
└── "需要跟主干对齐"
```

**安全同步流程**:
```bash
# 1. 获取远程更新 (清理已删除的远程分支)
git fetch --prune

# 2. 查看本地与远程差异
git log --oneline HEAD...origin/main

# 3. 安全更新 (仅快进合并)
git pull --ff-only

# 如果失败 (出现分叉)，选择策略:
# 保守: git merge origin/main
# 线性: git rebase origin/main (谨慎)
```

**push 被拒绝处理**:
```
push 被拒 (non-fast-forward)?
├── 原因: 本地落后于远程
├── 解决:
│   ├── git fetch
│   ├── git merge origin/<branch>
│   ├── 解决冲突 (如有)
│   └── git push
└── 禁止: git push --force (除非确认是个人分支)
```

**Claude 行为**:
| 操作 | 危险等级 | 行为 |
|------|---------|------|
| `git fetch --prune` | 🟢 安全 | 直接执行 |
| `git pull --ff-only` | 🟢 安全 | 直接执行 |
| `git pull` (可能产生 merge) | 🟡 谨慎 | 说明后执行 |
| `git reset --hard origin/xxx` | ⛔ 禁止 | **拒绝执行** |

---

### 场景5: 查看与对比

**典型触发条件**:
```
用户可能说:
├── "这个改动是谁改的/什么时候改的"
├── "对比两个版本差异"
├── "查看某个文件的历史"
├── "找出引入 bug 的提交"
└── "看看这次改动会影响哪些文件"
```

**常用查看命令**:
```bash
# 查看提交历史
git log --oneline --graph -20

# 查看某文件历史
git log --oneline -- <file>
git log -p -- <file>           # 包含改动内容

# 对比差异
git diff                       # 工作区 vs 暂存区
git diff --staged              # 暂存区 vs HEAD
git diff main..feature/xxx     # 两个分支差异
git diff HEAD~3..HEAD          # 最近 3 次提交差异

# 追溯某行代码
git blame <file>
git blame -w <file>            # 忽略空白变化

# 查找引入 bug 的提交 (二分查找)
git bisect start
git bisect bad                 # 当前版本有 bug
git bisect good <sha>          # 某个好的版本
# ... 测试并标记 good/bad ...
git bisect reset               # 结束
```

**Claude 行为**: 所有查看操作均为 🟢 安全，可直接执行

---

### 场景6: 清理与维护

**典型触发条件**:
```
用户可能说:
├── "这个分支合并完了想删掉"
├── "本地有一堆没用的分支"
├── "工作区很脏，想清理"
└── "误生成了很多临时文件"
```

**分支清理**:
```bash
# 查看已合并的分支
git branch --merged main

# 删除本地分支 (已合并)
git branch -d feature/xxx

# 强制删除本地分支 (未合并，谨慎!)
git branch -D feature/xxx

# 删除远程分支
git push origin --delete feature/xxx

# 清理远程追踪引用
git fetch --prune
```

**工作区清理**:
```bash
# ⚠️ 危险操作！先预览
git clean -n                   # 预览将删除的文件
git clean -nd                  # 预览将删除的文件和目录

# 确认后执行
git clean -f                   # 删除未跟踪文件
git clean -fd                  # 删除未跟踪文件和目录
git clean -fdx                 # 连 .gitignore 中的也删 (极危险!)
```

**Claude 行为**:
| 操作 | 危险等级 | 行为 |
|------|---------|------|
| `git branch --merged` | 🟢 安全 | 直接执行 |
| `git fetch --prune` | 🟢 安全 | 直接执行 |
| `git branch -d` (已合并) | 🟡 谨慎 | 确认后执行 |
| `git branch -D` | 🔴 危险 | **必须用户确认** |
| `git push origin --delete` | 🔴 危险 | **必须用户确认** |
| `git clean -n` (预览) | 🟢 安全 | 直接执行 |
| `git clean -f/-fd` | ⛔ 禁止 | **拒绝直接执行，先 -n 预览** |
| `git clean -fdx` | ⛔ 禁止 | **拒绝执行** |

---

## Claude 决策问句模板

当遇到需要确认的场景时，Claude 应使用以下问句模板:

```
1. 确认推送状态:
   "这个提交/分支是否已经推送到远程？"

2. 确认分支归属:
   "这是个人分支还是共享分支 (main/develop)？"

3. 确认改写历史:
   "是否允许改写历史 (会影响已推送的提交)？"

4. 确认丢弃改动:
   "是否允许丢弃本地未提交的改动？要不要先 stash 保存？"

5. 确认合并策略:
   "使用哪种合并策略: merge commit / squash / rebase？"

6. 确认撤销方式:
   "撤销是要保留历史 (revert) 还是回到过去状态 (reset)？"

7. 确认删除操作:
   "确认要删除分支 xxx 吗？此操作不可恢复。"

8. 确认清理范围:
   "以下文件将被删除，确认吗？[文件列表]"
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
