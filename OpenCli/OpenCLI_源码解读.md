# OpenCLI 源码解读

> 仓库: https://github.com/jackwener/OpenCLI  
> 版本: v1.7.4  
> 分析日期: 2026-04-16  
> 本地路径: `~/Documents/coding/github/OpenCLI`

---

## 一、项目概述

**OpenCLI** 是一个将任意网站或 Electron 桌面应用变成命令行工具的框架。核心理念是 "Make any website or Electron App your CLI"，由 AI 驱动 API 发现与适配器生成。

### 仓库规模

| 指标 | 数值 |
|------|------|
| 总文件数 | 1244 |
| TypeScript 源文件 | 179 |
| 内建适配器 (clis/) | 87+ 站点 |
| 核心源码 (src/) | ~30 模块 |

### 核心特性

| 特性 | 说明 |
|------|------|
| 浏览器自动化 | 通过 Chrome DevTools Protocol (CDP) 控制浏览器，支持点击/输入/截图/数据提取 |
| 桌面应用控制 | 驱动 Electron 应用（Cursor、ChatGPT、Notion 等） |
| 账号安全 | 复用 Chrome 已登录状态，凭证永远不离开浏览器 |
| 零 LLM 成本 | 运行时不消耗 token，确定性输出 |
| AI Agent Ready | explore/synthesize/cascade 等 AI 工作流内建 |
| 插件系统 | 支持 Git/本地/Monorepo 插件安装与管理 |

### 适用场景

- AI Agent 需要从网站结构化获取数据（如新闻、社交媒体、电商等）
- 自动化浏览器操作（登录态下抓取、发帖、评论等）
- 驱动 Electron 桌面应用
- CI/CD 环境中的网站数据采集管道

---

## 二、整体架构

```
+-----------------------------------------------------+
|                    opencli CLI                        |
|              (Commander.js 入口)                      |
+-----------------------------------------------------+
|                    引擎层                              |
|  +-------------+  +--------------+  +-------------+  |
|  |   Registry  |  |   Dynamic    |  |   Output    |  |
|  |  (命令注册) |  |   Loader     |  |  Formatter  |  |
|  +-------------+  +--------------+  +-------------+  |
+-----------------------------------------------------+
|                   适配器层                             |
|  +-----------------+  +--------------------------+   |
|  |    Pipeline     |  |  TypeScript Adapters     |   |
|  |  (声明式管道)   |  |  (browser/desktop/AI)    |   |
|  +-----------------+  +--------------------------+   |
+-----------------------------------------------------+
|                   连接层                               |
|  +-----------------+  +--------------------------+   |
|  | Browser Bridge  |  |  CDP (Chrome DevTools)   |   |
|  | (Extension+WS)  |  |  (Electron apps)         |   |
|  +-----------------+  +--------------------------+   |
+-----------------------------------------------------+
```

### 双引擎架构

OpenCLI 构建在 **Dual-Engine Architecture** 之上：

1. **Pipeline 引擎**（声明式）: 通过 YAML/JSON 定义 fetch -> select -> map -> filter -> limit 等步骤序列
2. **TypeScript Adapter 引擎**（编程式）: 直接编写 JS/TS 函数，拥有完整的浏览器页面控制能力

---

## 三、核心模块详解

### 3.1 入口与启动 (`src/main.ts`)

采用**分层启动**策略，极致优化冷启动性能：

```
Ultra-fast path（<5ms）:
  --version / -V  → 直接输出版本号退出
  completion <shell> → 打印补全脚本退出
  --get-completions → 从 manifest 读取补全候选退出

Full startup path（按需加载）:
  1. 并行发现: ensureUserCliCompatShims() | ensureUserAdapters() | discoverClis(BUILTIN)
  2. 串行发现: discoverClis(USER) → discoverPlugins()
  3. 注册更新检查钩子
  4. 运行 CLI
```

关键设计：用户 CLI 必须在内建 CLI 之后发现，以实现用户适配器覆盖内建适配器的机制。

### 3.2 命令注册中心 (`src/registry.ts`)

使用 `globalThis.__opencli_registry__` 确保跨模块实例的单例注册表。与 `hooks.ts` 一样，这是为了解决 npm link / peerDependency 导致的多模块拷贝问题：

```typescript
// globalThis 单例模式 — 无论多少份模块拷贝，registry 永远只有一个
declare global {
  var __opencli_registry__: Map<string, CliCommand> | undefined;
}
const _registry: Map<string, CliCommand> =
  globalThis.__opencli_registry__ ??= new Map();
```

- **`cli(opts)`**: 注册命令的统一入口
- **`normalizeCommand()`**: 将 `strategy` 解码为运行时字段 `browser`、`navigateBefore`
- **5 级认证策略**: PUBLIC → COOKIE → HEADER → INTERCEPT → UI

### 3.3 命令执行引擎 (`src/execution.ts`)

`executeCommand()` 是所有命令执行的单一入口，处理完整生命周期：

```
参数校验 → coerceAndValidateArgs()
  ↓
生命周期钩子 → emitHook('onBeforeExecute')
  ↓
路由决策 → shouldUseBrowserSession(cmd)
  ├── 浏览器命令 → browserSession() → preNav → runWithTimeout → closeWindow
  └── 非浏览器命令 → runCommand(cmd, null, kwargs)
  ↓
生命周期钩子 → emitHook('onAfterExecute')
```

**懒加载机制**: `InternalCliCommand._lazy` + `_modulePath` 实现模块按需加载，命令清单从 manifest 注册为占位符，实际执行时才 `import()` 加载模块。

### 3.4 Pipeline 引擎 (`src/pipeline/`)

声明式管道执行器，支持以下步骤类型：

| 步骤 | 说明 | 是否需要浏览器 |
|------|------|:---:|
| `fetch` | HTTP 请求 | 否 |
| `navigate` | 页面导航 | 是 |
| `click` / `type` / `press` | DOM 交互 | 是 |
| `wait` | 等待条件 | 是 |
| `snapshot` | DOM 快照 | 是 |
| `evaluate` | 执行 JS | 是 |
| `intercept` | 网络拦截 | 是 |
| `select` / `map` / `filter` / `sort` / `limit` | 数据变换 | 否 |
| `download` / `tap` | 下载/旁路 | 否 |

**关键特性**: 浏览器步骤自带 **retry 机制**（默认 2 次），仅对 `isTransientBrowserError()` 判定为瞬态的错误重试。

### 3.5 浏览器连接 (`src/browser/`)

```
CLI Process ←→ WebSocket Daemon ←→ Chrome Extension ←→ Chrome Browser
                    ↕                    ↕
              Daemon Client         CDP Protocol
```

- **BrowserBridge**: 通过 WebSocket 连接到本地 daemon
- **daemon-client.ts**: 管理 daemon 健康检查、会话列表
- **target-resolver.ts**: 解析 CDP target，处理 SPA 导航导致的 target 失效
- **errors.ts**: 统一错误分类（extension-transient / target-navigation / non-retryable）

### 3.6 错误体系 (`src/errors.ts`)

遵循 Unix sysexits.h 规范的退出码体系：

| 退出码 | 含义 | 错误类 |
|:---:|------|------|
| 0 | 成功 | - |
| 2 | 参数/用法错误 | `ArgumentError` |
| 66 | 无数据/空结果 | `EmptyResultError` |
| 69 | 服务不可用 | `BrowserConnectError`, `AdapterLoadError` |
| 75 | 临时故障 | `TimeoutError` |
| 77 | 需要认证 | `AuthRequiredError` |
| 78 | 配置错误 | `ConfigError` |

所有错误携带 `code`（机器可读）+ `hint`（人类可读修复建议）+ `exitCode`（Unix 退出码）。

### 3.7 插件系统 (`src/plugin.ts`)

支持三种安装源：

1. **Git 仓库**: `github:user/repo` / HTTPS / SSH / SCP 格式
2. **本地目录**: 符号链接，用于开发模式
3. **Monorepo**: 一个仓库包含多个子插件，各自通过符号链接指向 monorepo clone

安装生命周期：`clone → validate → installDependencies → linkHostOpencli → transpileTS → register`

**事务机制**: `Transaction` 类实现安装/更新的原子操作，失败时自动回滚（备份旧目录 → 替换 → finalize/rollback）。

### 3.8 能力路由 (`src/capabilityRouting.ts`)

决定一条命令是否需要浏览器会话的核心路由逻辑：

```typescript
const BROWSER_ONLY_STEPS = new Set([
  'navigate', 'click', 'type', 'wait', 'press',
  'snapshot', 'evaluate', 'intercept', 'tap',
]);

export function shouldUseBrowserSession(cmd: CliCommand): boolean {
  if (!cmd.browser) return false;          // 非浏览器命令 → 直接跳过
  if (cmd.func) return true;               // 有 func → 需要浏览器
  if (!cmd.pipeline || cmd.pipeline.length === 0) return true;
  if (cmd.navigateBefore) return true;     // 需要预导航 → 需要浏览器
  return pipelineNeedsBrowserSession(cmd.pipeline); // 检查 pipeline 步骤
}
```

`execution.ts` 中的 `executeCommand()` 调用此函数做路由分叉：浏览器命令走 `browserSession()` → 预导航 → 超时执行 → 关窗口；非浏览器命令直接 `runCommand()`。

### 3.9 CLI 命令定义 (`src/cli.ts`)

`cli.ts` 定义了所有内建管理命令（AI 工作流相关），均使用**懒加载**（`await import()`）：

| 命令 | 说明 | 关键选项 |
|------|------|---------|
| `opencli explore <url>` | 浏览器探索：发现 API、stores、推荐策略 | `--auto` `--click <labels>` `--wait <s>` `--goal` `--site` |
| `opencli synthesize <target>` | 从 explore 结果合成候选适配器 YAML | `--top <n>` |
| `opencli generate <url>` | 一键全流程: explore → synthesize → verify → register | `--goal` `--site` `--no-register` `--format` |
| `opencli record <url>` | 录制实时浏览器会话的 API 调用 → 生成候选 YAML | `--poll <ms>` `--timeout <ms>` `--out <dir>` |
| `opencli cascade <url>` | 策略级联：自动找到最简可行策略 | `--site` |

**record 命令**是 explore 的补充方案：explore 是自动化的一次性快照，record 则是**持续录制**模式 — 打开浏览器后人工操作页面，CLI 每隔 `--poll` 毫秒轮询新的网络请求，`--timeout` 毫秒后自动停止，最终从录制的请求中生成候选适配器。

```typescript
// cli.ts:234 — record 命令定义
program
  .command('record')
  .description('Record API calls from a live browser session → generate YAML candidates')
  .argument('<url>', 'URL to open and record')
  .option('--poll <ms>', 'Poll interval in milliseconds', '2000')
  .option('--timeout <ms>', 'Auto-stop after N milliseconds (default: 60000)', '60000')
```

---

## 四、CLI 生成流程 — Verified Generation Harness

> **核心思想**: AI 生成的适配器不是"生成即完成"，而必须经过完整的 harness 流程才能注册为可用命令。
>
> OpenCLI 将 "Harness" 定义为 **"约束+度量+闭环"** 的工程理念。本章以用户生成 CLI 的完整路径为主线，逐一剖析每个 Phase 的实现。

### 4.1 全流程总览与设计合约

**实现**: `src/generate-verified.ts`（973 行）

用户通过一条命令即可完成全流程：

```bash
opencli generate <url> --goal <text> --site <name>
```

也可以分步执行：

```bash
opencli explore <url>                            # Phase 1
opencli synthesize <site>                        # Phase 2
opencli cascade <url>                            # Phase 3
# Phase 4-6 由 generate 内部编排，无独立命令
```

**全流程编排图**:

```
opencli generate <url>
  │
  ├─ Phase 1: Explore ──────── exploreUrl()
  │    → 浏览器打开页面，拦截网络请求，发现 JSON API endpoints
  │    → 输出: endpoints[], capabilities[], 5 个 JSON artifacts
  │    → [EarlyHint: api-surface-looks-viable / no-viable-api-surface]
  │
  ├─ Phase 2: Synthesize ───── synthesizeFromExplore()
  │    → 从 explore 结果生成候选适配器 YAML
  │    → 输出: candidates[]
  │    → [EarlyHint: candidate-ready-for-verify / no-viable-candidate]
  │
  ├─ Phase 3: Cascade Probe ── cascadeProbe()
  │    → 在同一浏览器会话内探测最简可行认证策略
  │    → PUBLIC → COOKIE → HEADER 逐级降级
  │    → 输出: bestStrategy + confidence
  │
  ├─ Phase 4: Verify ────────── verifyCandidate()
  │    → 执行候选 pipeline，检查结果质量
  │    → assessResult(): 非数组 / 空数组 / 字段过少 → 失败
  │
  ├─ Phase 4.5: Bounded Repair
  │    → 首次验证失败且原因为 empty-result → 替换 itemPath 后重试一次
  │
  └─ Phase 5: Persist ────────── writeAdapter() + registerCommand()
       → 成功 → 写入 ~/.opencli/clis/<site>/<name>.js + .meta.json
       → 注册到 registry
```

**纯 CLI，零 Agent 依赖**: 整个流程是纯 TypeScript 代码，运行时零 LLM 消耗。Agent 的角色是**可选的**，仅在需要自修复（见 5.3）或自动化研究（见 6.1）时才介入。

#### Agent 的角色

Agent 不是必须的，但 OpenCLI 的设计是 **Agent-ready**：

| 场景 | 需要 Agent？ | 说明 |
|------|:---:|------|
| `opencli generate` 成功 | 不需要 | CLI 自己完成全流程 |
| `generate` 返回 `needs-human-check` | 可选 | Agent 可读取结构化输出，决定下一步 |
| `generate` 返回 `blocked` | 可选 | Agent 可根据 `reason` 判断是否切换策略 |
| 适配器运行时失败 | 需要 | Self-Repair（`opencli-autofix` skill）需要 Agent 来读诊断、改代码、重试（见 5.3） |
| AutoResearch 迭代 | 需要 | `autoresearch/engine.ts` 的 `modify` 回调需要 Agent 来 ideate + 改代码（见 6.1） |

**总结**: generate 的完整 harness 流程是**纯 CLI 内建**的，不依赖 Agent。Agent 的价值在于处理 **harness 失败后的恢复**（Self-Repair）和**持续优化**（AutoResearch），这是两个更高层级的 harness，它们构建在 CLI harness 之上。

```
层级关系:

  ┌─────────────────────────────────────────┐
  │   AutoResearch Harness (Agent 驱动)      │ ← 持续优化
  │   ┌─────────────────────────────────┐   │
  │   │ Self-Repair Harness (Agent 驱动) │   │ ← 失败恢复
  │   │   ┌─────────────────────────┐   │   │
  │   │   │ CLI Harness (纯 CLI)     │   │   │ ← 生成+运行
  │   │   │ generate/explore/verify  │   │   │
  │   │   └─────────────────────────┘   │   │
  │   └─────────────────────────────────┘   │
  └─────────────────────────────────────────┘
```

**record 命令 — Explore 的补充方案**: explore 是自动化的一次性快照，`opencli record <url>` 则是**持续录制**模式 — 打开浏览器后人工操作页面，CLI 每隔 `--poll` 毫秒轮询新的网络请求，`--timeout` 毫秒后自动停止，最终从录制的请求中生成候选适配器。

#### v1 合约范围

```typescript
// generate-verified.ts:5-9 — 设计约束
// v1 contract keeps scope narrow:
//   - PUBLIC + COOKIE only        — 不处理 HEADER/INTERCEPT/UI
//   - read-only JSON API surfaces — 只发现 GET 类 JSON API
//   - single best candidate only  — 不生成多候选
//   - bounded repair: select/itemPath replacement once — 修复上限 1 次
```

**合约设计原则**（源码注释）:
1. machine-readable — 所有输出可机器解析
2. explicit + explainable — 每个决策都有明确理由
3. testable + versioned — 可测试可版本化
4. taxonomy by skill decision needs — 分类按技能决策需求而非内部错误来源
5. early hint / terminal outcome share consistent decision language — 统一决策语言

#### 三级输出分类

| 状态 | 含义 | 可复用性标记 |
|------|------|---------|
| `success` | 验证通过 | `verified-artifact` |
| `needs-human-check` | 需人工检查 | `unverified-candidate` |
| `blocked` | 无法继续 | `not-reusable` |

---

### 4.2 Phase 1: Explore — 9 步确定性 API 发现

**实现**: `src/explore.ts`（516 行）+ `src/analysis.ts`（180 行）

**核心原理**: 用真实浏览器当"探针"，**被动抓包 + 主动触发**，再用**确定性规则**分析抓到的数据。完全不依赖 Agent，不依赖 LLM，全部在 `exploreUrl()` 函数内完成。

---

#### Step 1: 导航 + 开启网络捕获

```typescript
page.startNetworkCapture()  // 开始录制所有网络请求
page.goto(url)               // 打开目标页面
page.wait(3s)                // 等待页面加载
```

此时浏览器已经发出了**初始加载的所有 API 请求**，网络捕获器在后台默默记录。

---

#### Step 2: 自动滚动触发懒加载

```typescript
page.autoScroll({ times: 3, delayMs: 1500 })
```

模拟用户滚动页面 3 次，每次间隔 1.5 秒。目的是触发 **infinite scroll / lazy loading**，让页面发出更多的 API 请求。

---

#### Step 2.5: 交互式 Fuzzing（可选，`--auto` / `--click`）

这一步**默认关闭**，仅在显式指定选项时启用。目的是：很多 API 只在用户点击 tab/按钮后才会触发，通过自动点击来"钓出"更多隐藏的 API。

**模式一：默认模式（仅滚动）**
```bash
opencli explore <url>
```
- 只执行 Step 2 的 `autoScroll`，**不点击任何元素**
- 适合大多数 SPA 页面

**模式二：`--click` 精确标签点击**
```bash
opencli explore <url> --click "评论,字幕,热门"
```
- 根据提供的标签文本，精确定位并点击匹配的 Tab/Button：
```typescript
// 按标签点击特定元素
for (label of clickLabels) {
  // 找到文本包含该 label 的 button/tab/a/span → click()
}
```
- 在盲目 fuzzing 之前执行
- 适合需要切换特定 Tab 才能触发 API 的场景

**模式三：`--auto` 盲目模糊点击**
```bash
opencli explore <url> --auto
```
执行 `INTERACT_FUZZ_JS` 脚本（`src/scripts/interact.ts`, 23 行）：

```typescript
export async function interactFuzz() {
  const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));
  const clickables = Array.from(document.querySelectorAll(
    'button, [role="button"], [role="tab"], .tab, .btn, ' +
    'a[href="javascript:void(0)"], a[href="#"]'
  )).slice(0, 15); // 限制最多 15 个，防止死循环

  let clicked = 0;
  for (const el of clickables) {
    try {
      const rect = el.getBoundingClientRect();
      if (rect.width > 0 && rect.height > 0) { // 只点可见元素
        el.dispatchEvent(new MouseEvent('click', {
          bubbles: true, cancelable: true, view: window
        }));
        clicked++;
        await sleep(300); // 等 300ms 让 XHR 有时间发出去
      }
    } catch {} // 单个元素失败不影响后续
  }
  return clicked;
}
```

**安全约束**:
- 最多点击 **15 个元素**（`.slice(0, 15)`）
- 每次点击间隔 **300ms**，给网络请求触发时间
- 只点击**可见元素**（`rect.width > 0 && rect.height > 0`）
- 只点击不会导航离开页面的元素（排除真实链接，仅 `a[href="javascript:void(0)"]` 和 `a[href="#"]`）
- 使用 `dispatchEvent(new MouseEvent('click'))` 而非 `el.click()`，给页面框架处理冒泡的机会

---

#### Step 3: 读取页面元数据

```typescript
page.evaluate(() => ({ url: window.location.href, title: document.title }))
```

---

#### Step 4: 收割网络流量

```typescript
page.readNetworkCapture()   // 拿到所有捕获的网络请求
parseNetworkRequests()      // 统一格式化为 { method, url, status, contentType, responseBody }
```

到这一步，已经有了页面加载期间**所有 HTTP 请求的完整记录** — 包括初始加载、滚动触发和点击触发的全部请求。

---

#### Step 5: 补充抓取 JSON 响应体（iframe Re-Fetch）

对于只拦截到了 URL 但没有 body 的 JSON 请求，在浏览器内用 **iframe** 重新 fetch：

```typescript
// 创建一个干净的 iframe，用其 fetch（避免 SPA 框架拦截）
iframe = document.createElement('iframe');
iframe.contentWindow.fetch(url, { credentials: 'include' });
// 拿回 JSON body，限制 10KB 防止过大
```

**为什么用 iframe？** 因为很多 SPA 框架（React/Vue）会 monkey-patch `window.fetch`，直接调 fetch 可能被拦截或返回缓存数据。iframe 的 `contentWindow.fetch` 是**原始的、未被篡改的**。

---

#### Step 6: 检测前端框架和状态管理

注入 `src/scripts/framework.ts`（40 行）到页面上下文执行：

```typescript
export function detectFramework() {
  const app = document.querySelector('#app') as VueAppEl | null;
  const w = window as FrameworkWindow;
  return {
    vue3:  !!(app && app.__vue_app__),
    vue2:  !!(app && app.__vue__),
    react: !!w.__REACT_DEVTOOLS_GLOBAL_HOOK__ || !!document.querySelector('[data-reactroot]'),
    nextjs: !!w.__NEXT_DATA__,
    nuxt:  !!w.__NUXT__,
    pinia: !!(app?.__vue_app__?.config?.globalProperties?.$pinia),
    vuex:  !!(app?.__vue_app__?.config?.globalProperties?.$store),
  };
}
```

这些都是各框架留在 DOM / window 上的"指纹"。检测结果会写入 `manifest.json`，供后续 synthesize 阶段参考。

---

#### Step 7: 端点分析 — 确定性规则引擎（核心智能在这里）

`analyzeEndpoints()` + `src/analysis.ts`（180 行）用**纯规则引擎**完成以下分析：

**a) URL 归一化** (`urlToPattern`):

```
https://api.zhihu.com/topics/19551894/hot?limit=20&offset=0
  → api.zhihu.com/topics/{id}/hot?limit={}&offset={}
```

- 数字 ID → `{id}`
- Hex 串 → `{hex}`
- B 站 BV 号 → `{bvid}`
- 去掉易变参数（timestamp、callback 等 `VOLATILE_PARAMS`）

**b) 过滤噪音** (`isNoiseUrl`):

去掉图片/字体/CSS/JS/tracking/analytics 请求：
```typescript
const NOISE_URL_PATTERN = /\/(track|log|analytics|beacon|pixel|ping|heartbeat|keep.?alive)\b/i;
```

**c) JSON 响应体深度分析** (`findArrayPath`):

```typescript
// 给定 API 响应：
{ "data": { "items": [{ "title": "...", "author": "..." }, ...] } }

// findArrayPath() 递归搜索（最深 5 层）：
// → 找到最大的对象数组
// → 返回 { path: "data.items", items: [...] }
```

这是从 JSON 响应中提取"列表数据在哪里"的核心算法。要求数组长度 >= 2 且元素为对象，多个候选时选最大的。

**d) 字段语义识别** (`detectFieldRoles`):

通过预定义的 `FIELD_ROLES` 别名表，自动判断字段角色：

| 预定义别名 | 语义角色 |
|-----------|---------|
| `title` / `name` / `headline` | title |
| `url` / `link` / `href` | url |
| `author` / `user` / `creator` | author |
| `img` / `avatar` / `cover` / `thumbnail` | image |
| `score` / `likes` / `votes` | score |
| `created_at` / `pubDate` / `publish_time` | time |

**e) 认证策略检测** (`detectAuthFromHeaders` + `detectAuthFromContent`):

| 检测依据 | 推断的认证方式 |
|---------|-------------|
| 请求头含 `Authorization` | `bearer` |
| 请求头含 `X-CSRF` / `X-XSRF` | `csrf` |
| 请求头含 `X-S` / `X-T` / `X-S-Common` | `signature`（小红书/抖音特有的签名算法）|
| URL 含 `/wbi/` 或 `w_rid=` | `signature`（B 站 wbi 签名）|
| 响应体含 `sign` / `w_rid` / `token` 字段 | `signature` |

**f) 查询参数分类** (`classifyQueryParams`):

| 参数名 | 分类 |
|-------|------|
| `keyword` / `query` / `q` | 搜索参数 |
| `page` / `offset` / `cursor` | 分页参数 |
| `limit` / `count` / `size` / `pageSize` | 限制参数 |

---

#### Step 8: 推断 CLI 能力

`inferCapabilitiesFromEndpoints()` 对排名前 8 的端点，自动推断能力和参数：

**能力名推导**:

| URL 关键词 | 推断的能力名 |
|-----------|------------|
| `hot` / `popular` / `trending` / `ranking` | `hot` |
| `search` / `query` | `search` |
| `feed` / `timeline` / `dynamic` | `feed` |
| `comment` / `reply` | `comments` |
| `history` | `history` |
| `profile` / `userinfo` / `/me` | `me` |
| `favorite` / `collect` / `bookmark` | `favorite` |
| 无匹配 | 取 URL 最后一段路径 |

**参数推荐**:
- 有搜索参数 → 推荐 `args: [{ name: 'keyword', required: true }]`
- 有分页参数 → 推荐 `args: [{ name: 'page', default: 1 }]`
- 始终推荐 → `args: [{ name: 'limit', default: 20 }]`

---

#### Step 9: 写入磁盘

最终输出 **5 个 JSON 文件**到 `.opencli/explore/<site>/`：

| 文件 | 内容 |
|------|------|
| `manifest.json` | 站点元数据、框架信息、探索时间 |
| `endpoints.json` | 所有发现的 API 端点（pattern, arrayPath, fields, auth 等）|
| `capabilities.json` | 推断出的 CLI 能力（name, strategy, columns, args）|
| `auth.json` | 认证策略摘要 |
| `stores.json` | Pinia/Vuex Store 信息（如有）|

这些 JSON artifacts 是后续 Phase 2 (Synthesize) 的输入。

---

#### 零 LLM 的技术原理总结

整个 explore 过程没有任何 LLM 调用，依赖的全是确定性规则：

| 步骤 | 技术手段 | 不用 LLM 的原因 |
|------|---------|----------------|
| API 发现 | 浏览器网络抓包 | 网络请求是客观事实，不需要"理解" |
| 触发更多 API | 自动滚动 + 盲点击 | 机械操作，模拟用户行为 |
| URL 归一化 | 正则替换 | 数字→`{id}`、Hex→`{hex}` 是固定模式 |
| 数组发现 | 递归搜索 JSON | 找最大对象数组是纯算法 |
| 字段语义 | 预定义别名表 | `title`/`name`/`headline` 这类映射是有限集合 |
| 认证检测 | HTTP 头模式匹配 | Authorization / CSRF 等是标准协议 |
| 能力命名 | URL 路径关键词 | `hot`/`search`/`feed` 等是通用命名惯例 |

**设计哲学**: 把"网站通常长什么样"的领域知识硬编码为规则，用浏览器当被动探针，从而在不需要任何 AI 推理的情况下完成 API 发现。代价是对于非常规的网站结构可能识别不全，但对主流网站（知乎、B 站、微博、Twitter 等）的覆盖率很高。

---

### 4.3 Phase 2: Synthesize — 模板化候选生成

**实现**: `src/synthesize.ts`

从 Explore 阶段写入的 5 个 JSON artifacts 中读取数据，生成候选适配器 YAML：

```
读取 .opencli/explore/<site>/endpoints.json
  → 过滤: 只保留有 arrayPath 的端点（即返回列表数据的 API）
  → 排序: 按数组长度降序
  → 截取: 取 top N 个（默认 3）
  → 对每个端点生成 YAML pipeline:
      fetch: <url>
      select: <arrayPath>
      map: { <fieldRole>: <fieldPath>, ... }
  → 输出: CandidateYaml[]
```

接下来 `selectCandidate()` 根据用户提供的 `--goal` 从候选中选择最佳匹配。如果用户未指定 goal，默认选第一个（数据量最大的）。

**与 record 的关系**: synthesize 既可以消费 explore 的产物，也可以消费 record 的产物。两者输出格式相同（endpoints.json），synthesize 不关心数据来源。

---

### 4.4 Phase 3: Strategy Cascade — 5 级策略自动探测

**实现**: `src/cascade.ts`（184 行）

**核心思想**: 自动发现最小权限策略，不需要人工指定。

**完整 5 级策略链**:

```typescript
// cascade.ts:26 — 策略级联顺序
const CASCADE_ORDER: Strategy[] = [
  Strategy.PUBLIC,    // 直接 fetch, 无 credentials
  Strategy.COOKIE,    // fetch with credentials: 'include'
  Strategy.HEADER,    // fetch + 提取 CSRF token (ct0/csrf_token/_csrf)
  Strategy.INTERCEPT, // 需要签名/加密，站点特定实现
  Strategy.UI,        // 需要浏览器 DOM 交互获取数据
];
```

**探测流程**:

```
PUBLIC → page.evaluate(fetch(url, {}))
  → 成功且 hasData → 返回 PUBLIC (confidence: 1.0)
  → 失败 ↓
COOKIE → page.evaluate(fetch(url, { credentials: 'include' }))
  → 成功且 hasData → 返回 COOKIE (confidence: 0.9)
  → 失败 ↓
HEADER → page.evaluate(fetch(url, { credentials: 'include', headers: { 'X-Csrf-Token': csrf } }))
  → 成功且 hasData → 返回 HEADER (confidence: 0.8)
  → 失败 ↓
INTERCEPT / UI → 跳过（需要站点特定实现，不自动探测）
  → 默认回退 → COOKIE (confidence: 0.3)
```

**关键实现细节**:

```typescript
// cascade.ts:93 — 策略 → fetch 选项映射
const PROBE_OPTIONS = {
  [Strategy.PUBLIC]:  {},
  [Strategy.COOKIE]:  { credentials: true },
  [Strategy.HEADER]:  { credentials: true, extractCsrf: true },
  // INTERCEPT / UI 无映射 → 输出 "requires site-specific implementation"
};

// cascade.ts:156 — 置信度公式
confidence: 1.0 - (i * 0.1)  // 越简单的策略 → 越高的置信度

// cascade.ts:142 — 自动探测上限
const maxIdx = CASCADE_ORDER.indexOf(Strategy.HEADER); // 默认最多探测到 HEADER
```

**CSRF Token 提取逻辑**: 从 `document.cookie` 中查找 `ct0=`（Twitter）、`csrf_token=`、`_csrf=` 前缀的 cookie，提取值后设置到 `X-Csrf-Token` 和 `X-XSRF-Token` 请求头。

**数据有效性判定**: `hasData` 不仅检查响应是否非空，还会检查中国站点常见的 API 错误码模式 — 如果 `json.code !== undefined && json.code !== 0`，则判定为无有效数据。

**与 Explore 中认证检测的关系**: Explore 通过 `detectAuthFromHeaders()` 检测请求头中的认证特征并写入 `auth.json`，Cascade 则在运行时实际发送请求来验证哪种策略真正可行。两者互补：Explore 是静态推断，Cascade 是动态验证。

---

### 4.5 Phase 4-5: Verify & Bounded Repair — 质量评估与修复

**实现**: `src/generate-verified.ts` 中的 `verifyCandidate()`（:440）+ `assessResult()`（:357）+ `withItemPath()`（:379）

#### 具体例子：知乎热榜候选验证

假设 Phase 1-3 生成了如下候选：

```yaml
site: zhihu
name: hot
strategy: COOKIE
pipeline:
  - fetch: "https://www.zhihu.com/api/v3/feed/topstory/hot-lists/total?limit=20"
  - select: "data"
  - map: { title: "target.title", url: "target.link.url", heat: "detail_text" }
```

#### Phase 4: Verify — 真的跑一遍候选 pipeline

`verifyCandidate()` 在**当前浏览器页面**中，用 Pipeline 引擎**真实执行**这个候选 YAML：

```typescript
// generate-verified.ts:440-449
async function verifyCandidate(page, candidate, expectedFields) {
  const result = await executePipeline(page, candidate.pipeline, {
    args: buildDefaultArgs(candidate),
  });
  return assessResult(result, expectedFields);
}
```

然后用 `assessResult()` 对执行结果做 **4 道质量检查**：

```typescript
// generate-verified.ts:357-377
function assessResult(result, expectedFields) {
  // 检查 1: 结果是数组吗？
  if (!Array.isArray(result))       → { ok: false, reason: 'non-array-result' }

  // 检查 2: 数组非空吗？
  if (result.length === 0)          → { ok: false, reason: 'empty-result' }

  // 检查 3: 第一条记录的有效字段 >= 2 个？
  //   (排除 null / undefined / '' 后的字段数)
  if (populated.length < 2)         → { ok: false, reason: 'sparse-fields' }

  // 检查 4: 期望字段至少匹配一个？
  if (expectedFields 有值 && 无匹配) → { ok: false, reason: 'sparse-fields' }

  return { ok: true };              // 全部通过
}
```

**知乎例子 — 验证成功路径**:
```
pipeline 执行返回:
  [{ title: "如何看待...", url: "https://...", heat: "2300万热度" }, ...共20条]

assessResult:
  ✓ 是数组
  ✓ 非空 (20 条)
  ✓ 有效字段 3 个 (>= 2)
  → ok: true → 直接进 Phase 6 Persist
```

#### 验证失败时的异常处理

`verifyCandidate()` 对不同类型的执行异常做分类处理：

| 异常类型 | 判定 | 是否尝试修复 |
|---------|------|:---:|
| `BrowserConnectError` | `blocked` — 浏览器不可用 | 否，直接停止 |
| `AuthRequiredError` | `blocked` — 认证过于复杂 | 否，直接停止 |
| `SelectorError` | `needs-human-check` — 选择器不匹配 | 否 |
| `TimeoutError` | `needs-human-check` — 执行超时 | 否 |
| `CommandExecutionError` | `needs-human-check` — 执行出错 | 否 |
| 结果非数组 (`non-array-result`) | 非 terminal 失败 | 否 |
| 结果字段过少 (`sparse-fields`) | 非 terminal 失败 | 否 |
| **结果为空数组** (`empty-result`) | 非 terminal 失败 | **是 — 进入 Bounded Repair** |

#### Phase 4.5: Bounded Repair — 仅对 empty-result 尝试换路径

**触发条件极其严格**: 只有 `reason === 'empty-result'` 且 Explore 阶段记录了备选 `itemPath` 时才修复。

**知乎例子 — 需要修复的路径**:

```
假设知乎 API 真实返回:
{
  "data": {                    ← select: "data" 取到的是对象，不是数组
    "items": [                 ← 真正的数组在 data.items
      { "target": { "title": "..." }, "detail_text": "2300万热度" },
      ...
    ]
  }
}

第一次 verify: select: "data" → 取到对象 → map 失败 → 结果为空数组
assessResult → reason: 'empty-result'
```

**`withItemPath()` 做的事**（替换 select 路径）:

```typescript
// generate-verified.ts:379-389
function withItemPath(candidate, itemPath) {
  if (!itemPath) return null;                    // 没有备选路径 → 放弃
  if (current.select === itemPath) return null;  // 和当前一样 → 放弃
  next.pipeline[selectIndex] = { select: itemPath }; // 替换！
  return next;
}
```

```
Explore 阶段的 endpoint 记录: itemPath = "data.items"（findArrayPath 发现的）
当前 candidate: select: "data"
  → 不同 → 替换为 select: "data.items"
  → 第二次 verifyCandidate()

第二次结果:
  [{ title: "如何看待...", heat: "2300万热度" }, ...共20条]
  → assessResult: ok: true
  → 修复成功！写入时标记 repair_attempted: true
```

#### 完整决策树

```
verifyCandidate(candidate)
  │
  ├─ ok: true ───────────────→ Phase 6: Persist（status: 'success'）
  │
  ├─ blocked ────────────────→ 直接停止（status: 'blocked'）
  │   (浏览器不可用 / 需登录)    不尝试修复
  │
  ├─ needs-human-check ──────→ 直接停止（status: 'needs-human-check'）
  │   (Selector / Timeout       不尝试修复
  │    / 执行异常)
  │
  └─ 非 terminal 失败:
      ├─ non-array-result ───→ 不修复 → needs-human-check
      ├─ sparse-fields ──────→ 不修复 → needs-human-check
      └─ empty-result ───────→ Bounded Repair:
          │
          ├─ 有备选 itemPath 且不同？
          │   ├─ 否 → 放弃 → needs-human-check
          │   └─ 是 → 替换 select → 第二次 verify
          │       ├─ ok: true → Persist（repair_attempted: true）
          │       └─ 失败 → needs-human-check
          │
          └─ 修复预算: 严格 1 次，绝不做第三次
```

#### 为什么修复只限 1 次 itemPath 替换？

这是 v1 合约的刻意约束（`generate-verified.ts:9`）：
```
bounded repair: select/itemPath replacement once
```

设计哲学：如果换一条 JSON 路径都救不回来，说明问题不在路径选择上（可能是 API 需要登录、需要特殊参数、或返回结构完全不同）。这种情况交给人（`needs-human-check`）或 Agent（Self-Repair，见 5.3）处理更合适，而不是在确定性流程里盲目尝试更多修复。

---

### 4.6 Phase 6: Persist & Register — 产物持久化

**实现**: `src/generate-verified.ts` 中的 `candidateToJs()`（:479）+ `registerVerifiedAdapter()`（:554）+ `writeVerifiedArtifact()`（:565）

**核心**: 把验证通过的候选 YAML 对象变成用户可以直接用的 CLI 命令。做三件事：**编译（YAML → JS）→ 写磁盘 → 注册到内存**。

#### 具体例子：知乎热榜适配器持久化

Phase 4 验证通过后，内存中的候选对象是：

```typescript
candidate = {
  site: 'zhihu', name: 'hot', description: 'Zhihu hot topics',
  domain: 'zhihu.com', strategy: 'cookie', browser: true,
  args: { limit: { type: 'int', default: 20 } },
  columns: ['title', 'url', 'heat'],
  pipeline: [
    { fetch: "https://www.zhihu.com/api/v3/feed/topstory/hot-lists/total?limit=20" },
    { select: "data.items" },
    { map: { title: "target.title", url: "target.link.url", heat: "detail_text" } },
  ],
}
```

**Step 1: 编译** — `candidateToJs()` 把候选对象转为标准 JS 适配器代码：

```javascript
// candidateToJs() 生成的文件内容 — 和手写适配器格式完全一样
import { cli, Strategy } from '@jackwener/opencli/registry';

cli({
  site: 'zhihu',
  name: 'hot',
  description: 'Zhihu hot topics',
  domain: 'zhihu.com',
  strategy: Strategy.COOKIE,
  browser: true,
  args: [
    { name: 'limit', type: 'int', default: 20 },
  ],
  columns: ['title', 'url', 'heat'],
  pipeline: [
    { fetch: 'https://www.zhihu.com/api/v3/feed/topstory/hot-lists/total?limit=20' },
    { select: 'data.items' },
    { map: { title: 'target.title', url: 'target.link.url', heat: 'detail_text' } },
  ],
});
```

编译过程中 `candidateToJs()` 处理的细节：
- strategy 字符串 `'cookie'` → 枚举 `Strategy.COOKIE`
- browser 标志自动推断（`detectBrowserFlag()`）
- 字符串中的引号/反斜杠/换行正确转义
- args 对象 → 数组格式 `[{ name, type, default }]`

**Step 2: 写入磁盘** — 两个文件

```
~/.opencli/clis/zhihu/
  ├── hot.js              ← 可执行适配器（上面的 JS 代码）
  └── hot.meta.json       ← 元数据 sidecar
```

```typescript
// generate-verified.ts:554-562 — registerVerifiedAdapter
const siteDir = path.join(USER_CLIS_DIR, candidate.site);  // ~/.opencli/clis/zhihu/
await fs.promises.mkdir(siteDir, { recursive: true });
await fs.promises.writeFile(adapterPath, candidateToJs(candidate));   // hot.js
await fs.promises.writeFile(metadataPath, JSON.stringify(metadata));  // hot.meta.json
```

**元数据文件 `hot.meta.json` 内容**:

```json
{
  "artifact_kind": "verified",
  "schema_version": 1,
  "source_url": "https://www.zhihu.com/hot",
  "goal": "获取知乎热榜",
  "strategy": "cookie",
  "verified": true,
  "reusable": true,
  "reusability_reason": "verified-artifact"
}
```

元数据记录了这个适配器的"出生证明"：从哪个 URL 生成、用户目标是什么、用了什么策略、是否通过验证。后续 Self-Repair（5.3）读取 `source` 字段定位适配器源码。

**Step 3: 注册到内存** — `registerCommand()`

```typescript
// candidateToCommand() 转为 CliCommand，然后注册
registerCommand(candidateToCommand(candidate, adapterPath));
// → 注册到 globalThis.__opencli_registry__
// → 当前进程内立即可用
```

`candidateToCommand()` 把候选转为 `CliCommand` 结构（和内建命令格式一致），`registerCommand()` 写入全局单例 registry。

**注册后立即可用**:

```bash
# 刚 generate 完，当前进程内已注册，立即可用
opencli zhihu hot --limit 5 -f json

# 下次启动时的加载链路:
# main.ts → discoverClis(USER) → 扫描 ~/.opencli/clis/
# → 发现 zhihu/hot.js → import() 加载 → cli() 自动注册
# → opencli zhihu hot 可用
```

#### 两种写入模式

| 模式 | 触发方式 | 写入路径 | 是否注册 |
|------|---------|---------|:---:|
| 默认 | `opencli generate <url>` | `~/.opencli/clis/<site>/<name>.js` | 是 |
| `--no-register` | `opencli generate <url> --no-register` | `.opencli/explore/<site>/verified/<name>.verified.js` | 否 |

`--no-register` 模式用于测试生成质量 — 只编译和存档到 explore 产物目录，不注册为可用命令，不污染用户的命令空间。

---

### 4.7 决策语言与契约体系

`generate-verified.ts` 定义了一套**类型安全的决策语言**，贯穿 Phase 1-6 全流程：

#### EarlyHint — 阶段间门控

在 explore/synthesize/cascade 每个阶段通过 `onEarlyHint` 回调提前通知调用方，实现成本门控：

```typescript
// generate-verified.ts:83-104 — EarlyHint 接口
type EarlyHintReason =
  | 'api-surface-looks-viable'   // explore 发现可行的 API
  | 'candidate-ready-for-verify' // 候选准备好验证
  | 'no-viable-api-surface'      // 无可行 API → 建议停止
  | 'auth-too-complex'           // 认证过于复杂 → 建议停止
  | 'no-viable-candidate';       // 无可行候选 → 建议停止

interface EarlyHint {
  stage: 'explore' | 'synthesize' | 'cascade';
  continue: boolean;        // 是否继续（false = 建议停止）
  reason: EarlyHintReason;
  confidence: 'high' | 'medium' | 'low';
  candidate?: {             // 当前候选信息（如果有）
    name: string;
    command: string;
    path: string | null;
    reusability: 'unverified-candidate' | 'not-reusable';
  };
  message?: string;
}
```

设计原则是**纯门控**（pure gatekeeping）：Hint 不做终端决策，终端决策由 `GenerateOutcome` 统一负责。Hint 与 Outcome 共享同一套决策语言，Agent/Skill 看到的是一条连续的决策路径。

#### GenerateOutcome — 终端输出契约

```typescript
interface GenerateOutcome {
  status: 'success' | 'blocked' | 'needs-human-check';
  adapter?: VerifiedAdapter;        // success 路径: 已验证的适配器
  reason?: StopReason;              // blocked 路径: 停止原因
  escalation?: EscalationContext;   // needs-human-check 路径: 升级上下文
  reusability?: Reusability;        // 可复用性（success 和 needs-human-check 都有）
  stats: GenerateStats;             // 统计信息
  message?: string;                 // 人类可读摘要
}

interface GenerateStats {
  endpoint_count: number;       // 发现的总端点数
  api_endpoint_count: number;   // 其中 JSON API 端点数
  candidate_count: number;      // 生成的候选数
  verified: boolean;            // 是否通过验证
  repair_attempted: boolean;    // 是否尝试过修复
  explore_dir: string;          // explore 产物目录
}
```

#### 决策分类类型

```typescript
// 终止原因（为什么 blocked）— 按技能决策需求分类
type StopReason =
  | 'no-viable-api-surface'              // 未发现 JSON API 端点
  | 'auth-too-complex'                   // 认证超出 PUBLIC/COOKIE 范围
  | 'no-viable-candidate'               // 候选存在但不达质量门槛
  | 'execution-environment-unavailable'; // 浏览器/daemon 不可用

// 升级原因（为什么 needs-human-check）— 面向行动命名
type EscalationReason =
  | 'empty-result'               // pipeline 执行但返回空
  | 'sparse-fields'              // 结果字段过少
  | 'non-array-result'           // 结果不是数组
  | 'unsupported-required-args'  // 候选需要无法自动填充的参数
  | 'timeout'                    // 执行超时
  | 'selector-mismatch'         // DOM/JSON 路径不匹配
  | 'verify-inconclusive';       // 歧义验证失败

// 建议行动
type SuggestedAction =
  | 'stop'                    // 无更多可尝试
  | 'inspect-with-browser'    // 人工用浏览器调试
  | 'ask-for-login'           // 需要人工登录
  | 'ask-for-sample-arg'      // 需要人工提供参数示例值
  | 'manual-review';          // 通用人工审查

// 可复用性分级
type Reusability =
  | 'verified-artifact'       // 完全验证，可直接使用
  | 'unverified-candidate'    // 候选存在但未验证
  | 'not-reusable';           // 无可保留内容
```

---

## 五、CLI 运行时技术点

> 适配器生成并注册后，用户通过 `opencli <site> <command>` 运行 CLI。本章分析命令执行过程中涉及的运行时 Harness。

### 5.1 Lifecycle Hook Harness（生命周期钩子线束）

**实现**: `src/hooks.ts`（92 行）

**核心思想**: 插件可以接入命令执行的关键节点，实现横切关注点（如遥测、缓存预热、更新检查等）。

**globalThis 单例模式**:

与 `registry.ts` 相同，hooks 使用 `globalThis.__opencli_hooks__` 保证跨模块单例。这是因为 TS 插件通过 npm link / peerDependency 符号链接加载时，可能存在多份模块拷贝，`globalThis` 确保所有拷贝共享同一个 hook store：

```typescript
// hooks.ts:36-41 — globalThis 单例保证
declare global {
  var __opencli_hooks__: Map<HookName, HookFn[]> | undefined;
}
const _hooks: Map<HookName, HookFn[]> =
  globalThis.__opencli_hooks__ ??= new Map();
```

**HookContext 接口**:

```typescript
interface HookContext {
  command: string;                  // "site/name" 格式，或 "__startup__"
  args: Record<string, unknown>;    // 已校验的参数
  startedAt?: number;               // 执行开始时间 (epoch ms)
  finishedAt?: number;              // 执行结束时间 (epoch ms)
  error?: unknown;                  // 命令抛出的错误
  [key: string]: unknown;           // 插件可附加任意数据用于跨钩子通信
}
```

**三个生命周期钩子点**:

```
onStartup       → 所有命令和插件发现完成后（仅一次）
onBeforeExecute  → 每次命令执行前
onAfterExecute   → 每次命令执行后（携带结果/错误）
```

**隔离保证**: 每个 handler 用 try/catch 包裹，一个钩子失败不会阻塞命令执行：

```typescript
// hooks.ts:73-84 — 钩子发射（隔离执行）
export async function emitHook(name: HookName, ctx: HookContext, result?: unknown): Promise<void> {
  for (const fn of handlers) {
    try {
      await fn(ctx, result);
    } catch (err) {
      log.warn(`Hook ${name} handler failed: ${err.message}`);
      // 不抛出，不阻塞
    }
  }
}
```

**测试支持**: `clearAllHooks()` 方法用于测试时重置全局 hook store。

**执行引擎中的集成** (`src/execution.ts`):

```
用户调用 opencli <site> <command> <args>
  → coerceAndValidateArgs()             // 参数校验
  → emitHook('onBeforeExecute', ctx)    // 前置钩子
  → shouldUseBrowserSession(cmd)         // 能力路由（见 3.8）
    ├── 浏览器命令 → browserSession() → preNav → runWithTimeout → closeWindow
    └── 非浏览器命令 → runCommand(cmd, null, kwargs)
  → emitHook('onAfterExecute', ctx)     // 后置钩子
```

### 5.2 Diagnostic Harness（诊断线束）

**核心思想**: 命令执行失败时，系统自动收集结构化诊断上下文，供 AI Agent 消费和修复。

**实现**: `src/diagnostic.ts`

```typescript
// RepairContext 是核心数据契约
interface RepairContext {
  error:   { code, message, hint, stack }       // 结构化错误
  adapter: { site, command, sourcePath, source } // 适配器元数据+源码
  page:    { url, snapshot, networkRequests, capturedPayloads, consoleErrors } // 浏览器状态
  timestamp: string
}
```

**关键设计决策**:

1. **安全边界先行**: 敏感数据（Authorization、Cookie、JWT、API Key）在输出前被 `redactUrl()` / `redactHeaders()` / `redactText()` 脱敏
2. **大小预算控制**: DOM 快照限 100K 字符，源码限 50K，网络请求限 50 条，总输出限 256KB。超出预算时按优先级降级（先砍 snapshot → 砍 page）
3. **带超时的收集**: `PAGE_STATE_TIMEOUT_MS = 5000`，防止 CDP 连接卡死导致诊断本身挂住
4. **标记化输出**: 以 `___OPENCLI_DIAGNOSTIC___` 标记 JSON，方便任意 Agent 框架解析

**触发流程**:

```
命令失败 → execution.ts 捕获错误
  → isDiagnosticEnabled() 检查环境变量 OPENCLI_DIAGNOSTIC
  → collectDiagnostic() 并行收集页面状态（带 5s 超时）
  → emitDiagnostic() 脱敏 → 大小预算裁剪 → 输出到 stderr
```

### 5.3 Self-Repair Harness（自修复线束）

**核心思想**: 命令失败不是终点，Agent 应该 **自动诊断-修复-验证-上报**，形成闭环。

**设计文档**: `designs/self-repair-protocol.md`  
**实现载体**: `skills/opencli-autofix/SKILL.md`

**关键原则**:

> "The command itself is the spec." — 不需要预先编写 spec 文件，命令本身就是验证预言（verify oracle）

**自修复协议**:

```
Agent 执行 opencli <site> <command>
  → 成功 → 继续任务
  → 失败 →
    1. OPENCLI_DIAGNOSTIC=1 重新执行，收集 RepairContext（见 5.2）
    2. 从 RepairContext.adapter.sourcePath 读取适配器源码
    3. 分析: error code + DOM snapshot + network requests → 定位根因
    4. 编辑适配器文件
    5. 重试命令
    6. 仍然失败 → 重复（最多 3 轮）
    7. 3 轮耗尽 → 上报失败
```

**作用域约束**: 只允许修改 `RepairContext.adapter.sourcePath` 指向的文件。绝不修改 `src/`、`extension/`、`tests/`、`package.json`。

**非修复故障识别**: AUTH_REQUIRED / BROWSER_CONNECT / CAPTCHA / Rate Limit → 直接停止，不修改代码。这些是环境问题，修改适配器代码无法解决。

**与 Verified Generation 的关系**: Verified Generation 内部的 Bounded Repair（4.5）是**自动化的、无 Agent 参与的、仅限 itemPath 替换**；Self-Repair 是**Agent 驱动的、可修改任意适配器代码的、最多 3 轮**。前者在生成阶段，后者在运行阶段。

### 5.4 Validation Harness（校验线束）

**实现**: `src/validate.ts` + `src/verify.ts`

**核心思想**: 在运行前就发现问题，而非等到运行时报错。

双层校验：

1. **静态校验** (`validate`): 检查注册表中所有命令的定义完整性
   - description 是否存在
   - browser 命令是否有 domain
   - pipeline 步骤名是否合法
   - func/pipeline 是否至少有一个
   - 参数名是否重复

2. **运行时验证** (`verify`): 静态校验 + 可选的 smoke test（使用 vitest 运行 `tests/smoke/`）

```bash
opencli doctor          # 运行静态校验
opencli doctor --verify # 运行静态校验 + smoke test
```

---

## 六、持续改进

> CLI 生成后，如何系统性地持续提升质量？本章介绍 OpenCLI 的自动化改进和测试保障体系。

### 6.1 AutoResearch Harness（自动化研究线束）

**核心思想**: 借鉴 Karpathy 的 autoresearch 理念 — 用 **约束（scope） + 机械度量（verify） + 无限循环（iteration）** 驱动 AI 自动改进代码。

**实现**: `autoresearch/engine.ts`（363 行）+ `autoresearch/config.ts`（82 行）+ `autoresearch/commands/run.ts`（入口）

#### 谁在什么时候触发？

**AutoResearch 不是终端用户功能，不会被自动触发。** 它是给 OpenCLI **开发者/维护者**用来系统性改进框架自身代码的工具。

触发方式是开发者手动运行命令：

```bash
# 方式一：使用预置配置
npx tsx autoresearch/commands/run.ts --preset browser-reliability
npx tsx autoresearch/commands/run.ts --preset browser-reliability --iterations 5

# 方式二：自定义配置
npx tsx autoresearch/commands/run.ts --goal "..." --verify "..." --scope "src/*.ts" --iterations 10
```

#### 具体触发场景

| 场景 | 预置配置 | 目标 |
|------|---------|------|
| 浏览器命令可靠性不够 | `browser-reliability` | 59 个浏览器命令测试通过率 → 100% |
| Skill 质量不达标 | `skill-quality` | 提升 Skill 质量评分 |
| V2EX 适配器不稳定 | `v2ex-reliability` | 提升 V2EX 适配器可靠性 |
| 知乎适配器不稳定 | `zhihu-reliability` | 提升知乎适配器可靠性 |
| 保存功能不可靠 | `save-reliability` | 提升保存功能可靠性 |
| 综合改进 | `combined` | 多维度综合可靠性提升 |
| 自定义任意模块优化 | 无（通过 CLI 参数） | 开发者自定义 goal/scope/verify |

#### Engine 与 Agent 的分工 — AutoResearch 必须依赖 Agent

AutoResearch **不能自主发现优化点，也不能自主修改代码**。`Engine` 类本身只是一个**循环管理器**（"考官"），所有"发现问题"和"修改代码"的智能都在 `modify` 回调中，即 Agent（"学生"）。如果没有 Agent，Engine 每轮都会记录 `no-op`，什么也不会发生。

Engine 自身能力边界：

| 能力 | Engine 能自己做？ | 说明 |
|------|:---:|------|
| git 前置检查 | 能 | `checkPreconditions()` |
| 运行 verify 提取度量 | 能 | `runVerify()` + `extractMetric()` — 纯数值比较 |
| 运行 guard 回归检查 | 能 | `runGuard()` — 执行 shell 命令 |
| git commit / revert | 能 | 机械操作 |
| 判断 keep/discard | 能 | 度量值大小比较 |
| **发现优化点** | **不能** | 必须由 `modify` 回调（Agent）完成 |
| **修改代码** | **不能** | 必须由 `modify` 回调（Agent）完成 |

这与 Verified Generation（CLI Harness）形成鲜明对比 — CLI Harness 的 explore/synthesize/cascade/verify 全部是纯确定性规则，零 Agent 依赖；而 AutoResearch 的核心改进能力完全依赖外部 Agent。

在 `commands/run.ts` 中，`modify` 回调的实现是调用 Claude Code：

```typescript
// commands/run.ts:63 — modify 回调 = 调用 AI Agent
const result = execSync(
  `claude -p --dangerously-skip-permissions \
    --allowedTools "Bash(npm:*),Bash(npx:*),Bash(git:*),Read,Edit,Write,Glob,Grep" \
    --output-format text "${prompt}"`,
  { cwd: ROOT, timeout: 300_000 }
);
```

**分工关系**:

```
Engine (循环骨架)                     modify 回调 (AI Agent)
──────────────────                    ────────────────────────
Phase 0: 前置检查                      
Phase 1: 收集上下文 ──传递──→          Phase 2-3: 读代码 → 构思 → 改代码
                    ←──返回──          返回一句话描述
Phase 4: git commit                    
Phase 5: 运行 verify 命令              
Phase 5.5: 运行 guard 命令             
Phase 6: keep / discard (回滚)         
Phase 7: 写 TSV 日志                   
Phase 8: 回到 Phase 1                  
```

Engine 通过 `ModifyContext` 向 Agent 传递当前状态：

```typescript
interface ModifyContext {
  iteration: number;          // 当前迭代轮次
  bestMetric: number;         // 历史最佳度量
  currentMetric: number;      // 当前度量
  recentLog: IterationResult[]; // 最近 20 条迭代记录
  gitLog: string;             // 最近 20 条 git log
  scopeFiles: string[];       // 允许修改的文件列表
  consecutiveDiscards: number; // 连续丢弃次数
  stuckHint: string | null;   // 卡住提示（连续丢弃 > 5 时）
}
```

#### 8 阶段迭代循环

```
Phase 0: Precondition — git clean, 无锁, 非 detached HEAD
Phase 1: Review     — 读取 scope 文件 + 日志 + git 历史
Phase 2: Ideate     — 选择下一个变更方向（Agent 内部）
Phase 3: Modify     — 一次原子变更（委托给 modify 回调/Agent）
Phase 4: Commit     — git add + commit（仅 scope 内文件）
Phase 5: Verify     — 运行验证命令，提取度量值
Phase 5.5: Guard    — 可选回归检查（如 npm run build）
Phase 6: Decide     — keep（度量改善且通过 guard）/ discard（git revert 回滚）/ crash
Phase 7: Log        — TSV 日志追加记录
Phase 8: Repeat     — 回到 Phase 1
```

**度量值提取** (`extractMetric`): 从 verify 命令输出的最后几行中提取数值，支持 `SCORE=56` 格式或纯数字行。

**迭代上限**: `maxIter = config.iterations ?? Infinity` — 未指定时默认无限迭代。

**Baseline 测量**: 迭代 0 在主循环前运行 verify 命令，建立基准度量值。

#### Preset 示例 (browser-reliability)

```typescript
// autoresearch/presets/browser-reliability.ts
export const browserReliability: AutoResearchConfig = {
  goal: 'Increase browser command pass rate to 59/59 (100%)',
  scope: [
    'src/browser/dom-snapshot.ts',
    'src/browser/dom-helpers.ts',
    'src/browser/base-page.ts',
    'src/browser/page.ts',
    'src/cli.ts',
  ],
  metric: 'pass_count',
  direction: 'higher',
  verify: 'npx tsx autoresearch/eval-browse.ts 2>&1 | tail -1',
  guard: 'npm run build',
  minDelta: 1,
};
```

#### 决策逻辑与安全机制

**keep/discard 判定**:
```
度量改善 AND |delta| >= minDelta AND guard 通过 → keep（保留 commit）
度量改善但 guard 失败 → discard（git revert）
度量未改善 → discard（git revert）
```

**Commit 作用域约束**: `git add` 仅暂存 `config.scope` 匹配的文件，防止 Agent 越界修改。

**Hook 被拒处理**: 如果 `git commit` 被 pre-commit hook 拒绝，记录 `hook-blocked` 状态，不回滚，继续下一轮。

**卡住检测**: 当连续丢弃次数 > 5 时，提供递进式提示：
1. 重读所有 scope 文件，尝试完全不同的方法
2. 回顾历史日志，组合之前成功的策略
3. 尝试之前失败方向的**反面**
4. 激进的架构变更而非增量调整
5. 简化 — 减少复杂度而非增加

#### 与 Self-Repair 的本质区别

| 维度 | Self-Repair | AutoResearch |
|------|------------|-------------|
| 触发方式 | 被动 — 命令运行失败时 | 主动 — 开发者手动启动 |
| 使用者 | AI Agent（帮终端用户修复适配器） | OpenCLI 开发者（改进框架自身） |
| 目标 | 修复单个适配器让命令能跑通 | 系统性提升代码质量指标 |
| 迭代次数 | 最多 3 轮 | 无限（`Infinity`） |
| 修改范围 | 仅 `adapter.sourcePath` | `scope` 内任意文件 |
| 度量方式 | 命令是否执行成功 | 自定义 verify 命令输出的数值 |
| Agent 角色 | 读诊断 → 改适配器 → 重试 | 读上下文 → 构思 → 原子修改 |
| 回滚机制 | 无（修改就保留） | `git revert`（度量未改善就回滚） |

#### Skill 中不会主动触发 AutoResearch

经过对所有 6 个 Skill（`opencli-autofix`、`opencli-browser`、`opencli-explorer`、`opencli-oneshot`、`opencli-usage`、`smart-search`）的 SKILL.md 源码检查，**没有任何 Skill 包含 AutoResearch 的调用逻辑**。

AutoResearch 和 Skill 是**两条平行路径**：

```
面向终端用户/Agent（运行时）          面向 OpenCLI 开发者（开发时）
──────────────────────────          ──────────────────────────
opencli-usage       → 使用 CLI      autoresearch/run.ts → 改进 CLI 代码
opencli-explorer    → 生成 CLI      preset: browser-reliability → 提升浏览器命令通过率
opencli-autofix     → 修复 CLI      preset: skill-quality → 改进 SKILL.md 质量
opencli-browser     → 浏览器操作     preset: v2ex/zhihu-reliability → 提升站点适配器可靠性
```

唯一的交集是 `skill-quality` preset：它用 AutoResearch 循环来**改进 SKILL.md 本身的质量**（让 Agent 更好地使用 Skill），但仍然是开发者手动启动，不是 Skill 运行时自动触发。

#### 完整例子：browser-reliability 从 51/59 到 59/59

以下用 `browser-reliability` preset 走一遍完整链路，展示人、Agent、Engine、目标程序、文件之间的协作关系。

**参与角色**:

```
┌──────────┐   手动启动    ┌────────────────┐  调用 claude -p  ┌─────────────┐
│  人（开发者）│ ──────────→ │  Engine (CLI)    │ ───────────────→ │  Agent       │
│           │             │  循环管理器       │ ←─────────────── │ (Claude Code)│
└──────────┘             └────────────────┘  返回修改描述     └─────────────┘
                                 │                                     │
                         运行 verify 命令                         读取 + 修改
                                 ↓                                     ↓
                         ┌────────────────┐                  ┌──────────────────┐
                         │ 目标程序         │                  │ scope 文件        │
                         │ eval-browse.ts  │                  │ dom-snapshot.ts   │
                         │ 59 个测试用例    │                  │ dom-helpers.ts    │
                         │ + 真实网站       │                  │ base-page.ts      │
                         └────────────────┘                  │ page.ts / cli.ts  │
                                                             └──────────────────┘
```

**Step 0 — 人启动（人的工作到此结束）**:
```bash
npx tsx autoresearch/commands/run.ts --preset browser-reliability
```

**Step 1 — Engine 做前置检查 + 测量 Baseline**:
```
git status --porcelain    → 必须干净
运行 verify: npx tsx autoresearch/eval-browse.ts 2>&1 | tail -1
  → eval-browse.ts 逐个执行 59 个测试用例（真实 opencli 命令 + 真实网站）
  → 输出: SCORE=51/59
  → extractMetric() 提取: 51
记录: iteration=0, metric=51, status=baseline
```

**测试用例示例**（`browse-tasks.json`）:
```json
{
  "name": "extract-github-stars",
  "steps": [
    "opencli browser open https://github.com/browser-use/browser-use",
    "opencli browser eval \"document.querySelector('#repo-stars-counter-star')?.textContent?.trim()\""
  ],
  "judge": { "type": "matchesPattern", "pattern": "\\d" }
}
```

**Step 2 — 迭代 1: Engine 传上下文给 Agent**:

Engine 构建 prompt 传给 Claude Code：
```
Goal: Increase browser command pass rate to 59/59 (100%)
Metric (pass_count): 51 (best: 51)
Iteration: 1
Scope: src/browser/dom-snapshot.ts, dom-helpers.ts, base-page.ts, page.ts, cli.ts
Rules: Make ONE atomic change, read failing tests BEFORE modifying, DO NOT modify test files
```

**Step 3 — Agent 自主工作（Engine 不参与）**:
```
Agent 先跑一遍 eval-browse.ts 看哪些 task 失败
  → 发现 extract-github-stars 失败: selector 找不到元素
Agent 读 src/browser/dom-snapshot.ts
  → 发现 snapshot 等待超时过短
Agent 改 dom-snapshot.ts: 增加 waitForSelector 超时
Agent 返回: "Increase snapshot wait timeout from 3s to 8s for slow-loading pages"
```

**Step 4 — Engine commit + verify + guard + 决策**:
```
git add -- src/browser/dom-snapshot.ts ...    ← 只 add scope 内的文件
git commit -m "experiment(browser): Increase snapshot wait timeout from 3s to 8s"
运行 verify → SCORE=53/59 → 度量 53, delta = +2
运行 guard → npm run build → 成功
判定: 53 > 51 且 |+2| >= 1 且 guard pass → KEEP ✓
```

**Step 5 — 迭代 2: 假设 Agent 的修改没有效果**:
```
Engine 传新上下文: bestMetric=53, recentLog=[baseline 51, keep +2]
Agent 改了 dom-helpers.ts
verify → SCORE=53/59 → delta = 0
判定: 没有改善 → DISCARD ✗ → git revert HEAD --no-edit（回滚）
```

**Step N — 连续丢弃 6 次后，Engine 给 stuckHint**:
```
Engine 在 prompt 中追加:
  ## STUCK — Try a Different Approach
  Re-read ALL scope files from scratch. Try a completely different approach.
```

**最终产出**:

```
autoresearch-results.tsv            ← 每轮迭代记录（TSV 格式）
──────────────────────────────────────────────────────────────
iteration  commit   metric  delta  guard  status    description
0          abc1234  51      +0     pass   baseline  initial state — pass_count 51
1          ef01234  53      +2     pass   keep      Increase snapshot wait timeout...
2          -        53      +0     -      discard   Adjust retry logic in dom-helpers
3          gh56789  55      +2     pass   keep      Add fallback selector for...
...
N          xy98765  59      +4     pass   keep      Handle dynamic content loading...

autoresearch/results/browse-001.json ← 每次 verify 的详细结果
git history                          ← 保留的改进 commit（discard 的已被 revert）
```

**所有文件角色一览**:

| 文件 | 角色 | 由谁使用 |
|------|------|---------|
| `autoresearch/commands/run.ts` | 入口脚本 | 人手动运行 |
| `autoresearch/presets/browser-reliability.ts` | 配置（goal/scope/verify/guard） | Engine 读取 |
| `autoresearch/engine.ts` | 循环骨架 | 自动运行 |
| `autoresearch/config.ts` | 类型定义 + `extractMetric` | Engine 调用 |
| `autoresearch/logger.ts` | TSV 日志 | Engine 写入 |
| `autoresearch/eval-browse.ts` | 测试运行器（verify 命令） | Engine 通过 shell 执行 |
| `autoresearch/browse-tasks.json` | 59 个测试用例定义 | eval-browse.ts 读取 |
| `src/browser/dom-snapshot.ts` 等 | scope 文件（被优化的目标） | Agent 读取+修改 |
| `autoresearch-results.tsv` | 迭代度量日志 | Engine 写入, Agent 读取（作为历史） |

### 6.2 Testing Harness（测试线束）

**核心思想**: 四层测试在不同时机自动/手动触发，每层覆盖不同的验证目标，从编译检查到真实网站 E2E，形成完整的质量守护网。

#### 触发方式一览

| 测试层 | 触发时机 | 触发者 | 命令 |
|-------|---------|-------|------|
| Build + TypeCheck | 每次 push/PR | GitHub Actions CI (`ci.yml`) | `tsc --noEmit` + `npm run build` |
| Unit Tests | 每次 push/PR | GitHub Actions CI (`ci.yml`) | `vitest run --project unit --shard=N/2` |
| Adapter Tests | 每次 push/PR | GitHub Actions CI (`ci.yml`) | `npm run test:adapter` |
| Bun 兼容性 | 每次 push/PR | GitHub Actions CI (`ci.yml`) | `bun vitest run --project unit` |
| E2E Tests | push/PR 修改了浏览器相关路径时 | GitHub Actions CI (`e2e-headed.yml`) | `vitest run tests/e2e/` + 真实 Chrome |
| Smoke Tests | 每周一 08:00 UTC 定时 + 手动 | GitHub Actions 定时调度 (`ci.yml`) | `vitest run tests/smoke/` |
| 本地测试 | 开发者手动 | 开发者 | `npm test` / `npm run test:e2e` / `opencli doctor` |

#### 四层测试架构

```
Layer 1: Build + TypeCheck（~30s，快速门禁）
  → tsc --noEmit + npm run build
  → 跨 3 个 OS: ubuntu + macOS + Windows
  → 失败 → PR 直接标红，后续测试不用跑

Layer 2: Unit Tests（~1min，分 2 shard 并行）
  → 核心模块: browser, pipeline, registry, diagnostic, output, extension
  → PR: 仅 ubuntu（快速反馈）；push to main: 跨 3 OS
  → Bun 兼容性检查: 同一套 unit test 在 Bun 运行时下跑一遍

Layer 3: E2E Tests（~5-10min，真实 Chrome + 真实网站）
  → 子进程启动 opencli → 打开真实 Chrome → 访问真实网站 → 检查输出
  → Linux 用 xvfb-run 虚拟显示器
  → 路径过滤: 仅 extension/**, src/browser/**, src/daemon.ts 等修改时触发

Layer 4: Smoke Tests（~15min，定时调度）
  → 外部 API 可用性检查（hackernews/v2ex 等 API 还活着吗）
  → 全量 adapter 定义校验（调用 Validation Harness）
  → 命令注册完整性（17 个站点是否都在）
```

#### 具体例子：PR 修改了 `src/browser/dom-snapshot.ts`

假设开发者修改了浏览器快照逻辑，提了一个 PR 到 `main` 分支。

**自动触发链路**:

```
开发者提交 PR (修改 src/browser/dom-snapshot.ts)
  │
  ├─ ci.yml 自动触发:
  │   ├─ build job ────── tsc --noEmit + npm run build (3 OS)     ✓ 每次都跑
  │   ├─ unit-test job ── vitest --shard=1/2 + --shard=2/2        ✓ 每次都跑
  │   ├─ adapter-test ─── npm run test:adapter                     ✓ 每次都跑
  │   ├─ bun-test ─────── bun vitest run                           ✓ 每次都跑
  │   └─ smoke-test ───── ✗ 不触发（仅 schedule/dispatch）
  │
  └─ e2e-headed.yml 自动触发（因 paths 匹配 src/browser/**）:
      └─ e2e-headed job:
          ├─ Setup Chrome（安装真实 Chrome 浏览器）
          ├─ npm run build
          └─ xvfb-run vitest run tests/e2e/
              ├─ public-commands.test.ts ── apple-podcasts/hackernews/v2ex（纯 API）
              ├─ browser-public.test.ts ─── bilibili/zhihu/IMDb（真实浏览器）
              ├─ browser-auth.test.ts ───── 需登录的命令
              ├─ management.test.ts ─────── list/validate/help
              └─ output-formats.test.ts ── json/table/csv 格式
```

**E2E 实际执行过程**（以 B 站热门为例）:

```typescript
// tests/e2e/browser-public.test.ts
it('bilibili hot returns video entries', async () => {
  // 1. 启动子进程运行真实 CLI
  const result = await runCli(['bilibili', 'hot', '--limit', '5', '-f', 'json'], { timeout: 60_000 });
  // 内部实际执行: node dist/src/main.js bilibili hot --limit 5 -f json
  // → 启动真实 Chrome → 打开 bilibili.com → 通过 Browser Bridge 抓数据

  // 2. 环境问题优雅降级（不阻断 CI）
  if (isBrowserBridgeUnavailable(result)) {
    console.warn('skipped — Browser Bridge unavailable');
    return;
  }

  // 3. 验证输出结构
  const data = parseJsonOutput(result.stdout);
  expect(data.length).toBe(5);
  expect(data[0]).toHaveProperty('title');
});
```

**E2E 核心设计 — 优雅降级**:

```typescript
// 站点不稳定时 warn + pass，不让偶发问题阻断 CI
function expectDataOrSkip(data: any[] | null, label: string) {
  if (data === null) {
    console.warn(`${label}: skipped — likely bot detection or geo-blocking`);
    return;  // ← warn 但测试通过
  }
  expect(data.length).toBeGreaterThanOrEqual(1);
}

// 瞬态浏览器断连自动重试一次
async function runCliWithTransientRetry(args, timeout) {
  let result = await runCli(args, { timeout });
  if (result.code !== 0 && isTransientBrowserDetach(result)) {
    result = await runCli(args, { timeout });  // 重试
  }
  return result;
}
```

**E2E 路径过滤** (`e2e-headed.yml`): 如果 PR 只改了 `src/analysis.ts`（不在 paths 列表中），E2E 不会被触发，节省 CI 资源。触发路径包括：
- `extension/**` / `src/browser/**` / `src/daemon.ts` / `src/execution.ts`
- `src/interceptor.ts` / `tests/e2e/**` / `tests/smoke/**`

**Smoke Tests（本次 PR 不触发，每周一自动跑）**:

```typescript
// tests/smoke/api-health.test.ts
// 1. 外部 API 可用性
it('hackernews API is responsive', async () => {
  const { stdout, code } = await runCli(['hackernews', 'top', '--limit', '5', '-f', 'json']);
  expect(code).toBe(0);
  expect(data[0]).toHaveProperty('title');
  expect(data[0]).toHaveProperty('score');
});

// 2. 全量 adapter 定义校验（复用 Validation Harness）
it('all adapter definitions are valid', async () => {
  const { stdout, code } = await runCli(['validate']);
  expect(code).toBe(0);
  expect(stdout).toContain('PASS');
});

// 3. 命令注册完整性（17 个站点是否都在）
it('all expected sites are registered', async () => {
  const data = parseJsonOutput(stdout);
  const sites = new Set(data.map(d => d.site));
  for (const expected of ['hackernews','bilibili','zhihu','twitter',...]) {
    expect(sites.has(expected)).toBe(true);
  }
});
```

#### CI 策略细节

| 策略 | 实现 |
|------|------|
| PR 快速反馈 | 单元测试仅 ubuntu（不跨 OS） |
| main 全面覆盖 | push 后跨 3 OS + Bun 兼容 |
| E2E 按需触发 | paths 过滤，只在浏览器相关文件变更时跑 |
| Smoke 定时调度 | `cron: '0 8 * * 1'`，每周一 08:00 UTC |
| 并发控制 | `cancel-in-progress: true`，同分支新 push 取消旧 CI |
| 站点不稳定容忍 | E2E 中 warn + pass（不阻断 CI） |
| 瞬态重试 | `isTransientBrowserDetach()` 检测到断连后自动重试一次 |

#### 感知 → 触发 → 执行 → 反馈：完整链路

测试不是 OpenCLI 自己感知和运行的 — 整个流程由 **GitHub 平台** 通过 Webhook + GitHub Actions 机制驱动：

**第一步：感知** — 开发者 `git push` 到 GitHub，GitHub 服务器收到事件后扫描仓库根目录的 `.github/workflows/*.yml`。

**第二步：匹配** — 逐个检查每个 workflow 的 `on:` 触发条件：

```yaml
# ci.yml — 检查 on.pull_request.branches
on:
  pull_request:
    branches: [main, dev]    # ← PR 目标是 main → 匹配 ✓

# e2e-headed.yml — 还要检查 paths
on:
  pull_request:
    branches: [main, dev]    # ← PR 目标是 main → 匹配 ✓
    paths:
      - 'src/browser/**'     # ← 改了 dom-snapshot.ts → 匹配 ✓
      # 如果 PR 只改了 README.md，paths 不匹配 → E2E 不触发
```

**第三步：执行** — GitHub 从 Runner 池分配云端虚拟机（ubuntu/macOS/Windows），每台 VM 全新隔离，从克隆代码开始到测试完成后销毁：

```
ci.yml 分配的 VM（并行）:
  VM 1-3 (ubuntu/macos/windows) ── build: tsc --noEmit + npm run build
  VM 4-5 (ubuntu)               ── unit-test: vitest --shard=1/2 + 2/2
  VM 6 (ubuntu)                 ── adapter-test: npm run test:adapter
  VM 7 (ubuntu)                 ── bun-test: bun vitest run

e2e-headed.yml 分配的 VM（并行）:
  VM 8 (ubuntu) ── Setup Chrome → xvfb-run vitest run tests/e2e/
  VM 9 (macos)  ── Setup Chrome → vitest run tests/e2e/
```

**第四步：反馈** — 每个 job 执行完毕后结果回写到 PR 页面的 Checks 面板，如设置了 branch protection rule 则所有 required checks 通过才允许 Merge。

```
时间线: 开发者 git push → GitHub 感知 → 匹配 yml 规则 → 分配 VM → 执行测试 → 结果回写 PR
```

**职责分界**: Testing Harness 的运行依赖 GitHub Actions（GitHub 平台自带的 CI/CD 服务），OpenCLI 和 GitHub 各负责一半：

| OpenCLI 开发者负责（写在仓库里） | GitHub 平台负责（基础设施） |
|------|------|
| 写测试代码 (`tests/e2e/*.test.ts` 等) | 监听 git 事件 (push/PR/schedule) |
| 写 CI 配置 (`.github/workflows/*.yml`) | 读取 yml 并匹配触发规则 |
| 定义触发规则 (`on:` + `paths:` 过滤) | 分配云端 VM (ubuntu/macOS/Windows) |
| 定义执行步骤 (`steps:` npm ci → build → test) | 在 VM 上按 steps 执行、收集日志 |
| | 结果写回 PR 页面 Checks 面板 |
| | 执行完毕后销毁 VM |

PR 创建后测试是全自动触发的，开发者不需要点任何"运行测试"按钮。GitHub Actions 不是唯一选择 — GitLab CI (`.gitlab-ci.yml`)、Jenkins (`Jenkinsfile`)、CircleCI 等同类工具原理相同：仓库放配置文件 → 平台感知事件 → 按配置分配机器跑脚本。

#### Testing Harness 与其他 Harness 的配合

```
Validation Harness (5.4)
  └─ 被 Smoke Test 调用: opencli validate → PASS/FAIL
       → 静态校验不通过的适配器不会进入 E2E

AutoResearch (6.1)
  └─ guard 字段通常是 npm run build 或 npm test
       → Testing Harness 充当 AutoResearch 的"判分标准"

Verified Generation (4.5)
  └─ assessResult() 本质上是一次内嵌的 mini smoke test
       → 验证生成的适配器能否返回有效数据

Self-Repair (5.3)
  └─ 修复后重试命令 = 一次隐式的 E2E 验证
```

---

## 七、Harness 理念的统一框架

### 7.1 核心设计原则

| 原则 | 体现 |
|------|------|
| **命令即规范** | Self-Repair 不需要 spec 文件，命令本身是验证预言 |
| **最小权限** | Strategy Cascade 自动发现最低可行认证策略 |
| **安全边界前置** | Diagnostic 输出前脱敏；Self-Repair 限定修改 scope；AutoResearch 限定变更 scope |
| **渐进式降级** | 预算耗尽时按优先级砍信息；Cascade 探测失败后降级 |
| **机械可度量** | AutoResearch 要求度量值必须是数字，非数字无法进入循环 |
| **确定性优先** | 同命令同输出 schema；运行时零 LLM 消费；Explore 全程确定性规则 |
| **事务原子性** | 插件安装/更新使用 Transaction + rollback 机制 |
| **钩子隔离** | Hook handler 失败不阻塞主流程 |
| **globalThis 单例** | Registry、Hooks 通过 globalThis 解决 npm link 多拷贝问题 |
| **统一决策语言** | Verified Generation 中 EarlyHint 和 GenerateOutcome 共享类型体系 |
| **懒加载** | 命令模块按需 import()，CLI 管理命令按需 import()，极致冷启动 |

### 7.2 所有 Harness 的统一模式

```
约束（Constraint）→ 动作（Action）→ 度量（Metric）→ 决策（Decision）→ 闭环（Loop）
```

| Harness | 约束 | 动作 | 度量 | 决策 | 闭环 |
|---------|------|------|------|------|------|
| Verified Gen | PUBLIC/COOKIE + v1 合约 | explore→cascade→verify | assessResult 质量评估 | success/blocked/escalate | 含 1 次 itemPath 修复 |
| Cascade | 5 级策略列表 | 逐级 fetch 探测 | HTTP 响应状态 + hasData | 返回最佳策略 | 单次遍历 |
| Diagnostic | 大小预算 + 脱敏 | 收集浏览器上下文 | RepairContext 完整性 | 输出/降级丢弃 | 供 Agent 消费 |
| Self-Repair | sourcePath + 3 轮上限 | 诊断 + 修复适配器 | 重试是否成功 | keep/stop | 最多 3 轮 |
| AutoResearch | scope glob | AI 原子变更 + commit | verify 命令数值 | keep/discard | N 次迭代 |
| Validation | 命令定义 schema | 遍历注册表 | error/warning 计数 | PASS/FAIL | 单次 |
| Testing | 四层测试 | 运行测试套件 | 通过率 | 绿/红 | CI 持续运行 |
| Lifecycle Hook | try/catch 隔离 | 触发 handler | handler 返回值 | 继续/warn | 每次命令 |

---

## 八、值得借鉴的点

1. **结构化错误 + 可操作提示**: 每个错误类型都携带 `code`（机器读）、`hint`（人读修复建议）、`exitCode`（Unix 规范），使 AI Agent 和人类都能高效处理错误
2. **Diagnostic as API**: 将诊断信息视为给 AI Agent 的 API 契约（`RepairContext`），而非给人看的日志
3. **分层启动优化**: Ultra-fast path 让高频简单操作（--version、补全）不付代价
4. **策略自动发现**: CASCADE 机制免去人工标注认证策略的麻烦
5. **"命令即 Spec"**: 自修复不需要预先定义 spec 文件，极大降低维护成本
6. **事务化文件操作**: 插件安装/更新的 `beginReplaceDir` + `Transaction` 模式值得在任何需要原子文件替换的场景借鉴
7. **AutoResearch 循环**: 将 AI 改进代码的过程工程化为 constraint+metric+loop，可复制到其他 AI 自动化场景
8. **iframe re-fetch 绕过 SPA monkey-patching**: Explore 中用 `iframe.contentWindow.fetch` 获取原生 fetch，是对付 SPA 框架劫持网络层的精巧方案
9. **统一决策语言**: `generate-verified.ts` 中 EarlyHint 和 GenerateOutcome 共享同一套类型体系（StopReason / EscalationReason / Reusability），Agent 看到的是一条连续决策路径而非割裂的状态码
10. **globalThis 单例模式**: 用于解决 npm link / peerDependency 符号链接导致的模块多拷贝问题，registry 和 hooks 都采用此模式，值得在任何需要跨模块共享状态的 Node.js 项目中借鉴
11. **record 命令补充 explore**: explore 是自动化一次性快照，record 是人工操作+持续录制，两者互补覆盖不同场景
12. **能力路由分离**: `capabilityRouting.ts` 将"是否需要浏览器会话"的判定逻辑独立为纯函数，使 `execution.ts` 保持简洁

---

## 九、局限性

1. 浏览器适配器强依赖 Chrome 和 Browser Bridge 扩展，不支持 Firefox/Safari
2. Pipeline 引擎不支持并行步骤或条件分支
3. AutoResearch 的 `extractMetric()` 仅支持简单数字提取，复杂度量需自定义
4. 站点反爬/地域限制是运行时的主要不确定性来源，E2E 测试采用 warn+pass 策略作为妥协
5. Self-Repair 的 3 轮修复预算较保守，对于重大站点改版可能不够
6. Verified Generation v1 合约仅覆盖 PUBLIC/COOKIE + read-only JSON API，HEADER/INTERCEPT/UI 策略和写操作需要人工编写适配器
