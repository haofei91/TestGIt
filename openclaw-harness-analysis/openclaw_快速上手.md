# OpenClaw 快速上手 —— Harness 模块

> 目标：理解 Harness 模块，并在自己的插件或配置中实际使用它

---

## 1. 环境要求

- **Node.js** ≥ 20.x（大量使用 `globalThis` 符号和 ESM）
- **pnpm** ≥ 9.x（本项目使用 pnpm workspace）
- 推荐 **macOS / Linux**，部分功能在 Windows 上受限

```bash
# 检查版本
node --version   # 需要 >= 20
pnpm --version   # 需要 >= 9
```

---

## 2. 安装步骤

```bash
# 克隆仓库
git clone git@github.com:openclaw/openclaw.git
cd openclaw

# 安装依赖
pnpm install

# 构建（harness 模块在 src/agents/harness/，由 TypeScript 源码直接使用）
pnpm build
```

---

## 3. Harness 配置说明

### 3.1 运行时选择（OPENCLAW_AGENT_RUNTIME）

```bash
# 可选值：
#   "pi"    → 强制使用内置 PI 引擎
#   "auto"  → 自动选择（默认，优先插件 harness，失败后回退 PI）
#   <任意>  → 强制指定某个 harness ID

export OPENCLAW_AGENT_RUNTIME=auto      # 默认
export OPENCLAW_AGENT_RUNTIME=pi       # 强制 PI
export OPENCLAW_AGENT_RUNTIME=my-plugin # 强制某个插件 harness
```

### 3.2 回退策略（OPENCLAW_AGENT_HARNESS_FALLBACK）

```bash
# 可选值：
#   "pi"   → 失败时回退到 PI（默认）
#   "none" → 禁止回退，插件 harness 失败直接抛错

export OPENCLAW_AGENT_HARNESS_FALLBACK=pi   # 默认
export OPENCLAW_AGENT_HARNESS_FALLBACK=none # 严格模式，用于测试
```

### 3.3 配置文件（openclaw.json）

```json
{
  "agents": {
    "defaults": {
      "embeddedHarness": {
        "runtime": "auto",
        "fallback": "pi"
      }
    },
    "entries": [
      {
        "id": "my-agent",
        "embeddedHarness": {
          "runtime": "pi",
          "fallback": "none"
        }
      }
    ]
  }
}
```

---

## 4. 开发自己的 Harness 插件

### 4.1 实现 AgentHarness 接口

```typescript
// my-harness.ts
import type { AgentHarness, AgentHarnessSupportContext } from "./types.js";

export function createMyAgentHarness(): AgentHarness {
  return {
    id: "my-harness",
    label: "My Custom Agent Harness",
    pluginId: "my-plugin",

    supports(ctx: AgentHarnessSupportContext) {
      // 只在特定 provider/model 下启用
      if (ctx.provider === "openai" && ctx.modelId?.startsWith("gpt-4")) {
        return { supported: true, priority: 10 }; // 优先级高于 PI
      }
      return { supported: false, reason: "Only GPT-4 models supported" };
    },

    async runAttempt(params) {
      // params 是 EmbeddedRunAttemptParams
      // 返回 EmbeddedRunAttemptResult
      // ... 你的运行逻辑 ...
    },

    async compact(params) {
      // 可选：会话压缩逻辑
    },

    reset(params) {
      // 可选：会话重置
    },

    dispose() {
      // 可选：清理资源
    },
  };
}
```

### 4.2 注册 Harness

```typescript
// 在插件入口中
import { registerAgentHarness } from "./agents/harness/index.js";
import { createMyAgentHarness } from "./my-harness.js";

registerAgentHarness(createMyAgentHarness(), { ownerPluginId: "my-plugin" });
```

### 4.3 运行时验证

```bash
# 启动 OpenClaw，查看已注册的 harness
openclaw gateway start
# 或查看日志中的 "agents/harness" subsystem
```

---

## 5. 测试 Harness 选择逻辑

```typescript
import { selectAgentHarness } from "./agents/harness/index.js";

const harness = selectAgentHarness({
  provider: "openai",
  modelId: "gpt-4-turbo",
  config: yourConfig,
});

console.log(harness.id);     // "my-harness" 或 "pi"
console.log(harness.label);  // "My Custom Agent Harness" 或 "PI embedded agent"
```

---

## 6. 测试回退机制

```typescript
import { runAgentHarnessAttemptWithFallback } from "./agents/harness/index.js";

// 正常情况：优先用插件 harness
const result = await runAgentHarnessAttemptWithFallback(params);

// 如果插件 harness 抛错，且 fallback=pi（默认），会自动回退到 PI
// 如果 fallback=none，则抛错上浮
```

---

## 7. 常见问题

### Q: 为什么我的插件 harness 没有被选中？

1. 检查 `supports()` 是否返回 `supported: true`
2. 检查 `priority` 是否高于其他候选（PI 的 priority 是 0）
3. 检查 `OPENCLAW_AGENT_RUNTIME` 是否被设为 `"pi"` 或 `"none"`

### Q: 想强制只用插件 harness，不要 PI 兜底怎么办？

```bash
export OPENCLAW_AGENT_HARNESS_FALLBACK=none
```
这样插件 harness 失败时会直接抛错，而不是回退到 PI。

### Q: PI harness 是什么？

PI (Personal Intelligence) 是 OpenClaw 内置的嵌入式 Agent 运行器，基于 `pi-agent-core` 和 `pi-ai` 库。它是所有第三方 harness 的最低优先级兜底。

### Q: 如何查看所有已注册的 harness？

```typescript
import { listAgentHarnessIds } from "./agents/harness/index.js";
console.log(listAgentHarnessIds()); // ["pi", "acp", ...]
```

---

## 8. 下一步建议

1. **读源码**：先读 `src/agents/harness/types.ts` 接口定义，再读 `selection.ts` 选择逻辑
2. **读 PI runner**：深入 `src/agents/pi-embedded-runner/run/attempt.ts` 理解 `runAttempt` 的完整实现
3. **看实际插件**：参考 `extensions/acpx/` 下的某个插件，看它如何注册和使用 harness
4. **跑测试**：`pnpm test src/agents/harness/` 有完整的 registry 和 selection 测试用例
