# 为什么需要多 Agent

## 问题

为什么需要多个 Agent 协作？单个 Agent 全做不行吗？

## 主题

AI Agent 架构

## 难度

中级

## 参考答案

短答案：**简单任务单 Agent 完全够用，复杂任务单 Agent 会崩。** 多 Agent 不是为了炫技，是为了解决单 Agent 的硬伤。

### 单 Agent 的瓶颈

#### 1. Context Window 有限

LLM 的上下文窗口有上限（128K、200K token）。一个 Agent 又要记需求、又要读代码、又要存对话历史、又要放工具描述，很快就塞满了。塞满之后早期信息被截断，模型"忘事"，行为开始飘。

#### 2. 角色冲突

一个 Agent 同时扮演"产品经理 + 开发 + 测试"，System Prompt 里的指令互相矛盾。一个模型同时执行矛盾指令，输出质量下降。

#### 3. 工具太多，选择困难

给一个 Agent 挂 50 个工具，模型需要在每一步从 50 个里选对的那个。工具越多，选错的概率越高，token 消耗也越大。

#### 4. 错误传播无隔离

单 Agent 一步走错，后面全部基于错误结果继续推理，越走越偏。没有人"回头检查"。

### 多 Agent 怎么解决这些问题

| 单 Agent 的问题 | 多 Agent 的解法 |
|----------------|----------------|
| Context 爆了 | 每个 Agent 只关注自己的子任务，context 干净 |
| 角色冲突 | 每个 Agent 一个明确角色，System Prompt 专注 |
| 工具太多 | 每个 Agent 只挂自己需要的 3-5 个工具 |
| 错误传播 | 下游 Agent 可以质疑上游结果，形成校验 |
| 任务太复杂 | 拆成子任务并行执行，效率更高 |

### 多 Agent 的常见模式

#### 1. 管道式（Pipeline）

```
A → B → C → D
```

上游输出是下游输入，适合流程明确的任务。

#### 2. 主从式（Orchestrator + Workers）

```
     Orchestrator
    /     |      \
Worker  Worker  Worker
```

一个"经理"Agent 分配任务，多个"员工"Agent 并行执行，经理汇总结果。

#### 3. 辩论式（Debate）

```
Agent A ←→ Agent B
       ↓
    Judge Agent
```

两个 Agent 各持观点辩论，第三个 Agent 裁决。适合需要多角度分析的决策。

#### 4. 层级式（Hierarchical）

```
CEO Agent
├── Tech Lead Agent
│   ├── Frontend Agent
│   └── Backend Agent
└── QA Lead Agent
    └── Tester Agent
```

模拟组织架构，逐层分解任务。

### 什么时候用单 Agent 就够了

- 任务简单、步骤少（< 5 步）
- 只需要 2-3 个工具
- 不需要多角度校验
- Context 用量可控

**大多数日常场景，单 Agent 足够。** 比如：改个 bug、查个问题、写个函数。

### 什么时候该上多 Agent

- 任务涉及多个专业领域
- 需要互相校验（写代码的和测试的不该是同一个人）
- 单 Agent context 不够用
- 需要并行加速
- 需要不同的模型做不同的事（贵的模型做决策，便宜的做执行）

### 多 Agent 的代价

- **复杂度上升** — 通信协议、状态同步、错误处理都变复杂
- **延迟增加** — Agent 之间传递信息需要时间
- **Token 消耗翻倍** — 多个 Agent 各自消耗 token
- **调试困难** — 出了问题要追踪多个 Agent 的决策链路

### 一句话总结

> 单 Agent 是"全栈独立开发者"，多 Agent 是"专业分工的团队"。一个人能干的活不用组团队，但复杂项目一个人扛不住。

## 易错点

- 过早引入多 Agent，单 Agent 都没调好就追求复杂编排
- 忽略多 Agent 的通信成本和延迟开销
- 没有明确的任务边界划分，Agent 之间职责重叠
- 把"多 Agent"当银弹，实际上很多场景单 Agent + 好的 Prompt 就够了

## 知识延伸

- CrewAI / AutoGen / LangGraph 多 Agent 框架
- A2A（Agent-to-Agent）协议
- Agent 间的通信方式：共享内存 vs 消息传递
- 多 Agent 的可观测性与调试工具
