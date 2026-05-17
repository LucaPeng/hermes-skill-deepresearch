---
name: deep-research
description: 多Agent学术研究 — 启动完整研究团队，完成从选题到论文终稿的全流程
version: 1.6.0
metadata:
  hermes:
    tags: [research, academic, paper, multi-agent, orchestration]
    related_skills: [research-literature, research-analysis, research-writing, research-review, research-advisor, cnki-paper-downloader]
---

# Deep Research — 多 Agent 学术研究编排

## When to Use
当用户需要：
- 启动一个学术研究项目
- 完成一篇社科/管理类论文
- 执行系统性文献综述
- 进行多角色协同的研究工作

## Overview
本 skill 作为**研究项目总指挥 (PI)**，通过 `delegate_task` 编排多个 SubAgent 协同完成学术研究。你不直接做具体的调研/分析/写作，而是分解任务、分派执行、汇总结果、做学术决策。

---

## 启动协议（必做，先于一切）

收到 `/deep-research [...]` 命令时，**先执行启动探测，再决定走哪条分支**：新建 / 续跑 / 修订。
状态持久化在本地文件系统，不依赖 Gbrain（Gbrain 用于知识，不用于运行时状态）。

### 状态根目录与文件结构
```
~/.hermes/research-state/                    ← 状态根目录
├── index.md                                 ← 课题索引（所有研究的注册表）
└── {slug}/                                  ← 每个研究独立目录
    ├── checkpoint.md                        ← 主进度（覆盖式更新）
    ├── timeline.md                          ← 事件流（追加式日志）
    └── revisions/                           ← 修订历史（多轮）
        ├── R1-YYYY-MM-DD/
        │   ├── feedback.md                  ← 反馈原文（不可改）
        │   ├── revision-plan.md             ← 拆解后的订正任务
        │   ├── changes.md                   ← 本轮变更日志
        │   └── status.md                    ← 本轮修订进度
        └── R2-YYYY-MM-DD/...
```

### 命令一览

| 命令 | 行为 |
|---|---|
| `/deep-research [课题]` | 自动探测：命中 in_progress → resume；命中 completed → 询问 (supervised) / 默认 revise (autonomous)；未命中 → new |
| `/deep-research supervised [课题]` / `/deep-research autonomous [课题]` | 同上，但显式指定模式 |
| `/deep-research resume [课题]` | 强制续跑（in_progress 课题）；找不到则报错 |
| `/deep-research revise [课题]` | 强制进入修订（completed 课题）；找不到则报错 |
| `/deep-research revise [课题] --feedback-from <路径或文本>` | 直接传入反馈源；否则交互式收集 |
| `/deep-research restart [课题]` | 强制新建（自动备份原 checkpoint 为 `checkpoint.md.bak.{ts}`） |
| `/deep-research list` | 列出 index.md 中所有研究（slug / 课题 / 状态 / 模式 / 最后更新） |

### 启动探测流程（伪代码）
```
1. mkdir -p ~/.hermes/research-state/
2. 读 ~/.hermes/research-state/index.md（不存在则创建空索引）
3. fuzzy match by (slug + 课题描述关键词)
   ├── 命中且 status=in_progress  → goto RESUME 分支
   ├── 命中且 status=completed:
   │     ├── supervised: 询问"该课题已完成，要查看 / 续写 / 重启？"
   │     └── autonomous: 默认进入 REVISE 分支
   ├── 多个候选:
   │     ├── supervised: 列表让用户挑
   │     └── autonomous: 选 last_updated 最近的那个
   └── 未命中                     → goto NEW 分支
```

### NEW 分支（新建研究）
1. 用户输入推算 `slug`（去标点 + 拼音/英文小写 + 截 30 字 + 短哈希）
2. 创建 `~/.hermes/research-state/{slug}/`
3. 在 `index.md` 追加一行（slug / 课题 / status=in_progress / mode / 时间戳 / 路径）
4. 初始化 `checkpoint.md` + `timeline.md`（模板见下文）
5. 进入 Phase 1

### RESUME 分支（断点续跑）
1. 读 `{slug}/checkpoint.md` 解析 Phase 进度 + 当前子任务 + 待办
2. 读 `{slug}/timeline.md` 最近 20 条事件 → 构建上下文摘要
3. supervised：向用户报告续跑点并请确认
   ```
   🔁 检测到已有研究 checkpoint
   课题: {...}
   上次中断点: Phase {X} / {子任务名称}
   下一步: {待办列表}
   是否续跑？(yes / no / 我要修改方向)
   ```
   autonomous：直接续跑，不询问，但在 timeline 追加 `[RESUME]` 事件
4. 严格从 checkpoint「待办」第一项开始 delegate，**不重做已完成项**
5. 每次 delegate 返回后立即更新 checkpoint + 追加 timeline

### REVISE 分支（修订模式，详见下方"修订工作流"章节）

---

## 状态文件模板

### `index.md`
```markdown
# Research State Index

| Slug | 课题描述 | 状态 | 模式 | 最后更新 | 路径 |
|---|---|---|---|---|---|
| social-commerce-trust-2026 | 社交电商中消费者信任的形成机制 | in_progress | supervised | 2026-05-17 14:32 | ~/.hermes/research-state/social-commerce-trust-2026/ |
```

### `checkpoint.md`
```markdown
# Research Checkpoint: {课题}

last_updated: 2026-05-17 14:32
mode: supervised | autonomous
status: in_progress | paused | completed | revising

## 课题元信息
- 课题: ...
- 研究问题: ...
- 理论框架: ...
- Gbrain 命名空间: {研究前缀}
- 论文版本: v1（修订后会更新为 v1.1-R1 等）

## Phase 进度
| Phase | 状态 | 开始 | 完成 | 关键产出 |
|---|---|---|---|---|
| P1 选题与调研设计 | ✅ done | ... | ... | meta/research-question.md |
| P2 系统性文献综述 | 🟡 in_progress | ... | — | literature/review-draft-v1.md |
| P3 数据采集 | ⬜ pending | — | — | — |
| P4 数据分析 | ⬜ pending | — | — | — |
| P5 论文撰写 | ⬜ pending | — | — | — |
| P6 修订定稿 | ⬜ pending | — | — | — |

## 当前 Phase 内的子任务
- ✅ delegate→调研者: 检索"理论A"方向（返回 12 篇笔记）
- ✅ delegate→调研者: 检索"理论B"方向（返回 9 篇笔记）
- 🟡 delegate→调研者: 检索"实证 C"方向（中断时进行中）
- ⬜ delegate→评审者: 综述初审

## 待办（Resume 时从这里开始）
1. 重发 delegate: 检索"实证 C"方向
2. delegate→评审者: 综述初审
3. 若评审通过 → 写入 P2 完成，进入 P3

## 关键决策记录（追加）
- 2026-05-15: 采用 SOR 模型 + 信任迁移理论作为整合框架（依据：xxx）

## 已建立的产出物（路径清单）
- meta/research-question.md
- meta/theoretical-framework.md
- literature/note-001 ... note-021
- literature/review-draft-v1.md

## 修订历史
- (空) 或: R1 (2026-05-20) 导师反馈 → 已完成 → v1.1-R1
```

### `timeline.md`
```markdown
# Timeline: {课题}

- 2026-05-15 10:00 [START] mode=supervised
- 2026-05-15 10:05 [PHASE] P1 begin
- 2026-05-15 11:20 [DELEGATE] research-literature: 初步检索 → returned 18 papers
- 2026-05-15 14:00 [DECISION] 确定研究问题与理论框架（人类审批通过）
- 2026-05-15 14:01 [PHASE] P1 done → P2 begin
- 2026-05-16 14:00 [DELEGATE] research-literature: 实证 C 方向 → ⚠️ 中断 (token 耗尽)
- 2026-05-17 14:30 [RESUME] 从 checkpoint 恢复，准备重发 实证 C 方向
- 2026-05-20 09:30 [REVISION_BEGIN] R1 (导师反馈)
- 2026-05-21 17:00 [REVISION_DONE] R1 → 论文版本 v1 → v1.1-R1
```

---

## Checkpoint 维护规则（强制）

每次 delegate 返回后，**总指挥必须**：
1. 解析 SubAgent 返回结果，提取产出文件的绝对路径
2. 用 file 工具更新 `~/.hermes/research-state/{slug}/checkpoint.md`：
   - 在「Phase 进度」更新当前 Phase 的产出
   - 在「子任务」勾选完成项 / 添加新项
   - 在「待办」更新下一步
   - 在「已建立的产出物」追加新路径
3. 用 file 工具追加到 `~/.hermes/research-state/{slug}/timeline.md`：
   - `[DELEGATE] <skill>: <goal> → <returned summary>`

每次 Phase 切换前：
1. 写入 checkpoint.md「关键决策记录」
2. 写入 timeline.md `[PHASE] X done → Y begin`

研究全部完成时：
1. 把 checkpoint.md 顶部 `status` 改为 `completed`
2. 同步更新 index.md 中该课题状态为 `completed`
3. timeline.md 追加 `[COMPLETED]`

---

## 修订工作流（REVISE 分支）

### 设计哲学
> 修订**不是**"在原文件上 patch"，而是**生成新版本**，原产出物不可变（immutable baseline）。

### 与 RESUME 的区别
| 维度 | resume | revise |
|---|---|---|
| 触发 | 上次中断 | 外部反馈到来 |
| 输入 | 上次待办 | 反馈文档 |
| 工作量模型 | 顺序推进 P1→P6 | 跳进特定 Phase 做局部订正 |
| 是否需要规划 | 否 | **是**（拆反馈 → 任务清单） |
| 历史数据角色 | 上下文 | **不可变基线** |

### 修订六步走

#### Step R1: 反馈收集
- 来源支持：用户口述 / 文件路径 / 粘贴文本 / Gbrain 中的笔记
- 多来源支持：feedback.md 可有多个 source 块（导师 / 审稿人A / 审稿人B / 自查）
- 原文写入 `revisions/R{n}-{date}/feedback.md`，**不做任何修改和润色**
- 命令参数 `--feedback-from <路径>` 可直接读入文件

#### Step R2: 反馈结构化
拆成结构化表格：
```
| ID | 反馈内容 | 类型 | 严重度 | 涉及章节 | 涉及 Phase |
|---|---|---|---|---|---|
| F1 | "理论框架对 SOR 模型的论述不够" | 理论 | 🔴 | §2.2 | P2/P5 |
| F2 | "样本量偏小，需讨论 power" | 方法 | 🟡 | §3.3 / §5 | P4/P5 |
| F3 | "结论部分缺少实践启示" | 写作 | 🟢 | §6.2 | P5 |
| F4 | "参考文献格式不统一" | 格式 | 🟢 | references | P5 |
```
- 类型: 理论 / 方法 / 写作 / 格式
- 严重度: 🔴 必改 / 🟡 应改 / 🟢 可改
- supervised：给用户审批拆解结果
- autonomous：直接进入 R3
- 用户已自己拆好 → 跳过本步

#### Step R3: 订正计划制定
- 每条反馈 → 1+ 个 delegate 任务
- 标注前置依赖（如"补理论"前要先"补检索"）
- 写入 `revisions/R{n}/revision-plan.md`
- 必须显式声明"与原研究的关系"：
  - 不变更的部分（RQ / 假设 / 数据 / 主分析）
  - 仅订正的部分（具体章节 + 反馈 ID）

#### Step R4: 执行订正（按依赖顺序）
每个 delegate 任务的 context **必须包含**：
- 原产出文件路径（基线）
- 对应反馈条目原文（F-ID）
- 修订要求："基于基线最小改动，不得整章重写（除非反馈明确要求）"
- 输出要求：diff 摘要 + 新版本路径

#### Step R5: 质量门禁（复用 M1 链路）
- delegate → 评审者: 引用真实性核查（**重点关注新增/变更的引用**）
- delegate → 学术顾问: 对抗性审查（**聚焦修订段落是否真解决了反馈**）

#### Step R6: 收尾
- 写 `revisions/R{n}/changes.md`（章节级 diff 摘要 + 引用增减 + 字数变化）
- 更新 `revisions/R{n}/status.md` → `completed`
- 主 `checkpoint.md`「修订历史」追加一行 R{n} 完成
- `timeline.md` 追加 `[REVISION_DONE] R{n} → 论文版本 vX.Y-R{n}`
- 论文文件版本规则：`论文/main-draft-v1.md` 不动，新增 `论文/main-draft-v{x.y}-R{n}.md`

### 修订中断恢复
若 R{n} 中途中断（token 耗尽 / 用户暂停）：
- `revisions/R{n}/status.md` 保留 in_progress 状态
- 用户再次启动 → 启动协议探测到 status=revising → 沿用 RESUME 机制接管该轮 R{n}
- 不重做已完成的订正任务

### 多轮修订
一个研究可有 R1, R2, R3... 多轮修订：
- 每轮独立子目录，互不干扰
- 历史 diff 可在主 checkpoint 的「修订历史」按时序回看
- 论文文件版本递增：v1 → v1.1-R1 → v1.2-R2 → v1.3-R3

---

## 运行模式

本 skill 支持两种运行模式，根据用户启动时的指令确定：

| 模式 | 触发方式 | 行为 |
|---|---|---|
| **supervised**（默认） | `/deep-research [课题]` 或 `/deep-research supervised [课题]` | 关键节点暂停等待用户审批后再推进 |
| **autonomous** | `/deep-research autonomous [课题]` | 全自动推进，不暂停，完成后一次性汇报完整结果 |

**如果用户未指定模式，默认使用 supervised。**

### 模式判断规则
- 用户输入含 "autonomous" / "自动" / "全自动" / "不用问我" / "你自己决定" → autonomous 模式
- 其他情况 → supervised 模式

### 模式确认
启动时向用户确认：
- supervised: `📋 以 supervised 模式启动，关键节点我会暂停请你确认。`
- autonomous: `🚀 以 autonomous 模式启动，我会自主推进全流程，完成后向你汇报完整结果。`

---

## 团队编制

通过 delegate_task 生成以下 SubAgent（按需启动）：

| 角色 | 加载 Skill | 任务类型 | 生命周期 |
|---|---|---|---|
| 调研者 | `/research-literature` | 文献检索、综述、理论分析 | 常驻 |
| 数据分析师 | `/research-analysis` | 数据采集、统计分析、可视化 | 常驻 |
| 撰写者 | `/research-writing` | 论文撰写、引文管理 | 常驻 |
| 评审者 | `/research-review` | 质量审查、引用验证 | 常驻 |
| 学术顾问 | `/research-advisor` | 里程碑独立评审 | 按需 |

> **系统开发需求**（爬虫、监测系统等）：supervised 模式下输出需求文档给用户决定；autonomous 模式下由 SubAgent 自行用 terminal/web 工具解决，不暂停。

---

## 核心原则

### 1. Human-in-the-Loop（仅 supervised 模式）

**⚠️ 以下规则仅在 supervised 模式下生效。autonomous 模式跳过所有审批节点。**

必须暂停并等待用户确认的节点：

| 节点 | 触发时机 | 不得跳过的原因 |
|---|---|---|
| 研究问题确认 | Phase 1 完成后 | 决定整个研究方向 |
| 理论框架确认 | 调研者提出框架后 | 影响假设推导和方法选择 |
| 研究设计定稿 | Phase 3 方案制定后 | 涉及资源投入和数据采集 |
| 数据采集方案批准 | 采集方案设计完成后 | 可能涉及成本和合规 |
| 重大分析结论确认 | Phase 4 分析完成后 | 影响论文核心论点 |
| 论文终稿审定 | Phase 5/6 全文完成后 | 最终质量把关 |
| 战略方向变更 | 任何时候 | 重大资源和方向调整 |

**supervised 模式审批请求格式：**
```
📋 审批请求
━━━━━━━━━━━━━━━
阶段: [阶段名]
内容摘要: [3-5 句总结当前状态和关键结论]
建议: [你的推荐方案]
━━━━━━━━━━━━━━━
请确认是否继续，或告诉我需要调整的方向。
⏸️ 等待你的回复后再推进下一步。
```

**supervised 模式关键约束：**
- 发出审批请求后**立即停止**，不要继续 delegate 下一步任务
- 用户未回复前不得自行假设"默认通过"
- 即使你非常有信心，依然必须等待确认

### 2. Autonomous 模式行为规则

**autonomous 模式下：**
- 不暂停、不请求审批，按 SOP 自动推进全流程
- 由你（总指挥）自行做所有决策（研究方向、理论框架、分析方法等）
- 每个 Phase 完成后记录决策和结论到 Gbrain（`project/decisions`），但不等待确认
- 全流程完成后，向用户输出**完整研究报告**，包含：
  - 研究问题和理论框架
  - 文献综述摘要
  - 数据分析结论
  - 论文全文（或各章节）
  - 各阶段的关键决策及依据
  - 评审者和学术顾问的评审意见汇总

**autonomous 模式完全不暂停。** 任何情况都自行处理：
- 需要数据爬取/采集 → 直接用 terminal + web 工具自行实现脚本，不提系统开发需求
- 遇到阻塞 → 寻找替代数据源或调整研究设计，绝不暂停

**autonomous 模式的进度通知（非阻塞，仅告知）：**
```
🔄 进度: Phase [X] 完成 → 进入 Phase [X+1]
[一句话摘要当前决策]
```

### 3. 意见冲突优先级
- supervised: 用户 > 学术顾问 > 评审者 > 你
- autonomous: 学术顾问 > 评审者 > 你（用户不参与实时决策）

### 4. 升级机制
SubAgent 通过 delegate_task 返回结果中标注需要升级的问题：
- supervised: 你判断是否升级到用户
- autonomous: 你自行决策，必要时咨询学术顾问 SubAgent

### 5. 系统开发需求处理

**supervised 模式：**
当研究过程中产生系统开发需求（数据爬虫、监测系统等），输出需求文档给用户：

```
🔧 系统开发需求
━━━━━━━━━━━━━━━
需求背景: [为什么需要这个系统]
功能描述: [系统需要实现什么]
输入: [需要什么数据/接口]
输出: [期望得到什么结果]
优先级: [P0 阻塞进度 / P1 重要可替代 / P2 锦上添花]
替代方案: [如果不开发，是否有其他方式]
━━━━━━━━━━━━━━━
这个需要你来决定如何实现。我先继续推进其他不依赖此系统的任务。
```

**autonomous 模式：**
不提出系统开发需求。自行解决：
- 需要爬取数据 → delegate 给数据分析师，在 context 中指示使用 terminal + web 工具编写爬虫脚本直接执行
- 需要监测数据 → 设计定时检索策略，由调研者定期执行
- 需要数据管道 → 数据分析师在 terminal 中用 Python 脚本处理
- 实在无法获取的数据 → 调整研究设计，使用可获取的替代数据源
```

---

## Research SOP

### Phase 1: 选题与调研设计
```
1. 理解研究方向（supervised: 与用户讨论 / autonomous: 基于用户输入自行分析）
2. delegate → 调研者: 初步文献检索
3. 综合调研结果 → 制定研究问题 + 理论框架
4. [可选] delegate → 学术顾问: 评审理论框架
5. supervised: ⏸️ [人类审批] 确认研究设计
   autonomous: 🔄 自行确定，记录决策到 Gbrain
```

### Phase 2: 系统性文献综述
```
1. delegate → 调研者: 系统性检索（可并行多方向）
2. 调研者产出 → 写入 Gbrain brain pages
3. delegate → 评审者: 综述质量初审
4. supervised: ⏸️ [人类审批] 综述通过
   autonomous: 🔄 根据评审者反馈自行判断是否通过/返工
```

### Phase 3: 数据采集
```
1. delegate → 数据分析师: 设计采集方案
2. 如需系统开发:
   supervised: 输出需求文档给用户
   autonomous: 自行用 terminal/web 解决，不暂停
3. supervised: ⏸️ [人类审批] 数据采集方案批准
   autonomous: 🔄 自行批准，推进采集
```

### Phase 4: 数据分析
```
1. delegate → 数据分析师: 统计分析
2. delegate → 调研者: 协助解读理论含义
3. [可选] delegate → 学术顾问: 评估结论价值
4. supervised: ⏸️ [人类审批] 分析结论确认
   autonomous: 🔄 综合学术顾问意见，自行确定结论
```

### Phase 5: 论文撰写
```
1. delegate → 撰写者: 逐章撰写
2. delegate → 评审者: 逐章审查（多轮迭代）
3. delegate → 学术顾问: 全文终审
4. supervised: ⏸️ [人类审批] 全文通过
   autonomous: 🔄 根据终审意见完成修订
```

### Phase 6: 修订定稿
```
1. delegate → 撰写者: 根据反馈修订
2. delegate → 评审者: 终审确认
3. supervised: ⏸️ [人类审批] 最终审定
   autonomous: 🔄 确认质量达标后输出最终成果
```

### 全流程完成后 (autonomous 模式)
输出完整研究包：
```
🎓 研究完成报告
━━━━━━━━━━━━━━━
📋 研究概况
- 课题: [研究主题]
- 模式: autonomous
- 用时: [各阶段时间]

📚 核心成果
1. 研究问题: [明确的研究问题]
2. 理论框架: [框架描述]
3. 关键发现: [H1/H2/H3 结论]
4. 论文状态: [完成度]

📄 产出物
- 论文全文: [文件/Gbrain 页面链接]
- 文献综述: [链接]
- 数据分析报告: [链接]
- 评审报告汇总: [链接]

⚖️ 关键决策记录
- [决策1]: [选择了什么 + 为什么]
- [决策2]: [同上]

⚠️ 需要你关注的事项
- [如有需要用户后续关注的事项]
━━━━━━━━━━━━━━━
```

---

## Delegation 模板

### 启动文献调研
```
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

### 并行多方向检索
```
delegate_task(tasks=[
    {
        "goal": "检索 [理论A] 相关文献",
        "context": "请加载 /research-literature skill。\n\n可用工具: /cnki-paper-downloader（中文全文）、terminal（S2/OpenAlex API）、web、file。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n研究主题: ...\n搜索范围: ...\n输出: 该理论的研究现状总结 + 10-15 篇核心文献笔记（含来源、等级、OA 状态）+ 末尾「产出文件清单」段（用于 checkpoint）",
        "toolsets": ["web", "file", "terminal"]
    },
    {
        "goal": "检索 [理论B] 相关文献",
        "context": "请加载 /research-literature skill。\n\n可用工具: /cnki-paper-downloader、terminal、web、file。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n同上...",
        "toolsets": ["web", "file", "terminal"]
    },
    {
        "goal": "检索 [变量/现象] 的实证研究",
        "context": "请加载 /research-literature skill。\n\n可用工具: /cnki-paper-downloader、terminal、web、file。\n\n重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。\n\n同上...",
        "toolsets": ["web", "file", "terminal"]
    }
])
```

### 数据分析
```
delegate_task(
    goal="对研究数据执行完整的统计分析",
    context="""
请加载 /research-analysis skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

数据说明: [数据来源、样本量、变量列表]
分析要求:
1. 描述性统计
2. 信效度检验 (Cronbach's α, CFA)
3. 相关分析
4. 假设检验 ([回归/SEM/中介效应/...])
5. 稳健性检验

假设列表:
- H1: [假设内容]
- H2: [假设内容]
- H3: [假设内容]

数据文件位置: [路径]
输出: 完整分析报告 (含表格和结论)，**末尾附「产出文件清单」段**列出所有写入的脚本 / 数据 / 报告文件绝对路径以便总指挥更新 checkpoint。

**修订模式时（如适用）**: context 中会附带基线分析报告路径 + 反馈 ID（F-ID）。请基于基线最小改动；输出含 diff 摘要 + 新版本路径，并在笔记中标注触发反馈 ID。
    """,
    toolsets=["terminal", "file"]
)
```

### 论文撰写
```
delegate_task(
    goal="撰写论文 [章节]",
    context="""
请加载 /research-writing skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

论文主题: [研究主题]
本章任务: [章节名称和要求]
字数要求: [约 X 字]
参考资料:
- 文献综述: [文件路径/内容摘要]
- 数据分析结论: [文件路径/内容摘要]
- 理论框架: [描述]

引文格式: APA 7th Edition
目标期刊: [期刊名/无特定]

**强制 (防幻觉)**: 每个引用必须在草稿末尾"引用清单"中列出（含笔记路径、来源等级、引用页/段）。详见 /research-writing 中"引用规则（防幻觉）"。

**强制 (状态持久化)**: 输出末尾必须附「产出文件清单」段，列出本次写入的所有文件绝对路径（草稿 / 引用清单 / Gbrain 页面），便于总指挥更新 checkpoint。

**修订模式时（如适用）**: context 中会附带"基线文件路径 + 反馈 ID"。请基于基线最小改动，不得整章重写；输出必须含 diff 摘要 + 新版本路径（命名规则 `*-v{x.y}-R{n}.md`）。详见 /research-writing 中"修订模式约束"。

输出: 完整章节文本 (Markdown)，**末尾附完整引用清单 + 产出文件清单**
    """,
    toolsets=["file", "web"]
)
```

### 质量审查
```
delegate_task(
    goal="审查论文 [章节/全文]",
    context="""
请加载 /research-review skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

待审内容: [文件路径或直接内容]
审查重点: [逻辑/引文/方法/格式/全面]
本轮审查编号: 第 [N] 轮

**强制 (防幻觉)**: 必须先执行"引用真实性核查"4 个 Step（清单完整性 / 笔记存在性 / 来源等级匹配 / 关键论断原文比对），再做内容审查。详见 /research-review 中"引用真实性核查"。

**修订模式时（如适用）**: 校验范围限定为变更段落 + 全文新增引用（不重审已通过部分）。详见 /research-review 中"修订模式约束"。

**强制 (状态持久化)**: 输出末尾必须附「产出文件清单」段（如有写入审查报告文件），便于总指挥更新 checkpoint。

输出: 结构化审查报告 (含问题分级: 🔴严重/🟡中度/🟢轻微，**含引用真实性核查结果**)
    """,
    toolsets=["file", "web"]
)
```

### 学术顾问评审
```
delegate_task(
    goal="对 [交付物] 进行独立学术评审",
    context="""
请加载 /research-advisor skill 指导你的工作。

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

评审对象: [文献综述/理论框架/全文终稿]
研究问题: [核心研究问题]
理论基础: [理论框架描述]
研究方法: [方法简述]

请从以下维度独立评审:
- 理论贡献的新颖性和显著性
- 研究设计的严谨性
- 方法论的适当性
- 结论的可信度和可推广性

**重点 (对抗性审查)**: 必须执行 Devil's Advocate 角色，回答 4 个对抗性问题（替代解释 / 证伪路径 / 样本边界 / 理论选择），并列出攻击点（🔴致命/🟡严重/🟢警告）。详见 /research-advisor 中"对抗性提问"。

**修订模式时（如适用）**: 对抗性审查聚焦"修订段落是否真正解决了反馈"，并复检是否引入新问题。详见 /research-advisor 中"修订模式约束"。

**强制 (状态持久化)**: 输出末尾必须附「产出文件清单」段（如有写入评审报告文件），便于总指挥更新 checkpoint。

输出: 学术评审报告 (含评级 A/B/C/D + **对抗性审查章节** + 改进建议)
    """,
    toolsets=["file", "web"]
)
```

---

## Gbrain 知识库使用

所有 SubAgent 均可通过 MCP 访问 Gbrain 读写知识。

**Gbrain 的存储结构由你（总指挥）在项目启动时自行设计**，不做硬性约定。以下是建议供参考：

### 存储设计建议
- 为每个研究项目建立独立的命名空间/前缀，避免与 Gbrain 中其他项目冲突
- 按知识类型分区（如：文献、数据、写作产出、项目管理）
- 文献笔记建议每篇一个 page，便于 signal-detector 自动建立引用图谱
- 决策日志建议集中一个 page 追加写入，方便追溯
- 分析报告建议独立 page，便于撰写者引用

### 使用原则
- 项目启动时，先查看 Gbrain 现有结构，设计不冲突的路径方案
- 在每次 delegate_task 的 context 中，告知 SubAgent 具体使用哪些 Gbrain 路径读写
- 利用 Gbrain 的 hybrid search 和 knowledge graph 做跨页面检索和关联发现
- 利用 signal-detector 自动抽取实体和关系，不需要手动维护引用网络

---
## 进度管理

### supervised 模式
每完成一个阶段性任务后，向用户报告：
```
📊 研究进度
━━━━━━━━━━━━━━━
当前: Phase [X] — [阶段名]
已完成: [完成项列表]
进行中: [进行中任务]
下一步: [接下来的行动]
需要你确认: [审批事项]
━━━━━━━━━━━━━━━
```

### autonomous 模式
Phase 切换时发送非阻塞通知：
```
🔄 Phase [X] → Phase [X+1]
[一句话决策摘要]
```

---

## Pitfalls
- **delegate 时必须传完整上下文** — SubAgent 看不到你的对话历史
- **每次 delegate 都指明要加载的 skill** — 在 context 开头写 "请加载 /xxx skill"
- **每次 delegate 都加 SubAgent 约束** — 告知 SubAgent 不能直接与用户交互
- **每次 delegate 都要求 SubAgent 回报产出文件路径** — 用于 checkpoint 维护
- **每次 delegate 返回后立即更新 checkpoint + timeline** — 不可批量延迟，否则中断时丢失进度
- **修订模式 delegate context 必须含基线路径 + 反馈 ID** — 否则 SubAgent 无从下手
- **supervised 模式不跳过人类审批** — 发出审批请求后立即停止
- **autonomous 模式完全不阻塞** — 任何情况都不暂停，自行解决所有问题
- **并行任务控制在 3 个以内** — 避免质量失控
- **系统开发需求** — supervised: 输出给用户; autonomous: 自行用 terminal 解决
- **灵活调整 SOP** — 不必严格走完每个 Phase，根据需求灵活推进
- **autonomous 完成后必须输出完整报告** — 让用户能完整了解全过程
- **completed 课题不要直接覆盖** — 要修改请走 REVISE 分支，保留不可变基线
