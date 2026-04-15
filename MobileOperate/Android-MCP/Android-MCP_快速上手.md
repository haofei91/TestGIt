# Android-MCP (CursorTouch) 快速上手指南

> 仓库：https://github.com/CursorTouch/Android-MCP
> 定位：基于 uiautomator2 的轻量级 Android MCP Server
> 核心特性：Selector 元素查找、懒连接、截图标注与量化

---

## 一、环境要求

| 依赖 | 要求 |
|------|------|
| Python | **3.13**（严格限制，不支持其他版本） |
| UV | 推荐安装 `uv` 包管理器 |
| ADB | 已安装并加入 PATH |
| Android 设备 | Android 10+，真机或模拟器，USB 调试已开启 |

---

## 二、安装方式

### 方式一：UVX 直接运行（推荐）

无需手动安装，直接在 Claude Desktop / Cursor 配置中使用：

```json
{
  "mcpServers": {
    "android-mcp": {
      "command": "uvx",
      "args": ["--python", "3.13", "android-mcp"]
    }
  }
}
```

### 方式二：本地开发模式

```bash
git clone https://github.com/CursorTouch/Android-MCP.git
cd Android-MCP
uv sync
```

配置 Claude Desktop：

```json
{
  "mcpServers": {
    "android-mcp": {
      "command": "uv",
      "args": [
        "--directory", "/你的路径/Android-MCP",
        "run", "android-mcp"
      ]
    }
  }
}
```

### 安装 UV（如果没有）

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

---

## 三、设备连接

### 3.1 验证 ADB

```bash
adb devices
# 应看到：
# List of devices attached
# emulator-5554   device
```

### 3.2 设备指定方式

**优先级从高到低**：

| 方式 | 示例 |
|------|------|
| `--wifi HOST` | `--wifi 192.168.1.3` |
| `--usb [SERIAL]` | `--usb RFCN2013V8D` |
| `--device SERIAL` | `--device emulator-5554` |
| 环境变量 `ANDROID_MCP_DEVICE` | `emulator-5554` |
| 环境变量 `ANDROID_MCP_CONNECTION` | `auto` / `usb` / `wifi` |
| 环境变量 `ANDROID_MCP_HOST` | `192.168.1.3` |
| 自动检测 | 使用第一个可用设备 |

**WiFi 连接示例**：

```json
{
  "mcpServers": {
    "android-mcp": {
      "command": "uvx",
      "args": ["--python", "3.13", "android-mcp"],
      "env": {
        "ANDROID_MCP_CONNECTION": "wifi",
        "ANDROID_MCP_HOST": "192.168.1.3"
      }
    }
  }
}
```

---

## 四、可用工具（MCP Tools）

### 4.1 设备管理

| Tool | 说明 |
|------|------|
| `ListDevices` | 列出所有 ADB 设备 |
| `ConnectDevice` | 连接到指定设备 |
| `Device` | 统一管理 (list/connect/disconnect) |

### 4.2 UI 操作

| Tool | 参数 | 说明 |
|------|------|------|
| `Click` | x, y | 点击坐标 |
| `ClickBySelector` | text/resourceId/className/description, timeout | 按选择器点击 |
| `LongClick` | x, y | 长按坐标 |
| `Swipe` | x1, y1, x2, y2 | 滑动 |
| `Drag` | x1, y1, x2, y2 | 拖拽 |
| `Type` | text, x, y, clear | 输入文本 |
| `Press` | button | 按键 (Back, Home 等) |

### 4.3 状态查询

| Tool | 参数 | 说明 |
|------|------|------|
| `Snapshot` | use_vision, use_annotation | 获取 UI 树状态 + 可选截图 |
| `Notification` | 无 | 打开通知栏 |
| `WaitForElement` | text/resourceId/className/description, timeout | 等待元素出现 |
| `Wait` | duration | 等待指定秒数 |

### 4.4 Snapshot 工具详解

`Snapshot` 是最重要的工具，返回结构化的 UI 状态：

- `use_vision=False`（默认）：仅返回 UI 树表格
- `use_vision=True`：返回 UI 树 + 标注截图
- `use_annotation=False`：截图不带 bounding box 标注

返回的 UI 树格式：
```
Label  Name              ResourceId      Class                          Coordinates
0      Settings          btn_settings    android.widget.Button          (540,120)
1      Search            search_input    android.widget.EditText        (540,240)
2      Profile           -               android.widget.ImageButton     (980,120)
```

LLM 可以用 Label 编号对应的坐标来执行操作。

---

## 五、环境变量

| 变量 | 说明 |
|------|------|
| `ANDROID_MCP_DEVICE` | 指定设备序列号 |
| `ANDROID_MCP_CONNECTION` | 连接类型 (auto/usb/wifi) |
| `ANDROID_MCP_HOST` | WiFi ADB 主机地址 |
| `SCREENSHOT_QUANTIZED` | 设为 `true` 启用截图量化（256 色），减少 token |

---

## 六、使用示例

### 在 Claude Desktop 中使用

配置好 MCP Server 后，可以直接用自然语言操作手机：

```
用户: 帮我打开设置，看看当前的 WiFi 名称
Claude: 
1. 调用 Snapshot 查看当前界面
2. 调用 Press("home") 回到桌面
3. 调用 ClickBySelector(text="设置") 打开设置
4. 调用 Snapshot 查看设置页面
5. 调用 ClickBySelector(text="WLAN") 进入 WiFi 设置
6. 调用 Snapshot 读取 WiFi 名称
```

### 关键 Tool 用法

```python
# 获取当前界面状态
Snapshot(use_vision=True)

# 按文本点击
ClickBySelector(text="确定")

# 按 resourceId 点击
ClickBySelector(resourceId="btn_submit")

# 等待加载完成
WaitForElement(text="加载完成", timeout=10)

# 滑动浏览
Swipe(x1=540, y1=1500, x2=540, y2=500)
```

---

## 七、常见问题

### Q1: Python 版本不匹配
Android-MCP 严格要求 Python 3.13。使用 uvx 时指定版本：
```
"args": ["--python", "3.13", "android-mcp"]
```

### Q2: uiautomator2 连接失败
u2 需要在设备端安装 ATX Agent。首次使用时 u2 会自动安装。如果失败：
```bash
python -m uiautomator2 init
```

### Q3: 设备无响应
检查 ATX Agent 是否在设备上运行。重启 ATX：
```bash
adb shell am start -n com.github.uiautomator/.MainActivity
```

### Q4: 截图太大/token 消耗高
启用截图量化：
```json
"env": {
  "SCREENSHOT_QUANTIZED": "true"
}
```

### Q5: Selector 找不到元素
- 先用 `Snapshot` 查看当前 UI 树，确认元素的 text/resourceId
- resourceId 可以用短格式（如 `btn_login`），会自动扩展为完整格式
- 增加 `timeout` 参数等待元素加载

### Q6: WiFi ADB 连接不上
```bash
# 先通过 USB 连接，然后切换到 WiFi
adb tcpip 5555
adb connect 192.168.1.3:5555
```

---

## 八、与其他项目的选型建议

| 场景 | 推荐项目 | 原因 |
|------|----------|------|
| 需要 Selector 元素查找 | **Android-MCP (CursorTouch)** | u2 原生 Selector + WaitForElement |
| 需要 iOS 支持 | mobile-mcp / appium-mcp | Android-MCP 仅支持 Android |
| Python MCP 生态 | **Android-MCP (CursorTouch)** | FastMCP 框架，Python 原生 |
| Node.js MCP 生态 | android-mcp-server / mobile-mcp | TypeScript 实现 |
| 自治 Agent 需求 | droidrun / AppAgent | Android-MCP 是被动工具 |
| 最轻量 MCP | android-mcp-server | 仅 ~200 行代码 |
| 最可靠的元素操作 | **Android-MCP (CursorTouch)** / appium-mcp | Selector 比坐标更稳定 |
| 模型训练/研究 | Open-AutoGLM / AppAgent | Android-MCP 不含 Agent 逻辑 |

### 核心差异

- **Android-MCP (CursorTouch)** 的独特价值在于 **uiautomator2 集成**：提供了 Selector 查找、等待元素、拖拽等其他 MCP Server 缺少的能力
- 如果只需基础操作且追求极简，选 **android-mcp-server**
- 如果需要跨平台，选 **mobile-mcp** 或 **appium-mcp**
- 如果需要 Agent 自主决策，选 **droidrun** 或 **AppAgent**

---

## 九、uiautomator2 补充说明

Android-MCP 底层使用的 [uiautomator2](https://github.com/openatx/uiautomator2) 是一个成熟的 Android 自动化库：

- **ATX Agent**：安装在设备端的服务进程，通过 HTTP 与主机通信
- **无需 root**：通过 ADB 安装 Agent，不需要 root 权限
- **高性能**：连接复用，操作延迟 2-4 秒
- **Selector API**：支持 text, resourceId, className, description, xpath 等多种选择器
- **自动安装**：首次连接时自动安装 ATX Agent 到设备

与 Appium 的区别：u2 更轻量（无需 Java/Node.js Server），但只支持 Android。
