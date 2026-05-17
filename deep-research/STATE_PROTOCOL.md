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

## SubAgent 通用约束（在每次 delegate context 中复述）

所有 SubAgent 共享下列约束，总指挥在 delegate context 开头务必声明：

- **不能直接与用户交互** — SubAgent 无法发消息给用户、请求确认或提问
- **需要升级的问题** — 在输出中标注 `[需升级]` 并说明原因，由总指挥决定是否升级
- **必须回报产出文件路径** — 输出末尾附「产出文件清单」段，列出所有写入文件的绝对路径，便于总指挥更新 checkpoint
- **修订模式必须含基线路径 + 反馈 ID** — 若处于修订模式，context 必须显式声明基线文件路径与反馈 ID（F-ID）；输出含 diff 摘要 + 新版本路径