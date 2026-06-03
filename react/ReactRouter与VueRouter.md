# React Router 与 Vue Router

> **主题**：前端路由 / 框架对比
> **难度**：中级

## 问题

React Router 和 Vue Router 有什么区别？

## 参考答案

两者都是 SPA 路由方案，核心目标一致：**根据 URL 渲染对应组件，管理导航状态**。但设计哲学差异明显。

### 1. 核心定位

| 维度 | React Router | Vue Router |
|------|-------------|-----------|
| 维护方 | 第三方（Remix 团队），非 React 官方 | Vue 官方维护 |
| 设计风格 | 路由即组件（JSX 声明） | 路由即配置（routes 对象数组） |
| 与框架耦合 | 松耦合，更"库"化 | 紧耦合，更"框架"化 |
| 主流版本 | v6 / v7（Data API） | v4（配 Vue 3） |

### 2. 路由配置写法

**React Router（JSX 声明式）**

```jsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/user/:id" element={<User />}>
      <Route path="profile" element={<Profile />} />
    </Route>
  </Routes>
</BrowserRouter>
```

**Vue Router（对象配置式）**

```js
const routes = [
  { path: '/', component: Home },
  {
    path: '/user/:id',
    component: User,
    children: [{ path: 'profile', component: Profile }]
  }
]
const router = createRouter({ history: createWebHistory(), routes })
```

React 把路由当组件树写，配置即 UI；Vue 把路由当数据结构，配置和渲染分离。

### 3. 导航 API

| 操作 | React Router | Vue Router |
|------|-------------|-----------|
| 编程式跳转 | `useNavigate()` → `navigate('/x')` | `useRouter()` → `router.push('/x')` |
| 链接 | `<Link>` / `<NavLink>` | `<router-link>` |
| 出口 | `<Outlet />` | `<router-view />` |
| 参数 | `useParams()` / `useSearchParams()` | `useRoute().params` / `.query` |

### 4. 导航守卫（最大区别）

**Vue Router** 提供完整的守卫体系，覆盖全局、路由、组件三个层级：

```js
router.beforeEach((to, from) => { /* 全局前置 */ })
router.afterEach(() => {})
// 组件内
beforeRouteEnter / beforeRouteUpdate / beforeRouteLeave
```

**React Router** 没有内置守卫概念：

- v5 时代用包裹组件 `<PrivateRoute>` 做权限拦截
- v6+ 推荐用 `loader` + `redirect` 在数据加载阶段拦截
- 离开拦截用 `useBlocker` / `usePrompt`

```jsx
{
  path: '/admin',
  loader: async () => {
    if (!isAuth()) throw redirect('/login')
    return null
  },
  element: <Admin />
}
```

### 5. 数据加载

**React Router v6.4+** 引入 Remix 风格的 Data API：

- `loader`：进入路由前并行加载数据
- `action`：处理表单提交
- `useLoaderData()`：组件中拿数据
- 支持 `defer` 流式渲染、`errorElement` 错误边界

**Vue Router** 没有内置数据加载，通常配合 Pinia 在 `setup` 或 `beforeRouteEnter` 中取数。Nuxt 才提供 `useAsyncData` 等能力。

### 6. 路由模式

两者都支持：

- History 模式（`createBrowserRouter` / `createWebHistory`）
- Hash 模式（`createHashRouter` / `createWebHashHistory`）
- Memory 模式（SSR / 测试）

### 7. 懒加载

```js
// Vue
{ path: '/x', component: () => import('./X.vue') }
```

```jsx
// React
const X = lazy(() => import('./X'))
<Route path="/x" element={<Suspense><X /></Suspense>} />
```

### 8. 动态路由匹配

| 能力 | React Router | Vue Router |
|------|-------------|-----------|
| 参数 | `/user/:id` | `/user/:id` |
| 通配 | `*` | `/:pathMatch(.*)*` |
| 可选参数 | 用多条 Route | `/user/:id?` |
| 正则约束 | 不支持 | 原生支持 `/:id(\\d+)` |

Vue Router 的路径匹配能力更强。

### 9. 设计哲学小结

- **React Router**：路由即 UI，逻辑跟着 JSX 走；v6.4 后又往"数据路由"演进，越来越像微型框架。
- **Vue Router**：配置驱动，路由表是独立数据结构，守卫、`meta`、滚动行为都在配置层解决，更像传统路由器。

## 追问链

- React Router v5 → v6 有哪些破坏性变化？
- Vue Router 的 `meta` 字段一般用来做什么？
- React 中如何实现类似 Vue `<keep-alive>` 的页面缓存？
- React Router v6.4 的 Data API 解决了什么问题？
- 如何在 Vue Router 中做权限路由？

## 易错点

- 把 React Router 当成 React 官方库（实际是 Remix 团队维护）
- 在 React Router v6 里继续用 `Switch` / `useHistory`（已被 `Routes` / `useNavigate` 替代）
- 用 React Router 做权限控制时只在组件里判断，忽略了直接访问 URL 的场景
- 误认为 Vue Router 的导航守卫能完全阻塞导航——`next(false)` 才会取消，新版直接 `return false`
- 在 React Router 中嵌套路由忘记加 `<Outlet />`，子路由不渲染

## 知识延伸

- Hash 模式与 History 模式的底层差异（详见 `Hash与History模式.md`）
- Remix / Next.js App Router 的文件系统路由
- Nuxt 的自动路由生成
- 微前端中的路由协调（qiankun、wujie 等）
- React Router v7（合并 Remix 后的新版本）
