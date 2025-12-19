# Python 脚本编写规范

本文档定义 Python 脚本（一次性任务、批量处理、数据迁移等）的编写规范。

---

## 核心原则

1. **从简原则**：简单一次性任务，怎么方便怎么来，只保留必要内容
2. **可测试**：支持少量测试和单条测试
3. **可恢复**：批量任务支持断点续传
4. **可观测**：实时进度条，清晰的日志输出

---

## 技术选型

| 组件 | 选型 | 说明 |
|------|------|------|
| 日志 | `loguru` | 简化配置，开箱即用 |
| 进度条 | `tqdm` | 实时进度展示 |
| 参数解析 | `argparse` / `typer` | 按需选择 |
| 多线程 | `concurrent.futures` | 配合 `as_completed` |

---

## 必备参数

所有涉及数据处理的脚本必须支持以下参数：

| 参数 | 说明 | 示例 |
|------|------|------|
| `--limit N` | 少量测试，只处理前 N 条 | `--limit 10` |
| `--test-key KEY` | 单条测试，按主键处理 | `--test-key eid:123` |
| `--dry-run` | 试运行，不写库 | 默认写库 |

**注意**：`--limit` 和 `--test-key` 互斥，不同时使用。

### 主键约定

| 数据类型 | 主键格式 |
|---------|---------|
| 企业数据 | `eid` |
| 新闻数据 | `md5-url` |
| 其他 | 根据实际情况定义 |

---

## 写入策略

| 模式 | 写入方式 | 说明 |
|------|---------|------|
| 测试（`--limit`/`--test-key`） | 实时写入 | 方便调试验证 |
| 跑批（无 limit） | 批量写入 | 提高性能 |

---

## 异常处理（固定行为）

不通过参数选择，行为固定：

| 模式 | 行为 |
|------|------|
| **测试模式** | 遇错立即终止，输出完整错误信息 |
| **跑批模式** | 记录错误继续执行，累计 **5 次**错误后停止 |

### 错误输出要求

错误信息必须包含足够定位问题的内容：
- 出错的数据主键
- 异常类型和消息
- 关键上下文（如当前处理的字段值）

### 失败记录

跑批模式下，失败项记录到文件（如 `failed_items.jsonl`），支持后续批量修正。

---

## 断点续传

批量任务必须支持断点续传：

### 方式一：进度文件

```python
PROGRESS_FILE = "progress.json"

def load_progress() -> set:
    """加载已处理的主键集合。"""
    if Path(PROGRESS_FILE).exists():
        return set(json.loads(Path(PROGRESS_FILE).read_text()))
    return set()

def save_progress(processed: set):
    """保存进度。"""
    Path(PROGRESS_FILE).write_text(json.dumps(list(processed)))
```

### 方式二：数据库状态字段

```sql
-- 添加处理状态字段
ALTER TABLE target_table ADD COLUMN process_status ENUM('pending', 'done', 'failed') DEFAULT 'pending';

-- 查询待处理数据
SELECT * FROM target_table WHERE process_status = 'pending';
```

---

## 多线程 + 进度条

使用 `as_completed` 实现处理完一条即更新进度：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from tqdm import tqdm
from loguru import logger


def run_batch(
    items: list,
    max_workers: int = 4,
    max_errors: int = 5
) -> list:
    """批量处理，累计错误超过阈值则停止。

    Args:
        items: 待处理数据列表
        max_workers: 并发线程数
        max_errors: 最大容错数

    Returns:
        失败项列表
    """
    error_count = 0
    failed_items = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {executor.submit(process_item, item): item for item in items}

        with tqdm(total=len(items), desc="处理中") as pbar:
            for future in as_completed(futures):
                item = futures[future]
                try:
                    result = future.result()
                    # 成功处理，更新进度文件/数据库状态
                except Exception as e:
                    error_count += 1
                    failed_items.append({"key": item, "error": str(e)})
                    logger.error(f"处理失败 [{item}]: {e}")

                    if error_count >= max_errors:
                        logger.error(f"累计错误达到 {max_errors}，终止任务")
                        executor.shutdown(wait=False, cancel_futures=True)
                        break
                finally:
                    pbar.update(1)

    # 输出失败项
    if failed_items:
        save_failed_items(failed_items)
        logger.warning(f"共 {len(failed_items)} 条失败，已保存到 failed_items.jsonl")

    return failed_items
```

---

## 脚本模板

```python
#!/usr/bin/env python3
"""脚本简要说明。

Usage:
    python script.py --limit 10          # 测试前 10 条
    python script.py --test-key eid:123  # 测试单条
    python script.py --dry-run           # 试运行
    python script.py                      # 正式执行
"""
import argparse
import sys
from pathlib import Path

from loguru import logger
from tqdm import tqdm


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=__doc__)

    # 必备参数
    group = parser.add_mutually_exclusive_group()
    group.add_argument("--limit", type=int, help="少量测试，只处理前 N 条")
    group.add_argument("--test-key", help="单条测试，格式: field:value")
    parser.add_argument("--dry-run", action="store_true", help="试运行，不写库")

    # 可选参数
    parser.add_argument("--workers", type=int, default=4, help="并发线程数")

    return parser.parse_args()


def is_test_mode(args: argparse.Namespace) -> bool:
    """判断是否为测试模式。"""
    return args.limit is not None or args.test_key is not None


def main() -> int:
    args = parse_args()

    # 配置日志
    logger.remove()
    logger.add(sys.stderr, level="INFO")

    # 获取待处理数据
    items = get_items(args)

    if not items:
        logger.info("无待处理数据")
        return 0

    logger.info(f"待处理: {len(items)} 条")

    if args.dry_run:
        logger.info("[DRY-RUN] 试运行模式，不写库")
        return 0

    # 处理数据
    if is_test_mode(args):
        # 测试模式：遇错即终止，实时写入
        for item in tqdm(items, desc="处理中"):
            process_item(item, write_immediately=True)
    else:
        # 跑批模式：记录错误继续，批量写入
        failed = run_batch(items, max_workers=args.workers)
        if failed:
            logger.warning(f"失败 {len(failed)} 条，请检查 failed_items.jsonl")
            return 1

    logger.info("处理完成")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

---

## 检查清单

- [ ] 支持 `--limit`、`--test-key`、`--dry-run` 参数
- [ ] `--limit` 和 `--test-key` 互斥
- [ ] 测试模式遇错即终止
- [ ] 跑批模式记录错误，累计 5 次后停止
- [ ] 失败项保存到文件
- [ ] 支持断点续传
- [ ] 多线程任务使用 `as_completed` + tqdm
