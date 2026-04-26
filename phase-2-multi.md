# 多 Agent 模式：阶段二

你已完成阶段一，用户已确认认知基础。工作目录中已有 `findings-final.md`（完整收集发现）。

---

## 步骤 1：分析（NC4）

spawn 子 agent 时，读取对应的 `agents/` 下的 prompt 文件作为子 agent 的指令传入。不要将 agent prompt 读入编排者自己的上下文。

### 1a. 并行分析

按 Phase 1 的子问题分组，每组 spawn 一个分析者（`agents/analyst.md`），传入：
- 问题描述
- 该组的子问题（含类型标注）
- `findings-final.md` 的路径（分析者自行读取相关部分）

每个分析者将产出写入 `analysis-{组名}.md`。

所有分析者独立工作，互不可见。

### 1b. 对抗性审查

所有分析者完成后，spawn 挑战者（`agents/challenger.md`），传入：
- 问题描述
- 完整的子问题清单
- 所有 `analysis-{组名}.md` 的路径（挑战者读取**结论部分**）
- `findings-final.md` 的路径（挑战者读取**原始证据**）

**不传分析者的推理链。** 挑战者从 analysis 文件中只读结论、关键证据、不确定性和替代结论，跳过推理过程。这确保挑战视角独立于分析过程。

挑战者将产出写入 `challenge.md`。

### 1c. 验证关卡

裁决之前，编排者扫描所有 `analysis-*.md` 和 `challenge.md` 中标注为 `[hypothesis]` 的声明：

1. **筛选关键假设**：哪些 `[hypothesis]` 是决策的关键依赖？如果它不成立，结论会改变吗？不影响结论的假设可以跳过。
2. **设计验证实验**：对每个关键假设，设计最小化的验证实验（一行命令、一个脚本、一次 API 调用）。
3. **执行验证**：跑实验，记录结果。
4. **更新置信度**：验证通过的假设升级为 `[verified]`，验证失败的假设标注为 `[refuted]`。

验证结果写入 `verification.md`。

**如果存在关键假设无法验证**（具体原因如：没有运行环境、需要用户配合、需要生产数据、依赖尚未上线的接口），必须在后续呈现中显著标注——不能让用户以为方案已经确认可行。无法验证的具体原因需要写清，不要用"等原因"含糊带过。

### 1d. 裁决与修正

编排者读取所有 `analysis-*.md`、`challenge.md` 和 `verification.md`，对每个子结论做裁决：
- **通过**的结论（关键假设全部 `[verified]`）：直接采用
- **需要修正**的结论（有假设被 `[refuted]`）：基于验证结果修正方案
- **不确定**的结论（关键假设无法验证）：标注为"待验证方案"，向用户说明风险
- **组合矛盾**：调整相关子结论使其兼容

裁决结果写入 `conclusion.md`。

## 步骤 2：呈现（NC5）

编排者基于 `conclusion.md` 和 `verification.md` 向用户呈现。

**读取 `presentation.md`**，按其中的原则与结构呈现。深潜与围猎共享同一套呈现纪律。

---

## 持续迭代

用户可能质疑结论、补充新信息、或引向更深的子问题。每轮反馈触发微循环：

1. **识别新信息** — 用户反馈本身就是新信息。他指出了什么错误？暗示了什么遗漏的维度？
2. **定向补充收集** — 穷尽自身能力去查，不推给用户。定向、轻量，不重做全面扫描。
3. **增量更新结论** — 更新 `conclusion.md`，标注"相比上一轮，什么变了、为什么变了"。

如果反馈触及根本性方向问题（目标本身变了），退回阶段一重新对齐。

---

## 报告生成

分析完成后（用户确认最终结论或会话即将结束时），生成可视化报告。

### 前置：确保 `final-report.md` 存在

在工作目录中写入 `final-report.md`——这是报告正文（Part I "报告"页签的内容）。用标准 markdown 格式写，生成器会渲染为 HTML。这是面向决策者的精炼叙述，不是分析过程的重复。

### 生成步骤

读取 `report-template.html`（skill 目录中），替换以下占位符，写入工作目录的 `report.html`：

**1. 元数据占位符（字符串替换）：**
- `__TITLE__`：报告标题（一句话描述问题）
- `__DESC__`：副标题（简要描述分析方法，如"基于多专家 Agent 并行收集、对抗审查与实验验证的深度分析"）
- `__DATE__`：分析日期（YYYY.MM.DD）
- `__METHOD__`：分析模式（"Multi-Agent 围猎模式"）

**2. 报告正文（HTML）：**
- `__REPORT_HTML__`：读取 `final-report.md`，用 markdown 渲染器转为 HTML 后注入。这是报告页签的全部内容。

**3. 分析过程数据（JSON，注入到 `<script type="application/json">` 块）：**

用 Python 脚本扫描工作目录生成，或手动构造：

- `__DOCS_JSON__`：扫描工作目录所有 `.md` 文件（排除 `final-report.md`），每个文件生成 `{file, id, stage, content, lines}`。`id` 从文件名去掉 `.md` 生成，`stage` 从文件名模式推断：
  - `terrain-map*` → `scout`
  - `findings-*-bottom-up*` / `findings-*-top-down*` → `collect`
  - `findings-merged*` → `merge`
  - `gaps*` → `gaps`
  - `analysis-*` → `analysis`
  - `challenge*` → `challenge`
  - `verification*` → `verify`
  - `conclusion*` → `conclude`
  - 其他 → `other`

- `__STAGES_JSON__`：流水线阶段定义数组。`desc` 中的数字按**本次实际运行的 agent 数量**填写（N 组→ collect 填 `"{2N} agents · {N}组×2方向"`，analysis 填 `"{N} agents · 推理与方案设计"`），不要照抄下面示例里的 6/3。如果某阶段本次未执行（例如 N=1 省了 gaps），相应对象可省略：
  ```json
  [
    {"id":"scout","label":"侦察","desc":"1 agent · 信息域地图"},
    {"id":"collect","label":"并行收集","desc":"6 agents · 3组×2方向"},
    {"id":"merge","label":"合并去重","desc":"编排者 · 去重标注来源"},
    {"id":"gaps","label":"缺口扫描","desc":"1 agent · 完备性审查"},
    {"id":"analysis","label":"并行分析","desc":"3 agents · 推理与方案设计"},
    {"id":"challenge","label":"对抗审查","desc":"1 agent · 对抗性质疑"},
    {"id":"verify","label":"验证关卡","desc":"编排者 · 关键假设实测"},
    {"id":"conclude","label":"裁决","desc":"编排者 · 综合结论"}
  ]
  ```

- `__LABELS_JSON__`：文档标签映射 `{docId: {tab: "短标签", title: "完整标题"}}`。为每个文档定义统一格式的标签（不要从 .md 第一行解析，那些格式不统一）。

- `__SESSIONS_JSON__`：扫描 `sessions/` 目录下所有含 `report.html` 的子目录，构造：
  ```json
  [{"title":"...", "date":"...", "method":"...", "path":"../YYYYMMDD-HHmmss/report.html", "current": true/false}]
  ```
  当前 session 标记 `current: true`。按日期倒序排列。
  **同时更新所有已有 report.html 中的 `__SESSIONS_JSON__` 数据**（用正则替换 `<script id="sessions-json"...>` 块的内容），使旧报告的"所有报告"列表也包含新报告。

**4. 告知用户**报告路径，用户可在浏览器中打开。
