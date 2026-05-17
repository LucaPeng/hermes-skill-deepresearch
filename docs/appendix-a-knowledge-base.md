# 附录 A：Gbrain 知识库使用指南

> 基于 Gbrain 的 Brain Pages + Knowledge Graph + Hybrid Search 构建研究知识库

---

## 一、Gbrain 在本项目中的定位

| 维度 | 说明 |
|---|---|
| 部署方式 | 共享实例 — 非本项目独占 |
| 访问协议 | MCP Server (30+ tools) |
| 路径管理 | **不硬编码** — 总指挥 (deep-research) 运行时设计 |
| 核心能力 | Brain Pages 存储 + Knowledge Graph 关联 + Hybrid Search 检索 |

---

## 二、核心能力

### 2.1 Brain Pages (知识页面)

结构化 Markdown 页面，支持 CRUD 操作。每篇文献笔记、分析报告、论文草稿都以 Brain Page 存储。

**写入时自动触发 signal-detector**：抽取实体（论文、理论、方法、变量等）和关系（引用、支持、反驳），自动扩充知识图谱。

### 2.2 Knowledge Graph (知识图谱)

自动构建的实体关系网络：

| 实体类型 | 说明 | 来源 |
|---|---|---|
| Paper | 学术论文 | 文献笔记写入时抽取 |
| Theory | 理论/框架 | 文献综述中抽取 |
| Method | 研究方法 | 方法论描述中抽取 |
| Variable | 研究变量 | 假设/模型中抽取 |
| Author | 学者 | 引用信息中抽取 |

| 关系类型 | 说明 |
|---|---|
| CITES | A 引用 B |
| SUPPORTS | 证据支持假设/理论 |
| CONTRADICTS | 证据反驳假设/理论 |
| EXTENDS | 对已有理论的扩展 |
| USES_METHOD | 论文使用某方法 |

### 2.3 Hybrid Search (混合搜索)

| 搜索模式 | 适用场景 | 示例 |
|---|---|---|
| 语义搜索 | 模糊概念检索 | "社交媒体对信任的影响" |
| 关键词搜索 | 精确术语定位 | "SEM" "中介效应" "Cronbach's α" |
| 图谱遍历 | 关联发现 | 从某论文出发，找引用链 |
| 混合模式 | 综合查询 | 语义 + 关键词 + 图谱上下文 |

---

## 三、路径设计策略

### 3.1 原则

Skills 中**不硬编码**任何 Gbrain 路径。总指挥在启动研究项目时：

1. 根据课题确定命名空间前缀
2. 设计子路径结构
3. 在 delegate_task 的 context 中传递给 SubAgent

### 3.2 建议结构（供总指挥参考）

```
{课题前缀}/
├── meta/                 # 项目元信息
│   ├── project-info      # 研究目标、时间线
│   └── decision-log      # 决策日志
│
├── literature/           # 文献知识
│   ├── note-{编号}       # 单篇文献笔记
│   ├── review            # 文献综述
│   └── framework         # 理论框架
│
├── data/                 # 数据分析
│   ├── plan              # 采集方案
│   └── report-{编号}     # 分析报告
│
└── writing/              # 写作产出
    ├── outline           # 论文大纲
    ├── draft-{章节}      # 各章节草稿
    └── feedback-{轮次}   # 审查反馈
```

### 3.3 多课题隔离

```
trust-research/literature/note-001
trust-research/data/report-001

user-behavior/literature/note-001
user-behavior/data/report-001
```

不同课题通过前缀天然隔离，互不影响。

---

## 四、Agent 使用 Gbrain 的方式

### 4.1 写入流程

```
SubAgent 完成工作
    │
    ▼
通过 brain-ops write 写入 Brain Page
    │
    ▼
signal-detector 自动触发
    • 抽取实体 (Paper, Theory, Method...)
    • 识别关系 (CITES, SUPPORTS...)
    • 建立 typed links
    │
    ▼
Knowledge Graph 自动增长
```

### 4.2 查询流程

```
SubAgent 需要信息
    │
    ▼
通过 hybrid-search 检索
    • 返回相关 Brain Pages
    • 附带 Graph Context (关联实体)
    │
    ▼
SubAgent 基于检索结果工作
```

### 4.3 各角色的典型 Gbrain 操作

| 角色 | 写入 | 读取 | 搜索 |
|---|---|---|---|
| 总指挥 | meta/, decision-log | 全部 | 进度查询 |
| 调研者 | literature/ | literature/ | 文献关联发现 |
| 数据分析师 | data/ | data/, literature/ | 变量和方法查询 |
| 撰写者 | writing/ | literature/, data/ | 素材检索 |
| 评审者 | — (只读+批注) | writing/, literature/ | 引用验证 |
| 学术顾问 | — (只读) | 全部 | 理论框架验证 |

---

## 五、自动知识图谱的价值

### 5.1 对调研者

- 写入文献笔记后，自动与已有论文建立引用网络
- 搜索时获得"相关论文"推荐（基于图谱邻居）
- 发现隐含的理论关联（A→B→C 间接关系）

### 5.2 对撰写者

- 写作时 hybrid-search 自动推荐可引用的文献
- 基于图谱的 concept-synthesis 辅助理论整合
- 引用链完整性自动检查

### 5.3 对评审者

- 验证引用是否存在于知识库中
- 检查论文间的逻辑一致性（是否存在矛盾引用）
- 发现遗漏的重要关联论文

---

## 六、维护策略

| 维护任务 | 触发方式 | 操作 |
|---|---|---|
| 断链修复 | 总指挥定期调用 maintain | 修复引用了不存在页面的链接 |
| 重复实体合并 | 发现重复时调用 | 合并指向同一论文/作者的实体 |
| 孤立实体清理 | 研究结束时 | 清理无关联的实体节点 |
| 路径重组 | 研究方向变更时 | 总指挥重新组织命名空间 |

---

## 七、跨项目复用

Gbrain Knowledge Graph 天然支持跨项目复用：

- **Method 实体**：同一统计方法在不同项目中共享
- **Paper 实体**：同一论文被多个项目引用时自动关联
- **Theory 实体**：理论框架的关系网络持续积累
- **Author 实体**：追踪学者的研究脉络

新研究项目启动时，总指挥可先搜索已有知识图谱，发现与新课题相关的已有知识，避免重复工作。
