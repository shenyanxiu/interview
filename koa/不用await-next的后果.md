# 如果不用 await next 会怎么样

## 问题

Koa 中间件中如果不用 `await next()` 会怎么样？

## 主题

Node.js / Koa 框架

## 难度

中级

## 参考答案

分两种情况：完全不调用 `next()`，以及调用了 `next()` 但没加 `await`。

### 1. 完全不调用 `next()`

后续中间件**不会执行**，请求到当前中间件就终止了。

```javascript
app.use(async (ctx, next) => {
  console.log('中间件 1');
  // 没有调用 next()
  ctx.body = '直接响应';
});

app.use(async (ctx, next) => {
  console.log('中间件 2'); // 永远不会执行
});
```

这在某些场景下是有意为之的，比如权限校验失败时直接返回 403，不需要继续往下走。

### 2. 调用了 `next()` 但没加 `await`

后续中间件**会执行**，但当前中间件不会等它们完成就继续执行 `next()` 之后的代码。洋葱模型的"回程"顺序被打破。

```javascript
app.use(async (ctx, next) => {
  console.log('1 - 进入');
  next(); // 没有 await
  console.log('1 - 离开');
});

app.use(async (ctx, next) => {
  console.log('2 - 进入');
  await someAsyncWork(); // 假设耗时 100ms
  await next();
  console.log('2 - 离开');
});

// 实际输出：
// 1 - 进入
// 2 - 进入
// 1 - 离开        ← 没等中间件2完成就执行了
// 2 - 离开        ← 异步完成后才到这里
```

这会导致几个问题：

- **计时不准**：外层中间件记录的响应时间不包含内层异步操作的耗时
- **错误捕获失效**：外层的 `try/catch` 捕获不到内层抛出的异步错误
- **响应状态不确定**：`ctx.body` 可能还没被内层设置，外层就已经读取了

```javascript
// 错误捕获失效的例子
app.use(async (ctx, next) => {
  try {
    next(); // 没有 await，catch 捕获不到内层异步错误
  } catch (err) {
    ctx.body = '错误处理'; // 不会执行
  }
});

app.use(async (ctx, next) => {
  await next();
  throw new Error('boom'); // 这个错误会变成 unhandled rejection
});
```

### 对比总结

| 情况 | 后续中间件是否执行 | 洋葱回程是否正常 | 错误能否被外层捕获 |
|------|------------------|----------------|------------------|
| `await next()` | ✅ | ✅ | ✅ |
| `next()`（无 await） | ✅ | ❌ | ❌ |
| 不调用 `next()` | ❌ | — | — |

## 追问链

- 如果多个中间件都没有 `await next()`，响应会以哪个中间件的 `ctx.body` 为准？
- Koa 内部是如何检测中间件多次调用 `next()` 的？
- 在什么业务场景下会故意不调用 `next()`？

## 易错点

- 以为不加 `await` 只是"风格问题"，实际上会导致错误捕获和执行顺序全部失控
- 以为不调用 `next()` 会报错，实际上 Koa 允许这样做，只是后续中间件不执行
- 混淆"不调用 next"和"调用 next 但不 await"这两种完全不同的情况

## 知识延伸

- Promise 链式调用与 async/await 的执行时序
- Koa 源码中 `next()` 被多次调用时的防护机制
- Express 中 `next()` 不返回 Promise，因此不存在 await 问题
