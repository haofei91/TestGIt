# AppAgent 快速上手指南

> 仓库：https://github.com/TencentQQGYLab/AppAgent
> 定位：基于多模态 LLM 的智能手机操作 Agent（学术原型）
> 核心特性：两阶段架构（探索 + 部署），自动生成 UI 元素文档

---

## 一、环境要求

| 依赖 | 要求 |
|------|------|
| Python | 3.x |
| ADB | 已安装并加入 PATH |
| Android 设备 | 真机或模拟器，已开启 USB 调试 |
| API Key | OpenAI (GPT-4V) 或 阿里云 DashScope (Qwen-VL) |
| 操作系统 | macOS / Linux / Windows |

---

## 二、安装步骤

### 2.1 克隆仓库

```bash
git clone https://github.com/TencentQQGYLab/AppAgent.git
cd AppAgent
```

### 2.2 安装依赖

```bash
pip install -r requirements.txt
```

依赖列表：
- `argparse` — 命令行参数解析
- `colorama` — 终端彩色输出
- `dashscope` — 阿里云 Qwen-VL SDK
- `opencv-python` — 图像处理（截图标注）
- `pyshine` — 图像上绘制带背景的文字标签
- `pyyaml` — YAML 配置解析
- `requests` — HTTP 请求（调用 OpenAI API）

### 2.3 配置 ADB

```bash
# 检查 ADB 是否可用
adb version

# 连接设备后检查
adb devices
```

确保至少有一个设备显示为 `device` 状态。

---

## 三、配置文件

编辑 `config.yaml`：

```yaml
# 模型选择: "OpenAI" 或 "Qwen"
MODEL: "OpenAI"

# OpenAI 配置
OPENAI_API_BASE: "https://api.openai.com/v1/chat/completions"
OPENAI_API_KEY: "sk-your-key-here"
OPENAI_API_MODEL: "gpt-4-vision-preview"
MAX_TOKENS: 300
TEMPERATURE: 0.0

# Qwen 配置 (使用 Qwen 时填写)
DASHSCOPE_API_KEY: "sk-your-key-here"
QWEN_MODEL: "qwen-vl-max"

# 请求间隔 (秒)，避免 API 限频
REQUEST_INTERVAL: 10

# 设备端截图/XML 临时存储路径
ANDROID_SCREENSHOT_DIR: "/sdcard"
ANDROID_XML_DIR: "/sdcard"

# 文档优化：已有文档时是否基于新演示优化
DOC_REFINE: false

# 探索最大轮数
MAX_ROUNDS: 20

# 暗色模式截图标注
DARK_MODE: false

# UI 元素去重距离阈值 (像素)
MIN_DIST: 30
```

**注意**：配置也支持环境变量覆盖（`config.py` 先读环境变量再合并 YAML）。

---

## 四、使用流程

AppAgent 分两个阶段使用：

### 阶段一：探索（学习阶段）

```bash
python learn.py --app <app_name>
```

启动后选择模式：
- 输入 `1`：**自主探索** — Agent 自动操作 App 并生成文档
- 输入 `2`：**人类演示** — 你手动操作，Agent 学习生成文档

#### 自主探索流程

1. 输入任务描述（如 "send a message to John"）
2. Agent 自动截图、分析、执行操作
3. 每步操作后反思效果，生成 UI 元素文档
4. 文档保存到 `apps/<app_name>/auto_docs/`

#### 人类演示流程

1. 输入任务描述
2. 看到标注截图后，选择操作类型：`tap`、`text`、`long press`、`swipe`、`stop`
3. 选择目标元素编号
4. 操作记录保存后，自动生成文档到 `apps/<app_name>/demo_docs/`

### 阶段二：部署（执行阶段）

```bash
python run.py --app <app_name>
```

1. 自动加载探索阶段生成的文档
2. 输入任务描述
3. Agent 参考文档自主完成任务

---

## 五、目录结构说明

运行后自动生成的目录：

```
AppAgent/
├── apps/
│   └── <app_name>/
│       ├── auto_docs/          # 自主探索生成的文档
│       │   └── <elem_id>.txt   # 每个 UI 元素一个文件
│       ├── demo_docs/          # 人类演示生成的文档
│       │   └── <elem_id>.txt
│       └── demos/
│           └── <task_name>/    # 每次探索/演示的数据
│               ├── raw_screenshots/
│               ├── labeled_screenshots/
│               ├── xml/
│               ├── record.txt
│               └── task_desc.txt
├── tasks/
│   └── <task_name>/            # 每次任务执行的数据
│       └── log_*.txt
```

---

## 六、使用示例

### 示例：探索微信

```bash
# 1. 确保手机打开微信主界面
# 2. 启动探索
python learn.py --app WeChat

# 选择模式 1 (自主探索)
# 输入任务: "navigate to the Moments page and like a post"
# Agent 开始自动操作...
```

### 示例：执行任务

```bash
# 确保已有探索阶段的文档
python run.py --app WeChat

# 输入任务: "send a message saying hello to the first chat"
# Agent 参考文档自主执行...
```

---

## 七、常见问题

### Q1: `ERROR: No device found!`
确保设备已连接且 `adb devices` 显示正常。模拟器用户检查端口连接：
```bash
adb connect 127.0.0.1:5555
```

### Q2: `uiautomator dump` 失败
某些 App 页面（如启动页、动画页）不支持 dump。等待页面稳定后重试。

### Q3: OpenAI API 报错
- 检查 API Key 是否有效
- 确认账户有 GPT-4V 访问权限
- `REQUEST_INTERVAL` 设大一些避免限频

### Q4: 截图标注看不清
- 暗色背景的 App 设置 `DARK_MODE: true`
- 检查 `pyshine` 是否正确安装

### Q5: 无文档也能执行任务吗？
可以。执行时如果没找到文档，会询问是否无文档继续。此时 Agent 没有历史知识，效果会差一些。

### Q6: Grid 模式什么时候触发？
当 Agent 判断标注的 UI 元素都无法完成当前步骤时，会输出 `grid()` 动作，系统切换为网格模式。网格将截图划分为小方格，Agent 可以指定任意方格+子区域进行操作。

---

## 八、与其他项目的选型建议

| 场景 | 推荐项目 | 原因 |
|------|----------|------|
| 学术研究/复现论文 | **AppAgent** | 论文原型，概念验证 |
| 需要 MCP 集成 | mobile-mcp / android-mcp-server | MCP 协议标准接入 |
| Android 自动化测试 | appium-mcp | Appium 生态成熟 |
| 生产级自治 Agent | droidrun | 工程化完善，多 LLM 支持 |
| 强化学习研究 | Open-AutoGLM | 端到端训练方案 |
| 跨平台 (iOS + Android) | mobile-mcp / appium-mcp | AppAgent 仅支持 Android |
| 快速原型验证 | **AppAgent** | 代码简洁，易于修改 |

### 核心差异

- **AppAgent** 的独特价值在于"先学后用"的两阶段范式，适合研究场景
- 如果需要工程化部署，推荐 **droidrun**（自治 Agent）或 **mobile-mcp**（MCP 工具）
- 如果需要跨平台支持，AppAgent 不适合，选择 **mobile-mcp** 或 **appium-mcp**
- 如果关注模型训练和端到端优化，参考 **Open-AutoGLM**

---

## 九、扩展建议

如果基于 AppAgent 做二次开发：

1. **增加操作种类**：参考 droidrun/mobile-mcp 的操作列表，补充 Home 键、通知栏、剪贴板等
2. **替换 LLM 调用**：使用 LiteLLM 或类似统一接口支持更多模型
3. **优化等待机制**：替换 `time.sleep()` 为基于 UI 变化检测的智能等待
4. **增加错误恢复**：ADB 失败时重试，超时自动恢复
5. **结构化文档存储**：将 dict 字符串改为 JSON 格式
6. **添加应用管理**：支持自动启动/切换目标 App
