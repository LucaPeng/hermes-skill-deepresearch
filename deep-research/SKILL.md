---
name: deep-research
description: 多Agent学术研究 — 启动完整研究团队，完成从选题到论文终稿的全流程
version: 1.7.0
metadata:
  hermes:
    tags: [research, academic, paper, multi-agent, orchestration]
    related_skills: [research-literature, research-analysis, research-writing, research-review, research-advisor, cnki-paper-downloader]
---

# Deep Research — 多 Agent 学术研究编排

## When to Use
当用户需要：启动学术研究项目、完成社科/管理类论文、执行系统性文献综述、或进行多角色协同的研究工作。

## Overview
本 skill 作为**研究项目总指挥 (PI)**，通过 `delegate_task` 编排多个 SubAgent 协同完成学术研究。你不直接做具体的调研/分析/写作，而是分解任务、分派执行、汇总结果、做学术决策。

## 附录文件（必读）
- [STATE_PROTOCOL.md](./STATE_PROTOCOL.md) — 状态持久化、启动探测、checkpoint 维护规则、SubAgent 通用约束
- [REVISION_WORKFLOW.md](./REVISION_WORKFLOW.md) — REVISE 分支六步走、不可变基线、多轮修订
- [DELEGATION_TEMPLATES.md](./DELEGATION_TEMPLATES.md) — 5 个 delegate 模板（1 完整 + 4 差异）

---

## 启动协议（先于一切）

收到 `/deep-research [...]` 时，**先执行启动探测**：新建 / 续跑 / 修订三选一。
状态持久化在本地 `~/.hermes/research-state/`，**不依赖 Gbrain**。

| 命令 | 行为 |
|---|---|
| `/deep-research [课题]` | 自动探测：in_progress → resume；completed → 询问 (supervised) / 默认 revise (autonomous)；未命中 → new |
| `/deep-research supervised \| autonomous [课题]` | 显式指定模式 |
| `/deep-research resume \| revise \| restart [课题]` | 强制对应分支 |
| `/deep-research revise [课题] --feedback-from <路径>` | 直接传入反馈源 |
| `/deep-research list` | 列出所有研究 |

完整探测流程、状态文件结构与模板、Checkpoint 维护规则 → **见 [STATE_PROTOCOL.md](./STATE_PROTOCOL.md)**。
REVISE 分支详细工作流 → **见 [REVISION_WORKFLOW.md](./REVISION_WORKFLOW.md)**。

---

## 运行模式

| 模式 | 触发 | 行为 |
|---|---|---|
| **supervised**（默认） | `/deep-research [课题]` 或 `supervised` | 关键节点暂停等待用户审批 |
| **autonomous** | `/deep-research autonomous [课题]` 或含"自动/全自动/不用问我"等词 | 全自动推进，完成后一次性汇报 |

启动时向用户确认模式：
- supervised: `📋 以 supervised 模式启动，关键节点我会暂停请你确认。`
- autonomous: `🚀 以 autonomous 模式启动，我会自主推进全流程，完成后向你汇报完整结果。`

---

## 团队编制

通过 `delegate_task` 启动以下 SubAgent：

| 角色 | 加载 Skill | 任务类型 | 生命周期 |
|---|---|---|---|
| 调研者 | `/research-literature` | 文献检索、综述、理论分析 | 常驻 |
| 数据分析师 | `/research-analysis` | 数据采集、统计分析、可视化 | 常驻 |
| 撰写者 | `/research-writing` | 论文撰写、引文管理 | 常驻 |
| 评审者 | `/research-review` | 质量审查、引用真实性核查 | 常驻 |
| 学术顾问 | `/research-advisor` | 里程碑独立评审、对抗性审查 | 按需 |

> **系统开发需求**（爬虫、监测系统等）：supervised 输出需求文档给用户决定；autonomous 由 SubAgent 自行用 terminal/web 解决。

---

## 核心原则

### 1. Human-in-the-Loop（仅 supervised 模式）

**⚠️ 以下规则仅在 supervised 模式下生效，autonomous 模式跳过所有审批节点。**

必须暂停并等待用户确认的节点：研究问题确认 / 理论框架确认 / 研究设计定稿 / 数据采集方案 / 重大分析结论 / 论文终稿审定 / 战略方向变更。

**审批请求格式：**
```
📋 审批请求
━━━━━━━━━━━━━━━
阶段: [阶段名]
内容摘要: [3-5 句总结]
建议: [推荐方案]
━━━━━━━━━━━━━━━
请确认是否继续，或告诉我需要调整的方向。
⏸️ 等待你的回复后再推进下一步。
```

**关键约束：** 发出审批请求后**立即停止**，不得继续 delegate；用户未回复前不得自行假设"默认通过"。

### 2. Autonomous 模式行为规则

- 不暂停、不请求审批，按 SOP 自动推进全流程
- 由总指挥自行做所有决策；每个 Phase 完成后记录决策到 Gbrain（`project/decisions`）
- 全流程完成后输出**完整研究报告**（研究问题 + 框架 + 综述摘要 + 分析结论 + 论文 + 决策记录 + 评审意见汇总）
- 任何阻塞都自行解决：需要爬取 → 直接用 terminal 写脚本；遇到障碍 → 找替代数据源或调整设计

**Phase 切换时的非阻塞通知：**
```
🔄 Phase [X] → Phase [X+1]
[一句话决策摘要]
```

### 3. 意见冲突优先级
- supervised: 用户 > 学术顾问 > 评审者 > 你
- autonomous: 学术顾问 > 评审者 > 你（用户不参与实时决策）

### 4. 升级机制
SubAgent 在返回结果中标注 `[需升级]` 的问题：
- supervised: 你判断是否升级到用户
- autonomous: 自行决策，必要时咨询学术顾问 SubAgent

### 5. 系统开发需求处理

**supervised**：输出需求文档，包含背景 / 功能 / 输入输出 / 优先级（P0 阻塞 / P1 重要 / P2 锦上添花）/ 替代方案。

**autonomous**：自行解决，不提需求 — 爬取用 terminal 脚本；监测用定时检索；管道用 Python；实在不行调整研究设计。

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
━━━━━━━━━━━━━━━
```

---

## Delegation 模板

5 个核心模板（启动文献调研 / 并行多方向检索 / 数据分析 / 论文撰写 / 质量审查 + 学术顾问评审）→ **见 [DELEGATION_TEMPLATES.md](./DELEGATION_TEMPLATES.md)**。

> **每次 delegate 必含：**
> 1. 在 context 开头声明 "请加载 /xxx skill"
> 2. SubAgent 约束（不能与用户交互、`[需升级]` 标注、产出文件清单）
> 3. 修订模式时附带"基线路径 + 反馈 ID"

---

## Gbrain 知识库使用

所有 SubAgent 通过 MCP 访问 Gbrain 读写**知识**（不存运行时状态）。

**Gbrain 存储结构由总指挥在项目启动时自行设计**，建议：
- 每个研究项目独立命名空间/前缀，避免与其他项目冲突
- 按知识类型分区：文献 / 数据 / 写作产出 / 项目管理
- 文献笔记每篇一个 page（便于 signal-detector 自动建图谱）
- 决策日志集中一个 page 追加写入
- 分析报告独立 page

**使用原则：** 启动时先看现有结构 → 在 delegate context 中告知 SubAgent 具体路径 → 利用 hybrid search + knowledge graph 跨页面检索 → 利用 signal-detector 自动抽取实体关系。

---

## 进度管理

### supervised 模式（阶段性报告）
```
📊 研究进度
━━━━━━━━━━━━━━━
当前: Phase [X] — [阶段名]
已完成: [完成项]
进行中: [进行中任务]
下一步: [接下来的行动]
需要你确认: [审批事项]
━━━━━━━━━━━━━━━
```

### autonomous 模式
Phase 切换时发非阻塞通知（见上文）。

---

## Pitfalls

- **delegate 时必须传完整上下文** — SubAgent 看不到你的对话历史
- **每次 delegate 都指明加载哪个 skill** — context 开头写 "请加载 /xxx skill"
- **每次 delegate 都加 SubAgent 约束** — 不能直接与用户交互；需要回报产出文件路径
- **每次 delegate 返回后立即更新 checkpoint + timeline** — 不可批量延迟，否则中断时丢失进度
- **修订模式 delegate context 必须含基线路径 + 反馈 ID** — 否则 SubAgent 无从下手
- **supervised 不跳过人类审批** — 发出审批请求后立即停止
- **autonomous 完全不阻塞** — 任何情况都不暂停，自行解决所有问题
- **并行任务控制在 3 个以内** — 避免质量失控
- **completed 课题不要直接覆盖** — 要修改请走 REVISE 分支，保留不可变基线
- **autonomous 完成后必须输出完整报告** — 让用户能完整了解全过程
