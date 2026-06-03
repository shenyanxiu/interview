# React Hooks 原理

## 问题

React Hooks 的底层原理是什么？为什么不能在条件语句中使用 Hooks？

## 难度

中级 / 高级

## 参考答案

### 核心机制

React Hooks 的底层实现依赖于**链表**和**闭包**：

1. **Fiber 节点上的链表** — 每个组件对应一个 Fiber 节点，节点上有一个 `memoizedState` 属性，指向一条由 hook 对象组成的单向链表。每次调用一个 hook（`useState`、`useEffect` 等），就在链表上创建或读取一个节点。

2. **调用顺序即索引** — React 不靠"名字"区分 hook，而是靠**调用顺序**。这就是为什么 hooks 不能放在条件/循环里——一旦顺序变了，链表对位就错了。

3. **两套 dispatcher** — React 内部维护 `mountDispatcher` 和 `updateDispatcher`。首次渲染走 mount 逻辑（创建 hook 节点），后续渲染走 update 逻辑（读取已有节点并比较）。

### useState 简化实现

```js
let workInProgressHook = null; // 当前正在处理的 hook 指针

function useState(initialValue) {
  // mount 阶段：创建 hook 节点
  if (isMount) {
    const hook = { memoizedState: initialValue, queue: [], next: null };
    // 挂到链表尾部
    appendHook(hook);
  }

  // 取当前 hook 节点
  const hook = workInProgressHook;
  workInProgressHook = hook.next;

  // 处理 pending 的 setState 队列
  hook.queue.forEach(action => {
    hook.memoizedState = typeof action === 'function'
      ? action(hook.memoizedState)
      : action;
  });
  hook.queue = [];

  // setState 闭包捕获当前 hook 引用
  const setState = (action) => {
    hook.queue.push(action);
    scheduleRerender(); // 触发重新渲染
  };

  return [hook.memoizedState, setState];
}
```

关键点：`setState` 通过闭包持有对 hook 对象的引用，所以即使组件重新执行，更新仍然能写到正确的位置。

### useEffect 原理

```js
function useEffect(callback, deps) {
  const hook = getCurrentHook();

  if (isMount) {
    hook.memoizedState = { deps, cleanup: undefined };
    // 标记为需要在 commit 阶段执行
    pushEffect(callback, hook);
  } else {
    const prevDeps = hook.memoizedState.deps;
    // 浅比较依赖数组
    if (!shallowEqual(prevDeps, deps)) {
      pushEffect(callback, hook);
    }
    hook.memoizedState.deps = deps;
  }
}
```

- effect 不在 render 阶段执行，而是在 **commit 阶段异步调度**（`useLayoutEffect` 则是同步）。
- 每次执行新 effect 前，先调用上一次的 cleanup 函数。

### 为什么不能在条件语句中使用 Hooks

```js
// ❌ 错误示例
if (condition) {
  const [a, setA] = useState(0);  // hook #1 可能消失
}
const [b, setB] = useState(1);    // hook #2 变成 #1，对位错乱
```

链表是按顺序遍历的，少了一个节点，后面所有 hook 都会读到错误的状态。

### 整体流程

```
组件函数执行
  ↓
按顺序调用 hooks → 遍历 Fiber.memoizedState 链表
  ↓
useState: 读取/更新 state
useEffect: 收集 effect，标记到 Fiber 的 updateQueue
useMemo/useCallback: 比较 deps，决定是否重新计算
  ↓
render 阶段结束，生成新的 Virtual DOM
  ↓
commit 阶段：DOM 更新 → 执行 effects
```

## 追问链

- `useMemo` 和 `useCallback` 的区别是什么？它们的缓存机制如何实现？
- React 的 Fiber 架构是什么？为什么需要它？
- `useRef` 为什么不会触发重新渲染？它和 `useState` 在实现上有什么区别？
- React 18 的并发模式下，Hooks 的行为有什么变化？

## 易错点

- 在条件语句、循环或嵌套函数中调用 Hooks，导致调用顺序不稳定
- 误以为 `setState` 是同步的，实际上是批量异步更新
- `useEffect` 的依赖数组遗漏变量，导致闭包捕获过期值（stale closure）
- 混淆 `useEffect`（异步）和 `useLayoutEffect`（同步）的执行时机

## 知识延伸

- **React Fiber 架构**：Hooks 依附于 Fiber 节点，理解 Fiber 的链表结构有助于深入理解 Hooks
- **闭包陷阱**：Hooks 大量使用闭包，理解 JS 闭包机制是前提
- **React 调度器（Scheduler）**：`setState` 触发的重渲染如何被调度和优先级排序
- **自定义 Hooks**：本质是对内置 Hooks 的组合封装，共享同一套链表机制
