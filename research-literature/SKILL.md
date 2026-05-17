---
name: research-literature
description: 学术文献检索与综述 — 系统性检索、文献笔记、理论分析、研究缺口识别
version: 1.6.0
metadata:
  hermes:
    tags: [research, literature, academic, survey, citation, cnki]
    related_skills: [deep-research, cnki-paper-downloader]
---

# Research Literature — 学术文献检索与综述

## When to Use
当你被 delegate 执行文献相关任务时加载此 skill：文献检索、文献综述撰写、理论框架梳理、研究缺口识别。

## SubAgent 约束
通用 SubAgent 行为约束（不能与用户交互、`[需升级]` 标注、产出文件清单回报、修订模式必须含基线 + F-ID）→ 见 [STATE_PROTOCOL.md](../deep-research/STATE_PROTOCOL.md#subagent-通用约束在每次-delegate-context-中复述)。
若独立被用户调用（非 delegate），则可直接对话，无此约束。

## 角色
你是一名学术调研专家，擅长社科/管理学领域的文献检索和综述。

---

## 数据源与工具

### 中文全文链路
- **CNKI**（主）：通过 `/cnki-paper-downloader` 逐篇下载
- 适用：中文社科 / 管理类核心期刊与博硕论文

### 英文元数据 / 图谱链路（通过 terminal 调免费 API）

> 用途定位：S2 与 OpenAlex 主要用于**论文发现 + 摘要 + 引文图谱**。
> 它们**不一定提供全文 PDF**，仅当返回字段中含 OA 链接时才能下载全文，否则只能拿摘要。

#### Semantic Scholar (S2)
- 端点: `https://api.semanticscholar.org/graph/v1/paper/search`
- 限频: 100 req / 5min（免费，无需 key）；申请 key 可放宽
- 推荐字段: `title,abstract,authors,year,venue,citationCount,openAccessPdf,externalIds,referenceCount`
- 示例:
  ```bash
  curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query=social+commerce+trust&limit=20&fields=title,abstract,authors,year,citationCount,openAccessPdf,externalIds"
  ```
- 引文/被引图谱:
  ```bash
  # 某论文的参考文献
  curl -s "https://api.semanticscholar.org/graph/v1/paper/{paperId}/references?fields=title,year,authors,abstract"
  # 某论文的被引列表
  curl -s "https://api.semanticscholar.org/graph/v1/paper/{paperId}/citations?fields=title,year,authors,abstract"
  ```

#### OpenAlex
- 端点: `https://api.openalex.org/works`
- 限频: 完全免费，10 req/s（建议在 User-Agent 加 `mailto:` 提速）
- 优势: 主题分类（concepts）、作者作品集、机构归属、OA 状态标签
- 示例:
  ```bash
  curl -s "https://api.openalex.org/works?search=social+commerce+trust&per-page=25" \
       -H "User-Agent: research-literature (mailto:your@email)"
  ```
- 引文/被引:
  ```bash
  # 某论文的参考文献 (referenced_works)
  curl -s "https://api.openalex.org/works/{work_id}" -H "User-Agent: ..."
  # 被引论文（cites this work）
  curl -s "https://api.openalex.org/works?filter=cites:{work_id}" -H "User-Agent: ..."
  ```

#### OA PDF 直链下载
- S2 返回的 `openAccessPdf.url` 或 OpenAlex 的 `open_access.oa_url` 非空时，可用 `wget`/`curl` 下载到本地，作为**全文笔记**来源
- 否则该论文只能作为**摘要笔记**或**元数据节点**

#### 限频与失败处理
- S2 限频时返回 429 → SubAgent 应退避（sleep 5s）后重试，最多 3 次
- 网络失败 → 跳过该论文，记录到获取状态表，不阻塞流程

### 论文下载工具（中文）: `/cnki-paper-downloader`
- **功能**: 从 CNKI 下载一篇具体论文的全文
- **输入**: 论文的**完整标题**（必须准确）
- **限制**: 每次只能下载一篇，需逐篇调用
- **使用方式**: 加载 `/cnki-paper-downloader` skill，提供论文标题

### 辅助检索（仅作发现，不一定能获取全文）
- Google Scholar — 用于引用关系交叉验证
- Web 搜索 — 用于补充信息、确认论文元数据

### 本地手动下载兜底通道（Tier 3）

当 CNKI 与 OA 全文均不可得，但论文对核心论据至关重要时，进入"人工下载兜底"流程：让用户手动下载并放入 `~/Downloads/essays/`。

完整规范（约定文件夹、三层兜底链、触发条件、Step A-D 操作流程、模式差异、与续跑/修订的协同、不要做的事）→ **见 [MANUAL_DOWNLOAD.md](./MANUAL_DOWNLOAD.md)**。

### 三类笔记体系（来源等级）

| 来源等级 | 来源 | 用途 |
|---|---|---|
| **全文** | CNKI 下载 / OA PDF / 用户手动下载 | 可作核心论据，必须含 ≥2-3 段原文 quote |
| **摘要** | S2 / OpenAlex / CNKI 仅摘要 | 背景引用，不得支撑具体数据/方法细节 |
| **元数据** | S2 / OpenAlex（仅引文图谱节点） | 仅作研究地形测绘，不得正文引用 |

---

## 工作流程

### Step 1: 搜索策略设计
```
核心概念拆解:
- 概念A: [主题词] / [近义词] / [英文对应词]
- 概念B: [主题词] / [近义词] / [英文对应词]

CNKI 检索式: (概念A) AND (概念B)
时间范围: 近 5-10 年为主
来源类别: 核心期刊 / CSSCI / 博硕论文（按需）
```

### Step 2: 文献发现（建立论文清单）

**目标：** 建立一份待处理的论文清单，并标注预期来源等级（全文 / 摘要 / 元数据）。

方法：
1. **CNKI 通道**（中文）：通过 web 工具搜索 `site:cnki.net [关键词]` 或直接搜索 + "知网"
2. **S2 / OpenAlex 通道**（英文 + 跨语种）：用 terminal 调 API
   ```bash
   # 主题搜索 (S2)
   curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query={关键词}&limit=25&fields=title,abstract,authors,year,citationCount,openAccessPdf,externalIds"
   # 主题搜索 (OpenAlex，含 concept 分类)
   curl -s "https://api.openalex.org/works?search={关键词}&per-page=25" -H "User-Agent: research (mailto:x@y)"
   ```
3. **引文图谱滚雪球**：拿到种子论文（综述 / 高引）后，用 S2 的 `/references` 与 `/citations` 端点批量发现关联论文
4. **Google Scholar 辅助**：补查作者作品集、识别中英文同主题对应论文

每篇论文做归类决定：
- 有 OA PDF → 进入"全文"候选 → 下载并精读
- 仅有摘要 → 进入"摘要"候选
- 仅在引文图谱中出现 → 进入"元数据"候选（不写笔记内容，仅记节点）

产出：一份**论文清单**（15–30 篇），含标题、来源、预期等级、优先级。

### Step 3: 论文获取（按等级分别处理）

#### 中文全文（CNKI）
```
/cnki-paper-downloader [论文完整标题]
```
- 标题必须准确完整（含副标题）
- 每次一篇，逐篇下载
- 下载失败 → 标记为 `[未获取]` 或降级为"摘要"等级（如 web 能拿到摘要）；若仍是核心论据所需，进入 Tier 3

#### 英文/OA 全文（S2 / OpenAlex）
对返回 OA PDF 链接的英文论文：
```bash
# 直接下载 OA PDF
wget -O paper-{shortid}.pdf "{openAccessPdf.url}"
# 或
curl -L -o paper-{shortid}.pdf "{open_access.oa_url}"
```
下载成功 → 全文笔记；失败 → 降级为摘要笔记或进入 Tier 3。

#### 摘要笔记
S2 / OpenAlex 返回的 `abstract` 字段可直接摘录，**不允许扩写或推测原文未提及的内容**。

#### 元数据节点
仅记录论文的 title / authors / year / 主题概念 / 引文位置，不写内容。

#### Tier 3 人工下载
触发条件、操作流程、模式差异 → 见 [MANUAL_DOWNLOAD.md](./MANUAL_DOWNLOAD.md)。

优先级建议：高引用 > 综述/核心 > 近年发表 > 与研究直接相关。

### Step 4: 精读与笔记

对每篇已下载的论文，写结构化笔记：

```markdown
# [论文标题]

**基本信息**
- 作者: [First Author et al., Year]
- 期刊: [Journal Name]
- 来源: CNKI / Semantic Scholar / OpenAlex / 用户手动下载 / [其他]
- 引用数: [N]（如可查）
- **来源等级**: 全文 / 摘要 / 元数据   ← 撰写者引用时要据此判断可否引用
- **外部 ID**:
  - DOI: [10.xxxx/yyyy 或 N/A]
  - S2 paperId: [xxxxxxxx 或 N/A]
  - OpenAlex Work ID: [Wxxxxxxxxx 或 N/A]
  - CNKI ID: [xxxxxx 或 N/A]
- **OA 链接 / 本地路径**: [openAccessPdf.url 或 oa_url 或 ~/Downloads/essays/xxx.pdf 或 N/A]

**核心观点**
[2-3 句概括核心论点和发现]

**研究方法**
- 设计: [实验/调查/案例/...]
- 样本: [N=?, 来源]
- 分析: [回归/SEM/质性编码/...]

**关键发现**
1. [发现1]
2. [发现2]
3. [发现3]

**与本研究关联**
- 支持: [如何支持我们的研究]
- 挑战/补充: [提出了什么不同视角]
- 可引用点: [具体可引用的内容]

**引用价值**: ⭐⭐⭐⭐⭐ (1-5星)
**获取状态**: ✅ 已下载全文 / ⚠️ 仅摘要 / ❌ 未获取
```

### Step 5: 文献综述整合

```markdown
# 文献综述: [研究主题]

## 1. 检索概况
- 主要数据源: 中文全文(CNKI) / 英文元数据图谱(S2, OpenAlex)
- 辅助来源: Google Scholar, Web
- 检索策略: [关键词和组合]
- 纳入标准: [筛选条件]
- 最终纳入: [N] 篇（全文 [a] / 摘要 [b] / 元数据 [c]）

## 2. 理论基础
### 2.1 [理论A]
[起源 → 核心观点 → 发展脉络 → 在本领域的应用]
- 代表文献: Author (Year), Author (Year)...
- 核心主张: ...
- 局限性: ...

### 2.2 [理论B]
[同上结构]

## 3. 研究现状
### 3.1 [维度1] / 3.2 [维度2]
[已有发现的共识和争议]

## 4. 研究缺口
1. **[缺口1]**: [描述] — 重要性: [为什么值得填补]
2. **[缺口2]**: ...
3. **[缺口3]**: ...

## 5. 本研究定位
[基于以上缺口，本研究的切入点和预期贡献]

## 参考文献
[GB/T 7714 或 APA 7 格式，按字母/拼音排序]

## 附录: 文献获取情况
| 序号 | 标题 | 来源 | 等级 | 状态 | 备注 |
|---|---|---|---|---|---|
| 1 | [标题] | CNKI | 全文 | ✅ | |
| 2 | [标题] | S2 | 摘要 | ✅ | 无 OA PDF |
| 3 | [标题] | OpenAlex | 元数据 | ✅ | 仅作图谱节点 |
| ... | | | | | |
```

---

## Gbrain 使用
- 文献笔记与综述写入 Gbrain（路径由总指挥在 delegate context 中指定）
- Gbrain signal-detector 会自动抽取实体和关系（论文 → 理论 → 方法）

## 质量标准
- 已下载全文必须精读（至少方法和结论），不能只看摘要
- 仅有摘要的论文可作补充引用，但不作核心论据
- 引用必须准确：作者名、年份、期刊
- 正反观点均需覆盖
- 标注不确定处 `[待确认]`
- 标注每篇文献的获取状态（全文/仅摘要/未获取）
- **全文笔记必须含 ≥2-3 段原文 quote**（带页码或章节定位），便于评审者比对撰写者的转述准确性
- **摘要笔记不得伪造 quote**，仅可摘录摘要中的原文片段

## 效率建议
- 先批量确定标题清单（20-30 篇）再集中下载，避免边找边下
- 优先下载综述论文 — 一篇综述可提供 30-50 个参考文献标题
- 英文文献：优先用 S2 / OpenAlex 拿摘要 + 引文图谱；只有当返回 OA PDF 时才下载全文
- 引文滚雪球：拿到种子论文 paperId / Work ID 后批量调 references / citations 端点
- 限频应对：S2 限频时 sleep 5s 重试；OpenAlex 在 User-Agent 带 `mailto:` 可获更稳定的 polite pool

## 升级条件（回报给总指挥）
- 研究方向可能需调整（领域饱和/无新发现空间）
- 大量核心文献无法从 CNKI 获取**且 S2/OpenAlex 也无 OA 全文**（仅摘要不够支撑核心论据）
- 跨学科内容超出理解能力
- 检索结果严重不足
- S2 / OpenAlex API 持续限频或不可达，影响发现进度

## 修订模式约束（当 delegate context 含「反馈 ID」或 revision-plan 时）

### 输入识别
context 包含 `F-ID`（如 F1 / F2）或 `revision-plan` 路径时，视为修订模式。

### 改动原则
- 检索范围限定为反馈 ID 直接相关的方向（如 F1 要求"补 SOR 模型论述" → 仅检索 SOR 相关文献）
- 不顺手扩大检索范围，不替换基线已纳入的文献
- 新笔记必须在元信息中标注触发反馈 ID

### 笔记追加字段（修订模式下必填）
```markdown
**触发反馈**: F1 (导师反馈 R1: 理论框架对 SOR 模型的论述不够)
**修订归属**: revisions/R1-2026-05-20/
```

### 输出要求
- 末尾「产出文件清单」段标明哪些笔记是新增（new）、哪些是补充（augment 已有笔记）
- 不要删除/覆盖基线已纳入的笔记，如确需更正信息，新建一个补充笔记并在原笔记顶部加 `[已被 R{n} 补充更新，见 ...]`
