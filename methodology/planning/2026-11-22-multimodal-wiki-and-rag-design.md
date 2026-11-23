# 多模态开源项目 Wiki 与 RAG 演示 —— 设计文档

> 日期：2026-11-22
> 状态：设计已确认；P0/P1/P2a 已执行并用实测数据校准本文 §2.2–§2.4 与 §9
> 归属仓：`projects`（工装、契约、文档语料与 RAG 应用）
> 前置：旧 `projects/docs/wiki/{db-wiki,rag-wiki,agent-wiki}` 94 篇（3.62 M 字）**全部废弃，不保留**；
> 新语料落在 `projects/docs/wiki/`，工装落在 `projects/scripts/wiki/`，RAG 应用落在 `projects/apps/wiki-rag/`。

---

## 0. 一句话

把「数据库 / RAG / Agent / LLM」四类开源项目的**每个机制单元**各写一篇 md 深度页，
用多 agent 并行按库流式生产，靠**机械门禁**而非人工 review 保质量，
再在此语料上建一个可演示的三栏 RAG 问答应用。

篇数由逐库机制盘点决定，不由文件数或配额倒推；当前已实测 151 页（postgres），其余库待盘点。

---

## 1. 已确认的决策

| # | 决策 | 选择 | 理由 |
|---|---|---|---|
| 1 | 文档源格式 | **Markdown**，wiki 只作渲染产物 | 三类内容（ASCII 架构图 32.1%、对比表 8.6%、`db_<模态>_<动作>` 类文本）正是 MediaWiki 语法失效率最高的地方 |
| 2 | 读者 | 人与 RAG **同等重要** | 两者都要，各自的结构约束都要满足 |
| 3 | RAG 定位 | **可演示的产品雏形** | 不是内部验证脚本，要有问答与引用回源 |
| 4 | 旧文档 | **db-wiki 全废，全部重写** | 新契约（无行号、结构化 refs、固定章节）与旧格式不兼容，迁移成本高于重写 |
| 5 | 项目范围 | 四类：数据库 / RAG / Agent / LLM | 清单本身决定篇数，不设篇数配额 |
| 6 | 特性粒度 | **机制单元**（A） | 「能不能不依赖别的篇讲清楚一个机制」为判据 |
| 7 | 深度 | 全量 L2，**不设行数上限** | 章节契约替代行数指标 |
| 8 | 源码引用 | **不要行号**，路径 + 符号名 | 行号腐化实测 7.3%（5/68 篇已记录失效，tdengine 单篇 75 处） |
| 9 | 引用校验 | **硬失败** | 符号查不到即不合格，必须修文档；不允许豁免 |
| 10 | refs 路径基准 | **相对项目仓根**（如 `src/backend/storage/buffer/bufmgr.c`） | 可直拼 raw URL，不依赖本地子模块路径 |
| 11 | 生产模式 | **多 agent 并行摊平** | 串行需 6.8 年，并行是唯一可行路径 |
| 12 | 推进方式 | **按库流式**（B） | 契约可边写边纠，首库问题不污染全量清单 |
| 13 | 开局库 | **postgres**（A） | 最坏的契约试金石；151 个机制页足以暴露全部规模化问题 |
| 14 | RAG 后端 | **独立第三方栈**，不接主核 `engine/db` | 本设计只做文档语料与演示，主核能力验证不在范围内 |
| 15 | 技术栈 | Python 3.11 + FastAPI + 嵌入式向量库 + 静态前端 | 零构建，与 `apps/` 既有风格一致 |
| 16 | 生成层 | **先抽取式**，预留 LLM 插槽 | 抽取式无幻觉、引用天然可溯源，适合稳定演示 |
| 17 | 界面 | 三栏工作台（方案 A，见 §7） | 问答 + 出处面板并列，引用可验证度就地可见 |

---

## 2. 范围与规模

### 2.1 项目清单分层

| 层 | 判定 | 约库数 |
|---|---|---|
| T1 核心 | GitHub ≥ 5k star，或 DB-Engines 前 30，或行业事实标准 | 60 |
| T2 重要 | 1k–5k star，或某模态代表实现 | 70 |
| T3 补充 | 其余在 `catalog.yml` 登记、有研究价值的 | 20 |
| **数据库小计** | | **150** |
| RAG / Agent / LLM | 同类分层口径 | **120** |
| **合计** | | **270** |

清单落盘时**必须记录取数日期**（`stars_as_of`）。star 数会变，一年后需能解释某库为何在 T1。

现有 `reference/reference/catalog.yml` 已有 97 条（`core` 25 / `representative` 56 / `innovative` 16），
覆盖 15 个 model 分类；本次在此基础上重新圈定到约 270 库。

### 2.2 规模实测依据

**已废止的折算法。** 早期用「核心源文件数 × 0.66 篇」估算总量（postgres 500 文件 → 332 篇）。
P2a 的全局源码盘点证明这个折算不成立：按文件体积切页会产生大量伪机制页，篇数由
**机制边界**决定，与文件数没有稳定比例关系。0.66 这个系数作废。

**当前唯一实测锚点是 postgres**（`postgres@9b5d4e3331753b3399e84965093109cac8f364bf`）：

| 项 | 实测值 | 取证方式 |
|---|---|---|
| 核心源文件（`src/backend/**/*.c`） | 904 个 `.c` | 逐文件 Ctags 扫描 |
| 函数定义索引 | 18,547 条 `kind=function` | Universal Ctags 6.1.0 |
| **机制页总数** | **151**（12 个领域） | 逐条机制盘点 + 人工 review |
| refs 对总数 | 604 组 `(path, symbol)`，命中失败 0 | 同 path 定义索引 + `wiki_refs.py` 词边界门禁 |
| 旧按文件生成的篇数 | 332，其中 259 篇（78.0%）标题是「<文件名> 模块」兜底 | 审计表 |

postgres 上「机制篇 / 核心源文件」实际约 **0.17**（151 / 904）。这个比值只能作为
**未盘点库的粗略上界参考**，不能当验收口径——每个库仍须独立做机制盘点。

### 2.3 总量

总量随盘点推进收敛，下面三种口径并存，只有第三种是验收口径：

| 口径 | 总量 | 性质 |
|---|---|---|
| 旧文件体积折算（已废止） | ~24,800 篇 | 由作废的 0.66 系数推出，仅作历史对照 |
| 机制盘点线性外推 | ~11,000 篇 | postgres 实测 0.17 篇/核心源文件线性外推到 152 个已登记库；**假设外推，不是盘点结果** |
| **逐库盘点结果之和** | **待定** | 唯一验收口径。每个库一篇机制页的产出量由该库自己的盘点决定 |

已登记库数同样是实数：当前 `catalog-projects.yml` 有 **152 条**（T1 39 / T2 85 / T3 28），
设计原定约 270 条，缺口在 §10 登记为未决项，不以「凑数」方式补齐。

### 2.4 工期与成本

**旧基线（人工撰写）已不适用于本流水线。** `git log -- docs/db-wiki` 的 68 篇 / 2.81 M 字 /
7 活跃日 ≈ 10 篇/天/operator，是纯人工撰写的速率；本设计的生产方式是 AI 多 agent 并行，
不可直接套用。

**当前唯一实测是 P2b 的 A 组试点**（20 页、4 个并行 writer）：

| 项 | 实测/估算 |
|---|---|
| A 组页数 | 20 页（B–L 尚有 131 页） |
| 各分片工作时长 | W1 ≈14 min、W2 ≈7 min、W3 ≈7 min、W4 ≈10 min |
| 并行墙钟 | 约 13–14 min（时间采样有近似，不作精确吞吐承诺） |
| token / 费用 | **不可观测**（provider 未暴露 usage），不做估算冒充实测 |
| 一次通过与返工 | 见 `projects/docs/wiki/p2-pilot-report.md` 的分项记录 |

按名义基线线性外推，剩余 131 页在 4 并发下约 22 分钟、8 并发约 11 分钟；该估算未计
跨主题学习、分片均衡、审阅与返工开销，实际更长。**并行度上限先取 4–8，不开 16。**

成本口径仍缺一项硬数据：单篇 token。契约 2 K + 源码 5–15 K + manifest 0.2 K + 输出 5 K
是**推导值**，不是实测值；provider 暴露 usage 之前，任何总量 token 数都不写进计划。

---

## 3. 目录结构与元数据

### 3.1 一个项目一个目录，一个特性一篇

```
projects/docs/wiki/
├── catalog-projects.yml                ← 文档生产清单（152 条，实测；目标约 270 条）
├── p2-pilot-report.md                  ← P2 A 组试点实测报告
├── projects/<project>/                 ← 一个已盘点项目一个目录
│   ├── _manifest.yml                   ← 该项目的机制页清单（写作的唯一输入）
│   ├── _index.md                       ← 项目总览（page_type: overview）
│   ├── a01-<feature>.md                ← 机制页，组字母 + 两位序号 + feature slug
│   ├── b01-<feature>.md
│   └── ...
└── wiki-schema.md                      ← 章节契约 + front-matter 定义（唯一权威）
```

`_index.md` 负责项目级元信息与特性导航；特性页只讲一个机制。

> **与早期设计的差异**：原设计按「模态 → 库」两层目录（`db-wiki/relational/postgres/`）。
> 实测下来模态层没有价值——`catalog-projects.yml` 的 `model` 字段已能表达模态，
> 而目录多一层只会让 writer 多一次路径判断。落地形态改为「一层项目目录 + 组字母前缀」。
> 对比页（`page_type: compare`）仍在项目目录下的 `_compare/` 子目录。

### 3.2 front-matter

```yaml
---
project: postgres                    # 项目唯一键
feature: buffer-pool                 # 特性唯一键，与文件名尾段一致
page_type: feature                   # overview | feature | compare
schema_version: 1
refs:                                # 机器可读引用，校验脚本只读此字段
  - path: src/backend/storage/buffer/bufmgr.c
    symbols: [ReadBufferExtended, PinBuffer, FlushBuffer, MarkBufferDirty]
  - path: src/backend/storage/buffer/freelist.c
    symbols: [StrategyGetBuffer, StrategySyncStart]
---
```

`feature_group` / `title` / `display_name` / `feature_summary` / `language` / `license` /
`repo` / `stars` / `stars_as_of` / `created` / `updated` **不在特性页 front-matter 里**：
分组与标题由该项目的 `_manifest.yml` 承载，项目级元信息由 `_index.md` 承载。
在页面上重复一遍只会带来两份可能不一致的真相。

---

## 4. 章节契约

每篇固定章节结构，**六个必含章节 + 一个条件章节**。缺章即不合格。

- 六个必含章节：一句话职责、机制是什么、怎么实现的、关键取舍、源码落点、参考
- 一个条件章节：与主核 / AI 的关系（内容可为空，但章节标题必须在）

```markdown
# Buffer Pool 主体

## 一句话职责
一句话，不含术语堆砌。

## 机制是什么
- 解决什么问题、不解决什么问题、边界在哪
- 与相邻机制的关系：上游是谁、下游是谁

## 怎么实现的
- 核心流程、关键数据结构、并发与生命周期
- **本章节不设字数上限** —— 深度所在

## 关键取舍
- 设计上放弃了什么，为什么
- 至少一个「换成 XXX 方案会怎样」

## 源码落点
路径 + 符号名，**不写行号**：

| 文件 | 符号 | 作用 |
|---|---|---|
| `src/backend/storage/buffer/bufmgr.c` | `ReadBufferExtended` | 对外读页入口 |

## 与主核 / AI 的关系
与 `engine/db` 主核的对应物；对 RAG / Agent 的启示。可为空，但章节标题必须在。

## 参考
论文 / 官方文档 / benchmark，至少 2 个。
```

现网 68 篇的实测构成（正文 59.3% / 代码块与 ASCII 图 32.1% / 表格 8.6%）说明骨架固定不会导致内容雷同：
第六章之后（实质章节）占 93.7%，模板部分（一~五章）仅 6.3%。章节契约复用的是那 6.3% 的骨架，深度落在「怎么实现的」里。

---

## 5. 校验门禁（硬失败）

### 5.1 `wiki_lint.py` —— 结构门禁

| 检查项 | 失败后果 |
|---|---|
| 六个必含章节齐全 | FAIL |
| front-matter 必填字段齐全 | FAIL |
| `feature` 与文件名尾段一致、全局唯一 | FAIL |
| 出现 `path:123` 形态的行号引用 | FAIL |
| 外部参考少于 2 个 | FAIL |
| `page_type: compare` 不在 `_compare/` 下 | FAIL |

### 5.2 `wiki_refs.py` —— 引用门禁（硬失败）

对每个 `refs[].path`：

1. 在本地浅克隆仓中确认文件存在 → 不存在即 FAIL
2. 对 `refs[].symbols` 逐个在文件内匹配 → **任一未命中即 FAIL**

无豁免机制。宏与内联函数的正确做法是把 refs 指向其真实定义文件（通常是 `.h`）。

**有效性已实测**：`bufmgr.c`（268 KB）中 `ReadBufferExtended` / `ReadBuffer_common` / `ReleaseBuffer` / `UnpinBuffer` / `PinBuffer` / `MarkBufferDirty` / `BufferAlloc` / `FlushBuffer` / `TerminateBufferIO` 共 9 个真实符号全部命中；随手虚构的 `BufferGetHashEntry` 被正确判为不存在。

**门禁的能力边界**：`wiki_refs.py` 只做**词边界匹配**，能证明「这个符号出现在这个文件里」，
不能证明「它就是这个机制的核心入口」，更不能证明页面正文对该符号的解释正确。
P2a 盘点期间 604 个 ref 对全部命中，但同批页面仍被源码复核发现 6 处语义错误
（见 §8）。**refs 命中是必要条件，不是充分条件。**

### 5.3 `wiki_refs.py --locate` —— 定位辅助

硬失败的最大风险是返工。为此提供反向工具：在浅克隆仓内全仓 grep 符号，返回真实文件路径与上下文行。写作者先 locate 再填 refs，把门禁从惩罚变成反馈。

### 5.4 契约变更预算

`schema_version` 从 1 起。**最多升 3 次**。每次升版必须写「本次改了什么、为什么」。已归档库不回溯，只在 `_index.md` 标记其 `schema_version`。第 4 次想改说明前 3 次没改对，须停下重审契约。

---

## 6. 并行生产流水线

### 6.1 源码缓存

**必须浅克隆，不用 raw 拉取。** 实测拉取 `bufmgr.c`（268 KB）耗时 **3005 ms**：

- raw 方案：postgres ~500 文件 = 25 分钟；270 库 × 平均 400 文件 = 56,000 次 ≈ **46 小时纯网络等待**（串行，不含重试）
- 浅克隆：整仓一次 `--depth 1`，270 库按均值 100 MB ≈ **14 GB**（本机余 434 GB，充足）

### 6.2 按库流式推进

```
for each 库（多库并行）:
    Phase 1  清单：读源码树 → 枚举机制单元 → _manifest.yml
             门禁: manifest_lint PASS
    Phase 2  写作：N 篇并行，写作用域 = 各自 page 文件
             门禁: wiki_lint + wiki_refs 双绿
    Phase 3  修复：只改失败页，不碰已通过页
             门禁: 双绿后该库归档
    Phase 4  索引：_index.md 生成 + 该库入 RAG 索引
```

**首库未 PASS 之前，不进第二库清单。** 这是流式的前提——契约问题必须在最小的污染面被抓住。

### 6.3 `_manifest.yml`（清单）

```yaml
project: postgres
schema_version: 1
pages:
  - feature: buffer-pool
    feature_group: A-存储与IPC
    title: 共享缓冲池与替换
    page_file: a01-buffer-pool.md
    refs:
      - path: src/backend/storage/buffer/bufmgr.c
        symbols: [ReadBufferExtended, PinBuffer, FlushBuffer]
```

清单是契约与写作之间的唯一接口。**Phase 1 全部 PASS 之前一篇都不写**——这是把「先写正文再统一口径」的顺序倒过来，
现有 68 篇的教训（README 深化列改了 5 轮、修复 16 处断链、5 篇 pin 失效）正来自反序。

**清单本身是代码产出的，不是手写的**：每个项目的机制条目放在
`projects/scripts/wiki/<project>_mechanism_specs.py` 里作为唯一数据源，
由 `gen_<project>_manifest.py` 生成 `_manifest.yml`，并由生成器自带的
`check_specs()` 拒绝重复 feature、重复 page_file、伪标题（兜底「…模块」）、
空 boundary、空 refs、缩写路径、非 `src/` 路径、未知分组。
手写 151 条 YAML 会引入漂移，而漂移在写作阶段表现为「writer 擅自改了 refs」。

### 6.4 写作用域隔离

并行 agent 共享同一文件系统，因此必须划分**不相交**写入范围：

- 一个 writer agent 只允许写自己 manifest 条目内的 `.md` 文件
- `_index.md` 与 `_manifest.yml` 由 Phase 1 / Phase 4 的单一 agent 负责，不与 writer 并发
- `wiki-schema.md` 与两个门禁脚本属于 P0 串行产物，写作期**只读**

写作用域重叠是 advisory 而非锁，因此 Phase 2 调度时必须按 `feature_group` 分片，保证同一时刻不同 agent 的 `page_file` 前缀互不交叠。

---

## 7. RAG 演示应用

### 7.1 界面（方案 A：三栏工作台）

已在浏览器中预览确认。三栏结构：

| 栏 | 内容 |
|---|---|
| 左 | 模态分类（12 类）+ 特性页数 + 校验状态（`PASS` / `FAIL`） |
| 中 | 提问区 → 回答正文 → 内嵌对比表 → 检索抽屉（向量召回条数 / BM25 命中段数 / 耗时） |
| 右 | 出处面板：每条引用带 `path` + 符号名 + 模态标签 + 片段预览；检索失败时显示回退态 |

**与旧预览的差异**：预览稿里右栏的「🔬 行号级 / 🔷 canonical / ◻ 待核实」三档标记在新契约下**不再存在**——
新文档全部无行号、全部过符号硬校验，因此只有 `PASS` / `FAIL` 两态。
左栏原先的「引用可验证度分布」相应改为「特性页数 + 校验状态」。

### 7.2 技术栈

| 层 | 选型 |
|---|---|
| 后端 | FastAPI |
| 向量 | 嵌入式向量库（chroma / lancedb），本地持久化 |
| 检索 | 向量 + BM25 混合，RRF 融合 |
| 生成 | **抽取式**：按融合分数拼回原文段落，引用由后端按段落编号生成，不由模型自编 |
| 前端 | 纯静态 HTML/CSS/JS，零构建 |

生成层预留 LLM 插槽：`generator` 接口先实现抽取式，后续可替换为 API 调用而不改检索与引用逻辑。

### 7.3 切块策略

- **一页 = 一个机制单元**，天然切块边界
- 页内再按 `##` 切，每块带 `project` / `feature` / `page_type` / `schema_version`
- `_index.md`（项目级入口块）**检索降权**，避免在「XX 是什么」问题上抢答
- `_compare/` 对比页单独索引，供跨库对比问答

### 7.4 增量索引

流式推进意味着语料逐步增长。索引必须**按库可独立重建**：

- 每库 Phase 4 完成后触发该库重建，其余库不动
- 索引结构在 Phase 0 定死，否则每库无法独立入库

这与「先写完文档再开发 RAG」不冲突：RAG 的**开发**在文档之后，但索引**数据结构**必须前置定义。

---

## 8. 质量上限 = 门禁严格度

这是本设计最重要的一条判断依据。

现有 68 篇已发生：README 口径改 5 轮、16 处断链、5 篇引用腐化（占 7.3%）。按 68 → 151 的实测放大倍数计，
约 11 处口径不一致 / 11 处断链 / 11 篇引用腐化；随库数增长线性放大。

**这个量级下人工 review 不存在。** 六必含章节契约 + 符号硬失败 + refs 结构化，是本设计唯一的质量保证。它必须比写作本身更早、更严。

实测补充（A 组试点）：**词边界命中不等于语义正确**——`wiki_refs.py` 只证明符号出现在该文件。
试点期间由源码复核发现并修正了 SSI 冲突方向写反、等待队列职责误挂、DSM 退化语义被夸大等 6 处问题，
这些错误双门禁全部放行。**门禁是下限，不是上限**；抽检与源码复核是发现语义错误的唯一手段。

---

## 9. 执行顺序总览

| 阶段 | 内容 | 门禁 | 状态 |
|---|---|---|---|
| P0 | `wiki-schema.md`、`wiki_lint.py`、`wiki_refs.py`、`locate` 辅助、浅克隆工具 | 脚本自测通过（含虚构符号被拒的用例） | **完成**（产物已迁 `projects/scripts/wiki/`） |
| P1 | 项目清单圈定（T1/T2/T3 分层，记 `stars_as_of`）与 manifest 读写/门禁工装 | 清单通过 schema 校验 | **完成**（清单 152 条；目标 270 条的缺口见 §10） |
| P2a | postgres 全局机制盘点，机制清单重建（332 → **151 页 / 12 组**） | manifest_lint PASS + 604 ref 对全部命中 | **完成** |
| P2b–P2d | postgres **A 组 20 页**正文试点 + `_index.md` + 试点报告 | 双绿 + 5 页深度抽检 | **完成**（抽检 29/30，一项取舍证据不足保留未决） |
| **P3** | postgres **B–L 共 131 页**正文批量写作 + 归档 + 速率/成本再校准 | 每组双绿 + 抽检 + 校准报告 | 待执行 → `2026-11-23-multimodal-wiki-p3-postgres-full.md` |
| **P4** | 其余库按库流式推进（每库独立盘点 → manifest → 正文 → 归档） | 每库双绿 | 待执行 → `2026-11-23-multimodal-wiki-p4-other-libraries.md` |
| **P5** | RAG 演示应用（三栏工作台 + 增量索引 + 抽取式生成） | 演示可用 | 待执行 → `2026-11-23-multimodal-wiki-p5-rag-app.md` |

P3/P4/P5 三份实施计划在 `ai-docs/methodology/planning/` 下，与 P0/P1/P2 同规格。

---

## 10. 未决与风险

| 项 | 状态 |
|---|---|
| token 预算天花板 | **仍未知**。provider 未暴露 usage，单篇 token 无实测值；P3 结束后若仍不可观测，只能给出「篇数 × 配额速率」的上界 |
| GitHub API 认证 | 未配置。未认证限流 60 次/时；当前清单 152 条的 stars 为实取值，继续扩库需 token |
| 清单规模 152 vs 270 | **未决**。缺口 118 条；不靠凑数补齐，扩容需逐条实取 stars/license/language 并说明分层依据 |
| postgres 机制页篇数 | 已定：**151 页**（A20 B19 C15 D17 E13 F13 G7 H5 I10 J12 K11 L9）。不再出现 332 这个数 |
| 抽检未满分项 | a19 的「取舍」项因缺少具体 caller 的 fallback 证据记 0；P3 写作同主题页时须补足证据再复核 |
| 旧 db-wiki 处置 | 已定：全废重写。`reference` 仓该目录 166 次提交历史保留（不删历史，只换内容） |
| RAG 落库位置 | **已定**：`projects/apps/wiki-rag/`，独立第三方栈，不接主核 `engine/db` |
| 写作用域隔离强度 | 写作用域重叠目前是 advisory 而非锁。P3 起 writer 分片必须按组字母前缀固定，同一时刻不得有两个 writer 触碰同一组 |
