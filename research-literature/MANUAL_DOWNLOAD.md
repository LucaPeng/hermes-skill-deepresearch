# Manual Download — 本地手动下载兜底通道（Tier 3）

> 本文件是 `research-literature` 的 Tier 3 兜底规范。当一篇论文**既无法通过 CNKI 自动下载，也没有 OA PDF**，但其内容对核心论据至关重要时，进入本流程让用户手动下载并放入约定文件夹。

---

## 约定文件夹

```
~/Downloads/essays/
```
- macOS 默认 Downloads 目录下的 `essays/` 子文件夹
- 调研者每次 delegate 启动时，**第一件事**是 `mkdir -p ~/Downloads/essays/` 确保存在
- 用户手动下载的 PDF 一律放在此目录，命名按调研者给出的"建议文件名"

---

## 三层兜底链

```
Tier 1 (自动): CNKI 全文 / OA PDF 直链下载
        ↓ 失败
Tier 2 (自动): 摘要笔记（S2/OpenAlex 摘要）
        ↓ 仍无法支撑核心论据
Tier 3 (人工): 列入「人工下载清单」→ 用户手动下载 → 调研者从该文件夹读取
```

---

## Tier 3 触发条件（同时满足）

1. Tier 1 自动获取失败（CNKI 下载失败 或 无 OA PDF）
2. 该论文对核心论据有实质支撑作用（不是泛泛背景引用）
3. 仅靠摘要无法满足「来源等级匹配论断」要求（撰写者会需要全文等级才能引用）

不满足以上 3 条 → 不要扔进 Tier 3，直接降级为摘要等级即可。

---

## 操作流程

### Step A: 启动时扫描已存在文件
每次被 delegate 调起时：
```bash
mkdir -p ~/Downloads/essays/
ls ~/Downloads/essays/
```
将已存在的 PDF 文件名记录下来，后续遇到清单中已落盘的论文直接读取，不再重复列入清单。

### Step B: 生成「人工下载清单」
对所有触发 Tier 3 条件的论文，输出统一格式的清单：

```markdown
## 人工下载清单 (manual-download-needed)

需要你手动下载下列论文 PDF：

| # | 标题 | 作者 | 年份 | 期刊 | DOI | 建议文件名 | 失败原因 | 优先级 |
|---|---|---|---|---|---|---|---|---|
| 1 | A Theory of Consumer Trust | Smith, J. | 2023 | AMJ | 10.xxx/yyy | smith-2023-amj-trust.pdf | CNKI 无收录 + 无 OA | 🔴 核心 |
| 2 | Social Commerce Review | Lee, K. | 2022 | JCR | 10.xxx/zzz | lee-2022-jcr-sc.pdf | OA 链接失效 | 🟡 重要 |
| 3 | ... | | | | | ... | ... | 🟢 备用 |

### 操作说明
1. 请手动下载上述论文（机构图书馆 / Sci-Hub / Google Scholar / 联系作者均可）
2. 按"建议文件名"重命名（小写 + 短横线 + 包含作者-年份-期刊关键词）
3. 存放到 `~/Downloads/essays/`
4. 完成后告知"已上传完毕"，或直接重新启动 /deep-research 续跑

### 优先级说明
- 🔴 核心: 缺此论文无法支撑某个核心论断，请优先获取
- 🟡 重要: 有助于增强综述深度
- 🟢 备用: 可有可无，时间允许再下

### 暂不处理也可以
若某篇论文确实无法获取，告知"放弃 #N"，调研者将降级为"摘要"等级处理（不能用于支撑核心论据）。
```

### Step C: 用户上传后的读取
当用户告知"已上传完毕"或下一次 delegate 启动时：
```bash
# 重新扫描
ls ~/Downloads/essays/
```
对每个文件名匹配清单中的"建议文件名"，匹配成功 → 读取 PDF 内容 → 按全文笔记标准写笔记，
笔记顶部注明：
```markdown
**来源**: 用户手动下载
**本地路径**: ~/Downloads/essays/smith-2023-amj-trust.pdf
**来源等级**: 全文
```

### Step D: 模式差异
- **supervised 模式**: 输出清单后**暂停一次**（这是 Phase 2 内的小型审批），等用户上传或明确"放弃"再推进
- **autonomous 模式**: 输出清单后**不阻塞**，按"摘要"等级先继续推进；在最终研究报告中显著标注"以下 N 篇论文需用户后续手动补充全文以提升综述深度"

---

## 与续跑/修订的协同

- checkpoint.md 应记录人工下载清单状态：哪些已收到、哪些仍缺
- 用户隔几天上传 PDF 后再启动 `/deep-research resume` → 调研者重新扫 `~/Downloads/essays/` → 把新落盘的论文补进笔记
- 修订模式下若新触发 Tier 3，新清单写入 `revisions/R{n}/manual-download-needed.md`

---

## 不要做的事

- ❌ 不要假装从清单中的论文"读到了内容"——必须真的从 `~/Downloads/essays/` 读 PDF 后才写全文笔记
- ❌ 不要把没列入清单的论文随手放进清单凑数
- ❌ 不要在清单中列基线已纳入的论文（避免用户重复劳动）
- ❌ 不要修改 `~/Downloads/essays/` 中的文件（只读）
