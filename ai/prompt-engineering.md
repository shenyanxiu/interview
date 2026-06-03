# 提示词工程（Prompt Engineering）

## 问题

提示词工程是什么，有哪些需要学习的？如何调节温度等参数？

## 主题

AI / 提示词工程

## 难度

中级

## 参考答案

### 核心定义

提示词工程是一门研究如何设计、优化输入给大语言模型（LLM）的指令/提示，以获得更准确、更可控输出的技术学科。本质是"如何跟 AI 说话，让它更好地完成你想要的任务"。

### 学习内容体系

#### 1. 基础技巧

| 技巧 | 说明 | 示例 |
|------|------|------|
| 角色设定 | 给模型一个身份 | "你是一位资深 React 开发者" |
| 明确指令 | 清晰描述期望输出 | "用 TypeScript 写，包含类型定义" |
| 格式约束 | 指定输出格式 | "以 JSON 格式返回" |
| 分步引导 | 让模型逐步思考 | "请一步步分析这个问题" |
| 示例驱动 | 给出 few-shot 示例 | 提供输入输出样例 |

#### 2. 推理策略

- **Chain of Thought (CoT)**：引导模型展示推理过程，提升复杂问题的准确率
- **Few-shot / Zero-shot / One-shot**：通过提供不同数量的示例来引导输出
- **Self-consistency**：多次采样取一致性最高的答案
- **ReAct（Reasoning + Acting）**：让模型交替进行推理和行动，是 Agent 的核心模式
- **Tree of Thought**：让模型探索多条推理路径，适合复杂决策

#### 3. 系统级提示词设计

- **System Prompt 设计**：定义 AI 的行为边界、能力范围、输出风格
- **上下文管理**：如何在有限的 token 窗口内组织最有效的上下文
- **指令层级**：系统指令 > 用户指令 > 历史对话的优先级设计
- **安全防护**：防止 prompt injection、越狱攻击
- **模板化**：用变量和模板构建可复用的提示词

#### 4. Agent 相关的提示词工程

- **工具调用描述**：如何描述工具让模型正确选择和使用
- **规划提示**：引导模型分解任务、制定计划
- **反思与纠错**：让模型检查自己的输出并修正
- **多轮对话管理**：维护上下文连贯性
- **Prompt Chaining**：将复杂任务拆成多步，前一步输出作为后一步输入

#### 5. 结构化提示词技巧

- **XML/Markdown 结构化**：用标记组织提示词，模型对层次化指令遵从度更高
- **分隔符技巧**：用明确分隔符区分指令和数据，防止数据被当作指令执行
- **负面提示**：告诉模型"不要做什么"（如"不要编造不存在的 API"）

#### 6. 温度与采样参数控制

| 参数 | 低值效果 | 高值效果 | 适用场景 |
|------|----------|----------|----------|
| temperature | 确定性强、输出稳定 | 创造性强、多样化 | 代码生成用低值（0-0.3），头脑风暴用高值（0.7-1.0） |
| top_p | 保守选词 | 丰富选词 | 精确任务用低值 |
| max_tokens | 简短回答 | 详细回答 | 根据任务复杂度调整 |
| frequency_penalty | 允许重复 | 减少重复 | 长文本生成时适当提高 |
| presence_penalty | 允许重复主题 | 鼓励新主题 | 创意写作时适当提高 |

**调参建议**：
- 代码生成：temperature 0~0.2，确保输出确定性
- 文案创作：temperature 0.7~1.0，增加多样性
- 问答/摘要：temperature 0.3~0.5，平衡准确与流畅
- temperature 和 top_p 通常只调一个，不建议同时调

#### 7. 多模态提示词

- 截图 + 文字描述结合（"根据这个设计稿生成 React 组件"）
- 图表理解（"分析这个架构图的瓶颈"）
- UI 还原（"把这个 Figma 截图转成 Tailwind 代码"）

#### 8. 评估与优化

- **A/B 测试**：对比不同提示词的效果
- **评估指标**：准确率、相关性、格式遵从度、鲁棒性、一致性
- **迭代优化**：根据失败案例调整提示词
- **Prompt 版本管理**：像管理代码一样管理提示词

#### 9. 安全相关

- **直接注入**：用户输入中包含"忽略以上指令"
- **间接注入**：通过外部数据源（网页、文件）注入恶意指令
- **防护手段**：输入过滤、指令隔离、输出校验、权限最小化

### 知识图谱总结

```
提示词工程
├── 基础技巧（角色、指令、格式、示例）
├── 推理策略（CoT、ToT、ReAct、Self-consistency）
├── 结构化设计（XML/Markdown、分隔符、负面提示）
├── 系统设计（System Prompt、上下文管理、安全防护）
├── Agent 应用（工具描述、规划、反思、Prompt Chaining）
├── 参数调优（temperature、top_p、max_tokens）
├── 多模态（图文结合）
└── 评估优化（指标、A/B 测试、版本管理）
```

## 追问链

- CoT 和 ToT 的区别是什么？分别适用于什么场景？
- 如何防止 Prompt Injection？
- RAG 和提示词工程的关系是什么？
- 如何设计一个好的 System Prompt？
- DSPy 是什么？如何用编程方式自动优化提示词？

## 易错点

- temperature 和 top_p 同时调容易产生不可预期的结果，通常只调一个
- 提示词越长不一定越好，冗余信息会稀释关键指令
- Few-shot 示例如果质量差，反而会误导模型
- 忽视安全防护（Prompt Injection）在生产环境中是严重隐患

## 知识延伸

- **RAG（检索增强生成）**：提示词 + 外部知识检索的结合
- **Fine-tuning vs Prompt Engineering**：什么时候该调提示词，什么时候该微调模型
- **MCP（Model Context Protocol）**：标准化的工具描述协议，本质也是提示词工程的一部分
- **DSPy**：用编程方式自动优化提示词的框架
- **元提示（Meta Prompting）**：让模型帮你写/优化提示词

---

# 温度等参数的深入调节

## 问题

如何调节大语言模型的温度（temperature）等采样参数？

## 主题

AI / 提示词工程 / 参数调优

## 难度

中级

## 参考答案

### 核心参数原理

#### Temperature（温度）

数学本质：对 logits 做除法后再 softmax：`softmax(logits / T)`

- T → 0：概率集中在最高分的 token 上，输出几乎确定
- T = 1：保持原始概率分布
- T > 1：分布更平坦，低概率 token 也有机会被选中

#### Top-p（核采样 / Nucleus Sampling）

只从累积概率达到 p 的最小 token 集合中采样：

- top_p = 0.1：只考虑概率最高的一小撮 token
- top_p = 1.0：考虑所有 token（等于不过滤）

#### Top-k

只从概率最高的前 k 个 token 中采样：

- top_k = 1：贪心解码（greedy），永远选最高概率
- top_k = 50：从前 50 个候选中采样

比 top_p 更粗暴，实际使用中 top_p 更常见。

#### Frequency Penalty（频率惩罚）

对已出现过的 token 按出现次数施加惩罚，减少逐字重复。

#### Presence Penalty（存在惩罚）

只要 token 出现过就施加固定惩罚，鼓励模型谈论新话题。

### 场景化调参方案

| 场景 | temperature | top_p | freq_penalty | presence_penalty |
|------|-------------|-------|--------------|-----------------|
| 代码生成 | 0~0.2 | 1 | 0 | 0 |
| 代码补全 | 0 | 1 | 0 | 0 |
| 数据提取/JSON | 0 | 1 | 0 | 0 |
| 翻译 | 0.3 | 1 | 0 | 0 |
| 对话聊天 | 0.7 | 0.9 | 0.3 | 0.3 |
| 创意写作 | 0.9 | 0.95 | 0.5 | 0.5 |
| 头脑风暴 | 1.0 | 1 | 0.8 | 0.8 |

### 调参直觉

```text
想要确定性 → 降 temperature
想要多样性 → 升 temperature 或降 top_p
输出重复啰嗦 → 加 frequency_penalty
总在同一个话题打转 → 加 presence_penalty
输出被截断 → 加 max_tokens
```

### 代码示例（OpenAI API）

```typescript
// 代码生成场景
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "写一个快排" }],
  temperature: 0,        // 代码生成用 0，确保确定性
  top_p: 1,             // 不做核采样过滤
  frequency_penalty: 0,  // 代码不需要避免重复
  presence_penalty: 0,
  max_tokens: 1024,
});

// 创意场景
const creative = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "给我的产品起 10 个名字" }],
  temperature: 1.0,       // 高创造性
  frequency_penalty: 0.8, // 避免重复用词
  presence_penalty: 0.6,  // 鼓励多样化
  max_tokens: 512,
});
```

## 易错点

- temperature=0 不代表每次输出完全一样（浮点精度、批处理等因素可能导致微小差异）
- temperature 和 top_p 通常只调一个，同时调容易产生不可预期的效果
- 不同模型对参数的敏感度不同，GPT-4 比 GPT-3.5 对 temperature 更"稳"
- max_tokens 是输出上限，不是"请写这么多"的指令，想控制长度应该在提示词里说明
- 某些 API（如 Claude）的参数名和范围可能略有不同，需查阅对应文档

## 知识延伸

- **Beam Search**：另一种解码策略，保留多条候选路径，适合翻译等任务
- **Repetition Penalty**：Hugging Face 模型常用，跟 frequency_penalty 类似但计算方式不同
- **Logit Bias**：直接对特定 token 的 logit 加减分，可以强制/禁止某些输出
- **Min-p Sampling**：较新的采样策略，设定相对于最高概率 token 的最小概率阈值
