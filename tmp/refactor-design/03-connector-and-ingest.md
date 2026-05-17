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

| profile.kind | queue 位置 | worker |
|---|---|---|
| `local` | daemon 内 SQLite queue | daemon 内 worker pool |
| `remote` | 服务端 Redis/Postgres queue | `mfs-worker` 进程 |

数据流向：HTTP 只走 control plane（path / option / status），数据（文件 bytes、记录内容、chunk 文本）都在 server 内部完成。详见 [06-architecture.md §3](06-architecture.md#3-control-plane-vs-data-plane).

## 5. Fingerprint 与变化检测

### 5.1 Per-artifact fingerprint chain

每种产物（cache / chunk / embedding）的 fingerprint = **sha1 of (上游 inputs + 所有影响产物的配置和工具版本)**。改任何一项相关项 → 自动失效相关产物。

```
cache_fp(converted_md)    = sha1( upstream + converter_name + converter_version )
cache_fp(vlm_text)        = sha1( upstream + vlm_model + vlm_prompt + vlm_provider )
cache_fp(page_cache)      = sha1( upstream )
cache_fp(head_cache)      = sha1( upstream + N )

chunk_fp(body)            = sha1( cache_fp(converted_md) + chunker_name + chunker_config + tokenizer_version )
chunk_fp(row_text)        = sha1( upstream + text_fields + template_version )
chunk_fp(thread_aggregate)= sha1( upstream + group_by + agg_template_version )
chunk_fp(vlm_chunk)       = sha1( cache_fp(vlm_text) )
chunk_fp(summary)         = sha1( cache_fp + summary_model + summary_prompt + summary_provider )
chunk_fp(schema_summary)  = sha1( upstream_schema + schema_summary_model + schema_summary_prompt )
chunk_fp(directory_summary)= sha1( child_object_uris + dir_summary_model + dir_summary_prompt )

embedding_fp              = sha1( chunk_fp + embedding_model + embedding_model_version )
```

每种 artifact 自己声明依赖的 input set，framework 自动算 fp。改任何一个 input → 对应 artifact 自动 stale。

### 5.2 Milvus 上的失效行为

**Milvus 不支持只更新一列**。任何 fingerprint 变化最终都走 **DELETE 该行 + INSERT 新行**。区别只在"上游有多少步可复用"：

| 变化 | upstream read | converter | chunker | embedder | summary/vlm | Milvus 操作 |
|---|---|---|---|---|---|---|
| upstream 变 | ✓ | ✓ | ✓ | ✓ | ✓ | DELETE + INSERT |
| converter 升级（pymupdf） | 跳过 | ✓ | ✓ | ✓ | 受影响才重算 | DELETE + INSERT |
| chunker config 变 | 跳过 | 跳过 | ✓ | ✓ | 跳过 | DELETE + INSERT |
| summary model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓ (仅 summary chunks) | DELETE + INSERT (仅 summary 行) |
| vlm model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓ (仅 vlm chunks) | DELETE + INSERT (仅 vlm 行) |
| embedding model 变 | 跳过 | 跳过 | 跳过 | ✓ | 跳过 | DELETE + INSERT |

实现上用 batch DELETE-by-filter + batch INSERT，比逐条 upsert 快得多。

Milvus 行同时存 `chunk_fingerprint` 和 `embedding_fingerprint` 两个字段；sync 时按 chunk_id 对比，相等就跳过整行。

### 5.3 每类 connector 的 fingerprint 算法

Connector plugin 必须实现 `fingerprint(path) -> str | None` 和 `change_set(since) -> ChangeSet`。

| Connector | 粒度 | Fingerprint 算法 | 增量手段 |
|---|---|---|---|
| **file** | path | `size + mtime_ns` 快速判断；不等再算 `sha1(content)` | scan + manifest diff |
| **web** | page | HTTP `ETag` 或 `Last-Modified`，否则 `sha1(html)` | recrawl 按 `revisit_interval` |
| **github code** | branch | `commit_sha` | `compare` API |
| | blob | `blob_sha` | tree API |
| **github issues/pulls** | record | `updated_at` | `issues?since=$cursor&state=all` |
| **gdrive** | file | `revision_id` | `changes.list?pageToken=$cursor` |
| **feishu docs** | file | `version` | OpenAPI 增量 events |
| **s3 / r2 / gcs** | object | `etag`；版本桶用 `version_id` | `ListObjectsV2?StartAfter=$cursor` |
| **slack** | (channel, day) | 当天最后一条 message 的 `ts` | `conversations.history?oldest=$cursor` |
| | thread | 最后一条 reply 的 `ts` | `conversations.replies` |
| **discord** | (channel, day) | 最后 message id（snowflake 含时间） | `messages?after=$id` |
| **gmail** | mailbox | `historyId` | `users.history.list?startHistoryId=$cursor` |
| | thread | `thread.historyId` | `users.threads.get` |
| **postgres / mysql** | table-level | `(table_oid, relpages, n_live_tup, last_analyze)` | 探测 schema/规模变化 |
| | row-level | `(pk, updated_at)` | `WHERE updated_at > $cursor`；无字段时用 CDC 或 snapshot |
| **mongodb** | document | `(_id, version)` 或 `updatedAt` | change streams 或 query |
| **bigquery / snowflake** | partition | partition meta + row_count | `_PARTITIONTIME > $cursor` |
| **linear / jira / notion** | record | `updatedAt` | API + 时间过滤 |
| **zendesk** | record | `generated_timestamp` | incremental export |
| **salesforce / hubspot** | record | `SystemModstamp` / `lastmodifieddate` | bulk API delta |
| **ssh / generic remote fs** | path | `size + mtime`；可选 sha1 | rsync-style scan |

### 5.4 ChangeSet 接口

```python
@dataclass
class ChangeSet:
    added:     list[str]      # 新出现
    modified:  list[str]      # fingerprint 变化
    deleted:   list[str]      # 消失
    unchanged: list[str]      # 同 fingerprint，跳过
    new_cursor: Cursor | None # 增量提交点；None = 该 connector 不用 cursor，每次全量 scan
```

cursor 只在 job 成功后提交；中途失败下次从旧 cursor 重试。

`new_cursor` 的两种语义：

- **有 cursor**（append-only / 时间递增数据流）：cursor 是 watermark，下次从这点续。`since` 参数对应这个 cursor。例：slack ts、postgres updated_at、gmail historyId、binlog 位置。
- **None**（无 cursor / 随机变化数据）：connector 用 **scan + diff** 策略，不持久化 cursor，每次全量列举 + 跟上次 manifest 对比。例：file source、S3 list、DB schema 探测、gdrive 文件列表 fallback。

### 5.5 每类 connector 的 ChangeSet 具体实现

| Connector | 实现 | new_cursor |
|---|---|---|
| **file** | (1) os.walk 得 current paths；(2) 对每个 path 比 manifest 的 (size, mtime_ns)；(3) 不同则算 sha1；(4) manifest 有但 current 没有 → deleted；(5) 更新 manifest | `None`（scan + diff） |
| **web** | 按 sitemap / `revisit_interval` 列 page URL；对每页 ETag / Last-Modified 检查；改了 → recrawl + 转 md | `None` |
| **github code** | `compare $last_commit...HEAD` 取变化的 blob；blob_sha 不同 → modified | `commit_sha` |
| **github issues/pulls** | `issues?since=$cursor&state=all`；按 updated_at 排；deleted 检测靠 cleanup pass | `max(updated_at)` |
| **gdrive** | `changes.list?pageToken=$cursor` 取所有变更（含 delete） | `next_page_token` |
| **feishu docs** | OpenAPI 增量 events | `event_offset` |
| **s3** | `ListObjectsV2 StartAfter=$cursor`；etag 比对；deleted 检测周期全量 list 一次 | `last_key` 或 `None` |
| **slack** | (1) 列 channel × day partition；(2) per day: `conversations.history?oldest=$ts`；(3) 新增 message → modified（per-day 重 chunk）；(4) thread reply 同理 | `max(ts) per channel` |
| **discord** | `channels/{id}/messages?after=$id`（snowflake 含时间） | `last_message_id` |
| **gmail** | `users.history.list?startHistoryId=$cursor` 取增量 thread 变化 | `historyId` |
| **postgres rows** | (1) `SELECT pk,updated_at WHERE updated_at>$cursor LIMIT N`；(2) 写 modified；(3) 周期全量 `SELECT pk` 检测 deleted | `max(updated_at)` |
| **postgres schema** | 探测 `pg_attribute` hash 变化；schema 变 → 重生成 schema.json | `None` |
| **mongodb** | change streams（首选）或 `_id+version` 周期对比 | `resume_token` 或 `None` |
| **bigquery / snowflake** | `WHERE _PARTITIONTIME > $cursor`；partition 粒度 | `max(_PARTITIONTIME)` |
| **linear / jira / notion** | API + `updatedAfter=$cursor` | `max(updatedAt)` |
| **zendesk** | incremental export API（`tickets?start_time=$cursor`） | `end_time` |
| **salesforce / hubspot** | bulk API delta（按 SystemModstamp / lastmodifieddate） | `max(timestamp)` |

### 5.6 file source 的 scan + manifest diff 详解

```
1. scan：os.walk(root) 应用 ignore rules（.gitignore + .mfsignore + 默认 binary 规则）
       → 得当前 paths 集合 current_paths

2. 对每个 path 与 manifest 表对比（in metadata DB）：
   - manifest 里有 + (size, mtime_ns) 完全一致 → unchanged，跳过
   - manifest 里有 + (size, mtime_ns) 变化 → 算 sha1(content)
     - sha1 跟 manifest 一致 → unchanged（只是 touch 了 mtime）→ 更新 manifest.mtime_ns
     - sha1 不一致 → modified → 入 worker queue
   - manifest 里没有 → added → 入 worker queue

3. manifest 里有但 current_paths 没有 → deleted → 入 worker queue（删 chunks）

4. 更新 manifest（写新 sha1 / size / mtime / last_seen）

5. 输出 ChangeSet（new_cursor = None）
```

manifest 记录：

```text
path
size
mtime_ns
sha1
last_seen
```

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

字段含义详见 [05-search-and-retrieval.md §4](05-search-and-retrieval.md#4-字段配置).

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

framework 全局配置（chunk size、embedding model 等）放 `~/.mfs/config.toml`，详见 [05-search-and-retrieval.md §10](05-search-and-retrieval.md#10-embedding--summary-providers).

**两层覆盖**：framework 全局 + connector/object 配置。`mfs config show --effective <uri>` 打印某路径的最终生效配置。

## 7. 凭据管理

connector TOML 里**不写明文凭据**，只写 `credential_ref`：

```toml
credential_ref = "secret:pg-prod-readonly"          # OS keychain
credential_ref = "env:PG_PROD_DSN"                  # 环境变量
credential_ref = "file:~/.mfs/secrets/pg-prod.toml" # 文件
credential_ref = "vault:secret/data/mfs/pg-prod"    # Vault (remote)
```

解析优先级和具体 schema 见 [06-architecture.md §7](06-architecture.md#7-凭据与认证).

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
    "efficient_tail": false,
    "paged_cat": true,
    "permission_snapshot": false
  },
  "credentials": {
    "required": true,
    "schema": "PostgresCredential"
  }
}
```

framework 根据这些字段派发：`grep_pushdown=true` 时 `mfs grep` 走 SQL ILIKE，否则走线性扫；`efficient_tail=true` 才允许 `tail -f`。

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

## 11. 错误恢复

- ingest 失败 → 写 `jobs` 表 status=failed + error，cursor 不前进。
- 用户重跑 `mfs add <uri>` → 从旧 cursor 续。
- daemon 崩溃恢复 → 启动时 scan jobs 表，将 running 的标 stale，按 cursor 重新调度。
- chunk 写 Milvus 失败 → 单条重试 N 次，超限标 failed，继续其他 chunk。
- 部分 chunk 失败的对象在 `mfs status --verbose <uri>` 里可见，下次 sync 重试。

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
uv tool install --upgrade mfs-cli
uv tool install mfs-server
mfs serve start
mfs profile add local --url http://127.0.0.1:8765 --kind local
mfs profile use local
mfs add . --force         # 重建本地索引（schema 升级）
```
