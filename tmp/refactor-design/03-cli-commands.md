# CLI 命令清单与行为契约

MFS 公开 **16 个顶级命令**：11 个 POSIX 风格动词命令 + 5 个名词管理命令。CLI 是 HTTP client，所有重活通过 `/v1` API 转给 local daemon 或 remote server。

## 1. 完整命令清单

### 动词命令（POSIX 风格）

| 命令 | 作用 |
| --- | --- |
| `mfs add <uri>` | 注册并同步本地路径或外部 connector。**幂等**：再跑等于"再同步" |
| `mfs status [<uri>]` | 看 daemon / profile / connector / freshness / job |
| `mfs search <query> <path>` | 语义 + 关键词混合搜索 |
| `mfs grep <pattern> <path>` | 精确搜索，能下推 connector 时下推 |
| `mfs ls <uri>` | 列子节点 |
| `mfs tree <uri>` | 树状浏览 |
| `mfs cat <uri>` | 读取对象；大对象拒绝并提示 head/tail/range/export |
| `mfs head <uri>` | 前 N 行/记录 |
| `mfs tail <uri>` | 后 N 行/记录（v0.4 不支持 `-f` 流式跟随） |
| `mfs export <uri> <file>` | 把对象写到本地文件 |
| `mfs remove <uri>` | 注销 connector + 删 chunks / cache / state（destructive，默认 confirm） |

### 名词管理命令

| 命令 | 子命令 |
| --- | --- |
| `mfs connector` | `add / probe / list / inspect / update / remove` — 注册和管理 connector |
| `mfs profile` | `add / use / list / status` — client endpoint profile |
| `mfs serve` | `start / stop / status / logs` — 本机起一个 server 进程 |
| `mfs job` | `list / inspect / cancel` — 后台任务 |
| `mfs config` | `show / set` — 查看与修改配置 |

### URI 写法约定

所有命令 `<uri>` 参数都是 connector URI。**本地路径是 file scheme URI 的简写**：

| 用户写 | CLI 内部规范化 |
|---|---|
| `./repo` （相对路径） | `file:///<resolved-abs-path>/repo` |
| `/abs/path` （绝对路径） | `file:///abs/path` |
| `file:///abs/path` | 不变（标准 URI 形式） |
| `file://./repo` | ❌ 报错（违反 URI 规范，相对路径不能跟 `file://` 一起用） |
| `postgres://prod` 等 | 不变（按 scheme 路由到对应 connector） |

## 2. 设计原则

- **动词命令跟 POSIX 同名同义**：agent 不学新词。
- **管理类操作集中到名词子树**：`mfs connector list` 不是 `mfs list-connectors`。
- **一个幂等命令搞定就够**：`mfs add` 一个动词承担注册和同步两件事。
- **隐藏复杂度**：单条 issue / row / message 通过 `locator` 表达，不让用户构造伪路径。
- **大对象有 guard**：`cat` 默认拒绝大对象并给替代建议。

## 3. `mfs add` 是 `mfs connector add` 的高频别名

所有"对 connector 的写操作"集中在 `mfs connector` 子树下；`mfs add` 是其中最高频的 `connector add` 的 **alias**（日常用）。其他 connector 操作（probe / list / inspect / update / remove）都直接走 `mfs connector` 子命令。

```bash
# 主入口（显式，所有操作齐全）
mfs connector add <uri> [--config <toml>]      # 注册并同步
mfs connector probe <uri> [--config <toml>]    # 试连接，不写状态
mfs connector list
mfs connector inspect <uri>
mfs connector update <uri> --config <toml>
mfs connector remove <uri>

# 高频简写（仅注册 + 同步走 alias）
mfs add <uri>                                  # alias = mfs connector add <uri>
```

`mfs add` 注册并同步一个 connector。**本地路径是 file scheme URI 的简写**（`./repo` 等价于 `file:///<resolved-abs-path>`，CLI 自动展开）；外部 connector 首次需要 `--config <toml>`。命令是幂等的：再跑一次 = 再同步一次。

```bash
# 本地路径（file connector）
mfs add .
mfs add ./repo
mfs add ./repo --watch                  # 启动 watcher
mfs add ./repo --force                  # 强制重建

# 外部 connector
mfs add postgres://prod --config x.toml             # 首次：注册 + 同步（默认 confirm）
mfs add postgres://prod --config x.toml --yes       # 跳过 confirm
mfs add postgres://prod                             # 已注册：再同步一次
mfs add postgres://prod --force                     # 强制重建
mfs add slack://eng --since 2026-05-01              # 时间游标增量

# 想先试连接、不写状态
mfs connector probe postgres://prod --config x.toml
```

5 个核心 flag：

| flag | 作用 |
|---|---|
| `--config <toml>` | 外部 connector 首次注册必填；已注册时忽略（要改配置用 `mfs connector update`） |
| `--yes` | 跳过 confirm。默认行为：首次注册外部 connector 估算成本后等确认；本地小目录直接跑 |
| `--watch` | 仅本地路径有效，启动 daemon 内 watcher |
| `--force` | 跳过 fingerprint 比对，重建可重建部分 |
| `--since <date>` | 仅时间游标 connector（postgres updated_at / slack ts / github / gmail）有效；其他报 `since_unsupported` |

试连接、注册管理等不在 `mfs add` 上挂 flag，拆到 `mfs connector` 子树（probe / list / inspect / update / remove）。

### 首次注册外部 connector 的默认行为

```text
$ mfs add postgres://prod --config .mfs/connectors/prod-postgres.toml
Connector validated: postgres://prod
Discovered: 38 tables / ~12.4M rows
Estimated sync (based on 1% probe sample, ±50% accuracy):
  embedding: ~2.4M tokens (~$48 at text-embedding-3-small)
  duration:  ~8h on 4 workers
  storage:   ~3.2GB index + cache

Continue? [y/N]
```

`--yes` 或本地路径直接开始：

```text
$ mfs add ./repo
Processing 184 files under /repo
Indexed: 184 files scanned, 37 touched, 2 deleted, 412 chunks queued.
Worker running in background. Run `mfs status` to check progress.
```

### URI 写法

connector URI 是主推风格（跟 DSN / connection string 一致，跟后续 `mfs ls postgres://prod` 风格统一）。脚本场景可以用 `--type / --alias` 的等价写法：

```bash
mfs add postgres://prod --config x.toml
mfs add --type postgres --alias prod --config x.toml      # 等价
```

输出（本地路径）：

```text
Processing 184 files under /repo
Indexed: 184 files scanned, 37 touched, 2 deleted, 412 chunks queued.
Worker running in background. Run `mfs status` to check progress.
```

remote profile（不共享 fs）下处理本地路径**自动走 upload flow**——CLI 端 scan + manifest diff + zip bundle 上传 + commit。默认 confirm：

```text
$ mfs add ./repo                                     # active profile = remote (https://mfs.example.com)
Scanning ./repo ... 184 files, 28 MB
Manifest diff against server: 37 changed, 2 deleted, 145 unchanged
Estimated upload: 8.3 MB (changes only)

Continue? [y/N]
```

`--yes` 跳过 confirm；`--no-upload` 显式拒绝上传（报错而不是发数据）。

upload 完成后 server 跑标准 chunk → embed → 写 Milvus 流程。`mfs status ./repo` 看进度。详见 [02-architecture.md §3.5](02-architecture.md#35-本地文件-upload-flow不共享-fs-场景)。

## 4. Search 行为

```bash
mfs search "session storage" ./src --top-k 5
mfs search "customer cannot login" postgres://prod/public/tickets
mfs search "session" --all                # 跨所有已注册 connector
```

输出（本地）：

```text
[1] src/session/store.py  score=0.884
 82  class SessionStore:
 83      def save(self, session: Session) -> None:

[2] src/auth/session.py  score=0.731
 14  SESSION_COOKIE_NAME = "sid"
```

输出（外部 connector）：

```text
[1] postgres://prod/public/tickets/rows.jsonl  score=0.842
     row: id=12
     subject: Login broken after SSO migration
     status: open
     priority: high
```

- 召回走 Milvus hybrid（dense + sparse + RRF）。
- 必须显式给 `<path>` 或 `--all`，不会默认搜全部。
- 返回结果必含可继续操作的 `source` URI 和 `locator`。

## 5. Grep 行为

```bash
mfs grep "ERR_TOKEN_EXPIRED" .
mfs grep -C 5 "OAuth" ./docs
mfs grep "SSO" postgres://prod/public/tickets/rows.jsonl
mfs grep "timeout" slack://eng/channels/incidents
```

输出按 path/URI 分组（unix grep 风格）：

```text
src/auth/token.py
167  raise TokenExpiredError("ERR_TOKEN_EXPIRED")

slack://eng/channels/incidents/2026-05-10/messages.jsonl
118  {"ts":"1715320060.456","user":"U2","text":"api timeout is rising"}
```

派发规则（详见 [05-browse-and-read.md §6](05-browse-and-read.md#6-grep-的派发)）：

- connector 声明 `grep_pushdown=true` → 下推为 SQL `ILIKE` / Slack search API / S3 Select。
- 有 cache → 扫 cache。
- 否则 connector.read() 流式扫。
- 标记 `indexable=true` 且对象已建 chunk → 走 Milvus BM25 召回。

## 6. 浏览三件套：ls / tree / cat

### ls

```bash
mfs ls postgres://prod/public/tickets
```

```text
TYPE  NAME            MEDIA-TYPE           SIZE      EXTRA
file  schema.json     application/json     2.1 KB
file  rows.jsonl      application/x-ndjson ~1.2 GB   ~12.4M rows (lazy)
```

- 数据从 metadata DB 取，stale 时后台 refresh（详见 [05-browse-and-read.md §1](05-browse-and-read.md#1-ls-与-tree-的后台行为)）。
- `--refresh` 强制刷新。
- 无界目录（slack 几百频道、s3 海量 key）默认截断 100 项 + 提示。

### tree

```bash
mfs tree --peek -L 2 ./docs/
mfs tree slack://eng -L 3
```

- 默认 `-L 2`。
- 大目录单层超过 100 截断。
- 时间分区目录默认时间倒序，只显示最近 30 天。

### cat

```bash
mfs cat ./README.md                                           # 完整读文件
mfs cat ./README.md -n 40:90                                  # 行范围
mfs cat postgres://prod/public/tickets/schema.json            # JSON
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100  # 区间
mfs cat ./docs/diagram.png --meta                             # 看 VLM description
```

- 完整 cat 大对象会被拒绝，提示用 head/tail/range/export。详见 [05-browse-and-read.md §4](05-browse-and-read.md#4-分页与大对象)。

## 7. 密度视图 `--peek / --skim / --deep`

`ls / tree / cat` 支持三档密度，**仅对 document / code / directory 形态生效**：

| 命令 | 用途 | 数据来源 |
|---|---|---|
| `--peek` | 只列名字 / 标题骨架 | metadata DB |
| `--skim` | + 每条 summary 一行 | Milvus 查 `directory_summary` / `summary` / `vlm_description` |
| `--deep` | 展开更多结构 | Milvus + cache head |

```bash
mfs tree --peek -L 2 ./
mfs ls --skim ./docs
mfs cat --skim ./docs/auth.md
```

**对结构化对象**（rows.jsonl / messages.jsonl / records.jsonl / schema.json）传 `--peek/--skim/--deep` 直接报错：

```text
density view not supported for application/x-ndjson
use head/tail/cat --range instead:
  mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
```

错误码 `density_unsupported`。理由：head/tail 已经完整覆盖结构化对象的预览需求，密度视图重复造轮子。

W/H/D 参数同样规则。

## 8. head / tail / export

```bash
mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
mfs tail -n 50 s3://logs/app/2026-05-10.jsonl
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
```

- `head -n N` / `tail -n N` 无状态。
- v0.4 **不支持 `-f` 流式跟随**——流式跟随需要每个 connector 单独实现 push/poll 通道，工程成本高、受益场景窄。需要监控类用例可以脚本化 `mfs add <uri>` 周期同步 + `mfs head -n N` 看快照。
- `export` 把对象完整写到本地文件——大对象遍历的标准做法。

## 9. Status 是统一状态入口

```bash
mfs status                              # 总览（local / remote profile 都支持）
mfs status postgres://prod              # 单 connector 详情
mfs status --verbose postgres://prod    # 含 retrieval index 细节
mfs status --diagnose                   # 自检 profile/connector/storage/search
mfs status --watch                      # 列正在 watch 的目录（仅 local profile）
```

`--watch` 只对 local profile 有意义；remote profile 下执行返回 `watch_unsupported_on_remote`（远端 server 不 watch 本地路径）。

样例输出：

```text
$ mfs status
Profile: local (is_local=true, machine-id matched)
Daemon:  running (pid=4112, port=8765, version=0.4.0)
Connectors: 3 active
  ./repo                    last_add=2026-05-14T09:21:00Z   index=fresh
  postgres://prod           last_add=2026-05-14T07:00:00Z   index=stale (3 tables changed)
  slack://eng               last_add=2026-05-13T22:00:00Z   index=fresh
Jobs:    1 running, 0 failed
Search:  available
```

健康检查、watch 状态、诊断都收敛到 `status` 这一个命令。

## 10. Connector / Profile / Daemon / Job 管理

### `mfs connector`（所有 connector 操作的主入口）

```bash
# 注册并同步（也可用高频简写 mfs add）
mfs connector add postgres://prod --config .mfs/connectors/prod-postgres.toml
mfs connector add ./repo                                                   # 本地，CLI 自动转 file://

# 试连接，不写任何状态
mfs connector probe postgres://prod --config .mfs/connectors/prod-postgres.toml

# 看 / 改 / 删
mfs connector list
mfs connector inspect postgres://prod                                      # 配置、能力声明、暴露的对象
mfs connector update postgres://prod --config .mfs/connectors/prod-postgres.toml
mfs connector remove postgres://prod
```

`mfs add <uri>` 是 `mfs connector add <uri>` 的 alias（日常用）。其他子命令没有 alias，因为使用频率低，直接走显式子命令更清楚。

### `mfs remove`（destructive，默认 confirm）

`mfs remove <uri>` 和 `mfs connector remove <uri>` 等价——注销 connector 并清理一切。

```bash
mfs remove postgres://prod
mfs remove ./repo
mfs remove postgres://prod --yes        # 跳过 confirm（脚本场景）
```

默认 confirm 列出将删的内容：

```text
$ mfs remove postgres://prod
This will permanently delete:
  - 12,453 chunks in Milvus
  - 3.2 GB cache in object store
  - 38 indexed objects
  - 1 running sync job (will be cancelled)

Continue? [y/N]
```

confirm 后流程：取消正在跑的 sync（如有）→ drop_partition + 清 cache + 删 metadata → 注销 connector。详细并发协调见 [02-architecture.md §5.11](02-architecture.md#511-操作之间的并发协调)。

幂等性：

- 对同一 connector 重复 `mfs remove` 返回 `already removing, see job <id>`（不重新触发清理）
- 对不存在的 connector 返回 `not registered`（不报错）

### `mfs profile`

```bash
mfs profile add local --url http://127.0.0.1:8765
mfs profile add prod  --url https://mfs.example.com --token-env MFS_TOKEN_PROD
mfs profile use local
mfs profile list
mfs profile status
```

`kind` 字段决定 client/server 是否共享文件系统命名空间，详见 [02-architecture.md §2](02-architecture.md#2-profile-与存储后端是正交的)。

### `mfs serve`

```bash
mfs serve start
mfs serve stop
mfs serve status
mfs serve logs
```

`mfs serve` 是 client-side 封装，本质是本机 spawn 一个 `mfs-server` 进程。服务端运维直接用 `mfs-server` binary（systemd / docker entrypoint），详见 [02-architecture.md §5](02-architecture.md#5-server-端启动).

如果只装了 `mfs` CLI 没装 `mfs-server`：

```text
mfs serve requires mfs-server package.
Install it with:
  uv tool install mfs-server
```

### `mfs job`

```bash
mfs job list [--failed]
mfs job inspect job_01HX...
mfs job cancel job_01HX...
```

失败时 `mfs add <uri>` 即可（幂等），所以不提供 `job retry`。

## 11. `--json` envelope

每个动词命令都支持 `--json`。统一 envelope：

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
    "chunk_kind": "row_text",
    "connector_type": "postgres",
    "media_type": "application/x-ndjson",
    "fields": {
      "status": "open",
      "priority": "high"
    }
  }
}
```

字段：

| 字段 | 含义 |
|---|---|
| `source` | 包含该结果的 object URI（可继续 `mfs cat`） |
| `lines` | 文本对象时的 line range `[start, end]`，否则 null |
| `content` | 召回 / 读取的文本 |
| `score` | search/grep 才有；ls/cat 为 null |
| `locator` | 容器内单元定位；schema per connector（见 [05-search §3](06-search-and-retrieval.md#3-locator-schema-per-connector)） |
| `metadata` | 包含 `kind` (search/cat/...) / `chunk_kind` / `connector_type` / `media_type` / connector-specific `fields` |

`cat --range A:B --json` 返回 `items` 数组 + `range` 信息：

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

## 12. 错误输出

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

稳定错误码：

| code | 含义 |
|---|---|
| `upload_rejected` | 用户显式 `--no-upload` 但本地路径 + remote profile 触发了 upload |
| `upload_bundle_too_large` | 单 bundle 超过 `max_bundle_size_mb` 阈值，建议过滤范围或拆分 |
| `object_too_large_for_cat` | cat 大对象未带 `--range` |
| `is_directory` | 对目录 cat |
| `connector_unhealthy` | connector healthcheck 失败 |
| `density_unsupported` | 对结构化对象用 `--peek/--skim/--deep` |
| `chunk_max_exceeded` | 该对象超过 `chunk_max`，部分索引 |
| `local_server_unavailable` | local profile 但本机 server 进程不可达 |
| `field_missing` | connector 数据缺 text_fields 配置的字段 |
| `since_unsupported` | 给不支持时间游标的 connector 传 `--since` |
| `watch_unsupported_on_remote` | remote profile 下用 `mfs status --watch` |
| `sync_already_running` | 同一 connector 已有 in-flight sync；返回 `see job <id>`，提示 `mfs job cancel` |
| `connector_removing` | connector 正在被 remove，拒绝新 add/sync；让用户等清理完成 |
| `op_conflict` | 通用并发拒绝（如 sync 中又来 update_config）；error 字段说明拒绝原因和当前 op |

## 13. Pipe 行为

**Pipe 是普通 unix 字节流**——MFS 不在 stdin/stdout 上发明私有协议，不识别"上游来自哪个 source"。这样每个新 connector 不需要做 pipe 元数据适配。

规则：

- 上游 `mfs cat / head / tail / grep / search` 输出**纯字节流**（默认）或 JSON（`--json`），没有 MFS header。
- `mfs search` / `mfs grep` 读 stdin 时**总是把 stdin 当临时文本处理**，对它做搜索/grep。
- 想限定到具体 source 就**传 path 参数**：`mfs search "..." <path>`，不要通过 pipe 表达。
- 无 path 且无 `--all` 且无 stdin：报错。

典型用法：

```bash
# 临时搜索 stdin 文本
git log --oneline | mfs search "fix auth"

# 大对象切片后用 jq
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100 --json \
  | jq '.items[] | select(.priority == "high")'

# 大对象先导出再处理
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl \
  && jq 'select(.priority == "high")' ./tickets.jsonl

# 限定 source 直接用 path 参数，不用 pipe
mfs search "token expiry" ./docs/auth.md
```
