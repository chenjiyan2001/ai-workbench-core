# Milvus 使用规范

## 适用范围

本规范适用于使用 Milvus 向量数据库进行向量存储、检索的场景。

## 命名规范

### Collection 命名

```
格式: {业务}_{实体}_{向量类型}
小写，下划线分隔

✅ 正确
doc_chunks_embedding         -- 文档片段向量
product_image_feature        -- 商品图片特征
user_query_history           -- 用户查询历史

❌ 错误
DocChunksEmbedding           -- 不要驼峰
doc-chunks-embedding         -- 不要横线
embedding                    -- 缺少业务前缀
```

### Field 命名

```
✅ 正确
id                           -- 主键
doc_id                       -- 文档 ID
chunk_index                  -- 片段索引
embedding                    -- 向量字段
text_content                 -- 原文内容
created_at                   -- 创建时间

❌ 错误
ID                           -- 应小写
docId                        -- 应使用下划线
vector                       -- 应更具体 (如 embedding)
```

## Collection 设计

### 基础结构

```python
from pymilvus import CollectionSchema, FieldSchema, DataType

# 定义字段
fields = [
    FieldSchema(
        name="id",
        dtype=DataType.VARCHAR,
        max_length=64,
        is_primary=True,
        description="主键",
    ),
    FieldSchema(
        name="doc_id",
        dtype=DataType.VARCHAR,
        max_length=64,
        description="文档 ID",
    ),
    FieldSchema(
        name="chunk_index",
        dtype=DataType.INT32,
        description="片段索引",
    ),
    FieldSchema(
        name="text_content",
        dtype=DataType.VARCHAR,
        max_length=65535,
        description="原文内容",
    ),
    FieldSchema(
        name="embedding",
        dtype=DataType.FLOAT_VECTOR,
        dim=1536,  # OpenAI text-embedding-ada-002
        description="文本向量",
    ),
    FieldSchema(
        name="created_at",
        dtype=DataType.INT64,
        description="创建时间戳",
    ),
]

schema = CollectionSchema(
    fields=fields,
    description="文档片段向量集合",
    enable_dynamic_field=True,  # 允许动态字段
)
```

### 向量维度参考

| Embedding 模型 | 维度 | 说明 |
|---------------|------|------|
| text-embedding-ada-002 | 1536 | OpenAI |
| text-embedding-3-small | 1536 | OpenAI |
| text-embedding-3-large | 3072 | OpenAI |
| bge-large-zh-v1.5 | 1024 | 智源 |
| bge-m3 | 1024 | 智源多语言 |
| m3e-base | 768 | Moka |

### 向量模型版本记录

```python
# ✅ 必须记录向量模型信息
COLLECTION_METADATA = {
    "collection_name": "doc_chunks_embedding",
    "embedding_model": "text-embedding-ada-002",
    "embedding_model_version": "v2",
    "embedding_dimension": 1536,
    "distance_metric": "IP",  # Inner Product
    "created_at": "2024-01-15",
}

# 可存储在 collection 的 description 或单独的元数据表中
```

## 索引配置

### 索引类型选择

| 索引类型 | 适用场景 | 特点 |
|---------|---------|------|
| IVF_FLAT | 小规模 (<100万) | 精确度高，速度中等 |
| IVF_SQ8 | 中规模 | 压缩存储，速度快 |
| IVF_PQ | 大规模 (>1000万) | 高压缩，速度快，精度略低 |
| HNSW | 通用推荐 | 高精度，高速度，内存占用大 |
| SCANN | 大规模 | Google 方案，平衡型 |

### HNSW 索引配置 (推荐)

```python
index_params = {
    "metric_type": "IP",  # IP (内积) 或 L2 (欧氏距离) 或 COSINE
    "index_type": "HNSW",
    "params": {
        "M": 16,              # 每层连接数 (8-64)
        "efConstruction": 256, # 构建时搜索宽度 (64-512)
    },
}

collection.create_index(
    field_name="embedding",
    index_params=index_params,
)
```

### IVF_FLAT 索引配置

```python
index_params = {
    "metric_type": "IP",
    "index_type": "IVF_FLAT",
    "params": {
        "nlist": 1024,  # 聚类中心数 (sqrt(n) ~ 4*sqrt(n))
    },
}
```

### 搜索参数配置

```python
# HNSW 搜索参数
search_params = {
    "metric_type": "IP",
    "params": {
        "ef": 64,  # 搜索时搜索宽度 (top_k ~ 2000)
    },
}

# IVF 搜索参数
search_params = {
    "metric_type": "IP",
    "params": {
        "nprobe": 16,  # 搜索的聚类数 (1 ~ nlist)
    },
}
```

## 操作规范

### 数据插入

```python
from pymilvus import Collection

collection = Collection("doc_chunks_embedding")

# ✅ 批量插入
data = [
    {"id": "1", "doc_id": "doc_001", "embedding": [0.1, 0.2, ...], ...},
    {"id": "2", "doc_id": "doc_001", "embedding": [0.3, 0.4, ...], ...},
    # ...
]

# 分批插入 (每批 1000-10000 条)
BATCH_SIZE = 5000
for i in range(0, len(data), BATCH_SIZE):
    batch = data[i:i + BATCH_SIZE]
    collection.insert(batch)

# 插入后刷新 (异步转同步)
collection.flush()

# ❌ 错误: 单条插入
for item in data:
    collection.insert([item])  # 性能差
```

### 向量搜索

```python
async def search_similar(
    query_embedding: list[float],
    top_k: int = 10,
    filter_expr: str | None = None,
) -> list[dict]:
    """相似向量搜索"""
    collection = Collection("doc_chunks_embedding")
    collection.load()  # 确保已加载到内存

    search_params = {
        "metric_type": "IP",
        "params": {"ef": max(top_k * 2, 64)},
    }

    results = collection.search(
        data=[query_embedding],
        anns_field="embedding",
        param=search_params,
        limit=top_k,
        expr=filter_expr,  # 例如: "doc_id == 'doc_001'"
        output_fields=["doc_id", "text_content", "chunk_index"],
    )

    return [
        {
            "id": hit.id,
            "score": hit.score,
            "doc_id": hit.entity.get("doc_id"),
            "text_content": hit.entity.get("text_content"),
        }
        for hit in results[0]
    ]
```

### 混合搜索 (向量 + 标量过滤)

```python
# ✅ 使用标量过滤减少搜索范围
results = collection.search(
    data=[query_embedding],
    anns_field="embedding",
    param=search_params,
    limit=10,
    expr="doc_id in ['doc_001', 'doc_002'] and created_at > 1704067200",
    output_fields=["doc_id", "text_content"],
)
```

### 删除数据

```python
# 按主键删除
collection.delete(expr="id in ['1', '2', '3']")

# 按条件删除
collection.delete(expr="doc_id == 'doc_001'")

# 删除后压缩 (清理已删除数据)
collection.compact()
```

## 性能优化

### 内存管理

```python
# 加载 collection 到内存
collection.load()

# 释放内存 (不常用的 collection)
collection.release()

# 检查加载状态
from pymilvus import utility
progress = utility.loading_progress("doc_chunks_embedding")
```

### 分区策略

```python
# 按时间或类别分区
collection.create_partition("partition_2024_01")
collection.create_partition("partition_2024_02")

# 插入到指定分区
collection.insert(data, partition_name="partition_2024_01")

# 搜索指定分区 (提高效率)
results = collection.search(
    data=[query_embedding],
    partition_names=["partition_2024_01"],
    ...
)
```

### 监控指标

```python
# 需要监控的指标
MILVUS_METRICS = {
    "collection_row_count": "向量数量",
    "index_build_progress": "索引构建进度",
    "query_latency_p99": "查询延迟 P99",
    "memory_usage": "内存使用",
    "recall_rate": "召回率 (需要评测)",
}
```

## 向量质量保障

### Embedding 一致性

```python
# ✅ 确保查询和存储使用相同的 embedding 配置
EMBEDDING_CONFIG = {
    "model": "text-embedding-ada-002",
    "normalize": True,  # 归一化 (使用 IP 时)
    "max_tokens": 8191,
    "truncation": "end",
}

def get_embedding(text: str) -> list[float]:
    """统一的 embedding 生成函数"""
    # 使用统一配置
    embedding = openai_client.embeddings.create(
        model=EMBEDDING_CONFIG["model"],
        input=text[:EMBEDDING_CONFIG["max_tokens"]],
    ).data[0].embedding

    if EMBEDDING_CONFIG["normalize"]:
        embedding = normalize(embedding)

    return embedding
```

### 召回率评测

```python
async def evaluate_recall(
    test_queries: list[dict],
    ground_truth: dict[str, list[str]],
    top_k: int = 10,
) -> float:
    """评测召回率"""
    hits = 0
    total = 0

    for query in test_queries:
        results = await search_similar(query["embedding"], top_k=top_k)
        result_ids = {r["id"] for r in results}
        expected_ids = set(ground_truth[query["id"]])

        hits += len(result_ids & expected_ids)
        total += len(expected_ids)

    return hits / total if total > 0 else 0
```

## 禁止事项

- ❌ 禁止混用不同模型的 embedding
- ❌ 禁止不记录向量模型版本
- ❌ 禁止单条插入大量数据
- ❌ 禁止长时间不释放不用的 collection
- ❌ 禁止忽略索引构建

## 检查清单

- [ ] Collection/Field 命名符合规范
- [ ] 向量维度与 embedding 模型匹配
- [ ] 记录了 embedding 模型版本
- [ ] 创建了合适的索引
- [ ] 搜索参数已调优
- [ ] 有召回率评测机制
- [ ] 大数据量使用分批插入
