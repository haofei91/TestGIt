# CLI-Anything 快速上手

> 项目：HKUDS/CLI-Anything  
> 快速入门文档（5 分钟上手）

---

## 一、环境要求

| 组件 | 要求 |
|------|------|
| **Python** | ≥ 3.10 |
| **目标软件** | 需要自动化的软件（如 GIMP、Blender、LibreOffice 等） |
| **AI Agent 平台** | Claude Code / OpenCode / OpenClaw / Codex / Qodercli / GitHub Copilot CLI 之一 |

---

## 二、安装步骤

### 2.1 选择你的 Agent 平台

#### Claude Code（推荐）

```bash
# 添加插件市场
/plugin marketplace add HKUDS/CLI-Anything

# 安装插件
/plugin install cli-anything
```

#### OpenCode

```bash
# 克隆仓库
git clone https://github.com/HKUDS/CLI-Anything.git

# 复制命令文件（全局）
cp CLI-Anything/opencode-commands/*.md ~/.config/opencode/commands/
cp CLI-Anything/cli-anything-plugin/HARNESS.md ~/.config/opencode/commands/
```

#### OpenClaw

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
mkdir -p ~/.openclaw/skills/cli-anything
cp CLI-Anything/openclaw-skill/SKILL.md ~/.openclaw/skills/cli-anything/SKILL.md
```

#### Codex

```bash
git clone https://github.com/HKUDS/CLI-Anything.git
bash CLI-Anything/codex-skill/scripts/install.sh
```

### 2.2 安装目标软件依赖

根据你要自动化的软件安装对应程序：

```bash
# Debian/Ubuntu
apt install gimp blender libreoffice ffmpeg

# macOS
brew install gimp blender libreoffice ffmpeg
```

---

## 三、快速开始

### 3.1 用 Claude Code 构建 GIMP CLI

```bash
# 进入 Claude Code
claude

# 构建 GIMP 的完整 CLI（7 个阶段自动执行）
/cli-anything ./gimp

# 如果 Claude Code 版本 < 2.x
/cli-anything ./gimp
```

### 3.2 用 OpenCode 构建

```bash
# 从 GitHub 构建
/cli-anything https://github.com/blender/blender

# 从本地路径构建
/cli-anything ./gimp
```

### 3.3 构建后安装使用

```bash
cd gimp/agent-harness
pip install -e .

# 验证安装
which cli-anything-gimp
cli-anything-gimp --help

# 进入交互 REPL
cli-anything-gimp

# 命令行模式
cli-anything-gimp --json project new -o poster.json --width 1920 --height 1080
```

### 3.4 REPL 交互示例

```
$ cli-anything-blender
╔══════════════════════════════════════════╗
║       cli-anything-blender v1.0.0       ║
║     Blender CLI for AI Agents           ║
╚══════════════════════════════════════════╝

blender> scene new --name ProductShot
✓ Created scene: ProductShot

blender[ProductShot]> object add-mesh --type cube --location 0 0 1
✓ Added mesh: Cube at (0, 0, 1)

blender[ProductShot]*> render execute --output render.png --engine CYCLES
✓ Rendered: render.png (1920×1080, 2.3 MB) via blender --background

blender[ProductShot]> exit
Goodbye! 👋
```

### 3.5 JSON 输出模式（Agent 使用）

```bash
# 所有命令都支持 --json 输出
cli-anything-gimp --json project info --project poster.json

# 输出示例
{
  "name": "poster",
  "width": 1920,
  "height": 1080,
  "layers": 1,
  "modified": true
}
```

---

## 四、已预置的 CLI 列表

安装后可用以下命令（以 pip install -e 对应目录的方式）：

| 软件 | 命令 | 后端 |
|------|------|------|
| GIMP | `cli-anything-gimp` | GIMP Script-Fu |
| Blender | `cli-anything-blender` | bpy (Python scripting) |
| Inkscape | `cli-anything-inkscape` | SVG/XML direct manipulation |
| Audacity | `cli-anything-audacity` | sox + Python wave |
| LibreOffice | `cli-anything-libreoffice` | ODF + headless LO |
| OBS Studio | `cli-anything-obs-studio` | obs-websocket |
| Kdenlive | `cli-anything-kdenlive` | MLT XML + melt |
| Shotcut | `cli-anything-shotcut` | MLT XML + melt |
| Zoom | `cli-anything-zoom` | Zoom REST API (OAuth2) |
| MuseScore | `cli-anything-musescore` | mscore CLI (MSCX/MusicXML) |
| Draw.io | `cli-anything-drawio` | mxGraph XML + draw.io CLI |
| ComfyUI | `cli-anything-comfyui` | ComfyUI REST API |
| Ollama | `cli-anything-ollama` | Ollama REST API |
| AdGuard Home | `cli-anything-adguardhome` | AdGuard Home REST API |

---

## 五、配置说明

### 5.1 CLI-Hub 自动发现

CLI-Anything 提供 CLI-Hub 元技能，AI Agent 可以自主发现和安装社区贡献的 CLIs：

```bash
# OpenClaw
openclaw skills install cli-anything-hub

# nanobot
nanobot skills install cli-anything-hub
```

### 5.2 环境变量

```bash
# 强制使用已安装的命令（CI/发布测试用）
CLI_ANYTHING_FORCE_INSTALLED=1 python3 -m pytest ...

# 指定测试输出路径
CLI_ANYTHING_TEST_OUTPUT=/tmp/artifacts
```

---

## 六、最小可运行示例

### 示例：生成一张 GIMP poster 并导出 PNG

```bash
# 1. 构建 GIMP CLI（如尚未构建）
# /cli-anything ./gimp

# 2. 安装
cd gimp/agent-harness && pip install -e .

# 3. 创建海报项目
cli-anything-gimp project new -o poster.json --width 1920 --height 1080

# 4. 添加图层
cli-anything-gimp --project poster.json layer add -n "Background" --type solid --color "#1a1a2e"
cli-anything-gimp --project poster.json layer add -n "Title" --type text --text "Hello CLI" --font "Sans" --size 72 --color "#ffffff"

# 5. 导出 PNG
cli-anything-gimp --project poster.json export image poster.png --format png --overwrite

# 6. JSON 模式查看项目信息
cli-anything-gimp --json project info --project poster.json
```

### 示例：LibreOffice 生成 PDF

```bash
# 构建并安装
# /cli-anything ./libreoffice
cd libreoffice/agent-harness && pip install -e .

# 创建 Writer 文档
cli-anything-libreoffice document new -o report.json --type writer --name "Q1 Report"

# 添加内容
cli-anything-libreoffice --project report.json writer add-heading -t "Q1 Report" --level 1
cli-anything-libreoffice --project report.json writer add-paragraph -t "Introduction paragraph..."

# 导出真实 PDF
cli-anything-libreoffice --project report.json export render output.pdf -p pdf --overwrite
# 实际调用：libreoffice --headless --convert-to pdf
```

---

## 七、常见问题

### Q1: 构建命令报网络错误？
GitHub 连接不稳定时，可以手动克隆：
```bash
git clone https://github.com/HKUDS/CLI-Anything.git
# 然后按上面 OpenCode 的方式复制命令文件
```

### Q2: 目标软件未安装时报错？
CLI-Anything 的后端会明确报错并给出安装指令，例如：
```
RuntimeError: GIMP is not installed. Install it with:
  apt install gimp   # Debian/Ubuntu
  brew install gimp  # macOS
```
**真实软件是硬依赖**，没有优雅降级。

### Q3: 生成的 CLI 缺少某些功能？
运行 refine 命令逐步扩展：
```bash
/cli-anything:refine ./gimp
# 或聚焦特定领域
/cli-anything:refine ./gimp "image batch processing and filters"
```

### Q4: 测试失败？
```bash
# 先确认真实软件已安装
which gimp blender libreoffice

# 强制使用已安装命令进行测试
CLI_ANYTHING_FORCE_INSTALLED=1 python3 -m pytest cli_anything/gimp/tests/ -v -s
```

### Q5: 多个 CLI 能否同时安装？
可以。CLI-Anything 使用 PEP 420 命名空间包架构，所有 CLI 共存于 `cli_anything.*` 命名空间，互不冲突：
```bash
pip install -e gimp/agent-harness
pip install -e blender/agent-harness
pip install -e libreoffice/agent-harness
# 三者同时可用
```

### Q6: Windows 支持？
支持，但部分软件需要 WSL 或 Cygwin：
> **Note for Win Users:** Claude Code runs shell commands via `bash`. On Windows, install Git for Windows (includes `bash` and `cygpath`) or use WSL; otherwise commands may fail with `cygpath: command not found`.

---

## 八、下一步建议

### 源码阅读顺序（推荐）

| 顺序 | 文件 | 说明 |
|------|------|------|
| 1 | `cli-anything-plugin/HARNESS.md` | 方法论总纲，必读 |
| 2 | `cli-anything-plugin/repl_skin.py` | 统一 REPL 界面实现 |
| 3 | `cli-anything-plugin/commands/cli-anything.md` | 7 阶段流水线的具体定义 |
| 4 | `gimp/agent-harness/cli_anything/gimp/gimp_cli.py` | Click CLI 入口点示例 |
| 5 | `gimp/agent-harness/cli_anything/gimp/core/project.py` | 项目状态管理 |
| 6 | `gimp/agent-harness/cli_anything/gimp/core/session.py` | 会话管理 + undo/redo |
| 7 | `gimp/agent-harness/cli_anything/gimp/utils/gimp_backend.py` | 真实软件调用封装 |
| 8 | `gimp/agent-harness/cli_anything/gimp/tests/test_full_e2e.py` | E2E 测试写法参考 |

### 扩展方向

1. **添加新软件支持**：参考 HARNESS.md，为新软件按 7 阶段生成 harness
2. **贡献社区 CLI**：在 `registry.json` 中添加新 CLI 条目，提交 PR
3. **改进测试覆盖**：在现有 harness 中补充 E2E 测试场景
4. **集成新 Agent 平台**：参考 `opencode-commands/` 格式，适配新平台

---

## 九、关键命令速查

```bash
# 构建
/cli-anything <软件路径或GitHub仓库URL>

# 改进
/cli-anything:refine <软件路径> [聚焦领域]

# 测试
/cli-anything:test <软件路径>

# 验证标准
/cli-anything:validate <软件路径>

# 列表（OpenCode）
/cli-anything-list
```

---

> 文档版本：1.0 | 分析日期：2026-03-24  
> 仓库：https://github.com/HKUDS/CLI-Anything  
> 许可证：Apache License 2.0
