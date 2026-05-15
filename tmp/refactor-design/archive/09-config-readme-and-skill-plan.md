# 配置、README 与 Skill 计划

## 1. 配置文件格式

项目配置统一使用 TOML；运行时输出使用 JSON；大批量对象使用 JSONL。

推荐目录：

```text
.mfs/
  workspace.toml
  sources/
    prod-postgres.toml
    slack-eng.toml
    zendesk-support.toml
  secrets.example.toml
```

真实凭据不进入仓库，只写 `credential_ref`。搜索字段、同步字段和 Index Plan 统一放在 source TOML 里。

Index Plan 按四层覆盖：

```text
connector built-in defaults
  -> workspace.toml defaults
  -> source TOML defaults
  -> object-level override
```

含义：

- connector 内置默认值保证 source 不写复杂配置也能跑起来。
- `workspace.toml` 放团队或项目的默认策略。
- source TOML 放某个 source 的主要 Index Plan。
- object-level override 处理表、频道、对象类型之间的差异。

运行时状态不写回 TOML。cursor、last_add、artifact freshness、job 状态、失败原因、已生成版本等放在本地 metadata DB 或服务端 Postgres。

`workspace.toml` 示例：

```toml
[index.defaults]
metadata = true
normalized_text = true
structure = true
chunks = true
summary = "auto"
retrieval = true

[index.limits]
max_read_bytes = "10MiB"
max_page_rows = 1000
max_summary_tokens = 800
```

## 2. Source 配置示例

Postgres：

```toml
[source]
type = "postgres"
root = "postgres://prod"
label = "Production Postgres"
credential_ref = "secret:postgres-prod-readonly"

[postgres]
schemas = ["public"]
default_row_limit = 1000
max_read_rows = 10000
max_read_bytes = "10MiB"

[index.defaults]
metadata = true
schema = true
sample = true
rows_page_cache = true
normalized_text = true
chunks = true
summary = "auto"
retrieval = true

[objects."public.tickets"]
kind = "table"
path = "public.tickets"
primary_key_fields = ["id"]
updated_at_field = "updated_at"

[objects."public.tickets".sample]
rows = 100

[objects."public.tickets".retrieval]
text_fields = ["subject", "description", "latest_comment"]
metadata_fields = ["id", "status", "priority", "assignee", "updated_at"]
```

Slack：

```toml
[source]
type = "slack"
root = "slack://eng"
label = "Engineering Slack"
credential_ref = "secret:slack-eng-reader"

[slack]
channels = ["general", "incidents", "deploys"]
history_window_days = 365
include_files = true

[index.messages]
group_by = "day"
unit = "thread"
thread_field = "thread_ts"

[index.messages.retrieval]
text_fields = ["text", "thread_summary"]
metadata_fields = ["channel", "user", "ts", "thread_ts"]

[objects."channels.incidents"]
history_window_days = 365
include_files = true
```

Zendesk：

```toml
[source]
type = "zendesk"
root = "zendesk://support"
label = "Support Zendesk"
credential_ref = "secret:zendesk-support-reader"

[zendesk]
objects = ["tickets", "users", "organizations"]
include_comments = true

[index.defaults]
schema = true
records_page_cache = true
comments = true
summary = "auto"
retrieval = true

[index.tickets.retrieval]
text_fields = ["subject", "description", "comments.body"]
metadata_fields = ["id", "status", "priority", "assignee_id", "updated_at"]
```

## 3. Index Plan 配置口径

Index Plan 属于 source 配置包。配置包可以是一个 TOML，也可以在变大后拆分，概念上仍属于 source：

```toml
[index]
include = "./index/prod-postgres-index.toml"
```

拆文件只解决文件大小和维护问题，不引入新的公开概念。

常见配置字段：

| 字段 | 作用 |
| --- | --- |
| `primary_key_fields` | 增量覆盖和 locator |
| `updated_at_field` | 增量同步 |
| `text_fields` | 生成 normalized text、chunks 和 embedding |
| `metadata_fields` | 搜索结果展示、过滤和排序 |
| `group_by` | 消息、日志、事件流的 JSONL 分组 |
| `history_window_days` | 初始同步窗口 |
| `max_read_rows` / `max_read_bytes` | 防止大对象一次性读取 |

`retrieval` 是 Index Plan 的检索层，由 `mfs add` 间接生成 Retrieval Index。

## 4. README 新故事

README 首屏直接讲清楚 MFS 的定位和入口。

定位（**新叙事，放弃旧的 Memory / Milvus File Search**）：

> **MFS** — Multi-source File-like Search.
>
> One shell-native CLI for agents to search and inspect files, repos, cloud docs,
> chats, tickets, databases, and SaaS records. Everything is addressed by path
> or URI; every command is POSIX-style (`ls` / `cat` / `grep` / `head` / `tail`).
> **No filesystem mount, no client library** — just a binary on `$PATH` and a
> `--json` flag.

"file-like" 含义必须在 README 第一段说清楚——指 URI 寻址 + POSIX 风格命令，**不是 FUSE mount**。这一行避免用户误以为 MFS 会把 source 挂成真实文件系统。

核心示例仍然从本地开始：

```bash
mfs daemon start
mfs profile use local
mfs add .
mfs search "where do we configure database retries" .
mfs grep "ERR_TIMEOUT" ./src
mfs tree --peek -L 2 .
mfs cat --skim ./docs/getting-started.md
```

然后展示多源：

```bash
mfs add slack://eng --config .mfs/sources/slack-eng.toml
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
mfs search "customer cannot login after deploy" postgres://prod/public/tickets
mfs grep '"id":12' postgres://prod/public/tickets/rows.jsonl
```

README 需要强调：

- 本地文件体验由 local daemon 提供，用户入口仍然是熟悉的 `mfs add/search/grep/cat`。
- 本地路径直接写普通路径；外部 source 使用 URI。
- `mfs add` 是统一注册 + 同步入口（幂等）。
- 多源 source 是扩展能力。
- Agent 通过 shell 命令工作，不需要 SDK。
- 搜索结果包含可继续操作的 URI 和 locator。
- 大对象用 `head/tail`、`cat --range`、`tail -f`、`export`；**没有 cursor token**。

## 5. 用户文档结构

```text
docs/
  getting-started.md
  local-files.md
  search-and-inspect.md
  sources/
    overview.md
    github.md
    google-drive.md
    slack.md
    postgres.md
    zendesk.md
  config/
    source-config.md
    mfs-index.md
    credentials.md
  agent/
    workflows.md
    json-output.md
    troubleshooting.md
```

每个 source guide 固定回答：

1. 怎么注册 source（`mfs add <root-uri> --config <toml>`）。
2. URI 长什么样、暴露哪些对象（带 `.jsonl`/`.json` 后缀）。
3. 怎么再同步（`mfs add <root-uri>`，幂等）。
4. 怎么浏览。
5. 怎么搜索。
6. 怎么打开或过滤搜索结果。
7. `--json` 输出是什么。
8. 需要哪些权限和凭据。

## 6. Skill 设计

Skill 的目标是教 agent 正确使用 MFS。

目录：

```text
skills/mfs/
  SKILL.md
  references/
    local-workflow.md
    source-workflow.md
    jsonl-workflow.md
    database-workflow.md
```

`SKILL.md` 核心规则：

- 先用 `mfs tree --peek` 或 `mfs ls` 了解范围。
- 本地文件任务先确认 local daemon 可用：`mfs daemon status`；不可用时运行 `mfs daemon start`。
- 本地任务优先 `mfs search <query> .`，再 `mfs cat -n` 精读。
- 多源任务先 `mfs source list`，再选择具体 source URI；用 `mfs source inspect <root>` 看 source 暴露哪些对象、有什么能力。
- 对结构化 source 先读 `schema` 和 `sample`。
- 对 JSONL 对象用 `mfs head`、`mfs tail`、`mfs cat --range`、`mfs grep`；遍历大对象用 `mfs export` 写本地再处理。
- 搜索结果必须再打开或过滤验证，snippet 只作为定位线索。
- 大对象 `cat` 会被拒绝；用 `head/tail/cat --range/export` 替代。
- 不要假设 cursor token——MFS 不暴露 cursor。

## 7. 文档中的命令口径

统一写法：

```bash
mfs search "query" <path-or-uri>
mfs grep "pattern" <path-or-uri>
mfs cat <path-or-uri>
mfs cat <uri> --range A:B
mfs head -n 20 <path-or-uri>
mfs tail -n 50 <path-or-uri>
mfs tail -f <append-only-uri>
mfs export <path-or-uri> <local-file>
```

多源场景使用同一套 CLI 风格：本地 path 和 source URI 都能进入 `search/grep/ls/tree/cat/head/tail/export`。
