# Changelog

本项目所有显著的 Skill 变更记录于此。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

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
