# Deep Research Skills — 安装说明

## 这是什么

一组 Hermes Skills，实现**多 Agent 学术研究协同**。

你无需修改 SOUL.md 或 config.yaml。只需安装这些 skills，在需要时对 Hermes 说：

```
/deep-research 帮我研究 [你的课题]
```

Hermes 会自动作为总指挥，通过 `delegate_task` 生成多个 SubAgent 协同完成研究。

## 安装

```bash
# 解压到 ~/.hermes/skills/
tar -xzf hermes-research-skills.tar.gz -C ~/.hermes/skills/
```

或手动复制：
```bash
cp -r deep-research/ ~/.hermes/skills/deep-research/
cp -r research-literature/ ~/.hermes/skills/research-literature/
cp -r research-analysis/ ~/.hermes/skills/research-analysis/
cp -r research-writing/ ~/.hermes/skills/research-writing/
cp -r research-review/ ~/.hermes/skills/research-review/
cp -r research-advisor/ ~/.hermes/skills/research-advisor/
```

验证安装：
```bash
hermes skills list | grep research
```

应看到 6 个 skill：
```
deep-research         多Agent学术研究 — 启动完整研究团队...
research-literature   学术文献检索与综述...
research-analysis     研究数据分析...
research-writing      学术论文撰写...
research-review       学术质量审查...
research-advisor      学术顾问...
```

## 前提条件

1. **Hermes Agent** — 已部署运行
2. **Gbrain MCP** — 已部署，Hermes 可通过 MCP 调用 Gbrain 工具
3. **delegate_task** — config.yaml 中 delegation 已启用（建议 `max_spawn_depth: 2`）

## Skill 协同架构

```
你: "/deep-research 帮我研究社交电商中的消费者信任"
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ deep-research (入口 skill = 总指挥 PI)                        │
│                                                             │
│ ┌─ 理解研究需求                                              │
│ ├─ 按 SOP 分解任务                                           │
│ ├─ delegate_task → SubAgent (加载对应 skill)                 │
│ ├─ 汇总 SubAgent 结果                                       │
│ ├─ ⏸️ 关键节点暂停 → 请求你审批                               │
│ ├─ 🔧 系统开发需求 → 输出需求文档给你                          │
│ └─ 推进下一阶段                                              │
└──────────┬──────────┬──────────┬──────────┬─────────────────┘
           │          │          │          │
      delegate   delegate   delegate   delegate
           │          │          │          │
           ▼          ▼          ▼          ▼
     ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
     │/research-││/research-││/research-││/research-│
     │literature││analysis  ││writing   ││review    │
     └──────────┘└──────────┘└──────────┘└──────────┘
           │          │          │          │
           └──────────┴──────────┴──────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Gbrain (MCP)   │
                 │ Brain Pages +   │
                 │ Knowledge Graph │
                 └─────────────────┘
```

## 关键设计点

### Human-in-the-Loop
总指挥在每个关键节点会**暂停并请求你确认**：
- 研究问题 / 理论框架确认
- 研究设计定稿
- 数据采集方案批准
- 重大分析结论确认
- 论文终稿审定

你回复 "继续" / "可以" 推进，或提出修改意见。

### 系统开发需求
当研究需要爬虫、监测系统等技术构建时，总指挥不会自行开发，而是输出结构化的需求文档给你，由你决定如何实现。

### SubAgent 隔离
每个 SubAgent：
- 只能看到 delegate_task 传入的 context
- 不能直接与你对话
- 遇到问题通过返回结果标注 `[需升级]`
- 由总指挥决定是否升级到你

### 多课题支持
Skills 不绑定具体研究课题。每次启动时告诉总指挥课题是什么即可。多个研究可以通过 Gbrain 不同的 brain page 路径隔离知识。

## 使用方式

### 两种运行模式

| 模式 | 命令 | 行为 |
|---|---|---|
| supervised (默认) | `/deep-research [课题]` | 关键节点暂停等你审批 |
| autonomous | `/deep-research autonomous [课题]` | 全自动推进，完成后汇报 |

### 启动新研究
```
/deep-research 我想研究社交电商环境下消费者信任的形成机制
```

### 继续推进
```
/deep-research 文献综述部分可以了，进入数据分析
```

### 单独调用某个角色（跳过总指挥编排）
```
/research-literature 帮我检索 2020-2025 年 TAM 的最新研究
/research-analysis 对 data/survey.csv 做描述性统计
/research-writing 撰写引言第一段
/research-review 审查这段文字的引文准确性
```

### 审批响应
```
可以，继续推进
```
或
```
不行，需要增加对 xxx 理论的讨论再重新整理
```

## Gbrain 知识库

Gbrain 的存储结构**不由 Skills 硬性规定**。总指挥（deep-research skill）在启动研究项目时会自行设计命名空间和页面路径，确保不与 Gbrain 中其他项目冲突。

建议的分区思路（总指挥会参考）：
- 项目管理类（元信息、决策日志、进度）
- 文献知识类（笔记、综述、理论框架）
- 数据分析类（采集方案、分析报告）
- 写作产出类（草稿、审查反馈）
## 自定义

| 需要调整的内容 | 修改哪个文件 |
|---|---|
| 引文格式 (默认 APA 7) | `research-writing/SKILL.md` |
| 检索数据库范围 | `research-literature/SKILL.md` |
| 统计分析方法 | `research-analysis/SKILL.md` |
| 审查标准和清单 | `research-review/SKILL.md` |
| 流程节点和审批规则 | `deep-research/SKILL.md` |
