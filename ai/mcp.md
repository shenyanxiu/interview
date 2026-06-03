# 什么是 MCP

## 问题

什么是 MCP（Model Context Protocol）？

## 主题

AI 基础概念

## 难度

中级

## 参考答案

MCP（Model Context Protocol）是 Anthropic 提出的一个**开放协议**，用来标准化 AI 模型与外部工具/数据源之间的连接方式。

### 解决的核心问题

以前每个 AI 应用要接一个工具（数据库、API、文件系统等），都得自己写一套集成代码。N 个 AI 应用 × M 个工具 = N×M 种适配。MCP 把这变成 N+M：大家都遵守同一个协议就行。

类比：**MCP 之于 AI 工具调用，就像 USB 之于外设连接。**

### 架构

```
AI 应用 (Client)  ←→  MCP 协议  ←→  MCP Server (工具/数据源)
```

- **MCP Client** — AI 应用端（比如 Kiro、Claude Desktop、Cursor）
- **MCP Server** — 提供具体能力的服务（比如读 GitHub、查数据库、操作浏览器）
- **协议层** — 定义了怎么发现工具、怎么调用、怎么传参数和返回结果

### MCP Server 暴露的能力类型

- **Tools** — 可执行的操作（查询、写入、调用 API）
- **Resources** — 可读取的数据/上下文
- **Prompts** — 预定义的提示模板

### 实际意义

开发者写一个 MCP Server，所有支持 MCP 的 AI 客户端都能直接用，不用重复造轮子。生态共享。

## 易错点

- 把 MCP 当成某个具体工具，实际上它是一个协议/标准
- 混淆 MCP Server 和普通 REST API：MCP Server 遵循特定协议格式，支持工具发现、上下文传递等 AI 特有需求

## 知识延伸

- MCP 与 OpenAI Function Calling 的区别
- MCP Server 的开发与部署
- MCP 生态中的 Transport 层（stdio、HTTP SSE）
- 安全性考量：权限控制、沙箱隔离
