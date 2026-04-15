# appium-mcp 快速上手

> 仓库地址：https://github.com/1405942836/appium-mcp
> 本地路径：`~/Documents/coding/github/appium-mcp`

---

## 一、环境要求

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Python | 3.9+ | 运行 MCP Server |
| pip | 随 Python 安装 | 包管理 |
| Appium Server | 2.x | 底层自动化引擎，需独立运行 |
| Node.js | v16+ | 安装和运行 Appium Server |
| Android SDK Platform Tools | 最新版 | 提供 `adb`（Android 设备） |
| Xcode + Command Line Tools | 最新版 | iOS 模拟器/真机（仅 macOS） |

### 核心依赖（自动安装）

| 包名 | 版本 | 用途 |
|------|------|------|
| `mcp` | >=1.12.0 | MCP 协议 SDK |
| `Appium-Python-Client` | >=4.0.0 | Appium WebDriver 客户端 |
| `pydantic` | >=2.0.0 | 数据模型校验 |
| `pyyaml` | >=6.0.0 | YAML 配置解析 |
| `click` | >=8.0.0 | CLI 命令行框架 |
| `structlog` | >=23.0.0 | 结构化日志 |
| `aiofiles` | >=23.0.0 | 异步文件操作 |
| `pillow` | >=10.0.0 | 图片处理（截图） |

---

## 二、安装与启动

### 步骤 1：安装 Appium Server

```bash
# 全局安装 Appium
npm install -g appium

# 安装 Android 驱动
appium driver install uiautomator2

# 安装 iOS 驱动（仅 macOS）
appium driver install xcuitest
```

### 步骤 2：安装 appium-mcp

```bash
# 方式 A：pip 安装（如已发布到 PyPI）
pip install appium-mcp-server

# 方式 B：从源码安装（开发模式）
git clone https://github.com/1405942836/appium-mcp.git
cd appium-mcp
pip install -e .

# 方式 C：带开发依赖
pip install -e ".[dev]"
```

### 步骤 3：生成默认配置（可选）

```bash
appium-mcp-server init-config
# 生成 config.yaml 到当前目录
```

### 步骤 4：启动服务

```bash
# 1. 先启动 Appium Server（独立终端）
appium --port 4723

# 2. 再启动 appium-mcp（另一个终端）
appium-mcp-server run
```

> **关键区别**：appium-mcp 需要 Appium Server 作为中间层，必须先启动 Appium。mobile-mcp 则直接调用 adb，无需额外进程。

---

## 三、MCP 客户端配置

### Claude Desktop

编辑 Claude Desktop 配置文件：

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "appium": {
      "command": "appium-mcp-server",
      "args": ["run"]
    }
  }
}
```

配置后重启 Claude Desktop。

### 其他 MCP 客户端（Cursor / VS Code 等）

在对应的 MCP 配置中添加相同的 command 和 args 即可。

---

## 四、配置说明（config.yaml）

appium-mcp 使用 YAML 配置文件，主要配置段：

### server — Appium Server 连接

```yaml
server:
  host: localhost        # Appium Server 地址
  port: 4723             # Appium Server 端口
  timeout: 30            # 连接超时（秒）
  new_command_timeout: 60 # 命令超时（秒）
```

### android — Android 设备参数

```yaml
android:
  platform_name: Android
  automation_name: UiAutomator2   # 使用 UiAutomator2 驱动
  implicit_wait: 10               # 隐式等待（秒）
  explicit_wait: 30               # 显式等待（秒）
  no_reset: false                 # 是否保留 App 数据
```

### ios — iOS 设备参数

```yaml
ios:
  platform_name: iOS
  automation_name: XCUITest       # 使用 XCUITest 驱动
  wda_local_port: 8100            # WDA 端口
  use_new_wda: true               # 每次新建 WDA
  xcode_org_id: null              # 真机签名（填你的 Team ID）
  xcode_signing_id: null          # 签名标识
```

### features — 功能开关

```yaml
features:
  auto_screenshot: true           # 操作后自动截图
  screenshot_on_error: true       # 错误时截图
  element_highlight: true         # 高亮被操作元素
  auto_detect_devices: true       # 自动发现设备
  device_health_check: true       # 设备健康检查
  health_check_interval: 30       # 检查间隔（秒）
```

---

## 五、设备准备

### Android 模拟器/真机

```bash
# 1. 确认 adb 可用
adb devices
# 应看到设备列表，如 emulator-5554

# 2. 确认 Appium Server 已启动
curl http://localhost:4723/status
# 应返回 JSON 状态信息
```

### iOS 模拟器

```bash
# 1. 启动模拟器
xcrun simctl list devices | grep Booted
# 或手动启动：
xcrun simctl boot "iPhone 16"

# 2. 确认 Appium Server 已安装 XCUITest 驱动
appium driver list --installed
# 应包含 xcuitest
```

### iOS 真机

需要配置代码签名：
1. 在 `config.yaml` 中填写 `xcode_org_id` 和 `xcode_signing_id`
2. 确保设备已信任开发者证书
3. 可能需要手动安装 WebDriverAgent

---

## 六、验证安装

启动 MCP Server 后，通过 AI Agent 发送以下指令验证：

**第 1 步：列出设备**
```
列出所有可用设备
```
Agent 调用 `list_devices` Tool，返回设备列表。

**第 2 步：连接设备**
```
连接设备 emulator-5554
```
Agent 调用 `connect_device` Tool，创建 Appium 会话。

**第 3 步：截图验证**
```
对当前设备截一张图
```
Agent 调用 `take_screenshot` Tool，返回 base64 截图。

**第 4 步：UI 交互**
```
查找屏幕上的按钮并点击
```
Agent 调用 `find_element` + `click_element` Tool。

---

## 七、可用 MCP Tool 列表速查

### 设备管理（8 个）

| Tool | 说明 |
|------|------|
| `list_devices` | 发现所有可用设备（调用 adb devices） |
| `connect_device` | 连接设备并创建 Appium 会话 |
| `disconnect_device` | 断开设备、关闭会话 |
| `get_device_info` | 获取设备详情（型号/系统版本/屏幕等） |
| `get_session_info` | 查看当前活跃会话信息 |
| `list_sessions` | 列出会话池中所有会话 |
| `cleanup_sessions` | 清理过期/失效会话 |
| `refresh_devices` | 重新扫描设备列表 |

### UI 自动化（5 个）

| Tool | 说明 |
|------|------|
| `find_element` | 按定位器查找元素（支持 10 种策略） |
| `click_element` | 定位元素并点击 |
| `input_text` | 定位输入框并输入文本 |
| `take_screenshot` | 截取屏幕（返回 base64 PNG） |
| `swipe` | 执行滑动手势 |

### 元素定位策略

appium-mcp 支持以下 10 种定位方式（通过 `locator_type` 参数指定）：

| locator_type | 说明 | 示例 |
|-------------|------|------|
| `id` | 资源 ID | `com.app:id/login_btn` |
| `name` | 名称 | `Login` |
| `class_name` | 类名 | `android.widget.Button` |
| `xpath` | XPath 表达式 | `//Button[@text="OK"]` |
| `css_selector` | CSS 选择器（WebView） | `.login-btn` |
| `accessibility_id` | 无障碍 ID | `login_button` |
| `android_uiautomator` | UiSelector 表达式 | `new UiSelector().text("OK")` |
| `ios_predicate` | NSPredicate（iOS） | `label == "OK"` |
| `ios_class_chain` | Class Chain（iOS） | `**/XCUIElementTypeButton` |
| `image` | 图片模板匹配 | base64 编码图片 |

---

## 八、开发命令

| 命令 | 说明 |
|------|------|
| `pip install -e ".[dev]"` | 安装开发依赖 |
| `pytest` | 运行测试 |
| `pytest --cov` | 运行测试（含覆盖率） |
| `black src/` | 代码格式化 |
| `isort src/` | import 排序 |
| `flake8 src/` | 代码风格检查 |
| `mypy src/` | 类型检查 |
| `pre-commit install` | 安装 git hook |

### CLI 子命令

```bash
appium-mcp-server run           # 启动 MCP 服务器
appium-mcp-server doctor        # 环境检查（检测 Appium / adb / Xcode 等）
appium-mcp-server init-config   # 生成默认配置文件
appium-mcp-server --version     # 查看版本
```

---

## 九、常见问题

### Q: 启动报错 "Could not connect to Appium Server"？

确认 Appium Server 已独立启动：
```bash
appium --port 4723
```
然后确认端口可达：
```bash
curl http://localhost:4723/status
```

### Q: connect_device 超时？

1. 检查设备是否在线：`adb devices`
2. 检查 `config.yaml` 中 `timeout` 和 `new_command_timeout` 是否足够
3. Android 模拟器首次连接可能较慢，建议 `timeout` 设为 60+

### Q: find_element 找不到元素？

1. 确认定位策略和值是否正确
2. 增大 `explicit_wait`（默认 30 秒）
3. 使用 Appium Inspector 辅助定位
4. 尝试不同定位策略（如从 `id` 切换到 `xpath`）

### Q: iOS 真机无法连接？

1. 确认 `xcode_org_id` 和 `xcode_signing_id` 已配置
2. 确认 WebDriverAgent 已正确签名安装到设备
3. 确认 `appium driver install xcuitest` 已执行

### Q: 会话泄漏（Session leak）？

appium-mcp 内置会话池管理和超时清理：
- 默认会话超时后自动清理
- 可手动调用 `cleanup_sessions` Tool
- 配置 `features.health_check_interval` 控制检查频率

---

## 十、与 mobile-mcp 的选型建议

| 场景 | 推荐 | 原因 |
|------|------|------|
| 快速接入、无额外依赖 | mobile-mcp | 无需 Appium Server，直接 adb 调用 |
| 复杂元素定位、测试流程 | appium-mcp | 10 种定位策略，WebDriverWait 显式等待 |
| 只用 Android | mobile-mcp | 功能最完整（20 个 Tool） |
| Android + iOS 统一测试 | 两者皆可 | 都支持跨平台，但 API 风格不同 |
| 需要并发多设备 | appium-mcp | 异步架构 + 会话池 + asyncio.Lock |
| 需要录屏功能 | mobile-mcp | 内置 screenrecord 支持 |
| 需要 App 安装/卸载 | mobile-mcp | 内置 install/uninstall Tool |
| CI/CD 集成、已有 Appium 生态 | appium-mcp | 复用已有 Appium 基础设施 |

---

## 十一、建议先读的源码

1. **`src/appium_mcp/tools/base.py`** -- Tool 基类，理解参数校验和会话锁机制
2. **`src/appium_mcp/tools/ui_tools.py`** -- UI 自动化的核心实现
3. **`src/appium_mcp/core/device_manager.py`** -- 设备发现和 Appium 会话创建
4. **`src/appium_mcp/core/session_manager.py`** -- 会话池管理和并发控制
5. **`src/appium_mcp/server.py`** -- MCP 处理器注册，理解 Tool 如何暴露给 AI
6. **`config.yaml`** -- 配置结构，理解所有可调参数
