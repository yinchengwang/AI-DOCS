# P2 PostgreSQL 机制盘点与 A 组试点实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 以真实、互不重复的 PostgreSQL 机制单元重建 P1 的 A–K manifest（不再把 332 当作机制篇数硬指标），再从重盘后的 A 组开展正文试点，双门禁通过、深度抽检合格后产出 `_index.md`，并记录实际耗时、返工率与一次通过率。

**Architecture:** P2a 审核现有 332 个 page spec 与其源码落点，合并重复条目、拆出确实独立的机制、补入被漏掉的真实机制，得到新的 A–K 篇数与总量；总数是盘点结果，不是约束。manifest spec 显式表达机制 slug/title 与一条或多条 refs，不再由文件体积机械生成。P2b 在重建 manifest 通过门禁后，从新的 A 组写机制页；用 disjoint writer 分片并行，主代理运行 `wiki_lint` + `wiki_refs` 硬门禁、随机抽 5 页深度复核，再生成 `_index.md`。

**Tech Stack:** Markdown + front-matter 受控子集、Python 3.11 标准库 `unittest`、PostgreSQL 本地浅克隆、`wiki_manifest` / `wiki_manifest_lint` / `wiki_lint` / `wiki_refs`。

## Global Constraints

- 文档源格式只允许 Markdown；禁止 MediaWiki 语法（`==标题==`、`[[链接]]`、`{|` 表格）。
- **禁止行号引用**。引用一律「相对项目仓根的路径 + 符号名」，不得出现 `path:123` 形态。
- refs 路径基准 = 项目仓根（如 `src/backend/storage/buffer/bufmgr.c`），不写 `projects/` 或 `.cache/repos/postgres/` 前缀。
- 符号校验**硬失败**，无豁免机制。符号未命中即整页 FAIL。
- 六个必含章节：`一句话职责`、`机制是什么`、`怎么实现的`、`关键取舍`、`源码落点`、`参考`。
- 一个条件章节：`与主核 / AI 的关系`（内容可为空，但标题必须在）。
- `schema_version` 当前为 1；本阶段不改 schema 常量，不引入新字段、不升版。
- RAG 定位是可演示的产品雏形；人类读者与 RAG 读者同等重要。
- 文档粒度是「机制单元」，不是源文件摘要；深度为 L2，不设行数上限。
- 一页 = 一个机制单元；引用只用于回到实现文件与有意义的符号，不以凑 refs 数量代替机制论述。
- 一律 UTF-8 无 BOM、LF；所有 commit message 使用中文。
- **机制质量优先于篇数**：不得为满足 332/29 而人为拆机制或保留文件名伪主题。篇数由全局源码盘点结果确定，可以低于、等于或高于 332。
- 全局盘点覆盖全部 11 组 A–K；必须去重、检查跨组重复、检查遗漏、明确机制边界与主 refs；不因源码文件数或现有组配额凑页。
- 机制事实需由 PostgreSQL 源码与权威文档交叉确认；`wiki_refs` 的词命中只是硬门禁，不等于语义正确，refs 还需人工核对属于该机制。
- P2 写作每页 front-matter 必须包含 `project`、`feature`、`page_type`、`schema_version`、`refs`；`page_type: feature`，`schema_version: 1`。
- 参考章节必须至少有 2 个 HTTP(S) 外链；优先 PostgreSQL 官方文档、PostgreSQL 官方源码仓/官方开发文档、原始论文/标准。引用不明来源不得编造。
- 浅克隆缓存路径 `projects/.cache/repos/postgres` 属运行时缓存，永不入库。
- 所有 writer 只能写其明确分配的 `.md` 文件；不得改 `_manifest.yml`、`_index.md` 或契约文件。

---

## 文件与写入范围

| 文件/目录 | P2 阶段责任 |
|---|---|
| `projects/scripts/wiki/gen_postgres_manifest.py` | P2a 移除“每个 `.c` 文件等于一页”的生成逻辑，存放经审核的 A–K mechanism specs |
| `projects/scripts/wiki/test_gen_postgres_manifest.py` | P2a 保护篇数来自 spec、跨组去重、语义表完整、refs 真实命中 |
| `projects/docs/wiki/projects/postgres/_manifest.yml` | P2a 按新的 mechanism specs 重生成，P2b/P2c/P2d 只读 |
| `projects/docs/wiki/projects/postgres/<new-A-page-files>.md` | P2b writer 仅写重盘后新 A 组的互斥文件分片 |
| `projects/docs/wiki/projects/postgres/_index.md` | P2d 单一作者在新 A 组全部双门禁通过后生成 |
| `projects/docs/wiki/p2-pilot-report.md` | P2d 记录新 A 组实测吞吐、一次通过率、返工数、token/成本观测、抽检结论 |
| 本计划 | P2a 盘点完成后记录真实 A–K 篇数、A 组页名与 writer shards |

P2 不改 `wiki_schema.py`、`wiki_frontmatter.py`、`wiki_lint.py`、`wiki_refs.py` 的行为；发现契约 bug 时暂停并单独建修复任务，避免写作中途漂移。

## 已确认的决策

1. **节奏：** 先全局重建 11 组机制 manifest，再从重建后的 A 组做试点；试点验收并记录真实数据后，再确定其余机制页并行度。
2. **范围：** A–K 全局盘点，不只修 A。去除跨组重复/伪主题、补录真实遗漏机制；篇数不预设。
3. **写作对象：** 以新 manifest 的机制单元为准；title/slug/refs 由 P2a 审核，writer 不得擅改。
4. **试点收尾：** 新 A 组所有页双门禁通过后生成 PostgreSQL `_index.md`。
5. **并行策略：** 默认 4 个 disjoint writer 分片；如果新 A 组少于 16 页，则减为 2 个分片，避免人为并发开销。分片按最终 page_file 列表预先固定。
6. **质量：** 机械双门禁 + 随机抽 5 篇深度复核。抽检维度：机制边界、实现流程/数据结构/生命周期、关键取舍、源码落点是否解释实际作用、参考质量、RAG 可切块性。
7. **正文最小深度：** `怎么实现的` 按执行顺序讲 3–8 个步骤，说明关键函数/结构如何协作并解释适用的并发/生命周期；`关键取舍` 至少 2 项真实设计取舍并说明替代方案影响；参考至少 2 个确实支撑正文的权威来源。
8. **禁止灌水：** 不允许为篇数或长度堆源码符号清单、复制官方介绍、写无信息段落；页数与篇长由真实机制复杂度决定。

## P2a：A–K 全局机制盘点与 manifest 重建

### 已审计的问题

P1 generator 的规则是“目录中 `.c` 文件按体积排序，一个文件一篇”，这不是机制建模。只读审阅已发现：

- A 组 method_worker 页面以 `pgaio_worker_pm_test_grow*` 测试辅助函数为 refs；标题是文件名兜底，不是机制。
- A 组 method_io_uring、AIO worker、AIO core 被按实现文件切成三页，需评估其是一个 AIO 机制的实现后端还是独立子机制。
- standby.c、dsm_impl.c 等文件名成为伪主题；`predicate.c` 的 `check_serial_buffers` 是校验回调而非 SSI 主机制。
- B 与 C 共用 rmgrdesc 源目录导致重复收录；当前多个组还存在源路径复用和按 executor/optimizer node 文件机械切页。
- `wiki_refs` 只检查符号字符串在目标文件出现，不证明它是函数定义或属于该机制。因此 P2a 必须人工核对符号定义及语义对应关系，不能把门禁当作语义证明。

### P2a 文件

- Modify `projects/scripts/wiki/gen_postgres_manifest.py`
- Modify `projects/scripts/wiki/test_gen_postgres_manifest.py`
- Regenerate `projects/docs/wiki/projects/postgres/_manifest.yml`
- Update this plan with the resulting A–K counts, A-group page names and writer shards

### Mechanism inventory schema

在 generator 中定义经审核的显式机制条目（常量或结构化数据），每条至少含：

```python
{
    "group": "A",
    "feature_group": "A-存储基础",
    "feature": "shared-buffer-read",
    "title": "共享缓冲读入与预取",
    "refs": [
        {"path": "src/backend/storage/buffer/bufmgr.c",
         "symbols": ["ReadBufferExtended", "PrefetchBuffer"]},
        {"path": "src/include/storage/bufmgr.h",
         "symbols": ["ReadBufferExtended"]},
    ],
    "boundary": "协调共享缓冲区中的页查找/读入/预取；不覆盖替换策略，替换策略另页。",
}
```

`boundary` 是生成期审核元数据，可留在 generator 中，不扩展公共 `_manifest.yml` schema；manifest 输出只包含契约字段。一个机制允许多个 refs 文件；同一文件可作为多个不同机制的 refs，但页的机制边界必须不同并在 boundary 中讲清。

### P2a 盘点步骤

1. 生成现有 332 条审计表：当前 group/feature/title/page_file/refs/path/symbols、源文件职责、跨组重复路径。
2. 对 A–K 每条标记：`保留并机制化`、`合并到 X`、`拆成独立机制 A/B`、`移组`、`删除伪页`、`新增遗漏机制`；每项给出真实源码路径与符号定义/结构依据。
3. 对 A 组重点核实：共享缓冲读/预取、替换策略与后台写回、本地缓冲、FSM、页布局、smgr、md 段文件、文件描述符/同步、BufFile、PGPROC、ProcArray/MVCC snapshot、lock table、关系锁入口、deadlock、LWLock、SSI 谓词锁、WaitEventSet、Latch、共享内存、DSM、shm_mq、sinval、procsignal、异步 IO、AIO worker/后端、read stream、large object。仅保留源码职责能证明为独立的机制；合并后按真实遗漏补入，不为达到 29 强拆。
4. 对 B–K 同样检查跨组重复路径及组内机制重叠；E/F 的节点文件只有在机制有独立输入/输出/状态/生命周期时才可成页，不得按 node 文件数计篇。
5. 汇总每组真实篇数与全量篇数；记录 A 组最终 page_file 列表与 writer shards。
6. 完成只读审计表后，先将结果交用户审阅；用户确认盘点结果后再改 generator 和 manifest。

### P2a 实施结果（已完成，`feature/wiki-p2a`）

盘点后把篇数从 332 降到 **151 个机制页、12 个领域**。这是盘点结果，不是配额：

| 领域 | 机制数 | 领域 | 机制数 |
|---|---:|---|---:|
| A 存储与 IPC | 20 | G 目录与缓存 | 7 |
| B 事务与 WAL | 19 | H 内存与排序 | 5 |
| C 堆表与元组 | 15 | I 复制与高可用 | 10 |
| D 索引族 | 17 | J 服务端与命令面 | 12 |
| E 优化器 | 13 | K SQL 处理与安全 | 11 |
| F 执行器 | 13 | L SQL 类型与内建函数 | 9 |

关键裁决（用户逐项确认）：

1. 共享缓冲池与 backend-local 池**拆两页**（并发语义不同）。
2. Shared Invalidation **传输原语归 A**，缓存对象失效语义归 G。
3. TABLESAMPLE 主归属 **K**（跨 parser/executor/access method 的一条完整机制）。
4. 旧 H 的 `utils/adt` **新建独立领域 L**，H 只留 memory/sort/resource owner。
5. 索引 WAL redo **合并为一页** `index-wal-redo`（redo 通用骨架远多于 AM 差异）。
6. lock + lmgr + deadlock **合并为一页** `lock-manager-deadlock`。

已落地的代码变更：

- 新增 `scripts/wiki/postgres_mechanism_specs.py`：151 条 spec，每条含 `feature_group`／`feature`／`title`／`page_file`／`boundary`／`refs`。这是 manifest 的唯一数据源。
- 重写 `scripts/wiki/gen_postgres_manifest.py`：删除按文件体积选页、按 basename 推 slug/title 的生产路径；新增 `check_specs()` 检查重复 feature、重复 page_file、伪标题、空 boundary、空 refs、缩写路径、非 `src/` 路径、未知分组。
- 重写 `scripts/wiki/test_gen_postgres_manifest.py`：33 项断言，覆盖分组篇数、编号连续性、`page_file` 尾段等于 feature、无「 模块」伪标题、无测试辅助符号、rmgrdesc 唯一归属、AIO 后端合一、local buffer 独立、refs 可跨文件。
- 重生成 `docs/wiki/projects/postgres/_manifest.yml`：151 页 / 12 组。

验证证据（全部实跑）：

```
python gen_postgres_manifest.py --repo-root <cache> --verify --output <manifest>
  → 151 页 12 组；refs 校验失败 0 条
python wiki_manifest_lint.py <manifest>            → PASS
python -m unittest <13 个 wiki 测试模块>            → Ran 312 tests, OK (skipped=1)
```

**范围说明**：这 151 条只覆盖 PostgreSQL；设计文档里的「332 篇」等旧数字不再作为任何验收口径。E–K 与 L 的 refs 也已用同一 Ctags 函数定义表核过，未命中 0。

### P2a 盘点的证据方法（可复现）

| 步骤 | 做法 | 结果 |
|---|---|---|
| 1 | 导出旧 manifest 332 条为审计表，逐条记录 group/feature/title/page_file/refs | 259/332（78.0%）标题是「<文件名> 模块」兜底 |
| 2 | 用 Universal Ctags 6.1.0 以 `kind=function` 扫 `src/backend/**/*.c`，建 18,547 条定义索引 | 旧 943 个 ref 符号中 940 个是同 path 函数定义，3 个不是（宏参数/类型名） |
| 3 | 统计跨组重复 path | 6 个 `rmgrdesc` 文件在 B、C 各出现一次，共 12 个旧页实体 |
| 4 | 按机制边界重新划分，合并/删除/移组/新增 | 332 → 151；rmgrdesc 归 1 页；旧 E 39→13；旧 F 41→13 |
| 5 | 对新清单逐条核 refs | 151 页全部 `(path, symbol)` 命中定义，0 未命中 |

Ctags 只能证明「该路径里有这个函数定义」（含 static），**不能**证明它是该页机制的核心入口；机制归属由 spec 的 `boundary` 描述并受人工审阅。


---

## P2b：重盘后 A 组机制正文试点

A 组已定稿为 **20 个机制页**（旧 A 组 29 页 → 20 页），文件与机制如下。writer 不得改 feature/title/refs：

| # | page_file | feature | 标题 |
|---:|---|---|---|
| 01 | `a01-buffer-pool.md` | `buffer-pool` | 共享缓冲池与替换 |
| 02 | `a02-local-buffer-pool.md` | `local-buffer-pool` | 后端本地缓冲池 |
| 03 | `a03-procarray-snapshot.md` | `procarray-snapshot` | 活跃事务集合与 MVCC 快照 |
| 04 | `a04-ssi-predicate-locks.md` | `ssi-predicate-locks` | SSI 谓词锁与冲突检测 |
| 05 | `a05-lock-manager-deadlock.md` | `lock-manager-deadlock` | 常规锁与死锁检测 |
| 06 | `a06-fd-cache-file-io.md` | `fd-cache-file-io` | 虚拟文件描述符与文件 I/O |
| 07 | `a07-backend-proc-slots.md` | `backend-proc-slots` | 后端进程槽 |
| 08 | `a08-smgr-md-storage.md` | `smgr-md-storage` | 关系物理存储（smgr 与 md） |
| 09 | `a09-wait-event-sets.md` | `wait-event-sets` | 等待事件集 |
| 10 | `a10-lwlocks.md` | `lwlocks` | 轻量锁 LWLock |
| 11 | `a11-read-stream-prefetch.md` | `read-stream-prefetch` | 扫描预读流 |
| 12 | `a12-shared-memory-allocation.md` | `shared-memory-allocation` | 主共享内存分配 |
| 13 | `a13-dynamic-shared-memory.md` | `dynamic-shared-memory` | 动态共享内存 DSM |
| 14 | `a14-shared-memory-message-queues.md` | `shared-memory-message-queues` | 共享内存消息队列 |
| 15 | `a15-aio-io-methods.md` | `aio-io-methods` | 异步 I/O 请求与执行后端 |
| 16 | `a16-free-space-map.md` | `free-space-map` | 空闲空间映射 FSM |
| 17 | `a17-temporary-buffiles.md` | `temporary-buffiles` | 临时 BufFile |
| 18 | `a18-process-signals-barriers.md` | `process-signals-barriers` | 进程信号与屏障 |
| 19 | `a19-parallel-query-dsm.md` | `parallel-query-dsm` | 并行查询 DSM 基础设施 |
| 20 | `a20-shared-invalidation.md` | `shared-invalidation` | 共享失效消息传输原语 |

### Writer 分片（4 个，互不重叠）

| Writer | 页 | 主题 |
|---|---|---|
| W1 | `a01`–`a05` | 缓冲池、本地缓冲、事务快照、SSI、常规锁与死锁 |
| W2 | `a06`–`a10` | 文件 I/O、进程槽、物理存储、等待事件、LWLock |
| W3 | `a11`–`a15` | 预读流、共享内存、DSM、消息队列、AIO |
| W4 | `a16`–`a20` | FSM、BufFile、进程信号、并行 DSM、失效消息 |

- 只写 P2a 最终 manifest 标为新 A 组的页。
- writer 只可写分配的 `.md`；不得自行变更 feature/title/refs 或改 `_manifest.yml`。
- 主代理按页运行 `wiki_lint`、`wiki_refs`；每页双绿后记录完成。

### Writer 单页模板

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
<准确一句话，说明所管理的机制状态/能力。>

## 机制是什么
<解决的问题、明确的边界、不负责什么、与上下游关系。>

## 怎么实现的
<按真实执行顺序解释关键流程、核心数据结构、并发与生命周期；3–8 个步骤。>

## 关键取舍
<至少两项可由源码/官方材料佐证的取舍、被放弃方案及代价。>

## 源码落点
| 文件 | 符号 | 本文对应作用 |
|---|---|---|
<逐条解释 refs 符号，不能把符号列表当正文。>

## 与主核 / AI 的关系
<给出真实对应或明确“暂无直接对应”，不硬套。>

## 参考
- [标题](https://...) —— 支撑的具体事实
- [标题](https://...) —— 支撑的具体事实
```

正文证据规则：
- 每个核心论断从真实源码与权威文档核实；不从符号名猜语义。
- 跨文件机制可在 manifest refs 里引用多个文件，逐个文件逐个符号硬校验。
- 允许引用真实 static 函数，但说明其内部职责；测试 harness / 测试专用函数不能作为唯一 refs。
- 数值要追溯到代码常量/配置源，不确定就不写。
- 论文仅在论证机制依据该论文时引用；不泛泛填充。

## P2c：双门禁与逐页修复

单页命令：

```powershell
cd C:\code\projects\scripts\wiki
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres\<page>.md --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres\<page>.md `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

整组验收：

```powershell
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

失败修复只改 FAIL 页；refs 失败先用 `wiki_refs.py --locate SYMBOL --repo-root ...` 定位；禁止放宽门禁。整组门禁连续通过一次后记录输出摘要、清理临时日志。

## P2d：深度抽检、_index 与试点报告

用 `random.Random(20261122)` 从新 A 组固定随机抽 5 页。每页检查 6 项：机制边界、实现映射、至少一个数据结构/生命周期/并发解释、至少两项真实取舍、至少两个权威且相关的外链、RAG 切块独立性。每项 0/1；返工后该页需 6/6。

新 A 组双门禁全通过且 5 页抽检通过后，单一作者生成 `_index.md`：`page_type: overview`、契约要求字段齐全、导航表覆盖新 A 组全部机制页、章节契约完整并通过 `wiki_lint`。

写 `projects/docs/wiki/p2-pilot-report.md`，记录新 A 组机制页数量、各 writer 实际起止时间、wall-clock、一次通过页数、返工页数/轮次、抽检详情、页长分布、token/成本真实观测（无 usage 则写“不可观测”）、新全量机制数，以及按并发 4/8/16 对剩余机制页的工程估算（标记“估算”，不得称为实测）。

## P2e：验收与后续调度

验收条件：
1. A–K manifest 页数与经批准的 mechanism inventory 完全一致；跨组无重复机制；不要求篇数为 332。
2. `wiki_manifest_lint.py _manifest.yml` PASS。
3. 新 A 组所有页 `wiki_lint` 与 `wiki_refs` PASS；`_index.md` lint PASS。
4. 5 篇深度抽检逐项 6/6，返工全部记录。
5. 全套 unittest：

```powershell
cd C:\code\projects\scripts\wiki
python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint `
  test_wiki_refs test_repo_cache test_wiki_schema_doc test_wiki_catalog `
  test_wiki_manifest test_wiki_manifest_lint test_wiki_tree test_build_catalog `
  test_catalog_additions test_gen_postgres_manifest
```

期望全绿；记录实际测试数和 skip 原因。
6. `.cache/repos/postgres` 被 gitignore 忽略且不在 git status。
7. 试点报告提交于 `projects/docs/wiki/p2-pilot-report.md`。
8. P2 交付完成且 branch review clean 后合并、推送 projects/main；不暂存或覆盖无关用户改动。

## 不在本试点范围

- 不写重盘后新 A 组以外的正文页。
- 不做 P3 全量 token 预算实测；只报告本试点可观测数据与估算。
- 不开发 RAG 应用。
- 不回填 catalog 旧条目占位 stars、不改 `ai/rag`/`ai/agent`/`ai/llm` 口径、不改 `schema_version`。

---

## P2 执行结果（已完成，`feature/wiki-p2a` → projects/main）

### P2a：机制盘点与 manifest 重建 —— 完成

盘点结论、裁决、代码变更与证据方法见上文「P2a 实施结果」。要点复述：

- 篇数 **332 → 151**，新增第 12 组 `L-SQL类型与内建函数`。
- 新增 `projects/scripts/wiki/postgres_mechanism_specs.py`（151 条 spec，唯一数据源）。
- 重写 `gen_postgres_manifest.py`（由 spec 生成 + `check_specs()` 自检）与 `test_gen_postgres_manifest.py`（33 项断言）。
- 重生成 `_manifest.yml`：151 页 / 12 组；604 个 `(path, symbol)` ref 对全部命中，0 失败。
- 合并提交 `058673f6d`，已推送 `projects/main`。

### P2b：A 组 20 页正文 —— 完成

4 个 disjoint writer 分片（W1–W4，各 5 页）写完 `a01`–`a20`。
页面路径、slug、标题、refs 与 `_manifest.yml` 逐页深度相等（20/20 一致）；
UTF-8 无 BOM、LF；每页 7 个契约 H2；无 `path:line` 引用。

### P2c：双门禁 —— 完成

```text
python wiki_lint.py docs/wiki/projects/postgres --quiet
  -- 检查 21 页，失败 0 页

python wiki_refs.py docs/wiki/projects/postgres \
  --repo-root .cache/repos/postgres --quiet
  -- 检查 21 页，失败 0 页
```

（21 页 = 20 个机制页 + `_index.md`。）失败修复只改 FAIL 页，未放宽门禁。

### P2d：深度抽检、`_index.md`、试点报告 —— 完成

固定 seed `random.Random(20261122)` 抽出 a19 / a17 / a08 / a13 / a02 五页，
按 6 项评分表逐项复核。初审 **27/30**，修复后 **29/30**；
a19 的「取舍」项因缺少具体 caller 的 fallback 证据保留 0，未凑满分。

`_index.md` 按 `page_type: overview` 生成（契约字段齐全 + refs），通过 lint 与 refs 两道门禁。
试点报告见 `projects/docs/wiki/p2-pilot-report.md`（含各分片时长、抽检分项、扩容估算）。

### P2e：验收 —— 完成

```text
python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint \
  test_wiki_refs test_repo_cache test_wiki_schema_doc test_wiki_catalog \
  test_wiki_manifest test_wiki_manifest_lint test_wiki_tree test_build_catalog \
  test_catalog_additions test_gen_postgres_manifest test_postgres_mechanism_pages
  Ran 315 tests ... OK (skipped=1)
```

提交与推送：`a7a86b1ce`（A 组 20 页 + `_index.md` + 试点报告 + 契约测试）、
`e98be5f15`（外链修正 + a19 退化语义收窄）、`d411d929a`（抽检结论修订）。
`.cache/` 被 gitignore 忽略，未入库；`engine/db` 下的用户改动全程未暂存。

### 遗留（转入下一阶段）

| 项 | 去向 |
|---|---|
| B–L 共 131 页正文 | P3 `2026-11-23-multimodal-wiki-p3-postgres-full.md` |
| a19 取舍证据补足 | P3 写作同主题页时补具体 caller 证据并回填评分 |
| 其余库流式推进 | P4 `2026-11-23-multimodal-wiki-p4-other-libraries.md` |
| RAG 应用 | P5 `2026-11-23-multimodal-wiki-p5-rag-app.md` |
| 清单 152 → 270 条 | P4 扩容批次，逐条实取 stars |
