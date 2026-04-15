# 移动端 MCP 工具层开源方案对比

> 筛选标准：**纯操控层 / 工具层**——通过 MCP 协议暴露截图、点击、滑动、元素获取等设备操作能力，供 Cursor / Claude 等外部 Agent 调用，**不内置 LLM 推理逻辑**。
>
> 调研时间：2026-04-15

---

## 一、核心对比表

| 维度 | **mobile-mcp** | **appium-mcp** | **Android-MCP** | **claude-in-mobile** |
|------|----------------|----------------|-----------------|---------------------|
| 仓库 | [mobile-next/mobile-mcp](https://github.com/mobile-next/mobile-mcp) | [appium/appium-mcp](https://github.com/appium/appium-mcp) | [CursorTouch/Android-MCP](https://github.com/CursorTouch/Android-MCP) | [AlexGladkov/claude-in-mobile](https://github.com/AlexGladkov/claude-in-mobile) |
| Stars | **4.5k** | 314 | 528 | 210 |
| 定位 | **纯工具层** | 工具层 + 部分 LLM 辅助 | **纯工具层** | **纯工具层** |
| 平台支持 | **Android + iOS（真机 + 模拟器）** | Android + iOS | **仅 Android** | **Android + iOS 模拟器 + 桌面** |
| 编程语言 | TypeScript | TypeScript | Python | TypeScript / Rust / Kotlin |
| 底层驱动 | 原生平台工具（ADB / xcrun） | **Appium（UiAutomator2 / XCUITest）** | ADB + Accessibility | ADB / simctl / WDA |
| 开源协议 | Apache-2.0 | Apache-2.0 | MIT | MIT |
| 最近更新 | 2026.4.13 (v0.0.52) | 2026.4.14 (v1.56.1) | 2025 | 2026.4.6 (v3.3.0) |
| 安装方式 | `npx @mobilenext/mobile-mcp` | `npx appium-mcp` | `uvx android-mcp` | npm 安装 |
| MCP 传输 | stdio + SSE (`--listen`) | stdio | stdio | stdio |

---

## 二、MCP Tools 能力对比

| 能力 | mobile-mcp | appium-mcp | Android-MCP | claude-in-mobile |
|------|:---:|:---:|:---:|:---:|
| 截图 | ✅ | ✅ | ✅ | ✅ |
| 坐标点击 | ✅ | ✅ | ✅ | ✅ |
| 元素点击 | ✅ | ✅ | ✅ | ✅ |
| 滑动 | ✅ | ✅ | ✅ | ✅ |
| 文本输入 | ✅ | ✅ | ✅ | ✅ |
| 长按 | ✅ | ✅ | ✅ | - |
| 拖拽 | - | - | ✅ | - |
| UI 元素树获取 | ✅ | ✅ | ✅ (Accessibility) | ✅ |
| App 安装 / 卸载 | ✅ | ✅ | - | - |
| Shell 命令执行 | - | - | ✅ | ✅ |
| 屏幕录制 | - | ✅ | - | - |
| 测试代码生成 | - | ✅ (内置 LLM) | - | - |
| 通知栏操作 | - | - | ✅ | - |
| 批量命令 | - | - | - | ✅ |
| 网络模式（远程调用） | ✅ (`--listen`) | - | - | - |

---

## 三、逐项分析

### 1. mobile-next/mobile-mcp — ⭐ 推荐首选

- **优势**
  - Star 最高（4.5k），社区活跃，更新频繁（几乎每周发版）
  - **双平台真机 + 模拟器**全覆盖，覆盖面最广
  - 纯工具层设计，不混入任何 LLM 逻辑，与外部 Agent 解耦最干净
  - 支持 `--listen` 网络模式，可远程调用，适合 CI / 云端部署
  - Accessibility Tree 结构化输出，对 Agent 理解 UI 非常友好
- **劣势**
  - 不支持 Shell 命令执行、屏幕录制
  - 不支持拖拽操作

### 2. appium/appium-mcp — Appium 官方出品

- **优势**
  - 基于成熟的 **Appium 生态**，设备兼容性最好，机型覆盖最广
  - Appium 官方维护，长期稳定性有保障
  - 支持屏幕录制、App 生命周期管理等高级功能
  - 一键配置 Cursor / Claude / Gemini CLI
- **劣势**
  - **不是纯工具层**——内置了 LLM 调用做语义元素发现和测试代码生成
  - 依赖 Appium Server 运行，部署链路较长（需先安装 Appium + 对应驱动）
  - Star 较低，社区规模小

### 3. CursorTouch/Android-MCP — Android 专精轻量方案

- **优势**
  - **纯工具层**，明确声明不依赖 CV 模型或 OCR
  - Python 实现，轻量易集成，`uvx` 一键安装
  - 提供拖拽、通知栏操作、Shell 命令等独特能力
  - MIT 协议最宽松
- **劣势**
  - **仅支持 Android**，无 iOS
  - 更新节奏较慢（最近更新停留在 2025 年）
  - 无网络模式，不支持远程调用

### 4. AlexGladkov/claude-in-mobile — 多平台统一操控

- **优势**
  - **纯工具层**，跨平台统一接口
  - 支持 Android + iOS 模拟器 + 桌面（Compose Multiplatform）+ Aurora OS
  - Rust 实现部分性能好
  - 支持批量命令执行，减少 MCP 调用次数
- **劣势**
  - iOS 仅支持**模拟器**（通过 simctl + WDA），不支持 iOS 真机
  - Star 较低（210），社区规模最小
  - 文档相对简略

---

## 四、带Agent的项目

以下项目 Star 较高但**不符合「纯工具层」要求**，内置了 LLM 推理 / Agent 编排逻辑：

| 项目 | Stars | 排除原因 |
|------|-------|---------|
| [droidrun/droidrun](https://github.com/droidrun/droidrun) | 8.2k | 完整 Agent 框架，内置 LLM 编排，无独立 MCP server 模式 |
| [zai-org/Open-AutoGLM](https://github.com/zai-org/Open-AutoGLM) | 24.9k | 绑定智谱 GLM 模型的完整 Agent，非通用工具层 |
| [zillow/auto-mobile](https://github.com/zillow/auto-mobile) | 79 | 内置 LLM 推理（自动写测试），仅 Android，Star 很低 |

---

## 五、选型建议

```
需要双平台 (Android + iOS) 真机支持   → mobile-mcp（首选）
已有 Appium 基础设施且接受部分 LLM 耦合 → appium-mcp
只做 Android + 需要 Shell / 拖拽能力   → Android-MCP
需要统一桌面 + 移动端操控              → claude-in-mobile
```

**综合推荐 `mobile-next/mobile-mcp`**：纯工具层设计最干净，双平台真机支持，Star 和活跃度最高，MCP 协议支持最完善（stdio + 网络模式），与 Cursor / Claude 等外部 Agent 集成最顺畅。
