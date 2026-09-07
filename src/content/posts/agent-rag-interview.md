---
title: 41道 Agent & RAG 高频面试题
published: 2026-09-07
description: Agent 与 RAG 高频面试题整理，涵盖架构设计、Memory、Tool Use、检索与评估。
tags: [Agent, RAG, 面试]
category: 学习笔记
draft: false
---





## 1. AI Agent 与普通 LLM API 的区别？

LLM：

```text
input → output
```

Agent：

```text
goal → reason → action → observation → state update → repeat
```

核心区别不是“用了工具”，而是**存在自主决策闭环**。

---

## 2. perception-planning-action 是什么？

Perception：

> 接收用户输入、Tool Result、环境状态、RAG Context。

Planning：

> 判断目标、拆任务、决定下一动作。

Action：

> Tool call/API/code/search/SQL。

然后 Observation 回到 State。

---

## 3. 为什么 Agent 需要 Tool Use？

因为 LLM：

- 不具备实时世界状态；

- 不应该直接计算复杂业务；

- 无法直接操作数据库；

- 知识存在截止时间。

Tool 把模型从：

> knowledge model

扩展成：

> action system。

---

## 4. Agent 和 Workflow Automation 区别？

Workflow：

```text
路径基本提前确定
```

Agent：

```text
路径由状态动态决定
```

实际生产一般是：

> **Agent + Workflow hybrid**

你的项目就是这种路线。

---

## 5. 什么叫 goal-driven execution？

系统不是只生成一次回答，而是根据目标不断决定下一步，直到：

```text
success
failure
timeout
max_steps
human intervention
```

之一。

---

## 6. Agent 和 RPA 区别？

RPA：

> 固定规则操作 UI/API。

Agent：

> 基于语义和环境动态决策。

---

## 7. 为什么 Agent 比 Chatbot 复杂？

因为多了：

- State

- Tool

- Memory

- Planning

- Error recovery

- Loop

- Permissions

- Observability

---

## 8. Autonomy 怎么定义？

不是“能不能自己运行”。

而是：

> **系统在多少决策点上可以不经过用户，自主选择行动。**

可以分：

```text
L0 纯 Workflow
L1 Tool selection
L2 Task planning
L3 Replanning
L4 Long-horizon autonomy
```

---

## 9. 为什么需要 Memory？

因为 LLM API 本身无状态。

没有 Memory：

> 每轮都是全新的请求。

---

## 10. Long-term / Short-term 怎么设计？

你的项目答案直接用：

```text
Redis List → session memory
Redis Hash → user preference
Summary → context compression
Shared Memory → cross-agent state
```

---

## 11. Agent loop 是什么？

经典：

```text
Think
↓
Act
↓
Observe
↓
Think
```

实际上生产系统一般不会无限 ReAct，而会加：

```text
max_steps
timeout
budget
stop condition
```

---

## 12. Agent 为什么容易 hallucination？

除了 LLM 本身，还因为：

- 错误 observation；

- tool result 被误读；

- stale memory；

- planning error；

- state propagation error；

- retrieval error；

- 多步错误累积。

所以 Agent hallucination 往往是**系统级问题**。

---

# 13. 设计完整 Agent 架构

建议你直接画：

```text
                 User
                   │
                Gateway
                   │
              Orchestrator
             /      |      \
        Planner   Memory   Policy
             \      |      /
                Agent State
                   │
             Tool Registry
        ┌──────────┼──────────┐
       RAG        SQL        API
        │
     Environment
        │
    Observation
        │
     State Update
        │
     Synthesizer
```

外面：

```text
Trace
Metrics
Guardrail
Retry
Checkpoint
Evaluation
```

---

# 14. Planner 职责？

负责：

- goal interpretation；

- decomposition；

- dependency；

- tool selection；

- sequencing；

- replanning。

不应该承担实际业务执行。

---

# 15. 为什么 Task Decomposition？

因为复杂任务直接一次 prompt：

- 推理难；

- 无法并行；

- 无法单步重试；

- 无法定位失败。

拆成 Task 后可以单独 trace。

---

# 16. Agent 怎么自动拆任务？

最简单：

```text
LLM → JSON Plan
```

例如：

```json
{
  "tasks": [
    {"id": "A", "tool": "rag"},
    {"id": "B", "tool": "sql"},
    {"id": "C", "depends_on": ["A", "B"]}
  ]
}
```

再交给 Runtime。

---

# 17. Orchestration Layer 要解决什么？

重点：

- state；

- routing；

- dependency；

- concurrency；

- retry；

- timeout；

- cancellation；

- resource limit；

- checkpoint；

- tracing。

---

# 18. LangChain / AutoGPT / CrewAI 怎么区分？

一句话版：

> LangChain 偏通用 LLM application components；LangGraph 偏 stateful orchestration；CrewAI 偏 role-based multi-agent；AutoGPT 是 autonomous agent 路线代表。

面试最好主动提 LangGraph。

---

# 19. Agent 如何调用 API / Tool？

通常：

```text
Tool Schema
↓
LLM Tool Selection
↓
Argument Validation
↓
Runtime Execute
↓
Observation
↓
LLM
```

不要直接：

```python
eval(model_output)
```

---

# 20. Tool Registry 怎么设计？

基本结构：

```python
Tool:
    name
    description
    schema
    permission
    timeout
    retry
    execute()
```

Registry：

```text
register
discover
get
invoke
```

你的项目已有类似 Tool Registry。

---

# 21. 如何避免无限 Tool Loop？

至少：

- max steps；

- timeout；

- tool call fingerprint；

- repeated-action detection；

- token budget；

- cost budget；

- no-progress detection；

- stop condition。

---

# 22. Observation 是什么？

Tool/环境执行后返回给 Agent 的状态。

例如：

```text
SQL result
HTTP response
search result
file content
error
```

---

# 23. Agent State 如何管理？

区分：

```text
Conversation State
Execution State
Business State
Memory State
```

不要全部塞 messages。

你的 `AgentState` 就包含：

- query

- stock

- intent

- workflow_id

- skill_results

- memory_context

- final_answer

- error

---

# 24. Checkpoint / Resume 怎么设计？

每个关键 Node 后保存：

```text
run_id
state
node
task_status
tool_result
timestamp
```

恢复时：

```text
load checkpoint
↓
rebuild DAG
↓
skip completed task
↓
continue
```

注意工具的 **idempotency**。

---

# 25. 百万用户 Agent 平台怎么设计？

不能直接百万用户 → 百万 LLM 调用。

至少：

```text
Gateway
↓
Rate Limit
↓
Queue
↓
Agent Runtime Pool
↓
Model Gateway
↓
Multiple Model Provider
```

同时：

- Redis State；

- distributed tracing；

- tool isolation；

- async execution；

- cache；

- tenant isolation；

- quota；

- circuit breaker。

---

# 26. 为什么 Agent 需要 RAG？

Agent 负责：

> 决策。

RAG 负责：

> grounding。

它们不是同一件事。

---

# 27. RAG 完整 Pipeline？

```text
Document
↓
Parse
↓
Clean
↓
Chunk
↓
Metadata
↓
Embedding
↓
Index

Query
↓
Rewrite
↓
Retrieve
↓
Filter
↓
Fusion
↓
Rerank
↓
Context
↓
LLM
```

---

# 28. 怎么提高 Recall？

优先：

1. better chunking；

2. metadata；

3. hybrid search；

4. query expansion；

5. top-k；

6. embedding；

7. index parameter。

不要第一反应只增大 top-k。

---

# 29. 怎么降低 RAG hallucination？

四层：

```text
Retrieval quality
↓
Context quality
↓
Generation constraint
↓
Verification
```

例如：

- citation；

- evidence-required prompt；

- no evidence → abstain；

- reranker；

- metadata；

- answer verification。

---

# 30. 向量 + Keyword 怎么结合？

你项目就是：

```text
Dense
+
BM25
↓
RRF
```

非常标准。

---

# 31. 什么是 Hybrid Search？

结合：

```text
semantic similarity
+
lexical exact match
```

Dense 擅长语义；

BM25 擅长：

- 公司名；

- 数字；

- 专有名词；

- 精确关键词。

---

# 32. 100M 文档 RAG 怎么设计？

不要做一个巨大单索引直接搜。

通常：

```text
Query Router
↓
Metadata Partition
↓
Shard
↓
Sparse + Dense Retrieval
↓
Candidate Merge
↓
Rerank
```

考虑：

- distributed vector DB；

- shard；

- replication；

- hot/cold storage；

- metadata partition；

- cache；

- batch embedding。

---

# 33. Chunk Size 怎么选？

核心 tradeoff：

小 chunk：

> 定位精确，但上下文不足。

大 chunk：

> 信息完整，但 embedding 信息稀释。

所以根据：

- document structure；

- task；

- embedding model；

- reranker context。

调。

---

# 34. Rerank 作用是什么？

Retriever：

> 高 Recall。

Reranker：

> 高 Precision。

典型：

```text
100 candidates
→ rerank
→ top 5
```

---

# 35. Embedding Model 怎么选？

看：

- language；

- domain；

- dimension；

- latency；

- benchmark；

- deployment cost。

不能只看 MTEB 排名。

---

# 36. RAG 怎么评估？

Retrieval：

```text
Recall@K
MRR
NDCG
Precision@K
```

Generation：

```text
faithfulness
answer relevance
citation accuracy
```

你当前公开评估脚本明确实现了 Recall@1/3/5/10 和 MRR。

这里提醒你一个细节：

README 写了 NDCG，但当前 `evaluate_retrieval.py` **没有实现 NDCG**。

所以面试不要说：

> “我的评估脚本里算了 NDCG。”

最好说：

> “核心可复现指标是 Recall@K 和 MRR。”

---

# 37. 为什么 Metadata Filtering？

例如：

> “万华化学 2024 年毛利率”

如果只 Dense：

可能搜到：

> 2021 年；  
> 其他公司。

所以先 filter：

```text
company=万华
year=2024
```

再 vector search。

---

# 38. 检索结果不稳定怎么办？

检查：

1. embedding；

2. chunk；

3. ANN；

4. query rewrite；

5. top-k；

6. metadata；

7. reranker；

8. duplicate；

9. index consistency。

一定要建固定 eval set。

---

# 39. Vector DB 索引怎么设计？

你项目：

```text
Dense:
384 dimension
Cosine

Sparse:
BM25 Sparse Index
```

生产进一步考虑：

- HNSW；

- IVF；

- PQ；

- shard；

- payload index；

- replication。

---

# 40. 怎么降低 RAG Latency？

```text
query cache
embedding cache
parallel dense/sparse
reduce candidate
fast reranker
metadata pruning
batch
async I/O
model warmup
```

另外不要把大量无用 context 交给 LLM。

---

# 41. 什么是 ReAct？

经典：

```text
Reason
↓
Act
↓
Observe
↓
Reason
```

优点：

动态。

最大问题：

> 长轨迹容易错误积累和无限循环。


