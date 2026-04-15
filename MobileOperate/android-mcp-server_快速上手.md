# android-mcp-server 快速上手

## 1. 项目简介

android-mcp-server 是一个 MCP 服务器，让 AI 助手（Claude、Cursor、VS Code 等）能够通过 ADB 控制 Android 设备。支持截图、UI 树解析、触控操作、日志读取等 25 个工具。

**核心特点**：零配置安装（npx 一行启动）、纯 ADB 通信（无需修改目标 App）、持久化 Shell 高性能执行。

---

## 2. 环境要求

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| Node.js | >= 18 | 运行时 |
| Android SDK | - | 需要 `platform-tools`（ADB）和 `emulator` |
| ADB | - | 包含在 Android SDK platform-tools 中 |
| 设备/模拟器 | - | 已连接的 Android 设备或运行中的模拟器 |

### 检查环境

```bash
# 检查 Node.js
node --version  # 需要 v18+

# 检查 ADB
adb version

# 检查 ADB 路径（macOS）
ls ~/Library/Android/sdk/platform-tools/adb

# 检查设备连接
adb devices
```

### ANDROID_HOME 设置

服务器会按以下顺序查找 SDK：
1. `$ANDROID_HOME` 环境变量指定的路径
2. `~/Library/Android/sdk`（macOS 默认路径）
3. 系统 `$PATH` 中的 `adb`

如果 SDK 不在默认位置，需要设置 `ANDROID_HOME` 环境变量。

---

## 3. 安装与配置

### 方式一：npx 直接使用（推荐）

无需安装，MCP 客户端会自动通过 npx 拉取并运行。

### 方式二：从源码构建

```bash
git clone https://github.com/martingeidobler/android-mcp-server.git
cd android-mcp-server
npm install
npm run build
```

### MCP 客户端配置

根据你使用的 AI 工具选择对应配置：

#### Claude Code

```bash
# 全局注册（所有项目可用）
claude mcp add --scope user android -- npx -y android-mcp-server

# 指定 SDK 路径
claude mcp add --scope user --env ANDROID_HOME=/path/to/sdk android -- npx -y android-mcp-server

# 仅当前项目
claude mcp add --scope project android -- npx -y android-mcp-server
```

#### Claude Desktop

编辑配置文件：
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "android": {
      "command": "npx",
      "args": ["-y", "android-mcp-server"],
      "env": {
        "ANDROID_HOME": "/path/to/android/sdk"
      }
    }
  }
}
```

#### VS Code

编辑 `.vscode/settings.json`：

```json
{
  "mcp": {
    "servers": {
      "android": {
        "command": "npx",
        "args": ["-y", "android-mcp-server"],
        "env": {
          "ANDROID_HOME": "/path/to/android/sdk"
        }
      }
    }
  }
}
```

#### Cursor

编辑 `~/.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "android": {
      "command": "npx",
      "args": ["-y", "android-mcp-server"],
      "env": {
        "ANDROID_HOME": "/path/to/android/sdk"
      }
    }
  }
}
```

#### Windsurf

编辑 `~/.codeium/windsurf/mcp_config.json`：

```json
{
  "mcpServers": {
    "android": {
      "command": "npx",
      "args": ["-y", "android-mcp-server"],
      "env": {
        "ANDROID_HOME": "/path/to/android/sdk"
      }
    }
  }
}
```

#### 项目级配置（.mcp.json）

在项目根目录创建 `.mcp.json`，可纳入版本控制：

```json
{
  "mcpServers": {
    "android": {
      "command": "npx",
      "args": ["-y", "android-mcp-server"],
      "env": {
        "ANDROID_HOME": "/path/to/android/sdk"
      }
    }
  }
}
```

#### 源码构建方式的配置

```bash
claude mcp add --scope user android -- node /path/to/android-mcp-server/dist/index.js
```

---

## 4. 工具速查表

### 设备管理
| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `list_devices` | 列出已连接设备 | 无 |
| `list_avds` | 列出可用模拟器 | 无 |
| `start_emulator` | 启动模拟器 | `avd_name` |

### 截图与 UI 分析
| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `screenshot` | 截图（自动压缩至 1280px） | `save_path?` |
| `get_ui_tree` | 获取 UI 元素层级 | 无 |

### 交互操作
| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `tap` | 坐标点击 | `x`, `y` |
| `tap_element` | 按属性查找并点击 | `by`, `value` |
| `tap_and_wait` | 点击 + 等待 + 返回 UI 树 | `by`, `value`, `wait_ms?` |
| `long_press` | 长按 | `x`, `y`, `duration_ms?` |
| `double_tap` | 双击 | `x`, `y` |
| `multi_tap` | 多次点击 | `x`, `y`, `count` |
| `tap_sequence` | 复合操作序列 | `steps[]` |
| `type_text` | 输入文本 | `text` |
| `press_key` | 按键 | `key` |
| `swipe` | 滑动手势 | 起止坐标 |
| `scroll_to_element` | 滚动查找元素 | `by`, `value` |
| `wait_for_element` | 等待元素出现 | `by`, `value`, `timeout_ms?` |

### 应用管理
| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `launch_app` | 启动应用 | `package_name` |
| `install_apk` | 安装 APK | `apk_path` |
| `get_current_activity` | 获取前台 Activity | 无 |
| `pull_file` | 拉取设备文件 | `remote_path`, `local_path` |
| `adb_shell` | 执行任意 ADB 命令 | `command` |

### 诊断工具
| 工具 | 说明 | 关键参数 |
|------|------|----------|
| `get_logs` | 获取日志 | `package_name?`, `level?`, `lines?`, `since?` |
| `clear_logs` | 清空日志 | 无 |
| `get_device_info` | 获取设备信息 | 无 |

---

## 5. 典型使用场景

### 场景 1：Bug 复现与文档化

对 AI 说：
> "清空日志，打开设置页面，点击保存按钮，然后给我看日志和截图"

AI 会自动执行：
```
clear_logs → launch_app("com.example.app")
→ tap_element(by="text", value="Save")
→ get_logs(package_name="com.example.app", level="E")
→ screenshot(save_path="./bugs/settings-crash.png")
```

### 场景 2：UI 自动化测试

对 AI 说：
> "走一遍登录流程，验证每个页面的 UI 元素是否正确"

AI 会使用 `screenshot` + `get_ui_tree` 查看每个页面，`tap_element` / `type_text` 进行交互。

### 场景 3：冒烟测试

对 AI 说：
> "安装这个 APK，启动应用，点击所有主要页面，检查有没有崩溃"

AI 会执行：
```
install_apk("./app.apk") → launch_app("com.example.app")
→ [逐页 tap_element + get_logs(level="E")]
```

### 场景 4：复合操作

对 AI 说：
> "打开设置，搜索'显示'，点击第一个结果，然后返回"

AI 可以使用 `tap_sequence` 一次完成多步操作，或逐步执行：
```
launch_app("com.android.settings")
→ tap_and_wait(by="text", value="Search settings")
→ type_text("display")
→ tap_and_wait(by="text", value="Display")
→ press_key(key="back")
```

---

## 6. 元素定位策略

`tap_element` / `tap_and_wait` / `scroll_to_element` / `wait_for_element` 支持三种定位方式：

| 方式 | 匹配规则 | 示例 |
|------|----------|------|
| `resource-id` | 精确匹配或短名匹配（`search_bar` 匹配 `com.example:id/search_bar`） | `by="resource-id", value="search_bar"` |
| `text` | 精确匹配或大小写不敏感子串匹配 | `by="text", value="Settings"` |
| `content-desc` | 精确匹配或大小写不敏感子串匹配 | `by="content-desc", value="Menu"` |

**技巧**：先用 `get_ui_tree` 查看当前页面所有元素，找到目标元素的 id/text/desc 后再操作。

---

## 7. 支持的按键列表

`press_key` 工具支持以下按键：

| 键名 | 说明 | 键名 | 说明 |
|------|------|------|------|
| `back` | 返回 | `home` | 主页 |
| `enter` | 回车 | `tab` | Tab |
| `delete` | 删除 | `menu` | 菜单 |
| `recent_apps` | 最近应用 | `power` | 电源 |
| `volume_up` | 音量+ | `volume_down` | 音量- |
| `search` | 搜索 | `dpad_center` | 方向键确认 |
| `dpad_up` | 方向键上 | `dpad_down` | 方向键下 |
| `dpad_left` | 方向键左 | `dpad_right` | 方向键右 |

---

## 8. 常见问题

### Q: ADB 找不到设备

```bash
# 检查 USB 调试是否开启
adb devices
# 如果列表为空：
# 1. 确认设备已开启 USB 调试
# 2. 检查 USB 连接
# 3. 尝试 adb kill-server && adb start-server
```

### Q: ANDROID_HOME 未设置

在 MCP 配置的 `env` 字段中显式指定 `ANDROID_HOME`：
```json
"env": {
  "ANDROID_HOME": "/Users/username/Library/Android/sdk"
}
```

### Q: 截图返回空或报错

- 确保设备屏幕已解锁
- 检查 ADB 权限：`adb shell screencap -p /sdcard/test.png`

### Q: UI 树为空

- 某些页面（如视频播放器、游戏）使用自绘 UI，`uiautomator` 无法捕获
- 页面切换动画期间 UI 树不稳定，使用 `tap_and_wait` 等待 UI 稳定

### Q: 中文输入不支持

android-mcp-server 使用 `adb shell input text` 输入文本，不支持中文。如需中文输入，可以考虑：
- 通过 `adb_shell` 工具使用 ADB Keyboard 广播方式输入
- 使用支持中文的 Open-AutoGLM

### Q: 多设备怎么指定

所有工具都有 `device_id` 可选参数。先用 `list_devices` 查看设备 ID，然后在每次调用时传入 `device_id`。

---

## 9. 方案选型：何时选择 android-mcp-server

| 你的情况 | 推荐方案 | 原因 |
|----------|----------|------|
| 纯 Android 项目，想快速接入 | **android-mcp-server** | 一行 npx 启动，零配置 |
| 需要同时控制 iOS 设备 | **mobile-mcp** | 支持 Android + iOS |
| 团队已有 Appium 基础设施 | **appium-mcp** | 复用已有自动化框架 |
| 需要自主 Agent（非 MCP） | **Open-AutoGLM** | 内置 VLM 模型驱动 |
| 需要中文文本输入 | **Open-AutoGLM** | ADB Keyboard 方案支持中文 |
| 需要最多交互工具 | **android-mcp-server** | 25 个工具，粒度最细 |
| 需要复合操作减少延迟 | **android-mcp-server** | `tap_sequence` / `tap_and_wait` |

---

## 10. 参考资料

- GitHub 仓库: https://github.com/martingeidobler/android-mcp-server
- npm 包: https://www.npmjs.com/package/android-mcp-server
- MCP 协议: https://modelcontextprotocol.io
- 更多演示: [DEMOS.md](https://github.com/martingeidobler/android-mcp-server/blob/main/DEMOS.md)
- 提示词指南: [PROMPTING.md](https://github.com/martingeidobler/android-mcp-server/blob/main/PROMPTING.md)
