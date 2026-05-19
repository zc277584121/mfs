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
│  │ Object Processors (processors/<kind>/)                       │  │
│  │   document / code / table_rows / message_stream /            │  │
│  │   record_collection / image / binary                         │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Common Services                                              │  │
│  │   embedding / summary / VLM / retrieval / export             │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ DB-backed queue                                              │  │
│  │   connector_jobs + object_tasks tables                       │  │
│  │   SELECT ... FOR UPDATE SKIP LOCKED 取 task                 │  │
│  │   SQLite (local) or Postgres (remote)                        │  │
│  └────────────┬─────────────────────────────────────────────────┘  │
│               v                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Storage                                                      │  │
│  │   Metadata DB (SQLite / Postgres)                            │  │
│  │   Object store (local fs / S3 / R2 / MinIO)                  │  │
│  │   Milvus (Lite / self-hosted / Zilliz Cloud)                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 术语速览（分 3 组）

读不同章节需要的词不一样，**按受众分组**避免一次塞太多：

#### A. 对外抽象（用户 + 开发者都用）— 6 个

| 术语 | 是什么 |
|---|---|
| **Connector** | 一个注册的数据源插件实例（`postgres://prod` / `./repo`） |
| **Object** | connector 暴露的一条虚拟文件（URI + media_type） |
| └─ **Cache** | object 的派生：字节缓存（可选；加速 cat / head / tail） |
| └─ **Chunks** | object 的派生：Milvus 行（被 search / grep 召回） |
| **Profile** | client 端 endpoint 配置（连哪个 server + token） |
| **Job** | 用户操作记录：一次 `mfs add` / `mfs remove` → 一个 job（`connector_jobs` 表） |

抽象关系：`Connector` 暴露多个 `Object`；每个 `Object` 可能有 `Cache` 和 `Chunks` 作为派生产物。`Profile` 决定 client 连哪个 server。用户每次操作（mfs add / remove）创建一个 `Job`。

#### B. 队列系统（开发者读 §5 队列章节）— 2 个

| 术语 | 是什么 |
|---|---|
| **Task** | Job 内的子单元——每个变化的 object → 一个 task（`object_tasks` 表） |
| **Worker** | 跑 task 的 coroutine（业界标准词，跟 Sidekiq / Celery / Airflow / GHA 一致） |

> 注：DB 表（`connector_jobs` + `object_tasks`）起到队列容器的作用——文档里说"DB-backed queue"做形容词用，**不把 Queue 当独立术语**。

#### C. Server 代码层（开发者读 §11 工程目录）— 6 个

| 层名 | 职责 |
|---|---|
| **HTTP API** | FastAPI routes（`/v1/...`） |
| **Engine** | 业务编排：路由请求 → 创建 job → 调 connector；同一份代码兼容本机 / 远端两种部署 |
| **Connectors** | per-source 插件（file / postgres / slack / github / ...） |
| **Object Processors** | per-object_kind 加工（chunker / VLM 调用 / structure 构建） |
| **Common Services** | 通用工具集（embedding / summary / vlm / retrieval / export） |
| **Storage** | metadata DB / object store / Milvus 三套后端（adapter pattern 是实现细节） |

> **内部分区主键 `namespace_id`**（用户看不到）
> 所有数据表 + Milvus partition + object store prefix 都按 `namespace_id` 切。v0.4 只有一个 `default` namespace，用户感知不到。未来加 Workspace / User / Project 时通过 mapping 表实现，**底层 namespace 不动**——详见 [§9](#9-多租户与-namespace-设计)。

读者范围：

| 你是谁 | 需要学的术语 |
|---|---|
| **agent / 用户** | 只学 A 组（6 个） |
| **后台 Python 开发者** | A + B 组（8 个） |
| **读 server 架构 / 工程目录** | A + B + C 组（共 14 个，但 C 组只是层标签，自描述） |

每组都有明确读者 + context，单次记忆负担不超过 7 个。

## 2. Profile 与存储后端是正交的

### 2.0 配置文件命名约定

MFS 有**两个独立的配置文件**，各自负责不同身份的设置：

| 文件 | 路径 | 内容 | 谁读 |
|---|---|---|---|
| client 配置 | `~/.mfs/client.toml` | profiles、endpoint URL、API token | `mfs` CLI |
| server 配置 | `~/.mfs/server.toml`（本地 daemon）<br>`/etc/mfs/server.toml`（远端部署） | metadata backend / object_store / milvus / worker / embedding / chunker / cache / summary / vlm | `mfs-server` |

只装 CLI（`mfs` binary）的用户只接触 `client.toml`；只装 `mfs-server`（运维端）的人只接触 `server.toml`；两者都装（个人本机）的人各自维护。两个文件不会混在一起，schema 不会冲突。

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
client_id       = "01HX...XYZ"      # 首次启动自动生成的 UUIDv7；用于 connector_uri
                                    # 不要手动改——改了会导致已注册的 file connector 变孤儿

[[profiles]]
name = "local"
api_base_url = "http://127.0.0.1:8765"

[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_API_TOKEN"
```

不需要写 `kind` 字段，CLI 跟 server 握手时自动判断 `is_local`，结果缓存到 client.toml。

#### `client_id` 跟 machine-id 的分工

```
machine-id  ─┐
             ├─ 只用来判定 is_local（client/server 是否共享 fs）
client_id   ─┘  client.toml 持久化的随机 UUID（uuid7）
                进 connector_uri，作为"哪个 client 注册了这个 file connector"的稳定标识
```

`client_id` 在 CLI 首次启动 `ensure_mfs_home()` 时生成（`uuid.uuid7()`）写入 `client.toml`。**它跟 machine-id 解耦**，因为 machine-id 不稳定：

| 场景 | machine-id | client_id | connector 状态 |
|---|---|---|---|
| 同台机器持续使用 | 不变 | 不变 | 一直认识 |
| Docker 容器重启（无 volume 挂载 `~/.mfs`） | 每次新生成 | 每次新生成 → file connector 重复注册 | ⚠️ Docker 用户要把 `~/.mfs` 挂成 volume |
| Docker 容器重启（有 volume 挂载 `~/.mfs`） | 每次新生成 | client.toml 持久化，不变 | ✅ 仍认识之前的 connector |
| 笔记本重装系统但备份过 `~/.mfs` | 变了 | 不变 | ✅ 仍认识之前的 connector |
| 多机访问同一 NFS `~/repo` | 各自不同 | 各自不同（各自的 client.toml） | 各自独立 file connector（避免多 client 抢写同一 connector） |
| `systemd-machine-id-setup --commit` | 变了 | 不变 | ✅ 不受影响 |

`§3.5` 的 connector_uri 构造从 `file://<machine-id>/<path>` 改为 `file://<client_id>/<path>`。

profile 管理：

```bash
mfs profile add <name> --url <url>
mfs profile use <name>
mfs profile list
mfs profile status               # 显示当前 profile 是否本地（machine-id 比对结果）
```

### 多租户走 profile 层（v0.5+）

v0.4 server 端只有一个隐式 `default` namespace，client 不需要任何租户配置。

v0.5+ 引入多 namespace 后，namespace 选择**由 token 决定**——一个 token 在 server 端绑定一个或多个 namespace 的访问权限（详见 [§9](#9-多租户与-namespace-设计)）。client 端 profile 不需要新增字段，**用户通过切换 profile（= 换 token）切换租户**：

```toml
[[profiles]]
name = "acme-corp-prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_TOKEN_ACME"     # token 绑定 acme-corp 的 namespace

[[profiles]]
name = "globex-corp-prod"
api_base_url = "https://mfs.example.com"
credential_ref = "env:MFS_TOKEN_GLOBEX"   # token 绑定 globex-corp 的 namespace
```

`mfs profile use acme-corp-prod` 切换 = 切租户。同一台 client 可注册多个租户的 profile，按名字切换。connector URI 永远保持纯净（`postgres://prod`），不带租户信息。

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

client 端发的 `connector_uri` 形如 `file://<client_id>/<abs-path>`。`<client_id>` 是 client 端 `client.toml` 里持久化的 UUIDv7（首次启动 CLI 自动生成），让多个 client 不会撞同一 alias。一个 `connector_uri` 一辈子绑定一个 client（v0.4 禁止"多 client 共写同一 connector"——避免 manifest race）。

**为什么不用 machine-id**：machine-id 在 Docker 容器、`systemd-machine-id-setup --commit`、系统重装等场景会变，会导致已注册的 file connector 变孤儿。`client_id` 持久化在 `client.toml`，跟着用户数据走（用户备份 `~/.mfs` 就 OK），稳定。`client.toml` 跟 docker volume 一起挂载就解决了 Docker 场景。详见 [§2 client_id 跟 machine-id 的分工](#client_id-跟-machine-id-的分工)。

用户面仍然是 `mfs add ./repo`，CLI 自动规范化成 `file://<client_id>/<abs-path>`。

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

如果只装了 `mfs` CLI 没装 `mfs-server`：

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

> **本节用到的三个词**：
> - **Job** = 用户层操作记录（一次 mfs add / mfs remove → 一行 `connector_jobs`）
> - **Task** = Job 的子单元（每个变化的 object → 一行 `object_tasks`）
> - **Worker** = 从 DB 拉 task 跑的 coroutine

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

worker 从 DB 拉 task 时**按 `priority` 升序**，相同 priority 按入队时间。**SQL 形态因后端而异**：

**Postgres 路径（远端部署 / 多 worker 并发）**：

```sql
SELECT * FROM object_tasks
WHERE status = 'pending' AND connector_job_id = $1
ORDER BY priority ASC, started_at ASC NULLS FIRST
FOR UPDATE SKIP LOCKED
LIMIT $batch_size
```

`FOR UPDATE SKIP LOCKED` 是 PG 特有语法——多 worker 各拉各的批不冲突。

**SQLite 路径（本机 daemon）**：SQLite **不支持** `FOR UPDATE / SKIP LOCKED`。改成两步事务：

```sql
BEGIN IMMEDIATE;                                    -- 抢全库写锁

SELECT id FROM object_tasks
WHERE status = 'pending' AND connector_job_id = ?
ORDER BY priority ASC, started_at ASC
LIMIT $batch_size;

UPDATE object_tasks SET status = 'running', started_at = current_timestamp
WHERE id IN (<拿到的 id 列表>);

COMMIT;
```

`BEGIN IMMEDIATE` 立刻获取写锁，其他 worker 会等到本事务 commit。**SQLite 路径建议 worker `concurrency = 1`**——多 worker 在 SQLite 上互相 serialize 没有吞吐收益，反而增加锁竞争。本机部署单 worker 即可。

server 端 storage adapter 按 backend 选实现：

```python
class TaskQueue:
    async def claim_batch(self, job_id, batch_size, priority_order):
        if self.backend == "postgres":
            return await self._claim_postgres(...)   # SELECT ... FOR UPDATE SKIP LOCKED
        else:  # sqlite
            return await self._claim_sqlite(...)     # BEGIN IMMEDIATE + SELECT + UPDATE
```

`priority` 由 `connector.task_priority(change)` 在入队时一次性算出来写进 task 行——大多数 connector 默认 0（FIFO）。

```python
async def worker_loop():
    while True:
        tasks = await db.claim_batch(limit=BATCH_SIZE)     # 自动按 backend 选 PG/SQLite 实现
                                                            # ORDER BY priority ASC, started_at ASC
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

#### Task 调度顺序：`priority` 列

`object_tasks.priority` 列决定 worker 拉取顺序，**越小越先**。framework 在把 ObjectChange 入队时调一次 `connector.task_priority(change)` 算出 priority 写进 task 行，之后调度全靠 SQL `ORDER BY priority`。

**默认行为**：connector 不重写 `task_priority` → 返回 0 → 整个 job 按入队顺序 (FIFO)。Postgres / Slack / GitHub 一般不需要重写——connector 产出 ObjectChange 的顺序（按表 / 按 channel 时间 / 按 issue 时间）本身就是有意义的。

**file connector 重写它**，给关键文件更高优先级（值更小）：

| 文件 / 路径特征 | 相对 priority |
|---|---|
| `README.md` / `CLAUDE.md` / `SKILL.md` / `INDEX.md` | 最先（-350） |
| `pyproject.toml` / `package.json` / `Cargo.toml` / `go.mod` / ... | 很先（-260） |
| `src/` / `lib/` / `app/` / `services/` 下 | 较先（-220） |
| `docs/` / `guides/` 下 | 较先（-190） |
| `tests/` / `fixtures/` 下 | 较后（+80） |
| `dist/` / `build/` / `vendor/` / `generated/` 下 | 最后（+260） |

**为什么需要这套规则**：

1. **首屏可见**：用户 `mfs add .` 一个大 repo 后，sync 跑到 30% 时，README + 配置 + 核心源码已经索引完——agent 立刻能 `grep "ROUTING"` / `search "auth flow"` 拿到结果。如果先跑 tests / build artifacts，前半截 search 体感很差。
2. **graceful degradation**：跑到一半挂了，至少关键文件已经索引；剩下没跑完的多半是 tests / generated，影响最小。
3. **重跑顺序一致**：重试失败的 task 时，重要的还是先被重试——最快恢复"能搜到核心内容"的状态。

**幂等性不依赖顺序**：chunk_id = `sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)`（详见 [06 §1](06-search-and-retrieval.md#1-milvus-collection-schema)），跟处理顺序无关。priority 只影响**用户体感和调度便利性**，不影响正确性。

### 5.7 一致性四条规则

整套 sync 的正确性靠这四条保证：

**1. Chunk-level 幂等**

```
chunk_id = sha1(namespace_id + connector_uri + object_uri + locator + chunk_kind)
                                                                     ← 确定性 hash
写 chunk = DELETE WHERE chunk_id = X + INSERT new row
```

任何 worker / 任何重试 / 任何并发，对同 chunk_id 的写都等效。`namespace_id` 必须进 hash——否则两个 namespace 注册同一外部数据源（如双方都叫 `postgres://prod`）会让 chunk_id 撞车互相覆盖。

**2. Per-object 原子**

```
object_task.status = 'succeeded'
   ↔ 该 object 的所有 chunks 都写入 Milvus 且 cache 已更新
```

中途任一步失败 → object_task 保持 'running' 或退 'failed'，下次 sync 重试整个 object。

**3. State 末尾提交 + 可选 checkpoint**

```
connector.sync() 过程中的 self.state.set(...) 写入暂存（connector_jobs.state_snapshot）
默认行为：只有当 sync_job 所有 object_task 成功时才 commit 到 connector_state 表
可选：connector 调 self.state.checkpoint() → framework 立刻把当前 snapshot commit
```

中途崩溃 → 未 checkpoint 的 state 不 commit → 下次 sync 从上次成功的 state 重启。`connector.sync()` 必须 idempotent（同 state 输入 → 同 ObjectChange 输出，加上新数据）。

checkpoint API 详见 [04 §5.6](04-connector-and-ingest.md#56-mid-job-checkpoint-api)；不是所有 state 形态都能调，规范见那一节。

**4. Sync 末尾 reconcile pass**

connector.sync() 只负责报告 upstream 变化。但下游产物（cache / chunk / embedding）可能因 framework 配置变化（换 embedding 模型 / chunker 升级 / converter 升级）而 stale，**connector 不感知**。每次 sync_job 在 connector.sync() yield 完后，framework 跑一遍 **reconcile pass**：

```
sweep 当前 connector 下所有未在本次 sync 中被 yield 的 object：
  跑 fingerprint chain 比对（详见 04 §5.2）：
    - cache 层 fp 变了？→ 入队 cache rebuild
    - chunk 层 fp 变了？→ 入队 chunk rebuild
    - embedding 层 fp 变了？→ 入队 embed rebuild only
```

这条让 "换 embedding 模型 → 跑 `mfs add ./repo` → 自动重 embed" 能 work——用户**不需要 `--force-index`** 来触发模型变化的失效。完整逻辑详见 [04 §5.2 Reconcile pass](04-connector-and-ingest.md#52-reconcile-pass-framework-内部).

framework 不暴露 `commit()` 给 connector（只暴露 `checkpoint()`），commit 时机由 framework 控制。

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

### 5.8.1 完成 job 的归档（DB 队列的"出队"）

DB 队列没有传统 FIFO 的"出队"动作。任务靠 `status` 字段流转：

```
pending → running → succeeded / failed / cancelled
```

worker 只 `SELECT ... WHERE status='pending' FOR UPDATE SKIP LOCKED`，看不到 succeeded 行。终态行（succeeded / failed / cancelled）留在表里几天，供 `mfs job list` / `mfs job inspect` 查历史。

按 status 分级 retention（debug 用的失败任务保留更久）：

```toml
# server.toml
[jobs.retention]
succeeded_days = 7              # 成功的留 7 天
failed_days = 30                # 失败的留 30 天（方便 debug）
cancelled_days = 7              # 用户主动取消的留 7 天
running_timeout_hours = 24      # running 超过 24h 视为僵尸，标 failed
```

server 启动一个 housekeeping coroutine，每天跑两步：

**步骤 1：把僵尸 running 标 failed**（worker 崩溃没及时清理 heartbeat 的）：

```sql
UPDATE connector_jobs
SET status = 'failed',
    error  = 'stale (heartbeat timeout)',
    finished_at = now()
WHERE status = 'running'
  AND heartbeat < now() - interval '24 hours';
```

**步骤 2：按 status 分级删除（按 `finished_at` 过期）**：

```sql
DELETE FROM connector_jobs
WHERE (status = 'succeeded' AND finished_at < now() - interval '7 days')
   OR (status = 'failed'    AND finished_at < now() - interval '30 days')
   OR (status = 'cancelled' AND finished_at < now() - interval '7 days');

-- object_tasks 走外键 ON DELETE CASCADE 自动级联，或显式：
DELETE FROM object_tasks
WHERE connector_job_id NOT IN (SELECT id FROM connector_jobs);
```

为什么按 `finished_at` 而不是 `created_at`：长 sync 的 created_at 可能是 8 小时前，但完成才是历史归档的基准。

比 Redis 自动过期还简单（就是个 SQL DELETE）。不构成换队列后端的理由。

### 5.9 重跑语义

| 命令 | 行为 |
|---|---|
| `mfs add <uri>` 已注册 | 新建 sync_job → connector.sync() 从 connector_state 接续 → 增量出 ObjectChange |
| `mfs add <uri> --force-index` | 同上，但所有 object 视为 'modified'，跳过 fingerprint 比对，强制重建 chunks |
| `mfs add ./path --force-upload` | 仅 upload flow：忽略 client manifest cache 全量重传 + 强制重 index |
| `mfs add <uri>` 在前一个 sync 失败后 | 前次 state 没 commit；从上上次的 state 重跑——失败的 object 自然重新出现在流里 |
| 第二次 `mfs add <uri>` 在前一个 sync 还 running | 返回 `sync_already_running, see job <id>`（被 UNIQUE 约束拒绝） |

### 5.10 Worker 并发与 batch 配置

`/etc/mfs/server.toml`（或 `~/.mfs/server.toml`）：

```toml
[worker]
concurrency = 4                  # 同时跑几个 worker（asyncio task）
                                 # ⚠️ metadata backend = "sqlite" 时建议设为 1：
                                 # SQLite 不支持 SKIP LOCKED，多 worker 互相 serialize
                                 # 没吞吐收益。详见 §5.6
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

所有"对一个 connector 的操作"（sync / force_sync / remove / update_config）都统一进 `connector_jobs` 表，用 `op_kind` 区分。**两条 partial UNIQUE 约束 + 三条规则**覆盖所有并发场景。

#### 两条 partial UNIQUE 约束（§6.1 schema）

```sql
CREATE UNIQUE INDEX ux_connector_jobs_one_running ON connector_jobs (connector_id) WHERE status = 'running';
CREATE UNIQUE INDEX ux_connector_jobs_one_queued  ON connector_jobs (connector_id) WHERE status = 'queued';
```

含义：同 connector 任意时刻**至多一个 running** + **至多一个 queued**。`(running, queued)` 同时存在是合法的——这正是 sync→remove preempt 流程需要的。"任意 op_kind 都受这两条约束限制" 是关键：拒绝逻辑（"sync 中又来 sync 怎么办"）由**应用层**根据 `op_kind` 判断，而不是由 SQL 约束本身判断。

#### 三条核心规则（应用层）

**① 同种重复 → 拒绝（destructive 类除外，幂等）**

- `sync + sync` / `force_sync + force_sync` → 应用层拒绝（即使约束允许，业务层判断）
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
    │   （ux_connector_jobs_one_queued 此时允许：sync 在 running，remove 占 queued）
    ▼ ④ 把 running 的 sync 标 cancelling
    ▼ ⑤ worker 在每个 object_task 边界检查 cancel signal
    │   当前 task 完成后退出（per-object 原子）
    ▼ ⑥ sync_job status → 'cancelled'
    ▼ ⑦ remove job 从 'queued' → 'running'
    ▼ ⑧ 跑 remove 流程：
    │     - Milvus: DELETE WHERE namespace_id = X AND connector_uri = <root>
    │       （受 partition_key=connector_uri 物理路由加速；详见 §6.3）
    │     - object store: 删 cache 文件 + staging files/ 树
    │     - metadata DB: 删 caches / connector_state / objects / upload_manifests / object_tasks / connector_jobs
    ▼ ⑨ connectors row DELETE
    ▼ ⑩ remove job status → 'succeeded'
```

清理顺序确保**幂等可重入**：如果 step ⑧ 中途崩溃，下次重启 worker 重跑 remove job，可以从任何一步开始（DELETE 已无匹配行 / 删空目录都是 no-op）。

> 关于"是不是 drop_partition 更快"：Milvus 的 `partition_key` 是按字段哈希自动分桶，**不是 named partition**，没有对应的 `drop_partition(<value>)` API。remove 走 `DELETE WHERE connector_uri = ...`；partition_key 的好处是过滤只扫一个物理桶，但删除时仍要执行 expression-based delete（详见 §6.3 + [06-search-and-retrieval.md §1](06-search-and-retrieval.md#1-milvus-collection-schema)）。

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
  namespace_id    VARCHAR DEFAULT 'default',   -- 物理分区主键；v0.4 恒为 'default'
  root_uri        VARCHAR,
  type            VARCHAR,
  label           VARCHAR,
  status          VARCHAR DEFAULT 'active',    -- 'active' | 'removing'
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
  namespace_id    VARCHAR DEFAULT 'default',  -- 进主键，避免跨 namespace 同名 object_uri 撞车
  object_uri      VARCHAR,
  cache_kind      VARCHAR,
  storage_path    VARCHAR,                    -- ~/.mfs/cache/caches/<namespace_id>/<sha1>/<kind> 或 s3 key
  fingerprint     VARCHAR,
  size_bytes      INTEGER,
  built_at        TIMESTAMP,
  last_accessed   TIMESTAMP,
  PRIMARY KEY (namespace_id, object_uri, cache_kind)
);

-- ===== Job 队列：统一所有 connector op =====
connector_jobs (
  id                    VARCHAR PRIMARY KEY,
  namespace_id          VARCHAR DEFAULT 'default',
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
  state_snapshot        TEXT            -- pending：暂存的 connector state，sync 末尾才 commit
);

-- 关键约束：同 connector 同时至多一个 running、至多一个 queued。
-- 内联 partial UNIQUE 语法在 SQLite / Postgres 都不允许，必须拆成两条 partial UNIQUE INDEX。
-- 拆成两条而非合一条 (status IN ('queued','running')) 的理由：
-- §5.11 的 sync→remove preempt 流程要求 "sync running 时 remove queued" 共存——
-- 合一条约束会阻止 remove 入队。
CREATE UNIQUE INDEX ux_connector_jobs_one_running ON connector_jobs (connector_id)
  WHERE status = 'running';
CREATE UNIQUE INDEX ux_connector_jobs_one_queued  ON connector_jobs (connector_id)
  WHERE status = 'queued';

object_tasks (
  id                    VARCHAR PRIMARY KEY,
  connector_job_id      VARCHAR REFERENCES connector_jobs(id),   -- 仅 sync/force_sync 类 job 有 task
  object_uri            VARCHAR,
  change_kind           VARCHAR,        -- 'added' | 'modified' | 'deleted'
  status                VARCHAR,        -- 'pending' | 'running' | 'succeeded' | 'failed' | 'cancelled'
  priority              INTEGER DEFAULT 0,   -- 越小越先；connector.task_priority() 注入，默认 0 (FIFO)
  attempts              INTEGER DEFAULT 0,
  last_error            TEXT,
  started_at            TIMESTAMP,
  finished_at           TIMESTAMP
);
CREATE INDEX ix_object_tasks_sched ON object_tasks (connector_job_id, status, priority);
CREATE INDEX ix_object_tasks_running ON object_tasks (status, started_at) WHERE status = 'running';

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

所有顶层表都以 `namespace_id` 作为物理分区主键（v0.4 恒为 `'default'`）。Workspace / User / Project 等组织概念通过 mapping 表实现，**底层 schema 不动**——详见 [§9](#9-多租户与-namespace-设计)。

### 6.2 Object store (cache + upload staging)

object store 同时存两类东西：

- **cache 文件**：每个有 cache 的 object 各自的产物（每个 object 通常只对应一个 cache_kind）
- **upload staging**：client 上传的 zip bundle + 解压后的真实文件树（仅 §3.5 upload flow 用）

#### cache_kind ↔ object 类型对应

**一个 object 通常只有 1 个适用的 cache_kind**，不是每个 object 都有所有 cache。具体看 object 类型：

| object 类型 | cache_kind | 例 |
|---|---|---|
| PDF / DOCX 等可转 markdown 的文档 | `converted_md` | `manual.pdf` 的 markdown 缓存 |
| DB rows / API records 集合 | `page_cache.jsonl` | `postgres://.../rows.jsonl` 的物化页 |
| DB rows 的 head 预拉取 | `head_cache.jsonl` | 前 100 行预 cache，加速 head 命中 |
| 图片 | `vlm_text` | `diagram.png` 的 VLM description |
| DB schema / 元数据 dump | `schema_dump.json` | postgres `schema.json` 的物化 |
| markdown / code / 纯文本真实文件 | **无 cache** | 直接 read，没必要 cache |

实际目录布局（每个 sha1 子目录通常只 1 个文件，**最外层按 namespace_id 切**）：

```
~/.mfs/cache/
  caches/
    <namespace_id>/                              ← v0.4 恒为 "default"；多租户启用后是真值
      <sha1(./repo/manual.pdf)>/
        converted_md                             ← PDF 转 markdown
      <sha1(postgres://prod/.../rows.jsonl)>/
        page_cache.jsonl                         ← DB 物化页
      <sha1(./repo/diagram.png)>/
        vlm_text                                 ← 图片 VLM 描述
      <sha1(postgres://prod/.../schema.json)>/
        schema_dump.json                         ← DB schema 物化
  uploads/
    <namespace_id>/
      <connector_id>/
        <temp_file_id>.zip                       ← bundle，commit 后删
  files/
    <namespace_id>/
      <connector_id>/                            ← 解压后的真实文件树，作为 cache 保留
        src/cli.py
        README.md
        ...
```

**为什么按 namespace_id 切**：cache 内容来自 object_uri 的 sha1，但两个 namespace 注册同名 connector 时 object_uri 相同 → cache key 相同 → 一个 namespace 的 cache 内容可能被另一个 namespace 直接命中（ACL 跳过）。v0.4 单 namespace 只有 `default/` 一层，没成本；等 v0.5+ 多租户上线时**已经物理隔离**，不需要重组目录。

`caches` 表存 `(namespace_id, object_uri, cache_kind) → storage_path` 映射；`upload_staging` 表存 temp_file_id → 路径。

#### 默认后端：本地文件系统

**默认 `backend = local`**，最快、零运维。但**CS 部署需要按规模选择**：

#### 后端选择 trade-off

| 后端 | first byte 延迟 | 多 worker 共享 | 持久化 | 适合 |
|---|---|---|---|---|
| `local` （fs） | ~1ms | ❌（仅同进程） | 跟随 disk | 个人本机、单容器 server |
| `minio` | ~5-10ms | ✅ | 跟随 volume | Docker Compose 多 worker |
| `s3` / `r2` / `gcs` | ~50-100ms | ✅ | HA、跨实例 | K8s 生产部署 |

S3 延迟看起来比本地 fs 慢，但仍**远比 connector 重新拉取外部 API（100ms+）快**。对 throughput 影响小（streaming）。

**部署建议**：

| 部署形态 | object_store backend | 备注 |
|---|---|---|
| 个人本机 `mfs serve` | `local` | 默认；零配置 |
| Docker 单容器 server（demo / 小规模） | `local` | 必须 `-v /data/cache` bind volume 持久化 |
| Docker Compose 多 worker | `minio` | 容器间共享 |
| K8s / 商业化生产 | **S3 / R2 / GCS** | 首选；HA + 跨 replica |

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
per_namespace_quota_gb = 0           # 0 = 不限；多租户部署可按 namespace 设
```

### 6.3 Milvus

一张 collection `mfs_chunks`，`partition_key = connector_uri`。详见 [06-search-and-retrieval.md §1](06-search-and-retrieval.md#1-milvus-collection-schema)。

#### Partition by connector_uri：用 `partition_key`，不是 named partition

Milvus 里有**两套不同的 partition 机制**，文档里经常被混用：

| 机制 | API | 数量上限 | 适合 |
|---|---|---|---|
| **Named partition** | `create_partition("p1") / drop_partition("p1")` | 单 collection ~4096 个（可配置） | 你**显式管理**的少量分区（如按 month 分区） |
| **`partition_key` 字段** | schema 里 `partition_key=True`；写入/查询按字段哈希自动路由 | 由配置 `num_partitions` 决定，默认 64 | 多租户类**大量自动分桶**场景 |

MFS 用的是 **`partition_key`**：

- ✅ 不用代码显式 create / drop 物理分区
- ✅ 查询带 `connector_uri == X` filter 时只扫该 connector 命中的桶
- ❌ **没有对应的 "drop one connector 的桶" API**——remove 一个 connector 必须走 `DELETE WHERE connector_uri = X`

所以 §5.11 流程里 remove 用 `DELETE WHERE namespace_id = X AND connector_uri = <root>`。partition_key 仍然有用：DELETE 也会按 partition_key 路由，只扫该 connector 命中的桶（不是全表 scan），但仍是 expression-based delete，比 named partition 的 `drop_partition`（直接删一个物理文件夹）慢一个数量级。

**v0.4 选择 partition_key 而不是 named partition 的理由**：

| 维度 | partition_key | named partition |
|---|---|---|
| connector 数量上限 | 几万级（受 `num_partitions` 影响弱，主要受 collection 整体规模限制） | 单 collection ~4096，**真的有硬上限** |
| 注册/注销 connector 代码 | 不动 schema，写入即生效 | 每个 connector 显式 `create_partition`，状态需手动 GC |
| remove 一个 connector 性能 | DELETE WHERE（中等） | `drop_partition`（最快） |
| 多 namespace × 多 connector | scale OK | partition 数量爆炸 |

MFS 预期 connector 数量可能成百上千（个人 + 团队 + 多 namespace），partition_key 是唯一可 scale 的方案。**性能换扩展性的取舍**。

backends（推荐优先级 ↓）：

| 后端 | URI | 适合 | v0.4 推荐度 |
|---|---|---|---|
| **Milvus Lite 3.0+** | `~/.mfs/milvus.db` | 个人本机；零运维 | ⭐ 个人首选 |
| **Zilliz Cloud** | `https://*.zillizcloud.com` + token | CS 部署、商业化场景 | ⭐ CS 首选 |
| 自部署 Milvus 3.0+ | `http://host:19530` | 自有数据中心 / 不能上云的场景 | 可选 |

v0.4 主推 **Lite（个人）+ Zilliz Cloud（CS 部署）** 两条路。自部署 Milvus 因为它本身就是 Docker Compose 多容器（含 etcd / pulsar / object store），运维负担重，不作为默认推荐。

**Backend 能力矩阵**：

| 能力 | Lite 3.0 | Zilliz Cloud | 自部署 Milvus 3.0+ |
|---|---|---|---|
| `partition_key` 字段 | ⚠️ 待 verify（Lite 3.0 是 Python 全重写，能力跟历史版本不一定一致） | ✅ | ✅ |
| DELETE by expression | ✅ | ✅ | ✅ |
| `sparse_vec` + 内建 BM25 | ⚠️ 待 verify | ✅ | ✅ |
| `scalar index` 字段 | ✅ | ✅ | ✅ |
| JSON metadata filter | ✅ | ✅ | ✅ |
| 多 writer 并发 | ❌ | ✅ | ✅ |
| 横向扩展 | ❌ | ✅（托管） | ✅（需配 Pulsar/Kafka） |
| 备份 / 快照 | 文件 cp | 内建 | manual / operator |

> **⚠️ Caveat**：Milvus Lite 3.0 是大版本重构（Python 全重写），上面 Lite 列基于 2.x 资料的推断，**实现 v0.4 时需要按 Lite 3.0 / Milvus 3.0 最新代码实测重校准**。如果 Lite 3.0 不支持 sparse_vec 或 partition_key，需要降级方案（如个人本机退回 single-collection + scalar filter）。

**实际影响**：

- **Lite** 在个人本机场景够用，但能力略弱（多 worker 受限——本机本来就单 worker，不是问题）。
- **Zilliz Cloud** 是 CS 部署默认；商业化路径 = Zilliz Cloud。
- 文档默认假设 **Milvus 3.0+**（sparse_vec / BM25 / partition_key 必需）。

server 端配置：

```toml
[milvus]
uri = "~/.mfs/milvus.db"               # 默认 Lite（个人本机）
# uri = "https://xxx.zillizcloud.com"  # CS / 商业化
# token = "..."
# uri = "http://localhost:19530"       # 自部署（不推荐，运维负担重）
collection_strategy = "single"          # single | per_connector | per_namespace
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
# CLI（Rust 单 binary）
brew install mfs                          # macOS / Linux
# 或 scoop install mfs                    # Windows
# 或 cargo install mfs                    # 通过 cargo
# 或 curl -fsSL https://mfs.dev/install.sh | sh   # 直接下载 binary

# Server（Python）
uv tool install mfs-server                # 跑本机 server

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

### 8.3 Docker Compose + Zilliz Cloud（推荐 CS 部署）

mfs 本身的 API + worker 用 Docker Compose 起；Milvus 用 **Zilliz Cloud**（托管），不自部署：

```yaml
services:
  postgres:    image: postgres:16              # MFS metadata
  minio:       image: minio/minio              # object_store
  mfs-api:     image: ghcr.io/zilliztech/mfs-api:0.4.0
  mfs-worker:  image: ghcr.io/zilliztech/mfs-worker:0.4.0
```

server.toml 指向 Zilliz Cloud：

```toml
[milvus]
uri = "https://xxx.zillizcloud.com"
token = "<env:ZILLIZ_TOKEN>"
```

```bash
docker compose up -d
mfs profile add prod --url https://mfs.example.com
mfs add postgres://prod --config ...
```

> Milvus 本身是多容器拓扑（etcd / pulsar / object store 自带），自部署运维成本高。v0.4 默认假设用 Zilliz Cloud。如果你**必须**自部署 Milvus，去 milvus.io 找官方 docker compose；MFS 不重新发布 Milvus 容器组合。

### 8.4 Kubernetes

```bash
helm install mfs ./charts/mfs \
  --set api.replicas=2 \
  --set worker.replicas=4 \
  --set objectStore.type=s3 \
  --set search.type=zilliz \
  --set metadata.type=postgres
```

## 9. 多租户与 namespace 设计

### 9.1 核心决策：物理分区 ≠ 组织结构

借鉴 AWS（Account vs Organization）、GCP（Project vs Folder）、K8s（Namespace vs Project）的分层思路——**数据隔离边界和组织结构语义解耦**：

- **底层**：`namespace_id` 是唯一的物理分区主键。所有 DB 表 / Milvus partition / object store prefix 都按它切。namespace 是稳定的、不可重组的。
- **上层**：Workspace / User / Project / Team 是产品概念，通过 **mapping 表**指向 namespace。组织关系演化（个人 → 团队、跨 workspace 共享）只改 mapping，**底层数据零迁移**。

> 用户视角看不到 `namespace` 这个词——它纯粹是 server 内部的物理分区主键。对外暴露的概念（v0.5+ 引入）是 Workspace / User。

### 9.2 v0.4：单 namespace，零配置

v0.4 server 启动时自动创建一个 `default` namespace。所有数据写入都带 `namespace_id = "default"`，所有查询都 filter `namespace_id = "default"`。用户感知不到任何租户概念。

CLI 端：

- `client.toml` profile 没有租户字段
- HTTP 请求不带租户 header
- token 隐式绑定 `default` namespace（本机模式 token 通常省略）

### 9.3 v0.5+：加 Workspace + User mapping

v0.5 引入认证 + 多租户时，**底层 namespace schema 不动**，只新增 mapping 表：

```sql
-- v0.5+ 新增表
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
  role          VARCHAR,       -- 'owner' | 'member' | 'viewer'
  PRIMARY KEY (workspace_id, user_id)
);

-- mapping：namespace 归属
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

-- v0.4 已存在但 v0.5 才用 user_id
namespaces (
  id            VARCHAR PRIMARY KEY,    -- UUID
  slug          VARCHAR UNIQUE,         -- 内部识别用
  created_at    TIMESTAMP
);
```

请求作用域解析：

```
token → user_id → 该 user 能访问的 namespace_id 集合
  ↓
所有 query: WHERE namespace_id IN (resolved_set)
```

### 9.4 这套设计能拼出来的产品形态

| 产品概念 | mapping 表达 |
|---|---|
| 个人空间 | `user_namespaces (1:1)` |
| 团队 workspace | `workspace_namespaces (1:1)` + `workspace_members` |
| 一个 workspace 下多 project | `workspace_namespaces (1:N)`，每个 project 一个 namespace |
| 跨 workspace 共享数据 | 同一 namespace_id 出现在多个 `workspace_namespaces` 行 |
| 个人迁到团队 | 把 namespace 从 `user_namespaces` 移到 `workspace_namespaces`——**数据零迁移** |

### 9.5 Milvus 隔离策略

namespace 默认在 Milvus 层用 scalar filter (`namespace_id IN (...)`)。**当某个 namespace 数据量大或需要更强隔离**时，可按 `collection_strategy` 升级：

| 策略 | Milvus 实现 | 隔离强度 | 资源开销 | 适用 |
|---|---|---|---|---|
| `single` (默认) | 一张 collection，按 `namespace_id` scalar filter | 弱 | 最小 | v0.4 / 小规模 |
| `per_connector` | 一 connector 一张 collection | 中（连同 namespace filter） | 中 | 数据量大时按 connector 切 |
| `per_namespace` | 一 namespace 一张 collection | 强 | 大 | 多租户强隔离 / 合规要求 |
| `database_per_namespace` | 一 namespace 一 Milvus database | 最强 | 最大 | enterprise 合规 |

切换 strategy 通过迁移工具完成（roadmap）。

### 9.6 数据量分层策略

| 量级 | 策略 |
|---|---|
| < 几百万 chunks | 默认 `single` collection 够 |
| 几千万 chunks | 调 Milvus shard / HNSW 参数；考虑分 partition |
| 亿级 chunks | 切 `per_connector` 或 `per_namespace` collection；冷热分层 |

## 10. 发布与分发

### 10.1 交付物清单

| 交付物 | 实现 | 注册中心 | 用户安装 |
|---|---|---|---|
| **`mfs` CLI binary** | Rust | GitHub Releases + crates.io + Homebrew tap + Scoop bucket + PyPI wheel | `brew` / `scoop` / `cargo install` / `uv tool install` / `curl install.sh` |
| **`mfs-server`** | Python + Rust PyO3 | PyPI（wheel 多平台，含 Rust 编译产物） | `uv tool install mfs-server` |
| **Docker `mfs-api` / `mfs-worker` / `mfs-server-aio`** | — | ghcr.io / Docker Hub | `docker pull` / Compose / Helm |
| **`mfs-sdk` (Python)** | Python | PyPI | `pip install mfs-sdk`（程序化集成；独立于 CLI） |
| **`@mfs/sdk` (TS)** | TypeScript | npm | `npm install @mfs/sdk` |
| **Go SDK** | Go | Go module proxy | `go get github.com/zilliztech/mfs-sdk-go` |
| **Java SDK** | Java | Maven Central | `io.zilliz.mfs:mfs-sdk` |

多语言 SDK 都从 `protocol/openapi.yaml` 生成。

### 10.2 CLI 跨平台 build 与分发（cargo-dist）

CLI 是 Rust 单 binary，跟平台相关。用 **`cargo-dist`** 自动化整个发布——业界标准做法（uv / ruff / starship / zellij / atuin 都用）。

**Target matrix**（6 个 binary 覆盖 99% 用户）：

```
x86_64-unknown-linux-gnu        # Linux x86_64（大多 server / dev box）
aarch64-unknown-linux-gnu       # Linux ARM64（AWS Graviton / Ampere）
x86_64-unknown-linux-musl       # Alpine / 静态链接（可选）
x86_64-apple-darwin             # macOS Intel
aarch64-apple-darwin            # macOS Apple Silicon (M1+)
x86_64-pc-windows-msvc          # Windows x86_64
```

**发布触发**：一个 git tag 触发 GitHub Actions matrix：

```
git tag v0.4.0 && git push --tags
    ↓
GitHub Actions（cargo-dist 生成的 workflow）自动：
  ① matrix build 6 个平台 binary
  ② 打包 tar.gz / zip
  ③ 上传到 GitHub Releases
  ④ 更新 install.sh / install.ps1
  ⑤ 自动 PR 到 Homebrew tap repo
  ⑥ 可选：scoop manifest / AUR PKGBUILD
```

整个 build matrix 跑完通常 10-15 分钟。

### 10.3 Server 端 Rust 模块走 maturin → PyPI wheel

`server-rs/` 里的 Rust crate（`mfs-scan` / `mfs-jsonl` / `mfs-grep`）**不单独发布到 crates.io**——它们是 `mfs-server` Python 包的 native 扩展，跟 mfs-server 一起 build。

**工作流**：

```bash
# 开发时（贡献者）
cd server-rs/
maturin develop --release            # 编译 Rust → 安装到当前 Python venv
# Python 代码 from mfs_server_rs import scan_dir, parse_jsonl_stream

# 发布时（CI）
maturin build --release --target <platform>   # matrix build wheel
twine upload dist/*.whl                       # 上传 PyPI
```

每个 `mfs-server` 版本在 PyPI 上对应**多个 wheel**（每平台一个），里面已经包含编译好的 Rust 产物（`.so` / `.pyd`）。

**用户感知不到 Rust**：

```bash
uv tool install mfs-server
# pip / uv 自动选当前平台的 wheel
# wheel 里是 .py + .so，用户不需要装 Rust 工具链
```

业界先例：**pydantic-core**（pydantic 的 Rust 内核）/ **ruff_python_parser** / **polars** / **tokenizers** — 全是这个模式。

### 10.4 Rust 的"PyPI" = crates.io

| 生态 | 包注册中心 | 默认安装命令 |
|---|---|---|
| Python | **PyPI** | `pip install` / `uv tool install` |
| JS / TS | **npm** | `npm install` |
| **Rust** | **crates.io** | `cargo install` |
| Go | proxy.golang.org | `go install` |

**Rust 关键不同**：`cargo install <pkg>` 是**下载源码 + 本地编译**（不是下二进制），所以纯走 crates.io 用户要等几分钟编译。

业界主流 Rust CLI 因此**多渠道并行发布**：

| 渠道 | 体验 | 主要受众 |
|---|---|---|
| **GitHub Releases binary**（cargo-dist 自动） | 一行 install.sh / 解压即用 | 大多数用户 |
| **Homebrew tap**（cargo-dist 自动 PR） | `brew install` | macOS / Linux |
| **Scoop bucket** | `scoop install` | Windows |
| **PyPI wheel**（via maturin） | `pip install` / `uv tool install` | Python / agent 用户（最大群） |
| **crates.io**（`cargo publish`） | `cargo install`（编译慢） | Rust 开发者 |
| AUR / Snap / Chocolatey | 包管理器命令 | 社区维护，可选 |

crates.io 是"Rust 用户的便利入口"，**不是主分发路径**。

### 10.5 用户安装速查（按系统）

**macOS / Linux**

```bash
curl -fsSL https://mfs.dev/install.sh | sh        # 一行
brew install zilliztech/tap/mfs                   # Homebrew
```

**Windows**

```powershell
irm https://mfs.dev/install.ps1 | iex             # 一行
scoop bucket add zilliz https://github.com/zilliztech/scoop-bucket
scoop install mfs                                 # Scoop
```

**通过包管理器**

```bash
cargo install mfs                                 # Rust 用户
pip install mfs                                   # Python 用户（实际装 wheel，里面是 Rust binary）
uv tool install mfs                               # uv 用户
```

**Server 端**（Python；自带 Rust PyO3 wheel）

```bash
uv tool install mfs-server                        # 个人本机
# 或运维端走 Docker / K8s（见 §8）
```

### 10.6 Server optional extras（按 connector 安装）

```text
mfs-server[postgres]    mfs-server[mysql]       mfs-server[mongo]
mfs-server[bigquery]    mfs-server[snowflake]
mfs-server[slack]       mfs-server[discord]     mfs-server[gmail]
mfs-server[gdrive]      mfs-server[feishu]      mfs-server[s3]
mfs-server[github]      mfs-server[linear]      mfs-server[jira]
mfs-server[notion]
mfs-server[salesforce]  mfs-server[hubspot]     mfs-server[zendesk]
mfs-server[web]
mfs-server[embedding-onnx]   mfs-server[embedding-google]
mfs-server[llm-anthropic]    mfs-server[llm-google]
mfs-server[zilliz]
mfs-server[all]
```

### 10.7 发布动作总结

每次发版的实际工作流：

```
本地：
  bump 版本号（Cargo.toml + pyproject.toml + package.json 等）
  git commit -m "release v0.4.0"
  git tag v0.4.0 && git push --tags

GitHub Actions 自动跑：
  ① cargo-dist workflow: build CLI 多平台 binary → GH Releases + Homebrew PR
  ② maturin workflow:    build mfs-server wheel 多平台 → PyPI
  ③ docker workflow:     build / push mfs-api / mfs-worker / mfs-server-aio → ghcr.io
  ④ npm publish workflow: TS SDK → npm
  ⑤ Go SDK 通过 git tag 自动可用（无需 publish）
  ⑥ Maven publish workflow: Java SDK → Maven Central

一次 git tag 触发全套。

## 11. 工程目录结构

多语言混合：CLI 用 Rust（单 binary 体验），server 主体 Python，server 内部 hot path 用 Rust 通过 PyO3 绑定，多语言 SDK 走 OpenAPI 生成。

```
.
├── cli/                                       # ⭐ Rust CLI（单 binary 分发）
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs                            # CLI entry
│   │   ├── commands/                          # add / search / grep / ls / cat / ... 每条命令一个模块
│   │   ├── transport/                         # HTTP client（reqwest）+ machine-id 探测
│   │   ├── client_config/                     # client.toml 解析
│   │   ├── output/                            # 人类可读 + JSON envelope
│   │   └── models/                            # 从 protocol/openapi.yaml 生成
│   └── tests/
│
├── protocol/                                  # 跨语言契约
│   ├── openapi.yaml                           # client ↔ server HTTP API
│   ├── schemas/                               # JSON schema 共享
│   └── errors.md                              # 错误码表
│
├── server/
│   └── python/                                # PyPI: mfs-server
│       ├── pyproject.toml
│       └── src/mfs_server/
│           ├── api/
│           │   ├── app.py
│           │   ├── routes/                    # add / connectors / objects / search / jobs / files (upload)
│           │   └── middleware/
│           ├── server/                        # mfs-server CLI entrypoint (run / api / worker)
│           ├── worker/
│           ├── engine/
│           ├── connectors/                    # 每类 connector 自包含
│           │   ├── base.py
│           │   ├── registry.py
│           │   ├── file/
│           │   ├── web/
│           │   ├── postgres/
│           │   ├── slack/
│           │   ├── github/
│           │   └── ...
│           ├── processors/                    # Object Processors：按 object_kind 加工
│           │   ├── document/  code/  table_rows/  message_stream/
│           │   ├── record_collection/  image/  binary/
│           ├── common/                        # Common Services：通用工具集
│           │   ├── embedding/  summary/  vlm/
│           │   ├── retrieval/  export/
│           └── storage/                       # metadata / object_store / queue / search (Milvus)
│
│   # 注：engine/ 内含本机 daemon 和远端 server 两种部署形态（局部分支判断），
│   # 不再有独立的 runtime/ 目录。
│
├── server-rs/                                 # ⭐ Rust 加速模块（PyO3 绑定）
│   ├── Cargo.toml                             # workspace 声明子 crate
│   ├── pyproject.toml                         # maturin 构建配置（指向 PyPI mfs-server 包）
│   ├── mfs-scan/                              # 大目录扫描 + manifest hash
│   ├── mfs-jsonl/                             # JSONL / CSV / Parquet 流处理（polars 或自实现）
│   └── mfs-grep/                              # 高并发线性 grep
│   # 开发: maturin develop --release  → 安装到当前 venv，Python from mfs_server_rs import ...
│   # 发布: maturin build --release --target <platform> → wheel 含 .so/.pyd，上传 PyPI（跟 mfs-server 一起）
│   # 不单独发布到 crates.io；作为 mfs-server 包的 native 扩展（业界先例：pydantic-core/polars/ruff）
│
├── sdks/                                      # 多语言 SDK（OpenAPI 生成）
│   ├── python/                                # PyPI: mfs-sdk（程序化集成用，独立于 CLI）
│   ├── typescript/                            # npm
│   ├── go/
│   └── java/
│
├── deployments/
│   ├── docker/                                # mfs-api / mfs-worker / mfs-server-aio images
│   ├── compose/
│   └── helm/
│
├── tests/
│   ├── cli/                                   # Rust 测试
│   ├── server/                                # Python 测试
│   ├── connectors/
│   ├── e2e/                                   # CLI + server 联调
│   └── fixtures/
│
├── docs/
└── skills/                                    # agent skill（pure markdown）
```

**关键设计决策**：

- **CLI 跟 server 通过 HTTP 通信**，没有 import 关系——所以 CLI 用什么语言完全自由，跟 server 解耦。
- **Rust 加速模块通过 PyO3 暴露给 Python**——maturin 编译成 wheel，`pip install .` 后 Python 直接 `from mfs_server_rs import scan_dir`。跟 polars / pydantic-core / ruff 同套路。
- **多语言 SDK 都从 `protocol/openapi.yaml` 生成**——Python SDK 独立于 CLI（CLI 是 Rust，SDK 是 Python 给程序集成用）。

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
- Milvus collection schema（含 `namespace_id` 分区主键字段）。
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

**Rust CLI + Python server + Rust 加速模块（PyO3） + 多语言 SDK**。

| 模块 | 语言 | 理由 |
|---|---|---|
| **CLI** (`mfs` binary) | **Rust** | 单 binary 分发（brew / scoop / direct download），cold start 几十 ms，用户体验跟 docker / kubectl / git / ripgrep 一致 |
| API server / engine | Python + FastAPI + asyncio | IO-bound 为主；asyncio + FastAPI 性能足够；ecosystem 全 |
| Connectors | Python | 所有外部 SDK（postgres / slack / github / gdrive / openai / anthropic / google ...）first-class 是 Python |
| Embedding / LLM / VLM 调用 | Python | provider SDK 都是 Python first-class |
| Milvus 客户端 | pymilvus | 官方 |
| 大目录扫描 + hash | **Rust（PyO3 绑定）** | 千万文件场景，Python `os.walk` 太慢 |
| 大 JSONL / CSV / Parquet 流处理 | **Rust（polars / 自实现）** | 内存稳定，零拷贝 |
| AST tree-sitter 切分 | 已经 Rust | 沿用 tree-sitter 官方绑定 |
| 高并发 grep 线性扫 | Rust（可选优化） | 性能敏感场景 |
| Python SDK（程序化集成） | Python | 给写脚本的用户 |
| TypeScript / Go / Java SDK | 从 OpenAPI 生成 | 给跨语言用户 |

### 为什么 CLI 走 Rust

用户安装 CLI 的体验对比：

| 维度 | Python CLI (`uv tool install mfs-cli`) | Rust binary |
|---|---|---|
| 安装产物 | 一堆 `.py` 源码 + 几十个依赖包 | 一个 binary 文件 |
| 安装大小 | 100-300 MB（含 Python runtime） | ~30-50 MB |
| cold start | 200-500 ms | 几十 ms |
| 用户心智 | "我装了一个 Python 工具" | "我装了一个工具" |
| 分发方式 | PyPI / uv | brew / scoop / cargo / direct download |

CLI 跟 server 走 HTTP，**不需要 import server 代码**——选什么语言完全自由。Rust 单 binary 体验远好于 Python 工具链。这跟 docker / kubectl / git / ripgrep / fd 一致——agent 用户对单 binary 工具的预期。

### Python / Rust 互操作（PyO3 + maturin）

业界主流玩法，非常成熟。例子：

- **polars**（数据处理）：Rust + PyO3
- **pydantic v2**：Rust 核心 (`pydantic-core`) + PyO3
- **ruff** / **uv** / **rye**：Rust 实现 + Python 包装
- **tokenizers** (HuggingFace)：Rust + PyO3

工作流：

```bash
# server-rs/ 目录下
maturin develop          # 编译 Rust + 安装到当前 Python 环境
# 然后 Python 代码里
from mfs_server_rs import scan_dir, hash_files, parse_jsonl_stream
```

发布：`maturin build --release` 出 wheel，用户 `pip install mfs-server` 时**自动下载预编译 wheel**（manylinux / macOS / Windows 三套），不需要本地编 Rust。

### 性能模块的边界

Rust 模块封装在 server 内部，**connector 贡献者完全感知不到**——connector 全 Python，调 `from mfs_server_rs import scan_dir` 跟调普通 Python 函数一样。Rust 只在这几个 hot path 出现：

- `mfs_server_rs.scan_dir` — 大目录扫描 + manifest
- `mfs_server_rs.parse_jsonl_stream` — JSONL/CSV 流式处理
- `mfs_server_rs.linear_grep` — 高并发线性 grep

### 不推荐的方案

- **CLI 用 Python + pyapp/pex 编译 binary**：binary 仍 50-80 MB，cold start 100+ ms，劣于 Rust。
- **CLI 用 TypeScript + bun compile**：binary ~30-50 MB，cold start 比 Python 快但比 Rust 慢；TS 写 CLI 速度快但用户面体验不如 Rust。
- **Go 主体 server**：connector SDK 不齐（slack / linear 等 SaaS 有但 LLM/embedding 二级公民），开发速度低于 Python。
- **Rust 整体重写**：开发速度慢；社区贡献 connector 门槛太高。
