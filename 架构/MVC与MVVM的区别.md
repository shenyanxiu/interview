# MVC 与 MVVM 的区别

## 问题

MVC 和 MVVM 有什么区别？

## 主题

架构模式

## 难度

中级

## 参考答案

### MVC（Model-View-Controller）

```
用户操作 → Controller → Model → View
                ↑                    |
                └────────────────────┘
```

- **Model**：数据层，负责业务逻辑和数据管理
- **View**：视图层，负责 UI 展示
- **Controller**：控制器，接收用户输入，协调 Model 和 View

特点：
- Controller 是中心枢纽，负责接收请求、调用 Model、选择 View
- View 和 Model 之间可以直接通信（View 可以直接读取 Model 数据）
- 典型代表：Spring MVC、Express 路由模式

### MVVM（Model-View-ViewModel）

```
View ←→ ViewModel ←→ Model
     (双向绑定)    (数据请求)
```

- **Model**：数据层，同 MVC
- **View**：视图层，模板/DOM
- **ViewModel**：视图模型，连接 View 和 Model 的桥梁，处理视图逻辑

特点：
- View 和 ViewModel 之间通过**数据绑定**自动同步
- View 不直接操作 Model，完全通过 ViewModel 中转
- 典型代表：Vue、Angular

### 核心区别

| 维度 | MVC | MVVM |
|------|-----|------|
| 通信方式 | Controller 手动协调 | 数据绑定自动同步 |
| View 与 Model | 可直接通信 | 完全解耦，通过 ViewModel |
| 驱动方式 | 事件驱动 | 数据驱动 |
| Controller/ViewModel 职责 | 流程控制、路由分发 | 状态管理、视图逻辑 |
| 适用场景 | 后端（服务端渲染） | 前端（SPA 应用） |
| DOM 操作 | View 层手动更新 | 框架自动更新 |

### 通俗类比

- **MVC**：你是包工头（Controller），数据变了你手动通知画师（View）重新画，用户操作了你手动通知仓库（Model）更新。所有事你协调。
- **MVVM**：你有一台自动同步机器人（数据绑定），你只管改数据，页面自动更新。

```javascript
// MVC 思路：手动同步
addBtn.addEventListener('click', () => {
  const text = input.value;
  todoList.push(text);                       // 更新 Model
  const li = document.createElement('li');   // 手动更新 View
  li.textContent = text;
  ul.appendChild(li);
});

// MVVM 思路：改数据就行，页面自动变
const todos = reactive([]);
function addTodo() {
  todos.push(input.value);  // 就这一行，页面自动多出一个 <li>
}
```

### 为什么前端从 MVC 转向 MVVM

在传统前端 MVC（如 Backbone）中，Controller 需要手动监听事件、操作 DOM、同步数据，随着应用复杂度增长，维护成本急剧上升。MVVM 通过数据绑定解决了这个问题：开发者只需关注数据状态的变化，框架自动完成 DOM 更新。

## 追问链

- MVVM 中的双向绑定是怎么实现的？（Vue 2 用 Object.defineProperty，Vue 3 用 Proxy）
- React 算 MVVM 吗？（严格说不算，React 是单向数据流，但思想类似）
- MVC 在后端和前端的表现有什么不同？

## 易错点

- 误以为 MVC 中 View 和 Model 完全隔离（实际上 View 可以直接读取 Model）
- 混淆 Controller 和 ViewModel 的职责（Controller 做流程控制，ViewModel 做状态管理）
- 认为 MVVM 一定是双向绑定（React 就是单向数据流 + ViewModel 思想）

## 知识延伸

- MVP 模式（Model-View-Presenter）：View 和 Model 完全隔离，Presenter 做中间人
- 单向数据流（Flux/Redux）：React 生态的状态管理思路
- Vue 响应式原理：Proxy / Object.defineProperty 实现数据劫持
