# Droidrun 源码解读

## 1. 项目定位

**Droidrun** 是一个通过 LLM Agent 控制 Android 和 iOS 设备的完整框架。与 android-mcp-server、appium-mcp 等 MCP 工具服务器不同，Droidrun 是一个**自主 Agent 系统**，内置了规划、推理、执行的完整 Agent Loop，并支持多种 LLM 供应商（OpenAI、Anthropic、Gemini、Ollama、DeepSeek 等）。

核心特征：
- **自主 Agent**：不是 MCP 工具服务器，而是自带 LLM 推理循环的完整 Agent 框架
- **双平台支持**：Android（ADB + Portal App）+ iOS（HTTP REST Portal）
- **双模式架构**：FastAgent（直接执行）和 Manager+Executor（规划+执行）
- **Portal 架构**：需要在设备上安装 Droidrun Portal App 作为 Accessibility Service 桥接层
- **多 LLM 供应商**：基于 LlamaIndex 统一接入 6+ 种 LLM 供应商
- **Stealth 模式**：模拟人类操作的 Bezier 曲线滑动和随机延迟输入
- **企业级特性**：Trajectory 录制、Arize Phoenix 追踪、遥测、MCP 客户端集成

---

## 2. 架构总览

```
用户 (CLI / TUI / SDK)
    |
    | "open settings and turn on dark mode"
    |
    v
+------------------------------------------------------+
| DroidAgent (Workflow 协调器)                           |
|                                                      |
|  ┌──────────────┐    ┌──────────────┐                |
|  │ ManagerAgent │───>│ ExecutorAgent │    reasoning=True 模式   |
|  │ (规划子目标)  │<───│ (执行单步动作) │                |
|  └──────────────┘    └──────────────┘                |
|                                                      |
|  ┌──────────────┐                                    |
|  │  FastAgent   │    reasoning=False 模式 (直接执行)  |
|  │ (ReAct Loop) │                                    |
|  └──────────────┘                                    |
+------------------------------------------------------+
    |
    | ToolRegistry.execute(action_name, args, ctx)
    |
    v
+------------------------------------------------------+
| DeviceDriver 抽象层                                   |
|  ├─ AndroidDriver (ADB + Portal)                     |
|  ├─ IOSDriver (HTTP REST)                            |
|  ├─ StealthDriver (人类模拟装饰器)                    |
|  └─ RecordingDriver (轨迹录制装饰器)                  |
+------------------------------------------------------+
    |
    | ADB / HTTP
    |
    v
+------------------------------------------------------+
| 设备端: Droidrun Portal App                           |
|  ├─ Accessibility Service (UI 树获取)                 |
|  ├─ HTTP Server (TCP 通道)                            |
|  ├─ Content Provider (ADB 通道, 降级方案)             |
|  └─ Droidrun Keyboard (文本输入)                      |
+------------------------------------------------------+
    |
    v
Android / iOS 设备
```

### 核心分层

| 层次 | 模块 | 职责 |
|------|------|------|
| **CLI/TUI** | `droidrun/cli/` | 命令行入口、TUI 界面、配置向导 |
| **Agent 层** | `droidrun/agent/` | DroidAgent 协调器、Manager/Executor/FastAgent 子 Agent |
| **工具注册** | `droidrun/agent/tool_registry.py` | 统一工具注册表，支持动态注册/禁用 |
| **Driver 层** | `droidrun/tools/driver/` | 设备 I/O 抽象（Android/iOS/Stealth/Recording） |
| **UI 状态** | `droidrun/tools/ui/` | StateProvider + Filter + Formatter |
| **Portal 通信** | `droidrun/tools/android/portal_client.py` | TCP/ContentProvider 双通道通信 |
| **配置管理** | `droidrun/config_manager/` | YAML 配置加载、迁移、凭证管理 |
| **遥测** | `droidrun/telemetry/` | PostHog + Langfuse + Arize Phoenix |

---

## 3. 核心设计模式

### 3.1 DroidAgent 双模式工作流

DroidAgent 继承 LlamaIndex 的 `Workflow`，通过事件驱动实现两种模式：

```
伪代码:
class DroidAgent(Workflow):
    @step
    start_handler(ev: StartEvent):
        # 1. 创建 Driver (AndroidDriver / IOSDriver)
        # 2. 创建 StateProvider (Android/iOS)
        # 3. 构建 ToolRegistry
        # 4. 创建 ActionContext
        # 5. 装配子 Agent

        if reasoning == False:
            return FastAgentExecuteEvent  # 直接执行模式
        else:
            return ManagerInputEvent     # 规划+执行模式

    # ---- reasoning=False: FastAgent 直接执行 ----
    @step execute_task(ev: FastAgentExecuteEvent) -> FastAgentResultEvent
    @step handle_fast_agent_result(ev: FastAgentResultEvent) -> FinalizeEvent

    # ---- reasoning=True: Manager + Executor 循环 ----
    @step run_manager(ev: ManagerInputEvent) -> ManagerPlanEvent | FinalizeEvent
    @step handle_manager_plan(ev: ManagerPlanEvent) -> ExecutorInputEvent | FinalizeEvent
    @step run_executor(ev: ExecutorInputEvent) -> ExecutorResultEvent
    @step handle_executor_result(ev: ExecutorResultEvent) -> ManagerInputEvent  # 循环回 Manager

    @step finalize(ev: FinalizeEvent) -> ResultEvent
```

**reasoning=True 模式**循环：
```
Manager (规划) → Executor (执行单步) → Manager (评估+规划) → Executor → ... → 完成
```

**reasoning=False 模式**：FastAgent 使用 ReAct 循环直接执行，不需要 Manager。

### 3.2 FastAgent — XML Tool Calling

FastAgent 使用自定义的 XML 工具调用协议（类似 Anthropic 格式）：

```
伪代码:
class FastAgent(Workflow):
    @step prepare_chat:
        构建 system prompt（包含工具定义 XML）
        构建 user prompt（任务目标）

    @step handle_llm_input:
        # 1. 截图（如果 vision 开启）
        # 2. 获取 UI 状态 (state_provider.get_state())
        # 3. 将 device_state + screenshot 注入最后一条 user message
        # 4. 调用 LLM
        # 5. 解析 XML 工具调用: parse_tool_calls(response_text)

    @step handle_llm_output:
        if has_tool_calls:
            return FastAgentToolCallEvent
        else:
            提示 LLM "请提供工具调用"

    @step execute_code:
        for call in tool_calls:
            result = registry.execute(call.name, call.parameters, action_ctx)
        if shared_state.finished:  # complete() 工具被调用
            return FastAgentEndEvent

    @step handle_execution_result:
        将工具结果追加到 message_history
        return FastAgentInputEvent  # 循环回 LLM
```

LLM 输出格式：
```xml
<function_calls>
<invoke name="tap">
<parameter name="element_index">3</parameter>
</invoke>
</function_calls>
```

### 3.3 Portal 双通道通信

Android 设备通信通过 **Droidrun Portal App** 中转，支持两种通道：

```
伪代码:
class PortalClient:
    tcp_available: bool

    connect():
        if prefer_tcp:
            1. 从 ContentProvider 获取 auth_token
            2. 查找已有的 ADB port forward
            3. 没有则创建新的 port forward
            4. 测试 HTTP 连接
            5. 设置 tcp_available = True

    get_state():
        if tcp_available:
            return HTTP GET /state_full  # 快速通道
        else:
            return adb shell "content query --uri content://com.droidrun.portal/state_full"  # 降级通道

    input_text(text):
        if tcp_available:
            return HTTP POST /keyboard/input {base64_text: encode(text)}  # 支持中文
        else:
            return adb shell "content insert --uri .../keyboard/input --bind base64_text:s:..."

    take_screenshot():
        if tcp_available:
            return HTTP GET /screenshot  # 通过 Portal 截图（可隐藏 overlay）
        else:
            return adb screencap  # 降级到 ADB 原生截图
```

**关键点**：
- TCP 通道通过 ADB port forward 建立，速度更快
- ContentProvider 通道无需端口转发，但延迟更高
- Auth Token 通过 ContentProvider 安全通道获取，用于 HTTP 认证
- 401/403 时自动重新获取 token

### 3.4 DeviceDriver 抽象与装饰器模式

```
DeviceDriver (基类)
    ├── AndroidDriver  → ADB + PortalClient
    ├── IOSDriver      → HTTP REST
    ├── StealthDriver(inner)  → Bezier 曲线滑动 + 随机延迟输入 (装饰器)
    └── RecordingDriver(inner) → 记录所有操作轨迹 (装饰器)
```

装饰器可以组合：
```python
driver = AndroidDriver(serial)
driver = StealthDriver(driver)    # 包装隐身模式
driver = RecordingDriver(driver)  # 再包装录制
```

### 3.5 StealthDriver — 人类行为模拟

```
伪代码:
class StealthDriver:
    async swipe(x1, y1, x2, y2, duration_ms):
        # 不使用 input swipe，改用 motionevent 序列
        path = generate_curved_path(x1, y1, x2, y2)  # Bezier 曲线 + Perlin 微抖动
        shell("input motionevent DOWN x0 y0")
        for point in path:
            sleep(duration / len(path))
            shell("input motionevent MOVE x y")
        shell("input motionevent UP xn yn")

    async input_text(text):
        for word in text.split(" "):
            inner.input_text(word)
            inner.input_text(" ")
            sleep(random(0.1, 0.3))  # 词间随机延迟
```

### 3.6 UI 状态处理链

```
DeviceDriver.get_ui_tree()
    → raw a11y_tree + phone_state + device_context

StateProvider.get_state()
    → TreeFilter.filter()   # ConciseFilter (vision模式) 或 DetailedFilter
    → TreeFormatter.format() # IndexedFormatter (带索引号)
    → UIState(elements, formatted_text, focused_text, phone_state)
```

**Filter 策略**：
- `ConciseFilter`：vision 模式，过滤掉大量 UI 节点，只保留关键交互元素
- `DetailedFilter`：无 vision 模式，保留更多信息

**Formatter**：
- `IndexedFormatter`：给每个元素分配索引号（如 `[3] Button "Save"`），方便 LLM 通过 `element_index` 引用

### 3.7 ToolRegistry — 统一工具注册

```
伪代码:
class ToolRegistry:
    tools: Dict[name -> ToolEntry(fn, params, description, deps)]

    register(name, fn, params, description, deps):
        tools[name] = ToolEntry(...)

    disable_unsupported(capabilities):
        # 根据 driver.supported 和 state_provider.supported 自动禁用不支持的工具
        for tool in tools:
            if tool.deps not in capabilities:
                remove(tool)

    execute(name, args, action_ctx):
        entry = tools[name]
        result = await entry.fn(**args, ctx=action_ctx)
        return ActionResult(success, summary)

    get_tool_descriptions_xml():  # FastAgent 用
        return "<functions><function>...</function></functions>"

    get_signatures():  # Executor 用
        return {name: {parameters, description}}
```

**动态能力适配**：ToolRegistry 根据 `driver.supported` 集合自动禁用当前设备不支持的工具。例如 iOS 不支持 `drag`，该工具会被自动禁用。

---

## 4. 底层原子操作

### Android (通过 ADB + Portal)

| 操作 | 通信方式 | 命令/API |
|------|----------|----------|
| 点击 | ADB (async_adbutils) | `device.click(x, y)` |
| 滑动 | ADB | `device.swipe(x1, y1, x2, y2, duration)` |
| 按键 | ADB | `device.keyevent(keycode)` |
| 文本输入 | Portal (TCP/CP) | `POST /keyboard/input {base64_text}` |
| 截图 | Portal (TCP) 或 ADB | `GET /screenshot` 或 `device.screenshot_bytes()` |
| UI 树 | Portal (TCP/CP) | `GET /state_full` 或 `content query ...state_full` |
| 启动应用 | ADB | `device.app_start(package, activity)` |
| 安装 APK | ADB | `device.install(path, flags=["-g"])` |
| 获取日期 | ADB | `device.shell("date")` |
| 获取应用列表 | Portal (CP) | `content query ...packages` |
| 解析 Activity | ADB | `device.shell("cmd package resolve-activity --brief")` |
| Stealth 滑动 | ADB | `input motionevent DOWN/MOVE/UP` 序列 |

### iOS (通过 HTTP REST)

| 操作 | API 端点 | 说明 |
|------|----------|------|
| 点击 | `POST /gestures/tap` | `{rect, count, longPress}` |
| 滑动 | `POST /gestures/swipe` | `{x1, y1, x2, y2, durationMs}` |
| 文本输入 | `POST /inputs/type` | `{text, clear}` |
| Home 键 | `POST /inputs/key` | `{key: 1}` |
| 启动应用 | `POST /inputs/launch` | `{bundleIdentifier}` |
| 截图 | `GET /vision/screenshot` | 返回 PNG bytes |
| UI 树 | `GET /state` | 返回 a11y_tree + phone_state |
| 获取日期 | `GET /device/date` | 返回 `{date: "..."}` |

---

## 5. LLM 接入架构

通过 LlamaIndex 统一适配：

```
支持的 LLM 供应商:
├── OpenAI (llama-index-llms-openai)
├── Anthropic (llama-index-llms-anthropic) [可选依赖]
├── Google Gemini (llama-index-llms-google-genai)
├── Ollama (llama-index-llms-ollama, 本地部署)
├── DeepSeek (llama-index-llms-deepseek) [可选依赖]
├── OpenRouter (llama-index-llms-openrouter)
├── MiniMax (llama-index-llms-minimax)
└── OpenAI-Like (llama-index-llms-openai-like, 兼容 API)
```

**多 LLM 配置**（reasoning=True 模式）：
```python
llms = {
    "manager": Gemini_Flash,        # 规划用低成本模型
    "executor": GPT_4o_mini,        # 执行用另一个模型
    "fast_agent": Claude_Sonnet,    # 直接执行模式
    "app_opener": cheap_model,      # 启动应用辅助
    "structured_output": same_model # 结构化输出提取
}
```

**认证方式**：
- API Key（传统方式）
- OAuth（OpenAI、Gemini、Anthropic 均支持 OAuth 登录）

---

## 6. 关键代码走读

### 6.1 DroidAgent 初始化流程 (`droid_agent.py:121-312`)

```python
def __init__(self, goal, config, llms, ...):
    # 1. 初始化共享状态
    self.shared_state = DroidAgentState(instruction=goal)

    # 2. 初始化 Prompt Resolver（支持自定义 prompt）
    self.prompt_resolver = PromptResolver(custom_prompts=prompts)

    # 3. 加载凭证管理器
    self.credential_manager = FileCredentialManager(credentials_source)

    # 4. 加载 LLM（从 config 的 llm_profiles 或直接传入）
    llms = load_agent_llms(config=self.config, ...)

    # 5. 创建子 Agent（Manager + Executor 或 FastAgent）
    if reasoning:
        self.manager_agent = ManagerAgent(llm=self.manager_llm, ...)
        self.executor_agent = ExecutorAgent(llm=self.executor_llm, ...)

    # 6. 初始化 Trajectory（轨迹录制）
    self.trajectory = Trajectory(goal=goal)
```

### 6.2 Portal 状态获取与重试 (`provider.py:35-122`)

```python
async def fetch_state_with_retry(fetch, recovery, max_retries=7):
    """带重试、退避和中途恢复的状态获取"""
    for attempt in range(max_retries):
        try:
            data = await fetch()  # driver.get_ui_tree()
            # 校验必需字段
            required = ["a11y_tree", "phone_state", "device_context"]
            missing = [k for k in required if k not in data]
            if missing: raise Exception(...)
            return data

        except DeviceDisconnectedError: raise  # 设备断连立即终止
        except Exception:
            # 第 5 次失败后触发恢复（重启 Accessibility Service + TCP Server）
            if attempt + 1 >= recovery_after and not recovery_attempted:
                await recovery()  # _recover_portal()
            await asyncio.sleep(delays[attempt])  # 1, 2, 3, 5, 8, 10 秒退避
```

### 6.3 ToolRegistry 能力过滤 (`tool_registry.py:71-83`)

```python
def disable_unsupported(self, capabilities: Set[str]):
    """根据设备能力集自动禁用不支持的工具"""
    to_remove = [
        name for name, entry in self.tools.items()
        if entry.deps is not None and not entry.deps <= capabilities
    ]
    self.disable(to_remove)

# 使用:
capabilities = driver.supported | state_provider.supported
# AndroidDriver.supported = {"tap", "swipe", "input_text", "screenshot", "get_ui_tree", ...}
# IOSDriver.supported = {"tap", "swipe", "input_text", "screenshot", ...}  (没有 drag)
registry.disable_unsupported(capabilities)
```

---

## 7. 与 mobile-mcp、android-mcp-server、appium-mcp、Open-AutoGLM 的对比

### 7.1 架构本质对比

| 维度 | Droidrun | mobile-mcp | android-mcp-server | appium-mcp | Open-AutoGLM |
|------|----------|------------|-------------------|------------|--------------|
| **本质** | 自主 Agent 框架 | MCP 工具服务器 | MCP 工具服务器 | MCP 工具服务器 | 自主 Agent |
| **LLM 集成** | 内置（LlamaIndex） | 无（由 MCP 客户端提供） | 无 | 无 | 内置（AutoGLM-Phone-9B） |
| **Agent Loop** | 内置（Manager/Executor/FastAgent） | 无 | 无 | 无 | 内置（Step Loop） |
| **语言** | Python | TypeScript | TypeScript | Python | Python |
| **平台** | Android + iOS | Android + iOS | Android Only | Android + iOS | Android + HarmonyOS + iOS |
| **设备通信** | ADB + Portal App | 直连 ADB + iOS 工具链 | 直连 ADB | Appium Server | 直连 ADB + WDA + HDC |
| **中间层** | Portal App (Accessibility) | 无 | 无 | Appium Server | 无 |

### 7.2 设备端需求对比

| 项目 | 需要安装额外 App | 需要外部服务 | 设备端依赖 |
|------|-----------------|-------------|-----------|
| **Droidrun** | Portal App + Droidrun Keyboard | 无 | Accessibility Service 开启 |
| **mobile-mcp** | 无 | 无 | 开发者选项 + ADB |
| **android-mcp-server** | 无 | 无 | 开发者选项 + ADB |
| **appium-mcp** | 无（或 UiAutomator2） | Appium Server | 开发者选项 + ADB |
| **Open-AutoGLM** | ADB Keyboard | VLM API | 开发者选项 + ADB |

### 7.3 UI 分析能力对比

| 维度 | Droidrun | android-mcp-server | Open-AutoGLM |
|------|----------|-------------------|--------------|
| **UI 树来源** | Portal Accessibility Service | `uiautomator dump` | 截图 + VLM 视觉理解 |
| **UI 树质量** | 高（a11y_tree + phone_state + device_context） | 中（正则解析 XML） | N/A（纯视觉） |
| **元素索引** | 有（IndexedFormatter） | 有（数组索引） | 无（坐标定位） |
| **元素过滤** | 有（ConciseFilter / DetailedFilter） | 有（过滤无交互节点） | N/A |
| **视觉分析** | 支持（截图 + LLM vision） | 支持（返回 base64 图片） | 核心方式 |

### 7.4 核心技术对比

| 特性 | Droidrun | mobile-mcp | android-mcp-server | appium-mcp | Open-AutoGLM |
|------|----------|------------|-------------------|------------|--------------|
| **持久化连接** | Portal TCP + ADB | 无 | Persistent Shell | Appium Session | 无 |
| **截图压缩** | Portal 端处理 | 基础 | Sharp (1280px) | Appium 默认 | 无 |
| **文本输入** | 专用键盘 App (支持中文) | ADB input | ADB input (不支持中文) | Appium API | ADB Keyboard (支持中文) |
| **Stealth 模式** | Bezier 曲线 + 随机延迟 | 无 | 无 | 无 | 无 |
| **轨迹录制** | 完整 (RecordingDriver) | 无 | 无 | 无 | 无 |
| **Macro 回放** | 支持 | 无 | 无 | 无 | 无 |
| **多 LLM** | 不同 Agent 可用不同 LLM | N/A | N/A | N/A | 固定 AutoGLM |
| **可观测性** | Arize Phoenix + Langfuse + PostHog | 无 | 无 | 无 | 无 |
| **MCP 客户端** | 可作为 MCP 客户端连接外部 MCP 工具 | 本身是 MCP 服务器 | 本身是 MCP 服务器 | 本身是 MCP 服务器 | 无 |

### 7.5 代码规模对比

| 项目 | 核心文件数 | 核心代码行数 | 复杂度 |
|------|-----------|-------------|--------|
| **Droidrun** | ~60+ | ~8000+ | 高（完整 Agent 框架） |
| **mobile-mcp** | ~10-15 | ~3000+ | 中 |
| **android-mcp-server** | 2 | ~1200 | 低 |
| **appium-mcp** | ~10 | ~2000+ | 中 |
| **Open-AutoGLM** | ~15 | ~3000+ | 中 |

### 7.6 适用场景建议

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 需要独立运行的 Agent（不依赖 MCP 客户端） | **Droidrun** | 内置完整 Agent Loop |
| 需要 MCP 工具给 AI IDE 用 | **android-mcp-server / mobile-mcp** | 标准 MCP 服务器 |
| 需要 Stealth / 反检测 | **Droidrun** | Bezier 曲线 + 随机延迟 |
| 需要操作回放 / 自动化脚本 | **Droidrun** | Macro 录制+回放 |
| 纯 Android、追求极简 | **android-mcp-server** | 一行 npx 启动 |
| 已有 Appium 设施 | **appium-mcp** | 复用 Appium 生态 |
| 需要自定义 VLM | **Open-AutoGLM** | 内置 AutoGLM-Phone-9B |
| 跨 Android + iOS | **Droidrun / mobile-mcp** | 均支持双平台 |
| 企业级可观测需求 | **Droidrun** | Phoenix + Langfuse + PostHog |
| Benchmark 达标(91.4%) | **Droidrun** | 官方公布的 benchmark 成绩 |

---

## 8. 代码质量评估

**优点：**
- 架构清晰，层次分明（Agent → ToolRegistry → Driver → Portal）
- 装饰器模式优雅组合 Stealth/Recording 功能
- ToolRegistry 的能力过滤机制（`disable_unsupported`）自动适配不同平台
- Portal 双通道（TCP + ContentProvider）设计兼顾性能和兼容性
- 状态获取的重试+恢复策略健壮
- 多 LLM 按角色分配，可为不同子 Agent 选择不同模型
- 完整的可观测性（Tracing + Telemetry + Trajectory）

**局限：**
- 需要在设备上安装 Portal App（增加了部署复杂度）
- Portal App 需要 Accessibility Service 权限（某些设备可能受限）
- 依赖链较重（LlamaIndex + httpx + arize-phoenix + textual + mcp + ...）
- Python 3.14 不支持
- iOS 支持仍有限制（无法检测前台应用、应用列表为硬编码）
- 不是 MCP 服务器，无法被 AI IDE 直接调用（需要通过 CLI/SDK）

---

## 9. 总结

Droidrun 是这五个项目中架构最完整、功能最丰富的方案。它不是一个简单的 MCP 工具服务器，而是一个自带规划和推理能力的完整 Agent 框架。Portal App 架构虽然增加了部署步骤，但换来了更好的 UI 树质量、中文输入支持和隐身操作能力。适合需要独立运行 Agent、有企业级可观测需求、或者需要 Stealth 模式的场景。如果你只需要给 AI IDE 提供设备控制工具，那 android-mcp-server 或 mobile-mcp 更合适。
