# React 虚拟 DOM 与 Diff 算法

## 问题

React 的虚拟 DOM 是什么？Diff 算法是怎么工作的？

## 主题

React / 渲染机制

## 难度

中级 / 高级

## 参考答案

### 虚拟 DOM 是什么

虚拟 DOM 就是用 JS 对象描述真实 DOM 结构。

```js
// JSX
<div className="box"><span>hi</span></div>

// 编译后的虚拟 DOM
{
  type: 'div',
  props: { className: 'box' },
  children: [
    { type: 'span', props: {}, children: ['hi'] }
  ]
}
```

为什么要有它：

1. **批量更新** — 直接操作真实 DOM 一次只能改一处，频繁修改触发重排重绘。VDOM 在内存里先算出最终结果，再一次性提交到 DOM。
2. **跨平台** — VDOM 只是一棵 JS 对象树，可以渲染到浏览器（DOM）、原生（React Native）、Canvas 等。
3. **声明式编程** — 你只描述"UI 应该长什么样"，框架负责算"怎么变过去"。

注意：VDOM 不一定比直接操作 DOM 快，它的价值是**性能下限可控** + **开发体验**。

### 为什么需要 Diff 妥协

完整的树 diff 是经典算法问题，最优解 O(n³)。1000 个节点要十亿次比较，UI 框架根本扛不住。

React 用启发式策略换性能，基于两个假设把复杂度降到 **O(n)**：

1. **不同类型的元素产生不同的树** — `type` 不一样直接整棵替换，不深入比较
2. **同层级才比较** — 跨层级移动按"删除 + 新建"处理，不去找复用

代价是某些场景（跨层级移动、type 切换）会做不必要的销毁重建，但对 99% 的 UI 场景够用。

### Diff 的三个层级

#### Tree Diff（树层）

只对比同一层节点，发现差异走最简单的路径。

```jsx
// 旧
<div><Child /></div>
// 新 - 根节点 div 变成 section
<section><Child /></section>
```

`div ≠ section` → 整棵旧子树销毁，新子树重建。哪怕 `<Child />` 完全没变，state 也全丢。跨层级移动同理，不会被识别成"移动"，而是"删除 + 新增"。

#### Component Diff（组件层）

| 情况 | 处理 |
|---|---|
| type 相同 | 复用组件实例，更新 props，进入子节点 diff |
| type 不同 | 卸载旧组件，挂载新组件 |
| `shouldComponentUpdate` / `React.memo` 返回 false | 跳过整个子树的 diff |

#### Element Diff（元素层 / 列表层）

最关键的一层，处理同一父节点下的子节点列表，靠 **key** 识别身份。

### Element Diff 详细流程

源码在 `react-reconciler/src/ReactChildFiber.js` 的 `reconcileChildrenArray`。React 用**两轮单向遍历 + Map 查找**（不是双端对比，那是 Vue 的策略）。

#### 第一轮：从左向右顺序对比

假设大多数情况下列表头部稳定，先尝试快速通过：

```js
// 伪代码
let i = 0;
let oldFiber = firstChild;
let lastPlacedIndex = 0;

for (; oldFiber && i < newChildren.length; i++) {
  const newChild = newChildren[i];
  if (oldFiber.key !== newChild.key) break;     // key 对不上
  if (oldFiber.type !== newChild.type) break;   // type 不同

  cloneFiber(oldFiber, newChild.props);
  lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, oldFiber.index);
  oldFiber = oldFiber.sibling;
}
```

第一轮可能的结局：

- 老链表先用完 → 新的多出来 → 剩下的全部新建
- 新数组先用完 → 老的多出来 → 剩下的老 fiber 全标 deletion
- 两边都没用完，但 key 对不上 → 进入第二轮

#### 第二轮：剩余节点用 Map 匹配

```js
const existingChildren = new Map();
let f = oldFiber;
while (f) {
  existingChildren.set(f.key ?? f.index, f);
  f = f.sibling;
}

for (; i < newChildren.length; i++) {
  const newChild = newChildren[i];
  const matched = existingChildren.get(newChild.key ?? i);

  if (matched && matched.type === newChild.type) {
    const newFiber = useFiber(matched, newChild.props);
    existingChildren.delete(newChild.key ?? i);
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, matched.index);
  } else {
    placeChild(createFiber(newChild), lastPlacedIndex, -1);
  }
}

// Map 里剩下的全删除
existingChildren.forEach(f => deletions.push(f));
```

#### lastPlacedIndex：判断是否移动

React 列表 diff 最巧妙的地方。用一个递增指针记录"已确认不需要移动的最大老索引"：

```js
function placeChild(newFiber, lastPlacedIndex, oldIndex) {
  if (oldIndex < lastPlacedIndex) {
    // 节点在老列表里更靠前，但现在被排到后面 → 必须移动
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  } else {
    // 位置相对在后，正常往后挂
    return oldIndex;
  }
}
```

关键洞察：**只标记"哪些节点需要往后移"，不标记"哪些往前移"**。因为 DOM 操作是 `insertBefore`，把后面的节点往后挪一遍，前面的节点位置自然就对了。

#### 完整例子

```
旧: [A(idx=0), B(idx=1), C(idx=2), D(idx=3)]
新: [B, A, D, C]
```

第一轮：新[0]=B vs 旧[0]=A，key 不同 → 跳出，i=0

第二轮：

| 步骤 | 新节点 | oldIndex | lastPlacedIndex 更新 | 操作 |
|---|---|---|---|---|
| i=0 | B | 1 | 0 → 1 | 不动 |
| i=1 | A | 0 | 0 < 1 | **移动** A |
| i=2 | D | 3 | 1 → 3 | 不动 |
| i=3 | C | 2 | 2 < 3 | **移动** C |

最终 A 和 C 标记 Placement，commit 阶段执行两次 `insertBefore`。

### key 的精确语义

`key` 用来在两次渲染之间**识别同一个逻辑节点**。

复用一个 fiber 需要两个条件**同时满足**：

```text
type 相同  AND  key 相同（都为 undefined 也算相同）
```

| type | key | 结果 |
|---|---|---|
| 相同 | 相同 | 复用 fiber |
| 相同 | 不同 | 销毁重建 |
| 不同 | 任意 | 销毁重建 |

#### key 的选择策略

| 场景 | key 选择 |
|---|---|
| 静态列表（不会增删改顺序） | 用 index 也行 |
| 动态列表（会插入/删除/排序） | 必须用稳定 ID |
| 列表项有内部状态（input、表单） | 必须用稳定 ID |

用 index 当 key 的坑：

```jsx
// 列表头部插入新项
// 旧: [{id:1, val:'A'}, {id:2, val:'B'}]，index key: 0, 1
// 新: [{id:3, val:'C'}, {id:1, val:'A'}, {id:2, val:'B'}]，index key: 0, 1, 2

// React 看到 key 0 还在 → 复用第一个组件，把内容从 A 改成 C
// 如果第一个组件是 <input>，里面已输入的内容、focus 状态全错乱
```

key 不能用 `Math.random()`，每次都不一样等于全部新建。

### Fiber 架构对 Diff 的影响

React 16 之前是递归 diff，开始了就停不下来，大列表会卡顿。Fiber 把 VDOM 改成链表结构：

```js
{
  child,     // 第一个子节点
  sibling,   // 下一个兄弟
  return,    // 父节点（之所以叫 return 是因为遍历完子树要"返回"父节点）
  alternate, // 指向另一棵树的对应节点（双缓冲）
  flags,     // 副作用标记：Placement / Update / Deletion ...
}
```

#### Render 阶段（可中断）

```js
function workLoop(deadline) {
  while (workInProgress && deadline.timeRemaining() > 1) {
    workInProgress = performUnitOfWork(workInProgress);
  }
  if (workInProgress) {
    requestIdleCallback(workLoop); // 让出主线程，下帧继续
  } else {
    commitRoot();
  }
}
```

`performUnitOfWork` 做两件事：

- **beginWork**：向下走，对当前 fiber 做 diff，生成或复用子 fiber
- **completeWork**：没有子节点了，向上回溯，收集副作用

#### Commit 阶段（不可中断）

Diff 阶段只是在内存里标记 flags，真正的 DOM 操作在 commit。三个子阶段：

1. **before mutation**：读 DOM 状态，触发 `getSnapshotBeforeUpdate`
2. **mutation**：根据 flags 操作 DOM
3. **layout**：触发 `componentDidMount/Update`、`useLayoutEffect`

为什么 commit 不能中断？因为中途让出会出现"DOM 改了一半"的状态，用户能看到错乱画面。

#### 双缓冲（Double Buffering）

内存里始终有两棵 Fiber 树：

- `current`：当前屏幕显示的
- `workInProgress`：正在 diff 的新树

新树构建完，`fiberRoot.current = workInProgress` 一行代码切换。旧的 current 不丢，下次更新时清掉 flags 复用为新的 workInProgress。

### 双端 diff vs 单端 + Map（React vs Vue）

| 维度 | React | Vue 2 | Vue 3 |
|---|---|---|---|
| 策略 | 单向遍历 + Map | 双端对比 + Map 兜底 | 双端 + 最长递增子序列 |
| 移动判定 | lastPlacedIndex | 头尾指针四种命中 | LIS 算最少移动 |
| 反转列表 | 全部需要移动 | 一次性识别为反转 | 一次性识别 |
| 实现复杂度 | 较低 | 中等 | 较高 |

React 的策略对**列表反转**这种极端场景不友好（n-1 次移动），但代码简单、对常见场景（增删少量项）足够好。实际场景中 diff 性能一般不是瓶颈，差距在毫秒级。

## 追问链

- 为什么 React 选择单端 + Map 而不是双端对比？
- Fiber 链表结构相比之前的递归有哪些优势？除了可中断还有什么？
- React 18 的并发渲染下，diff 出来的结果可以被丢弃吗？
- `useDeferredValue` 和 `useTransition` 怎么影响 diff 的优先级？
- 为什么 `key` 不能加在 `<></>` 上但能加在 `<Fragment key=...>` 上？
- 组件级的 `type` 具体指什么？

## 易错点

- 以为虚拟 DOM 一定比真实 DOM 快（不一定，是"性能下限"，不是"性能上限"）
- 以为 React 会智能识别跨层移动（不会，跨层一律销毁重建）
- 以为 type 相同就一定复用 fiber（还要 key 也匹配）
- 把 key 设为数组 index，且列表会变化
- 把 key 设为 `Math.random()`，等于全部新建
- 期望 `React.memo` 自动深比较（默认是浅比较，引用类型 props 每次都不等）
- 混淆 re-render 和 DOM 更新（组件函数执行只是在算新 VDOM，diff 没差异 DOM 不动）
- 以为 commit 阶段也可中断（不行，会撕裂画面）

## 知识延伸

- **Reconciler vs Renderer**：diff 算法在 reconciler，DOM 操作在 react-dom。同一个 reconciler 可以驱动 DOM、Native、Canvas 等多种 renderer
- **Vue 3 的 LIS**：最长递增子序列算法，算最少移动次数
- **Inferno、Preact 的 diff**：思路类似 React，但实现细节有优化
- **Svelte 无 VDOM**：编译时直接生成精确的 DOM 操作代码，跳过 diff
- **Solid.js / Signals**：细粒度响应式，组件只执行一次，更新走 signal 而非 diff
- **React Compiler（React Forget）**：未来自动 memo 化，可能消除手动优化
