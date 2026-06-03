# RAG 知识库问答机器人（ClawBot）面试题

## Q1：三级检索架构是怎么设计的？为什么要分三级？

三级检索架构为：关键词匹配 → RAG 语义检索 → LLM 兜底。

设计思路：
1. **关键词匹配**（第一级）：成本最低、速度最快。对于精确匹配的问题（如"快应用manifest.json字段说明"），直接命中预设 QA 对，无需调用向量检索和大模型，节省 token 开销。
2. **RAG 语义检索**（第二级）：用户问题语义模糊时，通过 Embedding 将 query 向量化，在向量库中做余弦相似度检索，召回 top-k 文档片段，拼接进 prompt 让 LLM 生成回答。
3. **LLM 兜底**（第三级）：当前两级都无匹配（相似度低于阈值），退化为纯 LLM 通用问答，保证用户不会收到"无法回答"的冷回复。

分级的核心目的是**成本与质量的梯度权衡**——能用便宜方式解决的不上贵的，同时保证覆盖率。

### 追问1：关键词匹配你用的什么算法？怎么判断走第一级还是第二级？

关键词匹配层用的是 TF-IDF（词频-逆文档频率）。判断逻辑：

```javascript
const tfidfScore = tfidfSearch(query, docs);

if (tfidfScore >= THRESHOLD_HIGH) {
  // 关键词精确命中，直接返回
  return tfidfResult;
} else if (tfidfScore >= THRESHOLD_LOW) {
  // 模糊地带，走 RAG 语义检索
  return ragSearch(query);
} else {
  // 完全无匹配，走 LLM 兜底
  return llmFallback(query);
}
```

TF-IDF 的优势：纯本地计算，无网络开销，对"manifest.json 配置"这种关键词明确的查询命中率很高。缺点是无法理解同义词（如"怎么打包"和"如何构建"在 TF-IDF 下完全不同，但语义相近）。

### 追问2：阈值怎么定的？有没有做过评估？

- 初期靠人工标注测试集（约 100 条 QA 对）调参
- 观察指标：命中率、回答满意度（人工抽检）
- TF-IDF 阈值大概在 0.6 以上算强匹配，0.3-0.6 走 RAG，0.3 以下走兜底
- 向量检索的余弦相似度阈值在 0.75 左右，低于此阈值认为知识库无相关内容
- 最终通过预估人工介入率下降 40% 来验证整体效果

### 追问3：三级之间的降级是同步还是异步？延迟怎么控制？

是**串行同步降级**，但总延迟可控：

| 级别 | 耗时 | 说明 |
|------|------|------|
| 关键词匹配 | < 10ms | 本地计算，无 IO |
| RAG 语义检索 | 200-500ms | 向量相似度计算 + 召回 |
| LLM 生成 | 1-3s | 流式输出，用户体感还行 |

因为第一级极快，所以串行不会有累加延迟问题。如果命中第一级，整体 < 50ms 就能返回。

---

## Q2：Embedding 向量缓存是怎么设计的？

采用 MD5 哈希 + 批量请求，避免重复调用降低成本。

具体方案：
- 对每段文档内容计算 MD5 哈希作为 cache key
- 首次构建时批量调用 Embedding API，将结果持久化到本地（JSON/文件）
- 文档更新时，只对 MD5 变化的片段重新请求 Embedding，未变化的直接复用缓存
- 这样在文档频繁迭代时可以节省 70%+ 的 API 调用量

### 追问1：缓存存在哪里？为什么不用 Redis？

缓存存在本地 JSON 文件中（结构大概是 `{ [md5]: float[] }`）。

不用 Redis 的原因：
- 这是一个单机部署的微信机器人，没有分布式需求
- 文档库规模不大（几百篇 Markdown），全部向量加载进内存也就几十 MB
- 本地文件足够，启动时一次性 load 进内存，检索时直接内存计算
- 避免多引入一个 Redis 依赖增加运维复杂度

如果规模增长到万级文档，会考虑换成 FAISS 或 Chroma。

### 追问2：文档更新时怎么做增量更新的？

```javascript
async function updateEmbeddings(docs) {
  const cache = loadCache();
  const toEmbed = [];
  
  for (const doc of docs) {
    const hash = md5(doc.content);
    if (!cache[hash]) {
      toEmbed.push({ hash, content: doc.content });
    }
  }
  
  if (toEmbed.length > 0) {
    // 批量请求，减少 API 调用次数
    const vectors = await batchEmbed(toEmbed.map(d => d.content));
    toEmbed.forEach((d, i) => { cache[d.hash] = vectors[i]; });
    saveCache(cache);
  }
  
  // 清理已删除文档的缓存
  cleanStaleEntries(cache, docs);
}
```

关键点：
- MD5 哈希作为内容指纹，内容不变 = 向量不变
- 批量请求（一次传多条文本）降低 API 调用次数
- 文档删除时清理孤立缓存条目，防止缓存膨胀

### 追问3：Embedding 模型用的什么？维度多少？

用的 OpenAI 兼容 API（可能是通义千问或其他国产模型的 Embedding 接口），维度通常是 1536（text-embedding-ada-002）或 1024（通义）。

选型考虑：
- 中文语义理解能力（快应用文档全是中文）
- API 稳定性和成本
- 与生成模型共用同一个 API provider 降低集成成本

---

## Q3：分段（Chunk）策略是怎么做的？怎么优化的？

### chunk size 设多大？怎么确定的？

- chunk size 设在 500-800 字符左右（中文）
- 确定方式：
  - 太小（< 200）：语义不完整，检索到了也回答不好
  - 太大（> 1500）：噪声多，且占用上下文窗口，能塞入 prompt 的 chunk 数量变少
  - 结合知识库文档特点（Vela 快应用文档多是按 API/组件分节），500-800 刚好覆盖一个完整知识点

### 检索重排（Rerank）是怎么做的？

1. 向量检索先召回 top-10 个 chunk
2. 对这 10 个 chunk 做二次排序：
   - **相似度分数**权重最高
   - **文档来源权重**：官方文档 > 示例代码 > FAQ
   - **新鲜度**：最近更新的文档优先（应对 API 版本变更）
3. 取 top-3 拼入 prompt

没有用独立的 Rerank 模型（如 Cohere Rerank），因为规模小，基于规则的重排够用。

### prompt 怎么构造的？

```
你是 Vela 快应用技术支持助手。请根据以下参考文档回答用户问题。
如果参考文档中没有相关信息，请明确告知用户你不确定。

【参考文档】
{chunk_1}
{chunk_2}
{chunk_3}

【用户问题】
{query}

【回答要求】
1. 直接回答问题，不要重复问题本身
2. 如果涉及代码，给出示例
3. 如果不确定，说明哪部分是推测
```

没用 few-shot，原因：
- 上下文窗口有限，留给检索结果
- 快应用文档问答格式比较统一，zero-shot 效果已经够用
- 但在 system prompt 里做了角色设定和输出格式约束

---

## Q4：TF-IDF 和 Embedding 双引擎为什么要支持热切换？

原因：
- **TF-IDF**：无需外部 API，本地计算，零成本，适合关键词明确的查询，但语义理解弱
- **Embedding**：语义理解强，但依赖外部 API，有延迟和成本
- 热切换的意义：当 Embedding API 不可用（限流/故障）时自动降级到 TF-IDF；开发调试阶段用 TF-IDF 避免烧钱；可以通过配置按场景选择引擎

### 热切换怎么实现的？

```javascript
// 引擎抽象接口
class SearchEngine {
  search(query) { throw new Error('not implemented'); }
}

class TFIDFEngine extends SearchEngine { /* ... */ }
class EmbeddingEngine extends SearchEngine { /* ... */ }

// 引擎管理器
class EngineManager {
  constructor() {
    this.engines = {
      tfidf: new TFIDFEngine(),
      embedding: new EmbeddingEngine()
    };
    this.current = 'embedding';
  }
  
  switch(engineName) {
    this.current = engineName; // 运行时切换，无需重启
  }
  
  search(query) {
    return this.engines[this.current].search(query);
  }
}
```

通过策略模式实现，切换只是改变引用指向，无需重启进程。

### Embedding API 挂了怎么自动降级？

```javascript
async search(query) {
  try {
    const result = await this.engines.embedding.search(query);
    this.failCount = 0;
    return result;
  } catch (err) {
    this.failCount++;
    
    // 连续失败 3 次，自动切换到 TF-IDF
    if (this.failCount >= 3) {
      this.current = 'tfidf';
      // 设置定时器，5 分钟后尝试恢复
      setTimeout(() => this.tryRecover(), 5 * 60 * 1000);
    }
    
    return this.engines.tfidf.search(query);
  }
}
```

核心是**熔断 + 自动恢复**的思路，借鉴了后端微服务的熔断器模式。

---

## Q5：如果让你重新设计 ClawBot，你会怎么改进？

1. **向量存储升级**：从 JSON 文件换到 ChromaDB 或 FAISS，支持更大规模文档和更快检索
2. **Rerank 模型引入**：用 BGE-Reranker 或 Cohere Rerank 替代规则排序，提升召回质量
3. **对话记忆**：当前是单轮问答，加入对话历史窗口支持多轮追问
4. **评估体系**：建立自动化评估 pipeline（标注集 + 自动计算 recall/precision/MRR）
5. **混合检索**：BM25 + 向量检索融合，而不是串行降级，综合得分更准
