# INDEX

AI-DOCS 内容索引。收录边界见顶层 [README](README.md)，判定口诀：**换个项目还用得上 → AI-DOCS；只对这个项目有意义 → 留在本仓。**

| 目录 | 放什么 | 当前内容 | 详情 |
|---|---|---|---|
| `templates/` | 跨项目通用的 spec / doc / review / prompt 模板 | README 存根；`spec/`、`doc/`、`review/`、`prompt/` 各含 `.gitkeep` 占位 | [README](templates/README.md) |
| `methodology/` | 可复用的工作方法与流程（设计评审、调试、规划） | README 存根；`design-review/`、`debugging/`、`planning/` 各含 `.gitkeep` 占位 | [README](methodology/README.md) |
| `patterns/` | 通用架构模式、数据结构选型、流程编排 | README 存根；`architecture/`、`data-structure/`、`process/` 各含 `.gitkeep` 占位 | [README](patterns/README.md) |
| `assets/` | 通用图表、脚本等可复用资产 | README 存根；`diagrams/`、`scripts/` 各含 `.gitkeep` 占位 | [README](assets/README.md) |
| `prompts/` | 通用提示词库 | 仅 README 存根 | [README](prompts/README.md) |
| `agent-notes/` | agent 通用踩坑经验与最佳实践 | 仅 README 存根 | [README](agent-notes/README.md) |
| `references/` | 外部资料的链接与摘要索引 | 仅 README 存根 | [README](references/README.md) |
| `archive/` | 已废弃但保留归档的 AI 资产 | 仅 README 存根，暂无归档条目 | [README](archive/README.md) |

> 八个目录**全部**已有 README 存根（各自写明「放 / 不放」和「现有」段）；`templates/`、`methodology/`、`patterns/`、`assets/` 下共 12 个子目录仅有 0 字节 `.gitkeep` 占位，**尚无任何模板 / 方法论 / 模式 / 提示词 / 参考条目正文**，`archive/` 也是空的。

## 不收去哪里

- 项目级架构文档、openspec 变更、agent 规约（`.ai/`、`CLAUDE.md`）→ [`../projects/`](../projects/)
- 学习笔记与课程进度 → [`../learning/`](../learning/)
- 第三方源码镜像 → [`../../reference/`](../../reference/)
