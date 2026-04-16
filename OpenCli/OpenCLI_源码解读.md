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

Explore 负责从目标网页自动发现 API endpoints 和能力。**不依赖任何 LLM**，完全基于确定性规则。

**9 步探索流程**:

```
Step 1: Navigate + Network Capture
  → page.startNetworkCapture()  // 开始拦截网络请求
  → page.goto(url)               // 导航到目标 URL
  → page.wait(waitSeconds)        // 等待页面初始加载（默认 3s）

Step 2: Auto-Scroll (懒加载触发)
  → page.autoScroll({ times: 3, delayMs: 1500 })
  → 滚动 3 次，每次间隔 1.5s，触发所有 lazy-loading 和无限滚动

Step 2.5: Interactive Fuzzing (仅 --auto 模式)
  → page.evaluate(INTERACT_FUZZ_JS)
  → 详见 4.2.2 自动点击行为

Step 3: Metadata Extraction
  → 从 document.title, meta[name=description] 等提取页面元数据

Step 4: Network Harvest
  → page.readNetworkCapture() ?? page.networkRequests(false)
  → 收集所有网络请求（URL, method, status, headers, responseBody）

Step 5: iframe Re-Fetch (补全缺失的 JSON body)
  → 对响应体为空但 Content-Type 为 JSON 的请求，通过 iframe.contentWindow.fetch 重新请求
  → 使用 iframe 而非直接 fetch，是为了绕过 SPA 框架对 window.fetch 的 monkey-patching

Step 6: Framework Detection
  → page.evaluate(FRAMEWORK_DETECT_JS)
  → 通过 DOM/window 标记检测前端框架（详见 4.2.1）

Step 7: Endpoint Analysis (确定性规则引擎)
  → analyzeEndpoints(networkEntries) — src/analysis.ts
  → 包含以下子步骤:
    a. isNoiseUrl(): 过滤追踪/分析类 URL (google analytics, sentry, etc.)
    b. urlToPattern(): URL 模式化 (数字→{id}, 16进制→{hex}, BV号→{bvid})
    c. findArrayPath(): 在 JSON 响应中递归查找最大对象数组 (最深 5 层)
    d. detectFieldRoles(): 字段名→语义角色映射 (title/name→title, img/avatar→image, etc.)
    e. classifyQueryParams(): 参数分类 (keyword→search, page/offset→pagination, limit→limit)
    f. detectAuthFromHeaders(): 从请求头检测认证方式 (bearer/csrf/signature)

Step 8: Capability Inference
  → inferCapabilitiesFromEndpoints(analyzed, stores, opts)
  → 从 URL 关键词推导 CLI 能力名称:
    hot/popular/trending → hot
    search/query → search
    recommend → recommend
    detail/info → detail
    ...

Step 9: Write Artifacts (5 个 JSON 文件)
  → writeExploreArtifacts(targetDir, result, analyzed, stores)
  → 输出到 .opencli/explore/<site>/:
    manifest.json     — 页面元数据 + 框架信息
    endpoints.json    — 所有发现的 API endpoint 及其分析结果
    capabilities.json — 推导出的 CLI 能力列表
    auth.json         — 检测到的认证方式
    stores.json       — 发现的数据存储 (localStorage/cookie 等)
```

**设计要点**:
- 整个 Explore 过程是**一次性的浏览器会话**，从打开页面到写入 JSON 文件一气呵成
- 所有分析规则都是**硬编码的确定性规则**（正则 + 关键词映射），不依赖 LLM
- iframe re-fetch 是一个精巧的设计：SPA 应用通常会 monkey-patch `window.fetch`，但 iframe 中的 `contentWindow.fetch` 是原生的，能拿到真实的响应体

#### 4.2.1 Endpoint 分析引擎源码

`src/analysis.ts`（180 行）— 共享 API 分析助手，同时被 `explore.ts` 和 `record.ts` 使用：

```typescript
// URL 模式化 — 将具体 ID 替换为占位符
export function urlToPattern(url: string): string {
  const pathNorm = p.pathname
    .replace(/\/\d+/g, '/{id}')              // 数字 ID → {id}
    .replace(/\/[0-9a-fA-F]{8,}/g, '/{hex}') // 16 进制 hash → {hex}
    .replace(/\/BV[a-zA-Z0-9]{10}/g, '/{bvid}'); // B 站 BV 号 → {bvid}
  // 过滤掉 VOLATILE_PARAMS (时间戳、随机数等)
}

// 在 JSON 中递归查找最大对象数组 (最深 5 层)
export function findArrayPath(obj: unknown, depth = 0): ArrayDiscovery | null {
  if (depth > 5) return null;
  // 找到长度 >= 2 且元素为对象的数组 → 返回 { path, items }
  // 多个候选时选最大的
}

// 字段名 → 语义角色映射
export function detectFieldRoles(sampleFields: string[]): Record<string, string> {
  // 使用 FIELD_ROLES 常量表: { title: ['title','name','headline'], image: ['img','avatar','cover'], ... }
}

// URL 关键词 → CLI 能力名推导
export function inferCapabilityName(url: string, goal?: string): string {
  // hot/popular/trending → 'hot'
  // search/query → 'search'
  // feed/timeline/dynamic → 'feed'
  // comment/reply → 'comments'
  // favorite/collect/bookmark → 'favorite'
  // 无匹配 → 取 URL 最后一段路径
}

// 认证方式检测
export function detectAuthFromHeaders(headers?: Record<string, string>): string[] {
  // authorization → 'bearer'
  // x-csrf/x-xsrf → 'csrf'
  // x-s/x-t/x-s-common → 'signature' (小红书等签名机制)
}
export function detectAuthFromContent(url: string, body: unknown): string[] {
  // sign/w_rid/token → 'signature' (B 站 wbi 签名)
  // bearer/access_token → 'bearer'
}

// 噪声 URL 过滤
export function isNoiseUrl(url: string): boolean {
  // 过滤: track/log/analytics/beacon/pixel/ping/heartbeat/keep-alive
}

// 查询参数分类
export function classifyQueryParams(url: string): {
  hasSearch: boolean;     // keyword/q/query 等 → true
  hasPagination: boolean; // page/offset/cursor 等 → true
  hasLimit: boolean;      // limit/pageSize/count 等 → true
}

// 策略推导
export function inferStrategy(authIndicators: string[]): string {
  // signature → 'intercept'
  // bearer/csrf → 'header'
  // 其他 → 'cookie'
}
```

**框架检测脚本** (`src/scripts/framework.ts`, 40 行):

```typescript
// 注入到页面上下文执行，通过 DOM/window 标记检测前端框架
export function detectFramework() {
  const app = document.querySelector('#app') as VueAppEl | null;
  const w = window as FrameworkWindow;
  return {
    vue3:  !!(app && app.__vue_app__),
    vue2:  !!(app && app.__vue__),
    react: !!w.__REACT_DEVTOOLS_GLOBAL_HOOK__ || !!document.querySelector('[data-reactroot]'),
    nextjs: !!w.__NEXT_DATA__,
    nuxt:  !!w.__NUXT__,
    pinia: !!(app?.__vue_app__?.config?.globalProperties?.$pinia), // Vue 3 状态管理
    vuex:  !!(app?.__vue_app__?.config?.globalProperties?.$store), // Vue 2/3 状态管理
  };
}
```

#### 4.2.2 自动点击行为与安全约束

Explore 阶段对页面元素的点击操作**默认关闭**，仅在显式指定选项时启用。共有三种模式：

**模式一：默认模式（仅滚动）**
```bash
opencli explore <url>
```
- 只执行 `autoScroll`（3 次滚动）
- **不点击任何元素**
- 适合大多数 SPA 页面，滚动即可触发 lazy-loading API 请求

**模式二：`--auto` 盲目模糊点击**
```bash
opencli explore <url> --auto
```
- 执行 `INTERACT_FUZZ_JS` 脚本（`src/scripts/interact.ts`）
- 选择器范围: `button, [role="button"], [role="tab"], .tab, .btn, a[href="javascript:void(0)"], a[href="#"]`
- **安全约束**:
  - 最多点击 **15 个元素**（`.slice(0, 15)`）
  - 每次点击间隔 **300ms**（`await sleep(300)`）
  - 只点击**可见元素**（`rect.width > 0 && rect.height > 0`）
  - 只点击不会导航离开页面的元素（`a[href="javascript:void(0)"]` 和 `a[href="#"]`，排除真实链接）
  - 使用 `dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }))` 而非 `el.click()`，给页面框架处理冒泡的机会

**完整 fuzzing 脚本** (`src/scripts/interact.ts`, 23 行):

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
        await sleep(300); // 等 300ms 让网络请求触发
      }
    } catch {} // 单个元素失败不影响后续
  }
  return clicked;
}
```

**模式三：`--click` 精确标签点击**
```bash
opencli explore <url> --click "热门,推荐,排行"
```
- 根据提供的标签文本，精确定位并点击匹配的 Tab/Button
- 在盲目 fuzzing 之前执行
- 适合需要切换特定 Tab 才能触发 API 的场景

**点击的目的**: 不是为了"浏览页面"，而是为了**触发更多 API 请求**。许多 SPA 页面的数据接口只有在点击特定 Tab 或按钮后才会被调用。通过有限度的点击，Explore 能发现更多隐藏的 API endpoints。

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

**验证阶段**:

将 Cascade 确定的策略写入候选 YAML，然后通过 Pipeline 引擎实际执行：

```
verifyCandidate():
  1. 将 bestStrategy 注入候选 pipeline
  2. executePipeline(candidate) — 使用 src/pipeline/ 引擎执行
  3. assessResult(output):
     - 非数组 → 失败 (non-array-result)
     - 空数组 → 失败 (empty-result)
     - 字段过少 → 失败 (sparse-fields)
     - 通过 → success
```

**有界修复（Bounded Repair）**:

首次验证失败且原因为 `empty-result` 时，系统会尝试**一次**自动修复：

```
Verify 失败 (empty-result)
  → 检查是否有备选 itemPath
  → 替换 pipeline 中的 select 路径
  → 重新执行 pipeline
  → 再次 assessResult()
  → 成功 → 继续 Persist
  → 仍然失败 → 输出 needs-human-check
```

修复预算严格限定为 **1 次 itemPath 替换**，不做更复杂的修改。这是 v1 合约的设计约束。

---

### 4.6 Phase 6: Persist & Register — 产物持久化

验证通过后，将候选转化为可执行适配器：

```
writeAdapter():
  1. 将 YAML pipeline 编译为 .js 文件
  2. 写入 ~/.opencli/clis/<site>/<name>.js
  3. 生成 .meta.json sidecar 元数据文件:
     {
       generated_at: ISO timestamp,
       source_url: 原始 URL,
       strategy: 使用的认证策略,
       explore_dir: explore 产物路径,
       verified: true/false,
       repair_attempted: true/false
     }
  4. registerCommand() → 注册到内存中的 registry
  5. 下次启动时，discoverClis(USER) 会从 ~/.opencli/clis/ 发现并加载
```

**`--no-register` 选项**: 只验证不注册，用于测试生成质量。

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

**实现**: `autoresearch/engine.ts` + `autoresearch/config.ts`

**8 阶段迭代循环**:

```
Phase 0: Precondition — git clean, 无锁, 非 detached HEAD
Phase 1: Review     — 读取 scope 文件 + 日志 + git 历史
Phase 2: Ideate     — 选择下一个变更方向
Phase 3: Modify     — 一次原子变更（委托给回调）
Phase 4: Commit     — git add + commit（仅 scope 内文件）
Phase 5: Verify     — 运行验证命令，提取度量值
Phase 5.5: Guard    — 可选回归检查（如 npm run build）
Phase 6: Decide     — keep（度量改善且通过 guard）/ discard（回滚）/ crash
Phase 7: Log        — TSV 日志记录
Phase 8: Repeat     — 回到 Phase 1
```

**Preset 示例** (browser-reliability):

```typescript
{
  goal: 'Increase browser command pass rate to 59/59 (100%)',
  scope: ['src/browser/dom-snapshot.ts', 'src/browser/dom-helpers.ts', ...],
  metric: 'pass_count',
  direction: 'higher',
  verify: 'npx tsx autoresearch/eval-browse.ts 2>&1 | tail -1',
  guard: 'npm run build',
  minDelta: 1,
}
```

**卡住检测**: 当连续丢弃次数 > 5 时，提供递进式提示：重读 scope → 组合成功策略 → 反向思考 → 激进架构变更 → 简化。

**与 Self-Repair 的区别**: Self-Repair 是"命令失败时的应急修复"（被动触发、最多 3 轮、修改单个适配器）；AutoResearch 是"主动的系统性改进"（主动触发、无限迭代、可修改 scope 内任何文件）。

### 6.2 Testing Harness（测试线束）

四层测试架构：

```
Unit Tests (src/**/*.test.ts)
  → 核心模块：browser, pipeline, registry, diagnostic, output
  → 运行: npm test

Adapter Tests (clis/**/*.test.{ts,js})  
  → 各站点适配器逻辑测试
  → 运行: npm run test:adapter

E2E Tests (tests/e2e/*.test.ts)
  → 子进程运行真实 CLI，覆盖公开/浏览器/认证/管理/输出格式
  → 站点不稳定时 warn + pass（不让偶发网络问题阻断 CI）

Smoke Tests (tests/smoke/*.test.ts)
  → 外部 API 可用性 + 全量 adapter 校验 + 命令注册完整性
```

**CI 策略**: 单元测试分 2 shard 并行；E2E 使用真实 Chrome + xvfb-run；Smoke 定时调度运行。

**Testing 与其他 Harness 的配合**:
- Validation Harness（5.4）是测试的前置门禁 — 静态校验不通过的命令不会进入 E2E
- AutoResearch（6.1）的 `guard` 字段通常就是 `npm run build` 或 `npm test`
- Verified Generation（4.5）的 `assessResult()` 本质上是一次内嵌的 smoke test

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
