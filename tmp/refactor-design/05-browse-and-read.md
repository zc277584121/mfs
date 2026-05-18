# 浏览与读取

本文回答：`ls / tree / cat / head / tail / grep` 这六个命令的后台行为，cache 怎么用，大对象怎么处理。

## 1. ls 与 tree 的后台行为

```
mfs ls postgres://prod/public/tickets
  │
  ├─ 1. 查 metadata DB:
  │     SELECT virtual_path, media_type, size_hint, last_seen, fingerprint, extra
  │     FROM objects
  │     WHERE connector_id = $cid AND parent_path = '/public/tickets'
  │     ORDER BY virtual_path
  │
  ├─ 2. 如果 records 的 last_seen 超过 TTL（默认 1h）：
  │       触发后台 connector.list($path) 刷新（不阻塞当前请求）
  │       当前请求返回 cached 结果，附 "(may be stale)" 提示
  │
  └─ 3. 渲染输出
```

**所有 ls/tree 都走 metadata DB cache**，不直接打回 connector。metadata DB 就是虚拟文件系统的 path index——这是它必须存在的根本理由。

`--refresh` 强制同步刷新后再列：

```bash
mfs ls --refresh slack://eng/channels
```

`tree` 是递归 ls，同样机制。

## 2. ls 的输出格式

```text
$ mfs ls postgres://prod/public/tickets
TYPE  NAME            MEDIA-TYPE           SIZE        EXTRA
file  schema.json     application/json     2.1 KB
file  rows.jsonl      application/x-ndjson ~1.2 GB     ~12.4M rows (lazy)

$ mfs ls slack://eng/channels/incidents/2026-05-10
TYPE  NAME            MEDIA-TYPE           SIZE        EXTRA
file  messages.jsonl  application/x-ndjson 48 KB       342 messages
file  threads.jsonl   application/x-ndjson 12 KB       18 threads
dir   files/                                            7 attachments

$ mfs ls ./repo
TYPE  NAME            MEDIA-TYPE           SIZE
dir   docs/
dir   src/
file  README.md       text/markdown        4.2 KB
file  pyproject.toml  application/toml     1.1 KB
file  LICENSE         text/plain           11 KB
```

字段：

| 字段 | 含义 |
|---|---|
| TYPE | `file` / `dir` |
| NAME | 节点名（带 media type 后缀） |
| MEDIA-TYPE | MIME |
| SIZE | 字节数或带 `~` 的估算 |
| EXTRA | connector-specific hint（行数、记录数、`lazy` 等） |

`--json` 输出包含完整 `objects` 行：

```json
[
  {
    "type": "file",
    "name": "rows.jsonl",
    "path": "postgres://prod/public/tickets/rows.jsonl",
    "media_type": "application/x-ndjson",
    "size_hint": 1288490188,
    "lazy": true,
    "extra": {"row_count_hint": 12453000},
    "fingerprint": "abc123",
    "last_seen": "2026-05-15T09:21:00Z",
    "indexable": true,
    "capabilities": {
      "cat": "denied_unless_range",
      "grep": "pushdown",
      "tail": false,
      "range": true
    }
  }
]
```

`capabilities` 告诉 agent 这个对象能用哪些命令——避免 agent 试错。

## 3. tree 的无界处理

slack/discord/gmail 这种按日期递归很容易爆炸（365 天 × 100 频道）。规则：

- **默认 `-L 2`**，不是 unlimited。
- **每层最多 100 项**，超过显示 `... (N more, narrow with <path>)`。
- **时间分区目录**默认时间倒序，只展开最近 30 天。
- 用户加 `--limit N` 调整单层上限，`-L N` 调整深度。

```text
$ mfs tree slack://eng -L 3
slack://eng
├── channels/
│   ├── general__C01/
│   │   ├── 2026-05-15/  (today)
│   │   ├── 2026-05-14/
│   │   └── ... (28 more days, narrow with <path>)
│   ├── incidents__C02/
│   │   └── ... (similar)
│   └── ... (97 more channels, narrow with <path>)
├── dms/
└── users.jsonl
```

## 4. 分页与大对象

### 4.1 不用 cursor token

理由：

- cursor 是 stateful 复杂度，agent 难管，token 过期 / 兼容性问题多。
- 真要遍历大对象用 `mfs export` 物化到本地再处理。
- 大对象过滤用 `mfs grep`（server-side pushdown），不需要全量拉回。
- 增量数据走"周期 `mfs add <uri>` + `mfs head -n N` 看快照"，v0.4 不做流式跟随。
- DB query 结果不稳定靠 `export` 物化解决。

### 4.2 统一接口

```bash
mfs head -n N <uri>          # 固定前 N 行/记录，无状态
mfs tail -n N <uri>          # 固定后 N 行/记录，无状态
mfs cat <uri>                # 完整对象；大对象拒绝并提示
mfs cat <uri> --range A:B    # 按行/记录区间读取
mfs export <uri> <file>      # 完整导出到本地
```

职责完全不重叠：

- `head/tail` 只看端点，不带范围。
- `cat` 默认完整；`--range A:B` 取闭开区间。

### 4.3 `--range A:B` 单位

| 对象 media_type | `--range A:B` 单位 |
|---|---|
| `text/*` / `application/x-*-source-code` | 行号 |
| `application/x-ndjson` / `text/jsonl` | record 索引 |
| `text/csv` | row 索引（不含 header） |
| `application/pdf` / `application/x-pdf` | 页码 |
| `application/octet-stream` / `image/*` | 不支持，报 `range_unsupported` |

省略一边：`--range 100:` = 从 100 到末尾；`--range :100` = 0 到 100。

### 4.4 大对象拒绝

`cat` 不接受没有 `--range` 的大对象请求。阈值由 server 端 `server.toml` 控制：

```toml
[cat]
max_full_size = "10MiB"
max_full_records = 10000
```

错误模板：

```text
Object is too large for full cat: postgres://prod/public/tickets/rows.jsonl
size_hint: 4.2GiB
try:
  mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
  mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:1000
  mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
```

错误码 `object_too_large_for_cat`。

## 5. cat / head / tail 的数据流

```
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100
  │
  ├─ 1. 查 metadata DB：该 object 是否有 cache？
  │
  ├─ 2a. 有 cache && fresh（fingerprint 一致）：
  │       从 object store 读 cache bytes（按 range 切片）→ 流回 client
  │
  ├─ 2b. 有 cache && stale：
  │       异步触发 cache rebuild
  │       本次仍用 stale cache（附 `(stale)` 提示）
  │
  ├─ 2c. 无 cache：
  │       connector.read(path, range) → 流回 client
  │       同时写 cache（如该 object 的 cache_kind 配置允许）
  │
  └─ 3. 按 media_type 渲染输出
```

cache 不是必须——有些对象（小文件、纯文本）不值得 cache，每次现拉即可。每类 connector 在 `objects` 表里标记每个对象是否要 cache。

## 6. grep 的派发

```
mfs grep "ERR_TIMEOUT" <path>
  │
  ├─ 1. 解析 path → 拿到 object 元信息 + capabilities
  │
  ├─ 2a. capabilities.grep == "pushdown"：
  │       connector.grep(pattern, path) → 流式 yield matches（重写了下推实现）
  │       例：postgres → SQL ILIKE；slack → search.messages；s3 → S3 Select
  │
  ├─ 2b. object 在 Milvus 里（chunks 已建索引）且 --mode index：
  │       Milvus sparse_vec BM25 召回 → 返回带行号的 chunk
  │
  ├─ 2c. object 在 cache 里：
  │       线性扫 cache 字节
  │
  └─ 2d. 否则：
        connector.read() 流式扫描 + 限速
        超过 `max_grep_bytes` 时截断并提示
```

输出按 path/URI 分组（unix grep 风格）：

```text
$ mfs grep "ERR_TIMEOUT" s3://logs/app/2026-05-10
s3://logs/app/2026-05-10/app.jsonl
8842  {"level":"error","code":"ERR_TIMEOUT","request_id":"r_123"}
9105  {"level":"error","code":"ERR_TIMEOUT","request_id":"r_456"}
```

`-C N` 上下文行数；`-i` 忽略大小写；`-w` 整词；`-E` 扩展正则。

下推与否对用户透明：用户只用 `mfs grep`，框架根据 connector 能力派发。`mfs status --verbose <uri>` 可看到该对象的 grep 实现路径。

`mfs grep` 默认是字面精确匹配（符合 unix 习惯）；只有 `--mode index` 才走 Milvus BM25 召回。

## 7. cat 对非文本对象的渲染

cat 按 media_type 决定渲染方式：

| media_type | cat 默认行为 |
|---|---|
| `text/*` | 原文 |
| `application/json` | pretty print（缩进 2 空格） |
| `application/x-ndjson` | 原文（每行一个 JSON） |
| `text/csv` | 表格对齐渲染；`--raw` 出原 CSV |
| `application/pdf` | converted markdown（从 cache 取） |
| `application/vnd.openxmlformats-...` (docx) | converted markdown |
| `image/*` | 提示 `<binary image, 1.2MB>` + cache 中的 VLM description；`--raw` 输出 bytes |
| 其他 binary | 提示 `<binary, X bytes>`；`--raw` 输出 bytes |

`--raw` 强制原始字节。`--meta` 输出 metadata + 缩略 preview。`--json` 走 envelope。

## 8. 密度视图的适用范围

`--peek / --skim / --deep` 和 W/H/D 参数**只对 document / code / directory 形态生效**：

| 命令 | 适用对象 | 行为 |
|---|---|---|
| `cat --peek/--skim/--deep` | document / code | heading / symbol skeleton；段落首句；全文展开 |
| `ls --peek/--skim` | directory | 名称列表；+ 每条 summary |
| `tree --peek` | directory | skeleton 树 |

数据来源：

- `--peek`: metadata DB（无需 Milvus）。
- `--skim`: Milvus 查该 path 下的 `directory_summary` / `summary` / `vlm_description` chunk；没有则降级到 `--peek`。
- `--deep`: Milvus + 取 cache head。

**对结构化对象**（rows.jsonl / messages.jsonl / records.jsonl / schema.json / sample / page_cache）传 `--peek/--skim/--deep` 直接报错：

```text
density view not supported for application/x-ndjson
use head/tail/cat --range instead:
  mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
  mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:50
```

错误码 `density_unsupported`。理由：head/tail/range 已经完整覆盖结构化对象的预览需求；密度视图重复造轮子且语义模糊。

W/H/D 参数同样规则。

## 9. v0.4 不支持流式跟随 (`tail -f`)

`tail -f` 不在 v0.4 范围。理由：

- 实现一个真正有用的 `tail -f` 需要每个 connector 单独搭 push/poll 通道（slack events / discord WS / fs watcher / s3 list polling / DB CDC ...），工程成本高
- 受益场景窄——日志监控、聊天跟随只是少数场景
- 退化为"周期轮询"的伪流式没多大价值

替代做法：

```bash
# 周期同步 + 看快照
mfs add slack://eng                              # 触发增量同步
mfs head -n 50 slack://eng/.../messages.jsonl    # 看最新一批

# 用 cron / watch 命令自己包一层
watch -n 30 'mfs add slack://eng && mfs head -n 50 slack://eng/...'
```

之后视用户呼声决定是否在 v0.5+ 引入。届时优先支持 file connector 和 slack/discord 这种自带 push 的源。

## 10. Cache 层细节

### 10.1 `caches` 表 schema

```sql
caches (
  object_uri       TEXT,
  cache_kind       TEXT,            -- "converted_md" | "page_cache" | "head_cache" | "vlm_text" | "schema_dump"
  storage_path     TEXT,            -- ~/.mfs/cache/<sha1(object_uri)>/<cache_kind>
  fingerprint      TEXT,            -- 同上游 fingerprint，用于 stale check
  size_bytes       INTEGER,
  built_at         TIMESTAMP,
  PRIMARY KEY (object_uri, cache_kind)
)
```

### 10.2 几种 cache 类型

| cache_kind | 来源 | 谁用 |
|---|---|---|
| `converted_md` | PDF / DOCX / gdoc / HTML 转 markdown | `cat` 直接出 / chunker 输入 |
| `page_cache` | DB rows / Slack messages / S3 list | `cat / head / tail / grep` |
| `head_cache` | head N 的预拉取（如 DB 表前 100 行） | `head` 命中快路径；不暴露给用户 |
| `vlm_text` | 图片的 VLM description | `cat --meta` / `cat --skim` |
| `schema_dump` | DB schema / Mongo sample-inferred schema | `cat schema.json` |

### 10.3 何时 cache、何时不 cache

每个 connector plugin 决定。一般规则：

- **真实文件**（本地文件、GitHub blob、S3 object）→ 不 cache，每次 connector.read() 直接拉（要么 fast，要么需要凭据隔离）。
- **MFS 生成的虚拟对象**（schema.json / rows.jsonl 的 head / messages.jsonl）→ cache。
- **大对象 lazy 模式**（rows.jsonl）→ 不全量 cache；用户 cat --range 时局部拉，可选写局部 cache。
- **图片 VLM**→ cache description 文本（不 cache 图片本身）。
- **PDF/DOCX/HTML 转 markdown**→ cache markdown（原文件还在 source）。

### 10.4 cache 淘汰

server 端 `server.toml`：

```toml
[cache]
max_size_gb = 10
eviction = "lru"
```

超出时按 LRU 淘汰；fingerprint 变化时立即失效。

## 11. Pipe 与 stdin

**Pipe 是普通 unix 字节流**——MFS 不在 stdin/stdout 上发明私有协议，不识别"上游来自哪个 connector"。这样每个新 connector 不需要做 pipe 元数据适配，命令简单。

规则：

- 上游 `mfs cat / head / tail / grep / search` 输出**纯字节流**（默认）或 JSON（`--json`），没有 MFS header。
- `mfs search` / `mfs grep` 读 stdin 时**总是把 stdin 当临时文本处理**。
- 想限定到具体 connector 或对象，**传 path 参数**：`mfs search "..." <path>`。
- 无 path 且无 `--all` 且无 stdin：报错。

示例：

```bash
# 临时搜索 stdin 文本
git log --oneline | mfs search "fix auth"

# 大对象切片后 pipe 到 jq
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100 --json | jq '...'

# 限定 connector / 对象，用 path 参数（不要用 pipe 传 source 信息）
mfs search "token expiry" ./docs/auth.md
```

## 12. 端到端示例

### 场景：在 Postgres 大表里找特定 ticket

```bash
# 1. 先看表结构
mfs cat postgres://prod/public/tickets/schema.json

# 2. 看看数据长什么样
mfs head -n 5 postgres://prod/public/tickets/rows.jsonl

# 3. 语义搜索
mfs search "customer cannot login after SSO" postgres://prod/public/tickets --top-k 5
# → 返回 id=12 / id=41 等候选

# 4. 精确读单条
mfs grep '"id":12' postgres://prod/public/tickets/rows.jsonl

# 5. 想离线分析全部 high priority
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
jq 'select(.priority == "high")' ./tickets.jsonl | wc -l
```

### 场景：周期跟随今天的 incidents 频道

```bash
mfs tree slack://eng/channels -L 1                # 看有哪些频道
mfs ls slack://eng/channels/incidents__C02         # 看分了哪些日期

# 周期同步 + 看最新（v0.4 不内置 tail -f）
watch -n 60 'mfs add slack://eng && mfs head -n 20 slack://eng/channels/incidents__C02/today/messages.jsonl'
```

### 场景：在 ./repo 里找 ERR_TOKEN_EXPIRED 怎么处理

```bash
mfs grep "ERR_TOKEN_EXPIRED" ./repo
# src/auth/token.py
# 167  raise TokenExpiredError("ERR_TOKEN_EXPIRED")

mfs cat ./repo/src/auth/token.py -n 150:180
```
