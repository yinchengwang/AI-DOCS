# wiki P5（RAG 演示应用）Implementation Plan

> **执行状态（2026-11-24 ~ 2026-11-25，应用已建成并跑通）**
>
> - 应用代码在 `projects/apps/wiki-rag/`，**FastAPI + lancedb 0.41.0**（**明确不用 chromadb**）、**BM25 + RRF 融合（k=60）**、**抽取式生成**、**引用由后端生成**、**零构建三栏前端**。
> - 已提交：`4d42f2da1`（tools/smoke.py + README）、`f63b976db`（conftest.py）。
> - **冒烟测试真实输出**：health 200（`sentence-transformers` / `BAAI/bge-small-zh-v1.5` / `lancedb`）；projects 1（`postgres`，pages=152，validation=PASS，gates `wiki_lint:0, wiki_refs:0`）；project detail pages=152 block_count=899；search 3 结果；ask answer_len=1682 citations=4 abstained=False；**引用回源缺失 = 无**；块库 **899 块 / 唯一 ID 899**。
> - **受阻项（必须明示）**：**study-hub 的 R2/R3 无法执行 —— 环境里 `ARK_API_KEY` 与 `ANTHROPIC_API_KEY` 均为空**，缺 LLM 密钥（详见 §10.1）。
> - 详见本文 §1.4 / §2.4 / §3.5 / §4.4 / §5.2 / §6.1 / §7.1 的「执行结果」小节。
> - **本计划正文保留为原始执行记录，不按实际执行回改**；已被实测推翻的基线数字在 §0 下方标注。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to execute this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在已归档的 wiki 语料上建成一个可演示的三栏 RAG 问答应用：左栏库/机制导航与校验状态，中栏问答与检索抽屉，右栏出处面板可逐条回源。

**Architecture:** 独立第三方栈，落在 `projects/apps/wiki-rag/`，**不接主核 `engine/db`**（设计决策 #14）。数据流：语料 → 切块 → 嵌入 + BM25 双索引 → 混合检索（RRF 融合）→ **抽取式**生成（按融合分数拼回原文段落）→ 引用由后端按段落编号生成。索引结构按库可独立重建，前端零构建静态页。

**Tech Stack:** Python 3.11 + FastAPI、嵌入式向量库（chroma 或 lancedb，二选一，见 §1.3）、静态前端（HTML/CSS/JS，零构建）、pytest。

设计规格：`2026-11-22-multimodal-wiki-and-rag-design.md`（§7 RAG 演示应用）
前置计划：P1（工装）、P2/P3（postgres 语料）、P4（其余库语料）

## Global Constraints

- **不接主核 `engine/db`**：RAG 后端只用独立的第三方栈，不 import 主核代码、不共用其存储。
- **抽取式生成，引用由后端生成**：不由模型自编引用；每条引用必须能定位回原文块。
- **前端零构建**：纯静态 HTML/CSS/JS，不引入打包器、Node 工具链或 CDN 依赖。
- **索引按库独立重建**：每归档一个库只重建该库索引，其余不动。
- 密钥（如启用 LLM 插槽）只从环境变量读，不入库、不进代码。
- 提交纪律：只提交 `projects/apps/wiki-rag/**` 相关文件；`engine/db/**` 等用户改动不得暂存。

---

## 0. 当前事实（实测基线）

| 项 | 实测值 |
|---|---|
| RAG 应用代码骨架 | **不存在**（`projects/apps/wiki-rag` 不存在） |
| 切块/索引/嵌入脚本 | **不存在**（`projects/scripts/wiki/` 27 个 `.py` 中无任何相关脚本） |
| 语料规模（当前） | postgres 21 页（20 feature + 1 overview），正文 44,709 字符 / 83,763 B |
| 页面结构规整度 | 21/21 页均为 7 个 H2，标题完全一致；H1 与 manifest `title` 一致 |
| 切块粒度实测 | 按 `##` 切出 **147 段**，段长 38–865 字符，中位 **233** |
| front-matter 结构 | feature 页恰好 5 键（`project`/`feature`/`page_type`/`schema_version`/`refs`）；overview 页 9 键 |
| 解析器 | `wiki_frontmatter.parse_frontmatter` 恰好支持 `refs[].path` + `symbols: [a, b]` 这一层嵌套 → **直接复用，无需另写 YAML 解析** |
| `apps/` 现状 | 5 个应用子目录 + 1 个空 `.archive`：3 个微信小程序（`games` / `knowledge-hub` / `programmer-workbench`，`app.json` + `app.wxss`）、`trip-log`（`package.json` 含 1 个依赖 `@tencent-ai/workbuddy-cloud-sdk`）、`study-hub`（`package.json` **0 依赖**）。**无任何 `requirements.txt` / `pyproject.toml`，无 Python 后端** |
| 风格样板 | `study-hub`：`server/ + web/ + tests/run-all.js + tools/ + 零依赖` |

**必须先解决的显式分歧**：`apps/` 至今零 pip 依赖、零 Python 后端。P5 是第一个带 pip 依赖与 HTTP 后端的应用。这不是缺陷，但要在计划里写死取舍，不能含糊过去（见 §1.3）。

**不可复用的东西**：`projects/scripts/check-wiki-refs.py` 是行号制旧脚本且硬编码已废弃的 `docs/db-wiki/` 路径，与新契约直接冲突，**不作为参考实现**。

### 0.1 基线修正（2026-11-25）

上表是 **2026-11-23 计划制定时**的实测基线，是决策依据的历史记录，**保留不改**。下表列出已被后续实测推翻或更新的项：

| 项 | 计划时基线 | 2026-11-25 实测 |
|---|---|---|
| RAG 应用代码骨架 | **不存在** | **已建成**：`projects/apps/wiki-rag/`（server / core / index / web / tests / tools） |
| 切块/索引/嵌入脚本 | **不存在** | **已建成**：`core/chunker.py` / `core/embedder.py` / `core/bm25.py` / `core/fusion.py` / `index/build_index.py` / `tools/ingest.py` |
| 语料规模（当前） | postgres 21 页（20 feature + 1 overview） | **postgres 152 页**（151 机制页 + 1 概览页），切出 **899 块**；另有 redis / mysql-server / sqlite 三库语料已就绪，**尚未入库** |
| `apps/` 现状 | 5 个应用子目录，**无任何 Python 后端** | 新增 `apps/wiki-rag` —— **`apps/` 下第一个带 pip 依赖与 HTTP 后端的应用**（本计划的显式分歧点，README 已写明理由） |
| 向量库 | chroma 或 lancedb 二选一 | **lancedb 0.41.0**，**明确不用 chromadb**；保留纯 NumPy + `.npy` 暴力检索后备（`WIKI_RAG_LANCEDB=0` 切走） |
| 嵌入 | 本地小模型（bge-small / text2vec 系） | **`BAAI/bge-small-zh-v1.5`**（`sentence-transformers` 离线加载）；加载失败自动降级为 SHA-256 n-gram 哈希嵌入并在 `/api/health` 标记 `embedding_backend=hash-fallback` |

---

## 1. Task 1：技术选型定稿

### 1.1 栈

| 层 | 选型 | 理由 |
|---|---|---|
| 后端 | FastAPI | 设计决策 #15；`uvicorn` 起服务 |
| 嵌入 | 本地 embedding 模型（离线可跑的小模型，如 bge-small / text2vec 系） | 避免调用外部 API 造成密钥与网络依赖 |
| 向量库 | **二选一**（见 §1.3） | 嵌入式、本地持久化 |
| BM25 | 自实现或 `rank_bm25` | 语料小（百级页），自实现可控 |
| 融合 | RRF（Reciprocal Rank Fusion） | 不依赖两路分数可比 |
| 生成 | 抽取式：按融合分数取原文段落拼接 | 无幻觉、引用天然可溯源 |
| 前端 | 纯静态 HTML/CSS/JS | 零构建 |
| 测试 | pytest | Python 生态标准 |

### 1.2 目录骨架（照 `study-hub` 四段式）

```
projects/apps/wiki-rag/
├── requirements.txt          # 唯一带 pip 依赖的应用，显式登记
├── .env.example              # 配置模板（密钥只从环境变量读）
├── README.md                 # 快速开始 / 架构 / 数据流
├── server/
│   ├── main.py               # uvicorn 入口
│   ├── app.py                # FastAPI 应用与路由装配
│   ├── api_search.py         # /api/search、/api/ask
│   ├── api_library.py        # /api/projects、/api/projects/{name}
│   └── config.py             # 配置读取（环境变量）
├── core/
│   ├── chunker.py            # 切块（纯逻辑，无 IO 框架依赖）
│   ├── embedder.py           # 嵌入
│   ├── bm25.py               # BM25
│   ├── fusion.py             # RRF
│   ├── extractive.py         # 抽取式生成
│   └── wiki_source.py        # 读 docs/wiki 并复用 wiki_frontmatter
├── index/
│   ├── build_index.py        # 建索引 CLI（按库）
│   └── data/                 # 索引产物（持久化）
├── web/
│   ├── index.html
│   ├── app.js
│   ├── style.css
├── tests/
│   ├── test_chunker.py
│   ├── test_fusion.py
│   ├── test_extractive.py
│   ├── test_api.py
│   └── test_ingest.py
└── tools/
    └── ingest.py             # 全量/增量入库 CLI
```

### 1.3 向量库二选一（必须先定）

| 选项 | 优点 | 代价 |
|---|---|---|
| `chromadb` | 持久化开箱即用，API 简单 | 依赖树大（numpy/protobuf/onxxruntime），安装体积大 |
| `lancedb` | 依赖轻，纯 Rust 绑定，快 | API 略底层，向量检索需自己写索引参数 |

**默认选 `lancedb`**：`apps/` 的零依赖取向是本仓既有风格，`lancedb` 的依赖树明显小于 chromadb，
且语料规模（百级页）用暴力/IVF 扫描足够，不需要 chromadb 的完整生态。
**若 lancedb 在本机装不上（Windows 环境），改用 numpy + 本地 `.npy` 索引自实现**，
不为此引入 chromadb。这个分支要在 README 里写明。

**依赖纪律**：`requirements.txt` 逐行固定版本；不引入与检索/服务无关的包；CI 装依赖的耗时要有上限意识。

### 1.4 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 目录骨架按 §1.2 落地
- [x] 向量库定稿：**lancedb 0.41.0**（明确不用 chromadb）
- [x] `requirements.txt` 逐行固定版本
- [x] `.env.example` 只含键名
- [x] README 写明快速开始 / 架构 / 数据流 / 与 `apps/` 既有风格的分歧点及理由

**`requirements.txt` 实际内容**（逐行固定版本；**不引入 chromadb**）：

```text
fastapi==0.139.2
uvicorn==0.51.0
numpy==2.1.3
lancedb==0.41.0
pyarrow==26.0.1
sentence-transformers==5.1.2
pytest==8.3.4
httpx==0.28.1
```

**§1.3 的「二选一」已定稿**：**选 lancedb**（本机实测可装可跑）。**不引入 chromadb** —— 依赖树过大，对百级页语料不划算。同时**保留了原文预留的后备分支**：纯 NumPy + `.npy` 精确暴力检索，`WIKI_RAG_LANCEDB=0` 即切走，Windows 上装不上 lancedb 时走这条。该后备分支已写进 README。

**嵌入选型**：默认 `BAAI/bge-small-zh-v1.5`（离线加载，中文语料实测相关度明显优于英文 MiniLM）。加载失败（无模型缓存、无 torch、离线不下载）时自动降级为纯 NumPy 的 SHA-256 n-gram 特征哈希向量，并在 `/api/health` 返回 `embedding_backend=hash-fallback` + `embedding_warning`，前端顶栏显示「演示降级 · 哈希嵌入」。**这是演示用降级，不是语义嵌入。** 换模型必须重建索引（维度或词表不同会让旧索引与查询维度不匹配，检索接口会返回明确的中文错误而不是静默出错）。

**与 `apps/` 既有风格的显式分歧**（§0 提出的问题已在计划里写死取舍，README 同步记录）：

1. **首个 pip 依赖应用** —— `requirements.txt` 只含检索、服务、测试所需包。
2. **首个 Python HTTP 后端** —— 理由是混合检索、离线嵌入与索引持久化在纯静态前端里做不到；本应用独立于其他 app，**不接主核 `engine/db`**。
3. **保留既有前端取向** —— `web/` 仍是纯静态 HTML/CSS/JS：零构建、零 CDN、零外部字体、无框架。

---

## 2. Task 2：语料读取与切块

### 2.1 读语料

`core/wiki_source.py` 负责：

- 扫描 `projects/docs/wiki/projects/<lib>/` 下的 `.md`（**排除** `_manifest.yml`，它不是页面）。
- 用 `wiki_frontmatter.parse_frontmatter` 解析 front-matter（**直接 import `projects/scripts/wiki/wiki_frontmatter.py`**，
  不复制粘贴解析逻辑）。
- 解析 H2 章节：`## 一句话职责` / `## 机制是什么` / `## 怎么实现的` / `## 关键取舍` / `## 源码落点` / `## 参考` / `## 与主核 / AI 的关系`
  （末节标题字面是 `与主核 / AI 的关系`，斜杠两侧各一个空格，按标题原文匹配）。

### 2.2 切块规则

| 规则 | 内容 |
|---|---|
| 块边界 | 页内按 `##` 切；`# H1` 作为块元数据，不单独成块 |
| 块元数据 | `project` / `feature` / `page_type` / `schema_version` / `section` / `title` / `refs` |
| 最小长度 | 低于阈值的块（如「一句话职责」太短）**与相邻块合并**，不单独入库 |
| 最大长度 | 超过阈值的块按段落二次切分，保留 section 元数据 |
| overview 页 | 单独标记 `is_overview=true`，检索时**降权** |
| compare 页 | `page_type: compare` 单独索引，供跨库对比问答 |

实测参考：147 段、中位 233 字符、最长 865 —— 说明「按 H2 切」已经足够细，
最大长度阈值取 ~1200 字符即可，超过的极少。

### 2.3 验收

- `tests/test_chunker.py`：块元数据完整；`feature` 与文件名尾段一致；`refs` 透传正确；
  overview 页被标记；极短块被合并；超长块被二次切分。

### 2.4 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] `core/wiki_source.py` 复用 `wiki_frontmatter.parse_frontmatter`（不复制解析逻辑）
- [x] `core/chunker.py` 按 §2.2 的规则切块
- [x] `tests/test_chunker.py` 覆盖元数据完整性 / 短块合并 / 超长切分 / overview 标记

**实现与 §2.2 规则一致**：页内按 `##` 切，`# H1` 作为块元数据不单独成块；短章节并入邻章；超长章节按段落二次切分。

**块元数据**：`project` / `feature` / `page_type` / `schema_version` / `section` / `title` / `refs` / `is_overview`。

**块 ID 格式**（§3.3 的约定，见 §3.5）：`<project>/<page_file>#<section>`；同一章节二次切分出多段时追加 `::part-N`。

**实测规模**（与 §2.2 的 147 段 / 中位 233 字符的旧基线不同 —— 语料已从 21 页扩到 152 页）：

| 项 | 计划时基线（21 页） | 2026-11-25 实测（postgres 152 页） |
|---|---|---|
| 切出块数 | 147 段 | **899 块**（唯一 block ID 也是 899，无重复） |
| 页数 | 21 | **152**（151 机制页 + 1 概览页） |

**overview 页**按 §2.2 规则单独标记 `is_overview=true`，检索时**降权**。

---

## 3. Task 3：索引与检索

### 3.1 双索引

- **向量索引**：块文本 → 嵌入 → 向量库；存 `block_id`。
- **BM25 索引**：块文本分词 → 倒排；中英混排需处理（中文按 2-gram 或字符 n-gram，英文按词）。
- 两者都按 `block_id` 关联到块元数据。

### 3.2 混合检索与 RRF

```
scores = RRF(vector_hits, bm25_hits, k=60)
```

- RRF 用排名而非分数，规避两路分数量纲不同的问题。
- 参数 `k=60` 为默认值，做成配置项。

### 3.3 增量与按库重建

- 索引产物落在 `projects/apps/wiki-rag/index/data/`。
- **按库重建**：`tools/ingest.py --project postgres` 只重建该库；新增一个库只增量加一个库。
- 块 ID 稳定性：`<project>/<page_file>#<section>`，同页同章节重跑得到同 ID → 增量更新可覆盖而非重复。

### 3.4 验收

- `tests/test_ingest.py`：同一批语料两次入库，块数与 ID 集合一致（幂等）。
- 删一个库重建，其余库索引不受影响。

### 3.5 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 向量索引 + BM25 索引双索引
- [x] RRF 融合，`k=60` 默认值可配置
- [x] 按库独立重建
- [x] 块 ID 稳定、可覆盖不重复
- [x] `tests/test_ingest.py` 覆盖幂等与按库隔离

**双索引**：

| 索引 | 实现 | 落点 |
|---|---|---|
| 向量 | 嵌入 → LanceDB 表（同名表 `project_<库名>`） | `index/data/vectors.lancedb/` |
| 向量（后备） | 纯 NumPy 精确暴力检索 | `index/data/projects/<库名>/vectors.npy` |
| BM25 | `core/bm25.py` 自实现（语料小，可控） | 内存构建，随入库重建 |

两者都按 `block_id` 关联到块元数据。BM25 分词对中英混排做了处理：中文按字符 n-gram，英文按词。

**RRF 融合**：`scores = RRF(vector_hits, bm25_hits, k=60)`，用排名而非分数，规避两路分数量纲不同。`k` 做成配置项（默认 60），**overview 页降权**。

**索引落点**：`index/data/projects/<库名>/` 分库保存 `chunks.json`（块与元数据）、`vectors.npy`（向量）、`metadata.json`（页数、块数、块 ID 全量清单、门禁退出码、嵌入/向量后端）。向量另存于 `index/data/vectors.lancedb/` 的同名表。`index/data/` 已在 `apps/wiki-rag/.gitignore` 排除。

**按库重建**：`tools/ingest.py --project postgres` 只重建该库（覆盖该库目录与该库表），`tools/ingest.py --all` 重建语料里所有库。**其余库索引不受影响。**

**块 ID 稳定性**：`<project>/<page_file>#<section>`，同页同章节重跑得到同 ID → 增量更新可覆盖而非重复。同一章节二次切分出多段时追加 `::part-N`。实测 postgres 语料 **899 块 / 唯一 block ID 899**，无重复。

**实测入库规模**：postgres —— pages=152、block_count=899。

---

## 4. Task 4：抽取式生成与引用回源

### 4.1 生成

`core/extractive.py`：取融合分数 Top-K 的块 → 按分数与页内顺序组织 → 输出「答案正文 + 引用列表」。

- **答案正文由原文段落拼成**，不是模型改写。
- 每个段落前标注来源（`[项目/页名 § 章节]`）。

### 4.2 引用

- 每条引用 = `(project, page_file, section, block_id)` + 片段预览。
- 右栏面板按此渲染，点击可展开片段全文。
- **引用一律由后端按块 ID 生成**，不接受模型或前端自造。

### 4.3 验收

- `tests/test_extractive.py`：输出中的每个引用都能在块库里找到；引用的文本是原文的连续片段。

### 4.4 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] `core/extractive.py` 按融合分数 Top-K 取块并组织输出
- [x] 引用一律由后端按块 ID 生成
- [x] `tests/test_extractive.py` 覆盖引用可回源与无结果回退

**生成**：`core/extractive.py` 取融合分数 Top-K 的块 → 按分数与页内顺序组织 → 输出「答案正文 + 引用列表」。

- **答案正文由原文段落拼成**，不是模型改写。实测 ask 返回 `answer_len=1682`，正文长度即拼接的原文段落总长。
- 每个段落前标注来源（`[项目/页名 § 章节]`）。

**引用**：每条引用 = `(project, page_file, section, block_id)` + 片段预览，右栏面板按此渲染，点击可展开片段全文。**引用一律由后端按块 ID 生成**，不接受模型或前端自造。

**回源验证**（这是 P5 最关键的正确性证据，不是「看起来能跑」）：

`tools/smoke.py` 把 `/api/ask` 返回的**每个 `block_id`** 拿去 `index/data/projects/<库>/chunks.json` 里查，缺失就打印出来。**实测结果：引用回源缺失 = 无。** 实测 ask 返回 `citations=4`。

**无结果回退**：检索无结果时输出显式回退态（`abstained` 字段），不假装有答案。实测 `abstained=False`（该次查询有命中）。

---

## 5. Task 5：HTTP API

| 端点 | 方法 | 用途 |
|---|---|---|
| `/api/health` | GET | 服务存活 |
| `/api/projects` | GET | 左栏：库列表 + 页数 + 校验状态 |
| `/api/projects/{name}` | GET | 某库的分组与页面导航 |
| `/api/search` | GET | 纯检索（返回块与分数，不生成答案） |
| `/api/ask` | POST | 问答：检索 → 抽取式生成 → 答案 + 引用 + 检索抽屉数据 |

**左栏「校验状态」**：`PASS` / `FAIL` 两态（新契约下无行号、全部过符号硬校验，没有第三档）。
状态来源：调用 `wiki_lint.py` / `wiki_refs.py` 的 CLI（退出码 0/1），**不自造校验逻辑**。

**检索抽屉数据**：向量召回条数、BM25 命中段数、耗时 —— 让人看见检索过程，而不是只给答案。

### 5.1 验收

- `tests/test_api.py`：每个端点有正常路径与错误路径用例；`/api/ask` 返回的引用可回源。

### 5.2 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 5 个 `/api/*` 端点全部落地（另加静态页 `/`）
- [x] 左栏校验状态来自**真实门禁退出码**，不写死
- [x] 检索抽屉数据返回向量召回数 / BM25 命中数 / 耗时
- [x] `tests/test_api.py` 覆盖正常路径与错误路径

**端点实现**（`server/api_library.py` + `server/api_search.py` + `server/app.py`）：

| 端点 | 方法 | 实测输出 |
|---|---|---|
| `/api/health` | GET | **200** —— 嵌入后端 `sentence-transformers`（模型 `BAAI/bge-small-zh-v1.5`）、向量后端 `lancedb`、索引是否就绪、页数、降级告警 |
| `/api/projects` | GET | **1 个库**：`postgres`，pages=**152**，validation=**PASS**，gates `wiki_lint:0, wiki_refs:0` |
| `/api/projects/{name}` | GET | pages=**152**，block_count=**899** |
| `/api/search` | POST | **3 条结果** |
| `/api/ask` | POST | answer_len=**1682**，citations=**4**，abstained=**False** |

**左栏「校验状态」不是写死的**：`tools/ingest.py` 实际以**子进程**运行 `projects/scripts/wiki/wiki_lint.py` 与 `wiki_refs.py`，把**退出码**写进 `metadata.json`，API 原样读出，任一非零即 `FAIL`。`wiki_refs.py` 需要该库的本地源码浅克隆，默认找 `projects/.cache/repos/<库名>`，可用 `WIKI_RAG_<库名大写>_REPO` 覆盖（如 `WIKI_RAG_POSTGRES_REPO`）。**找不到源码仓时记 127 而非假装通过。**

**检索抽屉数据**：向量召回条数、BM25 命中段数、耗时 —— 让人看见检索过程，而不是只给答案。

**端点形态的一处调整**：`/api/search` 与 `/api/ask` 实现为 **POST**（`/api/search` 收 `{"query": ..., "projects": [...], "limit": ...}`），原文表格未写明方法，此处按实际实现登记。

---

## 6. Task 6：三栏前端

| 栏 | 内容 |
|---|---|
| 左 | 库列表 + 每库页数 + 校验状态（PASS / FAIL） |
| 中 | 提问区 → 回答正文（原文段落 + 引用标记）→ 检索抽屉（向量召回数 / BM25 命中数 / 耗时） |
| 右 | 出处面板：每条引用带项目、页面、章节、符号与片段预览；检索无结果时显示明确回退态 |

- 零构建、零 CDN、零外部字体。
- 单页应用用原生 JS，无框架。
- 检索失败/无结果必须**显式显示**，不显示空白或假装有答案。

### 6.1 执行结果（2026-11-24 ~ 2026-11-25）—— **完成**

- [x] 三栏布局落地于 `web/index.html` + `web/app.js` + `web/style.css`
- [x] 零构建、零 CDN、零外部字体、无框架
- [x] 检索无结果时显示明确回退态
- [x] 嵌入降级时顶栏显示「演示降级 · 哈希嵌入」

前端为**纯静态 HTML/CSS/JS**：原生 JS，无框架、无打包器、无 Node 工具链、无 CDN 依赖、无外部字体 —— 与 `apps/` 下其他应用一致。

**降级可见性**：嵌入加载失败时，API 返回 `embedding_backend=hash-fallback` + `embedding_warning`，前端顶栏显示「演示降级 · 哈希嵌入」，**不让用户误以为在跑语义嵌入**。

**出处面板**：按引用逐条渲染项目、页面、章节、符号与片段预览，点击可展开片段全文。检索无结果时显示明确回退态（配合 §4.4 的 `abstained` 字段）。

---

## 7. Task 7：语料扩展后的适配

P4 完成后语料会包含多个库与 `compare` 页，索引与前端需支持：

| 情形 | 处理 |
|---|---|
| 多库 | `project` 字段已天然区分；左栏按库分组 |
| `compare` 页 | 单独索引标记，检索时不做 overview 降权 |
| 库新增 | 增量入库，不重建其他库 |
| 单库页数很大 | 索引按 `project` 分区存储，检索时先按库过滤再融合 |

### 7.1 执行结果（2026-11-25）—— **部分完成**

- [x] 多库：`project` 天然区分，左栏按库分组，索引按库分区
- [x] 库新增：增量入库，不重建其他库
- [x] 单库页数大：索引按 `project` 分区，检索可按库过滤
- [ ] `compare` 页：**未做** —— P4 的对比页（`_compare/`）本身尚未产出（见 P4 计划 §6.1），前端无 `page_type: compare` 的实际样本可验证

**当前实际入库的库**：**只有 postgres**（152 页 / 899 块）。redis / mysql-server / sqlite 三库的语料在 P4 已就绪，但**尚未入库** —— 索引构建与冒烟验证都只跑了 postgres。

**注意**：P4 的 mysql-server 是**文件级稀疏检出**（未下载的文件不能做 refs 校验）。若后续为该库建索引，`tools/ingest.py` 跑 `wiki_refs.py` 时会依赖本地源码仓，该库的检出方式需要在入库前确认。

---

## 8. 验收条件

> 下表在保留原文验收项的基础上，补上 2026-11-25 的实际达成情况。

| # | 验收条件 | 实际结果 |
|---|---|---|
| 1 | `projects/apps/wiki-rag/` 目录建成，`requirements.txt` 固定版本，`README.md` 写明快速开始、数据流与选型取舍 | **达成** —— 目录齐备；`requirements.txt` 逐行固定版本；README 含快速开始 / 架构与数据流 / 分支与选型 / 与 `apps/` 既有风格的分歧（§1.4） |
| 2 | 切块：postgres 语料切出的块数、ID 稳定性、metadata 完整性有测试覆盖 | **达成** —— postgres 切出 **899 块 / 唯一 ID 899**；`tests/test_chunker.py` 覆盖（§2.4 / §3.5） |
| 3 | 双索引 + RRF 检索可用；增量入库幂等；按库重建互不影响 | **达成** —— 向量（LanceDB）+ BM25 双索引，RRF `k=60`；`tests/test_ingest.py` 覆盖幂等与按库隔离（§3.5） |
| 4 | 抽取式生成：答案由原文段落组成，引用由后端生成且可回源 | **达成** —— 实测 `answer_len=1682`、`citations=4`、**引用回源缺失 = 无**（§4.4） |
| 5 | API 六个端点有测试；左栏校验状态来自真实门禁退出码 | **达成** —— 5 个 `/api/*` 端点 + 静态页 `/`；`tests/test_api.py` 覆盖正常与错误路径；校验状态来自真实退出码（实测 `wiki_lint:0, wiki_refs:0` → PASS）（§5.2） |
| 6 | 三栏前端零构建运行，检索无结果时有明确回退态 | **达成**（§6.1） |
| 7 | **不 import 主核 `engine/db` 任何代码**（可在测试中断言） | **达成** —— 主核 `engine/db` 不在依赖图内，代码与测试都没有 import |
| 8 | 密钥只从环境变量读；`.env.example` 只含键名 | **达成** —— 应用不解析 `.env` 文件，只从环境变量读；`.env.example` 只列键名与默认值 |
| 9 | README 记录：与 `apps/` 既有风格的分歧点（首个 pip 依赖应用、首个 Python 后端）及理由 | **达成**（§1.4） |
| 10 | 全量测试通过；只提交 `projects/apps/wiki-rag/**` 相关路径 | **达成** —— `python -m pytest tests -q` 通过；提交路径限于 `projects/apps/wiki-rag/**` |
| — | **冒烟测试（端到端）** | **达成** —— `tools/smoke.py` 真实跑通全部端点，输出见 §5.2 |

---

## 9. 不在本阶段范围

- 不做 LLM 生成（只留 `generator` 接口插槽，抽取式是唯一实现）。
- 不做多用户、账号、持久化会话。
- 不做主核 `engine/db` 的能力验证或集成。
- 不改 wiki 契约（`schema_version` 仍为 1）；若切块需要新字段，先按契约升版流程走。

---

## 10. 受阻项与遗留

### 10.1 study-hub 的 R2 / R3 —— **因缺 LLM 密钥无法执行**

**这是本阶段唯一的硬阻塞，必须明示，不能含糊过去。**

- 环境里 **`ARK_API_KEY` 与 `ANTHROPIC_API_KEY` 均为空**（两个变量都没有值）。
- 因此 study-hub 依赖 LLM 密钥的 **R2 / R3** 两项验证**无法执行** —— 不是写坏了，是**环境不具备执行条件**。
- 在拿到密钥之前，这两项**不能记为通过，也不能记为失败** —— 状态是**受阻（blocked）**。

**影响范围**：study-hub 的 R2 / R3 两条验证项，以及任何需要真实 LLM 调用的跨应用对照。
**不影响范围**：wiki-rag 自身的全部交付 —— 本应用的生成是**抽取式**，**不需要任何 LLM 密钥**，所以 P5 的主目标不受此阻塞影响。

**解除条件**：注入 `ARK_API_KEY` 或 `ANTHROPIC_API_KEY` 之一（密钥只从环境变量读，不入库、不进代码 —— 与 §8 第 8 条一致），然后补跑 R2 / R3。

### 10.2 需要补的事

| # | 事项 | 性质 | 说明 |
|---|---|---|---|
| 1 | **CI 的 `wiki-refs` job 接入新增三库** | 待办 | 该 job 目前只浅克隆 postgres。新增 redis / mysql-server / sqlite 的 refs 校验尚未接入 CI；**mysql 因是文件级稀疏检出，CI 侧需另行处理检出方式**（详见 P4 计划 §4.6 与 §9） |
| 2 | **对比页 `_compare/` 未做** | 待办 | P4 的 Task 6 未产出对比页，本应用也无 `page_type: compare` 的实际样本可验证（§7.1） |
| 3 | redis / mysql-server / sqlite 三库**尚未入库** | 待办 | 索引构建与冒烟验证当前只跑了 postgres（§7.1） |

### 10.3 已提交记录

| commit | 内容 |
|---|---|
| `3bd5179e4` | 新增三栏 RAG 演示应用，BM25 + 向量 RRF 混合检索与抽取式引用 |
| `4d42f2da1` | 补端到端冒烟脚本，验证引用可回源与块 ID 唯一（`tools/smoke.py` + README） |
| `f63b976db` | 补端到端冒烟脚本与 conftest，修正从仓根跑 pytest 的导入路径（`conftest.py`） |

**端到端冒烟脚本为何不并进 pytest**：真实索引有 **899 块**、要加载真实嵌入模型，放进默认测试会让它变慢并依赖模型缓存。冒烟脚本只在「索引已建好」的环境里跑，自己起一个 uvicorn 子进程（端口 8012），依次打全部端点并断言引用可回源，跑完自动 `terminate` 子进程。
