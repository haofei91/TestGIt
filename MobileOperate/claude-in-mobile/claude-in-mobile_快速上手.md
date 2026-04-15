# claude-in-mobile 快速上手

> 仓库地址：https://github.com/AlexGladkov/claude-in-mobile
> 版本：v3.3.0 | 语言：TypeScript (Node.js 18+)
> 平台：Android + iOS + Desktop + Aurora OS + Browser

---

## 一、项目简介

claude-in-mobile 是一个**全平台 MCP Server**，通过 MCP 协议让 AI Agent 控制 Android、iOS、桌面应用、Aurora OS 和浏览器。它提供约 40 个工具，内置 Flow 引擎、模糊匹配、UI Diff 等高级特性。

**适合场景**：
- 需要同时支持多个平台的自动化
- 需要高容错性（130+ 别名映射）
- 需要复杂流程自动化（条件/循环/错误处理）
- 需要智能 UI 分析（模糊匹配、屏幕分析、diff）

---

## 二、环境准备

### 2.1 基础依赖

```bash
# Node.js 18+
node --version  # 确认 >= 18

# npm
npm --version
```

### 2.2 Android 环境

```bash
# ADB
adb version
adb devices  # 确认设备已连接

# 开发者选项
# 设备端开启: USB 调试 + 安装来源允许
```

### 2.3 iOS 环境 (可选)

```bash
# Xcode + simctl
xcrun simctl list devices  # 列出模拟器

# WebDriverAgent (WDA)
# 需要 Xcode 构建 WDA 到模拟器
```

### 2.4 浏览器环境 (可选)

```bash
# Chrome 浏览器
# 项目使用 chrome-launcher 自动启动
```

---

## 三、安装方式

### 方式 1: npm 全局安装

```bash
npm install -g claude-in-mobile
```

### 方式 2: 从源码构建

```bash
git clone https://github.com/AlexGladkov/claude-in-mobile.git
cd claude-in-mobile
npm install
npm run build
```

### 方式 3: Rust CLI (原生二进制)

```bash
# macOS via Homebrew
brew install alexgladkov/tap/claude-in-mobile

# ~2MB 二进制，无需 Node.js
```

---

## 四、MCP 客户端配置

### 4.1 Claude Desktop

编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "claude-in-mobile": {
      "command": "npx",
      "args": ["claude-in-mobile"]
    }
  }
}
```

从源码运行：

```json
{
  "mcpServers": {
    "claude-in-mobile": {
      "command": "node",
      "args": ["/path/to/claude-in-mobile/dist/index.js"]
    }
  }
}
```

### 4.2 Cursor

Settings → MCP → 添加 Server：

```json
{
  "claude-in-mobile": {
    "command": "npx",
    "args": ["claude-in-mobile"]
  }
}
```

### 4.3 Claude Code (插件模式)

```bash
# 作为 Claude Code 插件使用
claude plugin add claude-in-mobile
```

---

## 五、核心工具使用

### 5.1 交互工具

#### 点击

```
# 坐标点击
→ input_tap(x: 540, y: 960)

# 按文本查找并点击 (Android)
→ input_tap(text: "Settings")

# 按 resourceId 查找并点击
→ input_tap(resourceId: "com.example:id/btn_login")

# 按 index 点击 (需先调用 ui_tree 缓存元素)
→ input_tap(index: 5)

# iOS 按 label 点击
→ input_tap(label: "Settings")
```

#### 其他交互

```
# 长按
→ input_long_press(x: 540, y: 960, duration: 1000)

# 滑动
→ input_swipe(x1: 540, y1: 1500, x2: 540, y2: 500, duration: 300)

# 输入文本
→ input_text(text: "hello world")

# 按键
→ input_key(key: "BACK")
→ input_key(key: "HOME")
→ input_key(key: "ENTER")
```

**别名说明**：以下名称都指向同一个工具：
- `tap` = `click` = `press` = `input_tap`
- `long_tap` = `long_click` = `input_long_press`
- `screenshot` = `capture` = `screen_capture`
- `get_ui` = `hierarchy` = `ui_tree`

### 5.2 UI 分析工具

#### 获取 UI 树

```
→ ui_tree()
# 返回格式化的 UI 元素树，包含 index、text、resourceId、bounds 等
```

#### 模糊查找并点击

```
→ ui_find_tap(query: "登录", minConfidence: 50)
# 模糊匹配所有元素，找到最佳匹配并自动点击
# 返回匹配置信度 (0-100%)
```

#### 等待元素

```
→ ui_wait(text: "加载完成", timeout: 5000, interval: 500)
# 轮询等待元素出现，最多 5 秒
```

#### 断言

```
→ ui_assert_visible(text: "欢迎")
# 断言元素可见，返回 pass/fail (不截图)

→ ui_assert_gone(text: "加载中")
# 断言元素消失
```

#### 屏幕分析

```
→ ui_analyze()
# 返回结构化信息: 标题、对话框、导航、按钮、输入框等
```

### 5.3 截图工具

#### 截图

```
→ screen_capture()
# 智能截图: 稳定截图 + 自动压缩 + 缩放因子记录

# Diff 模式 (与上次截图对比)
→ screen_capture(diff: true)
# < 5% 变化: 文字描述
# 5-80% 变化: 裁剪变化区域
# > 80% 变化: 完整截图
```

#### 标注截图

```
→ screen_annotate()
# 绿色框: 可点击元素
# 红色框: 不可点击元素
# 数字标签: 元素编号
```

### 5.4 Flow 引擎

#### 批量执行

```
→ flow_batch(commands: [
    {tool: "input_tap", args: {text: "Settings"}},
    {tool: "input_tap", args: {text: "About"}},
    {tool: "screen_capture", args: {}}
  ])
# 单次 MCP 往返执行多个命令 (最多 50 个)
```

#### 条件流程

```
→ flow_run(steps: [
    {
      action: "input_tap",
      args: {text: "登录按钮"},
      if_not_found: "scroll_down"     // 找不到就下滑
    },
    {
      action: "input_text",
      args: {text: "password"},
      on_error: "skip"                // 出错就跳过
    },
    {
      action: "input_tap",
      args: {text: "确认"},
      repeat: {times: 3, until_found: "欢迎页"}  // 重复直到成功
    }
  ])
```

条件选项：
- `if_not_found`: `"skip"` | `"scroll_down"` | `"scroll_up"` | `"fail"`
- `on_error`: `"stop"` | `"skip"` | `"retry"`
- `repeat`: `{times, until_found, until_not_found}`

### 5.5 应用管理

```
# 启动应用
→ app_launch(package: "com.example.app")

# 停止应用
→ app_stop(package: "com.example.app")

# 安装应用
→ app_install(path: "/path/to/app.apk")

# 权限管理
→ permission_grant(package: "com.example.app", permission: "android.permission.CAMERA")
→ permission_revoke(package: "com.example.app", permission: "android.permission.CAMERA")
```

### 5.6 设备管理

```
# 列出设备
→ device_list()

# 选择设备
→ device_select(id: "emulator-5554")

# 获取系统信息
→ system_info()
```

---

## 六、高级特性

### 6.1 坐标缩放

截图发送给 LLM 前会被压缩，LLM 返回的坐标基于压缩后的分辨率。claude-in-mobile 自动记录缩放因子并修正：

```
设备 1080x1920 → 压缩 540x960 → LLM 坐标 (270,480) → 修正为 (540,960)
```

无需手动处理，系统自动完成。

### 6.2 Action Hints

操作后自动检测 UI 变化：

```
→ input_tap(x: 540, y: 960, hints: true)
# 返回: "页面切换到 Settings 页面，检测到 5 个可点击按钮"
```

减少 LLM 需要额外截图确认的次数，降低 token 消耗。

### 6.3 模糊匹配评分

`ui_find_tap` 的评分规则：

| 匹配类型 | 分数 |
|----------|------|
| 文本精确匹配 | 100 |
| content-desc 匹配 | 95 |
| 文本包含 | 80 |
| 描述包含 | 75 |
| resourceId 匹配 | 60 |
| 部分文本匹配 | 40 |
| 部分描述匹配 | 35 |
| 可点击元素加分 | +10 |

### 6.4 稳定截图

自动防止动画中间帧：
- 连续截 2 张，差异 < 2% 认为稳定
- 最多重试 3 次，间隔 300ms

---

## 七、平台特定说明

### 7.1 Android

- 通过 ADB 直接通信
- 支持全部工具
- `getCurrentActivity()` 使用 6 种正则兼容不同 Android 版本
- 剪贴板操作需 Android 10+

### 7.2 iOS

- 通过 simctl 管理模拟器
- 通过 WDA 获取 UI 和执行操作
- 自动检测已启动的模拟器
- 需要 Xcode 环境
- 不支持物理设备

### 7.3 Desktop (Compose Multiplatform)

- 通过 Accessibility API 交互
- 需要编译 Desktop Companion 应用
- 支持 macOS/Windows/Linux

### 7.4 Browser

- 通过 Chrome DevTools Protocol (CDP)
- chrome-launcher 自动启动
- 支持导航、点击、输入、截图、DOM 查询

---

## 八、常见问题

### Q1: 设备连接不上？

```bash
# 检查 ADB 连接
adb devices

# 重启 ADB server
adb kill-server && adb start-server

# 无线连接
adb connect <device-ip>:5555
```

### Q2: 工具名调用失败？

claude-in-mobile 有 130+ 别名，常见的名称都能识别。如果仍然失败：
- 检查别名映射是否包含你的工具名
- 使用规范名 (如 `input_tap` 而非 `click`)

### Q3: 坐标点击位置不对？

通常是坐标缩放问题：
- 确认 `screen_capture` 返回的截图与设备实际分辨率的关系
- 系统自动修正，但首次截图前需要先调用一次 `screen_capture`

### Q4: iOS WDA 启动失败？

```bash
# 确认 Xcode 已安装
xcode-select -p

# 确认模拟器已启动
xcrun simctl list devices | grep Booted

# WDA 需要先编译安装到模拟器
# 参考 WebDriverAgent 官方文档
```

### Q5: Flow 执行超时？

- 默认超时 60s，最多 20 步
- 检查 `if_not_found` 和 `on_error` 配置
- 避免无限循环 (最多 10 次重复)

### Q6: 截图 token 消耗太大？

- 使用 diff 模式: `screen_capture(diff: true)`
- Action Hints 减少截图需求
- 稳定截图避免重复捕获

---

## 九、与其他项目的选型建议

| 需求 | 推荐项目 |
|------|---------|
| 仅 Android + 快速集成 | android-mcp-server (200 行，最轻量) |
| Android + Selector 查找 | Android-MCP CursorTouch (u2 原生 Selector) |
| Android + iOS 双平台 | mobile-mcp (Maestro 集成) |
| 全平台 + 高级特性 | **claude-in-mobile** (5 平台 + Flow + Diff) |
| Android 自治 Agent | droidrun (Manager/Executor 模式) |
| 学术研究 + 自动化学习 | AppAgent (UI 文档 + 探索-利用) |
| 端到端模型训练 | Open-AutoGLM (强化学习 + 自定义 ROM) |
| Appium 生态集成 | appium-mcp (WebDriver 协议) |

### 选择 claude-in-mobile 的理由：

1. **最全面**：5 个平台、40 个工具、130+ 别名
2. **最容错**：别名系统解决 LLM 命名偏差
3. **最高效**：Flow 引擎减少 MCP 往返、diff 截图减少 token
4. **最智能**：模糊匹配 + Action Hints + 屏幕分析

### 不选择的理由：

1. **过于复杂**：仅需 Android 基础操作时，android-mcp-server 更轻量
2. **需要 Selector**：u2 原生 Selector 更可靠，选 Android-MCP CursorTouch
3. **需要自治 Agent**：选 droidrun 或 AppAgent
4. **资源受限**：75+ 文件的维护成本较高
