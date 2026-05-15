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
│  │ HTTP transport      │   profile.kind = local | remote            │
│  └─────────┬──────────┘                                             │
└────────────┼────────────────────────────────────────────────────────┘
             │ HTTP /v1 (control plane only; SSE for tail -f)
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
│  │   local: SQLite queue       remote: Redis/Postgres queue     │  │
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

MFS 有两个独立维度，常被混淆：

### 维度 A：profile.kind（client / server contract）

`kind` 字段决定 **client 和 server 是否共享文件系统命名空间**：

| kind | 含义 |
|---|---|
| `local` | CLI 进程和 daemon 共享同一文件系统命名空间；本地路径请求由 daemon 直接读文件 |
| `remote` | 跨命名空间（跨网络 / 跨容器 / 跨用户）；本地路径请求返回 `remote_server_cannot_read_local_path` |

**以下场景不算 local，应该按 `remote` 处理**：

- daemon 跑在 Docker 容器，CLI 在 host（除非显式 bind mount 整个相关目录树）。
- daemon 跑在 WSL，CLI 在 Windows host。
- daemon 跑在 SSH 远端 box，CLI 在笔记本（即使 port forward 把 127.0.0.1 指过去）。

### 维度 B：server 端存储后端

跟 profile.kind 完全无关。server 端配置文件 `~/.mfs/daemon.toml`（或部署时的 server config）决定用什么后端：

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

| profile.kind | metadata | object_store | milvus | 说明 |
|---|---|---|---|---|
| local | sqlite | local fs | Lite | 默认；个人本机最简 |
| local | postgres | local fs | Lite | 个人开发，跟生产 metadata 一致 |
| local | postgres | s3 | Zilliz | 个人 dogfood 生产配置 |
| remote | postgres | s3 | Zilliz | 团队部署 |
| remote | postgres | r2 | self-hosted | 自部署 |

「本地 daemon 又想用 PG / S3 / Zilliz」完全 OK，改 daemon 配置即可。

### profile 配置

`~/.mfs/config.toml`（client 侧）：

```toml
[client]
default_profile = "local"

[[profiles]]
name = "local"
api_base_url = "http://127.0.0.1:8765"
# kind 自动推断为 local；可省略

[[profiles]]
name = "prod"
api_base_url = "https://mfs.example.com"
# kind 自动推断为 remote
credential_ref = "env:MFS_API_TOKEN"
tenant_id = "tenant-a"      # 多租户场景

[[profiles]]
name = "docker-dev"
api_base_url = "http://127.0.0.1:8765"
kind = "remote"             # 显式覆盖：daemon 在 Docker，不共享文件系统
```

### kind 字段自动推断

`kind` 字段决定 path 处理逻辑，但 **99% 场景不用写**：

| URL host | 默认 kind |
|---|---|
| `127.0.0.1`、`localhost`、`::1` | `local` |
| 其他 | `remote` |

需要 `--kind` 显式覆盖的边界场景：

- daemon 跑在 Docker 容器，CLI 在 host
- daemon 跑在 SSH 远端 box，CLI 端用 port forward 把 `127.0.0.1:8765` 转过去
- daemon 跑在 WSL，CLI 在 Windows host

```bash
mfs profile add local --url http://127.0.0.1:8765                # 自动 local
mfs profile add prod  --url https://mfs.example.com              # 自动 remote
mfs profile add docker-dev --url http://127.0.0.1:8765 --kind remote   # 显式
```

profile 管理：

```bash
mfs profile add <name> --url <url> [--kind local|remote] [--tenant <id>]
mfs profile use <name>
mfs profile list
mfs profile status
```

### 多租户走 profile 层

tenant_id 放在 **profile 上**，不放 URI 里。connector URI 永远保持纯净（`postgres://prod`）。HTTP transport 自动按当前 profile 注入 `X-MFS-Tenant: <tenant_id>` header；server 端按这个 filter 数据。

```toml
[[profiles]]
name = "tenant-a-prod"
api_base_url = "https://mfs.example.com"
tenant_id = "tenant-a"
credential_ref = "env:MFS_TOKEN_A"

[[profiles]]
name = "tenant-b-prod"
api_base_url = "https://mfs.example.com"
tenant_id = "tenant-b"
credential_ref = "env:MFS_TOKEN_B"
```

`mfs profile use tenant-a-prod` 切换 = 切租户。同一台 client 可注册多个租户的 profile，按名字切换。

CLI 请求头：

```text
Authorization: Bearer <token>
MFS-CLI-Version: 0.4.0
MFS-API-Version: v1
```

## 3. Control plane vs data plane

**HTTP API 只走 control plane**：path / option / status / 元数据 / 搜索结果。**数据本身**（文件 bytes、external system 拉取的记录、chunk 文本、embedding）**都在 server 内部流动**，不经过 client。

### 同步场景的数据流

| profile + 输入 | 数据流 | HTTP 传什么 |
|---|---|---|
| `local` + 本地路径 | daemon 直接读本机文件 → 内部 chunk → 内部写 Milvus | 仅 path + options |
| `local` + 外部 URI | connector 调外部 API → 内部 chunk → 内部写 Milvus | 仅 URI + options |
| `remote` + 本地路径 | **拒绝** (`remote_server_cannot_read_local_path`) | 错误码 |
| `remote` + 外部 URI | 远端 server 调外部 API → server 内部 chunk → server 内部写 Milvus | 仅 URI + options |

### Client 不需要传 chunk 级数据

设计上 client 永远不需要做：

- 算文件 hash 给 server 比对（server 在本地路径下自己读、自己 hash）
- 拆 chunk 上传（chunk 是 server 内部产物）
- 上传 embedding（embedder 在 server 内部）

### 想用 remote server 处理本地文件？

走显式 **upload API**（v0.4 不开放，归商业化）：

```text
client 算本地文件 hash → diff server 已有 hash → 只 upload 变化的 bytes
server 拿到 bytes 后跑标准 chunk pipeline
```

这是唯一需要 client→server 传 data plane 的场景。v0.4 推荐的做法是**在 server 那台机器装 daemon，把本地文件放到 server 看得到的位置**，或者用 `kind=local` profile + 本机 daemon。

### tail -f 是 server → client 的 data plane

唯一例外：`mfs tail -f` 通过 SSE / chunked HTTP 把新增 record 推给 client。这是单向 control 触发 + 数据流，不构成"client 上传数据"。

## 4. 四种行为矩阵

| profile.kind | 输入类型 | 行为 |
|---|---|---|
| `local` | 本地路径 `./repo` | 支持。CLI/SDK 通过 HTTP 调本机 daemon，daemon 读本机文件 |
| `local` | 外部 URI `postgres://prod` | 支持。connector 在本机 daemon 里跑，索引写本机存储 |
| `remote` | 本地路径 `./repo` | 返回 `remote_server_cannot_read_local_path` |
| `remote` | 外部 URI `postgres://prod` | 支持。connector 在远端 server/worker 里跑，索引写远端存储 |

## 5. Server 端启动

MFS 的 server-side binary 是 **`mfs-server`**。`mfs serve` 是 client-side wrapper，本质是本机 spawn 一个 `mfs-server` 进程。

### 5.1 `mfs-server` 命令（运维用）

```bash
# 一体（demo / 小规模）
mfs-server run --bind 0.0.0.0:8765 --config /etc/mfs/daemon.toml

# 拆分（生产）
mfs-server api    --bind 0.0.0.0:8765 --config /etc/mfs/daemon.toml
mfs-server worker --concurrency 8     --config /etc/mfs/daemon.toml
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

## 6. 存储层

三套存储，职责清晰；每套的具体后端独立可换。

### 6.1 Metadata DB

local: SQLite `~/.mfs/metadata.db`
remote: Postgres

Schema:

```sql
connectors (
  id              VARCHAR PRIMARY KEY,
  tenant_id       VARCHAR DEFAULT 'default',
  root_uri        VARCHAR,
  type            VARCHAR,
  label           VARCHAR,
  config_json     TEXT,
  config_hash     VARCHAR,
  credential_ref  VARCHAR,
  registered_at   TIMESTAMP,
  last_health     TIMESTAMP,
  health_status   VARCHAR,
  UNIQUE (tenant_id, root_uri)
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

jobs (
  id              VARCHAR PRIMARY KEY,
  tenant_id       VARCHAR DEFAULT 'default',
  kind            VARCHAR,
  target_uri      VARCHAR,
  status          VARCHAR,
  progress_json   TEXT,
  started_at      TIMESTAMP,
  finished_at     TIMESTAMP,
  error           TEXT
);

manifest (
  connector_id    VARCHAR,
  path            VARCHAR,
  size            INTEGER,
  mtime_ns        BIGINT,
  hash            VARCHAR,
  last_seen       TIMESTAMP,
  PRIMARY KEY (connector_id, path)
);

cursors (
  connector_id    VARCHAR,
  key             VARCHAR,
  value           VARCHAR,
  updated_at      TIMESTAMP,
  PRIMARY KEY (connector_id, key)
);

watch_grants (
  path            VARCHAR PRIMARY KEY,
  granted_at      TIMESTAMP,
  profile_kind    VARCHAR
);
```

注意所有顶层表都预留 `tenant_id` 字段，默认 `'default'`，便于未来加多租户。

### 6.2 Object store (cache)

local: `~/.mfs/cache/`
remote: S3 / R2 / MinIO

目录布局：

```
~/.mfs/cache/
  <sha1(object_uri)>/
    converted_md
    page_cache.jsonl
    head_cache.jsonl
    vlm_text
    schema_dump.json
```

`caches` 表存 `(object_uri, cache_kind) → storage_path` 映射。

淘汰：

```toml
[cache]
max_size_gb = 10
eviction = "lru"
```

### 6.3 Milvus

一张 collection `mfs_chunks`，partition by connector。详见 [05-search-and-retrieval.md §1](05-search-and-retrieval.md#1-milvus-collection-schema)。

backends：

| 后端 | URI | 适合 |
|---|---|---|
| Milvus Lite | `~/.mfs/milvus.db` | local daemon 默认；零运维；单 writer |
| 自部署 Milvus | `http://host:19530` | self-host；多 writer；完整 BM25 |
| Zilliz Cloud | `https://*.zillizcloud.com` + token | 托管；多 writer；完整 BM25 |

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
kind = "remote"
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
mfs profile add local --url http://127.0.0.1:8765 --kind local
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
mfs profile add prod --url https://mfs.example.com --kind remote
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

- Metadata DB 所有顶层表加 `tenant_id` 字段（默认 `"default"`）
- Milvus `mfs_chunks` 表加 `tenant_id` scalar field（默认 `"default"`）
- 所有 query 默认 filter `tenant_id = current_tenant`
- HTTP header 预留 `X-MFS-Tenant`（v0.4 忽略；多租户启用后由 profile 的 `tenant_id` 自动注入）
- Profile 配置已支持 `tenant_id` 字段（v0.4 写了不报错，server 端忽略；启用多租户后生效）

### 未来启用方案

启用时 Milvus 隔离方案（按隔离强度排序）：

| 策略 | Milvus 实现 | 隔离强度 | 资源开销 |
|---|---|---|---|
| metadata filter | 共用 collection，每行带 `tenant_id`，查询时 filter | 弱（依赖 query 正确性） | 最小 |
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
- Milvus collection schema（含 `tenant_id` 多租户预留字段）。
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
