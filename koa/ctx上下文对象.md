# Koa 的 ctx 有什么

## 问题

Koa 的 ctx（Context）上有哪些东西？它是怎么设计的？

## 主题

Node.js / Koa 框架

## 难度

中级

## 参考答案

Koa 的 `ctx` 是对 Node.js 原生 `req` 和 `res` 的封装，每个请求都会创建一个新的 `ctx` 对象，贯穿整个中间件链。

### ctx 的组成结构

```javascript
ctx = {
  request,   // Koa Request（封装后的请求对象）
  response,  // Koa Response（封装后的响应对象）
  req,       // Node.js 原生 http.IncomingMessage
  res,       // Node.js 原生 http.ServerResponse
  app,       // Koa 实例
  state,     // 中间件间共享数据的推荐命名空间
  cookies,   // cookie 操作
  throw,     // 抛出 HTTP 错误
  assert,    // 断言，不满足则抛错
}
```

### 请求相关（代理自 ctx.request）

```javascript
ctx.method          // 'GET', 'POST' 等
ctx.url             // 完整 URL，如 '/users?page=1'
ctx.path            // 路径部分，如 '/users'
ctx.query           // 解析后的查询参数 { page: '1' }
ctx.querystring     // 原始查询字符串 'page=1'
ctx.host            // 主机名（含端口）
ctx.hostname        // 主机名（不含端口）
ctx.headers         // 请求头对象
ctx.get(field)      // 获取请求头
ctx.ip              // 客户端 IP
ctx.ips             // 代理链 IP 数组
ctx.is(types)       // 判断请求 Content-Type
ctx.accepts(types)  // 内容协商
ctx.protocol        // 'http' 或 'https'
ctx.secure          // 是否 HTTPS
ctx.fresh           // 缓存是否新鲜（304 判断）
ctx.originalUrl     // 原始 URL（不会被中间件修改）
```

### 响应相关（代理自 ctx.response）

```javascript
ctx.body            // 响应体（设置后自动推断 Content-Type）
ctx.status          // 响应状态码
ctx.message         // 响应状态消息
ctx.type            // 响应 Content-Type
ctx.length          // 响应 Content-Length
ctx.set(field, val) // 设置响应头
ctx.append(field, val) // 追加响应头
ctx.remove(field)   // 删除响应头
ctx.redirect(url)   // 重定向
ctx.attachment(filename) // 设置 Content-Disposition 触发下载
ctx.headerSent      // 响应头是否已发送
```

### 工具方法

```javascript
// 抛出 HTTP 错误，会被错误处理中间件捕获
ctx.throw(403);
ctx.throw(400, 'name required');

// 断言，不满足条件则抛错
ctx.assert(ctx.state.user, 401, 'User not found');

// Cookie 操作
ctx.cookies.get('session');
ctx.cookies.set('session', 'abc123', { httpOnly: true });
```

### ctx.state：中间件间传递数据

```javascript
// 认证中间件
app.use(async (ctx, next) => {
  const user = await authenticate(ctx.headers.authorization);
  ctx.state.user = user;  // 挂到 state 上
  await next();
});

// 业务中间件
app.use(async (ctx) => {
  console.log(ctx.state.user); // 从 state 取出
});
```

`ctx.state` 是官方推荐的中间件间共享数据的方式，避免直接往 `ctx` 上挂属性导致命名冲突。

### 代理机制（Delegate）

`ctx` 上很多属性其实是代理到 `ctx.request` 和 `ctx.response` 上的：

```javascript
// 这两种写法等价
ctx.url        === ctx.request.url
ctx.body       === ctx.response.body
ctx.status     === ctx.response.status
ctx.get(field) === ctx.request.get(field)
ctx.set(field) === ctx.response.set(field)
```

Koa 通过 `delegates` 库实现代理，让你不用写 `ctx.request.xxx` / `ctx.response.xxx`，直接 `ctx.xxx` 就行。

### ctx vs req/res 的关系

```
┌─────────────────────────────────┐
│            ctx                   │
│  ┌───────────┐  ┌────────────┐  │
│  │  request  │  │  response  │  │
│  │  (Koa封装) │  │  (Koa封装)  │  │
│  │     ↓     │  │     ↓      │  │
│  │   req     │  │    res     │  │
│  │  (原生)    │  │   (原生)    │  │
│  └───────────┘  └────────────┘  │
└─────────────────────────────────┘
```

一般情况下用 `ctx` 的代理属性就够了，只有需要底层操作（如 SSE、WebSocket）时才直接用 `ctx.req` / `ctx.res`。

## 追问链

- `ctx.request.url` 和 `ctx.req.url` 有区别吗？
- 为什么 Koa 要用 delegates 做代理，而不是直接把属性挂在 ctx 上？
- `ctx.state` 和直接 `ctx.user = xxx` 有什么区别？为什么推荐用 state？
- 如何扩展 ctx，给它添加自定义方法？

## 易错点

- 混淆 `ctx.request` / `ctx.response`（Koa 封装）和 `ctx.req` / `ctx.res`（Node 原生）
- 以为 `ctx.body` 是请求体，实际上是响应体；请求体需要用 `koa-bodyparser` 等中间件解析后挂到 `ctx.request.body`
- 直接往 `ctx` 上挂自定义属性，可能和 Koa 内部属性或其他中间件冲突

## 知识延伸

- JavaScript 的 `delegates` 库实现原理（`__defineGetter__` / `__defineSetter__`）
- Express 的 `req` / `res` 扩展方式对比
- Koa 源码中 `context.js`、`request.js`、`response.js` 的职责划分
- 如何通过 `app.context` 扩展所有请求的 ctx
