# 架构设计

MFS 的主架构是 client/server-first：CLI、SDK、Skill 都是 client；source/connector、同步、MFS Index、embedding、retrieval index 和存储都在 server 侧执行。本地体验由 CLI 连接本机 local daemon 提供。

## 0. 核心结论

统一模型：

```text
CLI / SDK / Skill
  -> HTTP API
  -> MFS server
  -> engine
  -> sources/connectors
  -> objects/pipeline
  -> storage/search
```

server 有两种部署位置：

| 部署位置 | 说明 |
| --- | --- |
| local daemon | 本机 server，与 CLI 共享文件系统命名空间，使用 SQLite、本地 object store、Milvus Lite 或本机 Milvus |
| remote server | 远程/云端 server，使用 Postgres、对象存储、queue、Milvus/Zilliz |

client 只负责命令/SDK 参数、HTTP、输出和本地配置。engine、source connector、Milvus/Zilliz 写入都属于 server 侧。

## 1. 总体架构

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                         Client side                                        │
│                                                                            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                      │
│  │ MFS CLI      │   │ Python SDK  │   │ TS/Go/Java  │                      │
│  │ mfs ...      │   │ MfsClient   │   │ SDKs        │                      │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                      │
│         │                 │                 │                             │
│         └─────────────────┴─────────────────┘                             │
│                           │                                                │
│                           v                                                │
│                 ┌────────────────────┐                                     │
│                 │ HTTP transport      │                                     │
│                 │ profile/auth/retry  │                                     │
│                 └─────────┬──────────┘                                     │
└───────────────────────────┼────────────────────────────────────────────────┘
                            │ HTTP / JSON / SSE for follow
                            v
┌────────────────────────────────────────────────────────────────────────────┐
│                         MFS server                                         │
│                                                                            │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐ │
│  │ API routes        │─────▶│ Engine           │─────▶│ Source plugins    │ │
│  │ /v1/add /v1/search│      │ orchestration    │      │ file/postgres/... │ │
│  └──────────────────┘      └─────────┬────────┘      └──────────────────┘ │
│                                      │                                     │
│                                      v                                     │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐ │
│  │ Worker runner     │─────▶│ Object handlers  │─────▶│ Pipeline/provider │ │
│  │ sync/index/export │      │ document/table   │      │ embedding/LLM     │ │
│  └──────────────────┘      └─────────┬────────┘      └──────────────────┘ │
│                                      │                                     │
│                                      v                                     │
│                         ┌──────────────────────────┐                       │
│                         │ Storage/search adapters  │                       │
│                         │ metadata/object/queue    │                       │
│                         │ Milvus/Zilliz/BM25       │                       │
│                         └──────────────────────────┘                       │
└────────────────────────────────────────────────────────────────────────────┘
```

## 2. Profile 与 daemon 是正交概念

`mfs daemon` 决定**本机是否跑 server 进程**。
`mfs profile` 决定**client 当前连哪个 endpoint**。

两者搭配出三种典型组合：

| 组合 | profile.kind | daemon 在本机 | 用途 |
| --- | --- | --- | --- |
| 本地一体 | `local` | 是 | 个人开发，daemon 跑本机，CLI 调本机 daemon |
| 远端 | `remote` | 否（或不相关） | 团队部署，CLI 调远端 server |
| 自部署 | `local` 指向 self-hosted box | 否（在那台 box 上） | 个人 self-host 一台 daemon，多台机器连过去 ❌ |

第三种**不算 local**：CLI 和 daemon 不共享文件系统，应改用 `remote` profile。

### 2.1 `kind = "local"` 的严格定义

`kind = "local"` 表示 **CLI 进程和 daemon 共享同一文件系统命名空间**——同一字符串路径在两边都能解析到同一文件。

以下场景**不满足**，应该按 `remote` 处理：

- daemon 跑在 Docker 容器，CLI 在 host（除非显式 bind mount 整个相关目录树）。
- daemon 跑在 WSL，CLI 在 Windows host。
- daemon 跑在 SSH 远端 box，CLI 在笔记本（即使用 port forward 把 127.0.0.1 指过去）。

`kind` 的实际作用：

- `local`：本地 path 请求由 daemon 直接读文件。
- `remote`：本地 path 请求返回 `remote_server_cannot_read_local_path`，强制用户用 source URI 或 upload API。

## 3. 四种行为矩阵

按"profile kind / 输入类型"理解：

| profile kind | 输入类型 | 行为 |
| --- | --- | --- |
| `local` | file path：`./repo`、`/data/repo` | 支持。CLI/SDK 发 HTTP 到本机 daemon，daemon 读取本机文件、scan、chunk、index |
| `local` | source URI：`postgres://prod` | 支持。connector 在本机 daemon 里跑，索引写本机存储 |
| `remote` | file path：`./repo` | 返回 `remote_server_cannot_read_local_path` |
| `remote` | source URI：`postgres://prod` | 支持。connector 在 remote server/worker 里跑，索引写远端存储 |

server mounted path 由管理员通过显式 source 配置表达；不复用普通本机路径语义。

## 4. 本地体验

本地体验使用简单命令：

```bash
mfs add .
mfs search "retry policy" .
mfs cat ./docs/auth.md
```

内部：

```text
mfs CLI
  -> http://127.0.0.1:<port>
  -> local daemon
  -> server engine
  -> sources/file/
  -> local SQLite/object store/Milvus Lite
```

本机 daemon 启动策略：

| 策略 | 行为 |
| --- | --- |
| 自动启动 | `mfs add .` 检测不到 daemon 时尝试启动，失败再提示 |
| 显式启动 | 用户先运行 `mfs daemon start`，CLI 只负责提示 |

默认显式启动；自动启动作为 client 体验增强能力，不影响 server 架构。

## 5. 远程文件策略

remote profile 下，本地 path 请求返回明确错误：

```bash
mfs add ./repo
```

错误输出：

```text
Local path requires local profile: ./repo
Run `mfs profile use <local-profile>` and `mfs daemon start` for local files.
```

云端文件能力使用两个显式入口：

| 能力 | 说明 |
| --- | --- |
| upload API | client 显式上传文件或 changed chunks 到 remote server（v0.4 不开放，归商业化） |
| server-side file source | 管理员显式注册 server mounted path，普通用户命令不隐式使用 |

## 6. Source、Object、Pipeline

职责边界：

| 层 | 职责 |
| --- | --- |
| `sources/` | 数据从哪里来、怎么连接、怎么同步、怎么映射成 MFS object |
| `objects/` | 数据属于什么形态，如何规范化、切 chunk、构建结构 |
| `pipeline/` | 执行通用 provider 和检索流程，例如 embedding、summary、ranking |
| `storage/` | metadata、object store、queue、search backend 的适配 |

中间层 `objects/` 按对象形态复用处理逻辑。例如 Google Doc、本地 PDF、Feishu Doc 都映射成 `object_kind=document`，统一走 `objects/document/`。

**对象命名约定**：每个 source root 下暴露什么对象由 source 决定，对象名带 media type 后缀（`schema.json` / `sample.jsonl` 等）。详细命名见 [17-source-sync-and-mfs-index.md](17-source-sync-and-mfs-index.md#3-uri-映射规则)。

## 7. Client、SDK 和 Skill

CLI 是官方 client，通过 Python SDK / HTTP client 访问 MFS API：

```text
mfs CLI
  -> Python SDK / HTTP client
  -> MFS API
```

SDK 分层：

| SDK | 交付 |
| --- | --- |
| Python SDK | PyPI 包内手写，供 CLI 复用，也供 Python 程序调用 |
| TypeScript SDK | npm 包，基于 OpenAPI 生成或手写薄封装 |
| Go/Java SDK | 基于 OpenAPI 生成 |

Skill 也只依赖 CLI 或 HTTP SDK，不直接触碰 engine、source、pipeline。

## 8. 部署形态

| 形态 | 用途 |
| --- | --- |
| local daemon | 本机开发、agent 本地文件搜索、个人工作流 |
| remote server | 团队/云端 source、后台同步、集中索引 |
| `mfs-server` Docker image | 单容器 server，适合 demo、小团队、本地 Docker |
| `mfs-api` + `mfs-worker` | 正式部署，API 和长任务分开扩缩容 |

local daemon 是 Python server 的一个 entrypoint。Docker 用户在本机运行 server 时使用 `mfs-server` image。
