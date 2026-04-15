# mobile-mcp 快速上手

> 仓库地址：https://github.com/mobile-next/mobile-mcp
> 本地路径：`/Users/yuhaofei/androidstudio/AI/mobile-mcp`

---

## 一、环境要求

| 依赖 | 最低版本 | 说明 |
|------|---------|------|
| Node.js | v18+（推荐 v22+） | 运行 MCP Server |
| npm | 随 Node.js 安装 | 包管理 |
| Android SDK Platform Tools | 最新版 | 提供 `adb` 命令 |
| Xcode Command Line Tools | 最新版 | iOS 模拟器支持（macOS） |
| go-ios | 最新版 | iOS 真机支持（可选） |
| WebDriverAgent | - | iOS 真机/模拟器 UI 交互（可选） |

### 环境变量

| 变量 | 用途 | 默认行为 |
|------|------|---------|
| `ANDROID_HOME` | Android SDK 路径 | 自动检测常见路径 |
| `GO_IOS_PATH` | go-ios 二进制路径 | 使用 PATH 中的 `ios` |
| `MOBILECLI_PATH` | mobilecli 路径 | 自动检测 node_modules 内 |
| `MOBILEMCP_DISABLE_TELEMETRY` | 禁用匿名遥测 | 遥测开启 |
| `MOBILEMCP_ALLOW_UNSAFE_URLS` | 允许非 http/https URL | 只允许 http/https |
| `MOBILEMCP_AUTH` | SSE 模式的 Bearer Token | 无认证 |
| `MOBILEFLEET_ENABLE` | 启用远程设备池功能 | 不启用 |
| `LOG_FILE` | 日志文件路径 | 仅输出到 stderr |

---

## 二、安装与启动

### 方式 1：npx 直接运行（推荐）

```bash
npx -y @mobilenext/mobile-mcp@latest
```

### 方式 2：本地开发

```bash
cd /Users/yuhaofei/androidstudio/AI/mobile-mcp
npm install
npm run build    # tsc 编译到 lib/
node lib/index.js
```

### 方式 3：SSE 服务器模式

```bash
# 监听 localhost:3000
npx @mobilenext/mobile-mcp@latest --listen 3000

# 监听指定地址
npx @mobilenext/mobile-mcp@latest --listen 0.0.0.0:3000

# 带认证
MOBILEMCP_AUTH=my-secret-token npx @mobilenext/mobile-mcp@latest --listen 3000
```

---

## 三、MCP 客户端配置

### Claude Desktop / Cursor / Cline / VS Code 等

在 MCP 配置文件中添加：

```json
{
  "mcpServers": {
    "mobile-mcp": {
      "command": "npx",
      "args": ["-y", "@mobilenext/mobile-mcp@latest"]
    }
  }
}
```

### Claude Code CLI

```bash
claude mcp add mobile-mcp -- npx -y @mobilenext/mobile-mcp@latest
```

---

## 四、设备准备

### Android 模拟器 / 真机

1. 确保 `adb` 可用：
   ```bash
   adb devices
   # 应能看到设备列表
   ```
2. 连接设备或启动模拟器后即可使用

### iOS 模拟器

1. 启动模拟器：
   ```bash
   xcrun simctl list        # 查看可用模拟器
   xcrun simctl boot "iPhone 16"
   ```
2. mobile-mcp 会自动检测已启动的模拟器

### iOS 真机（高级）

需要额外配置 go-ios + WebDriverAgent + iOS tunnel，参见 [Wiki](https://github.com/mobile-next/mobile-mcp/wiki)。

---

## 五、验证安装

启动 MCP Server 后，通过 AI Agent 发送以下指令：

```
列出所有可用设备
```

Agent 会调用 `mobile_list_available_devices` Tool，返回设备列表。

接着可以尝试：

```
对设备 {deviceId} 截一张屏幕截图
```

---

## 六、开发命令

| 命令 | 说明 |
|------|------|
| `npm run build` | TypeScript 编译 |
| `npm run watch` | 编译监听模式 |
| `npm run lint` | ESLint 检查 |
| `npm run fixlint` | ESLint 自动修复 |
| `npm test` | 运行 mocha 测试（nyc 覆盖率） |
| `npm run clean` | 清理编译产物 |

---

## 七、可用 MCP Tool 列表速查

### 设备管理
- `mobile_list_available_devices` — 列出所有设备
- `mobile_get_screen_size` — 获取屏幕尺寸
- `mobile_get_orientation` — 获取屏幕方向
- `mobile_set_orientation` — 设置屏幕方向

### App 管理
- `mobile_list_apps` — 列出已安装 App
- `mobile_launch_app` — 启动 App
- `mobile_terminate_app` — 停止 App
- `mobile_install_app` — 安装 App
- `mobile_uninstall_app` — 卸载 App

### 屏幕交互
- `mobile_take_screenshot` — 截图（返回图片给 AI）
- `mobile_save_screenshot` — 截图保存到文件
- `mobile_list_elements_on_screen` — 列出 UI 元素
- `mobile_click_on_screen_at_coordinates` — 点击
- `mobile_double_tap_on_screen` — 双击
- `mobile_long_press_on_screen_at_coordinates` — 长按
- `mobile_swipe_on_screen` — 滑动

### 输入与导航
- `mobile_type_keys` — 输入文本
- `mobile_press_button` — 按键（HOME/BACK/ENTER 等）
- `mobile_open_url` — 打开 URL

### 录屏
- `mobile_start_screen_recording` — 开始录屏
- `mobile_stop_screen_recording` — 停止录屏

### 设备池（需 MOBILEFLEET_ENABLE=1）
- `mobile_list_fleet_devices` — 列出远程设备
- `mobile_allocate_fleet_device` — 分配设备
- `mobile_release_fleet_device` — 释放设备

---

## 八、常见问题

### Q: Android 设备列表为空？

确认 `ANDROID_HOME` 环境变量设置正确，或 `adb` 在 PATH 中。运行 `adb devices` 确认设备在线。

### Q: 非 ASCII 字符（如中文）输入失败？

Android 的 `adb shell input text` 只支持 ASCII。需要安装 [DeviceKit](https://github.com/mobile-next/devicekit-android) 来支持非 ASCII 文本输入。

### Q: iOS 真机 "tunnel is not running" 错误？

iOS 17+ 设备需要运行 ios tunnel。参见 [Wiki](https://github.com/mobile-next/mobile-mcp/wiki/)。

### Q: uiautomator dump 失败（"null root node"）？

代码已内置最多 10 次重试。如果仍然失败，可能是当前界面正在动画或过渡中，稍后重试。

### Q: 截图文件很大，AI 处理慢？

安装 ImageMagick（`brew install imagemagick`）后，截图会自动压缩为 JPEG quality=75。macOS 上会优先使用系统自带的 sips。

---

## 九、建议先读的源码

1. **`src/robot.ts`** — Robot 接口定义，理解所有能力的边界
2. **`src/android.ts`** — 最完整的实现，展示了所有原子 adb 命令的用法
3. **`src/server.ts`** — Tool 注册层，理解参数如何校验、Robot 如何路由
4. **`src/webdriver-agent.ts`** — WDA HTTP 客户端，理解 iOS 侧的交互协议
5. **`src/utils.ts`** — 安全校验逻辑，理解输入防护策略
