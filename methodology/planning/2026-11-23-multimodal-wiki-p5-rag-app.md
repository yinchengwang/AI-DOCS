# wiki P5（RAG 演示应用）Implementation Plan

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

---

## 7. Task 7：语料扩展后的适配

P4 完成后语料会包含多个库与 `compare` 页，索引与前端需支持：

| 情形 | 处理 |
|---|---|
| 多库 | `project` 字段已天然区分；左栏按库分组 |
| `compare` 页 | 单独索引标记，检索时不做 overview 降权 |
| 库新增 | 增量入库，不重建其他库 |
| 单库页数很大 | 索引按 `project` 分区存储，检索时先按库过滤再融合 |

---

## 8. 验收条件

1. `projects/apps/wiki-rag/` 目录建成，`requirements.txt` 固定版本，`README.md` 写明快速开始、数据流与选型取舍。
2. 切块：postgres 语料切出的块数、ID 稳定性、metadata 完整性有测试覆盖。
3. 双索引 + RRF 检索可用；增量入库幂等；按库重建互不影响。
4. 抽取式生成：答案由原文段落组成，引用由后端生成且可回源。
5. API 六个端点有测试；左栏校验状态来自真实门禁退出码。
6. 三栏前端零构建运行，检索无结果时有明确回退态。
7. **不 import 主核 `engine/db` 任何代码**（可在测试中断言）。
8. 密钥只从环境变量读；`.env.example` 只含键名。
9. README 记录：与 `apps/` 既有风格的分歧点（首个 pip 依赖应用、首个 Python 后端）及理由。
10. 全量测试通过；只提交 `projects/apps/wiki-rag/**` 相关路径。

## 9. 不在本阶段范围

- 不做 LLM 生成（只留 `generator` 接口插槽，抽取式是唯一实现）。
- 不做多用户、账号、持久化会话。
- 不做主核 `engine/db` 的能力验证或集成。
- 不改 wiki 契约（`schema_version` 仍为 1）；若切块需要新字段，先按契约升版流程走。
