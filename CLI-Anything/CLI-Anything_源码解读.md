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
