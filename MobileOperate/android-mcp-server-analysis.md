# android-mcp-server 源码深度分析

> 项目地址: https://github.com/martingeidobler/android-mcp-server
> 版本: 1.3.0
> 技术栈: TypeScript + MCP SDK + sharp + zod

---

## 一、架构总览

项目由两个核心文件组成：

| 文件 | 职责 |
|------|------|
| `src/adb.ts` | **ADB 原子命令封装层** — 封装所有与 `adb` 二进制交互的底层能力 |
| `src/index.ts` | **MCP Tool 扩展层** — 基于 `Adb` 类组合出面向 AI 的高级 Tool |

架构分层：

```
┌─────────────────────────────────────────────┐
│  MCP Tool 层 (index.ts)                      │
│  20 个 Tool，面向 AI Agent 的语义化接口        │
├─────────────────────────────────────────────┤
│  Adb 封装层 (adb.ts)                         │
│  Adb 类，统一 exec/shellExec/execBuffer      │
├─────────────────────────────────────────────┤
│  原子 ADB 命令层                              │
│  adb devices / adb shell / adb exec-out ...  │
└─────────────────────────────────────────────┘
```

---

## 二、原子 ADB 命令清单

整个项目基于以下 **19 个原子 ADB/Emulator 命令**：

| # | 原子命令 | 用途 | 调用位置 |
|---|---------|------|---------|
| 1 | `adb devices -l` | 列出已连接设备 | `getDevices()` |
| 2 | `adb shell input tap <x> <y>` | 屏幕点击 | `tap()` |
| 3 | `adb shell input swipe <x1> <y1> <x2> <y2> [duration]` | 滑动手势 | `swipe()` |
| 4 | `adb shell input text <text>` | 文本输入 | `typeText()` |
| 5 | `adb shell input keyevent <keycode>` | 按键事件 | `pressKey()` |
| 6 | `adb shell uiautomator dump /dev/tty` | 导出 UI 树 XML（方式1） | `getUiTree()` |
| 7 | `adb shell uiautomator dump /sdcard/window_dump.xml` | 导出 UI 树 XML（方式2，回退） | `getUiTree()` |
| 8 | `adb shell cat /sdcard/window_dump.xml` | 读取设备文件 | `getUiTree()` (回退路径) |
| 9 | `adb exec-out screencap -p` | 截屏（二进制PNG） | `screenshot()` |
| 10 | `adb shell am start -n <pkg/activity>` | 启动指定 Activity | `launchApp()` |
| 11 | `adb shell monkey -p <pkg> -c ... 1` | 启动默认 Activity | `launchApp()` |
| 12 | `adb install -r <apk>` | 安装 APK | `installApk()` |
| 13 | `adb shell dumpsys activity activities` | 获取当前 Activity | `getCurrentActivity()` |
| 14 | `adb shell getprop <prop>` | 读取系统属性 | `getDeviceInfo()` |
| 15 | `adb shell wm size` / `wm density` | 获取屏幕尺寸和密度 | `getDeviceInfo()` |
| 16 | `adb pull <remote> <local>` | 拉取文件 | `pullFile()` |
| 17 | `adb logcat -d [-t N] [-T time] [*:level]` | 读取日志 | `getLogcat()` |
| 18 | `adb logcat -c` | 清除日志缓冲区 | `clearLogcat()` |
| 19 | `adb shell pidof <pkg>` | 获取进程PID（日志过滤用） | `getLogcat()` |

**Emulator 命令**（非 adb）：

| # | 命令 | 用途 | 调用位置 |
|---|------|------|---------|
| 1 | `emulator -list-avds` | 列出虚拟设备 | `getAvds()` |
| 2 | `emulator -avd <name>` | 启动模拟器 | `startEmulator()` |

---

## 三、命令执行引擎 — 三种执行模式

`Adb` 类提供了三种不同的命令执行通道：

```
exec(args, deviceId)
├── 如果 args[0] === "shell"
│   └── 转发到 shellExec() → 持久化 shell 通道（复用连接）
│       // 通过 stdin 写入 "command; echo __MCP_DONE__"
│       // 通过 stdout 读取直到看到 __MCP_DONE__ 标记
│       // 超时: 10 秒
└── 否则
    └── execFileAsync(adbPath, fullArgs) → 一次性进程
        // 超时: 10 秒, maxBuffer: 50MB

execBuffer(args, deviceId)
└── execFile() → 返回 Buffer（用于二进制数据如截图）
    // 超时: 10 秒, maxBuffer: 50MB
```

### 持久化 Shell 的关键设计

通过 `spawn("adb", ["shell"])` 保持一个长连接 shell 进程，用 `__MCP_DONE__` 作为命令完成的标记符，避免每次 `adb shell` 都创建新进程，显著降低延迟。

```pseudo
getPersistentShell(deviceId):
    key = deviceId ?? "__default__"
    if 缓存中已有且未退出:
        return 已有 shell

    proc = spawn(adbPath, ["-s", deviceId, "shell"])

    proc.stdout.on("data"):
        buffer += chunk
        if buffer 包含 "__MCP_DONE__":
            output = buffer 截取到标记之前
            resolve(output)

    proc.on("close"):
        清理缓存, reject 所有 pending 请求

shellExec(command, deviceId):
    shell = getPersistentShell(deviceId)
    shell.stdin.write("command; echo __MCP_DONE__\n")
    等待 stdout 中出现 __MCP_DONE__ (超时 10 秒)
```

### 工具路径发现机制

```pseudo
discoverPath(toolRelativePath, fallbackName):
    1. 优先: $ANDROID_HOME/{toolRelativePath}
    2. 其次: ~/Library/Android/sdk/{toolRelativePath}
    3. 兜底: 直接使用命令名（依赖 PATH）
```

---

## 四、全部 20 个 MCP Tool 详解（含核心伪代码）

### 4.1 设备管理类（3个）

#### `list_devices` — 列出已连接设备

```pseudo
function list_devices():
    output = adb devices -l
    解析每行: "emulator-5554  device  model:Pixel_6"
    返回 [{id, state, model}, ...]
```

#### `list_avds` — 列出可用虚拟设备

```pseudo
function list_avds():
    output = emulator -list-avds
    按行分割返回 AVD 名称列表
```

#### `start_emulator` — 启动模拟器

```pseudo
function start_emulator(avd_name):
    before = adb devices -l          // 记录当前设备列表
    spawn("emulator", ["-avd", avd_name])  // detached 后台启动
    loop (最多60秒, 每2秒轮询):
        current = adb devices -l
        if 发现新设备 且 state=="device":
            return 新设备ID
    throw 超时错误
```

---

### 4.2 截屏与 UI 分析类（2个）

#### `screenshot` — 截屏

```pseudo
function screenshot(device_id?, save_path?):
    raw_png = adb exec-out screencap -p     // 获取原始 PNG Buffer
    compressed = sharp(raw_png)
        .resize(maxWidth=1280)              // 超宽则缩放
        .png(quality=80)                    // 压缩
        .toBuffer()
    base64 = compressed.toString("base64")
    if save_path:
        mkdir(dirname(save_path))
        writeFile(save_path, compressed)
    返回 {image: base64, text: "WxH (sizeKB)"}
```

#### `get_ui_tree` — 获取 UI 元素树

```pseudo
function get_ui_tree(device_id?):
    // 尝试方式1: 直接输出到终端（更快，无需写磁盘）
    try:
        raw = adb shell uiautomator dump /dev/tty
        xml = raw.substring(从 "<?xml" 或 "<hierarchy" 开始)
    catch:
        // 尝试方式2: 先写文件再读
        adb shell uiautomator dump /sdcard/window_dump.xml
        xml = adb shell cat /sdcard/window_dump.xml

    // 正则解析 XML 中的 <node> 标签
    for each <node> in xml:
        提取属性: resource-id, text, content-desc, class, bounds,
                  clickable, enabled, focused, checked, scrollable
        过滤: 无 resourceId/text/contentDesc 且不可点击的元素 → 跳过
        bounds = parseBounds("[left,top][right,bottom]")
             → { x: left, y: top, width: right-left, height: bottom-top }
        center = (x + width/2, y + height/2)

    返回格式化列表:
        "[0] id:xxx text:"yyy" class:Button center:(540,960) [clickable]"
```

---

### 4.3 交互操作类（8个）

#### `tap` — 坐标点击

```pseudo
function tap(x, y, device_id?):
    adb shell input tap {x} {y}
    返回 "Tapped at (x, y)"
```

#### `tap_element` — 按属性点击元素

```pseudo
function tap_element(by, value, device_id?):
    elements = get_ui_tree(device_id)

    // 元素查找策略:
    if by == "resource-id":
        match = el.resourceId == value 或 el.resourceId.endsWith(":id/{value}")
    if by == "text":
        match = el.text 完全等于 value 或 小写包含 value
    if by == "content-desc":
        match = el.contentDesc 完全等于 value 或 小写包含 value

    if not found:
        返回错误 + 列出所有 clickable 元素 (id/text/desc) 供参考

    adb shell input tap {el.center.x} {el.center.y}
    返回 "Tapped by=value at (cx, cy) [ClassName]"
```

#### `tap_and_wait` — 点击并等待 UI 刷新

```pseudo
function tap_and_wait(by, value, wait_ms=1000, device_id?):
    elements = get_ui_tree(device_id)
    找到目标元素 el (同 tap_element 查找逻辑)
    if not found: 返回错误 + 可用元素列表

    adb shell input tap {el.center.x} {el.center.y}
    sleep(wait_ms)                          // 等待 UI 稳定
    new_elements = get_ui_tree(device_id)   // 重新获取 UI 树
    返回 "点击结果 + 新的 UI 元素列表（全量）"
```

#### `type_text` — 文本输入

```pseudo
function type_text(text, device_id?):
    escaped = escapeShellText(text)
        // 转义规则:
        // 空格→%s, \→\\, '→\', "→\", `→\`
        // $→\$, &→\&, |→\|, ;→\;
        // (→\(, )→\), <→\<, >→\>
    adb shell input text {escaped}
```

#### `press_key` — 按键

```pseudo
function press_key(key, device_id?):
    KEYCODE_MAP = {
        back:4, home:3, enter:66, tab:61, delete:67,
        menu:82, recent_apps:187,
        volume_up:24, volume_down:25, power:26, search:84,
        dpad_up:19, dpad_down:20, dpad_left:21, dpad_right:22, dpad_center:23
    }
    adb shell input keyevent {KEYCODE_MAP[key]}
```

#### `swipe` — 滑动

```pseudo
function swipe(start_x, start_y, end_x, end_y, duration_ms=300, device_id?):
    adb shell input swipe {start_x} {start_y} {end_x} {end_y} {duration_ms}
```

#### `long_press` — 长按

```pseudo
function long_press(x, y, duration_ms=1000, device_id?):
    // 关键技巧: 原地 swipe = 长按
    adb shell input swipe {x} {y} {x} {y} {duration_ms}
```

#### `double_tap` — 双击

```pseudo
function double_tap(x, y, interval_ms=100, device_id?):
    adb shell input tap {x} {y}
    sleep(interval_ms)
    adb shell input tap {x} {y}
```

#### `multi_tap` — 多次点击

```pseudo
function multi_tap(x, y, count, interval_ms=100, device_id?):
    for i in 0..count:
        adb shell input tap {x} {y}
        if i < count-1: sleep(interval_ms)
```

#### `tap_sequence` — 操作序列编排

```pseudo
function tap_sequence(steps[], device_id?):
    // 支持 6 种步骤类型的任意组合
    for step in steps:
        switch step.action:
            "tap"          → adb shell input tap {x} {y}
            "tap_and_wait" → tap(x,y) + sleep(wait_ms, 默认1000)
            "type_text"    → adb shell input text {escaped_text}
            "press_key"    → adb shell input keyevent {KEYCODE_MAP[key]}
            "swipe"        → adb shell input swipe {x1} {y1} {x2} {y2} {duration}
            "pause"        → sleep(ms)
    返回执行摘要: "tap(x,y) → type_text("hi") → press_key(enter)"
```

---

### 4.4 智能等待类（2个）

#### `scroll_to_element` — 滚动直到找到元素

```pseudo
function scroll_to_element(by, value, max_scrolls=10, device_id?):
    info = getDeviceInfo(device_id)   // 获取屏幕尺寸
    [w, h] = info.screenSize.split("x")
    centerX = w / 2
    fromY = h * 0.7    // 从屏幕 70% 处开始
    toY = h * 0.3      // 滑到屏幕 30% 处（向上滚动）

    for i in 0..max_scrolls:
        elements = get_ui_tree(device_id)
        if 找到匹配元素(by, value):
            return "找到元素, 在第{i}次滚动后, 坐标(cx, cy)"

        // 执行一次向上滚动
        adb shell input swipe {centerX} {fromY} {centerX} {toY} 500
        sleep(500)   // 等待滚动动画完成

    return 错误: "{by}={value} 在 {max_scrolls} 次滚动后未找到"
```

#### `wait_for_element` — 等待元素出现

```pseudo
function wait_for_element(by, value, timeout_ms=10000, device_id?):
    start = now()
    while (now() - start < timeout_ms):
        try:
            elements = get_ui_tree(device_id)
            if 找到匹配元素(by, value):
                return "元素出现, 耗时 {elapsed}ms, 坐标(cx, cy)"
        catch:
            pass   // UI dump 在页面转场时可能失败，静默忽略

        sleep(500)   // 每 500ms 轮询一次

    return 超时错误: "{by}={value} 在 {timeout_ms}ms 内未出现"
```

---

### 4.5 应用管理类（4个）

#### `launch_app` — 启动应用

```pseudo
function launch_app(package_name, activity?, device_id?):
    if activity 指定:
        adb shell am start -n {package_name}/{activity}
    else:
        // 无 activity 时使用 monkey 启动默认 launcher activity
        adb shell monkey -p {package_name} -c android.intent.category.LAUNCHER 1
```

#### `install_apk` — 安装 APK

```pseudo
function install_apk(apk_path, device_id?):
    adb install -r {apk_path}
    // -r: 替换已有安装（保留数据）
    // 超时: 60 秒（安装可能较慢）
    // maxBuffer: 10MB
```

#### `get_current_activity` — 获取当前 Activity

```pseudo
function get_current_activity(device_id?):
    output = adb shell dumpsys activity activities
    // 用正则匹配两种格式（兼容不同 Android 版本）:
    pattern1 = /topResumedActivity=.*{空格 空格 pkg/activity}/
    pattern2 = /mResumedActivity=.*{空格 空格 pkg/activity}/
    return { packageName, activity }
```

#### `adb_shell` — 执行任意 shell 命令

```pseudo
function adb_shell(command, device_id?):
    output = adb shell {command}   // 通过持久化 shell 执行
    return output 或 "(no output)"
```

---

### 4.6 诊断与文件类（4个）

#### `get_logs` — 获取 Logcat 日志

```pseudo
function get_logs(package_name?, level?, lines=200, since?, device_id?):
    args = ["logcat", "-d"]           // -d: dump 后退出

    if since:
        args += ["-T", since]         // 按时间戳过滤
    else:
        args += ["-t", lines]         // 按行数限制

    if level:
        args += ["*:{LEVEL}"]         // 如 *:E 只看 Error 及以上

    output = adb {args}               // 注意: logcat 不走 shell

    // 包名过滤: 先查 PID, 再按 PID 过滤日志行
    if package_name:
        pids = adb shell pidof {package_name}
        pidSet = Set(pids.split(" "))
        output = output 每行匹配 /^\S+\s+\S+\s+(\d+)\s/
                 只保留 PID 在 pidSet 中的行

    return output
```

#### `clear_logs` — 清除日志缓冲区

```pseudo
function clear_logs(device_id?):
    adb logcat -c
```

#### `get_device_info` — 获取设备信息

```pseudo
function get_device_info(device_id?):
    if 缓存命中(deviceId): return 缓存

    // 6 个 shell 命令并行执行（Promise.all）:
    [model, manufacturer, androidVersion, apiLevel, sizeOutput, densityOutput] =
        Promise.all([
            adb shell getprop ro.product.model,
            adb shell getprop ro.product.manufacturer,
            adb shell getprop ro.build.version.release,
            adb shell getprop ro.build.version.sdk,
            adb shell wm size,          // → "Physical size: 1080x2400"
            adb shell wm density,       // → "Physical density: 420"
        ])

    screenSize = sizeOutput.match(/(\d+x\d+)/)[1]
    density = densityOutput.match(/(\d+)/)[1]
    缓存结果到 Map
    返回: "Model: Samsung Pixel_6 | Android: 14 (API 34) | Screen: 1080x2400 @ 420dpi"
```

#### `pull_file` — 从设备拉取文件

```pseudo
function pull_file(remote_path, local_path, device_id?):
    mkdir -p dirname(local_path)    // 确保本地目录存在
    adb pull {remote_path} {local_path}
    // 超时: 30 秒
```

---

## 五、扩展设计模式总结

### 三种 Tool 扩展模式

```
原子 ADB 命令 ──→ Adb 类方法（封装+错误处理） ──→ MCP Tool（参数校验+结果格式化）
```

#### 模式一：1:1 直通映射（7个）

直接透传到单个 adb 命令，仅做参数封装和结果格式化。

| Tool | 对应 ADB 命令 |
|------|--------------|
| `tap` | `input tap` |
| `swipe` | `input swipe` |
| `type_text` | `input text` |
| `press_key` | `input keyevent` |
| `adb_shell` | `shell <任意命令>` |
| `clear_logs` | `logcat -c` |
| `pull_file` | `pull` |

#### 模式二：组合复用（5个）

基于已有的原子方法进行组合，产生新的交互语义。

| Tool | 组合方式 |
|------|---------|
| `long_press` | `swipe(x,y,x,y,duration)` — 原地滑动模拟长按 |
| `double_tap` | `tap()` × 2 + `sleep(interval)` |
| `multi_tap` | `tap()` × N + `sleep(interval)` 循环 |
| `tap_sequence` | `tap/swipe/typeText/pressKey/sleep` 串行编排 |
| `scroll_to_element` | `getUiTree()` + `swipe()` 循环直到找到元素 |

#### 模式三：智能增强（8个）

在原子命令基础上增加解析、过滤、压缩、轮询等智能逻辑。

| Tool | 增强逻辑 |
|------|---------|
| `screenshot` | `screencap` + sharp 图像压缩/缩放 + base64 编码 |
| `get_ui_tree` | `uiautomator dump` + XML 正则解析 + 元素过滤 + 坐标计算 |
| `tap_element` | `getUiTree()` 查找元素 + `tap()` 中心点 |
| `tap_and_wait` | `tap_element()` + `sleep()` + `getUiTree()` 刷新 |
| `wait_for_element` | `getUiTree()` 轮询（每500ms） + 超时控制 |
| `get_logs` | `logcat` + `pidof` 按包名过滤 |
| `get_device_info` | 6个 `getprop/wm` 命令并行 + 结果缓存 |
| `start_emulator` | `emulator -avd` + `devices` 轮询等待上线 |

---

## 六、关键设计细节

### 6.1 持久化 Shell 连接池

- 每个 `deviceId` 维护一个独立的持久化 shell 进程
- 用 `__MCP_DONE__` 标记分隔命令输出
- 进程退出时自动清理并 reject 所有 pending 请求
- 避免每次 `adb shell` 都 fork 新进程，显著降低延迟

### 6.2 UI 树获取的双重回退

1. 优先: `uiautomator dump /dev/tty` — 直接输出到终端，无磁盘 IO
2. 回退: `uiautomator dump /sdcard/window_dump.xml` + `cat` 读取 — 兼容不支持 /dev/tty 的设备

### 6.3 元素匹配策略

支持三种匹配方式，且都做了模糊匹配：
- `resource-id`: 精确匹配 或 后缀匹配 (`:id/xxx`)
- `text`: 精确匹配 或 大小写不敏感包含
- `content-desc`: 精确匹配 或 大小写不敏感包含

### 6.4 截图压缩管线

```
screencap -p (原始PNG) → sharp.resize(maxWidth=1280) → png(quality=80) → base64
```

减小传输体积，适配 MCP 协议的文本传输。

### 6.5 设备信息缓存

`getDeviceInfo()` 结果按 deviceId 缓存到 Map 中，避免重复查询（6个 shell 命令并行但仍有开销）。`scroll_to_element` 等需要屏幕尺寸的 Tool 受益于此缓存。
