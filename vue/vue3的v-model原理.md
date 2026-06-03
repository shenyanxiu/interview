# Vue 3 的 v-model 原理

> 主题：Vue ｜ 难度：中级

## 问题

Vue 3 的 `v-model` 是如何实现的？与 Vue 2 相比有哪些变化？

## 参考答案

`v-model` 是 Vue 提供的语法糖，本质是 **prop 传值 + event 监听** 的组合。Vue 3 在原生元素和组件两个层面都做了调整。

### 一、原生元素上的 v-model

编译器会根据元素类型生成不同代码。

```html
<input v-model="msg" />
```

会被编译为：

```js
<input
  :value="msg"
  @input="msg = $event.target.value"
/>
```

不同元素的处理：
- `text` / `textarea`：监听 `input` 事件，绑定 `value`
- `checkbox`：监听 `change` 事件，绑定 `checked`
- `radio`：监听 `change` 事件，绑定 `checked`
- `select`：监听 `change` 事件，绑定 `value`

修饰符处理：
- `.lazy`：把 `input` 改为 `change`
- `.number`：用 `parseFloat` 转换
- `.trim`：调用 `String.prototype.trim`

### 二、组件上的 v-model（与 Vue 2 的关键区别）

**Vue 2**：默认 prop 是 `value`，事件是 `input`，一个组件只能用一个 `v-model`，多绑定要用 `.sync` 修饰符。

**Vue 3**：默认 prop 改为 `modelValue`，事件改为 `update:modelValue`，支持多个 `v-model` 绑定，`.sync` 被移除。

```html
<MyInput v-model="msg" />
```

等价于：

```html
<MyInput
  :modelValue="msg"
  @update:modelValue="newVal => msg = newVal"
/>
```

子组件实现：

```js
// 选项式
export default {
  props: ['modelValue'],
  emits: ['update:modelValue'],
}

// 组合式 + <script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
emit('update:modelValue', newVal)
```

### 三、多 v-model 绑定

Vue 3 允许一个组件绑定多个 `v-model`，通过参数区分：

```html
<UserForm
  v-model:name="userName"
  v-model:age="userAge"
/>
```

编译为：

```html
<UserForm
  :name="userName"
  @update:name="val => userName = val"
  :age="userAge"
  @update:age="val => userAge = val"
/>
```

子组件中 prop 名就是参数名，事件名是 `update:参数名`。

### 四、自定义修饰符

Vue 3 支持自定义 v-model 修饰符。修饰符通过 `modelModifiers` prop 传给子组件。

```html
<MyInput v-model.capitalize="msg" />
```

子组件：

```js
const props = defineProps({
  modelValue: String,
  modelModifiers: { default: () => ({}) }
})
const emit = defineEmits(['update:modelValue'])

function onInput(e) {
  let val = e.target.value
  if (props.modelModifiers.capitalize) {
    val = val.charAt(0).toUpperCase() + val.slice(1)
  }
  emit('update:modelValue', val)
}
```

带参数时，修饰符 prop 名是 `xxxModifiers`，例如 `v-model:title.capitalize` 对应 `titleModifiers`。

### 五、底层编译流程

1. 模板编译阶段，`compiler-core` 中的 `transformModel` 把 `v-model` 转成 `modelValue` 绑定 + `onUpdate:modelValue` 事件。
2. 原生标签由 `compiler-dom` 的 `transformModel` 接管，根据 tag 选择对应运行时指令（`vModelText`、`vModelCheckbox` 等）。
3. 运行时阶段，`patchProp` 把 `onUpdate:modelValue` 绑成事件监听，`modelValue` 当作普通 prop 传入。

### 六、Vue 2 vs Vue 3 对比

| 维度 | Vue 2 | Vue 3 |
|------|-------|-------|
| 默认 prop | `value` | `modelValue` |
| 默认事件 | `input` | `update:modelValue` |
| 多绑定 | 用 `.sync` | 直接多个 `v-model:xxx` |
| 自定义修饰符 | 不支持 | 支持，通过 `modelModifiers` |
| 自定义 prop/event | `model: { prop, event }` 选项 | 直接用 `v-model:xxx` 参数 |

## 追问链

- 为什么 Vue 3 要把默认 prop 名从 `value` 改为 `modelValue`？
- `.sync` 修饰符为什么被移除？多个 `v-model` 怎么替代它？
- 自定义组件中如何同时支持 `v-model` 和受控/非受控两种模式？
- `v-model` 在 `<input type="checkbox">` 多选场景下绑定数组的内部实现？
- 如何在组合式 API 中封装一个可复用的 `useVModel`？
- IME（中文输入法）输入时 `v-model` 为什么默认不更新？编译时做了哪些处理？

## 易错点

- 误以为 Vue 3 中 `v-model` 默认 prop 仍然是 `value`，导致组件不响应。
- 子组件忘记声明 `emits: ['update:modelValue']`，事件能触发但会有警告，且 attrs 透传会异常。
- 在 `<script setup>` 中直接修改 `props.modelValue`，触发只读警告（props 是只读的）。
- 多 `v-model` 时 prop 名拼错或事件名 `update:xxx` 大小写不匹配，导致单向数据流。
- 自定义修饰符 prop 默认值忘记写 `() => ({})`，访问 `modelModifiers.xxx` 时报 undefined。
- 中文输入法场景未处理 `compositionstart` / `compositionend`，造成输入抖动。

## 知识延伸

- 单向数据流与受控组件思想：React 的 `value + onChange` 与 Vue `v-model` 的本质一致。
- `defineModel`（Vue 3.4+）：进一步简化，子组件可直接 `const model = defineModel()`，免去手动声明 props/emits。
- 与 `.sync` 修饰符在 Vue 2 的演化关系。
- 编译产物层面理解模板编译：`@vue/compiler-sfc` → `compiler-dom` → `compiler-core`。
- 响应式系统（`ref` / `reactive`）如何与 `v-model` 配合驱动视图更新。
