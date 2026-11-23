# wiki P4（其余库流式推进）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 P3 验证过的「盘点 → 清单 → 正文 → 门禁 → 归档」流水线推到清单里剩余的库，逐步扩大语料规模，同时把 postgres 专用的生成器改造成库无关工具，并让门禁进入 CI 形成可追溯证据。

**Architecture:** 每个库是一个独立单元，走五步：浅克隆 → 机制盘点 → 生成 manifest → 批量写正文 → 双门禁 + 抽检 + 归档。库之间可并行，库内部按组分片并行。首批选 2–3 个库验证流水线泛化性，再按批次推进其余库。

**Tech Stack:** Markdown + front-matter 受控子集、Python 3.11 标准库 `unittest`（工装层）、`wiki_lint.py` / `wiki_refs.py` / `wiki_manifest_lint.py` / `repo_cache.py`、Git 浅克隆、GitHub REST API（可选，取 stars）。

设计规格：`2026-11-22-multimodal-wiki-and-rag-design.md`
前置计划：P1（清单与工装）、P2（postgres 机制盘点与 A 组试点）、P3（postgres 剩余 131 页）

## Global Constraints

- 沿用 P2/P3 的全部契约约束：六个必含章节 + 一个条件章节；禁止行号引用；refs 路径基准为**该项目仓根**；符号校验硬失败；参考章节 ≥2 个可达外链；UTF-8 无 BOM、LF；commit message 用中文。
- **机制优先，不按文件数折算**：每个库的篇数由该库自己的机制盘点决定。没有盘点结果的库不写正文。
- **不得为凑清单数而新增库条目**：清单扩容必须逐条实取 `stars`/`license`/`language`。
- **写作用域与库隔离**：不同库可以并行，但同一库同一时刻只能有一个 writer 触碰该库的页面。
- **单库双绿才算入库**：一个库没双绿，不开下一个库（流式的污染面原则）。
- `.cache/repos/<name>` 是运行时缓存，永不入库。
- `engine/db/**` 等与本任务无关的用户改动不得暂存、不得覆盖。

---

## 0. 当前事实（实测基线）

| 项 | 实测值 |
|---|---|
| 清单条目 | **152**（T1 39 / T2 85 / T3 28），`catalog-projects.yml` |
| 其中 stars 为真实取数 | **55 条**（2026-11-22 实取） |
| 其中 stars 为占位 `0` / `1970-01-01` | **97 条**（占 64%） |
| 已落地库 | **1**（postgres）；`projects/docs/wiki/projects/` 下只有 `postgres/` |
| 源码缓存 | `.cache/repos/` 下只有 `postgres` |
| 门禁能力 | `wiki_lint` / `wiki_refs` / `wiki_manifest_lint` 均可用；**未接 CI** |
| 可复用工具 | `wiki_schema` / `wiki_frontmatter` / `wiki_manifest` / `wiki_manifest_lint` / `wiki_lint` / `wiki_refs` / `wiki_tree` / `repo_cache` —— **全部与 postgres 无耦合** |
| postgres 专用工具 | `gen_postgres_manifest.py`（5 处强耦合，见 §1.1）、`postgres_mechanism_specs.py` |

---

## 1. Task 1：把 manifest 生成器泛化为库无关

### 1.1 现状：`gen_postgres_manifest.py` 的 5 处 postgres 专用约束

| # | 位置 | 约束 | 后果 |
|---|---|---|---|
| 1 | `from postgres_mechanism_specs import ...` | 硬编码模块名与常量名 | 换库要复制整份脚本 |
| 2 | `build_manifest()` | 写死 `"project": "postgres"` | 同上 |
| 3 | `check_specs()` | `feature_group not in GROUP_ORDER` | 强绑 postgres 的 12 组 |
| 4 | `check_specs()` | `refs.path` 必须 `src/` 开头 | **SQLite（无 `src/`）、MySQL（`sql/`+`storage/`）直接判 FAIL** |
| 5 | `check_specs()` | `title` 不得以 `" 模块"` 结尾 | postgres 专项的 P1 遗留规则 |

可复用（库无关）：`wiki_manifest` 读写、`wiki_manifest_lint` 门禁、`check_specs` 的通用子规则（feature 唯一与正则、page_file 唯一与 `.md`、title/boundary 非空、refs/symbols 非空）、`verify_refs` 的词边界语义、`wiki_lint`/`wiki_refs` 全部门禁、`wiki_tree.list_module_dirs`。

### 1.2 改造目标

新建库无关模块 `projects/scripts/wiki/gen_manifest.py`：

```python
# projects/scripts/wiki/gen_manifest.py
# -*- coding: utf-8 -*-
"""库无关的机制 manifest 生成器。

- 机制条目从外部 spec 模块导入（每个库一个模块），
  由 --spec-module 指定，模块须暴露 MECHANISM_SPECS 与 GROUP_ORDER。
- project 名由 --project 指定。
- refs 路径前缀约束由 --src-prefix 指定（默认 src），允许逗号分隔多个前缀。
"""

def load_specs(spec_module: str) -> tuple[list[dict], list[str]]: ...
def check_specs(specs, group_order, src_prefixes) -> list[str]: ...
def build_manifest(project, specs, group_order) -> dict: ...
def verify_refs(manifest, repo_root) -> list[str]: ...
def main(argv=None) -> int: ...
```

CLI：

```text
python gen_manifest.py --project <name> --spec-module <module> \
  [--src-prefix src[,sql,...]] [--output <path>] [--repo-root <dir>] [--verify]
```

**保留** `gen_postgres_manifest.py` 作为薄封装（或直接改名为调用 `gen_manifest` 的薄壳），
使现有 `test_gen_postgres_manifest.py` 仍然成立 —— **不破坏已有门禁**。

### 1.3 库无关化后的规则调整

| 规则 | 调整 |
|---|---|
| `refs.path` 必须 `src/` 开头 | 改为「必须以该库配置的源码前缀之一开头」，前缀由 spec 模块声明 |
| `title` 不得以 `" 模块"` 结尾 | 升级为通用规则：**title 不得等于文件名去扩展名**，且不得以「模块」结尾（伪标题检测） |
| `feature_group` 必须在 `GROUP_ORDER` 内 | 保持，但 `GROUP_ORDER` 由 spec 模块提供 |
| `feature` 正则 `^[a-z][a-z0-9-]*$` | 保持（通用） |
| `page_file` 形如 `<字母><两位序号>-<feature>.md` | 保持（通用分组编号约定） |

### 1.4 验收

- 新增 `projects/scripts/wiki/test_gen_manifest.py`：至少覆盖「非 postgres project 可生成」「`sql/` 前缀可通过」「`refs.path` 指向仓外被拒」「伪 title 被拒」「跨库 spec 不会串味」。
- 现有 315 项测试仍全绿（新增测试另计）。

---

## 2. Task 2：清单扩容与 stars 回填

### 2.1 现状

152 条里 **97 条 `stars=0` / `stars_as_of=1970-01-01` / `language=unknown`**（占位值，
`build_catalog.py` 注释明确「绝不编造 stars」）。postgres 自身的 `_index.md` 也沿用占位值。

### 2.2 目标

1. **回填**：为 97 条占位条目实取 `stars` / `license` / `language`。
2. **扩容**：把清单从 152 扩到设计目标（最多 270），每条新条目同样必须实取。
3. **口径统一**：所有库的 `_index.md` 用同一套 stars 口径 —— 要么真实值，要么显式标注占位。

### 2.3 取数纪律

- 用 `https://api.github.com/repos/{owner}/{repo}` 实取 `stargazers_count` / `license` / `language`。
- **未认证限流 60 次/小时**：每次运行最多取 55 个，超限即停，脚本支持断点续跑（把已取结果落在
  `projects/scripts/wiki/catalog_additions.py` 的常量里，或单独的 `catalog_fetch_cache.json`）。
- 取不到的项目**不写入清单**，在脚本里留 `# 待取数` 注释。
- `stars_as_of` 写实际取数日，不沿用旧值。

### 2.4 验收

- `catalog-projects.yml` 中不存在 `stars: 0` + `stars_as_of: 1970-01-01` 的条目（或明确登记为「取数失败」并说明原因）。
- `test_catalog_additions.py` / `test_build_catalog.py` / `test_wiki_catalog.py` 全绿。
- 新增测试：每条条目的 `stars_as_of` 不得早于 `2026-11-22`（占位条目除外，且占位条目须有登记）。

---

## 3. Task 3：门禁接入 CI

### 3.1 现状

`projects/.github/workflows/` 只有 `asan.yml` 与 `ci.yml`，**两者都不引用 wiki**。
门禁能力具备但只在本地跑，缺可追溯证据。

### 3.2 目标

新增 `projects/.github/workflows/wiki.yml`：

| job | 内容 | 触发 |
|---|---|---|
| `wiki-lint` | `python wiki_lint.py docs/wiki --quiet` | push / PR |
| `wiki-refs` | `python wiki_refs.py docs/wiki/projects --repo-root .cache/repos/<lib> --quiet`（按已落地库逐个跑） | push / PR |
| `wiki-manifest` | `python wiki_manifest_lint.py docs/wiki/projects/<lib>/_manifest.yml` | push / PR |
| `wiki-tests` | `python -m unittest discover -s scripts/wiki -p 'test_*.py'` | push / PR |

**CI 上的 refs 门禁需要源码**：浅克隆在 CI 里现做（`repo_cache.ensure_repo`），缓存目录用
`actions/cache` 按 `repo-{name}-{sha}` 缓存，避免每次拉全量。

**已知冲突**：CI 缺 postgres 浅克隆时 `wiki_refs.py` 会硬失败（这是设计意图，不是 bug）。
因此 workflow 里必须先 `ensure_repo` 再跑门禁；clone 失败则 job 失败并明确报出是哪个库。

### 3.3 验收

- workflow 文件存在且语法有效（本地用 `actionlint` 或等价 YAML 解析 + 手工核对）。
- 至少有一个 PR 或本地模拟跑通全部门禁。

---

## 4. Task 4：单库推进 SOP

每个库走完这五步才算入库。

### Step 1：浅克隆

```python
from repo_cache import ensure_repo
path = ensure_repo("https://github.com/<owner>/<repo>", "<name>")
```

缓存目录 `projects/.cache/repos/<name>`；已有则直接返回（幂等，不动既有工作树）。
克隆失败会清理半成品目录。

### Step 2：机制盘点（Phase 1，本阶段最重的一步）

**不能按文件数或体积枚举**。P2a 的教训：postgres 按文件生成的 332 篇里 259 篇（78.0%）是伪主题。

盘点做法：

1. 用 `wiki_tree.list_module_dirs` 拿到源码目录骨架，作为**导航**，不作为页的划分依据。
2. 建立**定义索引**：对该库跑 Ctags（`kind=function`）拿到 `(path, name)` 全部定义，作为「符号确实存在」的证据底座。
3. 按**机制簇**归并：先列出该库的顶层机制领域（postgres 是 12 个），再在领域内识别独立机制单元。
4. 逐条确定 **feature slug / title / boundary / refs**，boundary 写「负责什么、不负责什么」——
   这是跨领域去重的判据。
5. 人工 review：跨领域查重复、查遗漏、查伪标题。

产出：`<name>_mechanism_specs.py`（`MECHANISM_SPECS` + `GROUP_ORDER`），结构与 postgres 版一致。

### Step 3：生成 manifest

```powershell
cd C:\code\projects\scripts\wiki
python gen_manifest.py --project <name> --spec-module <name>_mechanism_specs `
  --src-prefix <prefix> --output ..\..\docs\wiki\projects\<name>\_manifest.yml --verify `
  --repo-root ..\..\.cache\repos\<name>
```

`--verify` 逐条校验 refs 是否命中真实源码，0 失败才进入写作。

### Step 4：写正文

分片规则照 P3：**分片不跨领域**，每片 4–10 页；单页流程与 P3 §3 相同（读规格 → 读源码 → 写页 → 单页双门禁 → 外链核验 → 记录）。

### Step 5：双门禁 + 抽检 + 归档

- 整目录 `wiki_lint` + `wiki_refs` 0 失败。
- 每领域抽 1 页做 6 项深度评分（P3 的评分表，含「RAG 切块」项）。
- 生成该库的 `_index.md`（`page_type: overview`，含该库全部页的导航表）。
- 写该库的验收记录，归档。

---

## 5. Task 5：批次编排

### 5.1 首批（验证流水线泛化性）

选 **3 个库**，覆盖三种目录形态，用来暴露泛化问题：

| 库 | 选它的理由 | 预期暴露的问题 |
|---|---|---|
| `sqlite/sqlite` | 源码在根目录，**没有 `src/`** | `refs.path` 前缀规则（§1.3 已处理） |
| `mysql/mysql-server` | 多层 `sql/` + `storage/` + `plugin/` | 领域划分粒度、`plugin/` 边界 |
| `redis/redis` | `src/` 单层，结构接近 postgres | 相对简单，验证基线流程没被过度设计 |

三库并行，各自独立缓存与目录，写作用域互不重叠。

### 5.2 后续批次

首批双绿后再排后续批次。排期依据：

| 判据 | 动作 |
|---|---|
| 一次通过率 ≥ 80% 且抽检无系统性错误 | 批次并发从 3 库提到 5–6 库 |
| 出现 ≥3 处同类系统性问题 | 停下来做定向修正，修正后再排下一批 |
| 抽检「取舍」项连续不达标 | 说明 writer 缺源码证据的读取深度，追加 writer 指引 |

批次之间不重叠：同一批次内不同库可并行，同一库的写作在它自己的窗口内独占。

### 5.3 总量口径

每批结束后更新总量估算：**只报「已盘点库的实际篇数之和」**，不再用线性外推。
外推值在 §2.2 的设计文档里保留为历史对照，一旦实测库数 ≥ 20 就删掉外推行。

---

## 6. Task 6：对比页（`_compare/`）

设计文档 §3 提到 `page_type: compare` 的跨库对比页。**本阶段只做机制，不批量做对比页。**

- 首批三库完成后，若某主题在多库都有机制（如「缓冲池」「MVCC 可见性」「WAL 写入」），
  可产出 1–2 篇对比页作为样板，验证 `compare` 页的目录约束（必须在 `_compare/` 下）。
- 对比页同样受双门禁约束：六个必含章节 + ≥2 外链 + 禁止行号。
- 样板通过前不批量生产。

---

## 7. 验收条件

1. `gen_manifest.py` 库无关化完成；`gen_postgres_manifest.py` 与现有测试仍成立。
2. 清单扩容完成，占位 stars 回填完成（失败项有明确登记）。
3. CI workflow 存在且能跑通 wiki 门禁。
4. 首批 3 库各自：manifest 生成、refs verify 0 失败、正文双绿、`_index.md` 归档。
5. 每库至少抽 1 页深度评分并记录。
6. 全套 unittest 全绿（记录实际数与 skip 原因）。
7. `.cache/` 仍被 gitignore 忽略。
8. 每批次独立 commit + push，commit message 用中文；只提交 Wiki 与工装相关路径。
9. 每批结束更新 `projects/docs/wiki/` 下的进度记录（新增 `p4-progress.md`，按批次追加）。

## 8. 不在本阶段范围

- 不做 RAG 应用（P5）。
- 不做「按文件批量生成」的兜底路径 —— 没有机制盘点就不写正文。
- 不改 `wiki_schema.py` 的契约版本（`schema_version` 仍为 1）。
- 不碰主核 `engine/db`；RAG 栈与主核隔离这条决策在 P5 落地。
