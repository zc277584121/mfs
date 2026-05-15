# Search 与 Retrieval

本文回答：Milvus collection schema 怎么设计，每个 connector 把什么内容写进 Milvus，用户怎么配字段，search 流程怎么走。

## 1. Milvus collection schema

**默认全 MFS 只有一张 Milvus collection**：`mfs_chunks`。所有 connector / 所有 object_kind / 所有 chunk_kind 共用。

```python
collection_name = "mfs_chunks"
partition_key   = "connector_uri"        # 每个 connector 一个 partition，加速过滤

fields = [
    Field("chunk_id",       VARCHAR(128),  primary=True),
    Field("tenant_id",      VARCHAR(64),   index="scalar", default="default"),
    Field("connector_uri",  VARCHAR(256),  partition_key=True),
    Field("object_uri",     VARCHAR(1024), index="scalar"),
    Field("locator",        JSON),
    Field("content",        VARCHAR(65535)),                  # 召回展示 + BM25 输入
    Field("dense_vec",      FLOAT_VECTOR(N)),                 # N 来自 embedding model
    Field("sparse_vec",     SPARSE_FLOAT_VECTOR),             # Milvus 内建 BM25
    Field("chunk_kind",     VARCHAR(32),   index="scalar"),
    Field("metadata",       JSON),                            # connector-specific filter 字段
    Field("chunk_fingerprint",     VARCHAR(64)),
    Field("embedding_fingerprint", VARCHAR(64)),
    Field("indexed_at",     INT64),
]

index_params = {
    "dense_vec":  {"type": "HNSW",  "metric_type": "COSINE", "params": {"M":16,"efConstruction":200}},
    "sparse_vec": {"type": "SPARSE_INVERTED_INDEX", "metric_type": "BM25"},
}
```

### Partition by connector

`partition_key=connector_uri` 是 schema 阶段必须定死的字段。它让几个操作显著加速：

| 操作 | partition 带来的好处 |
|---|---|
| `mfs search "..." postgres://prod` | 只扫 `postgres://prod` 这一个 partition，跳过其他 connector |
| `mfs search "..." --all` | 多 partition 并行扫，scatter-gather |
| `mfs connector remove postgres://prod` | 直接 `drop_partition`，比 `delete by filter` 快几个数量级 |
| 大数据量切 collection 时迁移 | 按 partition 物理切到 `per_connector` collection 容易 |

后改 partition key 需要数据迁移，所以一开始定下来。

### tenant_id 字段是多租户预留

v0.4 默认 `tenant_id = "default"`，所有查询自动 filter `tenant_id = current`。预留这个字段后，未来加多租户时不需要改 schema。详见 [06-architecture.md §9](06-architecture.md#9-多租户预留).

### 可选 collection 策略（server 端配置）

```toml
# server 端配置
[milvus]
collection_strategy = "single"          # 默认；所有 connector 一张 collection
# collection_strategy = "per_connector" # 一 connector 一张 collection（隔离强；跨 search 多 RPC）
# collection_strategy = "per_tenant"    # 多租户场景
```

v0.4 默认 `single`。换策略时需要数据迁移工具（roadmap）。

### 字段说明

| 字段 | 含义 |
|---|---|
| `chunk_id` | uuid 或 `sha1(object_uri + locator + chunk_kind)`；幂等写入 |
| `tenant_id` | 多租户预留，v0.4 默认 `"default"` |
| `connector_uri` | 包含该 chunk 的 connector root，如 `postgres://prod` |
| `object_uri` | chunk 来自哪个 object，如 `postgres://prod/public/tickets/rows.jsonl` |
| `locator` | object 内单元定位，per-connector schema（见 §3） |
| `content` | chunk 文本：dense embed 输入 + BM25 输入 + 召回展示 |
| `dense_vec` | embedding 向量 |
| `sparse_vec` | Milvus 内置 BM25 sparse 向量 |
| `chunk_kind` | 8 种之一（见 §2） |
| `metadata` | filter / 展示用的 JSON：`{status, priority, author, updated_at, ...}` |
| `chunk_fingerprint` | stale check：上游 + chunk config |
| `embedding_fingerprint` | stale check：chunk + embedding model |
| `indexed_at` | unix ms |

## 2. chunk_kind 枚举（framework 固定）

固定 8 种，**connector 不能私加**：

| chunk_kind | 来源 | 例 |
|---|---|---|
| `body` | document / code 的正常段落 | 一段 markdown / 一个函数体 |
| `row_text` | DB row / SaaS record / 单条 issue | 单条 ticket 拼接字段后的文本 |
| `thread_aggregate` | 多消息聚合 | Slack thread / Discord forum thread / Gmail thread |
| `record_aggregate` | record 含子项 | GitHub issue 含 comments / Zendesk ticket 含 comments |
| `summary` | 显式 LLM summary | 文件 / record / thread 的 LLM 摘要 |
| `vlm_description` | 图片描述 | png / jpg / 表情图 |
| `directory_summary` | 目录概览 | 一个目录的功能描述 |
| `schema_summary` | 表/集合 schema 推导描述 | postgres 表 + 业务字段的语义描述 |

新增 chunk_kind 要走框架升级流程，不通过 connector TOML 加。

search 默认全 kind 召回；filter 时用 `mfs search ... --kind body,summary` 限定。

## 3. locator schema per connector

每个 connector 的 `locator` JSON schema 是稳定契约，agent 按这个 parse：

| Connector | locator schema |
|---|---|
| `file` | `null`（chunk 元数据自带 `lines: [start, end]`） |
| `github code` | `null`（同 file） |
| `web` | `null`（页面级；元数据自带 lines） |
| `github issues / pulls` | `{"number": int}` |
| `gdrive` | `{"file_id": str, "revision_id": str}` |
| `s3 / r2 / gcs` | `null`（同 file） |
| `slack` | `{"channel": str, "date": str, "thread_ts": str}` |
| `discord` | `{"channel_id": str, "thread_id": str}` |
| `gmail` | `{"thread_id": str}` |
| `postgres / mysql` | `{"schema": str, "table": str, "pk": {<pk_fields>}}` |
| `mongodb` | `{"db": str, "collection": str, "_id": str}` |
| `bigquery / snowflake` | `{"dataset": str, "table": str, "partition": str, "pk": {...}}` |
| `linear` | `{"team": str, "id": str}` |
| `jira` | `{"project": str, "key": str}` |
| `notion` | `{"database_id": str, "page_id": str}` |
| `zendesk` | `{"object": str, "id": int}`  ── e.g. `{"object":"tickets","id":123}` |
| `salesforce / hubspot` | `{"object": str, "id": str}` |

要打开单条对象用 `source` + `locator` 组合，agent 用 `mfs grep '"id":12' <source>` 或 `mfs export <source> ./tmp.jsonl && jq 'select(.id == 12)'`。

## 4. 字段配置

`connector TOML` 的 `[[objects]]` 段配三类字段 + chunk_strategy：

```toml
[[objects]]
match = "public.tickets"
text_fields       = ["subject", "description", "comments[].body"]
metadata_fields   = ["status", "priority", "assignee", "updated_at"]
locator_fields    = ["id"]
chunk_strategy    = "per_row"        # per_row | per_group | per_field_chunked

# 选填
group_by          = "thread_ts"      # chunk_strategy="per_group" 时
session_idle_min  = 10               # group_by="session" 时
text_template     = "..."            # 覆盖默认拼接模板（jinja-style）
max_text_chars    = 8000             # 单 chunk 上限，超出自动转 per_field_chunked
filter_expr       = "state == 'open'" # 只索引部分记录
chunk_max         = 100000           # 索引行数硬上限
indexable         = true             # 设 false 完全不进 Milvus
```

### text_fields

进 `chunk.content`。embedding 和 BM25 都用这份文本。

多字段时按默认模板拼接：

```
{field1_name}: {field1_value}

{field2_name}: {field2_value}
...
```

数组字段（`comments[].body`）按列表展开：

```
comments:
- {body_1}
- {body_2}
```

### metadata_fields

进 `chunk.metadata`。用于 search filter（`--filter status=open`）和结果展示。null 值跳过。数组用 `[*]`（如 `labels[*]`）展开。

### locator_fields

从 record 取值组成 `locator` dict。多字段时按字段名做 key：`locator_fields=["schema","table","id"]` → `{"schema":"public","table":"tickets","id":12}`。

### JSONPath 表达式（简化子集）

| 表达 | 含义 |
|---|---|
| `subject` | 直接字段 |
| `user.email` | 嵌套 |
| `comments[]` | 数组所有项（必须配 `.field` 后缀取 dict 内字段） |
| `comments[].body` | 数组每个元素的 body 拼接 |
| `comments[0:5].body` | 前 5 个 |
| `labels[*]` | 数组扁平 join |

只支持这 5 种。复杂表达请 connector plugin 在 fetch 时 pre-join。

### chunk_strategy

| strategy | 适用 | 一个 chunk 是 |
|---|---|---|
| `per_row` | issue / ticket / DB row / SaaS record | 一条记录 |
| `per_group` | slack / discord / gmail | 一组（thread / session / time-window） |
| `per_field_chunked` | 单字段长文本（如 Notion page body、web page） | 该字段按 markdown 切多个 chunk |

`per_field_chunked` 也是 fallback：text 超过 `max_text_chars` 时自动从 `per_row` 升级。

### filter_expr

只索引部分记录，避免 chunk 爆炸。语法是 Python expr（sandbox eval）：

```toml
filter_expr = "state == 'open' and priority in ['high', 'critical']"
filter_expr = "updated_at > '2025-01-01'"
filter_expr = "len(description) > 50"
```

### chunk_max

硬上限。超过时停止 chunk 生成并报 `chunk_max_exceeded`，让用户看到选择：要么加 `filter_expr`，要么开 windowed 策略。

## 5. 内置 preset

公共 SaaS / 消息 connector 自带 preset，用户不配也能跑：

```python
# framework 内置（伪代码）
PRESETS = {
    "github.issues": dict(
        text_fields=["title", "body", "comments[].body"],
        metadata_fields=["state", "labels[*]", "author", "assignees[*]", "updated_at"],
        locator_fields=["number"],
        chunk_strategy="per_row",
    ),
    "github.pulls": dict(
        text_fields=["title", "body", "reviews[].body", "comments[].body"],
        metadata_fields=["state", "draft", "labels[*]", "author", "merged_at", "updated_at"],
        locator_fields=["number"],
        chunk_strategy="per_row",
    ),
    "slack.messages": dict(
        chunk_strategy="per_group",
        group_by="thread_ts",
        fallback_group_by="session",
        session_idle_min=10,
        text_fields=["text"],
        metadata_fields=["channel", "users[*]", "start_ts", "end_ts"],
        locator_fields=["channel", "date", "thread_ts"],
    ),
    "discord.messages": dict(
        chunk_strategy="per_group",
        group_by="thread_id",
        text_fields=["content"],
        metadata_fields=["channel_id", "users[*]"],
        locator_fields=["channel_id", "thread_id"],
    ),
    "gmail.messages": dict(
        chunk_strategy="per_group",
        group_by="thread_id",
        text_fields=["subject", "body"],
        metadata_fields=["from", "to[*]", "cc[*]", "date", "labels[*]"],
        locator_fields=["thread_id"],
    ),
    "zendesk.tickets": dict(
        text_fields=["subject", "description", "comments[].body"],
        metadata_fields=["status", "priority", "requester_id", "assignee_id", "tags[*]", "updated_at"],
        locator_fields=["id"],
        chunk_strategy="per_row",
    ),
    "linear.issues": dict(
        text_fields=["title", "description", "comments[].body"],
        metadata_fields=["state", "team", "assignee", "labels[*]", "updated_at"],
        locator_fields=["team", "id"],
        chunk_strategy="per_row",
    ),
    "jira.issues": dict(
        text_fields=["summary", "description", "comments[].body"],
        metadata_fields=["status", "priority", "assignee", "labels[*]", "updated_at"],
        locator_fields=["key"],
        chunk_strategy="per_row",
    ),
    "web.page": dict(
        text_fields=["title", "body"],
        metadata_fields=["url", "domain", "fetched_at"],
        locator_fields=["url"],
        chunk_strategy="per_field_chunked",     # body 长，按 markdown 切
    ),
    # ... salesforce.account / hubspot.contact / 等
}
```

**Postgres / MySQL / MongoDB / 用户自定义 SaaS 对象没有 preset**——字段都是业务定义的，必须显式配。缺失时报错：

```text
Connector postgres://prod registered.
Warning: public.tickets has no text_fields configured.
search/grep will be unavailable for this object until you add:

  [[objects]]
  match = "public.tickets"
  text_fields = ["..."]
  locator_fields = ["id"]
```

## 6. 各 object_kind 的 chunk 来源

每类 object 进 Milvus 的内容是什么样的：

| object_kind | 进 Milvus 的 chunk_kind | 每条 chunk 是 |
|---|---|---|
| `document` (md/pdf/docx/gdoc/html→md) | `body` + 可选 `summary` | 一个 markdown 段落 / 一段抽取文本 |
| `code` | `body` | 一个函数 / 一个 class / 一段 region（AST 切分） |
| `image` | `vlm_description` | VLM 给出的描述文本 |
| `text_blob` (json/csv/log) | 默认不进 | 不索引；grep 兜底 |
| `binary` | 不进 | metadata only |
| `table_rows` (rows.jsonl) | `row_text` | 按 text_fields 拼接的单行 record 文本 |
| `table_schema` (schema.json) | 可选 `schema_summary` | LLM 推导的 schema 描述 |
| `message_stream` (messages.jsonl) | `thread_aggregate` + 可选 `summary` | 一个 thread / session 的对话拼接 |
| `record_collection` (issues.jsonl/records.jsonl) | `record_aggregate` 或 `row_text` | 一条 record 拼字段 |
| `directory` | 可选 `directory_summary` | LLM 给出的目录功能描述 |

## 7. Search 流程

```
mfs search "..." <path> --top-k 10
  │
  ├─ 1. embed query → dense_vec
  │
  ├─ 2. 解析 <path>:
  │     - 本地路径 → file connector 的 URI
  │     - URI → 解析 partition + path prefix
  │     - --all → 全 partition
  │
  ├─ 3. Milvus hybrid search:
  │     filter = {
  │       tenant_id     = current_tenant,
  │       connector_uri in [<partition>],
  │       object_uri    LIKE '<prefix>%' (optional),
  │       chunk_kind    in [...] (optional --kind),
  │       metadata.<field> = ... (optional --filter)
  │     }
  │     ranker  = RRF(dense_score, sparse_score)
  │     limit   = top_k
  │
  ├─ 4. 后处理:
  │     - 同 object_uri 去重 / 保留 top-1（可选 --collapse）
  │     - 按 score 排序
  │
  └─ 5. 渲染：{source, locator, content, score, metadata}
```

### 模式

```bash
mfs search "..." <path>                 # hybrid (默认)
mfs search "..." <path> --mode dense    # 仅 dense
mfs search "..." <path> --mode sparse   # 仅 BM25
```

### Filter

```bash
mfs search "login" postgres://prod --filter status=open
mfs search "login" postgres://prod --filter status=open,priority=high
mfs search "login" slack://eng --kind thread_aggregate
mfs search "login" --all --filter connector_type=postgres
```

filter 直接翻译成 Milvus `expr`，作用在 scalar field 或 JSON metadata 上。

### Collapse

默认按 chunk 召回，可能同一文件多个 chunk 都命中。`--collapse object` 让结果按 object_uri 去重，只保留每个 object 的 top score：

```bash
mfs search "session" ./src --top-k 5 --collapse object
```

## 8. Grep 流程

详细派发见 [04-browse-and-read.md §6](04-browse-and-read.md#6-grep-的派发)。本节补充 Milvus 召回路径。

对**已建 chunk 索引**的 object，`grep --mode index` 可走 Milvus sparse_vec（BM25）路径：

```
Milvus sparse search:
  filter: tenant_id = X AND connector_uri = Y AND object_uri = Z
  query : sparse vector from pattern
  返回带 chunk content 的 hits → 按行号在 chunk 内定位
```

优势：跨大文件查关键词不用线性扫。
缺点：召回 chunk 文本，不能保证精确字面匹配（BM25 是统计相关，不是 regex）。

`grep` 默认是字面精确匹配（符合 unix 习惯）；只有 `--mode index` 才走 Milvus BM25。

## 9. 跨 connector search 示例

```bash
$ mfs search "why did we change pricing limit" --all --top-k 5

[1] linear://product/teams/Pricing/issues.jsonl  score=0.882
    chunk_kind=record_aggregate  locator={"team":"Pricing","id":"LIN-88"}
    title: Lower enterprise pricing cap to $10k/month
    state: Done

[2] github://product/_meta/pulls.jsonl  score=0.821
    chunk_kind=record_aggregate  locator={"number":312}
    title: Update pricing rate limit config
    merged_at: 2026-04-22

[3] slack://eng/channels/pricing__C09/2026-04-22/threads.jsonl  score=0.794
    chunk_kind=thread_aggregate  locator={"channel":"pricing__C09","date":"2026-04-22","thread_ts":"1713780000.111"}
    thread: 12 messages, U1/U2/U3
    discussion of new pricing cap and rollout plan
```

Agent 可以同时拿到 Linear issue、GitHub PR、Slack thread 三类不同 connector 的结果，envelope 一样。

## 10. Embedding & Summary providers

framework 全局 `~/.mfs/config.toml` 配：

```toml
[embedding]
provider = "openai"             # openai | onnx | google | voyage | jina | mistral | ollama | local
model    = "text-embedding-3-small"
batch_size = 100

[summary]
enabled  = "auto"
provider = "openai"
model    = "gpt-4o-mini"
max_tokens = 800
min_size_kb = 8                 # auto 时阈值

[vlm]
provider = "openai"
model    = "gpt-4o-mini"        # 必须是 vision 模型
prompt   = "Describe this image..."
```

切换 embedding model 会让所有 chunk 的 `embedding_fingerprint` 失效，触发 **DELETE + re-INSERT**（chunk 文本不变，只 embed 重算；Milvus 不支持列级 update，所以仍是整行替换）。批量 DELETE-by-filter + 批量 INSERT 比逐条 upsert 快很多。

切换 summary / vlm 模型只影响对应 `chunk_kind` 的行（`summary` / `directory_summary` / `schema_summary` / `vlm_description`），其他 chunk 不动。通过 `chunk_kind` filter 做精准 DELETE。

完整的 per-artifact fingerprint 设计见 [03-connector-and-ingest.md §5.1](03-connector-and-ingest.md#51-per-artifact-fingerprint-chain).

## 11. 大对象索引控制

千万行的表如果默认 `chunk_strategy=per_row`，一次 `mfs add` 会写出千万 chunk + 调千万次 embedding。控制手段：

### 11.1 `chunk_max` 硬上限

```toml
[[objects]]
match = "public.events"
text_fields = ["event_type", "payload_summary"]
chunk_max = 100000
```

超过停止 + 报错：

```text
chunk_max_exceeded: public.events
This object would produce ~12.4M chunks, exceeding chunk_max=100000.
Add filter_expr or chunk_strategy to limit:

  [[objects]]
  match = "public.events"
  filter_expr = "updated_at > '2026-04-01'"
  # or
  chunk_strategy = "windowed"
  chunk_window = "30d"
```

### 11.2 windowed 策略

```toml
[[objects]]
match = "public.events"
chunk_strategy = "windowed"
chunk_window = "30d"            # 只索引最近 30 天（按 updated_at）
```

### 11.3 sampled 策略

```toml
[[objects]]
match = "public.audit_log"
chunk_strategy = "sampled"
sample_rate = 0.01              # 1% 抽样
```

### 11.4 estimate + 确认（首次 add）

```text
$ mfs add postgres://prod
Estimated work:
  scan: 12.4M rows across 38 tables
  embedding: ~2.4M tokens (~$48 at text-embedding-3-small)
  duration: ~8h on 4 workers
  storage: ~3.2GB index + cache

Continue? [y/N]
  Or limit scope:
    mfs add postgres://prod --tables-only public.tickets,public.accounts
    mfs add postgres://prod --schema-only
```

## 12. 删除与一致性

ingest 时如果对象消失：

```
Milvus DELETE WHERE tenant_id = X
                AND connector_uri = Y
                AND object_uri NOT IN (current_object_set)
```

ingest 时如果对象内 record 消失（per_row 模式）：

```
Milvus DELETE WHERE tenant_id = X
                AND connector_uri = Y
                AND object_uri = Z
                AND locator->>'id' NOT IN (current_pk_set)
```

per_row 模式下用 locator 做 record-level 删除。per_group 模式（slack thread）只能用粗粒度（删整 group 重新写）。

## 13. 搜索可用性 (search availability)

`mfs status` 输出包含每个 connector 的 search 状态：

| 状态 | 含义 |
|---|---|
| `available` | 该 connector 至少一个对象有 chunks 在 Milvus |
| `partial` | 部分对象有 chunks，部分未建 |
| `building` | 正在构建中 |
| `unavailable` | 无任何 chunks（未配 text_fields，或全 indexable=false） |

```text
$ mfs status postgres://prod
Connector: postgres://prod
Health: ok
Sync:   last 2026-05-15T07:00, status=fresh
Index:
  tables/public/tickets/rows.jsonl   12453 chunks (fresh)
  tables/public/accounts/rows.jsonl  890 chunks (fresh)
  tables/public/events/rows.jsonl    not indexed (chunk_max_exceeded)
  tables/public/audit_log/rows.jsonl not indexed (indexable=false)
Search: partial
```

## 14. JSON envelope (search/grep)

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "lines": null,
  "content": "subject: Login broken after SSO migration\n\ndescription: Enterprise users cannot complete SSO redirect.",
  "score": 0.842,
  "locator": {
    "schema": "public",
    "table": "tickets",
    "pk": {"id": 12}
  },
  "metadata": {
    "kind": "search",
    "chunk_kind": "row_text",
    "connector_type": "postgres",
    "media_type": "application/x-ndjson",
    "fields": {
      "status": "open",
      "priority": "high",
      "assignee": "alice",
      "updated_at": "2026-05-10T12:30:00Z"
    }
  }
}
```
