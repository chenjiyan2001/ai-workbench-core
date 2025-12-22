# 模板文件说明

本目录包含项目开发所需的各类模板文件。

---

## 目录结构

```
docs/templates/
├── README.md                   # 本文件
├── prompt-template.md          # Prompt 模板
└── ci-template-gitlab.yml      # GitLab CI 配置模板
```

---

## 一、Prompt 模板

### 文件

`prompt-template.md`

### 用途

用于创建符合 **Prompt → Test → Review → Implementation → Review** 流程的需求文档。

### 使用方法

**PowerShell**：
```powershell
# 1. 创建 prompts 目录
New-Item -ItemType Directory -Path docs\prompts -Force

# 2. 复制模板
Copy-Item docs\templates\prompt-template.md docs\prompts\feature-001-功能名称.md

# 3. 编辑 Prompt
code docs\prompts\feature-001-功能名称.md
```

### 填写指南

#### 必填项

| 部分 | 说明 | 示例 |
|------|------|------|
| **业务目标（Why）** | 为什么做这个功能 | "支持企业多市场上市标记" |
| **功能定义（What）** | 具体功能点 | "1. 支持 HK/SH/SZ 标记<br>2. 同一企业可多个标签" |
| **示例** | 正例和反例 | "正例：HK+SH 双上市<br>反例：退市企业不处理" |

#### 可选项

| 部分 | 说明 | 何时填写 |
|------|------|---------|
| **边界与排除** | 明确不做的内容 | 需求有歧义时 |
| **Claude 辅助提问区** | 记录与 Claude 的交互 | 需求不明确时 |

### 与 Claude 交互

Prompt 模板支持**渐进式补充**：

1. **初始版本**：只填写业务目标和核心功能
2. **与 Claude 交互**：Claude 会提问补充细节
3. **最终版本**：通过问答逐步完善 Prompt

**示例对话**：

```
用户：帮我实现企业上市标签功能
Claude：好的，我需要了解更多细节。请先创建 Prompt 文档：
        - 业务目标是什么？
        - 支持哪些市场？
        - 同一企业能否多个标签？
用户：支持 HK/SH/SZ，可以多个
Claude：明白了，我会将这些补充到 Prompt 中...
```

### 存档位置

```
docs/
└── prompts/
    ├── feature-001-企业上市标签.md
    ├── feature-002-数据导出.md
    └── ...
```

---

## 二、CI 配置模板

### 文件

`ci-template-gitlab.yml`

### 用途

用于创建符合 **8 层 CI 检查标准** 的 GitLab CI/CD 配置。

### 包含的层级

| 层级 | 名称 | 默认启用 |
|------|------|---------|
| 1️⃣ | 环境一致性 | ✅ |
| 2️⃣ | 依赖安装 | ✅ |
| 3️⃣ | 静态检查（ruff） | ✅ |
| 4️⃣ | 类型检查（mypy） | ✅ |
| 5️⃣ | 单元测试（pytest） | ✅ |
| 6️⃣ | 覆盖率 | ❌（需手动开启） |
| 7️⃣ | 变更约束 | ✅ |
| 8️⃣ | 产物检查 | ❌（需手动开启） |

### 使用方法

**PowerShell**：
```powershell
# 1. 复制模板到项目根目录
Copy-Item docs\templates\ci-template-gitlab.yml .gitlab-ci.yml

# 2. 根据项目调整（可选）
#    - Python 版本（默认 3.11）
#    - 项目路径（默认 src/ 和 tests/）
#    - 启用可选层级（覆盖率、产物检查）

# 3. 创建必要的配置文件
#    - pyproject.toml（ruff、mypy、pytest 配置）
#    - requirements.txt（依赖列表）

# 4. 提交到 GitLab
git add .gitlab-ci.yml
git commit -m "ci: 添加 CI 配置"
git push
```

### 启用可选层级

#### 启用覆盖率检查（层级 6）

在 `.gitlab-ci.yml` 中取消注释：

```yaml
coverage:
  stage: test
  script:
    - pip install pytest-cov
    - pytest tests/ --cov=src --cov-report=term-missing --cov-report=html
  artifacts:
    paths:
      - htmlcov/
  coverage: '/TOTAL.*\s+(\d+%)$/'
  allow_failure: true
```

#### 启用产物检查（层级 8）

在 `.gitlab-ci.yml` 中取消注释：

```yaml
build:
  stage: build
  script:
    - pip install build
    - python -m build
  artifacts:
    paths:
      - dist/
  allow_failure: true
```

### 必需的配置文件

#### 1. `pyproject.toml`

```toml
[project]
name = "your-project"
version = "0.1.0"
requires-python = ">=3.11"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
```

#### 2. `requirements.txt`

```txt
# 生产依赖
structlog==24.1.0
httpx==0.27.0

# 开发依赖
pytest==8.0.0
ruff==0.2.0
mypy==1.8.0
```

### 验证 CI

提交代码后，访问 GitLab 项目页面：

1. 点击 **"CI/CD"** → **"Pipelines"**
2. 查看 Pipeline 运行结果：
   - ✅ 绿色：所有检查通过
   - ❌ 红色：有检查失败，点击查看详情

### 本地预运行

在推送前本地验证（PowerShell）：

```powershell
# 运行所有检查
pytest tests/ -v           # 测试
ruff check src/ tests/     # 代码风格
mypy src/                  # 类型检查
```

---

## 三、模板维护

### 更新模板

当项目规范更新时，需同步更新模板：

1. 修改 `docs/templates/` 中的模板文件
2. 提交到 Git
3. 通知团队成员更新现有项目的配置

### 版本控制

- 模板文件纳入 Git 版本控制
- 使用 Git Tag 标记模板的重大更新
- 在 CHANGELOG.md 中记录模板变更

---

## 四、常见问题

### Q1: Prompt 模板太复杂，能简化吗？

**A**: 可以。当前模板已经是简化版，只包含核心必填项。通过与 Claude 交互，可以逐步补充细节，无需一次性填完。

### Q2: CI 模板适用于所有项目吗？

**A**: 基本适用于所有 Python 项目。特殊项目（如数据科学、Web 服务）可能需要额外配置：
- 添加数据库服务
- 添加环境变量
- 调整测试路径

### Q3: 如何在现有项目中应用模板？

**A**:
1. 备份现有配置
2. 复制模板到项目
3. 合并现有配置到模板
4. 测试 CI 是否正常运行

### Q4: 模板更新后，现有项目需要同步吗？

**A**: 建议同步，但不强制：
- **必须同步**：安全漏洞修复、关键规范变更
- **建议同步**：新增检查层级、性能优化
- **可选同步**：文档说明更新

---

## 五、相关文档

- **工作流程规范**：[development/workflow.md](../../development/workflow.md)
- **CI/CD 配置规范**：[development/ci.md](../../development/ci.md)
- **项目规范总览**：[CLAUDE.md](../../CLAUDE.md)

---

（文档完）
