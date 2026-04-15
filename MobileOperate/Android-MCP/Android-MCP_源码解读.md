# Android-MCP (CursorTouch) 源码解读

> 仓库地址：https://github.com/CursorTouch/Android-MCP
> 来源：CursorTouch 团队
> 语言：Python | 总文件数：10 个源码文件 | 代码量：~700 行
> 依赖：fastmcp, uiautomator2, pillow, tabulate

---

## 一、项目定位

Android-MCP 是一个基于 **FastMCP + uiautomator2** 的轻量级 MCP Server，作为 AI Agent 与 Android 设备之间的桥梁。与直接调用 ADB 命令不同，它通过 **uiautomator2** 库与设备交互，提供更高层的 API 抽象。

核心特点：
- **FastMCP 框架**：使用 Python 的 FastMCP 库快速构建 MCP Server
- **uiautomator2 通信**：通过 u2 库操控设备，而非直接执行 ADB 命令
- **UI 树解析 + 截图标注**：提供结构化的 UI 状态信息 + 可选的标注截图
- **懒连接**：Server 启动时不要求设备在线，工具调用时才连接
- **Selector 查找**：支持按 text/resourceId/className/description 查找元素

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────┐
│                     MCP 层                              │
│  __main__.py                                            │
│  ├── FastMCP Server 定义                                │
│  ├── 14 个 MCP Tool 注册                                │
│  ├── 设备选择/连接逻辑 (CLI + 环境变量)                   │
│  └── DevicePreference 数据类                            │
├─────────────────────────────────────────────────────────┤
│                     Mobile 层                           │
│  mobile/service.py   设备管理 + 截图 + 状态获取          │
│  mobile/views.py     MobileState / App 数据类           │
├─────────────────────────────────────────────────────────┤
│                     Tree 层                             │
│  tree/service.py     UI 树解析 + 截图标注                │
│  tree/views.py       ElementNode / TreeState 数据类      │
│  tree/config.py      交互元素类名白名单                   │
│  tree/utils.py       坐标提取工具函数                     │
├─────────────────────────────────────────────────────────┤
│                     底层                                │
│  uiautomator2        设备通信 (ATX Agent)                │
│  ADB                 设备发现/WiFi连接                    │
└─────────────────────────────────────────────────────────┘
```

### 数据流

```
MCP Client (Claude/Cursor)
    ↓ MCP 协议
FastMCP Server (__main__.py)
    ↓ require_device()
Mobile (mobile/service.py)
    ↓ uiautomator2
ATX Agent (设备端)
    ↓
Android 设备
```

---

## 三、核心模块详解

### 3.1 MCP 入口 (`__main__.py`)

#### 设备选择策略

支持多种设备指定方式，优先级从高到低：

1. `--wifi HOST` → WiFi ADB 连接
2. `--usb [SERIAL]` → USB 连接
3. `--device SERIAL` → 指定设备序列号
4. `ANDROID_MCP_DEVICE` 环境变量
5. `ANDROID_MCP_CONNECTION` + `ANDROID_MCP_HOST` 环境变量
6. 自动检测第一个可用设备

```python
@dataclass(frozen=True)
class DevicePreference:
    connection: str = "auto"     # auto / usb / wifi
    serial: Optional[str] = None
    source: str = "auto-detect"
```

#### MCP Tool 列表

共注册 **14 个 Tool**：

| Tool | 类型 | 说明 |
|------|------|------|
| `ListDevices` | 只读 | 列出 ADB 设备 |
| `ConnectDevice` | 操作 | 连接指定设备 |
| `Device` | 操作 | 统一设备管理 (list/connect/disconnect) |
| `Click` | 操作 | 点击坐标 (x, y) |
| `ClickBySelector` | 操作 | 按选择器点击元素 (text/resourceId/className/description) |
| `Snapshot` | 只读 | 获取设备状态 (UI 树 + 可选截图) |
| `LongClick` | 操作 | 长按坐标 |
| `Swipe` | 操作 | 滑动 (x1,y1) → (x2,y2) |
| `Type` | 操作 | 在坐标处输入文本 |
| `Drag` | 操作 | 拖拽 |
| `Press` | 操作 | 按键 (Back, Home 等) |
| `Notification` | 操作 | 打开通知栏 |
| `Wait` | 操作 | 等待指定秒数 |
| `WaitForElement` | 只读 | 等待元素出现 (Selector + timeout) |

关键设计：
- 使用 `ToolAnnotations` 标注 `readOnlyHint` 和 `destructiveHint`
- `require_device()` 实现懒连接，首次工具调用时才连接设备
- `ClickBySelector` 支持 resourceId 自动扩展（短 ID → 完整包名:id/短ID）

### 3.2 Mobile 服务层 (`mobile/service.py`)

#### 设备管理

```python
class Mobile:
    def connect(self, serial: str):
        self.device = u2.connect(serial)  # uiautomator2 连接
    
    def disconnect(self):
        self.device = None
```

- 通过 `uiautomator2.connect()` 建立连接
- 连接后 `self.device` 即为 u2 设备对象，可直接调用 `.click()`, `.swipe()` 等方法
- `list_devices()` 和 `adb_connect()` 仍使用 `subprocess.run(['adb', ...])` 直接调用 ADB

#### 截图与状态

```python
def capture_data(self, use_vision: bool = True):
    # 并行获取 XML dump 和截图
    data = {}
    threads = [threading.Thread(target=get_xml)]
    if use_vision:
        threads.append(threading.Thread(target=get_img))
    # ...
```

- **并行采集**：XML dump 和截图通过 threading 并行获取，减少延迟
- **截图量化**：`SCREENSHOT_QUANTIZED` 环境变量可启用 256 色量化，减少 token 消耗
- **格式支持**：截图可输出为 bytes / base64 / PIL Image

### 3.3 Tree 层 (`tree/service.py`)

#### UI 树解析

```python
class Tree:
    def get_interactive_elements(self, xml_data=None) -> list:
        element_tree = self.get_element_tree(xml_data=xml_data)
        nodes = element_tree.findall('.//node[@enabled="true"]')
        for node in nodes:
            if self.is_interactive(node):
                # 提取坐标、名称、resourceId
                interactive_elements.append(ElementNode(...))
```

交互性判断条件（`is_interactive()`）：
- `focusable="true"`
- `clickable="true"`
- `long-clickable="true"`
- `checkable="true"`
- `scrollable="true"`
- `selected="true"`
- `password="true"`
- `class` 在 `INTERACTIVE_CLASSES` 白名单中

白名单包括：Button, ImageButton, EditText, CheckBox, Switch, RadioButton, Spinner, SeekBar

#### 元素名称提取

`get_element_name()` 的智能策略：
1. 优先取 `content-desc` 或 `text`
2. 若无，递归子节点收集文本
3. 遇到可操作子节点时停止递归，将其文本作为 fallback
4. 合并所有文本作为元素名称

#### 截图标注

```python
def annotated_screenshot(self, nodes, scale=0.7, screenshot=None):
    draw = ImageDraw.Draw(screenshot)
    for i, node in enumerate(nodes):
        # 绘制 bounding box + 数字标签
        draw.rectangle(adjusted_box, outline=color, width=2)
        draw.text(..., str(label), fill=(255,255,255))
```

- 使用 Pillow 绘制，每个元素用随机颜色
- 标签放在 bounding box 右上角
- 支持缩放 (`scale` 参数)

### 3.4 数据模型 (`tree/views.py`, `mobile/views.py`)

```python
@dataclass
class ElementNode:
    name: str           # 元素文本/描述
    class_name: str     # Android 类名
    coordinates: CenterCord   # 中心坐标
    bounding_box: BoundingBox  # 边界框
    resource_id: str = ''      # 资源 ID (短格式)

@dataclass
class TreeState:
    interactive_elements: list[ElementNode]
    
    def to_string(self):
        # 输出为 tabulate 格式化的表格
        # Label | Name | ResourceId | Class | Coordinates

@dataclass
class MobileState:
    tree_state: TreeState
    screenshot: bytes | str | Image | None
```

TreeState 输出示例：
```
Label  Name              ResourceId      Class                          Coordinates
0      Settings          btn_settings    android.widget.Button          (540,120)
1      Search            search_input    android.widget.EditText        (540,240)
```

---

## 四、底层原子操作汇总

| 操作 | 实现方式 | 参数 |
|------|----------|------|
| 点击 | `u2.device.click(x, y)` | 坐标 |
| 长按 | `u2.device.long_click(x, y)` | 坐标 |
| 滑动 | `u2.device.swipe(x1,y1,x2,y2)` | 起止坐标 |
| 拖拽 | `u2.device.drag(x1,y1,x2,y2)` | 起止坐标 |
| 输入文本 | `u2.device.send_keys(text, clear)` | 文本 + 清除标志 |
| 按键 | `u2.device.press(button)` | 按键名称 |
| 通知栏 | `u2.device.open_notification()` | 无 |
| 等待 | `u2.device.sleep(duration)` | 秒数 |
| 截图 | `u2.device.screenshot(format="pillow")` | 格式 |
| UI dump | `u2.device.dump_hierarchy()` | 无 |
| Selector 点击 | `u2.device(**kwargs).click()` | 选择器参数 |
| Selector 等待 | `u2.device(**kwargs).wait(timeout)` | 超时时间 |
| 设备列表 | `subprocess: adb devices` | 无 |
| WiFi 连接 | `subprocess: adb connect` | host:port |

**关键区别**：与其他项目直接调用 ADB shell 不同，Android-MCP 通过 uiautomator2 的 ATX Agent 通信，操作延迟更低，API 更高层。

---

## 五、关键设计模式

### 5.1 FastMCP 框架集成

使用 Python FastMCP 库声明式定义 MCP Server：

```python
mcp = FastMCP(name="Android-MCP", instructions=instructions)

@mcp.tool(name="Click", description="...", annotations=ToolAnnotations(...))
def click_tool(x: int, y: int):
    device = require_device()
    device.click(x, y)
    return f"Clicked on ({x},{y})"
```

- 函数签名自动转为 MCP Tool Schema
- `ToolAnnotations` 标注工具特性 (只读/破坏性/幂等)
- 入口点通过 `pyproject.toml` 的 `[project.scripts]` 注册

### 5.2 懒连接模式

```python
mobile = Mobile()  # 启动时不连接

def require_device():
    _connect_preferred_device()  # 首次调用时连接
    return mobile.get_device()
```

Server 启动时不要求设备在线，MCP 握手不会因设备缺失而失败。

### 5.3 uiautomator2 vs 直接 ADB

| 维度 | uiautomator2 | 直接 ADB |
|------|-------------|----------|
| 通信 | HTTP (ATX Agent) | 子进程 ADB 命令 |
| 性能 | 连接复用，低延迟 | 每次新进程，高延迟 |
| API | Python 原生方法 | Shell 字符串拼接 |
| 元素查找 | 原生 Selector | 手动 XML 解析 |
| 依赖 | 需要设备端安装 ATX Agent | 仅需 ADB |

### 5.4 Selector + 坐标双模式

```python
# 坐标模式
@mcp.tool(name="Click")
def click_tool(x: int, y: int)

# Selector 模式
@mcp.tool(name="ClickBySelector")
def click_by_selector_tool(text=None, resourceId=None, className=None, description=None)
```

Selector 模式更可靠：
- 不受屏幕分辨率/布局变化影响
- 支持 `wait(timeout)` 等待元素出现
- resourceId 自动扩展短 ID

### 5.5 截图量化优化

```python
if os.getenv("SCREENSHOT_QUANTIZED") in ["1", "yes", "true", True]:
    screenshot = self.quantized_screenshot(screenshot)
```

将截图量化为 256 色 PNG，减少图片大小和 LLM token 消耗。

---

## 六、项目对比

### 6.1 与 mobile-mcp 对比

| 维度 | Android-MCP (CursorTouch) | mobile-mcp |
|------|--------------------------|------------|
| 语言 | Python (FastMCP) | TypeScript (MCP SDK) |
| 平台 | Android only | Android + iOS |
| 设备通信 | uiautomator2 (ATX Agent) | ADB + Maestro + idb |
| UI 解析 | XML 解析 + Selector API | XML 解析 + Accessibility |
| 截图 | Pillow 标注 + 量化 | Sharp 压缩 |
| Tool 数量 | 14 个 | 20+ 个 |
| Selector 查找 | 支持 (u2 原生) | 不支持 |
| 拖拽操作 | 支持 (Drag) | 不支持 |
| 应用管理 | 无 | 支持 (启动/安装) |
| 等待元素 | WaitForElement (Selector) | 无 |
| 安装方式 | uvx / uv (PyPI) | npm (Node.js) |

### 6.2 与 android-mcp-server 对比

| 维度 | Android-MCP (CursorTouch) | android-mcp-server |
|------|--------------------------|-------------------|
| 语言 | Python | TypeScript |
| 设备通信 | uiautomator2 | 直接 ADB 命令 |
| 代码量 | ~700 行 | ~200 行 |
| MCP 框架 | FastMCP (Python) | @anthropic-ai/sdk |
| 截图 | Pillow + 量化 | Sharp 压缩 |
| Shell 优化 | u2 连接复用 | 持久 Shell 会话 |
| UI 状态 | 表格格式 (tabulate) | 自定义格式 |
| 元素查找 | Selector API | 无 |
| 拖拽 | 支持 | 不支持 |
| 按键 | 支持任意 keyevent | 支持任意 keyevent |
| 设备发现 | 自动/手动/WiFi/USB | 自动/环境变量 |

### 6.3 与 appium-mcp 对比

| 维度 | Android-MCP (CursorTouch) | appium-mcp |
|------|--------------------------|------------|
| 底层引擎 | uiautomator2 (Python) | Appium (WebDriver) |
| 跨平台 | Android only | Android + iOS + Web |
| 元素查找 | u2 Selector (text/id/class) | XPath/ID/AccessibilityID |
| 等待机制 | WaitForElement + timeout | WebDriver 隐式/显式等待 |
| 设备端 | ATX Agent | UiAutomator2 Server |
| 协议 | HTTP (u2) | WebDriver 协议 |
| 复杂度 | 低 | 高 (Appium Server) |

### 6.4 与 Open-AutoGLM 对比

| 维度 | Android-MCP (CursorTouch) | Open-AutoGLM |
|------|--------------------------|--------------|
| 定位 | MCP 工具层 | 自治 Agent + 模型训练 |
| 智能层 | 外部 (MCP Client) | 内置 (ChatGLM) |
| 操作层 | uiautomator2 | Accessibility + 坐标 |
| 学习 | 无 | 强化学习 (阶段级奖励) |
| 运行需求 | Python + ADB | GPU 集群 + 自定义 ROM |
| 使用门槛 | 低 (uvx 一行安装) | 极高 |

### 6.5 与 AppAgent 对比

| 维度 | Android-MCP (CursorTouch) | AppAgent |
|------|--------------------------|----------|
| 定位 | MCP 工具层 | 自治 Agent (学术原型) |
| 智能层 | 外部 (MCP Client) | 内置 (GPT-4V/Qwen-VL) |
| 设备通信 | uiautomator2 | 直接 ADB 命令 |
| UI 解析 | XML + Selector | XML + 数字标注 |
| 文档系统 | 无 | 自动生成 UI 文档 |
| 探索学习 | 无 | 两阶段探索-利用 |
| 截图标注 | Pillow (随机色 bbox) | OpenCV (数字标签) |
| 工程化 | PyPI 包发布 | 学术脚本 |

### 6.6 与 droidrun 对比

| 维度 | Android-MCP (CursorTouch) | droidrun |
|------|--------------------------|----------|
| 定位 | MCP 工具层 | 自治 Agent 框架 |
| 设备通信 | uiautomator2 | 直接 ADB 命令 |
| Agent 模式 | 无 (被动工具) | Manager/Executor/FastAgent |
| Selector | u2 Selector | 无 |
| 拟人操作 | 无 | Stealth 模式 (Bezier 曲线) |
| Portal App | 无 | 设备端 Portal App |
| 多 LLM | 无 (由 Client 决定) | 6+ 种 LLM |
| 操作种类 | 14 个 Tool | 20+ 种操作 |

---

## 七、代码质量评估

### 优点
- **架构清晰**：MCP 层 / Mobile 层 / Tree 层分离良好
- **FastMCP 集成**：声明式 Tool 定义，代码简洁
- **uiautomator2**：比直接 ADB 更可靠、性能更好
- **Selector 支持**：`ClickBySelector` 和 `WaitForElement` 提升自动化可靠性
- **懒连接**：Server 启动不依赖设备在线
- **并行采集**：XML dump 和截图并行获取
- **截图量化**：可选的 token 优化
- **设备选择**：完善的 CLI + 环境变量 + 自动检测

### 局限
- **仅 Android**：不支持 iOS
- **依赖 ATX Agent**：uiautomator2 需要在设备端安装 ATX Agent
- **Python 版本限制**：`requires-python = ">=3.13,<3.14"`，仅支持 Python 3.13
- **无应用管理**：不能启动/安装/切换应用
- **无剪贴板操作**：缺少剪贴板读写
- **Type 工具设计**：接受 x, y 参数但实际未使用坐标（`send_keys` 不需要坐标）
- **随机标注颜色**：截图标注用随机颜色，可能可读性不一致

---

## 八、总结

Android-MCP (CursorTouch) 的核心价值在于**将 uiautomator2 的能力封装为 MCP Server**，提供了比直接 ADB 更高层、更可靠的操作接口。其 Selector 查找 + 等待元素的能力是其他 MCP 项目所缺乏的，使得自动化更加健壮。

与其他项目的核心差异：
- 与 mobile-mcp：Android-MCP 用 u2 而非 ADB，有 Selector 能力，但缺少 iOS 和应用管理
- 与 android-mcp-server：同为 MCP Server，但底层引擎不同（u2 vs ADB），功能更丰富
- 与 AppAgent/droidrun/Open-AutoGLM：Android-MCP 是被动工具，它们是自治 Agent
- **最佳使用场景**：需要可靠的 Android 元素查找能力、且仅需 Android 平台的 MCP 集成
