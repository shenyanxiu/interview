# gRPC 与 Protobuf

## 问题一：gRPC 是什么？

### 主题

架构设计 / RPC 框架

### 难度

中级

### 参考答案

**gRPC** = **g**oogle **R**emote **P**rocedure **C**all，是 Google 在 2015 年开源的高性能、跨语言 RPC 框架。

**核心一句话**：用 HTTP/2 传输，用 Protobuf 序列化，让远程调用像本地函数一样简单。

#### 核心组成

**1. HTTP/2（传输协议）**
- 多路复用：一个连接上并发多个请求，避免队头阻塞
- 二进制分帧：比 HTTP/1.1 文本协议更高效
- Header 压缩（HPACK）：减少传输开销
- 支持服务端推送、双向流

**2. Protocol Buffers（序列化）**
- Google 出品的二进制序列化协议
- 体积比 JSON 小 3~10 倍，解析速度快 20~100 倍
- 通过 `.proto` 文件定义接口和数据结构，跨语言生成代码

**3. IDL（接口定义语言）**
通过 `.proto` 文件定义服务契约，自动生成客户端和服务端代码。

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
  int64 id = 1;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

#### 四种调用模式

| 模式 | 说明 | 场景 |
|------|------|------|
| **Unary（一元）** | 一请求一响应，类似普通函数 | 普通查询 |
| **Server Streaming** | 一请求，服务端流式返回多个响应 | 大数据下载、订阅推送 |
| **Client Streaming** | 客户端流式发送，服务端一次响应 | 文件上传、批量上报 |
| **Bidirectional Streaming** | 双向流，互相独立发送 | 聊天、实时同步 |

#### 与 REST 的对比

| 维度 | gRPC | REST + JSON |
|------|------|-------------|
| 协议 | HTTP/2 | HTTP/1.1（多） |
| 数据格式 | Protobuf（二进制） | JSON（文本） |
| 性能 | 高（小、快） | 一般 |
| 浏览器支持 | 需 gRPC-Web | 原生支持 |
| 可读性 | 差（二进制） | 好 |
| 接口契约 | 强约束（.proto） | 弱（靠文档） |
| 流式 | 原生支持四种 | 较弱（SSE/WebSocket） |
| 跨语言 | 自动生成 stub | 手写或 OpenAPI |

#### 优势

1. **高性能**：HTTP/2 + Protobuf，吞吐量是 REST 的 5~10 倍
2. **强类型契约**：`.proto` 文件即文档，编译期就能发现错误
3. **跨语言**：支持十几种主流语言
4. **流式调用**：原生支持双向流
5. **生态完善**：拦截器、负载均衡、超时、重试、TLS、认证都有支持
6. **代码生成**：自动生成客户端 stub 和服务端骨架

#### 劣势

1. **浏览器不友好**：需要 gRPC-Web 转换层
2. **调试困难**：二进制不能直接 curl，要用 grpcurl、BloomRPC 等工具
3. **学习成本**：需理解 Protobuf 语法和工具链
4. **可读性差**：抓包看到的是二进制
5. **生态依赖**：要维护 `.proto` 文件、版本兼容（字段编号不能改）

### 追问链

- gRPC 为什么选 HTTP/2 而不是 HTTP/1.1？
- gRPC 的四种调用模式各自适合什么场景？
- gRPC 怎么做服务发现和负载均衡？
- gRPC 和 Dubbo、Thrift 怎么选？

### 易错点

- 误以为 gRPC 是协议，其实它是框架（协议是 HTTP/2）
- 想直接在浏览器调用 gRPC（实际需要 gRPC-Web 转换层）
- 忽略 `.proto` 文件版本管理

---

## 问题二：Protobuf 序列化是什么？

### 主题

序列化协议

### 难度

中级

### 参考答案

**Protocol Buffers（简称 Protobuf 或 PB）**：Google 开源的语言无关、平台无关、可扩展的二进制序列化格式。

#### 什么是序列化

**序列化**：把内存中的对象（结构化数据）转成可传输/存储的字节流
**反序列化**：把字节流还原成对象

```
JS 对象 { id: 1, name: "Tom" }
   ↓ 序列化
字节流 0x08 0x01 0x12 0x03 0x54 0x6F 0x6D
   ↓ 网络传输 / 存储
   ↓ 反序列化
还原为对象
```

常见方案：JSON、XML、Protobuf、Thrift、MessagePack、Hessian、Avro。

#### Protobuf 核心思想

通过 `.proto` 文件定义数据结构 → 用 protoc 编译器生成各语言代码 → 调用生成的代码进行序列化/反序列化。

```
.proto 文件（契约）
        │
   protoc 编译
        │
   ┌────┼────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼
  Go   Java  JS  Py   C++   ← 各语言的 Model 类
```

#### `.proto` 文件示例

```protobuf
syntax = "proto3";

message User {
  int64  id    = 1;          // 字段编号（关键！）
  string name  = 2;
  string email = 3;
  repeated string tags = 4;  // 数组
  Address addr = 5;          // 嵌套
}

message Address {
  string city = 1;
  string zip  = 2;
}
```

注意：**字段后面的数字是"字段编号"，不是值**。它在序列化时代替字段名，是 Protobuf 体积小的关键。

#### 编码原理（为什么这么小）

**1. Tag-Length-Value（TLV）结构**

每个字段编码成：`[Tag (字段编号 + 类型)] [Length (可选)] [Value]`

举例：`{ id: 1, name: "Tom" }`

```
字段编号 1 (id, int64) → Tag = 0x08
值 1 (varint)         → 0x01
字段编号 2 (name, string) → Tag = 0x12
长度 3                  → 0x03
值 "Tom"                → 0x54 0x6F 0x6D

总共 7 字节
```

而同样的 JSON `{"id":1,"name":"Tom"}` 是 **20 字节**，体积差 3 倍。

**2. Varint（变长整数编码）**
- 小数字用 1 字节，大数字才用更多字节
- `1` → 1 字节，`300` → 2 字节，远比 int32 固定 4 字节省

**3. 不传字段名**
- JSON 每次都带 `"name"` 字符串
- Protobuf 只传字段编号

**4. 不传默认值**
- 字段是默认值（0、空字符串）时直接省略

#### 和 JSON 的对比

| 维度 | Protobuf | JSON |
|------|----------|------|
| 格式 | 二进制 | 文本 |
| 体积 | 小（约 1/3 ~ 1/10） | 大 |
| 速度 | 快（5~100 倍） | 慢 |
| 可读性 | 不可读 | 可读 |
| 类型 | 强类型 | 弱类型 |
| 契约 | `.proto` 强约束 | 靠文档 |
| 跨语言 | 自动生成代码 | 原生支持 |
| 调试 | 难（要解码） | 容易 |
| 浏览器 | 不友好 | 原生 |

#### 字段编号的重要性

字段编号是 Protobuf 的灵魂，决定了**前后兼容性**。

**规则：**
- 编号 1~15：占 1 字节（高频字段优先用）
- 编号 16~2047：占 2 字节
- **一旦使用，绝对不能改！** 改了等于换了字段
- 删除字段时建议保留编号或加 `reserved`

```protobuf
message User {
  reserved 3, 5;              // 保留，防止重用
  reserved "old_field";
  int64 id = 1;
  string name = 2;
  string email = 4;           // 跳过 3
}
```

**兼容性：**
- 新增字段：旧代码读新数据 → 忽略未知字段 ✅
- 删除字段：新代码读旧数据 → 字段为默认值 ✅
- 改编号：💥 数据错乱
- 改类型：⚠️ 视类型而定

#### 优劣总结

**优势**
- 体积小、速度快
- 强类型、强契约
- 跨语言、自动生成代码
- 良好的向前/向后兼容性

**劣势**
- 不可读，调试麻烦
- 需要 `.proto` 文件管理和编译流程
- 字段编号不能乱改，有维护成本
- 浏览器场景不友好

### 追问链

- Protobuf 为什么比 JSON 快？
- Varint 编码原理是什么？
- 字段编号一旦使用为什么不能改？
- Protobuf 和 Avro、Thrift 的区别？

### 易错点

- 修改已使用的字段编号，导致数据错乱
- 在 JS 中处理 int64 直接用 Number，丢失精度
- 删除字段后没加 `reserved`，新人复用编号
- 把字段编号当成数组索引或值

---

## 问题三：Protobuf 一般用在什么地方？

### 主题

架构设计 / 协议选型

### 难度

中级

### 参考答案

核心判断标准：**追求性能 + 跨语言 + 强契约** 的场景就用 Protobuf；**对外暴露 + 需要可读** 就用 JSON。

#### 主要应用场景

**一、微服务内部 RPC 通信（最主要场景）**

典型代表：gRPC

公司内部服务之间互相调用，对性能、延迟、稳定性要求高，不需要人类直接看懂内容。

```
订单服务 ──gRPC+Protobuf──► 用户服务
        ──gRPC+Protobuf──► 库存服务
        ──gRPC+Protobuf──► 支付服务
```

真实案例：Google 内部、Kubernetes、Etcd、TiDB、字节跳动、阿里、腾讯。

**二、消息队列与流式数据**

Kafka / Pulsar 的消息体常用 Protobuf。

```
生产者 ──Protobuf 编码──► Kafka ──Protobuf 解码──► 消费者
```

为什么用：消息量大（每天千亿级），体积小直接省带宽和磁盘；消费方语言可能不同；Schema 演进可控。

**三、移动端与服务端通信**

典型代表：IM、游戏、直播

- 流量贵、电量贵，二进制比 JSON 小
- 弱网下传输更快
- 解析速度快，省 CPU 和电池

真实案例：微信 IM 协议、王者荣耀、抖音、Google Maps、YouTube App。

**四、IoT 与嵌入式设备**

设备上报、远程控制场景。

- 设备算力弱、内存小
- 流量按字节计费
- 协议要稳定、向前兼容

真实案例：智能家居网关、车联网、工业物联网。

**五、大数据与分布式存储**

Hadoop、Spark、Flink 生态。

- 海量数据，每条省 1 字节就是 TB 级别
- 跨语言（Java + Scala + Python）
- Schema 可演进

真实案例：Hadoop 内部 RPC、Etcd、TiDB / TiKV、Prometheus 远程读写。

**六、配置文件与协议定义**

- Envoy / Istio 的 xDS 协议
- Kubernetes 的 API Server
- Bazel 构建系统

**七、缓存与持久化**

当 JSON 体积成为瓶颈时，存到 Redis / 文件 / 数据库 BLOB 用 Protobuf。

#### 不该用 Protobuf 的场景

❌ **对外开放 API** → 用 JSON / OpenAPI（第三方接入要可读）
❌ **浏览器直接调用** → 用 JSON（除非用 gRPC-Web）
❌ **数据量小、QPS 低** → 引入工具链不划算
❌ **需要人工查日志** → 二进制不能直接看
❌ **Schema 频繁大改的早期项目** → JSON 更灵活

#### 选型决策树

```
要序列化数据
    │
    ├─ 对外 API / 浏览器 → JSON
    │
    ├─ 内部微服务 RPC ──┐
    │                   ├─ 性能敏感 → Protobuf (gRPC)
    │                   └─ 一般业务 → JSON (REST)
    │
    ├─ 大流量 MQ 消息 → Protobuf / Avro
    │
    ├─ 移动端/IoT 协议 → Protobuf
    │
    └─ 配置文件 → JSON / YAML / TOML
```

### 追问链

- 为什么对外 API 不推荐用 Protobuf？
- 移动端用 Protobuf 还是 JSON？怎么权衡？
- Kafka 消息体怎么选序列化协议？
- 配置文件用 Protobuf 合适吗？

### 易错点

- 给浏览器/第三方暴露 Protobuf 接口，导致接入困难
- 小项目过早引入 Protobuf，工具链成本高
- 同一系统内多种序列化协议混用，没有统一规范

### 知识延伸

- gRPC-Web：浏览器使用 Protobuf 的方案
- Avro vs Protobuf：大数据场景选型
- Schema Registry：Kafka 中的 Schema 管理
- FlatBuffers：零拷贝场景的更极致选择
