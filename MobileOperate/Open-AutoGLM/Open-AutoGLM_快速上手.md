# Open-AutoGLM 快速上手

> 仓库地址：https://github.com/zai-org/Open-AutoGLM
> 本地路径：`~/Documents/coding/github/Open-AutoGLM`

---

## 一、环境要求

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Python | 3.10+ | 运行 Agent |
| pip | 随 Python 安装 | 包管理 |
| ADB | 最新版 | Android 设备控制 |
| HDC | 最新版 | HarmonyOS 设备控制（可选） |
| libimobiledevice | 最新版 | iOS 设备发现（可选） |
| ADB Keyboard APK | - | Android 中文输入（必装） |
| WebDriverAgent | - | iOS 设备 UI 操作（可选） |
| VLM 模型服务 | - | AutoGLM-Phone-9B 或云 API |

### 核心依赖（自动安装）

| 包名 | 版本 | 用途 |
|------|------|------|
| `Pillow` | >=12.0.0 | 图片处理（截图） |
| `openai` | >=2.9.0 | OpenAI 兼容 API 客户端 |
| `requests` | >=2.31.0 | iOS WDA HTTP 通信 |

### 环境变量

| 变量 | 描述 | 默认值 |
|------|------|--------|
| `PHONE_AGENT_BASE_URL` | 模型 API 地址 | `http://localhost:8000/v1` |
| `PHONE_AGENT_MODEL` | 模型名称 | `autoglm-phone-9b` |
| `PHONE_AGENT_API_KEY` | API Key | `EMPTY` |
| `PHONE_AGENT_MAX_STEPS` | 每个任务最大步数 | `100` |
| `PHONE_AGENT_DEVICE_ID` | 设备 ID | 自动检测 |
| `PHONE_AGENT_DEVICE_TYPE` | 设备类型（adb/hdc/ios） | `adb` |
| `PHONE_AGENT_LANG` | 语言（cn/en） | `cn` |

---

## 二、安装与启动

### 步骤 1：安装 ADB 工具

```bash
# macOS
brew install android-platform-tools

# Linux
sudo apt install android-tools-adb

# 验证
adb version
```

### 步骤 2：安装 ADB Keyboard（Android 必须）

1. 下载 [ADB Keyboard APK](https://github.com/senzhk/ADBKeyBoard/blob/master/ADBKeyboard.apk)
2. 安装到手机：`adb install ADBKeyboard.apk`
3. 在手机 设置 > 语言和输入法 > 启用 ADB Keyboard
4. 或命令启用：`adb shell ime enable com.android.adbkeyboard/.AdbIME`

### 步骤 3：安装 Phone Agent

```bash
git clone https://github.com/zai-org/Open-AutoGLM.git
cd Open-AutoGLM
pip install -r requirements.txt
pip install -e .
```

### 步骤 4：配置模型服务（二选一）

**方式 A：使用云 API（推荐新手）**

```bash
# 智谱 BigModel
python main.py \
  --base-url https://open.bigmodel.cn/api/paas/v4 \
  --model "autoglm-phone" \
  --apikey "你的API Key" \
  "打开微信"

# ModelScope
python main.py \
  --base-url https://api-inference.modelscope.cn/v1 \
  --model "ZhipuAI/AutoGLM-Phone-9B" \
  --apikey "你的API Key" \
  "打开微信"
```

**方式 B：本地部署模型（需要 GPU 24GB+）**

```bash
# 安装 vLLM
pip install vllm

# 启动模型服务
python3 -m vllm.entrypoints.openai.api_server \
  --served-model-name autoglm-phone-9b \
  --allowed-local-media-path / \
  --mm-encoder-tp-mode data \
  --mm_processor_cache_type shm \
  --mm_processor_kwargs '{"max_pixels":5000000}' \
  --max-model-len 25480 \
  --chat-template-content-format string \
  --limit-mm-per-prompt '{"image":10}' \
  --model zai-org/AutoGLM-Phone-9B \
  --port 8000
```

### 步骤 5：连接设备

```bash
# 确认 USB 调试已开启
adb devices
# 应看到：emulator-5554    device
```

### 步骤 6：运行

```bash
# 交互模式
python main.py --base-url http://localhost:8000/v1

# 单次任务
python main.py --base-url http://localhost:8000/v1 "打开美团搜索附近的火锅店"
```

---

## 三、不同平台的使用

### Android 设备

```bash
# 默认即为 Android
python main.py --base-url {MODEL_URL} "你的任务"
```

### HarmonyOS 设备

```bash
# 指定 hdc 设备类型
python main.py --device-type hdc --base-url {MODEL_URL} "打开设置"

# 列出鸿蒙设备
python main.py --device-type hdc --list-devices
```

### iOS 设备

前置准备：
1. 安装 libimobiledevice：`brew install libimobiledevice`
2. 在 Xcode 中编译运行 WebDriverAgent
3. 启动端口转发：`iproxy 8100 8100`

```bash
# 使用 iOS 设备
python main.py --device-type ios --base-url {MODEL_URL} "打开 Safari 搜索天气"

# 指定 WDA 地址
python main.py --device-type ios --wda-url http://192.168.1.100:8100 --base-url {MODEL_URL}

# 查看 WDA 状态
python main.py --device-type ios --wda-status
```

---

## 四、验证安装

### 方式 1：运行系统检查

```bash
# 检查模型部署
python scripts/check_deployment_cn.py --base-url http://localhost:8000/v1 --model autoglm-phone-9b
```

预期输出：模型返回含 `<think>` 推理过程和 `do(action=...)` 动作指令。

### 方式 2：执行简单任务

```bash
python main.py --base-url {MODEL_URL} "打开微信，对文件传输助手发送消息：测试成功"
```

预期结果：手机自动打开微信 -> 找到文件传输助手 -> 发送消息。

---

## 五、CLI 命令速查

| 命令 | 说明 |
|------|------|
| `python main.py "任务"` | 执行单次任务 |
| `python main.py` | 交互模式 |
| `python main.py --list-devices` | 列出已连接设备 |
| `python main.py --list-apps` | 列出支持的应用 |
| `python main.py --connect {ip}:{port}` | 远程连接设备 |
| `python main.py --disconnect` | 断开所有远程设备 |
| `python main.py --enable-tcpip` | 启用 TCP/IP 调试 |
| `python main.py --lang en` | 使用英文 prompt |
| `python main.py --device-type hdc` | 使用鸿蒙设备 |
| `python main.py --device-type ios` | 使用 iOS 设备 |
| `python main.py --device-type ios --wda-status` | 检查 WDA 状态 |
| `python main.py --device-type ios --pair` | 配对 iOS 设备 |

### Python API

```python
from phone_agent import PhoneAgent
from phone_agent.model import ModelConfig

model_config = ModelConfig(
    base_url="http://localhost:8000/v1",
    model_name="autoglm-phone-9b",
)

agent = PhoneAgent(model_config=model_config)
result = agent.run("打开淘宝搜索无线耳机")
print(result)
```

---

## 六、可用动作列表

| 动作 | 说明 |
|------|------|
| `Launch` | 按名称启动 App |
| `Tap` | 点击屏幕坐标 |
| `Type` | 输入文本（自动清除旧内容） |
| `Swipe` | 滑动手势 |
| `Back` | 返回上一页 |
| `Home` | 返回桌面 |
| `Double Tap` | 双击 |
| `Long Press` | 长按 |
| `Wait` | 等待指定时间 |
| `Take_over` | 请求人工接管（登录/验证码） |
| `Note` | 记录页面内容 |
| `Call_API` | 调用 API 总结内容 |
| `Interact` | 请求用户选择 |
| `finish` | 任务完成 |

---

## 七、常见问题

### Q: `adb devices` 无输出？

1. 检查数据线是否支持数据传输（不是仅充电线）
2. 手机上点击"允许 USB 调试"
3. 重启 ADB：`adb kill-server && adb start-server`

### Q: 能打开应用但无法点击？

在 开发者选项 中同时开启：
- USB 调试
- USB 调试（安全设置）

### Q: 中文输入失败？

确认 ADB Keyboard 已安装并启用：
```bash
adb shell ime list -s
# 应包含：com.android.adbkeyboard/.AdbIME
```

### Q: 截图返回黑屏？

通常是敏感页面（支付/银行），系统会自动检测并设置 `is_sensitive=True`。Agent 会尝试人工接管。

### Q: 连接模型服务失败？

```bash
# 检查模型服务是否运行
curl http://localhost:8000/v1/models

# 如果使用云 API，检查 API Key 是否正确
python main.py --base-url https://open.bigmodel.cn/api/paas/v4 --apikey "你的key" --list-devices
```

### Q: iOS WDA 无法连接？

1. 确认 WebDriverAgent 在 Xcode 中正在运行
2. USB 设备需端口转发：`iproxy 8100 8100`
3. WiFi 设备直接使用 IP：`--wda-url http://设备IP:8100`
4. 浏览器验证：打开 `http://localhost:8100/status`

---

## 八、与 mobile-mcp / appium-mcp 的选型建议

| 需求场景 | 推荐 | 原因 |
|---------|------|------|
| 想要"说一句话自动完成任务" | **Open-AutoGLM** | 自主 Agent，不需要外部 AI 编排 |
| 已有 Claude/GPT 作为 AI Agent | **mobile-mcp** | MCP 协议，即插即用 |
| 需要精确元素定位 | **appium-mcp** | 10 种定位策略 + WebDriverWait |
| 需要支持鸿蒙设备 | **Open-AutoGLM** | 唯一支持 HarmonyOS 的项目 |
| 需要 App 安装/卸载/录屏 | **mobile-mcp** | 功能最完整（20 个 Tool） |
| CI/CD 自动化测试 | **appium-mcp** | Appium 生态 + 并发支持 |
| 不想安装额外服务 | **mobile-mcp** | 无需模型服务或 Appium Server |

---

## 九、建议先读的源码

1. **`phone_agent/agent.py`** -- Agent 主循环，理解"截图-推理-执行"闭环
2. **`phone_agent/actions/handler.py`** -- 动作解析 + 执行分发，理解模型输出如何变成设备操作
3. **`phone_agent/model/client.py`** -- VLM 客户端，理解流式推理和 thinking/action 解析
4. **`phone_agent/adb/device.py`** -- Android 底层操作，理解所有 ADB 原子命令
5. **`phone_agent/device_factory.py`** -- 工厂模式，理解多平台切换机制
6. **`phone_agent/config/prompts_zh.py`** -- 系统提示词，理解模型输出格式约束
