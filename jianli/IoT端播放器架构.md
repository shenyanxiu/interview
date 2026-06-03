# IoT 端播放器架构（Vela 带屏音箱 X4B）

## 项目概述

播放器运行在Vela系统和Js快应用引擎上。整个Vela系统最底层通过BSP连接硬件与操作系统，然后在Native层有音箱侧的mico服务，提供初始化、小爱唤醒、音频管控、焦点管控、家庭传声等功能，midess通过向云端发送请求可以向iot传输设备信息，然后都通过Service Agent传输到框架层，框架层通过相应的Feature接口对应用进行控制，同时框架层通过XMS、ams、事件监听服务等对多应用进行协调和管理，整个框架如下图所示。而X4B设备仅有128M内存需要运行21个应用，所以分给每个应用的内存是有限的。在内存仅 128M 的 Vela 带屏音箱（X4B）上开发播放器应用，需要控制应用内存在 6M 以内，同时实现帧率和时延达标。支持音乐、有声书、新闻、广播四种类型播放，与手机小爱音箱 App、桌面卡片、控制中心实现信息同步。

## 面试高频问题

### 1. 项目核心难点是什么？

**资源受限环境下的性能优化**：

| 约束 | 具体数值 |
|------|----------|
| 设备总内存 | 128M |
| 应用内存限制 | ≤ 6M |
| JS 引擎与渲染引擎 | 同一线程（不能阻塞 UI） |
| 帧率要求 | 播放列表 48fps，歌词滑动 54fps |

主要挑战：
- 单线程模型下保证 UI 流畅（JS 和渲染引擎共用一个线程）
- 多应用间数据同步（播放器、桌面卡片、控制中心）
- 播放器启动速度优化
- 多种播放类型的数据处理与页面渲染

### 2. 整体架构设计是怎样的？

**单例模式 + 组件化设计**：

- `MusicPlayer` 类使用单例模式，在 `app.ux` 中初始化，所有页面共享同一实例
- 播放列表、歌词页、首页通过单例的 `player` 实例实现状态同步
- 桌面和控制中心抽离为 `vd-music`、`vd-miplay` 业务组件，独立维护

```
app.ux (全局单例 player)
├── pages/player     (播放首页)
├── pages/playlist   (播放列表)
├── pages/lyric      (歌词页)
└── pages/vipGuide   (VIP引导页)
```

### 3. 播放器与底层 audio 如何通信？

通过 `micoaudio` feature 接口（音箱端实现的原生桥接）与底层 audio 服务通信：

```javascript
import audio from "@service.internal.micoaudio"
```

**事件驱动模型**：

| 事件 | 触发场景 | 应用处理 |
|------|----------|----------|
| `onnativeplay` | 小爱语音/App 播放歌曲 | 获取歌单，更新 UI |
| `onnativechange` | 歌单内自动切歌 | 更新当前歌曲信息 |
| `onnativepause` | 底层/App 控制暂停 | 修改按钮状态 |
| `onnativeerror` | 连续 10 首不能播放 | 退出应用 |
| `onnativeended` | 播放完成 | 修改按钮为暂停状态 |
| `onnativetimeupdate` | 播放进度更新 | 歌词滚动同步 |
| `onnativerefresh` | 收藏状态变化 | 重新拉取收藏状态 |

**本质上是发布-订阅模式**，上层应用注册监听，底层 audio 触发事件通知。

### 4. 多应用间如何实现数据同步？

**三个应用**：桌面卡片（launcher）、控制中心（controlcenter）、播放器（music_player）

**同步机制**：

1. 所有应用都注册 `micoaudio.onnativeplay()` 监听
2. 任何一方切歌时调用 `micoaudio.play()` → 底层播放 → 通过事件通知所有注册方
3. 通过 `audio.getPlayState()` 获取当前播放列表，保证列表同步

**冷启动同步方案**（开机推荐歌曲场景）：

- 使用 `exchange` 存储 audioId（键值对存储，限制 255 字符）
- 使用 `channel.notifyMessage` 发布跨应用消息通知
- 控制中心通过 `channel.setTopicListener` 监听变化

> 踩坑：最初尝试将所有歌曲信息存入 exchange，超过了 255 字符限制。改为只存 audioId，收到通知后各自请求详情。

### 5. 时延优化做了哪些？

| 场景 | 优化前 | 优化后 | 优化手段 |
|------|--------|--------|----------|
| 发现页→播放器 | 3195ms | 487ms | 首屏数据直接用第一个接口下发渲染 |
| 播放器→列表页 | 765ms | 405ms | onReady 先渲染3条，onShow 延时300ms渲染全部 |
| 桌面→发现页 | 2895ms | 575ms | 使用占位图代替接口请求 |
| 桌面→播放器 | 1143ms | 531ms | 路由传递歌曲信息，减少网络请求 |

**核心思路**：分阶段渲染 + 减少关键路径网络请求 + 数据预传递

### 6. 帧率优化做了哪些？

从 35fps 提升到 48~54fps：

- **减少重绘开销**：取消 `border-radius` 圆角属性，改用自带圆角的图片
- **使用 `background-image`** 而非 img + border-radius（避免毛边和额外计算）
- **进度条节流防抖**：减少进度条滑动时的页面渲染频率
- **切页优化**：`router.replace` 改为 `router.push`（replace 有性能问题）
- **分阶段渲染**：onReady 先渲染少量数据，延迟渲染剩余部分

### 7. 长列表性能怎么处理的？（虚拟滚动）

场景：有声书收藏列表可能有 2000+ 条数据。

**问题**：一次性渲染导致帧率仅 28fps，内存分配失败。

**解决方案 - 虚拟滚动**：

```javascript
// 核心思路：只渲染可视区域内的固定条数数据
handleScroll(e) {
  const { scrollX } = e;
  // 根据滚动位置计算可视区域的数据起始 index
  let newStart = Math.floor(scrollX / WIDTH) * 2;
  // 更新渲染范围
  this.start = Math.min(this.songlist.length - SHOW_NUM, newStart);
}
```

关键点：
- 只渲染可视区域固定条数（SHOW_NUM）
- 记录每次 start 变化历史，支持回滚计算
- 手停止滑动 2s 后再请求数据，避免边滑边请求阻塞渲染
- 效果：2000+ 图片场景帧率达到 44fps+

### 8. 歌词滚动怎么实现的？

1. **解析 LRC 格式歌词**：将 `[mm:ss.xx]歌词内容` 转为 `{t: ms, c: text}` 数组
2. **监听播放进度**：通过 `audio.onnativetimeupdate` 获取当前播放时间
3. **匹配当前歌词行**：根据时间找到对应歌词索引
4. **自动滚动**：调用 `scrollTo` 滚动到当前歌词位置并高亮

```javascript
audio.onnativetimeupdate = (data) => {
  let { currentTime } = data;
  let lyricNo = this.getCurLyricNo(currentTime);
  if (lyricNo === this.lyricNo) return; // 未变化不处理
  this.lyricNo = lyricNo;
  this.scrollToCurLyric(this.lyricNo);
}
```

### 9. 单线程环境下如何避免阻塞 UI？

Vela 系统中 JS 和渲染引擎是同一线程，策略：

- **异步优先**：所有网络请求使用 async/await
- **分阶段渲染**：利用 `suspendRenderFlush` / `triggerRenderFlush` 批量提交 DOM 操作
- **延迟加载**：非关键数据放 onShow + setTimeout 延迟处理
- **减少 DOM 操作**：用 `display: flex/none` 替代节点增删
- **节流防抖**：进度条、滚动事件加节流

### 10. 内存优化怎么做的？

- **图片云端缩放**：请求指定 size 的图片，而非原图
- **分页加载**：所有列表数据分页请求，避免一次加载全量数据
- **及时释放**：页面销毁时清除定时器、解绑事件、清空全局变量
- **虚拟滚动**：长列表只保留可视区域数据
- **数据类型控制**：网络请求返回类型使用 text 而非 json（避免大数字 ID 精度丢失）

> 踩坑：有声书收藏接口无分页，一次返回 3-4M 数据直接导致内存分配失败 → 推动服务端加分页。

### 11. 遇到过印象深刻的 Bug？

**Bug：编辑页退出后桌面音乐卡片不更新**

- 原因：音乐卡片组件将方法挂载到 `window` 全局对象上，编辑页创建新卡片组件覆盖了 window 上的方法，编辑页销毁后 this 指向已销毁的组件实例
- 本质：全局变量污染 + 组件生命周期管理不当
- 解决：组件销毁时清空 window 上挂载的方法；编辑页卡片改用静态图片

**教训**：减少全局方法挂载，如必须使用，组件/页面销毁时务必清空。

### 12. 收藏状态同步怎么处理延迟问题的？

手机端点击收藏后，服务端需要时间更新状态，立即拉取可能拿不到最新数据。

解决方案：双重轮询
```javascript
audio.onnativerefresh = () => {
  // 1秒后拉取一次
  setTimeout(() => this.updateLoveIcon(), 1000);
  // 3秒后再拉取一次（兜底）
  setTimeout(() => this.updateLoveIcon(), 3000);
}
```

## 可延伸的面试知识点

- **发布-订阅模式**：micoaudio 事件机制的设计模式
- **单例模式**：MusicPlayer 的设计
- **虚拟滚动**：长列表性能优化通用方案
- **跨应用通信**：exchange + channel 的消息机制
- **单线程模型**：类似浏览器的 JS 主线程，如何避免阻塞
- **性能优化方法论**：时延、帧率、内存的量化指标和优化手段
- **IoT 开发与 Web 开发的差异**：资源受限、单线程、原生桥接
