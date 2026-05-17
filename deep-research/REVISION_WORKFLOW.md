# Revision Workflow — 修订工作流（REVISE 分支）

> 本文件是 `deep-research` 的修订模式规范。修订**不是**"在原文件上 patch"，而是**生成新版本**，原产出物不可变（immutable baseline）。

---

## 设计哲学

> 修订**不是**"在原文件上 patch"，而是**生成新版本**，原产出物不可变（immutable baseline）。

## 与 RESUME 的区别

| 维度 | resume | revise |
|---|---|---|
| 触发 | 上次中断 | 外部反馈到来 |
| 输入 | 上次待办 | 反馈文档 |
| 工作量模型 | 顺序推进 P1→P6 | 跳进特定 Phase 做局部订正 |
| 是否需要规划 | 否 | **是**（拆反馈 → 任务清单） |
| 历史数据角色 | 上下文 | **不可变基线** |

---

## 修订六步走

### Step R1: 反馈收集
- 来源支持：用户口述 / 文件路径 / 粘贴文本 / Gbrain 中的笔记
- 多来源支持：feedback.md 可有多个 source 块（导师 / 审稿人A / 审稿人B / 自查）
- 原文写入 `revisions/R{n}-{date}/feedback.md`，**不做任何修改和润色**
- 命令参数 `--feedback-from <路径>` 可直接读入文件

### Step R2: 反馈结构化
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

### Step R3: 订正计划制定
- 每条反馈 → 1+ 个 delegate 任务
- 标注前置依赖（如"补理论"前要先"补检索"）
- 写入 `revisions/R{n}/revision-plan.md`
- 必须显式声明"与原研究的关系"：
  - 不变更的部分（RQ / 假设 / 数据 / 主分析）
  - 仅订正的部分（具体章节 + 反馈 ID）

### Step R4: 执行订正（按依赖顺序）
每个 delegate 任务的 context **必须包含**：
- 原产出文件路径（基线）
- 对应反馈条目原文（F-ID）
- 修订要求："基于基线最小改动，不得整章重写（除非反馈明确要求）"
- 输出要求：diff 摘要 + 新版本路径

### Step R5: 质量门禁（复用 M1 链路）
- delegate → 评审者: 引用真实性核查（**重点关注新增/变更的引用**）
- delegate → 学术顾问: 对抗性审查（**聚焦修订段落是否真解决了反馈**）

### Step R6: 收尾
- 写 `revisions/R{n}/changes.md`（章节级 diff 摘要 + 引用增减 + 字数变化）
- 更新 `revisions/R{n}/status.md` → `completed`
- 主 `checkpoint.md`「修订历史」追加一行 R{n} 完成
- `timeline.md` 追加 `[REVISION_DONE] R{n} → 论文版本 vX.Y-R{n}`
- 论文文件版本规则：`论文/main-draft-v1.md` 不动，新增 `论文/main-draft-v{x.y}-R{n}.md`

---

## 修订中断恢复
若 R{n} 中途中断（token 耗尽 / 用户暂停）：
- `revisions/R{n}/status.md` 保留 in_progress 状态
- 用户再次启动 → 启动协议探测到 status=revising → 沿用 RESUME 机制接管该轮 R{n}
- 不重做已完成的订正任务

## 多轮修订
一个研究可有 R1, R2, R3... 多轮修订：
- 每轮独立子目录，互不干扰
- 历史 diff 可在主 checkpoint 的「修订历史」按时序回看
- 论文文件版本递增：v1 → v1.1-R1 → v1.2-R2 → v1.3-R3
