# 从输入 URL 到页面渲染 — 全链路详解

---

## 一、用户输入与 URL 解析

浏览器判断输入内容：
- 合法 URL → 进入导航流程
- 非 URL → 拼接搜索引擎地址进行搜索

URL 结构解析：
```
https://www.example.com:443/api/users?id=1#section
协议     域名            端口  路径      查询参数  片段
```

---

## 二、DNS 解析（域名 → IP）

逐级查找，命中即停：

```
浏览器 DNS 缓存
  → 操作系统缓存（hosts 文件）
    → 路由器缓存
      → ISP 本地 DNS 服务器（递归查询）
        → 根域名服务器 → 顶级域名服务器(.com) → 权威域名服务器
```

**优化手段：**
- `dns-prefetch`：`<link rel="dns-prefetch" href="//cdn.example.com">`
- DNS 缓存 TTL
- HTTPDNS（绕过运营商劫持）

**dns-prefetch 详解：**

一种资源提示（Resource Hint），告诉浏览器「这个域名待会儿会用到，现在就先把 DNS 解析做了」。浏览器在空闲时提前完成 DNS 查询，等真正发请求时直接用缓存好的 IP，省掉 20~120ms 的等待。

```
正常流程：解析到资源 → DNS 解析（阻塞）→ TCP 连接 → 请求
优化流程：页面加载时预解析 DNS → 后续请求直接用缓存 IP → TCP 连接 → 请求
```

适用场景：页面会请求的第三方域名（CDN、API、统计脚本）、用户大概率跳转的下一页域名。

**资源提示对比：**

| 提示 | 做了什么 | 程度 |
|---|---|---|
| `dns-prefetch` | 只做 DNS 解析 | 最轻量 |
| `preconnect` | DNS + TCP + TLS 全部提前建好 | 更激进 |
| `prefetch` | 提前下载资源（低优先级，空闲时） | 下载整个资源 |
| `preload` | 提前下载资源（高优先级，当前页必用） | 强制立即下载 |

注意事项：
- 成本极低（只是一次 DNS 查询），用不上也没副作用
- 不要滥用，列太多域名会和真正的 DNS 查询竞争资源
- `href` 用 `//` 开头（协议相对）即可
- 对同源域名没意义（已经解析过了）

---

## 三、TCP 连接（三次握手）

```
客户端                    服务端
  |--- SYN (seq=x) ------->|     ① 客户端请求建立连接
  |<-- SYN+ACK (seq=y, ack=x+1) -|  ② 服务端同意并确认
  |--- ACK (ack=y+1) ----->|     ③ 客户端确认，连接建立
```

**为什么是三次握手？**

- **两次不行：** 客户端发了一个连接请求因网络延迟滞留了，客户端超时重发并正常完成通信后关闭。此时滞留的旧请求到达服务端，如果只有两次握手，服务端直接进入 ESTABLISHED 分配资源等待数据 — 但客户端根本不会发数据，资源白白浪费。三次握手时，服务端发了 SYN+ACK 还要等客户端 ACK，客户端知道这是过期请求不会回 ACK，连接就不会误建。
- **四次没必要：** 三次已经确认双方收发能力（第一次证明客户端能发，第二次证明服务端能收能发，第三次证明客户端能收）。第二步把 SYN 和 ACK 合并成一个包，没必要拆成两步。

**TCP 优化：**
- HTTP/1.1 `Keep-Alive` 复用连接
- HTTP/2 多路复用（一个连接并行多个请求）
- HTTP/3 基于 QUIC（UDP），0-RTT 建连

---

## 四、TLS 握手（HTTPS）

在 TCP 之上建立加密通道（以 TLS 1.2 为例）：

1. **Client Hello** — 支持的加密套件、随机数 A
2. **Server Hello** — 选定加密套件、随机数 B、下发证书
3. **客户端验证证书** — CA 证书链验证、域名匹配、有效期
4. **密钥交换** — 客户端生成预主密钥（Pre-Master Secret），用服务端公钥加密发送
5. **双方生成会话密钥** — 随机数 A + 随机数 B + 预主密钥 → 对称加密密钥
6. **后续通信使用对称加密**（性能远高于非对称）

TLS 1.3 优化到 1-RTT 甚至 0-RTT。

---

## 五、HTTP 请求

**请求报文结构：**
```
GET /api/users?id=1 HTTP/1.1
Host: www.example.com
Accept: text/html
Cookie: session=abc123
Connection: keep-alive
```

**请求方法语义：**
- GET — 获取资源（幂等）
- POST — 创建资源
- PUT — 全量更新
- PATCH — 部分更新
- DELETE — 删除

---

## 六、后端处理

### 1. 负载均衡（入口层）

请求先到达负载均衡器（Nginx / ALB / LVS）：
- 轮询、加权轮询、IP Hash、最少连接数
- 健康检查剔除故障节点

### 2. 反向代理 / 网关

- 路由分发（根据路径转发到不同微服务）
- 限流、熔断、降级
- 鉴权（JWT 验证、OAuth）
- 请求日志、链路追踪

### 3. Web 服务器 / 应用层

```
请求进入
  → 中间件链（日志、CORS、认证、参数校验）
    → 路由匹配
      → Controller 层（接收参数、调用 Service）
        → Service 层（业务逻辑）
          → DAO 层（数据访问）
```

### 4. 数据库查询

- **缓存层**：先查 Redis/Memcached（命中直接返回）
- **数据库**：MySQL/PostgreSQL 执行 SQL
  - 连接池获取连接
  - SQL 解析 → 查询优化器 → 执行计划 → 存储引擎
  - 走索引 or 全表扫描
- **ORM 映射**：数据库行 → 对象/结构体

### 5. 响应组装

- 序列化（JSON/Protobuf）
- 设置响应头（Content-Type、Cache-Control、Set-Cookie）
- 压缩（gzip/brotli）

---

## 七、HTTP 响应

**响应报文结构：**
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Cache-Control: no-cache
ETag: "abc123"
Set-Cookie: session=xyz; HttpOnly; Secure

<!DOCTYPE html>
<html>...
```

**常见状态码：**
| 状态码 | 含义 |
|---|---|
| 200 | 成功 |
| 301/302 | 永久/临时重定向 |
| 304 | 资源未修改（用缓存） |
| 400 | 请求参数错误 |
| 401/403 | 未认证 / 无权限 |
| 404 | 资源不存在 |
| 500/502/503 | 服务端错误 |

---

## 八、浏览器解析与渲染

### 1. 构建 DOM 树

```
字节流 → 字符 → Token（标记化）→ Node → DOM Tree
```

遇到 `<script>` 默认阻塞解析（除非 async/defer）。

### 2. 构建 CSSOM 树

```
CSS 字节 → 字符 → Token → Node → CSSOM Tree
```

CSS 不阻塞 DOM 解析，但阻塞渲染（必须等 CSSOM 构建完）。

### 3. 构建 Render Tree

- DOM + CSSOM 合并
- 排除不可见节点（`display: none`、`<head>`）
- `visibility: hidden` 仍占位，会进入 Render Tree

### 4. Layout（布局/回流）

- 计算每个节点的精确位置和大小
- 输出盒模型（Box Model）

### 5. Paint（绘制）

- 将布局信息转为实际像素
- 按层绘制（文本、颜色、边框、阴影等）

### 6. Composite（合成）

- 将多个图层交给 GPU 合成
- `transform`、`opacity` 等属性可直接在合成阶段处理（跳过回流和重绘）

---

## 九、子资源加载

HTML 解析过程中遇到外部资源：

| 资源 | 行为 |
|---|---|
| CSS `<link>` | 异步下载，阻塞渲染 |
| JS `<script>` | 阻塞解析和渲染 |
| JS `<script async>` | 异步下载，下载完立即执行 |
| JS `<script defer>` | 异步下载，DOMContentLoaded 前按序执行 |
| 图片/字体 | 异步下载，不阻塞 |

**预加载提示：**
```html
<link rel="preload" href="critical.css" as="style">
<link rel="prefetch" href="next-page.js">
<link rel="preconnect" href="https://api.example.com">
```

---

## 十、页面可交互

关键时间节点：
- **FP（First Paint）** — 首次绘制任何像素
- **FCP（First Contentful Paint）** — 首次绘制有意义内容
- **LCP（Largest Contentful Paint）** — 最大内容元素渲染完成
- **TTI（Time to Interactive）** — 页面完全可交互
- **CLS（Cumulative Layout Shift）** — 布局稳定性

---

## 十一、连接关闭（四次挥手）

```
客户端                    服务端
  |--- FIN ------------->|   ① 客户端请求断开
  |<-- ACK --------------|   ② 服务端确认
  |<-- FIN --------------|   ③ 服务端数据发完，请求断开
  |--- ACK ------------->|   ④ 客户端确认，等待 2MSL 后关闭
```

**为什么四次？** 服务端收到 FIN 时可能还有数据没发完，需要分开确认和关闭。

**详细解释：**

TCP 是全双工通信（双方可以同时收发），关闭是双向独立的：
- 客户端说「我发完了」只代表客户端→服务端方向关闭
- 服务端可能还有数据要发给客户端，不能立刻关闭服务端→客户端方向
- 所以第②步服务端只能先回 ACK（确认收到关闭请求），等自己数据发完后再发 FIN（第③步），这两步不能合并

**对比握手时为什么能合并：**
- 握手时服务端收到 SYN，自己也要发 SYN，两件事可以同时做（没有"还有数据没发完"的问题）
- 挥手时 ACK 和 FIN 之间可能有时间差（服务端还在发数据）

**特殊情况：** 如果服务端收到 FIN 时恰好也没有数据要发了，可以把 ACK 和 FIN 合并（TCP 延迟确认机制），这时变成"三次挥手"。但协议设计上必须支持四次的情况。

**为什么客户端最后要等 2MSL？**

MSL（Maximum Segment Lifetime）= 报文最大生存时间（通常 30s~2min）：
1. 确保服务端收到最后的 ACK — 如果 ACK 丢了，服务端会重发 FIN，客户端还能重新回 ACK
2. 让本次连接的所有报文在网络中消亡 — 防止旧连接的延迟报文干扰新连接

---

## 全流程一图总结

```
用户输入 URL
  → DNS 解析
    → TCP 三次握手
      → TLS 握手（HTTPS）
        → 发送 HTTP 请求
          → 负载均衡 → 网关 → 应用服务 → 缓存/数据库
        ← HTTP 响应
      ← 解析 HTML/CSS/JS
    ← 构建 DOM + CSSOM → Render Tree
  ← Layout → Paint → Composite
→ 页面可交互
  → TCP 四次挥手（连接关闭）
```
