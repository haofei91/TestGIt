# claude-in-mobile 源码解读

> 仓库地址：https://github.com/AlexGladkov/claude-in-mobile
> 来源：AlexGladkov
> 语言：TypeScript (Node.js) | 总文件数：75+ 个源码文件 | 代码量：~8000+ 行
> 依赖：@modelcontextprotocol/sdk, chrome-launcher, chrome-remote-interface, jimp, google-auth-library

---

## 一、项目定位

claude-in-mobile 是目前分析的所有项目中**规模最大、功能最全**的 MCP Server，定位为**全平台移动/桌面自动化工具**。它支持 5 个平台，提供约 40 个工具，拥有 130+ 别名映射，并内置了 Flow 引擎、UI Diff、应用商店管理等高级特性。

核心特点：
- **5 平台支持**：Android (ADB)、iOS (simctl + WDA)、Desktop (Compose Multiplatform)、Aurora OS (audb)、Browser (CDP)
- **~40 个 MCP Tool**：覆盖交互、UI 分析、截图、流程自动化、应用管理、商店上传等
- **130+ 别名**：解决 LLM 工具调用时的命名偏差问题
- **Flow 引擎**：支持条件逻辑、循环、错误处理的服务端多步骤自动化
- **智能 UI 分析**：模糊匹配、元素缓存、屏幕分析、UI Diff、Action Hints
- **坐标缩放**：自动修正压缩截图与实际设备坐标的差异
- **应用商店管理**：Google Play、Huawei AppGallery、RuStore 上传集成

---

## 二、整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                          MCP 层                                      │
│  src/index.ts                                                        │
│  ├── MCP Server 定义 + 16 组工具注册                                  │
│  ├── 130+ 别名映射 (registerAliases)                                  │
│  ├── 客户端检测与适配                                                  │
│  └── 优雅退出 + 清理                                                  │
├──────────────────────────────────────────────────────────────────────┤
│                        工具层 (src/tools/)                            │
│  interaction-tools.ts   交互工具 (tap/swipe/text/key)                 │
│  ui-tools.ts            UI 分析工具 (tree/find/wait/assert)           │
│  screenshot-tools.ts    截图工具 (capture/annotate)                    │
│  flow-tools.ts          流程引擎 (batch/run)                          │
│  context.ts             工具上下文 (缓存/缩放/hints)                   │
│  registry.ts            工具注册表 + 别名系统                          │
├──────────────────────────────────────────────────────────────────────┤
│                      适配器层 (src/adapters/)                         │
│  platform-adapter.ts    平台适配器接口 (PlatformAdapter)               │
│  android-adapter.ts     Android 适配器 → AdbClient                    │
│  ios-adapter.ts         iOS 适配器 → IosClient                        │
│  desktop-adapter.ts     Desktop 适配器 → Compose Accessibility        │
│  aurora-adapter.ts      Aurora OS 适配器                               │
│  browser-adapter.ts     Browser 适配器 → CDP                          │
├──────────────────────────────────────────────────────────────────────┤
│                      平台客户端层                                      │
│  src/adb/client.ts      ADB 客户端 (execSync, 30+ 方法)               │
│  src/adb/ui-parser.ts   Android UI XML 解析 + 模糊匹配                │
│  src/ios/client.ts      iOS 客户端 (simctl + WDA)                     │
│  src/store/store-client.ts  应用商店客户端 (Play/Huawei/RuStore)       │
├──────────────────────────────────────────────────────────────────────┤
│                      设备管理层                                       │
│  src/device-manager.ts  设备路由 + 平台分发 (~440行)                   │
│  ├── 5 平台适配器实例管理                                              │
│  ├── 自动设备检测                                                      │
│  └── FIX #8: Server 重启时设备自动重连                                 │
├──────────────────────────────────────────────────────────────────────┤
│                      底层                                             │
│  ADB (adb shell)        Android 设备通信                               │
│  simctl + WDA           iOS 模拟器通信                                 │
│  CDP                    浏览器 DevTools 协议                           │
│  Compose Accessibility  桌面应用无障碍 API                              │
└──────────────────────────────────────────────────────────────────────┘
```

### 数据流

```
MCP Client (Claude/Cursor)
    ↓ MCP 协议
MCP Server (src/index.ts)
    ↓ 别名解析 → 工具路由
工具层 (src/tools/*.ts)
    ↓ ToolContext (缓存/缩放)
DeviceManager (src/device-manager.ts)
    ↓ 平台路由
PlatformAdapter (android/ios/desktop/aurora/browser)
    ↓ 原生命令
设备/模拟器/浏览器
```

---

## 三、核心模块详解

### 3.1 MCP 入口 (`src/index.ts`)

#### 工具注册

16 组工具模块注册，每组包含多个工具：

```typescript
// 注册工具组
registerInteractionTools(server, ctx);
registerUiTools(server, ctx);
registerScreenshotTools(server, ctx);
registerFlowTools(server, ctx);
registerAppTools(server, ctx);
registerDeviceTools(server, ctx);
// ... 更多工具组
```

#### 别名系统 (130+)

解决 LLM 调用工具时命名偏差的问题：

```typescript
registerAliases({
  "tap": "input_tap",
  "click": "input_tap",
  "long_tap": "input_long_press",
  "screenshot": "screen_capture",
  "get_ui": "ui_tree",
  "find_element": "ui_find",
  "swipe_up": "input_swipe",
  "type_text": "input_text",
  // ... 120+ 更多别名
});
```

这是所有分析项目中独有的设计——其他项目都假设 LLM 会精确调用工具名。

#### 客户端检测

MCP 握手后检测客户端类型，进行适配：

```typescript
// 检测客户端 (Claude Desktop, Cursor, etc.)
// 根据客户端能力调整工具行为
```

### 3.2 工具注册表 (`src/tools/registry.ts`)

- 统一的工具注册 + 别名解析
- 工具调用时先查别名映射，找到规范名后路由到实际实现
- 支持工具分组，便于批量注册

### 3.3 交互工具 (`src/tools/interaction-tools.ts`)

6 个核心交互工具：

| 工具 | 说明 | 特殊能力 |
|------|------|----------|
| `input_tap` | 点击 | 支持坐标/text/resourceId/index |
| `input_double_tap` | 双击 | 坐标 |
| `input_long_press` | 长按 | 坐标 + 时长 |
| `input_swipe` | 滑动 | 坐标 + 方向 |
| `input_text` | 输入文本 | 支持追加/替换 |
| `input_key` | 按键 | 30+ 按键支持 |

#### 智能元素查找 (input_tap)

`input_tap` 支持多种元素定位方式：

```typescript
// 方式 1: 坐标点击
input_tap({x: 540, y: 960})

// 方式 2: 按文本查找 (Android)
input_tap({text: "Settings"})

// 方式 3: 按 resourceId 查找 (Android)
input_tap({resourceId: "com.example:id/btn"})

// 方式 4: 按 index 查找 (使用缓存元素)
input_tap({index: 5})

// 方式 5: 按 label/text 查找 (iOS)
input_tap({label: "Settings"})
```

#### 元素缓存

避免每次 tap 都重新解析 XML：

```typescript
if (args.index !== undefined && currentPlatform === "android") {
  let elements = ctx.getCachedElements("android");
  if (elements.length === 0) {
    const xml = await ctx.deviceManager.getUiHierarchyAsync("android");
    elements = parseUiHierarchy(xml);
    ctx.setCachedElements("android", elements);
  }
  const el = elements.find(e => e.index === idx);
  x = el.centerX; y = el.centerY;
}
```

#### 坐标缩放

截图可能被压缩传给 LLM，LLM 返回的坐标需要缩放回设备分辨率：

```typescript
// applyScale(): 从 screenshotScaleMap 获取缩放因子
// 将 LLM 感知到的坐标转换为设备实际坐标
const { x: deviceX, y: deviceY } = applyScale(x, y, platform);
```

#### Action Hints

操作后自动检测 UI 变化并反馈：

```typescript
// hints 参数: 操作后执行 UI diff
// 150ms 延迟 → 重新获取 UI → 与操作前对比 → 生成变化描述
```

### 3.4 UI 工具 (`src/tools/ui-tools.ts`)

7 个 UI 分析工具：

| 工具 | 说明 |
|------|------|
| `ui_tree` | 获取 UI 树结构 (解析 XML → 可读格式) |
| `ui_find` | 搜索元素 (多条件筛选) |
| `ui_find_tap` | 模糊匹配 → 自动点击 (置信度评分) |
| `ui_tap_text` | Desktop/macOS 按文本点击 |
| `ui_analyze` | 结构化屏幕分析 (标题/对话框/导航/按钮) |
| `ui_wait` | 轮询等待元素出现 |
| `ui_assert_visible` / `ui_assert_gone` | 元素可见性断言 |

#### 模糊匹配 (`ui_find_tap`)

通过置信度评分找到最匹配的元素：

```typescript
// findBestMatch() 评分规则:
// exact text match  = 100
// content-desc      = 95
// text contains     = 80
// desc contains     = 75
// resourceId match  = 60
// partial text      = 40
// partial desc      = 35
// clickable 加分    = +10

// minConfidence 阈值 (默认 30)
ui_find_tap({query: "登录", minConfidence: 50})
```

#### 屏幕分析 (`ui_analyze`)

返回结构化的屏幕信息：

```typescript
interface ScreenAnalysis {
  screenTitle: string;       // 标题检测 (Toolbar/ActionBar)
  hasDialog: boolean;        // 对话框检测 (面积 30-85%)
  navigationState: object;   // 导航状态 (返回键/菜单/标签)
  buttons: Element[];        // 按钮列表
  inputs: Element[];         // 输入框列表
  texts: Element[];          // 文本列表
  scrollable: Element[];     // 可滚动区域
}
```

#### 等待与断言

```typescript
// ui_wait: 轮询等待
ui_wait({text: "加载完成", timeout: 5000, interval: 500})

// ui_assert_visible: 断言可见 (不截图)
ui_assert_visible({text: "欢迎"})  // → pass/fail

// ui_assert_gone: 断言消失
ui_assert_gone({text: "加载中"})   // → pass/fail
```

### 3.5 截图工具 (`src/tools/screenshot-tools.ts`)

2 个工具：

| 工具 | 说明 |
|------|------|
| `screen_capture` | 截图 (支持 diff 模式) |
| `screen_annotate` | 标注截图 (绿色可点击/红色不可点击) |

#### 稳定截图

```typescript
// waitForStableScreenshot():
// 3 次重试, 300ms 间隔
// < 2% 变化 → 认为画面稳定
```

#### Diff 模式

智能决定返回内容：

```
< 5%  变化 → 仅返回文字描述 (无截图)
5-80% 变化 → 返回裁剪后的变化区域
> 80% 变化 → 返回完整截图
```

#### 自动压缩 + 缩放追踪

截图压缩后记录缩放因子，后续坐标操作自动修正：

```typescript
// 截图时记录: screenshotScaleMap[platform] = actualWidth / compressedWidth
// 点击时修正: x = x * scaleMap[platform], y = y * scaleMap[platform]
```

### 3.6 Flow 引擎 (`src/tools/flow-tools.ts`)

2 个工具实现服务端多步骤自动化：

#### `flow_batch` — 批量命令

单次 MCP 往返执行多个命令 (最多 50 个)：

```typescript
flow_batch({
  commands: [
    {tool: "input_tap", args: {x: 100, y: 200}},
    {tool: "input_text", args: {text: "hello"}},
    {tool: "screen_capture", args: {}}
  ]
})
```

#### `flow_run` — 条件流程

支持条件逻辑、循环、错误处理：

```typescript
flow_run({
  steps: [
    {
      action: "input_tap",
      args: {text: "登录"},
      if_not_found: "scroll_down",    // 找不到时下滑
      on_error: "skip"                // 出错时跳过
    },
    {
      action: "input_text",
      args: {text: "password123"},
      repeat: {times: 3, until_found: "欢迎"}  // 重复直到找到
    }
  ]
})
```

约束条件：
- 最多 20 步
- 60s 超时
- 最多重复 10 次
- 仅允许 15 种白名单操作

### 3.7 ADB 客户端 (`src/adb/client.ts`)

基于 `execSync` 执行 ADB 命令，30+ 方法：

```typescript
class AdbClient {
  // 交互
  tap(x, y)              // input tap
  doubleTap(x, y)        // 两次 tap (100ms 间隔)
  longPress(x, y, ms)    // input swipe (同坐标)
  swipe(x1,y1,x2,y2,ms) // input swipe
  inputText(text)        // input text (特殊字符 broadcast)
  pressKey(key)          // input keyevent

  // 截图
  screenshotAsync()      // screencap → PNG bytes

  // UI
  getUiHierarchy()       // uiautomator dump

  // 应用管理
  launchApp(pkg)         // monkey -c LAUNCHER
  stopApp(pkg)           // am force-stop
  installApp(path)       // pm install
  getCurrentActivity()   // 6 种正则匹配不同 Android 版本

  // 剪贴板
  selectAll()            // KEYCODE_A + META_CTRL
  copyToClipboard()      // KEYCODE_C + META_CTRL
  pasteFromClipboard()   // KEYCODE_V + META_CTRL
  getClipboardText()     // cmd clipboard get-primary-clip

  // 权限
  grantPermission(pkg, perm)   // pm grant
  revokePermission(pkg, perm)  // pm revoke

  // 系统信息
  getSystemInfo()        // wm size + dumpsys battery + getprop
}
```

超时配置：
- 默认命令：15s
- 截图/安装：30s

按键映射 (30+)：
```
BACK, HOME, APP_SWITCH, ENTER, DELETE, TAB, ESCAPE,
VOLUME_UP, VOLUME_DOWN, POWER, CAMERA,
MEDIA_PLAY_PAUSE, MEDIA_NEXT, MEDIA_PREVIOUS, ...
```

### 3.8 UI XML 解析器 (`src/adb/ui-parser.ts`)

#### 元素结构

```typescript
interface UiElement {
  index: number;
  resourceId: string;
  className: string;
  text: string;
  contentDesc: string;
  clickable: boolean;
  enabled: boolean;
  visible: boolean;
  bounds: {left, top, right, bottom};
  centerX: number;
  centerY: number;
  // ... 共 19 个字段
}
```

#### 解析方式

使用正则而非 DOM 解析 XML：

```typescript
function parseUiHierarchy(xml: string): UiElement[] {
  // 正则提取 bounds: [left,top][right,bottom]
  // 正则提取各属性
  // 计算 centerX, centerY
}
```

#### 多维查找

```typescript
function findElements(elements, criteria): UiElement[] {
  // 多条件筛选:
  // text, resourceId, className, clickable, enabled, visible
}
```

#### 模糊匹配评分

```typescript
function findBestMatch(elements, query): {element, confidence} {
  // exact text    → 100
  // content-desc  → 95
  // text contains → 80
  // desc contains → 75
  // resourceId    → 60
  // partial text  → 40
  // partial desc  → 35
  // clickable     → +10 bonus
}
```

#### 屏幕分析

```typescript
function analyzeScreen(elements): ScreenAnalysis {
  detectScreenTitle()   // Toolbar/ActionBar/NavigationBar 中的文本
  detectDialog()        // class 包含 Dialog + 面积占比 30-85%
  detectNavigation()    // 返回键、菜单、标签栏检测
  // 分类: buttons, inputs, texts, scrollable
}
```

#### UI Diff

```typescript
function diffUiElements(before, after): UiDiff {
  // 比较前后 UI 状态
  // > 60% 变化 → "screen changed"
  // 返回新增/移除/修改的元素列表
}

function suggestNextActions(elements): string[] {
  // 检测空输入框 → "填写输入"
  // 检测对话框按钮 → "点击确认/取消"
  // 检测可点击元素 → "点击 XXX"
  // 检测可滚动区域 → "可以滚动查看更多"
}
```

### 3.9 工具上下文 (`src/tools/context.ts`)

```typescript
interface ToolContext {
  deviceManager: DeviceManager;

  // 元素缓存
  getCachedElements(platform): UiElement[];
  setCachedElements(platform, elements): void;

  // 截图缩放
  screenshotScaleMap: Map<string, number>;

  // Action Hints
  generateActionHints(): Promise<string>;  // 150ms → re-fetch → diff → suggestions
}
```

iOS UI 树转换：

```typescript
function iosTreeToUiElements(tree): UiElement[] {
  // 将 iOS WDA 无障碍树转换为统一的 UiElement[] 格式
}

function formatIOSUITree(tree): string {
  // 递归格式化为可读树
}
```

嵌套深度限制：`MAX_RECURSION_DEPTH = 3`（防止 batch/flow 无限嵌套）

### 3.10 设备管理器 (`src/device-manager.ts`)

~440 行，核心职责是路由命令到正确的平台适配器：

```typescript
class DeviceManager {
  // 5 个适配器实例
  private androidAdapter: AndroidAdapter;
  private iosAdapter: IosAdapter;
  private desktopAdapter: DesktopAdapter;
  private auroraAdapter: AuroraAdapter;
  private browserAdapter: BrowserAdapter;

  // 设备接口
  interface Device {
    id: string;
    name: string;
    platform: "android" | "ios" | "desktop" | "aurora" | "browser";
    state: string;
    isSimulator: boolean;
  }

  // 自动设备检测 (FIX #8)
  autoDetectDevice(): Device;
}
```

### 3.11 平台适配器接口 (`src/adapters/platform-adapter.ts`)

统一的平台操作契约：

```typescript
interface PlatformAdapter {
  // 交互
  tap(x, y): Promise<void>;
  doubleTap(x, y): Promise<void>;
  longPress(x, y, duration): Promise<void>;
  swipe(x1, y1, x2, y2, duration): Promise<void>;
  swipeDirection(direction): Promise<void>;
  inputText(text): Promise<void>;
  pressKey(key): Promise<void>;

  // 信息获取
  screenshotAsync(): Promise<Buffer>;
  getUiHierarchy(): Promise<string>;

  // 应用管理
  launchApp(pkg): Promise<void>;
  stopApp(pkg): Promise<void>;
  installApp(path): Promise<void>;

  // 权限
  grantPermission(pkg, perm): Promise<void>;
  revokePermission(pkg, perm): Promise<void>;
  resetPermissions(pkg): Promise<void>;

  // 系统
  shell(cmd): Promise<string>;
  getLogs(filter): Promise<string>;
  clearLogs(): Promise<void>;
  getSystemInfo(): Promise<object>;
}
```

### 3.12 iOS 客户端 (`src/ios/client.ts`)

```typescript
class IosClient {
  // simctl: iOS 模拟器管理
  // WDA (WebDriverAgent): UI 检查

  // 自动检测已启动的模拟器
  // 自动启动 WDA
  // 支持: tap, swipe, screenshot, UI hierarchy
}
```

### 3.13 应用商店客户端 (`src/store/store-client.ts`)

```typescript
interface StoreClient {
  upload(apkPath): Promise<void>;
  setReleaseNotes(notes): Promise<void>;
  submit(): Promise<void>;
  getReleases(): Promise<Release[]>;
  discard(): Promise<void>;
}

// 实现:
// - Google Play (google-auth-library)
// - Huawei AppGallery
// - RuStore
```

---

## 四、底层原子操作汇总

### Android (ADB)

| 操作 | 实现方式 | 超时 |
|------|----------|------|
| 点击 | `adb shell input tap x y` | 15s |
| 双击 | 两次 tap (100ms 间隔) | 15s |
| 长按 | `adb shell input swipe x y x y duration` | 15s |
| 滑动 | `adb shell input swipe x1 y1 x2 y2 ms` | 15s |
| 输入文本 | `adb shell input text` + broadcast (特殊字符) | 15s |
| 按键 | `adb shell input keyevent KEYCODE` | 15s |
| 截图 | `adb exec-out screencap -p` | 30s |
| UI dump | `adb shell uiautomator dump` | 15s |
| 启动应用 | `adb shell monkey -c LAUNCHER` | 15s |
| 停止应用 | `adb shell am force-stop` | 15s |
| 安装应用 | `adb shell pm install` | 30s |
| 剪贴板 | `adb shell cmd clipboard` | 15s |
| 权限管理 | `adb shell pm grant/revoke` | 15s |
| 当前Activity | `adb shell dumpsys activity` (6种正则) | 15s |
| 系统信息 | `adb shell wm size` + `dumpsys battery` + `getprop` | 15s |

### iOS (simctl + WDA)

| 操作 | 实现方式 |
|------|----------|
| 设备管理 | `xcrun simctl list/boot/shutdown` |
| 截图 | `xcrun simctl io screenshot` |
| UI 树 | WDA accessibility tree |
| 点击 | WDA tap |
| 应用管理 | `xcrun simctl install/launch/terminate` |

### Browser (CDP)

| 操作 | 实现方式 |
|------|----------|
| 导航 | `Page.navigate` |
| 截图 | `Page.captureScreenshot` |
| 点击 | `Input.dispatchMouseEvent` |
| 输入 | `Input.dispatchKeyEvent` |
| DOM 查询 | `Runtime.evaluate` |

---

## 五、关键设计模式

### 5.1 别名系统 — 容错 LLM 调用

这是 claude-in-mobile 最独特的设计之一：

```typescript
// 问题：LLM 经常用错误的工具名调用
// "click" → 应该是 "input_tap"
// "screenshot" → 应该是 "screen_capture"

// 解决：130+ 别名映射
registerAliases({
  "tap": "input_tap",
  "click": "input_tap",
  "press": "input_tap",
  "long_tap": "input_long_press",
  "long_click": "input_long_press",
  "screenshot": "screen_capture",
  "take_screenshot": "screen_capture",
  "capture": "screen_capture",
  "get_ui": "ui_tree",
  "hierarchy": "ui_tree",
  "find_element": "ui_find",
  "search": "ui_find",
  // ... v3.0.x → v3.1.x 迁移别名
});
```

其他项目都没有这种容错机制，假设 LLM 会精确调用。

### 5.2 平台适配器模式

统一接口 + 5 个平台实现：

```
PlatformAdapter (interface)
├── AndroidAdapter → AdbClient (execSync)
├── IosAdapter → IosClient (simctl + WDA)
├── DesktopAdapter → Compose Accessibility
├── AuroraAdapter → audb
└── BrowserAdapter → CDP (chrome-remote-interface)
```

DeviceManager 负责路由：
```typescript
async tap(x, y) {
  const adapter = this.getAdapterForCurrentDevice();
  return adapter.tap(x, y);
}
```

### 5.3 坐标缩放系统

解决截图压缩后坐标偏差的问题：

```
1. 截图: 设备 1080x1920 → 压缩为 540x960 → 发给 LLM
2. LLM 返回坐标: (270, 480) — 基于 540x960
3. 缩放修正: (270*2, 480*2) = (540, 960) — 映射回 1080x1920
```

`screenshotScaleMap` 记录每个平台的缩放比，自动修正。

### 5.4 元素缓存

避免每次操作都重新解析 XML (耗时 ~1-2s)：

```typescript
// ui_tree 调用后缓存解析结果
ctx.setCachedElements("android", elements);

// input_tap(index: 5) 直接从缓存取
let elements = ctx.getCachedElements("android");
```

缓存在工具上下文中管理，新的 ui_tree 调用时自动刷新。

### 5.5 Action Hints — 操作反馈

操作后自动检测 UI 变化并提供反馈，减少 LLM 需要额外截图确认的次数：

```typescript
// 流程:
// 1. 执行操作 (如 tap)
// 2. 等待 150ms
// 3. 重新获取 UI 树
// 4. 与操作前的 UI 树做 diff
// 5. 生成文字描述: "页面已切换到 Settings", "出现了对话框"
// 6. 建议下一步操作: "有 2 个按钮可以点击: OK, Cancel"
```

### 5.6 Flow 引擎 — 服务端自动化

将多步骤操作收敛到单次 MCP 往返：

**无 Flow 引擎时**：
```
LLM → tap → 返回结果 → LLM → text → 返回结果 → LLM → screenshot → 返回结果
(3 次 MCP 往返，大量 token 消耗)
```

**有 Flow 引擎时**：
```
LLM → flow_batch([tap, text, screenshot]) → 返回所有结果
(1 次 MCP 往返)
```

**flow_run 条件逻辑**：
```
LLM → flow_run([
  {tap "登录", if_not_found: "scroll_down"},
  {text "密码", repeat: {until_found: "欢迎"}},
]) → 返回最终结果
(1 次 MCP 往返，自动处理异常情况)
```

### 5.7 稳定截图 — 防动画干扰

```typescript
async function waitForStableScreenshot() {
  // 重试 3 次，间隔 300ms
  // 连续两张截图差异 < 2% → 认为稳定
  // 避免捕获动画中间帧
}
```

### 5.8 智能 Diff 截图

根据变化程度返回不同内容：

```
变化 < 5%  → 仅文字描述 (节省 token)
变化 5-80% → 裁剪变化区域 (节省带宽)
变化 > 80% → 完整截图
```

---

## 六、项目对比

### 6.1 与 mobile-mcp 对比

| 维度 | claude-in-mobile | mobile-mcp |
|------|-----------------|------------|
| 语言 | TypeScript (Node.js) | TypeScript (Node.js) |
| 平台 | 5 平台 (Android/iOS/Desktop/Aurora/Browser) | 2 平台 (Android + iOS) |
| 设备通信 | ADB + simctl + WDA + CDP + Compose | ADB + Maestro + idb |
| UI 解析 | 正则解析 + 模糊匹配 + 缓存 | XML 解析 + Accessibility |
| 截图 | jimp + diff + stable + annotate | Sharp 压缩 |
| 别名系统 | 130+ 别名 | 无 |
| Flow 引擎 | batch + 条件 run | 无 |
| 坐标缩放 | 自动修正 | 无 |
| Action Hints | 操作后 UI diff 反馈 | 无 |
| 元素查找 | text/resourceId/index/label + 模糊匹配 | 无 |
| 应用商店 | Play/Huawei/RuStore 上传 | 无 |
| 工具数量 | ~40 个 | 20+ 个 |
| 代码量 | ~8000+ 行 | ~3000 行 |

### 6.2 与 android-mcp-server 对比

| 维度 | claude-in-mobile | android-mcp-server |
|------|-----------------|-------------------|
| 平台 | 5 平台 | Android only |
| 代码量 | ~8000+ 行 | ~200 行 |
| UI 分析 | 模糊匹配 + 缓存 + diff + 屏幕分析 | 基础文本格式 |
| 别名 | 130+ | 无 |
| Flow 引擎 | 有 | 无 |
| 坐标缩放 | 有 | 无 |
| 应用商店 | 有 | 无 |
| 工程化 | npm 包 + Rust CLI | npm 包 |

### 6.3 与 Android-MCP (CursorTouch) 对比

| 维度 | claude-in-mobile | Android-MCP (CursorTouch) |
|------|-----------------|--------------------------|
| 语言 | TypeScript | Python |
| 平台 | 5 平台 | Android only |
| 设备通信 | ADB (execSync) | uiautomator2 (ATX Agent) |
| 元素查找 | 模糊匹配 + 多维筛选 | u2 Selector (原生) |
| 等待元素 | ui_wait (轮询) | WaitForElement (u2 原生) |
| UI 分析 | 屏幕分析 + diff + 建议 | 标注截图 + 表格 |
| Flow | batch + run | 无 |
| 别名 | 130+ | 无 |
| 代码量 | ~8000+ 行 | ~700 行 |

### 6.4 与 appium-mcp 对比

| 维度 | claude-in-mobile | appium-mcp |
|------|-----------------|------------|
| 底层引擎 | ADB + simctl + WDA | Appium (WebDriver) |
| 元素查找 | 模糊匹配 (自实现) | XPath/AccessibilityID (Appium) |
| 等待机制 | ui_wait (轮询) | WebDriver 隐式/显式等待 |
| 跨平台 | 5 平台 (自实现适配器) | 3 平台 (Appium 统一) |
| 依赖 | Node.js + ADB | Appium Server + Node.js |
| 复杂度 | 中 | 高 (Appium 栈) |
| Flow 引擎 | 有 | 无 |
| 应用商店 | 有 | 无 |

### 6.5 与 Open-AutoGLM 对比

| 维度 | claude-in-mobile | Open-AutoGLM |
|------|-----------------|--------------|
| 定位 | MCP 工具层 (全平台) | 自治 Agent + 模型训练 |
| 智能层 | 外部 (MCP Client) | 内置 (ChatGLM) |
| 操作层 | ADB 命令 | Accessibility + 坐标 |
| 学习 | 无 | 强化学习 |
| Flow | 服务端条件流程 | Agent 自主规划 |
| 运行需求 | Node.js + ADB | GPU 集群 |
| 使用门槛 | 低 | 极高 |

### 6.6 与 AppAgent 对比

| 维度 | claude-in-mobile | AppAgent |
|------|-----------------|----------|
| 定位 | MCP 工具层 | 自治 Agent (学术) |
| 设备通信 | ADB | ADB |
| UI 分析 | 模糊匹配 + 屏幕分析 + diff | 数字标注 + GPT-4V |
| 文档系统 | 无 | 自动 UI 文档 |
| 探索学习 | 无 | 两阶段探索-利用 |
| Flow 引擎 | batch + run | Agent 自主循环 |

### 6.7 与 droidrun 对比

| 维度 | claude-in-mobile | droidrun |
|------|-----------------|----------|
| 定位 | MCP 工具层 (全平台) | 自治 Agent 框架 (Android) |
| 平台 | 5 平台 | Android only |
| Agent 模式 | 无 (被动工具) | Manager/Executor/FastAgent |
| UI 分析 | 模糊匹配 + diff | XML 解析 |
| Flow | 服务端条件流程 | Agent 自主循环 |
| 拟人操作 | 无 | Stealth 模式 (Bezier) |
| 别名 | 130+ | 无 |
| 应用商店 | Play/Huawei/RuStore | 无 |

---

## 七、代码质量评估

### 优点

- **架构清晰**：平台适配器模式解耦良好，新增平台只需实现 PlatformAdapter
- **容错设计**：130+ 别名解决 LLM 命名偏差，显著提升实际使用成功率
- **性能优化**：元素缓存、坐标缩放、稳定截图、diff 截图，多层优化
- **Flow 引擎**：将多步操作收敛到单次 MCP 往返，大幅减少 token 消耗
- **Action Hints**：操作反馈减少不必要的截图确认
- **全平台支持**：5 个平台使用统一 API
- **智能 UI 分析**：模糊匹配 + 屏幕结构分析 + diff，超越基础 XML 解析
- **工程化成熟**：npm 包 + Rust CLI + Claude Code 插件

### 局限

- **正则解析 XML**：`ui-parser.ts` 用正则而非 DOM 解析 XML，可能在复杂布局时出错
- **execSync 阻塞**：ADB 客户端使用同步命令执行，长操作会阻塞
- **无 Selector API**：不像 Android-MCP 使用 u2 原生 Selector，模糊匹配不如原生可靠
- **无持久化学习**：不像 AppAgent 有 UI 文档积累，每次会话从零开始
- **复杂度高**：75+ 文件、8000+ 行代码，维护成本较高
- **WDA 依赖**：iOS 功能依赖 WebDriverAgent，需要 Xcode 环境

---

## 八、总结

claude-in-mobile 是所有分析项目中**功能最全面、工程化程度最高**的 MCP Server。其核心价值在于：

1. **全平台统一接口**：5 个平台共用一套 API，是其他项目不具备的
2. **130+ 别名容错**：显著提升 LLM 调用成功率，是实战经验的沉淀
3. **Flow 引擎**：将复杂操作收敛到单次 MCP 往返，减少 token 和延迟
4. **智能 UI 分析**：模糊匹配 + diff + hints，超越基础的 XML 解析
5. **坐标缩放系统**：解决截图压缩后坐标偏差的实际问题

与其他项目的核心差异：
- 与 mobile-mcp/android-mcp-server：功能集是它们的超集，但复杂度也更高
- 与 Android-MCP (CursorTouch)：u2 Selector 更原生可靠，但 claude-in-mobile 更全面
- 与 AppAgent/droidrun/Open-AutoGLM：它们是自治 Agent，claude-in-mobile 是被动工具层
- **最佳使用场景**：需要全平台支持、高容错性、复杂流程自动化的 MCP 集成
