# React 元素的 type 是什么

## 问题

React 中"组件级的 type"具体指什么？diff 时是怎么用它的？

## 主题

React / 渲染机制

## 难度

中级

## 参考答案

每个 React 元素（JSX 编译后的产物）都有一个 `type` 字段，diff 时就是靠它判断"是不是同一种东西"。

JSX 经过编译变成 `React.createElement(type, props, ...children)`，第一个参数就是 `type`：

```jsx
<div />              // type === 'div'           （字符串）
<MyComp />           // type === MyComp          （函数/类引用）
<>...</>             // type === Symbol(react.fragment)
<Suspense />         // type === Symbol(react.suspense)
```

### type 的四种形态

#### 字符串：原生 DOM 标签

小写开头的标签，`type` 是字符串：

```jsx
<div />   // { type: 'div', ... }
<span />  // { type: 'span', ... }
```

字符串的 `===` 比较就是字符值相等。

#### 函数引用：函数组件

大写开头的组件，`type` 是函数本身：

```jsx
function Button() { return <button />; }

<Button />
// { type: Button, ... }   ← 这里的 Button 就是上面那个函数
```

注意 **type 是函数引用，不是名字**。两个同名但不同的函数是不同的 type：

```jsx
function makeButton() {
  return function Button() { return <button />; };
}

const A = makeButton();  // function Button
const B = makeButton();  // function Button（同名，但是不同函数）

A === B  // false
```

#### 类引用：Class 组件

```jsx
class List extends React.Component { ... }

<List />
// { type: List, ... }   ← 类构造器引用
```

和函数组件一样，type 是类本身的引用。

#### Symbol：内置特殊类型

Fragment、Suspense、Portal、Provider 等，`type` 是 React 内置的 Symbol：

```js
Symbol(react.fragment)
Symbol(react.suspense)
Symbol(react.portal)
Symbol(react.context)         // <Context.Provider>
Symbol(react.memo)            // React.memo 包出来的
Symbol(react.forward_ref)     // forwardRef 包出来的
```

`React.memo(Comp)` 返回的不是函数本身，而是一个 `{ $$typeof: Symbol(react.memo), type: Comp }` 对象，diff 时识别这个壳，再深入比较内部的 `Comp`。

### diff 怎么用 type

```js
// reconciler 简化逻辑
if (oldFiber.type === newElement.type) {
  // 复用 fiber 节点，更新 props，进入子节点 diff
} else {
  // 销毁旧 fiber 整棵子树，挂载新的
}
```

判断就是一个 `===`，没有任何"看起来像不像"的智能识别。

#### 几个具体例子

**标签变了**

```jsx
// 旧
<div><Child /></div>
// 新
<section><Child /></section>
```

`'div' !== 'section'` → 整棵子树销毁，`<Child />` 也跟着卸载，state 丢失。

**组件变了**

```jsx
// 旧: <Button label="ok" />
// 新: <Link label="ok" />
```

`Button !== Link` → Button 卸载，Link 挂载。

**同函数不同 props**

```jsx
// 旧: <Button label="ok" />
// 新: <Button label="cancel" />
```

`Button === Button` → 复用 fiber，只更新 props，state 保留。

**条件渲染换组件**

```jsx
{isAdmin ? <AdminPanel /> : <UserPanel />}
```

`isAdmin` 变化时 type 变，整个面板重建。两个面板内部即使有相同结构的输入框，输入内容也会丢。

### 常见的坑：内联组件定义

```jsx
function Parent({ data }) {
  // ❌ 每次 Parent 渲染，Inner 都是新函数
  const Inner = () => <div>{data}</div>;
  return <Inner />;
}
```

每次 Parent 渲染：

1. `Inner` 是新创建的箭头函数
2. `<Inner />` 的 type 是新函数引用
3. 旧 type !== 新 type → diff 判定不同，**Inner 整个销毁重建**
4. 如果 Inner 内部有 useState、useEffect，state 全丢、effect 重新跑

正确做法是把组件定义提到外面：

```jsx
// ✅ 提到外面
function Inner({ data }) { return <div>{data}</div>; }

function Parent({ data }) {
  return <Inner data={data} />;
}
```

### type 和 key 的关系

diff 复用一个 fiber 需要**两个条件同时满足**：

```text
type 相同  AND  key 相同（都为 undefined 也算相同）
```

| type | key | 结果 |
|---|---|---|
| 相同 | 相同 | 复用 fiber |
| 相同 | 不同 | 视为不同节点，销毁重建 |
| 不同 | 任意 | 销毁重建 |

key 主要在列表里区分兄弟节点，type 在所有 diff 比较里都用。

### 一句话总结

**type 就是 React 元素描述对象里的一个字段，对应 JSX 标签那个位置的值**——可能是字符串（原生标签）、函数/类引用（组件）、或内置 Symbol（Fragment 等）。diff 用 `===` 比较 type 决定复用还是重建。

## 追问链

- `React.memo` 返回的对象 `$$typeof` 是什么？为什么需要这个标记？
- 为什么 `<></>` 和 `<Fragment>` 在 type 上是同一个 Symbol？
- HMR（热更新）下，组件函数引用变了，状态为什么没丢？
- `forwardRef` 包装后的组件 type 是什么？diff 行为有何不同？
- 为什么 `$$typeof` 要用 Symbol 而不是字符串？（防 XSS）

## 易错点

- 以为同名函数就是同一个 type（type 是引用，不是名字）
- 在父组件里内联定义子组件，导致每次 re-render 都重建
- 以为 `React.memo(Comp)` 还是函数（其实是带 `$$typeof` 的对象）
- 三元渲染两个不同组件时，期望共享 state（type 不同会销毁重建）
- 把高阶组件返回的新组件用作 type，但每次渲染都重新调用 HOC（每次都是新 type）

## 知识延伸

- **`$$typeof` 防 XSS**：React 元素对象上还有一个 `$$typeof: Symbol(react.element)` 字段，因为 Symbol 不能被 JSON 序列化，所以从服务端返回的恶意数据无法伪造成合法 React 元素
- **`React.isValidElement`**：通过检查 `$$typeof` 判断是不是 React 元素
- **`React.Children.map` 的 key 重写**：内部会给元素的 key 加上前缀，避免和外层冲突
- **Reconciler 源码**：`createFiberFromElement` 中根据 type 决定创建哪种 fiber（HostComponent / FunctionComponent / ClassComponent / ...）
