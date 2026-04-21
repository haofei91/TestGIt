# AutoJs6 快速上手指南

> 仓库：[SuperMonster003/AutoJs6](https://github.com/SuperMonster003/AutoJs6)
>
> 定位：Android 平台支持无障碍服务的 JavaScript 自动化工具
>
> 编写时间：2026-04-20

---

## 一、环境要求

### 1.1 运行环境（使用 AutoJs6 App）

| 要求 | 最低版本 | 推荐版本 |
|------|---------|---------|
| Android 系统 | 7.0 (API 24) | 10.0+ (API 29+) |
| 存储空间 | ~100MB | 200MB+ |
| 权限 | 无障碍服务（必须） | + 悬浮窗 + 通知权限 |

可选权限（按需开启）：
- **Root 权限**：解锁坐标点击/滑动、Shell 命令等高级功能
- **Shizuku**：无需 Root 即可获取 ADB 级别权限
- **屏幕截图权限**：`captureScreen()` / 找色 / 找图功能需要 `MediaProjection`
- **悬浮窗权限**：浮动按钮、`floaty` 浮动窗口需要 `SYSTEM_ALERT_WINDOW`

### 1.2 开发环境（编译源码）

| 要求 | 版本 |
|------|------|
| Android Studio | 2023.3+ (Jellyfish+) |
| IntelliJ IDEA | 2023.3+ |
| JDK | 17+ |
| Gradle | 项目内置 Wrapper（无需手动安装） |
| NDK | 项目 `local.properties` 或自动下载 |

### 1.3 远程开发环境（VSCode）

| 要求 | 说明 |
|------|------|
| VSCode | 任意较新版本 |
| 插件 | [AutoJs6-VSCode-Extension](http://vscext-project.autojs6.com) |
| 连接方式 | LAN（手机和电脑同一网络）或 ADB |

---

## 二、安装步骤

### 2.1 方式一：直接安装 APK（推荐）

1. 前往 [GitHub Releases](http://download.autojs6.com) 下载最新 APK
2. 在 Android 设备上安装 APK（需允许"未知来源"安装）
3. 打开 AutoJs6 App

### 2.2 方式二：从源码编译

```bash
# 1. 克隆仓库
git clone https://github.com/SuperMonster003/AutoJs6.git
cd AutoJs6

# 2. 使用 Android Studio 打开项目（推荐）
#    或命令行编译：
./gradlew assembleDebug

# 3. 编译产物路径
# app/build/outputs/apk/debug/app-debug.apk
```

> **注意**：首次编译需要下载大量依赖，耐心等待。项目使用 Gradle Wrapper，无需手动安装 Gradle。

---

## 三、配置说明

### 3.1 首次启动必须配置

**开启无障碍服务（核心，必须）：**

1. 打开 AutoJs6 → 主页 → 点击提示 "无障碍服务未开启"
2. 跳转到系统设置 → 无障碍 → 找到 AutoJs6 → 开启
3. 返回 AutoJs6，确认无障碍服务状态为"已开启"

> 部分系统（如 MIUI/HyperOS）会在后台杀进程或定期关闭无障碍服务，需要在电池优化中将 AutoJs6 设为"不限制"。

### 3.2 可选配置

| 配置项 | 操作路径 | 说明 |
|--------|---------|------|
| 悬浮窗权限 | 系统设置 → 应用 → AutoJs6 → 悬浮窗 | 启用浮动按钮和 `floaty` 模块 |
| 通知权限 | 系统设置 → 通知 → AutoJs6 | 前台服务保活 + `notice()` API |
| 电池优化 | 系统设置 → 电池 → AutoJs6 → 不限制 | 防止后台被杀 |
| Root 权限 | 使用 Magisk/KernelSU 授予 | 启用 `RootAutomator` + Shell |
| Shizuku | 安装 [Shizuku App](https://shizuku.rikka.app/) → 启动 → 授权 | 无 Root 获取 ADB 权限 |
| 屏幕截图 | 运行脚本时弹出授权对话框 | `captureScreen()`、找色找图 |

### 3.3 VSCode 连接配置

1. VSCode 安装 AutoJs6-VSCode-Extension 插件
2. AutoJs6 App → 侧边栏 → 连接电脑
3. 输入电脑 IP 地址（手机和电脑需在同一局域网）
4. 连接成功后，VSCode 中编写脚本 → `Ctrl+Shift+F5` 推送到手机运行

---

## 四、最小可运行示例

### 4.1 Hello World

```javascript
// 最简脚本：弹出 Toast
toast("Hello AutoJs6!");
```

**运行方式**：AutoJs6 App → 右下角 "+" → 新建文件 → 粘贴代码 → 点击运行按钮（三角形）

### 4.2 无障碍自动化 - 打开设置

```javascript
"auto";  // 声明需要无障碍服务（首次运行会提示开启）

// 打开系统设置
app.launchApp("设置");
sleep(2000);

// 查找并点击 "WLAN" 或 "Wi-Fi"
textMatches(/WLAN|Wi-Fi|无线局域网/).findOne(5000).click();
sleep(1000);

toast("已打开 WLAN 设置");
```

### 4.3 UI 选择器基础

```javascript
"auto";

// 按文本查找并点击
text("确定").findOne(3000).click();

// 按 resource-id 查找
id("com.example:id/search_btn").findOne(3000).click();

// 按 content-description 查找
desc("搜索").findOne(3000).click();

// 组合条件
className("android.widget.Button").text("提交").findOne(3000).click();

// 等待元素出现
text("加载完成").waitFor();
toast("页面加载完成");
```

### 4.4 找色 / 截图

```javascript
// 请求截图权限（首次会弹出授权框）
requestScreenCapture();
sleep(1000);

// 截图
let img = captureScreen();
toast("截图尺寸: " + img.getWidth() + "x" + img.getHeight());

// 找色（在指定区域找红色像素）
let point = findColor(img, "#FF0000", {
    region: [0, 0, img.getWidth(), img.getHeight()],
    threshold: 10
});

if (point) {
    toast("找到红色点: (" + point.x + ", " + point.y + ")");
} else {
    toast("未找到红色点");
}
```

### 4.5 浮动窗口

```javascript
// 创建浮动窗口
let window = floaty.window(
    <frame gravity="center">
        <text id="text" textSize="16sp" textColor="#FF0000">浮动文字</text>
    </frame>
);

// 3 秒后关闭
setTimeout(() => {
    window.close();
    toast("浮动窗口已关闭");
}, 3000);
```

### 4.6 完整示例 - 自动签到流程

```javascript
"auto";

// 1. 启动目标 App
app.launchApp("目标应用名");
sleep(3000);

// 2. 等待首页加载
text("首页").waitFor();
log("首页已加载");

// 3. 点击"我的"Tab
text("我的").findOne(5000).click();
sleep(1500);

// 4. 点击签到按钮
let signBtn = text("签到").findOne(5000);
if (signBtn) {
    signBtn.click();
    sleep(2000);
    toast("签到完成!");
} else {
    toast("未找到签到按钮");
}

// 5. 返回桌面
home();
```

---

## 五、常见问题

### Q1: 无障碍服务经常自动关闭

**原因**：部分 Android 定制系统（MIUI/EMUI/ColorOS 等）会"优化"掉后台服务。

**解决**：
1. 电池优化 → AutoJs6 → 设为"不限制"
2. 最近任务 → AutoJs6 → 锁定（长按卡片或点击锁图标）
3. 自启动管理 → 允许 AutoJs6 自启动
4. 如仍不行，可在 AutoJs6 设置中开启"前台服务"

### Q2: `text("xxx").findOne()` 一直卡住不返回

**原因**：`findOne()` 无参数时会无限等待，直到找到匹配元素。

**解决**：使用带超时的版本 `findOne(5000)`（5秒超时），或用 `find()` 返回列表（可能为空列表）。

### Q3: `click()` 执行了但没效果

**可能原因**：
1. 目标控件的 `clickable` 属性为 `false` → 尝试 `click(x, y)` 坐标点击
2. 需要更高权限 → 开启 Root 或 Shizuku
3. 元素被遮挡 → 先关闭弹窗/广告

```javascript
// 方案 A: 坐标点击
let node = text("目标").findOne(3000);
if (node) {
    let bounds = node.bounds();
    click(bounds.centerX(), bounds.centerY());
}

// 方案 B: Root 点击（需 Root 权限）
Tap(500, 800);
```

### Q4: `captureScreen()` 返回 null

**原因**：未先调用 `requestScreenCapture()` 或用户拒绝了授权。

**解决**：
```javascript
if (!requestScreenCapture()) {
    toast("请授权截图权限");
    exit();
}
sleep(1000);
let img = captureScreen();
```

### Q5: 脚本运行中如何停止？

- **方法 1**：下拉通知栏 → 点击 AutoJs6 通知中的"停止"
- **方法 2**：浮动按钮 → 点击"停止所有脚本"
- **方法 3**：音量上键长按（需在设置中启用"音量上键停止脚本"）

### Q6: 如何打包脚本为独立 APK？

1. AutoJs6 App → 文件管理器 → 找到你的脚本文件
2. 长按文件 → 菜单 → "打包" (或 "构建 APK")
3. 配置包名、版本号、图标等
4. 点击"构建" → 等待完成
5. 安装生成的 APK → 启动即自动运行脚本

### Q7: 源码编译报错

常见问题及解决：
- **JDK 版本不对**：确保使用 JDK 17+
- **Gradle 下载失败**：检查网络/代理设置，项目使用 Wrapper 自动下载
- **NDK 缺失**：Android Studio → SDK Manager → SDK Tools → 安装 NDK
- **内存不足**：在 `gradle.properties` 中增加 `org.gradle.jvmargs=-Xmx4g`

---

## 六、下一步建议先读哪些源码

按照"由外到内、由简到深"的顺序，推荐以下阅读路径：

### 第一阶段：理解引擎启动流程

| 顺序 | 文件 | 行数 | 说明 |
|------|------|------|------|
| 1 | `app/src/main/assets/init.js` | ~60 | 引擎启动时第一个执行的 JS 文件，理解模块加载机制 |
| 2 | `app/.../engine/RhinoJavaScriptEngine.kt` | 243 | Rhino 引擎封装，理解 JS Context 和 scope 的创建 |
| 3 | `app/.../runtime/ScriptRuntime.kt` | ~500+ | **核心桥接层**，所有 ~50 个 JS API 在这里创建和绑定 |

### 第二阶段：理解无障碍核心

| 顺序 | 文件 | 行数 | 说明 |
|------|------|------|------|
| 4 | `app/.../core/accessibility/AccessibilityService.kt` | 317 | 无障碍服务入口，事件分发机制 |
| 5 | `app/.../core/accessibility/SimpleActionAutomator.kt` | ~300+ | click/scroll/gesture 等操作的实际实现 |
| 6 | `app/.../AbstractAutoJs.kt` | 147 | 全局初始化，理解委托链注册和引擎创建 |

### 第三阶段：理解 APK 打包

| 顺序 | 文件 | 行数 | 说明 |
|------|------|------|------|
| 7 | `app/.../apkbuilder/ApkBuilder.kt` | ~200+ | APK 打包全流程：解压模板 → 注入脚本 → 签名 |
| 8 | `app/.../inrt/launch/AssetsProjectLauncher.kt` | 142 | 打包后 APK 的脚本解密和启动逻辑 |
| 9 | `app/.../inrt/autojs/AutoJs.kt` | ~50 | 打包模式的初始化差异（对比主 `AutoJs.kt`） |

### 第四阶段：深入 API 模块

| 顺序 | 目录/文件 | 说明 |
|------|----------|------|
| 10 | `app/.../runtime/api/augment/` | 所有 JS API 的增强层，每个子目录 = 一个 JS 全局对象 |
| 11 | `app/.../core/image/` | 截图/找色/OpenCV 图像处理 |
| 12 | `app/src/main/assets/modules/` | 内置 JS 模块（axios/lodash/dayjs 等） |

### 阅读技巧

1. **从示例脚本入手**：`app/src/main/assets-app/sample/` 目录有 28 个分类示例，先看你感兴趣的场景
2. **搜索 `@ScriptVariable`**：快速定位所有暴露给 JS 的 API 字段
3. **搜索 `@ScriptInterface`**：找到所有暴露给 JS 调用的方法
4. **搜索 `BuildConfig.isInrt`**：理解 IDE 模式 vs 打包模式的分支逻辑
5. **对照 [官方文档](https://docs.autojs6.com)**：文档中的 API 说明与源码一一对应

---

> **文档说明**：本指南基于 AutoJs6 v6.7.0 (2026/03/14) 版本编写，实际操作以最新版本为准。详细 API 文档请参阅 [docs.autojs6.com](https://docs.autojs6.com)。
