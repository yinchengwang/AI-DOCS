# INDEX

AI-DOCS 内容索引。收录边界见顶层 [README](README.md)。

**收录判定（2026-11-22 修订）**：以**资产类型**划分，不以「是否项目特定」划分。

- 换个项目还用得上的**模板 / 方法论 / 模式 / 提示词 / 经验** → 本仓
- **长周期项目的设计规格与实施计划** → 本仓 `methodology/planning/`（这类文档记录的是可复用的决策与流程，不是某个项目的执行日志）
- 项目**当前**的架构文档、openspec 变更、agent 规约 → `projects/`

| 目录 | 放什么 | 当前内容 | 详情 |
|---|---|---|---|
| `templates/` | 跨项目通用的 spec / doc / review / prompt 模板 | README 存根；`spec/`、`doc/`、`review/`、`prompt/` 各含 `.gitkeep` 占位 | [README](templates/README.md) |
| `methodology/` | 可复用的工作方法与流程（设计评审、调试、规划） | README 存根；`planning/` 已有 **7 篇**（见下），`design-review/`、`debugging/` 仍为 `.gitkeep` 占位 | [README](methodology/README.md) |
| `patterns/` | 通用架构模式、数据结构选型、流程编排 | README 存根；`architecture/`、`data-structure/`、`process/` 各含 `.gitkeep` 占位 | [README](patterns/README.md) |
| `assets/` | 通用图表、脚本等可复用资产 | README 存根；`diagrams/`、`scripts/` 各含 `.gitkeep` 占位 | [README](assets/README.md) |
| `prompts/` | 通用提示词库 | 仅 README 存根 | [README](prompts/README.md) |
| `agent-notes/` | agent 通用踩坑经验与最佳实践 | 仅 README 存根 | [README](agent-notes/README.md) |
| `references/` | 外部资料的链接与摘要索引 | 仅 README 存根 | [README](references/README.md) |
| `archive/` | 已废弃但保留归档的 AI 资产 | 仅 README 存根，暂无归档条目 | [README](archive/README.md) |

## 已收录条目

### methodology/planning/

| 条目 | 内容 |
|---|---|
| [2026-11-22-multimodal-wiki-and-rag-design.md](methodology/planning/2026-11-22-multimodal-wiki-and-rag-design.md) | 多模态开源项目 Wiki 与 RAG 演示：设计规格。含范围分层（T1/T2/T3）、机制单元粒度判定、六必含章节契约、引用硬失败策略、按库流式并行生产流水线、三栏 RAG 界面。**已按 P2 实测校准**：篇数由逐库机制盘点决定（postgres 实测 151 页 / 12 组），作废按文件数折算的 0.66 系数；总量只认「逐库盘点结果之和」。可复用点：**大规模文档语料的契约化生产方式**。 |
| [2026-11-22-multimodal-wiki-p0-toolbox.md](methodology/planning/2026-11-22-multimodal-wiki-p0-toolbox.md) | 同项目 P0 工装实施计划。7 个 TDD 任务的完整代码与验收命令：契约常量、front-matter 受控子集解析、结构门禁、引用门禁（含 `--locate`）、源码浅克隆缓存、契约文档一致性测试。可复用点：**用机械门禁替代人工 review 的工装设计**。 |
| [2026-11-22-multimodal-wiki-p1-catalog-and-manifest.md](methodology/planning/2026-11-22-multimodal-wiki-p1-catalog-and-manifest.md) | 同项目 P1 计划。8 个任务：清单解析与校验、manifest 读写与门禁、源码目录枚举、清单扩容（stars 经 GitHub API 实取）、postgres manifest 生成。**执行后注记记录了两处与原文不符的实测结果**：清单实到 152 条（非 270），manifest 原为 332 篇并已在 P2a 废弃重建为 151 篇机制页。可复用点：**把「写哪些页」变成可校验的机器清单**，写作阶段只读清单、不自由发挥。 |
| [2026-11-22-multimodal-wiki-p2-a-group-pilot.md](methodology/planning/2026-11-22-multimodal-wiki-p2-a-group-pilot.md) | 同项目 P2 A 组试点计划：先修 manifest 机制边界/refs，再按 4 个不重叠分片写 20 篇正文；双门禁 + 5 篇深度抽检后生成 `_index.md`，记录真实吞吐与返工数据。**已回填 P2a–P2e 全部结果**：151 页 / 12 组、604 ref 对全命中、双门禁 21 页 0 失败、315 测试通过、抽检 29/30。 |
| [2026-11-23-multimodal-wiki-p3-postgres-full.md](methodology/planning/2026-11-23-multimodal-wiki-p3-postgres-full.md) | 同项目 P3 计划：postgres 剩余 **131 页**（B–L 11 组）批量写作。含 20 个分片的写作用域表、5 个波次的编排、单页工作流（读规格→读源码→写页→单页双门禁→外链核验）、每波每组抽 1 页的 6 项评分表、`_index.md` 重建、契约测试扩展到 151 页覆盖、全库验收报告口径。可复用点：**分片不跨领域 + 波次门禁 + 独立审阅者抽检**的规模化写作编排。 |
| [2026-11-23-multimodal-wiki-p4-other-libraries.md](methodology/planning/2026-11-23-multimodal-wiki-p4-other-libraries.md) | 同项目 P4 计划：其余库流式推进。含把 postgres 专用生成器泛化为库无关 `gen_manifest.py`（5 处专用约束逐条列出）、清单 152→270 扩容与 97 条占位 stars 回填、门禁接入 CI（当前 `wiki` 未接）、单库五步 SOP、首批 3 库（sqlite 无 `src/` / mysql 多层 / redis 单层）验证泛化性、批次编排判据。可复用点：**「没有机制盘点就不写正文」与「一个库双绿再开下一个」的流式纪律**。 |
| [2026-11-23-multimodal-wiki-p5-rag-app.md](methodology/planning/2026-11-23-multimodal-wiki-p5-rag-app.md) | 同项目 P5 计划：RAG 演示应用（**零起点**，`apps/wiki-rag` 不存在）。含向量库二选一决策与依赖纪律、切块规则（按 H2、overview 降权）、双索引 + RRF 融合、抽取式生成与引用回源、6 个 API 端点、三栏前端、按库增量重建、语料扩展后的适配。可复用点：**引用由后端按块 ID 生成、生成层可换但检索层不动**。 |

> 这几份文档的**代码产物**在 [`projects/scripts/wiki/`](../../projects/scripts/wiki/) 与 [`projects/docs/wiki/`](../../projects/docs/wiki/)，不重复存放在本仓。

## 不收去哪里

- 项目当前架构文档、openspec 变更、agent 规约（`.ai/`、`CLAUDE.md`）→ [`../projects/`](../projects/)
- 学习笔记与课程进度 → [`../learning/`](../learning/)
- 第三方源码镜像 → [`../../reference/`](../../reference/)
