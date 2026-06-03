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

## 定位：解决什么问题？

LLM 是"大脑"，但没有"手脚"。MCP 是 AI 领域的"USB-C 接口"——一套标准化协议，让任意 AI 模型能以统一方式连接任意外部工具和数据源。

> 💡 之前：每个 AI 应用 × 每个工具 = N × M 套对接代码。之后：每个 AI 应用接入 MCP（1次）× 每个工具实现 MCP Server（1次）= N + M。

## 架构原理

```plaintext
MCP Client（AI应用侧）  ◄── JSON-RPC (stdio/SSE) ──►  MCP Server（工具侧）
```

### 三个核心概念

| 概念 | 作用 | 类比 |
|------|------|------|
| Tools | 可执行操作（Agent 主动调用） | 函数/API |
| Resources | 数据源（Agent 读取上下文） | 文件/数据库表 |
| Prompts | 预定义的提示模板 | 函数模板 |

## 工作流程

1. Client 启动 Server 进程（或连接远程 Server）
2. Client 调用 `tools/list` 获取工具列表（名称 + 描述 + 输入 schema）
3. Agent 根据用户意图决定调用哪个 Tool
4. Client 发送 `tools/call` 请求 → Server 执行 → 返回结果
5. Agent 拿到结果继续推理

## 对比之前的方案（Function Calling）

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| 标准化 | 各家格式不同 | 统一协议，一次实现处处可用 |
| 复用性 | 工具绑定在特定 AI 应用 | Server 独立，任何 Client 都能接 |
| 生态 | 封闭 | 开放标准，社区共建 |
| 能力 | 只有 Tool 调用 | Tool + Resource + Prompt |
| 安全 | 交给应用层自己做 | 协议层内置权限机制 |


## 知识延伸

- MCP 与 OpenAI Function Calling 的区别
- MCP Server 的开发与部署
- MCP 生态中的 Transport 层（stdio、HTTP SSE）
- 安全性考量：权限控制、沙箱隔离

