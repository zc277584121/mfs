# Connector 与 Ingest

这一篇讲 connector 怎么注册、ingest 流程长什么样、变化检测怎么做、错误怎么恢复。看完知道 `mfs add` 背后实际跑了什么。

每类 connector 暴露的虚拟目录布局是单独参考清单，见 [09-connector-catalog.md](09-connector-catalog.md)。怎么写一个新 connector 的 plugin 接口在 [07-contributing-connector.md](07-contributing-connector.md)。

## 1. Connector 注册

`mfs add` 是注册 + 同步的统一入口，幂等：再跑一次 = 再同步一次。本地路径不需要 config TOML；外部 connector 首次需要 `--config`。

```bash
# 本地路径
mfs add ./repo
mfs add .

# 外部 connector（首次：含估算 + confirm）
mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml
mfs add slack://eng --config .mfs/connectors/slack-eng.toml
mfs add web://acme-docs --config .mfs/connectors/acme-docs.toml

# 跳过 confirm
mfs add postgres://prod --config x.toml --yes
```

想先验证凭据和连通性、不写状态，用 `mfs connector probe`：

```bash
mfs connector probe postgres://prod --config x.toml
```

注册后日常使用通过 connector URI：

```bash
mfs add postgres://prod                    # 再同步
mfs add postgres://prod --force-index      # 强制重 chunk + embed
mfs ls postgres://prod/public/tickets      # 浏览
mfs connector list
```

connector root URI 由 `scheme://<alias>` 组成。alias 是用户起的名（`prod` / `eng` / `acme-docs`），在当前 namespace 内唯一，会进入脚本和搜索结果；展示名放 `label`。本地路径的 connector URI 内部表示为 `file://<client_id>/<abs-path>`，但用户日常写普通 path 即可。

## 2. Ingest 流程

`mfs add` 触发的工作链：

```
mfs add <target>
  │
  ① 路径解析
       本地 path → file connector
       URI → 对应 connector plugin
  │
  ② 注册或拉取 connector 配置
       首次：parse TOML + validate + 写 connectors 表
       已注册：复用现有配置
  │
  ③ connector.sync() 流式 yield ObjectChange
       added / modified / deleted
  │
  ④ 对每个 ObjectChange，跑 reconcile 4 步比对（见 §5.2.3）
       fp 不等的层入队，相等的早退
  │
  ⑤ Sweep pass：扫未在 yield 集合里的 object
       捕获 framework 配置变化（换 embedding 模型等）
  │
  ⑥ Worker 跑 build task
       cache → chunk → embed → 写 Milvus
       chunker 按 object_kind 分派：
         document → markdown chunker
         code → AST chunker
         table_rows → row text chunker
         message_stream → thread aggregator
         record_collection → per-record chunker
         image → VLM description
  │
  ⑦ 删除消失的对象
       Milvus DELETE WHERE connector_uri = X AND object_uri NOT IN (current_set)
  │
  ⑧ commit connector state + 更新 job 状态
```

执行位置：

| 部署 | queue 位置 | worker |
|---|---|---|
| 本机 server | server 内 SQLite queue | server 内 worker pool（建议 concurrency=1） |
| 远端 server | Postgres queue（同 metadata DB） | `mfs-worker` 进程 |

HTTP 主要走 control plane，唯一例外是 remote profile 下本地文件 upload（详见 [02 §4](02-architecture.md#4-控制面-vs-数据面)）。

## 3. 首次注册外部 connector 的默认行为

```text
$ mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml
Connector validated: postgres://prod
Discovered: 38 tables / ~12.4M rows
Estimated work (based on 1% probe sample, ±50% accuracy):
  chunks:    ~14M
  tokens:    ~2.4M (use your provider's rate to compute cost)
  storage:   ~3.2GB index + cache

Continue? [y/N]
```

估算流程：

1. 探测 connector 暴露的对象总数和 size_hint（不读对象内容）
2. 抽样 1% 对象跑完整 chunk + embed（真实测一段）
3. 按抽样外推总 chunks / tokens / storage，明示 ±50% 精度

**只给物理量，不给钱和时间**——钱因 embedding provider 而异（OpenAI / Voyage / Cohere / 自部署 / 企业协议价都不同），时间受并发 / rate limit / 网络浮动 10x，硬给反而误导。token 数靠抽样 tokenizer 算出来，是个可靠的"工作量"指标，用户拿着自己 provider 的 rate 算钱。

`--yes` 或本地路径直接开始：

```text
$ mfs add ./repo
Processing 184 files under /repo
Indexed: 184 files scanned, 37 touched, 2 deleted, 412 chunks queued.
Worker running in background. Run `mfs status` to check progress.
```

## 4. Connector TOML 配置

```toml
# 顶层：connector 元信息
[connector]
type = "postgres"                       # 必填，决定走哪个 plugin
root = "postgres://prod"                # 必填
label = "Production Postgres"
credential_ref = "secret:pg-prod-readonly"

# connector 类型特定配置
[postgres]
schemas = ["public"]
max_read_rows = 10000
max_read_bytes = "10MiB"

# 对象级配置（array-of-tables；按顺序匹配，先匹配优先）
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
chunk_max = 100000
```

字段含义详见 [06 §4](06-search-and-retrieval.md#4-字段配置)。

Web connector 配置例子：

```toml
[connector]
type = "web"
root = "web://acme-docs"
label = "Acme Docs"

[web]
start_urls = ["https://docs.acme.com/"]
allowed_domains = ["docs.acme.com"]
sitemap = "https://docs.acme.com/sitemap.xml"
max_pages = 1000
crawl_depth = 3
respect_robots_txt = true
revisit_interval = "7d"

[[objects]]
match = "pages/**"
chunk_strategy = "per_field_chunked"
```

framework 全局配置（chunk size、embedding model 等）放 server 端 `server.toml`，详见 [06 §10](06-search-and-retrieval.md#10-embedding--summary-providers)。两层覆盖：framework 全局 + connector / object 配置。`mfs config show --effective <uri>` 打印某路径的最终生效配置。

## 5. 变化检测

`mfs add <uri>` 再跑时，怎么知道哪些 object 要重做、哪些可以跳过？这一节讲完整机制。

变化检测分两个层面：

- **upstream 层**——外部数据本身变了。由 connector 自己探测，最了解外部系统。
- **framework 层**——chunker / embedder / converter 等内部配置变了，外部没动但下游产物失效。由 framework 主动扫描。

### 5.1 Connector 契约：两条最小 API

```python
class ConnectorPlugin:
    async def sync(self) -> AsyncIterator[ObjectChange]:
        """流式 yield 每个变化的 object。cursor/manifest/etag/hash 怎么算、
        存哪、schema 长什么样，全在 connector 内部，framework 不 introspect。"""

    async def fingerprint(self, path: str) -> str | None:
        """返回 path 的当前 upstream fingerprint。framework 用这个跟自己
        存的对比，决定 cache / chunk / embedding 失效。"""


@dataclass
class ObjectChange:
    uri:  str
    kind: Literal["added", "modified", "deleted"]
```

Connector 内部 state（cursor / manifest / etag 表）用 framework 提供的 KV store 持久化：

```python
async def sync(self):
    last = await self.state.get("last_ts")
    rows = await self.api.fetch(since=last)
    for r in rows:
        yield ObjectChange(r.uri, "modified" if r.was_seen_before else "added")
    await self.state.set("last_ts", new_ts)
```

framework 不看 `self.state` 里存的是什么——postgres 存 `updated_at`、slack 存 ts、s3 存 page token、file 存 manifest map、github 存 `commit_sha`，schema 各不相同。

### 5.2 Fingerprint chain

framework 拿到 upstream fingerprint 后，自己组合 chunker / embedding model 等版本信息，算出每层产物的 fingerprint，决定哪层失效。

#### 产物模型：per-object_kind DAG

每个 object_kind（document / image / table_rows / message_stream / ...）在 `processors/<kind>/` 里 declare 自己的产物列表（`ArtifactSpec`）：

```python
@dataclass
class ArtifactSpec:
    kind:          str           # "cache.converted_md" / "chunk.body" / "embedding"
    depends_on:    list[str]     # 上游产物 kind（"upstream" 是特殊源）
    config_inputs: list[str]     # 影响 fp 的 framework 配置 key
    storage:       Literal["cache_table", "milvus_field", "milvus_row"]
```

framework 把这些声明组装成 DAG，每条边代表"我依赖什么"。各 object_kind 的链路：

```
document (md/code/text)
    upstream → chunk.body → embedding
                ↑ [chunker_name, chunker_config, tokenizer_version]
                              ↑ [embedding_model, embedding_version]

document (pdf/docx/gdoc)
    upstream → cache.converted_md → chunk.body → embedding
                ↑ [converter_name, converter_version]

image
    upstream → cache.vlm_text → chunk.vlm_description → embedding
                ↑ [vlm_model, vlm_prompt, vlm_provider]

table_rows (postgres rows.jsonl)
    upstream(per row) → chunk.row_text(per row) → embedding
                         ↑ [text_fields, template_version]

message_stream (slack messages.jsonl)
    upstream(per channel-day) → chunk.thread_aggregate(per thread) → embedding
                                 ↑ [group_by, session_idle_min, agg_template_version]

table_schema (postgres schema.json)
    upstream → chunk.schema_summary → embedding
                ↑ [schema_summary_model, schema_summary_prompt]

directory
    child_object_uris → chunk.directory_summary → embedding
                         ↑ [dir_summary_model, dir_summary_prompt]
```

不同 object_kind 步数不同：纯 markdown 没有 converter 层，图片没有 chunker 层（VLM 直接出 1 个 chunk）。

#### fp 计算

framework 集中实现一个通用函数：

```python
def compute_artifact_fp(spec: ArtifactSpec, upstream_fps: dict, config: dict) -> str:
    parts = [upstream_fps[k] for k in spec.depends_on]
    parts += [str(config[k]) for k in spec.config_inputs]
    return sha1("\x00".join(parts).encode()).hexdigest()
```

各层公式（按 storage 字段对应到存哪里）：

```
upstream_fp                = connector.fingerprint(path)          ← objects.fingerprint

cache_fp(converted_md)     = sha1( upstream + converter_name + converter_version )
cache_fp(vlm_text)         = sha1( upstream + vlm_model + vlm_prompt + vlm_provider )
cache_fp(page_cache)       = sha1( upstream )
cache_fp(head_cache)       = sha1( upstream + N )
                                                                  ← caches.fingerprint

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

Connector 不参与这套逻辑，只提供 upstream fingerprint。下游全由 framework 算。

公式里的 `chunker_config` / `vlm_prompt` 等是**序列化字符串**（JSON / TOML），把所有相关参数揉进去——chunk_size 改 1500→1000，序列化字符串就变，hash 就变。换 provider / model 同理：`embedding_model` 含 provider/model/version 三段。

注意：**配置字符串只进 fp 不进 chunk_id**。chunk_id 是主键，目标是"同一条 record 重 sync 时 UPSERT 覆盖同一行"，所以只用 namespace/connector/object/locator/chunk_kind（这些跟 config 无关）。chunk_fp 才是判断"内容是不是 stale"的指纹，含所有 config。否则换 config 后旧行就成"幽灵行"无法 UPSERT 覆盖。

#### Reconcile 4 步

每次 sync_job 对每个 object 跑这套比对（per-object，early-exit）：

```
Step 1: upstream 层比对
  new_fp = connector.fingerprint(path)
  old_fp = objects.fingerprint
  if new != old: enqueue full rebuild → return

Step 2: cache 层比对（仅对适用的 cache_kind）
  for cache_kind in processor.applicable_caches(object_kind):
    new_fp = compute_artifact_fp(...)
    if new != caches.fingerprint:
      enqueue cache_rebuild → 触发下游 chunk + embed
      continue

Step 3: chunk 层比对
  for chunk_row in milvus.query(object_uri=Y):
    if new_fp != chunk_row.chunk_fingerprint:
      enqueue chunk_rebuild → 触发下游 embed
      continue

Step 4: embedding 层比对
    if new_fp != chunk_row.embedding_fingerprint:
      enqueue embed_only（chunk 文本不动，只重 embed）
```

Early-exit：上游 fp 不等就整套重做，跳过下游检查（下游一定也 stale）。

#### Sweep pass：捕获 framework 配置变化

`connector.sync()` 只汇报 upstream 变化。换 embedding 模型 / 升级 chunker / 升级 converter 时，connector 看 upstream 完全没变，啥都不 yield。这时 framework 在 sync 末尾跑一遍 sweep：

```
for object in 当前 connector 下未在本次 yield 的 object:
  跑 Reconcile 4 步比对
```

这一步让"换 embedding 模型 → 跑 `mfs add` → 自动重 embed" 能 work，用户不需要加 `--force-index`。性能：sweep 是只读比对（不读 upstream），只查 metadata DB + Milvus 字段，100 万 chunk 几秒钟级。

### 5.3 Milvus 上的失效行为

Milvus 不支持只更新一列。任何 fingerprint 变化最终都是 **DELETE 该行 + INSERT 新行**。区别只在"上游有多少步可复用"：

| 变化 | upstream read | converter | chunker | embedder | summary/vlm | Milvus 操作 |
|---|---|---|---|---|---|---|
| upstream 变 | ✓ | ✓ | ✓ | ✓ | ✓ | DELETE + INSERT |
| converter 升级 | 跳过 | ✓ | ✓ | ✓ | 受影响才重算 | DELETE + INSERT |
| chunker config 变 | 跳过 | 跳过 | ✓ | ✓ | 跳过 | DELETE + INSERT |
| summary model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓（仅 summary） | DELETE + INSERT（仅 summary 行） |
| vlm model 变 | 跳过 | 跳过 | 跳过 | 跳过 | ✓（仅 vlm） | DELETE + INSERT（仅 vlm 行） |
| embedding model 变 | 跳过 | 跳过 | 跳过 | ✓ | 跳过 | DELETE + INSERT |

批量 DELETE-by-filter + 批量 INSERT 比逐条 upsert 快得多。

### 5.4 Connector 实现策略参考

下面两张表是各类 connector 常见实现策略，**仅供贡献者参考**——framework 不规定怎么算变化、用什么 cursor、存什么 state。

**Fingerprint 算法**：

| Connector | 粒度 | 算法 |
|---|---|---|
| file | path | `size + mtime_ns` 快速判断；不等再算 `sha1(content)` |
| web | page | HTTP `ETag` 或 `Last-Modified`，否则 `sha1(html)` |
| github code | blob | `blob_sha` |
| github issues / pulls | record | `updated_at` |
| gdrive | file | `revision_id` |
| feishu docs | file | `version` |
| s3 / r2 / gcs | object | `etag`；版本桶用 `version_id` |
| slack | (channel, day) | 当天最后一条 message 的 `ts` |
| discord | (channel, day) | 最后 message id（snowflake 含时间） |
| gmail | thread | `thread.historyId` |
| postgres / mysql | row | `(pk, updated_at)` |
| mongodb | document | `(_id, version)` 或 `updatedAt` |
| bigquery / snowflake | partition | partition meta + row_count |
| linear / jira / notion | record | `updatedAt` |
| zendesk | record | `generated_timestamp` |
| salesforce / hubspot | record | `SystemModstamp` / `lastmodifieddate` |
| ssh / generic remote fs | path | `size + mtime`；可选 sha1 |

**同步策略**：

| Connector | 增量手段 | state 里存什么 |
|---|---|---|
| file | scan + manifest diff | `{ path: (size, mtime_ns, sha1) }` |
| web | revisit_interval 触发 recrawl + ETag | `{ url: etag }` |
| github code | `compare $commit...HEAD` | `commit_sha` |
| github issues / pulls | `issues?since=$cursor&state=all` | `max(updated_at)` |
| gdrive | `changes.list?pageToken=$cursor` | `next_page_token` |
| feishu docs | OpenAPI 增量 events | `event_offset` |
| s3 | `ListObjectsV2 StartAfter=$cursor` + 周期全量 list 检 delete | `last_key` |
| slack | per channel × day: `conversations.history?oldest=$ts` | `{ channel: max(ts) }` |
| discord | `messages?after=$id` | `{ channel: last_msg_id }` |
| gmail | `users.history.list?startHistoryId=$cursor` | `historyId` |
| postgres rows | `SELECT pk,updated_at WHERE updated_at>$cursor` + 周期全量 pk diff | `max(updated_at)` |
| postgres schema | 探测 `pg_attribute` hash 变化 | hash of pg_attribute snapshot |
| mongodb | change streams（首选）或 `_id+version` 周期对比 | `resume_token` |
| bigquery / snowflake | `WHERE _PARTITIONTIME > $cursor` | `max(_PARTITIONTIME)` |
| linear / jira / notion | API + `updatedAfter=$cursor` | `max(updatedAt)` |
| zendesk | incremental export `tickets?start_time=$cursor` | `end_time` |
| salesforce / hubspot | bulk API delta | `max(SystemModstamp)` |

### 5.5 file connector 实现示例

file connector 的 sync 完整流程：

```
1. scan：os.walk(root) 应用 ignore rules（.gitignore + .mfsignore + 默认 binary 规则）
   得到当前 paths 集合 current_paths

2. 对每个 path 跟 self.state 里的 manifest 对比：
   - manifest 里有 + (size, mtime_ns) 完全一致 → 跳过
   - manifest 里有 + (size, mtime_ns) 变化 → 算 sha1(content)
     - sha1 跟 manifest 一致 → 只是 touch 了 mtime，更新 manifest.mtime_ns，跳过
     - sha1 不一致 → yield ObjectChange(path, "modified")
   - manifest 里没有 → yield ObjectChange(path, "added")

3. manifest 里有但 current_paths 没有 → yield ObjectChange(path, "deleted")

4. 更新 self.state 里的 manifest（写新 sha1 / size / mtime）
```

manifest 是 file connector 自己定义的结构，存在 `self.state` 里。其他 connector 按各自需要存自己的 state，framework 不 introspect。

**file connector 不调 checkpoint**——它的 state 是全量 map 形态，半截 commit 不合法（详见 §5.6.1）。

注意：file connector 的 manifest 只包含**文件本身的 fingerprint**（path / size / mtime_ns / sha1），**不包含** chunker / embedding model / converter 版本。framework 层的配置变化由 §5.2 的 sweep 步骤捕获，跟 manifest diff 无关。

### 5.6 Mid-job checkpoint

[02 §7 规则 ③](02-architecture.md#7-一致性) 的默认行为是"state 末尾提交"：connector 中途崩 → state 不 commit → 下次重头跑。file / github code 这种重跑 cheap 的 connector 没问题；Slack / Gmail / Salesforce 这种被 rate limit 的 connector 重头跑会被打爆。

framework 提供**可选**的 `self.state.checkpoint()` 让 connector 在 sync 中途显式 commit state：

```python
class StateStore(Protocol):
    async def get(self, key: str) -> Any | None: ...
    async def set(self, key: str, value: Any) -> None: ...
    async def delete(self, key: str) -> None: ...

    async def checkpoint(self) -> None:
        """把当前 state_snapshot commit 到 connector_state 表。
        只适合 cursor 推进型 state（见 §5.6.1）。"""
```

framework 拿到 checkpoint 调用 → 一个事务把 `connector_jobs.state_snapshot` 行 copy 到 `connector_state` 表 + 重置 snapshot → 之后即使本 job 失败，下次也从这里接续。

#### 5.6.1 哪些 state 能调 checkpoint

| state 形态 | 例 | 能调 | 理由 |
|---|---|---|---|
| 单调推进 cursor | postgres `max(updated_at)` / slack `{ch: max(ts)}` / gmail `historyId` / linear `updatedAt` / s3 `last_key` / mongodb `resume_token` | ✅ 推荐 | cursor 单调，半截 commit 后下次从该 cursor 接续，不丢数据 |
| paged token | gdrive `next_page_token` | 看 provider | gdrive token 长期有效 → OK；某些 provider token 短期失效 → 不推荐 |
| commit hash 类（A→B） | github code `commit_sha` / git tag / bigquery snapshot_time | ❌ | 一次 sync 是"从 A 跳到 B"的原子转换，没有合法的"半 commit" |
| 全量 map 快照 | file `{path: hash}` / web crawl `{url: etag}` | ❌ | 全量 map 半截不能宣称是某一时刻的真相 |

判断准则：你的 state 在 `checkpoint()` 那一瞬间，是不是一个**合法的"从此处接续"的起点**？是 → 可以调；不是 → 别调。

#### 5.6.2 推荐用法

```python
async def sync(self):
    last = await self.state.get("last_ts")
    page_index = 0
    async for page in self.api.fetch_paginated(since=last):
        for record in page:
            yield ObjectChange(record.uri, "modified")
        await self.state.set("last_ts", page.max_ts)
        page_index += 1
        if page_index % 50 == 0:           # 每 50 页 checkpoint
            await self.state.checkpoint()
```

频率取决于"重跑 50 页贵不贵 + checkpoint 自己的事务成本"。10 万量级数据 50~200 页一次是合理区间。

不调 checkpoint 的 connector 走默认末尾提交语义，靠 chunk_id 幂等保证重跑不会写脏 Milvus。

## 6. 凭据管理

connector TOML 不写明文，只写 `credential_ref`。v0.4 只支持环境变量：

```toml
credential_ref = "env:PG_PROD_DSN"
```

其他 scheme（OS keychain / 文件 / Vault）是 v0.5+ 的路线图，详见 [02 §11](02-architecture.md#11-凭据)。

## 7. Watch（本地路径专用）

```bash
mfs add ./repo --watch
mfs add ./repo --watch --interval 60s
```

- daemon 内启 watcher（`watchfiles` 或 OS-native）
- watch 事件只作触发信号，最终事实仍来自 scan + manifest 对比
- 查看正在 watch：`mfs status --watch`
- 停止：`mfs remove ./repo` 或 Ctrl+C
- 外部 connector 不支持 watch，用 scheduler 周期触发

首次 watch 某目录时弹权限确认：

```text
MFS local daemon will watch this directory:
  /repo

It will read file names, mtimes, sizes, and indexable file contents.
State is stored under ~/.mfs/.

Continue? [y/N]
```

## 8. Connector 能力声明

每个 connector plugin 声明能力，agent 通过 `mfs connector inspect <root>` 看到：

```json
{
  "connector_type": "postgres",
  "uri_scheme": "postgres",
  "sync": {
    "manual": true, "scheduled": true, "watch": false,
    "cursor": "updated_at", "full_scan": true, "delete_detection": true
  },
  "object": {
    "grep_pushdown": true, "search_pushdown": false,
    "paged_cat": true, "acl": false
  },
  "credentials": {
    "required": true, "schema": "PostgresCredential"
  }
}
```

framework 根据这些字段派发：`grep_pushdown=true` 时 `mfs grep` 走 SQL ILIKE，否则走线性扫；`paged_cat=true` 才允许 `cat --range`。

## 9. Sync 策略矩阵

| 策略 | 适合 connector | 触发方式 |
|---|---|---|
| 手动 | 所有 | `mfs add <uri>` |
| watch | 本地目录 | `mfs add . --watch` |
| 定时 | slack / github / SaaS / DB / web | daemon scheduler 或外部 cron |
| 游标 | slack / gmail / 部分 SaaS | provider cursor 或 updated_at |
| snapshot 对比 | s3 / gdrive / DB fallback | 周期列举对比 |
| append-only | logs / chat / events | 追加尾部 + 定期校准 |

daemon 内置简单 scheduler（基于 SQLite + APScheduler 风格），用户可在 connector TOML 写 `schedule = "*/15 * * * *"` cron 表达式。

## 10. 错误恢复与重跑

整套 sync 正确性靠 [02 §7](02-architecture.md#7-一致性) 的四条一致性规则保证，这一节只补"用户视角"。

### 10.1 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force-index` | 所有 object 视为 modified，跳过 fp 比对，强制重 chunk + embed |
| `mfs add ./path --force-upload` | 仅 upload flow：client 忽略 manifest cache 全量重传 + server 强制重 index |
| `mfs add <uri>` 在前次失败后 | 前次 state 未 commit，从上一个成功的 state 重跑——失败的 object 自然再次出现 |
| `mfs add <uri>` 在前次还 running | 拒绝 `sync_already_running, see job <id>` |
| `mfs remove <uri>` 在前次 sync running | preempt：sync 标 cancelling，当前 task 完成后退出，remove 接管 |
| `mfs add <uri>` 在 `status='removing'` 时 | 拒绝 `connector_removing`，等清理完才能重新注册 |

不提供 `mfs job retry`——重跑 = 下次 `mfs add`，state 没 commit 时自然接续。并发协调的完整语义表见 [02 §8](02-architecture.md#8-并发协调)。

### 10.2 单 object / 单 chunk 失败

- worker 处理单 object 时，可恢复错误（429 / timeout）自动 retry N 次（指数退避）
- 超限 → object_task 标 failed，**整 sync_job 继续跑其他 task**（不因单个对象失败放弃全部）
- 失败 task 在 `mfs status --verbose <uri>` 和 `mfs job inspect <id>` 可见
- 下次 `mfs add` 重新出 ObjectChange → 重新创建 task → 再试一次

## 11. 从当前代码版本迁移

当前代码（0.3.x）跟 v0.4 的映射：

| 当前 | v0.4 |
|---|---|
| `mfs add .` 本地索引（同进程） | `mfs add .` 通过 HTTP 调 daemon |
| `~/.mfs/milvus.db` + `~/.mfs/state.db` | 同位置，schema 升级（需 `mfs add . --force-index` 重建） |
| `mfs status` | 同名，语义扩展 |
| `mfs add --watch` | 同 |
| `mfs add --force` | 改名 `--force-index`；remote profile 下另有 `--force-upload` |
| Milvus chunk schema 字段 `account_id` | 改名 `namespace_id`，Milvus 不支持字段重命名，必须 drop collection 重建 |
| Milvus chunk_id 公式 `sha1(source + start + end + content_hash + model)` | 改 `sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)`，跟上一行的 rebuild 一并完成 |
| 没有外部数据源概念 | 加 `mfs add <uri> --config X` 注册外部 connector |
| 没有本机 server | 用 `mfs serve start` 启动；`MFS_AUTOSTART=1` 时首次 `mfs add` 也会触发 |

升级动作：

```bash
brew install mfs              # CLI（Rust binary）
uv tool install mfs-server    # server（Python）
mfs serve start
mfs profile add local --url http://127.0.0.1:8765
mfs profile use local
mfs add . --force-index       # 重建本地索引（schema 升级；底下会 drop 旧 collection）
```

> v0.3 → v0.4 升级会 drop Milvus collection（字段重命名 + chunk_id 公式变更），所有索引需要从 upstream 重 build。这一步会写进 release note。本机用户 `mfs add . --force-index` 重跑即可；CS 部署用户参考运维迁移文档。
