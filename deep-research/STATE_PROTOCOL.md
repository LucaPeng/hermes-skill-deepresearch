# State Protocol — 状态持久化与启动协议

> 本文件是 `deep-research` 的运行时状态规范。所有状态文件存于本地文件系统，不写入 Gbrain（Gbrain 用于知识，不用于运行时状态）。

---

## 状态根目录与文件结构

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

---

## 命令一览

| 命令 | 行为 |
|---|---|
| `/deep-research [课题]` | 自动探测：命中 in_progress → resume；命中 completed → 询问 (supervised) / 默认 revise (autonomous)；未命中 → new |
| `/deep-research supervised [课题]` / `/deep-research autonomous [课题]` | 同上，但显式指定模式 |
| `/deep-research resume [课题]` | 强制续跑（in_progress 课题）；找不到则报错 |
| `/deep-research revise [课题]` | 强制进入修订（completed 课题）；找不到则报错 |
| `/deep-research revise [课题] --feedback-from <路径或文本>` | 直接传入反馈源；否则交互式收集 |
| `/deep-research restart [课题]` | 强制新建（自动备份原 checkpoint 为 `checkpoint.md.bak.{ts}`） |
| `/deep-research list` | 列出 index.md 中所有研究（slug / 课题 / 状态 / 模式 / 最后更新） |

---

## 启动探测流程（伪代码）

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

### NEW 分支（新建研究）— 必须按顺序执行下列 file 工具调用

> ⚠️ 这是 hard requirement，**不是参考流程**。下列每一步都是真实的 file 工具调用，不是脑内决策。任一遗漏会导致 RESUME 失败。

```
1. 推算 slug
   - 去标点 + 拼音/英文小写 + 截 30 字 + 短哈希
   - 例：「社交电商中消费者信任的形成机制」→ social-commerce-trust-{hash6}

2. file.list ~/.hermes/research-state/
   - 不存在则用 file.write 工具创建（部分 IDE 可通过 file.write 自动建目录）

3. file.write ~/.hermes/research-state/{slug}/checkpoint.md
   - 严格套用本文档「状态文件模板」中的 checkpoint.md 模板
   - 必填: last_updated（now）/ mode / status=in_progress / 课题元信息 /
          Phase 进度（全部为 ⬜ pending）/ 当前待办（"启动 Phase 1.1"）

4. file.write ~/.hermes/research-state/{slug}/timeline.md
   - 首行: # Timeline: {课题}
   - 第二行: - {now} [START] mode={supervised|autonomous}

5. file.read ~/.hermes/research-state/index.md
   - 不存在则 file.write 创建（含表头 | Slug | 课题描述 | 状态 | 模式 | 最后更新 | 路径 |）
   - 用 file.write（覆盖式）追加本课题一行（| {slug} | ... | in_progress | {mode} | {now} | ~/.hermes/research-state/{slug}/ |）

6. ⛔ 验证门禁（必须输出给用户/日志）
   - 用 file.read 重新读三个文件确认内容已落盘
   - 输出自检确认行（见 SKILL.md 启动后强制状态初始化清单段）
   - 若任一文件 size=0 或读取失败 → 不得 delegate，先重写

7. 进入 Phase 1.1（此时才允许第一次 delegate_task）
```

### RESUME 分支（断点续跑）
1. `file.read` `{slug}/checkpoint.md` 解析 Phase 进度 + 当前子任务 + 待办
2. `file.read` `{slug}/timeline.md` 最近 20 条事件 → 构建上下文摘要
3. supervised：向用户报告续跑点并请确认
   ```
   🔁 检测到已有研究 checkpoint
   课题: {...}
   上次中断点: Phase {X} / {子任务名称}
   下一步: {待办列表}
   是否续跑？(yes / no / 我要修改方向)
   ```
   autonomous：直接续跑，不询问
4. ✅ `file.write` 在 timeline.md 追加 `- {now} [RESUME] from Phase {X} / 待办首项: {...}`（必须在 delegate 前写入）
5. 严格从 checkpoint「待办」第一项开始 delegate，**不重做已完成项**
6. 每次 delegate 返回后立即 `file.write` 更新 checkpoint + 追加 timeline（见下方 Checkpoint 维护规则）

### REVISE 分支
详见 [REVISION_WORKFLOW.md](./REVISION_WORKFLOW.md)。

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
| P1.1 调研草案与获取建议 | ✅ done | ... | ... | meta/acquisition-plan.md |
| P1.2 论文精读与滚雪球 | 🟡 in_progress (round 2) | ... | — | literature/intensive-read-round-1.md, extension-needed-round-1.md |
| P1.3 调研设计 review 与定稿 | ⬜ pending | — | — | — |
| P2 系统性文献综述 | ⬜ pending | — | — | — |
| P3 数据采集 | ⬜ pending | — | — | — |
| P4 数据分析 | ⬜ pending | — | — | — |
| P5 论文撰写 | ⬜ pending | — | — | — |
| P6 修订定稿 | ⬜ pending | — | — | — |

## 论文池状态（essays_pool）
- 本地路径锚: `~/Downloads/essays/`
- A 类清单（Phase 1.1）: 列入 25 篇 / 已下载 18 篇 / 已精读 18 篇
- Phase 1.2 滚雪球轮次: round 2 / 3（**硬上限 3 轮**）
  - round 1 B 类清单: 列入 8 篇 / 已下载 5 篇 / 已精读 5 篇 / 已跳过 0 篇
  - round 2 B 类清单: 列入 6 篇 / 已下载 4 篇 / 已精读 0 篇（进行中） / 已跳过 1 篇
  - round 3: 待执行
- 待人工归类 PDF: 0
- 读取失败 PDF: 0
- Gbrain 笔记池: 全文 23 / 摘要 0 / 元数据 12
- **extension-deferred.md**: Phase 1.2 三轮结束后未消化的 B 类论文（暂未下载，留待后续按需触发）
- **extension-extra.md**: Phase 1.3/2/4/5 按需追加下载清单（撰写过程中按需补充）

## 按需追加下载请求（extension_requests）

> 每次 SubAgent 在 P1.3/P2/P4/P5 触发「追加下载请求」时，总指挥追加一行；用户补料、调研者精读完成后更新 status。
> 去重规则: DOI 优先严格匹配；无 DOI 时标题 fuzzy match (编辑距离 < 5)。

| Req ID | 触发 | 触发时间 | 论文数 | 优先级 | 状态 | 草稿占位 | 完成时间 |
|---|---|---|---|---|---|---|---|
| R-1 | P1.3 (gate fail) | 2026-05-18 10:20 | 3 | 🔴 | read_back | drafts/research-design.partial.md | 2026-05-18 14:50 |
| R-2 | P5/Ch3 | 2026-05-22 09:10 | 5 | 🟡 | pending_user_download | drafts/Ch3.partial.md | — |

状态枚举：
- `pending_user_download`：已写入 extension-extra.md，等待用户下载
- `partial_downloaded`：用户已下载部分论文，剩余仍 pending
- `read_back`：用户下载完毕 + 调研者已精读 + 笔记池已更新
- `cancelled`：用户主动放弃该请求（autonomous 模式下也可标"按现有笔记池继续"）

## 当前 Phase 内的子任务
- ✅ delegate→调研者 [mode=intensive_read]: round-1 精读（返回 5 篇笔记）
- 🟡 delegate→调研者 [mode=intensive_read]: round-2 精读（中断时进行中）
- ⬜ delegate→调研者 [mode=snowball]: round-2 滚雪球
- ⬜ delegate→评审者: round-2 引用核查

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
- 2026-05-15 10:05 [PHASE] P1.1 begin
- 2026-05-15 11:20 [DELEGATE] research-literature [mode=draft]: 输出 acquisition-plan.md (A 类 25 篇)
- 2026-05-15 14:00 [DECISION] 草案审批通过
- 2026-05-15 14:01 [PHASE] P1.1 done → P1.2 begin (round 1)
- 2026-05-15 18:00 [DELEGATE] research-literature [mode=intensive_read, round 1]: 18 篇 → 18 笔记
- 2026-05-15 22:00 [DELEGATE] research-literature [mode=snowball, round 1]: → extension-needed-round-1.md (8 篇)
- 2026-05-16 10:00 [USER_RESPONSE] round 1: "已下载完毕"（5/8 实际下载）
- 2026-05-16 14:00 [PHASE] round 1 done → round 2 begin
- 2026-05-17 18:00 [PHASE] P1.2 done (rounds=3, deferred=4) → P1.3 begin
- 2026-05-18 10:20 [EXTENSION_REQUEST] R-1 | trigger=P1.3 | papers=3 | prio=🔴
- 2026-05-18 14:50 [EXTENSION_DONE] R-1 → 笔记池新增 3 篇全文笔记
- 2026-05-18 16:00 [PHASE] P1.3 done (gate_passed=true) → P2 begin
- 2026-05-22 09:10 [EXTENSION_REQUEST] R-2 | trigger=P5/Ch3 | papers=5 | prio=🟡
- 2026-05-20 09:30 [REVISION_BEGIN] R1 (导师反馈)
- 2026-05-21 17:00 [REVISION_DONE] R1 → 论文版本 v1 → v1.1-R1
```

事件类型枚举（追加式日志）：
- `[START]` / `[RESUME]` / `[COMPLETED]`
- `[PHASE] X begin/done` — 含 round_count 与 P1.2 deferred 数等关键状态
- `[DELEGATE] <skill> [mode=...]: <summary>` — SubAgent 调用记录
- `[USER_RESPONSE] round N: "..."` — Phase 1.2 用户三选一响应
- `[BLOCKED] <reason>` — 阻塞事件（如 essays_empty）
- `[EXTENSION_REQUEST] R-{seq} | trigger=P{x} | papers=N | prio=...` — 按需追加下载请求
- `[EXTENSION_DONE] R-{seq} → 笔记池新增 N 篇` — 请求 fulfilled
- `[DECISION] <description>` / `[REVISION_BEGIN/DONE]`

---

## Checkpoint 维护规则（强制）

> ⚠️ **这一段是 hard requirement，不是建议**。`~/.hermes/research-state/{slug}/` 目录中的文件**必须**真实落盘到磁盘（用 `file` 工具调用），而不是只在脑内"维护"。如果你只在响应中口头描述"我已更新 checkpoint"但没调用 `file.write`，等于状态丢失。

每次 delegate 返回后，**总指挥必须立即**：
1. 解析 SubAgent 返回结果，提取产出文件的绝对路径
2. ✅ **调用 `file.write`** 覆盖式更新 `~/.hermes/research-state/{slug}/checkpoint.md`：
   - 在「Phase 进度」更新当前 Phase 的产出
   - 在「子任务」勾选完成项 / 添加新项
   - 在「待办」更新下一步
   - 在「已建立的产出物」追加新路径
   - 顶部 `last_updated` 更新为 now
3. ✅ **调用 `file.write`** 追加到 `~/.hermes/research-state/{slug}/timeline.md`：
   - `- {now} [DELEGATE] <skill>[mode=...]: <goal> → <returned summary>`

每次 Phase 切换前：
1. 写入 checkpoint.md「关键决策记录」
2. 写入 timeline.md `[PHASE] X done → Y begin`

按需追加下载（Phase 1.3+）触发时：
1. SubAgent 输出「追加下载请求」前必须先把当前章节部分草稿写到
   `~/.hermes/research-state/{slug}/drafts/{section}.partial.md`，
   并在草稿中以 `[NEED:R-{seq}]` 占位符标注待补位置
2. 总指挥追加 `extension_requests` 表一行，status=`pending_user_download`
3. 用户补料 + 调研者 [mode=intensive_read] 单次精读完成后：
   - 更新该 R-{seq} 的 status: `read_back` + 完成时间
   - timeline 追加 `[EXTENSION_DONE]`
   - 重新 delegate 该章节，SubAgent 优先填充 `[NEED:R-{seq}]` 占位符

研究全部完成时：
1. 把 checkpoint.md 顶部 `status` 改为 `completed`
2. 同步更新 index.md 中该课题状态为 `completed`
3. timeline.md 追加 `[COMPLETED]`

---

## SubAgent 通用约束（在每次 delegate context 中复述）

所有 SubAgent 共享下列约束，总指挥在 delegate context 开头务必声明：

- **不能直接与用户交互** — SubAgent 无法发消息给用户、请求确认或提问
- **需要升级的问题** — 在输出中标注 `[需升级]` 并说明原因，由总指挥决定是否升级
- **必须回报产出文件路径** — 输出末尾附「产出文件清单」段，列出所有写入文件的绝对路径，便于总指挥更新 checkpoint
- **修订模式必须含基线路径 + 反馈 ID** — 若处于修订模式，context 必须显式声明基线文件路径与反馈 ID（F-ID）；输出含 diff 摘要 + 新版本路径