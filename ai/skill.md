# 什么是 Skill

## 问题

什么是 Skill（在 AI Agent 领域的通用概念）？

## 主题

AI 基础概念

## 难度

初级

## 参考答案

在 AI Agent 领域，Skill 是一个通用概念：**Agent 能执行的一项具体能力或任务单元。**

类比：Agent 是一个员工，Skill 是他简历上的一项技能（会写 Python、会做数据分析、会写测试）。

在不同框架里叫法不同但本质一样：

| 框架/产品 | 叫法 |
|-----------|------|
| OpenAI | Function / Tool |
| LangChain | Tool |
| AutoGPT | Ability |
| Semantic Kernel (微软) | Skill / Plugin |
| Kiro | Skill |

核心思想：**把一个可复用的能力抽象出来，让 Agent 在需要时调用。** 它可以是调用一个 API、执行一段代码、遵循一套流程，或者组合多个子步骤完成一个任务。

Skill 让 Agent 从"什么都要从零推理"变成"有现成能力可以直接用"，提高效率和可靠性。

## 易错点

- 把 Skill 和 Prompt 混淆：Prompt 是一次性指令，Skill 是持久化的、可被自动匹配和复用的能力定义
- 不同框架术语不同，但概念本质相同

## 知识延伸

- Skill 的组合与编排（多个 Skill 串联完成复杂任务）
- Skill 的动态发现与注册机制
- Skill Store / Plugin Marketplace 生态
