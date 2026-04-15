# appium-mcp 源码解读

> 仓库地址：https://github.com/1405942836/appium-mcp
> 本地路径：`~/Documents/coding/github/appium-mcp`
> 分析聚焦：原子命令层、Tool 扩展机制、核心伪代码、与 mobile-mcp 对比
> 分析日期：2026-04-15

---

## 一、项目定位

appium-mcp 是一个基于 **Appium + MCP 协议** 的移动设备自动化测试服务器。它将 Appium 的能力通过 MCP 协议暴露给 AI 助手（如 Claude）。

**核心区别：** 它不直接调用 adb / WDA 等底层命令，而是通过 **Appium Server 作为中间层**，所有设备操作都通过 Appium Python Client（WebDriver 协议）完成。

### 适用场景

- AI 驱动的移动应用自动化测试
- 通过自然语言指令控制设备
- 跨平台（Android + iOS）统一自动化

### 优点与局限

| 优点 | 局限 |
|------|------|
| 基于成熟的 Appium 生态，支持 10+ 种定位策略 | 需要额外运行 Appium Server 进程 |
| Python 异步架构，支持并发 | 当前实现的 Tool 只有 13 个（README 声称 40+） |
| 会话池管理、超时清理、健康检查 | 无截图压缩优化，返回原始 base64 |
| 丰富的 CLI 子命令（run/doctor/init-config 等） | 无录屏功能 |
| 结构化异常体系 | 无 App 安装/卸载 Tool |

---

## 二、整体架构

```
┌──────────────────────────────────────────────────────────┐
│                 AI Agent (Claude, ChatGPT, etc.)          │
│                        ↕ MCP Protocol (stdio)             │
├──────────────────────────────────────────────────────────┤
│               AppiumMCPServer (server.py)                 │
│   list_tools / call_tool / list_resources / get_prompt    │
├──────────────────────────────────────────────────────────┤
│   Tool 注册层                                             │
│   ┌─────────────────────┐  ┌─────────────────────┐       │
│   │  DeviceManagement   │  │   UIAutomation       │       │
│   │  Tools (8个)        │  │   Tools (5个)        │       │
│   └─────────┬───────────┘  └─────────┬───────────┘       │
│             ↓                        ↓                    │
│   ┌─────────────────────────────────────────────┐        │
│   │    AppiumTool 基类 (tools/base.py)           │        │
│   │    参数校验 / safe_execute / 会话锁            │        │
│   └──────────────────┬──────────────────────────┘        │
│                      ↓                                    │
│   ┌──────────────────────────────────────────────┐       │
│   │    SessionManager (会话池 / 超时 / 清理)       │       │
│   │            ↓                                  │       │
│   │    DeviceManager (设备发现 / 连接 / 断开)      │       │
│   └──────────────────┬───────────────────────────┘       │
│                      ↓                                    │
│   ┌──────────────────────────────────────────────┐       │
│   │    Appium Python Client (webdriver.Remote)    │       │
│   │    UiAutomator2Options / XCUITestOptions      │       │
│   └──────────────────┬───────────────────────────┘       │
├──────────────────────┼───────────────────────────────────┤
│                      ↓                                    │
│            Appium Server (http://localhost:4723)           │
│            ↓                        ↓                     │
│   UiAutomator2 Driver       XCUITest Driver               │
│        ↓                        ↓                         │
│   Android Device            iOS Device                    │
└──────────────────────────────────────────────────────────┘
```

---

## 三、"原子命令"层分析

### 关键差异：appium-mcp 没有直接的 adb 原子命令

与 mobile-mcp 不同，appium-mcp 的 **所有设备 UI 操作** 都通过 `Appium Python Client`（即 `webdriver.Remote`）进行，不直接执行 adb 命令。

**唯一直接调用 adb 的地方：** 设备发现阶段（`device_manager.py`）

### 3.1 设备发现阶段的原子命令

| # | 命令 | 用途 | 源码位置 |
|---|------|------|---------|
| 1 | `adb devices -l` | 列出 Android 设备 | device_manager.py:253 |
| 2 | `adb -s {id} shell getprop` | 获取 Android 设备全部属性 | device_manager.py:308 |
| 3 | `xcrun simctl list devices --json` | 列出 iOS 模拟器 | device_manager.py:360 |
| 4 | `idevice_id -l` | 列出 iOS 真机 | device_manager.py:412 |
| 5 | `ideviceinfo -u {id}` | 获取 iOS 真机详情 | device_manager.py:442 |

### 3.2 UI 操作的"原子操作" — Appium WebDriver API

所有 UI 交互都委托给 Appium WebDriver 协议：

| 操作 | Appium Python Client 调用 | 底层协议 |
|------|--------------------------|---------|
| 查找元素 | `WebDriverWait(driver, timeout).until(EC.presence_of_element_located(...))` | POST /session/{id}/element |
| 点击元素 | `element.click()` | POST /session/{id}/element/{eid}/click |
| 输入文本 | `element.send_keys(text)` | POST /session/{id}/element/{eid}/value |
| 清除文本 | `element.clear()` | POST /session/{id}/element/{eid}/clear |
| 截图 | `driver.get_screenshot_as_base64()` | GET /session/{id}/screenshot |
| 滑动 | `driver.swipe(x0, y0, x1, y1, duration)` | POST /session/{id}/touch/perform |
| 获取属性 | `element.get_attribute("content-desc")` | GET /session/{id}/element/{eid}/attribute/{name} |
| 窗口大小 | `driver.get_window_size()` | GET /session/{id}/window/current/size |
| 创建会话 | `webdriver.Remote(url, options=options)` | POST /session |
| 关闭会话 | `driver.quit()` | DELETE /session/{id} |

---

## 四、Tool 扩展机制

### 4.1 工具基类体系

```
AppiumTool (抽象基类)
├── DeviceManagementTool (设备管理类)
├── UIAutomationTool (UI 自动化类，含通用定位器参数)
├── AppControlTool (应用控制类，未实现具体 Tool)
├── SystemOperationTool (系统操作类，未实现具体 Tool)
└── FileOperationTool (文件操作类，未实现具体 Tool)
```

每个 Tool 需实现 4 个抽象属性/方法：
- `name` — Tool 名称
- `description` — Tool 描述
- `parameters` — JSON Schema 格式的参数定义
- `execute(arguments)` — 异步执行逻辑

### 4.2 Tool 注册流程

```pseudo
class AppiumMCPServer:
    def _initialize_tools():
        self.tools = [
            ListDevicesTool(session_manager),
            ConnectDeviceTool(session_manager),
            # ... 共 13 个 Tool
        ]

    def _register_handlers():
        @mcp_server.list_tools()
        async def list_tools():
            return [tool.to_mcp_tool() for tool in self.tools]

        @mcp_server.call_tool()
        async def call_tool(name, arguments):
            tool = find_tool_by_name(name)
            result = await tool.safe_execute(arguments)  # 含参数校验 + 错误包装
            return JSON(result)
```

### 4.3 会话锁机制

UI 操作 Tool 都通过 `get_session_with_lock()` 获取会话和异步锁：

```pseudo
async def execute(arguments):
    session, lock = await self.get_session_with_lock(session_id)
    async with lock:
        # 在锁保护下执行操作，防止并发冲突
        element = WebDriverWait(session.driver, timeout).until(...)
        element.click()
```

---

## 五、每个 MCP Tool 的核心伪代码

### 5.1 list_devices

```pseudo
function list_devices(platform?, status?):
    devices = device_manager.get_all_devices()  // 内存缓存
    for device in devices:
        if platform_filter and device.platform != platform: skip
        if status_filter and device.status != status: skip
    return { devices, total_count, platforms: {android: N, ios: N} }
```

### 5.2 connect_device

```pseudo
function connect_device(device_id, app_package?, app_activity?, bundle_id?, no_reset?, full_reset?):
    capabilities = {}
    if app_package: capabilities.appPackage = app_package
    if bundle_id: capabilities.bundleId = bundle_id
    // ...

    // 核心：通过 Appium 创建 WebDriver 会话
    if device.platform == "android":
        options = UiAutomator2Options()
        options.device_name = device.name
        options.udid = device_id
        options.automation_name = "UiAutomator2"
    elif device.platform == "ios":
        options = XCUITestOptions()
        options.automation_name = "XCUITest"

    driver = webdriver.Remote("http://localhost:4723", options=options)
    session = DeviceSession(device_info, driver, driver.session_id)
    session_pool.add(session)
    return { session_id, device_info, capabilities, status: "connected" }
```

### 5.3 disconnect_device

```pseudo
function disconnect_device(session_id):
    session = session_pool.get(session_id)
    session.driver.quit()  // DELETE /session/{id}
    session_pool.remove(session_id)
    return { session_id, device_id, status: "disconnected" }
```

### 5.4 get_device_info

```pseudo
function get_device_info(device_id):
    info = device_manager.get_device_info(device_id)
    if not info:
        await device_manager.discover_devices()  // 重新发现
        info = device_manager.get_device_info(device_id)
    return { device_id, found, device_info }
```

### 5.5 get_session_info / list_sessions

```pseudo
function get_session_info(session_id):
    session = session_manager.get_session(session_id)
    return session.to_dict()

function list_sessions():
    sessions = session_manager.get_all_sessions()
    stats = session_manager.get_session_stats()  // 活跃/空闲/利用率
    return { sessions, stats }
```

### 5.6 cleanup_sessions / refresh_devices

```pseudo
function cleanup_sessions():
    expired = session_pool.cleanup_expired_sessions()  // 基于 last_activity 超时
    device_manager.discover_devices()  // 刷新设备列表
    return { expired_count }

function refresh_devices():
    // 并行执行 Android + iOS 发现
    android_devices = await _discover_android_devices()  // adb devices -l
    ios_devices = await _discover_ios_devices()           // xcrun simctl list --json
    return { devices }
```

### 5.7 find_element

```pseudo
function find_element(session_id, locator_type, locator_value, timeout=10):
    session, lock = get_session_with_lock(session_id)
    async with lock:
        by = LOCATOR_MAP[locator_type]
        // LOCATOR_MAP: id→AppiumBy.ID, xpath→AppiumBy.XPATH, accessibility_id→AppiumBy.ACCESSIBILITY_ID, ...
        wait = WebDriverWait(session.driver, timeout)
        element = wait.until(EC.presence_of_element_located((by, locator_value)))
        return {
            element_id, tag_name, text, enabled, displayed, selected,
            location: {x, y}, size: {width, height}, rect,
            attributes: { content-desc, resource-id, class, clickable, focused, ... }
        }
```

### 5.8 click_element

```pseudo
function click_element(session_id, locator_type, locator_value, timeout=10):
    session, lock = get_session_with_lock(session_id)
    async with lock:
        by = LOCATOR_MAP[locator_type]
        wait = WebDriverWait(session.driver, timeout)
        element = wait.until(EC.element_to_be_clickable((by, locator_value)))
        element.click()
        return { success: true, element_info: {id, text, location, size} }
```

### 5.9 input_text

```pseudo
function input_text(session_id, locator_type, locator_value, text, clear_first=true, timeout=10):
    session, lock = get_session_with_lock(session_id)
    async with lock:
        by = LOCATOR_MAP[locator_type]
        wait = WebDriverWait(session.driver, timeout)
        element = wait.until(EC.presence_of_element_located((by, locator_value)))
        if clear_first:
            element.clear()
        element.send_keys(text)
        return { success: true, text, clear_first }
```

### 5.10 take_screenshot

```pseudo
function take_screenshot(session_id, filename?, format="png"):
    session, lock = get_session_with_lock(session_id)
    async with lock:
        base64_data = session.driver.get_screenshot_as_base64()
        if not filename:
            filename = f"screenshot_{device_name}_{timestamp}.{format}"
        // 尝试保存到 ./screenshots/ 目录
        save_to_file(base64_decode(base64_data), f"./screenshots/{filename}")
        return { success, filename, file_path, base64_data, format }
```

### 5.11 swipe

```pseudo
function swipe(session_id, start_x, start_y, end_x, end_y, duration=1000):
    session, lock = get_session_with_lock(session_id)
    async with lock:
        session.driver.swipe(start_x, start_y, end_x, end_y, duration)
        return { success, start_point, end_point, duration }
```

---

## 六、与 mobile-mcp 的关键差异对比

### 6.1 架构差异

| 维度 | mobile-mcp | appium-mcp |
|------|-----------|------------|
| **语言** | TypeScript (Node.js) | Python 3.9+ |
| **底层驱动** | 直接调用 adb / go-ios / WDA HTTP | 通过 Appium Server 间接调用 |
| **依赖层级** | MCP Server → adb/WDA (2层) | MCP Server → Appium → UiAutomator2/XCUITest (3层) |
| **外部依赖** | 仅需 adb / go-ios | 需要 Appium Server 进程 + Driver 插件 |
| **执行模型** | 同步 execFileSync (阻塞) | 异步 asyncio + aiofiles |
| **并发处理** | 无（单线程阻塞） | 会话锁（asyncio.Lock） |

### 6.2 功能覆盖对比

| 功能 | mobile-mcp | appium-mcp |
|------|:---------:|:---------:|
| 设备发现 | adb devices + go-ios list + mobilecli | adb devices + xcrun simctl + idevice_id |
| 点击（坐标） | input tap x y | -- (只有元素点击) |
| 点击（元素） | -- (只有坐标点击) | element.click() (10种定位策略) |
| 双击 | 支持 | 不支持 |
| 长按 | 支持 | 不支持 |
| 滑动 | 支持（方向 / 坐标） | 支持（坐标） |
| 文本输入 | adb input text (+ DeviceKit) | element.send_keys() |
| 元素查找 | uiautomator dump XML | WebDriverWait + 10种定位器 |
| 截图 | screencap + 压缩 | driver.get_screenshot_as_base64() |
| 截图压缩 | sips / ImageMagick → JPEG 75% | 无 |
| 按键 | input keyevent | 不支持 |
| App 启动 | monkey / go-ios launch | 通过 capabilities 启动 |
| App 停止 | am force-stop / go-ios kill | 不支持 |
| App 安装 | adb install / go-ios install | 不支持 |
| App 卸载 | adb uninstall / go-ios uninstall | 不支持 |
| 打开 URL | am start -a VIEW | 不支持 |
| 屏幕方向 | 支持设置/获取 | 不支持 |
| 录屏 | mobilecli screenrecord | 不支持 |
| 远程设备池 | Fleet 支持 | 不支持 |
| 会话管理 | 无状态（每次操作独立） | 有状态（会话池 + 超时 + 锁） |
| 健康检查 | 无 | 支持（驱动 + 会话健康） |
| CLI 工具 | 仅 --listen / --stdio | run / doctor / init-config / list-devices / version |
| MCP Resources | 不支持 | 支持 |
| MCP Prompts | 不支持 | 支持 |

### 6.3 元素交互模型差异

**mobile-mcp 的方式：坐标驱动**
```
1. list_elements_on_screen → 获取所有元素的坐标和属性
2. AI 根据元素列表选择目标
3. click_on_screen_at_coordinates(x, y) → adb shell input tap x y
```

**appium-mcp 的方式：元素定位器驱动**
```
1. find_element(locator_type="accessibility_id", value="login_button")
2. click_element(locator_type="accessibility_id", value="login_button")
   → WebDriverWait → element.click()
```

### 6.4 安全防护对比

| 安全措施 | mobile-mcp | appium-mcp |
|---------|:---------:|:---------:|
| 命令注入防护 | escapeShellText() | 不直接执行 shell（委托给 Appium） |
| 包名白名单 | validatePackageName 正则 | validate_device_id 正则 |
| 路径安全 | validateOutputPath (tmpdir/cwd) | sanitize_filename (移除非法字符) |
| URL 协议限制 | 默认 http/https only | 无 |
| 文件名清理 | 无 | sanitize_filename |
| Zip-Slip 防护 | 有 (iOS .zip 安装) | 无 (无安装功能) |
| 输出路径限制 | 严格白名单 | 写死 ./screenshots/ |

### 6.5 设计哲学差异

| 维度 | mobile-mcp | appium-mcp |
|------|-----------|------------|
| **设计哲学** | 最小依赖、直连设备 | 利用 Appium 生态、功能全面 |
| **抽象层级** | Robot 接口（轻量） | Tool 基类 + SessionManager + DeviceManager（重量） |
| **错误模型** | ActionableError (单一) | 16 种结构化异常类 |
| **配置** | 环境变量 | YAML 配置文件 + Pydantic 模型 |
| **遥测** | PostHog 匿名遥测 | 无 |
| **包分发** | npm (@mobilenext/mobile-mcp) | pip (appium-mcp-server) |

### 6.6 总结

- **mobile-mcp** 更适合 **轻量级、快速部署** 的场景。它直接操控设备，延迟低，Tool 功能覆盖面广（20+ 个 Tool 全部实现），但交互模式偏"坐标驱动"。
- **appium-mcp** 更适合 **测试工程化** 的场景。它利用 Appium 成熟的元素定位体系（10种定位策略），有会话池管理，但 Tool 实现不完整（声称 40+ 实际只有 13 个），且增加了 Appium Server 的部署和维护成本。

---

## 七、值得借鉴的设计

1. **Tool 基类 + 分类子类** — 比 mobile-mcp 的函数式注册更结构化，便于扩展
2. **会话池 + 异步锁** — 并发安全，适合多 Agent 同时操控
3. **结构化异常体系** — 16 种异常类，错误信息精确
4. **CLI doctor 命令** — 自动检查环境依赖，降低上手门槛
5. **设备发现超时保护** — 5 秒超时避免阻塞 MCP 握手

---

## 附：文件索引

| 文件 | 职责 | 行数 |
|------|------|------|
| `src/appium_mcp/server.py` | MCP Server 主类 | ~427 |
| `src/appium_mcp/tools/base.py` | Tool 基类 + 分类子类 | ~330 |
| `src/appium_mcp/tools/device_tools.py` | 设备管理 Tools (8个) | ~364 |
| `src/appium_mcp/tools/ui_tools.py` | UI 自动化 Tools (5个) | ~509 |
| `src/appium_mcp/core/device_manager.py` | 设备发现 + 连接 + WebDriver 创建 | ~665 |
| `src/appium_mcp/core/session_manager.py` | 会话池 + 超时 + 清理 | ~512 |
| `src/appium_mcp/core/config_manager.py` | YAML 配置 + Pydantic 模型 | - |
| `src/appium_mcp/core/resource_manager.py` | MCP Resource 管理 | - |
| `src/appium_mcp/core/prompt_manager.py` | MCP Prompt 模板管理 | - |
| `src/appium_mcp/utils/helpers.py` | 工具函数（校验/重试/超时/图片） | ~547 |
| `src/appium_mcp/utils/exceptions.py` | 16 种结构化异常 | ~290 |
| `src/appium_mcp/cli.py` | Click CLI 入口 | ~351 |
