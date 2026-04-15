# mobile-mcp 源码解读

> 仓库地址：https://github.com/mobile-next/mobile-mcp
> 本地路径：`/Users/yuhaofei/androidstudio/AI/mobile-mcp`
> 分析聚焦：原子 ADB 命令 -> Robot 抽象层 -> MCP Tool 编排层
> 分析日期：2026-04-15

---

## 一、项目定位

mobile-mcp 是一个 **Model Context Protocol (MCP) Server**，为 AI Agent（如 Claude、Copilot、Cursor 等）提供统一的移动设备自动化能力。核心价值是：

- **跨平台统一**：一套 Tool API 同时操控 Android 和 iOS 设备
- **LLM 友好**：优先使用无障碍树（Accessibility Tree）提取结构化数据，辅以截图兜底
- **零平台知识要求**：Agent 无需了解 adb / simctl / WDA 等底层细节

### 适用场景

- 原生 App 自动化测试
- Agent 驱动的多步骤用户旅程
- 表单填写、数据提取等批量任务
- 移动端 UI 交互的 Agent-to-Agent 通信

### 优点与局限

| 优点 | 局限 |
|------|------|
| 平台无关的 MCP Tool 接口 | WebView 内部元素不可交互 |
| 支持真机 + 模拟器 + 远程设备池 | iOS 真机需手动配置 go-ios + WDA + tunnel |
| 截图自动压缩，对 LLM 友好 | Android 非 ASCII 输入需额外安装 DeviceKit |
| 安全防注入（包名/路径/URL 校验） | 同步 execFileSync 可能阻塞（30s 超时） |

---

## 二、整体架构

```
┌───────────────────────────────────────────────────────────────┐
│                     AI Agent (Claude, Cursor, etc.)            │
│                            ↕ MCP Protocol                      │
├───────────────────────────────────────────────────────────────┤
│                     MCP Server  (server.ts)                    │
│           注册 20+ Tool，参数校验，结果格式化                    │
│                     ↕ Robot Interface                           │
├───────────────────────────────────────────────────────────────┤
│  Robot 接口 (robot.ts) — 16 个标准方法                          │
│  ┌─────────────┐ ┌──────────────┐ ┌────────────────────────┐  │
│  │ AndroidRobot │ │  IosRobot    │ │  MobileDevice          │  │
│  │ (adb 直连)   │ │(go-ios + WDA)│ │ (mobilecli CLI)        │  │
│  └──────┬──────┘ └──────┬───────┘ └───────────┬────────────┘  │
│         │               │                      │               │
│   ┌─────▼──────┐  ┌─────▼────────┐  ┌─────────▼───────────┐  │
│   │ adb CLI    │  │ go-ios CLI   │  │ mobilecli binary     │  │
│   │            │  │ WDA HTTP API │  │                      │  │
│   └────────────┘  └──────────────┘  └──────────────────────┘  │
├───────────────────────────────────────────────────────────────┤
│                   辅助模块                                      │
│  png.ts - PNG 头解析                                           │
│  image-utils.ts - sips/ImageMagick 图片压缩                    │
│  utils.ts - 安全校验（包名/路径/扩展名）                         │
│  logger.ts - 日志                                              │
│  mobilecli.ts - mobilecli 二进制包装                             │
└───────────────────────────────────────────────────────────────┘
```

此外还有一个 **Simctl** 实现（`iphone-simulator.ts`），通过 `xcrun simctl` 直连 iOS 模拟器，当前版本中它已被 `MobileDevice`（通过 mobilecli）替代，但源码仍保留。

### 四种 Robot 实现对比

| 实现类 | 目标设备 | 底层通道 | UI 交互 | App 管理 |
|--------|---------|---------|---------|---------|
| `AndroidRobot` | Android 真机/模拟器 | `adb` CLI | `input tap/swipe/text/keyevent` | `am/pm/monkey` |
| `IosRobot` | iOS 真机 | `go-ios` CLI + WDA HTTP | WDA W3C Actions API | `go-ios launch/kill/install` |
| `Simctl` | iOS 模拟器 (旧) | `xcrun simctl` + WDA HTTP | WDA W3C Actions API | `simctl install/launch` |
| `MobileDevice` | iOS 模拟器 (新) | `mobilecli` 二进制 | `mobilecli io tap/swipe/text` | `mobilecli apps install/launch` |

---

## 三、原子 ADB 命令完整清单

`AndroidRobot` 类（`src/android.ts`）中所有 `adb` 调用即为 Android 侧的"原子命令层"。

### 3.1 设备信息类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 1 | `adb devices` | 列出已连接设备 | android.ts:570 |
| 2 | `adb shell getprop ro.build.version.release` | 获取 Android 版本号 | android.ts:537 |
| 3 | `adb shell getprop ro.boot.qemu.avd_name` | 获取模拟器 AVD 名称 | android.ts:549 |
| 4 | `adb shell getprop ro.product.model` | 获取设备型号 | android.ts:559 |
| 5 | `adb shell pm list features` | 列出系统特性（判断 tv/mobile） | android.ts:96 |
| 6 | `adb shell wm size` | 获取屏幕分辨率 | android.ts:105 |

### 3.2 Display 管理类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 7 | `adb shell dumpsys SurfaceFlinger --display-id` | 获取 Display 数量 | android.ts:245 |
| 8 | `adb shell cmd display get-displays` | 获取活跃显示器列表（Android 11+） | android.ts:255 |
| 9 | `adb shell dumpsys display` | 获取 Display 信息（fallback） | android.ts:281 |

### 3.3 输入操作类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 10 | `adb shell input tap {x} {y}` | 点击屏幕坐标 | android.ts:452 |
| 11 | `adb shell input swipe {x0} {y0} {x1} {y1} {duration}` | 滑动手势 | android.ts:202, 241 |
| 12 | `adb shell input text {text}` | 输入 ASCII 文本 | android.ts:426 |
| 13 | `adb shell input keyevent {KEYCODE}` | 按键事件 | android.ts:448 |

### 3.4 App 管理类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 14 | `adb shell cmd package query-activities -a MAIN -c LAUNCHER` | 列出可启动 App | android.ts:121 |
| 15 | `adb shell pm list packages` | 列出所有已安装包 | android.ts:135 |
| 16 | `adb shell monkey -p {pkg} -c LAUNCHER 1` | 启动 App | android.ts:156 |
| 17 | `adb shell am force-stop {pkg}` | 强制停止 App | android.ts:372 |
| 18 | `adb shell am start -a VIEW -d {url}` | 打开 URL | android.ts:398 |
| 19 | `adb install -r {path}` | 安装 APK | android.ts:377 |
| 20 | `adb uninstall {pkg}` | 卸载 App | android.ts:389 |
| 21 | `adb shell cmd locale set-app-locales {pkg} --locales {locale}` | 设置 App 语言（Android 13+） | android.ts:149 |

### 3.5 剪贴板类（需 DeviceKit）

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 22 | `adb shell am broadcast -a devicekit.clipboard.set -e encoding base64 -e text {base64}` | 设置剪贴板内容 | android.ts:432 |
| 23 | `adb shell input keyevent KEYCODE_PASTE` | 粘贴 | android.ts:433 |
| 24 | `adb shell am broadcast -a devicekit.clipboard.clear` | 清除剪贴板 | android.ts:436 |

### 3.6 屏幕截取与 UI 层级类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 25 | `adb exec-out screencap -p [-d displayId]` | 截屏（PNG 格式） | android.ts:310-321 |
| 26 | `adb exec-out uiautomator dump /dev/tty` | 导出 UI 层级 XML | android.ts:481 |

### 3.7 系统设置类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 27 | `adb shell settings put system accelerometer_rotation 0` | 关闭自动旋转 | android.ts:470 |
| 28 | `adb shell content insert --uri content://settings/system --bind name:s:user_rotation --bind value:i:{v}` | 设置屏幕方向 | android.ts:471 |
| 29 | `adb shell settings get system user_rotation` | 获取屏幕方向 | android.ts:475 |

### 3.8 进程类

| # | 命令 | 用途 | 源码行 |
|---|------|------|--------|
| 30 | `adb shell ps -e` | 列出运行中进程 | android.ts:163 |

---

## 四、从原子命令到 MCP Tool 的扩展机制

### 4.1 三层架构设计

```
Layer 3: MCP Tool 注册层 (server.ts)
    ↓ 参数校验 + 结果格式化 + 错误包装
Layer 2: Robot 接口抽象层 (robot.ts)
    ↓ 统一 16 个方法签名
Layer 1: 平台实现层 (android.ts / ios.ts / mobile-device.ts)
    ↓ 调用原子命令
底层: adb / go-ios / mobilecli / WDA HTTP
```

### 4.2 Tool 注册机制

`server.ts` 中通过自定义 `tool()` 辅助函数注册 MCP Tool：

```typescript
// 伪代码
tool(name, title, description, zodSchema, annotations, async (args) => {
    // 1. 通过 getRobotFromDevice(device) 获取对应平台的 Robot 实现
    // 2. 调用 Robot 接口方法
    // 3. 格式化返回结果
})
```

`getRobotFromDevice()` 的设备路由逻辑：

```pseudo
function getRobotFromDevice(deviceId):
    // 优先检查 iOS 物理设备
    if deviceId in IosManager.listDevices():
        return new IosRobot(deviceId)

    // 再检查 Android 设备
    if deviceId in AndroidDeviceManager.getConnectedDevices():
        return new AndroidRobot(deviceId)

    // 最后检查 iOS 模拟器（通过 mobilecli）
    if deviceId in mobilecli.getDevices(platform:"ios", type:"simulator"):
        return new MobileDevice(deviceId)

    throw "Device not found"
```

### 4.3 命令组合关系

```
原子 ADB 命令                          MCP Tool
──────────────                         ────────
adb devices + getprop x3 ──────────→ mobile_list_available_devices
adb shell wm size ─────────────────→ mobile_get_screen_size
adb exec-out screencap -p ─────────→ mobile_take_screenshot / mobile_save_screenshot
adb shell input tap ───────────────→ mobile_click_on_screen_at_coordinates
adb shell input tap x2 ───────────→ mobile_double_tap_on_screen
adb shell input swipe (原地) ──────→ mobile_long_press_on_screen_at_coordinates
adb shell input swipe (移动) ──────→ mobile_swipe_on_screen (需先调 wm size 计算坐标)
adb shell input text ──────────────→ mobile_type_keys (ASCII 路径)
adb shell input keyevent ──────────→ mobile_press_button / mobile_type_keys(submit)
am broadcast + keyevent PASTE ─────→ mobile_type_keys (非 ASCII，需 DeviceKit)
adb exec-out uiautomator dump ─────→ mobile_list_elements_on_screen
cmd package query-activities ──────→ mobile_list_apps
adb shell monkey ──────────────────→ mobile_launch_app
cmd locale set-app-locales ────────→ mobile_launch_app (locale 参数)
adb shell am force-stop ───────────→ mobile_terminate_app
adb install -r ────────────────────→ mobile_install_app
adb uninstall ─────────────────────→ mobile_uninstall_app
adb shell am start -a VIEW -d ────→ mobile_open_url
settings put/get + content insert ─→ mobile_set/get_orientation
pm list features ──────────────────→ (内部: 判断设备类型 tv/mobile)
dumpsys SurfaceFlinger/display ────→ (内部: 多屏截图时定位 displayId)
```

---

## 五、每个 MCP Tool 的核心伪代码

### 5.1 mobile_list_available_devices

**功能：** 聚合三个平台的已连接设备列表

```pseudo
function list_available_devices():
    devices = []

    // 1. Android 设备
    for id in adb("devices").parse_online_lines():
        features = adb(id, "shell pm list features")
        type = features.has("android.software.leanback") ? "tv" : "mobile"
        version = adb(id, "shell getprop ro.build.version.release")
        name = adb(id, "shell getprop ro.boot.qemu.avd_name")
              || adb(id, "shell getprop ro.product.model")
        devices.add({id, name, platform: "android", type, version})

    // 2. iOS 物理设备
    for id in go_ios("list").deviceList:
        info = go_ios("info --udid", id)
        devices.add({id, info.DeviceName, platform: "ios", type: "real", version: info.ProductVersion})

    // 3. iOS 模拟器
    for d in mobilecli("devices --platform ios --type simulator --exclude-offline"):
        devices.add(d)

    return JSON.stringify({ devices })
```

### 5.2 mobile_take_screenshot

**功能：** 截取屏幕并以 base64 图片返回给 LLM

```pseudo
function take_screenshot(device):
    robot = getRobotFromDevice(device)

    // ---- Android 路径 ----
    display_count = parse(adb("shell dumpsys SurfaceFlinger --display-id")).count
    if display_count <= 1:
        png = adb("exec-out screencap -p")
    else:
        display_id = try_parse_active_display()
            // 优先: adb shell cmd display get-displays → 解析 uniqueId
            // 降级: adb shell dumpsys display → 匹配 DisplayViewport
        png = adb("exec-out screencap -p -d", display_id)

    // ---- iOS 路径 ----
    // WDA: GET http://localhost:8100/screenshot → json.value (base64) → Buffer

    // ---- 模拟器路径 ----
    // mobilecli screenshot --device {id} --format png --output -

    // ---- 通用后处理 ----
    validate_png_signature(png)
    dimensions = read_png_header(png)  // 偏移 16 读 width, 偏移 20 读 height

    if sips_available || imagemagick_available:
        compressed = resize(png, width / scale).jpeg(quality=75).toBuffer()
        return { type: "image", data: base64(compressed), mimeType: "image/jpeg" }
    else:
        return { type: "image", data: base64(png), mimeType: "image/png" }
```

### 5.3 mobile_save_screenshot

**功能：** 截屏并保存到指定文件路径

```pseudo
function save_screenshot(device, saveTo):
    validate_file_extension(saveTo, [".png", ".jpg", ".jpeg"])
    validate_output_path(saveTo)  // 只允许 tmpdir 和 cwd 下
    png = robot.getScreenshot()
    fs.writeFileSync(saveTo, png)
    return "Screenshot saved to: {saveTo}"
```

### 5.4 mobile_click_on_screen_at_coordinates

**功能：** 在指定坐标点击

```pseudo
function click(device, x, y):
    robot = getRobotFromDevice(device)
    // Android: adb shell input tap {x} {y}
    // iOS WDA: POST /session/{sid}/actions
    //   → [pointerMove(0ms, x,y), pointerDown, pause(100ms), pointerUp]
    // 模拟器: mobilecli io tap {x},{y}
    robot.tap(x, y)
```

### 5.5 mobile_double_tap_on_screen

**功能：** 在指定坐标双击

```pseudo
function double_tap(device, x, y):
    // Android: tap(x,y) → sleep(100ms) → tap(x,y)
    //   即两次 adb shell input tap
    // iOS WDA: 单次 POST /session/{sid}/actions
    //   → [pointerDown, pause(50ms), pointerUp, pause(100ms), pointerDown, pause(50ms), pointerUp]
    // 模拟器: tap(x,y) → tap(x,y)
    robot.doubleTap(x, y)
```

### 5.6 mobile_long_press_on_screen_at_coordinates

**功能：** 长按指定坐标

```pseudo
function long_press(device, x, y, duration=500):
    // Android: adb shell input swipe {x} {y} {x} {y} {duration}
    //   关键技巧：起点=终点的 swipe 等于长按
    // iOS WDA: POST /session/{sid}/actions
    //   → [pointerMove(x,y), pointerDown, pause(duration), pointerUp]
    // 模拟器: mobilecli io longpress {x},{y} --duration {duration}
    robot.longPress(x, y, duration)
```

### 5.7 mobile_swipe_on_screen

**功能：** 滑动屏幕，支持从中心或指定坐标开始

```pseudo
function swipe(device, direction, x?, y?, distance?):
    robot = getRobotFromDevice(device)

    if x != null && y != null:
        // 坐标滑动模式
        screenSize = robot.getScreenSize()
        defaultDist = screenSize.height * 0.3  // Android 默认 30% 屏幕
        dist = distance || defaultDist
        (x1, y1) = offset(x, y, direction, dist)
        // Android: adb shell input swipe {x} {y} {x1} {y1} 1000
        // iOS WDA: POST actions → pointerMove(start) + pointerDown + pointerMove(end, 1000ms) + pointerUp
        robot.swipeFromCoordinate(x, y, direction, distance)
    else:
        // 中心滑动模式
        // Android: 从屏幕 20% 到 80% 范围
        // iOS WDA: 从 center ± 30% 范围, duration=1000ms
        robot.swipe(direction)
```

### 5.8 mobile_type_keys

**功能：** 向当前焦点元素输入文本

```pseudo
function type_keys(device, text, submit):
    robot = getRobotFromDevice(device)

    // ---- Android 路径 ----
    if text == "": return  // 空字符串直接返回
    if is_ascii(text):
        escaped = escape_shell_chars(text)  // 转义 '"` |&;()<>{}[]$*? 等
        adb("shell input text", escaped)
    else:
        if is_devicekit_installed():  // 检查 com.mobilenext.devicekit
            base64 = Buffer.from(text).toString("base64")
            adb("shell am broadcast -a devicekit.clipboard.set"
                "-e encoding base64 -e text {base64}"
                "-n com.mobilenext.devicekit/.ClipboardBroadcastReceiver")
            adb("shell input keyevent KEYCODE_PASTE")
            adb("shell am broadcast -a devicekit.clipboard.clear ...")
        else:
            throw "Non-ASCII not supported, install DeviceKit"

    // ---- iOS WDA 路径 ----
    // POST /session/{sid}/wda/keys → { value: [text] }

    // ---- 模拟器路径 ----
    // mobilecli io text {text}

    if submit:
        robot.pressButton("ENTER")
```

### 5.9 mobile_press_button

**功能：** 模拟物理按键

```pseudo
function press_button(device, button):
    // ---- Android ----
    BUTTON_MAP = {
        HOME   → KEYCODE_HOME,
        BACK   → KEYCODE_BACK,
        ENTER  → KEYCODE_ENTER,
        VOLUME_UP   → KEYCODE_VOLUME_UP,
        VOLUME_DOWN → KEYCODE_VOLUME_DOWN,
        DPAD_CENTER → KEYCODE_DPAD_CENTER,  // Android TV
        DPAD_UP/DOWN/LEFT/RIGHT → KEYCODE_DPAD_*
    }
    adb("shell input keyevent", BUTTON_MAP[button])

    // ---- iOS WDA ----
    if button == "ENTER":
        wda.sendKeys("\n")  // POST /session/{sid}/wda/keys
    else:
        wda.pressButton(button)  // POST /session/{sid}/wda/pressButton

    // ---- 模拟器 ----
    // mobilecli io button {button}
```

### 5.10 mobile_list_elements_on_screen

**功能：** 列出屏幕上可交互的 UI 元素

```pseudo
function list_elements(device):
    robot = getRobotFromDevice(device)

    // ---- Android ----
    for tries in 0..9:  // 最多重试 10 次
        xml = adb("exec-out uiautomator dump /dev/tty")
        if "null root node" in xml: continue
        break

    parsed = XMLParser.parse(xml, { attributeNamePrefix: "" })
    elements = recursive_collect(parsed.hierarchy.node):
        // 遍历条件：node.text || node.content-desc || node.hint || node.resource-id || node.checkable
        if match:
            rect = parse_bounds("[left,top][right,bottom]")
            if rect.width > 0 && rect.height > 0:
                yield {
                    type: node.class,
                    text: node.text,
                    label: node["content-desc"] || node.hint,
                    identifier: node["resource-id"],
                    rect: { x: left, y: top, width, height },
                    focused: node.focused == "true" ? true : undefined
                }

    // ---- iOS WDA ----
    // GET http://localhost:8100/source/?format=json
    // 递归过滤 sourceTree：
    //   acceptedTypes = [TextField, Button, Switch, Icon, SearchField, StaticText, Image]
    //   只取 isVisible == "1" 且 rect 在屏幕内的元素

    // ---- 模拟器 ----
    // mobilecli dump ui → JSON → elements 数组

    return JSON.stringify(elements)
```

### 5.11 mobile_list_apps

**功能：** 列出设备上已安装的可启动 App

```pseudo
function list_apps(device):
    // Android:
    output = adb("shell cmd package query-activities -a android.intent.action.MAIN -c android.intent.category.LAUNCHER")
    packages = output.lines
        .filter(starts_with("packageName="))
        .map(extract_value)
        .deduplicate()
    return packages.map(p => { packageName: p, appName: p })

    // iOS 真机: go-ios apps --all --list → 按行解析 "bundleId appName"
    // iOS 模拟器 (Simctl): xcrun simctl listapps {uuid} → plutil 转 JSON
    // 模拟器 (MobileDevice): mobilecli apps list → JSON
```

### 5.12 mobile_launch_app

**功能：** 启动指定 App

```pseudo
function launch_app(device, packageName, locale?):
    validatePackageName(packageName)  // 正则: /^[a-zA-Z0-9._]+$/

    // Android:
    if locale:
        adb("shell cmd locale set-app-locales {pkg} --locales {locale}")  // Android 13+
    adb("shell monkey -p {pkg} -c android.intent.category.LAUNCHER 1")

    // iOS 真机: go-ios launch {bundleId} [-AppleLanguages (...) -AppleLocale ...]
    // 模拟器:   mobilecli apps launch {pkg} [--locale locale]
```

### 5.13 mobile_terminate_app

**功能：** 强制停止 App

```pseudo
function terminate_app(device, packageName):
    validatePackageName(packageName)
    // Android: adb shell am force-stop {packageName}
    // iOS:     go-ios kill {bundleId}
    // 模拟器:  mobilecli apps terminate {packageName}
```

### 5.14 mobile_install_app

**功能：** 安装 App（支持 .apk / .ipa / .zip / .app）

```pseudo
function install_app(device, path):
    // Android: adb install -r {path}
    // iOS 真机: go-ios install --path {path}
    // iOS 模拟器 (Simctl):
    //   if .zip: validateZipPaths(防 zip-slip) → unzip → 找 .app bundle
    //   simctl install {uuid} {path}
    // 模拟器 (MobileDevice): mobilecli apps install {path}
```

### 5.15 mobile_uninstall_app

```pseudo
function uninstall_app(device, bundle_id):
    // Android: adb uninstall {bundle_id}
    // iOS:     go-ios uninstall --bundleid {bundle_id}
    // 模拟器:  mobilecli apps uninstall {bundle_id}
```

### 5.16 mobile_open_url

**功能：** 在设备浏览器打开 URL

```pseudo
function open_url(device, url):
    if !MOBILEMCP_ALLOW_UNSAFE_URLS && !url.startsWith("http"):
        throw "Only http/https allowed"

    // Android: adb shell am start -a android.intent.action.VIEW -d {escaped_url}
    //   escapeShellText: 转义 \\'"` \t\n\r|&;()<>{}[]$*?
    // iOS WDA: POST /session/{sid}/url { url }
    // 模拟器:  mobilecli url {url}
```

### 5.17 mobile_get_screen_size

```pseudo
function get_screen_size(device):
    // Android: adb shell wm size → 解析 "Physical size: 1080x2340"
    //   返回 { width: 1080, height: 2340, scale: 1 }
    // iOS WDA: GET /session/{sid}/wda/screen → { screenSize: {w,h}, scale }
    // 模拟器:  mobilecli device info → screenSize 字段
```

### 5.18 mobile_set_orientation / mobile_get_orientation

```pseudo
function set_orientation(device, orientation):
    // Android:
    adb("shell settings put system accelerometer_rotation 0")
    value = (orientation == "portrait") ? 0 : 1
    adb("shell content insert --uri content://settings/system"
        "--bind name:s:user_rotation --bind value:i:{value}")

    // iOS WDA: POST /session/{sid}/orientation { orientation: "PORTRAIT"|"LANDSCAPE" }
    // 模拟器:  mobilecli device orientation set {orientation}

function get_orientation(device):
    // Android: adb shell settings get system user_rotation → "0" or "1"
    // iOS WDA: GET /session/{sid}/orientation
    // 模拟器:  mobilecli device orientation get
```

### 5.19 mobile_start_screen_recording / mobile_stop_screen_recording

**功能：** 录屏（仅通过 mobilecli，不直接使用 adb）

```pseudo
function start_recording(device, output?, timeLimit?):
    if activeRecordings.has(device):
        throw "Already recording"
    outputPath = output || os.tmpdir()/screen-recording-{timestamp}.mp4
    child = mobilecli.spawn(
        "screenrecord --device {device} --output {path} --silent [--time-limit N]"
    )
    activeRecordings.set(device, { process: child, outputPath, startedAt: now() })
    return "Recording started. Output: {outputPath}"

function stop_recording(device):
    recording = activeRecordings.get(device)
    if !recording: throw "No active recording"

    recording.process.kill("SIGINT")  // 优雅终止
    wait_for_exit(timeout: 5min)      // 超时则 SIGKILL
    activeRecordings.delete(device)

    stat = fs.stat(outputPath)
    return "File: {path} ({size}MB, ~{duration}s)"
```

### 5.20 Fleet 相关 Tools（MOBILEFLEET_ENABLE=1 时启用）

```pseudo
// mobile_list_fleet_devices
function list_fleet(): mobilecli("fleet list-devices")

// mobile_allocate_fleet_device
function allocate(platform): mobilecli("fleet allocate --platform {platform}")

// mobile_release_fleet_device
function release(device): mobilecli("fleet release --device {device}")
```

---

## 六、关键设计思想

### 6.1 Robot 接口模式（策略模式）

`robot.ts` 定义了 16 个标准方法的接口，所有平台实现（Android/iOS/Simulator）都遵循同一接口。`server.ts` 中的 Tool 层完全面向接口编程，不关心底层是 adb 还是 WDA。

### 6.2 同步执行模型

Android 侧全部使用 `execFileSync` 同步调用 adb，超时 30 秒，最大 buffer 8MB。优点是代码简洁且无需处理异步竞态；缺点是阻塞 Node.js 事件循环。

### 6.3 安全防线

- **命令注入防护**：`escapeShellText()` 转义所有 shell 特殊字符（android.ts:406）
- **包名白名单**：`validatePackageName()` 只允许 `[a-zA-Z0-9._]`（utils.ts:7）
- **Locale 白名单**：`validateLocale()` 只允许 `[a-zA-Z0-9,- ]`（utils.ts:13）
- **路径安全**：`validateOutputPath()` 限制输出路径只在 tmpdir 和 cwd 下（utils.ts:69）
- **URL 协议限制**：默认只允许 http/https，需 `MOBILEMCP_ALLOW_UNSAFE_URLS=1` 才能放开
- **Zip-Slip 防护**：iOS 模拟器安装 .zip 时验证路径无 `..` 和绝对路径（iphone-simulator.ts:136）

### 6.4 截图压缩优化

截图是 LLM 交互的核心通道，代码做了多级优化：
1. 多屏设备只截活跃屏幕（非所有屏幕拼接）
2. 有 sips (macOS) 或 ImageMagick 时，自动缩放 + 转 JPEG quality=75
3. 最终 base64 编码返回

### 6.5 错误分级

- `ActionableError`：用户可修复的错误（如 App 未安装、WDA 未运行），返回提示信息
- 普通 `Error`：系统级异常，返回 `isError: true`

---

## 七、值得借鉴的设计

1. **Robot 接口抽象** — 用一个轻量接口统一多平台差异，Tool 层零耦合
2. **原子命令封装** — `adb()` 方法只做一件事：执行命令返回 Buffer，所有高级逻辑在上层组合
3. **防注入即默认** — 包名、路径、URL 全部在入口处校验，不信任任何外部输入
4. **降级策略** — Display ID 解析有三级降级；截图压缩有 sips → ImageMagick → 原图降级
5. **uiautomator 重试** — dump 偶尔返回 "null root node"，自动重试 10 次

---

## 附：文件索引

| 文件 | 职责 | 行数 |
|------|------|------|
| `src/server.ts` | MCP Tool 注册、参数校验、结果格式化 | ~800 |
| `src/robot.ts` | Robot 接口定义 | ~148 |
| `src/android.ts` | Android ADB 实现 | ~611 |
| `src/ios.ts` | iOS 真机实现（go-ios + WDA） | ~304 |
| `src/iphone-simulator.ts` | iOS 模拟器实现（simctl + WDA） | ~283 |
| `src/mobile-device.ts` | 通用设备实现（mobilecli） | ~221 |
| `src/webdriver-agent.ts` | WDA HTTP 客户端 | ~454 |
| `src/mobilecli.ts` | mobilecli 二进制包装 | ~154 |
| `src/image-utils.ts` | 图片缩放（sips / ImageMagick） | ~164 |
| `src/png.ts` | PNG 头解析 | ~20 |
| `src/utils.ts` | 安全校验工具 | ~88 |
| `src/logger.ts` | 日志 | ~22 |
| `src/index.ts` | 入口（stdio / SSE 两种传输） | ~121 |
