# CLI-Anything 快速上手

## 环境要求

- **Python**: 3.10+
- **AI 编码助手**: Claude Code / Pi / OpenClaw / Codex / OpenCode / Qodercli / GitHub Copilot CLI 之一
- **目标软件**: 你想要生成 CLI 的那个软件（如 GIMP、Blender、LibreOffice 等）

---

## 安装步骤

### 方式一：通过插件市场（推荐用于 Claude Code）

```bash
# 1. 添加 CLI-Anything 到插件市场
/plugin marketplace add HKUDS/CLI-Anything

# 2. 安装 cli-anything 插件
/plugin install cli-anything

# 3. 重载插件
/reload-plugins
```

### 方式二：手动安装

```bash
# 克隆仓库
git clone https://github.com/HKUDS/CLI-Anything.git

# 手动复制插件（以 Claude Code 为例）
cp -r CLI-Anything/cli-anything-plugin ~/.claude/plugins/cli-anything

# 重载插件
/reload-plugins
```

---

## 构建第一个 CLI（Harness）

### 一行命令生成完整 CLI

```bash
# 为本地 GIMP 源码生成 CLI
/cli-anything ./gimp

# 或从 GitHub 仓库构建
/cli-anything https://github.com/blender/blender
```

这会自动执行完整的 7 阶段流水线：
```
[Analyze] → [Design] → [Implement] → [Test Plan] → [Write Tests] → [Document] → [Publish]
```

### 查看帮助

```bash
# CLI-Anything 命令列表
/cli-anything:list

# 具体命令帮助
/cli-anything --help
```

---

## 使用生成的 CLI

### 安装到系统 PATH

```bash
cd <software>/agent-harness
pip install -e .

# 验证安装
which cli-anything-<software>
```

### 双模式使用

#### 模式 1：子命令模式（适合脚本/管道）

```bash
# 创建项目
cli-anything-gimp project new --width 1920 --height 1080 -o poster.json

# 添加图层
cli-anything-gimp --json layer add -n "Background" --type solid --color "#1a1a2e"

# 导出
cli-anything-gimp export render output.png --width 1920 --height 1080
```

#### 模式 2：REPL 交互模式（适合 AI Agent）

```bash
# 直接运行进入 REPL
cli-anything-blender

# 输出示例：
# ╔══════════════════════════════════════════╗
# ║ cli-anything-blender v1.0.0             ║
# ║ Blender CLI for AI Agents                ║
# ╚══════════════════════════════════════════╝

blender> scene new --name ProductShot
✓ Created scene: ProductShot

blender[ProductShot]> object add-mesh --type cube --location 0 0 1
✓ Added mesh: Cube at (0, 0, 1)

blender[ProductShot]*> render execute --output render.png --engine CYCLES
✓ Rendered: render.png (1920×1080, 2.3 MB) via blender --background

blender[ProductShot]> exit
Goodbye! 👋
```

### JSON 输出模式（AI Agent 专用）

```bash
# 所有命令都支持 --json 标志
cli-anything-libreoffice --json document info --project report.json

# 输出示例：
{
  "name": "Q1 Report",
  "type": "writer",
  "pages": 1,
  "elements": 2,
  "modified": true
}
```

---

## 迭代优化 CLI

初始生成可能不完整，需要迭代优化：

```bash
# 全面优化 —— 分析所有能力的差距
/cli-anything:refine ./gimp

# 聚焦优化 —— 针对特定功能领域
/cli-anything:refine ./gimp "batch processing and filters"
```

每次 `/refine` 都是增量的、非破坏性的。

---

## 测试生成的 CLI

```bash
# 运行测试套件
cd <software>/agent-harness
python3 -m pytest cli_anything/<software>/tests/ -v

# 强制使用已安装的 CLI（而非源码模块）
CLI_ANYTHING_FORCE_INSTALLED=1 python3 -m pytest cli_anything/<software>/tests/ -v -s
```

测试输出示例：
```
================================ Test Summary ================================
gimp 107 passed ✅ (64 unit + 43 e2e)
blender 208 passed ✅ (150 unit + 58 e2e)
──────────────────────────────────────────────────────────────────────────────
TOTAL 2,120 passed ✅ 100% pass rate
```

---

## 常见工作流示例

### 工作流 1：用 Blender 创建产品渲染图

```bash
# 进入 REPL
cli-anything-blender

# 创建场景
blender> scene new --name ProductShot
✓ Created scene: ProductShot

# 添加物体
blender[ProductShot]> object add-mesh --type cube --location 0 0 1
✓ Added mesh: Cube at (0, 0, 1)

# 添加材质
blender[ProductShot]> material add --name "Metal" --type metallic
✓ Added material: Metal

# 渲染
blender[ProductShot]*> render execute --output render.png --engine CYCLES
✓ Rendered: render.png (1920×1080, 2.3 MB)

blender[ProductShot]> exit
```

### 工作流 2：用 LibreOffice 生成 PDF 报告

```bash
# 创建 Writer 文档
cli-anything-libreoffice document new -o report.json --type writer

# 添加内容
cli-anything-libreoffice --project report.json writer add-heading -t "Q1 Report" --level 1
cli-anything-libreoffice --project report.json writer add-paragraph -t "Summary of achievements..."
cli-anything-libreoffice --project report.json writer add-table --rows 4 --cols 3

# 导出为 PDF（调用真实 LibreOffice）
cli-anything-libreoffice --project report.json export render output.pdf -p pdf --overwrite
✓ Exported: output.pdf (42,831 bytes) via libreoffice-headless
```

### 工作流 3：用 Draw.io 绘制架构图

```bash
# 创建图表
cli-anything-drawio diagram new --name http_flow -o http.json

# 添加形状
cli-anything-drawio --diagram http.json shape add-rect --x 100 --y 100 --w 120 --h 60 --label "Client"
cli-anything-drawio --diagram http.json shape add-rect --x 400 --y 100 --w 120 --h 60 --label "Server"

# 添加连接线
cli-anything-drawio --diagram http.json connector add --from Client --to Server --label "HTTP"

# 导出为 PNG
cli-anything-drawio --diagram http.json export png output.png --scale 2
```

---

## CLI-Hub：社区 CLI 市场

AI Agent 可自主发现和安装社区贡献的 CLI：

```bash
# 安装 CLI-Hub 元技能（OpenClaw）
openclaw skills install cli-anything-hub

# 然后让 Agent 自动探索和安装
">Find appropriate CLI software in CLI-Hub and complete the task: ..."
```

---

## 依赖软件安装参考

| 软件 | 安装命令（Ubuntu/Debian） |
|------|--------------------------|
| GIMP | `apt install gimp` |
| Blender | `apt install blender` |
| Inkscape | `apt install inkscape` |
| LibreOffice | `apt install libreoffice` |
| Audacity | `apt install audacity sox` |
| OBS Studio | `apt install obs-studio` |
| Shotcut | `apt install shotcut` |
| FFmpeg | `apt install ffmpeg` |

---

## 下一步建议

1. **阅读 HARNESS.md**：深入理解 7 阶段流水线和方法论
   - 路径：`cli-anything-plugin/HARNESS.md`

2. **选择一个软件开始**：
   - 有源码？直接用 `/cli-anything` 生成
   - 无源码？找一个有源码的替代品（如用 Shotcut 代替 Premiere）

3. **运行测试**：
   ```bash
   python3 -m pytest cli_anything/<software>/tests/ -v
   ```

4. **阅读生成的代码**：
   - `cli_anything/<software>/core/` — 核心业务逻辑
   - `cli_anything/<software>/utils/<software>_backend.py` — 后端调用
   - `cli_anything/<software>/<software>_cli.py` — CLI 入口

5. **参与贡献**：
   - 提交新的 CLI harness 到社区
   - 改进现有 harness 的测试覆盖率
   - 扩展方法论（提交 PR 到 HARNESS.md）

---

## 故障排查

### Q: `/cli-anything` 命令找不到？
```bash
# 重新加载插件
/reload-plugins

# 或检查插件是否已安装
/plugin list
```

### Q: 测试失败 "software not found"？
```bash
# 确保目标软件已安装
which <software>

# Ubuntu/Debian 安装示例
apt install <software>
```

### Q: pip install -e . 失败？
```bash
# 检查 setup.py 语法
python3 setup.py check

# 或使用 pip verbose 模式
pip install -e . -v
```

### Q: REPL 模式无法启动？
```bash
# 确保使用正确的命令名
which cli-anything-<software>

# 直接指定子命令模式
cli-anything-<software> project new ...
```

---

*文档生成时间：2026-04-13*
*基于 CLI-Anything v0.2.0*
