# AGENTS.md — Trae IDE 开发指引

> 本文件供 Trae AI IDE 读取，指导开发者完成多 Agent DeepResearch 论文系统的后续构建与维护。

---

## 项目基本信息

- **项目名称**: Multi-Agent DeepResearch Paper System
- **技术栈**: Hermes Agent + Gbrain (均为共享实例)
- **核心实现**: 6 个 Hermes Skills (已完成)
- **架构模式**: Skills-only (无需修改 Hermes SOUL.md / config.yaml)
- **参考文档**: `project-docs/README.md`（项目方案）

---

## 项目结构

```
deep-research/
├── hermes-skills/                   # 核心交付物 — Hermes Skills (已完成)
│   ├── README.md                    # 安装说明 + 架构图
│   ├── deep-research/
│   │   └── SKILL.md                 # v1.3.0 — 总指挥 (入口 skill)
│   ├── research-literature/
│   │   └── SKILL.md                 # v1.1.0 — 调研者 (含 CNKI 集成)
│   ├── research-analysis/
│   │   └── SKILL.md                 # v1.0.0 — 数据分析师
│   ├── research-writing/
│   │   └── SKILL.md                 # v1.0.0 — 撰写者
│   ├── research-review/
│   │   └── SKILL.md                 # v1.0.0 — 评审者
│   └── research-advisor/
│       └── SKILL.md                 # v1.0.0 — 学术顾问
│
├── project-docs/                    # 项目文档
│   ├── README.md                    # 项目方案 (v3.0)
│   ├── AGENTS.md                    # 本文件 (Trae 开发指引)
│   ├── appendix-a-knowledge-base.md # Gbrain 知识库使用指南
│   └── appendix-b-skill-toolchain.md# Skill 工具链说明
│
├── mcp-servers/ (可选，按需开发)     # 自定义 MCP Server
│   ├── academic-search/             # 学术搜索 API 封装
│   │   ├── server.py
│   │   └── requirements.txt
│   └── sentiment-monitor/           # 舆情/情感分析
│       ├── server.py
│       └── requirements.txt
│
└── tests/ (可选)                    # 测试
    └── test_skill_flow.md           # 手动测试用例
```

---

## 当前状态

### 已完成 ✅

| 交付物 | 说明 |
|---|---|
| 6 个 Hermes Skills | 总指挥、调研者、数据分析师、撰写者、评审者、学术顾问 |
| 双模式支持 | supervised (Human-in-the-Loop) + autonomous (全自动) |
| CNKI 集成 | 调研者工作流使用 `/cnki-paper-downloader` |
| SubAgent 约束 | 所有 Skills 包含隔离约束块，防止 SubAgent 越权 |
| Gbrain 灵活设计 | 无硬编码路径，总指挥运行时设计存储结构 |
| 打包交付 | `hermes-skills.tar.gz` 可直接安装 |

### 待做 ⬜

| 任务 | 优先级 | 说明 |
|---|---|---|
| 安装验证 | P0 | 将 Skills 安装到 Hermes，验证 delegate_task 链路 |
| 端到端测试 | P0 | 选定研究课题，走完全流程 |
| MCP Server 开发 | P1 | 按需：academic-search、sentiment-monitor |
| Skill 调优 | P2 | 根据试跑反馈优化各 Skill 的 prompt 和流程 |

---

## 开发任务清单

### P0 — 安装与验证

- [ ] **T1: 安装 Skills**
  ```bash
  tar -xzf hermes-skills.tar.gz -C ~/.hermes/skills/
  hermes skills list | grep research
  ```
  验证 6 个 skill 全部识别。

- [ ] **T2: 验证前置条件**
  - Hermes `delegate_task` 功能正常（config.yaml 中 delegation 已启用）
  - `max_spawn_depth` 建议设为 2（总指挥 → SubAgent → 内部调用）
  - Gbrain MCP 端点可访问
  - `/cnki-paper-downloader` skill 已安装且可用

- [ ] **T3: 单 Skill 测试**
  ```
  # 单独测试调研者
  /research-literature 帮我检索 2020-2025 年关于 [主题] 的最新研究

  # 单独测试分析师
  /research-analysis 对这份数据做描述性统计

  # 单独测试撰写者
  /research-writing 撰写论文引言初稿
  ```

- [ ] **T4: 全流程测试 (supervised)**
  ```
  /deep-research 我想研究 [你的课题]
  ```
  验证：
  - 总指挥正确分阶段编排
  - SubAgent 正确加载对应 Skill
  - 审批节点正确暂停
  - CNKI 下载流程正常
  - Gbrain 写入正常

- [ ] **T5: Autonomous 模式测试**
  ```
  /deep-research autonomous [课题]
  ```
  验证无暂停、全自动推进到底。

### P1 — 可选增强

- [ ] **T6: academic-search MCP Server**
  - 场景：研究需要 Semantic Scholar / OpenAlex API 辅助检索
  - 开发：封装 API → MCP 协议暴露 → Hermes 配置端点
  - 注意：当前调研者已能通过 web 工具 + CNKI 完成基本检索

- [ ] **T7: 统计分析环境**
  - 场景：数据分析师执行 Python 统计脚本
  - 需要：Hermes terminal 中安装 pandas, statsmodels, scipy 等
  - 验证：分析师能通过 terminal 执行回归、SEM 等分析

- [ ] **T8: sentiment-monitor MCP Server**
  - 场景：社科研究需要舆情/文本情感分析
  - 按需开发，非所有研究课题都需要

### P2 — 持续优化

- [ ] **T9: Skill Prompt 调优**
  - 根据试跑中 SubAgent 的实际表现调整各 SKILL.md
  - 常见问题：context 传递不充分、输出格式不规范、升级判断不准确

- [ ] **T10: 并行效率优化**
  - 调整总指挥的并行 delegate 策略（当前最多 3 路并行）
  - 评估哪些阶段可以增加并行度

---

## Skill 修改指南

### 修改已有 Skill

直接编辑对应的 `SKILL.md` 文件：

```
~/.hermes/skills/deep-research/SKILL.md       # 总指挥流程/审批规则
~/.hermes/skills/research-literature/SKILL.md  # 检索策略/数据源
~/.hermes/skills/research-analysis/SKILL.md    # 分析方法/工具
~/.hermes/skills/research-writing/SKILL.md     # 写作规范/引文格式
~/.hermes/skills/research-review/SKILL.md      # 审查标准/清单
~/.hermes/skills/research-advisor/SKILL.md     # 评审标准/评级体系
```

### SKILL.md 格式规范

```markdown
---
name: skill-name
description: 一句话描述
version: x.y.z
metadata:
  hermes:
    tags: [tag1, tag2]
    related_skills: [related-skill-1]
---

# Skill 标题

## When to Use
[触发条件]

## SubAgent 约束（当被 delegate_task 调用时）
- 你不能直接与用户交互
- 需要升级的问题标注 [需升级]
- 所有产出通过返回值交给总指挥

## 角色
[角色定义]

## 工作流程
[步骤]

## Gbrain 使用
[路径由总指挥在 delegate context 中指定]

## 输出格式
[模板]
```

### 新增 Skill

1. 创建目录：`~/.hermes/skills/new-skill-name/SKILL.md`
2. 在 `deep-research/SKILL.md` 的 delegation 模板中添加对应的 delegate_task 模板
3. 在 `deep-research/SKILL.md` 的元数据 `related_skills` 中添加新 skill 名

---

## delegate_task 机制说明

### 基本用法

```python
# 同步阻塞调用 — 总指挥等待 SubAgent 完成
delegate_task(
    goal="明确的任务目标",
    context="""
完整的上下文信息（SubAgent 看不到对话历史，必须在这里传递所有必要信息）

请加载 /skill-name skill 指导你的工作。

可用工具:
- [列出 SubAgent 可用的工具]

重要约束: 你是 SubAgent，不能直接与用户交互。遇到问题在输出中标注 [需升级]。

具体任务:
1. [步骤]
2. [步骤]

输出: [期望的输出格式]
    """,
    toolsets=["web", "file"]  # SubAgent 可用的工具集
)
```

### 并行调用 (最多 3 路)

```python
delegate_task(tasks=[
    {"goal": "...", "context": "...", "toolsets": ["web", "file"]},
    {"goal": "...", "context": "...", "toolsets": ["web", "file"]},
    {"goal": "...", "context": "...", "toolsets": ["web", "file"]},
])
```

### 关键约束

| 约束 | 说明 |
|---|---|
| SubAgent 隔离 | 看不到父 Agent 对话历史，只有 goal + context |
| 无用户交互 | SubAgent 不能发消息给用户 |
| 同步阻塞 | 总指挥等待 SubAgent 返回才继续 |
| context 完备性 | 必须在 context 中传递所有必要信息 |
| 工具声明 | toolsets 决定 SubAgent 可用的工具范围 |

---

## Gbrain 集成说明

### 访问方式

Gbrain 通过 MCP 协议暴露，Hermes 的 SubAgent 可通过 MCP 工具调用 Gbrain 能力。

### 常用操作

| 操作 | MCP Tool | 说明 |
|---|---|---|
| 写入知识页面 | brain-ops write | 文献笔记、分析报告等 |
| 读取知识页面 | brain-ops read | 获取已有知识 |
| 搜索知识 | hybrid-search | 语义 + 关键词混合搜索 |
| 自动实体抽取 | signal-detector | 写入时自动触发 |
| 知识库维护 | maintain | 断链修复、重复合并 |

### 路径设计原则

- **不在 Skills 中硬编码路径**
- 总指挥启动研究时根据课题设计路径命名空间
- 多课题通过不同前缀隔离：`课题A/literature/...`、`课题B/literature/...`

---

## 常见问题

### Q: delegate_task 调用失败？
检查 Hermes config.yaml 中 delegation 是否启用，`max_spawn_depth` 是否足够。

### Q: CNKI 下载失败率高？
确认 `/cnki-paper-downloader` skill 正常工作。标题必须完整准确（含副标题）。

### Q: SubAgent 输出质量不稳定？
增强 delegate_task 的 context 内容，确保传递了充分的背景信息和明确的输出格式要求。

### Q: Gbrain 写入后找不到？
检查写入路径是否正确。使用 hybrid-search 搜索确认。检查总指挥传递给 SubAgent 的路径前缀是否一致。

### Q: autonomous 模式方向偏离？
在 `deep-research/SKILL.md` 中调整学术顾问的调用频率，增加自动审查节点。

---

## 关键文件路径

| 文件 | 用途 | 修改频率 |
|---|---|---|
| `hermes-skills/deep-research/SKILL.md` | 总指挥流程编排（最核心） | 高 |
| `hermes-skills/research-literature/SKILL.md` | 文献检索流程 | 中 |
| `hermes-skills/research-analysis/SKILL.md` | 分析方法定义 | 低 |
| `hermes-skills/research-writing/SKILL.md` | 写作规范 | 低 |
| `hermes-skills/research-review/SKILL.md` | 审查标准 | 低 |
| `hermes-skills/research-advisor/SKILL.md` | 评审标准 | 低 |
| `project-docs/README.md` | 整体方案文档 | 随大版本更新 |
