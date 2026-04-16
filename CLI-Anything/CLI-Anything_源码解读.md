# CLI-Anything 源码解读

> 仓库: https://github.com/HKUDS/CLI-Anything  
> 分析日期: 2026-04-16  
> 本地路径: `~/Documents/coding/github/CLI-Anything`

---

## 一、项目概述

**CLI-Anything** 是一个将任意 GUI 软件转换为 AI Agent 可用 CLI 工具的**元工具**。它不直接提供 CLI — 而是提供一套方法论（HARNESS.md）+ 插件命令，指导 AI Agent（Claude Code）为目标软件生成生产级 CLI 接口。

### 仓库规模

| 指标 | 数值 |
|------|------|
| 总文件数 | 1,057 |
| Python 源文件 | 762 |
| 已支持软件 | 44 款（含 agent-harness） |
| 测试文件 | 98 个 |
| 测试用例总数 | 2,130+ |
| SKILL.md 文件 | 43 个 |
| setup.py（可发布包） | 43 个 |
| 核心 SOP（HARNESS.md） | 592 行 |
| 指南文档（guides/） | 7 篇 |
| 插件命令（commands/） | 5 个 |

### 核心特性

| 特性 | 说明 |
|------|------|
| 方法论驱动 | 7 阶段 SOP（HARNESS.md）统一约束所有 CLI 的生成过程 |
| 真实软件硬依赖 | 不重新实现功能，调用 GIMP/Blender/LibreOffice 等真实软件后端 |
| Agent-Native | 每个命令支持 `--json`，AI Agent 可解析结构化输出 |
| 双模式 CLI | Click 子命令 + 交互式 REPL，同时服务脚本和交互场景 |
| 统一品牌体验 | ReplSkin 为所有 CLI 提供一致的终端 UI |
| 自包含 Skill | 每个 CLI 包内含 SKILL.md，Agent 可自动发现和学习 |
| PyPI 可发布 | PEP 420 namespace package，`pip install cli-anything-<software>` |
| 注册分发 | registry.json + CLI-Hub，Agent 可搜索/安装/管理 |

### 与 OpenCLI 的定位差异

| 维度 | CLI-Anything | OpenCLI |
|------|:---:|:---:|
| **目标** | GUI 软件 → CLI（GIMP、Blender、LibreOffice...） | 网站/App → CLI（Twitter、知乎、B 站...） |
| **生成者** | AI Agent（Claude Code）按 HARNESS.md 生成 | CLI 代码自动生成（`opencli generate`） |
| **后端** | 真实软件（subprocess 调用） | 浏览器 + HTTP API |
| **运行时 LLM** | 生成时需要大模型，运行时不需要 | 全程不需要 LLM |
| **产物语言** | Python（Click 框架） | TypeScript（cli() API） |
| **测试方式** | Agent 本地跑 pytest（依赖真实软件） | CI 自动跑（GitHub Actions） |
| **Harness 含义** | SOP 文档（教 Agent 怎么做） | 代码模块（CLI 自己执行的验证循环） |

**一句话对比**：OpenCLI 是"代码写代码"，CLI-Anything 是"文档教 Agent 写代码"。

---

## 二、整体架构

### 三层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: 插件层 — 教 Agent 怎么做                                   │
│  cli-anything-plugin/                                               │
│  ┌───────────┐ ┌──────────────┐ ┌───────────┐ ┌───────────────────┐│
│  │HARNESS.md │ │commands/*.md │ │guides/*.md│ │skill_generator.py ││
│  │(核心 SOP) │ │(5 个命令)    │ │(7 篇指南) │ │(SKILL.md 生成器)  ││
│  └───────────┘ └──────────────┘ └───────────┘ └───────────────────┘│
│  repl_skin.py (统一 REPL 皮肤) │ templates/ (Jinja2 模板)          │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: CLI 产物层 — Agent 生成的产物                              │
│  {software}/agent-harness/cli_anything/{software}/                  │
│  ┌────────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────────┐│
│  │{sw}_cli.py │ │core/   │ │utils/  │ │tests/    │ │skills/     ││
│  │Click+REPL  │ │project │ │backend │ │test_core │ │SKILL.md    ││
│  │入口        │ │session │ │repl_   │ │test_e2e  │ │Agent 技能  ││
│  │            │ │export  │ │skin    │ │TEST.md   │ │            ││
│  └────────────┘ └────────┘ └────────┘ └──────────┘ └────────────┘│
│  setup.py (PyPI 打包配置)                                          │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: 注册分发层 — 让 Agent 发现和安装                           │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────┐                  │
│  │registry.json │→ │CLI-Hub   │→ │PyPI          │                  │
│  │45 个 CLI 注册│  │搜索/安装 │  │pip install   │                  │
│  │              │  │cli_hub/  │  │cli-anything-* │                  │
│  └──────────────┘  └──────────┘  └──────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 关键文件清单

| 文件 | 层 | 作用 |
|------|---|------|
| `cli-anything-plugin/HARNESS.md` | 插件 | **核心 SOP**：7 阶段流水线 + 架构模式 + 原则规则 |
| `cli-anything-plugin/commands/cli-anything.md` | 插件 | `/cli-anything` 主命令：从零构建一个 CLI |
| `cli-anything-plugin/commands/refine.md` | 插件 | `/refine` 命令：迭代扩展 CLI 覆盖率 |
| `cli-anything-plugin/commands/test.md` | 插件 | `/test` 命令：运行测试 + 更新 TEST.md |
| `cli-anything-plugin/commands/validate.md` | 插件 | `/validate` 命令：52 项合规检查 |
| `cli-anything-plugin/repl_skin.py` | 插件 | 统一 REPL 皮肤（每个 CLI 复制一份） |
| `cli-anything-plugin/skill_generator.py` | 插件 | 从 CLI 元数据自动生成 SKILL.md |
| `{software}/agent-harness/setup.py` | 产物 | PyPI 打包配置（PEP 420 namespace） |
| `{software}/.../utils/{software}_backend.py` | 产物 | 真实软件后端调用封装 |
| `{software}/.../core/session.py` | 产物 | 状态管理 + Undo/Redo |
| `{software}/.../core/export.py` | 产物 | 渲染管线 + 滤镜翻译 |
| `{software}/.../tests/TEST.md` | 产物 | 测试计划（Phase 4）+ 结果（Phase 6） |
| `registry.json` | 分发 | 45 个 CLI 的注册表 |
| `cli-hub/cli_hub/registry.py` | 分发 | 注册表获取 + 缓存（TTL 1 小时） |
| `cli-hub/cli_hub/installer.py` | 分发 | CLI 安装/卸载（pip/npm 双策略） |

---

## 三、核心模块详解

### 3.1 HARNESS.md — 核心 SOP

HARNESS.md（592 行）是整个项目的"宪法"。所有插件命令（`/cli-anything`、`/refine`、`/test`、`/validate`）的第一条指令都是：

```
**Before doing anything else, you MUST read ./HARNESS.md.**
```

**它不是参考文档，是 Agent 的强制执行规范。** HARNESS.md 的结构：

| 段落 | 内容 | 行数 |
|------|------|------|
| General SOP | 7 阶段流水线详细规范 | ~270 行 |
| Architecture Patterns & Pitfalls | 设计模式 + 踩坑指南 | ~170 行 |
| Principles & Rules | 不可违反的原则列表 | ~50 行 |
| Directory Structure | 标准目录结构模板 | ~30 行 |
| Testing Strategy | 四层测试方法论 | ~70 行 |

**关键角色**：HARNESS.md 在 CLI-Anything 中的地位等价于 OpenCLI 中 `generate-verified.ts`（代码强制验证流程），只是一个用文档约束 Agent，一个用代码约束执行。

### 3.2 插件命令系统（commands/）

5 个命令构成 Agent 操作 CLI-Anything 的完整接口：

```
/cli-anything <path>          ← Phase 0~7: 从零到发布
/cli-anything:refine <path>   ← 迭代扩展功能覆盖率
/cli-anything:test <path>     ← 运行测试 + 更新 TEST.md
/cli-anything:validate <path> ← 52 项合规检查
/cli-anything:list            ← 列出已构建的 CLI
```

**命令间的调用关系**：

```
用户发起: /cli-anything https://github.com/GNOME/gimp
  │
  ▼
Agent 读 HARNESS.md
  │
  ├─ Phase 1-3: 分析 → 设计 → 实现
  ├─ Phase 4-6: 测试计划 → 测试实现 → 文档（等价于内嵌 /test）
  ├─ Phase 6.5: 生成 SKILL.md
  └─ Phase 7: PyPI 发布
  │
  ▼
用户后续: /cli-anything:refine /path/to/gimp "batch processing"
  │
  ├─ Step 1-3: 盘点覆盖率 → 分析能力 → Gap 分析
  ├─ Step 4: 实现新命令
  ├─ Step 5: 扩展测试 + 跑全量（等价于内嵌 /test）
  └─ Step 6: 更新文档
  │
  ▼
用户随时: /cli-anything:validate /path/to/gimp → 52 项检查
```

**每个命令都是一个 Markdown 文件**（不是可执行代码）。命令定义是给 Claude Code 读的 — Agent 读完后按指令操作。这与 OpenCLI 的 `commands/run.ts`（可执行 TypeScript）完全不同。

### 3.3 ReplSkin — 统一 REPL 皮肤

**源码位置**：`cli-anything-plugin/repl_skin.py`（每个 CLI 复制一份到 `utils/repl_skin.py`）

```python
class ReplSkin:
    def __init__(self, software: str, version: str = "1.0.0",
                 history_file=None, skill_path=None):
        self.software = software.lower().replace("-", "_")
        self.display_name = software.replace("_", " ").title()
        # 自动检测 SKILL.md 路径
        if skill_path is None:
            _auto = Path(__file__).resolve().parent.parent / "skills" / "SKILL.md"
            if _auto.is_file():
                skill_path = str(_auto)
        self.skill_path = skill_path
        # 每个软件有独立的品牌色
        self.accent = _ACCENT_COLORS.get(self.software, _DEFAULT_ACCENT)
```

**核心能力**：

| 方法 | 作用 | Agent 意义 |
|------|------|-----------|
| `print_banner()` | 打印品牌欢迎框 + SKILL.md 路径 | Agent 启动后知道去哪读 Skill |
| `success/error/warning/info()` | 彩色状态消息 | 人类可读的反馈 |
| `table(headers, rows)` | 格式化表格 | 结构化信息展示 |
| `progress(current, total)` | 进度条 | 长任务反馈 |
| `create_prompt_session()` | prompt_toolkit 会话 | 带历史记录的输入 |
| `get_input(session, ...)` | 获取用户输入 | REPL 主循环 |

**每个软件有独立的品牌色**（`_ACCENT_COLORS` 字典）：GIMP 用暖橙、Blender 用深橙、Inkscape 用亮蓝、LibreOffice 用绿色...

**设计决策**：ReplSkin 是纯 ANSI 转义码实现，**零外部依赖**（核心样式部分）。只有 `create_prompt_session()` 依赖 `prompt_toolkit`，作为可选依赖。

### 3.4 Backend 模式 — 真实软件后端调用

**源码位置**：`{software}/agent-harness/cli_anything/{software}/utils/{software}_backend.py`

所有 44 个 Backend 遵循统一的三步模式（以 Blender 为例）：

```python
# blender/agent-harness/cli_anything/blender/utils/blender_backend.py

# Step 1: Find — 查找可执行文件
def find_blender() -> str:
    for name in ("blender",):
        path = shutil.which(name)
        if path:
            return path
    raise RuntimeError(
        "Blender is not installed. Install it with:\n"
        "  apt install blender   # Debian/Ubuntu\n"
        "  brew install --cask blender  # macOS"
    )

# Step 2: Invoke — 调用真实软件
def render_script(script_path: str, timeout: int = 300) -> dict:
    blender = find_blender()
    cmd = [blender, "--background", "--python", script_path]
    result = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout)
    return {"stdout": result.stdout, "stderr": result.stderr, "returncode": result.returncode}

# Step 3: Verify — 验证输出（在 export.py 中）
def verify_render(output_path):
    assert os.path.exists(output_path)
    assert os.path.getsize(output_path) > 0
```

**各软件 Backend 的调用方式**：

| 软件 | Backend 调用方式 | 关键命令 |
|------|-----------------|---------|
| Blender | `blender --background --python script.py` | bpy 脚本 |
| GIMP | `gimp -i -b '(script-fu-console-eval ...)'` | Script-Fu |
| LibreOffice | `libreoffice --headless --convert-to pdf` | ODF → PDF/DOCX |
| Inkscape | `inkscape --actions="..." --export-filename=...` | SVG → PNG/PDF |
| Shotcut/Kdenlive | `melt project.mlt -consumer avformat:output.mp4` | MLT XML → MP4 |
| Audacity | `sox input.wav output.wav effects...` | 音频处理 |
| OBS Studio | obs-websocket 协议 | WebSocket RPC |
| Ollama | `http://localhost:11434/api/*` | REST API |

**硬依赖原则**：HARNESS.md 明确规定 — "The real software is a hard dependency. Do NOT gracefully degrade." 找不到软件时直接 `raise RuntimeError` 并附带安装指令。

### 3.5 Session + Undo/Redo

**源码位置**：`{software}/.../core/session.py`

```python
class Session:
    def __init__(self):
        self._project = None
        self.project_path = None
        self._undo_stack = []   # 快照栈
        self._redo_stack = []
        self._modified = False  # 是否有未保存修改

    def snapshot(self):
        """在每次变更前调用 — 保存当前状态到 undo 栈"""
        self._undo_stack.append(copy.deepcopy(self._project))
        self._redo_stack.clear()  # 新操作清空 redo
        self._modified = True

    def undo(self):
        if not self._undo_stack:
            return None
        self._redo_stack.append(copy.deepcopy(self._project))
        self._project = self._undo_stack.pop()
        return self._project

    def save_session(self):
        """原子写入 — 使用文件锁防止并发损坏"""
        _locked_save_json(self.project_path, self._project)
        self._modified = False
```

**文件锁机制**（`guides/session-locking.md`）：

```python
def _locked_save_json(path, data, **dump_kwargs):
    """HARNESS.md 要求的原子写入模式"""
    try:
        f = open(path, "r+")            # 不截断！
    except FileNotFoundError:
        f = open(path, "w")             # 首次创建
    with f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)  # 排他锁
        f.seek(0)
        f.truncate()                    # 锁内截断
        json.dump(data, f, **dump_kwargs)
        f.flush()
        fcntl.flock(f.fileno(), fcntl.LOCK_UN)
```

**关键细节**：不能用 `open("w")` — 因为 `"w"` 在获取锁之前就截断了文件。必须先 `"r+"` 打开，获取锁后再 `truncate()`。

**Auto-Save + --dry-run**（`guides/auto-save-dry-run.md`）：

```python
@cli.result_callback()
def auto_save_on_exit(result, use_json, project_path, dry_run, **kwargs):
    """One-shot 命令结束后自动保存（REPL 模式不触发）"""
    if _repl_mode or dry_run:
        return
    sess = get_session()
    if sess.has_project() and sess._modified and sess.project_path:
        sess.save_session()
```

**为什么需要**：One-shot 命令（如 `cli-anything-kdenlive --project p.json bin import video.mp4`）会修改内存中的项目，但进程退出时不会调用 `save_session()`。没有 auto-save，修改会丢失。

### 3.6 Export 渲染管线

**源码位置**：`{software}/.../core/export.py`

渲染管线的三优先策略（HARNESS.md "The Rendering Gap" 章节）：

```
优先级 1: 原生引擎
  └─ melt（for MLT XML），blender --render，libreoffice --headless
  └─ 最完整，所有滤镜/效果都能渲染

优先级 2: 滤镜翻译层
  └─ MLT filter → ffmpeg -filter_complex
  └─ 能处理大部分效果，但某些无法映射

优先级 3: 手动渲染脚本
  └─ 生成脚本让用户在 GUI 中打开执行
  └─ 最后手段，用于无法自动化的场景
```

**Rendering Gap 问题**：CLI 往项目文件（MLT XML/ODF/SVG）中添加了滤镜/效果。但如果用简单工具（如 ffmpeg concat）渲染，它只读原始媒体文件，**忽略所有项目级效果**。输出看起来和输入一模一样。

HARNESS.md 明确要求："**Every filter/effect in the registry MUST have a corresponding render mapping** or be explicitly documented as 'project-only (not rendered)'"。

### 3.7 skill_generator.py — SKILL.md 自动生成

**源码位置**：`cli-anything-plugin/skill_generator.py`

```python
def extract_cli_metadata(harness_path: str) -> SkillMetadata:
    """从 CLI 产物目录中提取元数据"""
    # 1. 找到 cli_anything/<software>/ 目录
    software_dir = find_software_dir(harness_path)

    # 2. 从 README.md 提取介绍文本和系统依赖
    skill_intro = extract_intro_from_readme(readme_content)
    system_package = extract_system_package(readme_content)  # 匹配 apt/brew install

    # 3. 从 setup.py 提取版本号
    version = extract_version_from_setup(setup_path)

    # 4. 从 {software}_cli.py 提取命令组和子命令
    command_groups = extract_commands_from_cli(cli_file)

    # 5. 根据软件类型生成示例
    examples = generate_examples(software_name, command_groups)

    return SkillMetadata(
        skill_name=f"cli-anything-{software_name}",
        skill_description=f"CLI for {display_name} — {intro[:100]}...",
        command_groups=command_groups,
        examples=examples, ...
    )
```

**提取 → 模板 → SKILL.md** 的流程：

```
{software}_cli.py  ──→ extract_commands_from_cli() ──→ CommandGroup[]
README.md          ──→ extract_intro_from_readme()  ──→ skill_intro
setup.py           ──→ extract_version_from_setup() ──→ version
                                    │
                                    ▼
                       SkillMetadata dataclass
                                    │
                                    ▼
                    Jinja2 template (SKILL.md.template)
                                    │
                                    ▼
                  cli_anything/<software>/skills/SKILL.md
```

**输出位置**：`cli_anything/<software>/skills/SKILL.md`（在 Python 包内部）。`setup.py` 的 `package_data` 会将其打包到 pip 安装中。ReplSkin 的 `print_banner()` 自动检测并显示该文件的绝对路径 — Agent 启动 REPL 后立刻知道去哪读 Skill。

---

## 四、7 阶段流水线详解

这是 CLI-Anything 的核心流程，对标 OpenCLI 的 Verified Generation 流水线（四、CLI 生成流程）。

### 4.1 Phase 1: Codebase Analysis — 分析目标软件

**HARNESS.md 要求**（原文）：

```
1. Identify the backend engine — Most GUI apps separate presentation from logic.
2. Map GUI actions to API calls — Every button click corresponds to a function call.
3. Identify the data model — What file formats does it use?
4. Find existing CLI tools — Many backends ship their own CLI.
5. Catalog the command/undo system — If the app has undo/redo, it likely uses a command pattern.
```

**Agent 做什么**：读目标软件的源码，找到后端引擎、数据模型、现有 CLI 工具。

**输出**：Agent 在脑中形成一个"软件地图"（不要求写 analysis.json，HARNESS.md 中没有强制要求持久化分析结果，但 `/cli-anything` 命令的 Phase 1 会让 Agent "Documents the architecture"）。

**例子 — 分析 Shotcut**：
- 后端引擎：MLT 框架
- 数据模型：MLT XML（`<mlt><playlist><producer>` 结构）
- 现有 CLI 工具：`melt`（MLT 的命令行渲染器）
- 命令/撤销系统：Shotcut 的 undo 基于 MLT XML diff

### 4.2 Phase 2: CLI Architecture Design — 设计 CLI 结构

**HARNESS.md 要求**：

| 决策点 | 选项 | 推荐 |
|--------|------|------|
| 交互模型 | Stateful REPL / Subcommand CLI / Both | **Both** |
| 命令分组 | 按软件的逻辑域分 | project, core ops, import/export, config, session |
| 状态模型 | 内存（REPL）/ 文件（CLI）/ 两者 | 两者 |
| 输出格式 | 人类可读 / JSON / 两者 | 两者，`--json` 控制 |

**输出**：`<SOFTWARE>.md`（软件特定的 SOP 文档），位于 `{software}/agent-harness/` 目录。

### 4.3 Phase 3: Implementation — 实现

**HARNESS.md 规定的实现顺序**：

```
1. 数据层     ── XML/JSON 操作（项目文件）
2. 探测命令   ── info, list, status（Agent 先观察再操作）
3. 变更命令   ── 一个命令一个操作（add, remove, set）
4. 后端集成   ── utils/{software}_backend.py（find → invoke → verify）
5. 渲染/导出  ── core/export.py（三优先策略）
6. 状态管理   ── core/session.py（Undo/Redo + 文件锁）
7. REPL       ── ReplSkin 集成（invoke_without_command=True）
```

**关键规则**：
- "**Use the Real Software — Don't Reimplement It**"（HARNESS.md #1 Rule）
- "**Fail loudly and clearly**" — Agent 需要明确的错误消息来自我纠正
- "**Be idempotent where possible**" — 同一命令跑两次应该安全
- "**JSON output mode**" — 每个命令都必须支持 `--json`

### 4.4 Phase 4-5: Test Planning + Implementation — 先计划，再写测试

**与 OpenCLI 的关键区别**：CLI-Anything 要求**先写 TEST.md 再写测试代码**，OpenCLI 没有这个步骤。

**Phase 4（先写计划）**：

```markdown
# TEST.md — Part 1（测试实现前写好）

## Test Inventory Plan
- test_core.py: 66 unit tests planned
- test_full_e2e.py: 37 E2E tests planned

## Unit Test Plan
- project.py: 创建/打开/保存/信息，边界条件（负数尺寸、无效模式）
- layers.py: 添加/删除/移动/复制，属性设置，ID 唯一性
...

## Realistic Workflow Scenarios
- "Photo editing pipeline": import → resize → brightness → sharpen → export PDF
- "Poster layout": new project → add text → add image → align → export PNG
```

**Phase 5（按计划实现）— 四层测试**：

```
第一层: 单元测试 (test_core.py)
  │   合成数据，无外部依赖，测每个函数
  │   例: assert create_project(name="test")["name"] == "test"
  │
第二层: E2E 中间文件 (test_full_e2e.py)
  │   验证生成的项目文件结构正确
  │   例: ZIP 结构是否包含 word/document.xml (DOCX)
  │   例: MLT XML 是否有正确的 producer/tractor 节点
  │
第三层: E2E 真后端 (test_full_e2e.py)
  │   调用真实软件，验证最终输出
  │   例: export PDF → 检查 magic bytes b"%PDF-"
  │   例: render video → ffprobe 检查帧数/时长/编码
  │   例: render image → PIL 检查像素值
  │   打印产物路径: print(f"\n  PDF: {path} ({size:,} bytes)")
  │
第四层: CLI 子进程 (test_full_e2e.py::TestCLISubprocess)
  │   通过 subprocess.run() 调用已安装的命令
  │   _resolve_cli() 查找已安装命令，fallback 到 python -m
  │   CLI_ANYTHING_FORCE_INSTALLED=1 强制使用安装版本
```

**`_resolve_cli()` 模式**（HARNESS.md 强制要求）：

```python
def _resolve_cli(name):
    """查找已安装命令；开发模式 fallback 到 python -m"""
    force = os.environ.get("CLI_ANYTHING_FORCE_INSTALLED", "").strip() == "1"
    path = shutil.which(name)
    if path:
        print(f"[_resolve_cli] Using installed command: {path}")
        return [path]
    if force:
        raise RuntimeError(f"{name} not found in PATH. Install with: pip install -e .")
    module = name.replace("cli-anything-", "cli_anything.") + "." + name.split("-")[-1] + "_cli"
    print(f"[_resolve_cli] Falling back to: {sys.executable} -m {module}")
    return [sys.executable, "-m", module]
```

**硬依赖原则**：HARNESS.md 写道 — "**No graceful degradation.** The real software MUST be installed. Tests must NOT skip or fake results when the software is missing." 这与 OpenCLI 的 E2E 测试使用 `warn + pass`（容忍站点不稳定）形成鲜明对比。

**输出验证方法论**（HARNESS.md 专节）：

```python
# 不能只看"跑完没报错" — 必须验证输出内容
# PDF: magic bytes
with open(path, "rb") as f:
    assert f.read(5) == b"%PDF-"

# DOCX/XLSX/PPTX: ZIP + OOXML 结构
with zipfile.ZipFile(path) as z:
    assert "word/document.xml" in z.namelist()

# Video: ffprobe 检查帧数
result = subprocess.run(["ffprobe", "-count_frames", ...])
assert int(result.stdout) == expected_count

# Image: PIL 像素级分析
img = Image.open(path)
pixels = np.array(img)
assert pixels.mean() > original_mean * 1.2  # brightness filter 后确实更亮
```

### 4.5 Phase 6-6.5: Test Documentation + SKILL.md Generation

**Phase 6**：Agent 运行 `pytest -v --tb=no`，将输出追加到 TEST.md（Part 2）：

```markdown
# TEST.md — Part 2（测试通过后追加）

## Test Results

Last run: 2026-04-15 10:30:00

```
test_core.py::TestProject::test_create_default PASSED
test_core.py::TestProject::test_create_custom_size PASSED
...
test_full_e2e.py::TestCLISubprocess::test_full_writer_pdf_workflow PASSED
```

**Summary**: 103 passed in 3.05s
```

**Phase 6.5**：调用 `skill_generator.py` 生成 SKILL.md（详见 3.7 节）。

### 4.6 Phase 7: PyPI Publishing — 发布

**PEP 420 Namespace Package** 原理：

```
cli_anything/           ← 无 __init__.py（namespace package）
├── gimp/               ← 有 __init__.py（regular package）
│   └── ...
├── blender/            ← 有 __init__.py（另一个 pip 包提供）
│   └── ...
└── libreoffice/        ← 有 __init__.py（又一个 pip 包提供）
    └── ...
```

多个独立 PyPI 包共享同一个 `cli_anything` 命名空间。`pip install cli-anything-gimp` 和 `pip install cli-anything-blender` 不会冲突 — 它们各自提供 `cli_anything/gimp/` 和 `cli_anything/blender/`。

**setup.py 关键配置**：

```python
setup(
    name="cli-anything-blender",
    packages=find_namespace_packages(include=["cli_anything.*"]),
    entry_points={
        "console_scripts": [
            "cli-anything-blender=cli_anything.blender.blender_cli:main",
        ],
    },
    package_data={
        "cli_anything.blender": ["skills/*.md"],  # SKILL.md 随包安装
    },
)
```

**验证链路**：
```bash
pip install -e .                    # 本地安装
which cli-anything-blender          # 确认在 PATH
cli-anything-blender --help         # 确认能运行
cli-anything-blender --json project new  # 确认 JSON 输出
```

### 4.7 具体例子：GIMP 从零到发布

**对标 OpenCLI 的"知乎热榜"全流程例子**：

```bash
# 用户执行:
/cli-anything https://github.com/GNOME/gimp
```

**Phase 1 — Agent 分析 GIMP 源码**：
- 后端引擎：GEGL（图像处理）+ Script-Fu（脚本接口）
- 数据模型：`.xcf` 文件（原生），但 CLI 用 JSON 项目文件（`.gimp-cli.json`）
- 现有 CLI：`gimp -i -b '(script-fu-...)'`，但过于底层
- 替代方案：用 Pillow 做图像操作，GEGL/Script-Fu 做高级效果

**Phase 2 — Agent 设计命令分组**：
```
cli-anything-gimp
├── project    ─ new, open, save, info, list-profiles
├── layer      ─ add, add-from-file, remove, move, duplicate, set
├── filter     ─ add, remove, set-param, list, info
├── canvas     ─ resize, scale, crop, set-mode, set-dpi
├── media      ─ probe, check
├── export     ─ render (→ 调用真实 GIMP/Pillow)
├── session    ─ undo, redo, status, history
└── repl       ─ 交互模式
```

**Phase 3 — Agent 实现**：
生成 `gimp/agent-harness/cli_anything/gimp/` 目录，含 core/（project.py, layers.py, filters.py, canvas.py, media.py, export.py, session.py）+ utils/（gimp_backend.py, repl_skin.py）+ gimp_cli.py

**Phase 4 — Agent 写 TEST.md 计划**：
"test_core.py: 66 unit tests planned; test_full_e2e.py: 37 E2E tests planned"

**Phase 5 — Agent 实现测试 + 运行**：

```python
# test_full_e2e.py — 真实像素验证
class TestBrightnessFilter:
    def test_brightness_increases_pixel_values(self, sample_image):
        proj = create_project()
        add_from_file(proj, sample_image, name="Photo")
        add_filter(proj, "brightness", 0, {"factor": 1.3})
        result = render(proj, ...)
        img = Image.open(result["output"])
        pixels = np.array(img)
        assert pixels.mean() > original_mean * 1.2  # 像素确实更亮

class TestCLISubprocess:
    CLI_BASE = _resolve_cli("cli-anything-gimp")
    def test_full_workflow(self, tmp_dir):
        self._run(["project", "new", "-o", proj_path])
        self._run(["--project", proj_path, "layer", "add-from-file", img_path])
        self._run(["--project", proj_path, "export", "render", out_path, "-p", "png"])
        assert os.path.exists(out_path)
```

```
Agent 运行: pytest -v -s → 103 passed in 3.05s
```

**Phase 6 — Agent 追加结果到 TEST.md**

**Phase 6.5 — Agent 生成 SKILL.md**

**Phase 7 — Agent 创建 setup.py + pip install -e . + 验证**

---

## 五、运行时技术点

### 5.1 Click + REPL 双模式 — invoke_without_command 机制

```python
# {software}_cli.py
@click.group(invoke_without_command=True)
@click.option("--json", "use_json", is_flag=True)
@click.option("--project", type=str, default=None)
@click.pass_context
def cli(ctx, use_json, project_path):
    ctx.ensure_object(dict)
    ctx.obj["json"] = use_json
    if ctx.invoked_subcommand is None:
        ctx.invoke(repl)    # ← 无子命令 → 进入 REPL
```

**双模式的设计意义**：
- **REPL 模式**：Agent 在交互式会话中保持状态（打开项目 → 多次操作 → 保存）
- **子命令模式**：Agent 在脚本/管道中使用（单次操作，auto-save 后退出）
- `--json` 标志：Agent 获取机器可解析的输出，人类获取格式化表格

### 5.2 输出验证方法论 — "跑完没报错"不够

HARNESS.md "Output Verification Methodology" 章节的核心思想：

```
"Never assume an export is correct just because it ran without errors."
```

| 输出类型 | 验证方法 | 为什么"没报错"不够 |
|---------|---------|------------------|
| PDF | magic bytes `%PDF-` | 有些软件报错写了空文件 |
| DOCX/XLSX | ZIP 结构 + OOXML 内部文件 | 可能生成了损坏的 ZIP |
| Video | ffprobe 帧数 + 时长 | ffmpeg 可能忽略了滤镜 |
| Image | PIL 像素值 + 亮度/色彩 | 渲染可能跳过了效果 |
| Audio | RMS 电平 + 频谱 | 可能输出了静音 |

### 5.3 渲染管线三优先策略

```
渲染优先级:
  1. 原生引擎（melt、blender --render、libreoffice --headless）
     └─ 最完整：读项目文件，应用所有效果
  2. 滤镜翻译层（MLT filter → ffmpeg -filter_complex）
     └─ 中等：能处理大部分效果，部分无法映射
  3. 手动渲染脚本（用户在 GUI 中打开执行）
     └─ 最后手段：用于无法自动化的场景
```

**滤镜翻译陷阱**（`guides/filter-translation.md`）：
- 重复滤镜合并
- 交错流排序
- 参数缩放差异（MLT 用 0-1000，ffmpeg 用 0.0-1.0）
- 不可映射效果

### 5.4 PEP 420 Namespace Package

```
pip install cli-anything-gimp      → cli_anything/gimp/
pip install cli-anything-blender   → cli_anything/blender/
pip install cli-anything-shotcut   → cli_anything/shotcut/
```

**关键**：`cli_anything/` 目录**没有** `__init__.py`。Python 3.3+ 的 implicit namespace packages 允许多个包共享同一个顶级命名空间。`find_namespace_packages(include=["cli_anything.*"])` 在 setup.py 中使用。

---

## 六、持续改进

### 6.1 /cli-anything:refine — 迭代扩展覆盖率

```
/cli-anything:refine <software-path> [focus]
```

**6 步流程**：

```
Step 1: 盘点当前覆盖率
  │   读 {software}_cli.py + 所有 core 模块
  │   构建覆盖地图: { function: covered | not_covered }
  │
Step 2: 分析目标软件完整能力
  │   重新扫描源码，聚焦 [focus] 领域（如果指定）
  │
Step 3: Gap 分析
  │   按优先级排序: 高影响 > 易实现 > 可组合
  │   向用户展示 gap 报告，确认优先级
  │
Step 4: 实现新命令
  │   遵循 HARNESS.md 的所有模式
  │
Step 5: 扩展测试 + 跑全量
  │   新测试 + 旧测试一起跑 → 确保无回归
  │
Step 6: 更新文档
```

**成功标准**：
- 所有旧测试仍然通过（无回归）
- 新测试 100% 通过
- 文档已更新

**Refine 是增量的** — 可以多次运行，每次聚焦不同领域。

### 6.2 /cli-anything:test — 独立测试

```
/cli-anything:test <software-path>
  │
  ├─ 1. 定位 CLI harness 目录
  ├─ 2. 运行 pytest -v -s --tb=short
  ├─ 3. 验证 [_resolve_cli] Using installed command: 出现在输出中
  ├─ 4. 全部通过 → 更新 TEST.md
  └─ 5. 有失败 → 不更新 TEST.md，保留上次通过的结果
```

**失败处理**：失败时不自动修复 — 只报告失败的测试名和错误信息，建议修复方向，让 Agent 决定下一步。这与 OpenCLI 的 Self-Repair（自动诊断 → 修复 → 重试 3 轮）完全不同。

### 6.3 /cli-anything:validate — 52 项静态合规检查

```
/cli-anything:validate <software-path>

输出:
  Directory Structure     (5/5 checks passed)
  Required Files          (9/9 files present)
  CLI Implementation      (7/7 standards met)
  Core Modules            (5/5 standards met)
  Test Standards          (10/10 standards met)
  Documentation           (4/4 standards met)
  PyPI Packaging          (7/7 standards met)
  Code Quality            (5/5 checks passed)
  Overall: PASS (52/52 checks)
```

**8 个检查维度**：

| 维度 | 检查项 | 示例 |
|------|--------|------|
| 目录结构 | 5 | `cli_anything/` 无 `__init__.py`（PEP 420） |
| 必需文件 | 9 | README.md、session.py、export.py、TEST.md... |
| CLI 实现 | 7 | Click 框架、`--json`、`--project`、REPL、handle_error |
| 核心模块 | 5 | project.py 有 create/open/save、session.py 有 undo/redo |
| 测试标准 | 10 | TEST.md 有计划+结果、`_resolve_cli` 使用、无 hardcoded 路径 |
| 文档标准 | 4 | README.md、SOFTWARE.md |
| PyPI 打包 | 7 | `find_namespace_packages`、entry_points、>=3.10 |
| 代码质量 | 5 | 无语法错误、无 bare except、PEP 8 |

**对标 OpenCLI**：等价于 OpenCLI 的 `opencli validate` 命令（5.4 Validation Harness），但 validate 的 52 项检查完全由 Agent 执行（读文件+判断），不是代码自动跑。

### 6.4 "修改后自动测试"的统一模式 — 文档驱动 vs 代码驱动

CLI-Anything 中"修改代码后 Agent 自动测试"**不是代码里硬编码的循环**，而是 **HARNESS.md SOP + 命令定义** 驱动 Agent 执行的：

```
OpenCLI:      代码驱动 — 测试逻辑写死在代码中
              engine.ts 自动调 runVerify()
              generate-verified.ts 自动调 assessResult()
              Agent 不需要知道怎么测试

CLI-Anything: 文档驱动 — 测试规范写在 HARNESS.md 中
              Agent 读文档后主动执行 pytest
              没有任何代码自动调用测试
              全靠 Agent 遵循 SOP
```

**三个触发测试的时机**：

| 时机 | 触发命令 | 测试范围 | 失败处理 |
|------|---------|---------|---------|
| 初次构建 | `/cli-anything` Phase 5 | 全量 | Agent 修复后重跑 |
| 迭代扩展 | `/refine` Step 5 | 全量（新+旧） | Agent 修复后重跑 |
| 独立测试 | `/test` | 全量 | 报告失败，不自动修复 |

**CI 层面：只有部署，没有自动测试**

| Workflow | 做什么 | 不做什么 |
|----------|--------|---------|
| `deploy-pages.yml` | 部署 registry 到 GitHub Pages | 不跑测试 |
| `publish-cli-hub.yml` | 发布 cli-hub 到 PyPI | 不跑测试 |

**为什么没有 CI 测试**：每个 CLI 依赖真实 GUI 软件（GIMP 占 200MB+、Blender 占 500MB+），无法在标准 CI VM 上安装 44 款软件。

---

## 七、Skill 体系与 CLI-Hub

### 7.1 SKILL.md — Agent 的自发现机制

每个 CLI 包内含一个 SKILL.md（位于 `cli_anything/<software>/skills/SKILL.md`），格式：

```yaml
---
name: "cli-anything-gimp"
description: "Image editing via GIMP — layers, filters, canvas, export"
---

# cli-anything-gimp

## Installation
pip install cli-anything-gimp
Requires: GIMP (`apt install gimp`)

## Commands
### project — Project management
- `project new` — Create a new project
- `project open <path>` — Open existing project
...

## Examples
```bash
cli-anything-gimp project new -o my_project.json
cli-anything-gimp --project my_project.json layer add-from-file photo.png
cli-anything-gimp --project my_project.json --json export render output.png
```
```

**自动发现链路**：
```
Agent 安装: pip install cli-anything-gimp
  → Agent 启动: cli-anything-gimp
  → ReplSkin.print_banner() 打印:
    "Skill: /path/to/site-packages/cli_anything/gimp/skills/SKILL.md"
  → Agent 读 SKILL.md → 学会所有命令
```

### 7.2 CLI-Hub — 注册与分发

**registry.json**（45 个 CLI 注册）：

```json
{
  "meta": {
    "repo": "https://github.com/HKUDS/CLI-Anything",
    "description": "CLI-Hub — Agent-native stateful CLI interfaces for softwares"
  },
  "clis": [
    {
      "name": "gimp",
      "display_name": "GIMP",
      "version": "1.0.0",
      "description": "Image editing via GIMP",
      "requires": "gimp",
      "install_cmd": "pip install git+https://github.com/HKUDS/CLI-Anything.git#subdirectory=gimp/agent-harness",
      "entry_point": "cli-anything-gimp",
      "skill_md": null,
      "category": "image"
    },
    ...
  ]
}
```

**CLI-Hub 模块**（`cli-hub/cli_hub/`）：

| 模块 | 作用 |
|------|------|
| `registry.py` | 从 GitHub Pages 获取 registry.json，本地缓存（TTL 1 小时） |
| `installer.py` | 安装/卸载 CLI（pip/npm 双策略），记录到 `~/.cli-hub/installed.json` |
| `analytics.py` | 匿名使用统计（可关闭） |
| `cli.py` | `cli-hub search/install/uninstall/list` 命令 |

**安装流程**：
```bash
cli-hub install gimp
  → registry.py 查 registry.json 找到 install_cmd
  → installer.py 执行: pip install git+https://...#subdirectory=gimp/agent-harness
  → 记录到 ~/.cli-hub/installed.json
  → 验证: which cli-anything-gimp
```

### 7.3 对比 OpenCLI 的 Skill 体系

| 维度 | CLI-Anything | OpenCLI |
|------|:---:|:---:|
| Skill 是什么 | CLI 的使用手册（自动生成） | Agent 的操作手册（手动编写） |
| 谁写的 | skill_generator.py 自动提取 | 开发者手动编写 26KB+ |
| 内容深度 | 命令列表 + 示例 | 决策树 + Tier 策略 + 陷阱表 |
| 存放位置 | Python 包内（随 pip 安装） | 仓库 skills/ 目录 |
| 发现方式 | ReplSkin banner 打印路径 | Agent 加载 Skill 文件 |
| 分发方式 | CLI-Hub + PyPI | 无独立分发 |

---

## 八、关键设计模式

### 8.1 数据优先 — 操作原生格式

```python
# 不重新实现 GIMP 的图像处理 — 操作原生格式文件
# GIMP：.xcf / CLI 用 .gimp-cli.json
# Inkscape：.svg (XML)
# Shotcut/Kdenlive：.mlt (XML)
# LibreOffice：ODF (.odt/.ods/.odp, ZIP + XML)
# Blender：.blend-cli.json → bpy 脚本
```

**关键区别**：CLI 操作的是项目文件（数据层），渲染交给真实软件。这就是为什么 HARNESS.md 要求"Phase 3 先实现数据层"。

### 8.2 "命令即 SOP" — HARNESS.md 统一约束 Agent

CLI-Anything 最核心的设计决策：**用文档而非代码约束 Agent 行为**。

```
OpenCLI:        代码约束 → Agent 调 CLI，CLI 内部确定性执行
                generate-verified.ts 保证 explore→cascade→verify 顺序

CLI-Anything:   文档约束 → Agent 读 HARNESS.md，按 SOP 自己执行
                HARNESS.md 保证 analyze→design→implement→test 顺序
```

**为什么选择文档而非代码**？因为 CLI-Anything 的目标软件种类繁多（3D/音频/视频/图表/办公...），每个软件的 Phase 1（分析）和 Phase 3（实现）完全不同。无法用一段确定性代码覆盖 44 款软件 — 只能用文档教 Agent 如何灵活处理。

### 8.3 硬依赖原则 — 不降级

```
HARNESS.md: "The real software is a hard dependency.
             Do NOT gracefully degrade to a fallback library.
             If the software is not installed, error with clear install instructions."
```

**对比 OpenCLI**：OpenCLI 的 E2E 测试对站点不稳定采用 `warn + pass`（容忍外部不确定性）。CLI-Anything 相反 — 真实软件是本地安装的，可控，所以要求 100% 通过。

### 8.4 渐进式披露

```
HARNESS.md (核心 SOP, 592 行)
  └─ 引用 guides/ (按需加载):
      ├── auto-save-dry-run.md    ← 只有 session-based CLI 需要
      ├── session-locking.md      ← 只有并发写入场景需要
      ├── filter-translation.md   ← 只有视频/音频 CLI 需要
      ├── mcp-backend.md          ← 只有 MCP 服务器方案需要
      ├── timecode-precision.md   ← 只有视频 CLI 需要
      ├── pypi-publishing.md      ← Phase 7 才需要
      └── skill-generation.md     ← Phase 6.5 才需要
```

Agent 不需要一次读完所有指南 — 每个指南在 HARNESS.md 中通过链接引用，Agent 到了对应 Phase 才按需加载。

---

## 九、与 OpenCLI 的系统性对比

| 维度 | CLI-Anything | OpenCLI |
|------|-------------|---------|
| **定位** | GUI 软件 → CLI | 网站/App → CLI |
| **核心文件** | HARNESS.md (592 行 Markdown) | generate-verified.ts (973 行 TypeScript) |
| **生成方式** | Agent 按文档手工生成 | CLI 代码自动生成 |
| **产物语言** | Python (Click) | TypeScript (cli() API) |
| **后端** | 真实 GUI 软件 (subprocess) | 浏览器 + HTTP API (CDP) |
| **认证** | 不涉及（本地软件） | 5 级 Tier 策略（PUBLIC→UI） |
| **测试驱动** | 文档驱动（Agent 按 SOP 跑 pytest） | 代码驱动（CLI 内嵌 assessResult） |
| **自动修复** | 无 | Bounded Repair + Self-Repair |
| **自动回滚** | 无 | git revert (AutoResearch) |
| **CI** | 无测试 CI（仅部署） | 四层 CI（unit/adapter/E2E/smoke） |
| **度量** | 布尔（100% 通过率） | 数值（SCORE=51/59） |
| **Skill 作用** | 使用手册（自动生成） | 操作手册（手动编写的决策树） |
| **分发** | PyPI + CLI-Hub | npm link + 本地注册 |
| **状态管理** | Session + Undo/Redo + 文件锁 | 无状态（每次命令独立） |
| **Harness 含义** | SOP 文档（教 Agent 怎么做） | 代码模块（CLI 自己执行的验证循环） |

**根本差异的一句话总结**：

```
OpenCLI:      确定性代码 → 可以用代码验证代码 → 代码驱动一切
CLI-Anything: 多样性软件 → 无法用代码覆盖所有情况 → 文档驱动 Agent
```

---

## 十、值得借鉴的点

1. **"先计划再实现"的测试流程**：Phase 4 先写 TEST.md 再写测试代码，避免测试覆盖偷工减料
2. **硬依赖原则**：不降级、不跳过、不 mock — 真实软件是真实测试的基础
3. **输出验证方法论**：magic bytes + ZIP 结构 + 像素级分析 + ffprobe，不信任"没报错"
4. **PEP 420 Namespace Package**：44 个独立 CLI 包共存于同一 `cli_anything` 命名空间
5. **渐进式披露**：核心 SOP 592 行，7 篇指南按需加载
6. **ReplSkin 统一品牌**：零外部依赖的 ANSI 样式系统，每个软件有独立品牌色
7. **SKILL.md 自发现**：随 pip 安装，ReplSkin banner 自动暴露路径，Agent 启动即可学习
8. **`_resolve_cli` 模式**：开发/安装双模式无缝切换，`CLI_ANYTHING_FORCE_INSTALLED` 保证 CI 纯度
9. **Session 文件锁**：`open("r+")` → `flock(LOCK_EX)` → `truncate()` → `json.dump()`，避免 `open("w")` 的竞态截断
10. **Auto-Save + --dry-run**：`result_callback` 钩子自动保存 one-shot 命令的修改，`--dry-run` 可抑制
11. **CLI-Hub 注册分发**：registry.json + 本地缓存 + pip/npm 双策略安装
12. **文档即代码约束**：HARNESS.md + commands/*.md 统一约束 Agent 行为，证明了"文档驱动"在 AI Agent 时代是可行的架构模式

---

## 十一、局限性

1. **强依赖大模型**：需要前沿级模型（如 Claude Opus）才能可靠执行 7 阶段流水线
2. **需要目标软件源码**：无源码时 Phase 1 分析质量大幅下降
3. **需迭代优化**：单次 `/cli-anything` 可能不完整，需多次 `/refine`
4. **渲染 Gap**：某些滤镜效果在 CLI 层面难以完美映射到渲染输出
5. **无 CI 自动测试**：44 款软件无法在标准 CI VM 上全部安装，测试完全依赖 Agent 本地执行
6. **无自动修复**：测试失败后不像 OpenCLI 那样有 Bounded Repair / Self-Repair，完全靠 Agent 人工判断
7. **文档约束的脆弱性**：如果 Agent 不严格遵循 HARNESS.md（如跳过 Phase 4 测试计划），无代码层面的强制保障
8. **跨平台安装复杂度**：GIMP/Blender/LibreOffice 在不同 OS 上的安装方式不同，Backend 的 `find_*()` 需要逐平台适配

---

*文档更新时间：2026-04-16*
*基于 CLI-Anything 仓库最新 main 分支*
