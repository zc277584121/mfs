# MFS 设计总览

MFS 是 **Multi-source File-like Search**：让 agent 用一套 shell-native CLI 搜索、浏览和读取本地文件、代码仓库、云文档、消息、ticket、数据库、SaaS 记录和网页。

- 所有数据通过路径或 URI 寻址
- 所有命令是 POSIX 风格（`ls / cat / grep / head / tail / tree`）
- 不挂载文件系统——"file-like" 指 URI 寻址 + POSIX 命令，不是 FUSE
- 检索是混合检索（语义 + 关键词），Milvus 一张 collection 共用

一句话：

> MFS lets agents search and inspect local files, repos, cloud docs, chats, tickets, databases, SaaS records, and web pages through one shell-native CLI. Shell-first — agents drive it via plain commands; SDKs (Python / TypeScript / Go / Java) are available when you need programmatic access.

## 四个核心抽象

```
Connector  ─ 一个注册的数据连接器       postgres://prod / ./repo / slack://eng
Object     ─ connector 暴露的一条虚拟文件（path + media_type）
Cache      ─ 一个 object 的本地缓存字节（可选，让 cat/head/tail 不打回 connector）
Chunk      ─ Milvus 一行：能被 search/grep 召回的最小单元
```

整个系统对外只有这四个概念。每个 connector 决定自己 root 下面暴露哪些 object，每个 object 按需生成 cache 和 chunks。

**本地文件也是一种 connector**：scheme 是 `file`，用户写普通 path 即可。`postgres connector` / `slack connector` / `file connector` 在概念上一视同仁——同样的 list / stat / read / fingerprint 契约，同样的 chunk pipeline，同样的搜索能力。

`Connector` 这个词沿用业界 ETL / iPaaS 的用法（Airbyte / Kafka Connect / Snowflake Connector），避免跟 shell `source` 命令的语义混淆。

## 系统全景

```
            ┌────────────────── Client ──────────────────┐
            │ mfs CLI / Python SDK / Skill                │
            │   parse args · profile · HTTP transport      │
            └──────────────────┬──────────────────────────┘
                               │  HTTP /v1（主要是 control plane）
                               ▼
            ┌────────────────── Server ──────────────────┐
            │  API routes  →  Engine  →  Connector        │
            │      │            │            │            │
            │      └──→  Object Processor →  Common       │
            │                                Service      │
            │                                (embed/VLM)  │
            │                       │                     │
            │                       ▼                     │
            │   ┌──────────┐  ┌──────────┐  ┌──────────┐ │
            │   │ Metadata │  │  Object  │  │  Milvus  │ │
            │   │   DB     │  │  store   │  │ 一张表   │ │
            │   │          │  │ (cache)  │  │          │ │
            │   └──────────┘  └──────────┘  └──────────┘ │
            └─────────────────────────────────────────────┘
```

三套存储职责清晰，互不重叠：

| 存什么 | 存在哪 | 目的 |
|---|---|---|
| Connector / Object / Cache / Job 关系 + 状态 | Metadata DB（SQLite 或 Postgres） | path index、状态、变化检测 |
| Cache 字节（converted markdown / page cache / VLM description） | Object store（本地 fs / S3 / R2 / MinIO） | 让 `cat / head / tail` 不打回 connector |
| 可检索的 chunk | Milvus 一张 collection | search / grep 召回 |

具体后端由 server 配置决定，跟 client 端 profile 无关。

## Client / Server 与 profile

CLI、SDK 是 client，所有重活在 server。server 有两种部署位置：

| 部署 | 用途 |
|---|---|
| 本机 server（`mfs serve`） | 个人本机，CLI 和 server 共享文件系统 |
| 远端 server | 团队 / 云端，CLI 通过 HTTPS 访问 |

CLI 跟 server 是否共享 fs 由 machine-id 自动比对决定，用户不需要手动配。一致就走"server 直接读本机"，不一致就走 upload flow（详见 [02 §4.2](02-architecture.md#42-本地文件-upload-flow不共享-fs-时)）。Docker / WSL2 / SSH forward 都会自动判 remote。

是否共享 fs 跟存储后端正交——本机 server 也能用 Postgres + Zilliz Cloud + S3，远端 server 也能用 SQLite + 本地 fs。详见 [02 §3](02-architecture.md#3-client-和-server)。

## 最小心智模型

```text
mfs serve start                       启动本机 server 进程
mfs profile use local                 选择本机 server
mfs add .                             注册并同步本地 file connector
mfs add <connector-uri> --config X    首次注册外部 connector
mfs add <connector-uri>               已注册再跑 = 再同步一次

mfs ls / tree <uri>                   浏览结构
mfs cat <uri>                         读对象（大对象拒绝，提示用 head/tail/range）
mfs head -n N / tail -n N             看端点
mfs cat <uri> --range A:B             按行/记录区间
mfs export <uri> <file>               完整导出到本地

mfs search "..." <path>               语义混合搜索
mfs grep "..." <path>                 精确搜索（connector 可下推）

mfs status [<uri>]                    看 server / connector / freshness / job
mfs connector list/inspect/probe/update/remove
mfs job list/inspect/cancel
```

## 关键设计决策

| 决策 | 详见 |
|---|---|
| `mfs add` 统一注册 + 同步入口，幂等 | [03 §3](03-cli-commands.md#3-mfs-add-是-mfs-connector-add-的高频别名) |
| 对象名带 media type 后缀（`schema.json` / `rows.jsonl` 等） | [09](09-connector-catalog.md) |
| 分页用 `--range A:B`，不需要 cursor token | [05 §4](05-browse-and-read.md#4-分页与大对象) |
| Milvus 一张 collection，partition_key 按 connector_uri | [06 §1](06-search-and-retrieval.md#1-milvus-collection-schema) |
| 检索字段配置：text_fields / metadata_fields / locator_fields + chunk_strategy | [06 §4](06-search-and-retrieval.md#4-字段配置) |
| Fingerprint chain：upstream → cache → chunk → embedding，分层失效 | [04 §5.2](04-connector-and-ingest.md#52-framework-内部per-artifact-fingerprint-chain) |
| HTTP 只走 control plane；client 上传只在 remote profile + 本地路径时发生 | [02 §4](02-architecture.md#4-控制面-vs-数据面) |
| 社区贡献新 connector ~500-1500 行 Python，集中在 `connectors/<name>/` | [07](07-contributing-connector.md) |

## 阅读顺序

按"从顶层 → 细节、从架构 → 命令、从用户 → 贡献者"递进：

| # | 文档 | 内容 | 适合谁 |
|---|---|---|---|
| 01 | [01-overview.md](01-overview.md) | 定位、抽象、系统图、决策索引 | 所有人 |
| 02 | [02-architecture.md](02-architecture.md) | client/server、profile、存储、队列、一致性、并发、多租户 | 所有人，运维和贡献者重点看 |
| 03 | [03-cli-commands.md](03-cli-commands.md) | 16 个公开命令、行为契约、JSON envelope、错误码 | 用户、agent 集成方 |
| 04 | [04-connector-and-ingest.md](04-connector-and-ingest.md) | connector 注册、ingest 流程、fingerprint chain、checkpoint | 用户、贡献者 |
| 05 | [05-browse-and-read.md](05-browse-and-read.md) | ls/tree/cat/head/tail/grep 的后台行为、cache、大对象 | 用户、agent skill 作者 |
| 06 | [06-search-and-retrieval.md](06-search-and-retrieval.md) | Milvus schema、chunk_kind、locator、字段配置、preset | 用户（高级配置）、贡献者 |
| 07 | [07-contributing-connector.md](07-contributing-connector.md) | 插件接口、骨架、对象命名规范 | 贡献者 |
| 08 | [08-agent-skill.md](08-agent-skill.md) | 给 LLM agent skill 作者的指南 | Skill 作者、prompt 工程师 |
| 09 | [09-connector-catalog.md](09-connector-catalog.md) | 每类 connector 的虚拟目录布局清单 | 用户、贡献者参考 |
| 10 | [10-packaging-and-deployment.md](10-packaging-and-deployment.md) | 打包、部署形态、发版、工程目录、运维 | 运维、维护者 |
| —  | [18-project-structure-flow.html](18-project-structure-flow.html) | 可交互目录树和信息流（HTML） | 所有人 |

## 与 Mirage 的关系

参考了 [strukto-ai/mirage](https://github.com/strukto-ai/mirage) 的两个核心思路：

- 每个 connector 自定义虚拟目录布局，用 PROMPT 描述给 agent 看
- 文件名带 media type 后缀（`rows.jsonl` / `schema.json`），cat 按后缀渲染

MFS 跟 Mirage 的核心区别：

| 维度 | Mirage | MFS |
|---|---|---|
| 协议 | FUSE mount + VFP | HTTP /v1，不挂载 |
| 检索 | 不做 | 混合检索（向量 + BM25） |
| 索引产物 | 仅 TTL cache | 持久化 cache + Milvus chunks |
| LLM/VLM 增强 | 不做 | summary、VLM description |
| 目标用户 | 想把数据源当文件系统的 agent | 想用 shell 命令做语义搜索的 agent |
