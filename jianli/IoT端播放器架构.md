# IoT 端播放器架构面试题

## Q1：player.js 单例模式是怎么设计的？为什么用单例？

```javascript
let instance = null;

class Player {
  constructor() {
    if (instance) return instance;
    this.playlist = [];
    this.currentIndex = 0;
    this.status = 'idle'; // idle | playing | paused
    instance = this;
  }
  
  static getInstance() {
    if (!instance) instance = new Player();
    return instance;
  }
}
```

用单例的原因：播放器、桌面、控制中心、有声书等多应用都需要读写播放状态，设备端只有一个音频通道，单例保证全局唯一数据源，避免状态不一致。

---

## Q2：多应用状态同步的事件机制具体怎么实现的？

快应用的多应用通信依赖系统级广播机制：

```javascript
class Player {
  constructor() {
    this.listeners = new Map(); // event -> Set<callback>
  }
  
  play(song) {
    this.currentSong = song;
    this.status = 'playing';
    this.emit('stateChange', { song, status: 'playing' });
  }
  
  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event).add(callback);
  }
  
  emit(event, data) {
    const callbacks = this.listeners.get(event);
    if (callbacks) {
      callbacks.forEach(cb => cb(data));
    }
  }
}
```

在 Vela 快应用环境下，跨应用通信：
- **应用内多页面**：单例 + 发布订阅
- **应用间**：系统广播（类似 Android BroadcastReceiver），通过 `system.broadcast` 发送/监听

---

## Q3：WiFi / 蓝牙 / MiPlay 三端统一抽象层怎么设计的？

```javascript
// 抽象播放通道接口
class PlayChannel {
  play(song) {}
  pause() {}
  seek(position) {}
  getProgress() {}
  getCapabilities() {} // 返回当前通道支持的能力
}

class WifiChannel extends PlayChannel {
  play(song) { return audioApi.play(song.url); }
  getCapabilities() { 
    return { canSeek: true, canGetLyric: true, canSwitchQuality: true }; 
  }
}

class BluetoothChannel extends PlayChannel {
  play(song) { return btApi.avrcpPlay(); }
  getCapabilities() { 
    return { canSeek: false, canGetLyric: false, canSwitchQuality: false }; 
  }
}

class MiPlayChannel extends PlayChannel {
  play(song) { return miplayApi.cast(song, targetDevice); }
  getCapabilities() {
    return { canSeek: true, canGetLyric: true, canSwitchQuality: false };
  }
}
```

UI 层根据 `getCapabilities()` 动态渲染：
```javascript
function renderControls(channel) {
  const caps = channel.getCapabilities();
  return {
    seekBar: caps.canSeek ? <SeekBar /> : null,
    qualitySwitch: caps.canSwitchQuality ? <QualityBtn /> : null,
    lyric: caps.canGetLyric ? <LyricView /> : null,
  };
}
```

切换通道时：
1. 暂停当前通道
2. 保存播放进度
3. 切换到新通道实例
4. 恢复播放（如果新通道支持 seek，从上次进度继续）

---

## Q4：列表滑动优化怎么做的？

核心思路类似虚拟列表：
- 只创建可视区域内的 DOM 节点（如屏幕内显示 5 个 item，实际只创建 5+2 个缓冲节点）
- 滚动时回收离屏节点、复用给新进入可视区的 item（类似 RecyclerView 的机制）
- 在 IoT 设备上内存极其有限（通常 < 64MB），不能像 Web 端那样放任 DOM 增长
- 配合图片懒加载，只在节点进入可视区时才发起图片请求

---

## Q5：IoT 设备上的虚拟列表和 Web 端有什么区别？

| 维度 | Web 端虚拟列表 | IoT 端（Vela 快应用） |
|------|--------------|---------------------|
| 内存 | 浏览器 GB 级 | 设备 32-64MB 总共 |
| DOM 引擎 | 完整 DOM + CSS | 轻量渲染引擎（非浏览器） |
| 滚动事件 | 高频触发，需 throttle | 帧率低，事件本身就不密集 |
| 图片 | 浏览器自动管理缓存 | 需手动管理 imageCache 容量 |
| 回收策略 | 移出可视区就移除 | 更激进，可能只保留 3-5 个节点 |

IoT 端额外要处理的：
- **imageCache 上限**：设备图片缓存有固定大小，超出会 crash，需主动调用清理接口
- **GC 压力**：频繁创建/销毁对象会触发 GC 导致卡顿，尽量复用节点（对象池模式）
- **帧率目标不同**：Web 追求 60fps，IoT 设备能稳定 30fps 就很好了

封装公共请求层并推动图片压缩接口降低 imageCache 占用——后端提供多分辨率图片，设备端请求小尺寸版本。

---

## Q6：快应用和 Web 前端的核心区别是什么？

| 维度 | Web 前端 | Vela 快应用 |
|------|---------|------------|
| 运行环境 | 浏览器（V8 + DOM） | 轻量 JS 引擎 + 自绘渲染 |
| 布局 | CSS 全集 | CSS 子集（Flexbox 为主） |
| 组件 | HTML 标准元素 | 自定义组件（类似小程序） |
| 内存 | GB 级 | 几十 MB |
| 调试 | Chrome DevTools | 串口日志 + 自研工具 |
| 生命周期 | 页面级 | 应用级 + 页面级 |
| 网络 | fetch/XHR | 系统 API 封装 |
