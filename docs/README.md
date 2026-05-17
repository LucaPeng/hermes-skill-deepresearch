# 多 Agent DeepResearch 论文系统 — 项目方案文档

> **版本**: v3.0  
> **日期**: 2026-05-17  
> **论文类型**: 社科/管理类  
> **技术选型**: Hermes Agent + Gbrain  
> **架构模式**: Skills 协同编排 + 双模式运行 (supervised / autonomous)

---

## 一、项目概述

### 1.1 目标

构建一个基于多 Agent 协同的学术研究系统，覆盖从**初步调研**到**论文终稿**的全生命周期。通过 Hermes Skills 定义不同研究角色，由总指挥 Skill 通过 `delegate_task` 动态生成 SubAgent 团队协同工作。

### 1.2 核心理念

- **Skills 即能力**：每个研究角色对应一个 Hermes Skill，Skill 定义角色的行为规范和工作流程
- **动态编排**：总指挥 Skill 按 SOP 阶段动态生成 SubAgent，无需常驻 Agent 占用资源
- **双模式运行**：`supervised`（Human-in-the-Loop）和 `autonomous`（全自动）满足不同场景
- **知识即资产**：Gbrain 知识图谱自动构建实体关联，研究过程中的知识沉淀可跨项目复用
- **零侵入安装**：无需修改 Hermes 的 SOUL.md 或 config.yaml，只需安装 Skills

### 1.3 技术选型

| 维度 | 选择 | 理由 |
|---|---|---|
| Agent 框架 | **Hermes Agent**（共享实例） | 原生 `delegate_task`、Skills 系统、MCP 集成 |
| 知识大脑 | **Gbrain**（共享实例） | PGLite 知识库、混合搜索 + 知识图谱、MCP Server (30+ tools) |
| 论文数据源 | **CNKI**（中国知网） | 已有 `/cnki-paper-downloader` Skill 对接 |
| 执行方式 | Hermes Skills + delegate_task | 总指挥编排 SubAgent，各 SubAgent 加载对应 Skill |
| 协同方式 | SubAgent 返回值 | SubAgent 无法直接交互用户，所有产出通过返回值传递 |

### 1.4 系统架构

```
用户: "/deep-research 帮我研究 [课题]"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  Hermes Agent (共享实例)                                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ /deep-research Skill (总指挥 PI)                      │    │
│  │  • 理解研究需求 → 设计 Gbrain 存储结构               │    │
│  │  • SOP 分阶段编排 → delegate_task 生成 SubAgent      │    │
│  │  • 汇总结果 → 推进下一阶段                           │    │
│  │  • supervised: 关键节点暂停请求审批                    │    │
│  │  • autonomous: 全自动推进，完成后汇报                 │    │
│  └──────────┬──────────┬──────────┬──────────┬──────────┘    │
│        delegate   delegate   delegate   delegate              │
│             │          │          │          │                 │
│             ▼          ▼          ▼          ▼                 │
│       ┌──────────┐┌──────────┐┌──────────┐┌──────────┐       │
│       │/research-││/research-││/research-││/research-│       │
│       │literature││analysis  ││writing   ││review    │       │
│       │+ /cnki-  ││          ││          ││          │       │
│       │ downlder ││          ││          ││          │       │
│       └──────────┘└──────────┘└──────────┘└──────────┘       │
│                                                   ↕           │
│       ┌──────────┐                         /research-advisor  │
│       │里程碑独立 │◄── on-demand delegate ─┘                   │
│       │评审      │                                            │
│       └──────────┘                                            │
└───────────────────────────────┬───────────────────────────────┘
                                │ MCP Protocol
                                ▼
┌───────────────────────────────────────────────────────────────┐
│  Gbrain (共享实例)                                             │
│  • Brain Pages (知识页面)                                      │
│  • Knowledge Graph (实体关系图谱)                              │
│  • Hybrid Search (向量 + 全文)                                │
│  • Signal Detector (自动实体抽取)                              │
│  • MCP Server (30+ tools)                                     │
└───────────────────────────────────────────────────────────────┘
```

---

## 二、团队角色

### 2.1 角色矩阵

| 角色 | 对应 Skill | 职责 | 生成方式 |
|---|---|---|---|
| **总指挥 (PI)** | `/deep-research` | 任务编排、资源调度、里程碑把控、审批门控 | 用户直接调用 |
| **调研者** | `/research-literature` | 文献检索、综述撰写、理论框架梳理、研究缺口识别 | delegate_task |
| **数据分析师** | `/research-analysis` | 数据清洗、统计分析、假设检验、可视化 | delegate_task |
| **撰写者** | `/research-writing` | 论文撰写、结构组织、引文管理 | delegate_task |
| **评审者** | `/research-review` | 逻辑检查、引用验证、方法评估、格式审查 | delegate_task |
| **学术顾问** | `/research-advisor` | 里程碑独立评审、理论框架验证、评级 | delegate_task (on-demand) |

### 2.2 已移除的角色

- **系统开发者**：不再作为独立 Agent。研究中的技术需求处理方式：
  - `supervised` 模式：总指挥输出结构化需求文档，由人类决定实现方式
  - `autonomous` 模式：总指挥直接使用 terminal/web 工具尝试自行解决

### 2.3 SubAgent 通信机制

```
通信规则:
1. SubAgent 只能看到 delegate_task 传入的 context
2. SubAgent 不能与用户直接交互
3. SubAgent 遇到问题 → 在输出中标注 [需升级]
4. 总指挥读取 SubAgent 返回值 → 决定是否升级到用户
5. SubAgent 之间不能直接通信 → 必须通过总指挥中转或 Gbrain 共享
```

---

## 三、双模式运行

### 3.1 模式对比

| 维度 | supervised (默认) | autonomous |
|---|---|---|
| 启动命令 | `/deep-research [课题]` | `/deep-research autonomous [课题]` |
| 审批节点 | 每阶段结束暂停，等待人类确认 | 无暂停，全自动推进 |
| 系统开发需求 | 输出需求文档，暂停等待人类处理 | 自行使用 terminal/web 解决 |
| 结果汇报 | 每阶段汇报 + 请求审批 | 全部完成后统一汇报 |
| 适用场景 | 重要研究、需要人类把控方向 | 探索性调研、时间紧迫 |

### 3.2 supervised 模式审批节点

| 阶段 | 审批内容 | 用户操作 |
|---|---|---|
| Phase 1 完成 | 研究问题 + 理论框架 | "可以" / 提出修改 |
| Phase 2 完成 | 文献综述初稿 | "可以" / "需要补充 X 方向" |
| Phase 3 完成 | 研究设计方案 | "可以" / 修改方法 |
| Phase 4 完成 | 数据分析结论 | "可以" / 调整分析 |
| Phase 5 完成 | 论文全文 + 终审意见 | "可以" / 修改 |
| Phase 6 完成 | 最终定稿 | "通过" |

---

## 四、SOP 流程

### 4.1 研究生命周期

```
Phase 1: 选题与调研设计
  └─ delegate → 调研者: 初步文献检索 (CNKI + web)
  └─ 总指挥 + 学术顾问: 确定研究问题和理论框架
  └─ supervised: ⏸️ 人类审批

Phase 2: 系统性文献综述
  └─ delegate → 调研者(并行×3): 多方向系统检索
  └─ delegate → 评审者: 综述初审
  └─ supervised: ⏸️ 人类审批

Phase 3: 研究设计
  └─ 总指挥: 制定方案（问卷/实验/案例/...）
  └─ delegate → 数据分析师: 设计采集方案
  └─ supervised: ⏸️ 人类审批

Phase 4: 数据分析
  └─ delegate → 数据分析师: 执行分析
  └─ delegate → 调研者: 解读结果的理论含义
  └─ delegate → 学术顾问: 评估学术价值
  └─ supervised: ⏸️ 人类审批

Phase 5: 论文撰写与审查
  └─ delegate → 撰写者: 逐章撰写
  └─ delegate → 评审者: 逐章审查（多轮迭代）
  └─ delegate → 学术顾问: 全文终审
  └─ supervised: ⏸️ 人类审批

Phase 6: 修订定稿
  └─ delegate → 撰写者: 根据反馈修订
  └─ delegate → 评审者: 终审确认
  └─ supervised: ⏸️ 人类最终审定
```

---

## 五、工具链

### 5.1 论文获取

| 工具 | 用途 | 调用方式 |
|---|---|---|
| `/cnki-paper-downloader` | 从 CNKI 下载论文全文 | Hermes Skill，输入论文完整标题，每次一篇 |
| Web 搜索 | 发现论文标题、获取摘要 | Hermes 内置 web 工具 |
| Google Scholar | 英文文献发现、引用网络 | 通过 web 工具访问 |

### 5.2 Gbrain 能力

| 能力 | 研究用途 | 调用 Agent |
|---|---|---|
| `brain-ops` | 知识页面读写 | 全体 |
| `hybrid-search` | 语义 + 全文检索 | 全体 |
| `signal-detector` | 自动实体抽取和关系识别 | 写入时自动触发 |
| `maintain` | 知识库健康检查 | 总指挥 |
| MCP tools (30+) | 按需调用 | 按场景 |

### 5.3 其他工具

| 工具 | 用途 | 使用者 |
|---|---|---|
| Terminal (Hermes) | 执行 Python 统计脚本 | 数据分析师 |
| File 工具 | 读写本地文件 | 全体 |
| Web 工具 | 网络信息获取 | 调研者、总指挥 |

---

## 六、Gbrain 知识库策略

### 6.1 设计原则

- **不硬编码路径**：Gbrain 是共享实例，可能同时服务多个项目
- **总指挥运行时设计**：启动研究时由总指挥根据课题设计命名空间和页面结构
- **自动知识图谱**：`signal-detector` 在每次写入时自动抽取实体和关系

### 6.2 建议分区（总指挥参考）

| 分区 | 内容 | 写入者 |
|---|---|---|
| 项目管理 | 元信息、决策日志、进度 | 总指挥 |
| 文献知识 | 笔记、综述、理论框架 | 调研者 |
| 数据分析 | 方案、报告、结论 | 数据分析师 |
| 写作产出 | 草稿、审查反馈 | 撰写者 |

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

## 七、CNKI 集成方案

### 7.1 工作流程

```
Step 1: 搜索策略设计
  关键词拆解 → CNKI 检索式 → 时间范围 → 来源类别

Step 2: 论文标题发现
  web 搜索 "site:cnki.net [关键词]"
  Google Scholar 辅助
  综述论文滚雪球

Step 3: 逐篇下载
  /cnki-paper-downloader [论文完整标题]
  每次一篇，标题必须准确

Step 4: 精读与笔记
  结构化笔记 → 写入 Gbrain

Step 5: 综述整合
  文献获取状态表 (✅/⚠️/❌)
```

### 7.2 获取状态追踪

| 状态 | 含义 | 后续处理 |
|---|---|---|
| ✅ 已下载全文 | 从 CNKI 成功下载 | 必须精读 |
| ⚠️ 仅摘要 | 无法获取全文 | 可作补充引用，不作核心论据 |
| ❌ 未获取 | 下载失败或未收录 | 标注原因，考虑替代文献 |

---

## 八、升级机制

### 8.1 升级路径

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

### 8.2 升级条件

| 条件 | 处理方 |
|---|---|
| 超出能力范围 | 总指挥 → 学术顾问 |
| 理论争议 | 学术顾问评估 |
| 战略方向变更 | 升级到人类 (supervised) |
| 大量文献无法获取 | 升级到人类 (supervised) |
| autonomous 模式下任何问题 | 总指挥自行决策或记录 |

---

## 九、风险评估

| 风险 | 等级 | 缓解措施 |
|---|---|---|
| LLM 幻觉影响学术准确性 | 高 | 评审者交叉验证 + 学术顾问独立审查 |
| CNKI 论文获取失败率高 | 中 | 获取状态追踪 + web 补充摘要信息 |
| SubAgent 产出质量不稳定 | 中 | 完整 context 传递 + 严格输出格式要求 |
| Gbrain 知识图谱噪声 | 低 | 定期 maintain + signal-detector 自动修正 |
| autonomous 模式方向偏离 | 中 | 学术顾问里程碑审查兜底 |

---

## 十、项目里程碑

| 阶段 | 状态 | 交付物 |
|---|---|---|
| M1: Skills 设计与开发 | ✅ 完成 | 6 个 Hermes Skills + README |
| M2: CNKI 集成 | ✅ 完成 | `/cnki-paper-downloader` + 调研者工作流更新 |
| M3: 安装验证 | ⬜ 待做 | Skills 安装到 Hermes + 端到端测试 |
| M4: 首个课题试跑 | ⬜ 待做 | 选定课题 → 完整流程验证 |
| M5: 迭代优化 | ⬜ 持续 | 根据试跑反馈调整 Skills |
