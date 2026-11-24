# wiki P4（其余库流式推进）Implementation Plan

> **执行状态（2026-11-24 ~ 2026-11-25，Task 1–5 已执行）**
>
> - **Task 1 泛化 manifest 生成器** —— **完成**。新增 `projects/scripts/wiki/gen_manifest.py`（库无关），5 处 postgres 专用耦合全部参数化；等价性验证通过（用新脚本生成的 postgres manifest 与仓库内 `_manifest.yml` **逐字节相同**）；`gen_postgres_manifest.py` 收敛为薄封装（24 增 / 126 删），行为不变；新增 `scripts/wiki/test_gen_manifest.py`（222 行）。
> - **Task 2 catalog stars 回填** —— **部分完成（150/152）**。新增 `scripts/wiki/fill_stars.py` 与缓存 `scripts/wiki/catalog_stars_cache.json`（96 条记录）；**150 条为真实取数**，`stars_as_of` 为 `2026-11-23`（1 条为 `2026-11-22` 的既有真实值）；2 条未回填并已登记原因（`openGauss` 非 GitHub 托管、`pklite` HTTP 404），**未编造任何数字**。
> - **Task 3 门禁接入 CI** —— **完成**。新增 `projects/.github/workflows/wiki.yml`，4 个 job（`wiki-lint` / `wiki-manifest` / `wiki-refs` / `wiki-tests`）。**修正一处真实缺陷**：原稿 4 个 job 都写了 `working-directory: projects`，在 `projects` 子仓里会变成不存在的 `projects/projects`，CI 必然全 job 失败 —— 已全部删除。
> - **Task 4 三库机制盘点** —— **完成**。redis 63 单元 / 13 组、mysql-server 67 / 14、sqlite 86 / 14。
> - **Task 5 三库页面撰写** —— **完成**。四库合计 **367 篇机制页 + 4 份概览页**，四库各自 lint / refs / manifest 全绿。
> - **Task 6 对比页（`_compare/`）** —— **未做**（按 §6 的口径，首批三库完成后才做样板）。
> - 进度记录（事实来源）：`projects/docs/wiki/p4-progress.md`（批次 0 / 批次 1）。
> - 详见本文 §1.5 / §2.5 / §3.4 / §4.6 / §5.4 的「执行结果」小节。
> - **本计划正文保留为原始执行记录，不按实际执行回改**；已被实测推翻的基线数字在 §0 下方标注。

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

> **口径说明**：下表是 **2026-11-23 计划制定时**的实测基线，是决策依据的历史记录，**保留不改**。
> 表下方的「基线修正（2026-11-25）」列出已被后续实测推翻或更新的项。

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

### 0.1 基线修正（2026-11-25）

上表已被实测推翻或更新的项：

| 项 | 计划时基线 | 2026-11-25 实测 | 依据 |
|---|---|---|---|
| 已落地库 | **1**（postgres） | **4**（postgres / redis / mysql-server / sqlite） | `projects/docs/wiki/projects/` 下四个目录，各有 `_manifest.yml` + `_index.md` |
| 机制页总数 | postgres 151 | **367** 机制页 + 4 份概览页 | postgres 151 / redis 63 / mysql-server 67 / sqlite 86 |
| 源码缓存 | 只有 postgres | postgres / redis / sqlite 为整仓检出；**mysql-server 为文件级稀疏检出**（见 §4.6） | `.cache/repos/<name>` |
| 门禁接入 CI | **未接 CI** | **已接**：`projects/.github/workflows/wiki.yml`，4 个 job | §3.4 |
| postgres 专用工具 | `gen_postgres_manifest.py`（5 处强耦合）+ `postgres_mechanism_specs.py` | **库无关生成器已落地**：`scripts/wiki/gen_manifest.py` 为主，`gen_postgres_manifest.py` 收敛为薄封装（24 增 / 126 删）。规格模块从 1 个扩到 **4 个**（postgres / redis / mysql / sqlite） | §1.5 |
| stars 真实取数 | 55 / 152 | **150 / 152**（2 条取数失败已登记） | §2.5 |
| 索引生成器 | `rebuild_postgres_index.py`（未提及泛化） | 新增 **`scripts/wiki/rebuild_project_index.py`**（库无关，`--project` / `--all`），原脚本收敛为其 postgres 分支 | §5.4 |
| 清单扩容（§2.2 目标 2） | 目标扩到 152 → 最多 270 | **未扩容**，清单仍为 152 条；本阶段只做 stars 回填 | §2.5 |

**未随实测改变的部分**：§2.2 的扩容目标、§5.2 的后续批次排期、§6 的对比页、§4 的 SOP 五步 —— 这些在本阶段**未执行**，仍按原文作为待办保留。

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

### 1.5 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 新建库无关生成器 `projects/scripts/wiki/gen_manifest.py`
- [x] 5 处 postgres 专用耦合全部参数化
- [x] `gen_postgres_manifest.py` 收敛为薄封装，行为不变
- [x] 等价性验证（postgres manifest 逐字节相同）
- [x] 新增 `scripts/wiki/test_gen_manifest.py`

**落地产物**：`projects/scripts/wiki/gen_manifest.py`

**CLI**（与 §1.2 的设计一致）：

```text
python gen_manifest.py --project <name> --spec-module <module> \
  [--src-prefix src[,sql,...]] [--output <path>] [--repo-root <dir>] [--verify]
```

**5 处耦合的参数化结果**（对照 §1.1 的表）：

| # | 原耦合 | 参数化后 |
|---|---|---|
| 1 | 硬编码 `from postgres_mechanism_specs import ...` | `--spec-module` 动态导入，模块须暴露 `MECHANISM_SPECS` 与 `GROUP_ORDER` |
| 2 | `build_manifest()` 写死 `"project": "postgres"` | `--project` |
| 3 | `check_specs()` 强绑 postgres 的 12 组 | `GROUP_ORDER` 由 spec 模块提供 |
| 4 | `refs.path` 必须 `src/` 开头 | `--src-prefix`，**支持逗号分隔多个前缀**（MySQL 需 `sql`+`storage`+`plugin` 等；SQLite 无前缀） |
| 5 | `title` 不得以 `" 模块"` 结尾 | 升级为通用伪标题检测：title 不得等于文件名去扩展名，且不得以「模块」结尾 |

**等价性验证**（这是泛化是否真的等价的关键证据，不是「新脚本能跑」）：

```text
python gen_manifest.py --project postgres --spec-module postgres_mechanism_specs \
  --src-prefix src --output <临时路径>
```

生成的 manifest 与仓库内 `docs/wiki/projects/postgres/_manifest.yml` **逐字节相同**。

**`gen_postgres_manifest.py` 收敛为薄封装**：**24 增 / 126 删**。行为不变 —— 其原有测试（`test_gen_postgres_manifest.py`）与「重新生成后与仓库内 manifest 一致」的验证仍通过，**已有门禁未被破坏**。

**新增测试**：`scripts/wiki/test_gen_manifest.py`（**222 行**），覆盖 spec 自洽、伪标题检测、前缀多值、refs 校验。

**后续影响**：§4 Task 4 / Task 5 的三库（redis / mysql-server / sqlite）**全部用这一个库无关生成器**产出 `_manifest.yml`，无需再复制一份脚本。

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

### 2.5 执行结果（2026-11-24 ~ 2026-11-25）—— **部分完成（150/152）**

- [x] 新增 `scripts/wiki/fill_stars.py`
- [x] 新增缓存 `scripts/wiki/catalog_stars_cache.json`
- [x] 回填 150 / 152 条为真实取数
- [x] 2 条取数失败的条目已登记原因并保持占位（不编造数字）
- [ ] **清单扩容（§2.2 目标 2）未做** —— 清单仍为 152 条，本阶段只做 stars 回填

**落地产物**：`projects/scripts/wiki/fill_stars.py`、`projects/scripts/wiki/catalog_stars_cache.json`

**取数实现**：

| 项 | 实现 |
|---|---|
| 接口 | GitHub REST `https://api.github.com/repos/{owner}/{repo}` |
| 取值 | `stargazers_count` / `license` / `language` |
| 单轮上限 | **55 次/轮**（对应 §2.3 的「未认证限流 60 次/小时」纪律，留出余量） |
| 停止条件 | 遇 **403 / 429** 或响应头 `X-RateLimit-Remaining=0` 即停，不硬扛 |
| 网络错误 | 重试 **3 次** |
| 辅助能力 | `--dry-run`、断点续跑（靠缓存文件） |
| 缓存格式 | `catalog_stars_cache.json`，**96 条**记录，每条带 ISO8601 UTC 的 `fetched_at` |

**回填结果**：**150 / 152 条为真实取数**，`stars_as_of` 为 `2026-11-23`（1 条是 `2026-11-22` 的既有真实值）。两轮实跑：第 1 轮 48 个成功（限流用尽即停），第 2 轮 1 个失败。

**未回填的 2 条 —— 保持占位并登记，不填假数**：

| 条目 | 原因 | 处置 |
|---|---|---|
| `openGauss` | repo 是 `https://gitee.com/opengauss/openGauss-server`，**非 GitHub 托管**，脚本按设计拒绝请求 | 保持 `stars: 0` / `stars_as_of: 1970-01-01` / `language: unknown`。要取真实值需另写 Gitee API 路径，属后续批次 |
| `pklite` | `https://github.com/nicholasgasior/pklite` 返回 **HTTP 404**（仓库已删除或改名） | 保持占位；缓存里留 `status: failed` + `http_status: 404` + `fetched_at`，**可追溯**。换真实 repo URL 或从清单移除属清单维护决策，本阶段未擅改 |

**「真实 0」与「占位 0」的区分**：`db-bench` 取到的是真实 `stars = 0`，`stars_as_of` 为当天，**不是**占位的 `1970-01-01`。这一条已用回归测试 **`test_real_zero_stars_is_not_placeholder`** 固化，防止后续把真实 0 误判成占位再刷一遍。

**测试**：`test_fill_stars.py` 通过；`test_catalog_additions.py` / `test_build_catalog.py` / `test_wiki_catalog.py` 全绿。

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

### 3.4 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 新增 `projects/.github/workflows/wiki.yml`
- [x] 4 个 job 全部落地
- [x] refs job 先 `ensure_repo` 浅克隆，clone 失败即 `exit 1` 并打 `::error::`
- [x] **修正原稿的一处真实缺陷**（见下）
- [ ] `actionlint` 本地不可用 —— 只做了人工核对 + 本地复现全部 4 个 job 的实际命令

**落地产物**：`projects/.github/workflows/wiki.yml`

**4 个 job**（与 §3.2 的表一致）：

| job | 内容 | 触发 |
|---|---|---|
| `wiki-lint` | `wiki_lint.py docs/wiki/projects --quiet` | push / PR |
| `wiki-manifest` | `wiki_manifest_lint.py` 对四库 `_manifest.yml` 逐个跑 | push / PR |
| `wiki-refs` | 先 `repo_cache.ensure_repo` 浅克隆 postgres，再 `wiki_refs.py --repo-root .cache/repos/postgres docs/wiki/projects --quiet` | push / PR |
| `wiki-tests` | `python -m unittest discover -s scripts/wiki -p 'test_*.py'` | push / PR |

触发路径：`docs/wiki/**`、`scripts/wiki/**`、`apps/wiki-rag/**`、workflow 自身。

**refs job 的克隆与缓存**：

- 先 `repo_cache.ensure_repo` 浅克隆 postgres，**再**跑门禁 —— 这是 §3.2「已知冲突」的应对：缺源码时 `wiki_refs.py` 硬失败是设计意图。
- `actions/cache` 键为 `wiki-repo-postgres-<sha>`，避免每次拉全量。
- clone 失败则 `exit 1` 并打 `::error::` 明确报出是哪个库，**不静默跳过**。

**门禁范围必须是 `docs/wiki/projects`**：**不能用 `docs/wiki`** —— 否则 `wiki-schema.md` 与三份报告会被判 FAIL。

**修正一处真实缺陷（重要）**：原稿 4 个 job 都写了 `working-directory: projects`。这是**顶层聚合仓 `C:\code`** 的相对写法，在 `projects` 子仓里会变成不存在的 **`projects/projects`**，**CI 必然全 job 失败**。已把 4 处全部删除，并在文件头注释里写明原因，避免后续有人「按惯例」再加回来。

**本地验证**（按 CI 里的原样命令在仓根执行）：`wiki_lint docs/wiki/projects` → 152 页 0 失败；`wiki_refs --repo-root .cache/repos/postgres docs/wiki/projects` → 152 页 0 失败；`wiki_manifest_lint` → PASS；`unittest discover` → 341 tests OK（批次 0 时点；三库落地后为 353 tests，见 §5.4）。

**未做**：**`actionlint` 本地不可用**。YAML 语法靠人工核对 + 与仓内既有 `ci.yml` 的约定对齐；GitHub 侧的真实跑通需等下一次 push 后的 Actions 记录。本批在本地复现了全部 4 个 job 的实际命令，但**这不等于 CI 实跑通过**。

**遗留**：CI 里 `wiki-refs` job 目前**只浅克隆 postgres**，新增三库的 refs 校验**尚未接入 CI**；mysql 因是文件级稀疏检出，CI 侧需另行处理检出方式（见 §4.6）。

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

### 4.6 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 三库机制盘点完成（redis / mysql-server / sqlite）
- [x] 三份 `_manifest.yml` 由库无关生成器产出，`--verify` 逐条核对 refs（失败 0 条）
- [x] **MySQL 的检出方式与另两库区分开**（文件级稀疏检出），边界已写进规格模块头注释
- [x] `dump_defs.py` 扩展：支持 C++ 成员函数（`Class::method`）与多行返回类型

**盘点结果**：

| 库 | 机制单元 | 组数 | 规格模块 | 清单 | 本地检出方式 |
|---|---|---|---|---|---|
| redis | 63 | 13 | `scripts/wiki/redis_mechanism_specs.py` | `docs/wiki/projects/redis/_manifest.yml` | 整仓检出 |
| mysql-server | 67 | 14 | `scripts/wiki/mysql_mechanism_specs.py` | `docs/wiki/projects/mysql-server/_manifest.yml` | **文件级稀疏检出** |
| sqlite | 86 | 14 | `scripts/wiki/sqlite_mechanism_specs.py` | `docs/wiki/projects/sqlite/_manifest.yml` | 整仓检出 |

**MySQL 的检出方式（必须与另两库区分开 —— 这是本批最重要的诚实性要点）**：

整仓检出在本机**实测不可行**，三条证据：

| 方案 | 实测结果 |
|---|---|
| 整仓 `git clone --depth 1 --single-branch` | **25 分钟 pack 仍为 0 字节** —— 已终止并删除半成品目录 |
| 整仓 tarball | 约 **370 MB** |
| `raw.githubusercontent.com` 单文件并行下载 | 约 **320 KB/s** |

因此改为**文件级稀疏检出**：

1. 先用 GitHub **tree API** 拿到 tag `mysql-8.0.40` 的完整文件清单，用作**导航**；
2. 再只下载清单引用到的 **256 个文件**（**16.8 MB**）；
3. 每个文件与上游 tag **逐字节一致**。

**代价与边界（必须写清，不能含糊）**：**未下载的文件不在本地，因此不能对未列出的路径做 refs 校验。**
这一点与 postgres / redis / sqlite 的整仓检出**不同** —— 那三库可以校验仓内任意路径的 refs，MySQL 只能校验已下载的这 256 个文件对应的路径。该边界已写进 `mysql_mechanism_specs.py` 的头注释，避免后续读者误以为已覆盖全仓。

**工具侧**：`scripts/wiki/dump_defs.py` 扩展到支持 C++ 成员函数（`Class::method`）与多行返回类型的定义抽取。另有逐行切分的定义抽取脚本（临时目录 `code-wiki-p4-mysql/`）用于区分「定义」与「调用点」，避免 PowerShell `Select-String` 造成的误判。

**refs 校验过程中的真实修正（refs 校验不是一次性过的）**：

| 原写法 | 实际（8.0 / 当前版本） |
|---|---|
| `mysql_parse` | 已重构为 `THD::sql_parser` |
| `page_cur_search` | `page_cur_search_with_match` |
| `row_delete_for_mysql` | 公开入口是 `row_delete_for_mysql_using_cursor` |
| `LimitIterator` | 不存在，实际是 `LimitOffsetIterator` |
| `CompositeIterator` | 不存在，实际是 `AppendIterator` / `NestedLoopIterator` |
| `sql/log.cc`（当作 binlog） | 该文件是**慢查询日志**实现；binlog 实现在 `sql/binlog.cc` |
| `sql/auth/acl.cc` | 8.0 改为 `sql/auth/sql_authorization.cc` |
| `src/cluster.c`（Redis 的 `clusterInit` / `clusterProcessPacket`） | 实际在 `src/cluster_legacy.c` |
| `sqlite3BtreeMoveto`（SQLite） | 不存在，实际是 `sqlite3BtreeTableMoveto` |

**最终 MySQL refs 校验失败 0 条。**

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

### 5.4 执行结果（2026-11-24 ~ 2026-11-25）—— **首批三库完成**

- [x] 首批三库（sqlite / mysql-server / redis）机制页全部写完并双绿
- [x] 四库各有 `_index.md`（`page_type: overview`，含完整导航表）
- [x] 索引生成器泛化：`scripts/wiki/rebuild_project_index.py`
- [x] 索引落点的三个真实修正
- [x] 全套 unittest：`Ran 353 tests ... OK (skipped=1)`
- [ ] 后续批次（§5.2）未排

**页面产出与门禁结果**：

| 库 | 机制页 | `_index.md` | 门禁结果 |
|---|---|---|---|
| redis | 63 | 有 | lint 64 页 0 失败 / refs 64 页 0 失败 / manifest PASS |
| mysql-server | 67 | 有 | lint 68 页 0 失败 / refs 68 页 0 失败 / manifest PASS |
| sqlite | 86 | 有 | lint 87 页 0 失败 / refs 87 页 0 失败 / manifest PASS |
| postgres（复核） | 151 | 有 | lint 152 页 0 失败 / refs 152 页 0 失败 / manifest PASS |

**四库合计 367 篇机制页 + 4 份概览页。**
（每库的「lint/refs 页数」比机制页数多 1，是把该库的 `_index.md` 一并计入 —— 概览页同样受门禁约束。）

**索引生成器泛化**：新增 `projects/scripts/wiki/rebuild_project_index.py`，把原 `rebuild_postgres_index.py` 的库专用逻辑抽成 **`LIBRARIES` 表**（概览正文 + 落点 + 参考按库配置）。**导航表仍完全由 `_manifest.yml` 驱动** —— 手写导航表会在页数变动时漂移。支持 `--project <名>` 与 `--all`。

**索引落点的三个真实修正**：

| 问题 | 修正 |
|---|---|
| `wiki_refs` 按**词边界**匹配，多符号用 `、` 连接会被当成**单个符号** | 改为 `, ` 分隔 |
| `serverMain` 在 Redis **unstable 分支不存在** | 实际是 `main` |
| `sqlite3BtreeMoveto` 在 SQLite 中**只是注释里提到的名字** | 实际是 `sqlite3BtreeTableMoveto` |

**工具链与测试**：

| 项 | 结果 |
|---|---|
| `scripts/wiki/gen_manifest.py` | 已能生成 redis / mysql-server / sqlite 三份清单，`--verify` 逐条核对 refs（失败 0 条） |
| `scripts/wiki/wiki_lint.py` | 四库目录级全过 |
| `scripts/wiki/wiki_refs.py` | 四库目录级全过（含 `_index.md`） |
| `scripts/wiki/wiki_manifest_lint.py` | 四份清单全 PASS |
| 全套 unittest | `py -3 -m unittest discover -s scripts/wiki` → **Ran 353 tests，OK (skipped=1)** |

**进度记录**：`projects/docs/wiki/p4-progress.md` 已追加「**批次 1**」章节（批次 0 记录基础设施，本批记录首批三库）。

---

## 6. Task 6：对比页（`_compare/`）

设计文档 §3 提到 `page_type: compare` 的跨库对比页。**本阶段只做机制，不批量做对比页。**

- 首批三库完成后，若某主题在多库都有机制（如「缓冲池」「MVCC 可见性」「WAL 写入」），
  可产出 1–2 篇对比页作为样板，验证 `compare` 页的目录约束（必须在 `_compare/` 下）。
- 对比页同样受双门禁约束：六个必含章节 + ≥2 外链 + 禁止行号。
- 样板通过前不批量生产。

### 6.1 执行结果（2026-11-25）—— **未做**

- [ ] 对比页（`_compare/`）未产出

首批三库已完成，但**对比页样板没有做**。原文的口径是「首批三库完成后……可产出 1–2 篇对比页作为样板」——「可产出」是许可而非本阶段的硬性交付项，本阶段据此未做。后续补做时须同时满足：目录约束（必须在 `_compare/` 下）+ 双门禁（六个必含章节 + ≥2 外链 + 禁止行号）。

---

## 7. 验收条件

> 下表在保留原文验收项的基础上，补上 2026-11-25 的实际达成情况。

| # | 验收条件 | 实际结果 |
|---|---|---|
| 1 | `gen_manifest.py` 库无关化完成；`gen_postgres_manifest.py` 与现有测试仍成立 | **达成** —— 5 处耦合参数化；薄封装 24 增 / 126 删；postgres manifest 逐字节等价（§1.5） |
| 2 | 清单扩容完成，占位 stars 回填完成（失败项有明确登记） | **部分达成** —— stars 回填 **150/152**，2 条失败已登记原因（§2.5）；**清单扩容未做**，仍 152 条 |
| 3 | CI workflow 存在且能跑通 wiki 门禁 | **部分达成** —— `wiki.yml` 存在，4 个 job 落地，本地复现全部命令通过；**未经 GitHub Actions 实跑**（`actionlint` 本地不可用）（§3.4） |
| 4 | 首批 3 库各自：manifest 生成、refs verify 0 失败、正文双绿、`_index.md` 归档 | **达成** —— 三库 + postgres 复核全部双绿，4 份 `_index.md` 齐备（§5.4） |
| 5 | 每库至少抽 1 页深度评分并记录 | **未在本计划内记录** —— 深度抽检记录在 P3 的 `p3-full-report.md` 与各批次审阅报告中；本阶段未另起抽检记录 |
| 6 | 全套 unittest 全绿（记录实际数与 skip 原因） | **达成** —— `Ran 353 tests ... OK (skipped=1)`（§5.4） |
| 7 | `.cache/` 仍被 gitignore 忽略 | **达成** —— `.cache/repos/<name>` 仍为运行时缓存，未入库 |
| 8 | 每批次独立 commit + push，commit message 用中文；只提交 Wiki 与工装相关路径 | **达成** —— 批次 0 / 批次 1 各自独立 commit + push，中文 message |
| 9 | 每批结束更新 `projects/docs/wiki/` 下的进度记录（新增 `p4-progress.md`，按批次追加） | **达成** —— `p4-progress.md` 含「批次 0」与「批次 1」两章（§5.4） |

---

## 8. 不在本阶段范围

- 不做 RAG 应用（P5）。
- 不做「按文件批量生成」的兜底路径 —— 没有机制盘点就不写正文。
- 不改 `wiki_schema.py` 的契约版本（`schema_version` 仍为 1）。
- 不碰主核 `engine/db`；RAG 栈与主核隔离这条决策在 P5 落地。

---

## 9. 遗留项（本阶段结束时仍未解决）

| # | 遗留项 | 性质 | 说明 |
|---|---|---|---|
| 1 | **对比页（`_compare/`）未做** | 待办 | §6.1。首批三库已完成，样板对比页尚未产出 |
| 2 | **CI 的 `wiki-refs` job 只浅克隆 postgres** | 待办 | 新增三库的 refs 校验尚未接入 CI；**mysql 因是文件级稀疏检出，CI 侧需另行处理检出方式**（§3.4 / §4.6） |
| 3 | `openGauss` / `pklite` 两条 stars 占位 | 清单维护决策 | Gitee 托管需另写 API 路径；`pklite` 仓库 404，需换 URL 或从清单移除（§2.5） |
| 4 | `actionlint` 未装 | 工具缺口 | workflow 只做了人工核对 + 本地命令复现，**未经 GitHub Actions 实跑**（§3.4） |
| 5 | MySQL 检出为文件级稀疏检出 | 能力边界 | 扩展机制面（如补 MVCC 内部、DDL、复制拓扑更多细节）前，需先补下载对应源文件；**未下载的文件不能做 refs 校验**（§4.6） |
| 6 | 清单扩容（§2.2 目标 2）未做 | 待办 | 清单仍为 152 条，未向设计目标（最多 270）扩容 |
| 7 | 后续批次（§5.2）未排 | 待办 | 首批三库双绿后，尚未按 §5.2 的判据排定下一批 |
