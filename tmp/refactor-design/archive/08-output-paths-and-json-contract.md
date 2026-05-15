# 输出、URI 与 JSON 契约

## 1. 输入地址

MFS 命令接受本地路径和外部 source URI。

本地路径：

```bash
mfs cat ./docs/auth.md
mfs cat /data/repo/README.md
mfs search "retry policy" ./src
```

外部 source URI：

```bash
mfs cat postgres://prod/public/tickets/schema.json
mfs head -n 20 postgres://prod/public/tickets/sample.jsonl
mfs tail -n 50 slack://eng/channels/incidents/2026-05-10/messages.jsonl
```

规则：

- 本地文件继续使用普通相对路径或绝对路径。
- 外部 source 使用 `postgres://prod/...`、`slack://eng/...`、`zendesk://support/...` 这类 URI。
- **对象名带 media type 后缀**（`schema.json` / `sample.jsonl` / `rows.jsonl` 等），跟 unix 习惯一致。`--json` 输出同时携带 `media_type` 字段（例如 `application/x-ndjson`）。
- `file:///...` 是规范化 file URI，主要用于 `--json` 输出和 HTTP API；日常命令直接写普通路径。
- 搜索结果里的地址不包含凭据。
- 概念解释见 [16-concepts-paths-anchors-and-command-decisions.md](16-concepts-paths-anchors-and-command-decisions.md)。

## 2. 地址示例

```text
./src/mfs/cli.py
/data/repo/docs/auth.md
github://mfs/README.md
gdrive://company-docs/Product/Pricing Strategy.gdoc
slack://eng/channels/incidents/2026-05-10/messages.jsonl
postgres://prod/public/tickets/rows.jsonl
zendesk://support/tickets/records.jsonl
```

数据库连接串不作为用户路径：

```text
postgresql://user:password@host:5432/db
```

它只进入 source TOML 或 secret provider。

## 3. 文本输出

本地搜索输出：

```text
[1] src/auth/token.py  score=0.890
142  def refresh_token(user_id: str, refresh_jwt: str) -> Token:
143      """Exchange a refresh token for a new access token.
```

外部 source 搜索输出延续这个形态，精确定位放在结构化字段和人类可读提示里：

```text
[1] postgres://prod/public/tickets/rows.jsonl  score=0.856
     row: id=123
     subject: Login broken after SSO migration
     status: open
     priority: high
```

`grep` 仍然按 path/URI 分组：

```text
slack://eng/channels/incidents/2026-05-10/messages.jsonl
118  {"ts":"1715320060.456","user":"U2","text":"api timeout is rising"}
```

## 4. JSON envelope

基础 envelope：

```json
{
  "source": "/repo/src/auth/oauth.py",
  "lines": [42, 98],
  "content": "class OAuthClient:\n    ...",
  "score": 0.87,
  "metadata": {
    "kind": "search",
    "media_type": "text/x-python"
  }
}
```

外部 source envelope：

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "lines": null,
  "content": "Login broken after SSO migration",
  "score": 0.842,
  "locator": {
    "kind": "row",
    "primary_key": {"id": 12}
  },
  "metadata": {
    "kind": "search",
    "source_type": "postgres",
    "source_name": "prod",
    "object_type": "row",
    "media_type": "application/x-ndjson",
    "fields": {
      "status": "open",
      "priority": "high",
      "updated_at": "2026-05-10T12:30:00Z"
    }
  }
}
```

消费者规则：

- 基础消费者读 `source`、`lines`、`content`、`score`、`metadata`。
- 结构化消费者从 `locator` 读精确定位信息。
- `source` 是包含该对象的可读 URI；精确定位放在 `locator`。
- `media_type` 在 `metadata` 里出现，是对象的 MIME 类型。

## 5. `cat` 输出

### 文件

```bash
mfs cat -n 40:45 ./docs/auth.md
```

输出：

```text
40  ## Token expiration
41
42  Access tokens live 15 minutes.
```

### JSONL 区间读取

```bash
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:2
```

输出：

```jsonl
{"id":12,"subject":"Login broken after SSO migration","status":"open"}
{"id":41,"subject":"SSO redirect loop","status":"pending"}
```

`--json`：

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "media_type": "application/x-ndjson",
  "range": {"start": 0, "end": 2, "total_hint": 12453},
  "items": [
    {"id": 12, "subject": "Login broken after SSO migration"},
    {"id": 41, "subject": "SSO redirect loop"}
  ]
}
```

继续读下一段（区间显式表达，**没有 cursor token**）：

```bash
mfs cat postgres://prod/public/tickets/rows.jsonl --range 100:200
```

### 大对象拒绝

`cat` 不接受没有 `--range` 的大对象请求。错误模板见 [16.7](16-concepts-paths-anchors-and-command-decisions.md#7-分页设计)。

### 精确过滤

数据库 row、消息 thread、SaaS record 使用 `grep`、source 支持的 query/filter，或者把对象 `export` 到本地后用 jq / Python / Pandas 处理：

```bash
mfs grep "id\":12" postgres://prod/public/tickets/rows.jsonl
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
jq 'select(.id == 12)' ./tickets.jsonl
```

MFS v0.4 不内置 `mfs jq`；之后若加上，定位为 server-side pushdown 的薄包装，详见 [15-command-inventory-and-scope.md](15-command-inventory-and-scope.md#3-与之前设计的差异)。

## 6. `head` / `tail`

`head` 是简单预览：

```bash
mfs head -n 2 postgres://prod/public/tickets/sample.jsonl
```

`tail` 是查看尾部，适合日志、聊天、事件流、追加型 JSONL：

```bash
mfs tail -n 50 s3://logs/app/2026-05-10
```

`tail -f` 跟随 append-only source：

```bash
mfs tail -f slack://eng/channels/incidents/today/messages.jsonl
mfs tail -f s3://logs/app/today
```

对象无法高效 `tail` 时返回明确错误和替代命令。

## 7. 错误输出

人类文本：

```text
Object is too large for full cat: postgres://prod/public/tickets/rows.jsonl
size_hint: 4.2GiB
try:
  mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
  mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:1000
  mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
```

JSON：

```json
{
  "error": {
    "code": "object_too_large_for_cat",
    "message": "Object is too large for full cat",
    "source": "postgres://prod/public/tickets/rows.jsonl",
    "size_hint": "4.2GiB",
    "suggestions": [
      "mfs head -n 20 postgres://prod/public/tickets/rows.jsonl",
      "mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:1000",
      "mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl"
    ]
  }
}
```

## 8. Pipe 行为

管道体验：

```bash
mfs cat --meta ./docs/auth.md | mfs search "token expiry"
```

行为：

- 如果 stdin 里有 MFS header，`search` 限定到该 source。
- 如果 stdin 是普通文本，`search` 对 stdin 做临时搜索。
- 如果没有 path 且没有 `--all`，无 pipe 时继续报错。

结构化 source 支持 pipe：

```bash
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100 --json \
  | jq '.items[] | select(.priority == "high")'

mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl \
  && jq 'select(.priority == "high")' ./tickets.jsonl
```
