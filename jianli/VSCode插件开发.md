# VSCode 插件开发（Vela IDE）面试题

## Q1：Extension Host 的运行机制是什么？插件代码跑在哪里？

VSCode 采用多进程架构：
- **主进程（Main Process）**：负责窗口管理、菜单、生命周期
- **渲染进程（Renderer）**：负责 UI 绘制（Electron 的 BrowserWindow）
- **Extension Host（扩展宿主进程）**：**独立的 Node.js 进程**，所有插件代码都跑在这里

@vs-vela/device 子包运行在 Extension Host 中。

这种设计的好处：
- 插件崩溃不会影响编辑器主 UI
- 插件之间共享同一进程，可以互相调用 API
- 但也意味着一个插件死循环会阻塞所有插件

WebView 面板则运行在**独立的 iframe 沙箱**中，与 Extension Host 通过 postMessage 异步通信。

---

## Q2：插件与 WebView 之间是怎么通信的？

通信机制：
```typescript
// Extension 侧 → WebView
panel.webview.postMessage({ command: 'updateProgress', data: { percent: 50 } });

// WebView 侧 → Extension
const vscode = acquireVsCodeApi();
vscode.postMessage({ command: 'startFlash', data: { file: '/path/to/firmware.bin' } });

// Extension 侧监听
panel.webview.onDidReceiveMessage(message => {
  switch (message.command) {
    case 'startFlash': handleFlash(message.data); break;
  }
});
```

本质是**基于 postMessage 的异步消息机制**，类似 iframe 通信。需要注意：
- WebView 运行在沙箱中，无法直接访问 Node.js API
- 需要自行设计消息协议（command + data 结构）
- 大数据传输要注意序列化性能

---

## Q3：WebView 的安全限制有哪些？你怎么处理的？

WebView 安全限制：
1. **无法直接访问 Node.js API**（没有 require、fs、path）
2. **CSP 内容安全策略**严格，默认不能加载外部资源
3. **无法直接访问 vscode API**，只能通过 `acquireVsCodeApi()` 拿到有限的消息通道

处理方式：
```typescript
// 设置 WebView 选项
panel.webview.options = {
  enableScripts: true,
  localResourceRoots: [
    vscode.Uri.joinPath(context.extensionUri, 'media')
  ]
};

// 本地资源需要转换为 webview 专用 URI
const scriptUri = panel.webview.asWebviewUri(
  vscode.Uri.joinPath(context.extensionUri, 'media', 'main.js')
);
```

对于 Vue 3 构建产物：
- 打包时 base 设为相对路径
- 所有静态资源通过 `asWebviewUri` 转换后注入 HTML
- CSP 中配置 nonce 允许内联脚本执行

---

## Q4：终端-进度条联动的生命周期管理具体指什么？

"联动"指的是：Terminal 执行烧录命令 → 解析输出 → 推送给 WebView 渲染进度条。这条链路的生命周期问题：

```typescript
class FlashSession {
  private terminal: vscode.Terminal | undefined;
  private panel: vscode.WebviewPanel | undefined;
  private disposed = false;

  async start() {
    this.terminal = vscode.window.createTerminal('Flash');
    this.terminal.sendCommand(flashCommand);
    
    // 问题1：用户手动关闭了终端
    vscode.window.onDidCloseTerminal(t => {
      if (t === this.terminal && !this.disposed) {
        this.handleUnexpectedClose();
      }
    });
    
    // 问题2：用户关闭了 WebView 面板
    this.panel.onDidDispose(() => {
      // 终端还在跑，但 UI 没了
      // 策略：继续烧录，不中断，用户重新打开面板时恢复进度
    });
    
    // 问题3：烧录超时
    this.timeout = setTimeout(() => {
      this.terminal?.dispose();
      this.notifyError('烧录超时');
    }, 120000);
  }
  
  dispose() {
    this.disposed = true;
    clearTimeout(this.timeout);
    this.terminal?.dispose();
    this.panel?.dispose();
  }
}
```

核心要处理的场景：

| 场景 | 处理策略 |
|------|---------|
| 终端被手动关闭 | 通知 WebView 显示异常状态 |
| WebView 被关闭 | 不中断烧录，保存进度，重新打开时恢复 |
| 烧录超时 | 强制 kill 终端，提示重试 |
| 插件被 deactivate | 清理所有资源 |
| 设备断开连接 | 检测串口状态，中止并提示 |

---

## Q5：烧录进度实时跟踪是怎么实现的？

实现方案：
1. 通过 `vscode.window.createTerminal` 创建终端执行烧录命令
2. 监听终端输出流，正则解析进度信息（如 `[====  ] 45%`）
3. 解析后通过 postMessage 推送给 WebView 面板渲染进度条
4. 生命周期管理：终端意外关闭/烧录超时/用户取消等异常状态处理

难点在于终端输出是字符流，需要做缓冲和行解析，且要处理 ANSI 转义序列。

---

## Q6：你怎么实现设备自动发现的？

```typescript
import { SerialPort } from 'serialport';

class DeviceDiscovery {
  private knownDevices = new Map<string, DeviceInfo>();
  private pollInterval: NodeJS.Timer;

  startPolling(interval = 2000) {
    this.pollInterval = setInterval(async () => {
      const ports = await SerialPort.list();
      
      // 过滤出 Vela 设备（通过 vendorId / productId 识别）
      const velaDevices = ports.filter(p => 
        VELA_VENDOR_IDS.includes(p.vendorId)
      );
      
      // diff 检测新增/移除
      const current = new Set(velaDevices.map(d => d.path));
      const previous = new Set(this.knownDevices.keys());
      
      // 新设备接入
      for (const path of current) {
        if (!previous.has(path)) {
          this.emit('deviceConnected', velaDevices.find(d => d.path === path));
        }
      }
      
      // 设备移除
      for (const path of previous) {
        if (!current.has(path)) {
          this.emit('deviceDisconnected', path);
        }
      }
    }, interval);
  }
}
```

- 定时轮询串口列表（2s 间隔）
- 通过 USB Vendor ID / Product ID 识别是否为 Vela 设备
- 检测到新设备时通知 UI 更新设备列表
- 跨平台：Windows 用 `COM*`，Linux 用 `/dev/ttyUSB*` 或 `/dev/ttyACM*`

---

## Q7：跨平台路径兼容怎么做的？

- Windows 使用 `\` 分隔符，Linux/Mac 使用 `/`
- 统一使用 Node.js `path` 模块（`path.join`、`path.resolve`）处理路径拼接
- 串口设备路径差异：Windows 是 `COM3`，Linux 是 `/dev/ttyUSB0`，Mac 是 `/dev/cu.usbserial`
- 通过 `process.platform` 判断平台，对设备发现逻辑做分支处理
- 配置文件中存储相对路径，运行时动态解析为绝对路径
