
<img width="956" height="638" alt="image" src="https://github.com/user-attachments/assets/fbae90d4-952c-42a2-bb66-e4e3c5b4bf65" />

# RAG 评估指南 — Embedding模型效果测试

> 面试知识点整理 | RAG + Agents

---

## 目录
1. [为什么需要评估？](#1-为什么需要评估)
2. [第一步：建立测试集 Ground Truth](#2-第一步建立测试集-ground-truth)
3. [第二步：检索阶段评估指标](#3-第二步检索阶段评估指标)
   - Recall@K
   - Precision@K
   - MRR
4. [第三步：端到端评估 RAGAS](#4-第三步端到端评估-ragas)
5. [第四步：A/B对比不同模型](#5-第四步ab对比不同模型)
6. [指标诊断对照表](#6-指标诊断对照表)
7. [常见公开Embedding模型对比](#7-常见公开embedding模型对比)
8. [面试回答模板](#8-面试回答模板)

---

## 1. 为什么需要评估？

MTEB leaderboard的分数是通用benchmark，**不代表在你自己数据上的效果**。

必须在自己的业务数据上测试，才能选出最合适的模型。不同场景差异很大：
- 中英混合数据 → BGE-M3 可能优于 OpenAI
- 垂直领域（金融/医疗） → 领域fine-tune模型更好
- 资源受限 → 轻量模型如 all-MiniLM-L6-v2

---

## 2. 第一步：建立测试集 Ground Truth

测试集包含：
- **问题列表** — 真实用户会问的问题（至少50-100条）
- **对应正确文档** — 每个问题对应哪些chunk是正确答案

```python
test_set = [
    {
        "question": "RAG的chunking策略有哪些？",
        "relevant_chunks": ["chunk_id_23", "chunk_id_45"]
    },
    {
        "question": "如何降低RAG延迟？",
        "relevant_chunks": ["chunk_id_67"]
    }
]
```

> 💡 **技巧**：可以用LLM自动生成问题，再人工过滤，大幅降低标注成本（称为 **Synthetic Test Set Generation**）

```python
# 用LLM自动生成测试问题
from ragas.testset import TestsetGenerator

generator = TestsetGenerator.with_openai()
testset = generator.generate_with_langchain_docs(
    documents, 
    test_size=100
)
```

---

## 3. 第二步：检索阶段评估指标

### 3.1 Recall@K — 召回率

**含义**：Top-K个召回结果里，正确答案有没有被找到？

$$Recall@K = \frac{|\text{正确答案} \cap \text{召回Top-K}|}{|\text{正确答案}|}$$

```python
def recall_at_k(retrieved, relevant, k=5):
    """
    retrieved: 检索返回的chunk id列表（按相关性排序）
    relevant:  ground truth中正确的chunk id列表
    k:         取前K个结果
    """
    retrieved_k = retrieved[:k]
    hits = len(set(retrieved_k) & set(relevant))
    return hits / len(relevant)

# 示例
retrieved = ["chunk_23", "chunk_99", "chunk_45", "chunk_7", "chunk_12"]
relevant  = ["chunk_23", "chunk_45"]
print(recall_at_k(retrieved, relevant, k=5))  # 输出：1.0（两个都找到了）
print(recall_at_k(retrieved, relevant, k=2))  # 输出：0.5（只找到chunk_23）
```

**Recall@K低** → embedding模型不够好，或chunking策略有问题

---

### 3.2 Precision@K — 精准率

**含义**：Top-K个召回结果里，有多少是真正有用的？

$$Precision@K = \frac{|\text{正确答案} \cap \text{召回Top-K}|}{K}$$

```python
def precision_at_k(retrieved, relevant, k=5):
    retrieved_k = retrieved[:k]
    hits = len(set(retrieved_k) & set(relevant))
    return hits / k

# 示例
print(precision_at_k(retrieved, relevant, k=5))  # 2/5 = 0.4
print(precision_at_k(retrieved, relevant, k=2))  # 1/2 = 0.5
```

**Precision@K低** → 召回了太多噪音，需要加Reranker精排

---

### 3.3 MRR — 平均倒数排名 (Mean Reciprocal Rank)

**含义**：正确答案排在第几位？越靠前越好。

$$MRR = \frac{1}{|Q|} \sum_{i=1}^{|Q|} \frac{1}{rank_i}$$

| 正确答案排名 | 倒数排名得分 |
|---|---|
| 第1位 | 1.0 |
| 第2位 | 0.5 |
| 第3位 | 0.33 |
| 第5位 | 0.2 |
| 未找到 | 0 |

```python
def mrr(retrieved, relevant):
    """计算单个query的倒数排名"""
    for i, chunk in enumerate(retrieved):
        if chunk in relevant:
            return 1 / (i + 1)
    return 0  # 没找到

def mean_mrr(test_set, retrieval_fn, k=10):
    """计算整个测试集的平均MRR"""
    scores = []
    for item in test_set:
        retrieved = retrieval_fn(item["question"], k=k)
        score = mrr(retrieved, item["relevant_chunks"])
        scores.append(score)
    return sum(scores) / len(scores)
```

---

### 3.4 完整检索评估代码

```python
from sentence_transformers import SentenceTransformer
import numpy as np

def evaluate_embedding_model(model_name, test_set, vector_db, k=5):
    model = SentenceTransformer(model_name)
    
    recall_scores = []
    precision_scores = []
    mrr_scores = []
    
    for item in test_set:
        # 向量化问题
        query_emb = model.encode(item["question"])
        
        # 检索Top-K
        retrieved = vector_db.search(query_emb, top_k=k)
        retrieved_ids = [r.id for r in retrieved]
        relevant_ids = item["relevant_chunks"]
        
        # 计算指标
        recall_scores.append(recall_at_k(retrieved_ids, relevant_ids, k))
        precision_scores.append(precision_at_k(retrieved_ids, relevant_ids, k))
        mrr_scores.append(mrr(retrieved_ids, relevant_ids))
    
    return {
        "model": model_name,
        f"Recall@{k}": np.mean(recall_scores),
        f"Precision@{k}": np.mean(precision_scores),
        "MRR": np.mean(mrr_scores)
    }
```

---

## 4. 第三步：端到端评估 RAGAS

RAGAS 是最流行的RAG端到端评估框架，同时评估检索+生成质量。

### 4.1 安装

```bash
pip install ragas langchain openai
```

### 4.2 四大核心指标

| 指标 | 含义 | 评估对象 |
|---|---|---|
| **Context Recall** | 相关文档有没有被召回 | 检索阶段 |
| **Context Precision** | 召回的文档有多少是相关的 | 检索阶段 |
| **Faithfulness** | 答案是否忠实于检索内容，有无幻觉 | 生成阶段 |
| **Answer Relevancy** | 答案是否回答了用户问题 | 生成阶段 |

### 4.3 使用示例

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision
)
from datasets import Dataset

# 准备数据格式
data = {
    "question": ["RAG的chunking策略有哪些？", "如何降低延迟？"],
    "answer": ["RAG的chunking策略包括...", "可以通过ANN搜索..."],
    "contexts": [
        ["chunk1内容", "chunk2内容"],  # 检索到的文档
        ["chunk3内容"]
    ],
    "ground_truth": ["正确答案1", "正确答案2"]  # 可选
}

dataset = Dataset.from_dict(data)

# 运行评估
results = evaluate(
    dataset=dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_recall,
        context_precision
    ]
)

print(results)
# 输出示例：
# faithfulness        0.89
# answer_relevancy    0.92
# context_recall      0.78   ← 低说明embedding不好！
# context_precision   0.85
```

---

## 5. 第四步：A/B对比不同模型

```python
models_to_compare = [
    "BAAI/bge-large-en-v1.5",       # 开源，中英文强
    "BAAI/bge-m3",                   # 开源，多语言
    "text-embedding-3-large",        # OpenAI，效果好但收费
    "Qwen3-Embedding-8B",            # 阿里，多语言SOTA
]

results = []
for model_name in models_to_compare:
    result = evaluate_embedding_model(model_name, test_set, vector_db, k=5)
    results.append(result)

# 排序选最好的
results.sort(key=lambda x: x["Recall@5"], reverse=True)
for r in results:
    print(f"{r['model']}: Recall@5={r['Recall@5']:.3f}, MRR={r['MRR']:.3f}")
```

---

## 6. 指标诊断对照表

| 现象 | 说明问题在哪 | 解决方案 |
|---|---|---|
| **Context Recall 低** | 正确文档没被召回 | 换更好的embedding / 改chunking策略 / 加Hybrid Search |
| **Context Precision 低** | 召回太多无关文档 | 加Reranker精排 / 减小top-k |
| **Faithfulness 低** | LLM答案超出文档范围（幻觉） | 改Prompt加约束 / 用CRAG验证文档 |
| **Answer Relevancy 低** | 答案跑题 | Query重写 / 改Prompt / 检查chunking |
| **MRR 低** | 正确答案排名靠后 | 加Reranker / 调整相似度算法 |

---

## 7. 常见公开Embedding模型对比

### 开源免费（可自部署）

| 模型 | 机构 | 参数量 | 多语言 | 特点 |
|---|---|---|---|---|
| **Qwen3-Embedding-8B** | 阿里巴巴 | 8B | ✅ 100+语言 | MTEB多语言榜最强开源，Apache 2.0 |
| **BGE-M3** | BAAI北京智源 | 568M | ✅ 100+语言 | 开源首选，中英文强 |
| **BGE-large-en-v1.5** | BAAI | 335M | ❌ 英文为主 | 英文效果优秀 |
| **E5-large-v2** | Microsoft | 335M | ❌ 英文 | 微软出品，英文RAG常用 |
| **multilingual-E5** | Microsoft | 560M | ✅ 多语言 | E5的多语言版 |
| **Nomic Embed Text** | Nomic AI | 137M | ❌ 英文 | 超轻量，开源，长文档支持好 |
| **EmbeddingGemma-300M** | Google | 300M | ✅ 100+语言 | 可在手机/笔记本运行 |
| **all-MiniLM-L6-v2** | sentence-transformers | 22M | ❌ 英文 | 超小超快，入门首选 |

### 商业API（收费）

| 模型 | 机构 | MTEB分 | 特点 |
|---|---|---|---|
| **Gemini Embedding** | Google | 🥇 最高 | 目前MTEB第一，支持Vertex AI |
| **text-embedding-3-large** | OpenAI | 64.6 | 生态最好，接入最简单 |
| **text-embedding-3-small** | OpenAI | 62.3 | 更便宜，速度更快 |
| **Cohere Embed v4** | Cohere | 65.2 | 企业级，支持私有云部署 |
| **Voyage AI** | Voyage (MongoDB) | 高 | 多模态，代码检索强 |
| **Jina Embeddings v3** | Jina AI | 高 | 长文档，8192 token上下文 |

### 如何选择

```
预算充足 + 简单接入    →  OpenAI text-embedding-3-large
中英文 + 开源免费      →  BGE-M3 或 Qwen3-Embedding
超轻量 + 本地部署      →  all-MiniLM-L6-v2 或 Nomic Embed
企业级 + 多模态        →  Cohere Embed v4 / Voyage
多语言SOTA + 开源      →  Qwen3-Embedding-8B
```

> ⚠️ **重要**：永远在自己的数据上测试！领域fine-tune可以提升10-30%的效果。

---

## 8. 面试回答模板

### 问：如何评估Embedding模型效果？

> "我评估embedding效果的流程分四步：
>
> 第一步，用真实业务问题构建Ground Truth测试集，可以用LLM辅助自动生成问题再人工过滤。
>
> 第二步，用Recall@5和MRR衡量检索阶段的质量——Recall看正确文档有没有被召回，MRR看正确答案排第几位。
>
> 第三步，用RAGAS框架端到端评估，包括Context Recall、Context Precision、Faithfulness和Answer Relevancy四个指标。
>
> 第四步，A/B对比多个候选模型，选指标最好的。
>
> 在我的项目里，通过这个方法发现BGE-M3在我们垂直领域数据上的Context Recall比OpenAI embedding高了约8个百分点，而且完全免费。"

### 问：Recall低了怎么办？

> "Recall低说明正确文档没被召回，可以从三个方向优化：一是换更好的embedding模型；二是改进chunking策略，比如加sliding window overlap保证语义连贯；三是引入Hybrid Search，把BM25关键词检索和语义检索结合起来，用RRF算法合并排名，覆盖不同类型的query。"

---

## 参考资源

- [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — Embedding模型实时排行榜
- [RAGAS文档](https://docs.ragas.io) — RAG评估框架
- [LangChain RAG教程](https://python.langchain.com/docs/use_cases/question_answering/)
- [BGE-M3 论文](https://arxiv.org/abs/2402.03216)

---

<img width="691" height="542" alt="image" src="https://github.com/user-attachments/assets/1dce826a-b47e-4ffe-9bd2-1e9a06086767" />

如果recall 很低说明文档没有找全，需要改变embedding 还有search methods 
如果precision 很低 召回很多无关文档 ，改变reranker 


*整理时间：2026年3月 | 面试备考笔记*
