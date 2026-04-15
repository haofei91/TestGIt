# OpenClaw 源码解读 —— Harness 模块

> 分析范围：聚焦 `src/agents/harness/` 核心实现，不做全库扫描
> 仓库：https://github.com/openclaw/openclaw
> 分支：main，commit 9b1b56aad1

---

## 1. 项目是什么

OpenClaw 是一个开源的 AI Agent 运行时框架，支持多渠道（QQ、Telegram、Discord、Signal 等）接入，提供插件化的 Agent 扩展机制、嵌入式 PI (Personal Intelligence) 运行器，以及完整的会话管理和安全审计能力。

## 2. Harness 在整个架构中的位置

```
用户请求
   ↓
Gateway / ACP Server
   ↓
selectAgentHarness()   ← Harness 选择层（registry + selection）
   ↓
┌─────────────────────────────┐
│   AgentHarness (接口抽象)     │
├─────────────────────────────┤
│ • pi        （内置 PI 引擎）  │
│ • acp       （插件提供的）    │
│ • 自定义插件 harness         │
└─────────────────────────────┘
   ↓
runAgentHarnessAttemptWithFallback()
   ↓
实际执行（PI 或插件 harness）
```

**核心定位**：Harness 是"运行时后端"的抽象接口，负责把"谁来跑 Agent"这件事解耦出来。它使得 PI（内置引擎）和第三方插件提供的运行时不耦合，用户可配置、可切换、可回退。

---

## 3. Harness 接口定义

文件：`src/agents/harness/types.ts`

```typescript
export type AgentHarness = {
  id: string;
  label: string;
  pluginId?: string;
  supports(ctx: AgentHarnessSupportContext): AgentHarnessSupport;
  runAttempt(params: AgentHarnessAttemptParams): Promise<AgentHarnessAttemptResult>;
  compact?(params: AgentHarnessCompactParams): Promise<AgentHarnessCompactResult | undefined>;
  reset?(params: AgentHarnessResetParams): Promise<void> | void;
  dispose?(): Promise<void> | void;
};
```

**接口契约解读**：

| 方法 | 必须 | 作用 |
|------|------|------|
| `id` | ✅ | 唯一标识，如 `"pi"`、`"acp"` |
| `label` | ✅ | 人类可读名称 |
| `supports(ctx)` | ✅ | 判断当前 runtime/provider/model 是否支持该 harness |
| `runAttempt(params)` | ✅ | **核心执行方法**，执行一次 Agent Run |
| `compact(params)` | ❌ | 可选：会话压缩/上下文精简 |
| `reset(params)` | ❌ | 可选：重置会话状态 |
| `dispose()` | ❌ | 可选：销毁 harness，释放资源 |

这是一个**命令模式**的接口抽象：外部只调用 `runAttempt()`，具体谁来跑、怎么跑，由各自的 harness 实现决定。

---

## 4. Registry —— 全局单例注册表

文件：`src/agents/harness/registry.ts`

```typescript
type AgentHarnessRegistryState = {
  harnesses: Map<string, RegisteredAgentHarness>;
};

function getAgentHarnessRegistryState(): AgentHarnessRegistryState {
  const globalState = globalThis as typeof globalThis & {
    [AGENT_HARNESS_REGISTRY_STATE]?: AgentHarnessRegistryState;
  };
  globalState[AGENT_HARNESS_REGISTRY_STATE] ??= {
    harnesses: new Map<string, RegisteredAgentHarness>(),
  };
  return globalState[AGENT_HARNESS_REGISTRY_STATE];
}
```

**设计亮点**：
- 使用 `globalThis` 符号作为 key，**跨 realm 共享状态**（Node.js 主进程和 worker 之间）
- 全局 Map：`Map<id, RegisteredAgentHarness>`，O(1) 查询
- 插件通过 `registerAgentHarness(harness, { ownerPluginId })` 注册自己
- 提供 `restoreRegisteredAgentHarnesses()` 支持序列化恢复（插件热重载场景）

**导出的核心 API**：

```typescript
registerAgentHarness(harness, options?)   // 注册
getAgentHarness(id)                        // 按 ID 查
listAgentHarnessIds()                      // 列出所有 ID
listRegisteredAgentHarnesses()            // 列出所有（含 ownerPluginId）
resetRegisteredAgentHarnessSessions(params) // 广播 reset 到所有 harness
disposeRegisteredAgentHarnesses()           // 广播 dispose 到所有 harness
```

---

## 5. Selection —— 选择与策略路由

文件：`src/agents/harness/selection.ts`

这是 harness 系统的**决策引擎**，负责"用哪个 harness"。

### 5.1 策略优先级

```
环境变量 OPENCLAW_AGENT_RUNTIME  >  配置 >  默认 "auto"
```

### 5.2 选择算法

```typescript
export function selectAgentHarness(params): AgentHarness {
  const policy = resolveAgentHarnessPolicy(params);

  // 硬编码：PI 不参与插件候选，它是 legacy 回退路径
  // 这样设计让 fallback: "none" 可以证明"只有插件 harness 在跑"
  const piHarness = createPiAgentHarness();

  if (policy.runtime === "pi") {
    return piHarness; // 强制用 PI
  }

  if (runtime !== "auto") {
    // 强制指定某个 harness ID
    const forced = pluginHarnesses.find(entry => entry.id === runtime);
    if (forced) return forced;
    if (policy.fallback === "none") throw new Error("...");
    return piHarness; // 指定的不存在，且禁止回退 → 报错
  }

  // === auto 模式 ===
  const supported = pluginHarnesses
    .map(harness => ({
      harness,
      support: harness.supports({ provider, modelId, requestedRuntime }),
    }))
    .filter(entry => entry.support.supported)
    .toSorted(compareHarnessSupport); // 按 priority 降序，再按 id 升序

  if (supported[0]) return supported[0].harness;
  if (policy.fallback === "none") throw new Error("...");
  return piHarness;
}
```

### 5.3 回退机制

```typescript
export async function runAgentHarnessAttemptWithFallback(params) {
  const harness = selectAgentHarness(params);

  if (harness.id === "pi") return harness.runAttempt(params);

  try {
    return await harness.runAttempt(params);
  } catch (error) {
    // 只有 auto 模式 + 非 none 回退才吃错误
    if (policy.runtime !== "auto" || policy.fallback === "none") throw error;
    log.warn(`${harness.label} failed; falling back to PI backend`, { error });
    return createPiAgentHarness().runAttempt(params); // 回退到 PI
  }
}
```

**设计思想**：插件 harness 优先，若插件失败则在 auto 模式下透明回退到 PI。`fallback: "none"` 可用于测试或严格隔离场景。

---

## 6. PI 内置 Harness

文件：`src/agents/harness/builtin-pi.ts`

```typescript
export function createPiAgentHarness(): AgentHarness {
  return {
    id: "pi",
    label: "PI embedded agent",
    supports: () => ({ supported: true, priority: 0 }), // 最低优先级
    runAttempt: runEmbeddedAttempt, // 直接复用 pi-embedded-runner
  };
}
```

PI 是默认的兜底 harness，`supports()` 永远返回 `supported: true`，但 `priority: 0` 最低，所以第三方 harness 总是优先于它。

---

## 7. 完整数据流时序

```
用户消息进入 ACP Server
    ↓
resolveAgentHarnessPolicy()  ← 读取 env + config 确定策略
    ↓
selectAgentHarness()          ← 按 runtime/fallback/priority 选 harness
    ↓
runAgentHarnessAttemptWithFallback()
    ├─→ harness.runAttempt()  ← 插件 harness 执行（可能失败）
    └─→ [失败且 auto 模式] → createPiAgentHarness().runAttempt()
    ↓
EmbeddedRunAttemptResult 返回
    ↓
maybeCompactAgentHarnessSession()  ← 可选的会话压缩
```

---

## 8. 值得借鉴的设计思想

### 8.1 插件化架构的经典套路
用 `Map<id, RegisteredAgentHarness>` 做注册中心，加上选择器模式，是插件化的事实标准（类似 VSCode 扩展、Webpack loader）。

### 8.2 回退链（Fallback Chain）
```
用户指定 → 插件候选 → (失败) → PI 兜底
```
比直接 panic 更好，给了用户"不想要 PI"的开关（`fallback: "none"`）。

### 8.3 globalThis 做单例跨 Realm 共享
在 Node.js 多进程/worker 场景下很有用，避免了模块级别的 `global` 变量问题。

### 8.4 策略对象解耦
把"用哪个 harness"和"怎么用"解耦，`AgentHarnessPolicy` 是纯值对象，不含副作用，方便测试和序列化。

---

## 9. 关键文件索引

| 文件 | 职责 |
|------|------|
| `types.ts` | `AgentHarness` 接口定义 |
| `registry.ts` | 全局 Map 注册表，支持 restore/dispose |
| `selection.ts` | 选择策略 + 回退逻辑 + policy 解析 |
| `builtin-pi.ts` | PI 内置 harness（最低优先级兜底） |
| `pi-embedded-runner/runtime.ts` | `EmbeddedAgentRuntime` 类型 + 环境变量解析 |
| `pi-embedded-runner/run/types.ts` | `EmbeddedRunAttemptParams/Result` 详细类型 |
