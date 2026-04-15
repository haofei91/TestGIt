# mobile-mcp 代码库分析报告

## 一、整体架构

```
┌─────────────────────────────────────────────────────┐
│                  MCP Server (server.ts)              │
│           注册 20+ Tool，统一暴露给 AI Agent          │
├─────────────────────────────────────────────────────┤
│              Robot 接口 (robot.ts)                    │
│     定义统一抽象：tap/swipe/screenshot/listApps/...   │
├──────────┬──────────────┬───────────────────────────┤
│ Android  │   iOS Real   │  iOS Simulator / 通用     │
│ Robot    │   Robot      │  MobileDevice             │
│(adb直连) │(go-ios+WDA)  │ (mobilecli CLI)           │
└──────────┴──────────────┴───────────────────────────┘
```

**三种设备驱动：**
- `AndroidRobot` — 直接调用 `adb` 命令
- `IosRobot` — 调用 `go-ios` CLI + WebDriverAgent HTTP API
- `MobileDevice` — 调用 `mobilecli` 二进制 CLI（主要用于 iOS 模拟器）

## 二、原子 ADB 命令清单（Android 侧）

`AndroidRobot` 类中所有 `adb shell ...` 调用即为"原子命令层"。完整列举如下：

| # | 原子 ADB 命令 | 用途 | 源码位置 |
|---|---|---|---|
| 1 | `adb shell wm size` | 获取屏幕分辨率 | `android.ts:105` |
| 2 | `adb shell input tap {x} {y}` | 点击屏幕坐标 | `android.ts:452` |
| 3 | `adb shell input swipe {x0} {y0} {x1} {y1} {duration}` | 滑动手势 | `android.ts:202` |
| 4 | `adb shell input text {text}` | 输入ASCII文本 | `android.ts:426` |
| 5 | `adb shell input keyevent {KEYCODE}` | 按键事件 (HOME/BACK/ENTER等) | `android.ts:448` |
| 6 | `adb exec-out screencap -p [-d displayId]` | 截屏 (PNG) | `android.ts:310-321` |
| 7 | `adb exec-out uiautomator dump /dev/tty` | 导出UI层级XML | `android.ts:481` |
| 8 | `adb shell pm list features` | 列出系统特性 | `android.ts:96` |
| 9 | `adb shell pm list packages` | 列出所有已安装包 | `android.ts:135` |
| 10 | `adb shell cmd package query-activities -a MAIN -c LAUNCHER` | 列出可启动App | `android.ts:121` |
| 11 | `adb shell monkey -p {pkg} -c LAUNCHER 1` | 启动App | `android.ts:156` |
| 12 | `adb shell am force-stop {pkg}` | 强制停止App | `android.ts:372` |
| 13 | `adb shell am start -a VIEW -d {url}` | 打开URL | `android.ts:398` |
| 14 | `adb shell am broadcast -a devicekit.clipboard.set ...` | 通过DeviceKit设置剪贴板(非ASCII文本) | `android.ts:432` |
| 15 | `adb shell am broadcast -a devicekit.clipboard.clear ...` | 清除剪贴板 | `android.ts:436` |
| 16 | `adb install -r {path}` | 安装APK | `android.ts:377` |
| 17 | `adb uninstall {pkg}` | 卸载App | `android.ts:389` |
| 18 | `adb shell settings put system accelerometer_rotation 0` | 关闭自动旋转 | `android.ts:470` |
| 19 | `adb shell content insert --uri settings/system --bind user_rotation` | 设置屏幕方向 | `android.ts:471` |
| 20 | `adb shell settings get system user_rotation` | 获取屏幕方向 | `android.ts:475` |
| 21 | `adb shell cmd locale set-app-locales {pkg} --locales {locale}` | 设置App语言 | `android.ts:149` |
| 22 | `adb shell cmd display get-displays` | 获取显示器列表 | `android.ts:255` |
| 23 | `adb shell dumpsys SurfaceFlinger --display-id` | 获取Display数量 | `android.ts:245` |
| 24 | `adb shell dumpsys display` | 获取Display信息 (fallback) | `android.ts:281` |
| 25 | `adb shell ps -e` | 列出运行中的进程 | `android.ts:163` |
| 26 | `adb shell getprop ro.build.version.release` | 获取Android版本 | `android.ts:537` |
| 27 | `adb shell getprop ro.boot.qemu.avd_name` | 获取模拟器AVD名称 | `android.ts:549` |
| 28 | `adb shell getprop ro.product.model` | 获取设备型号 | `android.ts:559` |
| 29 | `adb devices` | 列出已连接设备 | `android.ts:570` |

## 三、每个 MCP Tool 的内部逻辑与核心伪代码

### 1. `mobile_list_available_devices`

**功能：** 列出所有可用设备（Android + iOS 物理机 + iOS 模拟器）

```pseudo
function list_available_devices():
    devices = []

    // Android: adb devices -> 解析在线设备 -> getprop获取详情
    for id in adb("devices").parse_online():
        type = adb(id, "shell pm list features").has("leanback") ? "tv" : "mobile"
        version = adb(id, "shell getprop ro.build.version.release")
        name = adb(id, "shell getprop ro.boot.qemu.avd_name") || adb(id, "shell getprop ro.product.model")
        devices.add({id, name, platform:"android", version})

    // iOS物理机: go-ios list -> go-ios info
    for id in go_ios("list").deviceList:
        info = go_ios("info --udid", id)
        devices.add({id, info.DeviceName, platform:"ios", type:"real", version: info.ProductVersion})

    // iOS模拟器: mobilecli devices --platform ios --type simulator
    for d in mobilecli("devices --platform ios --type simulator"):
        devices.add(d)

    return devices
```

### 2. `mobile_take_screenshot`

**功能：** 截取设备屏幕，返回图片给 AI

```pseudo
function take_screenshot(device):
    robot = get_robot(device)

    // Android路径：
    if display_count <= 1:
        png_buffer = adb("exec-out screencap -p")
    else:
        display_id = parse_first_active_display()  // cmd display get-displays 或 dumpsys display
        png_buffer = adb("exec-out screencap -p -d", display_id)

    // iOS路径：WDA HTTP GET /screenshot -> base64解码
    // 模拟器路径：mobilecli screenshot --format png --output -

    // 通用后处理
    validate_png(png_buffer)
    if sharp_available:
        jpeg_buffer = resize_and_compress(png_buffer, quality=75)
    return base64_encode(buffer)
```

### 3. `mobile_click_on_screen_at_coordinates`

**功能：** 在指定坐标点击

```pseudo
function click(device, x, y):
    robot = get_robot(device)

    // Android: adb shell input tap {x} {y}
    // iOS:     WDA POST /session/{id}/actions -> pointerMove+pointerDown+pause(100ms)+pointerUp
    // 模拟器:  mobilecli io tap {x},{y}

    robot.tap(x, y)
```

### 4. `mobile_double_tap_on_screen`

**功能：** 在指定坐标双击

```pseudo
function double_tap(device, x, y):
    // Android: tap(x,y) -> sleep(100ms) -> tap(x,y)  (两次adb shell input tap)
    // iOS WDA: 单次actions请求，包含两组 down/up 序列，中间 pause(100ms)
    // 模拟器:  tap(x,y) -> tap(x,y)  (两次mobilecli io tap)
    robot.doubleTap(x, y)
```

### 5. `mobile_long_press_on_screen_at_coordinates`

**功能：** 长按指定坐标

```pseudo
function long_press(device, x, y, duration=500):
    // Android: adb shell input swipe {x} {y} {x} {y} {duration}  // 原地滑动=长按
    // iOS WDA: POST /session/{id}/actions -> pointerDown + pause(duration) + pointerUp
    // 模拟器:  mobilecli io longpress {x},{y} --duration {duration}
    robot.longPress(x, y, duration)
```

### 6. `mobile_swipe_on_screen`

**功能：** 滑动屏幕

```pseudo
function swipe(device, direction, x?, y?, distance?):
    if x && y:  // 从指定坐标开始滑
        screen_size = robot.getScreenSize()
        (x0, y0) = (x, y)
        (x1, y1) = calculate_endpoint(x, y, direction, distance || 30%_screen)
        // Android: adb shell input swipe {x0} {y0} {x1} {y1} 1000
        // iOS WDA: POST /session/{id}/actions -> pointerMove(start) + pointerDown + pointerMove(end, 1000ms) + pointerUp
    else:  // 从屏幕中心滑
        center = screen_size / 2
        // Android: 从 20%~80% 屏幕范围滑动
        // iOS WDA: 从 center+-30%范围 滑动，duration=1000ms
```

### 7. `mobile_type_keys`

**功能：** 在当前焦点元素输入文本

```pseudo
function type_keys(device, text, submit):
    // Android:
    if is_ascii(text):
        escaped = escape_shell_special_chars(text)
        adb("shell input text", escaped)
    else:
        if devicekit_installed:
            base64 = encode_base64(text)
            adb("shell am broadcast -a devicekit.clipboard.set -e encoding base64 -e text", base64)
            adb("shell input keyevent KEYCODE_PASTE")
            adb("shell am broadcast -a devicekit.clipboard.clear")
        else:
            throw "Non-ASCII not supported without devicekit"

    // iOS WDA: POST /session/{id}/wda/keys -> { value: [text] }
    // 模拟器:  mobilecli io text {text}

    if submit:
        robot.pressButton("ENTER")  // adb shell input keyevent KEYCODE_ENTER
```

### 8. `mobile_press_button`

**功能：** 模拟物理按键

```pseudo
function press_button(device, button):
    // Android: BUTTON_MAP = { HOME->KEYCODE_HOME, BACK->KEYCODE_BACK, ENTER->KEYCODE_ENTER, ... }
    //          adb("shell input keyevent", BUTTON_MAP[button])
    //
    // iOS WDA: if button == "ENTER" -> sendKeys("\n")
    //          else -> POST /session/{id}/wda/pressButton { name: button }
    //
    // 模拟器:  mobilecli io button {button}
    robot.pressButton(button)
```

### 9. `mobile_list_elements_on_screen`

**功能：** 列出屏幕上可交互的UI元素及坐标

```pseudo
function list_elements(device):
    // Android:
    xml = adb("exec-out uiautomator dump /dev/tty")  // 最多重试10次
    parsed = XMLParser.parse(xml)
    elements = recursive_collect(parsed.hierarchy.node):
        if node.text || node.content-desc || node.hint || node.resource-id || node.checkable:
            rect = parse_bounds("[left,top][right,bottom]")
            if rect.width > 0 && rect.height > 0:
                yield {type: node.class, text, label, identifier: resource-id, rect, focused}

    // iOS WDA: GET /source/?format=json -> 递归过滤
    //          只保留 TextField/Button/Switch/Icon/SearchField/StaticText/Image 类型
    //          且 isVisible=="1"

    // 模拟器: mobilecli dump ui -> JSON直接返回
    return elements
```

### 10. `mobile_list_apps`

**功能：** 列出设备上已安装的App

```pseudo
function list_apps(device):
    // Android: adb shell cmd package query-activities -a MAIN -c LAUNCHER
    //          解析输出中 "packageName=xxx" 行，去重
    //
    // iOS:     go-ios apps --all --list -> 解析 "bundleId appName" 格式
    //
    // 模拟器:  mobilecli apps list -> JSON解析
    return robot.listApps()
```

### 11. `mobile_launch_app`

**功能：** 启动指定App

```pseudo
function launch_app(device, packageName, locale?):
    validate_package_name(packageName)  // 防注入：只允许 [a-zA-Z0-9._]

    // Android:
    if locale:
        adb("shell cmd locale set-app-locales", packageName, "--locales", locale)  // Android 13+
    adb("shell monkey -p", packageName, "-c android.intent.category.LAUNCHER 1")

    // iOS:    go-ios launch {bundleId} [-AppleLanguages (...) -AppleLocale ...]
    // 模拟器: mobilecli apps launch {packageName} [--locale locale]
```

### 12. `mobile_terminate_app`

**功能：** 强制停止App

```pseudo
function terminate_app(device, packageName):
    validate_package_name(packageName)
    // Android: adb shell am force-stop {packageName}
    // iOS:     go-ios kill {bundleId}
    // 模拟器:  mobilecli apps terminate {packageName}
```

### 13. `mobile_install_app`

**功能：** 安装App

```pseudo
function install_app(device, path):
    // Android: adb install -r {path}   (.apk文件)
    // iOS:     go-ios install --path {path}   (.ipa文件)
    // 模拟器:  mobilecli apps install {path}  (.zip/.app)
```

### 14. `mobile_uninstall_app`

**功能：** 卸载App

```pseudo
function uninstall_app(device, bundle_id):
    // Android: adb uninstall {bundle_id}
    // iOS:     go-ios uninstall --bundleid {bundle_id}
    // 模拟器:  mobilecli apps uninstall {bundle_id}
```

### 15. `mobile_open_url`

**功能：** 在设备浏览器中打开URL

```pseudo
function open_url(device, url):
    if !ALLOW_UNSAFE_URLS && !url.startsWith("http"):
        throw "Only http/https allowed"

    // Android: adb shell am start -a android.intent.action.VIEW -d {escaped_url}
    // iOS WDA: POST /session/{id}/url { url: url }
    // 模拟器:  mobilecli url {url}
```

### 16. `mobile_get_screen_size`

**功能：** 获取屏幕分辨率

```pseudo
function get_screen_size(device):
    // Android: adb shell wm size -> 解析 "Physical size: 1080x2340"
    // iOS WDA: GET /session/{id}/wda/screen -> { screenSize, scale }
    // 模拟器:  mobilecli device info -> screenSize字段
    return { width, height, scale }
```

### 17. `mobile_set_orientation` / `mobile_get_orientation`

**功能：** 设置/获取屏幕方向

```pseudo
function set_orientation(device, orientation):  // "portrait" | "landscape"
    // Android:
    adb("shell settings put system accelerometer_rotation 0")  // 关闭自动旋转
    value = orientation == "portrait" ? 0 : 1
    adb("shell content insert --uri content://settings/system --bind name:s:user_rotation --bind value:i:{value}")

    // iOS WDA: POST /session/{id}/orientation { orientation: "PORTRAIT"|"LANDSCAPE" }
    // 模拟器:  mobilecli device orientation set {orientation}

function get_orientation(device):
    // Android: adb shell settings get system user_rotation -> "0"=portrait, "1"=landscape
    // iOS WDA: GET /session/{id}/orientation
    // 模拟器:  mobilecli device orientation get
```

### 18. `mobile_save_screenshot`

**功能：** 截屏并保存到指定文件

```pseudo
function save_screenshot(device, saveTo):
    validate_extension(saveTo, [".png", ".jpg", ".jpeg"])
    validate_output_path(saveTo)  // 安全检查
    buffer = robot.getScreenshot()  // 同 take_screenshot 的底层逻辑
    fs.writeFile(saveTo, buffer)
```

### 19. `mobile_start_screen_recording` / `mobile_stop_screen_recording`

**功能：** 录屏（通过 mobilecli 子进程实现）

```pseudo
function start_recording(device, output?, timeLimit?):
    if activeRecordings.has(device): throw "already recording"
    outputPath = output || tempdir/screen-recording-{timestamp}.mp4
    child = mobilecli.spawn("screenrecord --device {device} --output {path} --silent [--time-limit N]")
    activeRecordings.set(device, { child, outputPath, startedAt })

function stop_recording(device):
    recording = activeRecordings.get(device)
    recording.process.kill("SIGINT")     // 优雅终止
    wait_for_exit(timeout=5min)          // 超时则 SIGKILL
    return { path, fileSize, duration }
```

### 20. `mobile_list_fleet_devices` / `mobile_allocate_fleet_device` / `mobile_release_fleet_device`

**功能：** 远程设备池管理（需要 `MOBILEFLEET_ENABLE=1`）

```pseudo
function list_fleet():    mobilecli("fleet list-devices")
function allocate(platform):  mobilecli("fleet allocate --platform {platform}")
function release(device):     mobilecli("fleet release --device {device}")
```

## 四、原子命令到 Tool 的组合关系图

```
原子ADB命令                          MCP Tool
────────────                         ────────
adb devices + getprop x3 ──────────→ mobile_list_available_devices
adb shell wm size ─────────────────→ mobile_get_screen_size
adb exec-out screencap -p ─────────→ mobile_take_screenshot / mobile_save_screenshot
adb shell input tap ───────────────→ mobile_click_on_screen_at_coordinates
adb shell input tap x2 ───────────→ mobile_double_tap_on_screen
adb shell input swipe (原地) ──────→ mobile_long_press_on_screen_at_coordinates
adb shell input swipe (移动) ──────→ mobile_swipe_on_screen  (需先调 wm size 计算坐标)
adb shell input text ──────────────→ mobile_type_keys (ASCII)
adb shell input keyevent ──────────→ mobile_press_button / mobile_type_keys(submit)
am broadcast + keyevent PASTE ─────→ mobile_type_keys (非ASCII，需DeviceKit)
adb exec-out uiautomator dump ─────→ mobile_list_elements_on_screen
cmd package query-activities ──────→ mobile_list_apps
adb shell monkey ──────────────────→ mobile_launch_app
cmd locale set-app-locales ────────→ mobile_launch_app (locale参数)
adb shell am force-stop ───────────→ mobile_terminate_app
adb install -r ────────────────────→ mobile_install_app
adb uninstall ─────────────────────→ mobile_uninstall_app
adb shell am start -a VIEW -d ────→ mobile_open_url
settings put/get + content insert ─→ mobile_set/get_orientation
pm list features ──────────────────→ (内部: 判断设备类型 tv/mobile)
dumpsys SurfaceFlinger/display ────→ (内部: 多屏截图时定位displayId)
```

## 五、关键设计总结

1. **统一抽象层**：`Robot` 接口定义了 16 个标准方法，Android/iOS/模拟器三种实现各自适配底层命令
2. **Android 侧的"原子命令"全部是 `adb shell` 子命令**，通过 `execFileSync` 同步调用
3. **iOS 侧有两套通道**：`go-ios` CLI（设备管理、App操作）+ `WebDriverAgent` HTTP API（UI交互、截图）
4. **Tool 层是"编排层"**，负责参数校验、坐标计算、结果格式化，底层逻辑完全委托给 Robot 实现
5. **安全措施**：`escapeShellText` 防止命令注入，`validatePackageName` 防止恶意包名，URL 默认只允许 http/https
