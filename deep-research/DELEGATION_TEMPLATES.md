# Delegation Templates — Delegate 任务模板

> 本文件汇总 `deep-research` 总指挥派发任务时使用的 5 个核心模板。第一个模板是**完整版（启动文献调研）**，其余 4 个为差异点说明（goal / context / toolsets 与第一个模板的区别）。

---

## 通用 SubAgent 约束（每次 delegate 必含）

每次 delegate context 都必须包含下列声明（详见 [STATE_PROTOCOL.md](./STATE_PROTOCOL.md) 末尾）：

> 重要约束: 你是 SubAgent，不能直接与用户交互。如果遇到超出能力的问题，在输出中标注 [需升级] 并说明原因，由总指挥决定是否升级到用户。
>
> 输出末尾必须附「产出文件清单」段，列出所有写入文件的绝对路径，便于总指挥更新 checkpoint。
>
> 修订模式时（如适用）：context 中会附带"基线文件路径 + 反馈 ID（F-ID）"。请基于基线最小改动；输出含 diff 摘要 + 新版本路径（命名规则 `*-v{x.y}-R{n}.md`）。

---

## 模板 1：启动文献调研（完整模板）

> ⚠️ 自 v1.8.0 起，文献调研 delegate **必须**在 context 中带 `mode` 字段，明确本次任务属于哪种模式：
> - `mode=draft` (Phase 1.1) - 仅输出研究方向草案 + A 类下载清单，禁止虚构论文
> - `mode=intensive_read` (Phase 1.2) - 仅对 `~/Downloads/essays/` 真实落盘 PDF 写全文笔记
> - `mode=snowball` (Phase 1.2) - 基于已精读论文 references 输出 B 类下载清单
> - `mode=review` (Phase 1.3) - 基于真实笔记池 review 草案，产出 research-design 定稿
> - `mode=synthesis` (Phase 2) - 基于已有笔记池写综述，严禁新检索

### 模板 1a：Phase 1.1 调研草案（mode=draft）

```python
delegate_task(
    goal="围绕 [主题] 输出研究方向草案 + 论文获取建议清单（A 类下载清单）",
    context="""
mode: draft
研究方向: [用户给出的方向]
学科领域: [社科/管理/经济/...]

请加载 /research-literature skill，按 Mode A (draft) 工作流执行。

⚠️ 严格红线（防选题幻觉）:
- 不得声称"已检索到/已读过"任何具体论文
- 不得调 S2/OpenAlex/CNKI 进行真实检索
- 关键学者-代表作信息可基于通识知识给出，但必须标注「信息源：通识/记忆，请用户在下载时核实」
- 输出的 A 类清单中每条 DOI 都要标注「（建议核实）」

可用工具: file（仅写产出文件，不调 web 检索）

重要约束: 你是 SubAgent，不能直接与用户交互。如果遇到超出能力的问题，在输出中标注 [需升级] 并说明原因。

工作步骤: 见 /research-literature 的 Mode A 工作流（Step A1-A4）

输出:
- ~/.hermes/research-state/{slug}/meta/research-direction-draft.md
- ~/.hermes/research-state/{slug}/meta/acquisition-plan.md（含 A 类下载清单）
末尾附「产出文件清单」段，列出绝对路径。
    """,
    toolsets=["file"]
)
```

### 模板 1b：Phase 1.2 论文精读（mode=intensive_read）

```python
delegate_task(
    goal="对 ~/Downloads/essays/ 中尚未精读的 PDF 写结构化全文笔记",
    context="""
mode: intensive_read
课题: [...]
Gbrain 笔记路径: literature/{slug}/note-*

请加载 /research-literature skill，按 Mode B (intensive_read) 工作流执行。

⚠️ 严格红线（防论文幻觉）:
- 仅对 ~/Downloads/essays/ 中真实落盘的 PDF 写笔记
- 必须用 file 工具读 PDF 文本后写笔记（含 ≥2-3 段原文 quote 带页码）
- 不得用摘要冒充全文，不得对未落盘论文写笔记
- 文件名匹配不上清单 → 列入「待人工归类」清单
- PDF 读取失败 → 列入「读取失败」清单，绝不伪造笔记

可用工具: file (读 PDF + 写笔记), terminal（仅 ls 扫描）

输出:
- 每篇 PDF 一个 Gbrain 笔记
- ~/.hermes/research-state/{slug}/literature/intensive-read-round-N.md（精读报告）
末尾附「产出文件清单」段。
    """,
    toolsets=["file", "terminal"]
)
```

### 模板 1c：Phase 1.2 滚雪球扩展（mode=snowball）

```python
delegate_task(
    goal="基于已精读论文 references 输出 B 类下载清单（第 N 轮 / 3）",
    context="""
mode: snowball
round_count: N  # 必填，1 / 2 / 3，硬上限 3 轮
课题: [...]
已精读笔记列表: [note-001, note-002, ...] (DOI / paperId 见各笔记元信息)
本地已下载 PDF: ~/Downloads/essays/*.pdf
上轮 B 类清单: ~/.hermes/research-state/{slug}/literature/extension-needed-round-{N-1}.md (round 1 时为空)

请加载 /research-literature skill，按 Mode C (snowball) 工作流执行。

⚠️ 红线:
- 不得把 references 中的论文当作"已读"，仅输出"建议下载"
- B 类清单中每条都标注「触发原因（被哪几篇本地论文引用）」+ 频次
- 剔除已在 ~/Downloads/essays/ 中的论文 + 上轮已列入但已下载的论文
- 本任务为第 N / 3 轮，N>3 时拒绝执行并 [需升级]

可用工具: terminal (curl S2/OpenAlex), file

输出:
- ~/.hermes/research-state/{slug}/literature/extension-needed-round-N.md
- 含「轮次状态」段（N/3、本轮新增数、上轮新增数）
- 含「后续动作」段（继续 / 自然收敛 / 即将转入 deferred）
末尾附「产出文件清单」段。
    """,
    toolsets=["file", "terminal"]
)
```

> 📌 总指挥在第 3 轮结束后**必须**：
> 1. 把仍未下载的 B 类论文聚合写入 `~/.hermes/research-state/{slug}/literature/extension-deferred.md`
> 2. timeline 追加 `[PHASE] P1.2 done (rounds=3) → P1.3 begin`
> 3. **不再发起第 4 轮 snowball delegate**，直接进入 Phase 1.3

### 模板 1d：Phase 1.3 调研设计 review（mode=review）

```python
delegate_task(
    goal="基于真实笔记池 review Phase 1.1 草案，产出 research-design 定稿",
    context="""
mode: review
课题: [...]
草案路径: ~/.hermes/research-state/{slug}/meta/research-direction-draft.md
笔记池: Gbrain literature/{slug}/* + ~/Downloads/essays/*.pdf
笔记池统计: 全文 N / 摘要 M / 元数据 K

请加载 /research-literature skill，按 Mode D (review) 工作流执行。

⚠️ 红线:
- research-design-v1.md 中每条核心论断必须 anchor 到 ≥1 篇全文等级笔记
- 引用清单中的本地路径必须真实存在于 ~/Downloads/essays/
- 不得引用笔记池外的论文

可用工具: file

输出:
- ~/.hermes/research-state/{slug}/meta/research-design-v1.md
末尾附「产出文件清单」段。
    """,
    toolsets=["file"]
)
```

### 模板 1e：Phase 2 系统性综述（mode=synthesis）

```python
delegate_task(
    goal="基于已有笔记池撰写系统性文献综述",
    context="""
mode: synthesis
课题: [...]
笔记池路径: Gbrain literature/{slug}/*
研究设计: ~/.hermes/research-state/{slug}/meta/research-design-v1.md

请加载 /research-literature skill，按 Mode E (synthesis) 工作流执行。

⚠️ 红线:
- 严禁新检索 + 引用笔记池外论文
- 综述每个段落末尾必须列出引用的笔记 ID
- 缺少引用支撑的论断 → 标 [待补充全文笔记]，不要泛泛陈述

可用工具: file

输出:
- Gbrain literature/{slug}/review-draft-v1
- ~/.hermes/research-state/{slug}/literature/review-draft-v1.md
末尾附「产出文件清单」段。
    """,
    toolsets=["file"]
)
```

---

## 模板 1（旧版兼容）：完整启动文献调研模板

> 用于非分阶段的简化场景。生产环境优先使用 1a-1e 分阶段模板。

```python
delegate_task(
    goal="围绕 [主题] 进行初步文献检索，识别核心理论、关键学者、主要方法和研究缺口",
    context="""
研究方向: [用户给出的方向]
学科领域: [社科/管理/经济/...]
时间范围: 近 5-10 年为主
语言: 中英文均可

请加载 /research-literature skill 指导你的工作。

可用工具:
- /cnki-paper-downloader: 从 CNKI 下载中文论文全文，输入为论文的完整标题，每次仅一篇
- terminal: 调用 Semantic Scholar / OpenAlex API 进行英文论文发现、摘要获取、引文图谱遍历，以及 OA PDF 下载
- web 工具: Google Scholar 辅助、确认论文元数据
- file: 保存笔记和综述

重要约束: 你是 SubAgent，不能直接与用户交互。如果遇到超出能力的问题，在输出中标注 [需升级] 并说明原因，由总指挥决定是否升级。

工作步骤:
1. 设计搜索策略（关键词 + 布尔组合）
2. 文献发现（双通道）:
   - 中文: web 搜索 site:cnki.net + 关键词
   - 英文: terminal curl S2 / OpenAlex API（含引文图谱滚雪球）
3. 论文获取（按等级）:
   - CNKI 中文全文 → /cnki-paper-downloader
   - OA PDF 全文 → wget/curl 下载
   - 仅摘要 → S2/OpenAlex abstract 字段
   - 元数据节点 → 仅记录基本信息
4. 精读已下载论文，写结构化笔记（含来源等级、外部 ID、OA 链接）
5. 整合为初步文献综述报告
6. 识别 3-5 个研究缺口

输出: 一份完整的初步文献综述报告 (Markdown)，含文献获取状态表（来源 + 等级），**末尾附「产出文件清单」段**列出所有写入的笔记 / 综述文件绝对路径以便总指挥更新 checkpoint。

**修订模式时（如适用）**: context 中会附带反馈 ID（F-ID）和"针对哪条反馈做哪种检索"。请在产出笔记的元信息中标注触发反馈 ID，便于追溯。详见 /research-literature 中"修订模式约束"。
    """,
    toolsets=["web", "file", "terminal"]
)
```

---

## 模板 2：并行多方向检索（差异点）

- **goal**: 拆分为多条 `goal="检索 [理论A/B/变量C] 相关文献"`
- **context**: 同模板 1 框架，但每条任务的研究主题/搜索范围不同；保留 SubAgent 约束 + 「产出文件清单」要求
- **toolsets**: 同 `["web", "file", "terminal"]`
- **额外注意**: 并行任务控制在 **3 个以内**

调用示例（结构）：
```python
delegate_task(tasks=[
    {"goal": "检索 [理论A] 相关文献", "context": "<同模板 1 简化版>", "toolsets": ["web", "file", "terminal"]},
    {"goal": "检索 [理论B] 相关文献", "context": "<同模板 1 简化版>", "toolsets": ["web", "file", "terminal"]},
    {"goal": "检索 [变量C] 实证研究", "context": "<同模板 1 简化版>", "toolsets": ["web", "file", "terminal"]},
])
```

---

## 模板 3：数据分析（差异点）

- **goal**: `"对研究数据执行完整的统计分析"`
- **context 关键差异**:
  - 加载 skill: `/research-analysis`
  - 数据说明: 数据来源、样本量、变量列表
  - 分析要求: 描述性统计 / 信效度（Cronbach's α, CFA） / 相关分析 / 假设检验（回归/SEM/中介） / 稳健性
  - 假设列表（H1/H2/H3...）+ 数据文件位置
  - **修订模式**: 必须附基线分析报告路径 + 反馈 ID；最小改动 + diff 摘要
- **toolsets**: `["terminal", "file"]`（无需 web）

---

## 模板 4：论文撰写（差异点）

- **goal**: `"撰写论文 [章节]"`
- **context 关键差异**:
  - 加载 skill: `/research-writing`
  - 论文主题 / 本章任务 / 字数要求
  - 参考资料: 文献综述路径 + 数据分析结论路径 + 理论框架描述
  - 引文格式: APA 7th
  - **强制（防幻觉）**: 每个引用必须在草稿末尾"引用清单"列出（含笔记路径、来源等级、引用页/段）。详见 /research-writing 中"引用规则（防幻觉）"
  - **修订模式**: 必须附"基线文件路径 + 反馈 ID"；不得整章重写；输出必须含 diff 摘要 + 新版本路径（`*-v{x.y}-R{n}.md`）
- **toolsets**: `["file", "web"]`

---

## 模板 5：质量审查 / 学术顾问评审（差异点）

### 5a. 质量审查
- **goal**: `"审查论文 [章节/全文]"`
- **context 关键差异**:
  - 加载 skill: `/research-review`
  - 待审内容（文件路径或直接内容） + 审查重点（逻辑/引文/方法/格式/全面） + 本轮审查编号
  - **强制（防幻觉）**: 必须先执行"引用真实性核查"4 个 Step（清单完整性 / 笔记存在性 / 来源等级匹配 / 关键论断原文比对），再做内容审查
  - **修订模式**: 校验范围限定为变更段落 + 全文新增引用（不重审已通过部分）
- **toolsets**: `["file", "web"]`
- **输出**: 结构化审查报告（问题分级 🔴严重/🟡中度/🟢轻微，含引用真实性核查结果）

### 5b. 学术顾问评审
- **goal**: `"对 [交付物] 进行独立学术评审"`
- **context 关键差异**:
  - 加载 skill: `/research-advisor`
  - 评审对象（文献综述/理论框架/全文终稿）+ 研究问题 + 理论基础 + 研究方法
  - **重点（对抗性审查）**: 必须执行 Devil's Advocate 角色，回答 4 个对抗性问题（替代解释 / 证伪路径 / 样本边界 / 理论选择），并列出攻击点（🔴致命/🟡严重/🟢警告）
  - **修订模式**: 对抗性审查聚焦"修订段落是否真正解决了反馈"，并复检是否引入新问题
- **toolsets**: `["file", "web"]`
- **输出**: 学术评审报告（评级 A/B/C/D + 对抗性审查章节 + 改进建议）
