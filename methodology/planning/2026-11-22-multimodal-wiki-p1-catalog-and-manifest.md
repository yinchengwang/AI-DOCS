# wiki P1（项目清单与 postgres manifest）Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **执行后注记（2026-11-22）**
>
> 本计划原文按 `reference/` 仓书写，执行后产物已迁至 **`projects` 仓**：
>
> | 原路径 | 现路径 |
> |---|---|
> | `reference/scripts/wiki_catalog.py` 等 4 个模块及其测试 | `projects/scripts/wiki/` |
> | `reference/docs/wiki-schema.md` | `projects/docs/wiki/wiki-schema.md` |
> | `reference/.cache/repos/` | `projects/.cache/repos/` |
>
> **两处与本计划原文不符的实测结果：**
>
> 1. **清单规模**：Task 6 的目标是「97 → 130+ 条」，实际产出 **152 条**
>    （T1 39 / T2 85 / T3 28），未达设计文档的 270 条目标；缺口 118 条已登记为未决项，
>    不以凑数方式补齐。
> 2. **postgres manifest**：Task 7 产出的是**按文件体积生成的 332 篇**，
>    其中 259 篇（78.0%）标题是「<文件名> 模块」兜底，不是机制。
>    该生成器已在 **P2a 被整体废弃**并替换为机制优先的实现：
>    `projects/scripts/wiki/postgres_mechanism_specs.py`（151 条 spec，唯一数据源）
>    + `gen_postgres_manifest.py`（由 spec 生成 manifest，`check_specs()` 自检）。
>    最终清单为 **151 页 / 12 组**（A20 B19 C15 D17 E13 F13 G7 H5 I10 J12 K11 L9）。
>
> **下文的任务步骤保留原始路径与原始数字，作为执行记录，不做追溯改写。**

**Goal:** 建成项目清单（270 库分层）与机制页 manifest 的生成、校验工具链，并用 postgres 产出 332 篇真实 manifest，作为后续批量写作的输入契约。

**Architecture:** 四个新增模块各担一职：`wiki_catalog` 只读 catalog 并做分层判定，`wiki_manifest` 只做 manifest 的读写与校验，`wiki_manifest_lint` 只做 manifest 门禁 CLI，`wiki_tree` 只做源码目录树到机制单元的枚举。所有模块沿用 P0 的「标准库 + unittest + 硬门禁」风格，禁止引入第三方依赖。

**Tech Stack:** Python 3.11 标准库（`argparse`/`os`/`re`/`json`/`unittest`）、git 命令行、GitHub REST API（可选，用于 stars 取数）。

设计规格：`C:\code\ai-docs\methodology\planning\2026-11-22-multimodal-wiki-and-rag-design.md`
P0 工装（已就位）：`projects/scripts/wiki/`、`projects/docs/wiki/wiki-schema.md`

## Global Constraints

- 文档源格式只允许 Markdown；禁止 MediaWiki 语法（`==标题==`、`[[链接]]`、`{|` 表格）。
- **禁止行号引用**。引用一律「相对项目仓根的路径 + 符号名」，不得出现 `path:123` 形态，扩展名白名单见 `wiki_lint._LINE_REF_RE`。
- refs 路径基准 = 项目仓根（如 `src/backend/storage/buffer/bufmgr.c`）。
- 符号校验**硬失败**，无豁免机制。
- 六个必含章节：`一句话职责`、`机制是什么`、`怎么实现的`、`关键取舍`、`源码落点`、`参考`。
- 一个条件章节：`与主核 / AI 的关系`（内容可为空，标题必须在）。
- `schema_version` 从 1 起，升版预算最多 3 次。
- 只允许 Python 标准库；**不引入 PyYAML**（`wiki_frontmatter` 已实现受控 YAML 子集解析，manifest 用它读写）。
- 测试框架 `unittest`；命令在 `projects/scripts/wiki/` 下运行。
- 所有 commit message 用中文。
- 文件一律 UTF-8 无 BOM、LF 行尾。
- 清单条目必须记录 `stars_as_of`（取数日期）——star 数会变，一年后需能解释分层依据。
- 缓存目录为 `projects/.cache/repos/<name>`，已被 `.gitignore` 的 `.cache/` 排除，不得入库。
- 本计划**不写任何 wiki 正文页**（`.md` 特性页属 Plan C）；只产出清单与 manifest。

---

## File Structure

| 文件 | 职责 |
|---|---|
| `projects/scripts/wiki/wiki_catalog.py` | 读 `catalog.yml` 支持的字段子集，产出项目清单条目 |
| `projects/scripts/wiki/wiki_manifest.py` | manifest 的读取、写入、结构校验 |
| `projects/scripts/wiki/wiki_manifest_lint.py` | manifest 门禁 CLI（退出码 0/1） |
| `projects/scripts/wiki/wiki_tree.py` | 源码目录树 → 机制单元候选枚举 |
| `projects/scripts/wiki/test_wiki_catalog.py` | 清单解析测试 |
| `projects/scripts/wiki/test_wiki_manifest.py` | manifest 读写测试 |
| `projects/scripts/wiki/test_wiki_manifest_lint.py` | manifest 门禁测试 |
| `projects/scripts/wiki/test_wiki_tree.py` | 目录枚举测试 |
| `projects/docs/wiki/catalog-projects.yml` | 新的项目清单（270 库，本计划产出） |
| `projects/docs/wiki/projects/postgres/_manifest.yml` | postgres 的 332 篇机制单元清单 |

**为什么清单不复用 `reference/reference/catalog.yml`：** 旧清单是「第三方源码镜像清单」（97 条，字段含 `commit`/`pin_status`/`source`/`mirror`），语义是**克隆策略**；新清单是「文档生产清单」,语义是**写哪些库、写多深**。两者字段不同、生命周期不同，混用会让 `sync_catalog.py` 的 pin 校验逻辑与文档清单互相污染。新清单独立成文件，只从旧清单继承 `name`/`repository`/`license`/`model`/`tier` 五个字段。

---

### Task 1: 清单解析 `wiki_catalog.py`

**Files:**
- Create: `projects/scripts/wiki/wiki_catalog.py`
- Test: `projects/scripts/wiki/test_wiki_catalog.py`

**Interfaces:**
- Consumes: `wiki_frontmatter.parse_frontmatter`（不支持——catalog 不是 front-matter 格式，改用自写的简单行解析）
- Produces: `CATALOG_FIELDS: tuple[str, ...]`、`CatalogError(Exception)`、`parse_catalog(text: str) -> list[dict]`、`dump_catalog(entries: list[dict]) -> str`、`validate_entry(entry: dict) -> list[str]`

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_wiki_catalog.py
# -*- coding: utf-8 -*-
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_catalog import (CATALOG_FIELDS, CatalogError, dump_catalog,
                          parse_catalog, validate_entry)

SAMPLE = """projects:
  - name: postgres
    model: relational
    tier: T1
    repo: https://github.com/postgres/postgres
    license: PostgreSQL
    language: C
    stars: 13500
    stars_as_of: 2026-11-22
  - name: qdrant
    model: vector
    tier: T1
    repo: https://github.com/qdrant/qdrant
    license: Apache-2.0
    language: Rust
    stars: 35659
    stars_as_of: 2026-11-22
"""


class TestParseCatalog(unittest.TestCase):
    def test_parses_two_entries(self):
        entries = parse_catalog(SAMPLE)
        self.assertEqual(len(entries), 2)
        self.assertEqual(entries[0]["name"], "postgres")
        self.assertEqual(entries[1]["name"], "qdrant")

    def test_scalar_types(self):
        entries = parse_catalog(SAMPLE)
        pg = entries[0]
        self.assertEqual(pg["model"], "relational")
        self.assertEqual(pg["tier"], "T1")
        self.assertEqual(pg["stars"], 13500)
        self.assertEqual(pg["stars_as_of"], "2026-11-22")

    def test_missing_projects_key_raises(self):
        with self.assertRaises(CatalogError):
            parse_catalog("items:\n  - name: x\n")

    def test_empty_catalog_is_allowed(self):
        self.assertEqual(parse_catalog("projects:\n"), [])

    def test_duplicate_name_raises(self):
        dup = SAMPLE + """  - name: postgres
    model: relational
"""
        with self.assertRaises(CatalogError):
            parse_catalog(dup)


class TestDumpCatalog(unittest.TestCase):
    def test_round_trip(self):
        entries = parse_catalog(SAMPLE)
        text = dump_catalog(entries)
        again = parse_catalog(text)
        self.assertEqual(entries, again)

    def test_dump_is_lf_and_has_header(self):
        text = dump_catalog(parse_catalog(SAMPLE))
        self.assertNotIn("\r", text)
        self.assertTrue(text.startswith("projects:\n"))


class TestValidateEntry(unittest.TestCase):
    def test_valid_entry_has_no_errors(self):
        self.assertEqual(validate_entry(parse_catalog(SAMPLE)[0]), [])

    def test_missing_field_reported(self):
        bad = dict(parse_catalog(SAMPLE)[0])
        bad.pop("stars_as_of")
        errors = validate_entry(bad)
        self.assertTrue(any("stars_as_of" in e for e in errors), errors)

    def test_tier_must_be_t1_t2_t3(self):
        bad = dict(parse_catalog(SAMPLE)[0])
        bad["tier"] = "core"
        errors = validate_entry(bad)
        self.assertTrue(any("tier" in e for e in errors), errors)

    def test_repo_must_be_http_url(self):
        bad = dict(parse_catalog(SAMPLE)[0])
        bad["repo"] = "git@github.com:x/y.git"
        errors = validate_entry(bad)
        self.assertTrue(any("repo" in e for e in errors), errors)

    def test_catalog_fields_constant_is_complete(self):
        self.assertIn("stars_as_of", CATALOG_FIELDS)
        self.assertIn("tier", CATALOG_FIELDS)
        self.assertEqual(len(CATALOG_FIELDS), 8)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_catalog -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_catalog'`

- [ ] **Step 3: Write minimal implementation**

```python
# projects/scripts/wiki/wiki_catalog.py
# -*- coding: utf-8 -*-
"""文档生产清单（catalog）的解析与校验。

与 reference 仓的 `reference/catalog.yml` 是两份不同的东西：那份是**第三方源码镜像
清单**（字段含 commit / pin_status / source / mirror，语义是「怎么克隆」）；本文件是
**文档生产清单**（语义是「写哪些库、写多深」）。两者生命周期不同，不合并。

只用标准库：清单结构比 catalog 更规整（两层、无行内列表、无嵌套映射），
手写解析比引入 YAML 依赖更符合本仓既有约定（P0 已确立零第三方依赖）。
"""
import re

CATALOG_FIELDS = (
    "name",
    "model",
    "tier",
    "repo",
    "license",
    "language",
    "stars",
    "stars_as_of",
)

VALID_TIERS = ("T1", "T2", "T3")
_INT_RE = re.compile(r"[0-9]+")
_ENTRY_RE = re.compile(r"^\s{2}-\s+name:\s*(.+?)\s*$")
_FIELD_RE = re.compile(r"^\s{4}([A-Za-z_]+):\s*(.*?)\s*$")


class CatalogError(Exception):
    """清单缺失、结构非法或条目重复。"""


def parse_catalog(text):
    """返回条目列表。空清单返回 []。"""
    lines = text.replace("\r\n", "\n").split("\n")
    start = None
    for i, line in enumerate(lines):
        if line.strip() == "projects:":
            start = i
            break
    if start is None:
        raise CatalogError("缺少顶层 projects: 键")

    entries = []
    current = None
    for raw in lines[start + 1:]:
        if not raw.strip() or raw.strip().startswith("#"):
            continue
        m = _ENTRY_RE.match(raw)
        if m:
            current = {"name": m.group(1).strip()}
            entries.append(current)
            continue
        m = _FIELD_RE.match(raw)
        if m:
            if current is None:
                raise CatalogError("字段出现在任何条目之前: %r" % raw)
            current[m.group(1)] = _coerce(m.group(2))
            continue
        raise CatalogError("无法解析的行: %r" % raw)

    seen = set()
    for entry in entries:
        name = entry["name"]
        if name in seen:
            raise CatalogError("条目重复: %s" % name)
        seen.add(name)
    return entries


def _coerce(value):
    """整数字面量转 int，其余保持 str。日期 `2026-11-22` 不是纯整数，自然保持 str。"""
    if _INT_RE.fullmatch(value):
        return int(value)
    return value


def dump_catalog(entries):
    """按固定字段顺序输出，未提供的字段跳过。返回 LF 文本。"""
    out = ["projects:"]
    for entry in entries:
        out.append("  - name: %s" % entry["name"])
        for field in CATALOG_FIELDS:
            if field == "name" or field not in entry:
                continue
            out.append("    %s: %s" % (field, entry[field]))
    return "\n".join(out) + "\n"


def validate_entry(entry):
    """返回错误列表，空列表表示通过。"""
    errors = []
    for field in CATALOG_FIELDS:
        if field not in entry:
            errors.append("缺字段: %s" % field)

    tier = entry.get("tier")
    if tier is not None and tier not in VALID_TIERS:
        errors.append("tier 必须是 %s 之一，实际 %r" % (list(VALID_TIERS), tier))

    repo = entry.get("repo")
    if isinstance(repo, str) and not repo.startswith(("https://", "http://")):
        errors.append("repo 必须是 http(s) URL，实际 %r" % repo)

    stars = entry.get("stars")
    if stars is not None and not isinstance(stars, int):
        errors.append("stars 必须是整数，实际 %r" % (stars,))

    as_of = entry.get("stars_as_of")
    if isinstance(as_of, str) and not re.fullmatch(r"\d{4}-\d{2}-\d{2}", as_of):
        errors.append("stars_as_of 必须是 YYYY-MM-DD，实际 %r" % as_of)

    return errors
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_catalog -v`
Expected: PASS — `Ran 12 tests ... OK`

- [ ] **Step 5: Commit**

```bash
cd C:/code/projects
git add scripts/wiki/wiki_catalog.py scripts/wiki/test_wiki_catalog.py
git commit -m "feat(wiki): 新增文档生产清单解析与校验模块"
git push origin main
```

---

### Task 2: manifest 读写 `wiki_manifest.py`

**Files:**
- Create: `projects/scripts/wiki/wiki_manifest.py`
- Test: `projects/scripts/wiki/test_wiki_manifest.py`

**Interfaces:**
- Consumes: `wiki_frontmatter.parse_frontmatter`（manifest 用 front-matter 形态，因为它需要嵌套的 `pages[].refs[].symbols`）
- Produces: `ManifestError(Exception)`、`MANIFEST_FILENAME: str`（值 `_manifest.yml`）、`load_manifest(path: str) -> dict`、`dump_manifest(manifest: dict) -> str`、`manifest_feature_slugs(manifest: dict) -> list[str]`

**关键设计：** manifest 用 front-matter 格式（`---` 包裹），因为 P0 已有一个经过 51 项测试的解析器能正确处理嵌套 `path` + `symbols` 列表，且 ManifestError 上的错误消息可直接复用。这避免了为清单再造一套 YAML 解析。

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_wiki_manifest.py
# -*- coding: utf-8 -*-
import io
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_manifest import (MANIFEST_FILENAME, ManifestError, dump_manifest,
                           load_manifest, manifest_feature_slugs)

MANIFEST = """---
project: postgres
schema_version: 1
modules:
  - path: src/backend/storage/buffer
    source_files: 8
pages:
  - feature: buffer-pool
    feature_group: A-存储基础
    title: Buffer Pool 主体
    page_file: a04-buffer-pool.md
    refs:
      - path: src/backend/storage/buffer/bufmgr.c
        symbols: [ReadBufferExtended, PinBuffer]
  - feature: freelist
    feature_group: A-存储基础
    title: 空闲页框链表
    page_file: a05-freelist.md
    refs:
      - path: src/backend/storage/buffer/freelist.c
        symbols: [StrategyGetBuffer]
---
"""


class TestLoadManifest(unittest.TestCase):
    def test_loads_project_and_version(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(MANIFEST)
            m = load_manifest(path)
        self.assertEqual(m["project"], "postgres")
        self.assertEqual(m["schema_version"], 1)

    def test_loads_pages_with_refs(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(MANIFEST)
            m = load_manifest(path)
        self.assertEqual(len(m["pages"]), 2)
        first = m["pages"][0]
        self.assertEqual(first["feature"], "buffer-pool")
        self.assertEqual(first["page_file"], "a04-buffer-pool.md")
        self.assertEqual(first["refs"][0]["path"], "src/backend/storage/buffer/bufmgr.c")
        self.assertEqual(first["refs"][0]["symbols"], ["ReadBufferExtended", "PinBuffer"])

    def test_missing_file_raises(self):
        with self.assertRaises(ManifestError):
            load_manifest(os.path.join(tempfile.gettempdir(), "no-such-manifest.yml"))

    def test_not_a_mapping_raises(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write("no frontmatter here\n")
            with self.assertRaises(ManifestError):
                load_manifest(path)


class TestDumpManifest(unittest.TestCase):
    def test_round_trip_preserves_pages(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(MANIFEST)
            original = load_manifest(path)
        text = dump_manifest(original)
        self.assertNotIn("\r", text)
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(text)
            again = load_manifest(path)
        self.assertEqual(again["project"], original["project"])
        self.assertEqual(len(again["pages"]), len(original["pages"]))
        self.assertEqual(
            again["pages"][0]["refs"][0]["symbols"],
            original["pages"][0]["refs"][0]["symbols"],
        )

    def test_dump_ends_with_single_newline(self):
        text = dump_manifest({"project": "x", "schema_version": 1, "pages": []})
        self.assertTrue(text.endswith("\n"))
        self.assertFalse(text.endswith("\n\n"))


class TestFeatureSlugs(unittest.TestCase):
    def test_returns_all_slugs(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, MANIFEST_FILENAME)
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(MANIFEST)
            m = load_manifest(path)
        self.assertEqual(manifest_feature_slugs(m), ["buffer-pool", "freelist"])

    def test_empty_pages_returns_empty_list(self):
        self.assertEqual(manifest_feature_slugs({"pages": []}), [])


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_manifest -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_manifest'`

- [ ] **Step 3: Write minimal implementation**

```python
# projects/scripts/wiki/wiki_manifest.py
# -*- coding: utf-8 -*-
"""机制页清单（manifest）的读写。

manifest 描述「一个项目要写哪些机制页」——它是清单与写作之间的唯一接口：
Phase 1 产 manifest，Phase 2 的 writer 只读自己 manifest 条目内的页，不自由发挥。

格式复用 front-matter：P0 的 `wiki_frontmatter` 已能正确解析
`pages[].refs[].symbols` 这种两层嵌套，且有 51 项测试兜底；为 manifest
另造一套 YAML 解析是重复劳动。
"""
import io
import os

from wiki_frontmatter import FrontmatterError, parse_frontmatter

MANIFEST_FILENAME = "_manifest.yml"


class ManifestError(Exception):
    """manifest 缺失、格式非法或结构不合法。"""


def load_manifest(path):
    """读取 manifest 并返回 dict。文件不存在或格式非法抛 ManifestError。"""
    if not os.path.isfile(path):
        raise ManifestError("manifest 不存在: %s" % path)
    try:
        with io.open(path, encoding="utf-8") as f:
            text = f.read()
    except OSError as exc:
        raise ManifestError("manifest 读取失败 (%s): %s" % (path, exc))

    try:
        meta, body = parse_frontmatter(text)
    except FrontmatterError as exc:
        raise ManifestError("manifest 解析失败 (%s): %s" % (path, exc))

    if not isinstance(meta, dict):
        raise ManifestError("manifest 顶层不是映射: %s" % path)
    if body.strip():
        raise ManifestError("manifest 在 front-matter 之后不允许有正文: %s" % path)
    return meta


def dump_manifest(manifest):
    """把 manifest 序列化为 front-matter 文本（LF，单尾换行）。"""
    lines = ["---"]
    lines.extend(_dump_scalar_block(manifest, skip=("pages", "modules")))
    for key in ("modules", "pages"):
        items = manifest.get(key)
        if not items:
            continue
        lines.append("%s:" % key)
        for item in items:
            lines.extend(_dump_list_item(item))
    lines.append("---")
    return "\n".join(lines) + "\n"


def _dump_scalar_block(mapping, skip=()):
    out = []
    for key, value in mapping.items():
        if key in skip:
            continue
        out.append("%s: %s" % (key, value))
    return out


def _dump_list_item(item):
    """输出一个列表项；首行 `  - k: v`，其余键缩进 4 空格，内嵌 refs 缩进 6。"""
    out = []
    first = True
    for key, value in item.items():
        if key == "refs":
            continue
        prefix = "  - " if first else "    "
        out.append("%s%s: %s" % (prefix, key, value))
        first = False
    refs = item.get("refs")
    if refs:
        out.append("    refs:")
        for ref in refs:
            symbols = ref.get("symbols") or []
            out.append("      - path: %s" % ref.get("path", ""))
            out.append("        symbols: [%s]" % ", ".join(str(s) for s in symbols))
    return out


def manifest_feature_slugs(manifest):
    """返回全部 feature slug，保持 manifest 内顺序。"""
    return [page["feature"] for page in manifest.get("pages") or [] if "feature" in page]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_manifest -v`
Expected: PASS — `Ran 9 tests ... OK`

- [ ] **Step 5: Verify round-trip against the real front-matter parser**

```bash
cd C:/code/projects/scripts/wiki
python -c "from wiki_manifest import dump_manifest, load_manifest, MANIFEST_FILENAME; import tempfile, os, io; m={'project':'x','schema_version':1,'pages':[{'feature':'f','feature_group':'A','title':'T','page_file':'a01-f.md','refs':[{'path':'src/a.c','symbols':['Foo','Bar']}]}]}; d=tempfile.mkdtemp(); p=os.path.join(d,MANIFEST_FILENAME); io.open(p,'w',encoding='utf-8',newline='\n').write(dump_manifest(m)); r=load_manifest(p); print(r['pages'][0]['refs'][0])"
```

Expected: `{'path': 'src/a.c', 'symbols': ['Foo', 'Bar']}`

- [ ] **Step 6: Commit**

```bash
cd C:/code/projects
git add scripts/wiki/wiki_manifest.py scripts/wiki/test_wiki_manifest.py
git commit -m "feat(wiki): 新增 manifest 读写模块，复用 front-matter 解析器"
git push origin main
```

---

### Task 3: manifest 门禁 `wiki_manifest_lint.py`

**Files:**
- Create: `projects/scripts/wiki/wiki_manifest_lint.py`
- Test: `projects/scripts/wiki/test_wiki_manifest_lint.py`

**Interfaces:**
- Consumes: `wiki_manifest.load_manifest`、`wiki_manifest.manifest_feature_slugs`、`wiki_manifest.ManifestError`、`wiki_schema.SCHEMA_VERSION`、`wiki_schema.PAGE_TYPES`
- Produces: `lint_manifest(manifest: dict, project_dir: str | None = None) -> list[str]`、`main(argv=None) -> int`

门禁检查项（全部硬失败）：

| 检查 | 说明 |
|---|---|
| `project` 非空字符串 | manifest 必须声明所属项目 |
| `schema_version` 等于 `wiki_schema.SCHEMA_VERSION` | 与契约版本对齐 |
| `pages` 是非空列表 | 空 manifest 无意义 |
| 每个 page 有 `feature`/`feature_group`/`title`/`page_file`/`refs` | 五个必填键 |
| `feature` 全局唯一 | 重复即歧义 |
| `page_file` 以 `.md` 结尾 | 防写错扩展名 |
| `page_file` 与 `feature` 尾段一致 | 与 `wiki_lint` 同一规则 |
| `refs` 是非空列表，每项有 `path` 与非空 `symbols` | 无 refs 的机制页不可校验 |
| 若给出 `project_dir`，`page_file` 不得与已有文件冲突 | 防止 manifest 与既有页不一致 |

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_wiki_manifest_lint.py
# -*- coding: utf-8 -*-
import io
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_manifest_lint import lint_manifest, main

GOOD = {
    "project": "postgres",
    "schema_version": 1,
    "pages": [
        {
            "feature": "buffer-pool",
            "feature_group": "A-存储基础",
            "title": "Buffer Pool 主体",
            "page_file": "a04-buffer-pool.md",
            "refs": [{"path": "src/backend/storage/buffer/bufmgr.c",
                      "symbols": ["ReadBufferExtended"]}],
        }
    ],
}


def _good():
    return {
        "project": GOOD["project"],
        "schema_version": GOOD["schema_version"],
        "pages": [dict(p) for p in GOOD["pages"]],
    }


class TestLintGood(unittest.TestCase):
    def test_good_manifest_passes(self):
        self.assertEqual(lint_manifest(_good()), [])


class TestLintFailures(unittest.TestCase):
    def test_missing_project(self):
        m = _good()
        m.pop("project")
        self.assertTrue(any("project" in e for e in lint_manifest(m)))

    def test_wrong_schema_version(self):
        m = _good()
        m["schema_version"] = 99
        self.assertTrue(any("schema_version" in e for e in lint_manifest(m)))

    def test_empty_pages(self):
        m = _good()
        m["pages"] = []
        self.assertTrue(any("pages" in e for e in lint_manifest(m)))

    def test_missing_required_page_key(self):
        m = _good()
        m["pages"][0].pop("title")
        self.assertTrue(any("title" in e for e in lint_manifest(m)))

    def test_duplicate_feature(self):
        m = _good()
        m["pages"].append(dict(m["pages"][0], page_file="a05-buffer-pool.md"))
        self.assertTrue(any("feature 重复" in e for e in lint_manifest(m)))

    def test_page_file_must_be_markdown(self):
        m = _good()
        m["pages"][0]["page_file"] = "a04-buffer-pool.txt"
        self.assertTrue(any(".md" in e for e in lint_manifest(m)))

    def test_page_file_tail_must_match_feature(self):
        m = _good()
        m["pages"][0]["page_file"] = "a04-other.md"
        self.assertTrue(any("尾段" in e for e in lint_manifest(m)))

    def test_empty_refs_rejected(self):
        m = _good()
        m["pages"][0]["refs"] = []
        self.assertTrue(any("refs" in e for e in lint_manifest(m)))

    def test_ref_without_symbols_rejected(self):
        m = _good()
        m["pages"][0]["refs"][0]["symbols"] = []
        self.assertTrue(any("symbols" in e for e in lint_manifest(m)))

    def test_ref_without_path_rejected(self):
        m = _good()
        m["pages"][0]["refs"][0].pop("path")
        self.assertTrue(any("path" in e for e in lint_manifest(m)))


class TestMainCli(unittest.TestCase):
    def _write(self, manifest_text):
        d = tempfile.mkdtemp()
        path = os.path.join(d, "_manifest.yml")
        with io.open(path, "w", encoding="utf-8", newline="\n") as f:
            f.write(manifest_text)
        return path

    def test_exit_zero_on_good(self):
        text = (
            "---\n"
            "project: postgres\n"
            "schema_version: 1\n"
            "pages:\n"
            "  - feature: buffer-pool\n"
            "    feature_group: A-存储基础\n"
            "    title: Buffer Pool 主体\n"
            "    page_file: a04-buffer-pool.md\n"
            "    refs:\n"
            "      - path: src/backend/storage/buffer/bufmgr.c\n"
            "        symbols: [ReadBufferExtended]\n"
            "---\n"
        )
        self.assertEqual(main([self._write(text), "--quiet"]), 0)

    def test_exit_one_on_bad(self):
        text = "---\nproject: postgres\nschema_version: 99\npages: []\n---\n"
        self.assertEqual(main([self._write(text), "--quiet"]), 1)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_manifest_lint -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_manifest_lint'`

- [ ] **Step 3: Write minimal implementation**

```python
# projects/scripts/wiki/wiki_manifest_lint.py
# -*- coding: utf-8 -*-
"""manifest 门禁（设计文档 §6.2 Phase 1）。

清单是契约与写作之间的唯一接口：Phase 1 产 manifest，Phase 2 的 writer 只读
自己条目内的页。因此 manifest 必须先过硬门禁，否则 332 篇会按错的结构铺开。
"""
import argparse
import os
import sys

import wiki_schema
from wiki_manifest import ManifestError, load_manifest, manifest_feature_slugs

REQUIRED_PAGE_KEYS = ("feature", "feature_group", "title", "page_file", "refs")
_PAGE_PREFIX_RE = None  # 延迟编译，避免 import 期开销


def _page_tail(page_file):
    """`a04-buffer-pool.md` -> `buffer-pool`。"""
    import re

    stem = os.path.splitext(page_file)[0]
    m = re.match(r"^[A-Za-z]\d+-(.+)$", stem)
    return m.group(1) if m else stem


def lint_manifest(manifest, project_dir=None):
    """返回错误列表，空列表表示通过。"""
    errors = []

    project = manifest.get("project")
    if not isinstance(project, str) or not project.strip():
        errors.append("缺 project 或不是非空字符串")

    version = manifest.get("schema_version")
    if version != wiki_schema.SCHEMA_VERSION:
        errors.append(
            "schema_version=%r，当前契约版本为 %d" % (version, wiki_schema.SCHEMA_VERSION)
        )

    pages = manifest.get("pages")
    if not isinstance(pages, list) or not pages:
        errors.append("pages 必须是非空列表")
        return errors

    for index, page in enumerate(pages):
        if not isinstance(page, dict):
            errors.append("pages[%d] 不是映射" % index)
            continue
        for key in REQUIRED_PAGE_KEYS:
            if key not in page:
                errors.append("pages[%d] 缺字段: %s" % (index, key))

        page_file = page.get("page_file")
        if isinstance(page_file, str):
            if not page_file.endswith(".md"):
                errors.append("pages[%d].page_file 必须以 .md 结尾: %r" % (index, page_file))
            feature = page.get("feature")
            if isinstance(feature, str) and page_file.endswith(".md"):
                tail = _page_tail(page_file)
                if tail != feature:
                    errors.append(
                        "pages[%d].page_file 尾段(%r) 与 feature(%r) 不一致"
                        % (index, tail, feature)
                    )
            if project_dir:
                conflict = os.path.join(project_dir, page_file)
                if os.path.exists(conflict):
                    errors.append(
                        "pages[%d].page_file 已存在同名的既有文件: %s" % (index, page_file)
                    )

        refs = page.get("refs")
        if not isinstance(refs, list) or not refs:
            errors.append("pages[%d].refs 必须是非空列表" % index)
            continue
        for ridx, ref in enumerate(refs):
            if not isinstance(ref, dict):
                errors.append("pages[%d].refs[%d] 不是映射" % (index, ridx))
                continue
            if not ref.get("path"):
                errors.append("pages[%d].refs[%d] 缺 path" % (index, ridx))
            symbols = ref.get("symbols")
            if not isinstance(symbols, list) or not symbols:
                errors.append("pages[%d].refs[%d].symbols 必须是非空列表" % (index, ridx))

    slugs = manifest_feature_slugs(manifest)
    seen = set()
    for slug in slugs:
        if slug in seen:
            errors.append("feature 重复: %s" % slug)
        seen.add(slug)

    return errors


def main(argv=None):
    parser = argparse.ArgumentParser(description="wiki manifest 门禁")
    parser.add_argument("manifests", nargs="+", help="manifest 文件或所在目录")
    parser.add_argument("--quiet", action="store_true")
    args = parser.parse_args(argv)

    failed = 0
    checked = 0
    for target in args.manifests:
        paths = [target]
        if os.path.isdir(target):
            paths = [
                os.path.join(target, name)
                for name in sorted(os.listdir(target))
                if name.endswith(".yml") or name.endswith(".yaml")
            ]
        for path in paths:
            checked += 1
            try:
                manifest = load_manifest(path)
            except ManifestError as exc:
                failed += 1
                print("FAIL %s" % path)
                print("     - %s" % exc)
                continue
            errors = lint_manifest(manifest)
            if errors:
                failed += 1
                print("FAIL %s" % path)
                for e in errors:
                    print("     - %s" % e)
            elif not args.quiet:
                print("PASS %s" % path)

    print("-- 检查 %d 个 manifest，失败 %d 个" % (checked, failed))
    return 1 if failed else 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_manifest_lint -v`
Expected: PASS — `Ran 12 tests ... OK`

- [ ] **Step 5: Commit**

```bash
cd C:/code/projects
git add scripts/wiki/wiki_manifest_lint.py scripts/wiki/test_wiki_manifest_lint.py
git commit -m "feat(wiki): 新增 manifest 门禁 CLI，检查必填键、slug 唯一与 refs 非空"
git push origin main
```

---

### Task 4: 源码目录枚举 `wiki_tree.py`

**Files:**
- Create: `projects/scripts/wiki/wiki_tree.py`
- Test: `projects/scripts/wiki/test_wiki_tree.py`

**Interfaces:**
- Consumes: 无（纯文件系统遍历）
- Produces: `SOURCE_EXTS: tuple[str, ...]`、`SKIP_DIRS: tuple[str, ...]`、`list_module_dirs(repo_root: str, depth: int = 2) -> list[dict]`、`count_source_files(dir_path: str) -> int`、`module_key_from_dir(repo_root: str, dir_path: str) -> str`

**用途：** postgres 332 篇不是拍脑袋列的——必须从真实目录结构推导。本模块给出「每个目录有多少核心源文件」，让 Phase 1 的 agent 按目录重要性决定拆几篇。

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_wiki_tree.py
# -*- coding: utf-8 -*-
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from wiki_tree import (SKIP_DIRS, SOURCE_EXTS, count_source_files,
                       list_module_dirs, module_key_from_dir)


def _build(root):
    tree = {
        "src/backend/storage/buffer/bufmgr.c": "int a;",
        "src/backend/storage/buffer/freelist.c": "int b;",
        "src/backend/storage/buffer/README": "doc",
        "src/backend/storage/buffer/t/": None,
        "src/backend/access/heap/heapam.c": "int c;",
        "src/backend/access/heap/heapam_xlog.c": "int d;",
        "src/test/regress/foo.c": "int e;",
        "src/backend/postmaster/main.c": "int f;",
        "build/gen.c": "int g;",
    }
    for rel, content in tree.items():
        path = os.path.join(root, rel.replace("/", os.sep))
        if content is None:
            os.makedirs(path, exist_ok=True)
            continue
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, "w", encoding="utf-8", newline="\n") as f:
            f.write(content)


class TestConstants(unittest.TestCase):
    def test_source_exts_include_c(self):
        self.assertIn(".c", SOURCE_EXTS)
        self.assertIn(".cc", SOURCE_EXTS)

    def test_skip_dirs_include_build_and_test(self):
        self.assertIn("build", SKIP_DIRS)
        self.assertIn("test", SKIP_DIRS)


class TestCountSourceFiles(unittest.TestCase):
    def test_counts_only_source_exts(self):
        with tempfile.TemporaryDirectory() as d:
            _build(d)
            self.assertEqual(count_source_files(os.path.join(d, "src", "backend", "storage", "buffer")), 2)

    def test_missing_dir_returns_zero(self):
        self.assertEqual(count_source_files(os.path.join(tempfile.gettempdir(), "no-such-dir")), 0)


class TestListModuleDirs(unittest.TestCase):
    def test_lists_dirs_with_source_count(self):
        with tempfile.TemporaryDirectory() as d:
            _build(d)
            modules = list_module_dirs(d, depth=3)
        keys = {m["path"] for m in modules}
        self.assertIn("src/backend/storage/buffer", keys)
        self.assertIn("src/backend/access/heap", keys)

    def test_skips_build_and_test_dirs(self):
        with tempfile.TemporaryDirectory() as d:
            _build(d)
            modules = list_module_dirs(d, depth=3)
        joined = " ".join(m["path"] for m in modules)
        self.assertNotIn("build", joined)
        self.assertNotIn("src/test", joined)

    def test_source_files_field_matches_count(self):
        with tempfile.TemporaryDirectory() as d:
            _build(d)
            modules = {m["path"]: m for m in list_module_dirs(d, depth=3)}
        self.assertEqual(modules["src/backend/storage/buffer"]["source_files"], 2)

    def test_sorted_by_path(self):
        with tempfile.TemporaryDirectory() as d:
            _build(d)
            modules = list_module_dirs(d, depth=3)
        paths = [m["path"] for m in modules]
        self.assertEqual(paths, sorted(paths))


class TestModuleKey(unittest.TestCase):
    def test_relative_posix_path(self):
        rel = module_key_from_dir("/repo", "/repo/src/backend/buffer")
        self.assertEqual(rel, "src/backend/buffer")

    def test_uses_forward_slashes(self):
        rel = module_key_from_dir("C:\\repo", "C:\\repo\\src\\access")
        self.assertNotIn("\\", rel)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_tree -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'wiki_tree'`

- [ ] **Step 3: Write minimal implementation**

```python
# projects/scripts/wiki/wiki_tree.py
# -*- coding: utf-8 -*-
"""源码目录树 → 机制单元候选。

postgres 332 篇不是拍脑袋列的：每篇对应一个真实目录或一组紧密相关源文件。
本模块只回答「哪些目录有源码、各有多少」，把「拆几篇」留给 Phase 1 判定。
"""
import os

SOURCE_EXTS = (
    ".c", ".cc", ".cpp", ".cxx", ".h", ".hpp", ".hh",
    ".go", ".rs", ".java", ".kt", ".scala", ".py", ".ts",
    ".sql", ".lua", ".rb", ".php", ".m", ".mm", ".swift",
)

# 这些目录要么是构建产物，要么是测试夹具，不承载「机制」。
SKIP_DIRS = (
    ".git", "build", "build-msvc", "node_modules", "target", "vendor",
    "third_party", "thirdpart", "deps", "dist", "test", "tests", "regress",
    "examples", "doc", "docs", "contrib",
)


def count_source_files(dir_path):
    """统计目录下（不递归）符合 SOURCE_EXTS 的文件数。目录不存在返回 0。"""
    if not os.path.isdir(dir_path):
        return 0
    total = 0
    for name in os.listdir(dir_path):
        full = os.path.join(dir_path, name)
        if os.path.isfile(full) and name.endswith(SOURCE_EXTS):
            total += 1
    return total


def module_key_from_dir(repo_root, dir_path):
    """返回相对仓根、正斜杠分隔的路径。"""
    rel = os.path.relpath(dir_path, repo_root)
    return rel.replace(os.sep, "/")


def list_module_dirs(repo_root, depth=2):
    """列出深度不超过 depth 的目录，附带其直接源文件数。

    返回 [{"path": "src/backend/buffer", "source_files": 8}, ...]，按 path 排序。
    只返回 source_files > 0 的目录：空目录与纯文档目录不构成机制单元。
    """
    repo_root = os.path.abspath(repo_root)
    results = []
    root_depth = repo_root.rstrip(os.sep).count(os.sep)

    for dirpath, dirnames, _ in os.walk(repo_root):
        dirnames[:] = sorted(
            d for d in dirnames
            if d not in SKIP_DIRS and not d.startswith(".")
        )
        current_depth = dirpath.rstrip(os.sep).count(os.sep) - root_depth
        if current_depth >= depth:
            dirnames[:] = []

        count = count_source_files(dirpath)
        if count > 0:
            results.append(
                {
                    "path": module_key_from_dir(repo_root, dirpath),
                    "source_files": count,
                }
            )

    return sorted(results, key=lambda item: item["path"])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_wiki_tree -v`
Expected: PASS — `Ran 9 tests ... OK`

- [ ] **Step 5: Run against the real postgres clone**

```bash
cd C:/code/projects/scripts/wiki
python -c "from repo_cache import ensure_repo; print(ensure_repo('https://github.com/postgres/postgres.git','postgres'))"
python -c "from wiki_tree import list_module_dirs; mods=list_module_dirs(r'C:\code\projects\.cache\repos\postgres', depth=3); print('目录数', len(mods)); print('源文件总数', sum(m['source_files'] for m in mods)); print(mods[:5])"
```

Expected: 目录数在 50–150 之间，源文件总数约 1000–2000，输出前 5 个目录含 `src/backend/access/...`

- [ ] **Step 6: Commit**

```bash
cd C:/code/projects
git add scripts/wiki/wiki_tree.py scripts/wiki/test_wiki_tree.py
git commit -m "feat(wiki): 新增源码目录枚举模块，为机制单元划分提供实测依据"
git push origin main
```

---

### Task 5: 生成 270 库项目清单

**Files:**
- Create: `projects/docs/wiki/catalog-projects.yml`
- Create: `projects/scripts/wiki/build_catalog.py`
- Test: `projects/scripts/wiki/test_build_catalog.py`

**Interfaces:**
- Consumes: `wiki_catalog.dump_catalog`、`wiki_catalog.validate_entry`、`wiki_catalog.CATALOG_FIELDS`
- Produces: `build_entry(name, model, tier, repo, license, language, stars, stars_as_of) -> dict`、`merge_from_reference(reference_catalog_path: str, overrides: dict) -> list[dict]`

**清单来源策略（明确，不留含糊）：**

1. **基线**：从 `reference/reference/catalog.yml` 的 97 条继承 `name`/`model`/`tier`/`repository`/`license`，`tier` 做映射（`core` → `T1`，`representative` → `T2`，`innovative` → `T3`）。
2. **扩充**：按设计文档 §2.1 的 T1/T2/T3 口径补足到约 270 库。补充条目的 `repo` 用 GitHub 官方地址，`language` 按主语言填，`stars` 与 `stars_as_of` 从 GitHub API 取。
3. **不许猜**：`stars` 必须来自 API 实测或明确标注 `stars: 0` + `stars_as_of` 为手填日期，不得编造。

本任务**只建工具与首版清单骨架**（97 条继承 + 结构校验），补足到 270 的分批入库放在 Task 6。

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_build_catalog.py
# -*- coding: utf-8 -*-
import io
import os
import sys
import tempfile
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from build_catalog import build_entry, map_tier, merge_from_reference

REFERENCE_CATALOG = """
projects:
  - name: postgres
    model: relational
    tier: core
    repository: https://github.com/postgres/postgres.git
    license: PostgreSQL
    status: active
  - name: qdrant
    model: vector
    tier: representative
    repository: https://github.com/qdrant/qdrant.git
    license: Apache-2.0
    status: active
"""


class TestMapTier(unittest.TestCase):
    def test_core_to_t1(self):
        self.assertEqual(map_tier("core"), "T1")

    def test_representative_to_t2(self):
        self.assertEqual(map_tier("representative"), "T2")

    def test_innovative_to_t3(self):
        self.assertEqual(map_tier("innovative"), "T3")

    def test_unknown_tier_raises(self):
        with self.assertRaises(ValueError):
            map_tier("unknown")


class TestBuildEntry(unittest.TestCase):
    def test_all_fields_present(self):
        entry = build_entry(
            name="qdrant",
            model="vector",
            tier="T1",
            repo="https://github.com/qdrant/qdrant",
            license="Apache-2.0",
            language="Rust",
            stars=35659,
            stars_as_of="2026-11-22",
        )
        self.assertEqual(entry["name"], "qdrant")
        self.assertEqual(entry["tier"], "T1")
        self.assertEqual(entry["stars"], 35659)

    def test_repo_with_git_suffix_stripped(self):
        entry = build_entry(
            name="x", model="vector", tier="T2",
            repo="https://github.com/x/y.git", license="MIT",
            language="Go", stars=1, stars_as_of="2026-11-22",
        )
        self.assertEqual(entry["repo"], "https://github.com/x/y")


class TestMergeFromReference(unittest.TestCase):
    def test_maps_tier_and_keeps_repo(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, "catalog.yml")
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(REFERENCE_CATALOG)
            entries = merge_from_reference(path, overrides={})
        by_name = {e["name"]: e for e in entries}
        self.assertEqual(by_name["postgres"]["tier"], "T1")
        self.assertEqual(by_name["qdrant"]["tier"], "T2")
        self.assertEqual(by_name["postgres"]["repo"], "https://github.com/postgres/postgres")
        self.assertEqual(by_name["qdrant"]["license"], "Apache-2.0")

    def test_overrides_apply(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, "catalog.yml")
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(REFERENCE_CATALOG)
            entries = merge_from_reference(
                path,
                overrides={"postgres": {"language": "C", "stars": 13500,
                                        "stars_as_of": "2026-11-22"}},
            )
        by_name = {e["name"]: e for e in entries}
        self.assertEqual(by_name["postgres"]["language"], "C")
        self.assertEqual(by_name["postgres"]["stars"], 13500)

    def test_entries_have_eight_fields(self):
        with tempfile.TemporaryDirectory() as d:
            path = os.path.join(d, "catalog.yml")
            with io.open(path, "w", encoding="utf-8", newline="\n") as f:
                f.write(REFERENCE_CATALOG)
            entries = merge_from_reference(path, overrides={})
        for entry in entries:
            self.assertEqual(len(entry), 8, entry)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_build_catalog -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'build_catalog'`

- [ ] **Step 3: Write minimal implementation**

```python
# projects/scripts/wiki/build_catalog.py
# -*- coding: utf-8 -*-
"""从 reference 仓的镜像清单继承基线，产出文档生产清单。

两份清单语义不同（镜像清单管「怎么克隆」，生产清单管「写哪些库」），
这里只继承 name / model / tier / repository / license 五个字段，
不继承 commit / pin_status / source —— 那些属于克隆策略，与文档生产无关。
"""
import os
import re
import sys

from wiki_catalog import CATALOG_FIELDS, dump_catalog, validate_entry

TIER_MAP = {"core": "T1", "representative": "T2", "innovative": "T3"}

_ENTRY_RE = re.compile(r"^\s{2}-\s+name:\s*(.+?)\s*$")
_FIELD_RE = re.compile(r"^\s{4}([A-Za-z_]+):\s*(.*?)\s*$")


def map_tier(reference_tier):
    """core→T1 / representative→T2 / innovative→T3。未知值抛 ValueError。"""
    try:
        return TIER_MAP[reference_tier]
    except KeyError:
        raise ValueError("未知的 reference tier: %r" % (reference_tier,))


def build_entry(name, model, tier, repo, license, language, stars, stars_as_of):
    """构造一条清单条目。repo 末尾的 .git 会被去掉。"""
    if isinstance(repo, str) and repo.endswith(".git"):
        repo = repo[: -len(".git")]
    return {
        "name": name,
        "model": model,
        "tier": tier,
        "repo": repo,
        "license": license,
        "language": language,
        "stars": stars,
        "stars_as_of": stars_as_of,
    }


def _parse_reference(path):
    """解析镜像清单里的 name / model / tier / repository / license。"""
    with open(path, encoding="utf-8") as f:
        text = f.read().replace("\r\n", "\n")

    entries = []
    current = None
    for raw in text.split("\n"):
        m = _ENTRY_RE.match(raw)
        if m:
            current = {"name": m.group(1).strip()}
            entries.append(current)
            continue
        m = _FIELD_RE.match(raw)
        if m and current is not None:
            current[m.group(1)] = m.group(2).strip()
    return entries


def merge_from_reference(reference_catalog_path, overrides):
    """产出完整条目列表；overrides 按 name 覆盖字段。"""
    out = []
    for raw in _parse_reference(reference_catalog_path):
        name = raw["name"]
        override = dict(overrides.get(name, {}))
        entry = build_entry(
            name=name,
            model=raw.get("model", ""),
            tier=map_tier(raw.get("tier", "")),
            repo=raw.get("repository", ""),
            license=raw.get("license", ""),
            language=override.get("language", "unknown"),
            stars=override.get("stars", 0),
            stars_as_of=override.get("stars_as_of", "1970-01-01"),
        )
        out.append(entry)
    return out


def main(argv=None):
    import argparse

    parser = argparse.ArgumentParser(description="生成文档生产清单")
    parser.add_argument("reference_catalog", help="reference/reference/catalog.yml 路径")
    parser.add_argument("output", help="输出路径（projects/docs/wiki/catalog-projects.yml）")
    args = parser.parse_args(argv)

    entries = merge_from_reference(args.reference_catalog, overrides={})
    bad = 0
    for entry in entries:
        errors = validate_entry(entry)
        if errors:
            bad += 1
            print("FAIL %s" % entry.get("name"))
            for e in errors:
                print("     - %s" % e)
    if bad:
        print("-- %d 条不合格，未写出" % bad)
        return 1

    text = dump_catalog(entries)
    os.makedirs(os.path.dirname(os.path.abspath(args.output)), exist_ok=True)
    with open(args.output, "w", encoding="utf-8", newline="\n") as f:
        f.write(text)
    print("-- 写出 %d 条到 %s" % (len(entries), args.output))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_build_catalog -v`
Expected: PASS — `Ran 8 tests ... OK`

- [ ] **Step 5: Generate the baseline catalog**

```bash
cd C:/code/projects/scripts/wiki
python build_catalog.py C:/code/reference/reference/catalog.yml C:/code/projects/docs/wiki/catalog-projects.yml
```

Expected: `-- 写出 97 条到 C:\code\projects\docs\wiki\catalog-projects.yml`，退出码 0

- [ ] **Step 6: Verify the generated file**

```bash
cd C:/code/projects/scripts/wiki
python -c "from wiki_catalog import parse_catalog, validate_entry; import io; e=parse_catalog(io.open(r'C:\code\projects\docs\wiki\catalog-projects.yml',encoding='utf-8').read()); print('条数', len(e)); print('全部合格', all(not validate_entry(x) for x in e)); print(e[0])"
```

Expected: `条数 97`、`全部合格 True`、第一条是 postgres 且 tier 为 T1

- [ ] **Step 7: Commit**

```bash
cd C:/code/projects
git add docs/wiki/catalog-projects.yml scripts/wiki/build_catalog.py scripts/wiki/test_build_catalog.py
git commit -m "feat(wiki): 生成 97 条基线项目清单，从 reference 镜像清单继承并映射分层"
git push origin main
```

---

### Task 6: 补齐清单到约 270 库

**Files:**
- Modify: `projects/docs/wiki/catalog-projects.yml`
- Create: `projects/scripts/wiki/catalog_additions.py`
- Test: `projects/scripts/wiki/test_catalog_additions.py`

**Interfaces:**
- Consumes: `wiki_catalog.parse_catalog`、`wiki_catalog.validate_entry`
- Produces: `ADDITIONS: list[dict]`（待补条目，字段齐全，`stars` 已实取）、`apply_additions(existing: list[dict], additions: list[dict]) -> list[dict]`

**取数规则（硬性）：**

- `stars` 与 `language` 必须用 `https://api.github.com/repos/{owner}/{repo}` 实取，禁止编造。
- 取不到的项目**不写入清单**，在 `catalog_additions.py` 里留 `# 待取数` 注释说明。
- 未认证限流 60 次/小时。**每次运行最多取 55 个**，超限就停、下次继续（脚本要能断点续跑）。

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_catalog_additions.py
# -*- coding: utf-8 -*-
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from catalog_additions import ADDITIONS, apply_additions
from wiki_catalog import validate_entry


class TestAdditionsData(unittest.TestCase):
    def test_additions_is_non_empty(self):
        self.assertGreater(len(ADDITIONS), 0)

    def test_every_addition_has_eight_fields(self):
        for entry in ADDITIONS:
            self.assertEqual(len(entry), 8, entry)
            self.assertEqual(set(entry), {
                "name", "model", "tier", "repo", "license",
                "language", "stars", "stars_as_of",
            })

    def test_every_addition_validates(self):
        for entry in ADDITIONS:
            self.assertEqual(validate_entry(entry), [], entry)

    def test_names_are_unique(self):
        names = [e["name"] for e in ADDITIONS]
        self.assertEqual(len(names), len(set(names)))

    def test_stars_are_non_negative_ints(self):
        for entry in ADDITIONS:
            self.assertIsInstance(entry["stars"], int)
            self.assertGreaterEqual(entry["stars"], 0)

    def test_tiers_are_known(self):
        for entry in ADDITIONS:
            self.assertIn(entry["tier"], ("T1", "T2", "T3"))


class TestApplyAdditions(unittest.TestCase):
    def test_appends_new_entries(self):
        existing = [{"name": "postgres", "model": "relational", "tier": "T1",
                     "repo": "https://github.com/postgres/postgres",
                     "license": "PostgreSQL", "language": "C",
                     "stars": 13500, "stars_as_of": "2026-11-22"}]
        merged = apply_additions(existing, ADDITIONS[:2])
        self.assertEqual(len(merged), 3)

    def test_does_not_duplicate_existing(self):
        existing = [ADDITIONS[0]]
        merged = apply_additions(existing, ADDITIONS[:2])
        names = [e["name"] for e in merged]
        self.assertEqual(len(names), len(set(names)))

    def test_existing_wins_on_conflict(self):
        dup = dict(ADDITIONS[0], stars=1)
        existing = [dict(ADDITIONS[0], stars=999)]
        merged = apply_additions(existing, [dup])
        by_name = {e["name"]: e for e in merged}
        self.assertEqual(by_name[ADDITIONS[0]["name"]]["stars"], 999)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_catalog_additions -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'catalog_additions'`

- [ ] **Step 3: Fetch stars/language for the candidate list (API 实测)**

先建候选名单，再实取数据。候选按设计文档的 T1/T2 口径列出，每个库都要有真实来源：

```bash
cd C:/code/projects/scripts/wiki
python - <<'PY'
import json, sys, urllib.request

# 候选名单：name, model, tier, github owner/repo
CANDIDATES = [
    # relational
    ("clickhouse", "columnar", "T1", "ClickHouse/ClickHouse"),
    ("duckdb", "embedded", "T1", "duckdb/duckdb"),
    ("sqlite", "relational", "T1", "sqlite/sqlite"),
    ("cockroach", "distributed-sql", "T1", "cockroachdb/cockroach"),
    ("nebulagraph", "graph", "T2", "vesoft-inc/nebula"),
    ("greptimedb", "time-series", "T2", "GreptimeTeam/greptimedb"),
    # vector
    ("pgvector", "extension", "T1", "pgvector/pgvector"),
    ("milvus", "vector", "T1", "milvus-io/milvus"),
    ("weaviate", "vector", "T1", "weaviate/weaviate"),
    ("chroma", "vector", "T1", "chroma-core/chroma"),
    ("lancedb", "vector", "T1", "lancedb/lancedb"),
    ("faiss", "vector", "T1", "facebookresearch/faiss"),
    # key-value / cache
    ("redis", "key-value", "T1", "redis/redis"),
    ("rocksdb", "key-value", "T1", "facebook/rocksdb"),
    ("tikv", "key-value", "T1", "tikv/tikv"),
    # search
    ("elasticsearch", "search", "T1", "elastic/elasticsearch"),
    ("meilisearch", "search", "T1", "meilisearch/meilisearch"),
    ("tantivy", "search", "T2", "quickwit-oss/tantivy"),
    # stream
    ("kafka", "stream", "T1", "apache/kafka"),
    ("pulsar", "stream", "T2", "apache/pulsar"),
    ("nats", "stream", "T2", "nats-io/nats-server"),
    # ai/rag
    ("langchain", "rag", "T1", "langchain-ai/langchain"),
    ("llama-index", "rag", "T1", "run-llama/llama_index"),
    ("haystack", "rag", "T1", "deepset-ai/haystack"),
    ("dify", "rag", "T1", "langgenius/dify"),
    ("ragflow", "rag", "T1", "infiniflow/ragflow"),
    # ai/agent
    ("langgraph", "agent", "T1", "langchain-ai/langgraph"),
    ("autogen", "agent", "T1", "microsoft/autogen"),
    ("crewai", "agent", "T1", "crewAIInc/crewAI"),
    ("openai-agents", "agent", "T1", "openai/openai-agents-python"),
    ("metagpt", "agent", "T2", "geekan/MetaGPT"),
    # llm
    ("vllm", "llm", "T1", "vllm-project/vllm"),
    ("sglang", "llm", "T1", "sgl-project/sglang"),
    ("llama.cpp", "llm", "T1", "ggml-org/llama.cpp"),
    ("tgi", "llm", "T2", "huggingface/text-generation-inference"),
    ("ollama", "llm", "T1", "ollama/ollama"),
    ("deepspeed", "llm", "T2", "deepspeedai/DeepSpeed"),
    ("megatron-lm", "llm", "T2", "NVIDIA/Megatron-LM"),
    ("transformer-engine", "llm", "T2", "NVIDIA/TransformerEngine"),
]

out = []
for name, model, tier, slug in CANDIDATES:
    url = "https://api.github.com/repos/%s" % slug
    try:
        with urllib.request.urlopen(url, timeout=20) as resp:
            data = json.load(resp)
        out.append({
            "name": name,
            "model": model,
            "tier": tier,
            "repo": data["html_url"],
            "license": (data.get("license") or {}).get("spdx_id") or "unknown",
            "language": data.get("language") or "unknown",
            "stars": data["stargazers_count"],
            "stars_as_of": "2026-11-22",
        })
        print("OK   %-20s %6d  %s" % (name, data["stargazers_count"], data.get("language")))
    except Exception as exc:
        print("FAIL %-20s %s" % (name, exc))

with open(r"C:\code\.scratch\catalog_additions.json", "w", encoding="utf-8") as f:
    json.dump(out, f, ensure_ascii=False, indent=2)
print("\n实取成功 %d / %d" % (len(out), len(CANDIDATES)))
PY
```

Expected: 每个候选打印 `OK <name> <stars> <language>`，最后输出成功数（应 ≥ 35）。失败的留着下轮补。若触发限流（403），等一小时后重跑；脚本已按 name 去重，可断点续跑。

- [ ] **Step 4: Write the additions module from the fetched JSON**

```bash
cd C:/code/projects/scripts/wiki
python - <<'PY'
import io, json

with io.open(r"C:\code\.scratch\catalog_additions.json", encoding="utf-8") as f:
    data = json.load(f)

lines = [
    "# -*- coding: utf-8 -*-",
    '"""文档生产清单的补充条目。',
    "",
    "每条的 stars / language / license 均由 GitHub REST API 实取于 2026-11-22，",
    "不是估算值。取不到的项目不写入——宁可缺条，不可编数。",
    "",
    "分批入库：受未认证限流 60 次/小时约束，每次最多新增 55 条；",
    "重跑脚本会按 name 去重，可断点续跑。",
    '"""',
    "",
    "ADDITIONS = [",
]
ORDER = ("name", "model", "tier", "repo", "license", "language", "stars", "stars_as_of")
for entry in data:
    lines.append("    {")
    for key in ORDER:
        value = entry[key]
        if isinstance(value, str):
            lines.append('        "%s": "%s",' % (key, value))
        else:
            lines.append('        "%s": %d,' % (key, value))
    lines.append("    },")
lines.append("]")
lines.append("")
lines.append("")
lines.append("def apply_additions(existing, additions):")
lines.append('    """把新增条目并入既有清单；已存在的 name 以 existing 为准。"""')
lines.append("    seen = {entry[\"name\"] for entry in existing}")
lines.append("    merged = list(existing)")
lines.append("    for entry in additions:")
lines.append("        if entry[\"name\"] in seen:")
lines.append("            continue")
lines.append("        seen.add(entry[\"name\"])")
lines.append("        merged.append(entry)")
lines.append('    return sorted(merged, key=lambda e: (e["tier"], e["model"], e["name"]))')
lines.append("")

with io.open("catalog_additions.py", "w", encoding="utf-8", newline="\n") as f:
    f.write("\n".join(lines))
print("写出 catalog_additions.py，%d 条" % len(data))
PY
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd projects/scripts/wiki && python -m unittest test_catalog_additions -v`
Expected: PASS — `Ran 9 tests ... OK`

- [ ] **Step 6: Merge into the catalog and regenerate**

```bash
cd C:/code/projects/scripts/wiki
python - <<'PY'
import io
from catalog_additions import ADDITIONS, apply_additions
from wiki_catalog import dump_catalog, parse_catalog

path = r"C:\code\projects\docs\wiki\catalog-projects.yml"
existing = parse_catalog(io.open(path, encoding="utf-8").read())
merged = apply_additions(existing, ADDITIONS)
with io.open(path, "w", encoding="utf-8", newline="\n") as f:
    f.write(dump_catalog(merged))
print("清单从 %d 条增至 %d 条" % (len(existing), len(merged)))
PY
```

Expected: `清单从 97 条增至 130+ 条`

- [ ] **Step 7: Verify all entries validate**

```bash
cd C:/code/projects/scripts/wiki
python -c "
from wiki_catalog import parse_catalog, validate_entry
import io
e = parse_catalog(io.open(r'C:\code\projects\docs\wiki\catalog-projects.yml', encoding='utf-8').read())
bad = [(x['name'], validate_entry(x)) for x in e if validate_entry(x)]
print('总条数', len(e))
print('不合格', bad or '无')
import collections
print('tier 分布', dict(collections.Counter(x['tier'] for x in e)))
print('model 分布', dict(collections.Counter(x['model'] for x in e)))
"
```

Expected: 总条数 ≥ 130，不合格为「无」，tier 分布三项都有

- [ ] **Step 8: Commit**

```bash
cd C:/code/projects
git add docs/wiki/catalog-projects.yml scripts/wiki/catalog_additions.py scripts/wiki/test_catalog_additions.py
git commit -m "feat(wiki): 清单扩容至 130+ 库，stars 与 language 均由 GitHub API 实取"
git push origin main
```

---

### Task 7: 产出 postgres 332 篇 manifest

**Files:**
- Create: `projects/docs/wiki/projects/postgres/_manifest.yml`
- Create: `projects/scripts/wiki/gen_postgres_manifest.py`
- Test: `projects/scripts/wiki/test_gen_postgres_manifest.py`

**Interfaces:**
- Consumes: `wiki_tree.list_module_dirs`、`wiki_manifest.dump_manifest`、`wiki_manifest_lint.lint_manifest`
- Produces: `PAGE_SPECS: list[dict]`（332 条页规格：feature / feature_group / title / page_file / refs）、`build_manifest(repo_root: str) -> dict`、`write_manifest(manifest: dict, output_path: str) -> None`

**332 篇的来源：按设计文档已枚举的 A–K 十一组分组。** 每组对应 postgres 的真实目录结构：

| 组 | 目录前缀 | 篇数 |
|---|---|---|
| A 存储基础 | `src/backend/storage/{buffer,freespace,file,ipc,lmgr,page,smgr,sync,aio,large_object}` | 29 |
| B 事务/WAL | `src/backend/access/transam` | 28 |
| C 堆表元组 | `src/backend/access/{heap,common,table,sequence,tablesample}` | 35 |
| D 索引族 | `src/backend/access/{nbtree,gin,gist,hash,brin,spgist,index}` | 33 |
| E 优化器 | `src/backend/optimizer/{plan,path,prep,util,geqo}` | 39 |
| F 执行器 | `src/backend/executor` | 41 |
| G 目录缓存 | `src/backend/{catalog,utils/cache}` | 28 |
| H 内存排序 | `src/backend/{utils/mmgr,utils/sort,lib}` | 28 |
| I 复制 | `src/backend/replication` | 22 |
| J 服务端命令面 | `src/backend/{commands,postmaster,tcop,utils/{misc,error,activity}}` | 35 |
| K 横切主题 | 跨模块 | 14 |

**K 组是唯一不绑单一目录的一组**：它讲的是「跨模块的整体设计」（查询编译流水线、扩展点、并行查询整体设计等），refs 指向多个模块的代表性符号。

- [ ] **Step 1: Write the failing test**

```python
# projects/scripts/wiki/test_gen_postgres_manifest.py
# -*- coding: utf-8 -*-
import os
import sys
import unittest

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from gen_postgres_manifest import GROUP_SPECS, PAGE_SPECS, build_manifest
from wiki_manifest_lint import lint_manifest


class TestPageSpecs(unittest.TestCase):
    def test_total_page_count_is_332(self):
        self.assertEqual(len(PAGE_SPECS), 332)

    def test_group_specs_sum_matches(self):
        self.assertEqual(sum(g["count"] for g in GROUP_SPECS), 332)

    def test_eleven_groups(self):
        self.assertEqual(len(GROUP_SPECS), 11)

    def test_group_labels_are_a_to_k(self):
        labels = [g["group"] for g in GROUP_SPECS]
        for expected in ("A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K"):
            self.assertIn(expected, labels)

    def test_feature_slugs_unique(self):
        slugs = [p["feature"] for p in PAGE_SPECS]
        self.assertEqual(len(slugs), len(set(slugs)))

    def test_page_files_match_feature(self):
        for spec in PAGE_SPECS:
            stem = os.path.splitext(spec["page_file"])[0]
            tail = stem.split("-", 1)[1] if "-" in stem else stem
            self.assertEqual(tail, spec["feature"], spec)

    def test_every_page_has_refs_with_symbols(self):
        for spec in PAGE_SPECS:
            self.assertTrue(spec["refs"], spec)
            for ref in spec["refs"]:
                self.assertTrue(ref.get("path"), spec)
                self.assertTrue(ref.get("symbols"), spec)

    def test_page_files_are_markdown(self):
        for spec in PAGE_SPECS:
            self.assertTrue(spec["page_file"].endswith(".md"), spec)


class TestBuildManifest(unittest.TestCase):
    def test_manifest_has_all_pages(self):
        manifest = build_manifest(repo_root=None)
        self.assertEqual(len(manifest["pages"]), 332)

    def test_manifest_passes_lint(self):
        manifest = build_manifest(repo_root=None)
        self.assertEqual(lint_manifest(manifest), [])

    def test_manifest_project_is_postgres(self):
        manifest = build_manifest(repo_root=None)
        self.assertEqual(manifest["project"], "postgres")

    def test_manifest_schema_version_is_one(self):
        manifest = build_manifest(repo_root=None)
        self.assertEqual(manifest["schema_version"], 1)


if __name__ == "__main__":
    unittest.main()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd projects/scripts/wiki && python -m unittest test_gen_postgres_manifest -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'gen_postgres_manifest'`

- [ ] **Step 3: Write the group spec module (数据先落，代码后写)**

PAGE_SPECS 的 332 条要逐条列全，不能循环生成——因为每条的 feature slug、标题、refs 都是独立设计的。分两步：先写 `GROUP_SPECS` 与 `_page_specs_for_group()` 骨架，再逐组填。

先建骨架与 A 组 29 条（其余组用同一模式补齐）：

```python
# projects/scripts/wiki/gen_postgres_manifest.py
# -*- coding: utf-8 -*-
"""生成 postgres 的 332 篇机制页 manifest。

332 这个数字不是拍脑袋：它等于 A–K 十一组实测枚举之和，每篇对应
postgres 真实目录或一组紧密相关源文件。分组依据见设计文档 §2.
"""
import os

from wiki_manifest import dump_manifest

GROUP_SPECS = [
    {"group": "A", "label": "A-存储基础", "count": 29,
     "prefix": "src/backend/storage"},
    {"group": "B", "label": "B-事务与WAL", "count": 28,
     "prefix": "src/backend/access/transam"},
    {"group": "C", "label": "C-堆表与元组", "count": 35,
     "prefix": "src/backend/access"},
    {"group": "D", "label": "D-索引族", "count": 33,
     "prefix": "src/backend/access"},
    {"group": "E", "label": "E-优化器", "count": 39,
     "prefix": "src/backend/optimizer"},
    {"group": "F", "label": "F-执行器", "count": 41,
     "prefix": "src/backend/executor"},
    {"group": "G", "label": "G-目录与缓存", "count": 28,
     "prefix": "src/backend"},
    {"group": "H", "label": "H-内存与排序", "count": 28,
     "prefix": "src/backend"},
    {"group": "I", "label": "I-复制与高可用", "count": 22,
     "prefix": "src/backend/replication"},
    {"group": "J", "label": "J-服务端与命令面", "count": 35,
     "prefix": "src/backend"},
    {"group": "K", "label": "K-横切主题", "count": 14,
     "prefix": ""},
]


def _spec(group, seq, feature, title, refs):
    """按组生成一条页规格。page_file 形如 a04-buffer-pool.md。"""
    return {
        "feature": feature,
        "feature_group": group,
        "title": title,
        "page_file": "%s%02d-%s.md" % (group.lower(), seq, feature),
        "refs": refs,
    }


# A 组 29 篇：存储层基础
_A = [
    _spec("A", 1, "page-layout", "页面布局与 PageHeaderData",
          [{"path": "src/include/storage/bufpage.h",
            "symbols": ["PageHeaderData", "PageGetItemId"]}]),
    _spec("A", 2, "smgr", "存储管理器抽象 smgr",
          [{"path": "src/backend/storage/smgr/smgr.c",
            "symbols": ["smgropen", "smgrread", "smgrwrite"]}]),
    _spec("A", 3, "md-storage", "段文件与表空间文件布局",
          [{"path": "src/backend/storage/smgr/md.c",
            "symbols": ["mdextend", "mdread", "mdwrite"]}]),
    _spec("A", 4, "buffer-pool", "Buffer Pool 主体",
          [{"path": "src/backend/storage/buffer/bufmgr.c",
            "symbols": ["ReadBufferExtended", "PinBuffer", "MarkBufferDirty"]}]),
    _spec("A", 5, "clock-sweep", "Clock-Sweep 置换与 freelist",
          [{"path": "src/backend/storage/buffer/freelist.c",
            "symbols": ["StrategyGetBuffer"]}]),
    _spec("A", 6, "buffer-hash", "缓冲区哈希表",
          [{"path": "src/backend/storage/buffer/buf_table.c",
            "symbols": ["BufTableHashCode", "BufTableLookup"]}]),
    _spec("A", 7, "local-buffer", "本地缓冲",
          [{"path": "src/backend/storage/buffer/localbuf.c",
            "symbols": ["LocalBufferAlloc"]}]),
    _spec("A", 8, "buffer-init", "缓冲初始化",
          [{"path": "src/backend/storage/buffer/buf_init.c",
            "symbols": ["InitBufferPool"]}]),
    _spec("A", 9, "freespace-map", "空闲空间映射 FSM",
          [{"path": "src/backend/storage/freespace/freespace.c",
            "symbols": ["GetPageWithFreeSpace", "RecordPageWithFreeSpace"]}]),
    _spec("A", 10, "shmem-init", "共享内存初始化",
          [{"path": "src/backend/storage/ipc/ipci.c",
            "symbols": ["CreateSharedMemoryAndSemaphores"]}]),
    _spec("A", 11, "procarray", "进程数组与快照导出",
          [{"path": "src/backend/storage/ipc/procarray.c",
            "symbols": ["GetSnapshotData", "ProcArrayAdd"]}]),
    _spec("A", 12, "sinval", "失效消息传播",
          [{"path": "src/backend/storage/ipc/sinvaladt.c",
            "symbols": ["SIInsertDataEntries", "SIGetDataEntries"]}]),
    _spec("A", 13, "latch", "Latch 等待机制",
          [{"path": "src/backend/storage/ipc/latch.c",
            "symbols": ["InitLatch", "SetLatch", "WaitLatch"]}]),
    _spec("A", 14, "dsm", "动态共享内存 DSM",
          [{"path": "src/backend/storage/ipc/dsm.c",
            "symbols": ["dsm_create", "dsm_attach"]}]),
    _spec("A", 15, "waiteventset", "等待事件集",
          [{"path": "src/backend/storage/ipc/waiteventset.c",
            "symbols": ["WaitEventSetWait"]}]),
    _spec("A", 16, "barrier", "屏障",
          [{"path": "src/backend/storage/ipc/barrier.c",
            "symbols": ["BarrierInit", "BarrierArriveAndWait"]}]),
    _spec("A", 17, "shm-mq", "共享消息队列",
          [{"path": "src/backend/storage/ipc/shm_mq.c",
            "symbols": ["shm_mq_send", "shm_mq_receive"]}]),
    _spec("A", 18, "shm-toc", "共享目录表",
          [{"path": "src/backend/storage/ipc/shm_toc.c",
            "symbols": ["shm_toc_allocate", "shm_toc_lookup"]}]),
    _spec("A", 19, "signals", "信号与 PMSIGNAL",
          [{"path": "src/backend/storage/ipc/pmsignal.c",
            "symbols": ["SendPostmasterSignal", "CheckPostmasterSignal"]}]),
    _spec("A", 20, "sync-rep-queue", "同步提交队列",
          [{"path": "src/backend/storage/sync/sync.c",
            "symbols": ["SyncPreCheckpoint", "SyncPostCheckpoint"]}]),
    _spec("A", 21, "aio", "异步 IO",
          [{"path": "src/backend/storage/aio/aio.c",
            "symbols": ["pgaio_submit"]}]),
    _spec("A", 22, "large-object", "大对象",
          [{"path": "src/backend/storage/large_object/inv_api.c",
            "symbols": ["inv_create", "inv_read", "inv_write"]}]),
    _spec("A", 23, "lock-manager", "常规锁系统",
          [{"path": "src/backend/storage/lmgr/lock.c",
            "symbols": ["LockAcquire", "LockRelease"]}]),
    _spec("A", 24, "lwlock", "轻量锁 LWLock",
          [{"path": "src/backend/storage/lmgr/lwlock.c",
            "symbols": ["LWLockAcquire", "LWLockRelease"]}]),
    _spec("A", 25, "proc-slots", "PGPROC 与进程槽",
          [{"path": "src/backend/storage/lmgr/proc.c",
            "symbols": ["InitProcess", "ProcSleep"]}]),
    _spec("A", 26, "deadlock", "死锁检测",
          [{"path": "src/backend/storage/lmgr/deadlock.c",
            "symbols": ["DeadLockCheck"]}]),
    _spec("A", 27, "ssi", "SSI 可串行化快照隔离",
          [{"path": "src/backend/storage/lmgr/predicate.c",
            "symbols": ["CheckForSerializableConflictOut"]}]),
    _spec("A", 28, "spinlock", "自旋锁与内存屏障",
          [{"path": "src/backend/storage/lmgr/s_lock.c",
            "symbols": ["s_lock"]}]),
    _spec("A", 29, "condition-variable", "条件变量",
          [{"path": "src/backend/storage/lmgr/condition_variable.c",
            "symbols": ["ConditionVariableSleep", "ConditionVariableBroadcast"]}]),
]


PAGE_SPECS = list(_A)


def build_manifest(repo_root=None):
    """组装 manifest dict。repo_root 预留给后续按真实目录补 modules 段。"""
    manifest = {
        "project": "postgres",
        "schema_version": 1,
        "pages": [dict(spec) for spec in PAGE_SPECS],
    }
    return manifest


def write_manifest(manifest, output_path):
    """写出 manifest，自动建父目录。"""
    os.makedirs(os.path.dirname(os.path.abspath(output_path)), exist_ok=True)
    with open(output_path, "w", encoding="utf-8", newline="\n") as f:
        f.write(dump_manifest(manifest))
```

- [ ] **Step 4: 按同一模式补齐 B–K 十组**

每组照 A 组的写法：`_spec(组, 序号, feature-slug, 中文标题, [{"path": ..., "symbols": [...]}])`。序号在组内从 1 连续编到该组篇数。

**硬要求：每条的 `path` 必须真实存在，每个 `symbol` 必须真在该文件里。** 写完一组立刻验证：

```bash
cd C:/code/projects/scripts/wiki
python -c "
from gen_postgres_manifest import PAGE_SPECS
from wiki_refs import check_refs_from_specs
bad = check_refs_from_specs(PAGE_SPECS, r'C:\code\projects\.cache\repos\postgres')
print('不合格页数', len(bad))
for name, errs in bad[:10]:
    print(name, errs)
"
```

若 `check_refs_from_specs` 不存在，先用临时脚本遍历：对每条 refs 用 `wiki_refs._scan_symbols` 校验（该函数已在 P0 实现）。

**补齐顺序**：B → C → D → E → F → G → H → I → J → K。每补完一组就：
1. 跑 `python -m unittest test_gen_postgres_manifest -v`，确认 `len(PAGE_SPECS)` 等于已完成组的累计数
2. 跑上面的符号校验，确认不合格页数为 0
3. `git commit` 一次（`feat(wiki): postgres manifest 补至 X 篇`）

- [ ] **Step 5: Run test to verify all 332 pass**

Run: `cd projects/scripts/wiki && python -m unittest test_gen_postgres_manifest -v`
Expected: PASS — `Ran 12 tests ... OK`

- [ ] **Step 6: Verify every symbol against real postgres source**

```bash
cd C:/code/projects/scripts/wiki
python - <<'PY'
import sys
sys.path.insert(0, '.')
from gen_postgres_manifest import PAGE_SPECS
from wiki_refs import _scan_symbols

repo = r'C:\code\projects\.cache\repos\postgres'
bad = []
for spec in PAGE_SPECS:
    for ref in spec['refs']:
        hits, reason = _scan_symbols(ref['path'].join([repo + '\\', '']), ref['symbols'])
        if reason:
            bad.append((spec['feature'], ref['path'], reason))
            continue
        missing = [s for s in ref['symbols'] if s not in hits]
        if missing:
            bad.append((spec['feature'], ref['path'], '未命中 %s' % missing))
print('总页数', len(PAGE_SPECS))
print('不合格', len(bad))
for item in bad[:20]:
    print(' ', item)
PY
```

Expected: `总页数 332`、`不合格 0`

**若出现未命中，不要放宽校验——改 refs 指向符号的真实定义文件（宏/内联通常在 `src/include/` 下的 `.h`）。**

- [ ] **Step 7: Generate the manifest file**

```bash
cd C:/code/projects/scripts/wiki
python -c "
from gen_postgres_manifest import build_manifest, write_manifest
m = build_manifest()
write_manifest(m, r'C:\code\projects\docs\wiki\projects\postgres\_manifest.yml')
print('已写出', len(m['pages']), '页')
"
```

Expected: `已写出 332 页`

- [ ] **Step 8: Verify the written manifest passes the gate**

```bash
cd C:/code/projects/scripts/wiki
python wiki_manifest_lint.py C:/code/projects/docs/wiki/projects/postgres/_manifest.yml --quiet
```

Expected: `-- 检查 1 个 manifest，失败 0 个`，退出码 0

- [ ] **Step 9: Commit**

```bash
cd C:/code/projects
git add docs/wiki/projects/postgres/_manifest.yml scripts/wiki/gen_postgres_manifest.py scripts/wiki/test_gen_postgres_manifest.py
git commit -m "feat(wiki): 产出 postgres 332 篇机制页 manifest，全部 refs 经真实源码符号校验"
git push origin main
```

---

### Task 8: 全套回归与 P1 验收

**Files:**
- Modify: 无

**Interfaces:**
- Consumes: Task 1–7 全部模块
- Produces: 可复现的 P1 验收结论

- [ ] **Step 1: Run the full test suite**

```bash
cd C:/code/projects/scripts/wiki
python -m unittest test_wiki_schema test_wiki_frontmatter test_wiki_lint test_wiki_refs test_repo_cache test_wiki_schema_doc test_wiki_catalog test_wiki_manifest test_wiki_manifest_lint test_wiki_tree test_build_catalog test_catalog_additions test_gen_postgres_manifest
```

Expected: `Ran 246 tests ... OK (skipped=1)`（P0 的 174 项 + P1 的 ~72 项）

- [ ] **Step 2: Verify the catalog gate**

```bash
cd C:/code/projects/scripts/wiki
python -c "
from wiki_catalog import parse_catalog, validate_entry
import io
e = parse_catalog(io.open(r'C:\code\projects\docs\wiki\catalog-projects.yml', encoding='utf-8').read())
print('条数', len(e))
print('不合格', [x['name'] for x in e if validate_entry(x)] or '无')
"
```

Expected: 条数 ≥ 130，不合格为「无」

- [ ] **Step 3: Verify the manifest gate**

```bash
cd C:/code/projects/scripts/wiki
python wiki_manifest_lint.py C:/code/projects/docs/wiki/projects/postgres/_manifest.yml
```

Expected: `PASS`，退出码 0

- [ ] **Step 4: Verify refs against real source one more time**

```bash
cd C:/code/projects/scripts/wiki
python -c "
from gen_postgres_manifest import PAGE_SPECS
print('页数', len(PAGE_SPECS))
print('分组分布', {g: sum(1 for p in PAGE_SPECS if p['feature_group']==g) for g in sorted({p['feature_group'] for p in PAGE_SPECS})})
"
```

Expected: `页数 332`，分组分布为 A:29 B:28 C:35 D:33 E:39 F:41 G:28 H:28 I:22 J:35 K:14

- [ ] **Step 5: Confirm the cache is not tracked**

```bash
cd C:/code/projects
git status --short
git check-ignore -v .cache/repos/postgres
```

Expected: `.cache/repos/postgres` 被 `.gitignore:165:.cache/` 忽略，`git status` 里不出现 `.cache`

- [ ] **Step 6: Commit the acceptance note**

```bash
cd C:/code/projects
git commit --allow-empty -m "test(wiki): P1 验收通过，246 项测试全绿，清单与 postgres manifest 双门禁通过"
git push origin main
```

---

## 交付物清单

P1 完成后可用（**实际落盘状态见文首执行后注记**）：

| 能力 | 入口 |
|---|---|
| 文档生产清单（**实测 152 条**，T1 39 / T2 85 / T3 28，stars 实取） | `projects/docs/wiki/catalog-projects.yml` |
| 清单解析与校验 | `wiki_catalog.parse_catalog` / `validate_entry` |
| 清单从镜像清单继承 | `build_catalog.merge_from_reference` |
| manifest 读写 | `wiki_manifest.load_manifest` / `dump_manifest` |
| manifest 门禁 | `python wiki_manifest_lint.py <path>` |
| 源码目录枚举 | `wiki_tree.list_module_dirs` |
| postgres 机制页清单（**P2a 重建后为 151 页 / 12 组**） | `projects/docs/wiki/projects/postgres/_manifest.yml` |

后续计划依赖：

- **P3（postgres 余下 131 页写作）** 依赖本计划的 `_manifest.yml` 作为唯一输入契约
- **P4（其余库流式推进）** 依赖本计划的清单与 manifest 工具链
- **P5（RAG 应用）** 依赖 `wiki_frontmatter` 与 `_manifest.yml` 做增量索引

## 未决事项（不在本计划范围）

1. 清单是否要扩到 270 库：本计划做到 130+ 并建好工具，继续扩容属增量工作，可按需分批。
   **当前实况 152 条，缺口 118 条**；扩容须逐条实取 stars/license/language，不得凑数。
2. `feature_group` / `feature_summary` / `created` / `updated` 是否纳入 `wiki_schema` 必填字段：P0 评审已记为待决，需契约升版才能改。
   **P2a 已定**：这些字段不进特性页 front-matter，分组与标题由 `_manifest.yml` 承载，项目级元信息由 `_index.md` 承载。
3. `refs: []` 是否允许：同上，属契约决策。**P2a 实操已定**：refs 必须非空。
