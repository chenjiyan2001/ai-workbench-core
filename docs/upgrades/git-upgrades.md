# 变更日志 (Git Upgrades)

本文件记录项目的所有版本变更历史，由 Claude 在每次提交时自动更新。

---

## 版本记录

### [未发布] - Unreleased

> 待下次发布的变更

---

### [v0.1.0] - 2024-XX-XX

#### 📋 变更摘要
初始版本，建立项目规范文档体系。

#### ✨ 新增 (Added)
- 建立开发规范文档 (git/python/logging/config/error-handling)
- 建立数据库规范文档 (mysql/mongodb/redis/milvus/dameng)
- 建立集成规范文档 (llm/api)
- 建立架构文档 (overview/adr)

#### 🐛 修复 (Fixed)
- 无

#### 🔧 变更 (Changed)
- 无

#### 🗑️ 移除 (Removed)
- 无

#### 📁 涉及文件
```
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
```

#### 💬 备注
项目初始化，建立规范文档体系。

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
