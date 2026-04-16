# CLI-Anything 源码解读

## 项目概述

**CLI-Anything** (GitHub: [HKUDS/CLI-Anything](https://github.com/HKUDS/CLI-Anything)) 是一个将任意 GUI 软件转换为 AI Agent 可用 CLI 工具的元工具。通过 7 阶段自动化流水线，为 Blender、GIMP、LibreOffice 等专业软件生成生产级 CLI 接口。

### 核心定位
- **使命**：让 AI Agent 能像人类一样操控真实的专业软件
- **方法**：不重新实现软件功能，而是生成调用真实软件后端的 CLI 包装器
- **范围**：支持 16+ 款专业软件，2000+ 测试用例，100% 通过率

---

## 整体架构

```
CLI-Anything
├── cli-anything-plugin/     # Claude Code 插件入口
│   ├── HARNESS.md           # 【核心】方法论 SOP（7 阶段流水线规范）
│   ├── commands/            # Claude Code 命令定义
│   ├── repl_skin.py         # 统一 REPL 界面
│   └── scripts/
│
├── {software}/agent-harness/  # 各软件的 CLI 包
│   ├── setup.py             # PyPI 发布配置
│   └── cli_anything/
│       └── {software}/
│           ├── __init__.py
│           ├── __main__.py
│           ├── {software}_cli.py   # Click CLI + REPL 入口
│           ├── core/               # 核心业务模块
│           │   ├── project.py      # 项目管理
│           │   ├── export.py        # 渲染管线
│           │   └── session.py      # 状态管理
│           ├── utils/
│           │   ├── {software}_backend.py  # 【关键】真实软件后端调用
│           │   └── repl_skin.py     # 统一 REPL 皮肤
│           └── tests/
│               ├── test_core.py     # 单元测试
│               └── test_full_e2e.py # E2E 测试
│
└── guides/                  # 详细指南
    ├── session-locking.md
    ├── skill-generation.md
    ├── pypi-publishing.md
    └── ...
```

---

## Harness 实现核心原理

### 什么是 Harness？

**Harness（挽具）** 是 CLI-Anything 的核心概念 —— 它是连接 AI Agent 与 GUI 软件的"桥梁"。Harness 不是重新实现软件，而是：

1. **生成结构化的 CLI 接口**（基于 Click 框架）
2. **操作软件的原生格式**（ODF、MLT XML、SVG 等）
3. **调用真实软件后端**进行渲染/导出

### 7 阶段流水线

| 阶段 | 名称 | 输入 | 输出 |
|------|------|------|------|
| 1 | Analyze（分析） | 软件源码/仓库 | analysis.json |
| 2 | Design（设计） | analysis.json | architecture.json |
| 3 | Implement（实现） | architecture.json | 完整 CLI 目录 |
| 4 | Plan Tests（测试计划） | 分析结果 | TEST.md 测试计划 |
| 5 | Implement Tests（测试实现） | TEST.md | 测试代码 + 结果 |
| 6 | Document（文档） | 测试结果 | TEST.md 最终版 |
| 6.5 | Generate SKILL.md（技能生成） | CLI 元数据 | skills/SKILL.md |
| 7 | Package & Publish（发布） | 完整 CLI | PyPI 包 |

---

## 核心模块详解

### 1. Backend 模块（真实软件调用）

Backend 是 Harness 的"心脏"，负责调用真实软件：

```python
# utils/lo_backend.py (LibreOffice 为例)
def find_libreoffice():
    """查找 LibreOffice 可执行文件"""
    lo = shutil.which("libreoffice") or shutil.which("libreoffice-headless")
    if not lo:
        raise RuntimeError(
            "LibreOffice not found. Install: apt install libreoffice"
        )
    return lo

def convert_odf_to(odf_path, output_format, output_path=None, overwrite=False):
    """将 ODF 文件转换为指定格式（调用真实 LibreOffice）"""
    lo = find_libreoffice()
    subprocess.run([
        lo, "--headless", "--convert-to", output_format,
        "--outdir", output_dir, odf_path
    ], check=True)
    return {"output": final_path, "format": output_format, "method": "libreoffice-headless"}
```

**关键原则**：
- 真实软件是**硬依赖**，不是可选的
- 不重新实现渲染功能
- 软件不存在时**报错**，而非优雅降级

### 2. CLI 入口（Click + REPL 双模式）

```python
# {software}_cli.py
@click.group(invoke_without_command=True)
@click.pass_context
def cli(ctx):
    """主入口：无子命令时进入 REPL 模式"""
    if ctx.invoked_subcommand is None:
        ctx.invoke(repl)

@cli.command()
def repl(project_path=None):
    """交互式 REPL 模式"""
    skin = ReplSkin("gimp", version="1.0.0")
    skin.print_banner()
    pt_session = skin.create_prompt_session()
    
    while True:
        line = skin.get_input(pt_session, project_name=project_name)
        # 处理命令...
```

**双模式设计**：
- **REPL 模式**：交互式会话，保持状态上下文
- **子命令模式**：脚本/管道使用，如 `cli-anything-gimp project new`

### 3. 项目状态管理（Session + Undo/Redo）

```python
# core/session.py
class SessionState:
    def __init__(self, project_path):
        self.project_path = project_path
        self.history = []      # 命令历史
        self.checkpoints = []  # 检查点（用于撤销）
    
    def execute_command(self, cmd):
        # 保存当前状态到检查点
        self.checkpoints.append(self._snapshot())
        # 执行命令
        result = cmd.execute()
        self.history.append(cmd)
        return result
    
    def undo(self):
        if self.checkpoints:
            self._restore(self.checkpoints.pop())
    
    def redo(self):
        if self.history:
            cmd = self.history.pop()
            cmd.execute()
```

### 4. REPL 皮肤（Unified ReplSkin）

统一所有 CLI 的交互体验：

```python
from cli_anything.<software>.utils.repl_skin import ReplSkin

skin = ReplSkin("blender", version="1.0.0")
skin.print_banner()        # 打印品牌欢迎界面
skin.success("Done")       # ✓ 绿色成功消息
skin.error("Failed")       # ✗ 红色错误消息
skin.warning("Warning")    # ⚠ 黄色警告
skin.table(headers, rows)  # 格式化表格
skin.progress(3, 10)      # 进度条
```

---

## Harness 关键设计模式

### 模式 1：数据优先，操作原生格式

```python
# GIMP：操作 .xcf 文件（原生格式）
# Inkscape：操作 .svg 文件
# Blender：生成 .blend-cli.json 或 bpy 脚本
# Shotcut/Kdenlive：操作 .mlt XML 文件
# LibreOffice：生成 ODF（.odt/.ods/.odp）

def create_project(name, width, height):
    """生成原生项目文件"""
    if format == "svg":
        return generate_svg(name, width, height)
    elif format == "mlt":
        return generate_mlt_xml(name, clips)
    elif format == "odf":
        return generate_odf_doc(name, doc_type)
```

### 模式 2：渲染管线三优先策略

```
┌─────────────────────────────────────────────┐
│  渲染优先级                                  │
├─────────────────────────────────────────────┤
│  1. 原生引擎（如 melt for MLT）              │
│  2. 滤镜翻译层（MLT → ffmpeg filter_complex）│
│  3. 手动渲染脚本（用户执行）                  │
└─────────────────────────────────────────────┘
```

### 模式 3：输出验证（魔法字节 + 结构检查）

```python
def verify_pdf(path):
    """验证 PDF 输出"""
    with open(path, "rb") as f:
        magic = f.read(5)
    assert magic == b"%PDF-", "Not a valid PDF"

def verify_docx(path):
    """验证 DOCX 输出（OOXML 是 ZIP 结构）"""
    with zipfile.ZipFile(path) as z:
        assert "word/document.xml" in z.namelist()

def verify_video_frames(path, expected_count):
    """验证视频帧"""
    result = subprocess.run([
        "ffprobe", "-v", "error", "-count_frames",
        "-select_streams", "v:0", "-show_entries",
        "stream=nb_read_frames", "-of", "csv=p=0", path
    ], capture_output=True, text=True)
    assert int(result.stdout) == expected_count
```

### 模式 4：Namespace Package（PEP 420）

```
cli_anything/           # 无 __init__.py（namespace package）
├── __init__.py        # 不存在！
├── gimp/
│   └── __init__.py
└── blender/
    └── __init__.py
```

**优势**：多个独立 PyPI 包可共存于同一 Python 环境

---

## 测试策略（四层测试）

| 层级 | 文件 | 目的 | 依赖 |
|------|------|------|------|
| 单元测试 | test_core.py | 测试核心函数（合成数据） | 无外部依赖 |
| E2E（原生） | test_full_e2e.py | 验证生成的项目文件结构 | 无 |
| E2E（真后端） | test_full_e2e.py | 调用真实软件生成最终输出 | 真实软件必须安装 |
| CLI 子进程 | test_full_e2e.py | 测试安装后的 CLI 命令 | pip install -e . |

---

## 支持的软件矩阵

| 软件 | 领域 | 后端 | 测试数 |
|------|------|------|--------|
| GIMP | 图像编辑 | Pillow + GEGL/Script-Fu | 107 |
| Blender | 3D 建模 | bpy (Python scripting) | 208 |
| Inkscape | 矢量图形 | SVG/XML 直接操作 | 202 |
| Audacity | 音频制作 | Python wave + sox | 161 |
| LibreOffice | 办公套件 | ODF + headless LO | 158 |
| OBS Studio | 直播录制 | obs-websocket | 153 |
| Shotcut | 视频编辑 | MLT XML + melt | 154 |
| Draw.io | 图表 | mxGraph XML | 138 |
| Ollama | 本地 LLM | Ollama REST API | 98 |
| ... | ... | ... | ... |

**总计：2,130+ 测试，100% 通过率**

---

## "修改代码后自动测试" — 机制分析

### 核心答案

CLI-Anything 中"修改代码后 Agent 自动测试"**不是代码里硬编码的自动化循环**，而是通过 **HARNESS.md SOP 规范 + Claude Code 插件命令** 驱动 Agent 执行的。Agent 读了 HARNESS.md 后，会按规范在每次修改后主动运行测试 — 这是"文档驱动的自动测试"，不同于 OpenCLI 那种代码内嵌的 `verifyCandidate()` 或 `runVerify()` 循环。

### 与 OpenCLI 的根本差异

```
OpenCLI:    代码驱动 — 测试逻辑写死在 engine.ts / generate-verified.ts 中
            CLI 自己调用 assessResult()、runVerify()、executePipeline()
            Agent 不需要知道怎么测试，代码自动完成

CLI-Anything: 文档驱动 — 测试规范写在 HARNESS.md + 命令定义(.md) 中
              Agent (Claude Code) 读文档后主动执行 pytest
              没有任何代码自动调用测试，全靠 Agent 遵循 SOP
```

### 三个触发测试的机制

#### 机制 1：`/cli-anything` 主命令（初次构建）

Agent 执行 `/cli-anything <software-path>` 时，按 HARNESS.md 7 阶段流水线工作。**Phase 5（Test Implementation）** 要求 Agent 在写完代码后：

```
Phase 3: 实现 CLI 代码
  │
  ▼
Phase 4: 创建 TEST.md 测试计划（先计划再写代码）
  │
  ▼
Phase 5: 实现测试代码 + 运行全部测试
  │   Agent 运行: pytest -v -s cli_anything/<software>/tests/
  │
  │   测什么（四层）:
  │   ├─ test_core.py:     单元测试（合成数据，无外部依赖）
  │   ├─ test_full_e2e.py: E2E 中间文件（验证 XML/ZIP 结构正确性）
  │   ├─ test_full_e2e.py: E2E 真后端（调用真实软件生成 PDF/图片/视频）
  │   └─ test_full_e2e.py: CLI 子进程测试（subprocess 调用已安装命令）
  │
  ▼
Phase 6: 将 pytest 输出追加到 TEST.md
```

**源码位置**：`cli-anything-plugin/commands/cli-anything.md:59-69`

**关键约束**：
- HARNESS.md 明确要求 "**No graceful degradation**" — 真实软件必须安装，测试不允许 skip
- 必须 100% 通过率才算 Phase 5 完成
- Success Criteria 第 4 条：`All tests pass (100% pass rate)`

#### 机制 2：`/cli-anything:refine` 命令（迭代改进）

Agent 执行 `/cli-anything:refine <software-path> [focus]` 添加新功能后，**Step 5** 强制要求跑全量测试：

```
Step 4: 实现新命令
  │
  ▼
Step 5: 扩展测试
  │   - 为每个新函数添加单元测试到 test_core.py
  │   - 为新命令添加 E2E 测试到 test_full_e2e.py
  │   - 添加将新命令与已有命令组合的 workflow 测试
  │   - 运行全部测试（新 + 旧），确保无回归
  │
  ▼
Step 6: 更新 TEST.md 测试结果
```

**源码位置**：`cli-anything-plugin/commands/refine.md:67-71`

**关键约束**：
- "Run all tests (old + new) to ensure no regressions" — 不是只跑新的
- Success Criteria: "All existing tests still pass (no regressions)" + "New tests achieve 100% pass rate"
- refine 是增量的，可多次运行，但每次都要跑全量

#### 机制 3：`/cli-anything:test` 命令（独立测试）

专门的测试命令，Agent 可随时调用：

```
/cli-anything:test <software-path>
  │
  ├─ 1. 定位 CLI harness 目录
  ├─ 2. 运行 pytest -v -s --tb=short
  ├─ 3. 验证 [_resolve_cli] Using installed command: 出现在输出中
  ├─ 4. 如果全部通过 → 更新 TEST.md
  └─ 5. 如果有失败 → 不更新 TEST.md，保留上次通过的结果
```

**源码位置**：`cli-anything-plugin/commands/test.md`

**失败处理**（与 OpenCLI 的对比）：
- 失败时**不自动修复** — 只报告哪些测试失败，建议修复方向
- "Offers to re-run after fixes" — 让 Agent 决定是否重跑
- 这与 OpenCLI 的 Self-Repair（自动诊断+修复+重试 3 轮）完全不同

### 另外一个机制：`/cli-anything:validate` 命令（结构验证）

这不是运行 pytest，而是检查 harness 是否符合 HARNESS.md 规范的 **52 项静态检查**：

```
/cli-anything:validate <software-path>
  │
  ├─ 1. 目录结构 (5 checks): PEP 420 namespace, __init__.py 有/无
  ├─ 2. 必需文件 (9 checks): README.md, session.py, export.py, TEST.md...
  ├─ 3. CLI 实现标准 (7 checks): Click 框架, --json, --project, REPL
  ├─ 4. 核心模块标准 (5 checks): project/session/export 的必需函数
  ├─ 5. 测试标准 (10 checks): TEST.md 有计划+结果, _resolve_cli 使用
  ├─ 6. 文档标准 (4 checks): README, SOFTWARE.md
  ├─ 7. PyPI 打包标准 (7 checks): setup.py, namespace, entry_points
  └─ 8. 代码质量 (5 checks): 语法, import, PEP 8, 错误处理
  │
  └─ 输出: Overall: PASS (52/52 checks) 或 FAIL (N/52)
```

**源码位置**：`cli-anything-plugin/commands/validate.md`

### CI 层面：只有部署，没有自动测试

CLI-Anything 的 GitHub Actions **不包含自动测试 CI**：

| Workflow | 做什么 | 触发条件 |
|----------|--------|---------|
| `deploy-pages.yml` | 部署 registry 到 GitHub Pages | push main + 改了 agent-harness/ 或 registry.json |
| `publish-cli-hub.yml` | 发布 cli-hub 包到 PyPI | push main + 改了 cli-hub/ |

没有 `ci.yml`，没有 `on: pull_request` 触发测试。每个软件的测试（2130+ 个）完全由 Agent 在本地运行 — 因为每个 CLI 依赖真实软件（GIMP/Blender/LibreOffice 等），无法在标准 CI VM 上安装全部依赖。

### 测试内容详解：四层验证

```
第一层: 单元测试 (test_core.py)
  │   合成数据，无外部依赖
  │   测函数逻辑：create_project(), add_layer(), add_filter() 等
  │   例：assert loaded["layers"][0]["filters"][0]["name"] == "brightness"
  │
第二层: E2E 中间文件 (test_full_e2e.py)
  │   验证生成的项目文件结构正确
  │   例：ZIP 结构是否包含 word/document.xml (DOCX)
  │   例：MLT XML 是否有正确的 producer/tractor 节点
  │
第三层: E2E 真后端 (test_full_e2e.py)
  │   调用真实软件，验证最终输出
  │   例：export PDF → 检查 magic bytes b"%PDF-"
  │   例：render video → ffprobe 检查帧数/时长/编码
  │   例：render image → PIL 检查像素值（brightness filter 后像素确实更亮）
  │   打印产物路径: print(f"\n  PDF: {path} ({size:,} bytes)")
  │
第四层: CLI 子进程 (test_full_e2e.py 中的 TestCLISubprocess)
  │   通过 subprocess.run() 调用已安装的 cli-anything-<software> 命令
  │   验证安装后端到端能跑通
  │   _resolve_cli() 查找已安装命令，fallback 到 python -m
  │   CLI_ANYTHING_FORCE_INSTALLED=1 强制使用已安装版本
```

**具体例子 — GIMP E2E 测试**（源码：`gimp/agent-harness/cli_anything/gimp/tests/test_full_e2e.py`）：

```python
class TestProjectLifecycle:
    def test_create_save_open_roundtrip(self, tmp_dir):
        proj = create_project(name="roundtrip")
        path = os.path.join(tmp_dir, "project.gimp-cli.json")
        save_project(proj, path)
        loaded = open_project(path)
        assert loaded["name"] == "roundtrip"    # 项目名保持
        assert loaded["canvas"]["width"] == 1920 # 默认尺寸正确

class TestBrightnessFilter:
    def test_brightness_increases_pixel_values(self, sample_image):
        proj = create_project()
        add_from_file(proj, sample_image, name="Photo")
        add_filter(proj, "brightness", 0, {"factor": 1.3})
        result = render(proj, ...)
        # 像素级验证：亮度滤镜后像素确实更亮
        img = Image.open(result["output"])
        pixels = np.array(img)
        assert pixels.mean() > original_mean * 1.2
```

### 全景对比：CLI-Anything vs OpenCLI

| 维度 | CLI-Anything | OpenCLI |
|------|:---:|:---:|
| **测试驱动方式** | 文档驱动（HARNESS.md SOP） | 代码驱动（内嵌函数调用） |
| **测试执行者** | Agent (Claude Code) 手动跑 pytest | CLI 代码自动调用 |
| **自动 CI** | 无（无法在 VM 上装 GIMP/Blender） | 有（GitHub Actions 四层 CI） |
| **修复机制** | Agent 人工修复，无自动修复 | Bounded Repair / Self-Repair 自动修复 |
| **度量方式** | 100% 通过率（布尔） | 数值度量（SCORE=51/59） |
| **回滚机制** | 无自动回滚 | git revert 自动回滚（AutoResearch） |
| **测试计划** | 先写 TEST.md 再写代码（Phase 4→5） | 无显式测试计划 |
| **输出验证深度** | 像素级（PIL/ffprobe/magic bytes） | 结构级（assessResult 4 项检查） |

**核心差异的根源**：

```
OpenCLI 的适配器是确定性代码 → 可以用代码验证代码（assessResult）
CLI-Anything 的 CLI 操控真实软件 → 只能用真实软件验证（pytest + 真后端）

OpenCLI 在云端 CI 跑测试 → 不依赖本地软件
CLI-Anything 在本地跑测试 → 强依赖真实软件安装（GIMP 占 200MB+）

所以：
  OpenCLI 走"代码自动测试" — 因为能在 VM 上跑
  CLI-Anything 走"Agent 按 SOP 手动测试" — 因为只能在本地跑
```

---

## 设计亮点与借鉴

### 1. 渐进式披露（Progressive Disclosure）
HARNESS.md 采用分层设计，核心规则精简，详尽指南按需加载。

### 2. 真实软件硬依赖
坚持"不降级"原则 —— 软件不存在时测试失败，而非跳过。

### 3. Agent-Native 输出
每个命令支持 `--json` 标志，输出机器可解析的结构化数据。

### 4. 统一 REPL 体验
ReplSkin 确保所有生成的 CLI 有一致的交互体验。

### 5. 自包含 Skill 定义
每个 CLI 包内含 SKILL.md，AI Agent 可自动发现和学习工具用法。

---

## 局限性与挑战

1. **强依赖大模型**：需要前沿级模型（如 Claude Opus 4.6）才能可靠生成
2. **需要源码**：无源码时质量大幅下降
3. **需迭代优化**：单次生成可能不完整，需多次 /refine
4. **渲染 Gap**：某些滤镜效果在 CLI 层面难以完美映射

---

## 核心文件速查

| 文件 | 作用 |
|------|------|
| HARNESS.md | 方法论 SOP，7 阶段流水线规范 |
| repl_skin.py | 统一 REPL 界面组件 |
| {software}_backend.py | 真实软件后端调用封装 |
| setup.py | PyPI 发布配置 |
| SKILL.md | AI Agent 技能定义文件 |

---

*文档生成时间：2026-04-13*
*基于 CLI-Anything v0.2.0*
