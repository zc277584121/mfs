# CLI 行为设计

## 1. 本地 CLI 入口

本地文件用普通 path：

```bash
mfs add .
mfs status
mfs search "oauth callback flow" ./docs --top-k 5
mfs grep -C 5 "OAuth" ./docs
mfs tree --peek -L 2 .
mfs ls --skim ./docs
mfs cat --skim ./docs/auth.md
mfs cat -n 40:90 ./docs/auth.md
```

CLI 行为规则：

- `mfs add <path-or-uri>` 是**统一注册 + 同步入口**，幂等。
- `mfs search <query> <path>` 仍然要求显式 path 或 `--all`。
- `mfs grep <pattern> <path>` 仍然要求显式 path 或 `--all`。
- `ls/tree/cat` 支持 `--peek/--skim/--deep`。
- `cat -n start:end` 支持本地文本的行范围（保持 v0.3 习惯）；外部对象用 `cat --range start:end` 表达 record 区间。
- `--json` 使用统一 envelope。
- pipe-aware：`mfs cat ... | mfs search ...` 可以限定到 header source；普通 stdin 文本只搜索 stdin 本身。

本地目录在 server 侧表示为 file source，执行位置是 local daemon。CLI 负责参数、HTTP、profile 和输出；扫描文件、生成 MFS Index、写 Retrieval Index 都由 daemon 执行。

## 2. 本地路径和外部 URI 共用一套命令

用户继续传真实路径：

```bash
mfs search "queue retry" .
mfs cat ./src/mfs/cli.py
```

外部 source 用 URI：

```bash
mfs search "queue retry" github://mfs/src
mfs cat github://mfs/src/mfs/cli.py
```

规则：

- 相对路径和绝对本地路径按 CLI 工作目录解析。
- 外部 source 用 `postgres://prod/...`、`slack://eng/...`、`github://mfs/...` 这类 URI。
- 对象名带 media type 后缀（`schema.json` / `sample.jsonl` 等）；`--json` 输出同时携带 `media_type`。
- 本地路径日常直接写普通 path；JSON 输出、HTTP API 和审计日志内部可用规范化 file URI。
- 搜索结果优先显示用户输入同一风格的路径；跨 source 搜索显示 source URI。
- 普通本地路径由 local profile + daemon 执行；remote profile 下收到本地 path 时返回 `remote_server_cannot_read_local_path`。

## 3. `mfs add` 是统一入口

### 3.1 本地路径

```bash
mfs add .
```

输出：

```text
Processing 184 files under /repo
Indexed: 184 files scanned, 37 touched, 2 deleted, 412 chunks queued.
Worker running in background. Run `mfs status` to check progress.
```

行为：

- CLI 调 local daemon 的 HTTP API（`POST /v1/add`）。
- local daemon 扫描本地目录。
- daemon 更新 file source、Index Artifacts、Retrieval Index。
- 支持 `--force`、`--sync`、`--watch`。

remote profile 下执行 `mfs add .` 返回：

```text
Local path requires local profile: .
Run `mfs profile use <local>` and `mfs daemon start` for local files.
```

### 3.2 外部 source

首次注册：

```bash
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
```

输出：

```text
Added source: postgres://prod
type: postgres
health: ok
objects:
  postgres://prod/public/tickets/schema.json
  postgres://prod/public/tickets/sample.jsonl
  postgres://prod/public/tickets/rows.jsonl
Sync started.
job: job_01HX...
```

已注册再同步：

```bash
mfs add postgres://prod
mfs add postgres://prod --force
mfs add slack://eng --since 2026-05-01
```

试连不写入：

```bash
mfs add postgres://prod --probe --config .mfs/sources/prod-postgres.toml
```

`mfs add` 是幂等的：

- 首次：校验配置 + 注册 source + 触发同步。
- 已注册：等价于"再同步一次"。
- `--config` 在已注册时被忽略；要更新配置用 `mfs source update <uri> --config <toml>`。

**不再有 `mfs source add` 和 `mfs sync` 顶级命令**。详情见 [15-command-inventory-and-scope.md](15-command-inventory-and-scope.md)。

source/connector 在 server 侧执行：local profile 下在本机 daemon，remote profile 下在远端 server/worker。

### 3.3 本地文件高级配置

本地文件主入口仍然是普通 path：

```bash
mfs add .
mfs status .
```

ignore、watch、index limits 等高级配置放到 workspace TOML：

```toml
[file_index]
include = ["src/**", "docs/**"]
exclude = ["**/.venv/**", "**/node_modules/**"]
summary = "auto"
```

`file://...` 只出现在 JSON 输出、HTTP API 和审计日志这类需要规范化地址的场景。

## 4. Search 行为

### 4.1 本地文件搜索

```bash
mfs search "session storage" ./src --top-k 5
```

输出：

```text
[1] src/session/store.py  score=0.884
 82  class SessionStore:
 83      def save(self, session: Session) -> None:

[2] src/auth/session.py  score=0.731
 14  SESSION_COOKIE_NAME = "sid"
```

### 4.2 外部 source 搜索

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
     subject: SSO redirect loop for enterprise customer
     status: pending
     priority: medium
```

行为：

- `search` 查的是 MFS Index 里的 Retrieval Index。
- 结构化 source 的 embedding 字段来自 source TOML 里的搜索配置。
- 返回结果必须给出可继续操作的 URI 和 locator；agent 根据 Skill 和任务目标判断下一步。

## 5. Grep 行为

```bash
mfs grep "ERR_TOKEN_EXPIRED" .
```

输出：

```text
src/auth/token.py
167  raise TokenExpiredError("ERR_TOKEN_EXPIRED")
```

多源示例：

```bash
mfs grep "timeout" slack://eng/channels/incidents
```

输出：

```text
slack://eng/channels/incidents/2026-05-10/messages.jsonl
118  {"ts":"1715320060.456","user":"U2","text":"api timeout is rising","thread_ts":"1715320000.123"}
```

行为：

- 能下推的 source 用 provider search、SQL 或全文索引。
- 不能下推时用缓存或分页扫描。
- 输出按 path/URI 分组，行号位置可以是 JSONL record number、message index 或文本行号。

## 6. 结构化和大对象命令

`ls/tree/cat` 已经覆盖文件浏览。结构化和大对象额外命令：

```bash
mfs head -n 20 <path-or-uri>
mfs tail -n 50 <path-or-uri>
mfs tail -f <append-only-uri>
mfs cat <uri> --range A:B
mfs export <path-or-uri> <local-file>
```

**没有 cursor token**。分页用 `--range` 表达。详细分页设计见 [16-concepts-paths-anchors-and-command-decisions.md](16-concepts-paths-anchors-and-command-decisions.md#7-分页设计)。

## 7. `--json` 输出

基础 JSON envelope：

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

外部 source 使用同一 envelope，并增加结构化定位字段：

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
    "media_type": "application/x-ndjson",
    "fields": {
      "status": "open",
      "priority": "high"
    }
  }
}
```

基础消费者读取 `source/content/score/metadata`；结构化消费者读取 `locator` 和更完整的 `metadata`。
