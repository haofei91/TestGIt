# android-mcp-server 源码解读

## 1. 项目定位

**android-mcp-server** 是一个基于 MCP（Model Context Protocol）协议的 Android 设备控制服务器，使用 TypeScript/Node.js 开发。它通过 ADB 直接与 Android 设备通信，为 AI 助手（如 Claude、Cursor、VS Code 等）提供截图、UI 树解析、触控操作、日志读取等 25 个工具。

核心特征：
- **仅支持 Android**，不支持 iOS 或 HarmonyOS
- **零应用修改**：纯 ADB 外部控制，无需目标 App 集成任何 SDK
- **极简设计**：整个项目核心仅 2 个源文件（`index.ts` + `adb.ts`），约 1200 行代码
- **直连 ADB**：不依赖 Appium 或其他中间层

---

## 2. 架构总览

```
AI 助手 (Claude / Cursor / VS Code)
    |
    | stdio (JSON-RPC / MCP 协议)
    |
    v
+---------------------------+
|  index.ts  (MCP Server)   |   <- 注册 25 个 tool，参数校验 (zod)，Sharp 截图压缩
+---------------------------+
    |
    | 方法调用
    |
    v
+---------------------------+
|  adb.ts   (ADB Layer)     |   <- 持久化 Shell、UI 树解析、设备发现、命令执行
+---------------------------+
    |
    | spawn / execFile
    |
    v
+---------------------------+
|  ADB (platform-tools)     |   <- Android SDK 自带
+---------------------------+
    |
    | USB / TCP
    |
    v
+---------------------------+
|  Android 设备 / 模拟器     |
+---------------------------+
```

核心组件只有两层：
1. **index.ts**（711 行）：MCP 协议层，负责工具注册、参数校验、结果格式化
2. **adb.ts**（513 行）：ADB 封装层，负责设备通信、持久化 Shell、UI 树解析

---

## 3. 核心设计模式

### 3.1 持久化 Shell（Persistent Shell）

这是 android-mcp-server 最关键的性能优化。传统方式每次执行 `adb shell <cmd>` 都会 spawn 一个新进程，开销约 100-200ms。持久化 Shell 只创建一次 `adb shell` 进程，后续命令通过 stdin 发送，用特殊标记 `__MCP_DONE__` 检测输出结束。

```
伪代码：
class Adb:
    persistentShells = Map<deviceKey, ShellEntry>

    getPersistentShell(deviceId):
        key = deviceId ?? "__default__"
        existing = persistentShells.get(key)
        if existing && existing.process.exitCode == null:
            return existing

        // 创建新的持久化 shell
        proc = spawn(adbPath, ["-s", deviceId, "shell"])
        entry = { process: proc, pending: [] }

        proc.stdout.on("data", chunk):
            buffer += chunk
            if buffer.contains("__MCP_DONE__"):
                output = buffer.substring(0, markerIndex)
                pending.shift().resolve(output)  // 完成一个请求

        proc.on("close"):
            // 清理并拒绝所有待处理请求
            persistentShells.delete(key)

        persistentShells.set(key, entry)
        return entry

    shellExec(command, deviceId):
        shell = getPersistentShell(deviceId)
        return new Promise((resolve, reject):
            timer = setTimeout(10s, reject("timeout"))
            shell.pending.push({ resolve, reject })
            shell.process.stdin.write(command + "; echo __MCP_DONE__\n")
        )
```

**关键细节**：
- 每个设备维护独立的持久化 Shell（按 `deviceId` 索引）
- 使用 `__MCP_DONE__` 标记作为输出分隔符
- 10 秒超时保护，防止命令卡死
- 进程异常关闭时自动 reject 所有待处理请求

### 3.2 命令路由策略

`exec()` 方法会自动判断是否走持久化 Shell：

```typescript
async exec(args, deviceId):
    if args[0] == "shell" && args.length >= 2:
        return shellExec(args.slice(1).join(" "), deviceId)  // 走持久化 Shell
    else:
        return execFileAsync(adbPath, fullArgs)               // 走独立进程
```

`shell` 类命令走持久化通道，`devices`、`install`、`pull`、`exec-out` 等走独立进程。

### 3.3 UI 树解析

UI 树解析使用 `uiautomator dump`，有两级降级策略：

```
伪代码：
getUiTree(deviceId):
    try:
        raw = exec(["shell", "uiautomator", "dump", "/dev/tty"])  // 方案A: 直接输出到 stdout
        xml = raw.substring(raw.indexOf("<?xml") or raw.indexOf("<hierarchy"))
    catch:
        exec(["shell", "uiautomator", "dump", "/sdcard/window_dump.xml"])  // 方案B: 写文件再读
        xml = exec(["shell", "cat", "/sdcard/window_dump.xml"])

    return parseUiXml(xml)
```

XML 解析使用正则而非 DOM 解析器：

```
parseUiXml(xml):
    nodeRegex = /<node\s[^>]*?\/?>/g
    for each match in xml:
        提取 resource-id, text, content-desc, bounds, clickable 等属性
        过滤无交互意义的节点 (无 id/text/desc 且不可点击)
        计算 center 坐标 = bounds 中心点
    return UiElement[]
```

**过滤规则**：只保留有 `resourceId`、`text`、`contentDesc` 或 `clickable=true` 的节点，大幅减少返回数据量。

### 3.4 截图压缩

截图流程避免了写临时文件：

```
伪代码：
screenshot(deviceId):
    buffer = execBuffer(["exec-out", "screencap", "-p"], deviceId)  // 管道直出，不写文件
    return buffer

compressScreenshot(pngBuffer, maxWidth=1280):
    metadata = sharp(pngBuffer).metadata()
    if metadata.width > maxWidth:
        return sharp(pngBuffer).resize(width=maxWidth).png(quality=80).toBuffer()
    return sharp(pngBuffer).png(quality=80).toBuffer()
```

`exec-out` 是 ADB 的二进制管道模式，直接将 screencap 的原始 PNG 数据通过 stdout 传出，比先写 `/sdcard/tmp.png` 再 `adb pull` 快很多。

### 3.5 ADB 路径发现

```
discoverPath(toolRelativePath, fallbackName):
    1. 检查 $ANDROID_HOME/<toolRelativePath>
    2. 检查 ~/Library/Android/sdk/<toolRelativePath>  (macOS 默认路径)
    3. 回退到 fallbackName (依赖 $PATH)
```

### 3.6 设备信息缓存

`getDeviceInfo()` 使用 Map 缓存，同一设备只查询一次 `getprop` 等命令。6 个属性（model, manufacturer, androidVersion, apiLevel, screenSize, density）通过 `Promise.all` 并行获取。

---

## 4. 底层原子操作清单

以下是 android-mcp-server 使用的所有 ADB 原子命令：

| 分类 | ADB 命令 | 用途 | 对应方法 |
|------|----------|------|----------|
| **设备管理** | `adb devices -l` | 列出已连接设备 | `getDevices()` |
| | `emulator -list-avds` | 列出可用 AVD | `getAvds()` |
| | `emulator -avd <name>` | 启动模拟器 | `startEmulator()` |
| **截图** | `adb exec-out screencap -p` | 截图（二进制管道） | `screenshot()` |
| **UI 树** | `adb shell uiautomator dump /dev/tty` | UI 树转储（直输） | `getUiTree()` |
| | `adb shell uiautomator dump /sdcard/window_dump.xml` | UI 树转储（写文件，降级方案） | `getUiTree()` |
| | `adb shell cat /sdcard/window_dump.xml` | 读取 UI 转储文件 | `getUiTree()` |
| **触控输入** | `adb shell input tap <x> <y>` | 点击 | `tap()` |
| | `adb shell input swipe <x1> <y1> <x2> <y2> [duration]` | 滑动 / 长按 | `swipe()` / `longPress()` |
| | `adb shell input text <escaped>` | 文本输入 | `typeText()` |
| | `adb shell input keyevent <keycode>` | 按键 | `pressKey()` |
| **应用管理** | `adb shell am start -n <pkg>/<activity>` | 启动指定 Activity | `launchApp()` |
| | `adb shell monkey -p <pkg> -c android.intent.category.LAUNCHER 1` | 启动 App 默认入口 | `launchApp()` |
| | `adb install -r <apk>` | 安装 APK | `installApk()` |
| | `adb shell dumpsys activity activities` | 获取当前前台 Activity | `getCurrentActivity()` |
| **日志** | `adb logcat -d -t <N>` | 读取最近 N 条日志 | `getLogcat()` |
| | `adb logcat -d -T <timestamp>` | 读取指定时间后的日志 | `getLogcat()` |
| | `adb logcat -c` | 清空日志缓冲区 | `clearLogcat()` |
| | `adb shell pidof <package>` | 按包名过滤日志 | `getLogcat()` |
| **设备属性** | `adb shell getprop ro.product.model` | 设备型号 | `getDeviceInfo()` |
| | `adb shell getprop ro.product.manufacturer` | 制造商 | `getDeviceInfo()` |
| | `adb shell getprop ro.build.version.release` | Android 版本 | `getDeviceInfo()` |
| | `adb shell getprop ro.build.version.sdk` | API 级别 | `getDeviceInfo()` |
| | `adb shell wm size` | 屏幕分辨率 | `getDeviceInfo()` |
| | `adb shell wm density` | 屏幕密度 | `getDeviceInfo()` |
| **文件传输** | `adb pull <remote> <local>` | 从设备拉取文件 | `pullFile()` |
| **Shell** | `adb shell <any>` | 任意 shell 命令 | `shell()` |

---

## 5. 25 个 MCP 工具一览

### 设备管理 (3)
| 工具 | 输入参数 | 功能 |
|------|----------|------|
| `list_devices` | 无 | 列出已连接设备，显示 ID、状态、型号 |
| `list_avds` | 无 | 列出可用 Android 虚拟设备 |
| `start_emulator` | `avd_name` | 启动模拟器，等待最长 60s |

### 截图与 UI 分析 (2)
| 工具 | 输入参数 | 功能 |
|------|----------|------|
| `screenshot` | `device_id?`, `save_path?` | 截图并压缩，返回 base64 图片 |
| `get_ui_tree` | `device_id?` | 获取 UI 元素层级树 |

### 交互操作 (12)
| 工具 | 输入参数 | 功能 |
|------|----------|------|
| `tap` | `x`, `y` | 坐标点击 |
| `tap_element` | `by`, `value` | 按 resource-id / text / content-desc 查找并点击元素 |
| `tap_and_wait` | `by`, `value`, `wait_ms?` | 点击元素 + 等待 UI 稳定 + 返回新 UI 树 |
| `long_press` | `x`, `y`, `duration_ms?` | 长按（通过 swipe 到同一点实现） |
| `double_tap` | `x`, `y`, `interval_ms?` | 双击 |
| `multi_tap` | `x`, `y`, `count`, `interval_ms?` | 多次点击 |
| `tap_sequence` | `steps[]` | 复合操作序列（tap/type/key/swipe/pause 任意组合） |
| `type_text` | `text` | 向焦点输入框输入文本 |
| `press_key` | `key` | 按键（back/home/enter 等 16 种） |
| `swipe` | `start_x/y`, `end_x/y`, `duration_ms?` | 滑动手势 |
| `scroll_to_element` | `by`, `value`, `max_scrolls?` | 循环滚动直到目标元素可见 |
| `wait_for_element` | `by`, `value`, `timeout_ms?` | 轮询等待元素出现 |

### 应用管理 (5)
| 工具 | 输入参数 | 功能 |
|------|----------|------|
| `launch_app` | `package_name`, `activity?` | 启动应用 |
| `install_apk` | `apk_path` | 安装 APK |
| `get_current_activity` | 无 | 获取前台 Activity |
| `pull_file` | `remote_path`, `local_path` | 从设备拉取文件 |
| `adb_shell` | `command` | 执行任意 ADB shell 命令 |

### 诊断工具 (3)
| 工具 | 输入参数 | 功能 |
|------|----------|------|
| `get_logs` | `package_name?`, `level?`, `lines?`, `since?` | 获取 logcat 日志 |
| `clear_logs` | 无 | 清空日志缓冲区 |
| `get_device_info` | 无 | 获取设备型号、版本、分辨率等 |

---

## 6. 关键代码走读

### 6.1 元素定位与点击 (`tap_element`)

```typescript
// index.ts:184-219
// 1. 获取当前 UI 树
const elements = await adb.getUiTree(device_id);

// 2. 按匹配策略查找
const finder = {
  "resource-id": (el) => el.resourceId === value || el.resourceId.endsWith(`:id/${value}`),
  "text":        (el) => el.text === value || el.text.toLowerCase().includes(value.toLowerCase()),
  "content-desc":(el) => el.contentDesc === value || el.contentDesc.toLowerCase().includes(value.toLowerCase()),
};
const el = elements.find(finder[by]);

// 3. 找不到时列出所有可点击元素辅助调试
if (!el) {
  return { content: [...], isError: true };
}

// 4. 点击元素中心坐标
await adb.tap(el.center.x, el.center.y, device_id);
```

特点：
- `resource-id` 支持短名匹配（如 `search_bar` 匹配 `com.example:id/search_bar`）
- `text` 和 `content-desc` 支持大小写不敏感的子串匹配
- 错误时返回可点击元素列表，方便 AI 自行修正

### 6.2 复合操作序列 (`tap_sequence`)

```typescript
// adb.ts:340-368
async tapSequence(steps, keycodeMap, deviceId):
    for step of steps:
        switch step.action:
            "tap"          -> tap(x, y)
            "tap_and_wait" -> tap(x, y) + sleep(wait_ms)
            "type_text"    -> typeText(text)
            "press_key"    -> pressKey(keycodeMap[key])
            "swipe"        -> swipe(x1, y1, x2, y2, duration)
            "pause"        -> sleep(ms)
```

这是一个减少 AI 与 MCP Server 往返次数的关键设计。多步操作（如 "点击搜索框 → 输入文字 → 按回车"）可以打包成一个 `tap_sequence` 调用完成。

### 6.3 滚动查找元素 (`scroll_to_element`)

```typescript
// index.ts:482-518
// 1. 获取屏幕尺寸
const info = await adb.getDeviceInfo(device_id);
const [w, h] = info.screenSize.split("x").map(Number);
const centerX = w / 2;
const fromY = h * 0.7;  // 从 70% 高度
const toY = h * 0.3;    // 滑到 30% 高度

// 2. 循环: 检查 UI 树 → 没找到就滚动
for (let i = 0; i < max_scrolls; i++):
    elements = getUiTree()
    if elements.find(finder[by]):
        return "Found!"
    swipe(centerX, fromY, centerX, toY, 500ms)
    sleep(500ms)
```

### 6.4 文本输入转义

```typescript
// adb.ts:46-61
function escapeShellText(text):
    text.replace(/ /g, "%s")     // ADB 空格转义
        .replace(/'/g, "\\'")    // 引号转义
        .replace(/&/g, "\\&")    // shell 特殊字符转义
        ...
```

**局限**：使用 `adb shell input text`，不支持中文等非 ASCII 字符。对比 Open-AutoGLM 使用 ADB Keyboard 广播方式，后者通过 Base64 编码支持中文输入。

---

## 7. 与 mobile-mcp、appium-mcp 的对比

### 7.1 架构对比

| 维度 | android-mcp-server | mobile-mcp | appium-mcp |
|------|-------------------|------------|------------|
| **语言** | TypeScript/Node.js | TypeScript/Node.js | Python |
| **核心代码量** | ~1200 行 (2 文件) | ~3000+ 行 | ~2000+ 行 |
| **平台支持** | Android Only | Android + iOS | Android + iOS |
| **中间层** | 无（直连 ADB） | 无（直连 ADB + iOS 工具链） | 需要 Appium Server |
| **协议** | MCP (stdio) | MCP (stdio) | MCP (stdio) |
| **设备通信** | ADB | ADB + idb/xcrun | Appium (WebDriver) |
| **截图方式** | `exec-out screencap -p` + Sharp 压缩 | `screencap` + base64 | Appium screenshot API |
| **UI 分析** | `uiautomator dump` + 正则解析 | `uiautomator dump` + 正则 / Accessibility | Appium page_source |
| **元素定位** | resource-id / text / content-desc | XPath + Accessibility | XPath / id / class_name 等 |

### 7.2 工具数量与分类

| 分类 | android-mcp-server | mobile-mcp | appium-mcp |
|------|-------------------|------------|------------|
| 设备管理 | 3 | 2-3 | 2-3 |
| 截图/UI | 2 | 2-3 | 2-3 |
| 触控操作 | 12 | 5-8 | 5-8 |
| 应用管理 | 5 | 3-5 | 3-5 |
| 诊断工具 | 3 | 1-2 | 1-2 |
| **合计** | **25** | **15-20** | **15-20** |

android-mcp-server 在交互工具的粒度上更细（如 `multi_tap`、`tap_sequence`、`tap_and_wait`、`scroll_to_element`、`wait_for_element`），减少了 AI 的工具调用轮次。

### 7.3 性能优化对比

| 优化手段 | android-mcp-server | mobile-mcp | appium-mcp |
|----------|-------------------|------------|------------|
| 持久化 Shell | 有 | 无 | N/A (通过 Appium) |
| 截图压缩 | Sharp (max 1280px) | 基础压缩 | Appium 默认 |
| 设备信息缓存 | 有 | 部分 | 无 |
| 复合操作 | `tap_sequence` / `tap_and_wait` | 无 | 无 |
| UI 节点过滤 | 过滤无交互意义节点 | 全量返回 | 全量返回 |

### 7.4 各自优势

**android-mcp-server 优势：**
- 极简部署（`npx -y android-mcp-server` 一行启动）
- 持久化 Shell 带来的命令执行速度提升
- 丰富的复合操作（tap_sequence、tap_and_wait）减少 AI 交互轮次
- `adb_shell` 万能逃生舱，支持任意 ADB 命令
- 完善的日志工具（按包名、级别、时间过滤）

**mobile-mcp 优势：**
- 同时支持 Android 和 iOS
- 架构更完整（Provider 模式、平台抽象层）
- 与 mobile-mcp 生态集成更紧密

**appium-mcp 优势：**
- 基于 Appium 生态，复用成熟的自动化框架
- 支持更多定位策略（XPath、Accessibility ID 等）
- 可扩展性强，支持 Appium 插件生态

### 7.5 适用场景建议

| 场景 | 推荐方案 |
|------|----------|
| 纯 Android 项目、追求极简部署 | android-mcp-server |
| Android + iOS 跨平台项目 | mobile-mcp |
| 已有 Appium 基础设施的团队 | appium-mcp |
| 需要中文输入 | Open-AutoGLM（ADB Keyboard 方案） |
| 需要自主 Agent 循环 | Open-AutoGLM（非 MCP，内置 VLM） |
| 快速原型验证 / Bug 复现 | android-mcp-server（一行 npx 启动） |

---

## 8. 代码质量评估

**优点：**
- 代码简洁，2 个文件覆盖全部功能，认知负担低
- 持久化 Shell 是实用的性能优化
- UI 节点过滤减少无用数据
- `tap_and_wait` 等复合工具减少 AI 往返
- 良好的错误信息（找不到元素时返回可用元素列表）
- zod 参数校验，类型安全

**局限：**
- 仅支持 Android，无 iOS 支持
- 正则解析 XML 不够健壮（复杂 XML 可能失败）
- `input text` 不支持中文输入
- 无 WebView 调试能力
- 持久化 Shell 的 `__MCP_DONE__` 标记如果出现在命令输出中会导致解析错误
- 没有重连/重试机制

---

## 9. 总结

android-mcp-server 是一个"小而精"的 Android 设备 MCP 控制器。它的核心理念是：用最少的代码、最直接的方式（ADB）、最简单的部署（npx）来解决 AI 控制 Android 设备的问题。持久化 Shell 和复合操作工具是其技术亮点。如果你的场景是纯 Android 项目，且追求快速接入，这是一个值得首选的方案。
