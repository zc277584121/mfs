# Connector 注册与 Ingest

本文回答：每类 connector 暴露什么对象、怎么注册、怎么同步、怎么判断 stale。

`Connector` 是 MFS 对数据源的统称（postgres / slack / github / file / web / ...），跟业界 ETL/iPaaS 用法一致。**本地文件是一种 file connector**（scheme=`file`，用户写普通 path 即可），跟其他 connector 一视同仁。每个 connector 通过插件实现，详见 [07-contributing-connector.md](07-contributing-connector.md)。

## 1. Connector 注册

`mfs add` 是注册 + 同步的统一入口（幂等）。本地路径无需 config TOML；外部 connector 首次需要 `--config`。

```bash
# 本地路径（file connector，无需 config）
mfs add ./repo
mfs add .

# 外部 connector（首次：含估算 + confirm）
mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml
mfs add slack://eng --config .mfs/connectors/slack-eng.toml
mfs add web://acme-docs --config .mfs/connectors/acme-docs.toml

# 跳过 confirm
mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml --yes
```

正式 add 前想验证凭据和连通性，用 `mfs connector probe`（不写状态）：

```bash
mfs connector probe postgres://prod --config .mfs/connectors/prod-postgres.toml
```

注册后日常使用通过 connector URI 即可：

```bash
mfs add postgres://prod                      # 再同步（幂等）
mfs add postgres://prod --force-index        # 强制重 chunk + embed
mfs ls postgres://prod/public/tickets        # 浏览
mfs connector list                           # 看已注册的
```

connector root URI 由 `scheme://<alias>` 组成。`alias` 是用户起的名（`prod` / `eng` / `acme-docs`），在当前 namespace 内唯一（v0.4 单 namespace 下即全局唯一），会进入脚本和搜索结果；展示名放 `label`。本地路径的 connector URI 内部表示为 `file://./repo` 或 `file:///abs/path`，但用户日常写普通 path 即可。

## 2. Connector 类型清单（v0.4 目标）

| 类别 | Connectors |
|---|---|
| 文件 / 对象存储 | local file, GitHub repo, Google Drive, Feishu Docs, S3 / R2 / GCS / MinIO, SSH remote fs |
| 网页 / 网站 | web (HTML crawl → markdown) |
| 消息 | Slack, Discord, Gmail, 飞书群聊, Telegram, Email |
| 任务 / 协作 | GitHub Issues / PRs, Jira, Linear, Notion |
| 数据库 | Postgres, MySQL, MongoDB, BigQuery, Snowflake |
| 业务 SaaS | Salesforce, HubSpot, Zendesk |

每类的具体 path 布局见 §3。

## 3. 每类 connector 的 path 布局

每个 connector 决定自己 root 下面暴露哪些 object。**对象名带 media type 后缀**（`.json` / `.jsonl` / `.patch` / `.md` 等），跟 unix 和 Mirage 一致。**目录节点没有后缀**，`cat` 它返回 `is_directory`。

### File connector

```text
./repo/
  <原始文件树，保留原文件名和后缀>
```

代码、文档、图片、JSON、CSV 都按各自扩展名进入对应处理器。`mfs ls ./repo` 看到的就是真实目录。

**两种 scope**：file connector 实现复用同一份代码，根据 client / server 是否共享文件系统注入不同 scope：

| client/server 是否共享 fs | scope | 数据来源 |
|---|---|---|
| 共享 | `LocalFS(path)` | server 直接读 client 端写的真实路径 |
| 不共享 | `StagingArea(connector_id)` | server 读 client 通过 upload flow 上传的解压目录 |

file connector 代码上**抽象为"扫一个目录 root"**，不关心 root 是真本机 fs 还是 object_store 中的 staging 子目录。详见 [02-architecture.md §3.5](02-architecture.md#35-本地文件-upload-flow不共享-fs-场景) 和 [07-contributing-connector.md §3](07-contributing-connector.md#3-connectorplugin-契约)。

### Web connector

抓取网页并转 markdown 缓存。配置见 §6。

```text
web://<alias>/
  index.json                                   # crawl 后的页面索引（URL → title / fingerprint）
  pages/
    <url-path>.md                              # HTML 转 markdown，按 URL path 镜像
    api/auth.md
    guides/quickstart.md
  assets/
    <hash>.<ext>                               # 引用的图片 / 附件
```

`cat web://acme-docs/pages/api/auth.md` 返回 markdown；`cat web://acme-docs/assets/abc123.png` 返回图片字节。

### GitHub repo（代码 + issues/pulls）

```text
github://<alias>/
  <原始 repo 文件树，按 branch HEAD>
  _meta/
    issues.jsonl                               # 全部 issues
    pulls.jsonl                                # 全部 PRs
    pulls/<n>/diff.patch                       # 单 PR 的 diff
    pulls/<n>/reviews.jsonl                    # 单 PR 的 reviews
    pulls/<n>/comments.jsonl                   # 单 PR 的 comments
```

`_meta/` 用来跟真实文件树区分。

### Google Drive

```text
gdrive://<alias>/
  <镜像 Drive 文件树>
    Pricing.gdoc                               # cat → 导出为 markdown
    Roadmap.gsheet/                            # 目录
      Sheet1.jsonl
      Sheet2.jsonl
    Chart.png                                  # cat → bytes
    DPA.pdf                                    # cat → bytes；索引时转 markdown
```

### S3 / 对象存储

```text
s3://<alias>/
  <object key 树，保留原 key 名>
    app/2026-05-10/app.jsonl
    reports/Q1.csv
    images/header.png
```

### Slack

```text
slack://<alias>/
  channels/
    <name>__<channel-id>/
      <yyyy-mm-dd>/
        messages.jsonl                         # 当天消息
        threads.jsonl                          # 当天 thread 聚合
        files/                                 # 当天附件目录
          <name>__<F-id>.<ext>
  dms/
    <user>__<dm-id>/
      <yyyy-mm-dd>/
        messages.jsonl
        files/
  users.jsonl                                  # 全部用户
```

频道和 DM 目录用 `<sanitized-name>__<id>` 命名（ID 必有，因为名字可重复）。

### Discord

```text
discord://<alias>/
  channels/
    <name>__<channel-id>/
      <yyyy-mm-dd>/
        messages.jsonl
        files/
  forums/
    <forum>__<id>/
      <thread>__<id>/
        messages.jsonl
        files/
  users.jsonl
```

### Gmail

```text
gmail://<alias>/
  labels/
    inbox/<yyyy-mm>/
      messages.jsonl
      threads.jsonl
    support/<yyyy-mm>/
      ...
  attachments/
    <file_id>__<name>.<ext>
  users.jsonl                                  # contacts（可选）
```

### Postgres / MySQL

```text
postgres://<alias>/
  database.json                                # 跨 schema 概览
  <schema>/
    tables/
      <table>/
        schema.json                            # column / PK / FK / index
        rows.jsonl                             # 全部行（lazy，大表不物化）
    views/
      <view>/
        schema.json
        rows.jsonl
```

`head -n N rows.jsonl` 下推为 `SELECT ... LIMIT N`。首次 head 的预 cache 由 framework 内部处理（`head_cache`），用户不感知。

### MongoDB

```text
mongo://<alias>/
  <db>/
    collections/
      <col>/
        schema.json                            # 采样推断 + confidence
        documents.jsonl                        # lazy
```

### BigQuery / Snowflake

```text
bigquery://<alias>/
  <dataset>/
    tables/
      <table>/
        schema.json
        rows.jsonl                             # lazy
        partition/<partition-key>.jsonl        # 分区数据
```

### Linear / Jira / Notion database

```text
linear://<alias>/
  teams/
    <team>/
      issues.jsonl
      cycles.jsonl
  users.jsonl
  workflows.json

jira://<alias>/
  projects/
    <proj>/
      issues.jsonl
      sprints.jsonl
      versions.jsonl
  users.jsonl

notion://<alias>/
  pages/<id>.md                                # 页面文本
  databases/<id>/
    schema.json
    records.jsonl
```

### Zendesk / Salesforce / HubSpot

```text
zendesk://<alias>/
  tickets/
    schema.json
    records.jsonl                              # 全部 ticket
    comments.jsonl                             # 全部 comment（带 ticket_id）
  users/records.jsonl
  organizations/records.jsonl

salesforce://<alias>/
  <object>/                                    # Account / Opportunity / Case / ...
    schema.json
    records.jsonl
    activities.jsonl

hubspot://<alias>/
  <object>/                                    # contacts / companies / deals / ...
    schema.json
    records.jsonl
```

## 4. Ingest 流程

`mfs add` 触发的工作链：

```
mfs add <target>
  │
  ├─ 1. 路径解析
  │     - 是本地 path → file connector
  │     - 是 URI → 对应 connector plugin
  │
  ├─ 2. 注册或拉取 connector 配置
  │     - 首次：parse TOML、validate、写 connectors 表
  │     - 已注册：复用现有配置
  │
  ├─ 3. 探测对象集合（connector.list / connector.discover）
  │     - 写 objects 表（virtual_path / media_type / size_hint）
  │     - 对比上次发现，找出 added / deleted
  │
  ├─ 4. 对每个对象，按 fingerprint 决定要不要重建
  │     - 取上次记录的 upstream_fingerprint
  │     - 调 connector.fingerprint(path) 拿当前值
  │     - 不变跳过；变了入 worker queue
  │
  ├─ 5. Worker 执行 build job
  │     - cache 层：connector.read() → 写 object store + caches 表
  │     - chunk 层：按 object_kind 走对应 chunker
  │         document → markdown chunker
  │         code → AST chunker
  │         table_rows → row text chunker（按 text_fields 拼接）
  │         message_stream → thread/session aggregator
  │         record_collection → per-record chunker
  │         image → VLM description → 单 chunk
  │     - embedding → dense_vec + sparse_vec → 写 Milvus
  │
  ├─ 6. 删除消失的对象的 chunk
  │     - Milvus DELETE WHERE connector_uri = X AND object_uri NOT IN (current_set)
  │
  └─ 7. 更新 cursors / job status
```

执行位置：

| 部署 | queue 位置 | worker |
|---|---|---|
| 本机 server（共享 fs） | server 内 SQLite queue | server 内 worker pool |
| 远端 server | server 侧 Postgres queue（同 metadata DB） | `mfs-worker` 进程 |

数据流向：HTTP 主要走 control plane（path / option / status），唯一例外是 remote profile 下本地文件 upload（client 上传 zip bundle 到 server）。详见 [02-architecture.md §3](02-architecture.md#3-control-plane-vs-data-plane).

## 5. 变化检测

变化检测**对 framework 是黑盒**：framework 只规定两条最小契约（怎么通知变化 + 怎么算单 object 的 fingerprint），具体策略（cursor / scan / etag / hash / CDC）完全由 connector 自己决定。下面分三层讲：用户面、framework 内部、connector 实现。

### 5.1 Framework ↔ connector 的两条契约

```python
class ConnectorPlugin:
    # ① 同步：connector 自己算变化，流式 yield 每个变化的 object。
    #    cursor / manifest / etag / hash 怎么算、存哪、schema 长什么样，全在 connector 内部。
    async def sync(self) -> AsyncIterator[ObjectChange]: ...

    # ② 单 object 的当前 upstream fingerprint。
    #    framework 用这个跟自己存的对比，决定 cache / chunk / embedding 失效。
    async def fingerprint(self, path: str) -> str | None: ...


@dataclass
class ObjectChange:
    uri:  str
    kind: Literal["added", "modified", "deleted"]
```

仅此两条。没有 `ChangeSet` / `Cursor` / `new_cursor` / `unchanged` 列表——这些都是 connector 内部细节。

Connector 内部 state（cursor / manifest / etag 表 / ...）用 framework 提供的命名空间 KV store 持久化：

```python
async def sync(self):
    last = await self.state.get("last_ts")              # connector 自己定义的 key
    rows = await self.api.fetch(since=last)
    for r in rows:
        yield ObjectChange(r.uri, "modified" if r.was_seen_before else "added")
    await self.state.set("last_ts", new_ts)             # connector 自己存
```

framework 不 introspect `self.state` 里存的是什么——postgres 存 `updated_at`、slack 存 ts、s3 存 page token、file 存 manifest map、github 存 `(commit_sha, tree_sha)`，schema 各不相同。

### 5.2 Framework 内部：per-artifact fingerprint chain

这是 **framework 内部机制**，跟 connector 接口分开。framework 拿到 upstream fingerprint 后，自己组合 chunker / embedding model 等版本信息，算出每层产物的 fingerprint，决定哪层失效。

#### 5.2.1 产物模型：per-object_kind DAG

每个 object_kind（document / image / table_rows / message_stream / ...）有自己的**产物依赖图**。framework 用 `ArtifactSpec` 声明：

```python
@dataclass
class ArtifactSpec:
    kind:          str                  # "cache.converted_md" / "chunk.body" / "embedding"
    depends_on:    list[str]            # 上游产物 kind（"upstream" 是特殊源）
    config_inputs: list[str]            # 影响 fp 的 framework 配置 key
    storage:       Literal["cache_table", "milvus_field", "milvus_row"]
```

每个 object_kind 在 `processors/<kind>/` 里 declare 自己的 ArtifactSpec 列表。framework 把这些列表组装成 DAG，每条边代表"我依赖什么"。

各 object_kind 的具体 DAG：

```
document (md/code/text/...)
    ┌──────────┐
    │ upstream │
    └────┬─────┘
         ▼
    ┌───────────────┐    [chunker_name, chunker_config, tokenizer_version]
    │ chunk.body    │ ←──
    └────┬──────────┘
         ▼
    ┌───────────────┐    [embedding_model, embedding_version]
    │ embedding     │ ←──
    └───────────────┘

document (pdf/docx/gdoc)
    upstream
       │  [converter_name, converter_version]
       ▼
    cache.converted_md
       │  [chunker_name, chunker_config, tokenizer_version]
       ▼
    chunk.body
       │  [embedding_model, embedding_version]
       ▼
    embedding

image
    upstream
       │  [vlm_model, vlm_prompt, vlm_provider]
       ▼
    cache.vlm_text
       │  (1:1 wrap)
       ▼
    chunk.vlm_description
       │  [embedding_model]
       ▼
    embedding

table_rows (postgres rows.jsonl)
    upstream(per row)
       │  [text_fields, template_version]
       ▼
    chunk.row_text (per row)
       │  [embedding_model]
       ▼
    embedding

message_stream (slack messages.jsonl)
    upstream(per channel-day)
       │  [group_by, session_idle_min, agg_template_version]
       ▼
    chunk.thread_aggregate (per thread)
       │  [embedding_model]
       ▼
    embedding

table_schema (postgres schema.json)
    upstream
       │  [schema_summary_model, schema_summary_prompt]
       ▼
    chunk.schema_summary
       │  [embedding_model]
       ▼
    embedding

directory
    child_object_uris (隐式 upstream)
       │  [dir_summary_model, dir_summary_prompt]
       ▼
    chunk.directory_summary
       │  [embedding_model]
       ▼
    embedding
```

#### 5.2.2 fp 计算公式（framework 集中实现）

```python
# framework
def compute_artifact_fp(spec: ArtifactSpec, upstream_fps: dict, config: dict) -> str:
    parts = [upstream_fps[k] for k in spec.depends_on]   # 上游 fp（按声明顺序）
    parts += [str(config[k]) for k in spec.config_inputs]  # framework 配置 key
    return sha1("\x00".join(parts).encode()).hexdigest()
```

具体到每层公式（按 storage 字段对应到存哪里）：

```
upstream_fp                = connector.fingerprint(path)     ← objects.fingerprint

cache_fp(converted_md)     = sha1( upstream + converter_name + converter_version )      ← caches.fingerprint
cache_fp(vlm_text)         = sha1( upstream + vlm_model + vlm_prompt + vlm_provider )   ← caches.fingerprint
cache_fp(page_cache)       = sha1( upstream )                                            ← caches.fingerprint
cache_fp(head_cache)       = sha1( upstream + N )                                        ← caches.fingerprint

chunk_fp(body)             = sha1( cache_fp(converted_md) + chunker_name + chunker_config + tokenizer_version )
chunk_fp(row_text)         = sha1( upstream + text_fields + template_version )
chunk_fp(thread_aggregate) = sha1( upstream + group_by + agg_template_version )
chunk_fp(vlm_chunk)        = sha1( cache_fp(vlm_text) )
chunk_fp(schema_summary)   = sha1( upstream_schema + schema_summary_model + schema_summary_prompt )
chunk_fp(directory_summary)= sha1( child_object_uris + dir_summary_model + dir_summary_prompt )
                                                              ← Milvus 行 chunk_fingerprint

embedding_fp               = sha1( chunk_fp + embedding_model + embedding_model_version )
                                                              ← Milvus 行 embedding_fingerprint
```

Connector **不参与这套逻辑**，只提供 `upstream fingerprint`（per object）。下游 fp 全由 framework 算。

#### 5.2.3 Reconcile pass: framework 内部

每次 sync_job 处理流：

```
sync_job 开始
   ▼
① connector.sync() 流式 yield ObjectChange(uri, kind)
   │
   │  for each ObjectChange:
   │    - added  → 整套 build（没有 old fp 可比，直接走 4 步链）
   │    - modified → 走 Reconcile（4 步，下面）
   │    - deleted → 删 Milvus 行 + caches 行 + objects 行
   │
   ▼  yield 完
② Sweep pass: framework 扫描"未在 yield 集合里的 object"
   for object in connectors 表 WHERE connector_id = X AND object_uri NOT IN yielded_set:
      跑 Reconcile（4 步）—— 用于探测 framework 配置变化
```

**4 步 Reconcile**（per object, early-exit）：

```
Step 1: upstream 层比对
  new_fp = connector.fingerprint(path)
  old_fp = objects 表的 fingerprint 列
  if new_fp != old_fp:
      enqueue full rebuild → 整套 4 步重做
      return
  # else: upstream 没变，继续

Step 2: cache 层比对（仅对该 object_kind 适用的 cache_kind）
  for cache_kind in processor.applicable_caches(object_kind):
      new_fp = compute_artifact_fp(spec, {upstream: new_fp}, framework_config)
      old_fp = caches 表的 fingerprint 列
      if new_fp != old_fp:
          enqueue cache_rebuild → 触发下游 chunk + embed
          continue   # 这一支下游会被这个 task 触发

Step 3: chunk 层比对
  for chunk_row in milvus.query(connector_uri=X, object_uri=Y):
      new_fp = compute_artifact_fp(...)
      if new_fp != chunk_row.chunk_fingerprint:
          enqueue chunk_rebuild → 触发下游 embed
          continue

Step 4: embedding 层比对
      new_fp = compute_artifact_fp(...)
      if new_fp != chunk_row.embedding_fingerprint:
          enqueue embed_only (chunk 文本不动，只重 embed)
```

**Early-exit 性质**：上游 fp 不等就直接整套重做，跳过下游检查（下游一定也 stale）。

**为什么 sync 末尾要 sweep**：connector 只汇报 upstream 变化，不知道 framework 配置变了。**没有 sweep pass，换 embedding 模型后 `mfs add ./repo` 啥也不会发生**——connector 一看 manifest unchanged，啥都不 yield。sweep 让 framework 主动检查自己产物的状态。

**性能**：sweep 是只读比对（不读 upstream，不打 connector），只查 metadata DB + Milvus 字段。100 万 chunk 几秒钟级。

### 5.3 Milvus 上的失效行为

Milvus 不支持只更新一列。任何 fingerprint 变化最终都走 **DELETE 该行 + INSERT 新行**。区别只在"上游有多少步可复用"：

| 变化 | upstream read | converter | chunker | embedder | summary/vlm | Milvus 操作 |
|---|---|---|---|---|---|---|
| upstream 变 | ✓ | ✓ | ✓ | ✓ | ✓ | DELETE + INSERT |
| converter 升级（pymupdf） | 跳过 | ✓ | ✓ | ✓ | 受影响才重算 | DELETE + INSERT |
| chunker config 变 | 跳过 | 跳过 | ✓ | ✓ | 跳过 | DELETE + INSERT |
| summary model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓ (仅 summary chunks) | DELETE + INSERT (仅 summary 行) |
| vlm model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓ (仅 vlm chunks) | DELETE + INSERT (仅 vlm 行) |
| embedding model 变 | 跳过 | 跳过 | 跳过 | ✓ | 跳过 | DELETE + INSERT |

批量 DELETE-by-filter + 批量 INSERT 比逐条 upsert 快得多。Milvus 行同时存 `chunk_fingerprint` 和 `embedding_fingerprint`，sync 时按 chunk_id 对比，相等就跳过整行。

### 5.4 Connector 实现策略参考

**下面两张表是各类 connector 常见实现策略，仅供贡献者参考——framework 不规定怎么算变化、用什么 cursor、存什么 state。** 这些是经验性建议，每个 connector 实现时可自行调整。

**Fingerprint 算法（实现示例）**：

| Connector | 粒度 | Fingerprint 算法 |
|---|---|---|
| **file** | path | `size + mtime_ns` 快速判断；不等再算 `sha1(content)` |
| **web** | page | HTTP `ETag` 或 `Last-Modified`，否则 `sha1(html)` |
| **github code** | blob | `blob_sha` |
| **github issues/pulls** | record | `updated_at` |
| **gdrive** | file | `revision_id` |
| **feishu docs** | file | `version` |
| **s3 / r2 / gcs** | object | `etag`；版本桶用 `version_id` |
| **slack** | (channel, day) | 当天最后一条 message 的 `ts` |
| **discord** | (channel, day) | 最后 message id（snowflake 含时间） |
| **gmail** | thread | `thread.historyId` |
| **postgres / mysql** | row | `(pk, updated_at)` |
| **mongodb** | document | `(_id, version)` 或 `updatedAt` |
| **bigquery / snowflake** | partition | partition meta + row_count |
| **linear / jira / notion** | record | `updatedAt` |
| **zendesk** | record | `generated_timestamp` |
| **salesforce / hubspot** | record | `SystemModstamp` / `lastmodifieddate` |
| **ssh / generic remote fs** | path | `size + mtime`；可选 sha1 |

**同步策略（实现示例）**：

| Connector | 增量手段 | state 里存什么（举例） |
|---|---|---|
| **file** | scan 全量 + manifest diff | `{ path: (size, mtime_ns, sha1) }` 映射 |
| **web** | `revisit_interval` 触发 recrawl；ETag 检查 | `{ url: etag }` |
| **github code** | `compare $commit...HEAD` 拿变化 blob | `commit_sha` |
| **github issues/pulls** | `issues?since=$cursor&state=all` | `max(updated_at)` |
| **gdrive** | `changes.list?pageToken=$cursor` | `next_page_token` |
| **feishu docs** | OpenAPI 增量 events | `event_offset` |
| **s3** | `ListObjectsV2 StartAfter=$cursor`；周期全量 list 检测 delete | `last_key` |
| **slack** | per channel × day: `conversations.history?oldest=$ts` | `{ channel: max(ts) }` |
| **discord** | `messages?after=$id` | `{ channel: last_msg_id }` |
| **gmail** | `users.history.list?startHistoryId=$cursor` | `historyId` |
| **postgres rows** | `SELECT pk,updated_at WHERE updated_at>$cursor`；周期全量 pk diff 检 deleted | `max(updated_at)` |
| **postgres schema** | 探测 `pg_attribute` hash 变化 | hash of pg_attribute snapshot |
| **mongodb** | change streams（首选）或 `_id+version` 周期对比 | `resume_token` |
| **bigquery / snowflake** | `WHERE _PARTITIONTIME > $cursor` | `max(_PARTITIONTIME)` |
| **linear / jira / notion** | API + `updatedAfter=$cursor` | `max(updatedAt)` |
| **zendesk** | incremental export `tickets?start_time=$cursor` | `end_time` |
| **salesforce / hubspot** | bulk API delta | `max(SystemModstamp)` |

cursor / state schema 在 connector 内部自定义，framework 不参与。Sync 失败时 connector 决定怎么回滚或保留状态。

### 5.5 file connector 的实现示例（manifest 不含 framework 配置）

注意：file connector 的 manifest 只包含**文件本身的 fingerprint**（path / size / mtime_ns / sha1），**不包含** chunker / embedding model / converter 版本。这些 framework 层的配置变化由 [§5.2.3 Reconcile pass](#523-reconcile-pass-framework-内部) 的 sweep 步骤捕获，跟 manifest diff 无关。

### 5.6 Mid-job checkpoint API

[02 §5.7 规则 ③](02-architecture.md#57-一致性四条规则) 的默认行为是"state 末尾提交"：connector 中途崩 → state 不 commit → 下次重头跑。对 file / github code 这种**重跑 cheap** 的 connector 没问题；对 Slack / Gmail / Salesforce 等**外部 API rate limited** 的 connector，重头跑会被 rate limit 打爆。

framework 提供一个 **可选的** `self.state.checkpoint()` API，让 connector 在 sync 中途显式 commit state：

```python
class StateStore(Protocol):
    async def get(self, key: str) -> Any | None: ...
    async def set(self, key: str, value: Any) -> None: ...
    async def delete(self, key: str) -> None: ...

    async def checkpoint(self) -> None:
        """显式把当前 state_snapshot commit 到 connector_state 表。

        ⚠️ 仅适合 "cursor 推进型" state。半截 commit 必须是合法的"可接续状态"——
        即下次 sync 从这个 state 启动时，能正确从中断点接续，不丢数据也不重做太多。
        """
```

framework 拿到 checkpoint 调用 → 一个事务把 `connector_jobs.state_snapshot` 行 copy 到 `connector_state` 表 + 重置 snapshot → 之后即使本 job 失败，下次也从这里接续。

#### 5.6.1 哪些 state 形态能调 checkpoint

| state 形态 | 例 | 能调 checkpoint? | 理由 |
|---|---|---|---|
| **单调推进 cursor** | postgres `max(updated_at)` / slack `{ch: max(ts)}` / gmail `historyId` / linear `updatedAt` / s3 `last_key` / mongodb `resume_token` / bigquery `max(_PARTITIONTIME)` | ✅ 推荐 | cursor 是单调的，半截 commit 后下次从该 cursor 继续，跳过已处理的，不会丢数据 |
| **paged token** | gdrive `next_page_token` | ⚠️ 看 provider | gdrive `changes.list` token 长期有效 → OK；某些 provider token 短期失效 → 不推荐 |
| **commit hash 类（A→B）** | github code `commit_sha`、git tag、bigquery `snapshot_time` | ❌ 不要 | 一个 sync 是"从 commit A 跳到 commit B"的原子转换，中间没有合法的"半 commit" |
| **全量 map 快照** | file `{path: hash}`、web crawl `{url: etag}` | ❌ 不要 | state 是"对完整目录树/全站点的快照"。半截的 map 不能宣称是某一时刻的真相 |

**判断准则**：你的 state 在 `checkpoint()` 调用的那一瞬间，是不是一个**合法的"从此处接续"的起点**？

- 是 → 可以调
- 不是（半截 map / 中间转换状态） → 不要调

#### 5.6.2 推荐 checkpoint 频率

```python
# 推荐模式：每处理 N 个分组（page / channel / day）调一次
async def sync(self):
    last = await self.state.get("last_ts")
    page_index = 0
    async for page in self.api.fetch_paginated(since=last):
        for record in page:
            yield ObjectChange(record.uri, "modified")
        await self.state.set("last_ts", page.max_ts)
        page_index += 1
        if page_index % 50 == 0:                # 每 50 页 checkpoint
            await self.state.checkpoint()
```

频率取决于"重跑 50 页贵不贵 + checkpoint 自己的事务成本"。100k 量级数据 50~200 是合理区间。

#### 5.6.3 不调 checkpoint 的 connector 走默认行为

file / github code / web crawl 这些**不应该调** checkpoint 的 connector 走 [02 §5.7 规则 ③](02-architecture.md#57-一致性四条规则) 默认的"末尾提交"语义。它们都满足：

- 重头跑 cheap（本地 walk / git diff / 用户限定的 crawl 范围）
- chunk_id 幂等保证不重写 Milvus
- 即使全部 ObjectChange 重 yield 一遍，绝大多数 task 在 Reconcile 早退（upstream 没变）

不调 checkpoint 是合理选择，不要勉强加。

### 5.7 file connector 的 sync 完整流程示例

```
1. scan：os.walk(root) 应用 ignore rules（.gitignore + .mfsignore + 默认 binary 规则）
       → 得当前 paths 集合 current_paths

2. 对每个 path 与 self.state 里的 manifest 对比：
   - manifest 里有 + (size, mtime_ns) 完全一致 → 跳过
   - manifest 里有 + (size, mtime_ns) 变化 → 算 sha1(content)
     - sha1 跟 manifest 一致 → 只是 touch 了 mtime，更新 manifest.mtime_ns，跳过
     - sha1 不一致 → yield ObjectChange(path, "modified")
   - manifest 里没有 → yield ObjectChange(path, "added")

3. manifest 里有但 current_paths 没有 → yield ObjectChange(path, "deleted")

4. 更新 self.state 里的 manifest（写新 sha1 / size / mtime）
   ⚠️ file connector **不调 checkpoint**：全量 map 形态不能半截 commit（详见 §5.6.1）
```

manifest 是 file connector 自己定义的数据结构，存在 `self.state` 里。framework 不关心它的形状。其他 connector（postgres / slack / s3 / ...）按各自需要在 `self.state` 里存自己的 state。

## 6. Connector TOML 配置规则

```toml
# ───── 顶层：connector 元信息 ─────
[connector]
type = "postgres"                       # 必填，决定走哪个 plugin
root = "postgres://prod"                # 必填
label = "Production Postgres"
credential_ref = "secret:pg-prod-readonly"

# ───── connector 类型特定配置 ─────
[postgres]
schemas = ["public"]
max_read_rows = 10000
max_read_bytes = "10MiB"

# ───── 对象级配置（array-of-tables；按顺序匹配，先匹配优先） ─────

[[objects]]
match = "public.audit_*"
indexable = false

[[objects]]
match = "public.tickets"
text_fields = ["subject", "description", "latest_comment"]
metadata_fields = ["status", "priority", "assignee", "updated_at"]
locator_fields = ["id"]
chunk_strategy = "per_row"

[[objects]]
match = "public.events"
text_fields = ["event_type", "payload_summary"]
chunk_strategy = "windowed"
chunk_window = "30d"
chunk_max = 100000                      # 硬上限
```

`[[objects]]` array-of-tables：每条规则一段，按顺序匹配（先匹配优先），用 `match` 字段写 glob。比 quoted key 带星号的写法可读性好。

字段含义详见 [06-search-and-retrieval.md §4](06-search-and-retrieval.md#4-字段配置).

Web connector 例：

```toml
[connector]
type = "web"
root = "web://acme-docs"
label = "Acme Docs"

[web]
start_urls = ["https://docs.acme.com/"]
allowed_domains = ["docs.acme.com"]
sitemap = "https://docs.acme.com/sitemap.xml"     # 可选
max_pages = 1000
crawl_depth = 3
respect_robots_txt = true
revisit_interval = "7d"

[[objects]]
match = "pages/**"
chunk_strategy = "per_field_chunked"
```

framework 全局配置（chunk size、embedding model 等）放 server 端 `~/.mfs/server.toml`（本地 daemon）或 `/etc/mfs/server.toml`（远端部署），详见 [06-search-and-retrieval.md §10](06-search-and-retrieval.md#10-embedding--summary-providers).

**两层覆盖**：framework 全局 + connector/object 配置。`mfs config show --effective <uri>` 打印某路径的最终生效配置。

## 7. 凭据管理

connector TOML 里**不写明文凭据**，只写 `credential_ref`：

```toml
credential_ref = "secret:pg-prod-readonly"          # OS keychain
credential_ref = "env:PG_PROD_DSN"                  # 环境变量
credential_ref = "file:~/.mfs/secrets/pg-prod.toml" # 文件
credential_ref = "vault:secret/data/mfs/pg-prod"    # Vault (remote)
```

解析优先级和具体 schema 见 [02-architecture.md §7](02-architecture.md#7-凭据与认证).

## 8. Watch（本地路径专用）

```bash
mfs add ./repo --watch
mfs add ./repo --watch --interval 60s
```

- daemon 内启 watcher（`watchfiles` 或 OS-native）。
- watch 事件只作触发信号，**最终事实仍来自 scan + manifest 对比**。
- 查看正在 watch：`mfs status --watch`。
- 停止：`mfs remove ./repo` 或 Ctrl+C。
- 外部 connector 不支持 watch；用 scheduler 周期触发。

权限确认（首次 watch 一个目录时）：

```text
MFS local daemon will watch this directory:
  /repo

It will read file names, mtimes, sizes, and indexable file contents.
State is stored under ~/.mfs/.

Continue? [y/N]
```

## 9. Connector 能力声明

每个 connector plugin 声明能力，agent 通过 `mfs connector inspect <root>` 看到：

```json
{
  "connector_type": "postgres",
  "uri_scheme": "postgres",
  "sync": {
    "manual": true,
    "scheduled": true,
    "watch": false,
    "cursor": "updated_at",
    "full_scan": true,
    "delete_detection": true
  },
  "object": {
    "grep_pushdown": true,
    "search_pushdown": false,
    "paged_cat": true,
    "acl": false
  },
  "credentials": {
    "required": true,
    "schema": "PostgresCredential"
  }
}
```

framework 根据这些字段派发：`grep_pushdown=true` 时 `mfs grep` 走 SQL ILIKE，否则走线性扫；`paged_cat=true` 才允许 `cat --range`。

## 10. Sync 策略矩阵

| 策略 | 适合 connector | 触发方式 |
|---|---|---|
| 手动 | 所有 | `mfs add <uri>` |
| watch | 本地目录 | `mfs add . --watch` |
| 定时 | slack / github / SaaS / DB / web | daemon scheduler 或外部 cron |
| 游标 | slack / gmail / 部分 SaaS | provider cursor 或 updated_at |
| snapshot 对比 | s3 / gdrive / DB fallback | 周期列举对比 |
| append-only | logs / chat / events | 追加尾部 + 定期校准 |

daemon 内置简单 scheduler（基于 SQLite + APScheduler 风格），用户可在 connector TOML 里写 `schedule = "*/15 * * * *"` cron 表达式。

## 11. 错误恢复与重跑

整套 sync 的正确性靠**三条一致性规则**保证；具体队列/恢复实现见 [02-architecture.md §5](02-architecture.md#55-job-队列用关系型-db-做队列)。

### 11.1 三条一致性规则

**① Chunk-level 幂等**

```
chunk_id = sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)
                                                                       ← 确定性 hash
写 chunk = DELETE WHERE chunk_id = X + INSERT new row
```

任何 worker / 重试 / 并发，对同 chunk_id 的写都等效。`namespace_id` 必须进 hash——否则两个 namespace 注册同名外部数据源（如双方 alias 都叫 `prod`）会让 chunk_id 撞车互相覆盖。

**② Per-object 原子**

```
object_task.status = 'succeeded'
   ↔ 该 object 的所有 chunks 写入 Milvus 且 cache 已更新
```

中途任一步失败 → object_task 保持 'running' 或退 'failed'，下次 sync 重试整个 object。

**③ Connector state 末尾提交**

connector 在 `sync()` 过程中通过 `self.state.set()` 写入的 state，**framework 暂存在 `connector_jobs.state_snapshot`**；只有 sync_job 所有 task 成功才 commit 到 `connector_state` 表。

中途崩溃 → state 不 commit → 下次 sync 从上一个成功的 state 接续。`connector.sync()` 必须 idempotent。

framework 不暴露 `commit()` 给 connector——commit 时机由 framework 控制。

### 11.2 故障恢复

daemon / worker 重启时扫一次：

```sql
UPDATE connector_jobs   SET status='failed', error='interrupted'
  WHERE status='running' AND heartbeat < now() - interval '5 minutes';

UPDATE object_tasks SET status='pending'
  WHERE status='running'
    AND connector_job_id IN (SELECT id FROM connector_jobs WHERE status='failed');
```

object_tasks 重置为 pending → worker 重新取走重跑。chunk-level 幂等 + per-object 原子保证重跑结果一致。

connector_state 因为没 commit，下次 `mfs add` 自然从上一个成功状态接续。

### 11.3 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force-index` | 同上但所有 object 视为 'modified'，跳过 fingerprint 比对，强制重 chunk + embed |
| `mfs add ./path --force-upload` | 仅 upload flow：client 忽略 manifest cache 全量重传 + server 强制重 index |
| `mfs add <uri>` 在前次失败后 | 前次 state 未 commit；从上一个成功的 state 重跑——失败的 object 自然再次出现 |
| 第二次 `mfs add <uri>` 在前次还 running | 返回 `sync_already_running, see job <id>`（被 connector_jobs UNIQUE 约束拒绝） |
| `mfs remove <uri>` 在前次 sync running | preempt：sync 标 `cancelling`，当前 object_task 完成后退出，remove 接管 |
| `mfs add <uri>` 在 connector `status='removing'` 时 | 拒绝 `connector_removing`，等清理完才能重新注册 |

**不提供 `mfs job retry` 命令**——重跑 = 下次 `mfs add`，state 没 commit 时自然接续。

并发协调的完整语义表见 [02-architecture.md §5.11](02-architecture.md#511-操作之间的并发协调)。

### 11.4 单 object / 单 chunk 失败

- worker 处理单 object 时，可恢复错误（429 / timeout）自动 retry N 次（指数退避）。
- 超限 → object_task 标 'failed'，**整 sync_job 继续跑其他 task**（不因单个对象失败放弃全部）。
- 失败 task 在 `mfs status --verbose <uri>` 和 `mfs job inspect <id>` 可见。
- 下次 `mfs add` 重新出 ObjectChange → 重新创建 task → 再试一次。

## 12. 从当前代码版本迁移

当前代码（0.3.x）的概念与 v0.4 的映射：

| 当前 | v0.4 |
|---|---|
| `mfs add .` 本地索引（同进程） | `mfs add .` 通过 HTTP 调 daemon |
| `~/.mfs/milvus.db` + `~/.mfs/state.db` | 同位置，schema 升级（要 `mfs add . --force-index` 重建一次） |
| `mfs status` | 同名，语义扩展（增加 connector / job 维度） |
| `mfs add --watch` | 同 |
| `mfs add --force` | 改名 `--force-index`（语义不变）；remote profile 下另有 `--force-upload` |
| Milvus chunk schema 字段 `account_id` | 改名 `namespace_id`（多租户底层主键，详见 [02 §9](02-architecture.md#9-多租户与-namespace-设计)）。**Milvus 不支持字段重命名 in-place**——升级时必须 drop collection 重建，所有数据走一次重 index。这一步在 release note 里要明确标记。 |
| Milvus chunk_id 公式 `sha1(source + start + end + content_hash + model)` | 改 `sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)`。新 schema 需要重 build，跟上一行的 collection rebuild 一并完成 |
| 没有外部数据源概念 | 加 `mfs add <uri> --config X` 注册外部 connector |
| 没有本机 server | 用 `mfs serve start` 启动；`MFS_AUTOSTART=1` 时首次 `mfs add` 也会触发 |

升级动作：

```bash
brew install mfs           # CLI（Rust binary）
uv tool install mfs-server # server（Python）
mfs serve start
mfs profile add local --url http://127.0.0.1:8765
mfs profile use local
mfs add . --force-index    # 重建本地索引（schema 升级；底下会 drop 旧 collection）
```

> ⚠️ **Release note 必须标注**：v0.3 → v0.4 升级会 drop Milvus collection（字段重命名 + chunk_id 公式变更）。所有索引需要从 upstream 重 build。本机部署用户 `mfs add . --force-index` 重跑一次即可；CS 部署用户参考运维迁移文档。
