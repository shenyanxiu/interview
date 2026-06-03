# I/O 密集型与 CPU 密集型

## 问题：什么是 I/O 密集型和 CPU 密集型

### 主题
操作系统 / 计算机基础

### 难度
初级

### 参考答案

#### I/O 密集型

程序大部分时间在**等待外部资源**响应，CPU 基本空闲。

常见场景：
- 读写数据库
- 调用第三方 API / 微服务间通信
- 读写文件、磁盘操作
- 网络请求（HTTP、WebSocket）

特征：代码执行到 I/O 操作就"卡住"等结果，CPU 没事干。瓶颈在网络带宽、磁盘速度、对方服务响应时间。

```javascript
// 典型 I/O 密集：CPU 几乎不工作，全在等
const user = await db.query('SELECT * FROM users WHERE id = ?', [id])
const order = await fetch('https://order-service/api/orders/' + user.id)
const result = await redis.get('cache:' + order.id)
```

#### CPU 密集型

程序大部分时间在**用 CPU 做计算**，不怎么等外部资源。

常见场景：
- 图片/视频压缩、转码
- 加密解密、哈希计算
- JSON 解析大数据量
- 数据排序、复杂算法
- 机器学习推理

```javascript
// 典型 CPU 密集：CPU 一直在算，没有等待
function fibonacci(n) {
  if (n <= 1) return n
  return fibonacci(n - 1) + fibonacci(n - 2)
}
fibonacci(45) // CPU 疯狂递归计算
```

#### 直观类比

- **I/O 密集** = 厨师同时煮 10 锅汤，大部分时间等水开，人很闲，可以同时照看很多锅
- **CPU 密集** = 厨师在切 10 斤土豆丝，手一直在动，没法同时干别的

#### 对比总结

| | I/O 密集 | CPU 密集 |
|--|---------|---------|
| 瓶颈 | 等待时间 | 计算时间 |
| 加线程有用吗 | 有用（等待时切换做别的） | 超过 CPU 核心数就没意义 |
| Node.js 表现 | 很好（事件循环天然适合等待） | 很差（单线程，计算卡住所有请求） |
| 优化方向 | 并发数、连接池、缓存 | 多核并行、算法优化、C/Rust 扩展 |

现实中大多数 Web 服务是 **I/O 密集为主、偶尔 CPU 密集**，这也是 Node.js 在 Web 后端流行的原因。

### 追问链
- Node.js 如何处理 CPU 密集任务？
- 如何判断一个服务是 I/O 密集还是 CPU 密集？
- 线程池大小应该怎么设置？（I/O 密集可以多开，CPU 密集开核心数就够）

### 易错点
- 不要以为"异步"就能解决 CPU 密集问题——异步解决的是等待问题，不是计算问题
- `JSON.parse()` 处理大数据时也是 CPU 密集操作，容易被忽略
- 数据库查询本身对应用来说是 I/O 密集，但对数据库服务器来说可能是 CPU 密集

### 知识延伸
- Node.js 的 worker_threads 处理 CPU 密集任务
- Java 线程池参数设置公式：I/O 密集型线程数 = CPU 核心数 × (1 + 等待时间/计算时间)
- Rust/C++ 编写 Node.js 原生扩展加速 CPU 密集计算
