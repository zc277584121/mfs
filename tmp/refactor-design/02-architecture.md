# 架构与部署

本文回答：client / server 怎么分，daemon / profile 怎么用，控制面 vs 数据面，存储层，部署形态，凭据，多租户预留。

## 1. 总体架构

```
┌─────────────────────────── Client side ────────────────────────────┐
│  mfs CLI / Python SDK / TS/Go/Java SDK / Skill                      │
│        │                                                            │
│        │     parse args · profile resolve · HTTP transport          │
│        v                                                            │
│  ┌────────────────────┐                                             │
│  │ HTTP transport      │   is_local: auto-detected by machine-id   │
│  └─────────┬──────────┘                                             │
└────────────┼────────────────────────────────────────────────────────┘
             │ HTTP /v1 (control plane only)
             v
┌─────────────────────────── Server side ────────────────────────────┐
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ API routes  /v1/add  /v1/objects/*  /v1/search  /v1/grep    │  │
│  │             /v1/connectors/*  /v1/jobs/*  /v1/status         │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Engine        路由 path/uri → connector plugin                │  │
│  │               策划 job、状态管理                              │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Connector plugins (connectors/<name>/)                       │  │
│  │   file / web / postgres / slack / github / gdrive / s3 / ... │  │
│  │   实现 list / stat / read / fingerprint / change_set         │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Object handlers (objects/<kind>/)                            │  │
│  │   document / code / table_rows / message_stream /            │  │
│  │   record_collection / image / binary                         │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Pipeline                                                     │  │
│  │   embedding / summary / VLM / retrieval / export             │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Worker queue                                                 │  │
│  │   DB-backed: SQLite (local) or Postgres (remote)             │  │
│  │   SELECT ... FOR UPDATE SKIP LOCKED 取 task                 │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Storage adapters                                             │  │
│  │   Metadata DB (SQLite / Postgres)                            │  │
│  │   Object store (local fs / S3 / R2 / MinIO)                  │  │
│  │   Milvus (Lite / self-hosted / Zilliz Cloud)                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Profile 与存储后端是正交的

### 2.0 配置文件命名约定

MFS 有**两个独立的配置文件**，各自负责不同身份的设置：

| 文件 | 路径 | 内容 | 谁读 |
|---|---|---|---|
| client 配置 | `~/.mfs/client.toml` | profiles、endpoint URL、API token、workspace_id | `mfs` CLI |
| server 配置 | `~/.mfs/server.toml`（本地 daemon）<br>`/etc/mfs/server.toml`（远端部署） | metadata backend / object_store / milvus / worker / embedding / chunker / cache / summary / vlm | `mfs-server` |

只装 `mfs-cli` 的用户只接触 `client.toml`；只装 `mfs-server`（运维端）的人只接触 `server.toml`；两者都装（个人本机）的人各自维护。两个文件不会混在一起，schema 不会冲突。

**server.toml 查找优先级**（第一个找到就用，不合并）：

1. `--config <path>` 命令行参数
2. `$MFS_SERVER_CONFIG` 环境变量
3. `./server.toml`（当前 cwd，开发调试）
4. `~/.mfs/server.toml`（个人本机）
5. `/etc/mfs/server.toml`（系统部署）
6. 内置默认值（兜底）

`mfs serve start`（client wrapper）发现 `~/.mfs/server.toml` 不存在时自动生成最小默认配置；`mfs-server run`（运维直接用）找不到就报错退出。client.toml 用类似的查找顺序，没有第 5 项。

下面分两个维度讨论 profile 和 server 后端——它们正交。

### 维度 A：client / server 是否共享文件系统（自动判断）

决定 **client 和 server 是否共享同一文件系统命名空间**——也决定本地路径请求是 server 直接读、还是走 upload 流。

**自动判断**：CLI 和 server 握手时比对 machine-id。**用户不需要手动配 `kind` 字段**。

```python
# CLI 启动 / 首次连一个 profile 时
client_machine_id = read_machine_id()     # /etc/machine-id (Linux/WSL) / ioreg (Mac) / registry (Windows)
server_resp = await client.get(profile.url + "/v1/server/info")
profile.is_local = (client_machine_id == server_resp["machine_id"])    # cached in client.toml
```

各种场景自动判断结果：

| 场景 | machine-id 比对 | 模式 |
|---|---|---|
| 同机 `mfs serve start` | 一致 | local（直接读本机） |
| 远端 https server | 不一致 | remote（走 upload） |
| Docker 容器（容器独立 machine-id） | 不一致 | remote |
| WSL2（独立 machine-id） | 不一致 | remote |
| SSH port forward | 不一致 | remote |

**Override（极少用）**：调试 upload 流时 `export MFS_FORCE_REMOTE=1` 强制 remote。

### 维度 B：server 端存储后端

跟 client / server 是否共享 fs 完全无关。server 端配置文件 `~/.mfs/server.toml`（本地 daemon）或 `/etc/mfs/server.toml`（远端部署）决定用什么后端：

```toml
[metadata]
backend = "sqlite"                        # 或 "postgres"
# url = "postgresql://localhost/mfs"

[object_store]
backend = "local"                         # 或 "s3" / "r2" / "minio"
# bucket = "my-mfs-cache"

[milvus]
uri = "~/.mfs/milvus.db"                  # 或 http://localhost:19530 / https://*.zillizcloud.com
# token = "..."
```

### 维度 A × 维度 B 自由组合

| 是否共享 fs | metadata | object_store | milvus | 说明 |
|---|---|---|---|---|
| 共享 | sqlite | local fs | Lite | 默认；个人本机最简 |
| 共享 | postgres | local fs | Lite | 个人开发，跟生产 metadata 一致 |
| 共享 | postgres | s3 | Zilliz | 个人 dogfood 生产配置 |
| 不共享 | postgres | s3 | Zilliz | 团队部署 |
| 不共享 | postgres | r2 | self-hosted | 自部署 |

「本机 server 又想用 PG / S3 / Zilliz」完全 OK，改 server 配置即可。

### profile 配置

`~/.mfs/client.toml`（client 侧）：

```toml
[client]
default_profile = "local"

[[profiles]]
name = "local"
api_base_url = "http://127.0.0.1:8765"

[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_API_TOKEN"
workspace = "acme-corp"           # 多 workspace 场景
```

不需要写 `kind` 字段，CLI 跟 server 握手时自动判断 `is_local`，结果缓存到 client.toml。

profile 管理：

```bash
mfs profile add <name> --url <url> [--workspace <id>]
mfs profile use <name>
mfs profile list
mfs profile status               # 显示当前 profile 是否本地（machine-id 比对结果）
```

### 多 workspace 走 profile 层

`workspace` 放在 **profile 上**，不放 URI 里。connector URI 永远保持纯净（`postgres://prod`）。HTTP transport 自动按当前 profile 注入 `X-MFS-Workspace: <workspace>` header；server 端按这个 filter 数据，server schema 内部用 `workspace_id` 列名。

```toml
[[profiles]]
name = "acme-corp-prod"
api_base_url = "https://mfs.example.com"
workspace = "acme-corp"
credential_ref = "env:MFS_TOKEN_ACME"

[[profiles]]
name = "globex-corp-prod"
api_base_url = "https://mfs.example.com"
workspace = "globex-corp"
credential_ref = "env:MFS_TOKEN_GLOBEX"
```

`mfs profile use acme-corp-prod` 切换 = 切租户。同一台 client 可注册多个租户的 profile，按名字切换。

CLI 请求头：

```text
Authorization: Bearer <token>
MFS-CLI-Version: 0.4.0
MFS-API-Version: v1
```

## 3. Control plane vs data plane

**HTTP API 大部分走 control plane**（path / option / status / 元数据 / 搜索结果），数据在 server 内部流动。**唯一例外**是 remote profile 下处理本地文件——client 需要把 bytes 上传到 server，详见 §3.5。

### 同步场景的数据流

| 是否共享 fs | 输入 | 数据流 | HTTP 传什么 |
|---|---|---|---|
| 共享 fs | 本地路径 | server 直接读本机文件 → 内部 chunk → 内部写 Milvus | 仅 path + options |
| 共享 fs | 外部 URI | connector 调外部 API → 内部 chunk → 内部写 Milvus | 仅 URI + options |
| 不共享 | 本地路径 | client manifest diff → upload 变化的 bytes → server 处理 | **bytes**（§3.5） |
| 不共享 | 外部 URI | 远端 server 调外部 API → server 内部 chunk → server 内部写 Milvus | 仅 URI + options |

### 大部分场景 client 不传 bytes

除了 §3.5 的"本地文件 → remote server"这一条路径，其他场景 client 都不需要：

- 算文件 hash 给 server 比对（server 自己读、自己 hash）
- 拆 chunk 上传（chunk 是 server 内部产物）
- 上传 embedding（embedder 在 server 内部）

### 3.5 本地文件 upload flow（不共享 fs 场景）

当 client 和 server 不共享文件系统，`mfs add ./repo` 走 **manifest diff + zip bundle + temp_file_id + commit** 四步协议。

#### 协议四步

```
① client 端：scan + 本地 manifest cache (~/.mfs/clients/<server>/<connector>/manifest.db)
              得到 current_paths（不含已删除的文件）

② POST /v1/files/manifest
   body: {
     connector_uri: "file://<machine-id>/abs/path/repo",
     paths: [{path, size, mtime_ns, sha1}, ...]
   }
   server 对照自己的 manifest（上次 commit 成功后的），返回:
     {
       missing:    [client 端新增的 path],
       stale:      [client 端 hash 变了的 path],
       extra:      [server 有但 client 没有的 path]     ← 这些是用户本地删了的
     }

③ client 把 missing + stale 文件打包成一个 zip bundle，发送：
   PUT /v1/files/upload?connector_uri=...  (multipart, streaming)
   body: <bundle.zip bytes>
   返回: { temp_file_id }
   server 把字节落到 staging area:
     object_store://uploads/<connector_uri>/<temp_file_id>.zip

④ POST /v1/files/commit
   body: {
     connector_uri,
     temp_file_id,
     deletions: [extra...]              ← server 删除这些 path 对应的所有数据
   }
   返回: { job_id }
   
   server 在 commit 处理里：
     - 解压 staging zip 到 staging://files/<connector_uri>/
     - 对每个 deletions[i]:
         · 删 staging://files/.../<path>
         · DELETE FROM mfs_chunks WHERE object_uri LIKE 'file://.../<path>%'
         · DELETE FROM objects   WHERE object_uri = '...<path>'
     - 触发 file connector sync（扫 staging area → chunk → embed → 写 Milvus）
     - 更新 server-side manifest 为 client 提交的 paths
     - bundle.zip 处理完删除（staging files 树保留作为 cache）
```

#### 一致性契约

**每次 `mfs add` 上传完成后，server 端 file connector 的状态（cache files + Milvus chunks + objects 表）与 client 端的目录树结构完全一致**：本地新增 / 修改 → 上传，本地删除 → 通过 commit deletions 同步删除。

简单说：sync 后 server 的 file connector "看到的"目录树 = client 当前目录树。

#### 海量小文件 / 巨大单文件

- **海量小文件**：打包成 zip bundle 一次上传，HTTP roundtrip 数量恒定（不随文件数线性增长）。
- **巨大单文件**：单文件 size 超过 `max_bundle_size_mb` 时**不打包**，独立 multipart streaming PUT。
- 不做 chunk-level rsync / 断点续传（复杂度 vs v0.4 范围内的收益不划算）。
- 大于 `max_bundle_size_mb` 单文件 server 端 staging path 独立。

#### connector_uri 的构造

client 端发的 `connector_uri` 形如 `file://<client-machine-id>/<abs-path>`。`<client-machine-id>` 是 client 端 machine-id，让多个 client 不会撞同一 alias。一个 `connector_uri` 一辈子绑定一个 client（v0.4 禁止"多 client 共写同一 connector"——避免 manifest race）。

用户面仍然是 `mfs add ./repo`，CLI 自动规范化成 `file://...`。

#### 错误恢复（简化版，先不考虑灾难）

- ③ 上传到一半失败：bundle.zip 没传完，server 不写任何状态；下次 `mfs add` 重新 manifest diff + 上传
- ④ commit 没成功：server staging zip 留着但没解压、deletions 未应用；下次 `mfs add` 会发现 client manifest 跟 server manifest 还不一致，重新走流程
- 断网 / 中途挂等灾难恢复细节是 P2，v0.4 简化版即可

#### Client 端的 manifest cache

`~/.mfs/clients/<server-id>/<connector-uri-hash>/manifest.db`（SQLite）：

```sql
manifest (
  path        VARCHAR PRIMARY KEY,
  size        INTEGER,
  mtime_ns    BIGINT,
  sha1        VARCHAR,
  last_seen   TIMESTAMP
);
```

client 启动时不每次全量重 hash，只 stat 比对 `size + mtime_ns`，相等就复用旧 sha1。差不多 zero-overhead 的 incremental scan。

## 4. 四种行为矩阵

| 是否共享 fs | 输入类型 | 行为 |
|---|---|---|
| 共享 | 本地路径 `./repo` | server 直接读本机文件，scan + chunk + 写 Milvus 都在 server 内部完成 |
| 共享 | 外部 URI `postgres://prod` | connector 在 server 内执行，索引写本机存储 |
| 不共享 | 本地路径 `./repo` | 走 upload flow（§3.5）：client manifest diff + zip bundle + commit |
| 不共享 | 外部 URI `postgres://prod` | connector 在远端 server/worker 内执行，索引写远端存储 |

## 5. Server 端启动

MFS 的 server-side binary 是 **`mfs-server`**。`mfs serve` 是 client-side wrapper，本质是本机 spawn 一个 `mfs-server` 进程。

### 5.1 `mfs-server` 命令（运维用）

```bash
# 一体（demo / 小规模）
mfs-server run --bind 0.0.0.0:8765 --config /etc/mfs/server.toml

# 拆分（生产）
mfs-server api    --bind 0.0.0.0:8765 --config /etc/mfs/server.toml
mfs-server worker --concurrency 8     --config /etc/mfs/server.toml
```

部署方式：

- **Docker**：`docker run ghcr.io/.../mfs-server`、`docker compose up`
- **Kubernetes**：Helm chart
- **systemd**：包成 service，`systemctl start mfs-server`

### 5.2 `mfs serve` 命令（client-side 封装）

给个人本机开发者用：

```bash
mfs serve start                    # 等价 mfs-server run --bind 127.0.0.1:<port>
mfs serve stop
mfs serve status                   # pid / port / version / uptime / health
mfs serve logs                     # ~/.mfs/server.log
```

如果只装了 `mfs-cli` 没装 `mfs-server`：

```text
mfs serve requires mfs-server package.
Install it with:
  uv tool install mfs-server
```

### 5.3 本地鉴权

本机 server 监听 `127.0.0.1` 仍需要鉴权（多用户主机场景）：

- **优先 Unix socket**，权限 0600，只有 owner 能连。
- 否则 loopback TCP + token：server 启动时生成随机 token 写 `~/.mfs/server.token` (0600)，CLI 读这个文件作为 Bearer。
- Windows 用 named pipe + ACL。

### 5.4 自动启动

默认**显式启动**（用户跑 `mfs serve start`）。`MFS_AUTOSTART=1` 时 CLI 检测不到本机 server 会自动 spawn 一次。

### 5.5 Job 队列：用关系型 DB 做队列

MFS 的 job 队列**直接用 metadata DB 表**（`connector_jobs` + `object_tasks`），不引入 Redis / RabbitMQ / Celery。

理由：

| 维度 | DB 队列（Postgres + SKIP LOCKED） | 外部 broker（Redis/RabbitMQ/Celery） |
|---|---|---|
| 部署组件 | 零额外（复用 metadata DB） | 多一个 broker（装/监控/HA） |
| 一致性 | 事务内 enqueue + state 一起 commit | 跨系统，需要 outbox 才一致 |
| 持久化 | ACID 天然 | Redis 默认非持久；要配 AOF |
| 可观测 | SQL 直接查 job 表 | Redis MONITOR / RabbitMQ UI |
| Schema 演进 | 加字段 / 加索引 | 改 JSON 结构 / 改 hash key |
| 吞吐上限 | Postgres 单实例几千-几万 op/s | Redis 几十万 op/s |
| 本地 daemon | SQLite 直接复用 | 用户要装 Redis ❌ |
| Celery 绑 Python | n/a | 跨语言 worker（如 Rust）受限 |

业界先例：GitLab / Sentry / Trigger.dev / Inngest / Hatchet 都用 Postgres + `SKIP LOCKED`；专用库有 `pgmq`、River。Airflow / Dagster 任务调度也用关系型 DB。

MFS 的瓶颈不是队列吞吐——**embedding API rate limit、LLM 速率、Milvus 写入吞吐**才是上限。任务规模典型每天几十到几万 task，远低于 Postgres 上限。

**未来 escape hatch**：`connector_jobs` + `object_tasks` 表当稳定 API；如果真撑不住，broker 换 Redis / NATS 作为中间派发器，**表 schema 不变**，迁移路径可控。

### 5.6 Worker 模型与 batching

DB 队列不阻止 batch，关键是 **batching layer** 显式分两层放：

**第一层：worker 一次拉 N 个 task**（减少 DB round-trip + Milvus batch INSERT）

```python
async def worker_loop():
    while True:
        tasks = await db.fetch_pending(limit=BATCH_SIZE)   # SELECT ... FOR UPDATE SKIP LOCKED
        if not tasks:
            await asyncio.sleep(POLL_INTERVAL_MS / 1000)
            continue

        all_chunks = []
        for t in tasks:
            chunks = await chunk_object(t)                  # read + cache + chunk
            all_chunks.extend(chunks)

        # batch embedding（API 一次或几次调用）
        vecs = await embedding_client.batch_embed([c.content for c in all_chunks])
        for c, v in zip(all_chunks, vecs):
            c.dense_vec = v

        # batch Milvus INSERT（一次 RPC 写几百到几千行）
        await milvus.batch_upsert(all_chunks)

        await db.mark_succeeded([t.id for t in tasks])
```

**第二层：API client 内的 micro-batcher**（处理高并发的小请求合并）

```python
class BatchingEmbeddingClient:
    """攒到 max_batch 或超时就 flush，对 worker 透明。
    多个 asyncio 并发 await embed() 会自动合并成一次 API 调用。
    """
    async def embed(self, text: str) -> Vector: ...
```

DataLoader pattern。embedding / summary / VLM 三类外部 API 都用这个模式。Milvus 不需要 client batcher（worker 显式 batch 写入）。

**不引入 staging 表**（如 `chunks_pending`）。Staging 表会让 chunk lifecycle 从"object_task 内串行"变成"跨表状态机"，复杂度收益不值。

### 5.7 一致性三条规则

整套 sync 的正确性靠这三条保证：

**1. Chunk-level 幂等**

```
chunk_id = sha1(object_uri + locator + chunk_kind)    ← 确定性 hash
写 chunk = DELETE WHERE chunk_id = X + INSERT new row
```

任何 worker / 任何重试 / 任何并发，对同 chunk_id 的写都等效。

**2. Per-object 原子**

```
object_task.status = 'succeeded'
   ↔ 该 object 的所有 chunks 都写入 Milvus 且 cache 已更新
```

中途任一步失败 → object_task 保持 'running' 或退 'failed'，下次 sync 重试整个 object。

**3. State 末尾提交**

```
connector.sync() 过程中的 self.state.set(...) 写入暂存（connector_jobs.state_snapshot）
只有当 sync_job 所有 object_task 成功时才 commit 到 connector_state 表
```

中途崩溃 → state 不 commit → 下次 sync 从上次成功的 state 重启。`connector.sync()` 必须 idempotent（同 state 输入 → 同 ObjectChange 输出，加上新数据）。

framework 不暴露 `commit()` 给 connector——commit 时机由 framework 控制。

### 5.8 故障恢复

daemon / worker 重启时扫一次：

```sql
-- 心跳超时的 sync_job 标 failed
UPDATE connector_jobs   SET status='failed', error='interrupted'
  WHERE status='running' AND heartbeat < now() - interval '5 minutes';

-- 对应 object_tasks 重置为 pending（依赖幂等性，再次被 worker 拉走重跑）
UPDATE object_tasks SET status='pending'
  WHERE status='running'
    AND connector_job_id IN (SELECT id FROM connector_jobs WHERE status='failed');
```

connector_state 因为没被 commit（state_snapshot 在 connector_jobs 里随 job 失败一起作废），下次 `mfs add` 自然从上一个成功的 state 接续。

**没有 `mfs job retry` 命令**——重跑 = 下次 `mfs add`，state 没 commit 时自然接续。

### 5.9 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新建 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force` | 同上，但所有 object 视为 'modified'，跳过 fingerprint 比对，强制重建 chunks |
| `mfs add <uri>` 在前一个 sync 失败后 | 前次 state 没 commit；从上上次的 state 重跑——失败的 object 自然重新出现在流里 |
| 第二次 `mfs add <uri>` 在前一个 sync 还 running | 返回 `sync_already_running, see job <id>`（被 UNIQUE 约束拒绝） |

### 5.10 Worker 并发与 batch 配置

`/etc/mfs/server.toml`（或 `~/.mfs/server.toml`）：

```toml
[worker]
concurrency = 4                  # 同时跑几个 worker（asyncio task）
batch_size = 50                  # 一个 worker 一次拉多少 object_task
poll_interval_ms = 200           # 队列空时轮询间隔
heartbeat_interval_s = 30        # worker 心跳频率

[embedding]
batch_size = 100                 # micro-batch 满 100 chunks 触发
batch_max_wait_ms = 100          # 或等 100ms

[summary]
batch_size = 20
batch_max_wait_ms = 500

[vlm]
batch_size = 10
batch_max_wait_ms = 500

[milvus]
insert_batch_size = 1000         # 单次 INSERT 上限
```

worker 自适应：根据上一轮平均 chunk 数动态调 `batch_size`，避免极大对象（单 task 出 10 万 chunks）和极小对象（单 task 1 chunk）的两种极端。

### 5.11 操作之间的并发协调

所有"对一个 connector 的操作"（sync / force_sync / remove / update_config）都统一进 `connector_jobs` 表，用 `op_kind` 区分。**一条 UNIQUE 约束 + 三条规则**覆盖所有并发场景。

#### 三条核心规则

**① 同种重复 → 拒绝（destructive 类除外，幂等）**

- `sync + sync` / `force_sync + force_sync` → 拒绝
- `remove + remove` → 幂等成功（目标状态就是"消失"）
- `update_config + update_config` → 拒绝

**② 不同种竞争 → `remove` 是 destructive superset，优先**

`remove` 永远能 preempt sync / force_sync / update_config。其他方向反过来不行：sync / force / update 都不能 preempt 已经 running 的 remove。

**③ 同方向不允许 preempt**

`sync` 中又来 `sync` / `force_sync` → 拒绝。理由：

- sync 已经在做你想做的事，preempt 没有收益
- force_sync 是 user-explicit destructive 操作，应该让用户显式 `mfs job cancel` 后再来——避免"以为只是 add 结果触发了全量重跑"

跟 git 的设计哲学一致：rebase / merge 中间不能再触发同类操作，必须先 `--abort`。

#### 完整语义表

| 当前 in-flight | 新来 | 行为 |
|---|---|---|
| 无 | 任意 | OK |
| `sync` | `sync` / `add <uri>` | 拒绝 `sync_already_running, see job <id>` |
| `sync` | `force_sync` / `add --force` | 拒绝 `sync_already_running`；提示先 `mfs job cancel` |
| `sync` | `remove` | preempt：sync 标 `cancelling`，当前 object_task 完成后退出 → remove 入队 → 跑 |
| `sync` | `update_config` | 拒绝：先等 sync 完成或先 cancel |
| `force_sync` | `sync` / `force_sync` | 拒绝 |
| `force_sync` | `remove` | preempt（同 sync） |
| `remove` | `add` / `sync` / `force_sync` | 拒绝 `connector_removing, retry after cleanup` |
| `remove` | `remove` | 幂等成功 `already removing, see job <id>` |
| `remove` | `update_config` | 拒绝 |
| `update_config` | `sync` | 拒绝；等 update 完成或 cancel |
| `update_config` | `remove` | preempt |

#### Sync 中 Remove 的具体流程

```
sync running on connector C
    │
    ▼ mfs remove C 来了
    │
    ▼ ① 检查 connectors.status='active'（否则按重复 remove 处理）
    ▼ ② connectors.status = 'removing'        ← 立刻设置；后续 add/sync 看到就拒绝
    ▼ ③ INSERT connector_jobs (op_kind='remove', status='queued')
    │   （UNIQUE 约束此时允许：sync 是 running，remove 是 queued）
    ▼ ④ 把 running 的 sync 标 cancelling
    ▼ ⑤ worker 在每个 object_task 边界检查 cancel signal
    │   当前 task 完成后退出（per-object 原子）
    ▼ ⑥ sync_job status → 'cancelled'
    ▼ ⑦ remove job 从 'queued' → 'running'
    ▼ ⑧ 跑 remove 流程：
    │     - Milvus: drop_partition(connector_uri)   ← 比 delete by filter 快很多
    │     - object store: 删 cache 文件
    │     - metadata DB: 删 caches / connector_state / objects / connector_jobs / object_tasks
    ▼ ⑨ connectors row DELETE
    ▼ ⑩ remove job status → 'succeeded'
```

清理顺序确保**幂等可重入**：如果 step ⑧ 中途崩溃，下次重启 worker 重跑 remove job，可以从任何一步开始（DROP 一次空 partition / 删空目录都是 no-op）。

用户那边 `mfs status C` 在 ④-⑥ 期间看到：

```
Connector: postgres://prod
Status:    removing
Current job: job_remove_xx (queued)
  waiting for: job_sync_yy (cancelling)
```

#### Worker 端的 cancel 检查

worker 在两个地方检查 cancel signal：

```python
async def process_object_task(task):
    if await is_cancelled(task.connector_job_id):
        task.status = 'cancelled'
        return

    # 整 task 是原子单元，中途不打断（per-object 原子规则）
    await do_work(task)
    task.status = 'succeeded'
```

对单 task 耗时极长的场景（一个 object 出 10 万 chunk），可在 chunk 批次边界加细检查点：

```python
for chunk_batch in chunks.chunked(BATCH):
    if await is_cancelled(task.connector_job_id):
        task.status = 'cancelled'
        return    # 整 task 算 cancelled，已写入的 chunks 留着（下次 sync 幂等覆盖）
    await embed_and_write(chunk_batch)
```

`is_cancelled()` 查询是 in-memory cached（每 N 秒刷新一次），不每次都打 DB。

#### Scheduler / Watcher 的协调

定时同步和 watch 触发的 sync 也通过同一个表入口。`connectors.status='removing'` 时：

- scheduler 看到就**跳过**这个 connector
- daemon 内 file watcher 检测到 `status='removing'` 立刻**停止该 root 的 watcher**
- 即使刚刚 race 触发了一个 sync，会被 UNIQUE 约束拒绝或被随后的 cancel 流程清掉

## 6. 存储层

三套存储，职责清晰；每套的具体后端独立可换。

### 6.1 Metadata DB

local: SQLite `~/.mfs/metadata.db`
remote: Postgres

Schema:

```sql
connectors (
  id              VARCHAR PRIMARY KEY,
  workspace_id       VARCHAR DEFAULT 'default',
  root_uri        VARCHAR,
  type            VARCHAR,
  label           VARCHAR,
  status          VARCHAR DEFAULT 'active',   -- 'active' | 'removing'
  config_json     TEXT,
  config_hash     VARCHAR,
  credential_ref  VARCHAR,
  registered_at   TIMESTAMP,
  last_health     TIMESTAMP,
  health_status   VARCHAR,
  UNIQUE (workspace_id, root_uri)
);

objects (
  connector_id    VARCHAR REFERENCES connectors(id),
  object_uri      VARCHAR,
  parent_path     VARCHAR,                  -- ls 用
  type            VARCHAR,                  -- "file" | "dir"
  media_type      VARCHAR,
  size_hint       INTEGER,
  extra_json      TEXT,
  fingerprint     VARCHAR,
  indexable       BOOLEAN,
  capabilities    TEXT,                     -- JSON
  last_seen       TIMESTAMP,
  PRIMARY KEY (connector_id, object_uri),
  INDEX (connector_id, parent_path)
);

caches (
  object_uri      VARCHAR,
  cache_kind      VARCHAR,
  storage_path    VARCHAR,                  -- ~/.mfs/cache/<sha1>/<kind> 或 s3 key
  fingerprint     VARCHAR,
  size_bytes      INTEGER,
  built_at        TIMESTAMP,
  last_accessed   TIMESTAMP,
  PRIMARY KEY (object_uri, cache_kind)
);

-- ===== Job 队列：统一所有 connector op =====
connector_jobs (
  id                    VARCHAR PRIMARY KEY,
  workspace_id             VARCHAR DEFAULT 'default',
  connector_id          VARCHAR REFERENCES connectors(id),
  op_kind               VARCHAR,        -- 'sync' | 'force_sync' | 'remove' | 'update_config'
  trigger               VARCHAR,        -- 'manual' | 'scheduled' | 'watch'
  status                VARCHAR,        -- 'queued' | 'running' | 'cancelling' | 'cancelled' | 'succeeded' | 'failed'
  started_at            TIMESTAMP,
  finished_at           TIMESTAMP,
  heartbeat             TIMESTAMP,      -- worker 心跳，用于检测 crash
  total_objects         INTEGER,        -- 仅 op_kind IN ('sync','force_sync')
  succeeded_objects     INTEGER,
  failed_objects        INTEGER,
  cancelled_objects     INTEGER,
  error                 TEXT,           -- 顶层失败原因
  state_snapshot        TEXT,           -- pending：暂存的 connector state，sync 末尾才 commit
  -- 关键约束：同 connector 同时只能有一个 in-flight op
  UNIQUE (connector_id) WHERE status IN ('queued', 'running')
);

object_tasks (
  id                    VARCHAR PRIMARY KEY,
  connector_job_id      VARCHAR REFERENCES connector_jobs(id),   -- 仅 sync/force_sync 类 job 有 task
  object_uri            VARCHAR,
  change_kind           VARCHAR,        -- 'added' | 'modified' | 'deleted'
  status                VARCHAR,        -- 'pending' | 'running' | 'succeeded' | 'failed' | 'cancelled'
  attempts              INTEGER DEFAULT 0,
  last_error            TEXT,
  started_at            TIMESTAMP,
  finished_at           TIMESTAMP,
  INDEX (connector_job_id, status),
  INDEX (status, started_at) WHERE status = 'running'   -- 心跳超时检测
);

-- ===== Connector 内部 state =====
connector_state (
  connector_id          VARCHAR,
  key                   VARCHAR,        -- connector 自定义的 key（cursor/manifest 等）
  value                 TEXT,           -- JSON-serializable，schema 由 connector 自定义
  updated_at            TIMESTAMP,
  PRIMARY KEY (connector_id, key)
);

watch_grants (
  path            VARCHAR PRIMARY KEY,
  granted_at      TIMESTAMP
);

-- ===== Upload flow（client → server 上传本地文件） =====
upload_manifests (                          -- server 端存的 client manifest 快照
  connector_id    VARCHAR REFERENCES connectors(id),
  path            VARCHAR,
  size            INTEGER,
  mtime_ns        BIGINT,
  sha1            VARCHAR,
  last_commit_at  TIMESTAMP,                -- 最后一次 commit 成功时间
  PRIMARY KEY (connector_id, path)
);

upload_staging (                            -- 当前未 commit 的临时 upload
  temp_file_id    VARCHAR PRIMARY KEY,
  connector_id    VARCHAR,
  storage_path    VARCHAR,                  -- object_store://uploads/<connector_id>/<temp_file_id>.zip
  size_bytes      INTEGER,
  uploaded_at     TIMESTAMP,
  expires_at      TIMESTAMP                 -- 自动清理过期未 commit 的（默认 1h）
);
```

注意所有顶层表都预留 `workspace_id` 字段，默认 `'default'`，便于未来加多租户。

### 6.2 Object store (cache + upload staging)

object store 同时存两类东西：

- **cache 文件**：connector 拉取的对象缓存（converted_md / page_cache / vlm_text / schema_dump 等）
- **upload staging**：client 上传的 zip bundle + 解压后的真实文件树（仅 §3.5 upload flow 用）

#### 默认后端：本地文件系统

**默认 `backend = local`**，最快、零运维：

```
~/.mfs/cache/
  caches/<sha1(object_uri)>/
    converted_md
    page_cache.jsonl
    head_cache.jsonl
    vlm_text
    schema_dump.json
  uploads/
    <connector_id>/
      <temp_file_id>.zip                # bundle，commit 后删
  files/
    <connector_id>/                     # 解压后的真实文件树，连续作为 cache
      src/cli.py
      README.md
      ...
```

`caches` 表存 `(object_uri, cache_kind) → storage_path` 映射；`upload_staging` 表存 temp_file_id → 路径。

#### 后端选择 trade-off

| 后端 | first byte 延迟 | 多 worker 共享 | 持久化 | 适合 |
|---|---|---|---|---|
| `local` （fs） | ~1ms | ❌（仅同进程） | 跟随 disk | 个人本机、单容器 server |
| `minio` | ~5-10ms | ✅ | 跟随 volume | Docker Compose 多 worker |
| `s3` / `r2` / `gcs` | ~50-100ms | ✅ | HA、跨实例 | K8s 生产部署 |

S3 延迟看起来比本地 fs 慢，但仍**远比 connector 重新拉取外部 API（100ms+）快**。对 throughput 影响小（streaming）。

**部署建议**：

| 部署形态 | object_store backend |
|---|---|
| 个人本机 `mfs serve` | `local` |
| Docker 单容器 server | `local`（bind volume `/data/cache`） |
| Docker Compose 多 worker | `minio`（容器间共享） |
| K8s 生产 | `s3` / `r2`（HA + 多 replica） |

不做"S3 + 本机磁盘两级 cache"——v0.4 范围用户察觉不到 50ms 区别，复杂度收益不值。

#### 淘汰与配额

```toml
[cache]
max_size_gb = 10
eviction = "lru"

[upload]
max_bundle_size_mb = 500             # 单 bundle 上限；超过用独立单文件 PUT
staging_path = "uploads/"            # object store 下的 staging 子路径
staging_expiry_hours = 1             # 未 commit 的 staging zip 多久过期清理
per_workspace_quota_gb = 0           # 0 = 不限；多 workspace 部署可设
```

### 6.3 Milvus

一张 collection `mfs_chunks`，partition by connector。详见 [06-search-and-retrieval.md §1](06-search-and-retrieval.md#1-milvus-collection-schema)。

backends：

| 后端 | URI | 适合 |
|---|---|---|
| Milvus Lite | `~/.mfs/milvus.db` | local daemon 默认；零运维；单 writer |
| 自部署 Milvus | `http://host:19530` | self-host；多 writer；完整 BM25 |
| Zilliz Cloud | `https://*.zillizcloud.com` + token | 托管；多 writer；完整 BM25 |

**Backend 能力矩阵**：MFS v0.4 依赖几个 Milvus 特性，三种 backend 支持情况不同：

| 能力 | Lite | 自部署 Milvus 2.5+ | Zilliz Cloud |
|---|---|---|---|
| `partition_key` 字段 | ⚠️ 简化版（按 hash 分区，无显式 partition 管理） | ✅ | ✅ |
| `drop_partition` 快速删除 | ❌（fallback 到 delete by filter） | ✅ | ✅ |
| `sparse_vec` + BM25 内建 | ✅（2.5+） | ✅ | ✅ |
| `scalar index` on `connector_uri` / `chunk_kind` | ✅ | ✅ | ✅ |
| `JSON` filter on `metadata` | ✅ | ✅ | ✅ |
| 多 writer 并发 | ❌（单进程独占） | ✅ | ✅ |
| 横向扩展 | ❌ | ✅（需配 Pulsar/Kafka） | ✅（托管） |
| 备份 / 快照 | 文件 cp | manual / operator | 内建 |

**实际影响**：

- **Lite** 在 local daemon 场景够用。`mfs remove` 在 Lite 上走 `delete by filter` 而不是 `drop_partition`（慢但能用）；多 worker 并发受限，但 local daemon 默认就是单 worker pool。
- **自部署 / Zilliz Cloud** 才能跑 `mfs-worker` 多 replica。
- 文档默认假设 **Milvus 2.5+**（sparse_vec / BM25 必需）。

切 backend 时数据迁移用 `mfs admin migrate-milvus --to <new-uri>`（roadmap 工具，v0.4 手动做）。

如果用户用 Lite 起手，后期切自部署：drop_partition 突然变快，多 writer 解锁——无 schema 改动，纯 backend 替换。

server 端配置：

```toml
[milvus]
uri = "~/.mfs/milvus.db"               # 默认 Lite
# uri = "http://localhost:19530"
# uri = "https://xxx.zillizcloud.com"
# token = "..."
collection_strategy = "single"          # single | per_connector | per_tenant
```

## 7. 凭据与认证

### 7.1 Connector 凭据

connector TOML 用 `credential_ref` 引用，不写明文：

```toml
credential_ref = "env:PG_PROD_DSN"
credential_ref = "secret:pg-prod-readonly"
credential_ref = "file:~/.mfs/secrets/pg-prod.toml"
credential_ref = "vault:secret/data/mfs/pg-prod"
```

解析优先级：

1. `env:VAR` — 环境变量
2. `secret:NAME` — OS keychain（macOS Keychain / Linux secret-service / Windows Credential Manager）
3. `file:PATH` — 本地文件（默认权限 0600）
4. `vault:PATH` — HashiCorp Vault（remote profile 主流）

凭据 schema 由 connector plugin 定义（PostgresCredential / SlackCredential / ...）。

### 7.2 Remote server 认证

```toml
[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_API_TOKEN"
```

请求头：

```text
Authorization: Bearer <token>
```

`mfs login` 命令（v0.4 不内置）属于商业化能力。

## 8. 部署形态

### 8.1 个人本机

```bash
uv tool install mfs-cli
uv tool install mfs-server
mfs serve start
mfs profile add local --url http://127.0.0.1:8765
mfs profile use local
mfs add .
```

默认存储：

- Metadata DB: `~/.mfs/metadata.db` (SQLite)
- Object store: `~/.mfs/cache/` (本地文件)
- Milvus: `~/.mfs/milvus.db` (Lite)

### 8.2 单容器 server

适合 demo、小规模自部署。

```bash
docker run --rm -p 8765:8765 \
  -v mfs-data:/data \
  ghcr.io/zilliztech/mfs-server:0.4.0
```

容器内 API + worker 同进程；存储用容器内 SQLite + Milvus Lite。

### 8.3 Docker Compose（正式自部署）

```yaml
services:
  postgres:    image: postgres:16
  minio:       image: minio/minio
  redis:       image: redis:7
  milvus:      image: milvusdb/milvus:v2.5.0
  mfs-api:     image: ghcr.io/zilliztech/mfs-api:0.4.0
  mfs-worker:  image: ghcr.io/zilliztech/mfs-worker:0.4.0
```

```bash
docker compose up -d
mfs profile add prod --url https://mfs.example.com
mfs add postgres://prod --config ...
```

### 8.4 Kubernetes

```bash
helm install mfs ./charts/mfs \
  --set api.replicas=2 \
  --set worker.replicas=4 \
  --set objectStore.type=s3 \
  --set search.type=zilliz \
  --set metadata.type=postgres
```

## 9. 多租户预留

v0.4 **不实现多租户**，但 schema 全部预留。多租户指**一个 MFS 实例服务多个 organization / workspace**——每个租户的数据完全物理隔离。

### 预留点

- Metadata DB 所有顶层表加 `workspace_id` 字段（默认 `"default"`）
- Milvus `mfs_chunks` 表加 `workspace_id` scalar field（默认 `"default"`）
- 所有 query 默认 filter `workspace_id = current_workspace`
- HTTP header 预留 `X-MFS-Workspace`（v0.4 忽略；多租户启用后由 profile 的 `workspace_id` 自动注入）
- Profile 配置已支持 `workspace_id` 字段（v0.4 写了不报错，server 端忽略；启用多租户后生效）

### 未来启用方案

启用时 Milvus 隔离方案（按隔离强度排序）：

| 策略 | Milvus 实现 | 隔离强度 | 资源开销 |
|---|---|---|---|
| metadata filter | 共用 collection，每行带 `workspace_id`，查询时 filter | 弱（依赖 query 正确性） | 最小 |
| partition by tenant | collection 内按 tenant 分 partition | 中 | 中 |
| collection per tenant | 每 tenant 一张 collection | 强 | 大 |
| database per tenant | 每 tenant 一个 Milvus database（2.4+） | 最强 | 最大 |

推荐路径：v1.0 加多租户时切到 **collection per tenant**（`[milvus] collection_strategy = "per_tenant"`），通过迁移工具从 single → per_tenant。

### 数据量大时的策略

| 量级 | 策略 |
|---|---|
| < 几百万 chunks | 默认 single collection 够 |
| 几千万 chunks | 调 Milvus shard / 升级 HNSW 参数；考虑分 partition |
| 亿级 chunks | 切 `per_connector` 或 `per_tenant` collection；冷热分层 |

## 10. 镜像与包

| 交付物 | 内容 | 入口 |
|---|---|---|
| PyPI `mfs-cli` | CLI、Python SDK、HTTP transport、profile、输出 | `mfs` |
| PyPI `mfs-server` | API、daemon、worker、engine、connectors、objects、pipeline、storage | `mfs-server` |
| Docker `mfs-server` | 单容器：API + worker | demo / 小规模 |
| Docker `mfs-api` | 只跑 API | 正式部署 |
| Docker `mfs-worker` | 只跑 worker | 正式部署 |

便利 extra：

```bash
uv tool install "mfs-cli[local]"     # 自动连带 mfs-server
```

server optional extras（按 connector 安装）：

```text
mfs-server[postgres]
mfs-server[mysql]
mfs-server[mongo]
mfs-server[bigquery]
mfs-server[snowflake]
mfs-server[slack]
mfs-server[discord]
mfs-server[gmail]
mfs-server[gdrive]
mfs-server[feishu]
mfs-server[s3]
mfs-server[github]
mfs-server[linear]
mfs-server[jira]
mfs-server[notion]
mfs-server[salesforce]
mfs-server[hubspot]
mfs-server[zendesk]
mfs-server[web]
mfs-server[embedding-onnx]
mfs-server[embedding-google]
mfs-server[llm-anthropic]
mfs-server[llm-google]
mfs-server[zilliz]
mfs-server[all]
```

多语言 SDK：npm / Go module / Maven，基于 `protocol/openapi.yaml` 生成。

## 11. 工程目录结构

```
.
├── clients/
│   └── python/                              # PyPI: mfs-cli
│       └── src/mfs_client/
│           ├── cli/
│           │   └── commands/
│           │       ├── add.py
│           │       ├── connector.py
│           │       ├── profile.py
│           │       ├── daemon.py
│           │       ├── job.py
│           │       └── ...                  # search/grep/ls/tree/cat/head/tail/export/status/remove/config
│           ├── sdk/
│           ├── transport/
│           ├── models/
│           └── config/
│
├── protocol/
│   ├── openapi.yaml
│   ├── schemas/
│   └── errors.md
│
├── server/
│   └── python/                              # PyPI: mfs-server
│       └── src/mfs_server/
│           ├── api/
│           │   ├── app.py
│           │   ├── routes/
│           │   │   ├── add.py
│           │   │   ├── connectors.py
│           │   │   ├── objects.py
│           │   │   ├── search.py
│           │   │   └── jobs.py
│           │   └── middleware/
│           ├── daemon/                       # mfs-server daemon entrypoint
│           ├── worker/                       # mfs-server worker entrypoint
│           ├── engine/
│           ├── connectors/                   # 每类 connector 自包含
│           │   ├── base.py                   # ConnectorPlugin 抽象
│           │   ├── registry.py
│           │   ├── file/
│           │   ├── web/
│           │   ├── postgres/
│           │   ├── slack/
│           │   ├── github/
│           │   └── ...
│           ├── objects/                      # object_kind handlers
│           │   ├── base.py
│           │   ├── document/
│           │   ├── code/
│           │   ├── table_rows/
│           │   ├── message_stream/
│           │   ├── record_collection/
│           │   ├── image/
│           │   └── binary/
│           ├── pipeline/
│           │   ├── embedding/
│           │   ├── summary/
│           │   ├── vlm/
│           │   ├── retrieval/
│           │   └── export/
│           ├── storage/
│           │   ├── metadata/
│           │   ├── object_store/
│           │   ├── queue/
│           │   └── search/                   # Milvus adapter
│           └── runtime/
│               ├── local_daemon.py
│               └── remote_server.py
│
├── deployments/
│   ├── docker/
│   ├── compose/
│   └── helm/
│
├── tests/
│   ├── client/
│   ├── server/
│   ├── connectors/
│   ├── e2e/
│   └── fixtures/
│
├── docs/
└── skills/
```

## 12. 版本策略

```text
0.x.y    快速迭代
1.0.0    CLI / local daemon / 多 connector 基础行为 / API v1 / Milvus schema 全部稳定
```

兼容面：

- `mfs` CLI 命令和参数（16 个顶级命令）。
- Python SDK 方法和错误类型。
- HTTP API `/v1`（additive only；breaking 进 `/v2`）。
- JSON envelope。
- connector URI 与对象命名约定（带 media type 后缀）。
- connector TOML + chunk_kind 枚举 + locator schema。
- Milvus collection schema（含 `workspace_id` 多租户预留字段）。
- server image entrypoint。

CLI/server 兼容关系写入 release note：

```text
MFS CLI 0.4.x supports MFS API v1.
Minimum server version: 0.4.0.
```

## 13. 运维可观测

`mfs status` 是用户面入口。后台指标（Prometheus / OpenTelemetry）：

- connector healthcheck 通过率
- sync lag
- job queue depth / latency
- worker heartbeat
- API latency
- search latency
- embedding cost (token / $)
- storage 用量
- cache hit rate
- Milvus query QPS / latency

operation log 存 `~/.mfs/audit.log` (local) 或 server 侧 audit table。

## 14. 实现语言选择

**主体 Python + 性能模块 Rust 的混合方案**。

| 模块 | 语言 | 理由 |
|---|---|---|
| CLI / Python SDK | Python | 跟 server 统一；启动慢用 lazy import |
| API server / engine | Python + FastAPI + asyncio | IO-bound 为主；asyncio + FastAPI 性能足够 |
| Connectors | Python | 所有外部 SDK（postgres / slack / github / gdrive / openai / anthropic / google 等）first-class 是 Python |
| Embedding / LLM / VLM 调用 | Python | provider SDK 都是 Python first-class |
| Milvus 客户端 | pymilvus | 官方 |
| 大目录扫描 + hash | Rust（PyO3 绑定） | 千万文件场景，Python `os.walk` 太慢 |
| 大 JSONL / CSV / Parquet 流处理 | Rust（polars / pyarrow） | 内存稳定，零拷贝 |
| AST tree-sitter 切分 | 已经 Rust | 沿用 tree-sitter 官方绑定 |
| 高并发 grep 线性扫 | Rust（可选优化） | 性能敏感场景 |
| 多语言 SDK（贡献者用） | TypeScript / Go / Java | 基于 `protocol/openapi.yaml` 生成 |

### 不推荐的方案

- **Go 主体**：connector SDK 不齐（slack / linear 等 SaaS 有但 LLM/embedding 是二级公民），写起来比 Python 慢。
- **Rust 整体重写**：开发速度太慢；社区贡献 connector 的门槛太高（每个 connector 多了 1-2 天学习成本）。

### CLI 启动速度

Python CLI 冷启动通常 200-500ms。缓解：

- lazy import：CLI 入口只 import argparse + 命令分发，子命令的 heavy import 延迟到执行时。
- 编译单 binary：`pyapp` / `pex` 把 Python 运行时和依赖打包成单 binary。
- 后期可出 Rust CLI launcher（轻量 binary，fork 出 Python worker 处理真请求）。

v0.4 直接 Python ship；性能瓶颈出现后再插 Rust 模块。

### 性能模块的边界

Rust 模块封装在 server 内部，**不影响 connector 贡献者**——connector 全 Python。Rust 只在以下边界出现：

- `mfs_server.io.scan` — 大目录扫描
- `mfs_server.io.jsonl` — JSONL/CSV 流式处理
- `mfs_server.grep.linear` — 线性扫优化

这些模块对外是普通 Python 函数（PyO3 绑定），调用方无感。
