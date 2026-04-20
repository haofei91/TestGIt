# Appium 方案复盘与总结

> 目标：点击 Android 真机（小米 M2007J1SC，Android 13，设备 ID: 883e043e）招行 App「我的自选」页面中的「120057A」理财产品行，触发页面跳转。

---

## 一、安装与配置内容

### 电脑端

| 内容 | 路径 / 说明 |
|------|------------|
| Appium Server v3.3.0 | `npm install -g appium`，全局安装 |
| UIAutomator2 Driver v7.1.2 | `appium driver install uiautomator2` |
| UIAutomator2 Driver 文件 | `~/.appium/node_modules/appium-uiautomator2-driver/` |
| Python 3.11 虚拟环境 | `~/Documents/coding/github/appium-mcp/.venv/` |
| appium-mcp-server | 通过 `uv pip install -e .` 安装在上述虚拟环境中 |
| Chromedriver 135 | Appium 启动时自动下载，适配设备 Chrome 版本 |

### 手机端（均已卸载）

| APK 包名 | 说明 |
|---------|------|
| `io.appium.settings` | Appium 辅助应用，用于修改系统设置 |
| `io.appium.uiautomator2.server` | UIAutomator2 HTTP 服务端，运行在设备上 |
| `io.appium.uiautomator2.server.test` | UIAutomator2 测试运行器（Instrumentation） |

> ✅ 手机端 3 个 APK 已通过 `adb uninstall` 卸载完毕。

---

## 二、为什么尝试了这么多还没成功？

### 根本原因：遇到了两道独立的墙

#### 第一道墙：Android 13 封锁输入注入权限

Android 13 收紧了 `INJECT_EVENTS` 权限，所有通过 ADB 或 UIAutomator2 服务注入触摸事件的方式均被拒绝：

| 方案 | 失败原因 |
|------|---------|
| `adb shell input tap` | `SecurityException: Injecting input events requires INJECT_EVENTS permission` |
| Appium `mobile: clickGesture` | 底层同样调用 `UiAutomation.injectInputEvent()`，同样被拒绝 |
| Appium W3C Actions API | 同上 |

唯一能成功注入点击的是 `android-operate-cli`，因为它通过 `am instrument` 运行 UIAutomator，这个模式自动获得注入权限。

#### 第二道墙：点击成功但 WebView 不响应

`android-operate-cli` 确实成功点击了（坐标 362, 1865），**但页面没有跳转**。  
原因：招行 App 的列表是 WebView 渲染的，JS 点击事件绑定在 DOM 元素上，而不是原生 View 上。UIAutomator 的坐标 tap 在触摸事件分发上与真实手指操作有差异，WebView 内的 JS `onClick` 没有被触发。

#### 问题被放大的附加原因

引入 Appium 后，接连暴露了一系列与核心问题无关的环境问题：

| 问题 | 解决方式 |
|------|---------|
| Python 3.9 不兼容（需 3.10+） | 用 `uv` 创建 Python 3.11 虚拟环境 |
| APK 安装需要 USB 安装权限 | 用户手动开启「USB 安装」，再手动执行 `adb install` |
| `WRITE_SECURE_SETTINGS` 权限错误 | 添加 `appium:ignoreHiddenApiPolicyError: true` |
| Chromedriver 版本不匹配（Chrome 135） | Appium 重启时加 `--allow-insecure=uiautomator2:chromedriver_autodownload` |
| WebView 远程调试端口不匹配 | 招行生产包未开放 WebView 调试，无法通过 Chromedriver 切换上下文，无解 |
| UiSelector 找不到 WebView 内元素 | WebView 内容不在原生 Accessibility Tree 中，无解 |

**每修一个问题，就暴露出下一个，但始终未能解决核心问题。**

---

## 三、两个命令的区别

```bash
# 1. 启动 Appium Server（独立终端）
appium --port 4723

# 2. 启动 appium-mcp（另一个终端）
appium-mcp-server run
```

### Appium Server（`appium --port 4723`）

- **角色**：核心自动化服务端，监听 HTTP 请求（默认端口 4723）
- **功能**：管理设备会话、调用 UIAutomator2/XCUITest 等 Driver、转发控制指令到手机
- **通信方式**：标准 WebDriver 协议（W3C），任何支持 Selenium/Appium 的客户端（Python/Java/Node）都能连接
- **类比**：相当于「控制中心后台服务」

### appium-mcp-server（`appium-mcp-server run`）

- **角色**：MCP（Model Context Protocol）适配层，把 Appium 的能力封装成 AI 可调用的工具
- **功能**：暴露 `launch_app`、`find_element`、`click_element`、`take_screenshot` 等工具给 AI Agent（如 Claude / Qoder）
- **通信方式**：MCP 协议（stdio/SSE），供 AI 工具链调用，不直接处理设备
- **依赖关系**：必须先启动 Appium Server，appium-mcp-server 才能正常工作——它只是 Appium 的「AI 接口层」
- **类比**：相当于「给控制中心加装了一个 AI 能理解的遥控器」

### 关系图

```
AI Agent (Claude/Qoder)
        ↓ MCP 协议
appium-mcp-server          ← 第 2 步启动
        ↓ HTTP (WebDriver)
Appium Server :4723         ← 第 1 步启动
        ↓ ADB / USB
Android 设备
```

> 简单说：**Appium Server 是引擎，appium-mcp-server 是让 AI 能开这辆车的方向盘。**  
> 缺少 Appium Server，appium-mcp-server 无法工作；  
> 单独有 Appium Server，可以用 Python/Java 代码直接控制，但 AI Agent 无法调用。

---

## 四、后续备选方案

如果仍需实现自动点击招行 App WebView 内元素，可考虑：

1. **adb shell sendevent**：直接写内核输入设备节点，完全绕过 Android 权限体系（需要找到正确的 `/dev/input/eventX` 设备号）
2. **自定义 AccessibilityService APK**：开发一个无障碍服务应用，安装后可模拟真实触摸事件，绕过 `INJECT_EVENTS` 限制
3. **放弃自动化**：手动点击，毕竟这只是个一次性操作
