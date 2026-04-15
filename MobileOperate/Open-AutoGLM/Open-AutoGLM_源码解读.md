# Open-AutoGLM 源码解读

> 仓库地址：https://github.com/zai-org/Open-AutoGLM
> 本地路径：`~/Documents/coding/github/Open-AutoGLM`
> 分析聚焦：底层原子操作、Agent 循环机制、多平台适配、与 mobile-mcp / appium-mcp 对比
> 分析日期：2026-04-15

---

## 一、项目定位

Open-AutoGLM（内部名 Phone Agent）是智谱 AI 推出的**端侧手机智能助理框架**。它的核心思路是：

**截图 -> VLM（视觉语言模型）理解界面 -> 输出操作指令 -> 执行设备操作 -> 循环**

与 mobile-mcp / appium-mcp 的根本区别在于：**Open-AutoGLM 不是 MCP 工具服务器，而是一个自主 Agent 闭环系统**。它自带 VLM 推理客户端，不需要外部 AI Agent 调度。

### 适用场景

- 用自然语言驱动手机完成复杂多步任务（"打开小红书搜索美食"）
- 跨平台支持：Android（ADB）+ HarmonyOS（HDC）+ iOS（WDA）
- 自主循环决策，不依赖外部编排

### 优点与局限

| 优点 | 局限 |
|------|------|
| 端到端自主 Agent，无需外部 MCP 客户端 | 需要额外部署 VLM 模型服务（GPU 或云 API） |
| 三平台支持（Android + HarmonyOS + iOS） | 纯坐标驱动，无 UI 元素语义定位 |
| 内置敏感操作确认 + 人工接管机制 | 无 App 安装/卸载、录屏等管理功能 |
| 支持本地部署和云 API 两种模型接入方式 | 同步阻塞，无并发多设备支持 |
| 50+ 中文应用 + 60+ 鸿蒙应用映射 | 不支持 MCP 协议，无法被其他 AI Agent 调用 |

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户（自然语言任务）                      │
│                           ↓                                  │
├─────────────────────────────────────────────────────────────┤
│                   PhoneAgent / IOSPhoneAgent                 │
│       Agent 循环：截图 → 构建消息 → 模型推理 → 解析动作 → 执行    │
│                    ↕                    ↕                     │
│            ModelClient              ActionHandler            │
│        (OpenAI 兼容 API)         (动作解析 + 分发执行)          │
│                                       ↓                      │
├────────────┬──────────────┬───────────────────────────────────┤
│   adb/     │    hdc/      │         xctest/                   │
│  Android   │  HarmonyOS   │          iOS                      │
│            │              │                                   │
│ device.py  │  device.py   │  device.py                        │
│ input.py   │  input.py    │  input.py                         │
│ screenshot │  screenshot  │  screenshot.py                    │
│ connection │  connection  │  connection.py                    │
├────────────┴──────────────┴───────────────────────────────────┤
│              DeviceFactory（工厂模式，ADB/HDC 切换）             │
├─────────────────────────────────────────────────────────────┤
│                     底层命令/协议                               │
│   adb shell input ...   hdc shell uitest ...   WDA HTTP API  │
│        ↓                      ↓                    ↓          │
│   Android 设备          HarmonyOS 设备          iOS 设备        │
└─────────────────────────────────────────────────────────────┘
```

### 关键设计决策

1. **Agent 闭环**：不暴露 MCP Tool，而是自己内部完成"感知-思考-执行"循环
2. **工厂模式**：`DeviceFactory` 统一 ADB/HDC 接口，iOS 单独走 WDA
3. **坐标驱动**：模型输出相对坐标（0-1000），ActionHandler 转换为绝对像素
4. **流式推理**：ModelClient 使用 streaming API，实时打印思考过程

---

## 三、底层原子操作清单

### Android（ADB）— 12 条原子命令

| # | ADB 命令 | 文件 | 用途 |
|---|---------|------|------|
| 1 | `adb shell input tap {x} {y}` | adb/device.py | 点击 |
| 2 | `adb shell input swipe {x1} {y1} {x2} {y2} {dur}` | adb/device.py | 滑动 / 长按 |
| 3 | `adb shell input keyevent 4` | adb/device.py | 返回键 |
| 4 | `adb shell input keyevent KEYCODE_HOME` | adb/device.py | Home 键 |
| 5 | `adb shell monkey -p {pkg} -c android.intent.category.LAUNCHER 1` | adb/device.py | 启动 App |
| 6 | `adb shell dumpsys window` | adb/device.py | 获取当前前台 App |
| 7 | `adb shell screencap -p /sdcard/tmp.png` | adb/screenshot.py | 截图（设备端） |
| 8 | `adb pull /sdcard/tmp.png {local}` | adb/screenshot.py | 拉取截图 |
| 9 | `adb shell am broadcast -a ADB_INPUT_B64 --es msg {b64}` | adb/input.py | 通过 ADB Keyboard 输入文本 |
| 10 | `adb shell am broadcast -a ADB_CLEAR_TEXT` | adb/input.py | 清除输入框文本 |
| 11 | `adb shell settings get secure default_input_method` | adb/input.py | 获取当前输入法 |
| 12 | `adb shell ime set {ime}` | adb/input.py | 切换输入法 |
| 13 | `adb devices -l` | adb/connection.py | 列出设备 |
| 14 | `adb connect {addr}` | adb/connection.py | 远程连接 |
| 15 | `adb disconnect {addr}` | adb/connection.py | 断开连接 |
| 16 | `adb tcpip {port}` | adb/connection.py | 启用 TCP/IP |
| 17 | `adb shell ip route` / `ip addr show wlan0` | adb/connection.py | 获取设备 IP |
| 18 | `adb kill-server` / `adb start-server` | adb/connection.py | 重启 ADB 服务 |

### HarmonyOS（HDC）— 9 条原子命令

| # | HDC 命令 | 文件 | 用途 |
|---|---------|------|------|
| 1 | `hdc shell uitest uiInput click {x} {y}` | hdc/device.py | 点击 |
| 2 | `hdc shell uitest uiInput doubleClick {x} {y}` | hdc/device.py | 双击 |
| 3 | `hdc shell uitest uiInput longClick {x} {y}` | hdc/device.py | 长按 |
| 4 | `hdc shell uitest uiInput swipe {x1} {y1} {x2} {y2} {dur}` | hdc/device.py | 滑动 |
| 5 | `hdc shell uitest uiInput keyEvent Back` | hdc/device.py | 返回键 |
| 6 | `hdc shell uitest uiInput keyEvent Home` | hdc/device.py | Home 键 |
| 7 | `hdc shell aa start -b {bundle} -a {ability}` | hdc/device.py | 启动 App |
| 8 | `hdc shell aa dump -l` | hdc/device.py | 获取前台 App |
| 9 | `hdc shell snapshot_display` | hdc/screenshot.py | 截图 |

### iOS（WDA HTTP API）— 10 个端点

| # | WDA 端点 | 方法 | 文件 | 用途 |
|---|---------|------|------|------|
| 1 | `/session/{id}/actions` | POST | xctest/device.py | 点击 / 双击 / 长按（W3C Actions） |
| 2 | `/session/{id}/wda/dragfromtoforduration` | POST | xctest/device.py | 滑动 / 返回手势 |
| 3 | `/wda/homescreen` | POST | xctest/device.py | Home 键 |
| 4 | `/session/{id}/wda/apps/launch` | POST | xctest/device.py | 启动 App |
| 5 | `/wda/activeAppInfo` | GET | xctest/device.py | 获取前台 App |
| 6 | `/session/{id}/window/size` | GET | xctest/device.py | 获取屏幕尺寸 |
| 7 | `/wda/pressButton` | POST | xctest/device.py | 按物理按钮 |
| 8 | `/screenshot` | GET | xctest/screenshot.py | 截图 |
| 9 | `/session/{id}/wda/keys` | POST | xctest/input.py | 输入文本 / 发送按键 |
| 10 | `/session/{id}/element/active` + `/element/{id}/clear` | GET+POST | xctest/input.py | 清除文本 |
| 11 | `/wda/keyboard/dismiss` | POST | xctest/input.py | 隐藏键盘 |
| 12 | `/wda/setPasteboard` / `/wda/getPasteboard` | POST | xctest/input.py | 剪贴板操作 |
| 13 | `/status` | GET | xctest/connection.py | WDA 状态检查 |
| 14 | `/session` | POST | xctest/connection.py | 创建 WDA 会话 |

---

## 四、核心模块伪代码

### 4.1 Agent 主循环（agent.py）

```python
class PhoneAgent:
    def run(task: str) -> str:
        context = []
        step_count = 0
        
        # 第一步：附带用户任务描述
        result = execute_step(task, is_first=True)
        if result.finished: return result.message
        
        # 后续步骤：循环直到完成或达到 max_steps
        while step_count < max_steps:
            result = execute_step(is_first=False)
            if result.finished: return result.message
        
        return "Max steps reached"
    
    def execute_step(user_prompt, is_first):
        step_count += 1
        
        # 1. 截图 + 获取当前 App
        screenshot = device_factory.get_screenshot(device_id)
        current_app = device_factory.get_current_app(device_id)
        
        # 2. 构建消息（system prompt + 截图 + 屏幕信息）
        if is_first:
            context.append(system_message(system_prompt))
            context.append(user_message(
                text=f"{user_prompt}\n\n{screen_info(current_app)}",
                image=screenshot.base64_data
            ))
        else:
            context.append(user_message(
                text=f"Screen Info: {screen_info(current_app)}",
                image=screenshot.base64_data
            ))
        
        # 3. 调用 VLM 模型推理（流式）
        response = model_client.request(context)  # -> ModelResponse(thinking, action)
        
        # 4. 解析模型输出的动作指令
        action = parse_action(response.action)  # "do(action='Tap', element=[500,300])" -> dict
        
        # 5. 移除上下文中的图片（节省 token）
        context[-1] = remove_images(context[-1])
        
        # 6. 执行动作
        result = action_handler.execute(action, screenshot.width, screenshot.height)
        
        # 7. 记录助手回复到上下文
        context.append(assistant_message(f"<think>{thinking}</think><answer>{action}</answer>"))
        
        # 8. 判断是否完成
        finished = action._metadata == "finish" or result.should_finish
        return StepResult(finished=finished, ...)
```

### 4.2 模型客户端（model/client.py）

```python
class ModelClient:
    def __init__(config):
        self.client = OpenAI(base_url=config.base_url, api_key=config.api_key)
    
    def request(messages) -> ModelResponse:
        start_time = now()
        
        # 流式调用 OpenAI 兼容 API
        stream = client.chat.completions.create(
            messages=messages,
            model=config.model_name,
            max_tokens=3000,
            temperature=0.0,
            stream=True
        )
        
        # 流式读取，区分 thinking 和 action 阶段
        raw_content = ""
        for chunk in stream:
            content = chunk.choices[0].delta.content
            raw_content += content
            
            # 遇到 "do(action=" 或 "finish(message=" 时进入 action 阶段
            if "do(action=" in buffer or "finish(message=" in buffer:
                in_action_phase = True  # 停止打印，静默收集
            else:
                print(content)  # 实时打印思考过程
        
        # 解析 thinking 和 action
        thinking, action = parse_response(raw_content)
        return ModelResponse(thinking, action, metrics...)
    
    def parse_response(content) -> (thinking, action):
        # 规则 1: 按 "finish(message=" 分割
        # 规则 2: 按 "do(action=" 分割
        # 规则 3: 回退到 <think>/<answer> XML 标签
        # 规则 4: 全部内容作为 action
```

### 4.3 动作解析器（actions/handler.py）

```python
def parse_action(response: str) -> dict:
    """
    将模型输出的字符串解析为动作字典。
    
    输入示例：
      'do(action="Tap", element=[500, 300])'
      'do(action="Launch", app="微信")'
      'do(action="Type", text="你好")'
      'finish(message="任务完成")'
    
    解析方式：
      - Type 动作：字符串分割提取 text
      - 其他 do()：使用 ast.parse() 安全解析（不用 eval）
      - finish()：字符串分割提取 message
    """
    if response.startswith('do(action="Type"'):
        text = response.split("text=", 1)[1][1:-2]  # 提取引号内文本
        return {"_metadata": "do", "action": "Type", "text": text}
    
    elif response.startswith("do"):
        tree = ast.parse(response, mode="eval")  # 安全 AST 解析
        action = {"_metadata": "do"}
        for keyword in tree.body.keywords:
            action[keyword.arg] = ast.literal_eval(keyword.value)
        return action
    
    elif response.startswith("finish"):
        return {"_metadata": "finish", "message": ...}

class ActionHandler:
    """将解析后的动作分发到对应的设备操作。"""
    
    handlers = {
        "Launch":     -> device_factory.launch_app(app_name),
        "Tap":        -> device_factory.tap(x, y),     # 坐标从 0-1000 转换为绝对像素
        "Type":       -> 切换 ADB Keyboard -> 清除 -> 输入 -> 恢复键盘,
        "Swipe":      -> device_factory.swipe(start, end),
        "Back":       -> device_factory.back(),
        "Home":       -> device_factory.home(),
        "Double Tap": -> device_factory.double_tap(x, y),
        "Long Press": -> device_factory.long_press(x, y),
        "Wait":       -> time.sleep(duration),
        "Take_over":  -> 调用回调函数，暂停等待用户手动操作,
    }
    
    def _convert_relative_to_absolute(element, screen_w, screen_h):
        x = element[0] / 1000 * screen_w  # 0-1000 映射到 0-screen_w
        y = element[1] / 1000 * screen_h
        return x, y
```

### 4.4 设备工厂（device_factory.py）

```python
class DeviceType(Enum):
    ADB = "adb"   # Android
    HDC = "hdc"   # HarmonyOS
    IOS = "ios"   # iPhone

class DeviceFactory:
    """工厂模式：根据 device_type 延迟加载对应的平台模块。"""
    
    def __init__(device_type=DeviceType.ADB):
        self.device_type = device_type
        self._module = None  # 延迟加载
    
    @property
    def module(self):
        if self._module is None:
            if device_type == ADB:
                from phone_agent import adb
                self._module = adb
            elif device_type == HDC:
                from phone_agent import hdc
                self._module = hdc
        return self._module
    
    # 所有方法都委托给具体平台模块
    def tap(x, y, device_id):      return self.module.tap(x, y, device_id)
    def swipe(...):                 return self.module.swipe(...)
    def get_screenshot(device_id):  return self.module.get_screenshot(device_id)
    def launch_app(app_name):       return self.module.launch_app(app_name)
    ...

# 全局单例
_device_factory = None

def set_device_type(device_type):
    global _device_factory
    _device_factory = DeviceFactory(device_type)

def get_device_factory() -> DeviceFactory:
    global _device_factory
    if _device_factory is None:
        _device_factory = DeviceFactory(DeviceType.ADB)  # 默认 Android
    return _device_factory
```

### 4.5 ADB 设备层（adb/device.py + input.py + screenshot.py）

```python
# --- device.py ---
def tap(x, y, device_id=None):
    subprocess.run(["adb", "-s", device_id, "shell", "input", "tap", str(x), str(y)])
    time.sleep(delay)

def long_press(x, y, duration_ms=3000):
    # 技巧：用 swipe 到同一点实现长按
    subprocess.run([..., "shell", "input", "swipe", str(x), str(y), str(x), str(y), str(duration_ms)])

def launch_app(app_name):
    package = APP_PACKAGES[app_name]  # "微信" -> "com.tencent.mm"
    subprocess.run([..., "shell", "monkey", "-p", package, "-c", "android.intent.category.LAUNCHER", "1"])

def get_current_app():
    output = subprocess.run([..., "shell", "dumpsys", "window"])
    for line in output:
        if "mCurrentFocus" in line:
            for app_name, package in APP_PACKAGES.items():
                if package in line:
                    return app_name
    return "System Home"

# --- input.py ---
def type_text(text):
    # 使用 ADB Keyboard 的 base64 广播协议（支持中文）
    encoded = base64.b64encode(text.encode("utf-8")).decode("utf-8")
    subprocess.run([..., "shell", "am", "broadcast", "-a", "ADB_INPUT_B64", "--es", "msg", encoded])

def detect_and_set_adb_keyboard():
    current_ime = subprocess.run([..., "shell", "settings", "get", "secure", "default_input_method"])
    if "adbkeyboard" not in current_ime:
        subprocess.run([..., "shell", "ime", "set", "com.android.adbkeyboard/.AdbIME"])
    return current_ime  # 返回原始 IME 以便恢复

# --- screenshot.py ---
def get_screenshot(device_id):
    subprocess.run([..., "shell", "screencap", "-p", "/sdcard/tmp.png"])
    subprocess.run([..., "pull", "/sdcard/tmp.png", temp_path])
    img = Image.open(temp_path)
    return Screenshot(base64_data=base64_encode(img), width=w, height=h)
    # 失败时返回黑色 1080x2400 图片（检测敏感页面）
```

### 4.6 iOS 设备层（xctest/device.py）

```python
SCALE_FACTOR = 3  # iPhone Retina 缩放因子

def tap(x, y, wda_url, session_id):
    # 使用 W3C WebDriver Actions API
    actions = {
        "actions": [{
            "type": "pointer", "id": "finger1",
            "parameters": {"pointerType": "touch"},
            "actions": [
                {"type": "pointerMove", "x": x / SCALE_FACTOR, "y": y / SCALE_FACTOR},
                {"type": "pointerDown", "button": 0},
                {"type": "pause", "duration": 0.1},
                {"type": "pointerUp", "button": 0},
            ]
        }]
    }
    requests.post(f"{wda_url}/session/{session_id}/actions", json=actions)

def swipe(start_x, start_y, end_x, end_y):
    # 使用 WDA 专有的 dragfromtoforduration 端点
    payload = {
        "fromX": start_x / SCALE_FACTOR, "fromY": start_y / SCALE_FACTOR,
        "toX": end_x / SCALE_FACTOR, "toY": end_y / SCALE_FACTOR,
        "duration": calculated_duration
    }
    requests.post(f"{wda_url}/session/{session_id}/wda/dragfromtoforduration", json=payload)

def back():
    # iOS 没有返回键，模拟左边缘向右滑动手势
    swipe(fromX=0, fromY=640, toX=400, toY=640, duration=0.3)

def home():
    requests.post(f"{wda_url}/wda/homescreen")

def launch_app(app_name):
    bundle_id = APP_PACKAGES_IOS[app_name]
    requests.post(f"{wda_url}/session/{session_id}/wda/apps/launch", json={"bundleId": bundle_id})
```

---

## 五、模型输出格式与 Prompt 设计

### 模型输出格式

```
<think>当前在系统桌面，需要先启动小红书应用</think>
<answer>do(action="Launch", app="小红书")</answer>
```

或者新格式（不含 XML 标签）：

```
当前在系统桌面，需要先启动小红书应用
do(action="Launch", app="小红书")
```

### 坐标系统

- 模型输出 **相对坐标**：(0, 0) 到 (999, 999)
- ActionHandler 转换：`x_abs = x_rel / 1000 * screen_width`
- iOS 额外除以 `SCALE_FACTOR=3`（Retina 缩放）

### 支持的动作指令

| 动作 | 格式 | 说明 |
|------|------|------|
| Launch | `do(action="Launch", app="微信")` | 按名称启动 App |
| Tap | `do(action="Tap", element=[x,y])` | 点击坐标 |
| Tap(敏感) | `do(action="Tap", element=[x,y], message="确认支付")` | 带确认的点击 |
| Type | `do(action="Type", text="你好")` | 输入文本 |
| Swipe | `do(action="Swipe", start=[x1,y1], end=[x2,y2])` | 滑动 |
| Back | `do(action="Back")` | 返回 |
| Home | `do(action="Home")` | 回桌面 |
| Long Press | `do(action="Long Press", element=[x,y])` | 长按 |
| Double Tap | `do(action="Double Tap", element=[x,y])` | 双击 |
| Wait | `do(action="Wait", duration="2 seconds")` | 等待 |
| Take_over | `do(action="Take_over", message="请输入验证码")` | 人工接管 |
| Note | `do(action="Note", message="True")` | 记录页面 |
| Call_API | `do(action="Call_API", instruction="总结内容")` | 内容总结 |
| Interact | `do(action="Interact")` | 请求用户选择 |
| finish | `finish(message="任务完成")` | 结束任务 |

---

## 六、与 mobile-mcp / appium-mcp 的对比

### 6.1 架构维度对比

| 维度 | Open-AutoGLM | mobile-mcp | appium-mcp |
|------|-------------|------------|------------|
| **定位** | 自主 Agent（自带 AI） | MCP 工具服务器 | MCP 工具服务器 |
| **语言** | Python | TypeScript | Python |
| **AI 角色** | 内置 VLM 推理客户端 | 被动，由外部 AI 调用 | 被动，由外部 AI 调用 |
| **协议** | 无（CLI + Python API） | MCP（stdio/SSE） | MCP（stdio） |
| **设备交互层数** | 2 层（Agent→设备） | 2 层（MCP→设备） | 3 层（MCP→Appium→设备） |
| **平台支持** | Android + HarmonyOS + iOS | Android + iOS | Android + iOS |
| **执行模型** | 同步阻塞 | 同步阻塞（execFileSync） | 异步（asyncio） |
| **并发支持** | 无 | 无 | 有（SessionPool + asyncio.Lock） |

### 6.2 底层命令覆盖对比

| 操作 | Open-AutoGLM | mobile-mcp | appium-mcp |
|------|-------------|------------|------------|
| 点击 | `input tap` / `uitest click` / WDA Actions | `input tap` | Appium click |
| 双击 | 两次 tap / `uitest doubleClick` / WDA Actions | `input tap` x2 | 无 |
| 长按 | `input swipe` 同点 / `uitest longClick` / WDA Actions | `input swipe` 同点 | 无 |
| 滑动 | `input swipe` / `uitest swipe` / WDA drag | `input swipe` | Appium swipe |
| 输入文本 | ADB Keyboard base64 广播 | `input text`（仅 ASCII） | Appium send_keys |
| 截图 | `screencap` + `pull` / WDA `/screenshot` | `screencap`（管道直读） | Appium screenshot |
| 获取前台 App | `dumpsys window` / `aa dump` / WDA activeAppInfo | `dumpsys window` | 无 |
| 启动 App | `monkey` / `aa start` / WDA apps/launch | `monkey` / `am start` | 无 |
| 安装/卸载 App | 不支持 | 支持（`install` / `uninstall`） | 不支持 |
| UI 元素获取 | 不支持（纯坐标） | `uiautomator dump`（XML 解析） | Appium find_element（10 种定位器） |
| 录屏 | 不支持 | 支持（`screenrecord`） | 不支持 |
| 屏幕旋转 | 不支持 | 支持（`settings put`） | 不支持 |

### 6.3 交互模型差异

```
Open-AutoGLM:
  用户 --自然语言--> Agent --VLM推理--> 动作指令 --ADB/WDA--> 设备
  (完全自主循环，用户只提供初始任务)

mobile-mcp:
  AI Agent --MCP Tool调用--> MCP Server --ADB/WDA--> 设备
  (AI Agent 自己决定调用哪个 Tool，MCP Server 只是执行层)

appium-mcp:
  AI Agent --MCP Tool调用--> MCP Server --Appium Client--> Appium Server --Driver--> 设备
  (多了 Appium Server 中间层)
```

### 6.4 文本输入方案对比

| 方案 | Open-AutoGLM | mobile-mcp | appium-mcp |
|------|-------------|------------|------------|
| 方式 | ADB Keyboard base64 广播 | `adb shell input text` | Appium send_keys |
| 中文支持 | 原生支持 | 不支持（仅 ASCII） | 原生支持 |
| 额外依赖 | 需安装 ADB Keyboard APK | 无（非 ASCII 需 DeviceKit） | 需 Appium Server |
| 输入前清除 | 自动清除 + ADB_CLEAR_TEXT 广播 | 不自动清除 | WebDriverWait 定位后清除 |

### 6.5 安全机制对比

| 维度 | Open-AutoGLM | mobile-mcp | appium-mcp |
|------|-------------|------------|------------|
| 敏感操作确认 | 有（confirmation_callback） | 无 | 无 |
| 人工接管 | 有（Take_over 动作 + takeover_callback） | 无 | 无 |
| 截图黑屏检测 | 有（返回 is_sensitive=True） | 无 | 无 |
| 命令注入防护 | 无（使用 AST 解析代替 eval） | 有（输入校验） | 无（不直接执行 shell） |
| URL 限制 | 无 | 有（MOBILEMCP_ALLOW_UNSAFE_URLS） | 无 |

### 6.6 设计哲学差异

| 维度 | Open-AutoGLM | mobile-mcp | appium-mcp |
|------|-------------|------------|------------|
| 核心思想 | **端到端自主 Agent** | **工具即服务** | **Appium 生态桥接** |
| AI 位置 | AI 在框架内部 | AI 在框架外部 | AI 在框架外部 |
| 扩展方式 | 修改 System Prompt + App 映射 | 注册新 MCP Tool | 继承 AppiumTool 基类 |
| 适合谁 | 想要开箱即用的自主手机助理 | 想让已有 AI Agent 控制手机 | 已有 Appium 生态的测试团队 |
| 独特优势 | 三平台 + VLM 推理 + 人工接管 | 功能最全 + 轻量无额外进程 | 10 种元素定位 + 并发 |

---

## 七、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.py` | ~854 | CLI 入口、系统检查、参数解析 |
| `phone_agent/__init__.py` | 12 | 包导出 |
| `phone_agent/agent.py` | 254 | Android/HarmonyOS Agent 主循环 |
| `phone_agent/agent_ios.py` | 278 | iOS Agent 主循环 |
| `phone_agent/device_factory.py` | 168 | 工厂模式，ADB/HDC 切换 |
| `phone_agent/model/client.py` | 291 | OpenAI 兼容 VLM 客户端 |
| `phone_agent/actions/handler.py` | 400 | 动作解析 + Android 执行 |
| `phone_agent/actions/handler_ios.py` | 281 | 动作解析 + iOS 执行 |
| `phone_agent/adb/device.py` | 253 | Android ADB 设备控制 |
| `phone_agent/adb/input.py` | 110 | Android 文本输入（ADB Keyboard） |
| `phone_agent/adb/screenshot.py` | 110 | Android 截图 |
| `phone_agent/adb/connection.py` | 354 | ADB 连接管理 |
| `phone_agent/hdc/device.py` | 311 | HarmonyOS HDC 设备控制 |
| `phone_agent/xctest/device.py` | 459 | iOS WDA 设备控制 |
| `phone_agent/xctest/input.py` | 300 | iOS 文本输入 |
| `phone_agent/xctest/screenshot.py` | 231 | iOS 截图 |
| `phone_agent/xctest/connection.py` | 383 | iOS 连接 + WDA 管理 |
| `phone_agent/config/prompts_zh.py` | ~70 | 中文系统提示词 |
| `phone_agent/config/prompts_en.py` | ~70 | 英文系统提示词 |
| `phone_agent/config/apps.py` | - | Android App 名称→包名映射（50+） |
| `phone_agent/config/apps_ios.py` | - | iOS App 名称→BundleID 映射 |
| `phone_agent/config/apps_harmonyos.py` | - | 鸿蒙 App 名称→包名映射（60+） |
| `phone_agent/config/timing.py` | - | 操作延迟时间配置 |
| `phone_agent/config/i18n.py` | - | 国际化消息 |
