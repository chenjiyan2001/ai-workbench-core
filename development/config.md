# 配置管理规范

## 适用范围

本规范适用于所有涉及配置管理的场景，包括环境变量、配置文件、密钥管理等。

## 配置分层

```
优先级 (高 → 低):
1. 环境变量
2. .env 文件 (本地开发)
3. 配置文件 (YAML/TOML)
4. 代码中的默认值
```

## 环境变量

### 命名规范

```bash
# 格式: {APP}_{MODULE}_{NAME}
# 全大写，下划线分隔

# ✅ 正确
WORKBENCH_DB_HOST=localhost
WORKBENCH_DB_PORT=3306
WORKBENCH_REDIS_URL=redis://localhost:6379
WORKBENCH_LLM_API_KEY=sk-xxx
WORKBENCH_LOG_LEVEL=INFO

# ❌ 错误
db_host=localhost        # 应全大写
WorkbenchDbHost=xxx      # 应使用下划线
DATABASE_HOST=xxx        # 缺少应用前缀
```

### 常用环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `{APP}_ENV` | 运行环境 | development/staging/production |
| `{APP}_DEBUG` | 调试模式 | true/false |
| `{APP}_LOG_LEVEL` | 日志级别 | DEBUG/INFO/WARNING/ERROR |
| `{APP}_DB_*` | 数据库配置 | HOST/PORT/USER/PASSWORD/NAME |
| `{APP}_REDIS_*` | Redis配置 | URL/HOST/PORT/PASSWORD |
| `{APP}_*_API_KEY` | API密钥 | LLM_API_KEY/OPENAI_API_KEY |

## 配置类设计

### 使用 Pydantic Settings

```python
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseSettings(BaseSettings):
    """数据库配置"""
    model_config = SettingsConfigDict(env_prefix="WORKBENCH_DB_")

    host: str = "localhost"
    port: int = 3306
    user: str = "root"
    password: SecretStr  # 敏感信息使用 SecretStr
    name: str = "workbench"
    pool_size: int = Field(default=10, ge=1, le=100)

    @property
    def url(self) -> str:
        """生成连接 URL (不含密码)"""
        return f"mysql://{self.user}@{self.host}:{self.port}/{self.name}"


class RedisSettings(BaseSettings):
    """Redis 配置"""
    model_config = SettingsConfigDict(env_prefix="WORKBENCH_REDIS_")

    host: str = "localhost"
    port: int = 6379
    password: SecretStr | None = None
    db: int = 0


class LLMSettings(BaseSettings):
    """LLM 配置"""
    model_config = SettingsConfigDict(env_prefix="WORKBENCH_LLM_")

    provider: str = "openai"  # openai/anthropic/azure
    api_key: SecretStr
    model: str = "gpt-4"
    max_tokens: int = 4096
    temperature: float = Field(default=0.7, ge=0, le=2)


class Settings(BaseSettings):
    """应用配置"""
    model_config = SettingsConfigDict(
        env_prefix="WORKBENCH_",
        env_file=".env",
        env_file_encoding="utf-8",
    )

    env: str = "development"
    debug: bool = False
    log_level: str = "INFO"

    # 嵌套配置
    db: DatabaseSettings = Field(default_factory=DatabaseSettings)
    redis: RedisSettings = Field(default_factory=RedisSettings)
    llm: LLMSettings = Field(default_factory=LLMSettings)

    @property
    def is_production(self) -> bool:
        return self.env == "production"


# 全局配置实例
settings = Settings()
```

### 配置使用

```python
from config import settings

# ✅ 正确: 通过配置对象访问
db_host = settings.db.host
api_key = settings.llm.api_key.get_secret_value()

# ❌ 错误: 直接读取环境变量
import os
db_host = os.getenv("WORKBENCH_DB_HOST")  # 应使用配置类
```

## .env 文件

### 格式规范

```bash
# .env.example (提交到仓库，作为模板)
# 应用配置
WORKBENCH_ENV=development
WORKBENCH_DEBUG=true
WORKBENCH_LOG_LEVEL=DEBUG

# 数据库配置
WORKBENCH_DB_HOST=localhost
WORKBENCH_DB_PORT=3306
WORKBENCH_DB_USER=root
WORKBENCH_DB_PASSWORD=  # 本地填写，不提交
WORKBENCH_DB_NAME=workbench

# Redis 配置
WORKBENCH_REDIS_HOST=localhost
WORKBENCH_REDIS_PORT=6379

# LLM 配置
WORKBENCH_LLM_PROVIDER=openai
WORKBENCH_LLM_API_KEY=  # 本地填写，不提交
WORKBENCH_LLM_MODEL=gpt-4
```

### .env 文件管理

```
.env.example    # 模板文件，提交到仓库
.env            # 本地配置，不提交 (在 .gitignore 中)
.env.test       # 测试环境配置
.env.staging    # 预发环境配置
```

## 配置文件 (YAML)

### 适用场景

- 复杂的结构化配置
- 多环境配置对比
- 配置版本管理

### 格式示例

```yaml
# config/base.yaml - 基础配置
app:
  name: workbench
  version: "1.0.0"

logging:
  level: INFO
  format: json

database:
  pool_size: 10
  connect_timeout: 30

# config/production.yaml - 生产环境覆盖
database:
  pool_size: 50
  connect_timeout: 10

logging:
  level: WARNING
```

### 加载配置

```python
from pathlib import Path
import yaml

def load_config(env: str = "development") -> dict:
    """加载配置文件"""
    config_dir = Path("config")

    # 加载基础配置
    with open(config_dir / "base.yaml") as f:
        config = yaml.safe_load(f)

    # 加载环境配置并合并
    env_file = config_dir / f"{env}.yaml"
    if env_file.exists():
        with open(env_file) as f:
            env_config = yaml.safe_load(f)
            deep_merge(config, env_config)

    return config
```

## 密钥管理

### 本地开发

```bash
# 使用 .env 文件，确保在 .gitignore 中
.env
```

### 生产环境

推荐方案 (按优先级):
1. **云密钥服务**: AWS Secrets Manager / Azure Key Vault / 阿里云密钥管理
2. **HashiCorp Vault**: 自建密钥管理
3. **环境变量**: K8s Secrets 注入

### 密钥访问示例

```python
from pydantic import SecretStr

class Settings(BaseSettings):
    # SecretStr 防止意外打印
    db_password: SecretStr
    api_key: SecretStr

settings = Settings()

# ✅ 正确: 显式获取密钥值
password = settings.db_password.get_secret_value()

# 打印时自动隐藏
print(settings.db_password)  # 输出: SecretStr('**********')
```

## 配置校验

### 启动时校验

```python
def validate_config(settings: Settings) -> None:
    """启动时校验配置"""
    errors = []

    # 必填项检查
    if not settings.db.password.get_secret_value():
        errors.append("WORKBENCH_DB_PASSWORD 未配置")

    if not settings.llm.api_key.get_secret_value():
        errors.append("WORKBENCH_LLM_API_KEY 未配置")

    # 生产环境额外检查
    if settings.is_production:
        if settings.debug:
            errors.append("生产环境禁止开启 DEBUG 模式")
        if settings.log_level == "DEBUG":
            errors.append("生产环境日志级别不应为 DEBUG")

    if errors:
        raise ValueError(f"配置校验失败: {errors}")
```

## 禁止事项

```python
# ❌ 禁止硬编码敏感信息
db_password = "my_password"
api_key = "sk-xxx"

# ❌ 禁止在日志中打印配置
logger.info("配置", settings=settings.model_dump())  # 可能泄露密钥

# ❌ 禁止将 .env 提交到仓库
# 确保 .gitignore 包含 .env

# ❌ 禁止在代码中直接使用 os.getenv
import os
host = os.getenv("DB_HOST", "localhost")  # 应使用配置类
```

## 检查清单

- [ ] 敏感信息使用 SecretStr
- [ ] .env 文件已加入 .gitignore
- [ ] 提供 .env.example 模板
- [ ] 配置类有类型注解和校验
- [ ] 启动时进行配置校验
- [ ] 生产环境使用密钥管理服务
