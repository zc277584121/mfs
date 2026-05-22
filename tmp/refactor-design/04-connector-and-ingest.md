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
  ④ 对每个 ObjectChange（yield 出来的）跑 reconcile 4 步比对（见 §5.2.3）
       fp 不等的层入队，相等的早退
       （框架配置变化 = 换 embedding 模型/chunker → v0.4 用户手动 --force-index，
         不在这里自动检测；详见 §5.2 末尾）
  │
  ⑤ Worker 跑 build task
       artifact → chunk → embed → 写 Milvus
       chunker 按 object_kind 分派：
         document → markdown chunker
         code → AST chunker
         table_rows → row text chunker
         message_stream → thread aggregator
         record_collection → per-record chunker
         image → VLM description
       embed/vlm/summary 都走 CachingXxxClient → transformation cache (02 §10.4)
       命中 → 零 API；miss → 真 API + 异步写回 cache
  │
  ⑥ Deletion reconcile（详见 [02 §7.4](02-architecture.md#74-deletion-策略)）
       incremental sync → 跳过（cursor 只 yield 变化，推不出删除）
       full scan sync   → 全集 diff，to_delete = (objects ∩ Milvus) - 本次全集，DELETE
       explicit "deleted" event → 任何模式都直接删
       （枚举不完整时 connector 必须 raise → sync 失败 → 不删，详见 §7.4.3 契约）
  │
  ⑦ commit connector state + 更新 job 状态
```

执行位置：

| 部署 | queue 位置 | worker |
|---|---|---|
| 本机 server | server 内 SQLite queue | server 内 worker pool（默认自适应：SQLite 强制 concurrency=1） |
| 远端 server | Postgres queue（同 metadata DB） | `mfs-worker` 进程（默认自适应：Postgres 默认 concurrency=4） |

HTTP 主要走 control plane，唯一例外是 remote profile 下本地文件 upload（详见 [02 §4](02-architecture.md#4-控制面-vs-数据面)）。

## 3. 首次注册外部 connector 的默认行为

```text
$ mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml
Connector validated: postgres://prod
Discovered: 38 tables / ~12.4M rows
Estimated (local chunker + tokenizer only — no embedding API calls):
  chunks:    ~14M    (chunker dry-run on a sample of up to 1000 records)
  tokens:    ~2.4M   (apply your provider's per-token rate to estimate $)

Continue? [y/N]
```

估算流程：

1. 探测 connector 暴露的对象总数和 size_hint（不读对象内容，只走 metadata 类 API：`SELECT count(*)` / `list objects` 等）
2. 抽样小批量 record（默认 min(1000, 1%)）跑 **chunker + 本地 tokenizer**：
   - chunker 是确定性算法、tokenizer 是本地库，零外部成本
   - 不调 embedding API，不写 Milvus
3. 按抽样外推总 chunks / tokens，明示 ±50% 精度

**估算阶段零计费**——这是个硬约束：用户敲 `mfs add` 看到 prompt 时，不能已经把钱花了。**只给物理量，不给钱和时间**——钱因 embedding provider 而异（OpenAI / Voyage / Cohere / 自部署 / 企业协议价都不同），时间受并发 / rate limit / 网络浮动 10x，硬给反而误导。token 数靠抽样 tokenizer 算出来是可靠的"工作量"指标，用户拿着自己 provider 的 rate 算钱。**storage 不估**（≈ chunks × dim × 4byte，跟选哪个 embedding model 强相关，误差比 chunks/tokens 还大）。

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
    async def sync(self, opts: SyncOptions) -> AsyncIterator[ObjectChange]:
        """流式 yield 每个变化的 object。cursor/manifest/etag/hash 怎么算、
        存哪、schema 长什么样，全在 connector 内部，framework 不 introspect。"""

    async def fingerprint(self, path: str) -> str | None:
        """返回 path 的当前 upstream fingerprint。framework 用这个跟自己
        存的对比，决定 artifact / chunk / embedding 失效。"""


@dataclass
class ObjectChange:
    uri:     str
    kind:    Literal["added", "modified", "deleted", "renamed"]
    old_uri: Optional[str] = None    # 仅 renamed：原 path / URI

@dataclass
class SyncOptions:
    full:  bool = False              # 用户 --force-index 时为 True
    since: Optional[str] = None      # 用户 --since <date>
```

`renamed` 是个可选 kind——connector 能可靠识别"同内容换 path"时主动 yield，让 framework 跳过重 chunk / 重 embed，只改 Milvus 的 chunk_id 主键。其他 connector 不强制实现，照旧 yield `added` + `deleted` 也正确（只是更贵）。详见 [§5.7](#57-rename-detection)。

`SyncOptions` 由 framework 注入。connector 用 `opts.full` 决定要不要走全量扫描而非游标增量，用 `opts.since` 覆盖 state 里的 cursor。不支持 `since` 的 connector 看到非 None 时报 `since_unsupported`。

Connector 内部 state（cursor / manifest / etag 表）用 framework 提供的 KV store 持久化：

```python
async def sync(self):
    last = await self.state.get("last_ts")
    rows = await self.api.fetch(since=last)
    for r in rows:
        yield ObjectChange(r.uri, "modified" if r.was_seen_before else "added")
    await self.state.set("last_ts", new_ts)
```

framework 不看 `self.state` 里存的是什么——postgres 存 `updated_at`、slack 存 ts、s3 存 page token、github 存 `commit_sha`，schema 各不相同。**file connector 是特例**：它的 path manifest 走专属的 `file_state` 表而不是 `self.state` K/V（详见 §5.5），但其他状态（如 `platform_id`）仍然存 `self.state`。

### 5.2 Fingerprint chain

framework 拿到 upstream fingerprint 后，自己组合 chunker / embedding model 等版本信息，算出每层产物的 fingerprint，决定哪层失效。

#### 产物模型：per-object_kind DAG

每个 object_kind（document / image / table_rows / message_stream / ...）在 `processors/<kind>/` 里 declare 自己的产物列表（`ArtifactSpec`）：

```python
@dataclass
class ArtifactSpec:
    kind:          str           # "artifact.converted_md" / "chunk.body" / "embedding"
    depends_on:    list[str]     # 上游产物 kind（"upstream" 是特殊源）
    config_inputs: list[str]     # 影响 fp 的 framework 配置 key
    storage:       Literal["artifact_cache_table", "milvus_field", "milvus_row"]
```

framework 把这些声明组装成 DAG，每条边代表"我依赖什么"。各 object_kind 的链路：

```
document (md/code/text)
    upstream → chunk.body → embedding
                ↑ [chunker_name, chunker_config, tokenizer_version]
                              ↑ [embedding_model, embedding_version]

document (pdf/docx/gdoc)
    upstream → artifact.converted_md → chunk.body → embedding
                ↑ [converter_name, converter_version]

image
    upstream → artifact.vlm_text → chunk.vlm_description → embedding
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
upstream_fp                  = connector.fingerprint(path)          ← objects.fingerprint

artifact_fp(converted_md)    = sha1( upstream + converter_name + converter_version )
artifact_fp(vlm_text)        = sha1( upstream + vlm_model + vlm_prompt + vlm_provider )
artifact_fp(page_cache)      = sha1( upstream )
artifact_fp(head_cache)      = sha1( upstream + N )
                                                                    ← artifact_cache.fingerprint

chunk_fp(body)               = sha1( artifact_fp(converted_md) + chunker_name + chunker_config + tokenizer_version )
chunk_fp(row_text)           = sha1( upstream + text_fields + template_version )
chunk_fp(thread_aggregate)   = sha1( upstream + group_by + agg_template_version )
chunk_fp(vlm_chunk)          = sha1( artifact_fp(vlm_text) )
chunk_fp(schema_summary)     = sha1( upstream_schema + schema_summary_model + schema_summary_prompt )
chunk_fp(directory_summary)  = sha1( child_object_uris + dir_summary_model + dir_summary_prompt )
                                                                    ← Milvus 行 chunk_fingerprint

embedding_fp                 = sha1( chunk_fp + embedding_model + embedding_model_version )
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

Step 2: artifact cache 层比对（仅对适用的 artifact_kind）
  for artifact_kind in processor.applicable_artifacts(object_kind):
    new_fp = compute_artifact_fp(...)
    if new != artifact_cache.fingerprint:
      enqueue artifact_rebuild → 触发下游 chunk + embed
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

#### 框架配置变化：v0.4 用户手动重建，自动检测留 v0.5+

reconcile（上面 4 步）只对 **connector yield 出来的 object** 跑——也就是只处理 **upstream 变化**。

但还有一类变化 connector 看不见：换 embedding 模型 / 升级 chunker / 改 text_fields 这类**框架配置变化**，upstream 完全没动、connector 啥都不 yield，但下游产物其实 stale。

**v0.4 对这类变化不做自动检测**——用户改了配置自己知道，手动重建：

```bash
mfs add <connector> --force-index      # 重建单个 connector
mfs add --all --force-index            # 重建全部（换全局 embedding 模型时用）
```

`--force-index` 跳过 fp 比对、把所有 object 当 modified 强制重 chunk + 重 embed（embed 走 transformation cache，内容没变的命中复用、省钱）。

> **简单 breadcrumb（仅记录、不检测）**：建索引 / `--force-index` 跑完时，framework 在 metadata DB 记一条"这次用的什么全局配置"（embedding model/version、chunker version、converter version）。v0.4 **不拿它做任何检测或提示**，纯粹留作 v0.5+ 自动检测的起点。

**为什么 v0.4 不做自动检测**：自动检测要一整套机制——sweep 全量扫 + config-hash gate 防止每次 sync 白扫 + index_state 比对 + 分级提示（embedding 破坏 search 必须重建 / chunker 只是 stale 可选）+ 维度变了要蓝绿重建 collection + 全局 fan-out 到所有 connector。这是一块**相对独立、且偏重**的能力，跟主链路（注册 / sync / 检索）解耦，放 v0.5+ 单独做更干净。v0.4 的原则简单：**用户手动改了配置，用户自己 `--force-index`**。

> v0.5+ 计划：基于上面那条 breadcrumb，在 `mfs status / add / search` 三处检测配置漂移并按严重度提示，用户确认后台重建（含维度变化的蓝绿迁移）。connector 数据版本（DATA_VERSION）的自动失效也一并放 v0.5+。
>
> 实现上 v0.5+ 的"per-connector 检测"和"全局 fan-out"**不是两套逻辑，是同一个 reconcile 引擎的两个触发器**：reconcile（扫一个 connector 的 object、比 fp、入队重建）只有一份；per-connector 触发器在该 connector sync 时检查它自己的 config fp，全局触发器在 server 检测到全局配置变化时把同一个 reconcile 对所有 connector 各调一遍。两个触发器对应配置的两个 scope（connector TOML vs server.toml 全局），但不重复实现重建逻辑。

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

**状态存储**：file connector 不用 `self.state` K/V（其他 connector 存 cursor 的地方），而用 framework 提供的 `self.file_state` 表接口（背后是 02 §10.1 的 `file_state` 表）。理由：

- 几十万 path 不挤进一个 JSON blob
- CS 模式 upload commit 步骤要直接 UPSERT `status='staged'` / `renamed_from` 字段（02 §4.2 ④）
- 按 `status` 索引能直接查"还没索引的"

file connector 的 sync 流程：

```
1. scan：os.walk(root) 应用 ignore rules（.gitignore + .mfsignore + 默认 binary 规则）
   得到当前 paths 集合 current_paths

2. 对每个 path 跟 file_state 对比 (stat-first lazy hashing):
   - file_state 里有 + (size, mtime_ns) 完全一致 → 跳过
   - file_state 里有 + (size, mtime_ns) 变化 → 算 sha1(content)
     - sha1 跟 file_state 一致 → 只 touch 了 mtime，UPDATE file_state.mtime_ns，跳过
     - sha1 不一致 → 加进 modified 候选
   - file_state 里没有 → 加进 added 候选

3. file_state 里有但 current_paths 没有 → 加进 deleted 候选

4. 处理已有 renamed_from 标记（CS 模式 commit 步预写）:
   - 对 file_state.status='staged' AND renamed_from IS NOT NULL 的行
     直接 yield ObjectChange("renamed", new=path, old_uri=renamed_from)
     不走下面的配对算法

5. 剩余 added × deleted 做 inode + sha1 配对（详见 §5.7.2）：
   - 本机模式：这是 rename detection 的主路径
   - CS 模式：作为 client 端配对失败的兜底——client 没识别出来 rename 时，
              client 会按 added + deleted 上报，bytes 上传到 staging；
              这里 sha1 配对仍能匹配，yield "renamed"，省下 embed 钱（但带宽已花）
   配对结果:
   - inode 一致（同 fs mv）→ yield ObjectChange("renamed", new, old_uri=old)，零 sha1
   - inode 不可信 / 失败 → size 预过滤后算 sha1，匹配 → yield "renamed"
   - 都没匹配 → yield "added" / "deleted"

6. yield ObjectChange，framework 处理:
   - 处理完写入 file_state: 更新 sha1 / size / mtime / inode，
                           status='indexed', renamed_from=null, indexed_at=now()
```

`file_state` 表行的核心字段：`(path, size, mtime_ns, inode, sha1, status, renamed_from)`，完整 schema 见 [02 §10.1](02-architecture.md#101-metadata-db)。

平台漂移处理：file connector 用 `connector_state` 表（K/V）存一个 `platform_id` 键，值是当前 inode 来源平台标识（如 `linux:ext4:/dev/sda1` / `darwin:apfs` / `windows:ntfs`）。connector 启动时比对当前 platform_id 跟 K/V 里存的：

- 一致 → file_state 的 inode 字段可信，正常 inode 配对
- 不一致 → 视所有 inode 字段为 NULL（rename 配对直接走 sha1 fallback，避免"另一平台上的 inode 数字撞到当前平台某个文件 inode"），sha1 字段仍可信

这处理"备份 `~/.mfs` 跨平台搬迁 / Docker volume 挂到不同 host fs"的场景：mtime 保留时常规变化检测照常 size+mtime 匹配跳过，**只有真的被 mv 的文件才进 sha1 配对路径**，不雪崩。

**file connector 不调 checkpoint**——file_state 是全量结构化映射，半截 commit 不合法（详见 §5.6.1）。

注意：file_state 只包含**文件本身的 fingerprint**（size / mtime_ns / inode / sha1），**不包含** chunker / embedding model / converter 版本。framework 层的配置变化跟 file_state 无关——v0.4 由用户手动 `--force-index` 处理（详见 §5.2 末尾）。

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
| 单调推进的 set + 关联 map | web crawl `visited_urls` 集合 + `{url: etag}` map | ✅ 推荐 | visited 集合只增不减，是 cursor 等价物——下次接续会"跳过已 visited"，合法 |
| paged token | gdrive `next_page_token` | 看 provider | gdrive token 长期有效 → OK；某些 provider token 短期失效 → 不推荐 |
| commit hash 类（A→B） | github code `commit_sha` / git tag / bigquery snapshot_time | ❌ | 一次 sync 是"从 A 跳到 B"的原子转换，没有合法的"半 commit" |
| 快照型全量映射（要原子替换才合法） | file `file_state` 表（必须反映"此刻整棵目录树"） | ❌ | 半截的映射不能宣称是某一时刻的快照真相 |

判断准则：你的 state 在 `checkpoint()` 那一瞬间，是不是一个**合法的"从此处接续"的起点**？

- 是 → 可以调（包括 cursor 类、单调推进 set + map）
- 不是（"原子替换才合法"的全量快照 / 中间转换状态） → 别调

举例对比：

- **web crawl** 的 `visited_urls = {url_a, url_b, ...}` 是单调增长——commit 到这个 set 后崩了重启，下次跑会跳过已 visited，BFS 继续。合法 ✓
- **file connector** 的 `file_state` 表是"对完整目录树的快照"——半截的映射没法说"已扫的就是真相"，下次再 walk 时拿不出"deleted = file_state - current_paths"这个集合的正确答案。不合法 ✗

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

### 5.7 Rename detection

笔记类场景里改文件夹名 / 改文件名很常见。如果当成 `deleted + added` 处理，所有 chunks 都要重新切分 + 重新 embed——既花钱又花时间，但文件内容根本没变。Framework 提供一套机制让 connector 报告 rename，跳过重 embed。

#### 5.7.1 协议层

connector 可选 yield `renamed` ObjectChange：

```python
yield ObjectChange(kind="renamed", uri=new_uri, old_uri=old_uri)
```

不能识别的 connector 照旧 yield `added` + `deleted`，正确但更贵。

#### 5.7.2 配对算法（推荐 inode + sha1 fallback）

两层配对，**绝大多数场景零 sha1 计算**。算法相同，跑在哪边由部署模式决定：

- **本机模式**：file connector sync 跑在 server 内，对 `file_state` 表 vs 真实目录树做配对
- **CS 模式**：client 端跑（基于 `/v1/files/manifest` 响应里的 `deletion_candidates`），结果作为 `renames_hint` 提交。Server 端 commit 步用 sha1 验证后直接写 `file_state.renamed_from`——server file connector sync 不再二次配对

每个 added 候选的决策树（逐层退化，坏情况不损坏）：

```
对每个 added 候选 new_path:
   │
   ├─ inode 可信 且 命中 deleted 里同 inode?
   │     │
   │     ├─ 是 ─► size 双重校验通过?
   │     │          ├─ 是 ─► renamed（零 sha1，同 fs mv 的快路径）
   │     │          └─ 否 ─► 往下（inode 复用，不可信）
   │     └─ 否 ─► 往下
   │
   ├─ size 跟某个 deleted 候选相同?（预过滤，避免无谓 sha1）
   │     │
   │     ├─ 是 ─► 算 sha1(new_path)，命中 deleted 里同 sha1?
   │     │          ├─ 是 ─► renamed（跨 fs / 网络 fs 的慢路径）
   │     │          └─ 否 ─► added（真新增）
   │     └─ 否 ─► added（真新增）
   │
配对剩下的 deleted 候选 ─► deleted（真删）
```

退化链：**inode 配对 → sha1 配对 → 当 added+deleted**。每往下一层只是更慢，最坏退到原始行为，不会数据损坏。下面是代码版：

```python
# 本机模式下 file connector sync 用到的配对（CS 模式下 client 端跑同一套，只是替换数据源）
# 把 deleted/added 各自按可配对的 key 索引，遍历 added 路径优先 inode 配对、失败回退 sha1。
deleted_by_inode = {file_state[p].inode: p for p in deleted_paths
                    if file_state[p].inode is not None}
deleted_by_sha1  = {file_state[p].sha1:  p for p in deleted_paths}

for new_path in sorted(added_paths):                  # 字典序保证 deterministic
    new_inode = get_stable_inode(new_path)            # os.stat().st_ino，平台不可信时返 None

    # 优先 inode（同 fs mv 场景，零 sha1）
    if new_inode and new_inode in deleted_by_inode:
        old_path = deleted_by_inode[new_inode]
        # 双重校验防 inode 复用
        if file_state[old_path].size == os.stat(new_path).st_size:
            yield ObjectChange("renamed", new_path, old_uri=old_path)
            deleted_by_inode.pop(new_inode)
            deleted_by_sha1.pop(file_state[old_path].sha1, None)
            continue

    # inode 配对失败（跨 fs / 网络 fs / inode 复用），按 size 预过滤再算 sha1
    new_size = os.stat(new_path).st_size
    has_size_match = any(file_state[p].size == new_size for p in deleted_by_sha1.values())
    if has_size_match:
        new_sha1 = compute_sha1(new_path)             # 真读盘
        old_path = deleted_by_sha1.pop(new_sha1, None)
        if old_path:
            yield ObjectChange("renamed", new_path, old_uri=old_path)
            continue

    # 真新增
    yield ObjectChange("added", new_path)

# 配对剩下的：真删
for old_path in set(deleted_by_sha1.values()):
    yield ObjectChange("deleted", old_path)
```

`get_stable_inode` 在不可信平台（Windows FAT32 / 部分网络 fs）返回 None，直接退到 sha1 路径——**没 inode 也能跑**，只是慢点。

CS 模式下 client 端跑同样算法时，`file_state[p]` 的字段是从 `/v1/files/manifest` 响应里的 `deletion_candidates` 携带的（server 把这些 path 的 size/inode/sha1 一起发回 client，详见 [02 §4.2 ③](02-architecture.md#42-本地文件-upload-flow不共享-fs-时)）。

#### 5.7.3 Framework 处理 `renamed`

renamed event 进 framework 后，**chunk 内容、所有 fingerprint 都不变**——只有 chunk_id 主键变（因为 `chunk_id = sha1(... + object_uri + ...)`）。处理流程：

```
对 old_uri 在 Milvus 里的所有 chunks:
  ① 读出 dense_vec / sparse_vec / content / locator / chunk_kind /
       chunk_fingerprint / embedding_fingerprint / metadata
  ② 算新 chunk_id = sha1(namespace + connector + new_uri + locator + chunk_kind)
  ③ INSERT 新行（向量 + content + 所有 fp 字段直接复用，不调 embedder）
  ④ DELETE 旧行（按旧 chunk_id 或按 object_uri 批量删）

artifact_cache 表: UPDATE object_uri = new_uri WHERE object_uri = old_uri
objects 表:        UPDATE object_uri = new_uri, parent_path = ... WHERE object_uri = old_uri
object_store:      物理 mv artifact cache 目录
  ~/.mfs/cache/artifacts/<ns>/<sha1(old_uri)>/  →  ~/.mfs/cache/artifacts/<ns>/<sha1(new_uri)>/
```

`chunk_fp` / `embedding_fp` 公式里都不含 `object_uri`（详见 [§5.2](#52-fingerprint-chain)），所以 rename 后 fp 真的不变，零外部 API 调用。

#### 5.7.4 边界情况

| 场景 | inode 行为 | 处理 |
|---|---|---|
| 同 fs mv | inode 不变 | inode 配对，零 sha1 |
| 跨 fs mv（fs1 → fs2）| 实际是 cp + rm，inode 变 | 退化 sha1 配对 |
| 编辑器保存（vim `:w` 走 rename-after-write） | 新 inode + 新 sha1 | 不进 rename 路径，走 modified 重 embed（语义正确）|
| 硬链接 `ln a.md b.md` | 两 path 同 inode，都还在 | 不在 deleted+added 集合，不进 rename 路径 |
| Windows FAT32 / 网络 fs | inode 不可信 | get_stable_inode 返 None，退化 sha1 |
| inode 复用（删原 → fs 把 inode 给新文件） | 理论可能 | size 双重校验拒掉；极端 case 仍走 sha1 兜底 |
| FS case-insensitive 改大小写（`Foo → foo`）| scan 看不到变化 | 不支持，用户自己绕（先改临时名）|
| 1 → N copy + 删原 | 1 个 inode/sha1 配对成功，剩下走 added | 部分优化 |
| 改名 + 编辑内容 | sha1 不匹配 | 退化为 deleted + added，重 embed（语义正确）|
| 同 sha1 多个文件（重复内容） | 按字典序配对，outcome 等价 | 任意配对都对，少一次 embed 就赢 |
| connector root 自己改名（`mv ~/notes ~/notes-v2`） | 新 connector_uri = 新 connector | 不在本机制范围；用户需 `mfs remove + add` |

#### 5.7.5 性能与正确性的兜底

inode 配对失败时退化到 sha1 配对，sha1 配对失败时退化到 `deleted + added`——**坏情况下退化为原行为，不会数据损坏**。

#### 5.7.6 不加 `mfs rename` 命令

考虑过但不采纳：

1. 用户已经 `mv` 了，daemon 自己 scan 就能发现，再多一条命令是冗余
2. 其他 connector（postgres / slack / github）上游没有 rename 概念
3. bulk rename 让用户自己 glob 出参数列表太丑

让 connector 自己负责 rename detection 比让用户手动报告靠谱。

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
- 停止单个 watch（保留 connector）：`mfs add ./repo --no-watch`
- 连 connector 一起删：`mfs remove ./repo`
- 停整个 daemon（所有 watch 一起停）：`mfs serve stop`
- 外部 connector 不支持 watch；要周期刷新用系统 cron / CI 调 `mfs add`（v0.4 不内置 scheduler，见 §9）

> Ctrl+C `mfs add --watch` CLI 进程只杀 CLI 自己，**不影响 daemon 内已经登记的 watcher**——watch 的生命周期跟 daemon 绑定，不跟启动它的那次 CLI 调用绑定。

首次 watch 某目录时弹权限确认：

```text
MFS local daemon will watch this directory:
  /repo

It will read file names, mtimes, sizes, and indexable file contents.
State is stored under ~/.mfs/.

Continue? [y/N]
```

### 7.1 跨重启恢复

watch 是 daemon 内存里的状态，daemon 挂了/机器重启就没了。靠 `watch_grants` 表（02 §10.1）持久化、daemon 启动时 replay 重建：

```
daemon 启动:
  for grant in SELECT path FROM watch_grants:
    重新登记 watcher（无需再弹权限确认，grant 已记录授权）
  → mfs serve restart / 机器重启后 daemon 起来，watch 自动续上
```

为什么这件事不大：

- **watch 只在本机 profile 有意义**——生产 CS 部署根本不用 watch，用系统 cron 调 `mfs add` 周期刷新（§9）。所以"daemon 重启 watch 丢"在生产场景不存在
- **就算 watch 真断了一阵也不丢数据**：watch 只是"触发信号"，最终事实来自 scan + manifest 对比（§5.5）。断的这段时间发生的改动，下次 `mfs add`（或 daemon 重启 replay 后第一次扫）会全量对比补上
- **已索引的成果都在 DB / Milvus，重启不丢**：connector_state（cursor）、file_state、Milvus chunks 都持久化；in-flight 的 `running` job 靠 heartbeat 超时（02 §7.1）重置 pending、幂等重跑

想要本机 daemon 开机常驻（这样 watch 一直活着）的用户，自己挂个 launchd（macOS）/ systemd user unit（Linux）拉起 `mfs serve`，配合上面的 watch_grants replay 就无缝。`MFS_AUTOSTART=1` 只在首次 `mfs add` 检测不到 server 时 spawn 一次，不保证常驻。

> 生产 server（非 watch 场景）的重启恢复是另一回事：server 一般拆成 `mfs-api` + `mfs-worker` 跑在各自 Docker/K8s Pod 里，进程挂了编排器拉起来，状态全在外部 DB，重启即从 DB 续——详见 [02 §5](02-architecture.md#5-server-启动) / [§7.1](02-architecture.md#71-故障恢复)。

## 8. Connector 能力声明

每个 connector plugin 声明能力，agent 通过 `mfs connector inspect <root>` 看到：

```json
{
  "connector_type": "postgres",
  "uri_scheme": "postgres",
  "sync": {
    "manual": true, "watch": false,
    "cursor": "updated_at", "full_scan": true,
    "delete_detection": "full_scan"
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
| watch | 本地目录 | `mfs add . --watch`（仅本地文件，实时）|
| 游标 | slack / gmail / 部分 SaaS | provider cursor 或 updated_at（连接器内部增量手段，仍由一次 `mfs add` 触发）|
| snapshot 对比 | s3 / gdrive / DB fallback | 全量列举对比（由一次全量 `mfs add` 触发）|
| append-only | logs / chat / events | 追加尾部 + 全量校准 |

**v0.4 不内置定时调度（scheduler）**。除本地文件的 `--watch` 外，所有同步都由用户主动 `mfs add` 触发。游标 / snapshot / append-only 说的是"连接器内部怎么算增量"，不是"什么时候自动跑"——它们都还是等一次 `mfs add` 来触发。

想要外部 connector 周期性刷新，用系统 cron / CI 调 CLI 即可（MFS 不拥有这件事）：

```bash
*/15 * * * *  mfs add slack://eng        # 自己的 crontab，零 MFS 代码
```

`mfs status` 显示每个 connector "上次同步多久前"，用户据此决定何时手动刷。

> 内置 scheduler（在 connector TOML 写 `schedule = "*/15 * * * *"` + daemon 自动触发 + 多副本单飞协调）是 v0.5+ 范畴——独立的便利能力，v0.4 用"手动 + 自带 cron"兜底，不引入调度复杂度。

## 10. 错误恢复与重跑

整套 sync 正确性靠 [02 §7](02-architecture.md#7-一致性) 的四条一致性规则保证，这一节只补"用户视角"。

### 10.1 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force-index` | 所有 object 视为 modified，跳过 fp 比对，强制重 chunk + embed |
| `mfs add ./path --force-upload` | 仅 upload flow：跳过 manifest diff，所有 path 按 stale 处理全量重传 + server 强制重 index |
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
