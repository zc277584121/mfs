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
mfs add postgres://prod --force              # 强制重建
mfs ls postgres://prod/public/tickets        # 浏览
mfs connector list                           # 看已注册的
```

connector root URI 由 `scheme://<alias>` 组成。`alias` 是用户起的名（`prod` / `eng` / `acme-docs`），在 workspace 内唯一，会进入脚本和搜索结果；展示名放 `label`。本地路径的 connector URI 内部表示为 `file://./repo` 或 `file:///abs/path`，但用户日常写普通 path 即可。

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

这是 **framework 内部机制**，跟 connector 接口分开。framework 拿到 upstream fingerprint 后，自己组合 chunker / embedding model 等版本信息，算出每层 cache 的 fingerprint，决定哪层失效。

```
cache_fp(converted_md)     = sha1( upstream + converter_name + converter_version )
cache_fp(vlm_text)         = sha1( upstream + vlm_model + vlm_prompt + vlm_provider )
cache_fp(page_cache)       = sha1( upstream )
cache_fp(head_cache)       = sha1( upstream + N )

chunk_fp(body)             = sha1( cache_fp(converted_md) + chunker_name + chunker_config + tokenizer_version )
chunk_fp(row_text)         = sha1( upstream + text_fields + template_version )
chunk_fp(thread_aggregate) = sha1( upstream + group_by + agg_template_version )
chunk_fp(vlm_chunk)        = sha1( cache_fp(vlm_text) )
chunk_fp(summary)          = sha1( cache_fp + summary_model + summary_prompt + summary_provider )
chunk_fp(schema_summary)   = sha1( upstream_schema + schema_summary_model + schema_summary_prompt )
chunk_fp(directory_summary)= sha1( child_object_uris + dir_summary_model + dir_summary_prompt )

embedding_fp               = sha1( chunk_fp + embedding_model + embedding_model_version )
```

每种 artifact 声明自己依赖的 input set，framework 自动算 fp。改 chunker config / 换 embedding model / 升级 converter 时，对应层 fp 变 → 该层 stale → 自动 rebuild。

Connector 不参与这套逻辑，只提供 `upstream fingerprint`。

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

### 5.5 file connector 的实现示例

完整流程：

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
chunk_id = sha1(object_uri + locator + chunk_kind)    ← 确定性 hash
写 chunk = DELETE WHERE chunk_id = X + INSERT new row
```

任何 worker / 重试 / 并发，对同 chunk_id 的写都等效。

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
| `mfs add <uri> --force` | 同上但所有 object 视为 'modified'，跳过 fingerprint 比对，强制重建 chunks |
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
| `~/.mfs/milvus.db` + `~/.mfs/state.db` | 同位置，schema 升级（要 `mfs add . --force` 重建一次） |
| `mfs status` | 同名，语义扩展（增加 connector / job 维度） |
| `mfs add --watch` | 同 |
| 没有外部数据源概念 | 加 `mfs add <uri> --config X` 注册外部 connector |
| 没有本机 server | 用 `mfs serve start` 启动；`MFS_AUTOSTART=1` 时首次 `mfs add` 也会触发 |

升级动作：

```bash
brew install mfs           # CLI（Rust binary）
uv tool install mfs-server # server（Python）
mfs serve start
mfs profile add local --url http://127.0.0.1:8765
mfs profile use local
mfs add . --force         # 重建本地索引（schema 升级）
```
