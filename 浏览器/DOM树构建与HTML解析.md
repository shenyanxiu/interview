# DOM 树构建与 HTML 解析

## 问题 1：浏览器怎么构建 DOM 树的

### 主题
浏览器渲染原理

### 难度
中级

### 参考答案

浏览器将 HTML 原始字节流转换为内存中树形结构，分为以下阶段：

```
字节流 → 字符 → Token → Node → DOM 树
```

**1. 编码转换：字节 → 字符**

根据 `Content-Type` 响应头或 `<meta charset="utf-8">` 确定编码，将字节解码为字符串。

**2. 词法分析（Tokenization）：字符 → Token**

HTML 解析器使用**状态机**逐字符扫描，拆分为 Token：

```html
<div class="box"><p>hello</p></div>
```

产生 Token 序列：
```
StartTag: div (attr: class="box")
StartTag: p
Character: "hello"
EndTag: p
EndTag: div
```

Token 类型：DOCTYPE、StartTag、EndTag、Character、Comment、EOF

**3. 建树（Tree Construction）：Token → DOM 树**

解析器维护一个**开放元素栈**，边生成 Token 边构建树：

```
遇到 StartTag → 创建元素节点，压入栈，挂到栈顶节点下
遇到 Character → 创建文本节点，挂到栈顶节点下
遇到 EndTag   → 弹出栈顶（匹配的开始标签）
```

**4. 遇到特殊资源时的行为**

| 遇到什么 | 行为 |
|---------|------|
| `<script>`（无 async/defer） | 阻塞解析，下载并执行完才继续 |
| `<script async>` | 并行下载，下载完立即执行 |
| `<script defer>` | 并行下载，DOM 解析完毕后按序执行 |
| `<link rel="stylesheet">` | 不阻塞 DOM 解析，但阻塞渲染 |
| `<img>` | 不阻塞，异步加载 |

**5. 预解析扫描器（Preload Scanner）**

主解析器被 JS 阻塞时，浏览器启动预解析扫描器，提前发现后续需要下载的资源并发起请求。

**关键特性：** DOM 构建是**增量式**的，网络上每到达一段 HTML 就立即解码、分词、建树，不需要等整个文档下载完。

### 追问链
- 为什么 script 会阻塞 DOM 解析？（因为 JS 可以 `document.write()` 修改 HTML 流）
- async 和 defer 的区别？
- CSS 会阻塞 DOM 解析吗？（不阻塞解析，但阻塞渲染）
- 如果 HTML 有错误，解析会失败吗？（见下方问题 2）

### 易错点
- CSS 不阻塞 DOM 解析，但阻塞渲染（CSSOM 构建完才能合成渲染树）
- `<script>` 前面有未加载完的 CSS 时，JS 执行也会被阻塞（因为 JS 可能读取样式）
- 预解析扫描器只是提前发请求，不会构建 DOM

### 知识延伸
- 浏览器的渲染流水线：DOM + CSSOM → 渲染树 → 布局 → 绘制 → 合成
- `DOMContentLoaded` 事件在 DOM 解析完成时触发（不等图片和样式）
- `document.write()` 在异步脚本中调用会清空整个文档

---

## 问题 2：如果返回的内容有错误，HTML 解析会失败吗

### 主题
浏览器渲染原理

### 难度
初级

### 参考答案

**HTML 解析几乎不会"失败"**，这是它和 XML/JSON 解析最大的区别。

HTML 规范（WHATWG）明确定义了各种错误情况的处理方式，解析器必须"容错"而不是抛异常。无论给它什么内容，都会产出一棵 DOM 树。

**常见错误的处理方式：**

| 错误情况 | 处理方式 |
|---------|---------|
| 标签未闭合 `<p>text` | 自动补全闭合标签 |
| 嵌套错误 `<b><i>text</b></i>` | 领养算法（Adoption Agency Algorithm）重组结构 |
| 不认识的标签 `<foo>` | 当作 HTMLUnknownElement 正常挂到树上 |
| 返回 JSON/纯文本 | 整段内容当文本节点塞进 `<body>` |
| 编码错误 | 不会失败，只是显示乱码 |
| 二进制内容 | 当字符处理，产出乱码文本节点的 DOM 树 |

**真正不走 HTML 解析的情况：**

| 情况 | 浏览器行为 |
|------|-----------|
| 网络错误（DNS 失败、超时） | 显示错误页面，不进入解析 |
| `Content-Type: application/xhtml+xml` | 用 XML 解析器，格式错误会直接报错白屏 |

**对比 XML 解析器：**

XML 解析器遇到格式错误会直接停止并报错，这也是 XHTML 没有流行的原因——太严格。

**核心设计哲学：** 尽最大努力呈现内容，门槛低，容错高。

### 追问链
- XHTML 和 HTML 的区别？
- `document.write()` 写入非法内容会怎样？
- 浏览器的错误恢复机制具体是怎么实现的？

### 易错点
- 不要以为 HTML 写错了浏览器就不渲染——它永远会尝试渲染
- `Content-Type` 决定了浏览器用哪个解析器，不是文件后缀
- HTTP 4xx/5xx 响应体中的 HTML 照样会被解析渲染（错误页面本身就是 HTML）

### 知识延伸
- WHATWG HTML 规范中的解析错误处理章节
- 浏览器的"怪异模式"（Quirks Mode）vs 标准模式
- HTML Validator 工具可以检查 HTML 是否规范，但不规范不代表不能渲染
