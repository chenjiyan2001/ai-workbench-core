# CI/CD 配置规范

> **目标**：在 AI 驱动的开发中，构建自动化质量守门系统，防止低质量代码进入主干。

本文档定义项目 CI/CD 的配置标准。模板文件见 [../docs/templates/ci-template-gitlab.yml](../docs/templates/ci-template-gitlab.yml)

**项目环境**：
- **代码仓库**：GitLab
- **操作系统**：Windows
- **终端**：PowerShell
- **Python**：3.11+

---

## 一、CI 的作用

### 在 Prompt → Test → Review → Implementation → Review 流程中的定位

```
Prompt（需求定义）
    ↓
Test（可执行规范）
    ↓
Review（Codex + 人工）
    ↓
Implementation（实现代码）
    ↓
Review（Codex + 人工）
    ↓
CI/CD（自动守门员）← 阻止低质量代码进入 main
    ↓
Merge to main
```

### 核心价值

| 价值 | 说明 |
|------|------|
| **自动裁判** | 测试失败 = 不允许合并 |
| **早期拦截** | 在人工 Review 后再次验证 |
| **可复现环境** | 避免"在我机器上能跑" |
| **约束 AI** | 防止 AI 生成代码绕过质量标准 |

---

## 二、CI 检查层级（8 层）

### 层级概览

| 层级 | 目的 | 是否推荐 | 说明 |
|------|------|---------|------|
| 1️⃣ 环境一致性 | 可复现 | ✅ **必须** | 统一 Python 版本、操作系统 |
| 2️⃣ 依赖安装 | 确定性 | ✅ **必须** | 锁定依赖版本 |
| 3️⃣ 静态检查（lint） | 早期错误 | ✅ **推荐** | 代码风格、语法错误 |
| 4️⃣ 类型检查 | 约束 AI | ✅ **强烈推荐** | 防止 AI 生成类型错误代码 |
| 5️⃣ 单元测试 | 行为裁判 | ✅ **必须** | 验证 Prompt → Test 的契约 |
| 6️⃣ 覆盖率 | 盲区感知 | ⚠️ 可选 | **默认不启用**，需要时手动开启 |
| 7️⃣ 变更约束 | 防违规 | ✅ **推荐** | 禁止修改特定文件 |
| 8️⃣ 产物检查 | 可发布性 | ⚠️ 视项目 | **默认不启用**，打包项目需要 |

---

### 1️⃣ 环境一致性

**目的**：确保所有开发者和 CI 环境使用相同的 Python 版本。

**GitLab CI 配置示例**：
```yaml
# 使用 Python 官方 Docker 镜像
image: python:3.11

variables:
  PYTHON_VERSION: "3.11"

before_script:
  - python --version
  - pip --version
```

**本地验证（PowerShell）**：
```powershell
python --version  # 应输出：Python 3.11.x
```

**为什么重要**：
- 避免"在我机器上能跑"问题
- Python 3.11 和 3.12 可能行为不同
- 保证 AI 生成的代码在所有环境都能运行

---

### 2️⃣ 依赖安装

**目的**：确保依赖版本锁定，安装过程可复现。

**GitLab CI 配置示例**：
```yaml
before_script:
  - python -m pip install --upgrade pip
  - pip install -r requirements.txt
```

**依赖管理规范**：

| 文件 | 用途 | 版本锁定 |
|------|------|---------|
| `requirements.txt` | 生产依赖 | 精确版本 `package==1.2.3` |
| `requirements-dev.txt` | 开发依赖 | 精确版本 |
| `pyproject.toml` | 项目元数据 | 版本范围 `package>=1.2,<2.0` |

**示例 requirements.txt**：
```txt
# 生产依赖
structlog==24.1.0
httpx==0.27.0
pydantic==2.6.0

# 开发依赖（或单独文件 requirements-dev.txt）
pytest==8.0.0
ruff==0.2.0
mypy==1.8.0
```

**本地安装（PowerShell）**：
```powershell
pip install -r requirements.txt
```

**为什么重要**：
- 防止"依赖地狱"
- 确保 CI 和本地环境一致
- 避免 AI 引入不兼容的依赖版本

---

### 3️⃣ 静态检查（Lint）

**目的**：在运行前发现语法错误、代码风格问题。

**GitLab CI 配置示例**：
```yaml
lint:
  stage: test
  script:
    - ruff check src/ tests/
  allow_failure: false  # 失败则阻止合并
```

**推荐工具**：
- **ruff**（推荐）：速度极快，集成多种检查
- ~~black~~（不推荐）：ruff 已包含格式化功能
- ~~flake8~~（不推荐）：ruff 更快

**ruff 配置示例**（`pyproject.toml`）：
```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
]
ignore = [
    "E501",  # line too long (由 formatter 处理)
]
```

**本地检查（PowerShell）**：
```powershell
# 检查代码
ruff check src/ tests/

# 自动修复
ruff check --fix src/ tests/
```

**为什么重要**：
- AI 可能生成不符合 PEP 8 的代码
- 早期发现未使用的导入、变量
- 统一团队代码风格

---

### 4️⃣ 类型检查

**目的**：约束 AI 生成代码的类型正确性。

**GitLab CI 配置示例**：
```yaml
type-check:
  stage: test
  script:
    - mypy src/
  allow_failure: false
```

**mypy 配置示例**（`pyproject.toml`）：
```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

# 第三方库无类型注解时
[[tool.mypy.overrides]]
module = "untyped_library.*"
ignore_missing_imports = true
```

**类型注解规范**：
```python
# ✅ 正确：完整类型注解
def process_data(items: list[dict[str, Any]]) -> list[str]:
    """处理数据"""
    return [item["name"] for item in items]

# ❌ 错误：缺少类型注解
def process_data(items):
    return [item["name"] for item in items]
```

**本地检查（PowerShell）**：
```powershell
mypy src/
```

**为什么重要**：
- **约束 AI**：防止 AI 生成类型错误的代码
- **早期发现错误**：类型错误在运行前暴露
- **自文档化**：类型注解即文档

---

### 5️⃣ 单元测试

**目的**：验证 Prompt → Test 的契约，是 CI 的核心。

**GitLab CI 配置示例**：
```yaml
test:
  stage: test
  script:
    - pytest tests/ -v
  allow_failure: false
```

**pytest 配置示例**（`pyproject.toml`）：
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",                    # 详细输出
    "--strict-markers",      # 严格标记
    "--tb=short",           # 简短的 traceback
]
```

**测试组织结构**：
```
tests/
├── unit/               # 单元测试
│   ├── test_enterprise.py
│   └── test_listing.py
├── integration/        # 集成测试
│   └── test_api.py
└── conftest.py         # pytest 配置
```

**本地运行（PowerShell）**：
```powershell
# 运行所有测试
pytest tests/ -v

# 运行特定测试
pytest tests/unit/test_enterprise.py -v
```

**为什么重要**：
- **Tests are the source of truth**
- 测试失败 = 实现不符合需求
- 防止 AI 修改测试绕过约束

---

### 6️⃣ 覆盖率（默认不启用）

**目的**：识别测试盲区，但**不作为质量标准**。

**为什么默认不启用**：
- 覆盖率 ≠ 测试质量
- 强制覆盖率容易产生"为覆盖而测试"的无意义测试
- 增加 CI 时间

**如需启用**（可选，GitLab CI）：
```yaml
coverage:
  stage: test
  script:
    - pip install pytest-cov
    - pytest tests/ --cov=src --cov-report=term-missing --cov-report=html
  artifacts:
    paths:
      - htmlcov/
  coverage: '/TOTAL.*\s+(\d+%)$/'  # GitLab 覆盖率徽章
  allow_failure: true  # 覆盖率低不阻止合并
```

**覆盖率配置**（`pyproject.toml`）：
```toml
[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/__init__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

**本地运行（PowerShell）**：
```powershell
pytest tests/ --cov=src --cov-report=html
# 打开 htmlcov/index.html 查看报告
```

**使用建议**：
- 不设置强制覆盖率门槛（如 80%）
- 用覆盖率发现遗漏的测试，而非强制指标
- 关键模块可手动检查覆盖率

---

### 7️⃣ 变更约束

**目的**：防止违规修改特定文件（如测试文件）。

**GitLab CI 配置示例**：
```yaml
check-test-changes:
  stage: test
  script:
    - |
      # 检查 MR 是否修改了测试文件
      CHANGED_FILES=$(git diff origin/main...HEAD --name-only)
      if echo "$CHANGED_FILES" | grep -q "tests/"; then
        echo "错误：禁止修改测试文件"
        exit 1
      fi
  only:
    - merge_requests
  allow_failure: false
```

**约束规则**：

| 场景 | 约束 | 原因 |
|------|------|------|
| **Implementation 阶段** | 禁止修改 `tests/` | 测试是契约，不能改 |
| **Test 阶段** | 禁止修改 `src/` | 先写测试，再实现 |
| **重构** | 测试必须保持通过 | 重构不改变行为 |

**本地验证（PowerShell）**：
```powershell
# 查看修改的文件
git diff origin/main...HEAD --name-only

# 检查是否修改了测试文件
$changedFiles = git diff origin/main...HEAD --name-only
if ($changedFiles -match "tests/") {
    Write-Error "禁止修改测试文件"
}
```

**为什么重要**：
- 防止 AI 修改测试让其"通过"
- 强制执行 Prompt → Test → Review → Impl 流程
- 保护测试的"裁判"地位

---

### 8️⃣ 产物检查（默认不启用）

**目的**：验证项目可以正确打包和发布。

**为什么默认不启用**：
- 大多数项目不需要打包发布
- 内部项目、脚本项目无需此检查

**如需启用**（可选，GitLab CI）：
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

**适用场景**：
- Python 库项目（需要发布到 PyPI）
- 需要打包为 wheel 的项目

**本地构建（PowerShell）**：
```powershell
pip install build
python -m build
# 产物在 dist/ 目录
```

---

## 三、完整 CI 配置模板

### GitLab CI 模板

见 [../docs/templates/ci-template-gitlab.yml](../docs/templates/ci-template-gitlab.yml)

**模板包含**：
- ✅ 层级 1: 环境一致性
- ✅ 层级 2: 依赖安装
- ✅ 层级 3: 静态检查（ruff）
- ✅ 层级 4: 类型检查（mypy）
- ✅ 层级 5: 单元测试（pytest）
- ✅ 层级 7: 变更约束
- ❌ 层级 6: 覆盖率（默认关闭，可手动开启）
- ❌ 层级 8: 产物检查（默认关闭，可手动开启）

---

## 四、如何在新项目中启用 CI

### 第一步：复制模板

**PowerShell**：
```powershell
# 复制 CI 模板到项目根目录
Copy-Item docs\templates\ci-template-gitlab.yml .gitlab-ci.yml
```

### 第二步：根据项目调整

**必须修改**：
- Python 版本（默认 3.11）
- 项目路径（如果不是 `src/` 和 `tests/`）

**可选修改**：
- 添加数据库服务（MySQL、Redis 等）
- 添加环境变量
- 启用覆盖率检查（层级 6）
- 启用产物检查（层级 8）

### 第三步：配置项目文件

**创建 `pyproject.toml`**（如果不存在）：
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

**创建 `requirements.txt`**：
```txt
# 生产依赖
structlog==24.1.0
httpx==0.27.0

# 开发依赖
pytest==8.0.0
ruff==0.2.0
mypy==1.8.0
```

### 第四步：提交到 GitLab

**PowerShell**：
```powershell
git add .gitlab-ci.yml
git add pyproject.toml
git add requirements.txt
git commit -m "ci: 添加 CI 配置"
git push
```

### 第五步：验证 CI

1. 推送代码到 GitLab
2. 访问项目页面，点击 **"CI/CD"** → **"Pipelines"**
3. 查看 Pipeline 运行结果：
   - ✅ **绿色**：所有检查通过
   - ❌ **红色**：有检查失败，点击查看详情

---

## 五、CI 失败后的处理流程

### 典型失败场景

| 失败类型 | 原因 | 解决方法 |
|---------|------|---------|
| **Lint 失败** | 代码风格不符合规范 | 运行 `ruff check --fix src/` 自动修复 |
| **Type 失败** | 类型注解错误 | 添加类型注解或修复类型错误 |
| **Test 失败** | 实现不符合测试 | 修改实现，**不修改测试** |
| **变更约束失败** | 修改了禁止的文件 | 回滚修改，遵循流程 |

### 处理步骤（PowerShell）

```powershell
# 1. 本地复现问题
pytest tests/ -v           # 运行测试
ruff check src/            # 检查代码风格
mypy src/                  # 检查类型

# 2. 修复问题
# （根据具体错误修复）

# 3. 再次验证
pytest tests/ -v
ruff check src/
mypy src/

# 4. 提交修复
git add .
git commit -m "fix: 修复 CI 失败问题"
git push
```

---

## 六、本地开发建议

### Pre-commit Hook（可选）

在提交前自动运行检查：

**PowerShell**：
```powershell
# 安装 pre-commit
pip install pre-commit

# 创建 .pre-commit-config.yaml
@"
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.2.0
    hooks:
      - id: ruff
        args: [--fix]
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
"@ | Out-File -FilePath .pre-commit-config.yaml -Encoding UTF8

# 安装 hook
pre-commit install
```

**效果**：
- 每次 `git commit` 时自动运行 ruff 和 mypy
- 检查失败 = 提交中止
- 避免 CI 失败

---

## 七、Claude 执行规范

### 新项目初始化时

当 Claude 协助初始化新项目时，**必须主动**：

1. 询问用户是否启用 CI
2. 如果启用，复制模板并根据项目调整
3. 创建必要的配置文件（`pyproject.toml`、`requirements.txt`）
4. 提交 CI 配置到仓库

### 代码修改时

**在推送代码前**，Claude 应：

1. 本地运行 CI 检查（PowerShell）：
   ```powershell
   pytest tests/ -v
   ruff check src/
   mypy src/
   ```

2. 确保所有检查通过后再推送

3. 如果检查失败，修复问题而非绕过检查

### 禁止行为

- ❌ 修改测试让其通过
- ❌ 添加 `# type: ignore` 绕过类型检查
- ❌ 添加 `# noqa` 绕过 lint 检查
- ❌ 在 CI 失败时仍然推送代码

---

## 八、总结

### CI 的核心价值

```
CI = 自动化的质量守门员
测试失败 = 不允许合并
约束 AI = 防止低质量代码
```

### 必须启用的层级

1. ✅ 环境一致性
2. ✅ 依赖安装
3. ✅ 静态检查（ruff）
4. ✅ 类型检查（mypy）
5. ✅ 单元测试（pytest）
7. ✅ 变更约束

### 可选层级

6. ⚠️ 覆盖率（需要时手动开启）
8. ⚠️ 产物检查（打包项目需要）

### 心智模型

```
Prompt 决定方向
Test 冻结决策
Review 守护质量（Codex + 人工）
Implementation 实现功能
CI/CD 自动验证 ← 最后一道防线
```

---

（文档完）
