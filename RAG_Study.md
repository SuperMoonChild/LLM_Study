
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


# RAG 面试 Cheatsheet — 重点易错知识点

> 专项复习：MRR、Precision & Recall、RAGAS诊断

---

## 1. Precision & Recall — 彻底搞清楚

### 核心公式

```
Precision = 正确结果数 / 你返回的总数        ← 分母是"你返回的"
Recall    = 正确结果数 / 数据库里总正确数     ← 分母是"全世界正确的"
```

### 最容易混淆的点

| 指标 | 问的问题 | 分母是谁 | 衡量什么 |
|---|---|---|---|
| Precision | 我说的对不对？ | 我返回的文档数 | 准不准 |
| Recall | 我找全了没？ | 数据库里所有正确文档数 | 全不全 |

### 计算例子（必须会）

```
场景：数据库里有50篇相关文档
      系统返回了20篇
      其中15篇是真正相关的

Precision = 15 / 20 = 75%   ← 除以"你返回的20篇"
Recall    = 15 / 50 = 30%   ← 除以"数据库里总共50篇"
```

### 生活记忆法

> 老师出了10道题，你答了8道，其中6道对了
> - Precision = 6/8 = 75%（你答的里面有多少对）
> - Recall    = 6/10 = 60%（全部题目里你对了多少）

### F1 Score — 平衡两者

```
F1 = 2 × (Precision × Recall) / (Precision + Recall)

例子：P=0.8，R=0.4
F1 = 2 × (0.8 × 0.4) / (0.8 + 0.4)
   = 2 × 0.32 / 1.2
   = 0.53

注意：F1比普通平均值(0.6)更低！调和平均数对低值更敏感
```

### Trade-off 记忆表

| 想提高 | 代价 | RAG里怎么做 |
|---|---|---|
| Recall ↑ | Precision ↓ | 撒大网：Hybrid Search，增大Top-K |
| Precision ↑ | Recall ↓ | 精挑细选：加Reranker精排 |
| 两者都要 | 两步走 | 先Hybrid Search保Recall → 再Reranker保Precision |

### 不同场景优先谁？

| 场景 | 优先 | 原因 |
|---|---|---|
| RAG检索阶段 | Recall | 漏掉文档→LLM没素材→答案不完整 |
| Reranking阶段 | Precision | 已召回，现在要过滤噪音 |
| 医疗诊断 | Recall | 漏诊比误诊危险 |
| 垃圾邮件过滤 | Precision | 误判正常邮件更烦 |

---

## 2. MRR — 平均倒数排名

### 核心概念

**MRR只看第一个正确答案排在第几位，越靠前越好**

### 倒数排名得分表（必背）

| 正确答案位置 | 得分 |
|---|---|
| 第1位 | 1/1 = **1.0** 🏆 |
| 第2位 | 1/2 = **0.5** |
| 第3位 | 1/3 = **0.33** |
| 第4位 | 1/4 = **0.25** |
| 第5位 | 1/5 = **0.20** |
| 没找到 | **0** |

### 完整计算例子

```
问题1：返回 [❌, ❌, ✅, ❌, ❌]  → 第3位 → 1/3 = 0.33
问题2：返回 [✅, ❌, ❌, ❌, ❌]  → 第1位 → 1/1 = 1.0
问题3：返回 [❌, ✅, ❌, ❌, ❌]  → 第2位 → 1/2 = 0.5

MRR = (0.33 + 1.0 + 0.5) / 3 = 1.83 / 3 = 0.61
```

### 关键点

- 只看**第一个**正确答案，后面有几个对的不管
- 多个问题取**平均值**才是最终MRR
- 用倒数是因为：第1→2位影响大(差0.5)，第9→10位影响小(差0.01)

### MRR vs Recall@K 的区别

| 指标 | 关注什么 | 适合场景 |
|---|---|---|
| MRR | 第一个正确答案的位置 | 用户只看第一个结果 |
| Recall@K | Top-K里有没有正确答案 | 用户会浏览多个结果 |

---

## 3. RAGAS框架诊断表 — 最重要！

### 四大指标含义

| 指标 | 衡量什么阶段 | 含义 |
|---|---|---|
| **Context Recall** | 检索阶段 | 相关文档有没有被召回 |
| **Context Precision** | 检索阶段 | 召回的文档有多少是真正相关的 |
| **Faithfulness** | 生成阶段 | 答案是否忠实于检索内容，有无幻觉 |
| **Answer Relevancy** | 生成阶段 | 答案是否回答了用户的问题 |

---

### 诊断 + 补救完整指南

#### ① Context Recall 低
```
症状：正确文档没被召回，LLM缺乏素材
根本原因：检索阶段漏掉了相关文档
```
**补救方案：**
- 换更强的Embedding模型（BGE-M3 / Qwen3-Embedding）
- 加入Hybrid Search（BM25 + 语义检索 + RRF）
- 增大Top-K召回数量（从5增到20）
- 优化Chunking策略（加Sliding Window overlap）
- 用HyDE：先让LLM生成假设答案再检索

---

#### ② Context Precision 低
```
症状：召回了太多无关文档，LLM被噪音干扰
根本原因：初步召回过于宽松
```
**补救方案：**
- 加入Reranker精排（BGE-reranker / Cohere Rerank）
- 减小最终传给LLM的Top-K数量
- 提高相似度阈值，过滤低分文档
- 用CRAG验证文档相关性

---

#### ③ Faithfulness 低
```
症状：LLM答案里有文档没有的内容（幻觉）
根本原因：生成阶段LLM在编造内容
```
**补救方案：**
- 改Prompt：明确加入"只能基于以下文档回答，不能编造"
- 用CRAG：生成后验证答案是否被文档支持
- 用Self-RAG：让LLM生成反思token自我评估
- 换更听话的LLM（GPT-4o比老版本更可控）

---

#### ④ Answer Relevancy 低
```
症状：答案跑题，没有回答用户真正的问题
根本原因：Query理解偏差或Prompt设计问题
```
**补救方案：**
- Query重写：用LLM先把用户问题改写得更清晰
- 改Prompt：明确"请直接回答用户的问题：{question}"
- 检查Chunking：chunk太大可能导致检索到不相关段落

---

### 一眼看懂诊断图

```
                    ┌─────────────────────────────┐
                    │     RAGAS 诊断速查           │
                    └─────────────────────────────┘

Context Recall 低   →  检索漏文档  →  换Embedding / Hybrid Search
Context Precision 低 → 噪音太多   →  加Reranker
Faithfulness 低     →  LLM幻觉    →  改Prompt / CRAG / Self-RAG  
Answer Relevancy 低 →  答案跑题   →  Query重写 / 改Prompt

组合诊断：
Context Recall高 + Faithfulness低  = 检索好但LLM在编造 → 改Prompt
Context Recall低 + Faithfulness高  = LLM很老实但没素材 → 改Embedding
两个Context都低                    = 根本没检索到好东西 → 重做检索系统
```

---

## 4. 面试万能回答模板

### 问：Precision和Recall有什么区别？
> "Precision衡量返回结果的质量——我返回的文档里有多少是真正相关的；Recall衡量覆盖度——数据库里所有相关文档我找到了多少。分母是关键区别：Precision除以我返回的总数，Recall除以全库相关文档总数。在RAG里，检索阶段优先Recall，Reranking阶段优先Precision。"

### 问：MRR是什么？
> "MRR是平均倒数排名，衡量第一个正确答案排在什么位置。排第1位得1.0分，第2位得0.5分，以此类推取倒数。多个问题取平均就是MRR。MRR低说明正确文档排名靠后，需要优化Reranker。"

### 问：RAG效果不好怎么排查？
> "我会用RAGAS框架系统排查：先看Context Recall，低说明检索漏文档，换更好的Embedding或加Hybrid Search；再看Context Precision，低说明噪音太多，加Reranker；再看Faithfulness，低说明LLM幻觉，改Prompt加约束；最后看Answer Relevancy，低说明答案跑题，做Query重写。"

---

*面试备考 | 2026年3月*
