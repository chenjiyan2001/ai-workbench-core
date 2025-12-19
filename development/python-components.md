# Python 通用组件规范

本文档定义项目中常用第三方组件的使用规范。

---

## 日志 - loguru

### 基础配置

```python
from loguru import logger
import sys

# 移除默认 handler
logger.remove()

# 添加自定义 handler
logger.add(
    sys.stderr,
    level="INFO",
    format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>"
)
```

### 文件轮转

```python
logger.add(
    "logs/{time:YYYY-MM-DD}.log",
    rotation="00:00",      # 每天午夜轮转
    retention="30 days",   # 保留 30 天
    compression="gz",      # 压缩旧日志
    encoding="utf-8"
)
```

### 脚本中的简化用法

```python
from loguru import logger

# 简单脚本直接使用，无需配置
logger.info("处理开始")
logger.error(f"处理失败: {e}")
```

### 级别使用规范

| 级别 | 使用场景 |
|------|---------|
| `DEBUG` | 调试信息，生产环境不输出 |
| `INFO` | 正常流程关键节点 |
| `WARNING` | 异常但可继续执行 |
| `ERROR` | 错误，需要关注 |

---

## 进度条 - tqdm

### 基础用法

```python
from tqdm import tqdm

for item in tqdm(items, desc="处理中"):
    process(item)
```

### 手动更新

```python
with tqdm(total=100, desc="下载中") as pbar:
    for chunk in download():
        pbar.update(len(chunk))
```

### 多线程（推荐 as_completed）

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from tqdm import tqdm

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(process, item): item for item in items}

    with tqdm(total=len(items), desc="处理中") as pbar:
        for future in as_completed(futures):
            try:
                result = future.result()
            except Exception as e:
                logger.error(f"失败: {e}")
            finally:
                pbar.update(1)
```

### 嵌套进度条

```python
for batch in tqdm(batches, desc="批次", position=0):
    for item in tqdm(batch, desc="处理", position=1, leave=False):
        process(item)
```

### 禁用进度条（非交互环境）

```python
from tqdm import tqdm

# 通过环境变量或参数控制
disable = not sys.stderr.isatty()
for item in tqdm(items, desc="处理中", disable=disable):
    process(item)
```

---

## 命令行 - argparse

> 简单脚本使用标准库 argparse，复杂 CLI 工具可选用 typer/click

### 基础模板

```python
import argparse

def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="脚本说明")

    # 互斥参数组
    group = parser.add_mutually_exclusive_group()
    group.add_argument("--limit", type=int, help="限制处理数量")
    group.add_argument("--test-key", help="单条测试")

    # 开关参数
    parser.add_argument("--dry-run", action="store_true", help="试运行")

    # 带默认值
    parser.add_argument("--workers", type=int, default=4, help="并发数")

    return parser.parse_args()
```

---

## 日期时间 - dateparser / pendulum

### dateparser（解析不规则日期）

```python
import dateparser

# 自然语言解析
dateparser.parse("3天前")           # datetime 对象
dateparser.parse("2024年1月15日")
dateparser.parse("last monday")

# 指定语言
dateparser.parse("15 janvier 2024", languages=["fr"])
```

### pendulum（日常日期操作）

```python
import pendulum

# 时区感知
now = pendulum.now("Asia/Shanghai")
utc = pendulum.now("UTC")

# 日期运算
tomorrow = now.add(days=1)
last_week = now.subtract(weeks=1)

# 格式化
now.to_datetime_string()    # "2025-01-15 14:30:00"
now.to_date_string()        # "2025-01-15"

# 人性化输出
now.diff_for_humans()       # "刚刚"
```

### 选型建议

| 场景 | 推荐 |
|------|------|
| 解析爬虫/用户输入的不规则日期 | `dateparser` |
| 日常日期操作、时区处理 | `pendulum` |
| 简单场景、无额外依赖 | 标准库 `datetime` + `zoneinfo` |

---

## 检查清单

- [ ] loguru 配置了合适的日志级别
- [ ] tqdm 多线程使用 as_completed 模式
- [ ] 非交互环境禁用进度条
- [ ] 日期时间处理考虑时区
