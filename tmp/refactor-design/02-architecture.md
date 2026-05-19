# 架构

这一篇讲 MFS 的整体架构、客户端跟服务端的分工、数据怎么流、存储怎么放、任务队列怎么跑、多租户怎么演进。看完知道整个系统长什么样。

实施细节（部署形态、发版、工程目录、运维指标）在 [10-packaging-and-deployment.md](10-packaging-and-deployment.md)。

## 1. 整体长什么样

```
┌─────────────────── Client ──────────────────┐
│  mfs CLI / Python SDK / TS·Go·Java SDK       │
│                                              │
│   parse args · 解析 profile · HTTP transport │
└──────────────────────┬───────────────────────┘
                       │  HTTP /v1（主要是 control plane）
                       ▼
┌─────────────────── Server ──────────────────┐
│  API routes                                  │
│    /v1/add  /v1/objects/*  /v1/search        │
│    /v1/grep /v1/connectors/* /v1/jobs/*      │
│       │                                      │
│       ▼                                      │
│  Engine（业务编排：路由 → 起 job → 调插件）  │
│       │                                      │
│       ▼                                      │
│  Connectors                                  │
│    file / web / postgres / slack / github /…│
│    实现 list / stat / read / fingerprint /   │
│    sync 等契约                               │
│       │                                      │
│       ▼                                      │
│  Object Processors（按 object_kind 加工）   │
│    document / code / table_rows /            │
│    message_stream / record_collection /      │
│    image / binary                            │
│       │                                      │
│       ▼                                      │
│  Common Services                             │
│    embedding · summary · VLM · retrieval     │
│       │                                      │
│       ▼                                      │
│  DB 任务队列 + Worker pool                  │
│       │                                      │
│       ▼                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Metadata │  │  Object  │  │  Milvus  │  │
│  │   DB     │  │  store   │  │          │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────┘
```

CLI 跟 server 走 HTTP，互相不 import。server 端按层组织：API → Engine 编排 → Connector 拉数据 → Processor 加工 → Common Service 提供 embedding/VLM/retrieval → 三套存储落地。

## 2. 术语

MFS 的术语按受众分三组，避免一次塞太多。

### 用户和开发者都用（6 个）

| 术语 | 是什么 |
|---|---|
| **Connector** | 一个已注册的数据源实例（`postgres://prod` / `./repo`） |
| **Object** | connector 暴露的一条虚拟文件（URI + media_type） |
| └─ **Cache** | object 派生：字节缓存，可选，让 cat/head/tail 不打回 connector |
| └─ **Chunks** | object 派生：Milvus 行，被 search/grep 召回 |
| **Profile** | client 端的 endpoint 配置（连哪个 server + token） |
| **Job** | 用户一次操作的记录：`mfs add` / `mfs remove` 各产生一个 job |

关系：一个 Connector 暴露多个 Object；每个 Object 可能有 Cache 和 Chunks。Profile 决定 client 连哪个 server。用户每次操作创建一个 Job。

### 后台开发者额外用（2 个）

| 术语 | 是什么 |
|---|---|
| **Task** | Job 内的子单元，每个变化的 object 一个 task |
| **Worker** | 从 DB 拉 task 跑的 coroutine |

> DB 表（`connector_jobs` + `object_tasks`）起到队列容器的作用——文中说"DB 队列"是描述，不把 Queue 当独立术语。

### 读 server 代码时额外用（6 个）

| 层名 | 职责 |
|---|---|
| **HTTP API** | FastAPI routes（`/v1/...`） |
| **Engine** | 业务编排：路由请求 / 起 job / 调 connector |
| **Connectors** | 各数据源的插件 |
| **Object Processors** | 各 object_kind 的加工逻辑 |
| **Common Services** | 通用工具（embedding / summary / VLM / retrieval / export） |
| **Storage** | metadata DB / object store / Milvus 三套后端 |

整个系统里还有一个不对外的内部主键 `namespace_id`，所有顶层表 + Milvus partition + object store prefix 都按它切。v0.4 单一 `default` namespace，用户感知不到；多租户演进见 [§9](#9-多租户与-namespace)。

## 3. Client 和 Server

CLI、SDK 是 client，所有重活在 server。server 有两种部署位置：

| 部署 | 用途 |
|---|---|
| 本机 server（`mfs serve` 起一个 `mfs-server` 进程） | 个人本机，CLI 跟 server 共享文件系统 |
| 远端 server | 团队/云端，CLI 通过 HTTPS 访问 |

### 3.1 配置文件

两个文件分别属于 client 和 server：

| 文件 | 路径 | 内容 | 谁读 |
|---|---|---|---|
| client 配置 | `~/.mfs/client.toml` | profiles、endpoint URL、API token | `mfs` CLI |
| server 配置 | `~/.mfs/server.toml`（本机）<br>`/etc/mfs/server.toml`（远端） | 存储后端、worker、embedding、cache 等 | `mfs-server` |

只装 CLI 的用户只接触 `client.toml`；只装 server 的运维只接触 `server.toml`。

`server.toml` 查找顺序：`--config` → `$MFS_SERVER_CONFIG` → `./server.toml` → `~/.mfs/server.toml` → `/etc/mfs/server.toml` → 内置默认值。

### 3.2 Profile 配置

```toml
# ~/.mfs/client.toml
[client]
default_profile = "local"
client_id       = "01HX...XYZ"      # 首次启动自动生成的 UUIDv7

[[profiles]]
name = "local"
api_base_url = "http://127.0.0.1:8765"

[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_API_TOKEN"
```

`client_id` 是这台 client 的稳定身份（下面 §3.4 详讲）。profile 里**不需要写 `kind`**，CLI 自动判断 client 和 server 是否共享 fs。

profile 管理：

```bash
mfs profile add <name> --url <url>
mfs profile use <name>
mfs profile list
mfs profile status
```

### 3.3 自动判定是否共享 fs

CLI 握手时用 machine-id 比对，决定走"server 直接读本机"还是"upload 流"：

```python
client_machine_id = read_machine_id()
server_resp = await client.get(profile.url + "/v1/server/info")
profile.is_local = (client_machine_id == server_resp["machine_id"])
```

| 场景 | machine-id 比对 | 结果 |
|---|---|---|
| 同机 `mfs serve start` | 一致 | local（server 直接读本机） |
| 远端 HTTPS server | 不一致 | remote（走 upload） |
| Docker / WSL2 / SSH forward | 不一致 | remote |

调试时可以 `MFS_FORCE_REMOTE=1` 强制 remote。

### 3.4 client_id：client 的稳定身份

machine-id 用来判 is_local，**不**用来当 client 长期身份。原因：machine-id 在 Docker 容器、`systemd-machine-id-setup --commit`、系统重装等场景会变。如果用 machine-id 进 `connector_uri`，那些场景下已有的 file connector 都会变成孤儿。

所以 `client_id` 由 CLI 首次启动生成（UUIDv7），写在 `client.toml` 里。`file://<client_id>/<abs-path>` 是 file connector 的稳定标识。备份 `~/.mfs` 就能跨机器迁移；Docker 容器把 `~/.mfs` 挂成 volume 就能跨重启保持身份。

### 3.5 Server 端存储后端跟 profile 无关

profile 只管 client 怎么连 server。server 端用什么 backend 由 `server.toml` 决定：

```toml
[metadata]
backend = "sqlite"                  # 或 "postgres"

[object_store]
backend = "local"                   # 或 "s3" / "r2" / "minio"

[milvus]
uri = "~/.mfs/milvus.db"            # 或 http://host:19530 / https://*.zillizcloud.com
```

常见组合：

| 是否共享 fs | metadata | object store | milvus | 适合 |
|---|---|---|---|---|
| 共享 | sqlite | local | Lite | 个人本机默认 |
| 共享 | postgres | s3 | Zilliz | 个人 dogfood 生产配置 |
| 不共享 | postgres | s3 | Zilliz | 团队部署 |

本机 server 完全可以用 Postgres + S3 + Zilliz Cloud——这两个维度正交。

### 3.6 多租户走 profile 切换

v0.4 server 只有一个隐式 `default` namespace，client 不感知。v0.5+ 引入认证后，**namespace 选择由 token 决定**——一个 token 在 server 端绑定一个或多个 namespace 的访问权限。client 切租户 = 切 profile：

```toml
[[profiles]]
name = "acme-corp-prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_TOKEN_ACME"

[[profiles]]
name = "globex-corp-prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_TOKEN_GLOBEX"
```

connector URI 保持纯净（`postgres://prod`），不带租户信息。

请求头：

```
Authorization: Bearer <token>
MFS-CLI-Version: 0.4.0
MFS-API-Version: v1
```

## 4. 控制面 vs 数据面

HTTP API 主要走 control plane（路径、option、状态、搜索结果），数据在 server 内部流动。**唯一例外**是 remote profile 下处理本地路径——这时 client 需要把字节上传给 server。

### 4.1 一般情况

| 是否共享 fs | 输入 | 数据流 | HTTP 传什么 |
|---|---|---|---|
| 共享 fs | 本地路径 | server 直接读本机文件 → 内部 chunk → 内部写 Milvus | 仅 path + options |
| 共享 fs | 外部 URI | connector 调外部 API → 内部 chunk → 内部写 Milvus | 仅 URI + options |
| 不共享 | 外部 URI | 远端 server 调外部 API → 内部 chunk → 内部写 Milvus | 仅 URI + options |
| 不共享 | 本地路径 | client manifest diff → upload 变化的 bytes → server 处理 | **bytes**（§4.2） |

除了"不共享 + 本地路径"这一条，client 都不传 bytes，也不算 hash、不拆 chunk、不上传 embedding——这些都是 server 的事。

### 4.2 本地文件 upload flow（不共享 fs 时）

`mfs add ./repo` 在 remote profile 下走四步协议：

```
① client scan + 本地 manifest cache (~/.mfs/clients/<server>/<connector>/manifest.db)
  得到当前的 (path, size, mtime_ns, sha1) 集合

② POST /v1/files/manifest
  body: { connector_uri, paths: [...] }
  server 比对自己的 manifest，返回:
    missing  → client 端新增的
    stale    → client hash 变了的
    extra    → server 多余的（client 本地删了的）

③ client 把 missing + stale 文件打成 zip，
  PUT /v1/files/upload?connector_uri=...
  server 把字节放到 staging area: object_store://uploads/<connector_uri>/<temp_file_id>.zip
  返回 temp_file_id

④ POST /v1/files/commit
  body: { connector_uri, temp_file_id, deletions: [extra...] }
  server:
    - 解压 zip 到 staging files/<connector_uri>/
    - 处理 deletions（删 staging files + Milvus chunks + objects 行）
    - 触发 file connector sync（扫 staging area → chunk → embed → 写 Milvus）
    - 更新 server 端 manifest
    - bundle.zip 处理完删除，staging files 树保留作为 cache
  返回 job_id
```

一致性保证：每次 `mfs add` 上传完成后，server 端 file connector 看到的目录树跟 client 当前目录树一致——本地新增/修改 → 上传；本地删除 → 通过 commit deletions 同步删除。

设计取舍：

- **海量小文件**打 zip 一次上传，HTTP roundtrip 数量不随文件数线性增长
- **巨大单文件**（超过 `max_bundle_size_mb`）不打包，独立 multipart streaming PUT
- 不做 chunk-level rsync / 断点续传（v0.4 范围内 ROI 不划算）

错误恢复：

- 上传到一半失败：server 不写任何状态，下次重新 manifest diff
- commit 没成功：staging zip 留着没解压，下次 `mfs add` 重新走

Client 端 manifest cache（SQLite）记录每个 path 的 `(size, mtime_ns, sha1, last_seen)`，每次启动只 stat 比对 size+mtime，不变就复用旧 sha1——增量扫描接近 zero overhead。

`connector_uri` 构造为 `file://<client_id>/<abs-path>`；一个 connector_uri 一辈子绑定一个 client，v0.4 禁止多 client 共写同一 connector。

## 5. Server 启动

`mfs-server` 是服务端 binary，`mfs serve` 是 client 端的便利封装。

### 5.1 运维侧

```bash
# 一体（demo / 小规模）
mfs-server run --bind 0.0.0.0:8765 --config /etc/mfs/server.toml

# 拆分（生产）
mfs-server api    --bind 0.0.0.0:8765 --config /etc/mfs/server.toml
mfs-server worker --concurrency 8     --config /etc/mfs/server.toml
```

### 5.2 个人本机

```bash
mfs serve start             # 等价 mfs-server run --bind 127.0.0.1:<port>
mfs serve stop
mfs serve status            # pid / port / version / uptime / health
mfs serve logs              # ~/.mfs/server.log
```

如果没装 `mfs-server` 包，`mfs serve` 会提示装。`MFS_AUTOSTART=1` 时首次 `mfs add` 检测不到本机 server 会自动 spawn 一次。

### 5.3 本地鉴权

监听 `127.0.0.1` 也需要鉴权（多用户主机场景）：

- 优先 Unix socket，权限 0600，只有 owner 能连
- 否则 loopback TCP + token：server 启动时生成随机 token 写 `~/.mfs/server.token`（0600），CLI 读这个文件作为 Bearer
- Windows 用 named pipe + ACL

## 6. 任务队列

所有 sync / remove / update_config 等操作都进 `connector_jobs` 表，按 `op_kind` 区分；具体的 chunk + embed 任务进 `object_tasks` 表。MFS 直接用 metadata DB 表当队列，不引入 Redis / Celery 等额外组件。

### 6.1 为什么用 DB 当队列

| 维度 | DB 队列 | 外部 broker |
|---|---|---|
| 部署组件 | 零额外（复用 metadata DB） | 多一个 broker |
| 一致性 | 事务内 enqueue + 状态一起 commit | 跨系统需要 outbox |
| 可观测 | SQL 直接查 | broker UI / MONITOR |
| 吞吐上限 | Postgres 几千-几万 op/s | Redis 几十万 op/s |
| 本机部署 | SQLite 复用 | 用户要额外装 Redis |

MFS 的瓶颈不是队列吞吐——embedding API rate limit、LLM 速率、Milvus 写入吞吐才是上限。典型每天几十到几万 task，远低于 Postgres 上限。业界 GitLab / Sentry / Trigger.dev / Hatchet 都用 Postgres + SKIP LOCKED 这套。

未来如果真撑不住，可以换 Redis / NATS 做中间派发，**表 schema 不变**，迁移路径可控。

### 6.2 Worker 怎么拉 task

worker 一次拉一批 task，跑完批量写 Milvus，减少 round-trip。SQL 形态分两种后端：

**Postgres**（多 worker 并发）：

```sql
SELECT * FROM object_tasks
WHERE status = 'pending' AND connector_job_id = $1
ORDER BY priority ASC, started_at ASC NULLS FIRST
FOR UPDATE SKIP LOCKED
LIMIT $batch_size
```

**SQLite**（本机 daemon）：SQLite 不支持 `FOR UPDATE / SKIP LOCKED`，改用显式事务：

```sql
BEGIN IMMEDIATE;
SELECT id FROM object_tasks
  WHERE status = 'pending' AND connector_job_id = ?
  ORDER BY priority ASC, started_at ASC
  LIMIT $batch_size;
UPDATE object_tasks SET status = 'running', started_at = current_timestamp
  WHERE id IN (...);
COMMIT;
```

SQLite 路径建议 `concurrency = 1`，多 worker 在 SQLite 上互相 serialize 没收益。本机部署单 worker 够用。

### 6.3 优先级

`object_tasks.priority` 越小越先。framework 在入队时调一次 `connector.task_priority(change)` 算出 priority 写进 task 行。**默认 0**，FIFO；只有有"首屏可见"诉求的 connector 才重写，比如 file connector 让 README / 核心源码先索引：

| 文件特征 | 相对 priority |
|---|---|
| `README.md` / `CLAUDE.md` / `SKILL.md` / `INDEX.md` | 最先（-350） |
| `pyproject.toml` / `package.json` / `Cargo.toml` / ... | 很先（-260） |
| `src/` / `lib/` / `app/` 下 | 较先（-220） |
| `docs/` / `guides/` 下 | 较先（-190） |
| `tests/` / `fixtures/` 下 | 较后（+80） |
| `dist/` / `build/` / `vendor/` 下 | 最后（+260） |

效果：`mfs add .` 跑到 30% 时核心文件已经索引完，agent 立刻可以搜到关键内容；剩下没跑完的多半是 tests / generated。Postgres / Slack / GitHub 这些 connector 一般保持默认即可——它们产出 ObjectChange 的顺序本身就有意义。

幂等性不依赖顺序：`chunk_id = sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)` 跟处理顺序无关。priority 只影响体感和调度便利性，不影响正确性。

### 6.4 Batching

两层 batching，互不影响。

**第一层：每个外部 API 调用都有 client-side micro-batcher**（DataLoader 模式）

embedding / summary / VLM / converter 这类外部 API 调用，client 包一层 micro-batcher——多个并发 await 自动合并成一次 API 调用，对调用方透明：

```python
class BatchingEmbeddingClient:
    """攒到 max_batch 或超时就 flush，对 worker 透明。
    embedding / summary / VLM / converter 各有一个独立实例，互不干扰。"""
    async def embed(self, text: str) -> Vector: ...
```

这层是吞吐的关键——一次 API 调用做几十到几百个 chunks 远比逐个调省钱省时间。配合 worker 内的 `asyncio.gather`（下面）多个 task 并发调用 API，micro-batcher 自然把它们合并掉。Milvus 不需要 client batcher（worker 显式 batch 写入）。

**第二层：worker 一次拉 N 个 task，并行处理**

```python
async def worker_loop():
    while True:
        tasks = await db.claim_batch(limit=BATCH_SIZE)
        if not tasks:
            await asyncio.sleep(POLL_INTERVAL_MS / 1000)
            continue

        # 并行跑 chunker（IO 并发 + Rust PyO3 模块释放 GIL，CPU bound 部分也能多核）
        chunk_lists = await asyncio.gather(*[chunk_object(t) for t in tasks])
        all_chunks = [c for lst in chunk_lists for c in lst]

        # embedding micro-batcher 会自动 batch；这里 await 一次拿到全部
        vecs = await asyncio.gather(*[embedding_client.embed(c.content) for c in all_chunks])
        for c, v in zip(all_chunks, vecs):
            c.dense_vec = v

        # 一次 RPC 写几百到几千行
        await milvus.batch_upsert(all_chunks)
        await db.mark_succeeded([t.id for t in tasks])
```

`chunk_object(t)` 按 object_kind 分派到对应 processor（document → markdown chunker / pdf → converter → markdown chunker / image → VLM / table_rows → row text 拼接 / ...）。CPU 重的部分（AST 切分、JSONL 流处理、大目录 scan）走 server-rs 的 Rust PyO3 模块，调用时释放 GIL，所以 `asyncio.gather` 在多核上是真并行。

### 6.5 配置

```toml
# server.toml
[worker]
concurrency = 4                  # asyncio task 数（SQLite 设 1）
batch_size = 50
poll_interval_ms = 200
heartbeat_interval_s = 30

[embedding]
batch_size = 100
batch_max_wait_ms = 100

[summary]
batch_size = 20
batch_max_wait_ms = 500

[vlm]
batch_size = 10
batch_max_wait_ms = 500

[converter]                       # PDF / DOCX 等可批量 converter，可选
batch_size = 4
batch_max_wait_ms = 200

[milvus]
insert_batch_size = 1000
```

worker 自适应：每跑完一批记一下"平均 chunks-per-task"，下一轮按"目标 chunks 总数 / 平均 chunks-per-task"调 `batch_size`，并加一道"单批 chunks 硬上限"防止抓到一个产 10 万 chunks 的大 task 卡住整批。这样既不让小 task（如 image 出 1 chunk）每批只攒到几个，也不让大 task（如 `rows.jsonl` 出几万 chunks）独占资源。

## 7. 一致性

**Source of truth 是外部数据源（upstream）**：Metadata DB 只是"我对 upstream 的认知"，Object store 和 Milvus 是从 DB 的 fingerprint 派生出来的产物。这个分层意味着 MFS 的一致性保证只覆盖"upstream 变了我能感知"，不覆盖"派生层被外部直接改了/损坏了"——后者的托底是 `mfs remove + mfs add` 重建。

整套 sync 的正确性靠四条规则：

**① Chunk-level 幂等**

```
chunk_id = sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)
写 chunk = DELETE WHERE chunk_id = X + INSERT
```

任何 worker / 重试 / 并发，对同 chunk_id 的写都等效。`namespace_id` 必须进 hash——否则两个 namespace 注册同名外部数据源（如都叫 `prod`）会让 chunk_id 撞车。

**② Per-object 原子**

```
object_task.status = 'succeeded'
   ↔ 该 object 的所有 chunks 写入 Milvus 且 cache 已更新
```

中途任一步失败 → object_task 保持 'running' 或退 'failed'，下次 sync 重试整个 object。

**③ State 末尾提交（可选 checkpoint）**

`connector.sync()` 过程中 `self.state.set(...)` 写入暂存（`connector_jobs.state_snapshot`），默认只在 sync_job 所有 task 成功时才 commit 到 `connector_state` 表。

中途崩溃 → state 不 commit → 下次 sync 从上次成功的 state 重启。`connector.sync()` 必须 idempotent。

外部 API 大数据量场景（Slack / Gmail 等）可以调 `self.state.checkpoint()` 主动 commit 当前 state，避免崩溃后整批重跑被 rate limit 打爆。详见 [04 §5.6](04-connector-and-ingest.md#56-mid-job-checkpoint-api)。

**④ Sync 末尾 reconcile pass**

connector 只负责报告 upstream 变化，但下游产物（cache / chunk / embedding）可能因 framework 配置变化（换 embedding 模型 / chunker 升级）而 stale，connector 不感知。每次 sync_job 在 connector yield 完后，framework 跑一遍 sweep：

```
for object in 当前 connector 下未在本次 yield 的 object:
  跑 fingerprint chain 比对：
    cache 层 fp 变了？     → 入队 cache rebuild
    chunk 层 fp 变了？     → 入队 chunk rebuild
    embedding 层 fp 变了？ → 入队 embed only
```

这条让"换 embedding 模型 → 跑 `mfs add` → 自动重 embed"能 work，用户不需要加 `--force-index`。完整逻辑见 [04 §5.2](04-connector-and-ingest.md#52-framework-内部per-artifact-fingerprint-chain)。

framework 不暴露 `commit()` 给 connector（只暴露 `checkpoint()`），commit 时机由 framework 控制。

### 7.1 故障恢复

daemon / worker 重启时扫一次：

```sql
-- 心跳超时的 sync_job 标 failed
UPDATE connector_jobs SET status='failed', error='interrupted'
  WHERE status='running' AND heartbeat < now() - interval '5 minutes';

-- 对应 object_tasks 重置为 pending，依赖幂等性重跑
UPDATE object_tasks SET status='pending'
  WHERE status='running'
    AND connector_job_id IN (SELECT id FROM connector_jobs WHERE status='failed');
```

state 没 commit，下次 `mfs add` 自然从上一个成功的 state 接续。没有 `mfs job retry` 命令——重跑 = 下次 `mfs add`。

### 7.2 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新建 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force-index` | 所有 object 视为 modified，跳过 fp 比对，强制重 chunk + embed |
| `mfs add ./path --force-upload` | 仅 upload flow：client 忽略 manifest cache 全量重传 + server 强制重 index |
| `mfs add <uri>` 在前次失败后 | state 未 commit，从上一个成功的 state 重跑 |
| 第二次 `mfs add <uri>` 在前次还 running | 拒绝 `sync_already_running, see job <id>` |

### 7.3 完成 job 的归档

DB 队列没有传统 FIFO 的"出队"动作。任务靠 `status` 流转：

```
pending → running → succeeded / failed / cancelled
```

worker 只看 `status='pending'`，终态行留在表里供 `mfs job list / inspect` 查。按 status 分级保留：

```toml
[jobs.retention]
succeeded_days = 7
failed_days = 30                # 失败的留久点方便 debug
cancelled_days = 7
running_timeout_hours = 24      # 超过 24h 视为僵尸标 failed
```

server 启一个 housekeeping coroutine，每天跑两步：标僵尸 + 按 retention 删过期。`finished_at` 而不是 `created_at`——长 sync 的 created_at 不该作为归档基准。

## 8. 并发协调

所有"对一个 connector 的操作"（sync / force_sync / remove / update_config）都进 `connector_jobs` 表，用 `op_kind` 区分。

### 8.1 约束

```sql
CREATE UNIQUE INDEX ux_connector_jobs_one_running
  ON connector_jobs (connector_id) WHERE status = 'running';
CREATE UNIQUE INDEX ux_connector_jobs_one_queued
  ON connector_jobs (connector_id) WHERE status = 'queued';
```

同 connector 任意时刻**至多一个 running** + **至多一个 queued**。两者可以同时存在——这正是 sync → remove preempt 流程需要的。具体哪些 op 跟哪些 op 冲突，应用层判断。

### 8.2 三条规则

**① 同种重复 → 拒绝**（destructive 除外，幂等）

- `sync + sync` / `force_sync + force_sync` / `update_config + update_config` → 拒绝
- `remove + remove` → 幂等成功（目标状态就是"消失"）

**② 不同种竞争 → remove 优先**

`remove` 是 destructive superset，永远能 preempt sync / force_sync / update_config。其他方向反过来不行。

**③ 同方向不允许 preempt**

`sync` 中又来 `sync` / `force_sync` → 拒绝。理由：

- sync 已经在做你想做的事，preempt 没收益
- force_sync 是 destructive，应该显式 `mfs job cancel` 后再来——避免"以为只是 add 结果触发了全量重跑"

跟 git 一致：rebase / merge 中间不能再触发同类操作，必须 `--abort`。

完整语义表：

| 当前 in-flight | 新来 | 行为 |
|---|---|---|
| 无 | 任意 | OK |
| `sync` | `sync` | 拒绝 `sync_already_running` |
| `sync` | `force_sync` | 拒绝；提示先 `mfs job cancel` |
| `sync` | `remove` | preempt：sync 标 cancelling，当前 task 完成后退出 → remove 入队 |
| `sync` | `update_config` | 拒绝 |
| `force_sync` | `sync` / `force_sync` | 拒绝 |
| `force_sync` | `remove` | preempt |
| `remove` | 任意非 remove | 拒绝 `connector_removing` |
| `remove` | `remove` | 幂等 |
| `update_config` | `sync` | 拒绝 |
| `update_config` | `remove` | preempt |

### 8.3 Sync 中的 Remove 流程

```
sync running on connector C
  │
  ▼ mfs remove C 来了
  │
  ① 检查 connectors.status='active'（否则按重复 remove）
  ② connectors.status = 'removing'           ← 立刻设置；之后的 add/sync 看到就拒绝
  ③ INSERT connector_jobs (op_kind='remove', status='queued')
  ④ 把 running 的 sync 标 cancelling
  ⑤ worker 在每个 object_task 边界检查 cancel signal，当前 task 完成后退出
  ⑥ sync_job status → 'cancelled'
  ⑦ remove job 'queued' → 'running'
  ⑧ 清理：
     - Milvus: DELETE WHERE namespace_id = X AND connector_uri = <root>
       （按 partition_key 路由，只扫该 connector 的桶，不是全表）
     - object store: 删 cache + staging files/ 树
     - metadata DB: 删 caches / connector_state / objects / upload_manifests / object_tasks / connector_jobs
  ⑨ connectors row DELETE
  ⑩ remove job 'succeeded'
```

清理顺序保证幂等可重入：⑧ 中途崩，下次重启 worker 从任何一步开始都是 no-op。

期间 `mfs status C` 看到：

```
Connector: postgres://prod
Status:    removing
Current job: job_remove_xx (queued)
  waiting for: job_sync_yy (cancelling)
```

worker 端检查 cancel signal：

```python
async def process_object_task(task):
    if await is_cancelled(task.connector_job_id):
        task.status = 'cancelled'
        return
    # 整 task 是原子单元，中途不打断
    await do_work(task)
    task.status = 'succeeded'
```

单 task 耗时极长（一个 object 出 10 万 chunk）的场景可以在 chunk 批次边界加细检查点，cancel 后已写入的 chunks 留着（下次 sync 幂等覆盖）。`is_cancelled()` 是 in-memory cached，不每次都打 DB。

Scheduler / Watcher 也通过同一个表入口：`connectors.status='removing'` 时 scheduler 跳过、watcher 停止该 root，避免 race。

## 9. 多租户与 namespace

### 9.1 设计原则

借鉴 AWS（Account vs Organization）/ GCP（Project vs Folder）/ K8s（Namespace vs Project）：**数据隔离边界跟组织结构语义解耦**。

- **底层**：`namespace_id` 是唯一的物理分区主键。所有 DB 表 / Milvus 数据 / object store prefix 都按它切。稳定、不可重组。
- **上层**：Workspace / User / Project / Team 是产品概念，通过 **mapping 表**指向 namespace。组织关系演化（个人 → 团队、跨 workspace 共享）只改 mapping，底层数据零迁移。

> 用户视角看不到 "namespace" 这个词——它是 server 内部主键。对外暴露的概念是 Workspace / User（v0.5+ 引入）。

### 9.2 v0.4：单 namespace，零配置

v0.4 server 启动时自动创建一个 `default` namespace。所有数据写入都带 `namespace_id = "default"`，所有查询都 filter `namespace_id = "default"`。client.toml 没有租户字段，HTTP 请求不带租户 header。

### 9.3 v0.5+：加 Workspace + User mapping

底层 namespace schema 不动，新增 mapping 表：

```sql
users (
  id            VARCHAR PRIMARY KEY,
  email         VARCHAR UNIQUE,
  created_at    TIMESTAMP
);

workspaces (
  id            VARCHAR PRIMARY KEY,
  name          VARCHAR,
  billing_id    VARCHAR,
  created_at    TIMESTAMP
);

workspace_members (
  workspace_id  VARCHAR REFERENCES workspaces(id),
  user_id       VARCHAR REFERENCES users(id),
  role          VARCHAR,                          -- 'owner' | 'member' | 'viewer'
  PRIMARY KEY (workspace_id, user_id)
);

workspace_namespaces (
  workspace_id  VARCHAR REFERENCES workspaces(id),
  namespace_id  VARCHAR REFERENCES namespaces(id),
  PRIMARY KEY (workspace_id, namespace_id)
);

user_namespaces (
  user_id       VARCHAR REFERENCES users(id),
  namespace_id  VARCHAR REFERENCES namespaces(id),
  role          VARCHAR,
  PRIMARY KEY (user_id, namespace_id)
);

namespaces (
  id            VARCHAR PRIMARY KEY,
  slug          VARCHAR UNIQUE,
  created_at    TIMESTAMP
);
```

请求作用域：

```
token → user_id → 该 user 能访问的 namespace_id 集合
  ↓
所有 query: WHERE namespace_id IN (resolved_set)
```

能拼出来的产品形态：

| 产品概念 | mapping 表达 |
|---|---|
| 个人空间 | `user_namespaces` 1:1 |
| 团队 workspace | `workspace_namespaces` 1:1 + `workspace_members` |
| 一个 workspace 下多 project | `workspace_namespaces` 1:N，每个 project 一个 namespace |
| 跨 workspace 共享数据 | 同一 namespace_id 出现在多个 `workspace_namespaces` 行 |
| 个人迁到团队 | 把 namespace 从 `user_namespaces` 移到 `workspace_namespaces`——数据零迁移 |

### 9.4 Milvus 隔离策略

默认在 Milvus 层用 scalar filter（`namespace_id IN (...)`）。某些 namespace 数据量大或需要更强隔离时，可以按 `collection_strategy` 升级：

| 策略 | Milvus 实现 | 隔离强度 | 资源开销 | 适用 |
|---|---|---|---|---|
| `single`（默认） | 一张 collection，按 `namespace_id` scalar filter | 弱 | 最小 | v0.4 / 小规模 |
| `per_connector` | 一 connector 一张 collection | 中 | 中 | 数据量大时按 connector 切 |
| `per_namespace` | 一 namespace 一张 collection | 强 | 大 | 多租户强隔离 / 合规 |
| `database_per_namespace` | 一 namespace 一 Milvus database | 最强 | 最大 | enterprise 合规 |

切策略通过迁移工具（roadmap）。

### 9.5 数据量分层

| 量级 | 策略 |
|---|---|
| < 几百万 chunks | 默认 `single` 够 |
| 几千万 chunks | 调 Milvus shard / HNSW 参数；考虑分 partition |
| 亿级 chunks | 切 `per_connector` 或 `per_namespace`；冷热分层 |

## 10. 存储层

三套存储，职责清晰；每套的具体后端独立可换。

### 10.1 Metadata DB

本机 SQLite `~/.mfs/metadata.db`，远端 Postgres。

```sql
connectors (
  id              VARCHAR PRIMARY KEY,
  namespace_id    VARCHAR DEFAULT 'default',
  root_uri        VARCHAR,
  type            VARCHAR,
  label           VARCHAR,
  status          VARCHAR DEFAULT 'active',        -- 'active' | 'removing'
  config_json     TEXT,
  config_hash     VARCHAR,
  credential_ref  VARCHAR,
  registered_at   TIMESTAMP,
  last_health     TIMESTAMP,
  health_status   VARCHAR,
  UNIQUE (namespace_id, root_uri)
);

objects (
  connector_id    VARCHAR REFERENCES connectors(id),
  object_uri      VARCHAR,
  parent_path     VARCHAR,
  type            VARCHAR,                          -- "file" | "dir"
  media_type      VARCHAR,
  size_hint       INTEGER,
  extra_json      TEXT,
  fingerprint     VARCHAR,
  indexable       BOOLEAN,
  capabilities    TEXT,
  last_seen       TIMESTAMP,
  PRIMARY KEY (connector_id, object_uri),
  INDEX (connector_id, parent_path)
);

caches (
  namespace_id    VARCHAR DEFAULT 'default',
  object_uri      VARCHAR,
  cache_kind      VARCHAR,
  storage_path    VARCHAR,
  fingerprint     VARCHAR,
  size_bytes      INTEGER,
  built_at        TIMESTAMP,
  last_accessed   TIMESTAMP,
  PRIMARY KEY (namespace_id, object_uri, cache_kind)
);

-- 任务队列
connector_jobs (
  id                    VARCHAR PRIMARY KEY,
  namespace_id          VARCHAR DEFAULT 'default',
  connector_id          VARCHAR REFERENCES connectors(id),
  op_kind               VARCHAR,        -- 'sync' | 'force_sync' | 'remove' | 'update_config'
  trigger               VARCHAR,        -- 'manual' | 'scheduled' | 'watch'
  status                VARCHAR,        -- 'queued' | 'running' | 'cancelling' | 'cancelled' | 'succeeded' | 'failed'
  started_at            TIMESTAMP,
  finished_at           TIMESTAMP,
  heartbeat             TIMESTAMP,
  total_objects         INTEGER,
  succeeded_objects     INTEGER,
  failed_objects        INTEGER,
  cancelled_objects     INTEGER,
  error                 TEXT,
  state_snapshot        TEXT            -- pending：暂存的 connector state，sync 末尾才 commit
);
CREATE UNIQUE INDEX ux_connector_jobs_one_running ON connector_jobs (connector_id) WHERE status = 'running';
CREATE UNIQUE INDEX ux_connector_jobs_one_queued  ON connector_jobs (connector_id) WHERE status = 'queued';

object_tasks (
  id                    VARCHAR PRIMARY KEY,
  connector_job_id      VARCHAR REFERENCES connector_jobs(id),
  object_uri            VARCHAR,
  change_kind           VARCHAR,        -- 'added' | 'modified' | 'deleted'
  status                VARCHAR,
  priority              INTEGER DEFAULT 0,
  attempts              INTEGER DEFAULT 0,
  last_error            TEXT,
  started_at            TIMESTAMP,
  finished_at           TIMESTAMP
);
CREATE INDEX ix_object_tasks_sched ON object_tasks (connector_job_id, status, priority);
CREATE INDEX ix_object_tasks_running ON object_tasks (status, started_at) WHERE status = 'running';

-- Connector 内部 state
connector_state (
  connector_id          VARCHAR,
  key                   VARCHAR,
  value                 TEXT,
  updated_at            TIMESTAMP,
  PRIMARY KEY (connector_id, key)
);

watch_grants (
  path            VARCHAR PRIMARY KEY,
  granted_at      TIMESTAMP
);

-- Upload flow（client → server 上传本地文件）
upload_manifests (
  connector_id    VARCHAR REFERENCES connectors(id),
  path            VARCHAR,
  size            INTEGER,
  mtime_ns        BIGINT,
  sha1            VARCHAR,
  last_commit_at  TIMESTAMP,
  PRIMARY KEY (connector_id, path)
);

upload_staging (
  temp_file_id    VARCHAR PRIMARY KEY,
  connector_id    VARCHAR,
  storage_path    VARCHAR,
  size_bytes      INTEGER,
  uploaded_at     TIMESTAMP,
  expires_at      TIMESTAMP                 -- 默认 1h 后自动清理未 commit 的
);
```

所有顶层表都以 `namespace_id` 作为物理分区主键。v0.5+ 多租户上线只加 mapping 表，底层 schema 不动。

### 10.2 Object store

存两类东西：

- **cache 文件**：每个有 cache 的 object 的产物（每个 object 通常只对应一个 cache_kind）
- **upload staging**：client 上传的 zip bundle + 解压后的文件树（仅 §4.2 upload flow 用）

cache_kind 跟 object 类型的对应：

| object 类型 | cache_kind | 例子 |
|---|---|---|
| PDF / DOCX 等可转 markdown | `converted_md` | `manual.pdf` 的 markdown 缓存 |
| DB rows / API records 集合 | `page_cache.jsonl` | DB 物化页 |
| DB rows 的 head 预拉取 | `head_cache.jsonl` | 前 100 行预 cache，加速 head 命中 |
| 图片 | `vlm_text` | 图片 VLM description |
| DB schema dump | `schema_dump.json` | postgres schema.json 的物化 |
| markdown / code / 纯文本真实文件 | **无 cache** | 直接 read |

目录布局（**按 namespace_id 切**）：

```
~/.mfs/cache/
  caches/
    <namespace_id>/                          ← v0.4 恒为 "default"
      <sha1(./repo/manual.pdf)>/
        converted_md
      <sha1(postgres://prod/.../rows.jsonl)>/
        page_cache.jsonl
      <sha1(./repo/diagram.png)>/
        vlm_text
      <sha1(postgres://prod/.../schema.json)>/
        schema_dump.json
  uploads/
    <namespace_id>/<connector_id>/
      <temp_file_id>.zip                     ← commit 后删
  files/
    <namespace_id>/<connector_id>/           ← upload flow 解压后的真实文件树，作为 cache 保留
      src/cli.py
      README.md
```

按 namespace_id 切的原因：cache 内容来自 object_uri 的 sha1，但两个 namespace 可能注册同名 connector → object_uri 相同 → cache key 撞。v0.4 单 namespace 只是 `default/` 一层，没成本；v0.5+ 已经物理隔离不需要重组。

#### 后端选择

| 后端 | 首字节延迟 | 多 worker 共享 | 持久化 | 适合 |
|---|---|---|---|---|
| `local`（fs） | ~1ms | 仅同进程 | 跟随 disk | 个人本机、单容器 server |
| `minio` | ~5-10ms | ✅ | 跟随 volume | Docker Compose 多 worker |
| `s3 / r2 / gcs` | ~50-100ms | ✅ | HA、跨实例 | K8s 生产部署 |

S3 看起来比本地慢，但仍然远比 connector 重新拉取外部 API 快。对吞吐影响小（streaming）。

部署建议：

| 部署形态 | object_store backend |
|---|---|
| 个人本机 `mfs serve` | `local`（零配置） |
| Docker 单容器 server | `local`（必须 `-v /data/cache` 持久化） |
| Docker Compose 多 worker | `minio` |
| K8s / 商业化生产 | `s3 / r2 / gcs` |

不做"S3 + 本机磁盘两级 cache"——v0.4 用户察觉不到 50ms 区别，ROI 不值。

```toml
[cache]
max_size_gb = 10
eviction = "lru"

[upload]
max_bundle_size_mb = 500
staging_path = "uploads/"
staging_expiry_hours = 1
per_namespace_quota_gb = 0           # 0 = 不限；多租户部署可按 namespace 设
```

### 10.3 Milvus

一张 collection `mfs_chunks`，`partition_key = connector_uri`。详细 schema 见 [06 §1](06-search-and-retrieval.md#1-milvus-collection-schema)。

**partition_key 不是 named partition**。Milvus 里有两套不同机制：

| 机制 | API | 数量上限 | 适合 |
|---|---|---|---|
| Named partition | `create_partition / drop_partition` | ~4096 / collection | 你显式管理的少量分区 |
| `partition_key` 字段 | schema 声明，写入/查询自动哈希路由 | 由 `num_partitions` 配置，默认 64 桶 | 多租户类大量自动分桶 |

MFS 用 partition_key，理由是 connector 数量预期上千：

| 维度 | partition_key | named partition |
|---|---|---|
| connector 数量上限 | 几万级 | 单 collection ~4096，硬上限 |
| 注册/注销代码 | 不动 schema | 显式 create/drop，需手动 GC |
| remove 性能 | DELETE WHERE（中等） | drop_partition（最快） |
| 多 namespace × 多 connector | 可 scale | partition 数量爆炸 |

代价是 remove 走 `DELETE WHERE connector_uri = X`（按 partition_key 路由，只扫该桶不是全表），比 drop_partition 慢一个数量级，但比"全表 scan delete"快很多。

backend 推荐：

| 后端 | URI | 适合 |
|---|---|---|
| Milvus Lite 3.0+ | `~/.mfs/milvus.db` | 个人本机，零运维 |
| Zilliz Cloud | `https://*.zillizcloud.com` + token | CS 部署、商业化 |
| 自部署 Milvus 3.0+ | `http://host:19530` | 自有数据中心 |

v0.4 主推 Lite（个人）+ Zilliz Cloud（CS）。自部署 Milvus 是多容器拓扑（etcd / pulsar / object store），运维负担重，不作为默认推荐。文档默认假设 Milvus 3.0+（sparse_vec / BM25 / partition_key 必需）。

> Milvus Lite 3.0 是大版本重构（Python 全重写），实施 v0.4 时按 Lite 3.0 最新代码实测对齐能力。如果 Lite 3.0 不支持 sparse_vec 或 partition_key，个人本机退回 single collection + scalar filter 也行。

```toml
[milvus]
uri = "~/.mfs/milvus.db"               # 默认 Lite
# uri = "https://xxx.zillizcloud.com"  # CS
# token = "..."
collection_strategy = "single"          # single | per_connector | per_namespace
```

## 11. 凭据

### 11.1 Connector 凭据

connector TOML 用 `credential_ref` 引用，不写明文：

```toml
credential_ref = "env:PG_PROD_DSN"
credential_ref = "secret:pg-prod-readonly"
credential_ref = "file:~/.mfs/secrets/pg-prod.toml"
credential_ref = "vault:secret/data/mfs/pg-prod"
```

解析顺序：

1. `env:VAR`——环境变量
2. `secret:NAME`——OS keychain（macOS Keychain / Linux secret-service / Windows Credential Manager）
3. `file:PATH`——本地文件，默认权限 0600
4. `vault:PATH`——HashiCorp Vault（远端部署主流）

凭据 schema 由 connector plugin 自己定义（PostgresCredential / SlackCredential / ...）。

### 11.2 Remote server 认证

```toml
[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_API_TOKEN"
```

请求头：`Authorization: Bearer <token>`。

`mfs login` 命令 v0.4 不内置，属于商业化能力。
