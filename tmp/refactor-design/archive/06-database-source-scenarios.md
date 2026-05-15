# 数据库型 Source 场景

数据库 source 的用户体验是「可浏览、可搜索、可脚本处理的数据资源」。用户通常先看 schema 和 sample，再决定搜索哪些字段。

代表：

- Postgres。
- MySQL。
- MongoDB。
- Snowflake。
- BigQuery。

## 1. Postgres

### 添加 source

```bash
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
```

输出：

```text
Added source: postgres://prod
type: postgres
health: ok
schemas:
  public
Sync started.
job: job_01HX...
```

配置：

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

`index.defaults` 配置 source 默认 Index Plan；`objects."public.tickets"` 配置单表覆盖。Retrieval Index 由 `mfs add` 间接生成。

### 再同步

```bash
mfs add postgres://prod              # 幂等：再同步一次
mfs add postgres://prod --force      # 强制重建
mfs add postgres://prod --since 2026-05-01  # 增量
```

输出：

```text
Sync started: postgres://prod
job: job_01HX...
index:
  schema: public.tickets, public.accounts
  sample: public.tickets, public.accounts
  retrieval_index: public.tickets
```

## 2. 浏览数据库结构

### 看有哪些表

```bash
mfs ls postgres://prod/public
```

输出：

```text
public/
  accounts/
    schema
    sample
    rows
  tickets/
    schema
    sample
    rows
```

每个表下面暴露三个对象（`schema.json` / `sample.jsonl` / `rows.jsonl`），由 postgres source 决定。对象名带 media type 后缀。

### 看 schema

```bash
mfs cat postgres://prod/public/tickets/schema.json
```

输出：

```json
{
  "table": "public.tickets",
  "primary_key": ["id"],
  "columns": [
    {"name": "id", "type": "bigint", "nullable": false},
    {"name": "subject", "type": "text", "nullable": false},
    {"name": "description", "type": "text", "nullable": true},
    {"name": "status", "type": "text", "nullable": false},
    {"name": "priority", "type": "text", "nullable": true},
    {"name": "updated_at", "type": "timestamp", "nullable": false}
  ]
}
```

### 看样例

```bash
mfs head -n 3 postgres://prod/public/tickets/sample.jsonl
```

输出：

```jsonl
{"id":12,"subject":"Login broken after SSO migration","status":"open","priority":"high","updated_at":"2026-05-10T12:30:00Z"}
{"id":41,"subject":"SSO redirect loop","status":"pending","priority":"medium","updated_at":"2026-05-10T13:10:00Z"}
```

### 区间读取 rows

```bash
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100
```

输出是 JSONL；`--json` 返回 `items` 和 `range`，**没有 cursor**。大对象无 `--range` 的 `cat` 请求会被拒绝并提示 head/tail/export。

## 3. 搜索数据库记录

```bash
mfs search "customer cannot login after SSO migration" postgres://prod/public/tickets
```

输出：

```text
[1] postgres://prod/public/tickets/rows.jsonl  score=0.842
     row: id=12
     subject: Login broken after SSO migration
     status: open
     priority: high
     updated_at: 2026-05-10T12:30:00Z

[2] postgres://prod/public/tickets/rows.jsonl  score=0.776
     row: id=41
     subject: SSO redirect loop
     status: pending
     priority: medium
```

搜索结果的 `source` 指向包含记录的 rows 对象；精确对象放在 `locator`。要打开单条记录：

```bash
mfs grep '"id":12' postgres://prod/public/tickets/rows.jsonl
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
jq 'select(.id == 12)' ./tickets.jsonl
```

输出：

```json
{
  "id": 12,
  "subject": "Login broken after SSO migration",
  "description": "Enterprise users cannot complete SSO redirect.",
  "status": "open",
  "priority": "high",
  "updated_at": "2026-05-10T12:30:00Z"
}
```

## 4. Grep 和 query

```bash
mfs grep "SSO" postgres://prod/public/tickets/rows.jsonl
```

输出：

```text
postgres://prod/public/tickets/rows.jsonl
12  {"id":12,"subject":"Login broken after SSO migration","status":"open"}
41  {"id":41,"subject":"SSO redirect loop","status":"pending"}
```

行为：

- 字段配置明确时，可以下推成 SQL。
- 没有全文索引时，可以分页扫描但受 row/byte 限制。
- 输出保持 JSONL 行，便于脚本处理。

受控 query（保存为 query 对象）：

```bash
mfs cat 'postgres://prod/public/tickets/query/open-tickets'
```

query 对象的注册和参数化由 source TOML 配置；运行时安全限制由 source 内部 enforce。

## 5. BigQuery / Snowflake

```bash
mfs add bigquery://warehouse --config .mfs/sources/bigquery.toml
mfs ls bigquery://warehouse/analytics
mfs cat bigquery://warehouse/analytics/events/schema.json
mfs head -n 20 bigquery://warehouse/analytics/events/sample.jsonl
```

行为：

- 查询成本要在输出中提示。
- 默认 sample 和 schema 低成本。
- 大表搜索必须依赖 source TOML 里的搜索字段、分区字段和同步字段。
- 强制重建：`mfs add bigquery://warehouse --force`。

## 6. MongoDB

URI：

```text
mongo://prod/app/users/schema.json
mongo://prod/app/users/sample.jsonl
mongo://prod/app/users/documents.jsonl
```

命令：

```bash
mfs head -n 20 mongo://prod/app/users/sample.jsonl
mfs grep '"plan":"enterprise"' mongo://prod/app/users/sample.jsonl
```

行为：

- schema 是采样和推断结果，需要标明 confidence。
- 文档集合默认是 JSONL 形态的对象。
- identity 可以使用 `_id`。

## 7. 数据库型 source 的 UX 原则

- `schema` 是用户第一站。
- `sample` 帮用户理解数据。
- `rows` 是可区间读取的对象，不默认一次性读完。
- embedding 字段必须配置或来自明确 preset。
- metadata 字段用于过滤和展示。
- 搜索结果返回 URI 加 locator；用户用 `grep`、`cat --range` 或 `export + jq` 精确打开对象。
- 所有数据库访问默认只读、限时、限行、限字节。
