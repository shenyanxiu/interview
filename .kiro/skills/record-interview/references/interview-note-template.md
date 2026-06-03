# 面试笔记 Markdown 模板

本参考文件定义面试笔记的字段结构和 Markdown 格式。生成内容时按此模板组织，
保持格式统一，便于后续检索和复习。

## 文件头部模板

每个主题文件的开头包含主题说明和简要目录。

```markdown
# <主题名，如 JavaScript 闭包>

> 面试问答归档，按追加顺序记录。

## 目录

- [<问题 1 标题>](#问题-1-标题)
- [<问题 2 标题>](#问题-2-标题)
```

## 单个问题条目模板

每个问题作为一个二级标题块，包含以下字段：

```markdown
## <问题标题，精简版>

**难度**: 初级 / 中级 / 高级
**主题**: <细分主题或标签>
**记录时间**: YYYY-MM-DD

### 问题

<问题原文，保留完整描述>

### 参考答案

<标准或推荐回答要点，可以用列表、代码块、表格>

### 追问链

面试官可能进一步追问：

- <追问 1>
- <追问 2>

### 易错点

- <常见错误 1>
- <常见错误 2>

### 知识延伸

- <相关概念或学习方向 1>
- <相关概念或学习方向 2>

---
```

## 字段说明

| 字段 | 是否必填 | 说明 |
| ---- | -------- | ---- |
| 问题标题 | 必填 | 简洁概括，便于目录检索 |
| 难度 | 推荐 | 初级 / 中级 / 高级，基于经验判断 |
| 主题 | 推荐 | 细分标签，如"闭包"、"作用域"、"事件循环" |
| 记录时间 | 推荐 | YYYY-MM-DD 格式 |
| 问题 | 必填 | 完整问题描述，保留上下文 |
| 参考答案 | 必填 | 核心要点，可用列表、代码、表格 |
| 追问链 | 可选 | 面试官常见追问方向 |
| 易错点 | 可选 | 常见错误或陷阱 |
| 知识延伸 | 可选 | 相关概念或进阶学习方向 |

## 文件命名规范

| 类别 | 文件名示例 |
| ---- | --------- |
| 语言基础 | `javascript-closure.md`、`typescript-generics.md` |
| 框架 | `react-hooks.md`、`vue-reactivity.md` |
| 工程 | `webpack-optimization.md`、`performance-tuning.md` |
| 系统设计 | `system-design-rate-limiter.md`、`system-design-url-shortener.md` |
| 算法 | `algorithm-dp.md`、`algorithm-graph.md` |
| 行为面试 | `behavioral-teamwork.md`、`behavioral-conflict.md` |

## 追加规则

- 新问题追加到文件末尾
- 在目录区域同步添加新条目链接
- 保持原有条目不变
- 问题之间用水平分隔线（`---`）分隔

## 主题归类建议

| 关键词 | 建议文件 |
| ------ | -------- |
| 闭包、作用域、变量提升 | `javascript-scope.md` 或 `javascript-closure.md` |
| Promise、async/await、事件循环 | `javascript-async.md` |
| 原型、继承、this | `javascript-prototype.md` |
| Hooks、生命周期、Fiber | `react-hooks.md` 或 `react-internals.md` |
| 响应式、组合式 API | `vue-reactivity.md` |
| HTTP、缓存、CORS | `network-http.md` |
| 浏览器渲染、重绘重排 | `browser-rendering.md` |
| 限流、缓存、高并发 | `system-design-<具体主题>.md` |
| STAR 法则、团队冲突 | `behavioral-<具体主题>.md` |
