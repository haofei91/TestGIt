# Droidrun 快速上手

## 1. 项目简介

Droidrun 是一个通过 LLM Agent 控制 Android 和 iOS 设备的框架。你可以用自然语言告诉它要做什么，它会自动规划并在设备上执行操作。支持 OpenAI、Anthropic、Gemini、Ollama、DeepSeek 等多种 LLM 供应商。

**核心区别**：Droidrun 不是 MCP 工具服务器，而是一个自带 LLM 推理循环的完整 Agent 系统，可以独立运行。

---

## 2. 环境要求

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| Python | 3.11 - 3.13 | **不支持 Python 3.14** |
| ADB | - | Android SDK platform-tools |
| Android 设备/模拟器 | - | 已开启 USB 调试 |
| iOS 设备（可选） | - | 需要 Droidrun iOS Portal App |
| LLM API Key | - | OpenAI / Anthropic / Gemini / DeepSeek 等 |

---

## 3. 安装

### 基础安装

```bash
pip install droidrun
```

### 安装可选 LLM 支持

```bash
# Anthropic (Claude) 支持
pip install droidrun[anthropic]

# DeepSeek 支持
pip install droidrun[deepseek]

# 开发模式
pip install droidrun[dev]
```

---

## 4. 三步快速上手

### Step 1: 安装 Portal App 到设备

```bash
droidrun setup
```

这会自动下载并安装 Droidrun Portal APK 到你的 Android 设备，并启用 Accessibility Service。

**Portal App 的作用**：
- 提供 Accessibility Service 获取完整 UI 树
- 提供 Droidrun Keyboard 支持中文输入
- 提供 HTTP Server 快速通信通道

### Step 2: 配置 LLM

```bash
droidrun configure
```

交互式向导会引导你选择：
1. LLM 供应商（Gemini、OpenAI、Anthropic 等）
2. 认证方式（API Key 或 OAuth）
3. 模型选择

### Step 3: 运行

```bash
droidrun run "open settings and turn on dark mode"
```

---

## 5. CLI 命令一览

### 核心命令

| 命令 | 说明 |
|------|------|
| `droidrun run "指令"` | 执行自然语言命令 |
| `droidrun setup` | 安装 Portal App |
| `droidrun configure` | 配置 LLM 供应商 |
| `droidrun devices` | 列出已连接设备 |
| `droidrun ping` | 检查 Portal 是否就绪 |
| `droidrun doctor` | 诊断系统健康 |
| `droidrun tui` | 启动终端 UI |
| `droidrun connect <serial>` | TCP 连接设备 |
| `droidrun disconnect <serial>` | 断开连接 |

### droidrun run 参数

```bash
droidrun run "指令" [选项]
```

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--config, -c` | 自定义配置文件路径 | `~/.droidrun/config.yaml` |
| `--device, -d` | 指定设备序列号 | 自动选择第一个 |
| `--provider, -p` | LLM 供应商 | 配置文件中的值 |
| `--model, -m` | LLM 模型名 | 配置文件中的值 |
| `--steps` | 最大步数 | 配置文件中的值 |
| `--vision/--no-vision` | 启用截图分析 | 配置文件中的值 |
| `--reasoning/--no-reasoning` | 启用规划推理 | 配置文件中的值 |
| `--stream/--no-stream` | 流式输出 LLM 响应 | 配置文件中的值 |
| `--tracing/--no-tracing` | Arize Phoenix 追踪 | 配置文件中的值 |
| `--debug/--no-debug` | 详细调试日志 | false |
| `--tcp/--no-tcp` | TCP 通信模式 | false |
| `--save-trajectory` | 轨迹保存级别 (none/step/action) | none |
| `--ios` | iOS 设备模式 | false |
| `--temperature` | LLM 温度参数 | 默认值 |

### OAuth 认证

```bash
# OpenAI OAuth 登录
droidrun openai login

# Gemini OAuth 登录
droidrun gemini login

# Anthropic 登录
droidrun anthropic login
```

### Macro 宏命令

```bash
# 回放宏文件
droidrun macro replay <file.json>

# 回放目录下所有宏
droidrun macro replay-folder <dir>
```

---

## 6. SDK / Python API 使用

### 最简示例

```python
import asyncio
from droidrun import DroidAgent, load_llm

async def main():
    llm = load_llm("OpenAI", model="gpt-4o")

    agent = DroidAgent(
        goal="open settings and search for display",
        llms=llm,
    )

    handler = agent.run()
    async for event in handler.stream_events():
        print(event)  # 实时事件

    result = await handler
    print(f"Success: {result.success}, Reason: {result.reason}")

asyncio.run(main())
```

### 使用配置文件

```python
from droidrun import DroidAgent, DroidConfig
from droidrun.config_manager import ConfigLoader

config = ConfigLoader.load("config.yaml")

agent = DroidAgent(
    goal="send a message to John",
    config=config,
)
```

### 使用不同 LLM 分配

```python
agent = DroidAgent(
    goal="book a flight",
    llms={
        "manager": load_llm("GoogleGenAI", model="gemini-2.0-flash"),
        "executor": load_llm("OpenAI", model="gpt-4o-mini"),
        "fast_agent": load_llm("Anthropic", model="claude-sonnet-4-20250514"),
    },
)
```

### 自定义工具

```python
async def my_custom_tool(query: str, ctx=None):
    return f"Custom result for: {query}"

agent = DroidAgent(
    goal="do something",
    llms=llm,
    custom_tools={
        "my_tool": {
            "function": my_custom_tool,
            "parameters": {
                "query": {"type": "string", "description": "The query"}
            },
            "description": "A custom tool",
        }
    },
)
```

### 结构化输出

```python
from pydantic import BaseModel

class FlightInfo(BaseModel):
    airline: str
    departure: str
    arrival: str
    price: float

agent = DroidAgent(
    goal="find the cheapest flight from NYC to LA",
    llms=llm,
    output_model=FlightInfo,
)

result = await agent.run()
if result.structured_output:
    print(result.structured_output)  # FlightInfo 实例
```

---

## 7. 两种工作模式

### FastAgent 模式 (reasoning=False, 默认)

```bash
droidrun run "open settings" --no-reasoning
```

- 单个 Agent 直接执行
- ReAct 循环：思考 → 工具调用 → 观察 → 重复
- 适合简单任务
- 延迟更低

### Manager + Executor 模式 (reasoning=True)

```bash
droidrun run "book a hotel for next weekend" --reasoning
```

- Manager Agent 负责规划和分解子目标
- Executor Agent 负责执行每个子步骤
- 适合复杂多步任务
- 更好的错误恢复能力

---

## 8. 配置文件

默认位置：`~/.droidrun/config.yaml`

```yaml
agent:
  reasoning: false
  max_steps: 20
  streaming: true
  fast_agent:
    vision: true
  manager:
    vision: true
  executor:
    vision: false

device:
  serial: null          # 自动选择
  platform: android     # android 或 ios
  use_tcp: false
  auto_setup: true      # 自动安装 Portal

tools:
  stealth: false        # Stealth 模式
  disabled_tools: []    # 禁用特定工具

logging:
  debug: false
  save_trajectory: none  # none / step / action

tracing:
  enabled: false

llm_profiles:
  default:
    provider: OpenAI
    model: gpt-4o
    api_key_env: OPENAI_API_KEY
```

---

## 9. Portal App 详解

### 功能

Portal App 是安装在 Android 设备上的辅助应用，提供：

1. **Accessibility Service**：获取完整的 UI 无障碍树（比 `uiautomator dump` 更快更准确）
2. **HTTP Server**：通过 ADB port forward 建立 TCP 快速通道
3. **Content Provider**：通过 ADB shell 的降级通信通道
4. **Droidrun Keyboard**：Base64 编码的键盘输入，支持中文
5. **Overlay 控制**：截图时自动隐藏 Portal 的 overlay

### 手动设置

```bash
# 指定设备
droidrun setup -d <device_serial>

# 指定 APK 路径
droidrun setup --path /path/to/portal.apk

# 安装指定版本
droidrun setup --portal-version 0.4.7

# 安装最新版
droidrun setup --latest
```

### 验证 Portal

```bash
# 基础检查
droidrun ping

# TCP 模式检查
droidrun ping --tcp

# 全面诊断
droidrun doctor
```

---

## 10. iOS 支持

### 前置条件

1. 在 iOS 设备上安装 Droidrun iOS Portal App
2. 使用 `iproxy` 转发端口

### 使用

```bash
# 自动发现
droidrun run "open settings" --ios

# 指定 Portal URL
droidrun run "open settings" --ios -d http://127.0.0.1:6643
```

### 限制

- 无法检测前台应用名
- 应用列表为硬编码系统应用
- 只支持 Home 键（不支持 Back）
- 不支持 drag 操作

---

## 11. 常见问题

### Q: Portal 安装失败

```bash
# 检查 ADB 连接
adb devices

# 手动安装
droidrun setup --path /path/to/portal.apk --debug

# 运行诊断
droidrun doctor
```

### Q: Accessibility Service 未启用

```bash
# Portal 会尝试自动启用，如失败需手动：
# 设备上：设置 → 无障碍 → Droidrun Portal → 开启
```

### Q: Portal TCP 连接失败

```bash
# 检查端口转发
adb forward --list

# 使用 Content Provider 降级模式（默认自动降级）
droidrun run "指令" --no-tcp
```

### Q: 中文输入不生效

确保 Droidrun Keyboard IME 已启用。`droidrun setup` 会自动处理，但运行结束后会自动禁用。如果手动使用，需要：
```bash
adb shell ime enable com.droidrun.portal/.input.DroidrunKeyboardIME
adb shell ime set com.droidrun.portal/.input.DroidrunKeyboardIME
```

### Q: LLM 调用失败

```bash
# 检查 API Key
droidrun configure

# 使用 debug 模式查看详细错误
droidrun run "指令" --debug
```

---

## 12. 方案选型（全项目对比）

| 你的情况 | 推荐方案 | 原因 |
|----------|----------|------|
| 需要独立运行的 Agent | **Droidrun** | 内置完整 Agent Loop，不依赖 MCP 客户端 |
| 给 Claude/Cursor 提供设备工具 | **android-mcp-server / mobile-mcp** | 标准 MCP 服务器，IDE 直接集成 |
| 全平台 MCP + 高级特性 | **claude-in-mobile** | 5 平台 + Flow 引擎 + 130+ 别名容错 |
| 需要反检测 / Stealth | **Droidrun** | Bezier 曲线滑动 + 随机延迟 |
| 需要操作回放 / 自动化脚本 | **Droidrun** | Macro 录制+回放 |
| 极简部署（不装额外 App） | **android-mcp-server** | 一行 npx 启动，零设备修改 |
| Android + Selector 查找 | **Android-MCP (CursorTouch)** | u2 原生 Selector + WaitForElement |
| 已有 Appium 设施 | **appium-mcp** | 复用 Appium 生态 |
| 需要自定义视觉模型 | **Open-AutoGLM** | 内置 AutoGLM-Phone-9B |
| 学术研究 + 自动化学习 | **AppAgent** | UI 文档自动生成 + 探索-利用两阶段 |
| 企业级可观测需求 | **Droidrun** | Phoenix + Langfuse + PostHog |
| 不同 Agent 用不同 LLM | **Droidrun** | Manager/Executor/FastAgent 独立 LLM 配置 |
| LLM 调用工具名容错 | **claude-in-mobile** | 130+ 别名映射，容忍 LLM 命名偏差 |
| 应用商店上传集成 | **claude-in-mobile** | Google Play / Huawei / RuStore |
| 跨平台最广 | **claude-in-mobile** | Android + iOS + Desktop + Aurora + Browser |

### Droidrun vs claude-in-mobile 核心差异

| 维度 | Droidrun | claude-in-mobile |
|------|----------|-----------------|
| 定位 | 自主 Agent（内置 LLM 推理） | MCP 工具服务器（由 Client 驱动） |
| 平台 | Android + iOS | 5 平台 |
| 独立运行 | 可以（CLI/TUI/SDK） | 不行（需要 MCP Client） |
| Stealth 模式 | Bezier 曲线 + 随机延迟 | 无 |
| 别名容错 | 无 | 130+ 别名 |
| UI 分析 | Portal Accessibility（质量更高） | 正则解析 + 模糊匹配 + diff |
| 可观测性 | Phoenix + Langfuse + PostHog | 无 |
| 应用商店 | 无 | Play/Huawei/RuStore |

### Droidrun vs AppAgent 核心差异

| 维度 | Droidrun | AppAgent |
|------|----------|----------|
| 定位 | 工程化 Agent 框架 | 学术研究原型 |
| Agent 模式 | Manager/Executor/FastAgent | 探索-利用两阶段 |
| 学习能力 | 无（每次从零） | UI 文档积累 |
| 设备通信 | ADB + Portal | 直接 ADB |
| 多 LLM | 6+ 供应商 | GPT-4V / Qwen-VL |
| Stealth | Bezier 曲线 | 无 |
| 可观测性 | 完整 | 无 |
| 工程化 | PyPI 包 + CLI + TUI | 学术脚本 |

---

## 13. 参考资料

- 官方文档: https://docs.droidrun.ai
- GitHub: https://github.com/droidrun/droidrun
- 云服务: https://cloud.droidrun.ai
- Benchmark: https://droidrun.ai/benchmark
- Discord: https://discord.gg/ZZbKEZZkwK
