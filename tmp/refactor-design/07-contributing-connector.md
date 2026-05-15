# 贡献新 Connector 的规范

本文写给想给 MFS 加新 connector 的开发者。社区贡献的工作量目标：**约 500-1500 行 Python，集中在 `connectors/<name>/`**，按你实现到哪一层而定。

## 0. 三层 opt-in 深度

Connector 实现是分层的。你可以**只写 Layer 0**做出能用的版本，需要性能/能力时再向上加。

```
Layer 0 (必须)
  list / stat / read / fingerprint / change_set / object_kind_of
  ─► framework 自动给你 cat / head / tail / grep（线性扫）/ search（如有 chunks）
  ─► 约 500 行 Python，简单 connector 就够了

Layer 1 (可选下推)
  grep_pushdown / search_pushdown / follow
  ─► 不实现 → framework 走 Layer 0 fallback
  ─► 想要 SQL ILIKE 下推、provider search API、tail -f 时加
  ─► +200-500 行

Layer 2 (可选高级)
  chunk_overrides / custom_renderer / permission_snapshot
  ─► 不实现 → framework 用默认
  ─► 想自定义 chunk 拼接模板、特殊格式渲染、ACL 快照时加
  ─► +200-500 行
```

不暴露更深的扩展点（自定义 chunker、自定义 cache 格式、直接写 Milvus 等）——开太多 layer 会让 framework 难维护，也让贡献者负担过重。这些底层完全由 framework 接管。

## 1. 你需要写什么（vs 不需要写什么）

| 关注点 | 你写 | 你不写 / 复用 framework |
|---|---|---|
| 连接外部系统 / 认证 | ✅ 用对应 SDK | 凭据通过 `credential_ref` 解析 |
| 决定 URI 树长什么样 | ✅ 写 `PROMPT.md` + `layout.py` | 命名规范见 §10 |
| `stat / list / read` 实现 | ✅ 三个 method | API 路由、HTTP、SSE 都是 framework |
| 变化检测 (`fingerprint / change_set`) | ✅ 算法 per connector | `ChangeSet` 数据结构 framework 给 |
| 对象 → object_kind 映射 | ✅ 一个 dict | 每个 object_kind 的 chunker / structure 全 framework |
| chunk 切分 / embedding / summary / VLM | ❌ | framework pipeline |
| `cat / head / tail / grep / ls / tree` 命令 | ❌ | framework shell helpers |
| Retrieval Index (Milvus) | ❌ | framework |
| metadata DB / cache 存储 | ❌ | framework storage |
| HTTP API / SDK | ❌ | framework |
| `mfs add` / `mfs connector` / `mfs status` 行为 | ❌ | framework engine |
| 配置 schema 验证 | ✅ 用 pydantic | framework 调验证 |
| 内置 preset（如有） | ✅ 可选；提供默认 text_fields/locator_fields/... | 否则用户必须显式配 |

## 2. 文件骨架

```
connectors/<name>/
├── __init__.py
├── plugin.py            # ConnectorPlugin 子类；入口
├── config.py            # pydantic schema for connector TOML
├── connector.py         # 封装外部系统的 SDK / HTTP 调用，凭据持有
├── layout.py            # URI path ↔ 外部资源映射；object_kind_of()
├── sync.py              # change_set / fingerprint 实现
├── PROMPT.md            # 目录布局 ASCII 描述（给 agent skill）
├── presets.py           # 可选：内置 text_fields / metadata_fields preset
└── tests/
    ├── test_layout.py
    ├── test_sync.py
    ├── test_e2e.py
    └── fixtures/        # fake API 响应数据
```

## 3. ConnectorPlugin 契约

```python
# server/python/src/mfs_server/connectors/base.py

class ConnectorPlugin(ABC):

    # ─────── 元信息（class attribute）─────────────
    NAME: str                          # "postgres"
    URI_SCHEME: str                    # "postgres"
    DISPLAY_NAME: str                  # "Postgres"
    PROMPT: str                        # 目录布局描述，agent skill 直接拼用
    CAPABILITIES: "Capabilities"       # 见 §4
    CONFIG_SCHEMA: type[BaseModel]     # pydantic model 校验 TOML

    # ─────── 生命周期 ──────────────────────────────
    def __init__(self, config: BaseModel, credential: Any) -> None: ...
    async def connect(self) -> None: ...
    async def close(self) -> None: ...
    async def healthcheck(self) -> HealthStatus: ...

    # ─────── Layer 0：核心 IO（必须实现）───────────────────
    async def stat(self, path: str) -> FileStat: ...
    async def list(self, path: str) -> list[Entry]: ...
    async def read(self, path: str, range: Range | None = None) -> bytes | AsyncIterator[bytes]: ...

    # ─────── Layer 0：变化检测（必须实现）──────────────────
    async def fingerprint(self, path: str) -> str | None:
        """返回该 path 的当前 upstream fingerprint。None 表示总是 fresh。"""

    async def change_set(self, since: Cursor | None) -> ChangeSet:
        """增量拉取。返回 added/modified/deleted/unchanged + new_cursor。"""

    # ─────── Layer 0：路径分类（必须实现）──────────────────
    def object_kind_of(self, path: str) -> ObjectKind:
        """把虚拟 path 映射到 object_kind。
        例：rows.jsonl → "table_rows"
            messages.jsonl → "message_stream"
            schema.json → "table_schema"
            真实 .md/.py/.png → 按扩展名"""

    # ─────── Layer 1：可选下推（默认 None / False）─────────
    async def grep_pushdown(
        self, pattern: str, path: str, options: GrepOptions
    ) -> AsyncIterator[GrepMatch] | None:
        return None

    async def search_pushdown(
        self, query: str, path: str, options: SearchOptions
    ) -> AsyncIterator[Hit] | None:
        return None

    async def follow(self, path: str) -> AsyncIterator[bytes] | None:
        """tail -f 实现；不支持返回 None。"""
        return None

    # ─────── Layer 2：可选高级 ─────────────────────────
    def chunk_overrides(self, path: str) -> dict | None:
        """覆盖 framework 默认 chunk strategy/preset。
        90% 的 connector 不需要 override。"""
        return None

    def custom_renderer(self, path: str, media_type: str) -> str | None:
        """自定义 cat 渲染格式（例如 parquet → table）。
        不实现 → framework 按 media_type 默认渲染。"""
        return None

    async def permission_snapshot(self, path: str) -> dict | None:
        """ACL 快照（多租户 enterprise 场景）。v0.4 暂不启用。"""
        return None
```

`Capabilities`：

```python
@dataclass
class Capabilities:
    # sync
    manual_sync: bool = True
    scheduled_sync: bool = True
    watch: bool = False                 # 仅 file connector true
    cursor_kind: str | None = None      # "updated_at" / "snowflake" / "etag" / None
    full_scan: bool = True
    delete_detection: bool = True

    # object access
    grep_pushdown: bool = False
    search_pushdown: bool = False
    efficient_tail: bool = False
    paged_cat: bool = True               # 是否支持 --range
    permission_snapshot: bool = False
```

`mfs connector inspect <root>` 直接 dump 这个。

## 4. 数据结构

```python
@dataclass
class FileStat:
    path: str
    type: Literal["file", "dir"]
    media_type: str | None             # "application/x-ndjson" 等
    size_hint: int | None
    fingerprint: str | None
    extra: dict                        # connector-specific hint (row_count, etc.)

@dataclass
class Entry:
    name: str
    type: Literal["file", "dir"]
    media_type: str | None
    size_hint: int | None
    extra: dict

@dataclass
class Range:
    start: int
    end: int                            # 闭开 [start, end)；约定 -1 表示末尾

@dataclass
class ChangeSet:
    added:     list[str]
    modified:  list[str]
    deleted:   list[str]
    unchanged: list[str]
    new_cursor: Cursor | None

@dataclass
class GrepMatch:
    path: str
    line_no: int
    line_content: str
    context_before: list[str]
    context_after: list[str]

@dataclass
class HealthStatus:
    ok: bool
    detail: str
    extra: dict                         # connection latency, permissions, ...
```

## 5. PROMPT.md 范本

每个 connector 写一段 ASCII，描述自己 root 下的目录布局。`mfs connector inspect` 和 agent skill 都会用到。

```
{prefix}                                          # = connector root URI 例如 postgres://prod

  database.json                                   # cross-schema 概览
  <schema>/
    tables/<table>/
      schema.json                                 # column / PK / FK / index
      rows.jsonl                                  # 全部行（lazy，大表不物化）
    views/<view>/
      schema.json
      rows.jsonl

Hints:
  - Read database.json first to understand schema layout.
  - rows.jsonl is large; cat refuses without --range.
    use head/tail/grep (which push down to SQL) instead.
  - search runs against row_text chunks built from configured text_fields.
```

`{prefix}` 是占位符，运行时 framework 替换成具体 connector root URI。

## 6. 最小可工作例子

`connectors/example/`：实现一个虚拟 connector `example://demo`，root 下有 `hello.txt` 和 `counters.jsonl`。

### plugin.py

```python
from mfs_server.connectors.base import ConnectorPlugin, Capabilities, HealthStatus
from .config import ExampleConfig
from .layout import resolve, object_kind_of, list_root
from .sync import fingerprint, change_set

class ExamplePlugin(ConnectorPlugin):
    NAME = "example"
    URI_SCHEME = "example"
    DISPLAY_NAME = "Example"
    PROMPT = open(__file__.replace("plugin.py", "PROMPT.md")).read()
    CONFIG_SCHEMA = ExampleConfig
    CAPABILITIES = Capabilities(
        grep_pushdown=False,
        efficient_tail=False,
        paged_cat=True,
    )

    def __init__(self, config, credential):
        self.config = config
        self.credential = credential

    async def connect(self): pass
    async def close(self): pass
    async def healthcheck(self):
        return HealthStatus(ok=True, detail="example always ok", extra={})

    async def stat(self, path):
        return resolve(self.config, path).to_stat()

    async def list(self, path):
        return list_root(self.config, path)

    async def read(self, path, range=None):
        return resolve(self.config, path).read_bytes(range)

    def object_kind_of(self, path):
        return object_kind_of(path)

    async def fingerprint(self, path):
        return fingerprint(self.config, path)

    async def change_set(self, since):
        return change_set(self.config, since)
```

### config.py

```python
from pydantic import BaseModel

class ExampleConfig(BaseModel):
    counter_start: int = 0
    counter_count: int = 100
```

### TOML（用户写）

```toml
[connector]
type = "example"
root = "example://demo"
label = "Example demo"

[example]
counter_start = 0
counter_count = 1000
```

### 用户体验

```bash
mfs add example://demo --config .mfs/connectors/example-demo.toml
mfs ls example://demo
# file  hello.txt        text/plain          12 B
# file  counters.jsonl   application/x-ndjson 21 KB    1000 records
mfs cat example://demo/hello.txt
# Hello, World!
mfs head -n 3 example://demo/counters.jsonl
# {"n": 0}
# {"n": 1}
# {"n": 2}
mfs search "hello" example://demo
# [1] example://demo/hello.txt  score=0.91
#  1  Hello, World!
```

## 7. 注册 plugin

`connectors/__init__.py` 维护注册表：

```python
from .file import FilePlugin
from .postgres import PostgresPlugin
from .slack import SlackPlugin
from .web import WebPlugin
from .example import ExamplePlugin

REGISTRY = {
    cls.URI_SCHEME: cls
    for cls in [FilePlugin, PostgresPlugin, SlackPlugin, WebPlugin, ExamplePlugin]
}
```

framework 根据 URI 的 scheme 查表实例化。

## 8. 测试要求

每个 connector 必须有：

### contract test

framework 提供 `tests/connectors/_contract.py`，对任意 plugin 跑一组通用 assertion：

```python
@pytest.mark.parametrize("plugin", [PostgresPlugin, SlackPlugin, ...])
async def test_connector_contract(plugin):
    # stat 必须返回有效 FileStat
    # list 返回 list[Entry]，按 name 排序
    # read 范围内可重入
    # fingerprint 同 input 同 output
    # change_set 在无变化时 added/modified/deleted 全空
    # ...
```

新 connector 跑通这个就 PR-ready。

### fake connector 集成测试

不要求真连外部系统。提供 fixture-based fake：

```python
# tests/connectors/postgres/fixtures/tickets.sql
# tests/connectors/postgres/test_e2e.py

async def test_postgres_end_to_end(fake_postgres):
    plugin = PostgresPlugin(config=..., credential=...)
    # mfs add → 检查 objects 表
    # mfs ls → 检查列表
    # mfs head → 检查样本
    # mfs search → 检查 retrieval index
```

CI 自动跑 contract + fake。真连测试可选。

## 9. PR checklist

| 项 | 必须 |
|---|---|
| `connectors/<name>/` 下面文件齐全（plugin/config/connector/layout/sync/PROMPT.md/tests） | ✅ |
| `CONFIG_SCHEMA` 用 pydantic | ✅ |
| `CAPABILITIES` 准确（不撒谎说支持某能力但实际报错） | ✅ |
| `PROMPT.md` 写清 root 下面有什么对象、cat 行为、限制 | ✅ |
| `object_kind_of(path)` 覆盖该 connector 暴露的所有 path 模式 | ✅ |
| `fingerprint / change_set` 实现增量 | ✅ |
| 对象命名遵循 §10 的规范 | ✅ |
| contract test 全过 | ✅ |
| fake E2E test 至少跑通 add / ls / head / search | ✅ |
| docstring 提到所需 OAuth scope / 权限 | ✅ |
| 不在 `objects/` 加新 kind（如确实需要新 kind 先开 RFC） | ✅ |
| 不在 `pipeline/` 改通用组件 | ✅ |
| `mfs-server[<name>]` extra 声明 SDK 依赖 | ✅ |

## 10. 对象命名规范

每个 connector 决定自己 root 下面有什么 object、叫什么名字、什么 media_type。下面是必须遵守的规范。

### 10.1 文件名按数据形态选后缀

| 数据形态 | 后缀 | 例 |
|---|---|---|
| 多条结构化记录的集合 | `.jsonl` | `rows.jsonl`、`messages.jsonl`、`issues.jsonl`、`records.jsonl`、`comments.jsonl`、`users.jsonl` |
| 单个 schema / 元数据描述 | `.json` | `schema.json`、`database.json`、`workflows.json`、`index.json` |
| 长文本对象（connector 自己生成的） | `.md` | `pages/<url>.md`（web）、`<id>.md`（notion） |
| 真实文件 | 保留原文件名和原后缀 | `README.md`、`config.toml`、`chart.png`、`app.jsonl` |
| 目录节点 | 无后缀，路径末尾 `/` | `tables/`、`channels/`、`pulls/42/`、`pages/` |

### 10.2 集合用 JSONL，不要造目录里全是单 JSON

❌ **不要**：

```text
tickets/
  1.json
  2.json
  3.json
  ...
```

理由：scale 不好；`ls` 巨慢；agent 没法 head/grep 整个集合。

✅ **要**：

```text
tickets/
  schema.json                  # 元数据描述
  records.jsonl                # 全部 ticket，一行一个
  comments.jsonl               # 全部 comment（带 ticket_id 反向引用）
```

用 `mfs grep '"id":12' tickets/records.jsonl` 或 `export + jq` 取单条。

### 10.3 单 record 的精确定位走 locator，不暴露成 path

不要给单条 record / row / issue 分配独立 path。它们由搜索结果的 `locator` JSON 定位，详见 [05-search-and-retrieval.md §3](05-search-and-retrieval.md#3-locator-schema-per-connector).

例外：单条对象天然有持久 path 且数量可控时可以暴露（如 GitHub PR `pulls/42/diff.patch`）。

### 10.4 目录节点不能 cat

`cat` 一个目录路径返回 `is_directory` 错误。所有目录节点统一行为，不要在某些 connector 让 `cat dir/` 返回"目录概览"——那是 `ls` 的事。

### 10.5 真实文件透传

GitHub blob、S3 object、Drive file、本地文件这些**真有文件实体**的对象：

- 保留原文件名和后缀
- 不在路径上"装饰"任何东西
- `cat` 返回原始 bytes（除非是 PDF/DOCX 等 framework 知道怎么转 markdown 的类型）

### 10.6 命名词汇约定

常见对象的统一命名（每类 connector 暴露同概念时尽量用同名）：

| 概念 | 推荐名 |
|---|---|
| schema 描述 | `schema.json` |
| 全集合数据 | `<concept>s.jsonl`，复数（`rows`、`messages`、`issues`、`records`、`comments`、`users`、`threads`、`activities`） |
| 跨集合的元数据概览 | `database.json`、`workflows.json`、`index.json` |
| 当天 / 当前 partition | 路径段 `<yyyy-mm-dd>/`、`today/` 别名 |
| 附件目录 | `files/` |
| 真实文件 | 原名 |

## 11. 边界规则

| 想做的事 | 该不该做 |
|---|---|
| 给 `chunk_kind` 加一种新值 | ❌ 8 种已固定，要加走 framework RFC |
| 给 `object_kind` 加新值 | ❌ 同上 |
| 在 connector 里直接写 Milvus | ❌ 走 framework pipeline |
| 在 connector 里直接调 OpenAI embedding | ❌ 同上 |
| 在 connector 里读 `~/.mfs/cache/` | ❌ 走 framework storage adapter |
| 用新的 URI scheme（如 `myco://`） | ✅ 注册即可 |
| 让 cat 渲染特殊格式 | ✅ 在 `object_kind_of` 标个合适的 kind 用 framework 已有 handler |
| 在 connector 里写 schedule cron | ❌ 用户写 connector TOML 的 `schedule` 字段，framework scheduler 调 |
| 暴露不在 PROMPT.md 描述里的 path | ❌ 暴露 = 文档化 |
| 自定义 `tenant_id` 行为 | ❌ tenant_id 由 framework 注入 |

## 12. 写 connector 前的设计检查

写第一行代码前回答：

1. 你的 connector root 下要暴露哪些 object？写出来。
2. 每个 object 是什么 media_type、什么 object_kind？
3. 列目录 / 读对象的成本如何？需要 cache 吗？
4. 怎么判断对象变化？fingerprint 算什么？
5. 哪些对象要索引（进 chunk）？text_fields 默认是什么？
6. 能否下推 grep / search / tail？
7. 凭据是什么？OAuth scope 要哪些？
8. 用户必填配置最少是什么？

回答完了再开始写。
