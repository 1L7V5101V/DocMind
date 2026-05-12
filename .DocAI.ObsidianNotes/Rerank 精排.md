Rerank 精排（Reranker.rerank）
对候选集的**每个片段**重新打分，支持三种策略：

| 策略 | 新分数计算方式 | 权重 |
|------|-------------|------|
| **BM25_FUSION** | `原分 × 0.6 + BM25分 × 0.4` | 偏向量 |
| **CROSS_ENCODER** | 调 DashScope rerank API（`gte-rerank-v2`）→ 返回 `relevance_score` | 全看 rerank 模型 |
| **HYBRID** | `原分 × 0.5 + (BM25分 + CrossEncoder分)/2 × 0.5` | 各一半 |

CROSS_ENCODER 和 HYBRID 会先尝试调 DashScope 远程 rerank API，不可用时降级到本地启发式重排（关键词匹配 + 位置权重）。

### 排序输出
按 `finalScore` 降序 → 取 topK 返回。

```
最终输出：finalResult = [{id, content, originalScore, rerankScore, finalScore, source, rerankProvider}]
```

一句话：**向量兜语义，BM25 补精确，Rerank 压掉误召回**。