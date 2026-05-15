# Source 同步与 MFS Index 设计

本文定义每类 source 的 **MFS Index**，以及每类 source 在 URI 树下暴露哪些对象。

## 1. 三类概念

### 1.1 原始对象

外部系统里的真实数据：

- 本地文件。
- GitHub 文件、issue、PR。
- Slack 消息。
- Postgres row。
- Zendesk ticket。

原始对象通常仍留在原系统。MFS 只在需要时读取、缓存或导出。

### 1.2 MFS Index（用户面总称）

**MFS Index 是 MFS 为一个 source 维护的全部索引集合**。它分两个子层：

| 子层 | 内容 | 用途 |
| --- | --- | --- |
| **Index Artifacts** | metadata、normalized content、structure、chunks、summary、schema/sample/rows/messages/records 等 JSONL 缓存 | 给 `cat / ls / tree / head / tail` 和密度视图使用 |
| **Retrieval Index** | dense vector、sparse/BM25、metadata filter、field index | 给 `search / grep` 召回排序使用 |

用户面只暴露 **MFS Index** 一个总称。开发者文档才出现 Index Artifacts 和 Retrieval Index 这两个细分。

第三个内部概念：

- **Index Plan**：source TOML 里描述「该生成哪些 Artifacts、用哪些字段做检索」的**配置**。Plan 是输入，Artifacts 和 Retrieval Index 是输出。

`mfs status` 输出里 `index=fresh/stale` 指的是 Artifacts 的 freshness 汇总。

### 1.3 对象（Object）

每个 source 在 URI 树下暴露一组**对象**。对象就是「能 `cat` 的东西」。它可能是：

- 真实文件（本地文件、GitHub blob、S3 object）。
- MFS 从 source 拉取并维护的 JSONL（DB rows、Slack messages、Zendesk records）。
- MFS 从 source 抽取的 schema 描述。
- MFS 为 source 计算的 summary。

**用户视角不区分"原对象 vs MFS 生成"**。所有对象都只是「source 下面的对象」，cat 它就行。每个对象的具体形态（文件 vs 表 vs 流）由其类型决定，agent 通过 `mfs stat`/`mfs source inspect` 发现。

## 2. URI 与对象命名

每个 source root 下暴露什么对象，由 source 类型决定。文档约定：

- **对象名带 media type 后缀**：`schema.json` / `sample.jsonl` / `rows.jsonl` / `messages.jsonl` / `threads.jsonl` / `records.jsonl` / `comments.jsonl` / `users.jsonl` / `issues.jsonl` / `pulls.jsonl` / `database.json` 等。后缀就是 unix 风格的 media type hint，跟 Mirage 一致。
- 真实文件保留原文件名和后缀：`github://mfs/README.md`、`gdrive://docs/Pricing.gdoc`、`s3://logs/app.jsonl`。
- 同一容器的不同视角通过不同对象表达（`schema.json` 一份、`sample.jsonl` 一份、`rows.jsonl` 一份），不引入"view"概念。
- 目录节点没有后缀（`tables/tickets/`、`channels/incidents/`、`pulls/42/`），`cat` 它返回 `is_directory` 错误，跟 unix 一致。

`mfs ls` 列出一个 source 子节点时，看到的就是该 source 决定暴露的对象集合，例如：

```bash
$ mfs ls postgres://prod/public/tickets
schema.json    # cat 得到表 schema
sample.jsonl   # head 得到前 N 条样本
rows.jsonl     # 全部行（大对象 cat 会拒绝，用 head/tail/grep/export）
```

每个对象 `mfs stat` 给出 media type、估计大小、能力（能否 tail / grep pushdown / cat --range）。

## 3. URI 映射规则

各 source 类型下暴露的对象命名约定。对象名带 media type 后缀。

### 3.1 数据库

```text
postgres://{alias}/{schema}/{table}/schema
postgres://{alias}/{schema}/{table}/sample
postgres://{alias}/{schema}/{table}/rows
postgres://{alias}/{schema}/{table}/query/{query_id}
```

### 3.2 消息

```text
slack://{alias}/channels/{channel}/{yyyy-mm-dd}/messages
slack://{alias}/channels/{channel}/{yyyy-mm-dd}/threads
slack://{alias}/users
slack://{alias}/files/{file_id}
```

### 3.3 GitHub

```text
github://{alias}/{path}              # 真实文件，保留原后缀
github://{alias}/issues
github://{alias}/pulls
github://{alias}/pulls/{number}/diff
github://{alias}/pulls/{number}/reviews
```

### 3.4 Zendesk / SaaS

```text
zendesk://{alias}/tickets/schema
zendesk://{alias}/tickets/records
zendesk://{alias}/tickets/comments
zendesk://{alias}/users/records
zendesk://{alias}/organizations/records
```

每个 source 类型的具体对象集合写在自己的 `sources/<type>/` 文档里，作为稳定契约。

## 4. Index Plan 配置位置

Index Plan 属于 source 配置包，通过 source TOML 和 workspace TOML 表达。

覆盖顺序：

```text
connector built-in defaults
  -> workspace.toml defaults
  -> source TOML defaults
  -> object-level override
```

示例：

```toml
[index.defaults]
metadata = true
normalized_text = true
structure = true
chunks = true
summary = "auto"
retrieval = true

[objects."public.tickets"]
kind = "table"
path = "public.tickets"
primary_key_fields = ["id"]
updated_at_field = "updated_at"

[objects."public.tickets".retrieval]
text_fields = ["subject", "description", "latest_comment"]
metadata_fields = ["id", "status", "priority", "updated_at"]
```

运行时状态写入 metadata store：

- provider cursor。
- last_add（最近一次同步时间）。
- artifact freshness 与版本。
- job 状态和失败原因。
- retrieval index 构建版本。
- 权限快照版本。

source TOML 描述「Plan」；metadata store 记录「Artifacts 和 Retrieval Index 已经生成到哪里」。

每个 source 的默认 Plan 放在 `sources/<type>/index_artifacts.py`。

原则：

- source 决定「哪些对象需要生成哪些 artifacts」。
- `objects/` 决定「这个对象形态怎么转成 metadata / normalized content / structure / chunks」。
- `pipeline/` 提供 provider 和检索执行。
- worker 执行 source 规划的 job。

例：

| Source | `sources/<type>/index_artifacts.py` 选择的对象处理 |
| --- | --- |
| file source | `objects/document`、`objects/code`、`objects/table`、可选 summary |
| GitHub repo | `objects/code`、`objects/document`、branch/tree/blob metadata |
| Postgres/MySQL | `objects/table` 生成 schema、sample、rows、row text |
| Slack/Gmail | `objects/message_thread` 生成 messages、thread 聚合、thread summary |
| Zendesk/Salesforce | `objects/business_record` 生成 records、comments/timeline、record summary |

## 5. 同步入口

`mfs add` 是统一的同步入口。本地路径和外部 source URI 行为对称：

```bash
mfs add ./repo                    # 本地：首次注册 + 同步
mfs add ./repo                    # 已注册：再同步一次
mfs add ./repo --force            # 强制重建
mfs add ./repo --watch            # 启动 watcher

mfs add postgres://prod --config .mfs/sources/prod-postgres.toml   # 首次
mfs add postgres://prod                                            # 再同步
mfs add postgres://prod --force                                    # 强制
mfs add slack://eng --since 2026-05-01                             # 从某时间点
```

同步语义：让 source 的 MFS Index 跟外部系统对齐。包括 schema、sample、page cache、normalized text、structure、chunks、summary 和 retrieval index。

`mfs source` 子树只做管理（list/inspect/update/remove），**不再有 `mfs sync` 和 `mfs source add` 顶级动词**。

执行位置取决于 active profile：

| profile kind | `mfs add` 行为 | queue 位置 | 执行者 |
| --- | --- | --- | --- |
| `local` | client 调 `POST /v1/add`，本机 daemon 创建 job | daemon 侧 SQLite queue 或进程内 runner | local daemon worker |
| `remote` | client 调 `POST /v1/add`，remote server 创建 job | server 侧 Redis/Postgres queue | `mfs-worker` |

worker 从队列取 `sync_source`、`build_index_artifacts`、`build_retrieval_index`、`build_summary`、`export_object` 这些 job，调用共享 engine。

## 6. 同步类型

| 同步类型 | 适合 source | 触发方式 |
| --- | --- | --- |
| 手动同步 | 所有 | `mfs add <uri>` |
| watch 同步 | 本地目录 | `mfs add . --watch` |
| 定时同步 | Slack、GitHub、SaaS、DB | local daemon scheduler 或 remote worker 周期触发 |
| 游标同步 | Slack、Gmail、部分 SaaS | provider cursor 或 updated_at |
| snapshot 对比 | S3、Drive、DB fallback | 周期列举对比 hash/mtime/version |
| append-only 同步 | logs、chat、events | 追加尾部 + 周期校准 |

变化检测属于具体 source 的 `sync.py`：

| Source | 变化检测 |
| --- | --- |
| file source | manifest 对比：path、size、mtime、hash |
| GitHub repo | branch commit、tree hash、blob sha、issue/PR updated_at |
| Google Drive / Feishu Docs | provider revision、modified time、file id |
| S3/object store | object key、etag、size、last modified、version id |
| Slack/Gmail/chat | provider cursor、channel timestamp、thread timestamp |
| DB | primary key、updated_at、CDC、snapshot fallback |
| SaaS | provider cursor、updated_at、object version |

共享 engine 消费统一的 `ChangeSet`：added、modified、deleted、unchanged、cursor、watermark。具体 cursor/etag/updated_at 细节封装在 source 内部。

### 6.1 本地文件变化检测

file source 使用 manifest 策略：

```text
path
size
mtime_ns
file_hash
file_type
encoding
last_seen_at
artifact_version
```

默认流程：

1. 扫描目录。
2. path 集合差异判断创建/删除。
3. 已记录文件先比较 `size + mtime_ns`。
4. 未变化跳过；变化则计算内容 hash。
5. hash 变化才重建 normalized content、chunks、summary、retrieval index。
6. `--force` 跳过快速判断，重建可重建的 Artifacts。

`watchfiles` 等 watcher 提供触发信号；最终事实来自扫描和 manifest 对比。

| 场景 | 地址 | 执行位置 | 变化检测 |
| --- | --- | --- | --- |
| local profile + 本地路径 | `./repo` | local daemon | `sources/file/sync.py` manifest |
| remote profile + 本地路径 | `./repo` | client 返回 `remote_server_cannot_read_local_path` | 本机文件由 local daemon 处理 |

Merkle tree 作为 manifest 加速层，是可选优化，不是默认机制。

## 7. 各类 source 暴露的对象

| Source 类型 | 默认 URI | 暴露的对象 | Retrieval 单元 | 同步策略 |
| --- | --- | --- | --- | --- |
| file source | `./repo` 或 `/data/repo` | 真实文件 + summary | 文件 chunk | `mfs add`、`--watch`、手动 |
| GitHub repo | `github://{alias}/...` | 真实文件 + `issues`、`pulls`、`pulls/{n}/diff`、`pulls/{n}/reviews` | 文件 chunk、issue/PR 聚合对象 | 手动、定时、branch revision |
| Google Drive / Feishu Docs | `gdrive://{alias}/...` | 真实文件 + 导出文本 + 表格 JSONL | 文档 chunk、表格 row | 手动、定时、provider revision |
| S3 / 对象存储 | `s3://{alias}/...` | 真实对象 + JSONL page cache | 文件 chunk、JSONL record/window | 手动、定时、snapshot、append-only |
| Slack / Discord / Feishu chat | `slack://{alias}/...` | `messages`、`threads`、`users`、`files/{id}` | thread / session / time-window | 手动、定时、游标 |
| Gmail / Email | `gmail://{alias}/...` | `messages`、`threads`、`attachments` | thread | 手动、定时、游标 |
| GitHub Issues/PRs | `github://{alias}/issues` `pulls` | `issues`、`pulls`、`pulls/{n}/diff`、`pulls/{n}/reviews` | issue/PR 聚合 | 手动、定时、updated_at |
| Jira / Linear / Notion | `linear://{alias}/...` | `issues` / `records`、`comments`、`timeline` | 对象聚合 | 手动、定时、updated_at |
| Postgres / MySQL | `postgres://{alias}/{schema}/{table}/...` | `schema`、`sample`、`rows`、`query/{id}` | row 或聚合对象 | 手动、定时、updated_at / CDC / snapshot |
| BigQuery / Snowflake | `bigquery://{alias}/...` | `schema`、`sample`、`query/{id}` | row / window / partition | 手动、定时、partition |
| Salesforce / HubSpot / Zendesk | `zendesk://{alias}/...` | `schema`、`records`、`comments`、`activities`、`timeline` | record / ticket / account 聚合 | 手动、定时、updated_at / cursor |

## 8. 哪些全量保存，哪些只保存 Artifacts

### 8.1 本地文件

默认保存 Artifacts，原始文件留在用户本地。MFS 保存：

- 文件 hash、文件级 metadata。
- chunk metadata。
- embedding 与关键词索引。
- 可选转换文本缓存。
- 可选 summary。

### 8.2 云端文件

默认保存 metadata 和抽取文本缓存：

- 文件 ID、path、revision。
- 导出文本。
- 文档 chunk。
- 权限摘要。
- summary。

大文件、二进制、附件按需读取或 `mfs export`。

### 8.3 消息型

默认保存 JSONL page cache 或 thread 聚合：

- channel/date/thread metadata。
- `messages` page cache。
- thread index text。
- 附件 metadata。
- thread/session summary。

### 8.4 数据库

默认保存表的 Artifacts：

- `schema`。
- `sample`。
- `rows` page cache。
- 由 Plan 指定字段生成的 normalized text。
- 主键、updated_at、metadata fields。
- retrieval index。

只有显式 `mfs export` 或配置 raw cache 时才物化更大的数据。

### 8.5 业务 SaaS

默认保存对象 Artifacts：

- `schema`。
- `records` page cache。
- comments / activities / timeline。
- Plan 指定字段。
- 权限和对象版本。
- 对象 summary。

## 9. Source 能力声明

每个 source 类型需要声明：

- 是否支持自动同步。
- 是否支持手动同步。
- 是否支持 provider cursor。
- 是否支持 full scan。
- 是否支持 delete detection。
- 是否支持 permission snapshot。
- 是否支持 efficient tail。
- 是否支持 grep pushdown。
- 是否支持 paged cat（按 `--range` 切片）。

示例：

```json
{
  "source_type": "postgres",
  "sync": {
    "manual": true,
    "scheduled": true,
    "watch": false,
    "cursor": "updated_at",
    "full_scan": true
  },
  "object": {
    "grep_pushdown": true,
    "tail": false,
    "paged_cat": true
  }
}
```

能力声明通过 `mfs source inspect <root-uri>` 暴露给 client，agent 据此判断哪些命令对该 source 可用。
