# OpenCLI 源码解读

> 仓库: https://github.com/jackwener/OpenCLI  
> 版本: v1.7.4  
> 分析日期: 2026-04-16  
> 本地路径: `~/Documents/coding/github/OpenCLI`

---

## 一、项目概述

**OpenCLI** 是一个将任意网站或 Electron 桌面应用变成命令行工具的框架。核心理念是 "Make any website or Electron App your CLI"，由 AI 驱动 API 发现与适配器生成。

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

使用 `globalThis.__opencli_registry__` 确保跨模块实例的单例注册表：

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

---

## 四、Harness 理念深度分析

OpenCLI 项目深度践行了 **Test Harness**、**Self-Repair Harness**、**AutoResearch Harness** 三层 Harness 架构。"Harness" 在这里不仅是测试框架的意思，更是一种 **"约束+度量+闭环"** 的工程理念。

### 4.1 理念一：Diagnostic Harness（诊断线束）

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

```
命令失败 → execution.ts 捕获错误
  → isDiagnosticEnabled() 检查环境变量
  → collectDiagnostic() 并行收集页面状态
  → emitDiagnostic() 脱敏+限制大小+输出 stderr
```

### 4.2 理念二：Self-Repair Harness（自修复线束）

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
    1. OPENCLI_DIAGNOSTIC=1 重新执行，收集 RepairContext
    2. 从 RepairContext.adapter.sourcePath 读取适配器源码
    3. 分析: error code + DOM snapshot + network requests → 定位根因
    4. 编辑适配器文件
    5. 重试命令
    6. 仍然失败 → 重复（最多 3 轮）
    7. 3 轮耗尽 → 上报失败
```

**作用域约束**: 只允许修改 `RepairContext.adapter.sourcePath` 指向的文件。绝不修改 `src/`、`extension/`、`tests/`、`package.json`。

**非修复故障识别**: AUTH_REQUIRED / BROWSER_CONNECT / CAPTCHA / Rate Limit → 直接停止，不修改代码。

### 4.3 理念三：Verified Generation Harness（验证生成线束）

**核心思想**: AI 生成的适配器不是"生成即完成"，而必须经过 **探索→合成→候选→策略探测→验证→修复→注册** 的完整 harness 流程。

**实现**: `src/generate-verified.ts`

**Pipeline**:

```
Phase 1: Explore (exploreUrl)
  → 打开页面，拦截网络请求，发现 JSON API endpoints
  → 输出: endpoints[], capabilities[]

Phase 2: Synthesize (synthesizeFromExplore)
  → 从 explore 结果生成候选适配器 YAML
  → 输出: candidates[]

Phase 3: Select (selectCandidate)
  → 根据 goal 选择最佳候选

Phase 4: Cascade Probe (cascadeProbe)
  → 在单个浏览器会话内探测最简可行策略
  → PUBLIC → COOKIE → HEADER 逐级降级

Phase 5: Verify (verifyCandidate)
  → 执行候选 pipeline，检查结果质量
  → assessResult(): 非数组 / 空数组 / 字段过少 → 失败

Phase 5.5: Bounded Repair
  → 首次验证失败且原因为 empty-result → 替换 itemPath 后重试一次

Phase 6: Persist
  → 成功 → 写入 ~/.opencli/clis/<site>/<name>.js + .meta.json
  → 注册到 registry
```

**关键设计: 三级输出分类**:

| 状态 | 含义 | 可复用性 |
|------|------|---------|
| `success` | 验证通过 | `verified-artifact` |
| `needs-human-check` | 需人工检查 | `unverified-candidate` |
| `blocked` | 无法继续 | `not-reusable` |

**Early Hint 机制**: 在 explore/synthesize/cascade 每个阶段通过 `onEarlyHint` 回调提前通知调用方，实现成本门控（不适合的 URL 尽早退出，减少不必要的浏览器会话开销）。

### 4.4 理念四：Strategy Cascade Harness（策略级联线束）

**核心思想**: 自动发现最小权限策略，不需要人工指定。

**实现**: `src/cascade.ts`

```
PUBLIC (直接 fetch, 无 credentials)
  → 成功 → 返回 PUBLIC
  → 失败 ↓
COOKIE (fetch with credentials: 'include')
  → 成功 → 返回 COOKIE
  → 失败 ↓
HEADER (fetch + 提取 CSRF token)
  → 成功 → 返回 HEADER
  → 失败 ↓
默认回退 → COOKIE (confidence: 0.3)
```

置信度评分: 越简单的策略成功时置信度越高 (1.0 - index * 0.1)。

### 4.5 理念五：AutoResearch Harness（自动化研究线束）

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

### 4.6 理念六：Testing Harness（测试线束）

三层测试架构：

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

### 4.7 理念七：Lifecycle Hook Harness（生命周期钩子线束）

**实现**: `src/hooks.ts`

三个生命周期钩子点：

```
onStartup       → 所有命令和插件发现完成后
onBeforeExecute  → 每次命令执行前
onAfterExecute   → 每次命令执行后（携带结果/错误）
```

**隔离保证**: 每个 handler 用 try/catch 包裹，一个钩子失败不会阻塞命令执行。

### 4.8 理念八：Validation Harness（校验线束）

**实现**: `src/validate.ts` + `src/verify.ts`

双层校验：

1. **静态校验** (`validate`): 检查注册表中所有命令的定义完整性
   - description 是否存在
   - browser 命令是否有 domain
   - pipeline 步骤名是否合法
   - func/pipeline 是否至少有一个
   - 参数名是否重复

2. **运行时验证** (`verify`): 静态校验 + 可选的 smoke test（使用 vitest 运行 `tests/smoke/`）

---

## 五、设计思想总结

### 5.1 核心设计原则

| 原则 | 体现 |
|------|------|
| **命令即规范** | Self-Repair 不需要 spec 文件，命令本身是验证预言 |
| **最小权限** | Strategy Cascade 自动发现最低可行认证策略 |
| **安全边界前置** | Diagnostic 输出前脱敏；Self-Repair 限定修改 scope；AutoResearch 限定变更 scope |
| **渐进式降级** | 预算耗尽时按优先级砍信息；Cascade 探测失败后降级 |
| **机械可度量** | AutoResearch 要求度量值必须是数字，非数字无法进入循环 |
| **确定性优先** | 同命令同输出 schema；运行时零 LLM 消费 |
| **事务原子性** | 插件安装/更新使用 Transaction + rollback 机制 |
| **钩子隔离** | Hook handler 失败不阻塞主流程 |

### 5.2 Harness 理念的统一框架

所有 Harness 都遵循一个统一的模式：

```
约束（Constraint）→ 动作（Action）→ 度量（Metric）→ 决策（Decision）→ 闭环（Loop）
```

| Harness | 约束 | 动作 | 度量 | 决策 | 闭环 |
|---------|------|------|------|------|------|
| Diagnostic | 大小预算+脱敏 | 收集上下文 | RepairContext | 输出/丢弃 | 供 Agent 消费 |
| Self-Repair | sourcePath+3轮上限 | 诊断+修复 | 重试是否成功 | keep/stop | 最多3轮 |
| Verified Gen | PUBLIC/COOKIE 限定 | explore→verify | 结果质量评估 | success/blocked/escalate | 含1次修复重试 |
| Cascade | 策略列表 | 逐级探测 | HTTP 响应状态 | 返回最佳策略 | 单次遍历 |
| AutoResearch | scope glob | AI 修改+commit | verify 命令 | keep/discard | N 次迭代 |
| Validation | 命令定义 schema | 遍历注册表 | error/warning 计数 | PASS/FAIL | 单次 |

---

## 六、值得借鉴的点

1. **结构化错误 + 可操作提示**: 每个错误类型都携带 `code`（机器读）、`hint`（人读修复建议）、`exitCode`（Unix 规范），使 AI Agent 和人类都能高效处理错误
2. **Diagnostic as API**: 将诊断信息视为给 AI Agent 的 API 契约（`RepairContext`），而非给人看的日志
3. **分层启动优化**: Ultra-fast path 让高频简单操作（--version、补全）不付代价
4. **策略自动发现**: CASCADE 机制免去人工标注认证策略的麻烦
5. **"命令即 Spec"**: 自修复不需要预先定义 spec 文件，极大降低维护成本
6. **事务化文件操作**: 插件安装/更新的 `beginReplaceDir` + `Transaction` 模式值得在任何需要原子文件替换的场景借鉴
7. **AutoResearch 循环**: 将 AI 改进代码的过程工程化为 constraint+metric+loop，可复制到其他 AI 自动化场景

---

## 七、局限性

1. 浏览器适配器强依赖 Chrome 和 Browser Bridge 扩展，不支持 Firefox/Safari
2. Pipeline 引擎不支持并行步骤或条件分支
3. AutoResearch 的 `extractMetric()` 仅支持简单数字提取，复杂度量需自定义
4. 站点反爬/地域限制是运行时的主要不确定性来源，E2E 测试采用 warn+pass 策略作为妥协
5. Self-Repair 的 3 轮修复预算较保守，对于重大站点改版可能不够
