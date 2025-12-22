# AI 驱动的工作流程规范

> **目标**：在 AI 时代构建一套**可审计、可回滚、可并行、可维护**的软件工程流程。

本文档是 [CLAUDE.md](../CLAUDE.md) 中"工作流程"章节的详细说明。

---

## 一、背景与问题

### AI 开发的双刃剑

随着 Claude Code / Copilot / GPT 等工具进入日常开发：

**优势**：
- AI 可以**极快地产生大量代码**
- 提高开发效率数倍

**风险**：
- 人类没有足够上下文审查每一行
- 事后才发现"逻辑假设错了"
- 回滚成本极高
- 代码演化为不可维护的"石山代码"

### 传统流程的失效

传统的开发流程：

```
需求 → 实现 →（可能有）测试 →（可能有）Review
```

在 AI 场景下会**放大风险**，因为：
- AI 生成速度太快，人类来不及思考
- 测试滞后，问题发现时已经积累大量代码
- 需求理解偏差被快速固化到代码中
- Review 滞后，低质量代码已经进入主干

---

## 二、核心思想（结论先行）

### 1️⃣ 测试不是验证实现，而是**定义行为**

```
传统思维：实现 → 测试验证
新思维：  测试定义 → 实现满足
```

**测试的新定位**：
- 测试 = 机器可执行的需求
- 测试 = 决策冻结点
- 实现 = 可替换的技术细节

### 2️⃣ Prompt 是第一等工程资产

```
Prompt ≈ 需求文档 + 决策记录
```

**为什么重要**：
- Prompt 不可丢弃、不可只存在于对话框
- Prompt 是测试的来源
- Prompt 是决策的审计追踪

### 3️⃣ Review 是强制质量关卡

```
Codex Review + 人工 Review = 双重保险
```

**为什么重要**：
- Codex 发现技术问题（代码质量、性能、安全）
- 人工 Review 确认业务逻辑正确
- 防止 AI 生成的代码绕过质量标准

### 4️⃣ 流程必须"反 AI 直觉"

> 人类 & AI 都**很想直接写实现**

所以流程必须**强制顺序**：

```
Prompt → Test → Review → Implementation → Review → CI/CD
```

**禁止跳过任何阶段。**

---

## 三、五阶段模型详解

---

## 阶段 0：Prompt（人类主导）

### 目的

- 明确"我们要什么"
- 明确"什么不算对"
- 明确隐含假设

### Prompt 应包含的内容

| 必须项 | 说明 | 示例 |
|--------|------|------|
| **业务目标（Why）** | 为什么做这个功能 | "支持企业多市场上市标记" |
| **行为定义（What）** | 期望的行为是什么 | "支持同时标记 HK / SH / SZ" |
| **边界条件** | 什么情况下适用 | "港股 ≠ A 股" |
| **排除项** | 什么不做 | "不包含退市逻辑" |
| **示例** | 正例 / 反例 | "正例：HK+SH 双重上市" |
| **Out of Scope** | 暂不关心的内容 | "历史变更追溯" |

### 使用 Prompt 模板

复制 [../docs/templates/prompt-template.md](../docs/templates/prompt-template.md) 快速创建 Prompt。

### 产出物

- **文本 Prompt**（存档到 `docs/prompts/` 或 Git）
- **不写代码、不写测试**

---

## 阶段 1：Test（AI 主导，强约束）

### 目的

- 将 Prompt 转化为 **可执行规范**
- 明确所有边界与反例

### 测试的地位

> **Tests are the source of truth.**

**测试即规范**：
- 测试失败 = 行为不符合约定
- 测试通过 = 行为符合需求
- 测试 ≠ 代码覆盖率工具

### 测试应覆盖

| 类型 | 说明 | 示例 |
|------|------|------|
| **正常路径** | Happy Path | 标准输入 → 预期输出 |
| **边界条件** | 临界值 | 空列表、单个市场、多个市场 |
| **反例** | 明确禁止的情况 | 港股不等于 A 股 |
| **不变量** | 长期必须成立的规则 | 标签不能重复 |

### 规则（必须遵守）

- ❌ **不允许写实现**
- ❌ **不允许假设实现细节**
- ✅ **可以写失败的测试**（红灯状态）

---

## 阶段 2：Test Review（Codex + 人工，强制）

### 目的

- **Codex**：检查测试的完整性、正确性
- **人工**：确认测试符合业务需求

### Review 流程

#### 第一步：Codex Review（自动）

**Claude 的操作**：
1. 完成测试编写后，通过 **codex-ccb** 技能调用 Codex
2. 提交审查请求：
   ```
   请 review 以下测试代码：
   - 测试覆盖是否完整？
   - 边界条件是否遗漏？
   - 测试逻辑是否正确？
   - 是否有冗余测试？

   [测试代码内容]
   ```

**Codex 的检查重点**：
- ✅ 测试覆盖所有 Prompt 中的功能点
- ✅ 边界条件（空值、极值、特殊情况）
- ✅ 反例（明确禁止的行为）
- ✅ 测试断言是否清晰
- ❌ 测试是否假设了实现细节
- ❌ 是否有重复或冗余的测试

**Codex 反馈格式**：
```markdown
## Test Review 报告

### 覆盖度分析
- ✅ 正常路径覆盖
- ⚠️  边界条件：缺少空列表测试
- ✅ 反例覆盖

### 建议
1. 添加空列表边界测试
2. test_duplicate_markets 可以简化

### 总体评价
测试基本完整，建议补充边界测试后通过。
```

#### 第二步：人工 Review（必须）

**用户的检查重点**：
- 测试是否符合**业务需求**
- Codex 的建议是否合理
- 是否需要补充测试

**用户决策**：
- ✅ **批准**：进入 Implementation 阶段
- ❌ **拒绝**：返回 Test 阶段修改

### 产出物

- Codex review 报告（存档）
- 人工批准记录

### 禁止项

- ❌ 跳过 Codex review
- ❌ 跳过人工确认
- ❌ 在 review 未通过时进入 Implementation

---

## 阶段 3：Implementation（AI 主导，最小化）

### 目的

- 让所有测试通过（绿灯）
- **不引入额外行为**

### 实现原则

| 原则 | 说明 |
|------|------|
| **Tests 决定"对错"** | 测试说对就对，测试说错就错 |
| **实现只负责"如何做到"** | 实现是技术细节 |
| **不做顺手优化** | 不添加测试未覆盖的功能 |

### 禁止项

- ❌ **修改测试**（除非测试本身被否定）
- ❌ **扩展需求**（测试未覆盖的功能不要加）
- ❌ **提前抽象**（不要"顺手"重构）

---

## 阶段 4：Implementation Review（Codex + 人工，强制）

### 目的

- **Codex**：检查代码质量、性能、安全
- **人工**：确认实现符合团队标准

### Review 流程

#### 第一步：Codex Review（自动）

**Claude 的操作**：
1. 完成实现后，通过 **codex-ccb** 技能调用 Codex
2. 提交审查请求：
   ```
   请 review 以下实现代码：
   - 代码质量（可读性、可维护性）
   - 是否遵循规范（命名、注释、类型注解）
   - 性能问题
   - 安全隐患
   - 是否引入额外功能（未测试覆盖）

   [实现代码内容]
   ```

**Codex 的检查重点**：
- ✅ 代码可读性（命名、结构、注释）
- ✅ 类型注解完整性
- ✅ 是否遵循项目规范（Python、Git 等）
- ✅ 性能问题（N+1 查询、低效算法）
- ✅ 安全隐患（SQL 注入、XSS、敏感信息泄露）
- ❌ 是否添加了测试未覆盖的功能
- ❌ 是否修改了测试

**Codex 反馈格式**：
```markdown
## Implementation Review 报告

### 代码质量
- ✅ 命名清晰
- ✅ 类型注解完整
- ⚠️  缺少文档字符串

### 规范遵循
- ✅ 符合 Python 命名规范
- ✅ 符合项目日志规范

### 性能与安全
- ✅ 无明显性能问题
- ✅ 无安全隐患

### 建议
1. 添加类和方法的文档字符串
2. `process_markets` 方法建议加上类型注解

### 总体评价
代码质量良好，建议补充文档后通过。
```

#### 第二步：人工 Review（必须）

**用户的检查重点**：
- 代码是否符合**团队标准**
- Codex 的建议是否合理
- 是否需要重构

**用户决策**：
- ✅ **批准**：合并到主干
- ❌ **拒绝**：返回 Implementation 阶段修改

### 产出物

- Codex review 报告（存档）
- 人工批准记录

### 禁止项

- ❌ 跳过 Codex review
- ❌ 跳过人工确认
- ❌ 在 review 未通过时合并代码

---

## 阶段 5：CI/CD（自动验证）

### 目的

- 自动运行所有测试
- 自动检查代码质量（lint、type check）
- 确保主干代码始终可发布

### CI/CD 配置

详见 [ci.md](ci.md)

---

## 四、Git Worktree 在流程中的作用

### 核心目的

> **物理隔离不同阶段的风险**

### 推荐映射

| 阶段 | Worktree 路径 | 允许修改 | 分支名 |
|------|--------------|---------|--------|
| **Prompt** | `worktrees/prompt-feature-xxx` | 仅文档 | `prompt/feature-xxx` |
| **Test** | `worktrees/test-feature-xxx` | `tests/` | `test/feature-xxx` |
| **Test Review** | （同 Test worktree） | - | - |
| **Impl** | `worktrees/impl-feature-xxx` | `src/` | `impl/feature-xxx` |
| **Impl Review** | （同 Impl worktree） | - | - |

### 使用时机

**建议使用 Worktree**：
- 多人协作
- 并行开发多个功能
- 复杂功能，需要物理隔离风险

**可不使用 Worktree**：
- 简单功能
- 个人项目
- 单线开发

---

## 五、完整流程示例

### 场景：实现企业上市标签功能

#### 阶段 0：Prompt

```markdown
# 需求：企业上市标签

## 业务目标
支持标记企业在不同市场的上市状态。

## 行为定义
- 支持多市场上市（HK / SH / SZ）
- 港股（HK）和 A 股（SH / SZ）是不同的标签
- 同一企业可能同时存在多个标签

## 示例
- 企业 A 在 HK 和 SH 上市 → 标签：["HK", "SH"]
```

#### 阶段 1：Test

```python
# tests/test_listing.py

def test_multi_market_listing():
    """企业可以在多个市场上市"""
    ent = Enterprise(markets=["HK", "SH"])
    assert ent.is_listed is True
    assert "HK" in ent.markets

def test_hk_not_equal_a_share():
    """港股不等于 A 股"""
    ent = Enterprise(markets=["HK"])
    assert ent.is_a_share is False
```

#### 阶段 2：Test Review

**Claude → Codex（通过 codex-ccb）**：
> 请 review 以下测试代码...

**Codex 反馈**：
> ✅ 测试覆盖基本完整
> ⚠️  建议添加空列表边界测试

**用户确认**：
> 批准，补充空列表测试后进入 Implementation

#### 阶段 3：Implementation

```python
# src/enterprise.py

class Enterprise:
    def __init__(self, markets: list[str]):
        self.markets = list(set(markets))

    @property
    def is_listed(self) -> bool:
        return len(self.markets) > 0

    @property
    def is_a_share(self) -> bool:
        return "SH" in self.markets or "SZ" in self.markets
```

#### 阶段 4：Implementation Review

**Claude → Codex（通过 codex-ccb）**：
> 请 review 以下实现代码...

**Codex 反馈**：
> ✅ 代码质量良好
> ⚠️  建议添加类和方法的文档字符串

**用户确认**：
> 批准，补充文档后合并

#### 阶段 5：CI/CD

```bash
git push origin impl/feature-listing
# GitLab CI 自动运行：
# ✅ pytest
# ✅ ruff check
# ✅ mypy
```

---

## 六、Claude 执行规范

### 阶段切换自动化

**Claude 必须在每个阶段结束时**：

1. **Test 完成后**：
   - 自动通过 codex-ccb 调用 Codex
   - 展示 Codex 反馈
   - 询问用户是否批准

2. **Implementation 完成后**：
   - 自动通过 codex-ccb 调用 Codex
   - 展示 Codex 反馈
   - 询问用户是否批准

### 禁止行为

- ❌ 跳过任何 Review 阶段
- ❌ 在 Codex review 前询问用户
- ❌ 在用户未批准时进入下一阶段

---

## 七、总结

### 核心流程

```
Prompt → Test → Review(Codex+人工) → Implementation → Review(Codex+人工) → CI/CD
```

### 核心价值

| 价值 | 说明 |
|------|------|
| **可审计** | Prompt 和 Review 记录保存决策 |
| **可回滚** | 测试不变，实现随时可以重来 |
| **可并行** | Worktree 物理隔离，支持并行开发 |
| **可维护** | 测试是最好的文档，代码质量有保障 |

### 心智模型

```
Prompt 决定方向
Test 冻结决策
Review 守护质量（Codex + 人工）
Implementation 可随时推翻
CI/CD 自动验证
```

---

（文档完）
