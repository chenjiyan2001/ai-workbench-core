# LLM 调用规范

## 适用范围

本规范适用于所有调用大语言模型 (LLM) 的场景，包括数据打标、摘要生成、信息抽取等。

## 核心原则

1. **可追溯**: 每次调用记录 model/prompt/input/output
2. **可评测**: 建立评测基准，持续监控质量
3. **可控成本**: 监控 Token 消耗和费用
4. **结构化输出**: 强制 JSON Schema 约束

## 模型配置

### 模型选择

| 任务类型 | 推荐模型 | 说明 |
|---------|---------|------|
| 简单分类/打标 | GPT-3.5-Turbo / Claude Haiku | 成本低，速度快 |
| 复杂分析/推理 | GPT-4 / Claude Sonnet | 准确度高 |
| 长文本处理 | GPT-4-Turbo / Claude | 支持长上下文 |
| 私有化部署 | Qwen / ChatGLM / Baichuan | 数据合规 |

### 配置示例

```python
from pydantic import BaseModel, Field


class LLMConfig(BaseModel):
    """LLM 配置"""
    provider: str = "openai"  # openai / anthropic / azure / local
    model: str = "gpt-4-turbo"
    temperature: float = Field(default=0.0, ge=0, le=2)  # 生产环境建议 0
    max_tokens: int = 4096
    timeout: int = 60
    max_retries: int = 3


# 不同任务的配置
LLM_CONFIGS = {
    "labeling": LLMConfig(model="gpt-3.5-turbo", temperature=0, max_tokens=100),
    "summarization": LLMConfig(model="gpt-4-turbo", temperature=0.3, max_tokens=1000),
    "extraction": LLMConfig(model="gpt-4", temperature=0, max_tokens=2000),
}
```

## Prompt 管理

### Prompt 模板化

```python
from string import Template

# ✅ 使用模板管理 Prompt
PROMPTS = {
    "label_document": Template("""
你是一个文档分类专家。请根据以下文档内容，从给定的标签中选择最合适的标签。

## 可选标签
${labels}

## 文档内容
${content}

## 输出要求
只输出标签名称，不要有其他内容。
"""),

    "extract_entities": Template("""
从以下文本中提取实体信息，按照指定的 JSON Schema 输出。

## 文本
${text}

## JSON Schema
${schema}

## 输出
请直接输出符合 Schema 的 JSON，不要有其他内容。
"""),
}


def render_prompt(template_name: str, **kwargs) -> str:
    """渲染 Prompt 模板"""
    template = PROMPTS.get(template_name)
    if not template:
        raise ValueError(f"未知的 Prompt 模板: {template_name}")
    return template.substitute(**kwargs)
```

### Prompt 版本管理

```python
# prompts/v1/label_document.txt
# prompts/v2/label_document.txt

PROMPT_VERSIONS = {
    "label_document": {
        "v1": "prompts/v1/label_document.txt",
        "v2": "prompts/v2/label_document.txt",
        "current": "v2",
    },
}


def get_prompt(name: str, version: str | None = None) -> str:
    """获取指定版本的 Prompt"""
    config = PROMPT_VERSIONS[name]
    version = version or config["current"]
    path = config[version]
    with open(path) as f:
        return f.read()
```

## 结构化输出

### 使用 JSON Schema 约束

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()


class LabelResult(BaseModel):
    """标签结果"""
    label: str
    confidence: float
    reasoning: str


async def label_document(content: str, labels: list[str]) -> LabelResult:
    """文档打标 - 结构化输出"""
    response = client.chat.completions.create(
        model="gpt-4-turbo",
        messages=[
            {"role": "system", "content": "你是文档分类专家"},
            {"role": "user", "content": f"对以下文档分类，可选标签: {labels}\n\n{content}"},
        ],
        response_format={"type": "json_object"},  # 强制 JSON 输出
        temperature=0,
    )

    result = json.loads(response.choices[0].message.content)
    return LabelResult(**result)
```

### 使用 Function Calling

```python
async def extract_person_info(text: str) -> dict:
    """使用 Function Calling 提取结构化信息"""
    tools = [
        {
            "type": "function",
            "function": {
                "name": "extract_person",
                "description": "提取人物信息",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string", "description": "姓名"},
                        "age": {"type": "integer", "description": "年龄"},
                        "occupation": {"type": "string", "description": "职业"},
                    },
                    "required": ["name"],
                },
            },
        }
    ]

    response = client.chat.completions.create(
        model="gpt-4-turbo",
        messages=[{"role": "user", "content": text}],
        tools=tools,
        tool_choice={"type": "function", "function": {"name": "extract_person"}},
    )

    return json.loads(response.choices[0].message.tool_calls[0].function.arguments)
```

## 调用追踪

### 必须记录的信息

```python
from dataclasses import dataclass, field
from datetime import datetime
import hashlib


@dataclass
class LLMCallRecord:
    """LLM 调用记录"""
    # 标识
    call_id: str
    task_type: str  # labeling / summarization / extraction

    # 模型信息
    model: str
    prompt_version: str

    # 输入输出
    input_text: str
    input_hash: str  # MD5 hash，用于去重和对比
    output_text: str
    output_parsed: dict | None  # 解析后的结构化数据

    # 性能指标
    input_tokens: int
    output_tokens: int
    latency_ms: int
    cost_usd: float

    # 质量指标 (可选)
    schema_valid: bool = True
    manual_review: str | None = None  # approved / rejected / pending

    # 元数据
    created_at: datetime = field(default_factory=datetime.utcnow)


def calculate_input_hash(text: str) -> str:
    """计算输入文本的哈希值"""
    return hashlib.md5(text.encode()).hexdigest()
```

### 调用封装

```python
import time
import structlog

logger = structlog.get_logger()


class LLMClient:
    """LLM 客户端封装"""

    def __init__(self, config: LLMConfig) -> None:
        self.config = config
        self.client = self._create_client()

    async def call(
        self,
        prompt: str,
        task_type: str,
        prompt_version: str = "v1",
    ) -> LLMCallRecord:
        """调用 LLM 并记录"""
        call_id = str(uuid.uuid4())
        input_hash = calculate_input_hash(prompt)
        start_time = time.monotonic()

        try:
            response = await self._do_call(prompt)
            latency_ms = int((time.monotonic() - start_time) * 1000)

            record = LLMCallRecord(
                call_id=call_id,
                task_type=task_type,
                model=self.config.model,
                prompt_version=prompt_version,
                input_text=prompt,
                input_hash=input_hash,
                output_text=response.content,
                output_parsed=self._try_parse_json(response.content),
                input_tokens=response.usage.prompt_tokens,
                output_tokens=response.usage.completion_tokens,
                latency_ms=latency_ms,
                cost_usd=self._calculate_cost(response.usage),
            )

            # 记录日志
            logger.info(
                "LLM 调用完成",
                call_id=call_id,
                task_type=task_type,
                model=self.config.model,
                input_tokens=record.input_tokens,
                output_tokens=record.output_tokens,
                latency_ms=latency_ms,
                cost_usd=record.cost_usd,
            )

            # 持久化记录
            await self._save_record(record)

            return record

        except Exception as e:
            logger.error(
                "LLM 调用失败",
                call_id=call_id,
                task_type=task_type,
                error=str(e),
            )
            raise
```

## 成本控制

### Token 计算

```python
import tiktoken


def count_tokens(text: str, model: str = "gpt-4") -> int:
    """计算文本的 Token 数"""
    try:
        encoding = tiktoken.encoding_for_model(model)
    except KeyError:
        encoding = tiktoken.get_encoding("cl100k_base")
    return len(encoding.encode(text))


def estimate_cost(input_tokens: int, output_tokens: int, model: str) -> float:
    """估算调用成本 (USD)"""
    # 价格表 (per 1K tokens)
    PRICING = {
        "gpt-4-turbo": {"input": 0.01, "output": 0.03},
        "gpt-4": {"input": 0.03, "output": 0.06},
        "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
        "claude-3-opus": {"input": 0.015, "output": 0.075},
        "claude-3-sonnet": {"input": 0.003, "output": 0.015},
        "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
    }

    price = PRICING.get(model, {"input": 0.01, "output": 0.03})
    return (input_tokens * price["input"] + output_tokens * price["output"]) / 1000
```

### 成本监控

```python
from collections import defaultdict
from datetime import date


class CostTracker:
    """成本追踪器"""

    def __init__(self, daily_limit_usd: float = 100.0) -> None:
        self.daily_limit = daily_limit_usd
        self.daily_costs: dict[date, float] = defaultdict(float)

    async def check_budget(self) -> bool:
        """检查是否超出预算"""
        today = date.today()
        return self.daily_costs[today] < self.daily_limit

    async def record_cost(self, cost_usd: float) -> None:
        """记录成本"""
        today = date.today()
        self.daily_costs[today] += cost_usd

        if self.daily_costs[today] >= self.daily_limit * 0.8:
            logger.warning(
                "LLM 成本预警",
                daily_cost=self.daily_costs[today],
                daily_limit=self.daily_limit,
                usage_percent=self.daily_costs[today] / self.daily_limit * 100,
            )
```

## 错误处理与重试

### 重试策略

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)


class RateLimitError(Exception):
    pass


class InvalidResponseError(Exception):
    pass


@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=60),
    retry=retry_if_exception_type((RateLimitError, TimeoutError)),
)
async def call_llm_with_retry(prompt: str) -> str:
    """带重试的 LLM 调用"""
    try:
        return await llm_client.call(prompt)
    except RateLimitError:
        logger.warning("LLM 限流，等待重试")
        raise
```

### 降级策略

```python
async def call_with_fallback(prompt: str, task_type: str) -> str:
    """带降级的 LLM 调用"""
    models = ["gpt-4-turbo", "gpt-3.5-turbo"]  # 主模型 → 备用模型

    for model in models:
        try:
            return await call_llm(prompt, model=model)
        except Exception as e:
            logger.warning(f"模型 {model} 调用失败，尝试降级", error=str(e))
            continue

    raise Exception("所有模型调用失败")
```

## 批量处理

### 批处理模式

```python
async def batch_process(
    items: list[str],
    task_type: str,
    batch_size: int = 10,
    concurrency: int = 5,
) -> list[dict]:
    """批量处理"""
    results = []
    semaphore = asyncio.Semaphore(concurrency)

    async def process_one(item: str) -> dict:
        async with semaphore:
            return await call_llm(item, task_type=task_type)

    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        batch_results = await asyncio.gather(
            *[process_one(item) for item in batch],
            return_exceptions=True,
        )
        results.extend(batch_results)

        # 批次间暂停，避免限流
        await asyncio.sleep(1)

    return results
```

## 人工审核 (HITL)

```python
class HITLQueue:
    """人工审核队列"""

    async def submit_for_review(
        self,
        call_record: LLMCallRecord,
        reason: str,
    ) -> None:
        """提交人工审核"""
        await db.hitl_queue.insert_one({
            "call_id": call_record.call_id,
            "task_type": call_record.task_type,
            "input_text": call_record.input_text,
            "output_text": call_record.output_text,
            "reason": reason,  # low_confidence / schema_invalid / random_sample
            "status": "pending",
            "created_at": datetime.utcnow(),
        })

    async def process_review(
        self,
        call_id: str,
        decision: str,  # approved / rejected
        corrected_output: str | None = None,
    ) -> None:
        """处理审核结果"""
        await db.hitl_queue.update_one(
            {"call_id": call_id},
            {"$set": {
                "status": decision,
                "corrected_output": corrected_output,
                "reviewed_at": datetime.utcnow(),
            }},
        )
```

## 禁止事项

- ❌ 禁止不记录调用日志
- ❌ 禁止不设置 timeout
- ❌ 禁止在 Prompt 中包含敏感信息
- ❌ 禁止不做成本监控
- ❌ 禁止信任未校验的 LLM 输出
- ❌ 禁止生产环境使用高 temperature

## 检查清单

- [ ] 每次调用记录 model/prompt/input/output
- [ ] 使用结构化输出 (JSON Schema / Function Calling)
- [ ] 有成本监控和预警
- [ ] 有重试和降级策略
- [ ] Prompt 版本化管理
- [ ] 敏感信息脱敏后再调用
- [ ] 有质量评测机制
