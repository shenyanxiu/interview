# Vue 与 React 的区别

## 问题

Vue 和 React 有什么区别？

## 主题

前端框架

## 难度

中级

## 参考答案

### 响应式原理

**Vue：自动追踪依赖（推 push）**

Vue 通过 Proxy 劫持数据，自动知道"谁用了这个数据"，数据变了精确通知对应的组件更新。

```javascript
const count = ref(0);
count.value++;  // 精确更新，不多不少
```

**React：手动触发对比（拉 pull）**

React 不追踪数据，你调 setState 后，它从当前组件开始重新执行整棵子树的 render，然后通过 diff 算出哪里变了。

```javascript
const [count, setCount] = useState(0);
setCount(count + 1);  // 触发整棵子树 re-render，再 diff
```

**类比**：
- Vue 像快递员有你的精确地址，包裹直接送到门口
- React 像快递员只知道你在这栋楼，每次都挨家挨户敲门问"是你的吗"（diff）

### 模板 vs JSX

```html
<!-- Vue：模板语法，HTML 增强 -->
<template>
  <ul>
    <li v-for="item in list" :key="item.id">
      {{ item.name }}
    </li>
  </ul>
</template>
```

```jsx
// React：JSX，JavaScript 里写 HTML
function List({ list }) {
  return (
    <ul>
      {list.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

### 数据流与状态管理

| 维度 | Vue | React |
|------|-----|-------|
| 数据绑定 | 双向（v-model） | 单向（受控组件） |
| 状态修改 | 直接赋值 `state.x = 1` | 必须用 setter `setState(1)` |
| 组件通信 | props / emit / provide-inject | props / callback / context |
| 副作用 | watchEffect / watch | useEffect |

### 性能优化策略

**Vue：默认就很快，几乎不用手动优化**

响应式是精确追踪的，只有真正变了的组件才会更新。

**React：需要手动优化**

默认会 re-render 整棵子树，需要用 `memo`、`useMemo`、`useCallback` 来避免不必要的重渲染。

```jsx
// React 常见优化
const MemoChild = React.memo(Child);
const handler = useCallback(() => {}, []);
const value = useMemo(() => expensiveCalc(), [dep]);
```

Vue 里这些基本不需要。

### 心智模型对比

| | Vue | React |
|--|-----|-------|
| 核心思想 | 可变数据 + 自动追踪 | 不可变数据 + 函数式 |
| 写法感受 | 更像传统开发，直觉友好 | 更函数式，需要理解闭包/引用 |
| 学习曲线 | 上手快，模板好理解 | 概念少但陷阱多（闭包陈旧值等） |
| 灵活性 | 框架约束多，写法统一 | 极度灵活，写法多样 |
| 生态 | 官方全家桶（Router/Pinia/Vite） | 社区方案为主，选择多 |

### 怎么选

- **Vue**：团队水平参差、想快速交付、中后台项目 → 约束多不容易写烂
- **React**：团队能力强、需要高度灵活、复杂交互型应用 → 天花板高但下限也低

两者都能做任何项目，核心区别不在能力，在编程范式和开发体验上。

## 追问链

- Vue 的响应式原理具体怎么实现的？（Proxy + effect + track/trigger）
- React 的 Fiber 架构解决了什么问题？（可中断渲染、时间切片）
- Vue 的编译时优化有哪些？（静态提升、patchFlag、Block Tree）
- React 的 useMemo 和 Vue 的 computed 有什么区别？

## 易错点

- 误以为 React 没有"响应式"（React 有响应式，只是粒度不同——组件级而非属性级）
- 认为 Vue 双向绑定 = 性能差（v-model 只是语法糖，底层仍是单向数据流）
- 混淆 Virtual DOM 的作用（两者都用 VDOM，但更新触发机制不同）
- 认为 React 的 re-render 等于 DOM 更新（re-render 只是重新执行函数，不一定操作 DOM）

## 知识延伸

- Vue 3 编译优化：静态提升、patchFlag 标记动态节点
- React Server Components：服务端组件，减少客户端 JS 体积
- Signals 模式：Solid.js / Preact Signals，细粒度响应式的新方向
- React Compiler（React Forget）：自动 memo 化，未来可能消除手动优化
