# 多 Agent DeepResearch 论文系统 — 项目方案文档

> **版本**: v3.5
> **日期**: 2026-05-17
> **论文类型**: 社科/管理类
> **技术选型**: Hermes Agent + Gbrain
> **架构模式**: Skills 协同编排 + 双模式运行 (supervised / autonomous) + 三启动分支 (new / resume / revise)

---

## 一、项目概述

### 1.1 目标

构建一个基于多 Agent 协同的学术研究系统，覆盖从**初步调研**到**论文终稿**的全生命周期，并支持**断点续跑**与**版本化修订**。通过 Hermes Skills 定义不同研究角色，由总指挥 Skill 通过 `delegate_task` 动态生成 SubAgent 团队协同工作。

### 1.2 核心理念

- **Skills 即能力**：每个研究角色对应一个 Hermes Skill，Skill 定义角色行为规范和工作流程
- **动态编排**：总指挥按 SOP 阶段动态生成 SubAgent，无需常驻 Agent 占用资源
- **双模式运行**：`supervised`（Human-in-the-Loop）和 `autonomous`（全自动）满足不同场景
- **三启动分支**：`new` / `resume` / `revise` —— 同一课题可中断续跑、可基于反馈版本化修订
- **不可变基线**：修订生成新版本文件（`*-v{x.y}-R{n}.md`），原产出物不可覆盖
- **状态层与知识层分离**：运行时状态用本地 `~/.hermes/research-state/`，学术知识用 Gbrain
- **防幻觉与对抗性审查**：引用必须可追溯到 Gbrain 笔记 + 来源等级匹配；学术顾问扮演 Devil's Advocate
- **零侵入安装**：无需修改 SOUL.md 或 config.yaml，只装 Skills

### 1.3 技术选型

| 维度 | 选择 | 理由 |
|---|---|---|
| Agent 框架 | **Hermes Agent**（共享实例） | 原生 `delegate_task`、Skills 系统、MCP 集成 |
| 知识大脑 | **Gbrain**（共享实例） | 仅承载学术知识；运行时状态不入图 |
| 中文论文数据源 | **CNKI** | `/cnki-paper-downloader` 提供全文下载 |
| 英文论文数据源 | **Semantic Scholar + OpenAlex**（Terminal 调免费 API） | 论文发现 + 摘要 + 引文图谱（OA 时附 PDF 链接） |
| 人工兜底通道 | `~/Downloads/essays/` 文件夹 | 当 CNKI / OA 全部失败时由用户手动补充 PDF |
| 状态持久化 | 本地 `~/.hermes/research-state/{slug}/` | 轻量、可读、与 Gbrain 解耦 |
| 执行方式 | Hermes Skills + delegate_task | 总指挥编排 SubAgent，各 SubAgent 加载对应 Skill |
| 协同方式 | SubAgent 返回值 + 产出文件清单 | SubAgent 不能直接交互用户；产出路径回报给总指挥 |

### 1.4 系统架构

```
用户: "/deep-research [new|resume|revise|list] [课题]"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Hermes Agent (共享实例)                                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ /deep-research Skill (总指挥 PI)                              ││
│  │  • 启动探测：~/.hermes/research-state/index.md → 三分支    ││
│  │  • 理解研究需求 → 设计 Gbrain 存储结构                      ││
│  │  • SOP 分阶段编排 → delegate_task 生成 SubAgent             ││
│  │  • 每次 delegate 返回 → 立即更新 checkpoint + timeline      ││
│  │  • supervised: 关键节点暂停请求审批                           ││
│  │  • autonomous: 全自动推进，完成后汇报                        ││
│  │  • revise: 反馈结构化 → 订正计划 → 局部重写 → 质量门禁       ││
│  └──────────┬──────────┬──────────┬──────────┬─────────────────┘│
│        delegate   delegate   delegate   delegate                 │
│             │          │          │          │                   │
│             ▼          ▼          ▼          ▼                   │
│       ┌──────────┐┌──────────┐┌──────────┐┌──────────┐         │
│       │/research-││/research-││/research-││/research-│         │
│       │literature││analysis  ││writing   ││review    │         │
│       │+ /cnki   ││+ terminal││          ││（含引用真│         │
│       │ + S2/OA  ││          ││          ││ 实性核查）│         │
│       │ + Tier3  ││          ││          ││          │         │
│       └──────────┘└──────────┘└──────────┘└──────────┘         │
│                                                   ↕             │
│       ┌──────────┐                         /research-advisor    │
│       │里程碑独立 │◄── on-demand delegate ─┘ (Devil's Advocate)│
│       │评审      │                                              │
│       └──────────┘                                              │
└──────────┬───────────────────────────────────┬──────────────────┘
           │ 状态层（运行时元数据）             │ 知识层（学术内容）
           ▼                                   ▼ MCP Protocol
  ┌─────────────────────────────┐  ┌────────────────────────────┐
  │ ~/.hermes/research-state/   │  │  Gbrain (共享实例)          │
  │  index.md                   │  │  • Brain Pages              │
  │  {slug}/                    │  │  • Knowledge Graph          │
  │   ├── checkpoint.md         │  │  • Hybrid Search            │
  │   ├── timeline.md           │  │  • Signal Detector          │
  │   └── revisions/Rn-{date}/  │  │  • MCP Server (30+ tools)   │
  │ ~/Downloads/essays/         │  │                             │
  │   (Tier 3 人工下载 PDF)    │  │                             │
  └─────────────────────────────┘  └────────────────────────────┘
```

---

## 二、团队角色

### 2.1 角色矩阵

| 角色 | 对应 Skill | 职责 | 生成方式 |
|---|---|---|---|
| **总指挥 (PI)** | `/deep-research` | 启动探测、任务编排、checkpoint 维护、审批门控、修订计划 | 用户直接调用 |
| **调研者** | `/research-literature` | 文献检索（CNKI + S2/OpenAlex 双通道）、综述、研究缺口、Tier 3 清单生成 | delegate_task |
| **数据分析师** | `/research-analysis` | 数据清洗、统计分析、假设检验、可视化 | delegate_task |
| **撰写者** | `/research-writing` | 论文撰写、引文管理（含来源等级匹配） | delegate_task |
| **评审者** | `/research-review` | 逻辑检查、**引用真实性 4 步核查**、方法评估、格式审查 | delegate_task |
| **学术顾问** | `/research-advisor` | 里程碑独立评审 + **Devil's Advocate 对抗性审查**、评级 | delegate_task (on-demand) |

### 2.2 已移除的角色

- **系统开发者**：不再作为独立 Agent。研究中的技术需求处理方式：
  - `supervised` 模式：总指挥输出结构化需求文档，由人类决定实现方式
  - `autonomous` 模式：总指挥（或被 delegate 的 SubAgent）直接使用 terminal/web 工具自行解决

### 2.3 SubAgent 通信机制

```
通信规则:
1. SubAgent 只能看到 delegate_task 传入的 context
2. SubAgent 不能与用户直接交互
3. SubAgent 遇到问题 → 在输出中标注 [需升级]
4. SubAgent 输出末尾必须附「产出文件清单」段（绝对路径），便于总指挥更新 checkpoint
5. 总指挥读取 SubAgent 返回值 → 决定是否升级到用户
6. SubAgent 之间不能直接通信 → 必须通过总指挥中转或 Gbrain 共享
7. 修订模式下，delegate context 必须显式声明「基线文件路径 + 反馈 ID（F-ID）」
```

通用约束的单一信息源 → 见 `deep-research/STATE_PROTOCOL.md` 末尾「SubAgent 通用约束」段。

---

## 三、双模式 + 三启动分支

### 3.1 模式对比

| 维度 | supervised (默认) | autonomous |
|---|---|---|
| 启动命令 | `/deep-research [课题]` 或 `/deep-research supervised [课题]` | `/deep-research autonomous [课题]` |
| 审批节点 | 每阶段结束暂停，等待人类确认 | 无暂停，全自动推进 |
| 系统开发需求 | 输出需求文档，暂停等待人类处理 | 自行使用 terminal/web 解决 |
| Tier 3 人工下载 | 输出清单后**暂停一次**等用户上传 | 不阻塞，按摘要等级先推进，最终报告显著标注待补 |
| 结果汇报 | 每阶段汇报 + 请求审批 | 全部完成后统一汇报 |
| 适用场景 | 重要研究、需要人类把控方向 | 探索性调研、时间紧迫 |

### 3.2 启动分支（new / resume / revise）

| 命令 | 行为 |
|---|---|
| `/deep-research [课题]` | 自动探测：命中 in_progress → resume；命中 completed → 询问 (supervised) / 默认 revise (autonomous)；未命中 → new |
| `/deep-research resume [课题]` | 强制续跑（in_progress 课题）；找不到则报错 |
| `/deep-research revise [课题]` | 强制进入修订（completed 课题）；可加 `--feedback-from <路径>` 直接传入反馈源 |
| `/deep-research restart [课题]` | 强制新建（自动备份原 checkpoint） |
| `/deep-research list` | 列出 index.md 中所有研究 |

详细启动探测流程、状态文件结构 → `deep-research/STATE_PROTOCOL.md`。

### 3.3 supervised 模式审批节点

| 阶段 | 审批内容 | 用户操作 |
|---|---|---|
| Phase 1 完成 | 研究问题 + 理论框架 | "可以" / 提出修改 |
| Phase 2 完成 | 文献综述初稿 | "可以" / "需要补充 X 方向" |
| Phase 2 中（如触发 Tier 3） | 人工下载清单 | 上传 PDF 至 `~/Downloads/essays/` 或 "放弃 #N" |
| Phase 3 完成 | 研究设计方案 | "可以" / 修改方法 |
| Phase 4 完成 | 数据分析结论 | "可以" / 调整分析 |
| Phase 5 完成 | 论文全文 + 终审意见 | "可以" / 修改 |
| Phase 6 完成 | 最终定稿 | "通过" |
| Revise R2/R3（如有） | 反馈拆解结果 | "可以" / 调整 F-ID 优先级 |

---

## 四、SOP 流程

### 4.1 研究生命周期（new / resume 共用）

```
Phase 1: 选题与调研设计
  └─ delegate → 调研者: 初步文献检索 (CNKI 中文 + S2/OpenAlex 英文)
  └─ 总指挥 + (可选) 学术顾问: 确定研究问题和理论框架
  └─ supervised: ⏸️ 人类审批

Phase 2: 系统性文献综述
  └─ delegate → 调研者(并行 ≤3): 多方向系统检索
  └─ 如失败：触发 Tier 3 人工下载清单（写入 ~/Downloads/essays/）
  └─ delegate → 评审者: 综述初审（含引用真实性 4 步核查）
  └─ supervised: ⏸️ 人类审批

Phase 3: 研究设计
  └─ 总指挥: 制定方案（问卷/实验/案例/...）
  └─ delegate → 数据分析师: 设计采集方案
  └─ supervised: ⏸️ 人类审批

Phase 4: 数据分析
  └─ delegate → 数据分析师: 执行分析
  └─ delegate → 调研者: 解读结果的理论含义
  └─ delegate → 学术顾问: 对抗性评估学术价值
  └─ supervised: ⏸️ 人类审批

Phase 5: 论文撰写与审查
  └─ delegate → 撰写者: 逐章撰写（强制三铁律 + 引用清单）
  └─ delegate → 评审者: 逐章审查（引用真实性核查 + 多轮迭代）
  └─ delegate → 学术顾问: 全文 Devil's Advocate 终审
  └─ supervised: ⏸️ 人类审批

Phase 6: 修订定稿
  └─ delegate → 撰写者: 根据反馈修订
  └─ delegate → 评审者: 终审确认
  └─ supervised: ⏸️ 人类最终审定

每次 delegate 返回 → 总指挥更新 checkpoint.md + 追加 timeline.md
Phase 切换 → 写「关键决策记录」+ timeline `[PHASE] X done → Y begin`
研究完成 → checkpoint.status=completed + index.md 同步
```

### 4.2 修订生命周期（revise 分支）

```
R1: 反馈收集
  └─ 多源支持（导师 / 审稿人 / 自查）→ 写入 revisions/R{n}/feedback.md（不修改原文）

R2: 反馈结构化
  └─ 拆成 F-ID 表格：内容 / 类型（理论/方法/写作/格式）/ 严重度（🔴/🟡/🟢）/ 涉及章节 / 涉及 Phase

R3: 订正计划制定
  └─ 每条反馈 → 1+ 个 delegate 任务 + 依赖标注
  └─ 显式声明"不变更项 vs 仅订正项"
  └─ 写入 revisions/R{n}/revision-plan.md

R4: 执行订正（按依赖顺序）
  └─ 每个 delegate context 必含: 基线文件路径 + F-ID + 最小改动要求 + diff 输出要求

R5: 质量门禁（复用 M1 链路）
  └─ delegate → 评审者: 引用真实性核查（重点：新增/变更引用）
  └─ delegate → 学术顾问: 对抗性审查（聚焦：修订段落是否真正解决反馈）

R6: 收尾
  └─ revisions/R{n}/changes.md（章节级 diff、引用增减、字数变化）
  └─ 论文文件命名：main-draft-v1.md → main-draft-v1.1-R1.md（基线不动）
  └─ checkpoint「修订历史」追加 + timeline [REVISION_DONE]
```

详见 `deep-research/REVISION_WORKFLOW.md`。

---

## 五、工具链

### 5.1 论文获取（三层兜底）

```
Tier 1 (自动): CNKI 全文 / OA PDF 直链
        ↓ 失败
Tier 2 (自动): 摘要笔记（S2 / OpenAlex / CNKI 仅摘要）
        ↓ 仍无法支撑核心论据
Tier 3 (人工): 列入「人工下载清单」→ 用户手动放 ~/Downloads/essays/ → 调研者读取
```

| 工具 | 用途 | 调用方式 |
|---|---|---|
| `/cnki-paper-downloader` | CNKI 中文全文下载 | Hermes Skill，输入完整标题，每次一篇 |
| Terminal + S2 API | 英文论文发现 + 摘要 + 引文图谱 | curl `https://api.semanticscholar.org/graph/v1/...`（限频 100/5min） |
| Terminal + OpenAlex API | 英文论文发现（含 concepts、机构）+ 引文 | curl `https://api.openalex.org/works`（10 req/s，建议 User-Agent 加 mailto:） |
| Terminal + wget/curl | OA PDF 直链下载 | 仅当 `openAccessPdf.url` / `oa_url` 非空 |
| `~/Downloads/essays/` | Tier 3 人工下载 PDF 落盘 | 调研者扫描该目录，按"建议文件名"匹配清单 |
| Web 搜索 | 论文标题发现、元数据确认 | Hermes 内置 web 工具 |
| Google Scholar | 英文文献发现、引用网络辅助 | 通过 web 工具访问 |

curl / wget 详细示例见 `research-literature/SKILL.md`；Tier 3 操作流程见 `research-literature/MANUAL_DOWNLOAD.md`。

### 5.2 三类笔记体系（来源等级）

| 来源等级 | 来源 | 用途 |
|---|---|---|
| **全文 (S1)** | CNKI 下载 / OA PDF / 用户手动下载 | 可作核心论据，必须含 ≥2-3 段原文 quote（带页码定位） |
| **摘要 (S2)** | S2 / OpenAlex / CNKI 仅摘要 | 背景引用，不得支撑具体数据/方法细节 |
| **元数据 (S3)** | S2 / OpenAlex（仅引文图谱节点） | 仅作研究地形测绘，**不得正文引用** |

撰写者引用时必须做「来源等级 ↔ 论断粒度」匹配；评审者通过引用真实性 4 步核查兜底。

### 5.3 Gbrain 能力

| 能力 | 研究用途 | 调用 Agent |
|---|---|---|
| `brain-ops` | 知识页面读写 | 全体 |
| `hybrid-search` | 语义 + 全文检索 | 全体 |
| `signal-detector` | 自动实体抽取和关系识别（笔记入库即触发） | 写入时自动 |
| `maintain` | 知识库健康检查 | 总指挥 |
| MCP tools (30+) | 按需调用 | 按场景 |

> Gbrain 仅承载学术知识，**不存运行时状态**（避免 signal-detector 抽取出 Phase / 待办等噪声实体）。

### 5.4 其他工具

| 工具 | 用途 | 使用者 |
|---|---|---|
| Terminal (Hermes) | 执行 Python 统计脚本 / curl API / wget OA PDF | 数据分析师、调研者 |
| File 工具 | 读写本地文件（含 ~/.hermes/research-state/、~/Downloads/essays/） | 全体 |
| Web 工具 | 网络信息获取 | 调研者、总指挥 |

---

## 六、Gbrain 知识库策略

### 6.1 设计原则

- **不硬编码路径**：Gbrain 是共享实例，可能同时服务多个项目
- **总指挥运行时设计**：启动研究时由总指挥根据课题设计命名空间和页面结构
- **自动知识图谱**：`signal-detector` 在每次写入时自动抽取实体和关系
- **状态层与知识层分离**：运行时状态（progress、checkpoint）属于元数据，**严禁写入 Gbrain**

### 6.2 建议分区（总指挥参考）

| 分区 | 内容 | 写入者 |
|---|---|---|
| 项目管理 | 元信息、决策日志（**不含 progress**） | 总指挥 |
| 文献知识 | 笔记（含来源等级、外部 ID、OA 链接、Tier 3 本地路径）、综述、理论框架 | 调研者 |
| 数据分析 | 方案、报告、结论 | 数据分析师 |
| 写作产出 | 草稿、引用清单、审查反馈 | 撰写者 |

### 6.3 跨项目复用

```
项目 A (信任研究)        项目 B (用户行为)
       │                        │
       └── 共享 Theory 实体 ────┘
       │                        │
       └── 共享 Method 实体 ────┘
       │                        │
       └── 共享 Paper 实体 ─────┘
```

---

## 七、CNKI + 英文双通道集成方案

### 7.1 工作流程

```
Step 1: 搜索策略设计
  关键词拆解 → CNKI 检索式 + S2/OpenAlex 关键词 → 时间范围 → 来源类别

Step 2: 论文标题发现（双通道）
  中文: web 搜索 "site:cnki.net [关键词]"
  英文: terminal curl S2 + OpenAlex（含引文图谱滚雪球）
  Google Scholar 辅助

Step 3: 论文获取（按来源等级）
  CNKI 中文全文     → /cnki-paper-downloader [完整标题]
  OA PDF 全文       → wget/curl 下载到本地
  仅摘要            → S2 / OpenAlex abstract 字段直接摘录
  元数据节点        → 仅记 title/authors/year/concepts
  以上均失败且核心 → Tier 3 清单 → ~/Downloads/essays/

Step 4: 精读与笔记
  结构化笔记（含来源等级、外部 ID、OA 链接、原文 quote）→ 写入 Gbrain

Step 5: 综述整合
  文献获取状态表（来源 + 等级 + 状态 ✅/⚠️/❌）
```

### 7.2 获取状态追踪

| 状态 | 含义 | 后续处理 |
|---|---|---|
| ✅ 已下载全文 | CNKI / OA / 用户手动下载 | 必须精读 + 提取 ≥2-3 段原文 quote |
| ⚠️ 仅摘要 | 无法获取全文 | 可作背景引用，不作核心论据 |
| ❌ 未获取 | 下载失败或未收录 | 标注原因，考虑替代文献或进入 Tier 3 |
| 📥 待人工下载 | Tier 3 清单中 | 等用户上传到 `~/Downloads/essays/` 后补笔记 |

---

## 八、防幻觉与对抗性审查

### 8.1 防幻觉三铁律（撰写者）

1. **只引读过的**：每条引用必须能在 Gbrain 找到对应笔记
2. **来源等级匹配论断**：S1 全文笔记可支撑具体数据/方法细节；S2 摘要仅作背景；S3 元数据**禁止正文引用**
3. **不做二手转引**：不引用"X 引用 Y 时说……"式的间接引用，必须读到原始 Y

每章草稿末尾必须附「引用清单」（含笔记路径、来源等级、引用页/段）。

### 8.2 评审者引用真实性 4 步核查

1. **清单完整性**：草稿引用与引用清单 1:1 对齐
2. **笔记存在性**：用 `gbrain hybrid-search` / `brain-ops read` 验证笔记存在
3. **来源等级匹配**：核对论断粒度与来源等级
4. **关键论断原文比对**：把撰写者转述与全文笔记中的 quote 比对

### 8.3 学术顾问 Devil's Advocate（4 题）

1. **替代解释**：是否存在另一种理论可同样解释结果？
2. **证伪路径**：什么样的数据会推翻当前结论？
3. **样本/边界**：结论的可推广性边界在哪里？
4. **理论选择**：为何选择 X 理论而不是 Y 理论？是否有理论选择偏倚？

输出含攻击点分级（🔴致命 / 🟡严重 / 🟢警告）。

### 8.4 修订模式专属审查

- 评审者：聚焦"新增/变更引用"的真实性核查；不重审已通过部分
- 学术顾问：聚焦"修订段落是否真正解决反馈"，复检是否引入新问题

---

## 九、升级机制

### 9.1 升级路径

```
SubAgent 遇到问题
    │
    ▼ (在输出中标注 [需升级])
总指挥评估
    │
    ├─ 可自行处理 → 直接处理
    ├─ 需学术判断 → delegate → 学术顾问
    └─ 需人类决策 → ⏸️ 暂停 (supervised) / 自行决策 (autonomous)
```

### 9.2 升级条件

| 条件 | 处理方 |
|---|---|
| 超出能力范围 | 总指挥 → 学术顾问 |
| 理论争议 | 学术顾问评估 |
| 战略方向变更 | 升级到人类 (supervised) |
| 大量核心文献无法获取（CNKI + OA + Tier 3 都不行） | 升级到人类 (supervised) |
| ≥3 处疑似幻觉 / 同一论文多处自相矛盾 | 评审者强制升级 |
| 修订反馈与基线立场不可调和 | 升级到人类 |
| autonomous 模式下任何问题 | 总指挥自行决策或记录 |

---

## 十、风险评估

| 风险 | 等级 | 缓解措施 |
|---|---|---|
| LLM 幻觉影响学术准确性 | 高 | 三铁律 + 引用清单 + 评审者 4 步核查 + 学术顾问对抗审查 |
| CNKI 论文获取失败率高 | 中 | 三层兜底（CNKI / OA / Tier 3 人工）+ 获取状态追踪 |
| 长流程中断丢失进度 | 中 | 状态持久化 + checkpoint + timeline + RESUME 分支 |
| 修订覆盖原稿历史 | 中 | 不可变基线（生成 v{x.y}-R{n} 新版本，原文件永不动） |
| SubAgent 产出质量不稳定 | 中 | 完整 context 传递 + 严格输出格式（含产出文件清单） |
| Gbrain 知识图谱噪声 | 低 | 状态不入图 + 定期 maintain + signal-detector 自动修正 |
| autonomous 模式方向偏离 | 中 | 学术顾问里程碑审查 + Devil's Advocate 兜底 |
| S2 / OpenAlex API 不可达 | 低 | 限频退避（5s × 3 次重试）+ 降级 web 搜索 |

---

## 十一、项目里程碑

| 阶段 | 状态 | 交付物 / 关键 commit |
|---|---|---|
| **M0**: Skills 设计与初版开发 | ✅ 完成 | 6 个 Hermes Skills + README（`74ff03f`） |
| **M1**: 防幻觉与对抗性审查 | ✅ 完成 | 引用三铁律 + 4 步核查 + Devil's Advocate（`6f3633c`） |
| **M2**: 英文文献元数据/图谱通道 | ✅ 完成 | S2 + OpenAlex Terminal 集成 + 三类笔记（`38397ea`） |
| **M3**: 状态持久化 + 断点续跑 + 修订模式 | ✅ 完成 | `~/.hermes/research-state/` + resume / revise 分支 + 不可变基线（`ca5c0a8`） |
| **M3.5**: Tier 3 人工下载兜底 | ✅ 完成 | `~/Downloads/essays/` + manual-download-needed 清单 |
| **Refactor**: 长文件拆分 + 冗余精简 | ✅ 完成 | 4 个附录 .md + 主 SKILL 精简 +SubAgent 约束单一信息源（`f1caf72`） |
| M4: 端到端验证（mini-topic 试跑） | ⬜ 待做 | 选定小课题 → 全流程跑通 → 验证英文引文图谱与 Tier 3 兜底 |
| M5: 长流程稳定性回归（中断 → 续跑 → 修订） | ⬜ 待做 | 验证 token 中断、checkpoint 恢复、R1/R2 多轮修订 |
| M6: 迭代优化 | ⬜ 持续 | 根据试跑反馈调整 Skills |

---

## 十二、文件结构

```
.
├── deep-research/
│   ├── SKILL.md                     ← 总指挥主入口（精简版导航）
│   ├── STATE_PROTOCOL.md            ← 状态持久化、启动探测、checkpoint 维护、SubAgent 通用约束
│   ├── REVISION_WORKFLOW.md         ← REVISE 分支六步走、不可变基线、多轮修订
│   └── DELEGATION_TEMPLATES.md      ← 5 个 delegate 模板（1 完整 + 4 差异）
├── research-literature/
│   ├── SKILL.md                     ← 文献检索与综述（含 S2/OpenAlex/wget curl 全集）
│   └── MANUAL_DOWNLOAD.md           ← Tier 3 人工下载兜底通道完整规范
├── research-analysis/SKILL.md
├── research-writing/SKILL.md        ← 含引用三铁律 + 修订模式约束
├── research-review/SKILL.md         ← 含引用真实性 4 步核查 + 修订模式审查
├── research-advisor/SKILL.md        ← 含 Devil's Advocate 4 题 + 修订模式约束
├── docs/
│   ├── README.md                    ← 本文件（项目方案）
│   ├── AGENTS.md
│   ├── appendix-a-knowledge-base.md
│   └── appendix-b-skill-toolchain.md
├── README.md                        ← 安装说明 + 文件结构
└── CHANGELOG.md                     ← M0 → M3 + Refactor 完整变更历史
```
