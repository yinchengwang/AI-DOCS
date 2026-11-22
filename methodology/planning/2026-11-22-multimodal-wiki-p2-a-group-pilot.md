# P2 PostgreSQL A 组试点（清单修订 + 29 篇正文）实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 先修正 P1 机械生成造成的 A–K 机制边界与 refs 质量问题，再完成 PostgreSQL A 组 29 篇正文试点，双门禁通过、深度抽检合格后产出 `_index.md`，并记录实际耗时、返工率与一次通过率。

**Architecture:** P2a 将 `gen_postgres_manifest.py` 从“一个 `.c` 文件强行等于一篇”改为“manifest 条目明确表示一个机制单元，refs 可包含同一机制涉及的多个文件与符号”；保持已批准的 332 篇总量及 A 组 29 篇数量不变。P2b 严格按 manifest 写正文，4 个 writer 负责互不重叠的文件分片；P2c 主代理运行 `wiki_lint` + `wiki_refs` 硬门禁并逐页修复；P2d 随机抽 5 页做人工深度审阅并生成项目 `_index.md`；P2e 汇总试点数据，合并推送并据真实指标安排剩余 303 页。

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
- P2a 保持总 manifest 332 页、A 组 29 页、A–K 每组篇数不变；只可重划单页覆盖的文件集合、修订 slug/title/refs。若无法在总数固定下满足独立机制边界，停止并提交证据，不私自拆并篇数。
- P2b 每篇 front-matter 必须包含 `project`、`feature`、`page_type`、`schema_version`、`refs`；A 组 `page_type: feature`，`schema_version: 1`。
- 参考章节必须至少有 2 个 HTTP(S) 外链；优先 PostgreSQL 官方文档、PostgreSQL 官方源码仓/官方开发文档、原始论文/标准。引用不明来源不得编造。
- 浅克隆缓存路径 `projects/.cache/repos/postgres` 属运行时缓存，永不入库。
- 所有 writer 只能写其明确分配的 `.md` 文件；不得改 `_manifest.yml`、`_index.md` 或契约文件。

---

## 文件与写入范围

| 文件/目录 | P2 阶段责任 |
|---|---|
| `projects/scripts/wiki/gen_postgres_manifest.py` | P2a 修复机械生成规则，显式表达机制边界与多文件 refs |
| `projects/scripts/wiki/test_gen_postgres_manifest.py` | P2a 增加 manifest 质量/每组数量/引用真实命中测试 |
| `projects/docs/wiki/projects/postgres/_manifest.yml` | P2a 重新生成同样 332 页，A 组 29 页；P2b/P2c/P2d 只读 |
| `projects/docs/wiki/projects/postgres/a01-*.md` 至 `a29-*.md` | P2b writer 分片写入，每个文件只归一个 writer |
| `projects/docs/wiki/projects/postgres/_index.md` | P2d 单一作者在 29 页双门禁通过后生成 |
| `projects/docs/wiki/p2-pilot-report.md` | P2d 记录 29 页实测吞吐、一次通过率、返工数、token/成本估算、5 页抽检结论 |

P2 不改 `wiki_schema.py`、`wiki_frontmatter.py`、`wiki_lint.py`、`wiki_refs.py` 的行为；发现契约 bug 时暂停并单独建修复任务，避免写作中途漂移。

## 明确决策

1. **推进节奏：** A 组 29 页试点；试点验收并记录真实数据后，再确定其余 303 页并行度。
2. **写作对象：** 以 manifest 条目定义的机制单元为准；标题/slug/refs 由 P2a 在写作前修正，writer 不得擅改。
3. **试点收尾：** 29 页全部双门禁通过后生成 PostgreSQL `_index.md`。
4. **并行策略：** 4 个 disjoint writer 分片。分片依文件编号预先固定，不按模型或符号动态抢文件。
5. **质量：** 机械双门禁 + 随机抽 5 篇深度复核。抽检维度：机制边界、实现流程/数据结构/生命周期、关键取舍、源码落点是否解释实际作用、参考质量、RAG 可切块性。
6. **正文的 A 组最小深度结构：**
   - 机制是什么：解决的问题、明确不解决的边界、上下游关系。
   - 怎么实现的：按执行顺序讲 3–8 个步骤；每步说明关键函数/结构如何协作；至少解释一条并发/生命周期路径。
   - 关键取舍：至少 2 项真实设计取舍，并说明替代方案影响。
   - 源码落点：每个表格行写 path + symbol + 作用；无行号。
   - 参考：至少 2 个权威外链，且确实支撑本文机制论述。
7. **禁止灌水：** 不允许为了篇幅堆源码符号清单、复制官方介绍、写“代码通过检查”等无信息段落；篇长由机制复杂度决定。

## P2a：机制边界清单修订

**目标：** 当前生成器按“文件体积降序取 N 个 `.c` 文件”导致标题为 `method_worker 模块`、refs 为 `pgaio_worker_pm_test_*` 的文件级清单。先把 332 页转成稳定的机制单元定义。

### P2a 文件

- Modify `projects/scripts/wiki/gen_postgres_manifest.py`
- Modify `projects/scripts/wiki/test_gen_postgres_manifest.py`
- Regenerate `projects/docs/wiki/projects/postgres/_manifest.yml`

### 接口约束

- `GROUP_SPECS` 仍为 11 组，标签逐字为 A–K，计数仍为 `29/28/35/33/39/41/28/28/22/35/14`。
- 每个 page spec 仍为 `feature`、`feature_group`、`title`、`page_file`、`refs`。
- `refs` 可含多个文件对象。每个文件对象仍为 `{"path": str, "symbols": list[str]}`。
- 同一机制可以引用多个实现文件（例如 bufmgr 主流程 + 本地缓冲路径），但每篇必须有一个明确主机制，不能把多个无关文件拼成一页。
- `feature` 是全局主键，slug 应命名机制，不直接机械地把 `.c` basename 改横线；允许同一源文件以不同机制页为证据，但同一个机制不得重复建页。

### P2a 验收测试（TDD）

新增/改造测试覆盖：

```python
def test_group_counts_remain_exact():
    self.assertEqual({g["label"]: g["count"] for g in GROUP_SPECS}, EXPECTED_COUNTS)

def test_a_group_has_29_mechanism_specs():
    pages = build_manifest(POSTGRES_ROOT)["pages"]
    a_pages = [p for p in pages if p["feature_group"] == "A-存储基础"]
    self.assertEqual(len(a_pages), 29)

def test_mechanism_slugs_are_unique_and_not_raw_file_list():
    pages = build_manifest(POSTGRES_ROOT)["pages"]
    slugs = [p["feature"] for p in pages]
    self.assertEqual(len(slugs), len(set(slugs)))
    # 具体断言聚焦已知问题：不是 `method-worker` / `dsm-impl` 等文件名兜底。
    self.assertNotIn("method-worker", slugs)
    self.assertNotIn("dsm-impl", slugs)

def test_all_refs_hit_real_files_and_symbols():
    manifest = build_manifest(POSTGRES_ROOT)
    self.assertEqual(verify_refs(manifest, POSTGRES_ROOT), [])
```

- `build_manifest()` 必须能从 manifest spec 直接表达 1 个机制多文件 refs；不再把“文件个数”硬编码成“篇数”。
- 真实 source scan 的每个 refs path 存在且 symbols 全部命中；`wiki_manifest_lint.lint_manifest` 0 错误。
- 所有 332 个 page_file 唯一、feature 唯一、filename tail 与 feature 一致。
- 逐组核对 A 组 29 页的机制边界是否互不重复、无“文件名+模块”兜底标题；其余组只确认计数与生成接口兼容，不在 P2a 重写 303 条机制标题。

### P2a 分步

1. 为 A 组已知问题写 RED 测试（`method-worker`、`dsm-impl` 机制化，不再把测试辅助符号当主题）。
2. 运行 `python -m unittest test_gen_postgres_manifest -v`，确认 RED 指向 generator 质量断言。
3. 重构 generator 的数据层：按 `PAGE_SPECS` 显式列机制 slug/title/refs；允许 refs 多文件；生成逻辑不得按 source file 体积替代机制划分。
4. 为 A 组 29 页逐条验证 refs；批量运行真实 `_scan_symbols`，任何失败必须改 refs 到真实定义文件，不放宽校验。
5. 更新 manifest 并运行 `wiki_manifest_lint.py <_manifest.yml>`。
6. `python -m unittest test_gen_postgres_manifest -v` 全绿；再运行 P0/P1 全量 unittest。
7. 提交中文 commit（仅生成器、测试、manifest）。

**P2a stop condition：** 若 332 的固定篇数与“一个机制一篇”无法同时成立，或 A 组 29 个机制无法经真实源码证据区分，停止并报告冲突，不用文件数凑数。

---

## P2b：A 组 29 页正文写作

### Writer 分片（固定且不重叠）

| Writer | 页范围 | 主题集中区 |
|---|---|---|
| W1 | `a01`–`a07` | 缓冲池、共享状态、快照、段存储、文件资源 |
| W2 | `a08`–`a14` | 等待事件、LWLock、预读、恢复冲突、页面、共享内存、DSM |
| W3 | `a15`–`a21` | 共享消息队列、AIO、死锁、smgr、锁层次、AIO worker |
| W4 | `a22`–`a29` | 本地缓冲、FSM、BufFile、信号、大对象、失效广播、io_uring、Clock-Sweep |

Writer 接收其确切 7 或 8 个 page spec、PostgreSQL 源码目录、P0 契约和本计划；不得自行加页、删页、改 feature、改 page_file、改 refs。

### Writer 单页模板

每页独立 Markdown，front-matter 先于正文。以下为格式骨架，正文不允许保留括号里的提示词：

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
<准确一句话；指出本机制管理什么状态/提供什么能力，不写营销描述。>

## 机制是什么
<问题、边界、不负责什么、与相邻机制的上下游。>

## 怎么实现的
<按真实执行顺序解释关键流程；用 Mermaid/ASCII 图仅在能降低理解成本时使用。>

## 关键取舍
<至少两项由源码或官方资料可证实的取舍；列替代方案及代价。>

## 源码落点
| 文件 | 符号 | 本文对应作用 |
|---|---|---|
<逐条解释 refs symbols 的落点与上下文，不写行号。>

## 与主核 / AI 的关系
<与 `engine/db` 的对应机制或可迁移设计；没有可靠对应时写“暂无直接对应”，不硬套。>

## 参考
- [标题](https://www.postgresql.org/docs/current/...) —— 此来源支撑哪项事实
- [标题](https://git.postgresql.org/...) —— 此来源支撑哪项事实
```

### 写作证据规则

- 机制事实从本地浅克隆源码与 PostgreSQL 官方文档提取；不得把符号名字本身当成语义证据。
- 页面若解释跨文件流程，可追加同机制文件到 refs，但需保证符号确实在其路径对应文件内。
- 引用 `static` 函数允许（门禁按词边界检查），但“源码落点”要说明它是内部实现函数。
- 大量测试辅助函数、测试 harness、宏名不能当机制页唯一 refs；若机制是宏/inline，refs 指向定义头文件。
- 每个数字（上限、缓冲数、阈值）必须指出其配置源或代码常量；无法确认就不写精确数字。
- 论文只在机制确实依据该论文时引用；不以泛泛论文充数。

### P2b 分步

1. 主代理从 manifest 提取 W1–W4 各自准确 JSON/page-spec 清单，存到任务临时根目录并在 dispatch 提示内直接附上，不把整个仓复制给 writer。
2. 同时启动 4 个 writer，每个只写自己的页文件范围（具体 29 路径在 dispatch 里展开）。
3. 每个 writer 汇报真实起止时间、页数、每页是否读源码/官方资料、引用 URL 列表；不得声称 token 数如无供应商 usage 证据。
4. 主代理验证实际新增页路径正好等于 manifest A 组页，不多不少；检查没有 writer 越界改 manifest/_index/别人的页。
5. 未通过门禁前不合入任何 writer commit（分片可各自提交到共享 feature 分支，但每次只 add 自己路径）。

---

## P2c：双门禁与逐页修复

每个页面必须同时通过：

```powershell
cd C:\code\projects\scripts\wiki
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres\a01-bufmgr.md --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres\a01-bufmgr.md `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

整组验收（退出码 0）：

```powershell
python wiki_lint.py C:\code\projects\docs\wiki\projects\postgres --quiet
python wiki_refs.py C:\code\projects\docs\wiki\projects\postgres `
  --repo-root C:\code\projects\.cache\repos\postgres --quiet
```

失败页流程：
1. 只读取该 FAIL 页、对应 manifest 条目及 refs 文件，不批量改其它 PASS 页。
2. lint 错误逐项修正 front-matter/章节/行号/外链。
3. refs 错误先用 `wiki_refs.py --locate SYMBOL --repo-root ...` 定位，再修路径/符号；不能通过添加豁免。
4. 对该页重跑两个 CLI；通过后立即将失败记录移出待修清单。
5. 全组至少连续跑一次双门禁，保存输出摘要到临时日志，随后清理临时日志。

门禁前的人工静态自查（全部 29 页）：
- page_file / feature / refs 与 manifest 完全一致。
- 所有源码表格没有 `:123` 行号形态。
- 全文没有未验证的数字、含糊的“通常/一般”结论无来源。
- 不存在 placeholder、TODO、`待补`、复制模板提示词。

---

## P2d：深度抽检、_index 与试点报告

### 5 篇随机抽检

用 Python 标准库 `random.Random(20261122)` 固定 seed，从 29 页抽 5 篇（可复现）。每篇按 0/1 检查：

1. **机制边界：** 能说清解决/不解决的问题与上下游。
2. **真实实现：** `怎么实现的` 能与 refs 中符号、源码流程对应，不是函数列表。
3. **机制深度：** 至少解释一个核心数据结构及生命周期/并发边界；该机制确无并发时解释为何单线程/为何不需要同步。
4. **真实取舍：** 至少两项可由源码或官方资料佐证的设计选择。
5. **来源质量：** 两个以上外链打开到权威且与具体论述相关的页面。
6. **RAG 可切块：** `##` 章节能独立提供清晰事实，避免同一概念拆散到互相依赖的段落。

抽检允许指出缺陷并要求返工；返工后重新抽检该页，全部 5 篇达到 6/6 才过。记录抽检文件名、分项得分、缺陷、返工次数。

### `_index.md`

29 页双门禁全通过且 5 页抽检通过后，由一个作者单独写 `projects/docs/wiki/projects/postgres/_index.md`：
- `page_type: overview`，字段按 `INDEX_REQUIRED_FIELDS` 精确提供（project/page_type/display_name/repo/license/language/stars/stars_as_of）；
- 六必含章节与条件章节都保留；
- 机制导航表按 A 组编号顺序列出 29 个 feature/title/page_file，并链接相对路径；
- 项目定位/历史/许可证/核心机制范围有官方/上游来源；
- `_index.md` 同样过 `wiki_lint`。overview 是否跑 `wiki_refs` 由门禁行为决定（`refs` 非必填），本 P2 不新增契约要求。

### `p2-pilot-report.md`

记录：
- A 组页数 29、双门禁最终结果、各 writer 起止时间、实际总 wall-clock。
- 一次通过页数、返工页数、返工轮数；不要用“几乎一次通过”等模糊词。
- 5 页抽检明细与分项结果。
- 页面字数/行数/代码块或图表数量分布（仅作观测，不作为新的质量门槛）。
- token/成本：如果工具能提供供应商 usage 则引用真实值；否则标记“不可观测”，不得按经验编造。
- 清单/manifest 修订是否改变过任何 page_file 或 refs，以及变更审计。
- 由此推导剩余 303 篇并发度：用实测平均耗时 × 303，分别估算并发 4/8/16 下 wall-clock；明确该值是工程估算，不是假称实测。

---

## P2e：验收、提交与后续计划

### 全量 P2 验收门

1. manifest A 组仍 exactly 29；A–K 总 332 不变；差异仅限已批准的 P2a mechanism refs/title/slug 修订。
2. `wiki_manifest_lint.py _manifest.yml` PASS。
3. 29 页 `wiki_lint` PASS；29 页 `wiki_refs` PASS。
4. `_index.md` `wiki_lint` PASS。
5. 人工深度抽检 5/5 满分，缺陷/返工有记录。
6. 相关测试命令：

```powershell
cd C:\code\projects\scripts\wiki
python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint `
  test_wiki_refs test_repo_cache test_wiki_schema_doc test_wiki_catalog `
  test_wiki_manifest test_wiki_manifest_lint test_wiki_tree test_build_catalog `
  test_catalog_additions test_gen_postgres_manifest
```

期望：运行全套成功（当前基线 `Ran 313 ... OK (skipped=1)`；P2a 若加测试，记录新总数）。

7. `git status --short` 不应出现 `.cache/`；`git check-ignore -v .cache/repos/postgres` 仍命中。
8. P2 试点报告已写在 `projects/docs/wiki/p2-pilot-report.md`。

### 提交策略

- P2a manifest 修订单独中文 commit，并立即推 `origin/feature/wiki-p2`（若采用 PR 流程）或在最终合并时遵守仓库 main push 纪律。
- 每个 writer 只 stage 自己的文件路径，不得混入其他 writer 页或用户已有改动。
- P2 整体验收前暂不合并至 main；验收后单一 merge commit 或 fast-forward，运行整套门禁后推 `origin/main`。
- 不清理、不暂存、不提交 projects 工作树里与本任务无关的既有改动。

## 不在本试点范围

- 不写 B–K 剩余 303 页。
- 不做 P3 全量资源/token 预算精确校准；只写 A 组可观测数据与估算。
- 不开发 RAG 应用。
- 不回填 97 条旧 catalog 占位 stars、不改 `ai/rag`/`ai/agent`/`ai/llm` 口径、不改变 schema_version。
