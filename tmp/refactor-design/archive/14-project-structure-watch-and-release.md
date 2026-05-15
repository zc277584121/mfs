# 项目目录结构、目录监控与发布版本

本文定义 MFS 的工程组织。MFS 是 client/server-first：CLI、SDK、Skill 都是 client；source/connector、同步、MFS Index、embedding、retrieval index 和存储都在 server 侧执行。本地体验通过本机 local daemon 实现。

## 1. 目标目录结构

目标结构按 client、protocol、server 和部署物分开。下面是 `tree` 风格的目录图，每行 `#` 后面说明职责。

```text
.
├── clients/                                      # 所有客户端实现：CLI、SDK、HTTP transport、输出
│   ├── python/                                  # PyPI distribution: mfs-cli
│   │   ├── pyproject.toml                       # mfs-cli 包配置，包含 CLI、Python SDK、HTTP client
│   │   └── src/mfs_client/                      # Python client package
│   │       ├── cli/                             # `mfs` 命令行入口和终端输出
│   │       │   ├── main.py                      # 注册 CLI 子命令
│   │       │   ├── commands/                    # 命令参数解析、确认和输出
│   │       │   │   ├── add.py                   # `mfs add` 统一注册+同步入口（本地 path 或 source URI）
│   │       │   │   ├── source.py                # `mfs source list/inspect/update/remove`
│   │       │   │   ├── daemon.py                # `mfs daemon start/stop/status/logs`
│   │       │   │   ├── profile.py               # `mfs profile add/use/list/status`
│   │       │   │   ├── job.py                   # `mfs job list/inspect/cancel`
│   │       │   │   └── ...                      # search/grep/ls/tree/cat/head/tail/export/status/remove/config
│   │       │   ├── output/                      # 人类可读输出、颜色、pipe-aware 行为
│   │       │   └── profiles.py                  # local daemon / remote server profile 选择
│   │       ├── sdk/                             # Python SDK：MfsClient、请求封装、分页 iterator
│   │       ├── transport/                       # HTTP、retry、timeout、streaming page、错误映射
│   │       ├── models/                          # 从 protocol 生成或同步的请求/响应模型
│   │       └── config/                          # client config、profile、token/credential_ref 读取
│   ├── typescript/                              # npm SDK，基于 OpenAPI 生成或薄封装
│   ├── go/                                      # Go SDK
│   └── java/                                    # Java SDK
├── protocol/                                    # client/server 的稳定协议，独立于具体运行时
│   ├── openapi.yaml                             # HTTP API 合同
│   ├── schemas/                                 # Source、Object、Job、SearchResult、Error 等 JSON schema
│   ├── errors.md                                # 稳定错误码和用户可见修复说明
│   └── compatibility.md                         # CLI/API/server 版本兼容规则
├── server/                                      # 服务端实现；本地 daemon、远端 API、worker 共用
│   └── python/                                  # PyPI distribution: mfs-server
│       ├── pyproject.toml                       # mfs-server 包配置，包含 API、daemon、worker、engine、sources
│       └── src/mfs_server/                      # Python server package
│           ├── api/                             # HTTP API server
│           │   ├── app.py                       # app 创建、middleware、路由挂载
│           │   ├── routes/                      # `/v1` API 路由
│           │   │   ├── add.py                   # `/v1/add` 统一入口（file path 或 source URI）
│           │   │   ├── sources.py               # source list/inspect/update/remove
│           │   │   ├── objects.py               # ls/tree/stat/cat/head/tail/follow/export
│           │   │   ├── search.py                # search/grep
│           │   │   ├── profiles.py              # （可选）remote 侧的 endpoint capability/version
│           │   │   └── jobs.py                  # job list/inspect/cancel
│           │   └── middleware/                  # request id、错误转换、版本检查、认证占位
│           ├── daemon/                          # 本机 local daemon 入口
│           │   ├── main.py                      # `mfs-server daemon` entrypoint，监听 127.0.0.1
│           │   ├── lifecycle.py                 # pid、socket/port、日志、启动/停止
│           │   └── autostart.py                 # 自动启动体验增强
│           ├── worker/                          # 后台任务执行入口
│           │   ├── main.py                      # `mfs-server worker` entrypoint
│           │   ├── runner.py                    # queue 消费、并发、重试、心跳
│           │   └── jobs/                        # job handler
│           │       ├── sync_source.py           # source 同步
│           │       ├── build_index_artifacts.py # metadata/text/structure/chunk/summary
│           │       ├── build_retrieval_index.py # embedding、BM25、Milvus/Zilliz 写入
│           │       ├── export_object.py         # 大对象或查询结果导出
│           │       └── summarize.py             # summary/description 生成
│           ├── runtime/                         # server 运行时依赖装配
│           │   ├── local_daemon.py              # SQLite、本地 object store、本地 queue、Milvus Lite/本机 Milvus
│           │   └── remote_server.py             # Postgres、对象存储、Redis/Postgres queue、Milvus/Zilliz
│           ├── engine/                          # server 侧业务编排
│           │   ├── service.py                   # add/search/grep/cat/sync/status 的统一编排
│           │   ├── resolver.py                  # path/URI 解析和 object 定位
│           │   ├── index_artifacts.py           # MFS Index 计划、版本、freshness
│           │   ├── policy.py                    # 限制、确认、权限、可见性策略
│           │   ├── jobs.py                      # job 规划和状态模型
│           │   └── errors.py                    # server 内部错误到 protocol 错误码的映射
│           ├── sources/                         # source 插件层；每类 source 自包含
│           │   ├── base.py                      # SourcePlugin、Connector、Sync、Mapper 抽象接口
│           │   ├── registry.py                  # source 类型注册、发现和实例化
│           │   ├── file/                        # 本地文件 source；只在能读到 path 的 daemon/server 上执行
│           │   │   ├── plugin.py                # 注册 file source 类型和能力
│           │   │   ├── config.py                # file source 配置、默认排除规则
│           │   │   ├── connector.py             # 读取本机文件对象和基础 metadata
│           │   │   ├── sync.py                  # manifest diff、mtime/hash 判断、删除检测
│           │   │   ├── scanner.py               # 目录遍历、stat、文件类型初判
│           │   │   ├── manifest.py              # path、size、mtime_ns、hash、last_seen_at
│           │   │   ├── ignore.py                # `.gitignore`、默认忽略和用户忽略规则
│           │   │   ├── watch.py                 # watch 事件转成重新扫描提示
│           │   │   ├── mapper.py                # 本地 path 到 MFS object/path/locator 的映射
│           │   │   └── defaults.py              # file source 默认 MFS Index 和搜索字段
│           │   ├── postgres/                    # DB source 示例：schema、sample、rows、query result
│           │   ├── slack/                       # 消息 source 示例：channel、message、thread、file
│           │   └── ...                          # github/gdrive/feishu/s3/gmail/mysql/bigquery/jira/linear/notion/zendesk/salesforce/hubspot
│           ├── objects/                         # 中间对象层；按对象形态复用处理逻辑
│           │   ├── base.py                      # ObjectHandler、ObjectReader、IndexArtifactBuilder 抽象
│           │   ├── document/                    # PDF/DOCX/Markdown/Google Doc/Feishu Doc 等文档对象
│           │   │   ├── normalize.py             # 文档抽取文本、encoding、媒体类型判断
│           │   │   ├── chunk.py                 # 文档 chunk、section path、range pointer
│           │   │   └── structure.py             # heading tree、目录结构、skim/peek/deep 输入
│           │   ├── code/                        # 代码文件对象
│           │   │   ├── ast.py                   # AST 解析
│           │   │   ├── symbols.py               # symbol tree
│           │   │   └── chunk.py                 # AST-aware chunk
│           │   ├── table/                       # DB table、CSV、Sheet、warehouse result
│           │   │   ├── schema.py                # schema/sample/field metadata
│           │   │   ├── jsonl.py                 # rows/sample/query result 的 JSONL 表达
│           │   │   └── chunk.py                 # row/window/field 拼接文本
│           │   ├── message_thread/              # Slack/Discord/Gmail/飞书群聊
│           │   │   ├── window.py                # day/session/thread/time-window
│           │   │   └── chunk.py                 # thread/session 聚合文本
│           │   ├── task_record/                 # Jira/Linear/GitHub Issues/PRs/Notion database
│           │   ├── business_record/             # Salesforce/HubSpot/Zendesk 等业务对象
│           │   └── binary/                      # 图片、附件、未知二进制对象
│           ├── pipeline/                        # 通用 provider 和检索执行，source 语义由 sources 提供
│           │   ├── embedding/                   # embedding provider 抽象和实现
│           │   ├── generation/                  # summary、image description、LLM provider
│           │   ├── retrieval/                   # search/grep planning、ranking、density、index unit
│           │   └── export/                      # 大对象导出、分页流、格式转换
│           ├── storage/                         # 存储适配层，engine 只依赖接口
│           │   ├── metadata/                    # SQLite/Postgres metadata store
│           │   ├── object_store/                # local/S3/R2/MinIO object store
│           │   ├── queue/                       # in-process/SQLite/Redis/Postgres queue
│           │   └── search/                      # Milvus Lite/Milvus/Zilliz/BM25 backend
│           └── config/                          # server/source/provider TOML、secret 引用、schema 校验
├── deployments/                                 # 部署物和运维脚本
│   ├── docker/                                  # Dockerfile 和 entrypoint
│   │   ├── mfs-server.Dockerfile                # 单容器 server：API + worker，适合 demo/小规模
│   │   ├── mfs-api.Dockerfile                   # 只跑 API
│   │   └── mfs-worker.Dockerfile                # 只跑 worker
│   ├── compose/                                 # docker compose 示例
│   ├── helm/                                    # Kubernetes chart
│   └── systemd/                                 # VM/systemd 部署示例
├── tests/                                       # 单测、协议测试、fake source、E2E
│   ├── client/                                  # CLI/SDK 输出、profile、HTTP 错误映射
│   ├── server/                                  # API、runtime、engine、storage
│   ├── sources/                                 # 各 source 的 fake connector 和 contract tests
│   ├── e2e/                                     # daemon + file、daemon + source、remote + source
│   └── fixtures/                                # fake Postgres/Slack/Zendesk/GitHub/Drive 数据
├── docs/                                        # 面向用户和开发者的正式文档
├── skills/                                      # Agent Skill，调用 CLI 或 HTTP SDK
└── tmp/refactor-design/                         # 重构设计文档
```

交互版目录和信息流见 [18-project-structure-flow.html](18-project-structure-flow.html)。

## 2. 依赖方向

关键规则是：client 依赖 protocol 和 HTTP transport；server 通过 protocol 对外暴露稳定接口。

```text
┌──────────────────────────────────────────────────────────────────────┐
│                            Client packages                           │
│                                                                      │
│  mfs CLI  ─┐                                                         │
│  Python SDK├──▶ HTTP transport ───▶ protocol models/errors           │
│  Skill    ─┘                                                         │
└───────────────────────────────────────────────┬──────────────────────┘
                                                │ HTTP / JSON / pages
                                                v
┌──────────────────────────────────────────────────────────────────────┐
│                             MFS server                                │
│                                                                      │
│  API routes ─▶ runtime ─▶ engine ─▶ sources ─▶ objects ─▶ pipeline    │
│      │              │          │          │          │               │
│      │              └──────────┴──────────┴──────────┴──▶ storage    │
│      │                                                          │     │
│      └────────────── job queue ───────────────▶ worker ─────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

这带来几个直接约束：

- `clients/python` 只依赖 protocol 和 HTTP transport。
- `mfs_client.transport` 只知道 HTTP、profile、retry、错误码和分页。
- `mfs_server.sources` 产出 object、change set 和 MFS Index 计划；Milvus/Zilliz 写入由 pipeline/storage 执行。
- `mfs_server.objects` 复用文档、代码、表格、消息、ticket 等对象形态的处理。
- `mfs_server.pipeline` 放 embedding、summary、ranking、export 这些通用执行能力。
- `mfs_server.storage` 是 metadata、object store、queue、search backend 的适配层。

## 3. 四种行为矩阵

四种情况按“server 位置 * 输入类型”理解。

| server 位置 | 输入类型 | 行为 |
| --- | --- | --- |
| profile.kind=local | file path：`./repo`、`/data/repo` | 支持。CLI/SDK 通过 HTTP 调本机 daemon，daemon 读取本机文件、扫描、变化检测、构建 MFS Index 并写本机 Retrieval Index |
| profile.kind=local | source URI：`postgres://prod` | 支持。connector 在本机 daemon 里执行，适合个人本地 source 或开发调试 |
| profile.kind=remote | file path：`./repo` | 返回 `remote_server_cannot_read_local_path`；本机文件由 local profile + daemon 处理 |
| profile.kind=remote | source URI：`postgres://prod` | 支持。connector 在远端 API/worker 侧执行，MFS Index 和 Retrieval Index 写远端存储 |

`kind=local` 的严格定义见 [11-architecture-design.md](11-architecture-design.md#21-kind--local-的严格定义)：CLI 进程和 daemon 共享同一文件系统命名空间。Docker / SSH / WSL 等跨命名空间组合按 `remote` 处理。

### 3.1 Local daemon + file path

```text
mfs add ./repo
  -> mfs_client.cli.commands.add
  -> mfs_client.transport HTTP POST /v1/add
  -> local daemon mfs_server.api.routes.add
  -> mfs_server.runtime.local_daemon
  -> mfs_server.sources.file sync/manifest/scanner
  -> mfs_server.objects document/code/table handlers
  -> mfs_server.pipeline embedding/retrieval/generation
  -> mfs_server.storage local metadata/object/search
```

mtime、hash、manifest、chunk 与 Milvus Lite/本机 Milvus 的状态对比都发生在 daemon 内部。client 只发送 path 和选项；stored `file_hash` 读取和 changed chunks 生成都在 daemon 内部完成。

### 3.2 Local daemon + source URI

```text
mfs add postgres://prod
  -> client HTTP POST /v1/add
  -> local daemon API
  -> local daemon runtime
  -> sources.postgres connector/sync/mapper
  -> objects.table schema/jsonl/chunk
  -> local metadata/object/search
```

这条路径适合本机有数据库或 SaaS 凭据、希望本地索引和搜索的用户。

### 3.3 Remote profile + file path

```text
mfs add ./repo
  -> active profile.kind = remote
  -> client resolver sees local path
  -> return remote_server_cannot_read_local_path
```

云端索引本地文件使用显式能力（v0.4 不开放）：

- `mfs upload ./repo --to <source-or-workspace>`：用户明确上传（归商业化）。
- server-side file source：管理员明确注册 server mounted path。

### 3.4 Remote profile + source URI

```text
mfs add postgres://prod
  -> client HTTP POST /v1/add
  -> remote API validates source
  -> queue sync/build/index jobs
  -> mfs-worker consumes jobs
  -> mfs_server.runtime.remote_server
  -> sources.postgres connector/sync/mapper
  -> objects.table schema/jsonl/chunk
  -> mfs_server.pipeline embedding/retrieval/generation
  -> storage Postgres/object store/Zilliz
```

这是团队或云端部署的主路径。

## 4. Source、Object、Pipeline 的边界

每个 source 的配置、变化检测、同步和路径映射都由 source 目录自包含。

每类 source 的内部骨架保持一致：

```text
sources/<type>/
├── plugin.py          # 注册 source type、URI scheme、能力和默认行为
├── config.py          # 解析该 source 的 TOML 和 secret 引用
├── connector.py       # 连接外部系统或本机文件系统
├── sync.py            # cursor、updated_at、revision、manifest、CDC、snapshot 等变化检测
├── mapper.py          # 外部对象到 MFS path/object/locator 的映射
├── index_artifacts.py # 该 source 默认生成哪些 MFS Index 内容
├── defaults.py        # 默认搜索字段、metadata 字段、分页限制
└── schema.py          # 配置 schema 和能力声明
```

`sources/file/` 额外包含 `scanner.py`、`manifest.py`、`ignore.py`、`watch.py`。这些文件只属于 file source。

`objects/` 是 source 和 pipeline 中间的抽象层。它回答“这个对象是什么形态”，source 负责回答“来自哪个系统”：

| Object kind | 复用场景 | 默认处理 |
| --- | --- | --- |
| `document` | 本地 PDF/DOCX/Markdown、Google Doc、Feishu Doc、Notion page | 抽取文本、heading/section、文档 chunk |
| `code` | 本地 repo、GitHub repo、代码附件 | AST、symbol tree、code chunk |
| `table` | Postgres/MySQL、CSV、Sheet、BigQuery/Snowflake result | schema、sample、JSONL、row/window chunk |
| `message_thread` | Slack、Discord、Gmail、飞书群聊 | `messages`、`threads`、thread/session/time-window、thread summary |
| `task_record` | Jira、Linear、GitHub Issues/PRs、Notion database | issue/PR/record、comments、timeline |
| `business_record` | Salesforce、HubSpot、Zendesk | record、comments、activities、业务字段摘要 |
| `binary` | 图片、附件、未知二进制 | metadata、可选 OCR/图片描述、按需导出 |

`pipeline/` 提供可复用执行能力，source 策略由 `sources/<type>/` 提供：

- `pipeline/embedding/`：embedding provider。
- `pipeline/generation/`：summary、image description、LLM provider。
- `pipeline/retrieval/`：search/grep planning、ranking、density、index unit。
- `pipeline/export/`：大对象导出和分页流。

## 5. 目录监控与本地文件同步

目录监控在能读取文件的 server 侧执行；本机 path 的 watcher 由 local daemon 执行。

命令保持简单：

```bash
mfs daemon start
mfs add . --watch --interval 60s
```

行为：

- CLI 把 path、`--watch`、`--force` 等参数通过 HTTP 发给 local daemon。
- daemon 启动时先做一次完整扫描。
- watch 事件只作为“需要重新扫描”的提示。
- 最终事实来自 `sources/file/manifest.py` 的扫描和对比。
- manifest 默认使用 `path + size + mtime_ns` 快速判断，必要时再算内容 hash。
- hash 变化后才重建 normalized content、structure、chunks、summary 和 index。
- `--force` 跳过快速判断，重建可重建的 MFS Index 内容。

manifest 记录：

```text
path
size
mtime_ns
file_hash
file_type
encoding
last_seen_at
artifact_version
```

权限确认示例：

```text
MFS local daemon will watch this directory:
  /repo

It will read file names, mtimes, sizes, and indexable file contents.
State is stored under ~/.mfs/.

Continue? [y/N]
```

确认后记录在本机 daemon 状态里：

```toml
[[watch.allowed_roots]]
path = "/repo"
granted_at = "2026-05-11T12:00:00Z"
profile_kind = "local"
```

查看正在 watch 的目录：

```bash
mfs status --watch
```

remove 同时停 watcher：

```bash
mfs remove ./repo
```

remote profile 下的普通本地路径返回：

```text
Local path requires local profile: ./repo
Run `mfs profile use <local>` and `mfs daemon start` for local files.
```

## 6. PyPI 包设计

对外命令名是 `mfs`。PyPI 包分成 client 包 `mfs-cli` 和服务端包 `mfs-server`。

| PyPI 包 | 内容 | 边界 | 入口 |
| --- | --- | --- | --- |
| `mfs-cli` | Python SDK、CLI、HTTP transport、profile、输出、protocol models | client only | `mfs` |
| `mfs-server` | API、local daemon、worker、engine、sources、objects、pipeline、storage | server only | `mfs-server` |

安装体验：

```bash
uv tool install mfs-cli
uv tool install mfs-server
mfs daemon start
mfs profile add local --url http://127.0.0.1:8765 --kind local
mfs profile use local
mfs add .
```

提供一个 convenience extra，用于缩短本地 quickstart；它只安装依赖，不改变包边界：

```bash
uv tool install "mfs-cli[local]"
```

`mfs-cli[local]` 依赖 `mfs-server`，让本地 quickstart 更短；代码仍然分在两个 distribution 里。

服务端 optional extras 放在 `mfs-server`：

```text
mfs-server[postgres]
mfs-server[mysql]
mfs-server[bigquery]
mfs-server[slack]
mfs-server[gdrive]
mfs-server[feishu]
mfs-server[s3]
mfs-server[salesforce]
mfs-server[hubspot]
mfs-server[zendesk]
mfs-server[embedding-local]
mfs-server[embedding-openai]
mfs-server[llm-openai]
mfs-server[llm-anthropic]
mfs-server[zilliz]
mfs-server[all]
```

多语言 SDK 进入各自生态：

| SDK | 包管理生态 |
| --- | --- |
| TypeScript SDK | npm |
| Go SDK | Go module |
| Java SDK | Maven/Gradle |

这些 SDK 都从 `protocol/openapi.yaml` 生成或手写薄封装，行为与 Python SDK 对齐。

## 7. Docker image 切分

Docker 交付 server 侧。CLI 通常通过 PyPI 安装；CLI 工具镜像作为独立部署物。

| Image | 用途 | 入口 | 包含 |
| --- | --- | --- | --- |
| `ghcr.io/zilliztech/mfs-server:<version>` | demo、小规模自部署、单机试用 | API + worker 同容器 | `mfs-server` package、server runtime、sources、objects、pipeline、storage |
| `ghcr.io/zilliztech/mfs-api:<version>` | 正式部署的 HTTP API | API server | API、runtime、engine、轻量 object read、job 创建 |
| `ghcr.io/zilliztech/mfs-worker:<version>` | 正式部署的后台任务 | worker runner | worker、sources、objects、pipeline、embedding/index/export |

`mfs-api` 和 `mfs-worker` 使用同一个 `mfs-server` distribution，只是 entrypoint 和依赖组合不同。边界按运行职责切：

- API 镜像负责 HTTP API 和 job 创建。
- Worker 镜像负责长任务循环。
- 两者共享 protocol、engine、source、object、storage 接口。
- source/provider SDK 按镜像需要安装，API 镜像只安装读取和任务创建所需依赖，worker 镜像安装同步、embedding 和 retrieval index 构建所需依赖。

local daemon 是 `mfs-server` 的一个本机 entrypoint。Docker 用户要本地跑 server，直接用 `mfs-server` image 或 compose。

## 8. 部署文件关系

```text
deployments/docker/mfs-server.Dockerfile
  -> installs mfs-server[all] or selected extras
  -> entrypoint runs API + worker for demo/small use

deployments/docker/mfs-api.Dockerfile
  -> installs mfs-server[api,zilliz,postgres]
  -> entrypoint runs API only

deployments/docker/mfs-worker.Dockerfile
  -> installs mfs-server[worker,zilliz,postgres,slack,...]
  -> entrypoint runs worker only
```

Compose 示例：

```bash
docker compose up -d postgres minio redis milvus mfs-api mfs-worker
```

单容器示例：

```bash
docker run --rm -p 8765:8765 ghcr.io/zilliztech/mfs-server:0.4.0
```

## 9. 模块归位

单机 Python package 的模块按职责归位到 client 或 server。

| 源模块 | 目标位置 | 说明 |
| --- | --- | --- |
| `src/mfs/cli.py`、`src/mfs/__main__.py` | `clients/python/src/mfs_client/cli/` | CLI 作为 HTTP client，只处理命令、profile 和输出 |
| `src/mfs/cli_config.py`、`src/mfs/config.py` | `clients/python/.../config/` 和 `server/python/.../config/` | client profile 与 server/source/provider 配置分开 |
| `src/mfs/ingest/scanner.py` | `server/python/src/mfs_server/sources/file/scanner.py` | file source 专属扫描 |
| `src/mfs/ingest/queue.py` | `server/python/src/mfs_server/storage/queue/` | queue 变成 server 侧适配器 |
| `src/mfs/ingest/worker.py` | `server/python/src/mfs_server/worker/` | worker 是 server 长任务入口 |
| `src/mfs/ingest/converter.py` | `server/python/src/mfs_server/objects/document/`、`objects/table/`、`objects/binary/` | 按对象形态拆分 |
| `src/mfs/ingest/chunker.py` | `server/python/src/mfs_server/objects/document/chunk.py`、`objects/table/chunk.py` | 普通文本/表格 chunk 归对象层 |
| `src/mfs/ingest/ast_chunker.py` | `server/python/src/mfs_server/objects/code/` | 代码 AST 和 symbol tree 归 code object |
| `src/mfs/embedder/*` | `server/python/src/mfs_server/pipeline/embedding/` | embedding provider 属于 server pipeline |
| `src/mfs/llm/*` | `server/python/src/mfs_server/pipeline/generation/` | summary/description provider 属于 server pipeline |
| `src/mfs/search/searcher.py`、`src/mfs/search/density.py` | `server/python/src/mfs_server/pipeline/retrieval/` | 检索规划、排序、density 属于 server pipeline |
| `src/mfs/search/summary.py` | `server/python/src/mfs_server/pipeline/generation/` | summary 是 MFS Index 的生成类内容 |
| `src/mfs/output/*` | `clients/python/src/mfs_client/cli/output/` 和 `protocol/schemas/` | 终端展示与稳定 JSON schema 分开 |
| `src/mfs/store.py` | `server/python/src/mfs_server/storage/` | 拆成 metadata、object store、queue、search backend |

## 10. 版本策略

版本使用语义化版本：

```text
0.x.y  快速迭代阶段
1.0.0  CLI、local daemon、多源基础行为和 API v1 稳定
```

兼容面：

- `mfs` CLI 命令和参数（16 个顶级命令，详见 [15](15-command-inventory-and-scope.md#12-数量总结)）。
- Python SDK 方法和错误类型。
- HTTP API `/v1`。
- JSON/JSONL 输出 envelope（无 cursor token）。
- source URI 与对象命名约定（对象名带 media type 后缀）。
- source TOML / Index Plan schema。
- server image entrypoint。

CLI/server 兼容关系写入 release note：

```text
MFS CLI 0.4.x supports MFS API v1.
Minimum server version: 0.4.0.
Recommended server version: 0.4.x.
```

## 11. Release 检查

```bash
uv run pytest
uv build
docker build -f deployments/docker/mfs-server.Dockerfile .
docker build -f deployments/docker/mfs-api.Dockerfile .
docker build -f deployments/docker/mfs-worker.Dockerfile .
```

发布前需要确认：

- `mfs-cli` 只依赖 protocol、client models 和 HTTP transport。
- `mfs-server` 只复用 protocol，不依赖 CLI 交互层。
- OpenAPI 与 Python client/server models 同步。
- `mfs add` 在本地路径和 source URI 上幂等 E2E 通过。
- remote profile + file path 返回明确错误。
- remote profile + source URI E2E 通过。
- `cat --range` / `head` / `tail -f` / `export` 在 fake source 上 E2E 通过。
- 大对象 `cat`（无 `--range`）拒绝并给出建议。
- Docker compose 能跑 API + worker + storage/search backend。
- release note 只写用户可见变化、兼容性和升级说明。
