# 附录 B：Skill 与工具链说明

> 基于 Hermes Skills + Gbrain MCP + CNKI Downloader 的工具链

---

## 一、工具层架构

```
┌───────────────────────────────────────────────────────────────┐
│  用户                                                          │
│  "/deep-research [课题]" 或 "/deep-research autonomous [课题]" │
└───────────────────────────────┬───────────────────────────────┘
                                │
                                ▼
┌───────────────────────────────────────────────────────────────┐
│  Hermes Skills 层                                             │
│                                                               │
│  /deep-research (总指挥, 入口)                                 │
│       │                                                       │
│       ├── delegate → /research-literature (调研者)             │
│       │                  └── 调用 /cnki-paper-downloader       │
│       ├── delegate → /research-analysis (数据分析师)           │
│       ├── delegate → /research-writing (撰写者)               │
│       ├── delegate → /research-review (评审者)                │
│       └── delegate → /research-advisor (学术顾问, on-demand)  │
│                                                               │
└──────────────┬────────────────────────────┬───────────────────┘
               │                            │
               ▼                            ▼
┌──────────────────────────┐  ┌─────────────────────────────────┐
│  Hermes 内置工具          │  │  Gbrain (MCP)                   │
│                          │  │                                 │
│  • web (搜索/浏览)       │  │  • brain-ops (知识页面读写)      │
│  • file (文件读写)       │  │  • hybrid-search (混合搜索)     │
│  • terminal (脚本执行)   │  │  • signal-detector (实体抽取)   │
│                          │  │  • maintain (知识库维护)        │
└──────────────────────────┘  │  • 30+ 其他 MCP tools          │
                              └─────────────────────────────────┘
```

---

## 二、Skill 清单

### 2.1 研究系统 Skills (本项目)

| Skill | 版本 | 角色 | 核心能力 |
|---|---|---|---|
| `deep-research` | v1.3.0 | 总指挥 (PI) | 任务编排、delegate_task、审批门控、模式切换 |
| `research-literature` | v1.1.0 | 调研者 | 文献检索、CNKI 下载、综述撰写、研究缺口识别 |
| `research-analysis` | v1.0.0 | 数据分析师 | 统计分析 pipeline、假设检验、可视化 |
| `research-writing` | v1.0.0 | 撰写者 | 论文撰写、结构组织、APA 7 引文管理 |
| `research-review` | v1.0.0 | 评审者 | 逻辑/引用/方法/数据/语言/格式审查 |
| `research-advisor` | v1.0.0 | 学术顾问 | 里程碑评审、评级 (A/B/C/D) |

### 2.2 依赖 Skills

| Skill | 来源 | 用途 |
|---|---|---|
| `cnki-paper-downloader` | 已安装 (用户自建) | 从 CNKI 下载论文全文，输入完整标题 |

---

## 三、各 Skill 的工具依赖

| Skill | Hermes 工具 | Gbrain MCP | 其他 Skill |
|---|---|---|---|
| deep-research | web, file | brain-ops, hybrid-search, maintain | 所有子 Skill |
| research-literature | web, file | brain-ops, hybrid-search | cnki-paper-downloader |
| research-analysis | file, terminal | brain-ops, hybrid-search | — |
| research-writing | file | brain-ops, hybrid-search | — |
| research-review | web, file | brain-ops, hybrid-search | — |
| research-advisor | file | brain-ops, hybrid-search | — |

---

## 四、delegate_task 中的工具传递

总指挥在 delegate_task 时通过 `toolsets` 参数控制 SubAgent 可用的工具：

| 被委派角色 | toolsets | 说明 |
|---|---|---|
| 调研者 | `["web", "file"]` | 需要 web 搜索论文 + file 保存笔记 |
| 数据分析师 | `["file", "terminal"]` | 需要 terminal 执行统计脚本 |
| 撰写者 | `["file"]` | 主要是文件读写 |
| 评审者 | `["web", "file"]` | 可能需要 web 验证引用 |
| 学术顾问 | `["file"]` | 主要是阅读和输出 |

---

## 五、CNKI 集成细节

### 5.1 `/cnki-paper-downloader` 使用约束

| 约束 | 说明 |
|---|---|
| 输入 | 论文的**完整标题**（必须准确，含副标题） |
| 频率 | 每次只能下载一篇 |
| 数据源 | 仅 CNKI |
| 失败处理 | 标题不匹配或未收录时返回失败 |

### 5.2 调研者的 CNKI 工作流

```
1. 搜索策略设计 (关键词 + 检索式)
        │
        ▼
2. 论文标题发现 (web 搜索 / Google Scholar)
   • site:cnki.net [关键词]
   • 综述论文参考文献滚雪球
        │
        ▼
3. 建立待下载清单 (15-30 篇，按优先级排序)
        │
        ▼
4. 逐篇下载 (/cnki-paper-downloader)
   • 高引用优先
   • 核心期刊优先
   • 近年发表优先
        │
        ▼
5. 精读 + 结构化笔记 → 写入 Gbrain
        │
        ▼
6. 综述整合 (含获取状态表)
```

### 5.3 获取状态追踪

每次文献检索任务的输出必须包含获取状态表：

```markdown
| 序号 | 标题 | 状态 | 备注 |
|---|---|---|---|
| 1 | 社交电商中信任传递... | ✅ 全文 | CNKI 下载 |
| 2 | Trust in Social Commerce... | ⚠️ 仅摘要 | 英文文献，CNKI 未收录 |
| 3 | 消费者感知风险与... | ❌ 未获取 | 标题可能不准确 |
```

---

## 六、可选 MCP Server 扩展

以下 MCP Server 为可选增强，按需开发：

### 6.1 academic-search

| 属性 | 说明 |
|---|---|
| 用途 | 封装 Semantic Scholar / OpenAlex API |
| 能力 | 结构化论文搜索、引文网络获取、参考文献导出 |
| 优先级 | P1 — 当 web 搜索 + CNKI 不够用时开发 |
| 调用者 | 调研者 |

### 6.2 sentiment-monitor

| 属性 | 说明 |
|---|---|
| 用途 | 文本情感分析、主题发现、趋势分析 |
| 能力 | 情感极性判断、aspect-based 观点抽取、时序趋势 |
| 优先级 | P2 — 仅当研究课题涉及舆情分析时需要 |
| 调用者 | 数据分析师 |

---

## 七、Skill 演进路线

```
v1.x (当前): 基础能力
  ✅ 6 Skill 协同框架
  ✅ CNKI 集成
  ✅ 双模式运行
  ✅ SubAgent 约束体系

v2.x (未来): 增强能力
  ○ 更多数据源 (WOS, Scopus — 需获取权限)
  ○ 统计分析模板库 (回归/SEM/中介效应)
  ○ 多语言支持优化 (中英混合文献)
  ○ 学习循环 (基于历史表现自动优化 Skill)

v3.x (远期): 高级特性
  ○ 跨项目知识迁移
  ○ 自动定时文献监控 (Hermes cron)
  ○ 论文投稿适配 (不同期刊格式)
```

---

## 八、Troubleshooting

| 问题 | 可能原因 | 解决方案 |
|---|---|---|
| SubAgent 不加载 Skill | context 中 skill 路径错误 | 检查 `/skill-name` 格式 |
| CNKI 下载失败 | 标题不准确 | 用 web 确认准确标题后重试 |
| Gbrain 写入无反应 | MCP 端点不可达 | 检查 Gbrain 服务状态 |
| 并行 delegate 报错 | 超过 3 路并行 | 减少并行数或改为串行 |
| SubAgent 试图与用户对话 | 约束块未生效 | 检查 context 中是否包含 SubAgent 约束文本 |
| autonomous 模式意外暂停 | SKILL.md 中残留审批逻辑 | 检查 deep-research 的模式判断逻辑 |
