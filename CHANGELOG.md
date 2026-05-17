# Changelog

本项目所有显著的 Skill 变更记录于此。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

---

## [M4.2: 状态写入强制化 — 修复 research-state 目录长期为空] - 2026-05-17

**问题诊断**：实际运行中发现 `~/.hermes/research-state/` 目录一直为空。根因是设计文档中"创建 checkpoint.md / timeline.md / index.md"的描述用了模糊的命令式表述（"创建 / 初始化 / 维护"），LLM 在执行时把这些当成了"参考流程"而非"必须的 file 工具调用"，导致状态文件从未真实落盘，进而 RESUME 永远找不到任何课题。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.8.1 | **1.8.2** |

### 修复 (Fixed)

- **deep-research/SKILL.md** 主文档新增 **「🚨 启动后强制状态初始化清单」** 一级段：
  - 明确 NEW / RESUME / REVISE 三分支各自必须立即执行的 `file.write` / `file.read` 调用清单
  - 引入 **⛔ 验证门禁**：发起第一次 `delegate_task` 之前必须输出自检确认行（`✅ 状态已初始化: ...`）
  - 引入 **♻️ 运行期持续维护**：每次 delegate 返回后必须立即 `file.write` 更新 checkpoint + 追加 timeline，且如有遗漏必须在下一次 delegate 之前补写
- **deep-research/STATE_PROTOCOL.md** NEW 分支由 5 步通用伪代码 → **7 步具体 file 工具调用清单**（含 `file.list` / `file.write` / `file.read` 显式标注）+ 验证门禁
- **STATE_PROTOCOL.md** RESUME 分支补充 `file.read` / `file.write` 显式标注，timeline 写入 `[RESUME]` 必须在 delegate 前完成
- **STATE_PROTOCOL.md** Checkpoint 维护规则段顶部加入显著警告框：「不是建议是 hard requirement，文件必须真实落盘」
- **Pitfalls 新增 2 条状态红线**（顶部最高优先级）：
  - 启动后必须立即写入状态文件，未写入严禁 delegate
  - 每次 delegate 返回后必须立即更新 checkpoint + timeline，空目录 = 协议违规
- **docs/README.md** Checkpoint 维护流程框补充 hard requirement 警示

### 设计动机

Skills 系统的执行体是 LLM，对模糊的命令式动词（"创建"/"维护"/"初始化"）默认理解为"流程参考"而非"必须真实调用工具"。修复策略不是改变设计，而是把所有应当落盘的动作都用 `file.write` / `file.read` 显式动词替代，并加入"自检确认行"作为验证门禁，让漏写在第一次启动时就能被发现。

---

## [M4.1: 边界一致性修复 — 退路冲突 / 用户响应 / 空 essays / 状态衔接] - 2026-05-17

针对 M4 引入后的 Review 发现的 4 个关键问题做边界对齐修复，避免实际运行时出现死循环、阻塞或状态错乱。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.8.0 | **1.8.1** |

### 修复 (Fixed)

- **S1 Phase 1.3 定稿门禁失败的退路冲突** — 原方案"退回 Phase 1.2 继续精读 / 扩展下载"会触发 round_count > 3 与硬上限冲突；改为：评审者输出 `gate_passed: true/false`，false 时**走「按需追加下载」机制**（round_count 冻结），与 Phase 2/4/5 同一通道
- **S2 Phase 1.2 supervised 用户响应不明** — supervised 模式下 B 类清单审批暂停时，用户必须从三选一明确响应：① "已下载完毕" → 进入下一轮；② "部分跳过 #N1, #N2" → 跳过项写入 `skipped.md`；③ "停止扩展" → 立即结束 Phase 1.2，未下载并入 `extension-deferred.md`
- **S3 Phase 1.2 入口 essays 为空的早期失败** — Step 1 增加显式校验：essays < 启动下限（默认 5 篇）时，supervised 阻塞等用户继续下载；autonomous 写 `[BLOCKED] essays_empty` 并 `[需升级]`。**严禁 essays 完全为空时自动跳 Tier 3**（Tier 3 触发条件是"已有核心论据需补强"）
- **M1 按需追加下载机制与 checkpoint/timeline 状态衔接缺失** — 完整定义状态闭环：
  - SubAgent 输出请求**前**必须先写 partial 草稿到 `drafts/{section}.partial.md`，并在引用位置用 `[NEED:R-{seq}]` 占位符标注
  - 总指挥写入 `extension-extra.md` + checkpoint 的 `extension_requests` 表新增一行（status: pending_user_download / partial_downloaded / read_back / cancelled）
  - timeline 新增事件类型 `[EXTENSION_REQUEST]` / `[EXTENSION_DONE]`
  - 用户补料后调研者 [mode=intensive_read] 单次精读 → 状态回写 `read_back` → 重新 delegate 时优先填充 `[NEED:R-{seq}]` 占位符，避免整章重写
  - 去重规则：DOI 优先严格匹配；无 DOI 时标题 fuzzy match (编辑距离 < 5)

### 变更 (Changed)

- **「按需追加下载」适用范围扩展为 Phase 1.3 / 2 / 4 / 5 通用**（原 Phase 2/4/5 通用）
- **Pitfalls 调整为 7 条**（原 5 条）：新增 essays 为空不得自动 Tier 3 / Phase 1.3 门禁失败不得退回 Phase 1.2 / 追加下载请求必须配套部分草稿 + 占位符
- **STATE_PROTOCOL.md** 新增「按需追加下载（Phase 1.3+）触发时」维护规则段
- **审批表**新增 "Phase 1.2 入口（essays 不足）" 行；"Phase 1.3 完成" 行带 gate_passed 判定逻辑

---

## [M4: 防选题幻觉 — Phase 1 三段式重排] - 2026-05-17

针对"调研者在选题阶段虚构论文文献"的幻觉风险，将 Phase 1（选题与调研设计）拆分为 1.1 / 1.2 / 1.3 三个强约束子阶段，并把整个研究的论文池强制锚定到 `~/Downloads/essays/` 真实落盘 PDF。

> **2026-05-17 微调**: Phase 1.2 滚雪球循环增加**硬上限 3 轮**（不强求收敛）；超出部分写入 `extension-deferred.md`，由后续 Phase 2/4/5「按需追加下载」机制处理。避免论文池过度膨胀同时保留补强通道。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.7.0 | **1.8.0** |
| `research-literature` | 1.6.0 | **1.7.0** |

### 新增 (Added)

- **research-literature 引入 5 种工作模式**：每次 delegate 必须在 context 带 `mode` 字段
  - `draft` (Phase 1.1)：仅基于通识知识输出研究方向草案 + A 类下载清单，**严禁声称已检索/已读过任何论文**
  - `intensive_read` (Phase 1.2)：仅对 `~/Downloads/essays/` 中真实落盘 PDF 写全文笔记
  - `snowball` (Phase 1.2)：基于已精读论文 references 拉 S2/OpenAlex 引文图谱 → 输出 B 类下载清单（含频次统计 + 轮次状态 N/3）
  - `review` (Phase 1.3)：基于真实笔记池审视 Phase 1.1 草案 → 产出 research-design 定稿
  - `synthesis` (Phase 2)：基于已有笔记池写综述，**严禁新检索**
- **deep-research/SKILL.md** 新增 SOP 设计原则段：声明"防选题幻觉"约束 + Phase 1 三段式说明
- **DELEGATION_TEMPLATES.md** 新增模板 1a-1e：分别对应 5 种 mode 的 delegate 模板，每个模板都内置严格的红线与防幻觉约束
- **STATE_PROTOCOL.md** checkpoint 模板新增 P1.1 / P1.2 / P1.3 三个 Phase 行 + 「论文池状态（essays_pool）」字段（含 A 类 / B 类清单进度、滚雪球轮次 N/3、extension-deferred / extension-extra 入口）
- **Pitfalls 新增 5 条防幻觉红线**：Phase 1.1 严禁虚构 / Phase 1.2 必须基于真实 PDF / Phase 1.2 循环硬上限 3 轮 / Phase 1.3 定稿前必须通过引用核查 / 后续阶段走按需追加而非回退 Phase 1.2

### 变更 (Changed)

- **Research SOP 重排**：原 Phase 1（单一阶段）→ 拆为 Phase 1.1 / 1.2 / 1.3，且 Phase 1.2 是"扫描 → 精读 → 滚雪球 → 用户补充下载"的循环过程，**硬上限 3 轮**
- **Phase 1.2 循环硬上限 3 轮**：第 3 轮结束后无论 B 类清单是否仍有未下载论文，立即结束循环进入 Phase 1.3；剩余写入 `extension-deferred.md`
- **新增「按需追加下载」机制（Phase 2/4/5 通用）**：撰写过程中暴露的支撑缺口由 SubAgent 输出「追加下载请求」段，汇总到 `extension-extra.md`，单次精读补入笔记池；**不重启 Phase 1.2 滚雪球循环**
- **Phase 1.3 定稿门禁**：每条核心论断必须 anchor ≥1 篇全文等级笔记，且引用清单中的本地路径必须真实存在于 `~/Downloads/essays/`，否则退回 Phase 1.2 ~~（M4.1 已修订为：走「按需追加下载」机制，不再退回 Phase 1.2，避免与 3 轮硬上限冲突）~~
- **Phase 2 文献池锚定**：综述只能引用 Phase 1.2 已建立的笔记池，不得新检索"凭空"论文
- **docs/README.md → v3.6**：架构模式新增"Phase 1 三段式防选题幻觉"；4.1 SOP 重写 Phase 1.1/1.2/1.3 流程 + 按需追加机制；审批表新增 1.1/1.2/1.3 三行；M4 写入里程碑表

### 设计动机

旧 Phase 1 流程让调研者直接"做初步文献检索 + 制定研究问题 + 理论框架"，但 SubAgent 会在没有真实下载/读过论文的情况下，基于 LLM 通识"凭空生成"具体的作者-年份-标题，污染后续整个研究链路。

新流程通过三个手段彻底切断幻觉路径：
1. **物理隔离**：Phase 1.1 草案模式不接触任何 API 检索 + 真实文献，只输出"建议清单"；用户作为唯一的"下载执行者"，把真实 PDF 放入 `~/Downloads/essays/`
2. **真实性门禁**：Phase 1.2 精读模式仅对真实落盘 PDF 写笔记，文件名/读取失败的 PDF 不允许伪造；Phase 1.3 定稿前必须通过评审者的"引用真实性 4 步核查"
3. **池子封闭**：后续 Phase 2/4/5 引用的论文池就是 `~/Downloads/essays/` + Gbrain 笔记，需扩展必须回 Phase 1.2 走滚雪球流程，不允许任何阶段"凭空新检索"

---

## [Refactor: 长文件拆分 + 冗余精简 + Tier 3 兜底通道] - 2026-05-17

把过长的 SKILL.md 拆分到附录文件，统一收口冗余的 SubAgent 约束块，并把"人工下载兜底通道"
（Tier 3）整合进 research-literature。本次为重构 + 收尾 commit，非功能性新增（除 Tier 3 已在前一会话写入）。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.6.0 | **1.7.0** |
| `research-literature` | 1.4.0 | **1.6.0** |
| `research-writing` | 1.2.0 | 1.2.0（仅链接精简） |
| `research-review` | 1.2.0 | 1.2.0（仅链接精简） |
| `research-advisor` | 1.2.0 | 1.2.0（仅链接精简） |
| `research-analysis` | 1.1.0 | 1.1.0（仅链接精简） |

### 新增 (Added)

- **deep-research/STATE_PROTOCOL.md**：从 SKILL.md 抽出状态根目录结构、命令列表、启动探测伪代码、状态文件模板（index/checkpoint/timeline）、Checkpoint 维护规则，并在末尾首次集中收口「SubAgent 通用约束」（不能与用户交互、`[需升级]`、产出文件清单、修订模式 F-ID）。
- **deep-research/REVISION_WORKFLOW.md**：从 SKILL.md 抽出 REVISE 分支六步走（R1-R6）、与 RESUME 的对比、修订中断恢复、多轮修订规则。
- **deep-research/DELEGATION_TEMPLATES.md**：保留 1 个完整 delegate 模板（启动文献调研），其余 4 个（并行检索 / 数据分析 / 论文撰写 / 质量审查 + 学术顾问）以**差异点**形式呈现，避免大段重复。
- **research-literature/MANUAL_DOWNLOAD.md**：从 SKILL.md 抽出 Tier 3 人工下载兜底通道完整规范（约定文件夹、三层兜底链、触发条件、Step A-D 操作流程、模式差异、与续跑/修订协同、不要做的事）。
- **research-literature/SKILL.md**：在原有数据源章节中增加 Tier 3 入口段（一句话描述 + 链接到 MANUAL_DOWNLOAD.md），并在 Step 3 论文获取流程中接入 Tier 3 触发说明。
- **README.md**：新增「文件结构」章节，呈现重构后的目录树。

### 变更 (Changed)

- **deep-research/SKILL.md**：从 745 行精简到约 210 行：
  - 启动协议、状态文件模板、checkpoint 维护规则全部下沉到 STATE_PROTOCOL.md
  - 修订工作流下沉到 REVISION_WORKFLOW.md
  - 5 个 delegate 模板下沉到 DELEGATION_TEMPLATES.md
  - Research SOP 改为单一总览表（六阶段一行一阶段，列出主要动作、关键 delegate、审批节点）
  - Pitfalls 清单保留全部要点
- **research-literature/SKILL.md**：从 401 行精简到约 307 行：
  - 保留**所有** S2 / OpenAlex / wget / curl 命令示例不动
  - 笔记模板与综述模板压缩注释行
  - Tier 3 详细操作流程下沉到 MANUAL_DOWNLOAD.md
  - 工作流程 Step 3 给 Tier 3 留入口
- **4 个子 Skill（writing / review / advisor / analysis / literature）**：原各自重复的 5 行「SubAgent 约束（当被 delegate_task 调用时）」块替换为 2 行链接 + 一句独立调用说明，整体收口到 STATE_PROTOCOL.md。

### 设计要点

- **单一信息源原则**：SubAgent 通用约束只在 STATE_PROTOCOL.md 写一次，其他 Skill 通过链接引用，避免后续修改时多处不同步。
- **主 SKILL.md 是导航**：让总指挥在加载主 SKILL.md 时能快速看完六阶段 SOP 与 Pitfalls，详细规范按需 Read 附录。
- **附录可独立阅读**：每个附录 Markdown 自成体系，不依赖主 SKILL.md 上下文。
- **保留 curl 全量**：根据用户要求，S2 / OpenAlex / wget / 引文图谱遍历的 curl 示例**全部保留**，不做压缩。
- **行数对照**（精简前/后）：deep-research 745→210；research-literature 401→307（含 Tier 3 入口）；其他 4 个子 Skill 各减 3 行。

### 升级影响

- **向后兼容**：所有 SOP / delegate 协议 / 状态文件结构 / 修订流程**完全不变**。
- **行为差异**：无（仅文档组织重构）。
- **依赖**：无新增。

### 故意未做 (Deferred)

- 进一步合并多 Skill 共享的「修订模式约束」段（4 个子 Skill 各自的修订约束细节有差异，保留各自独立段落）
- 把 docs/ 历史设计文档同步重构（保持原貌作为历史档案）

---

## [M3: 状态持久化 + 断点续跑 + 修订模式] - 2026-05-17

让长流程研究**可中断、可续跑、可修订**。状态持久化在本地文件系统（不依赖 Gbrain），
新增第三种启动模式 `revise` 用于已完成课题的反馈订正。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.5.0 | **1.6.0** |
| `research-writing` | 1.1.0 | **1.2.0** |
| `research-review` | 1.1.0 | **1.2.0** |
| `research-advisor` | 1.1.0 | **1.2.0** |
| `research-literature` | 1.3.0 | **1.4.0** |
| `research-analysis` | 1.0.0 | **1.1.0** |

### 新增 (Added)

#### 状态持久化与断点续跑
- **deep-research**: 新增「启动协议」整章 — 任何启动命令先做探测，分流到 NEW / RESUME / REVISE 三种分支
- **deep-research**: 新增状态文件结构 `~/.hermes/research-state/{slug}/`，含 `index.md` / `checkpoint.md` / `timeline.md` / `revisions/`
- **deep-research**: 新增 5 条命令：
  - `/deep-research resume [课题]` 强制续跑
  - `/deep-research revise [课题] [--feedback-from <路径>]` 强制修订
  - `/deep-research restart [课题]` 强制新建（自动备份原 checkpoint）
  - `/deep-research list` 列出所有研究索引
- **deep-research**: 新增「Checkpoint 维护规则」章节 — 每次 delegate 返回后强制更新 checkpoint + timeline
- **deep-research**: 提供 `index.md` / `checkpoint.md` / `timeline.md` 的标准模板
- **deep-research**: 所有 delegate 模板（调研者/分析师/撰写者/评审者/学术顾问）context 中追加 **「产出文件清单」回报要求**，让总指挥能正确填写 checkpoint

#### 修订模式（REVISE 分支）
- **deep-research**: 新增「修订工作流」整章，定义 6 步流程：
  - R1 反馈收集（多源支持：导师 / 审稿人 / 自查）
  - R2 反馈结构化（ID / 类型 / 严重度 / 章节 / Phase 表格）
  - R3 订正计划制定（拆任务 + 标依赖 + 声明不变更项）
  - R4 执行订正（基于基线最小改动）
  - R5 质量门禁（复用 M1：评审者 + 学术顾问对抗审查）
  - R6 收尾（changes.md + 论文版本号 v1.1-R1）
- **deep-research**: 引入「不可变基线」原则 — 修订生成新版本文件，原文件不动
- **deep-research**: 修订支持多轮（R1, R2, R3...），每轮独立子目录
- **deep-research**: 修订中断 → 沿用 RESUME 机制接管 R{n} 续跑
- **research-writing**: 新增「修订模式约束」章节 — 最小改动 + 逐条对应反馈 ID + 输出 diff 摘要
- **research-review**: 新增「修订模式审查」章节 — 范围限定为变更段落 + 4 项修订专属审查（反馈采纳完整性 / 新增引用真实性 / 基线一致性 / 实质改进）
- **research-advisor**: 新增「修订模式约束」章节 — 4 个修订专属对抗问题（是否真正解决 / 是否引入新问题 / 基线相容性 / 过度修订风险）
- **research-literature**: 新增「修订模式约束」章节 — 检索范围限定 + 笔记追加「触发反馈 ID」字段 + 不覆盖基线笔记
- **research-analysis**: 新增「修订模式约束」章节 — **绝不重做主分析**，仅做补充分析（power / 稳健性 / 子样本），追加而非覆盖

### 变更 (Changed)

- **deep-research**: Pitfalls 增补 4 条 —
  - 每次 delegate 都要求 SubAgent 回报产出文件路径
  - 每次 delegate 返回后立即更新 checkpoint + timeline（不可批量延迟）
  - 修订模式 delegate context 必须含基线路径 + 反馈 ID
  - completed 课题不要直接覆盖（要改请走 REVISE）
- **research-writing**: 升级条件追加 "修订模式下反馈与基线立场不可调和"

### 设计要点

- **状态层与知识层分离**：运行时状态用本地文件系统（轻量、可读），学术知识用 Gbrain（图谱、检索）；signal-detector 不再被 Phase/待办污染
- **不可变基线**：所有修订生成新版本文件（`*-v{x.y}-R{n}.md`），保留完整版本演进史；论文/分析报告/笔记的原始版本永不被覆盖
- **修订是新循环**：REVISE 与 RESUME 在语义、输入、规划方式上有本质区别，因此独立建模
- **质量门禁复用 M1**：修订段落仍走"评审者引用真实性核查 + 学术顾问对抗性审查"，但范围限定，避免重审已通过部分
- **多轮修订支持**：R1（导师反馈）→ R2（外审反馈）→ R3（自查）逻辑天然映射到 `revisions/Rn-{date}/` 目录树

### 升级影响

- **向后兼容**：现有 SOP / delegate 协议不变；启动协议是叠加层，对老用法无破坏
- **行为差异**：
  - 启动 `/deep-research [课题]` 会先探测状态目录（首次为空时自动初始化）
  - SubAgent 输出会在末尾多一段「产出文件清单」
  - 修订流程下论文文件会有版本号后缀 `-v1.1-R1` 等
- **依赖**：
  - 总指挥需要 `file` 工具能读写 `~/.hermes/research-state/`
  - 不需要任何新的 MCP server 或外部依赖

### 故意未做 (Deferred)

- 状态目录跨设备同步（建议用户自行用 git 或 rsync 处理）
- 状态文件指纹/校验（避免外部改动）
- 自动化的 slug 冲突解决（撞 slug 时仍提示用户）
- 修订评审的量化 Rubric（仍沿用 PASS / 小修 / 大修 / REJECT 文字判定）

---

## [M2: 英文文献元数据/图谱通道] - 2026-05-17

本轮迭代为调研者补充**英文文献的发现 + 摘要 + 引文图谱通道**，
通过 `terminal` 直接调免费 API（Semantic Scholar、OpenAlex），不引入新的 MCP server。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `research-literature` | 1.2.0 | **1.3.0** |
| `deep-research` | 1.4.0 | **1.5.0** |

### 新增 (Added)

- **research-literature**: 「数据源与工具」章节重写，划分两条通道：
  - **中文全文**：CNKI（仍主路径）
  - **英文元数据/图谱**：Semantic Scholar + OpenAlex（terminal curl）
- **research-literature**: 提供 S2/OpenAlex 的端点、限频、字段、引文/被引图谱遍历的 curl 示例。
- **research-literature**: 新增「三类笔记体系（来源等级）」表格，明确 全文 / 摘要 / 元数据 的来源与用途边界。
- **research-literature**: 笔记模板新增「外部 ID」与「OA 链接」字段，便于评审者交叉验证：
  - DOI / S2 paperId / OpenAlex Work ID / CNKI ID
  - openAccessPdf.url 或 oa_url
- **research-literature**: 综述模板「检索概况」与「文献获取情况附录」更新，区分来源 + 等级。
- **research-literature**: 效率建议加入"引文滚雪球"和"S2 限频应对"。
- **research-literature**: 升级条件增加 "S2/OpenAlex 持续不可达"。
- **deep-research**: 调研者 delegate 模板（单任务 + 并行 3 路）的 `toolsets` 加入 `terminal`，工作步骤更新为"双通道发现 + 按等级获取"。

### 变更 (Changed)

- **research-literature** Step 2 / Step 3 工作流由"先列标题清单 → 逐篇下载 CNKI"
  扩展为"双通道发现 → 按等级（全文 / 摘要 / 元数据）分别处理"。
- **research-literature** Step 3 文件夹依旧扁平，**不引入 S1/S2/S3 文件命名分流**（保持精简，由笔记顶部"来源等级"字段区分）。

### 设计要点

- **API 免费、无需 key**：S2 / OpenAlex 都可匿名调用；S2 限频 100/5min，OpenAlex 10 req/s
- **不强求英文全文**：S2/OpenAlex 主要给"摘要 + 引文图谱"，仅当返回 `openAccessPdf` / `oa_url` 时才能拿全文
- **图谱信息直接进 Gbrain**：S2 references/citations 端点返回的引文关系，借助 signal-detector 自动建图
- **失败可降级**：API 不可达 → 降级为 web + Google Scholar；OA PDF 拿不到 → 降级为摘要笔记

### 升级影响

- **向后兼容**：现有中文 / CNKI 流程不变，新通道是叠加能力
- **行为差异**：调研者输出的获取状态表会多两列（来源、等级）；英文文献覆盖能力提升
- **依赖**：需要 SubAgent 的 `terminal` 环境可访问外网（`api.semanticscholar.org` / `api.openalex.org`）

---

## [M1: 防幻觉与对抗性审查] - 2026-05-17

本轮迭代聚焦两个核心目标：**保证论文引用的真实性（防幻觉）** 与 **强化对抗性审查**。
所有改动以最小必要为原则，未触及双模式编排、Gbrain 路径策略、CNKI 集成等既有设计。

### Skill 版本变更

| Skill | 旧版本 | 新版本 |
|---|---|---|
| `deep-research` | 1.3.0 | **1.4.0** |
| `research-literature` | 1.1.0 | **1.2.0** |
| `research-writing` | 1.0.0 | **1.1.0** |
| `research-review` | 1.0.0 | **1.1.0** |
| `research-advisor` | 1.0.0 | **1.1.0** |
| `research-analysis` | 1.0.0 | 1.0.0（未变） |

### 新增 (Added)

#### 防幻觉机制
- **research-writing**: 新增「引用规则（防幻觉，强制）」章节，确立三条铁律：
  1. 只引读过的（必须有 Gbrain 笔记）
  2. 来源等级匹配论断（具体数据/方法细节仅限全文笔记可引）
  3. 不做二手转引
- **research-writing**: 撰写者每次交付必须在草稿末尾附「引用清单」，含笔记路径 / 来源等级 / 引用页段。
- **research-literature**: 文献笔记模板顶部新增 `**来源等级**` 字段（全文 / 摘要 / 元数据）。
- **research-literature**: 质量标准追加：全文笔记必须含 ≥2-3 段原文 quote（带页码/章节定位）；摘要笔记不得伪造 quote。
- **research-review**: 新增「引用真实性核查（必做，最高优先级）」4 步流程：
  - Step 1：引用清单完整性
  - Step 2：笔记存在性核验（gbrain hybrid-search / brain-ops read）
  - Step 3：来源等级匹配
  - Step 4：关键论断的原文比对
- **research-review**: 输出格式增加「引用真实性核查结果」区块 + 决策档位（PASS / MINOR_REVISION / MAJOR_REVISION / REJECT）。

#### 对抗性审查
- **research-advisor**: 角色重定义，加入 **Devil's Advocate (对抗性审查)** 职责，明确"主动攻击研究"为核心使命。
- **research-advisor**: 新增「对抗性提问（必做）」章节，强制回答 4 个问题：
  1. 替代解释
  2. 证伪路径
  3. 样本/边界
  4. 理论选择
- **research-advisor**: 输出格式新增「对抗性审查」章节 + 攻击点分级（🔴致命 / 🟡严重 / 🟢警告）。
- **research-advisor**: 原则追加 **对抗性默认**（先假设研究有问题）。

#### 编排层提醒
- **deep-research**: 撰写者 / 评审者 / 学术顾问的 delegate 模板 context 中分别追加 **强制提醒**，确保 SubAgent 不会跳过新增的防幻觉与对抗性流程。

### 变更 (Changed)

- **research-writing**: 升级条件追加 "核心论断找不到全文级别支撑文献"。
- **research-review**: 升级条件追加 "≥3 处疑似幻觉" 与 "同一论文多处自相矛盾"。

### 未变更 (Unchanged)

- 双模式 (supervised / autonomous) 行为
- Gbrain 路径策略（仍由总指挥运行时设计）
- CNKI 集成与 `/cnki-paper-downloader` 调用方式
- SubAgent 隔离 / 升级机制 / 审批节点
- `research-analysis` 全部内容

### 故意未做 (Deferred)

为保持本轮改动最小，以下方案推迟到 M2 / M3 再视试跑反馈引入：

- 复杂的 Paper ID 协议（PID:source-shortid）
- 量化 Rubric（多维度加权打分 + 阈值）
- 机器可读 yaml 输出区块
- 三类笔记的文件级命名分流（S1/S2/S3-fulltext/abstract/metadata.md）
- Gate 自动判定逻辑
- Semantic Scholar / OpenAlex API 接入

### 防御层一览

```
                    撰写者
                       │
              附「引用清单」
                       │
                       ▼
            ┌──── 评审者 ────┐
            │  4 步真实性核查 │
            └────────┬────────┘
                     │
              PASS / 返工
                     │
                     ▼
              学术顾问 (Challenger)
              4 个对抗性问题 + 攻击点
                     │
                     ▼
                  Phase 6
```

### 升级影响

- **向后兼容**：所有改动均为追加约束，不破坏现有 SOP / delegate 协议。
- **行为差异**：撰写者输出会变长（多一节"引用清单"）；评审者轮次可能增加（多了真实性核查这一关）；学术顾问报告会更"刺"（多了 4 个对抗问题）。
- **建议**：升级后用一个小课题先跑一遍，观察评审者退回率与学术顾问攻击点质量，再决定是否进入 M2。
