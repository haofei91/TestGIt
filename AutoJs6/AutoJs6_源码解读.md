# AutoJs6 源码解读

> 仓库：[SuperMonster003/AutoJs6](https://github.com/SuperMonster003/AutoJs6)
>
> 定位：Android 平台支持无障碍服务的 JavaScript 自动化工具
>
> 分析时间：2026-04-20

---

## 一、项目概述

### 1.1 项目定位

AutoJs6 是 Auto.js 的延续维护版本，核心理念是**在 Android 设备上运行 JavaScript 脚本来实现 UI 自动化**。与 mobile-mcp、droidrun 等方案不同，AutoJs6 不是一个 MCP 工具层或 LLM Agent 框架，而是一个**完整的 Android 应用（APK）**，内嵌 Rhino JS 引擎，通过无障碍服务（AccessibilityService）实现 UI 操控。

### 1.2 仓库量化统计

| 指标 | 数值 |
|------|------|
| 总文件数 | 4928 |
| Java 文件 | 1081 |
| Kotlin 文件 | 1057 |
| JavaScript 文件 | 194 |
| XML 文件（资源/布局） | 771 |
| C/C++ 文件 | 63 |
| SO 原生库 | 8 |
| Gradle 构建文件 | 33 |
| Markdown 文档 | 39 |
| 主要语言 | Java/Kotlin（核心）+ JavaScript（运行时模块/用户脚本）+ C/C++（JNI 原生） |

### 1.3 一级目录分布

| 目录 | 文件数 | 功能 |
|------|--------|------|
| `app/` | 4046 | **核心应用代码**，包含全部 Java/Kotlin 源码、JS 运行时模块、资源文件 |
| `modules/` | 580 | 独立功能模块（颜色选择器、日期选择器、APK 解析、分词等） |
| `libs/` | 129 | 第三方库依赖（OpenCV、终端模拟器、图片量化、Markdown 渲染等） |
| `plugin-api/` | 29 | Paddle OCR 插件 API 接口定义 |
| `build-logic/` | 24 | Gradle 构建约定插件（版本码处理等） |
| `gradle/` | 27 | Gradle Wrapper 和数据文件 |

### 1.4 核心特性

- JavaScript IDE（代码补全、变量重命名、代码格式化）
- 基于无障碍服务的 UI 自动化（选择器 API，类似 UiAutomator）
- 浮动按钮快捷操作（脚本录制/运行/布局分析）
- 屏幕截图/找色/图像匹配（OpenCV）
- E4X 编写 UI 界面
- **脚本打包为独立 APK**
- Root 权限扩展（屏幕点击/滑动/录制/Shell）
- Shizuku ADB 特权
- Tasker 插件集成
- VSCode 连接（AutoJs6-VSCode-Extension）
- WebSocket 支持
- OCR（MLKit / RapidOCR / PaddleOCR）

### 1.5 两大核心能力的定位

AutoJs6 的能力可以归纳为两大类，各自用途不同：

**能力一：UI 模块 — 用 JS 创建原生 Android 界面（面向"脚本使用者"）**

通过 E4X 语法在 JS 中声明 Android View 布局，创建脚本自己的操作界面：

```javascript
"ui";
ui.layout(
    <vertical padding="16">
        <text text="自动化控制台" textSize="22sp"/>
        <button id="start" text="开始运行"/>
        <button id="stop" text="停止"/>
    </vertical>
);
ui.start.on("click", () => { /* 启动自动化流程 */ });
```

用途：为自动化脚本提供**人机交互界面**。典型场景：
- 脚本的配置面板（让用户输入参数、选择选项）
- 运行状态展示（日志、进度条）
- 启动/停止控制按钮
- 多脚本的入口菜单

本质上是脚本的"前端"，让非技术用户也能通过界面操控自动化流程，而不是直接改代码。打包为独立 APK 后，这个界面就是用户看到的 App 主界面。

**能力二：Automator 模块 — 触发点击/滑动/手势（面向"被操控的目标 App"）**

通过无障碍服务/Root/Shizuku 操控设备上的其他应用：

```javascript
click("登录");                         // 点击文本为"登录"的控件
swipe(500, 1800, 500, 200, 500);       // 上滑手势
setText(0, "hello");                    // 设置输入框文本
```

用途：**代替人手去操作手机上的其他应用**。典型场景：
- 自动签到/打卡
- 批量操作（群发消息、批量点赞）
- 自动化测试
- 无人值守的定时任务

**两者的协作关系：**

```
┌─────────────────────────────────────┐
│  UI 模块（造界面）                    │  ← 给"脚本使用者"看的
│  用户点击"开始签到"按钮               │
└──────────────┬──────────────────────┘
               │ 触发
               ▼
┌─────────────────────────────────────┐
│  Automator 模块（操控其他 App）       │  ← 对"目标 App"动手的
│  打开微信 → 找到签到入口 → 点击       │
└─────────────────────────────────────┘
```

UI 模块不是必须的——纯后台脚本不需要界面。但有了它，自动化脚本就能变成一个"有界面的小工具"分发给不懂代码的人使用（尤其打包成独立 APK 后，UI 模块构建的界面 = 用户看到的 App）。

---

## 二、整体架构

### 2.1 分层架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户 JS 脚本层                                │
│   用户编写的 .js 文件 (如 click("登录"), text("搜索").findOne())      │
├─────────────────────────────────────────────────────────────────────┤
│                     JS 运行时 API 层 (augment/)                      │
│   ~50 个模块: Automator, Selector, Images, Device, Http, Files...    │
│   每个模块 = Java/Kotlin 实现 + @ScriptInterface 注解暴露给 JS       │
├─────────────────────────────────────────────────────────────────────┤
│                      ScriptRuntime (桥接层)                          │
│   创建所有 API 实例 → 绑定到 JS global scope → @ScriptVariable       │
├─────────────────────────────────────────────────────────────────────┤
│                    Rhino JS Engine 层                                │
│   RhinoJavaScriptEngine → Context/TopLevelScope → require/module    │
│   init.js 初始化 → 加载 assets/modules/ 下的 JS 模块                 │
├─────────────────────────────────────────────────────────────────────┤
│                   ScriptEngineService (引擎管理层)                    │
│   线程管理 → 脚本执行生命周期 → UI/后台模式分发                       │
├─────────────────────────────────────────────────────────────────────┤
│                     核心能力层 (core/)                                │
│   accessibility/ → 无障碍服务 + UiSelector + SimpleActionAutomator   │
│   automator/     → GlobalActionAutomator + ActionFactory             │
│   image/         → 截图/找色/OpenCV 图像处理                         │
│   inputevent/    → Root/Shizuku 输入事件注入                         │
│   floaty/        → 浮动窗口管理                                      │
│   ui/            → E4X 界面渲染                                      │
│   console/       → 控制台输出                                        │
│   eventloop/     → 事件循环                                          │
│   http/web/      → 网络请求/WebSocket                                │
├─────────────────────────────────────────────────────────────────────┤
│                     外部接口层 (external/)                            │
│   Tasker 插件 → Intent → 快捷方式 → Widget → BroadcastReceiver      │
│   VSCode Extension → WebSocket/TCP 连接                              │
├─────────────────────────────────────────────────────────────────────┤
│                     Android 系统层                                    │
│   AccessibilityService → MediaProjection → Root Shell → Shizuku     │
│   WindowManager → PackageManager → NotificationManager               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `AbstractAutoJs.kt` | 147 | 应用全局初始化：注册引擎、创建 Runtime、配置无障碍委托 |
| `ScriptRuntime.kt` | ~500+ | **核心桥接**：创建全部 ~50 个 API 实例，绑定到 JS global scope |
| `RhinoJavaScriptEngine.kt` | 243 | Rhino 引擎封装：Context 创建、init.js 执行、require 系统 |
| `ScriptEngineService.java` | 300 | 引擎生命周期管理：execute/stop/event 分发 |
| `AccessibilityService.kt` | 317 | 无障碍服务：事件分发、委托链、根节点获取 |
| `SimpleActionAutomator.kt` | ~300+ | 基于无障碍的操作：click/longClick/scroll/gesture/screenshot |
| `UiSelector.kt` | 54728 字节 | UI 选择器：text/id/desc/className 等多维查找 |
| `ApkBuilder.kt` | ~200+ | 脚本打包 APK：解压模板 → 注入脚本 → 修改 Manifest → 签名 |
| `AssetsProjectLauncher.kt` | 142 | 打包后的 APK 启动器：解密 → 加载 → 运行脚本 |

---

## 三、为什么需要一个 APK（核心设计决策分析）

### 3.1 Android 无障碍服务的系统约束

**这是 AutoJs6 必须作为 APK 存在的根本原因。**

Android 的 `AccessibilityService` 有严格约束：
1. 必须在 `AndroidManifest.xml` 中声明为 `<service>` 组件
2. 必须由用户在"系统设置 → 无障碍"中手动启用
3. 只能由安装在设备上的 APK 提供
4. 运行在应用进程内，拥有系统级 UI 树访问权限

ADB 命令（如 `input tap`、`uiautomator dump`）从 PC 端控制设备，但：
- 延迟高（每次新进程）
- 无法监听 UI 事件
- 无法获取实时 UI 树变化通知
- 无法执行 `AccessibilityService` 特有的 GlobalAction（如手势注入）

**AutoJs6 通过 APK 内嵌无障碍服务，获得了：**
- 实时 UI 事件回调（`onAccessibilityEvent`）
- 实时获取 `rootInActiveWindow`（UI 树根节点）
- 原生 `performAction`（click/scroll/setText 等）
- `dispatchGesture` API（精确手势路径注入）
- `GLOBAL_ACTION_*`（home/back/recents/lockScreen/takeScreenshot）

### 3.2 APK 的代码布局

```
app/src/main/
├── java/org/autojs/autojs/
│   ├── AbstractAutoJs.kt          # 全局初始化入口
│   ├── AutoJs.kt                  # 具体初始化（非 inrt 版本）
│   ├── core/                      # 核心能力实现
│   │   ├── accessibility/         # 无障碍服务 + UiSelector + ActionAutomator
│   │   ├── automator/             # 操作工厂 + GlobalActionAutomator
│   │   ├── image/                 # 截图/找色/OpenCV
│   │   ├── inputevent/            # Root 输入事件注入
│   │   ├── floaty/                # 浮动窗口
│   │   ├── console/               # 控制台
│   │   ├── ui/                    # E4X 界面
│   │   ├── http/web/              # 网络
│   │   └── ...25 个子目录
│   ├── engine/                    # 脚本引擎
│   │   ├── RhinoJavaScriptEngine.kt   # Rhino 封装
│   │   ├── ScriptEngineService.java   # 引擎生命周期
│   │   ├── ScriptEngineManager.java   # 引擎工厂注册
│   │   └── module/                # CommonJS require 实现
│   ├── runtime/                   # JS 运行时
│   │   ├── ScriptRuntime.kt      # 桥接层（~50 个 API 绑定）
│   │   ├── api/                   # API 实现层
│   │   │   ├── Images.java        # 图像处理
│   │   │   ├── Shell.java         # Shell 执行
│   │   │   ├── Ocr.kt             # OCR
│   │   │   └── augment/           # ~50 个 JS API 模块
│   │   │       ├── automator/     # auto(), click(), swipe()
│   │   │       ├── selector/      # text(), id(), desc()
│   │   │       ├── images/        # captureScreen(), findColor()
│   │   │       ├── files/         # read(), write(), open()
│   │   │       └── ...
│   │   └── exception/             # 脚本异常体系
│   ├── apkbuilder/                # APK 打包器
│   │   ├── ApkBuilder.kt         # 打包流程
│   │   ├── ApkPackager.java       # ZIP 操作
│   │   ├── ManifestEditor.java    # Manifest 修改
│   │   └── TinySign.java          # APK 签名
│   ├── inrt/                      # 打包后运行时（In-Runtime）
│   │   ├── autojs/AutoJs.kt      # 打包版初始化
│   │   ├── launch/                # 脚本启动器
│   │   └── SplashActivity.kt     # 打包版启动页
│   ├── external/                  # 外部接口
│   │   ├── tasker/                # Tasker 插件
│   │   ├── shortcut/              # 快捷方式
│   │   └── widget/                # 桌面小组件
│   ├── service/                   # Android 服务
│   │   ├── AccessibilityService.kt  # 无障碍服务注册
│   │   ├── ForegroundService.kt   # 前台服务
│   │   └── NotificationService.kt  # 通知监听
│   ├── ui/                        # 应用 UI
│   │   ├── main/                  # 主界面
│   │   ├── edit/                  # 代码编辑器
│   │   ├── floating/              # 浮动工具栏
│   │   └── settings/              # 设置页
│   └── rhino/                     # Rhino 定制
│       ├── AndroidContextFactory.kt
│       ├── TopLevelScope.kt
│       └── extension/             # Rhino 扩展
├── assets/
│   ├── init.js                    # 引擎初始化脚本
│   └── modules/                   # 内置 JS 模块（promise/axios/lodash/dayjs...）
├── assets-app/
│   ├── sample/                    # 28 个分类示例脚本
│   ├── doc/                       # 内置文档
│   └── editor/                    # 代码编辑器资源
├── assets-inrt/                   # 打包版专用资源
├── jniLibs/                       # 原生库（4 架构：arm64-v8a/armeabi-v7a/x86/x86_64）
└── res/                           # Android 资源（10 种语言适配）
```

### 3.3 APK 的双重身份设计

AutoJs6 的 APK 有两种构建模式（通过 `BuildConfig.isInrt` 控制）：

**模式 1：开发环境 APK（AutoJs6 App）**
- 包含完整 IDE（代码编辑器 + 文件管理 + 控制台）
- 可以新建/编辑/运行 JS 脚本
- 可以连接 VSCode 进行远程开发
- 可以将脚本打包为独立 APK

**模式 2：打包后 APK（inrt 模式）**
- 用户脚本加密嵌入到 `assets-inrt/project/` 中
- `AssetsProjectLauncher` 在启动时解密并执行脚本
- 不包含 IDE 和开发工具
- 使用 `LoopBasedJavaScriptEngineWithDecryption` 解密脚本
- `ApkBuilder` 负责此打包过程：解压模板 APK → 注入脚本 → 修改包名/图标/版本 → 签名

```
AbstractAutoJs.kt:61-64
when (isInrt) {
    true -> LoopBasedJavaScriptEngineWithDecryption(applicationContext)
    else -> LoopBasedJavaScriptEngine(applicationContext)
}
```

---

## 四、JS 能力如何提供给外部

### 4.1 JS → Java 桥接机制

AutoJs6 使用 **Mozilla Rhino** 引擎（v2.0.0-SNAPSHOT）作为 JS 运行时。JS 能力通过以下机制暴露：

**Step 1: ScriptRuntime 创建 API 实例**

```kotlin
// ScriptRuntime.kt:233-349
@ScriptVariable val app = builder.appUtils
@ScriptVariable val console: GlobalConsole = builder.getConsole()
@ScriptVariable val automator: SimpleActionAutomator
@ScriptVariable val images: ApiImages
@ScriptVariable val files = ApiFiles(this)
@ScriptVariable val http: ApiHttp
@ScriptVariable val device: ApiDevice
@ScriptVariable val ocr = ApiOcr()
// ... 约 50 个 @ScriptVariable 标注的字段
```

**Step 2: RhinoJavaScriptEngine.init() 将 API 绑定到 JS scope**

```kotlin
// RhinoJavaScriptEngine.kt:118-121
scriptable.defineProp("global", scriptable, PERMANENT)
scriptable.defineProp("__engine__", this, READONLY or DONTENUM or PERMANENT)
scriptable.defineProp("Promise", runtime.js_Promise, READONLY or PERMANENT)
```

`ScriptRuntime.initPrologue()` 和 `initEpilogue()` 通过 `@ScriptVariable` 注解自动扫描所有字段，将它们注入到 Rhino 的全局 scope 中。

**Step 3: augment/ 目录的 API 包装层**

`runtime/api/augment/` 下有 ~50 个子目录，每个对应一个 JS 全局对象或函数：

| augment 模块 | JS 全局名 | 核心能力 |
|-------------|-----------|---------|
| `automator/` | `click()`, `swipe()`, `scrollDown()` | 无障碍操作 |
| `selector/` | `text()`, `id()`, `desc()`, `className()` | UI 元素选择器 |
| `images/` | `captureScreen()`, `findColor()`, `findImage()` | 图像处理 |
| `files/` | `read()`, `write()`, `open()` | 文件操作 |
| `http/` | `http.get()`, `http.post()` | HTTP 请求 |
| `device/` | `device.width`, `device.height`, `device.brand` | 设备信息 |
| `shell/` | `shell()`, `Shell` | Shell 命令 |
| `dialogs/` | `dialogs.alert()`, `dialogs.input()` | 对话框 |
| `floaty/` | `floaty.window()`, `floaty.rawWindow()` | 浮动窗口 |
| `threads/` | `threads.start()` | 多线程 |
| `timers/` | `setTimeout()`, `setInterval()` | 定时器 |
| `events/` | `events.on()`, `keys.on()` | 事件监听 |
| `engines/` | `engines.execScript()` | 引擎管理 |
| `ocr/` | `ocr.detect()`, `ocr.recognize()` | OCR 识别 |
| `base64/` | `base64.encode()`, `base64.decode()` | Base64 |
| `crypto/` | `crypto.encrypt()`, `crypto.decrypt()` | 加密解密 |
| `sqlite/` | `sqlite.open()`, `sqlite.exec()` | SQLite 数据库 |
| `web/` | `web.newWebSocket()` | WebSocket |
| `shizuku/` | `shizuku.execCommand()` | Shizuku ADB 特权 |
| `console/` | `console.log()`, `console.error()` | 控制台 |
| `toast/` | `toast()`, `toastLog()` | Toast 提示 |
| `canvas/` | `canvas.bindTo()`, `canvas.bindWindow()` | 画布绘制 |
| `colors/` | `colors.toString()`, `colors.rgb()` | 颜色处理 |
| `app/` | `app.launch()`, `app.openUrl()` | 应用管理 |
| `ui/` | `ui.layout()`, `ui.inflate()` | UI 界面 |
| `notice/` | `notice()`, `notice.config()` | 通知 |
| `zip/` | `zip.open()`, `zip.compress()` | 压缩解压 |

### 4.2 JS 模块系统

AutoJs6 实现了 CommonJS 风格的 `require()` 系统：

```kotlin
// RhinoJavaScriptEngine.kt:179-188
private fun initRequireBuilder(context: Context, scope: Scriptable) {
    val provider = AssetAndUrlModuleSourceProvider(
        androidContext, MODULES_ROOT_PATH, listOf<URI>(File(File.separator).toURI())
    )
    RequireBuilder()
        .setModuleScriptProvider(SoftCachingModuleScriptProvider(provider))
        .setSandboxed(true)
        .createRequire(context, scope)
        .install(scope)
}
```

模块查找路径：
1. `assets/modules/` — 内置 JS 模块（axios、lodash、dayjs、promise 等）
2. 用户脚本的工作目录
3. URL 模块（通过 `UrlModuleSourceProvider`）

### 4.3 init.js 模块扩展机制

`init.js` 是引擎启动时最先执行的文件，提供灵活的模块扩展方式：

```javascript
// 字符串形式：引入并扩展到 global
"apple";

// 数组形式：批量引入
[ "apple", "orange", "banana" ];

// 管道符形式
"apple|orange|banana";

// 函数形式：自定义初始化
function(scriptRuntime, scope) {
    scope.myModule = require("my-module.js");
};

// 对象形式：名称重映射
{ myAppleObject: "apple.js" }
```

### 4.4 完整案例：`click("登录")` 从 JS 到 Android 的翻译全过程

以最常见的 `click("登录")` 为例，展示一次 JS 调用如何逐层翻译为 Android 原生 API 调用。

#### 4.4.1 注册阶段 — `click` 如何成为 JS 全局函数

引擎初始化时（`ScriptRuntime.kt:757`）：

```kotlin
Automator(this).augment(target, true)
```

`Automator` 类通过 `selfAssignmentFunctions` 声明要注册的函数：

```kotlin
// Automator.kt:46
override val selfAssignmentFunctions = listOf(
    ::click.name to AS_GLOBAL,   // AS_GLOBAL = 注册到 JS 全局 scope
    ::longClick.name to AS_GLOBAL,
    ::swipe.name to AS_GLOBAL,
    // ... 更多全局函数
)
```

`Augmentable` 基类的 `augmentFunctionsBy()` 方法为每个函数名创建一个 Rhino `BaseFunction`，并通过 **Java 反射** 桥接到 Kotlin 方法：

```kotlin
// Augmentable.kt:440-448
val f = newBaseFunction(funcName, { args ->
    javaClass
        .getMethod(funcName, ScriptRuntime::class.java, Array<Any>::class.java)
        .invoke(this, scriptRuntime, args)   // ← Java 反射调用
}, NOT_CONSTRUCTABLE)
destination.defineWith("click", f, PERMANENT)
// → JS 全局 scope 中 "click" = 这个 BaseFunction
```

**关键点**：Rhino 和 JVM 在同一进程，JS→Java 无需序列化/IPC，就是一次 `Method.invoke()`。

#### 4.4.2 调用阶段 — 参数分发

当 JS 执行 `click("登录")` 时，Rhino 调用该 BaseFunction，反射转发到：

```kotlin
// Automator.kt:92-128
fun click(scriptRuntime: ScriptRuntime, args: Array<out Any?>): Boolean {
    when (argList.size) {
        1 -> when (val o = argList[0]) {
            is Rect -> ...          // click(rect) — 坐标点击
            is UiObject -> ...      // click(widget) — 控件点击
            is AndroidPoint -> ...  // click(point) — 点点击
            // "登录" 是 String，不匹配以上类型
        }
        2 -> if (x.isJsNumber() && y.isJsNumber()) → ...  // click(x, y)
    }
    // 字符串参数 → 走文本过滤路径：
    return performAction(scriptRuntime, { target -> scriptRuntime.automator.click(target) }, argList)
}
```

#### 4.4.3 构造 Action — TextFilter + SearchUp 策略

```kotlin
// Automator.kt:652-659
performAction(scriptRuntime, action, arrayOf("登录", -1))
    → scriptRuntime.automator.text("登录", -1)     // 创建 TextActionTarget
        .let { action(it) }                        // 调用 automator.click(target)

// SimpleActionAutomator.kt:79
fun click(target: ActionTarget) = performAction(
    target.createAction(AccessibilityNodeInfoCompat.ACTION_CLICK)
)

// ActionTarget.kt:14
class TextActionTarget(text, index) : ActionTarget {
    override fun createAction(action, ...) = ActionFactory.createActionWithTextFilter(action, text, index)
}

// ActionFactory.kt:23-24
// 因为 ACTION_CLICK 在 searchUpActions 列表中：
→ SearchUpTargetAction(ACTION_CLICK, TextFilter("登录", -1))
```

#### 4.4.4 执行阶段 — 遍历 UI 树 + 调用 Android 原生 API

```kotlin
// SimpleActionAutomator.kt:227-235 — 获取 UI 树根节点
private fun performAction(simpleAction: SimpleAction): Boolean {
    accessibilityBridge.windowRoots()    // ← Android: AccessibilityService.getWindows()
        .map { root ->
            simpleAction.perform(UiObject.createRoot(root))
        }
}
```

分三步执行：

**Step A: TextFilter 文本搜索**
```kotlin
// FilterAction.kt:21-23
UiSelector().textContains("登录").findAndReturnList(root)
// ← 深度优先遍历 AccessibilityNodeInfo 树，比对每个节点的 getText()
// ← 返回所有文本包含"登录"的节点列表
```

**Step B: SearchUp 向上查找可点击祖先**
```kotlin
// SearchUpTargetAction.kt:12-18
override fun searchTarget(node: UiObject?): UiObject? {
    var temp = node
    while (temp != null && !temp.clickable()) {  // ← AccessibilityNodeInfo.isClickable()
        temp = temp.parent()                     // ← AccessibilityNodeInfo.getParent()
    }
    return temp  // 最近的 clickable=true 的祖先（最多向上 20 层）
}
```

> 设计原因：文本"登录"通常在 TextView 上，而 `clickable=true` 在其父级 Button/Layout 上。

**Step C: 执行点击**
```kotlin
// SearchTargetAction.kt:22
node.performAction(action)
// ← 最终调用: AccessibilityNodeInfo.performAction(AccessibilityNodeInfo.ACTION_CLICK)
```

#### 4.4.5 全链路总结

| 层级 | 代码位置 | 做什么 | Android 原生 API |
|------|---------|--------|------------------|
| JS 调用 | `click("登录")` | 触发 Rhino BaseFunction | — |
| 反射桥接 | `Augmentable.kt:447` | `Method.invoke()` → Kotlin 方法 | — |
| 参数分发 | `Automator.kt:127` | 识别为文本 → `performAction` | — |
| 构造 Action | `ActionFactory.kt:23` | `SearchUpTargetAction + TextFilter` | — |
| 获取 UI 树 | `SimpleActionAutomator.kt:232` | 获取窗口根节点 | `AccessibilityService.getWindows()` |
| 文本匹配 | `FilterAction.TextFilter.filter()` | 深度遍历，匹配文本 | 遍历 `AccessibilityNodeInfo`，比对 `getText()` |
| 向上搜索 | `SearchUpTargetAction.searchTarget()` | 找 clickable 祖先 | `AccessibilityNodeInfo.getParent()` + `isClickable()` |
| 执行点击 | `SearchTargetAction.performAction()` | 执行 ACTION_CLICK | **`AccessibilityNodeInfo.performAction(ACTION_CLICK)`** |

---

## 五、底层原子能力分析

### 5.1 无障碍服务能力（AccessibilityService）

| 原子能力 | Android API | AutoJs6 封装 |
|---------|-------------|-------------|
| UI 树遍历 | `getRootInActiveWindow()` | `UiSelector.kt` — text/id/desc/className 多维选择 |
| 节点操作 | `performAction(ACTION_CLICK)` | `SimpleActionAutomator.click/longClick/scrollUp/Down` |
| 文本设置 | `performAction(ACTION_SET_TEXT)` | `SimpleActionAutomator.setText/appendText` |
| 全局操作 | `performGlobalAction()` | `back()/home()/recents()/notifications()/lockScreen()` |
| 手势注入 | `dispatchGesture()` | `GlobalActionAutomator.gesture/gestures/swipe/click` |
| 截图 | `takeScreenshot()` (API 30+) | `SimpleActionAutomator.takeScreenshot()` |
| 按键监听 | `onKeyEvent()` | `OnKeyListener.Observer` |
| 窗口信息 | `AccessibilityEvent` | `ActivityInfoProvider` — 当前包名/活动名 |

**引用：**
```kotlin
// AccessibilityService.kt:86-121
override fun onAccessibilityEvent(event: AccessibilityEvent) {
    instance = this
    val type = event.eventType
    markOperationalStateIfNeeded()
    // ... 事件分发给注册的 delegate 和 callback
}
```

### 5.2 Root 权限能力

| 原子能力 | 实现方式 | AutoJs6 封装 |
|---------|---------|-------------|
| 坐标点击 | `/dev/input/eventX` 写入 | `RootAutomator.java` — 直接写入 input 设备文件 |
| 坐标滑动 | 多个 touch event 序列 | `RootAutomator.touchMove/touchDown/touchUp` |
| Shell 命令 | `su` 进程 | `Shell.java` — root/non-root 双模式 |
| 屏幕录制 | `MediaProjection` API | `core/record/` |

### 5.3 Shizuku ADB 特权

```kotlin
// WrappedShizuku.kt
// 通过 Shizuku 获取 ADB 级别权限，无需 Root
// 可执行 shell 命令、操控输入事件
```

### 5.4 图像处理能力

| 原子能力 | 底层技术 | JS API |
|---------|---------|--------|
| 屏幕截图 | `MediaProjection` + `ImageReader` | `captureScreen()` |
| 找色 | 像素遍历 + OpenCV | `findColor()`, `findMultiColors()` |
| 找图 | OpenCV 模板匹配 | `findImage()` |
| OCR | MLKit / RapidOCR / PaddleOCR | `ocr.detect()`, `ocr.recognize()` |
| 图片编辑 | `Bitmap` + OpenCV | `images.clip()`, `images.resize()` |

### 5.5 其他系统能力

| 能力 | Android API | JS 封装 |
|------|-------------|--------|
| 浮动窗口 | `WindowManager` + `SYSTEM_ALERT_WINDOW` | `floaty.window()` |
| 通知 | `NotificationManager` | `notice()` |
| 传感器 | `SensorManager` | `sensors.register()` |
| 剪贴板 | `ClipboardManager` | `clip` 属性 |
| 定时任务 | `AlarmManager` | `timing/` |
| 广播 | `BroadcastReceiver` | `events.onBroadcast()` |
| WebSocket | `okhttp3.WebSocket` | `web.newWebSocket()` |
| HTTP | `okhttp3` | `http.get/post/put/delete` |
| SQLite | `android.database.sqlite` | `sqlite.open()` |
| 文件 | `java.io.File` + `SAF` | `files.read/write/open` |

---

## 六、外部人员写 JS 如何跑自动化脚本

### 6.1 运行方式总览

```
                         ┌──────────────────────────┐
                         │    方式 1: APP 内直接运行    │
                         │    打开 AutoJs6 → 编辑器    │
                         │    → 编写 JS → 运行按钮     │
                         └──────────────────────────┘
                         ┌──────────────────────────┐
                         │    方式 2: VSCode 远程      │
                         │    VSCode + Extension      │
                         │    → 编辑 → Ctrl+Shift+F5  │
                         │    → 推送到手机执行          │
                         └──────────────────────────┘
                         ┌──────────────────────────┐
                         │    方式 3: 脚本文件运行      │
                         │    将 .js 文件推到手机        │
                         │    → AutoJs6 文件管理器      │
                         │    → 点击运行               │
                         └──────────────────────────┘
                         ┌──────────────────────────┐
                         │    方式 4: 打包为独立 APK    │
                         │    脚本 → ApkBuilder 打包    │
                         │    → 分发 APK → 安装即运行   │
                         └──────────────────────────┘
                         ┌──────────────────────────┐
                         │    方式 5: Tasker/快捷方式   │
                         │    Tasker 插件 / 桌面快捷    │
                         │    → Intent 触发脚本运行     │
                         └──────────────────────────┘
```

### 6.2 典型自动化脚本示例

```javascript
// 示例：自动打开微信搜索联系人
"auto";  // 声明使用无障碍服务

// 打开微信
app.launchApp("微信");
sleep(2000);

// 点击搜索按钮（通过 content-desc 定位）
desc("搜索").findOne().click();
sleep(1000);

// 输入搜索文本
setText(0, "张三");
sleep(500);

// 点击搜索结果
text("张三").findOne().click();
```

### 6.3 选择器 API（UiSelector）

AutoJs6 提供了非常丰富的选择器 API（`UiSelector.kt` 是全项目最大的单文件，54728 字节）：

```javascript
// 按文本查找
text("登录").findOne().click();

// 按 resource-id 查找
id("com.example:id/btn_login").findOne().click();

// 按 content-desc 查找
desc("搜索按钮").findOne().click();

// 按 className 查找
className("android.widget.EditText").findOne().setText("hello");

// 组合查找
text("确定").className("Button").findOne().click();

// 正则匹配
textMatches(/\d+\.\d+元/).find();

// 层级关系
id("parent_view").findOne().child(0).click();

// 等待元素出现
text("加载完成").waitFor();

// 遍历所有匹配元素
text("选项").find().forEach(item => item.click());
```

### 6.4 脚本执行流程

```
用户点击"运行" 或 VSCode 推送脚本
    ↓
ScriptEngineService.execute(task)
    ↓
ScriptExecutionTask → 判断执行模式
    ├── UI 模式 → ScriptExecuteActivity
    └── 后台模式 → 新线程 → LoopBasedJavaScriptExecution
        ↓
    ScriptEngineManager.createEngine()
        ↓
    RhinoJavaScriptEngine 创建
        ↓
    engine.setRuntime(ScriptRuntime)
        → runtime.bridges.setup(engine)
        → 绑定 topLevelScope
        ↓
    engine.init()
        → initRequireBuilder (设置 require 模块路径)
        → runtime.initPrologue() (注入所有 @ScriptVariable 到 scope)
        → 执行 init.js (加载内置 JS 模块)
        → runtime.initEpilogue()
        ↓
    engine.doExecution(source)
        → context.compileReader(reader, path, 1, null)
        → script.exec(context, scriptable, scriptable)
        ↓
    用户脚本开始执行
        → auto(); // 启用无障碍
        → text("xxx").findOne().click(); // 查找并点击
```

---

## 七、APK 打包机制详解

### 7.1 打包流程

```kotlin
// ApkBuilder.kt
ApkBuilder(apkInputStream, outApkFile, buildPath)
```

流程：
1. **解压模板 APK**：将 AutoJs6 自身的 APK 作为模板解压到 `buildPath`
2. **注入用户脚本**：将用户的 JS 脚本（加密后）放入 `assets-inrt/project/`
3. **修改 AndroidManifest.xml**：修改包名、版本号、权限等
4. **修改 resources.arsc**：修改应用名称、图标等
5. **重新打包**：压缩为新的 APK
6. **签名**：使用 `TinySign` 或用户自定义 keystore 签名

### 7.2 脚本加密

```kotlin
// AssetsProjectLauncher.kt:128-141
private fun initKey(projectConfig: ProjectConfig) {
    val key = MD5.md5(projectConfig.packageName + projectConfig.versionName + projectConfig.mainScriptFileName)
    val vec = MD5.md5(projectConfig.buildInfo.buildId + projectConfig.name).substring(0, 16)
    // 通过反射设置 ScriptEncryption 的 key 和 initVector
}
```

打包时脚本通过 AES 加密，运行时通过 `LoopBasedJavaScriptEngineWithDecryption` 解密后执行。

---

## 八、与其他移动端自动化方案的对比

### 8.1 定位差异

| 维度 | AutoJs6 | mobile-mcp | droidrun | appium-mcp |
|------|---------|------------|----------|------------|
| **本质** | Android App（内嵌 JS 引擎） | MCP 工具层（PC 控制） | Python Agent（PC 控制） | MCP 工具层（PC 控制） |
| **运行位置** | 设备端 | PC 端 | PC 端 | PC 端 |
| **通信方式** | 设备内直接调用 | ADB CLI | Portal TCP/ADB | Appium HTTP |
| **延迟** | 极低（进程内调用） | 中（ADB 进程） | 低-中（TCP/ADB） | 高（3 层架构） |
| **UI 操控** | 无障碍服务（原生） | ADB input（外部） | 无障碍+ADB（混合） | Appium UiAutomator |
| **编程语言** | JavaScript | Python/TS（用户） | Python（用户） | Python（用户） |
| **LLM 集成** | 无 | 被动工具层 | 内置 Agent | 被动工具层 |
| **独立运行** | ✅ 无需 PC | ❌ 需要 PC | ❌ 需要 PC | ❌ 需要 PC |
| **打包分发** | ✅ 可打包 APK | ❌ | ❌ | ❌ |

### 8.2 核心优劣势

**AutoJs6 的独特优势：**
1. **设备端独立运行**：无需 PC 连接，脚本在手机上直接执行
2. **极低延迟**：无障碍服务在进程内调用，无网络/ADB 开销
3. **UI 选择器最丰富**：`UiSelector.kt`（54KB）提供 text/id/desc/className/bounds 等多维组合查找
4. **脚本打包 APK**：用户可以把自动化脚本打包成独立 App 分发
5. **完整 IDE**：内置代码编辑器、控制台、文件管理
6. **Root + Shizuku + 无障碍**三种权限模式，灵活适配

**AutoJs6 的局限：**
1. **仅 Android**：不支持 iOS
2. **非 MCP 协议**：无法被 Cursor/Claude 等 AI 工具直接调用
3. **需要安装 APK**：对于 CI/CD 自动化测试场景不方便
4. **用户需要学 JavaScript**：不适合 LLM 驱动的纯自然语言自动化
5. **无障碍服务依赖**：部分设备（如 MIUI）可能限制无障碍

---

## 九、关键设计模式

### 9.1 委托链模式（AccessibilityService Delegate）

```kotlin
// AbstractAutoJs.kt:100-104
AccessibilityService.addDelegate(100, infoProvider)
AccessibilityService.addDelegate(200, notificationObserver)
AccessibilityService.addDelegate(300, accessibilityActionRecorder)
```

无障碍事件通过优先级排序的委托链分发，每个委托可以选择消费事件（返回 true）或传递给下一个。

### 9.2 Builder + 注解注入模式（ScriptRuntime）

`ScriptRuntime` 使用 Builder 模式构造，通过 `@ScriptVariable` 注解自动将 Java/Kotlin 字段注入到 JS 全局 scope，实现了 Java ↔ JS 的声明式绑定。

### 9.3 双模式构建（isInrt）

通过编译时常量 `BuildConfig.isInrt` 区分"开发 IDE"和"打包运行时"两种模式，共享 95% 以上的代码。

### 9.4 augment 模块化

每个 JS API（如 `images`、`files`、`http`）都有独立的 `augment/` 子包，形成"API 增强层"模式，解耦底层实现和 JS 接口。

---

## 十、值得借鉴的点

1. **@ScriptVariable 声明式绑定**：通过注解自动将 Java 字段暴露给 JS，减少手动绑定代码
2. **init.js 灵活模块扩展**：支持字符串/数组/对象/函数四种形式加载模块，对开发者友好
3. **UiSelector 的丰富查找策略**：text/textContains/textMatches/id/idContains/desc/className/bounds 等组合，远超 ADB uiautomator dump 的能力
4. **APK 打包闭环**：从脚本开发到 APK 分发的完整链路，解决了"自动化脚本如何给非技术用户使用"的问题
5. **三级权限降级**：Root → Shizuku → AccessibilityService，自动适配设备能力
6. **设备端独立运行**：不依赖 PC/ADB，适合"无人值守"场景（如定时任务、桌面小组件触发）
7. **28 个分类示例脚本**：覆盖 OCR/Shell/HTTP/控件/画布/传感器等全场景

---

## 十一、局限性

1. **仅 Android 平台**：无 iOS 支持
2. **非标准协议**：不支持 MCP/Appium WebDriver 等标准协议
3. **Rhino 引擎性能**：JavaScript 执行效率低于 V8（Chrome/Node.js 引擎）
4. **无障碍服务风险**：Google Play 政策限制，部分无障碍功能可能被拒审
5. **脚本加密弱**：AES 密钥由包名+版本名 MD5 生成，安全性有限
6. **单设备运行**：无设备池/远程控制能力
7. **调试能力有限**：无断点调试（仅 console.log 调试）

---

## 十二、开发指引导航

基于源码与 [官方文档](https://docs.autojs6.com) 的交叉分析，我们生成了一套完整的开发指引文档系统（位于 `开发指引/` 目录），覆盖全部 JS 接口的桥接代码位置、Android 实现源码位置、能力描述和参数描述。

### 12.1 JS-Java 桥接机制

AutoJs6 的 JS API 通过以下三步暴露给用户脚本：

**Augmentable 基类**（`runtime/api/augment/Augmentable.kt`）提供统一注入机制：

- `selfAssignmentProperties` — 注入静态属性（如 `isAutoJs6 = true`）
- `selfAssignmentFunctions` — 注入函数（如 `sleep()`, `exit()`, `click()`）
- `selfAssignmentGetters` — 注入 getter 属性（如 `WIDTH`, `HEIGHT`, `axios`）
- `assign()` 方法 — 将上述属性/函数写入 `ScriptableObject`（Rhino 全局对象）

**ScriptRuntime.augment() 注册**（`ScriptRuntime.kt` 第 737-820 行）是所有模块的注册入口：

```kotlin
// ScriptRuntime.kt:737-820
private fun augment(target: ScriptableObject) {
    Global(this).assignWithRuntime(target, this, GlobalClasses)
    App(this).augment(target, app, true)
    Automator(this).augment(target, true)
    Images(this).augment(target, true)
    // ... 共约 50 个模块
}
```

**引擎初始化流程**：

```
RhinoJavaScriptEngine.init()
  → runtime.initPrologue()     // 创建 threads/timers/events 等实例, 调用 augment()
  → 写入 global/Promise 等     // defineProp("global", scriptable)
  → 执行 init.js               // 加载内置 JS 模块
  → runtime.initEpilogue()     // Object.observe polyfill, Module/require 重定向
```

### 12.2 API 开发指引文档

按功能域分为 9 个子文档，每个子文档内部遵循「总述 → 源码定位表 → 逐个函数详解」的总分结构：

| 文档 | 覆盖模块 | 说明 |
|------|---------|------|
| [01-自动化操控](开发指引/01-自动化操控.md) | Global, Automator, Keys, UiSelector, UiObject | 核心自动化能力：全局函数、UI 操控、按键、选择器 |
| [02-应用与设备](开发指引/02-应用与设备.md) | AutoJs6, App, Device, Shell, Shizuku | 应用管理、设备信息、Shell 命令、Shizuku 特权 |
| [03-图像与视觉](开发指引/03-图像与视觉.md) | Image, Color, OCR, Barcode, QR Code, Canvas | 截图、找色、找图、OCR、条码、画布 |
| [04-文件与数据](开发指引/04-文件与数据.md) | Files, Storages, SQLite, Base64, Crypto | 文件操作、键值存储、数据库、编解码、加密 |
| [05-界面与交互](开发指引/05-界面与交互.md) | UI, Dialogs, Floaty, Toast, Notice, Console | 脚本 UI、对话框、悬浮窗、消息提示、控制台 |
| [06-网络与通信](开发指引/06-网络与通信.md) | HTTP, Web, WebSocket | HTTP 请求、WebView、WebSocket 通信 |
| [07-引擎与运行时](开发指引/07-引擎与运行时.md) | Engines, Tasks, Modules, Plugins, Threads, Timers, Continuation, Events, Sensors, Recorder, Media | 脚本引擎、定时任务、多线程、事件、传感器 |
| [08-工具与扩展](开发指引/08-工具与扩展.md) | OpenCC, i18n, S13n, E4X, Polyfill, Arrayx, Numberx, Mathx, Pinyin, Zip, NanoID, Mime 等 | 中文转换、国际化、标准化、JS 扩展、工具类 |
| [09-类型系统参考](开发指引/09-类型系统参考.md) | UiSelector, UiObject, UiObjectCollection, UiObjectActions, ImageWrapper, WebSocket, App, Color, Version 等 | 所有复合类型的属性和方法参考 |

### 12.3 源码目录速查

```
app/src/main/java/org/autojs/autojs/
├── runtime/
│   ├── ScriptRuntime.kt              # 桥接注册中心 (augment 方法)
│   ├── ScriptBridges.kt              # JS 回调桥接
│   └── api/
│       ├── augment/                   # ~114 个 Kotlin 桥接文件 (~50 个子包)
│       │   ├── global/Global.kt       #   全局函数 (sleep/exit/click 等)
│       │   ├── automator/Auto.kt      #   自动化操作
│       │   ├── images/Images.kt       #   图像处理
│       │   ├── ...                    #   (其余模块同理)
│       │   └── Augmentable.kt         #   桥接基类
│       ├── AppUtils.kt                # App 工具实现
│       ├── Images.java                # 图像 API 实现
│       ├── Dialogs.java               # 对话框实现
│       ├── Floaty.kt                  # 悬浮窗实现
│       ├── Shell.java                 # Shell 实现
│       └── ...
├── engine/
│   ├── RhinoJavaScriptEngine.kt       # Rhino 引擎封装
│   ├── ScriptEngineService.java       # 引擎生命周期管理
│   └── module/                        # CommonJS require 实现
├── core/
│   ├── accessibility/                 # 无障碍服务 + UiSelector
│   ├── automator/                     # UiObject 操作
│   ├── image/                         # 截图/找色/OpenCV
│   ├── console/                       # 控制台
│   ├── floaty/                        # 浮动窗口
│   ├── ui/                            # E4X 界面
│   └── web/                           # WebView/WebSocket
└── rhino/
    ├── TopLevelScope.java             # Rhino 全局作用域
    └── ProxyObject.kt                 # 代理对象基类
```

### 12.4 阅读建议

**JS 脚本开发者**（使用 AutoJs6 编写自动化脚本）：
1. 先阅读 [AutoJs6 快速上手](AutoJs6_快速上手.md) 了解环境配置和基础用法
2. 按需查阅各子文档的「API 列表总览」和「JS 使用示例」部分
3. 重点关注 [01-自动化操控](开发指引/01-自动化操控.md)（核心 UI 操作）和 [03-图像与视觉](开发指引/03-图像与视觉.md)（找色找图）

**Android 源码贡献者**（理解或修改 AutoJs6 源码）：
1. 先阅读本文档的前十一章了解整体架构和设计决策
2. 本章的「桥接机制」部分帮助理解 JS-Java 如何互通
3. 每个 API 子文档的「源码定位」部分直接指向对应的 augment 和 core 文件

**快速查找某个 JS API**：
1. 在各子文档的「源码定位总表」中按 JS 全局名查找
2. 或直接在源码中搜索 `augment/` 对应子包名

---

> **文档说明：** 本文基于 AutoJs6 仓库源码（shallow clone，commit 截止 2026-04-20）的深度解读生成。核心分析聚焦于引擎架构、无障碍服务、APK 打包机制和 JS API 桥接层，未覆盖全部 4928 个文件的逐一分析。开发指引子文档基于 [docs.autojs6.com](https://docs.autojs6.com) 全部 API 页面与源码交叉分析生成。
