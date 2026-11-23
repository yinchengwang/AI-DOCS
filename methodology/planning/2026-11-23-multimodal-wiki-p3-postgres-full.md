# wiki P3（postgres 剩余 131 页批量写作）Implementation Plan

> **执行状态（2026-11-23 完成）**：131 页全部写完并入库。
> - 目录门禁：`wiki_lint` 152 页 0 失败、`wiki_refs` 152 页 0 失败、`wiki_manifest_lint` PASS
> - 契约测试：`test_postgres_mechanism_pages` 扩展到 **151 页全覆盖**（6/6 通过）
> - 全套工装测试：`Ran 353 tests ... OK (skipped=1)`
> - 导航：`_index.md` 由 `scripts/wiki/rebuild_postgres_index.py` 从 manifest 生成，151 行
> - 验收与抽检记录：`projects/docs/wiki/p3-full-report.md`、`p3-review-a-bc.md`（A/B/C）、
>   `p3-review-d-ef.md`（D/E/F）、`p3-review-g-l.md`（G–L 抽样）
> - 与原计划的偏差（并发度、波次口径、抽检页数）与 10 页抽检修订明细见 `p3-full-report.md` §5
> - **本计划正文保留为原始执行记录，不按实际执行回改。**

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 按 P2 定稿的机制清单写完 postgres 剩余 **131 页**（B–L 共 11 组），把 A 组试点的结论放大到全库，并产出可核对的吞吐/返工/质量记录。

**Architecture:** 沿用 P2 的「清单驱动 + disjoint writer + 机械双门禁 + 深度抽检」流水线，但把粒度从「20 页一个试点」变成「按组分波次推进」。每波 4 个互不重叠的分片并行写，波次结束跑整组门禁 + 抽样复核，全部 11 组写完后重建 `_index.md` 与全库验收报告。机制清单是唯一输入，writer 不得改 feature/title/refs。

**Tech Stack:** Markdown + front-matter 受控子集、Python 3.11 标准库 `unittest`、`wiki_lint.py` / `wiki_refs.py` / `wiki_manifest_lint.py`、PostgreSQL 本地浅克隆（`projects/.cache/repos/postgres`）。

设计规格：`2026-11-22-multimodal-wiki-and-rag-design.md`
前置计划：`2026-11-22-multimodal-wiki-p2-a-group-pilot.md`（P2a 清单 + P2b–P2d A 组试点）
试点报告（事实来源）：`projects/docs/wiki/p2-pilot-report.md`

## Global Constraints

- 沿用 P2 的全部约束：六个必含章节 + 一个条件章节；禁止行号引用；refs 路径基准为项目仓根；符号校验硬失败无豁免；参考章节至少 2 个可达外链；UTF-8 无 BOM、LF；commit message 用中文。
- **页数与清单一致即正确**：131 页，不多不少。不为凑数补页，不因难写而删页。
- **writer 不得改** feature / title / refs / `feature_group`；发现清单有错只上报，不自行修正。
- **写作用域按分片固定**：同一时刻不同 writer 的 `page_file` 前缀不重叠。分片表在下文预先写死，中途不得重排。
- **单页失败只改该页**：禁止放宽门禁，禁止删 refs 符号绕过检查。
- **语义正确性由抽检兜底**：`wiki_refs.py` 只证明符号出现在该文件，不证明正文解释正确（P2 试点实测 6 处双门禁放行的语义错误）。
- `.cache/repos/postgres` 是运行时缓存，永不入库。
- `projects` 工作树中与本任务无关的用户改动（如 `engine/db/**`）不得暂存、不得覆盖。

---

## 文件与写入范围

| 文件/目录 | P3 阶段责任 |
|---|---|
| `projects/docs/wiki/projects/postgres/b*.md` … `l*.md` | 131 页正文，由分片 writer 写 |
| `projects/docs/wiki/projects/postgres/_index.md` | 全部 151 页写完后重建（含完整导航表） |
| `projects/docs/wiki/projects/postgres/_manifest.yml` | **只读**；由 `gen_postgres_manifest.py` 重新生成并校验一致性，不手改 |
| `projects/scripts/wiki/postgres_mechanism_specs.py` | **只读**；发现清单问题上报到计划待办，不在写作期改动 |
| `projects/scripts/wiki/test_postgres_mechanism_pages.py` | 扩展为「151 页全覆盖 + refs 与 manifest 深度相等」 |
| `projects/docs/wiki/p3-full-report.md`（新建） | 记录各波次实测吞吐、返工轮次、抽检分项、遗留问题 |

**P3 不改** `wiki_schema.py`、`wiki_frontmatter.py`、`wiki_lint.py`、`wiki_refs.py` 的行为。发现契约 bug 时暂停并单独建修复任务（与 P2 同纪律）。

---

## 1. 分片表（20 个分片，131 页，互不重叠）

按机制簇切分，每个分片 4–10 页；**分片不跨组**（跨组会让 writer 同时承担两套领域语汇，试点已证明主题跨度越大措辞越难统一）。

| 分片 | 页 | 组 | 主题范围 |
|---|---|---|---|
| S1 | `b01`–`b10` | B 事务与 WAL | WAL 写入/检查点/恢复/reader、standby 冲突、multixact、2PC、事务状态 SLRU、XID 状态与分配 |
| S2 | `b11`–`b19` | B 事务与 WAL | 提交时间戳、子事务父映射、事务生命周期、WAL 预取与资源管理、rmgr 描述、归档/备份接口、同步等待、通用 xlog API |
| S3 | `c01`–`c08` | C 堆表与元组 | 表访问方法、vacuum/pruning、可见性图、MVCC 可见性、rewrite、元组布局、关系选项、TOAST |
| S4 | `c09`–`c15` | C 堆表与元组 | TID store、同步顺序扫描、属性映射与转换、堆 WAL redo、大对象、堆页布局、heap→index 取元组 |
| S5 | `d01`–`d09` | D 索引族 | 索引 AM 框架、元组表示、btree 扫描/查找/插入分裂/键预处理、btree 构建、GIN 写入与 pending list、GIN 检索 |
| S6 | `d10`–`d17` | D 索引族 | GIN 数据页、GiST 检索/插入、GiST 构建与页分裂、hash、BRIN 核心与 opclass、SP-GiST、索引 WAL redo |
| S7 | `e01`–`e07` | E 优化器 | planner 编排、查询预处理、子查询改写、rel 元数据、路径生成、join 搜索、geqo |
| S8 | `e08`–`e13` | E 优化器 | 路径排序、成本与基数、等价类与排序消除、谓词推理、计划构造、计划引用修正 |
| S9 | `f01`–`f07` | F 执行器 | executor 生命周期、executor state、节点分发、表达式求值、tuple slot、扫描节点、索引与 bitmap 扫描 |
| S10 | `f08`–`f13` | F 执行器 | join 算法、聚合与窗口、排序与物化、append 与 subplan、并行执行、modify 与分区路由 |
| S11 | `g01`–`g07` | G 目录与缓存 | catalog 存储与 bootstrap、syscache、relcache、缓存失效语义、对象依赖、namespace 解析、ACL 与属主 |
| S12 | `h01`–`h05` | H 内存与排序 | 内存上下文、resource owner、tuplesort、logical tape、排序原语 |
| S13 | `i01`–`i05` | I 复制与高可用 | 物理流复制、复制槽、同步复制、逻辑解码、逻辑复制 apply |
| S14 | `i06`–`i10` | I 复制与高可用 | reorder buffer、快照构建、表同步、origin 跟踪、归档与基础备份 |
| S15 | `j01`–`j06` | J 服务端与命令面 | postmaster 生命周期、后端协议会话、utility 命令、DDL 对象命令、COPY、后台 worker 框架 |
| S16 | `j07`–`j12` | J 服务端与命令面 | autovacuum launcher、checkpointer 进程、pgstat 活动视图、错误报告、异步通知与 LISTEN、会话与 GUC |
| S17 | `k01`–`k06` | K SQL 处理与安全 | parser 流水线、parse analysis、rewrite 与视图、节点树工具、TABLESAMPLE、客户端认证 HBA |
| S18 | `k07`–`k11` | K SQL 处理与安全 | 认证方法、TLS 传输、行级安全、权限检查、事件触发器 |
| S19 | `l01`–`l05` | L SQL 类型与内建函数 | 类型系统与 typcache、数组类型、numeric、日期时间、JSON/JSONB |
| S20 | `l06`–`l09` | L SQL 类型与内建函数 | text/varlena、range/multirange、网络类型、正则与全文检索 |

合计 131 页。**分片表是写作用域的唯一来源**；调度时一个分片对应一个 writer，同一分片不并发。

---

## 2. 波次编排

| 波次 | 分片 | 页数 | 并发 | 覆盖组 |
|---|---|---:|---:|---|
| Wave 1 | S1–S4 | 34 | 4 | B、C |
| Wave 2 | S5–S8 | 30 | 4 | D、E |
| Wave 3 | S9–S12 | 25 | 4 | F、G、H |
| Wave 4 | S13–S16 | 22 | 4 | I、J |
| Wave 5 | S17–S20 | 20 | 4 | K、L |

合计 **131 页**（34+30+25+22+20）。页数由上方分片表逐片相加得出，与
`postgres_mechanism_specs.py` 的实际 `page_file` 清单一一对应（已逐片核对）。

**并发上限先取 4。** 试点实测：4 路并行、20 页、各分片 7–14 分钟，未见冲突或质量下降；8 路没有实测依据。
每个波次跑完后再决定下一波是否提到 8：**判据是「一次通过率 ≥ 80% 且抽检无系统性错误」**，不是「时间紧」。
若某波次抽检发现 ≥3 处同类系统性问题（如都把同一概念讲反），下一波先停下来做定向修正，再继续。

单波次内部分片可并行，但**同一分片内的页必须顺序写**：同组页面共享术语，后一页要沿用前一页已确立的措辞。

---

## 3. Writer 单页工作流

每个 writer 拿到分片后按此流程逐页执行，**每页独立走完全流程再进下一页**：

### Step 1：读规格

从 `_manifest.yml`（或 `postgres_mechanism_specs.py`）取该页的 `feature`、`title`、`refs`。
**refs 是硬约束**：正文必须解释这些符号的实际作用，不能换成别的符号。

### Step 2：读源码

在 `projects/.cache/repos/postgres` 下读 `refs` 指向的文件，确认：

- 每个符号在该文件里的**真实职责**（读函数体，不靠符号名猜）。
- 该机制涉及的关键数据结构（结构体字段、常量）与生命周期。
- 并发边界（谁持锁、谁等待、失败路径）。

### Step 3：写页

模板（与 P2 一致）：

```markdown
---
project: postgres
feature: <manifest feature>
page_type: feature
schema_version: 1
refs:
  - path: <manifest refs path>
    symbols: [<manifest refs symbols>]
---

# <manifest title>

## 一句话职责
<一句话，说明所管理的机制状态或能力，不堆术语。>

## 机制是什么
<解决的问题、明确边界、不负责什么、与上下游关系。>

## 怎么实现的
<按真实执行顺序 3–8 步：关键流程、核心数据结构、并发与生命周期。>

## 关键取舍
<至少 2 项真实取舍：放弃了什么、为什么、替代方案会怎样。区分源码事实与架构推论。>

## 源码落点
| 路径 | 符号 | 本文对应作用 |
|---|---|---|
<逐条解释 refs 符号，不能把符号列表当正文。>

## 与主核 / AI 的关系
<真实对应或明确「暂无直接对应」，不硬套。>

## 参考
- [标题](https://...) —— 支撑的具体事实
- [标题](https://...) —— 支撑的具体事实
```

### Step 4：单页门禁

```powershell
cd C:\code\projects\scripts\wiki
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres\<page>.md --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres\<page>.md `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

任一失败就修这一页，修完重跑，直到双绿。**不进入下一页。**

### Step 5：外链核验

`参考` 章节的每个外链做一次 HTTP GET，确认可达且与该页论点相关。
已知失效并已替换的样本：`docs/current/locking.html`（404 → `explicit-locking.html`）、
`docs/current/storage-buffer.html`（404 → 官方源码链接）。

### Step 6：记录

记录本页：一次通过还是返工、返工原因、耗时、读过的源码文件、发现的问题。

---

## 4. 波次门禁与抽检

### 4.1 整组门禁

每个波次结束，波次覆盖的全部页跑整目录门禁：

```powershell
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

### 4.2 深度抽检

**每波覆盖的每个组抽 5 页**（试点期是 20 页抽 5 页，抽样比例沿用 25%；本阶段按组固定 5 页，
便于跨组横向比较）。各波组数与页数：

| 波次 | 覆盖组 | 抽检页数 |
|---|---|---:|
| Wave 1 | B、C | 10 |
| Wave 2 | D、E | 10 |
| Wave 3 | F、G、H | 15 |
| Wave 4 | I、J | 10 |
| Wave 5 | K、L | 10 |
| **合计** | **11 组** | **55** |

抽样用固定 seed `random.Random(20261123)` 从各组页列表中按序取，保证可复现。

每页按 6 项评分（每项 0/1），抽检须由**独立于该页 writer 的审阅者**执行：

| # | 评分项 | 判定要点 |
|---|---|---|
| 1 | 机制边界 | 负责/不负责清楚，与相邻机制不重叠 |
| 2 | 实现映射 | 流程能落到 refs 里的真实源码符号，符号职责解释正确 |
| 3 | 机制深度 | 至少一个数据结构/生命周期/并发边界的实质说明 |
| 4 | 关键取舍 | 至少 2 项，且能区分源码事实与架构推论 |
| 5 | 来源质量 | ≥2 个权威且与论点相关的外链，URL 可达 |
| 6 | RAG 切块 | 章节可独立理解，不依赖未说明的上下文 |

**评分 4 项（取舍）必须有源码或权威材料支撑**，不能只写「这样设计更简单」。
试点中 a19 的「取舍」项因缺具体 caller 证据记 0 —— 同样的问题不允许在 P3 重演：
**涉及退化/回退路径的取舍，必须点名至少一个真实 caller 及其源码位置**。

抽检不通过的页：返工 → 重评 → 记录。若同一评分项在 ≥3 页上同时失败，判为**系统性缺陷**，
先定位根因（清单问题？模板问题？领域理解偏差？），修完再继续下一波。

### 4.3 全库抽检

11 组写完后，对全库再抽 10 页做终检（抽样 seed `random.Random(20261123)` 续用同一序列），
重点看跨组一致性：术语是否统一、机制边界是否互相越界、refs 符号职责是否解释正确。

---

## 5. `_index.md` 重建

131 页全部双绿后重建 `_index.md`（单一作者，不与 writer 并发）：

- `page_type: overview`，契约必填字段齐全（`project` / `display_name` / `repo` / `license` / `language` / `stars` / `stars_as_of`）。
- **完整导航表**：12 个分组 × 组内全部 151 页，列出 `page_file` 与标题。这是 A 组试点版 `_index.md` 缺的部分，本阶段补上。
- `refs` 字段填项目级代表性符号（后端主循环、planner、executor、WAL、缓冲读入、smgr）。
- `stars` / `stars_as_of` 沿用 catalog 现值；**占位值不得当作真实热度**，未回填前在页面上注明来源与占位状态。

---

## 6. 契约测试扩展

`projects/scripts/wiki/test_postgres_mechanism_pages.py` 扩展为：

| 断言 | 内容 |
|---|---|
| 页数 | postgres 目录恰好 151 个机制页（`[a-l]NN-*.md`），与 manifest 一致 |
| 覆盖 | manifest 中每个 `page_file` 都存在对应文件 |
| 一致性 | 每页 front-matter 的 `refs` 与 manifest 条目**深度相等** |
| 唯一性 | `feature` 全局唯一（`wiki_lint` 已查，这里做双保险） |
| 组完整性 | 12 个 `feature_group` 的页数与 spec 一致（A20 B19 C15 D17 E13 F13 G7 H5 I10 J12 K11 L9） |

同时保留 A 组时代的 3 项断言，不改其语义。

---

## 7. 全库验收报告

新建 `projects/docs/wiki/p3-full-report.md`，记录：

| 项 | 要求 |
|---|---|
| 页数 | 151 页全部双绿 |
| 波次记录 | 每波的分片、页数、起止时间、一次通过页数、返工页数与轮次 |
| 抽检 | 每波抽检样本、6 项分项得分、返工记录、系统性问题与根因 |
| 吞吐 | 各波次 wall-clock（注明采样口径，不写伪精确值） |
| token/成本 | provider 若仍未暴露 usage，写「不可观测」，**不做估算冒充实测** |
| 并行度建议 | 基于实测给出 P4 的并行度建议与依据 |
| 遗留 | 未解决问题、未抽检页、清单存疑处 |

---

## 8. 验收条件

1. `projects/docs/wiki/projects/postgres/` 下 151 个机制页全部存在，且 `wiki_lint` + `wiki_refs` 整目录 0 失败。
2. `wiki_manifest_lint.py _manifest.yml` PASS；重新生成的 manifest 与仓库内文件一致。
3. 扩展后的契约测试通过；全套 unittest 全绿（记录实际测试数与 skip 原因）：

```powershell
cd C:\code\projects\scripts\wiki
python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint `
  test_wiki_refs test_repo_cache test_wiki_schema_doc test_wiki_catalog `
  test_wiki_manifest test_wiki_manifest_lint test_wiki_tree test_build_catalog `
  test_catalog_additions test_gen_postgres_manifest test_postgres_mechanism_pages
```

4. 每波覆盖的每个组抽检 5 页（共 55 页），评分记录完整；全库终检 10 页完成。
5. `_index.md` 重建并含 151 页完整导航表，通过 lint 与 refs。
6. `p3-full-report.md` 落盘，含上述全部记录项。
7. `.cache/repos/postgres` 仍被 gitignore 忽略。
8. 只提交 Wiki 相关路径（131 页 + `_index.md` + 测试 + 报告）；`engine/db/**` 等用户改动全程未暂存。
9. 每波次独立 commit + `git push origin main`，commit message 用中文。

## 9. 不在本阶段范围

- 不改 `_manifest.yml` 的机制条目（发现问题上报，不在写作期改）。
- 不写 `_compare/` 对比页（对比页是 P4 跨库阶段的事）。
- 不开发 RAG 应用（P5）。
- 不回填 catalog 缺失的 118 条（属 P4 扩容批次）。
- 不做 token 预算实测（除非 provider 暴露 usage）。
