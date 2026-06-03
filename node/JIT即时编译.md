# JIT（即时编译）是什么

## 问题

JIT 是什么？它的工作原理是怎样的？

## 主题

Node / V8 / 编译原理

## 难度

中级

## 参考答案

JIT（Just-In-Time）是即时编译，指程序在运行时将代码编译为机器码执行，而不是提前编译或纯解释执行。

### 核心思路

传统有两种执行方式：

- **解释执行**：逐行翻译执行，启动快但运行慢
- **AOT 编译**（Ahead-Of-Time）：提前编译成机器码，启动慢但运行快

JIT 是两者的折中——先解释执行，对**热点代码**（频繁执行的函数/循环）进行编译优化，下次直接执行机器码。

### V8 中的 JIT 流程

1. **Ignition**（解释器）：将 JS 编译为字节码，逐条解释执行
2. **热点探测**：Ignition 收集类型反馈和执行频率信息
3. **TurboFan**（优化编译器）：对热点代码做类型特化、内联等优化，编译为高效机器码
4. **去优化（Deopt）**：如果运行时类型假设被打破，回退到字节码重新解释

```
JS 源码 → 字节码(Ignition) → 解释执行
                                  ↓ 热点
                            机器码(TurboFan) → 直接执行
```

### 为什么需要 JIT

- 纯解释太慢，无法满足复杂应用的性能需求
- 纯 AOT 对动态语言（如 JS）不现实，因为类型在运行时才确定
- JIT 能利用运行时信息做**投机优化**（speculative optimization），比静态编译更激进

### 常见使用 JIT 的运行时

| 运行时 | JIT 编译器 |
|--------|-----------|
| V8 (Node/Chrome) | TurboFan |
| JVM (Java) | C1/C2 (HotSpot) |
| .NET (C#) | RyuJIT |
| LuaJIT | DynASM |

## 追问链

- V8 的 Ignition 和 TurboFan 分别做了什么？
- 什么是去优化（Deoptimization）？什么情况下会触发？
- JIT 和 AOT 各自的优缺点是什么？适用场景？
- 隐藏类（Hidden Class）和内联缓存（Inline Cache）如何辅助 JIT 优化？

## 易错点

- 误以为 V8 直接将 JS 编译为机器码，忽略了字节码阶段
- 混淆 JIT 和 AOT，认为 JIT 就是"编译"
- 不了解去优化机制，认为一旦编译为机器码就不会回退
- 忽略 JIT 的预热成本——首次执行仍然是解释执行，性能提升需要多次调用后才体现

## 知识延伸

- V8 早期使用 Full-Codegen + Crankshaft，后来替换为 Ignition + TurboFan 架构
- WebAssembly 采用 AOT 模式，与 JS 的 JIT 形成互补
- GraalVM 的 Truffle 框架支持为任意语言生成 JIT 编译器
- Java 的分层编译（Tiered Compilation）思路与 V8 类似：C1 快速编译 → C2 深度优化
