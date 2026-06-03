# Hash 模式与 History 模式

> **主题**：前端路由 / 浏览器原理 / 部署
> **难度**：中级

## 问题

前端路由的 Hash 模式和 History 模式有什么区别？为什么 History 模式需要服务端配合？怎么配合？

## 参考答案

### 一、本质区别

| 维度 | Hash 模式 | History 模式 |
|------|----------|-------------|
| URL 形式 | `https://site.com/#/user/1` | `https://site.com/user/1` |
| 核心 API | `window.location.hash` + `hashchange` 事件 | `history.pushState` / `replaceState` + `popstate` 事件 |
| 是否发请求 | `#` 后内容**不会**发给服务端 | 完整路径**会**发给服务端 |
| 刷新页面 | 永远命中（服务端只看 `#` 前的部分） | 服务端找不到对应文件就 404 |
| SEO | 不友好（爬虫一般忽略 hash） | 友好 |
| 兼容性 | 几乎所有浏览器 | IE10+ |
| 美观度 | 带 `#`，不够干净 | 干净 |

### 二、为什么 History 模式需要服务端配合

SPA 的核心：**只有一个 `index.html`，所有路由由前端 JS 控制**。

**Hash 模式刷新 `/#/user/1`**

1. 浏览器只把 `https://site.com/` 发给服务端
2. 服务端返回 `index.html`
3. 前端 JS 加载，读到 `location.hash` 是 `/user/1`
4. 路由库渲染对应组件 ✅

**History 模式刷新 `/user/1`**

1. 浏览器把完整路径 `https://site.com/user/1` 发给服务端
2. 服务端去找 `/user/1` 这个资源——**不存在** → 返回 404 ❌

> 关键：从首页点链接进 `/user/1` 没问题（`pushState` 改 URL，不发请求）；但**直接访问或刷新**就暴露了问题。

**配合的核心一句话**：服务端对所有未匹配到静态资源的路径，统统返回 `index.html`，让前端路由接管。

### 三、各类服务端配置

#### 1. Nginx（最常见）

```nginx
server {
  listen 80;
  server_name site.com;
  root /var/www/dist;
  index index.html;

  location / {
    # 先找文件，再找目录，都没有就回退到 index.html
    try_files $uri $uri/ /index.html;
  }

  # 静态资源带 hash 文件名，可以长期缓存
  location ~* \.(js|css|png|jpg|svg|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }
}
```

`try_files` 是关键：按顺序尝试，最后兜底 `/index.html`。

#### 2. Apache（`.htaccess`）

```apache
<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

#### 3. Node / Express

```js
const express = require('express')
const path = require('path')
const history = require('connect-history-api-fallback')

const app = express()
app.use(history())                                  // 必须放在 static 之前
app.use(express.static(path.join(__dirname, 'dist')))
app.listen(3000)
```

#### 4. Koa

```js
const Koa = require('koa')
const serve = require('koa-static')
const historyApiFallback = require('koa2-connect-history-api-fallback')

const app = new Koa()
app.use(historyApiFallback({ whiteList: ['/api'] })) // API 路径不回退
app.use(serve('./dist'))
app.listen(3000)
```

#### 5. Webpack Dev Server / Vite

开发环境也要配，否则刷新一样 404。

```js
// webpack.config.js
devServer: {
  historyApiFallback: true
}

// vite 默认就开了
```

#### 6. Nginx + 后端 API（前后端分离）

```nginx
server {
  listen 80;
  root /var/www/dist;

  # API 转发（必须先于兜底规则）
  location /api/ {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }

  # 前端路由兜底
  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

#### 7. CDN / 静态托管

- **Vercel / Netlify**：默认支持，或 `_redirects`：`/* /index.html 200`
- **GitHub Pages**：不支持自定义 fallback，hack 是把 `404.html` 复制成 `index.html` 内容
- **阿里云 OSS / 腾讯云 COS**：静态网站托管设置里把"错误文档"指向 `index.html`

### 四、易错点

#### 1. 404 状态码问题

`try_files ... /index.html` 默认返回 200。但如果路径真的不存在，SEO 角度应返回 404。更严谨的做法是 SSR 或预渲染。

#### 2. API 路径被吞

如果 `try_files` 在 `/` 下，AJAX 请求 `/api/xxx` 找不到也会被改写成 `index.html`，前端拿到 HTML 当 JSON 解析就报错。**API 的 location 必须先于兜底规则**。

#### 3. 子路径部署

部署在 `https://site.com/admin/` 下，三处都要改：

- 前端 `router` 的 `base: '/admin/'`
- 打包工具的 `publicPath` / `base`
- Nginx 的 `try_files $uri $uri/ /admin/index.html`

#### 4. 缓存策略

`index.html` **绝对不能长期缓存**，否则发版用户拿不到新的 chunk hash：

```nginx
location = /index.html {
  add_header Cache-Control "no-cache, no-store, must-revalidate";
}
```

## 追问链

- 为什么 Hash 模式刷新页面不会 404？
- `pushState` 和 `replaceState` 的区别？为什么不会触发刷新？
- `popstate` 什么时候触发？手动 `pushState` 会触发吗？
- 如何在不刷新的前提下修改 URL？
- 服务端兜底到 `index.html` 后，前端是怎么知道要渲染哪个路由的？
- SSR 场景下 History 模式还需要 fallback 吗？
- 多个 SPA 部署在同一域名不同子路径下，路由如何隔离？

## 易错点

- 以为只在生产环境需要配置 fallback，其实开发服务器也要（`historyApiFallback: true`）
- 把 fallback 规则写在 API 路由前，导致接口请求被改写成 HTML
- `index.html` 设置长缓存，发版后用户白屏（旧 HTML 引用了已被删除的 chunk）
- 子路径部署时只改前端 base，没改 Nginx 兜底路径
- 用 GitHub Pages 部署 History 模式应用，刷新永远 404 —— 需要改用 Hash 模式或做 `404.html` hack

## 知识延伸

- HTML5 History API（`pushState` / `replaceState` / `popstate`）的设计初衷
- `hashchange` 事件与 `popstate` 事件的区别
- `try_files` 指令在 Nginx 里的工作机制
- SPA → SSR → SSG → ISR 的演进路径
- 服务端渲染（Next.js / Nuxt）如何同时解决 SEO 和直接访问问题
- 微前端中多个子应用共享一个 History 时的冲突处理
