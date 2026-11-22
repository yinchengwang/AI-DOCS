# wiki 工装（P0）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **执行后注记（2026-11-22）**
>
> 本计划原文按 `reference` 仓书写（`reference/scripts/`、`reference/docs/wiki-schema.md`），当时确实是那样执行的。
> P0 完成后，产物已迁至 **`projects` 仓**：
>
> | 原路径 | 现路径 |
> |---|---|
> | `reference/scripts/wiki_*.py`、`repo_cache.py` 及其测试 | `projects/scripts/wiki/` |
> | `reference/docs/wiki-schema.md` | `projects/docs/wiki/wiki-schema.md` |
> | `reference/.cache/repos/` | `projects/.cache/repos/` |
>
> 迁移带来的三处改动：`repo_cache` 的默认缓存目录上溯两级；`test_repo_cache` 的仓根常量同步；
> `test_wiki_schema_doc` 的契约文档路径改为 `../../docs/wiki/wiki-schema.md`。
> 迁移后全套 174 项测试通过（1 项为 Windows symlink 权限跳过）。
>
> **下文的任务步骤保留原始路径，作为执行记录，不做追溯改写。**

**Goal:** 建成 wiki 文档的契约常量、front-matter 解析器、结构门禁、引用门禁与源码缓存工具，让后续 2.5 万篇文档的写作能被机械判定合格与否。

**Architecture:** 五个只依赖标准库的 import-able 模块，各自单一职责：`wiki_schema` 只放常量，`wiki_frontmatter` 只解析，`wiki_lint` 只查结构，`wiki_refs` 只查引用，`repo_cache` 只管浅克隆。两个门禁都是 CLI，退出码 0/1，可直接接 CI 与并行流水线。

**Tech Stack:** Python 3.11、标准库（`argparse`/`re`/`os`/`subprocess`/`unittest`）、git 命令行。

设计规格：本目录同名的 `2026-11-22-multimodal-wiki-and-rag-design.md`

## Global Constraints

- 文档源格式只允许 Markdown；禁止 MediaWiki 语法（`==标题==`、`[[链接]]`、`{|` 表格）。
- **禁止行号引用**。引用一律「相对项目仓根的路径 + 符号名」，不得出现 `path:123` 形态。
- refs 路径基准 = 项目仓根（如 `src/backend/storage/buffer/bufmgr.c`），不写 `reference/relational/postgres/...`。
- 符号校验**硬失败**，无豁免机制。符号未命中即整页 FAIL，必须修文档或改指向真实定义文件（宏/内联通常在 `.h`）。
- 六个必含章节：`一句话职责`、`机制是什么`、`怎么实现的`、`关键取舍`、`源码落点`、`参考`。
- 一个条件章节：`与主核 / AI 的关系`（内容可为空，标题必须在）。
- `schema_version` 从 1 起，升版预算最多 3 次，每次升版必须记录「改了什么、为什么」。
- 所有脚本只用标准库，不引入 PyYAML / pytest 等外部依赖（沿用 `reference/scripts/sync_catalog.py` 的既有约定）。
- 测试框架为 `unittest`，在 `reference/scripts/` 目录下用 `python -m unittest <module> -v` 运行。
- 本计划**不修改** `reference/docs/db-wiki` 下任何现有 `.md` 内容；该目录的重写属 Plan C。
- 所有 commit message 用中文。
- 文件一律 UTF-8 无 BOM、LF 行尾。

---

## File Structure

| 文件 | 职责 |
|---|---|
| `reference/scripts/wiki_schema.py` | 契约常量（章节名、必填字段、页面类型、版本号） |
| `reference/scripts/wiki_frontmatter.py` | front-matter 极简解析器（受控 YAML 子集） |
| `reference/scripts/wiki_lint.py` | 结构门禁 CLI |
| `reference/scripts/wiki_refs.py` | 引用门禁 CLI + `--locate` 反查 |
| `reference/scripts/repo_cache.py` | 浅克隆缓存管理 |
| `reference/scripts/test_wiki_schema.py` | 常量自检 |
| `reference/scripts/test_wiki_frontmatter.py` | 解析器测试 |
| `reference/scripts/test_wiki_lint.py` | 结构门禁测试 |
| `reference/scripts/test_wiki_refs.py` | 引用门禁测试 |
| `reference/scripts/test_repo_cache.py` | 缓存测试 |
| `reference/scripts/test_wiki_schema_doc.py` | 契约文档与常量一致性测试 |
| `reference/docs/wiki-schema.md` | 契约文档（唯一权威的人类可读版） |
| `reference/.gitignore` | 追加 `.cache/` |

每个模块单一职责：常量模块不解析，解析模块不判合格，门禁模块不写文件，缓存模块不认识 wiki。

---

### Task 1: 契约常量 `wiki_schema.py`

**Files:**
- Create: `reference/scripts/wiki_schema.py`
- Test: `reference/scripts/test_wiki_schema.py`

**Interfaces:**
- Consumes: 无
- Produces: `SCHEMA_VERSION: int`、`REQUIRED_SECTIONS: tuple[str]`（6 项）、`CONDITIONAL_SECTIONS: tuple[str]`（1 项）、`REQUIRED_FIELDS: tuple[str]`、`INDEX_REQUIRED_FIELDS: tuple[str]`、`PAGE_TYPES: tuple[str]`、`COMPARE_DIR_NAME: str`、`required_fields_for(page_type: str) -> tuple[str]`

- [ ] **Step 1: Write the failing test**

```python
# reference/scripts/test_wiki_schema.py
# -*- coding: utf-8 -*-
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
import wiki_schema


class TestConstants(unittest.TestCase):
    def test_schema_version_is_positive_int(self):
        self.assertIsInstance(wiki_schema.SCHEMA_VERSION, int)
        self.assertGreaterEqual(wiki_schema.SCHEMA_VERSION, 1)

    def test_six_required_sections(self):
        self.assertEqual(len(wiki_schema.REQUIRED_SECTIONS), 6)

    def test_required_section_names_are_unique(self):
        self.assertEqual(len(set(wiki_schema.REQUIRED_SECTIONS)), 6)

    def test_exact_section_names(self):
        self.assertEqual(
            wiki_schema.REQUIRED_SECTIONS,
            ("一句话职责", "机制是什么", "怎么实现的", "关键取舍", "源码落点", "参考"),
        )

    def test_conditional_sections_not_in_required(self):
        for name in wiki_schema.CONDITIONAL_SECTIONS:
            self.assertNotIn(name, wiki_schema.REQUIRED_SECTIONS)

    def test_page_types(self):
        self.assertEqual(set(wiki_schema.PAGE_TYPES), {"overview", "feature", "compare"})

    def test_index_fields_include_stars_as_of(self):
        self.assertIn("stars_as_of", wiki_schema.INDEX_REQUIRED_FIELDS)

    def test_feature_fields_include_refs(self):
        self.assertIn("refs", wiki_schema.REQUIRED_FIELDS)

    def test_required_fields_for_feature(self):
        self.assertEqual(
            wiki_schema.required_fields_for("feature"), wiki_schema.REQUIRED_FIELDS
        )

    def test_required_fields_for_overview(self):
        self.assertEqual(
            wiki_schema.required_fields_for("overview"), wiki_schema.INDEX_REQUIRED_FIELDS
        )

    def test_required_fields_for_compare_includes_refs(self):
        self.assertIn("refs", wiki_schema.required_fields_for("compare"))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_wiki_schema -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_schema'`

- [ ] **Step 3: Write minimal implementation**

```python
# reference/scripts/wiki_schema.py
# -*- coding: utf-8 -*-
"""wiki 文档契约的常量定义。

这是全仓唯一权威：改这里的任何一项等于改契约，必须同时升 SCHEMA_VERSION，
并在 docs/wiki-schema.md 记一节「本次改了什么、为什么」。
设计文档 §5.4 把升版预算定为 3 次。
"""

SCHEMA_VERSION = 1

# 六个必含章节（缺一即 FAIL）。顺序即 _index.md 与 RAG 切块的默认顺序。
REQUIRED_SECTIONS = (
    "一句话职责",
    "机制是什么",
    "怎么实现的",
    "关键取舍",
    "源码落点",
    "参考",
)

# 一个条件章节：内容可为空，但标题必须在。
CONDITIONAL_SECTIONS = ("与主核 / AI 的关系",)

# 特性页（page_type=feature|compare）必填字段。
REQUIRED_FIELDS = (
    "project",
    "feature",
    "page_type",
    "schema_version",
    "refs",
)

# 项目总览页（page_type=overview，即 _index.md）必填字段。
INDEX_REQUIRED_FIELDS = (
    "project",
    "page_type",
    "display_name",
    "repo",
    "license",
    "language",
    "stars",
    "stars_as_of",
)

PAGE_TYPES = ("overview", "feature", "compare")

# page_type=compare 的页必须放在该名字的目录下。
COMPARE_DIR_NAME = "_compare"


def required_fields_for(page_type):
    """按页面类型返回必填字段。未知类型按特性页口径（从严）。"""
    if page_type == "overview":
        return INDEX_REQUIRED_FIELDS
    return REQUIRED_FIELDS
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_wiki_schema -v`
Expected: PASS — `Ran 11 tests ... OK`

- [ ] **Step 5: Commit**

```bash
git -C C:/code/reference add scripts/wiki_schema.py scripts/test_wiki_schema.py
git -C C:/code/reference commit -m "feat(wiki): 新增 wiki 契约常量模块，定义六必含章节与必填字段"
```

---

### Task 2: front-matter 解析器 `wiki_frontmatter.py`

受控 YAML 子集，只支持本契约用到的三种形态：`key: 标量`、`key:` + 缩进列表、列表项内嵌 `key: 标量` 与 `key: [a, b]` 行内列表。不支持多行块标量、锚点、嵌套列表——这些形态契约里不出现，出现即报错而不是静默误解析。

**Files:**
- Create: `reference/scripts/wiki_frontmatter.py`
- Test: `reference/scripts/test_wiki_frontmatter.py`

**Interfaces:**
- Consumes: 无
- Produces: `FrontmatterError(Exception)`、`parse_frontmatter(text: str) -> tuple[dict, str]`、`parse_scalar(v: str) -> str | int | list`

- [ ] **Step 1: Write the failing test**

```python
# reference/scripts/test_wiki_frontmatter.py
# -*- coding: utf-8 -*-
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_frontmatter import FrontmatterError, parse_frontmatter, parse_scalar

GOOD = """---
project: postgres
feature: buffer-pool
page_type: feature
schema_version: 1
stars: 13500
stars_as_of: 2026-11-22
refs:
  - path: src/backend/storage/buffer/bufmgr.c
    symbols: [ReadBufferExtended, PinBuffer, FlushBuffer]
  - path: src/backend/storage/buffer/freelist.c
    symbols: [StrategyGetBuffer]
---

# 正文从这里开始
"""


class TestParseFrontmatter(unittest.TestCase):
    def test_returns_meta_and_body(self):
        meta, body = parse_frontmatter(GOOD)
        self.assertIsInstance(meta, dict)
        self.assertIn("# 正文从这里开始", body)

    def test_scalar_fields(self):
        meta, _ = parse_frontmatter(GOOD)
        self.assertEqual(meta["project"], "postgres")
        self.assertEqual(meta["feature"], "buffer-pool")
        self.assertEqual(meta["page_type"], "feature")

    def test_pure_digits_become_int(self):
        meta, _ = parse_frontmatter(GOOD)
        self.assertEqual(meta["schema_version"], 1)
        self.assertEqual(meta["stars"], 13500)

    def test_date_like_stays_string(self):
        meta, _ = parse_frontmatter(GOOD)
        self.assertEqual(meta["stars_as_of"], "2026-11-22")

    def test_nested_list_of_maps(self):
        meta, _ = parse_frontmatter(GOOD)
        self.assertEqual(len(meta["refs"]), 2)
        self.assertEqual(meta["refs"][0]["path"], "src/backend/storage/buffer/bufmgr.c")
        self.assertEqual(
            meta["refs"][0]["symbols"],
            ["ReadBufferExtended", "PinBuffer", "FlushBuffer"],
        )
        self.assertEqual(meta["refs"][1]["symbols"], ["StrategyGetBuffer"])

    def test_missing_opening_fence(self):
        with self.assertRaises(FrontmatterError):
            parse_frontmatter("# 没有 front-matter\n")

    def test_missing_closing_fence(self):
        with self.assertRaises(FrontmatterError):
            parse_frontmatter("---\nproject: x\n")

    def test_blank_body_is_allowed(self):
        meta, body = parse_frontmatter("---\nproject: x\n---\n")
        self.assertEqual(meta["project"], "x")
        self.assertEqual(body.strip(), "")


class TestParseScalar(unittest.TestCase):
    def test_plain_string(self):
        self.assertEqual(parse_scalar("postgres"), "postgres")

    def test_inline_list(self):
        self.assertEqual(parse_scalar("[a, b, c]"), ["a", "b", "c"])

    def test_empty_inline_list(self):
        self.assertEqual(parse_scalar("[]"), [])

    def test_quoted_string_unwrapped(self):
        self.assertEqual(parse_scalar('"with: colon"'), "with: colon")

    def test_int_coercion(self):
        self.assertEqual(parse_scalar("7"), 7)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_wiki_frontmatter -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_frontmatter'`

- [ ] **Step 3: Write minimal implementation**

```python
# reference/scripts/wiki_frontmatter.py
# -*- coding: utf-8 -*-
"""front-matter 极简解析器。

只支持契约用到的受控 YAML 子集：
  key: 标量
  key:
    - 标量
    - key: 标量            ← 列表项内嵌标量
      symbols: [a, b, c]  ← 行内列表
不支持多行块标量（| / >）、锚点、行内映射。遇到不支持的形态一律抛
FrontmatterError，不静默误解析——契约里不出现这些，出现了就是写错了。
"""


class FrontmatterError(Exception):
    """front-matter 缺失或形态不受支持。"""


def parse_scalar(v):
    """把单个值转成 str / int / list。"""
    v = v.strip()
    if v.startswith("[") and v.endswith("]"):
        inner = v[1:-1].strip()
        if not inner:
            return []
        return [_unquote(x.strip()) for x in inner.split(",")]
    if len(v) >= 2 and v[0] == v[-1] and v[0] in ('"', "'"):
        return v[1:-1]
    if v.isdigit():
        return int(v)
    return v


def _unquote(v):
    if len(v) >= 2 and v[0] == v[-1] and v[0] in ('"', "'"):
        return v[1:-1]
    return v


def parse_frontmatter(text):
    """返回 (meta, body)。text 首行必须是 ---，且其后必须再有一行 ---。"""
    lines = text.replace("\r\n", "\n").split("\n")
    if not lines or lines[0].strip() != "---":
        raise FrontmatterError("缺少 front-matter 起始 ---（文件首行必须是 ---）")
    end = None
    for i in range(1, len(lines)):
        if lines[i].strip() == "---":
            end = i
            break
    if end is None:
        raise FrontmatterError("缺少 front-matter 结束 ---")
    return _parse_block(lines[1:end]), "\n".join(lines[end + 1:])


def _parse_block(block):
    meta = {}
    cur_key = None
    cur_item = None
    for raw in block:
        if not raw.strip() or raw.strip().startswith("#"):
            continue
        if "\t" in raw[: len(raw) - len(raw.lstrip())]:
            raise FrontmatterError("front-matter 缩进禁止使用制表符: %r" % raw)
        stripped = raw.lstrip(" ")
        indent = len(raw) - len(stripped)
        s = stripped

        if s.startswith("- "):
            if cur_key is None:
                raise FrontmatterError("列表项出现在键之前: %r" % raw)
            if not isinstance(meta.get(cur_key), list):
                raise FrontmatterError("键 %r 不是列表却出现列表项: %r" % (cur_key, raw))
            payload = s[2:].strip()
            if ":" in payload:
                k, _, v = payload.partition(":")
                if not k.strip():
                    raise FrontmatterError("列表项缺少键名: %r" % raw)
                cur_item = {k.strip(): parse_scalar(v)}
                meta[cur_key].append(cur_item)
            else:
                meta[cur_key].append(parse_scalar(payload))
                cur_item = None
            continue

        if ":" not in s:
            raise FrontmatterError("不是 key: value 形式: %r" % raw)
        k, _, v = s.partition(":")
        k = k.strip()
        if not k:
            raise FrontmatterError("键名为空: %r" % raw)
        v = v.strip()

        if indent == 0:
            cur_key = k
            cur_item = None
            if v == "":
                meta[k] = []
            else:
                meta[k] = parse_scalar(v)
                cur_key = None
        else:
            if cur_item is None:
                raise FrontmatterError("缩进键 %r 不在任何列表项内" % k)
            cur_item[k] = parse_scalar(v)

    return meta
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_wiki_frontmatter -v`
Expected: PASS — `Ran 13 tests ... OK`

- [ ] **Step 5: Commit**

```bash
git -C C:/code/reference add scripts/wiki_frontmatter.py scripts/test_wiki_frontmatter.py
git -C C:/code/reference commit -m "feat(wiki): 新增 front-matter 极简解析器，只支持契约用到的受控 YAML 子集"
```

---

### Task 3: 结构门禁 `wiki_lint.py`

**Files:**
- Create: `reference/scripts/wiki_lint.py`
- Test: `reference/scripts/test_wiki_lint.py`

**Interfaces:**
- Consumes: `wiki_schema.SCHEMA_VERSION`、`wiki_schema.REQUIRED_SECTIONS`、`wiki_schema.CONDITIONAL_SECTIONS`、`wiki_schema.PAGE_TYPES`、`wiki_schema.COMPARE_DIR_NAME`、`wiki_schema.required_fields_for(page_type)`、`wiki_frontmatter.parse_frontmatter`、`wiki_frontmatter.FrontmatterError`
- Produces: `lint_page(page_path: str) -> list[str]`（空列表 = 通过）、`main(argv=None) -> int`（0 全通过 / 1 有失败）

- [ ] **Step 1: Write the failing test**

```python
# reference/scripts/test_wiki_lint.py
# -*- coding: utf-8 -*-
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_lint import lint_page

GOOD_PAGE = """---
project: postgres
feature: buffer-pool
page_type: feature
schema_version: 1
refs:
  - path: src/backend/storage/buffer/bufmgr.c
    symbols: [ReadBufferExtended, PinBuffer]
---

# Buffer Pool 主体

## 一句话职责
页框缓存的主实现。

## 机制是什么
缓存页框，用 Clock-Sweep 置换。

## 怎么实现的
扫描指针推进，命中即置引用位。

## 关键取舍
放弃 LRU 的精确性，换取扫描友好的近似。

## 源码落点

| 文件 | 符号 | 作用 |
|---|---|---|
| `src/backend/storage/buffer/bufmgr.c` | `ReadBufferExtended` | 读页入口 |

## 与主核 / AI 的关系
对应 engine/db 的 pager 层。

## 参考
- [PostgreSQL buffer manager](https://www.postgresql.org/docs/current/buffer-manager.html)
- [PostgreSQL storage README](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob_plain;f=src/backend/storage/buffer/README)
"""


def _write(dirpath, name, content):
    path = os.path.join(dirpath, name)
    with open(path, "w", encoding="utf-8", newline="\n") as f:
        f.write(content)
    return path


class TestLintGoodPage(unittest.TestCase):
    def test_good_page_passes(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", GOOD_PAGE)
            self.assertEqual(lint_page(page), [])


class TestLintFailures(unittest.TestCase):
    def _page_with(self, dirpath, name, content):
        return _write(dirpath, name, content)

    def test_missing_frontmatter(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", "# 没有 front-matter\n")
            errors = lint_page(page)
            self.assertIn("front-matter 解析失败", errors[0])

    def test_missing_required_section(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace("## 关键取舍\n放弃 LRU 的精确性，换取扫描友好的近似。\n", "")
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("缺必含章节: ## 关键取舍" in e for e in errors), errors)

    def test_missing_conditional_section_heading(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace("## 与主核 / AI 的关系\n对应 engine/db 的 pager 层。\n", "")
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("缺条件章节标题" in e for e in errors), errors)

    def test_line_number_reference_rejected(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace("读页入口", "读页入口（bufmgr.c:1042）")
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("出现行号引用" in e for e in errors), errors)

    def test_line_number_reference_rejected_for_c_file(self):
        """裸 .c 是最容易被正则漏掉的扩展名（postgres/sqlite/redis 主力语言）。"""
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace("读页入口", "读页入口（bufmgr.c:1042）")
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("bufmgr.c:1042" in e for e in errors), errors)

    def test_feature_filename_mismatch(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", GOOD_PAGE.replace("feature: buffer-pool", "feature: bufferpool"))
            errors = lint_page(page)
            self.assertTrue(any("与文件名尾段" in e for e in errors), errors)

    def test_missing_required_field(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", GOOD_PAGE.replace("page_type: feature\n", ""))
            errors = lint_page(page)
            self.assertTrue(any("缺必填字段: page_type" in e for e in errors), errors)

    def test_missing_refs_field(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace(
                "refs:\n  - path: src/backend/storage/buffer/bufmgr.c\n    symbols: [ReadBufferExtended, PinBuffer]\n",
                "",
            )
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("缺必填字段: refs" in e for e in errors), errors)

    def test_only_one_reference_link(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace(
                "- [PostgreSQL storage README](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob_plain;f=src/backend/storage/buffer/README)\n",
                "",
            )
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("至少需要 2 个" in e for e in errors), errors)

    def test_bad_page_type(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", GOOD_PAGE.replace("page_type: feature", "page_type: article"))
            errors = lint_page(page)
            self.assertTrue(any("page_type 非法" in e for e in errors), errors)

    def test_compare_page_outside_compare_dir(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "_compare-qdrant-vs-milvus.md", GOOD_PAGE.replace("page_type: feature", "page_type: compare"))
            errors = lint_page(page)
            self.assertTrue(any("必须位于 _compare" in e for e in errors), errors)

    def test_version_mismatch(self):
        with tempfile.TemporaryDirectory() as d:
            page = _write(d, "a04-buffer-pool.md", GOOD_PAGE.replace("schema_version: 1", "schema_version: 99"))
            errors = lint_page(page)
            self.assertTrue(any("schema_version=99" in e for e in errors), errors)

    def test_empty_symbols_rejected(self):
        with tempfile.TemporaryDirectory() as d:
            content = GOOD_PAGE.replace("    symbols: [ReadBufferExtended, PinBuffer]", "    symbols: []")
            page = _write(d, "a04-buffer-pool.md", content)
            errors = lint_page(page)
            self.assertTrue(any("symbols 必须是非空列表" in e for e in errors), errors)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_wiki_lint -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_lint'`

- [ ] **Step 3: Write minimal implementation**

```python
# reference/scripts/wiki_lint.py
# -*- coding: utf-8 -*-
"""结构门禁：检查 wiki 页是否满足契约（设计文档 §5.1）。

只查结构与字段，不查源码引用是否真实——那是 wiki_refs.py 的职责。
退出码 0 = 全部通过，1 = 有任一页失败。可整目录递归。
"""
import argparse
import os
import re
import sys

import wiki_schema
from wiki_frontmatter import FrontmatterError, parse_frontmatter

# 禁止形态：<带扩展名的路径>:<数字>。命中即 FAIL。
# 扩展名顺序：长的排在短的之前（cxx 先于 cc，cpp 先于 c，hpp 先于 hh 先于 h），
# 否则短扩展名会抢先匹配掉长扩展名。裸 c 必须在内——postgres / sqlite / redis
# 的主力扩展名就是 .c，漏了它这条门禁在最该拦的地方失明（已由测试抓到过一次）。
_LINE_REF_RE = re.compile(
    r"([A-Za-z0-9_./\-]+\.(?:cxx|cpp|cc|c|hpp|hh|h|rs|go|java|kt|py|scala|sql|lua|rb|php|m|mm|swift)):(\d+(?:[,\-]\d+)*)"
)
# 参考章节的外部链接计数。只认 http(s)，不认相对路径。
_LINK_RE = re.compile(r"https?://")
_SECTION_RE = re.compile(r"^##\s+(.+?)\s*$", re.MULTILINE)
# 特性页文件名形如 a04-buffer-pool.md，取「首字母+编号+横线」之后的尾段。
_FILENAME_PREFIX_RE = re.compile(r"^[A-Za-z]\d+-(.+)$")
_MIN_REFERENCES = 2


def _split_sections(body):
    """返回 [(标题, 正文)]，标题不带 ## 前缀。"""
    out = []
    matches = list(_SECTION_RE.finditer(body))
    for i, m in enumerate(matches):
        start = m.end()
        end = matches[i + 1].start() if i + 1 < len(matches) else len(body)
        out.append((m.group(1).strip(), body[start:end]))
    return out


def _dirname_parts(path):
    return os.path.normpath(path).split(os.sep)


def _check_refs_shape(refs, errors):
    if not isinstance(refs, list):
        errors.append("refs 必须是列表")
        return
    for i, item in enumerate(refs):
        if not isinstance(item, dict):
            errors.append("refs[%d] 不是映射" % i)
            continue
        if not item.get("path"):
            errors.append("refs[%d].path 缺失" % i)
        syms = item.get("symbols")
        if not isinstance(syms, list) or not syms:
            errors.append("refs[%d].symbols 必须是非空列表" % i)


def lint_page(page_path):
    """返回错误列表，空列表表示通过。"""
    errors = []
    with open(page_path, encoding="utf-8") as f:
        text = f.read()

    try:
        meta, body = parse_frontmatter(text)
    except FrontmatterError as e:
        return ["front-matter 解析失败: %s" % e]

    page_type = meta.get("page_type")
    if page_type is not None and page_type not in wiki_schema.PAGE_TYPES:
        errors.append("page_type 非法: %r，允许 %s" % (page_type, list(wiki_schema.PAGE_TYPES)))

    for field in wiki_schema.required_fields_for(page_type):
        if field not in meta:
            errors.append("缺必填字段: %s" % field)

    if "schema_version" in meta and meta["schema_version"] != wiki_schema.SCHEMA_VERSION:
        errors.append(
            "schema_version=%r，当前契约版本为 %d" % (meta["schema_version"], wiki_schema.SCHEMA_VERSION)
        )

    if page_type == "compare":
        if wiki_schema.COMPARE_DIR_NAME not in _dirname_parts(page_path):
            errors.append("page_type=compare 的页必须位于 %s/ 下" % wiki_schema.COMPARE_DIR_NAME)

    feature = meta.get("feature")
    if "feature" in meta:
        if not isinstance(feature, str):
            errors.append("feature 必须是字符串，实际 %r" % (feature,))
        else:
            stem = os.path.splitext(os.path.basename(page_path))[0]
            m = _FILENAME_PREFIX_RE.match(stem)
            tail = m.group(1) if m else stem
            if tail != feature:
                errors.append("feature(%r) 与文件名尾段(%r) 不一致" % (feature, tail))

    if "refs" in meta:
        _check_refs_shape(meta["refs"], errors)

    sections = dict(_split_sections(body))
    for name in wiki_schema.REQUIRED_SECTIONS:
        if name not in sections:
            errors.append("缺必含章节: ## %s" % name)
    for name in wiki_schema.CONDITIONAL_SECTIONS:
        if name not in sections:
            errors.append("缺条件章节标题: ## %s" % name)

    for m in _LINE_REF_RE.finditer(body):
        errors.append("出现行号引用 %s:%s，契约禁止行号" % (m.group(1), m.group(2)))

    ref_body = sections.get("参考", "")
    n_links = len(_LINK_RE.findall(ref_body))
    if n_links < _MIN_REFERENCES:
        errors.append("参考章节外部链接 %d 个，至少需要 %d 个" % (n_links, _MIN_REFERENCES))

    return errors


def _iter_pages(root):
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = sorted(d for d in dirnames if not d.startswith("."))
        for fn in sorted(filenames):
            if fn.endswith(".md"):
                yield os.path.join(dirpath, fn)


def main(argv=None):
    parser = argparse.ArgumentParser(description="wiki 结构门禁（设计文档 §5.1）")
    parser.add_argument("paths", nargs="+", help="要检查的 .md 文件或目录（目录会递归）")
    parser.add_argument("--quiet", action="store_true", help="只打印失败项")
    args = parser.parse_args(argv)

    failed = 0
    checked = 0
    for target in args.paths:
        pages = list(_iter_pages(target)) if os.path.isdir(target) else [target]
        for page in pages:
            checked += 1
            errs = lint_page(page)
            if errs:
                failed += 1
                print("FAIL %s" % page)
                for e in errs:
                    print("     - %s" % e)
            elif not args.quiet:
                print("PASS %s" % page)

    print("-- 检查 %d 页，失败 %d 页" % (checked, failed))
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_wiki_lint -v`
Expected: PASS — `Ran 14 tests ... OK`

- [ ] **Step 5: Run the CLI against a real fixture**

先建一个脚手架外目录（`.scratch/` 已在 `code` 仓的 `.gitignore` 里，不会污染版本库）：

```powershell
New-Item -ItemType Directory -Force -Path C:/code/.scratch/wiki-lint-demo | Out-Null
```

把上面 `GOOD_PAGE` 的内容原样写入 `C:/code/.scratch/wiki-lint-demo/a04-buffer-pool.md`（UTF-8 无 BOM，LF 行尾），然后运行：

Run: `cd C:/code/reference/scripts; python wiki_lint.py C:/code/.scratch/wiki-lint-demo --quiet`
Expected: `-- 检查 1 页，失败 0 页`，退出码 0

- [ ] **Step 6: Commit**

```bash
git -C C:/code/reference add scripts/wiki_lint.py scripts/test_wiki_lint.py
git -C C:/code/reference commit -m "feat(wiki): 新增结构门禁 CLI，检查六必含章节、必填字段与禁止行号引用"
```

---

### Task 4: 引用门禁 `wiki_refs.py`

**Files:**
- Create: `reference/scripts/wiki_refs.py`
- Test: `reference/scripts/test_wiki_refs.py`

**Interfaces:**
- Consumes: `wiki_frontmatter.parse_frontmatter`、`wiki_frontmatter.FrontmatterError`
- Produces: `check_refs(page_path: str, repo_root: str) -> list[str]`、`locate_symbol(repo_root: str, symbol: str, exts=None, max_hits=20) -> list[tuple[str, int, str]]`、`main(argv=None) -> int`

- [ ] **Step 1: Write the failing test**

```python
# reference/scripts/test_wiki_refs.py
# -*- coding: utf-8 -*-
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_refs import check_refs, locate_symbol

PAGE = """---
project: demo
feature: buffer-pool
page_type: feature
schema_version: 1
refs:
  - path: src/bufmgr.c
    symbols: [ReadBufferExtended, PinBuffer]
  - path: src/freelist.c
    symbols: [StrategyGetBuffer]
---

# 占位
"""

BUFMGR = """/* 伪造的 bufmgr.c，用于测试符号匹配 */
void *ReadBufferExtended(void);
static void PinBuffer(void);
"""

FREELIST = """/* 伪造的 freelist.c */
int StrategyGetBuffer(int flags);
"""


def _write(path, content):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8", newline="\n") as f:
        f.write(content)
    return path


def _build_repo(root):
    _write(os.path.join(root, "src", "bufmgr.c"), BUFMGR)
    _write(os.path.join(root, "src", "freelist.c"), FREELIST)


class TestCheckRefs(unittest.TestCase):
    def test_all_refs_hit(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            page = _write(os.path.join(d, "a04-buffer-pool.md"), PAGE)
            _build_repo(repo)
            self.assertEqual(check_refs(page, repo), [])

    def test_missing_file(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            page = _write(os.path.join(d, "a04-buffer-pool.md"), PAGE)
            os.makedirs(repo)
            errors = check_refs(page, repo)
            self.assertTrue(any("不存在: src/bufmgr.c" in e for e in errors), errors)

    def test_missing_symbol_hard_fails(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            content = PAGE.replace("symbols: [ReadBufferExtended, PinBuffer]", "symbols: [ReadBufferExtended, NoSuchSymbol]")
            page = _write(os.path.join(d, "a04-buffer-pool.md"), content)
            errors = check_refs(page, repo)
            self.assertTrue(any("未命中: NoSuchSymbol" in e for e in errors), errors)

    def test_symbol_must_be_in_its_own_file(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            content = PAGE.replace("symbols: [StrategyGetBuffer]", "symbols: [ReadBufferExtended]")
            page = _write(os.path.join(d, "a04-buffer-pool.md"), content)
            errors = check_refs(page, repo)
            self.assertTrue(any("未命中: ReadBufferExtended (src/freelist.c)" in e for e in errors), errors)

    def test_no_refs_field(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            page = _write(
                os.path.join(d, "a04-buffer-pool.md"),
                PAGE.replace(
                    "refs:\n  - path: src/bufmgr.c\n    symbols: [ReadBufferExtended, PinBuffer]\n"
                    "  - path: src/freelist.c\n    symbols: [StrategyGetBuffer]\n",
                    "",
                ),
            )
            errors = check_refs(page, repo)
            self.assertTrue(any("refs 缺失或不是列表" in e for e in errors), errors)

    def test_utf8_content_with_bom_tolerated(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            page = _write(os.path.join(d, "a04-buffer-pool.md"), PAGE)
            path = os.path.join(repo, "src", "bufmgr.c")
            with open(path, "wb") as f:
                f.write(b"\xef\xbb\xbf" + BUFMGR.encode("utf-8"))
            self.assertEqual(check_refs(page, repo), [])


class TestLocateSymbol(unittest.TestCase):
    def test_finds_file_and_line(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            hits = locate_symbol(repo, "ReadBufferExtended")
            self.assertEqual(len(hits), 1)
            rel, lineno, text = hits[0]
            self.assertEqual(rel, "src/bufmgr.c")
            self.assertEqual(lineno, 2)

    def test_skips_dot_git(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            _write(os.path.join(repo, ".git", "fake.c"), "ReadBufferExtended\n")
            hits = locate_symbol(repo, "ReadBufferExtended")
            self.assertEqual([h[0] for h in hits], ["src/bufmgr.c"])

    def test_respects_max_hits(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            for i in range(5):
                _write(os.path.join(repo, "src", "f%d.c" % i), "int Dup(int x);\n")
            hits = locate_symbol(repo, "Dup", max_hits=2)
            self.assertEqual(len(hits), 2)

    def test_symbol_not_found_returns_empty(self):
        with tempfile.TemporaryDirectory() as d:
            repo = os.path.join(d, "repo")
            _build_repo(repo)
            self.assertEqual(locate_symbol(repo, "NoSuchSymbol"), [])


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_wiki_refs -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_refs'`

- [ ] **Step 3: Write minimal implementation**

```python
# reference/scripts/wiki_refs.py
# -*- coding: utf-8 -*-
"""引用门禁：refs 里的 path 必须存在，symbols 必须能在该文件内出现。

硬失败，无豁免（设计文档 §5.2）。宏与内联函数的正确做法是把 refs 指向
其真实定义文件（通常是 .h），而不是给校验加白名单。

--locate 是硬失败的反向工具：在本地浅克隆里全仓查符号，返回真实文件与行号，
让写作者先定位再填 refs，把门禁从惩罚变成反馈（设计文档 §5.3）。
"""
import argparse
import os
import re
import sys

from wiki_frontmatter import FrontmatterError, parse_frontmatter

# locate 默认扫描的源码扩展名。宽松一些，宁多勿漏。
CODE_EXTS = (
    ".c", ".cc", ".cpp", ".cxx", ".h", ".hpp", ".hh",
    ".go", ".rs", ".java", ".kt", ".scala", ".py", ".ts", ".pyx",
    ".sql", ".lua", ".rb", ".php", ".m", ".mm", ".swift",
)

# 全仓遍历时跳过的目录：这些内容没有引用价值，且体量大。
SKIP_DIRS = (".git", "build", "node_modules", "target", "vendor", "third_party", "thirdpart", "deps", "dist")


def _read_text(path):
    """按二进制读再解码，容忍非 UTF-8 与 BOM。源码里出现混合编码是常态。"""
    with open(path, "rb") as f:
        data = f.read()
    return data.decode("utf-8", errors="replace")


def check_refs(page_path, repo_root):
    """返回错误列表，空列表表示通过。repo_root 必须是该项目源码的本地根。"""
    with open(page_path, encoding="utf-8") as f:
        text = f.read()
    try:
        meta, _ = parse_frontmatter(text)
    except FrontmatterError as e:
        return ["front-matter 解析失败: %s" % e]

    refs = meta.get("refs")
    if not isinstance(refs, list):
        return ["refs 缺失或不是列表"]

    errors = []
    for i, item in enumerate(refs):
        if not isinstance(item, dict):
            errors.append("refs[%d] 不是映射" % i)
            continue
        rel = item.get("path")
        if not rel or not isinstance(rel, str):
            errors.append("refs[%d].path 缺失" % i)
            continue
        abs_path = os.path.join(repo_root, rel.replace("/", os.sep))
        if not os.path.isfile(abs_path):
            errors.append("refs[%d].path 不存在: %s" % (i, rel))
            continue
        syms = item.get("symbols")
        if not isinstance(syms, list) or not syms:
            errors.append("refs[%d].symbols 必须是非空列表" % i)
            continue
        content = _read_text(abs_path)
        for sym in syms:
            if not isinstance(sym, str) or not sym.strip():
                errors.append("refs[%d] 含非法符号 %r" % (i, sym))
                continue
            pat = re.compile(r"\b" + re.escape(sym.strip()) + r"\b")
            if pat.search(content) is None:
                errors.append("refs[%d].symbol 未命中: %s (%s)" % (i, sym.strip(), rel))
    return errors


def locate_symbol(repo_root, symbol, exts=CODE_EXTS, max_hits=20):
    """返回 [(相对路径, 首个命中行号, 该行文本)]，最多 max_hits 条。"""
    pat = re.compile(r"\b" + re.escape(symbol) + r"\b")
    hits = []
    for dirpath, dirnames, filenames in os.walk(repo_root):
        dirnames[:] = sorted(
            d for d in dirnames if d not in SKIP_DIRS and not d.startswith(".")
        )
        for fn in sorted(filenames):
            if not fn.endswith(exts):
                continue
            abs_path = os.path.join(dirpath, fn)
            try:
                text = _read_text(abs_path)
            except OSError:
                continue
            for lineno, line in enumerate(text.split("\n"), 1):
                if pat.search(line):
                    rel = os.path.relpath(abs_path, repo_root).replace(os.sep, "/")
                    hits.append((rel, lineno, line.strip()))
                    break
            if len(hits) >= max_hits:
                return hits
    return hits


def _iter_pages(root):
    for dirpath, dirnames, filenames in os.walk(root):
        dirnames[:] = sorted(d for d in dirnames if not d.startswith("."))
        for fn in sorted(filenames):
            if fn.endswith(".md"):
                yield os.path.join(dirpath, fn)


def main(argv=None):
    parser = argparse.ArgumentParser(description="wiki 引用门禁（设计文档 §5.2）")
    parser.add_argument("paths", nargs="*", help="要检查的 .md 文件或目录（目录会递归）")
    parser.add_argument("--repo-root", help="该项目源码的本地根目录（check 模式必填）")
    parser.add_argument("--locate", metavar="SYMBOL", help="全仓查找符号，打印真实文件与行号")
    parser.add_argument("--quiet", action="store_true", help="只打印失败项")
    args = parser.parse_args(argv)

    if args.locate:
        if not args.repo_root:
            parser.error("--locate 需要 --repo-root")
        hits = locate_symbol(args.repo_root, args.locate)
        if not hits:
            print("未找到符号 %s" % args.locate)
            return 1
        for rel, lineno, text in hits:
            print("%s:%d  %s" % (rel, lineno, text))
        return 0

    if not args.paths:
        parser.error("check 模式需要至少一个 paths")
    if not args.repo_root:
        parser.error("check 模式需要 --repo-root")
    if not os.path.isdir(args.repo_root):
        parser.error("--repo-root 不是目录: %s" % args.repo_root)

    failed = 0
    checked = 0
    for target in args.paths:
        pages = list(_iter_pages(target)) if os.path.isdir(target) else [target]
        for page in pages:
            checked += 1
            errs = check_refs(page, args.repo_root)
            if errs:
                failed += 1
                print("FAIL %s" % page)
                for e in errs:
                    print("     - %s" % e)
            elif not args.quiet:
                print("PASS %s" % page)

    print("-- 检查 %d 页，失败 %d 页" % (checked, failed))
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_wiki_refs -v`
Expected: PASS — `Ran 10 tests ... OK`

- [ ] **Step 5: Verify the gate catches a fabricated symbol on real source**

用真实源码验证门禁有效性——这一步是设计文档 §5.2 里那条实测的复现：

```bash
cd reference/scripts
python wiki_refs.py --locate ReadBufferExtended --repo-root <postgres 浅克隆路径>
```

Expected: 打印 `src/backend/storage/buffer/bufmgr.c:<行号>  ...`，退出码 0。
若尚未克隆 postgres（Plan A 不要求），跳过本步并在提交信息里注明「真源验证待 Plan C」。

- [ ] **Step 6: Commit**

```bash
git -C C:/code/reference add scripts/wiki_refs.py scripts/test_wiki_refs.py
git -C C:/code/reference commit -m "feat(wiki): 新增引用门禁 CLI，符号未命中硬失败，附 --locate 反查"
```

---

### Task 5: 源码缓存 `repo_cache.py`

**Files:**
- Create: `reference/scripts/repo_cache.py`
- Modify: `reference/.gitignore`（追加 `.cache/`）
- Test: `reference/scripts/test_repo_cache.py`

**Interfaces:**
- Consumes: 无
- Produces: `RepoCacheError(Exception)`、`repo_path(cache_dir: str, name: str) -> str`、`ensure_repo(repo_url: str, name: str, cache_dir=None, depth=1) -> str`、`DEFAULT_CACHE_DIR: str`

- [ ] **Step 1: Write the failing test**

测试用本地创建的 git 仓当上游，不依赖网络。

```python
# reference/scripts/test_repo_cache.py
# -*- coding: utf-8 -*-
import os
import subprocess
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from repo_cache import RepoCacheError, ensure_repo, repo_path

GIT_ENV = dict(
    os.environ,
    GIT_AUTHOR_NAME="wiki-test",
    GIT_AUTHOR_EMAIL="wiki-test@example.com",
    GIT_COMMITTER_NAME="wiki-test",
    GIT_COMMITTER_EMAIL="wiki-test@example.com",
)


def _git(args, cwd):
    return subprocess.run(
        ["git"] + args, cwd=cwd, capture_output=True, text=True, env=GIT_ENV
    )


def _make_upstream(root):
    """造一个带一次提交的本地上游仓，返回其 file:// URL。"""
    up = os.path.join(root, "upstream")
    os.makedirs(os.path.join(up, "src"))
    with open(os.path.join(up, "src", "a.c"), "w", encoding="utf-8", newline="\n") as f:
        f.write("int Alpha(void);\n")
    _git(["init", "-b", "main"], up)
    _git(["add", "."], up)
    _git(["commit", "-m", "init"], up)
    return "file:///" + up.replace(os.sep, "/").lstrip("/")


class TestEnsureRepo(unittest.TestCase):
    def test_clones_when_absent(self):
        with tempfile.TemporaryDirectory() as d:
            url = _make_upstream(d)
            cache = os.path.join(d, "cache")
            target = ensure_repo(url, "demo", cache_dir=cache)
            self.assertTrue(os.path.isdir(os.path.join(target, ".git")))
            self.assertTrue(os.path.isfile(os.path.join(target, "src", "a.c")))

    def test_idempotent_when_present(self):
        with tempfile.TemporaryDirectory() as d:
            url = _make_upstream(d)
            cache = os.path.join(d, "cache")
            first = ensure_repo(url, "demo", cache_dir=cache)
            mtime_before = os.path.getmtime(first)
            second = ensure_repo(url, "demo", cache_dir=cache)
            self.assertEqual(first, second)
            self.assertEqual(mtime_before, os.path.getmtime(second))

    def test_bad_url_raises(self):
        with tempfile.TemporaryDirectory() as d:
            cache = os.path.join(d, "cache")
            with self.assertRaises(RepoCacheError):
                ensure_repo("file:///no/such/repo.git", "bad", cache_dir=cache)

    def test_repo_path_is_join_of_cache_and_name(self):
        self.assertEqual(repo_path("/tmp/x", "demo"), os.path.join("/tmp/x", "demo"))


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_repo_cache -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'repo_cache'`

- [ ] **Step 3: Write minimal implementation**

```python
# reference/scripts/repo_cache.py
# -*- coding: utf-8 -*-
"""源码浅克隆缓存。

设计文档 §6.1：实测 raw 拉取 268 KB 需 3005 ms，270 库 × 平均 400 文件会变成
数十小时纯网络等待，因此改为每库整仓一次 --depth 1 克隆。

缓存目录默认 reference/.cache/repos/<name>，已在 .gitignore 里排除。
"""
import os
import subprocess

_HERE = os.path.dirname(os.path.abspath(__file__))
DEFAULT_CACHE_DIR = os.path.normpath(os.path.join(_HERE, "..", ".cache", "repos"))


class RepoCacheError(Exception):
    """克隆失败或缓存目录不可用。"""


def repo_path(cache_dir, name):
    """返回该项目在缓存里的目录路径（不保证存在）。"""
    return os.path.join(cache_dir, name)


def ensure_repo(repo_url, name, cache_dir=None, depth=1):
    """确保本地有一份该项目源码，返回目录路径。

    已存在则直接返回（幂等，不改动既有工作树）。失败抛 RepoCacheError。
    """
    cache_dir = cache_dir or DEFAULT_CACHE_DIR
    target = repo_path(cache_dir, name)
    if os.path.isdir(os.path.join(target, ".git")):
        return target

    os.makedirs(cache_dir, exist_ok=True)
    cmd = [
        "git", "clone",
        "--depth", str(depth),
        "--single-branch",
        repo_url,
        target,
    ]
    proc = subprocess.run(
        cmd, capture_output=True, text=True, encoding="utf-8", errors="replace"
    )
    if proc.returncode != 0:
        detail = (proc.stderr or proc.stdout or "").strip()
        raise RepoCacheError(
            "git clone 失败 (%s): %s" % (repo_url, detail[:500] or "rc=%d" % proc.returncode)
        )
    return target
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_repo_cache -v`
Expected: PASS — `Ran 4 tests ... OK`

- [ ] **Step 5: Add cache dir to gitignore**

在 `reference/.gitignore` 末尾追加 `.cache/`（若已存在则跳过）。先确认现状：

Run: `cd C:/code/reference; Get-Content .gitignore`
Expected: 输出当前内容，确认其中**没有** `.cache/`

然后追加（PowerShell 写法，保持 UTF-8 无 BOM 与 LF）：

```powershell
Add-Content -Path C:/code/reference/.gitignore -Value "`n# wiki 源码浅克隆缓存（repo_cache.py 生成，不入库）`n.cache/" -Encoding utf8NoBOM
```

Run: `cd C:/code/reference; git check-ignore -v .cache/repos/demo`
Expected: 输出匹配到 `.gitignore` 的行，退出码 0

- [ ] **Step 6: Commit**

```bash
git -C C:/code/reference add scripts/repo_cache.py scripts/test_repo_cache.py .gitignore
git -C C:/code/reference commit -m "feat(wiki): 新增源码浅克隆缓存模块，缓存目录排除出版本库"
```

---

### Task 6: 契约文档 `wiki-schema.md` 与一致性测试

文档是给人看的契约，常量是机器执行的契约。两者必须一致，因此加一条机械检查。

**Files:**
- Create: `reference/docs/wiki-schema.md`
- Test: `reference/scripts/test_wiki_schema_doc.py`

**Interfaces:**
- Consumes: `wiki_schema.REQUIRED_SECTIONS`、`wiki_schema.CONDITIONAL_SECTIONS`、`wiki_schema.REQUIRED_FIELDS`、`wiki_schema.INDEX_REQUIRED_FIELDS`、`wiki_schema.PAGE_TYPES`、`wiki_schema.SCHEMA_VERSION`
- Produces: 无（纯文档 + 一致性测试）

- [ ] **Step 1: Write the failing test**

```python
# reference/scripts/test_wiki_schema_doc.py
# -*- coding: utf-8 -*-
"""契约文档与常量必须一致：文档说漏了章节名，等于契约漂移。"""
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
import wiki_schema

SCHEMA_DOC = os.path.normpath(
    os.path.join(os.path.dirname(os.path.abspath(__file__)), "..", "docs", "wiki-schema.md")
)


class TestSchemaDocMatchesConstants(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        if not os.path.isfile(SCHEMA_DOC):
            raise AssertionError("wiki-schema.md 必须存在: %s" % SCHEMA_DOC)
        with open(SCHEMA_DOC, encoding="utf-8") as f:
            cls.doc = f.read()

    def test_doc_mentions_schema_version(self):
        self.assertIn("SCHEMA_VERSION = %d" % wiki_schema.SCHEMA_VERSION, self.doc)

    def test_doc_mentions_every_required_section(self):
        for name in wiki_schema.REQUIRED_SECTIONS:
            self.assertIn(name, self.doc, "契约文档漏了必含章节: %s" % name)

    def test_doc_mentions_conditional_section(self):
        for name in wiki_schema.CONDITIONAL_SECTIONS:
            self.assertIn(name, self.doc, "契约文档漏了条件章节: %s" % name)

    def test_doc_mentions_every_required_field(self):
        for field in wiki_schema.REQUIRED_FIELDS:
            self.assertIn(field, self.doc, "契约文档漏了必填字段: %s" % field)

    def test_doc_mentions_every_index_field(self):
        for field in wiki_schema.INDEX_REQUIRED_FIELDS:
            self.assertIn(field, self.doc, "契约文档漏了索引页字段: %s" % field)

    def test_doc_mentions_every_page_type(self):
        for t in wiki_schema.PAGE_TYPES:
            self.assertIn(t, self.doc, "契约文档漏了页面类型: %s" % t)

    def test_doc_states_no_line_numbers_rule(self):
        self.assertIn("禁止行号", self.doc)

    def test_doc_states_hard_failure_rule(self):
        self.assertIn("硬失败", self.doc)

    def test_doc_states_bump_budget(self):
        self.assertIn("3 次", self.doc)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd reference/scripts && python -m unittest test_wiki_schema_doc -v`
Expected: FAIL — `AssertionError: wiki-schema.md 必须存在`（红灯：程序不应把缺少契约文档当作“跳过”）

- [ ] **Step 3: Write the contract document**

```markdown
# wiki 文档契约 v1

> `SCHEMA_VERSION = 1`
> 本文件是契约的人类可读版；机器可读版是 `scripts/wiki_schema.py`。
> 两者由 `scripts/test_wiki_schema_doc.py` 保证不漂移。
> 设计依据：`../docs/superpowers/specs/2026-11-22-multimodal-wiki-and-rag-design.md`

## 1. 一条铁律：禁止行号

源码引用一律写「相对项目仓根的路径 + 符号名」，**禁止行号**。

行号腐化的实测依据：`reference/reference/catalog.yml` 已记录 5/68 篇的引用失效
（`time-series/tdengine.md` 单篇 75 处），腐化率 7.3%。乘以 2.5 万篇的规模不成立。

正确写法：`src/backend/storage/buffer/bufmgr.c` + `ReadBufferExtended`
错误写法：`src/backend/storage/buffer/bufmgr.c:1042`

## 2. 校验是硬失败

`scripts/wiki_refs.py` 对每个 `refs[].path`：

1. 在本地浅克隆中确认文件存在 —— 不存在即 FAIL
2. 对 `refs[].symbols` 逐个在该文件内匹配 —— **任一未命中即 FAIL**

**无豁免机制。** 宏与 `static inline` 函数的正确做法是把 refs 指向其真实定义文件
（通常是 `.h`），而不是给校验加白名单。

## 3. 页面类型（page_type）

| 值 | 用途 | 位置 |
|---|---|---|
| `overview` | 项目总览页，即 `_index.md` | 项目目录根部 |
| `feature` | 一个机制单元一篇 | 项目目录下 |
| `compare` | 跨项目对比页 | 必须在 `_compare/` 下 |

## 4. front-matter 字段

### 4.1 特性页必填（page_type = feature / compare）

| 字段 | 说明 |
|---|---|
| `project` | 项目唯一键，跨类不重名 |
| `feature` | 特性唯一键，必须与文件名尾段一致（`a04-buffer-pool.md` → `buffer-pool`） |
| `page_type` | 见 §3 |
| `schema_version` | 当前为 1 |
| `refs` | 结构化引用列表，校验脚本只读此字段 |

`refs` 单项结构：

```yaml
refs:
  - path: src/backend/storage/buffer/bufmgr.c
    symbols: [ReadBufferExtended, PinBuffer, FlushBuffer]
```

### 4.2 总览页必填（page_type = overview）

| 字段 | 说明 |
|---|---|
| `project` | 项目唯一键 |
| `display_name` | 展示名 |
| `repo` | 上游仓库 URL |
| `license` | 许可证标识 |
| `language` | 主要开发语言 |
| `stars` | 取数时的 star 数 |
| `stars_as_of` | **取数日期，必填** —— star 会变，一年后需能解释分层依据 |

## 5. 章节契约

六个必含章节（缺一即 FAIL）+ 一个条件章节（内容可为空，标题必须在）。

| 章节 | 要求 |
|---|---|
| `一句话职责` | 一句话，不含术语堆砌 |
| `机制是什么` | 解决什么、不解决什么、边界在哪；与相邻机制的上游/下游关系 |
| `怎么实现的` | 核心流程、关键数据结构、并发与生命周期。**不设字数上限** |
| `关键取舍` | 放弃了什么、为什么；至少一个「换成 XXX 方案会怎样」 |
| `源码落点` | 路径 + 符号名的表格，**不写行号** |
| `参考` | 论文 / 官方文档 / benchmark，**至少 2 个**外部链接 |
| `与主核 / AI 的关系` | 条件章节，标题必须在，内容可为空 |

完整样例见 `scripts/test_wiki_lint.py` 的 `GOOD_PAGE` 常量。

## 6. 升版预算

`schema_version` 从 1 起，**最多升 3 次**。每次升版必须在本文件追加一节：

```
## v2 变更（YYYY-MM-DD）
- 改了什么：
- 为什么：
```

已归档的项目目录不回溯，只在其 `_index.md` 标记自身 `schema_version`。
第 4 次想升版说明前 3 次没改对，必须停下重审契约。

## 7. 工具

| 命令 | 作用 |
|---|---|
| `python wiki_lint.py <path>...` | 结构门禁：章节、字段、禁止行号、参考数量 |
| `python wiki_refs.py <path>... --repo-root <dir>` | 引用门禁：路径存在 + 符号命中，硬失败 |
| `python wiki_refs.py --locate <SYMBOL> --repo-root <dir>` | 反查符号真实位置，先定位再填 refs |

三个命令退出码 0 = 通过，1 = 有失败，可直接接 CI 与并行流水线。
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd reference/scripts && python -m unittest test_wiki_schema_doc -v`
Expected: PASS — `Ran 9 tests ... OK`

- [ ] **Step 5: Commit**

```bash
git -C C:/code/reference add docs/wiki-schema.md scripts/test_wiki_schema_doc.py
git -C C:/code/reference commit -m "docs(wiki): 新增契约文档 v1，并以测试保证文档与常量不漂移"
```

---

### Task 7: 全套回归与真实源码验证

**Files:**
- Modify: 无

**Interfaces:**
- Consumes: Task 1–6 全部模块
- Produces: 一份可复现的回归结论

- [ ] **Step 1: Run the full test suite**

Run: `cd reference/scripts && python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint test_wiki_refs test_repo_cache test_wiki_schema_doc -v`
Expected: `Ran 61 tests ... OK`（11 + 13 + 14 + 10 + 4 + 9，且 9 项契约文档测试必须执行、不得 skip）

- [ ] **Step 2: Confirm the pre-existing suite still passes**

Run: `cd reference/scripts && python -m unittest test_sync_catalog -v`
Expected: `Ran 41 tests ... OK` —— 本计划不得破坏既有脚本

- [ ] **Step 3: Clone one real repo and verify the gate end-to-end**

```bash
cd reference/scripts
python -c "from repo_cache import ensure_repo; print(ensure_repo('https://github.com/postgres/postgres.git', 'postgres'))"
```

Expected: 打印本地路径，目录内含 `.git` 与 `src/backend/`

- [ ] **Step 4: Verify --locate against real PostgreSQL source**

```bash
cd reference/scripts
python wiki_refs.py --locate ReadBufferExtended --repo-root ../.cache/repos/postgres
```

Expected: 输出 `src/backend/storage/buffer/bufmgr.c:<行号>  ...`，退出码 0

- [ ] **Step 5: Verify a fabricated symbol is rejected**

新建 `C:/code/.scratch/wiki-gate-demo/a04-buffer-pool.md`，内容为 Task 3 的 `GOOD_PAGE`，
但把 `refs` 第一条的 `symbols` 改成虚构符号 `BufferGetHashEntry`，然后运行：

Run: `cd C:/code/reference/scripts; python wiki_refs.py C:/code/.scratch/wiki-gate-demo/a04-buffer-pool.md --repo-root ../.cache/repos/postgres`
Expected: `FAIL`，输出含 `symbol 未命中: BufferGetHashEntry`，退出码 1

这一步复现设计文档 §5.2 的实测：`bufmgr.c` 中 9 个真实符号全命中，虚构符号被拒。

- [ ] **Step 6: Commit regression note**

```bash
git -C C:/code/reference commit --allow-empty -m "test(wiki): 工装全套回归通过（61 项），真实源码下 --locate 与硬失败均按预期工作"
```

---

## 交付物清单

Plan A 完成后，以下能力可用：

| 能力 | 入口 |
|---|---|
| 契约常量 | `wiki_schema.py` |
| front-matter 解析 | `wiki_frontmatter.parse_frontmatter` |
| 结构判定 | `python wiki_lint.py <path>` |
| 引用判定（硬失败） | `python wiki_refs.py <path> --repo-root <dir>` |
| 符号反查 | `python wiki_refs.py --locate <SYM> --repo-root <dir>` |
| 源码缓存 | `repo_cache.ensure_repo(url, name)` |
| 契约文档 | `docs/wiki-schema.md` |

后续计划的依赖关系：

- **Plan B（P1 清单）** 依赖 `wiki_schema` 的字段定义与 `repo_cache` 的克隆能力
- **Plan C（P2 postgres 332 篇）** 依赖 `wiki_lint` + `wiki_refs` 双门禁
- **Plan D（P5 RAG）** 依赖 `wiki_frontmatter` 读取元数据做增量索引
