# AppAgent 源码解读

> 仓库地址：https://github.com/TencentQQGYLab/AppAgent
> 论文：AppAgent: Multimodal Agents as Smartphone Users (CHI 2025)
> 来源：腾讯 QQ-GY Lab
> 语言：Python | 总文件数：~16 个 | 代码量：极小（学术原型）

---

## 一、项目定位

AppAgent 是一个基于**多模态大语言模型（GPT-4V / Qwen-VL）**的智能手机操作 Agent。核心创新在于**两阶段架构**：

1. **探索阶段（Exploration Phase）**：通过自主探索或人类演示，为 App 的 UI 元素生成功能文档
2. **部署阶段（Deployment Phase）**：利用探索阶段积累的文档知识，自主完成用户下达的任务

这种"先学后用"的设计使 Agent 能够积累 App 操作经验，类似于人类先熟悉 App 再使用的过程。

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────┐
│                     入口层                               │
│  learn.py (探索入口)          run.py (部署入口)           │
│  ├── 模式1: 自主探索           └── task_executor.py       │
│  │   └── self_explorer.py                                │
│  └── 模式2: 人类演示                                      │
│      ├── step_recorder.py                                │
│      └── document_generation.py                          │
├─────────────────────────────────────────────────────────┤
│                     核心层                               │
│  model.py          LLM 抽象层 (OpenAI / Qwen)           │
│  prompts.py        Prompt 模板库                         │
│  and_controller.py ADB 控制器 + UI 树解析                │
├─────────────────────────────────────────────────────────┤
│                     工具层                               │
│  config.py         配置加载 (YAML + 环境变量)             │
│  utils.py          截图标注 / 网格绘制 / Base64编码       │
└─────────────────────────────────────────────────────────┘
```

### 数据流

```
探索阶段:
  截图 + XML → UI元素提取 → 标注截图 → LLM决策 → 执行动作 → 反思 → 生成文档
                                                                    ↓
                                                          apps/<app>/auto_docs/
                                                          apps/<app>/demo_docs/

部署阶段:
  截图 + XML → UI元素提取 → 加载文档 → 注入Prompt → LLM决策 → 执行动作 → 循环
```

---

## 三、核心模块详解

### 3.1 ADB 控制器 (`scripts/and_controller.py`)

#### AndroidController 类

通过 `subprocess.run()` 执行 ADB 命令，提供以下原子操作：

| 方法 | ADB 命令 | 说明 |
|------|----------|------|
| `get_screenshot()` | `adb shell screencap -p` + `adb pull` | 截图并拉取到本地 |
| `get_xml()` | `adb shell uiautomator dump` + `adb pull` | 导出 UI 树 XML |
| `tap(x, y)` | `adb shell input tap {x} {y}` | 点击坐标 |
| `text(input_str)` | `adb shell input text {str}` | 输入文本 |
| `long_press(x, y)` | `adb shell input swipe {x} {y} {x} {y} 1000` | 长按（用 swipe 模拟） |
| `swipe(x, y, dir, dist)` | `adb shell input swipe ...` | 滑动（支持方向+距离） |
| `back()` | `adb shell input keyevent KEYCODE_BACK` | 返回键 |
| `get_device_size()` | `adb shell wm size` | 获取屏幕尺寸 |

#### UI 元素解析

`traverse_tree()` 函数解析 `uiautomator dump` 生成的 XML：
- 遍历 XML 树，提取 `clickable=true` 或 `focusable=true` 的元素
- 用 `bounds` 属性计算 bounding box 和中心点
- 用 `resource-id` + `content-desc` 生成唯一的 `elem_id`
- 通过 `MIN_DIST` 配置去重距离过近的元素

```python
class AndroidElement:
    def __init__(self, uid, bbox, attrib):
        self.uid = uid        # 如 "com.example_button_Submit"
        self.bbox = bbox      # ((x1,y1), (x2,y2))
        self.attrib = attrib  # "clickable" 或 "focusable"
```

### 3.2 LLM 抽象层 (`scripts/model.py`)

#### 模型接口

```python
class BaseModel:
    def get_model_response(self, prompt: str, images: List[str]) -> (bool, str)

class OpenAIModel(BaseModel):  # 原生 HTTP 调用 GPT-4V
class QwenModel(BaseModel):    # 通过 dashscope SDK 调用通义千问 VL
```

- **OpenAI**：手动构建 multipart 请求，Base64 编码图片，直接 POST 到 API
- **Qwen**：使用 `dashscope.MultiModalConversation.call()`，图片以 `file://` 路径传入
- 无对话历史管理，每次请求都是独立的单轮调用

#### 响应解析器

三个解析函数用正则提取结构化输出：

| 解析器 | 用途 | 提取字段 |
|--------|------|----------|
| `parse_explore_rsp()` | 探索/部署阶段的动作解析 | Observation, Thought, Action, Summary |
| `parse_grid_rsp()` | Grid 模式下的动作解析 | 同上，但 Action 含 subarea 参数 |
| `parse_reflect_rsp()` | 反思阶段的决策解析 | Decision, Thought, Documentation |

### 3.3 自主探索 (`scripts/self_explorer.py`)

核心循环（最多 `MAX_ROUNDS` 轮）：

```
每轮:
1. 截图 + dump XML
2. 提取可交互元素 (clickable + focusable)
3. 在截图上标注数字标签 → labeled screenshot
4. 构建 prompt (任务描述 + 历史动作 + 标注截图)
5. 调用 LLM 获取动作决策
6. 执行动作 (tap/text/long_press/swipe)
7. 再次截图 → 与动作前对比
8. 反思 (Reflect)：
   - SUCCESS → 生成 UI 元素文档，继续
   - BACK → 生成文档，返回上一页
   - CONTINUE → 生成文档，继续尝试其他元素
   - INEFFECTIVE → 将元素加入 useless_list，跳过
```

关键设计：
- **useless_list**：记录无效元素的 resource_id，避免重复交互
- **反思机制**：通过前后截图对比判断动作效果
- **文档结构**：每个 UI 元素的文档是一个 dict，按操作类型存储

```python
doc_content = {
    "tap": "Tapping this element opens the settings menu",
    "text": "",
    "v_swipe": "",
    "h_swipe": "",
    "long_press": ""
}
```

### 3.4 人类演示 (`scripts/step_recorder.py`)

交互式录制流程：
1. 截图并标注 → 用 OpenCV 弹窗展示
2. 用户选择操作类型（tap/text/long_press/swipe/stop）
3. 用户选择目标元素编号
4. 执行并记录到 `record.txt`

记录格式：
```
tap(3):::com.example_button_send
text(5:sep:"Hello"):::com.example_edittext_input
swipe(2:sep:up):::com.example_scrollview
```

### 3.5 文档生成 (`scripts/document_generation.py`)

基于人类演示的记录生成文档：
- 读取 `record.txt` 的每一步操作
- 对每步的前后截图，构建文档生成 prompt
- LLM 基于截图差异描述 UI 元素功能
- 支持 `DOC_REFINE`：在已有文档基础上优化

### 3.6 任务执行 (`scripts/task_executor.py`)

部署阶段的核心，与自主探索类似但有关键区别：

1. **文档注入**：加载 `auto_docs/` 或 `demo_docs/` 中的文档，注入到 prompt 中
2. **Grid 模式**：当标注元素无法满足需求时，切换为网格覆盖截图
3. **无反思环节**：部署阶段不再反思和生成文档，只执行动作

Grid 模式的坐标计算：
```python
def area_to_xy(area, subarea):
    # 将网格编号+子区域 → 精确坐标
    # subarea: center/top-left/top/top-right/left/right/bottom-left/bottom/bottom-right
```

### 3.7 Prompt 模板 (`scripts/prompts.py`)

| 模板 | 用途 |
|------|------|
| `tap_doc_template` | 点击操作的文档生成 |
| `text_doc_template` | 文本输入的文档生成 |
| `long_press_doc_template` | 长按操作的文档生成 |
| `swipe_doc_template` | 滑动操作的文档生成 |
| `refine_doc_suffix` | 已有文档的优化后缀 |
| `task_template` | 部署阶段的任务执行（标注模式） |
| `task_template_grid` | 部署阶段的任务执行（网格模式） |
| `self_explore_task_template` | 探索阶段的动作决策 |
| `self_explore_reflect_template` | 探索阶段的反思决策 |

Prompt 设计特点：
- 要求输出固定格式：`Observation → Thought → Action → Summary`
- 反思阶段的四种决策：`SUCCESS / BACK / CONTINUE / INEFFECTIVE`
- 文档生成强调"通用性描述"，避免包含具体数据

---

## 四、底层原子操作汇总

| 操作 | 实现方式 | 参数 |
|------|----------|------|
| 截图 | `adb shell screencap` | 保存路径 |
| UI 树导出 | `adb shell uiautomator dump` | 保存路径 |
| 点击 | `adb shell input tap` | x, y 坐标 |
| 输入文本 | `adb shell input text` | 字符串（空格→%s） |
| 长按 | `adb shell input swipe x y x y 1000` | x, y, duration |
| 滑动 | `adb shell input swipe` | 起止坐标, duration |
| 返回 | `adb shell input keyevent KEYCODE_BACK` | 无 |
| 设备尺寸 | `adb shell wm size` | 无 |

特点：
- 纯 ADB 命令行操作，无 Accessibility Service
- 无 Home 键、多任务键、音量键等操作
- 无剪贴板操作
- 无应用启动/安装命令

---

## 五、关键设计模式

### 5.1 探索-利用范式（Explore-Exploit）

这是 AppAgent 的核心创新：
- **探索阶段**：Agent 像新用户一样熟悉 App，将操作经验固化为文档
- **利用阶段**：Agent 参考文档执行任务，类似"有经验的用户"

与传统 Agent 的"每次从零开始"相比，这种设计减少了部署阶段的试错成本。

### 5.2 反思学习（Reflective Learning）

自主探索中的"Act-Reflect"循环：
- 执行动作后，通过前后截图对比评估效果
- 四种反思结果映射到不同的后续策略
- 文档在反思过程中生成，确保只记录有效信息

### 5.3 UI 元素标注系统

- 在截图上用数字标签标注可交互元素
- LLM 通过数字编号引用元素，无需理解坐标
- 分两层标注：clickable（红色）和 focusable（蓝色）
- Grid 模式作为 fallback，处理标注系统无法覆盖的场景

### 5.4 文档驱动决策

部署阶段的 prompt 注入文档：
```
Documentation of UI element labeled with the numeric tag '3':
This UI element is clickable. Tapping this element opens the settings menu.
```

LLM 根据文档优先选择已知功能的元素，降低盲目探索的概率。

---

## 六、项目对比

### 6.1 与 mobile-mcp 对比

| 维度 | AppAgent | mobile-mcp |
|------|----------|------------|
| 定位 | 学术研究原型 | 工程化 MCP 工具集 |
| 架构 | 自治 Agent（两阶段） | LLM 工具层（MCP Server） |
| 平台 | Android only | Android + iOS |
| 设备通信 | 纯 ADB 命令 | ADB + Maestro + idb |
| UI 理解 | 截图 + XML + LLM视觉 | 截图 + XML + Accessibility |
| 智能层 | 内置（GPT-4V/Qwen-VL） | 外部（由 MCP Client 决定） |
| 文档机制 | 自动生成 UI 文档 | 无 |
| 操作种类 | 7 种基础操作 | 20+ 种丰富操作 |
| Grid 模式 | 支持 | 不支持 |

### 6.2 与 android-mcp-server 对比

| 维度 | AppAgent | android-mcp-server |
|------|----------|--------------------|
| 代码量 | ~1200 行 Python | ~200 行 TypeScript |
| 复杂度 | 高（Agent 完整闭环） | 低（纯 MCP 工具暴露） |
| 截图处理 | CV2 标注 + Base64 | Sharp 压缩 + MCP 返回 |
| 执行模式 | 多轮自主决策循环 | 单次工具调用 |
| Shell 优化 | 无（每次 subprocess） | 持久 Shell 会话 |
| 依赖 | opencv, pyshine, dashscope | sharp, @anthropic-ai/sdk |

### 6.3 与 appium-mcp 对比

| 维度 | AppAgent | appium-mcp |
|------|----------|------------|
| UI 定位 | 截图标注 + 数字编号 | Appium Selector + WebDriver |
| 测试框架 | 无 | Appium (WebDriver 协议) |
| 元素查找 | XML 遍历 + 距离去重 | XPath / ID / Accessibility ID |
| 跨平台 | Android only | Android + iOS + Web |
| 等待机制 | `time.sleep()` 固定等待 | WebDriver 隐式/显式等待 |

### 6.4 与 Open-AutoGLM 对比

| 维度 | AppAgent | Open-AutoGLM |
|------|----------|--------------|
| 来源 | 腾讯 QQ-GY Lab | 清华 × 智谱 AI |
| 模型 | GPT-4V / Qwen-VL（外部API） | ChatGLM（自有模型） |
| 核心创新 | 探索-利用范式 | 阶段级奖励训练 |
| 学习方式 | 文档生成 | 强化学习 |
| 操作方式 | ADB 命令 | Accessibility + 坐标 |
| 运行需求 | API Key + ADB | GPU 集群 + 自定义ROM |

### 6.5 与 droidrun 对比

| 维度 | AppAgent | droidrun |
|------|----------|----------|
| Agent 模式 | 单一循环（explore/execute） | Manager/Executor/FastAgent 三模式 |
| 探索机制 | 自主探索 + 人类演示 | 无专门探索阶段 |
| 文档系统 | 自动生成 UI 元素文档 | 无 |
| 多 LLM | OpenAI + Qwen (2种) | OpenAI + Gemini + Anthropic (6+种) |
| Stealth 模式 | 无 | Bezier 曲线拟人操作 |
| Portal App | 无 | 设备端 Portal App |
| 操作丰富度 | 7 种 | 20+ 种 |
| 包管理 | 无 | pip 包 (droidrun) |
| 成熟度 | 学术原型 | 工程化产品 |

---

## 七、代码质量评估

### 优点
- 代码简洁清晰，学术原型质量好
- 两阶段设计理念先进
- Prompt 设计精细，结构化输出易于解析
- 反思机制设计合理

### 局限
- **无错误恢复**：ADB 失败直接 break，无重试机制
- **无并发**：串行执行所有操作
- **固定等待**：`time.sleep(REQUEST_INTERVAL)` 而非智能等待
- **安全性**：`os.system()` 调用子脚本，config 中硬编码 API Key
- **平台限制**：仅支持 Android，仅支持 2 种 LLM
- **无状态持久化**：探索的文档以 txt 文件存储，格式为 Python dict 字符串
- **无应用管理**：不能启动/切换/安装应用
- **UI 解析**：`ast.literal_eval()` 解析文档，不够健壮

---

## 八、总结

AppAgent 的核心贡献在于提出了**探索-利用（Explore-Exploit）范式**和**反思驱动的文档生成**机制。作为学术原型，它验证了"LLM Agent 可以通过学习积累操作经验"的假设。但在工程化层面，它的操作种类有限、缺乏错误恢复、不支持多平台，适合作为研究参考而非生产使用。

与其他项目的核心差异：
- 与 MCP 工具（mobile-mcp / android-mcp-server / appium-mcp）：AppAgent 是自治 Agent，它们是被动工具
- 与 Open-AutoGLM：AppAgent 用文档学习，AutoGLM 用强化学习
- 与 droidrun：AppAgent 是轻量学术原型，droidrun 是工程化 Agent 框架
