# link 与 @import 的区别

## 问题：CSS 中 link 和 @import 的区别

### 主题
CSS / 性能优化

### 难度
初级

### 参考答案

**基本用法：**

```html
<!-- link：HTML 标签，放在 <head> 中 -->
<link rel="stylesheet" href="style.css">
```

```css
/* @import：CSS 语法，放在样式表最顶部 */
@import url('style.css');
```

**核心区别：**

| 维度 | `<link>` | `@import` |
|------|----------|-----------|
| 本质 | HTML 标签 | CSS 规则 |
| 加载时机 | 页面加载时**并行**下载 | 等宿主 CSS 下载解析后才发起请求 |
| 兼容性 | 无限制 | CSS2.1 引入，IE5+ |
| DOM 可操作 | 可用 JS 动态插入/删除 | 不能通过 DOM 操作 |
| 功能范围 | 可加载 CSS、favicon、预加载等多种资源 | 只能加载 CSS |
| 放置位置 | `<head>` 中任意位置 | 必须在样式表最前面 |

**最关键的区别：加载行为**

`<link>` 并行下载：
```html
<!-- 浏览器并行下载 a.css 和 b.css -->
<link rel="stylesheet" href="a.css">
<link rel="stylesheet" href="b.css">
```

`@import` 串行下载：
```
下载 HTML → 发现 link → 下载 a.css → 解析 a.css → 发现 @import → 下载 b.css
```

**最坏情况——瀑布式加载：**

```css
/* a.css */ @import url('b.css');
/* b.css */ @import url('c.css');
/* c.css */ @import url('d.css');
/* 加载顺序：a → b → c → d（串行） */
```

用 `<link>` 则四个并行下载。

**结论：几乎所有场景都应该用 `<link>`**
- 并行加载，性能更好
- 可被预加载扫描器发现，提前请求
- 可用 JS 动态控制
- 没有位置限制

`@import` 唯一合理的场景是 CSS 预处理器（Sass/Less）源码中做模块化组织，但编译产物应合并为一个文件，生产环境不应保留 `@import`。

### 追问链
- 多个 `<link>` 的样式优先级怎么确定？（后面的覆盖前面的，同权重下）
- `<link rel="preload">` 和普通 `<link rel="stylesheet">` 的区别？
- CSS 加载会阻塞什么？（阻塞渲染，不阻塞 DOM 解析）

### 易错点
- `@import` 必须放在 CSS 文件最顶部，放在其他规则后面会被忽略
- `<link>` 和 `@import` 混用时，旧浏览器可能出现 FOUC（页面闪烁）
- `@import` 的串行问题在 HTTP/2 多路复用下依然存在——因为瓶颈在"发现时机"而非连接数

### 知识延伸
- HTTP/2 Server Push 可以主动推送 CSS，绕过发现延迟
- `<link rel="preload" as="style">` 可以提前加载后续需要的 CSS
- CSS Modules、CSS-in-JS 等方案从根本上改变了 CSS 加载方式
