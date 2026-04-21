# 开发指引文档审查报告

## 审查概要

- **审查时间**：2026-04-20  
- **文档范围**：`开发指引/` 目录下 `README.md` 与 `01`–`09` 共 10 个 Markdown 文件（**不含**本报告文件）  
- **总行数**：5612 行（与 `wc -l` 汇总一致）  
- **对照源码仓库**：`/Users/yuhaofei/Documents/coding/github/AutoJs6/`（以该路径下 `ScriptRuntime.kt`、`runtime/api/augment/**` 与 `Global.kt` 为准）  
- **整体评价**：文档体系完整、与源码对照紧密，多数模块具备「概述 → 源码定位 → API 总览 → 详细说明」链条，交叉引用（如 01 → 09）与 `ScriptRuntime.augment()` 行号引用总体可用。存在少量**数量表述偏差**、**个别全局 API 未写入总表**、**09 文档机器绝对路径**、以及 **`Auto.kt` 部分方法未在 01 中枚举**等可改进点。

---

## 1. 完整性审查

### 通过项

- **README 导航**：表中列出的 9 个子文档（`01-自动化操控.md` … `09-类型系统参考.md`）均在同一目录下存在，相对链接形式（如 `[01-自动化操控](01-自动化操控.md)`）正确。  
- **`ScriptRuntime.augment()` 注册**：`private fun augment(target: ScriptableObject)` 位于 `ScriptRuntime.kt` **第 737–820 行**（含收尾 `augmentedApp.defineProp(...)`）。各注册项（含 `assignWithRuntime` / `proxying` / 嵌套 `Ocr*`、`NoticeChannel`、`BytesCvt`/`BytesFmt`、`util` 子挂载等）在 01–09 中均可找到对应说明或归入合理章节（例如 `Util*` 与 `MorseCode` 在 **08**、`GlobalLegacy`/`Species`/`IsNullish` 在 **01**）。  
- **README 六、七节** 指向上一级的 `AutoJs6_快速上手.md`、`AutoJs6_源码解读.md`：在 `github-analysis/AutoJs6/` 目录下**确实存在**这两份文件，相对路径 `../` 有效。

### 问题项

1. **`augment/` 子包数量**：源码下 `runtime/api/augment/` 的**一级子目录**为 **50 个**（`app` … `zip` 等），与 README 中「约 **40** 个子包」不一致。建议改为「约 50 个」或给出精确计数并注明是否含仅含辅助类、未在 `ScriptRuntime` 中单独注册者（如 **`proxy`** 主要为代理/辅助实现，**无**独立 `augment` 注册调用）。  
2. **「约 50 个模块」**：`augment()` 内**注册调用**（含 `augmentWithRuntime`、`proxying`、嵌套挂载）远多于 50；若指「顶层全局对象/命名空间」数量，建议在 README 中**定义计数口径**，避免与「约 50 个」字面冲突。  
3. **`08-工具与扩展.md`** 写「`ScriptRuntime.kt` 约 **737–817** 行」：实际 `augment()` 方法体至 **820 行** 结束；建议与 **08** 内其它「约 737–820」类表述对齐。

---

## 2. 源码路径准确性

### 抽查结果

以下路径在 `/Users/yuhaofei/Documents/coding/github/AutoJs6/` 中**存在**（抽查 ≥10 项）：

| 文档引用 | 验证 |
|----------|------|
| `runtime/api/augment/Augmentable.kt` | 存在 |
| `runtime/api/augment/global/Global.kt` | 存在（实际包路径为 `.../augment/global/Global.kt`） |
| `engine/RhinoJavaScriptEngine.kt` | 存在 |
| `core/accessibility/SimpleActionAutomator.kt` | 存在 |
| `runtime/api/augment/http/Http.kt` | 存在 |
| `runtime/api/augment/cryptyo/Crypto.kt` | 存在（文档已说明历史拼写 `cryptyo`） |
| `runtime/api/augment/plugins/Plugins.kt` | 存在；**07** 中代码引用 `18:20` 与当前 `selfAssignmentFunctions` 位置一致 |
| `core/accessibility/UiSelector.kt` | 存在 |
| `core/image/ImageWrapper.kt` | 存在 |
| `core/image/Colors.kt` | 存在（**09** 中「`Colors` 工具」指向 `core/image/Colors.kt` 合理） |

**`ScriptRuntime.kt` 行号**：README 与多份子文档采用的 **737–820**（或 **739–741**、**756–757** 等子段）与当前源码**一致**（允许 ±5 行误差的要求下为**通过**）。

---

## 3. API 覆盖度

### Global.kt（`selfAssignmentFunctions` / `selfAssignmentGetters`）

- **Getters**（`WIDTH`、`HEIGHT`、`axios`、`cheerio`、`dayjs`、`i18n`）：**01** 中「API 列表总览」与专节均有覆盖。  
- **Functions**：**01** 对 `sleep`、`exit`、`wait*`、`cX`/`cY`/`cYx`/`cXy`、类型工具函数、`TODO` 等覆盖较全；与 `Global.kt` 列表对照时：  
  - **`toString`**：在 `selfAssignmentFunctions` 中为首项（`"toString" to AS_LITERAL_TO_STRING`），**01** 的表格与「其它已注册函数」小结中**未单独列出**，属**轻微遗漏**（行为为 `Global.toString` → `"[object global]"`）。  
- **`selfAssignmentProperties`**：`isAutoJs6 = true` 在 **README** 桥接机制处有说明；**01** 全局 API 表未强调，若读者只读 01 可能忽略。

### 其它模块抽查

- **`Http.kt`**（`selfAssignmentFunctions`：`buildRequest`、`request`、`get`、`post*`、`delete`/`del` 等）：**06**「API 列表总览」与 `Http.kt` 公开列表**基本一致**，并注明无 `http.patch` 的变通写法。  
- **`Auto.kt`**：`selfAssignmentFunctions` 含 `start`、`stop`、`enable`、`disable`、`registerEvent`、`registerEvents`、`removeEvent`、`removeEvents`、`hasInstance`、`hasService`、`exists`、`isRunning`、`isOperational`、`stateListener`、`waitFor`、`setMode`、`setFlags`、`setWindowFilter`、`launchSettings`、`clearCache` 及 `currentPackage`/`Activity`/`Component` 等。**01**「`auto`（`Auto.kt`）」详述中列举了 `waitFor`、`setMode`、`setFlags`、`setWindowFilter`、`current*`，但**未系统列出** `registerEvent*`、`removeEvent*`、`hasInstance`、`hasService`、`exists`、`stateListener`、`launchSettings`、`clearCache`、`enable`/`disable`/`start`/`stop` 等，与源码**完整列表相比存在缺口**（属 **API 覆盖度** 方面的主要问题）。

---

## 4. 结构一致性

### 检查结果

- **01–06**：普遍具备文档级 **「概述」+「源码定位总表」**（或等价总表），再分模块的 **「模块概述」→「源码定位」→「API 列表总览」→「详细 API」**，与 README 声明的**总分结构**高度一致。  
- **07**：无单独的文档级 `## 源码定位总表`，改为**按模块分章**（Engines、Tasks、…），每章内含 **模块概述 / 源码定位 / API 列表总览 / 详细 API**，结构仍清晰，但与 01–06 的「单总表」形式**不完全统一**。  
- **08**：采用 **「一、二、三」大节 + 小节**（如 Polyfill、Jsox、OpenCC…），各小节多数具备概述/源码定位/API 表，但层级与 01–06 **不完全同构**（属风格差异，非错误）。  
- **09**：使用 **「类型总览表」** 替代「源码定位总表」，章节为 **「类型概述」+「源码定位」** 及分类型方法表，定位为**参考手册型**，与 API 教程型子文档**略有差异**，合理。  
- **交叉引用**：**01** 在 UiSelector 等处指向 **`09-类型系统参考.md`**，链接有效。

---

## 5. 内容质量

### 评价

- **占位与空白**：未发现大面积空白章节；部分文档如实标注官网「待补充」或实现中的 **FIXME/占位**（如 **07** continuation、**08** Formatter 占位、**03** Barcode/QR 以源码为准），属**诚实说明**而非劣质占位。  
- **示例代码**：**06** HTTP、**01** sleep/wait、**04** files 等示例与源码行为（如 `random` 单参返回 `NaN`）多处有**实现注意**标注，质量较好。  
- **参数描述**：主要 API 有参数/返回值说明；**Auto.kt** 未列全方法时，读者需直接读源码，影响「参数完整性」体验。  
- **可移植性**：**09** 开头写明「源码根目录：`/Users/yuhaofei/Documents/coding/github/AutoJs6/`」为**本机绝对路径**，不利于其他克隆路径的读者；建议改为「仓库根目录」相对表述或占位符说明。

---

## 6. 改进建议（按优先级）

1. **高**：在 **01-自动化操控.md** 的 `auto`/`Auto.kt` 小节补充与源码一致的 **方法总览表**（至少列出 `selfAssignmentFunctions` + 重要 getters），或明确指向 `Auto.kt` 行内列表，避免与 `Automator` 全局函数混淆。  
2. **高**：修正 **README** 中 **`augment/` 子包数量**（约 40 → 约 50，并注明 `proxy` 等非注册子包若存在）。  
3. **中**：在 **01** 的 Global 表格或「其它函数」中增加 **`toString`**（及可选 **`isAutoJs6`** 属性）一行，与 `Global.kt` 完全对齐。  
4. **中**：统一 **08** 与 README 中对 **`ScriptRuntime.augment` 行号** 的区间（建议统一为 **737–820**）。  
5. **中**：在 **09** 将绝对路径改为**相对仓库根**的说明，或删除机器相关前缀。  
6. **低**：在 README 中澄清 **「约 50 个模块」** 的统计口径（注册调用次数 vs 全局命名空间数）。  
7. **低**：若需满足「40 个子包均被提及」的严格口径，可在 README 增补一句：**`proxy` 包**为桥接辅助实现，**不**对应独立 `ScriptRuntime` 注册项。

---

*本报告由对照仓库源码与 10 份指引文档逐项审阅生成，客观问题以当时仓库版本为准；后续若 `ScriptRuntime.kt` 或 `Global.kt` 变更，请同步更新文档中的行号与 API 表。*
