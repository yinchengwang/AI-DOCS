# AI-DOCS

跨项目复用的 AI 工作资产仓。

本仓是 [StillEvolving](../README.md) 聚合仓下的 **AI 工作文档仓**（submodule，origin `https://github.com/yinchengwang/AI-DOCS.git`），收录跨项目复用的模板、方法论、模式、通用提示词、agent 通用踩坑经验、外部资料索引与通用图表脚本资产。

## 仓库关系

本仓是四仓体系中的一员，完整关系如下（含父仓本身）：

| 仓 | 获取方式 | 路径 | 角色 |
|---|---|---|---|
| StillEvolving | 父仓 | [`../`](../README.md) | 只放四仓关系，不放业务代码 |
| projects | submodule | [`../projects/`](../projects/) | 工程仓（引擎 + 小程序 + 项目级文档 + agent 规约） |
| learning | submodule | [`../learning/`](../learning/) | 学习仓（笔记 + 脚手架 + 题解 + 进度） |
| **ai-docs** | submodule | 本仓 | **AI 工作文档仓**（跨项目复用的模板与方法论） |
| reference | 独立仓 + 清单拉取 | [`../../reference/`](../../reference/) | 第三方源码镜像 + 源码笔记（不进 submodule 树） |

完整目录规范见父仓 [`../docs/architecture.md`](../docs/architecture.md)；机器可读版见父仓 [`../repos.yaml`](../repos.yaml) 的 `must_contain` 字段。

> 上述跨仓相对路径按本地 StillEvolving 工作区布局书写（父仓 `../`、同级 `../projects/`、`../learning/`、与 StillEvolving 平级但不在其内的 `../../reference/`）。

## 收录边界

| 收进来 | 不收 |
|---|---|
| 跨项目复用的模板、方法论、模式 | 项目级架构文档 → projects `docs/` |
| 通用提示词库 | 项目级 agent 规约 → projects `.ai/`、`CLAUDE.md` |
| agent 通用踩坑经验 | 具体项目的 openspec 变更 → projects `openspec/` |
| 外部资料的链接与摘要索引 | 学习笔记与课程进度 → learning |
| 通用图表与脚本资产 | 第三方源码镜像 → reference |

**判定口诀：换个项目还用得上 → AI-DOCS；只对这个项目有意义 → 留在本仓。**

> ⚠️ **红线**：项目级 agent 规约留在 projects（`.ai/`、`CLAUDE.md`、`AGENTS.md`、`openspec/`、`skills-lock.json`）—— 搬走会导致 agent 读不到规约。

## 目录索引

八个目录各自放什么、当前有没有内容，见 **[INDEX.md](INDEX.md)**。

```
templates/     methodology/   patterns/      assets/
prompts/       agent-notes/   references/    archive/
```

### 各目录现状（实测）

八个目录**全部**已有各自的 `README.md` 存根，存根里都有「放 / 不放」两行声明和「现有」段（当前均为「暂无条目」）：

| 目录 | 放什么（摘自各自 README） | 当前实际内容 |
|---|---|---|
| [`templates/`](templates/README.md) | 跨项目通用的 spec / doc / review / prompt 模板 | README 存根 + 4 个子目录各含 `.gitkeep` 占位：`spec/`、`doc/`、`review/`、`prompt/` |
| [`methodology/`](methodology/README.md) | 跨项目通用的方法论与流程（设计评审、调试、规划） | README 存根 + 3 个子目录各含 `.gitkeep` 占位：`design-review/`、`debugging/`、`planning/` |
| [`patterns/`](patterns/README.md) | 通用架构模式、数据结构选型、流程编排模式 | README 存根 + 3 个子目录各含 `.gitkeep` 占位：`architecture/`、`data-structure/`、`process/` |
| [`assets/`](assets/README.md) | 通用图表、通用脚本 | README 存根 + 2 个子目录各含 `.gitkeep` 占位：`diagrams/`、`scripts/` |
| [`prompts/`](prompts/README.md) | 跨项目通用的提示词、角色设定、提问模板 | 仅 README 存根，无子目录 |
| [`agent-notes/`](agent-notes/README.md) | 跨项目通用的 agent 经验、工具使用技巧、常见失败与规避方式 | 仅 README 存根，无子目录 |
| [`references/`](references/README.md) | 外部文章、文档、工具的链接与一句摘要 | 仅 README 存根，无子目录 |
| [`archive/`](archive/README.md) | 已失效但有参考价值的旧模板、旧方法论 | 仅 README 存根，无子目录，暂无归档条目 |

**结论**：当前是「**存根 + 占位骨架**」状态 —— `templates/`、`methodology/`、`patterns/`、`assets/` 下共 12 个子目录，每个只有一个 0 字节 `.gitkeep` 占位文件；**尚无任何模板 / 方法论 / 模式 / 提示词 / 参考条目正文**。`archive/` 同样是空的，没有任何已归档内容。骨架已就位，等待填入。

## 填写约定

- 新增内容放进对应目录，并在该目录 README 的「现有」段登记条目（当前均为「暂无条目」）。
- 只收换个项目还能直接复用的内容；一旦只对某个项目有意义，就移回 projects / learning / reference。
- 外部资料只存链接 + 一句摘要，不存副本（副本进 [`../../reference/`](../../reference/)）。
- 子目录里的 `.gitkeep` 在写入第一个真实文件后删除。
