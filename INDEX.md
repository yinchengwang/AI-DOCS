# INDEX

AI-DOCS 内容索引。收录边界见顶层 [README](README.md)。

**收录判定（2026-11-22 修订）**：以**资产类型**划分，不以「是否项目特定」划分。

- 换个项目还用得上的**模板 / 方法论 / 模式 / 提示词 / 经验** → 本仓
- **长周期项目的设计规格与实施计划** → 本仓 `methodology/planning/`（这类文档记录的是可复用的决策与流程，不是某个项目的执行日志）
- 项目**当前**的架构文档、openspec 变更、agent 规约 → `projects/`

| 目录 | 放什么 | 当前内容 | 详情 |
|---|---|---|---|
| `templates/` | 跨项目通用的 spec / doc / review / prompt 模板 | README 存根；`spec/`、`doc/`、`review/`、`prompt/` 各含 `.gitkeep` 占位 | [README](templates/README.md) |
| `methodology/` | 可复用的工作方法与流程（设计评审、调试、规划） | README 存根；`planning/` 已有 2 篇（见下），`design-review/`、`debugging/` 仍为 `.gitkeep` 占位 | [README](methodology/README.md) |
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
| [2026-11-22-multimodal-wiki-and-rag-design.md](methodology/planning/2026-11-22-multimodal-wiki-and-rag-design.md) | 多模态开源项目 Wiki 与 RAG 演示：设计规格。含范围分层（T1/T2/T3）、机制单元粒度判定、六必含章节契约、引用硬失败策略、按库流式并行生产流水线、三栏 RAG 界面。可复用点：**大规模文档语料的契约化生产方式**。 |
| [2026-11-22-multimodal-wiki-p0-toolbox.md](methodology/planning/2026-11-22-multimodal-wiki-p0-toolbox.md) | 同项目 P0 工装实施计划。7 个 TDD 任务的完整代码与验收命令：契约常量、front-matter 受控子集解析、结构门禁、引用门禁（含 `--locate`）、源码浅克隆缓存、契约文档一致性测试。可复用点：**用机械门禁替代人工 review 的工装设计**。 |
| [2026-11-22-multimodal-wiki-p1-catalog-and-manifest.md](methodology/planning/2026-11-22-multimodal-wiki-p1-catalog-and-manifest.md) | 同项目 P1 计划。8 个任务：清单解析与校验、manifest 读写与门禁、源码目录枚举、清单扩容（stars 经 GitHub API 实取）、postgres 332 篇 manifest 生成。可复用点：**把「写哪些页」变成可校验的机器清单**，写作阶段只读清单、不自由发挥。 |
| [2026-11-22-multimodal-wiki-p2-a-group-pilot.md](methodology/planning/2026-11-22-multimodal-wiki-p2-a-group-pilot.md) | 同项目 P2 A 组试点计划：先修 manifest 机制边界/refs，再按 4 个不重叠分片写 29 篇正文；双门禁 + 5 篇深度抽检后生成 `_index.md`，记录真实吞吐与返工数据。 |

> 这几份文档的**代码产物**在 [`projects/scripts/wiki/`](../../projects/scripts/wiki/) 与 [`projects/docs/wiki/`](../../projects/docs/wiki/)，不重复存放在本仓。

## 不收去哪里

- 项目当前架构文档、openspec 变更、agent 规约（`.ai/`、`CLAUDE.md`）→ [`../projects/`](../projects/)
- 学习笔记与课程进度 → [`../learning/`](../learning/)
- 第三方源码镜像 → [`../../reference/`](../../reference/)
