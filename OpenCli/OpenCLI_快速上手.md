# OpenCLI 快速上手指南

> 仓库: https://github.com/jackwener/OpenCLI  
> 版本: v1.7.4  
> 本地路径: `~/Documents/coding/github/OpenCLI`

---

## 一、环境要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| Node.js | >= 21.0.0 | 推荐使用最新 LTS |
| Chrome / Chromium | 最新稳定版 | 浏览器命令需要 |
| Browser Bridge 扩展 | 与 CLI 版本匹配 | 从 GitHub Releases 下载 |
| git | 任意 | 插件安装需要 |

---

## 二、安装

### 方式 1: npm 全局安装

```bash
npm install -g @jackwener/opencli
```

### 方式 2: 从源码安装（开发模式）

```bash
git clone https://github.com/jackwener/OpenCLI.git
cd OpenCLI
npm ci
npm run build
```

开发模式运行：

```bash
npx tsx src/main.ts <command>
# 或
npm run dev -- <command>
```

### 安装 Browser Bridge 扩展

1. 从 [GitHub Releases](https://github.com/jackwener/opencli/releases) 下载扩展包
2. 打开 `chrome://extensions/`，启用"开发者模式"
3. 点击"加载已解压的扩展程序"，选择扩展目录

### 验证安装

```bash
opencli --version          # 输出版本号
opencli doctor             # 诊断浏览器连接状态
opencli doctor --live      # 包含实际连通性测试
```

---

## 三、基本用法

### 3.1 查看可用命令

```bash
opencli list               # 列出所有命令
opencli list --site zhihu  # 列出特定站点的命令
```

### 3.2 运行公开 API 命令（不需要浏览器）

```bash
opencli hackernews top --limit 5
opencli v2ex hot
opencli wikipedia search "OpenAI"
```

### 3.3 运行浏览器命令（需要 Chrome + 扩展）

```bash
opencli zhihu hot --limit 10
opencli bilibili trending
opencli twitter trending
```

### 3.4 输出格式

```bash
opencli hackernews top -f json     # JSON 格式
opencli hackernews top -f yaml     # YAML 格式
opencli hackernews top -f csv      # CSV 格式
opencli hackernews top -f md       # Markdown 表格
opencli hackernews top -f table    # 终端表格（默认）
```

---

## 四、核心命令

### 4.1 命令管理

```bash
opencli validate                   # 校验所有适配器定义
opencli validate --target zhihu    # 校验特定站点
opencli verify                     # 校验 + 可选 smoke test
opencli verify --smoke             # 校验 + 运行 smoke test
```

### 4.2 浏览器诊断

```bash
opencli doctor                     # 基本诊断
opencli doctor --live              # 包含实际连通性测试
opencli doctor --sessions          # 显示活跃会话
opencli daemon stop                # 停止 daemon
```

### 4.3 AI 驱动的适配器生成

```bash
# 探索网站 API
opencli explore https://example.com

# 从探索结果合成候选适配器
opencli synthesize <explore-dir>

# 一键探索+合成+验证
opencli generate https://example.com --goal "获取热门帖子"
```

### 4.4 插件管理

```bash
opencli plugin install github:user/repo     # 安装 Git 插件
opencli plugin install /path/to/local       # 安装本地插件
opencli plugin list                         # 列出已安装插件
opencli plugin update <name>                # 更新插件
opencli plugin uninstall <name>             # 卸载插件
```

---

## 五、开发适配器

### 5.1 TypeScript 适配器（推荐）

在 `clis/<site>/` 目录下创建 JS 文件：

```javascript
// clis/example/hot.js
import { cli, Strategy } from '@jackwener/opencli/registry';

cli({
  site: 'example',
  name: 'hot',
  description: 'Get hot posts from Example',
  domain: 'example.com',
  strategy: Strategy.COOKIE,
  browser: true,
  args: [
    { name: 'limit', type: 'int', default: 20, help: 'Max items to return' },
  ],
  columns: ['title', 'author', 'score'],
  async func(page, { limit }) {
    await page.goto('https://example.com/hot');
    const snapshot = await page.snapshot();
    // 解析 snapshot 提取数据...
    return items.slice(0, limit);
  },
});
```

### 5.2 Pipeline 适配器（声明式）

```javascript
import { cli, Strategy } from '@jackwener/opencli/registry';

cli({
  site: 'example',
  name: 'top',
  description: 'Get top posts via API',
  strategy: Strategy.PUBLIC,
  browser: false,
  columns: ['title', 'url', 'score'],
  pipeline: [
    { fetch: 'https://api.example.com/top?limit={{limit}}' },
    { select: 'data.items' },
    { map: { title: '$.title', url: '$.link', score: '$.points' } },
    { limit: '{{limit}}' },
  ],
  args: [
    { name: 'limit', type: 'int', default: 20 },
  ],
});
```

### 5.3 认证策略选择

| 策略 | 何时使用 | 示例 |
|------|---------|------|
| `PUBLIC` | 无需认证的公开 API | HackerNews, BBC |
| `COOKIE` | 需要 Chrome 登录态 | Bilibili, Zhihu |
| `HEADER` | 需要自定义头（CSRF） | Twitter 部分 API |
| `INTERCEPT` | 需要拦截网络请求 | Twitter GraphQL |
| `UI` | 需要 DOM 交互 | 桌面应用写操作 |

---

## 六、测试

```bash
# 单元测试 + adapter 测试（日常开发）
npm test

# 只跑 adapter 测试
npm run test:adapter

# E2E 测试（需要 Chrome + 扩展）
npx vitest run tests/e2e/

# Smoke 测试
npx vitest run tests/smoke/

# 单个文件
npx vitest run src/pipeline/executor.test.ts

# Watch 模式
npx vitest src/
```

---

## 七、诊断与调试

### 调试模式

```bash
opencli zhihu hot --debug          # 显示 pipeline 步骤详情
```

### 诊断模式（供 AI Agent 使用）

```bash
OPENCLI_DIAGNOSTIC=1 opencli zhihu hot 2>diag.json
# 从 stderr 中提取 RepairContext JSON
```

### 常见问题

| 问题 | 解决方案 |
|------|---------|
| "Cannot connect to opencli daemon" | 运行 `opencli doctor`，确认 daemon 端口 19825 可用 |
| "Browser Bridge extension is not connected" | 安装/重新加载 Chrome 扩展 |
| "Not logged in to xxx" | 在 Chrome 中登录该网站 |
| "timed out after Ns" | 增加 `OPENCLI_BROWSER_COMMAND_TIMEOUT` 环境变量 |
| 站点返回空数据 | 可能是地域限制/反爬，在 Chrome 中手动确认数据可见 |

---

## 八、建议先读的源码

按以下顺序阅读，可以快速建立整体理解：

1. **`src/registry.ts`** — 命令定义与注册机制，理解 `cli()` 函数和 `CliCommand` 接口
2. **`src/execution.ts`** — 命令执行生命周期，理解浏览器会话管理
3. **`src/pipeline/executor.ts`** — Pipeline 引擎，理解声明式步骤如何执行
4. **`src/errors.ts`** — 错误体系设计，理解结构化错误和退出码
5. **`src/diagnostic.ts`** — 诊断系统，理解 RepairContext 和脱敏机制
6. **`src/cascade.ts`** — 策略级联，理解自动认证策略发现
7. **`src/generate-verified.ts`** — 验证生成 harness，理解 AI 生成适配器的完整流程
8. **`autoresearch/engine.ts`** — AutoResearch 引擎，理解 Karpathy 风格的自动化研究循环
9. **`designs/self-repair-protocol.md`** — 自修复协议设计文档
10. **`docs/developer/architecture.md`** — 官方架构文档
