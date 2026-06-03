# v-for 与 v-if 不能同时使用

## 问题

Vue 中为什么 v-for 和 v-if 不能同时用在同一个元素上？

## 主题

Vue

## 难度

中级

## 参考答案

核心是**优先级问题导致性能浪费或直接报错**，且 Vue 2 与 Vue 3 行为不同。

### Vue 2：v-for 优先级高于 v-if

```html
<!-- 反例 -->
<li v-for="user in users" v-if="user.isActive">
  {{ user.name }}
</li>
```

编译后大致等价于：

```js
users.map(user => {
  if (user.isActive) {
    return h('li', user.name)
  }
})
```

问题：**每次渲染都要遍历整个 `users` 数组**，即使绝大多数 `isActive` 都是 `false`。1000 项里只有 10 项激活，依然要循环 1000 次，浪费性能。

### Vue 3：v-if 优先级高于 v-for

```html
<li v-for="user in users" v-if="user.isActive">
  {{ user.name }}
</li>
```

Vue 3 会先执行 `v-if`，但此时 `user` 还没被 `v-for` 定义，会直接报错（`user is not defined`），代码根本跑不起来。

可以理解为：**Vue 3 用报错强制开发者规范写法**。

### 推荐写法

#### 方案一：用 computed 提前过滤（最佳实践）

```js
computed: {
  activeUsers() {
    return this.users.filter(user => user.isActive)
  }
}
```

```html
<li v-for="user in activeUsers" :key="user.id">
  {{ user.name }}
</li>
```

优势：
- 过滤结果会被缓存，依赖未变化时不会重复计算
- 模板更清晰，关注点分离
- 避免每次渲染都做一次过滤

#### 方案二：用 template 包一层（条件不依赖循环变量时）

```html
<template v-if="shouldShow">
  <li v-for="user in users" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

适用于"是否渲染整个列表"取决于外部变量的场景。

### 总结对比

| 维度 | Vue 2 | Vue 3 |
|------|-------|-------|
| 优先级 | v-for > v-if | v-if > v-for |
| 同时使用的后果 | 性能浪费，先遍历全部再过滤 | 直接报错，变量未定义 |
| 官方风格指南 | 强烈不推荐（A 级规则） | 强烈不推荐（A 级规则） |

## 追问链

- Vue 3 为什么要把 v-if 的优先级调到 v-for 之上？（强制规范写法、避免性能陷阱）
- 为什么 computed 比模板里的 filter 更好？（缓存、依赖追踪、可复用）
- 在 `<template v-for>` 上能不能加 v-if？（可以，但要看条件是否依赖循环变量）
- 大列表渲染除了过滤还有哪些优化手段？（虚拟列表、分页、key 优化、懒加载）

## 易错点

- 用 Vue 2 的认知去 Vue 3 写代码，结果直接报错
- 在循环里用 v-if 做"过滤"，列表很大时性能问题不易察觉
- 误以为 `<template>` 能解决所有 v-for + v-if 的场景（如果条件依赖循环变量，依然有问题）
- 把过滤逻辑塞在模板里而不是 computed，导致每次重渲染都重新计算

## 知识延伸

- Vue 官方风格指南把"避免 v-for 和 v-if 同元素使用"列为**必要规则（Priority A）**
- Vue 3 编译优化：静态提升、patchFlag、Block Tree 让动态节点的 diff 成本大幅下降
- 大列表性能优化：`vue-virtual-scroller`、`vue-virtual-scroll-list` 等虚拟滚动库
- React 中没有这个问题，因为 JSX 里循环和条件都是 JS 表达式，写法天然分离
